# 第21章 同じサイズだけなら簡単だ 〔v0.14:プールアロケーター〕

---

## この章のゴール

第20章で、個別解放が生む4つの問題を洗い出しました。

1. どこが空いたか覚える
2. サイズが分からない
3. 入る穴を探す(O(n))
4. 断片化

**このうち3つは、たった1つの制約で消えます。**

> **すべてのブロックが同じサイズなら。**

サイズが同じなら、「入る穴を探す」必要がありません。どの穴でも入ります。分割も合体も起きません。断片化は **原理的に発生しません**。

残るのは「空きをどう記録するか」だけ。そしてそれも、驚くほど安く済みます。

- **侵入的フリーリスト** という技法を理解する
- `Pool<T>` を30行で書く 〔**v0.14**〕
- パーティクル1万個で `new` と勝負する
- 確保速度だけでなく **走査速度** も比べる(こちらのほうが重要です)
- 何を捨てたのかを正直に整理する

---

## 21.1 サイズを固定すると何が消えるか

第20章の4つの問題を、順に潰していきます。

### 問題3(探索)が消える

```
空きブロック: [■][■][■][■]
要求: 1ブロック
```

**どれを選んでも同じです。** 探す必要がありません。先頭のものを取ればいい。**O(1)。**

first fit も best fit も存在しません。すべてのブロックが等価だからです。第20章で悩んだ配置方針の議論が、まるごと消滅します。

### 問題4(断片化)が消える

断片化とは、「合計では足りているのに、連続した領域が取れない」現象でした。

**要求が常に1ブロックなら、連続性は問題になりません。** 空きブロックが1つでもあれば、必ず確保できます。

```
外部断片化 = 常に 0
```

これは第19章で定義した指標が、構造的に 0 になるということです。バンプアロケーターと同じ性質を、**個別解放を許しながら** 保てます。

### 問題2(サイズが分からない)が消える

```cpp
void Deallocate(T* p);
```

サイズは `sizeof(T)` です。**ヘッダが要りません。**

第20章で「16 バイトのデータに 16 バイトのヘッダ」という悲惨な例を挙げましたが、プールではその問題自体が存在しません。

### 残るのは問題1(空きの記録)だけ

空いているブロックの集合を、どこかに持つ必要があります。

素朴に考えると、`std::vector<bool>` や `std::vector<T*>` を用意することになります。しかし、それでは追加のメモリが必要です。

**もっと良い方法があります。**

---

## 21.2 侵入的フリーリスト

**空いているブロックは、誰も使っていません。**

ならば、そこに何を書いても構いません。**次の空きブロックへのポインタを書きましょう。**

```
free_ ──┐
        ▼
   ┌────────┬────────┬────────┬────────┬────────┐
   │ 使用中  │ next──┐│ 使用中  │ next──┐│ next=∅ │
   └────────┴───┬───┴┴────────┴───┬───┴┴────────┘
                │                  │        ▲
                └──────────────────┴────────┘
```

空きブロック同士が、**自分自身の中に書かれたポインタ** で数珠つなぎになっています。

### 何が嬉しいのか

| | |
|---|---|
| 追加のメモリ | **ゼロ** |
| 確保 | 先頭を取って `free_` を進める。**O(1)** |
| 解放 | 先頭に繋ぎ直す。**O(1)** |
| 探索 | **なし** |

管理情報を、管理対象の内部に埋め込む。これを **侵入的**(intrusive)と呼びます。

第11章の破棄リストで、ノードをアリーナ内に置いたのと同じ発想です。ただし今回は、**さらに徹底しています**。ノード用の領域すら確保せず、空きブロックそのものをノードとして使い回しています。

### 制約

この技法には条件があります。

```
ブロックサイズ  >= sizeof(void*)      … ポインタが書けること
ブロックの整列  >= alignof(void*)     … ポインタが正しく置けること
```

x64 なら、**8 バイト以上・8 バイト境界** です。

`sizeof(T)` が 8 未満の型(たとえば `char` や `short`)では、ブロックを 8 バイトに広げる必要があります。**小さすぎる型では無駄が出る** ということです。

実際には、プールの対象になるのはパーティクル、敵、弾、ノードといった構造体なので、この制約が問題になることはほとんどありません。

---

## 21.3 `Pool<T>` を書く

```cpp
// ga/Pool.h
#pragma once

#include "ga/Core.h"

#include <cstddef>
#include <memory>
#include <utility>
#include <vector>

namespace ga
{
    namespace detail
    {
        struct FreeNode { FreeNode* next; };
    }

    template <class T>
    class Pool
    {
    public:
        static constexpr std::size_t kBlockSize =
            (sizeof(T)  > sizeof(detail::FreeNode))  ? sizeof(T)  : sizeof(detail::FreeNode);

        static constexpr std::size_t kBlockAlign =
            (alignof(T) > alignof(detail::FreeNode)) ? alignof(T) : alignof(detail::FreeNode);

        explicit Pool(std::size_t capacity)
            : buffer_(capacity * kBlockSize + kBlockAlign)
            , capacity_(capacity)
        {
            const auto raw = reinterpret_cast<std::uintptr_t>(buffer_.data());
            base_ = reinterpret_cast<std::byte*>(AlignUp(raw, kBlockAlign));

            // 末尾から前へ繋ぐと、確保はアドレスの小さい順になる
            for (std::size_t i = capacity_; i > 0; --i)
            {
                std::byte* block = base_ + (i - 1) * kBlockSize;
                free_ = std::construct_at(reinterpret_cast<detail::FreeNode*>(block),
                                          detail::FreeNode{ free_ });
            }
        }

        // --- 記憶域だけを配る ---
        [[nodiscard]] T* Allocate() noexcept
        {
            if (free_ == nullptr) { return nullptr; }

            detail::FreeNode* node = free_;
            free_ = node->next;
            ++live_;

            return reinterpret_cast<T*>(node);
        }

        void Deallocate(T* p) noexcept
        {
            if (p == nullptr) { return; }

            free_ = std::construct_at(reinterpret_cast<detail::FreeNode*>(p),
                                      detail::FreeNode{ free_ });
            --live_;
        }

        // --- 構築つき ---
        template <class... Args>
        [[nodiscard]] T* New(Args&&... args)
        {
            T* p = Allocate();
            if (p == nullptr) { return nullptr; }

            return std::construct_at(p, std::forward<Args>(args)...);
        }

        void Delete(T* p) noexcept
        {
            if (p == nullptr) { return; }

            std::destroy_at(p);
            Deallocate(p);
        }

        std::size_t Capacity()  const noexcept { return capacity_; }
        std::size_t Live()      const noexcept { return live_; }
        std::size_t Available() const noexcept { return capacity_ - live_; }

    private:
        std::vector<std::byte> buffer_;
        std::byte*             base_     = nullptr;
        detail::FreeNode*      free_     = nullptr;
        std::size_t            capacity_ = 0;
        std::size_t            live_     = 0;
    };
}
```

コメントと空行を除けば **約40行**。核心の `Allocate` と `Deallocate` は、それぞれ4行です。

### 設計上のポイント

**末尾から繋いでいる理由。** コンストラクタのループが `capacity_` から 1 へ降りているのは、フリーリストの順序を **アドレスの昇順** にするためです。

こうすると、最初の1万回の確保が **メモリ上で連続** します。前から繋ぐと、確保は末尾から始まり降順になります。動作は同じですが、キャッシュの効きが変わります。**細かいことですが、タダなので良いほうを選びます。**

**戻り値が `T*` で、失敗は `nullptr`。** 第7章では `std::expected` を選びました。ここで方針を変えたのは、**失敗の理由が1つしかない** からです。

`Bump` は「容量不足」「アラインメント不正」「サイズ過大」を区別する必要がありました。`Pool` の失敗は「満杯」だけです。エラーコードを運ぶ価値がありません。

> 第7章で「回復できるものはエラー、回復できないものはバグ」と述べました。ここではさらに一歩進んで、**情報量に見合った表現を選ぶ** という判断をしています。すべてを `expected` にするのが良い設計ではありません。

**`std::construct_at` でフリーノードを作っている。** 空きブロックにポインタを書き込む部分です。

生のメモリに `FreeNode` を「構築」しています。`T` の生存期間はすでに終わっているので、同じ場所に別の型のオブジェクトを作ってよい——というのが C++ の規則です。厳密な議論は第42章で扱います。

---

## 21.4 動かす

```cpp
struct Particle
{
    float x = 0, y = 0, z = 0;
    float vx = 0, vy = 0, vz = 0;
    float life = 0;
    int   id = 0;
};

int main()
{
    ga::Pool<Particle> pool(1000);

    std::println("容量: {}  ブロックサイズ: {}", pool.Capacity(), pool.kBlockSize);

    // 3個確保
    Particle* a = pool.New();
    Particle* b = pool.New();
    Particle* c = pool.New();

    a->id = 1; b->id = 2; c->id = 3;

    std::println("確保後: live={} available={}", pool.Live(), pool.Available());
    std::println("アドレス差: b-a={} c-b={}",
                 reinterpret_cast<std::byte*>(b) - reinterpret_cast<std::byte*>(a),
                 reinterpret_cast<std::byte*>(c) - reinterpret_cast<std::byte*>(b));

    // 真ん中を解放
    pool.Delete(b);
    std::println("b 解放後: live={}", pool.Live());

    // もう1つ確保すると、b の場所が返ってくる
    Particle* d = pool.New();
    std::println("d == b の場所? {}", static_cast<void*>(d) == static_cast<void*>(b));
}
```

```
容量: 1000  ブロックサイズ: 32
確保後: live=3 available=997
アドレス差: b-a=32 c-b=32
b 解放後: live=2
d == b の場所? true
```

**解放したブロックが、次の確保で即座に再利用されました。**

これは LIFO(後入れ先出し)の動作です。フリーリストの先頭に繋ぎ、先頭から取るので、こうなります。

**キャッシュの観点では望ましい性質です。** 直前まで使っていた領域は、まだキャッシュに乗っています。

### テスト

```cpp
void Test_PoolBasics()
{
    ga::Pool<Particle> pool(4);

    assert(pool.Capacity() == 4);
    assert(pool.Live() == 0);

    Particle* p[4];
    for (int i = 0; i < 4; ++i)
    {
        p[i] = pool.New();
        assert(p[i] != nullptr);
    }

    assert(pool.Live() == 4);
    assert(pool.Available() == 0);

    // 満杯なら nullptr
    assert(pool.New() == nullptr);

    // 1つ返せば1つ取れる
    pool.Delete(p[1]);
    assert(pool.Live() == 3);

    Particle* q = pool.New();
    assert(q == p[1]);          // 同じ場所が返る

    std::println("[ OK ] Test_PoolBasics");
}

void Test_PoolBlocksAreContiguous()
{
    ga::Pool<Particle> pool(100);

    std::vector<Particle*> ptrs;
    for (int i = 0; i < 100; ++i) { ptrs.push_back(pool.New()); }

    // 昇順に、ブロックサイズちょうど間隔で並んでいる
    for (std::size_t i = 1; i < ptrs.size(); ++i)
    {
        const auto diff = reinterpret_cast<std::byte*>(ptrs[i])
                        - reinterpret_cast<std::byte*>(ptrs[i - 1]);
        assert(diff == static_cast<std::ptrdiff_t>(ga::Pool<Particle>::kBlockSize));
    }

    std::println("[ OK ] Test_PoolBlocksAreContiguous");
}

void Test_PoolExternalFragmentationIsZero()
{
    ga::Pool<Particle> pool(1000);

    std::vector<Particle*> ptrs;
    for (int i = 0; i < 1000; ++i) { ptrs.push_back(pool.New()); }

    // ランダムに半分解放する
    std::mt19937 rng(42);
    std::shuffle(ptrs.begin(), ptrs.end(), rng);

    for (int i = 0; i < 500; ++i) { pool.Delete(ptrs[i]); }

    // どれだけ穴だらけでも、500個は必ず確保できる
    for (int i = 0; i < 500; ++i)
    {
        assert(pool.New() != nullptr);
    }

    std::println("[ OK ] Test_PoolExternalFragmentationIsZero");
}
```

**3つ目のテストが重要です。** 第20章の紙のシミュレーションでは、ランダムに解放した結果 5 マスが取れませんでした。プールでは、**どれだけばらばらに解放しても、空き数ぶんは必ず確保できます**。

---

## 21.5 パーティクル1万個で勝負

### 確保と解放の速度

```cpp
constexpr std::size_t kCount = 10'000;

// --- Pool 版 ---
ga::Pool<Particle> pool(kCount);
std::vector<Particle*> ptrs(kCount);

auto rPool = bench::MeasureBatch(200, kCount, [&, i = std::size_t{0}]() mutable {
    if (i < kCount) { ptrs[i] = pool.New(); }
    if (++i == kCount)
    {
        for (auto* p : ptrs) { pool.Delete(p); }
        i = 0;
    }
    bench::Escape(ptrs[0]);
});

// --- new 版 ---
auto rNew = bench::MeasureBatch(200, kCount, [&, i = std::size_t{0}]() mutable {
    if (i < kCount) { ptrs[i] = new Particle(); }
    if (++i == kCount)
    {
        for (auto* p : ptrs) { delete p; }
        i = 0;
    }
    bench::Escape(ptrs[0]);
});
```

```
Pool::New / Delete    median=      2.4  p95=      2.6  max=        5.1
new / delete          median=     21.8  p95=     22.9  max=     3140.0
```

**9倍速い。** 最大値では 600 倍以上の差があります。

これは第5章で `Bump` と `new` を比べたときと同じ傾向です。**やっている仕事の量が違う** からです。

### ここからが本番:走査速度

**実は、確保速度より重要な数字があります。**

パーティクルシステムは、毎フレーム全パーティクルを更新します。1万個を1回ずつ走査する処理が、毎フレーム走ります。確保は1回でも、走査は何千回も繰り返されます。

現実に近い条件で比べます。`new` 版では、パーティクルの確保の間に **他の確保が混ざる** ようにします。実際のプログラムでは、パーティクルだけを連続して確保することはないからです。

```cpp
// --- new 版:間に他の確保を挟む(現実に近い条件) ---
std::vector<Particle*> newPtrs;
std::vector<std::string*> noise;

for (std::size_t i = 0; i < kCount; ++i)
{
    newPtrs.push_back(new Particle());
    noise.push_back(new std::string(64, 'x'));    // ノイズ
}

// --- Pool 版 ---
ga::Pool<Particle> pool(kCount);
std::vector<Particle*> poolPtrs;
for (std::size_t i = 0; i < kCount; ++i) { poolPtrs.push_back(pool.New()); }

// --- 走査(更新処理)---
auto Update = [](std::vector<Particle*>& ps) {
    for (Particle* p : ps)
    {
        p->x += p->vx;
        p->y += p->vy;
        p->z += p->vz;
        p->life -= 0.016f;
    }
};

auto rScanNew  = bench::MeasureBatch(500, kCount, [&] { /* new 版を走査 */ });
auto rScanPool = bench::MeasureBatch(500, kCount, [&] { /* Pool 版を走査 */ });
```

```
走査 (new 版、散在)   median=      3.8 ns/個
走査 (Pool 版、連続)  median=      0.4 ns/個
```

**9.5 倍の差。** しかもこれは **毎フレーム** 発生します。

### なぜこれほど違うのか

`Pool` のパーティクルは、メモリ上に 32 バイト間隔で連続して並んでいます。

```
キャッシュライン(64バイト)
┌───────────────┬───────────────┐
│  Particle[0]  │  Particle[1]  │   ← 1回の読み込みで2個
└───────────────┴───────────────┘
```

**1回のメモリアクセスで、2個ぶんが読み込まれます。** さらに CPU のプリフェッチャが「順番に読んでいる」と判断し、次の領域を先読みします。

`new` 版では、パーティクルの間に `std::string` が挟まっています。

```
┌───────────────┬───────────────┐
│  Particle[0]  │  string(未使用) │   ← 半分が無駄
└───────────────┴───────────────┘
```

キャッシュラインの半分が無駄になり、プリフェッチも効きません。

> **「アロケーターの仕事は、速く返すことではなく、良い場所を返すこと」**
>
> 第9章で予告した言葉です。ここで初めて、それが数字になりました。第32章で、この現象をさらに詳しく測ります。

### 総合すると

```
1フレームあたりのコスト(1万パーティクル)

  new 版  : 走査 38 µs
  Pool 版 : 走査  4 µs
  ────────────────────────
  差       : 34 µs  = 16.6 ms 予算の 0.2%
```

1つのシステムで 0.2%。パーティクル、敵、弾、当たり判定、AI ノード——同じ構造のものが10種類あれば、2% になります。

**塵も積もれば、という話ではありません。** データ構造の並び方が性能を決める、という話です。

---

## 21.6 断片化を測る

第19章の指標を、プールに当てはめます。

```cpp
    FragmentationStats GetFragmentation() const noexcept
    {
        FragmentationStats f;
        f.capacity       = capacity_ * kBlockSize;
        f.used           = live_ * kBlockSize;
        f.internalWaste  = live_ * (kBlockSize - sizeof(T));
        f.freeTotal      = Available() * kBlockSize;
        f.freeLargest    = kBlockSize;                    // 常に1ブロック分
        f.freeBlockCount = Available();
        return f;
    }
```

### 外部断片化の扱いに注意

素直に式を当てはめると、おかしなことになります。

```
外部断片化 = 1 - (最大の連続空き / 空きの合計)
           = 1 - (32 / 16000)
           = 0.998
```

**指標上は最悪値に近い。** しかし実際には、確保は100%成功します。

これは第19章で述べた「この式は万能ではない」の実例です。**式が想定しているのは「連続領域を要求する」状況** ですが、プールは常に1ブロックしか要求しません。

> **プールにおいては、外部断片化という概念そのものが意味を持ちません。** 指標を機械的に適用せず、何を測っているのか常に意識してください。

### 代わりに測るべきもの

```
=== Pool<Particle> ===
  容量       : 10000 ブロック (320.00 KB)
  使用中     :  3421 ブロック (34.2%)
  ブロック    : 32 バイト (sizeof(T) = 32, 無駄 0)
  ピーク     :  8102 ブロック (81.0%)
```

**ピーク使用率** が最も重要な数字です。次節で説明します。

---

## 21.7 何を捨てたのか

第20章で「第3部は制約を付け替える章」と書きました。プールが捨てたものを整理します。

### 1. サイズの自由

**当然ですが、1つのプールは1つの型しか扱えません。**

パーティクル用、敵用、弾用、AI ノード用……**型ごとにプールが必要** です。

### 2. メモリの共有 ← これが本当の代償

こちらのほうが深刻です。

```
Pool<Particle>  容量 10000 → 320 KB
Pool<Enemy>     容量  1000 → 128 KB
Pool<Bullet>    容量  5000 → 160 KB
Pool<AINode>    容量  2000 →  96 KB
──────────────────────────────────
合計                         704 KB
```

**この 704 KB は、常に確保されっぱなしです。**

パーティクルが 0 個でも、320 KB は敵のために使えません。プールは互いに独立しているからです。

汎用アロケーターなら、その時々で必要なものに融通できます。**プールは融通が利きません。**

```
必要なメモリ = Σ(各プールの最大同時数)
```

各プールの **同時ピーク** の和が必要になります。もし全部のピークが同時に来ないなら、その差は丸ごと無駄です。

### 3. 容量の上限

固定容量なので、超えたら失敗します。

**「では大きめに取ればいい」——それが上の問題を悪化させます。** 安全側に倒すほどメモリを死蔵します。

### 判断の目安

| 状況 | プールが向くか |
|---|---|
| 同時存在数の上限が決まっている | **○** |
| 生成と破棄が頻繁 | **○** |
| 毎フレーム全件走査する | **◎**(走査速度が効く) |
| 同時数が読めない | △ |
| 種類が非常に多い | ×(プールだらけになる) |
| ピークが互いにずれている | ×(メモリの無駄が大きい) |

**ゲームのパーティクル、弾、敵は、ほぼ理想的な適用対象です。** 「最大 N 個まで」という上限は、そもそも設計上決まっていることが多いからです。

---

## 21.8 LIFO 順序の落とし穴

21.4 節で、解放したブロックが即座に再利用されることを見ました。**キャッシュには良い性質** です。

しかし、副作用があります。

```cpp
void ExperimentOrderDegradation()
{
    ga::Pool<Particle> pool(10'000);
    std::vector<Particle*> ptrs;

    // 1) 全部確保 → アドレス昇順
    for (int i = 0; i < 10'000; ++i) { ptrs.push_back(pool.New()); }

    const double before = MeasureScan(ptrs);

    // 2) ランダム順に全解放 → フリーリストの順序が乱れる
    std::mt19937 rng(1);
    std::shuffle(ptrs.begin(), ptrs.end(), rng);
    for (auto* p : ptrs) { pool.Delete(p); }

    // 3) 再確保 → アドレスがばらばらの順で返ってくる
    ptrs.clear();
    for (int i = 0; i < 10'000; ++i) { ptrs.push_back(pool.New()); }

    const double after = MeasureScan(ptrs);

    std::println("初回の走査   : {:.2f} ns/個", before);
    std::println("再確保後の走査: {:.2f} ns/個", after);
}
```

```
初回の走査   : 0.41 ns/個
再確保後の走査: 2.87 ns/個
```

**7倍遅くなりました。**

### 何が起きたか

パーティクル自体は、依然としてメモリ上に連続して置かれています。**変わったのは、ポインタ配列の並び順** です。

```
初回:   ptrs = [&block0, &block1, &block2, ...]   ← 昇順
再確保: ptrs = [&block7382, &block41, &block9901, ...]  ← ばらばら
```

`ptrs` を順に辿ると、メモリ上をランダムに飛び回ることになります。**連続配置の利点が失われました。**

### 対策

**対策1:ポインタではなく、配列として持つ。**

```cpp
std::span<Particle> particles = pool.AllBlocks();   // 全ブロックを配列として走査
for (Particle& p : particles)
{
    if (!p.alive) { continue; }
    Update(p);
}
```

生きているかどうかのフラグを見て飛ばします。**常にアドレス順に走査する** ので、順序が乱れません。

死んでいる要素も読むことになりますが、キャッシュラインは連続して読まれるので、飛び回るよりずっと速いことがほとんどです。

**対策2:定期的に詰め直す。**

生きているものを前に寄せる処理(コンパクション)を、たまに実行します。ただしポインタが無効になるので、第45章のハンドルが必要になります。

**対策3:そもそもポインタを持たない。**

インデックスで持つ、あるいはデータ指向設計に移行する。第53章で触れます。

> **プールを使えば自動的に速くなる、わけではありません。** 連続配置という利点を活かすには、**使う側の書き方** も合わせる必要があります。

---

## 演習

**演習21-1** `Pool<char>` を作ると、ブロックサイズは何バイトになりますか。無駄はどれくらいですか。

**演習21-2** フリーリストを昇順ではなく降順に初期化すると、初回の走査速度は変わりますか。実測してください。

**演習21-3** `Pool` に `Reset()`(全ブロックを空きに戻す)を実装してください。生きているオブジェクトのデストラクタはどうしますか。

**演習21-4** 21.8 の対策1(全ブロックを配列として走査)を実装し、生存率 10% / 50% / 90% のそれぞれで速度を比べてください。どこで逆転しますか。

**演習21-5** 満杯になったら容量を倍にして新しいブロック群を確保する「成長するプール」を設計してください。何が難しくなりますか。

**演習21-6** `Pool<T>` の `Allocate()` を `std::expected<T*, AllocError>` を返す形に変えてください。呼び出し側は読みやすくなりますか。

**演習21-7** 21.7 の「メモリの共有ができない」問題を、実際の数字で見積もってください。10種類のプールを作り、各ピークが同時に来る場合と、ずれる場合を比べてください。

---

## 章末チェックリスト

- [ ] サイズ固定によって第20章の4問題のうち3つが消える理由を説明できる
- [ ] 侵入的フリーリストの仕組みを図で説明できる
- [ ] `Pool<T>` を実装した 〔v0.14〕
- [ ] ブロックサイズが `sizeof(void*)` 以上必要な理由を説明できる
- [ ] `new` との速度差(確保・走査の両方)を測った
- [ ] **走査速度のほうが重要** な理由を説明できる
- [ ] 外部断片化の指標がプールでは意味をなさない理由を説明できる
- [ ] プールが捨てたもの3つを挙げられる
- [ ] LIFO 順序による走査速度の劣化を実演した

---

## 次章の予告

`Pool<T>` は動きます。しかし、実戦に出すには危うい点があります。

```cpp
pool.Delete(p);
pool.Delete(p);      // ← 二重解放。何が起きる?
```

**フリーリストが輪になります。** 同じブロックが2回リストに入るので、その後の確保で **同じアドレスが2回返ります**。2つの別々のオブジェクトが同じ場所を共有し、静かに壊れます。

```cpp
ga::Pool<Particle> poolA(100);
ga::Pool<Particle> poolB(100);

Particle* p = poolA.New();
poolB.Delete(p);     // ← 別のプールに返した。何が起きる?
```

**`poolB` のフリーリストが、`poolA` の領域を指します。** 以降、`poolB` は他人の領域を配り始めます。

第22章では、これらを検出できるようにします。所属チェック、二重解放の検出、そしてビットマップ方式との比較。第2部で作ったデバッグの道具立てを、プールにも展開します。

---

> **コラム:プールの兄弟たち**
>
> 「同じサイズだけ扱う」というアイデアは、単純すぎて誰でも思いつきます。だからこそ、あちこちで独立に発明され、さまざまな形に発展してきました。
>
> ---
>
> **バディシステム(Knowlton, 1965)**
>
> プールが1種類のサイズしか扱えないなら、**サイズごとにプールを用意すればいい**。これが素直な発展です。
>
> バディシステムは、サイズを **2の冪** に限定します。32、64、128、256……というサイズごとにフリーリストを持ちます。
>
> 面白いのは、ここからです。**128 のブロックが要るのに空きがないとき、256 のブロックを2つに割ります。** 割ってできた2つを「バディ(相棒)」と呼びます。
>
> 逆に、両方のバディが空きになったら、**統合して 256 に戻します**。相棒の位置はアドレスから計算だけで求まる——これがこの方式の美しいところです。
>
> つまりバディシステムは、**プールの集合に「分割」と「統合」を足したもの** と見ることができます。プールの融通が利かない問題(21.7)を、サイズ間で融通することで解決しています。
>
> 代償は内部断片化です。33 バイトの要求に 64 バイトのブロックを配ることになり、**最悪でほぼ半分が無駄** になります。第26章で実装し、実測します。
>
> ---
>
> **スラブアロケーター(Bonwick, 1994)**
>
> Solaris の開発者 Jeff Bonwick が発表した方式で、Linux カーネルにも取り入れられました。プールの直系の子孫です。
>
> 発想の核心は、**「オブジェクトを構築済みのまま保持する」** という点にあります。
>
> 通常、確保したメモリにはコンストラクタを走らせ、解放時にデストラクタを走らせます。しかし、同じ型を何度も作り直すなら、**初期化の一部は毎回同じ** です。ならば、解放時に完全に壊さず、「使える状態」で置いておけばいい。
>
> カーネルでは、同じ構造体(inode、ファイル記述子など)が大量に生成・破棄されます。この最適化がよく効きます。
>
> スラブはさらに、**CPU ごとのキャッシュ** を持ちます。これは第31章で扱うスレッドローカルアロケーターの先駆けでもあります。
>
> ---
>
> **オブジェクトプール(ゲーム業界)**
>
> ゲーム開発では、メモリ管理というより **設計パターン** として語られることが多い技法です。
>
> 「弾を100発ぶん確保しておき、発射時に空きを1つ取り、消滅時に返す」。よくあるコードです。多くの場合、アロケーターとしてではなく、**フラグ付きの固定配列** として実装されます。
>
> ```cpp
> struct BulletPool
> {
>     std::array<Bullet, 100> bullets;
>     std::array<bool, 100>   alive;
> };
> ```
>
> これは 21.8 の「対策1」そのものです。**走査を常にアドレス順に行える** という利点があり、実は本章のフリーリスト方式より適している場面が多くあります。
>
> 第22章で、この2つの方式(フリーリスト vs ビットマップ)を正面から比較します。
>
> ---
>
> **共通する洞察**
>
> どの方式も、同じ前提に立っています。
>
> > **同じ型のオブジェクトは、同じサイズである。**
>
> 当たり前のことですが、汎用アロケーターはこの情報を使えません。`malloc(32)` が来たとき、それが `Particle` なのか `char[32]` なのかを知る術がないからです。
>
> **私たちは知っています。** `Pool<Particle>` という型に、その知識が書かれています。第5章から繰り返している「知識を性能に変換する」の、最も直接的な形です。
