# 第12章 配列と `std::span` 〔v0.8〕

---

## この章のゴール

オブジェクトを1つ作れるようになりました。しかしゲームで扱うのは、たいてい **たくさん** です。

```cpp
auto particles = bump.NewArray<Particle>(10'000);
```

こう書けるようにします。

- `std::span<T>` を返す理由を理解する
- **掛け算の桁溢れ** を防ぐ(第7章で定義した `SizeTooLarge` が、ついに出番を迎えます)
- `n` 個の構築を、例外安全に行う
- 破棄リストのノードを **配列ごとに1本** で済ませる 〔**v0.8**〕
- 範囲 for と `std::ranges` をそのまま使う
- 境界外アクセスを Debug 構成で検出する
- 初期化コストという、配列特有の問題を測る

この章が終わると、アリーナ上のデータが「普通の C++」として扱えるようになります。

---

## 12.1 今の API で配列を確保すると

現状でも、やろうと思えばできます。

```cpp
constexpr std::size_t kCount = 10'000;

auto r = bump.Allocate(sizeof(Particle) * kCount, alignof(Particle));
if (!r) { return; }

Particle* particles = static_cast<Particle*>(*r);

// 構築は自分で
for (std::size_t i = 0; i < kCount; ++i)
{
    std::construct_at(particles + i);
}

// 使う
for (std::size_t i = 0; i < kCount; ++i)
{
    particles[i].Update();
}
```

問題が4つあります。

**1. `sizeof(Particle) * kCount` が桁溢れしうる。** 誰も検査していません。

**2. 要素数が失われる。** 返ってくるのは `Particle*` だけです。要素数は呼び出し側が別途覚えておかなければならず、渡すときも2つセットで持ち回る必要があります。

```cpp
void Update(Particle* particles, std::size_t count);   // 2引数
```

しかも、片方だけ間違えてもコンパイルは通ります。

**3. 構築ループを毎回書く。** 途中で例外が飛んだら、すでに構築した要素はどうなるのか。誰も考えていません。

**4. 範囲 for が使えない。** `for (auto& p : particles)` と書けません。`std::ranges::sort` も渡せません。

---

## 12.2 `std::span<T>` を返す

C++20 で入った `std::span<T>` は、**ポインタと要素数を1つにまとめた型** です。

```cpp
#include <span>

std::span<Particle> particles = /* ... */;

particles.size();        // 要素数
particles.data();        // 先頭ポインタ
particles[3];            // 要素アクセス
particles.subspan(10);   // 10番目以降
particles.first(5);      // 先頭5個

for (auto& p : particles) { p.Update(); }   // 範囲 for
```

### `span` は所有しない

重要な性質です。`std::span` は **参照するだけ** で、メモリを所有しません。

- コピーしても中身は複製されない(ポインタと個数がコピーされるだけ)
- デストラクタは何もしない
- サイズは 16 バイト(ポインタ + `size_t`)

つまり、**関数の引数として気軽に値渡しできます**。

```cpp
void Update(std::span<Particle> particles);   // 1引数で済む
```

`std::vector<Particle>&` を受け取る関数と違い、`std::array` からも生の配列からも、そして私たちのアリーナからも渡せます。

**アリーナが返す値として、これ以上ないほど適切な型です。** 所有権はアリーナにあり、`span` はその一部を指し示すだけ。役割が明確に分かれます。

---

## 12.3 掛け算が溢れる

`NewArray<T>(count)` が必要とするバイト数は `sizeof(T) * count` です。この掛け算は、**溢れます**。

```cpp
sizeof(Particle) == 32;
count == 0x2000'0000'0000'0000;   // 2^61

32 * 2^61 = 2^66 → std::size_t (64bit) では 0 になる
```

バイト数 0 で確保が **成功** し、要素数 2^61 の `span` が返ります。そこにアクセスすれば、プロセスの全メモリを踏み荒らすことになります。

### これは有名な脆弱性の型です

この種の桁溢れは、実際のソフトウェアで何度も深刻な脆弱性を生んできました。C 標準に `calloc(count, size)` という「個数とサイズを別々に渡す」関数があるのは、まさに **ライブラリ側で掛け算を検査するため** です。

私たちも検査します。

```cpp
if (count > (std::numeric_limits<std::size_t>::max)() / sizeof(T))
{
    return std::unexpected(AllocError::SizeTooLarge);
}
```

**割り算に置き換えるのが定石です。** 「掛けたら溢れるか」を、掛けずに判定します。`sizeof(T)` は必ず 1 以上なので、ゼロ除算の心配もありません。

> **`(std::numeric_limits<...>::max)()` の括弧について**
> `<Windows.h>` が定義する `max` マクロとの衝突を避けるため、関数名を括弧で囲んでいます。第29章で `<Windows.h>` を取り込むとき、この問題が現実になります。`NOMINMAX` を定義しておくのが本筋ですが、ライブラリ側では防御的に書いておくのが安全です。

### 第7章の宿題

第7章で `AllocError::SizeTooLarge` を定義しましたが、使い道がありませんでした(演習7-2)。

**ここが使いどころです。** `OutOfMemory` と区別する価値もあります。前者は「板を大きくすれば解決する」、後者は「そもそも計算が間違っている」。呼び出し側にとって、意味がまったく違います。

---

## 12.4 `n` 個を構築する

要素の構築は、標準ライブラリのアルゴリズムに任せます。`<memory>` にあります。

| 関数 | 動作 |
|---|---|
| `std::uninitialized_value_construct_n(p, n)` | **値初期化**。`int` なら 0 になる |
| `std::uninitialized_default_construct_n(p, n)` | **既定初期化**。`int` なら不定値 |
| `std::uninitialized_fill_n(p, n, value)` | 全要素を `value` のコピーで埋める |

### 例外安全性が保証されている

これらのアルゴリズムには重要な性質があります。

> **途中で例外が飛んだら、すでに構築した要素をすべて破棄してから、例外を投げ直す。**

自分でループを書くと、この処理を手で書かなければなりません。

```cpp
// 自前で書くと、これだけ必要
std::size_t constructed = 0;
try {
    for (; constructed < count; ++constructed) {
        std::construct_at(first + constructed);
    }
} catch (...) {
    for (std::size_t i = constructed; i > 0; --i) {
        std::destroy_at(first + (i - 1));
    }
    throw;
}
```

標準アルゴリズムを使えば1行です。**車輪を再発明しない。**

### 値初期化と既定初期化の違い

これは性能に直結します。

```cpp
struct Particle { float x, y, z, vx, vy, vz; int id; float life; };  // 32 バイト
```

`std::uninitialized_value_construct_n` は、この構造体を **すべてゼロで埋めます**。10,000 個なら 320 KB の `memset` が走ります。

`std::uninitialized_default_construct_n` は、何もしません(自明に既定構築可能な型の場合)。前の内容がそのまま残ります。

どちらが正しいかは、用途によります。

- 直後に全要素へ値を書き込むなら、初期化は **完全な無駄**
- 初期化を忘れて不定値を読むリスクを避けたいなら、値初期化が安全

本書では **既定を値初期化** にし、明示的に速い版も用意します。安全側を既定にする、という方針です。

---

## 12.5 破棄リストは配列ごとに1本

10,000 要素の配列に、10,000 個の `Finalizer` ノードを作るわけにはいきません。**配列全体で1本** にします。

そのために、`Finalizer` に要素数を持たせます。

```cpp
struct Finalizer
{
    void      (*destroy)(void*, std::size_t) noexcept;   // 引数が増えた
    void*       object;
    std::size_t count;                                    // 追加
    Finalizer*  next;
};

template <class T>
void DestroyThunk(void* p, std::size_t n) noexcept
{
    T* array = static_cast<T*>(p);

    // 配列は逆順に破棄する(C++ の delete[] と同じ規則)
    for (std::size_t i = n; i > 0; --i)
    {
        std::destroy_at(array + (i - 1));
    }
}
```

単体オブジェクトは `count = 1` として、同じ仕組みに乗せます。**分岐が増えません。**

ノードは 24 バイトから 32 バイトに増えますが、10,000 要素で1本しか作らないので、実質的なコストは激減します。

| | v0.7 の素朴な方式 | v0.8 |
|---|---|---|
| 10,000 要素のノード数 | 10,000 本 | **1本** |
| ノードのメモリ | 240,000 バイト | **32 バイト** |
| `Reset()` でのリスト走査 | 10,000 回 | **1回** |

### 逆順で破棄する理由

`std::destroy_n` という標準関数もありますが、こちらは **前から順に** 破棄します。

C++ の配列は、`delete[]` するとき **後ろから** 破棄されます。私たちも規格の挙動に合わせておきます。要素同士が参照し合うことは稀ですが、揃えておいて損はありません。

---

## 12.6 実装する 〔v0.8〕

```cpp
#include <limits>
#include <memory>
#include <span>

template <class T>
using ArrayResult = std::expected<std::span<T>, AllocError>;

class Bump
{
public:
    // --- v0.8:配列の確保 ---

    // 値初期化つき(安全側の既定)
    template <class T>
    [[nodiscard]]
    ArrayResult<T> NewArray(std::size_t count)
    {
        auto storage = AllocateArrayStorage<T>(count);
        if (!storage) { return std::unexpected(storage.error()); }
        if (count == 0) { return std::span<T>{}; }

        T* first = *storage;

        if constexpr (std::is_trivially_destructible_v<T>)
        {
            std::uninitialized_value_construct_n(first, count);
            return std::span<T>(first, count);
        }
        else
        {
            auto node = Allocate(sizeof(Finalizer), alignof(Finalizer));
            if (!node) { return std::unexpected(node.error()); }

            std::uninitialized_value_construct_n(first, count);

            auto* f = std::construct_at(
                static_cast<Finalizer*>(*node),
                Finalizer{ &DestroyThunk<T>, first, count, finalizers_ });
            finalizers_ = f;

            return std::span<T>(first, count);
        }
    }

    // 初期化を省略する版(自明な型のみ)
    template <class T>
    [[nodiscard]]
    ArrayResult<T> NewArrayUninit(std::size_t count)
    {
        static_assert(std::is_trivially_default_constructible_v<T> &&
                      std::is_trivially_destructible_v<T>,
                      "初期化を省略できるのは自明な型だけです");

        auto storage = AllocateArrayStorage<T>(count);
        if (!storage) { return std::unexpected(storage.error()); }
        if (count == 0) { return std::span<T>{}; }

        return std::span<T>(*storage, count);
    }

private:
    // 配列用の記憶域を確保する(桁溢れ検査つき)
    template <class T>
    [[nodiscard]]
    std::expected<T*, AllocError> AllocateArrayStorage(std::size_t count) noexcept
    {
        if (count == 0)
        {
            return nullptr;   // 空配列は確保しない
        }

        if (count > (std::numeric_limits<std::size_t>::max)() / sizeof(T))
        {
            return std::unexpected(AllocError::SizeTooLarge);
        }

        auto r = Allocate(sizeof(T) * count, alignof(T));
        if (!r) { return std::unexpected(r.error()); }

        return static_cast<T*>(*r);
    }
};
```

### `New<T>()` も合わせる

`Finalizer` のシグネチャが変わったので、単体版も更新します。

```cpp
            auto* f = std::construct_at(
                static_cast<Finalizer*>(*node),
                Finalizer{ &DestroyThunk<T>, obj, 1, finalizers_ });   // count = 1
```

**`New<T>()` は `NewArray<T>(1)` の特殊な場合** と見ることもできます。実際、そう実装しても構いません(演習12-1)。

---

## 12.7 使ってみる

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
    Bump bump(1024 * 1024);

    auto r = bump.NewArray<Particle>(10'000);
    if (!r)
    {
        std::println("確保失敗: {}", ToString(r.error()));
        return 1;
    }

    std::span<Particle> particles = *r;

    std::println("要素数: {}", particles.size());
    std::println("バイト数: {}", particles.size_bytes());

    // --- 範囲 for がそのまま使える ---
    int id = 0;
    for (auto& p : particles)
    {
        p.id   = id++;
        p.life = 1.0f;
    }

    // --- ranges のアルゴリズムも使える ---
    std::ranges::for_each(particles, [](Particle& p) { p.life -= 0.1f; });

    const auto alive = std::ranges::count_if(particles,
                                             [](const Particle& p) { return p.life > 0.0f; });
    std::println("生存中: {}", alive);

    // --- 部分ビュー ---
    auto firstTen = particles.first(10);
    auto rest     = particles.subspan(10);

    std::println("先頭10個: {} / 残り: {}", firstTen.size(), rest.size());

    // --- 関数に渡すのも簡単 ---
    UpdateAll(particles);
}

void UpdateAll(std::span<Particle> particles)
{
    for (auto& p : particles)
    {
        p.x += p.vx;
        p.y += p.vy;
        p.z += p.vz;
    }
}
```

```
要素数: 10000
バイト数: 320000
生存中: 10000
先頭10個: 10 / 残り: 9990
```

**アリーナ上のデータが、普通の C++ として扱えています。**

### `std::ranges` との相性

`std::span` は連続範囲(`contiguous_range`)なので、ほぼすべての標準アルゴリズムが使えます。

```cpp
std::ranges::sort(particles, {}, &Particle::life);          // life で昇順ソート
std::ranges::fill(particles, Particle{});                    // 全部リセット

auto visible = particles | std::views::filter(
    [](const Particle& p) { return p.life > 0.0f; });        // ビュー
```

**アロケーターを自作したせいで標準ライブラリが使えなくなる**、という事態は避けられました。これは大事な性質です。

### 配列の破棄も確認する

```cpp
struct Tracer
{
    int id;
    explicit Tracer() : id(0) { }
    ~Tracer() { std::println("  Tracer 破棄"); }
};

int main()
{
    Bump bump(4096);

    auto r = bump.NewArray<Tracer>(3);
    std::println("登録数: {}", bump.PendingFinalizerCount());

    std::println("--- Reset ---");
    bump.Reset();
}
```

```
登録数: 1
--- Reset ---
  Tracer 破棄
  Tracer 破棄
  Tracer 破棄
```

**ノードは1本、破棄は3回。** 狙いどおりです。

---

## 12.8 境界外アクセスを検出する

配列を扱う以上、はみ出しは必ず起きます。

### 生ポインタでは何も起きない

```cpp
Particle* raw = particles.data();
raw[10'000].life = 1.0f;      // ← 範囲外。何も言われない
```

板の内側なら、隣のデータが静かに壊れます。板の外なら、第7章で見たヒープ破壊です。

### `std::span` は Debug で検出してくれる

```cpp
particles[10'000].life = 1.0f;   // ← span 経由
```

Debug 構成で実行すると、こうなります。

```
Assertion failed: span index out of range
```

MSVC の標準ライブラリは、Debug 構成(`_ITERATOR_DEBUG_LEVEL` が有効なとき)に `span::operator[]` の範囲を検査します。**タダで手に入る安全網です。**

Release 構成では検査は消え、生ポインタと同じ速度になります。第3章で立てた「テストは Debug、計測は Release」という方針が、ここでも効いています。

### 検出させてみる

```cpp
void ExperimentOutOfBounds()
{
    Bump bump(4096);

    auto r = bump.NewArray<int>(10);
    if (!r) { return; }

    std::span<int> s = *r;

    std::println("s[9]  = {}", s[9]);    // OK
    std::println("s[10] = {}", s[10]);   // ← Debug で止まる
}
```

### イテレータでも守られる

```cpp
auto it = s.begin();
it += 100;      // ← Debug では、ここでも検査が働く
```

範囲 for やアルゴリズムを使っている限り、範囲外に出ることはありません。**添字を手で書く場面を減らすこと自体が、安全性を高めます。**

### さらに厳しくしたい場合

第31章で `VirtualAlloc` によるガードページ、第36章で AddressSanitizer を扱います。そちらは Release 構成でも、板の内側でのはみ出しでも検出できます。

現時点では `std::span` の Debug 検査で十分です。**コストゼロで、書き方を変えるだけで手に入る** のが利点です。

---

## 12.9 コストを測る

### 初期化のコスト

```cpp
int main()
{
    constexpr std::size_t kCount = 100'000;

    Bump bump(kCount * sizeof(Particle) * 2);

    auto rInit = bench::Measure(200, [&] {
        bump.Reset();
        bench::Escape(bump.NewArray<Particle>(kCount)->data());
    });

    auto rUninit = bench::Measure(200, [&] {
        bump.Reset();
        bench::Escape(bump.NewArrayUninit<Particle>(kCount)->data());
    });

    bench::Print("NewArray (値初期化)", rInit);
    bench::Print("NewArrayUninit     ", rUninit);
}
```

```
NewArray (値初期化)   median= 142000.0  p95= 149000.0  max=   310000.0
NewArrayUninit        median=    100.0  p95=    100.0  max=      900.0
```

**1400 倍の差** です。

10万個 × 32 バイト = 3.2 MB のゼロ埋めに 142 µs。メモリ帯域で決まる時間なので、これ以上速くはなりません。

`NewArrayUninit` は 100 ns、つまり **ほぼ確保だけ** です。ポインタを進めるだけなので当然です。

### この差をどう考えるか

「直後に全要素を書き込むなら、値初期化は完全な無駄」という状況は、ゲームでは頻繁に起きます。

```cpp
auto verts = bump.NewArrayUninit<Vertex>(count);   // ゼロ埋めしない
LoadVerticesFromFile(file, *verts);                // すぐ上書きする
```

一方で、初期化を省いた領域を読んでしまうバグは、非常に見つけにくい。第16章で `0xCD` 塗りつぶしを入れると、Debug 構成では「初期化されていない値」がはっきり分かるようになります。

**既定を安全側にし、必要なときだけ速い版を明示的に選ぶ。** それが v0.8 の設計です。

### `std::vector` との比較

```cpp
auto rVector = bench::Measure(200, [&] {
    std::vector<Particle> v(kCount);
    bench::Escape(v.data());
});
```

```
NewArray (値初期化)   median= 142000.0
std::vector           median= 168000.0
NewArrayUninit        median=    100.0
```

`std::vector` との差は 15% 程度です。どちらも時間の大半はゼロ埋めに費やされており、確保処理の差は誤差に埋もれます。

**この結果は、重要な教訓を含んでいます。**

> 確保が 10 倍速くても、初期化に 142 µs かかるなら、全体では 15% しか変わらない。

第5章で見た「`new` より 10 倍速い」は、**確保だけを取り出した数字** でした。実際のプログラムでは、確保の前後にデータの初期化や書き込みがあります。アロケーターの改善が全体に効くかどうかは、**確保がボトルネックになっている場合に限られます**。

だから測るのです。第4章で道具を先に作ったのは、このためでした。

---

## 12.10 この章の完成コード

差分のみ示します。

```cpp
// --- Finalizer に count を追加 ---
struct Finalizer
{
    void      (*destroy)(void*, std::size_t) noexcept;
    void*       object;
    std::size_t count;
    Finalizer*  next;
};

template <class T>
void DestroyThunk(void* p, std::size_t n) noexcept
{
    T* array = static_cast<T*>(p);
    for (std::size_t i = n; i > 0; --i)
    {
        std::destroy_at(array + (i - 1));
    }
}

// --- RunFinalizersUntil の呼び出しも合わせる ---
    void RunFinalizersUntil(Finalizer* stop) noexcept
    {
        while (finalizers_ != stop)
        {
            Finalizer* node = finalizers_;
            finalizers_ = node->next;
            node->destroy(node->object, node->count);
        }
    }
```

`NewArray` / `NewArrayUninit` / `AllocateArrayStorage` は 12.6 節のとおりです。

---

## 演習

**演習12-1** `New<T>(args...)` を `NewArray<T>(1)` を使って実装できますか。できない理由があるとしたら何でしょうか。

**演習12-2** `NewArrayFill<T>(count, value)` を実装してください。`std::uninitialized_fill_n` を使います。

**演習12-3** `NewArray<T>(0)` は空の `span` を返します。`.data()` は何を返しますか。それは安全ですか。

**演習12-4** 桁溢れの検査を外し、`NewArray<Particle>(0x2000'0000'0000'0000)` を呼んでみてください。何が起きますか。

**演習12-5** `std::span<const T>` を返す `NewArrayConst` を作る意味はありますか。どんな場面で役立つでしょうか。

**演習12-6** 12.9 の測定を、要素数 100 / 10,000 / 1,000,000 で行ってください。`NewArray` と `std::vector` の差は、要素数によってどう変わりますか。

**演習12-7** 2次元配列(`width × height`)をアリーナに確保する関数を書いてください。`std::mdspan`(C++23)を使うとどうなりますか。

**演習12-8** `NewArrayUninit<T>` が返した領域を読むと、何が入っていますか。`DumpBytes`(第3章)で確認してください。`Reset()` を挟むと変化しますか。

---

## 章末チェックリスト

- [ ] `NewArray<T>(n)` を実装し、`std::span<T>` を返した 〔v0.8〕
- [ ] 掛け算の桁溢れを、割り算による検査で防いだ
- [ ] `std::uninitialized_*` アルゴリズムの例外安全性を理解した
- [ ] 値初期化と既定初期化の違いを説明できる
- [ ] 破棄リストのノードが配列ごとに1本であることを確認した
- [ ] 範囲 for と `std::ranges` が使えることを確認した
- [ ] Debug 構成で `span` の境界検査が働くことを確認した
- [ ] **初期化コストが確保コストを圧倒する** ことを測定で確認した

---

## 次章の予告

`Bump` はかなり立派になりました。ここまでのコードは、すべて `Playground.cpp` という1つのファイルに詰め込まれています。そろそろ 500 行を超えているはずです。

第13章では、これを **ライブラリとして切り出します**。

- ソリューションに静的ライブラリ `AllocatorLib` を追加する
- 公開ヘッダと内部ヘッダを分ける
- インクルードパスとプリコンパイル済みヘッダを設定する
- テストとベンチマークを別のファイルへ移す

地味な作業ですが、ここを整えておかないと、第20章以降でアロケーターの種類が増えたときに手がつけられなくなります。あわせて、**なぜ本書が C++20 のモジュールを使わないのか** についても、ここで説明します。

第2部(見えるようにする)へ進む前の、区切りの章です。

---

> **コラム:ポインタと個数が離ればなれになった歴史**
>
> C の配列には、有名な弱点があります。関数に渡すと **ポインタに成り下がり、長さの情報が消える** ことです。
>
> ```c
> void f(int arr[10]) {
>     sizeof(arr);   // 10 * sizeof(int) ではなく、ポインタのサイズ
> }
> ```
>
> 引数の `[10]` は、コンパイラにとって何の意味もありません。ドキュメントとしての価値しかない。
>
> 結果、C の API は「ポインタと個数を別々に渡す」という形式に統一されました。
>
> ```c
> void process(int* data, size_t count);
> ```
>
> この形式が、どれだけのバグと脆弱性を生んだかは、想像に難くありません。片方だけ更新して不整合を起こす。個数を渡し忘れる。バイト数と要素数を取り違える。`memcpy` の第3引数を間違える——これは今なお最も多い脆弱性の原因の1つです。
>
> ---
>
> C++ は当初、この問題を `std::vector` で解決しようとしました。しかし `std::vector` は **所有** します。「既にあるメモリを参照したいだけ」という場面には重すぎます。
>
> そこで長い間、各所で似たような型が独自に作られてきました。
>
> - Microsoft の GSL(Guidelines Support Library)の `gsl::span`
> - Google の `absl::Span` と `absl::string_view`
> - Qt の `QSpan`、LLVM の `ArrayRef`
> - ゲームエンジン各社の `TArrayView`、`eastl::span`
>
> **同じものが、何度も何度も作られました。** それだけ必要とされていたということです。
>
> C++17 で `std::string_view` が、C++20 で `std::span` が標準に入り、ようやく決着がつきました。C++23 では多次元版の `std::mdspan` も追加されています。
>
> ---
>
> ここで面白いのは、**アロケーターを自作する私たちにとって、`span` が特に相性が良い** ことです。
>
> アリーナは所有権を握り続けます。個々のオブジェクトを解放する仕組みはありません。だから利用側に渡すのは「参照」で十分であり、むしろ所有権を持つ型を渡してはいけません。`std::vector` を返す設計にしたら、それはアリーナの意味を否定することになります。
>
> **所有はアリーナ、参照は `span`。** 役割がきれいに分かれます。第38章で `std::vector` に自作アロケーターを差し込むときにも、この区別が効いてきます。
