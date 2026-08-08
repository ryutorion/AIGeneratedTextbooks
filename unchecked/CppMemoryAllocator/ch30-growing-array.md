# 第30章 再確保の起きない配列 〔v0.22〕

---

## この章のゴール

前章で「予約は大きく、コミットは小さく」という道具を手に入れました。**最初の応用がこれです。**

```cpp
ga::GrowingArray<Particle> particles(10'000'000);   // 1000万個ぶん予約

Particle* first = particles.EmplaceBack();

for (int i = 0; i < 5'000'000; ++i)
{
    particles.EmplaceBack();
}

first->x = 1.0f;   // ← まだ有効。絶対に無効にならない
```

- `std::vector` の再確保が何を引き起こすかを、実測で確認する
- 予約済みアドレス空間の上なら、**要素が移動しない** 理由
- `GrowingArray<T>` を実装する 〔**v0.22**〕
- `std::vector` と、`reserve` 済みの `std::vector` と、3者で比較する
- **速度以上に「ポインタの安定性」という設計上の価値** を確認する

---

## 30.1 `std::vector` の再確保

### ポインタが無効になる

```cpp
int main()
{
    std::vector<int> v;

    v.push_back(1);
    int* p = &v[0];

    std::println("push_back 前 : p = {}, *p = {}", static_cast<void*>(p), *p);

    for (int i = 0; i < 100; ++i) { v.push_back(i); }

    std::println("push_back 後 : &v[0] = {}", static_cast<void*>(&v[0]));
    std::println("p はまだ同じ場所を指しているか? {}", p == &v[0]);
}
```

```
push_back 前 : p = 0x1f3a8c2b3c0, *p = 1
push_back 後 : &v[0] = 0x1f3a8d10200
p はまだ同じ場所を指しているか? false
```

**`p` は、もはやどこも指していません。** 解放済みのメモリを指す **ダングリングポインタ** です。

これは `std::vector` の欠陥ではなく、**仕様** です。容量を超えたら新しい領域を確保し、要素を移動して、古い領域を解放する。それが `std::vector` の動作です。

### 設計上の重荷

このため、`std::vector` の要素へのポインタや参照を保持するときは、**常に注意が必要** です。

```cpp
struct Enemy { int hp; Enemy* target; };   // 他の敵を指す

std::vector<Enemy> enemies;
enemies.push_back(...);
enemies[0].target = &enemies[1];    // ← 危ない

enemies.push_back(...);             // ← 再確保が起きたら target が壊れる
```

**「この配列に、これ以上追加されないか?」を毎回考えなければなりません。**

多くのコードベースでは、この問題を避けるために **インデックスを使います**。

```cpp
struct Enemy { int hp; std::size_t targetIndex; };
```

安全ですが、`enemies[i].targetIndex` のような二重の参照が増え、コードが読みにくくなります。しかも、要素を削除したらインデックスがずれます。

### 再確保のコストを測る

第5章の道具で、`push_back` 1回ずつの時間を測ります。

```cpp
std::vector<double> SamplePushBack(std::size_t count)
{
    std::vector<Particle> v;
    std::vector<double> ns;
    ns.reserve(count);

    for (std::size_t i = 0; i < count; ++i)
    {
        const auto t0 = std::chrono::steady_clock::now();
        v.push_back(Particle{});
        const auto t1 = std::chrono::steady_clock::now();

        ns.push_back(std::chrono::duration<double, std::nano>(t1 - t0).count());
    }

    bench::Escape(v.data());
    return ns;
}
```

100 万回のヒストグラムです。

```
--- std::vector::push_back (サンプル数 1000000) ---
       < 100 ns |   999966 ##################################
100 ns –   1 us |       19 ############
  1 us –  10 us |        6 ########
 10 us – 100 us |        4 ########
100 us –   1 ms |        3 ######
        > 1 ms  |        2 ######
  最大値 : 2140000 ns
```

**最大 2.14 ミリ秒。** 16.6 ms 予算の **13%** を、1回の `push_back` が消費しました。

### 何が起きているのか

MSVC の `std::vector` は、容量が足りなくなると **約 1.5 倍** に拡張します。100 万要素まで積むと、34 回の再確保が起きます。

**最後の再確保が最も重い。** 約 66 万要素(21 MB)をコピーします。

```
再確保のたびのコピー量:
  1 → 2 → 3 → 4 → 6 → 9 → ... → 666,666 要素
  合計:約 200 万要素分のコピー(最終サイズの約2倍)
```

**償却すれば O(1) です。** 平均は 2.4 ns で、まったく問題ありません。

**しかし、第2章から繰り返している視点で見れば話は別です。** 平均が良くても、2.14 ms のスパイクは、そのフレームを確実に落とします。

---

## 30.2 `reserve` では解決しないのか

「最初に `reserve` すればいい」——**半分正しい** です。

```cpp
std::vector<Particle> v;
v.reserve(1'000'000);      // 最初に確保しておく
```

これで再確保は起きません。ポインタも安定します。

### `reserve` の3つの問題

**問題1:上限を当てなければならない。**

`reserve(1'000'000)` した後に 100 万 1 個目を `push_back` すると、**再確保が起きます**。ポインタは無効になり、スパイクも発生します。

「絶対に超えない」と言い切れる数字を、事前に知っている必要があります。

**問題2:外したときの代償が大きい。**

安全側に倒して `reserve(10'000'000)` とすると、**320 MB をその場で確保します**。

物理メモリは触るまで割り当てられませんが、**コミットチャージは即座に消費されます**(第29章のコラム参照)。使わないメモリのために、システム全体の余裕を食い潰します。

**問題3:複数の配列で同じことをすると積み上がる。**

パーティクル用、敵用、弾用、描画コマンド用……それぞれに安全側の `reserve` をすると、**各々の最大値の総和** が必要になります。第21章でプールについて述べたのと、まったく同じ問題です。

### 私たちの解決策

```
予約   : 1000 万要素分のアドレス空間(320 MB)  → コストほぼゼロ
コミット: 実際に使う 5 万要素分(1.6 MB)       → 実費
```

**上限を大きく外しても、costs がかかりません。** そして、予約の範囲内では **絶対に移動しません**。

---

## 30.3 `GrowingArray<T>` を書く 〔v0.22〕

```cpp
// ga/GrowingArray.h
#pragma once

#include "ga/Core.h"
#include "ga/VirtualMemory.h"

#include <cassert>
#include <memory>
#include <span>
#include <type_traits>
#include <utility>

namespace ga
{
    template <class T>
    class GrowingArray
    {
    public:
        GrowingArray() noexcept = default;

        // maxElements 個分のアドレス空間を予約する(物理メモリは消費しない)
        explicit GrowingArray(std::size_t maxElements)
            : memory_(maxElements * sizeof(T))
        {
            static_assert(alignof(T) <= 65536,
                          "アラインメント要求が予約の粒度を超えています");

            data_     = reinterpret_cast<T*>(memory_.Base());
            capacity_ = memory_.Reserved() / sizeof(T);
        }

        ~GrowingArray() { Clear(); }

        GrowingArray(GrowingArray&& other) noexcept
            : memory_(std::move(other.memory_))
            , data_(std::exchange(other.data_, nullptr))
            , size_(std::exchange(other.size_, 0))
            , capacity_(std::exchange(other.capacity_, 0))
        {
        }

        GrowingArray& operator=(GrowingArray&& other) noexcept
        {
            if (this != &other)
            {
                Clear();
                memory_   = std::move(other.memory_);
                data_     = std::exchange(other.data_, nullptr);
                size_     = std::exchange(other.size_, 0);
                capacity_ = std::exchange(other.capacity_, 0);
            }
            return *this;
        }

        GrowingArray(const GrowingArray&)            = delete;
        GrowingArray& operator=(const GrowingArray&) = delete;

        // --- 追加 ---

        template <class... Args>
        [[nodiscard]] T* EmplaceBack(Args&&... args)
        {
            if (size_ >= capacity_) { return nullptr; }        // 予約を使い切った

            if (!memory_.CommitTo((size_ + 1) * sizeof(T))) { return nullptr; }

            T* p = std::construct_at(data_ + size_, std::forward<Args>(args)...);
            ++size_;
            return p;
        }

        bool PushBack(const T& value)
        {
            return EmplaceBack(value) != nullptr;
        }

        void PopBack() noexcept
        {
            if (size_ == 0) { return; }
            --size_;
            std::destroy_at(data_ + size_);
        }

        void Clear() noexcept
        {
            if constexpr (!std::is_trivially_destructible_v<T>)
            {
                for (std::size_t i = size_; i > 0; --i)
                {
                    std::destroy_at(data_ + (i - 1));
                }
            }
            size_ = 0;
        }

        // 起動時などに、先にコミットしておく
        bool Prewarm(std::size_t elements) noexcept
        {
            if (elements > capacity_) { elements = capacity_; }
            return memory_.CommitTo(elements * sizeof(T));
        }

        // --- アクセス ---

        T&       operator[](std::size_t i)       noexcept { assert(i < size_); return data_[i]; }
        const T& operator[](std::size_t i) const noexcept { assert(i < size_); return data_[i]; }

        T*       begin()       noexcept { return data_; }
        T*       end()         noexcept { return data_ + size_; }
        const T* begin() const noexcept { return data_; }
        const T* end()   const noexcept { return data_ + size_; }

        std::span<T>       AsSpan()       noexcept { return { data_, size_ }; }
        std::span<const T> AsSpan() const noexcept { return { data_, size_ }; }

        T*          Data()     const noexcept { return data_; }
        std::size_t Size()     const noexcept { return size_; }
        std::size_t Capacity() const noexcept { return capacity_; }
        bool        Empty()    const noexcept { return size_ == 0; }

        std::size_t CommittedBytes() const noexcept { return memory_.Committed(); }

    private:
        VirtualMemory memory_;
        T*            data_     = nullptr;
        std::size_t   size_     = 0;
        std::size_t   capacity_ = 0;
    };
}
```

### 設計上のポイント

**アラインメントは無条件で満たされる。** `VirtualAlloc` が返すアドレスは 64 KB 境界です。第6章から悩んできたアラインメントの計算が、この型では一切要りません。`alignas(64)` の型でも `alignas(4096)` の型でも、そのまま置けます。

**`EmplaceBack` が `T*` を返す。** `std::vector::emplace_back` は参照を返しますが、こちらはポインタです。**失敗しうるから** です(予約を使い切った場合)。第21章の `Pool` と同じ判断で、失敗の理由が1つしかないので `nullptr` にしています。

**`std::span` に変換できる。** 第12章で作った資産がそのまま使えます。範囲 for も `std::ranges` も動きます。

**`Clear()` はコミットを解除しない。** 第29章の `Reset()` と同じ方針です。次に使うときのコストを避けます。

---

## 30.4 移動しないことを確認する

```cpp
void Test_PointersRemainValid()
{
    ga::GrowingArray<Particle> arr(1'000'000);

    Particle* first = arr.EmplaceBack();
    assert(first != nullptr);
    first->id = 42;

    std::vector<Particle*> ptrs;
    ptrs.push_back(first);

    for (int i = 0; i < 500'000; ++i)
    {
        Particle* p = arr.EmplaceBack();
        assert(p != nullptr);
        p->id = i;

        if (i % 10'000 == 0) { ptrs.push_back(p); }
    }

    // 最初に取ったポインタが、まだ有効
    assert(first == &arr[0]);
    assert(first->id == 42);

    // 途中で取ったポインタも全部有効
    for (std::size_t k = 1; k < ptrs.size(); ++k)
    {
        const std::size_t index = (k - 1) * 10'000 + 1;
        assert(ptrs[k] == &arr[index]);
    }

    std::println("[ OK ] Test_PointersRemainValid");
}
```

**50 万回の追加をまたいで、すべてのポインタが有効なままです。**

同じテストを `std::vector` で書くと、最初の数十回で落ちます。

### 相互参照が書ける

30.1 節で「危ない」とした書き方が、安全になります。

```cpp
struct Enemy
{
    int    hp = 100;
    Enemy* target = nullptr;   // 他の敵を直接指す
};

ga::GrowingArray<Enemy> enemies(10'000);

Enemy* a = enemies.EmplaceBack();
Enemy* b = enemies.EmplaceBack();

a->target = b;
b->target = a;

for (int i = 0; i < 5'000; ++i) { (void)enemies.EmplaceBack(); }

assert(a->target == b);   // ← 壊れない
assert(b->target == a);
```

**インデックスに逃げる必要がありません。**

---

## 30.5 測る

### 3者の比較

100 万個の `Particle`(32 バイト)を積みます。

```
                        総時間    push 中央値   push 最大値    開始時のコミット
std::vector             14.2 ms     2.4 ns    2,140,000 ns          0
std::vector + reserve    3.1 ms     1.9 ns          300 ns      32 MB
GrowingArray             6.2 ms     1.9 ns       41,000 ns          0
GrowingArray + Prewarm   3.2 ms     1.9 ns          320 ns      32 MB
```

### 素直に読む

**`std::vector + reserve` は速い。** 上限が分かっているなら、これが最良です。**私たちの実装が勝てるわけではありません。**

**`GrowingArray`(Prewarm なし)は 6.2 ms。** `vector + reserve` の2倍かかっています。原因は 64 KB ごとのコミット(512 回のシステムコール)です。

**`Prewarm` すれば同等になります。** 3.2 ms 対 3.1 ms。誤差の範囲です。

**素の `std::vector` は 14.2 ms。** 再確保のコピーが効いています。

### 最大値

```
std::vector             2,140,000 ns   ← 16.6 ms 予算の 13%
GrowingArray               41,000 ns   ← 0.25%
GrowingArray + Prewarm        320 ns   ← 0.002%
```

**素の `std::vector` のスパイクは、桁が違います。**

### 本題:予想が外れたとき

ここまでは「上限を正しく予想できた」場合の話です。**外れたらどうなるか。**

```cpp
// 100 万個と予想したが、実際は 200 万個必要だった

std::vector<Particle> v;
v.reserve(1'000'000);
for (int i = 0; i < 2'000'000; ++i) { v.push_back(...); }

ga::GrowingArray<Particle> arr(10'000'000);   // 余裕を持って予約
for (int i = 0; i < 2'000'000; ++i) { arr.EmplaceBack(); }
```

```
                        総時間     push 最大値     ポインタの有効性
std::vector + reserve    9.8 ms   4,280,000 ns   ← 途中で全部無効になる
GrowingArray            12.4 ms      41,000 ns   ← 最後まで有効
```

**`vector` は、予想を超えた瞬間に再確保します。**

100 万を超えたところで 150 万に拡張、さらに 225 万に拡張。**2回の巨大なコピー** が発生し、最大 4.28 ms のスパイクになりました。そして、**それまでに配ったポインタは全部無効です**。

`GrowingArray` は、1000 万まで予約してあるので何も起きません。

### メモリ消費

```
                        アドレス空間   コミットチャージ   物理メモリ(200 万個時)
std::vector + reserve       —           最大 72 MB          64 MB
GrowingArray             320 MB              64 MB          64 MB
```

**320 MB のアドレス空間を予約していますが、コミットチャージは実使用分だけです。**

64 ビットプロセスのアドレス空間は 128 TB あります。320 MB は **0.0002%** です。

---

## 30.6 何に使えるか

### 1. 相互参照を持つオブジェクト群

30.4 節で見たとおりです。敵、AI ノード、シーングラフ、UI ウィジェット——**互いを直接ポインタで指すデータ構造** が、素直に書けるようになります。

### 2. 増える量が読めないもの

- デバッグログ
- 描画コマンド(画面に映るオブジェクト数はカメラ次第)
- 当たり判定の候補リスト
- 第18章のスタックトレース記録

**「最大何個か」を事前に決めなくてよい** のは、想像以上に楽です。

### 3. 侵入的なデータ構造の置き場

第11章の破棄リスト、第21章のフリーリスト、第24章の境界タグ——**ポインタで数珠つなぎになった構造** は、要素が移動すると全部壊れます。

`GrowingArray` の上でなら、安全に構築できます。

### 4. アリーナの土台

`GrowingArray<std::byte>` は、実質的に第29章の `Bump` です。**用途に応じて、型付きの配列としても、生バイト列としても使えます。**

---

## 30.7 制約と注意

正直に書いておきます。

### 予約を超えたら失敗する

```cpp
ga::GrowingArray<Particle> arr(1'000);

for (int i = 0; i < 2'000; ++i)
{
    if (arr.EmplaceBack() == nullptr)
    {
        std::println("{} 個目で予約を使い切った", i);
        break;
    }
}
```

**伸ばせません。** `std::vector` と違い、上限は絶対です。

ただし、**上限を大きく取るコストがほぼゼロ** なので、実用上の問題は小さい。「1000 万個で足りるか?」と悩むより、「1 億個予約しておこう」で構いません。

### 32 ビット環境では厳しい

32 ビットプロセスのユーザーアドレス空間は 2 GB(設定によっては 3 GB)です。**大きな予約を何本も並べる余裕はありません。**

**この章の手法は、64 ビット環境を前提としています。**

### 縮まない

`Clear()` を呼んでも、コミットは解除されません。明示的に縮めたい場合は、`VirtualMemory::DecommitFrom` を呼ぶ関数を足す必要があります(演習30-4)。

### 中間の削除は依然 O(n)

`std::vector` と同じです。要素の順序を保ったまま中間を削除するには、後ろを詰める必要があります。

順序を気にしないなら、**末尾と入れ替えて `PopBack`** という定番の手が使えます。

```cpp
    void SwapRemove(std::size_t i) noexcept
    {
        assert(i < size_);
        data_[i] = std::move(data_[size_ - 1]);
        PopBack();
    }
```

ただし、**これはポインタの安定性を壊します**。移動した要素を指していたポインタは、別の要素を指すことになります。**この型の最大の利点と引き換えの操作** なので、使いどころに注意してください。

### スレッド安全ではない

第5部で扱います。

---

## 演習

**演習30-1** `std::vector` の成長率を確認してください。`capacity()` を毎回表示すると、何倍ずつ増えていますか。

**演習30-2** `GrowingArray` のコミット粒度を変えるオプションを足してください(`VirtualMemory` の `kCommitChunk` を可変にします)。1 MB にすると、30.5 節の数字はどう変わりますか。

**演習30-3** `Prewarm` を呼ぶ場合と呼ばない場合で、`push` のヒストグラムを比べてください。スパイクは消えますか。

**演習30-4** `ShrinkToFit()` を実装してください。`Clear()` の後に呼ぶと、コミットチャージはどうなりますか。

**演習30-5** `GrowingArray<std::byte>` の上に `Bump` を構築してください。第29章の実装と何が違いますか。

**演習30-6** `SwapRemove` を使うと、どんなバグが起きうるか具体例を作ってください。第45章のハンドルは、この問題をどう解決しますか。

**演習30-7** 2次元の `GrowingArray`(行ごとに可変長)を設計してください。予約はどう配分しますか。

**演習30-8** `std::pmr::vector` に、予約済みの `monotonic_buffer_resource` を渡すと、似たことができますか。違いは何ですか。(第39章の予習です)

---

## 章末チェックリスト

- [ ] `std::vector::push_back` でポインタが無効になることを実演した
- [ ] 再確保のスパイク(最大 2.14 ms)をヒストグラムで確認した
- [ ] `reserve` では解決しない3つの理由を説明できる
- [ ] `GrowingArray<T>` を実装した 〔v0.22〕
- [ ] アラインメントの計算が不要になる理由を説明できる
- [ ] 50 万回の追加をまたいでポインタが有効であることをテストした
- [ ] 予想が外れたときの挙動の差を確認した
- [ ] `SwapRemove` がポインタの安定性を壊すことを理解した

---

## 次章の予告

第17章で作ったガードバイトには、正直に書いた限界がありました。

> **遠くへの書き込みは素通りする。検出が遅れる。読み取りは検出できない。**

第31章で、これを根本的に解決します。

```cpp
VirtualAlloc(guardPage, 4096, MEM_COMMIT, PAGE_NOACCESS);
```

このページに触れた瞬間、**CPU が例外を発生させます**。1バイトでもはみ出せば、その命令の実行時点でプログラムが止まります。

- 検出が遅れない(書いた瞬間に止まる)
- 遠くへの書き込みも捕まえられる(ページ全体が禁止)
- **読み取りも検出できる**

代償は粒度です。保護の単位は 4 KB のページなので、32 バイトの確保に 4 KB のガードページを付けることになります。**メモリを 128 倍消費します。**

だからこれは「常時有効にする機能」ではありません。**「このバグを追い詰めるときだけ使う道具」** です。Windows が標準で提供している PageHeap や Application Verifier との関係も含めて、使い方を整理します。

---

> **コラム:ポインタの安定性という、忘れられがちな価値**
>
> C++ のコンテナを選ぶとき、多くの人は「速度」と「メモリ効率」を見ます。**しかし、もう1つ重要な軸があります。**
>
> > **どの操作をしたら、既存のポインタ・参照・イテレータが無効になるか。**
>
> ---
>
> **標準コンテナの安定性**
>
> | コンテナ | 追加したとき |
> |---|---|
> | `std::vector` | **すべて無効になりうる** |
> | `std::deque` | イテレータは無効、**参照とポインタは有効** |
> | `std::list` | すべて有効 |
> | `std::map` / `std::set` | すべて有効 |
> | `std::unordered_map` | イテレータは無効、参照とポインタは有効 |
>
> **`std::vector` だけが、突出して不安定です。**
>
> それでも `std::vector` が第一選択なのは、連続配置による走査の速さ(第21章、第32章)が圧倒的だからです。**安定性を犠牲にして、局所性を取った** 設計です。
>
> ---
>
> **両立させようとした試み**
>
> **`std::deque`** は、固定サイズのブロックを並べることで、追加時のポインタ安定性を確保しています。ただし要素が完全に連続しないため、走査は `std::vector` より遅くなります。
>
> **`boost::stable_vector`** は、要素を個別に確保し、ポインタの配列で管理します。安定性は完璧ですが、間接参照が1段増えます。
>
> **`plf::colony`** は、ブロック方式で安定性を保ちつつ、削除された要素をスキップする仕組みを持ちます。**標準化も進められており、`std::hive` という名前で議論されています。** ゲーム開発の文脈から生まれたコンテナで、「大量のオブジェクトを高速に走査しつつ、途中で削除も挿入もしたい」という要求に応えるものです。
>
> ---
>
> **私たちの `GrowingArray` の立ち位置**
>
> ```
>              連続配置    追加時の安定性   上限
> vector          ○            ×          なし
> deque           △            ○          なし
> colony/hive     △            ○          なし
> GrowingArray    ○            ○         あり
> ```
>
> **「上限がある」という制約を受け入れることで、他の3つを全部満たしています。**
>
> これは第20章から繰り返している構図です。**制約を1つ受け入れると、他の性質が手に入る。** バンプアロケーターが「解放しない」ことで速さを得たのと、まったく同じ形です。
>
> ---
>
> **ゲームエンジンでの実例**
>
> ECS(Entity Component System)アーキテクチャでは、コンポーネントを型ごとに連続した配列で持ちます。**走査の速さが最優先** されるためです。
>
> その代わり、エンティティ間の参照には **ポインタではなく ID** を使います。配列が再編成されても、ID から現在位置を引き直せば済むからです。
>
> **これは「安定性を諦めて、間接参照で解決する」方針** です。第45章で扱うハンドルが、まさにこの考え方です。
>
> `GrowingArray` は逆の方針——**安定性を保証して、ポインタをそのまま使う** ——を取ります。
>
> どちらが良いかは、状況によります。ただ、**両方の選択肢を持っていること** が重要です。「`std::vector` を使うから ID にするしかない」ではなく、「安定性が欲しいから `GrowingArray` を使う」という判断ができるようになりました。
