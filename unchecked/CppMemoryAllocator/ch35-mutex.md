# 第35章 とりあえずロックする 〔v0.24〕

---

## この章のゴール

第34章で、8スレッドから叩いてアロケーターを壊しました。**いちばん素直な直し方を試します。**

```cpp
std::lock_guard lock(mutex_);
```

**5行の変更で、壊れなくなります。** 問題は、その代償です。

- `std::mutex` で守り、壊れなくなることを確認する 〔**v0.24**〕
- **単一スレッドでいくら払ったのか** を測る(答え:`new` より遅くなります)
- 8スレッドで叩き、**スレッドを増やすほど遅くなる** ことを確認する
- スピンロックを書き、どこで有利でどこで悲惨かを測る
- **偽共有**(false sharing)を実演し、20 倍の差を見る
- なぜアロケーターは細粒度ロックが難しいのかを理解する

そして、**ロックの限界** が第36章への動機になります。

---

## 35.1 5行で直す

### ロック方針を型で差し替えられるようにする

第34章の 34.7 節で述べたとおり、**共有しない使い方が第一選択** です。ロックを常に払うのは損です。

そこで、**ロックの種類をテンプレートパラメータにします。**

```cpp
// ga/Lock.h
#pragma once

#include <atomic>
#include <mutex>

namespace ga
{
    // 何もしないロック(単一スレッド用)
    struct NullLock
    {
        void lock()   noexcept {}
        void unlock() noexcept {}
    };
}
```

```cpp
// ga/Bump.h
template <class Lock = NullLock>
class BasicBump
{
public:
    [[nodiscard]]
    AllocResult Allocate(std::size_t size,
                         std::size_t alignment = kDefaultAlignment,
                         const std::source_location& loc = std::source_location::current()) noexcept
    {
        std::lock_guard guard(lock_);      // ← 追加

        // ... 従来の処理 ...
    }

    void Reset() noexcept
    {
        std::lock_guard guard(lock_);      // ← 追加
        // ...
    }

    void Rewind(const Marker& m) noexcept
    {
        std::lock_guard guard(lock_);      // ← 追加
        // ...
    }

private:
    mutable Lock lock_;
    // ... 従来のメンバ ...
};

using Bump           = BasicBump<NullLock>;
using ConcurrentBump = BasicBump<std::mutex>;
```

**`NullLock` を使えば、`lock()` / `unlock()` は空の関数なのでインライン展開されて消えます。コストはゼロです。**

第14章で「機能を消すならテンプレートパラメータにするのが正解」と書きました。**その方式です。**

### ⚠ `Allocate` だけでは足りない

第34章 34.5 節の棚卸しを思い出してください。**競合する状態は 12 か所ありました。**

したがって、**状態を変えるすべてのメンバ関数** にロックが必要です。

```
Allocate / Reset / Rewind / Mark / New / NewArray
PushTag / PopTag / SetLogCallback / SetTraceConfig
```

**読むだけの関数も安全ではありません。**

```cpp
std::size_t Used() const noexcept { return offset_; }   // ← 競合する
```

他のスレッドが `offset_` を書いている最中に読めば、データ競合です。第34章で見たとおり、**未定義動作** です。

```cpp
    std::size_t Used() const noexcept
    {
        std::lock_guard guard(lock_);
        return offset_;
    }
```

`lock_` を `mutable` にしているのは、このためです。

### 壊れなくなったことを確認する

第34章 34.1 節の実験を、`ConcurrentBump` で実行します。

```
確保成功 : 800000
重複     : 0 (0.00%)
Used()   : 51200000 (期待値 51200000)
他人に書き換えられた領域: 0
```

**完璧に直りました。**

---

## 35.2 いくら払ったのか

**単一スレッドで測ります。**

```
Bump(NullLock)              median=      2.1 ns
Bump(std::mutex)            median=     23.4 ns
new                         median=     17.6 ns
```

**11 倍遅くなり、`new` にも負けました。**

第5章から30章かけて積み上げた優位が、**ロック1つで消えました。**

### なぜ非競合でもこれほどかかるのか

「誰とも競合していないのだから、ロックは一瞬では?」と思うかもしれません。

**ロックの取得は、アトミックな読み書き(RMW)を伴います。**

```
lock()   : アトミックに「空いているか確認して、確保する」
unlock() : アトミックに「解放する」
```

アトミック操作は、CPU にとって特別な命令です。

- **キャッシュラインの排他所有権** を取得する必要がある
- **メモリバリア** が伴い、それ以前の書き込みを完了させる
- 命令の並べ替えが制限され、パイプラインの効率が落ちる

**1回のアトミック RMW は、通常のメモリ書き込みの 10〜20 倍かかります。** それが `lock` と `unlock` で2回。

さらに `std::mutex` には、待機・起床の仕組みや、規格が要求する各種の保証が含まれています。

### 軽いロックを試す

Windows には、より軽量な排他制御があります。

```cpp
// ga/Lock.h に追加(実装は .cpp)
class SrwLock
{
public:
    SrwLock() noexcept;
    void lock()   noexcept;
    void unlock() noexcept;
private:
    void* handle_ = nullptr;   // SRWLOCK を隠す
};
```

```
Bump(std::mutex)            median=     23.4 ns
Bump(SRWLOCK)               median=     12.8 ns
```

**ほぼ半分になりました。**

`SRWLOCK` は、Windows Vista で導入された軽量な同期オブジェクトです。ポインタ1個分のサイズで、非競合時は数命令で済みます。

> **MSVC の `std::mutex` が重いのは、歴史的な事情によるところが大きい** と言われています。`std::mutex` には ABI 互換性の制約があり、内部構造を自由に変えられません。**同じ「排他制御」でも、実装によって倍の差があります。**

---

## 35.3 スケールしない

単一スレッドで遅いのは分かりました。**では、複数スレッドではどうか。**

```cpp
template <class Alloc>
double MeasureThroughput(Alloc& alloc, int threadCount, int opsPerThread)
{
    std::atomic<bool> start{ false };
    std::vector<std::thread> threads;

    for (int t = 0; t < threadCount; ++t)
    {
        threads.emplace_back([&] {
            while (!start.load(std::memory_order_acquire)) { }

            for (int i = 0; i < opsPerThread; ++i)
            {
                bench::Escape(alloc.Allocate(64, 16).value_or(nullptr));
            }
        });
    }

    const auto t0 = std::chrono::steady_clock::now();
    start.store(true, std::memory_order_release);

    for (auto& th : threads) { th.join(); }
    const auto t1 = std::chrono::steady_clock::now();

    const double seconds = std::chrono::duration<double>(t1 - t0).count();
    return (threadCount * static_cast<double>(opsPerThread)) / seconds;
}
```

### 結果

```
スレッド数   共有 + mutex     スレッドごとに別インスタンス
    1         42.7 M ops/s          476 M ops/s
    2         28.4 M ops/s          951 M ops/s
    4         21.2 M ops/s        1,890 M ops/s
    8         17.9 M ops/s        3,760 M ops/s
```

**共有版は、スレッドを増やすほど遅くなります。**

### なぜ増やすほど遅くなるのか

**理由1:直列化。**

クリティカルセクションの中は、**1度に1スレッドしか実行できません**。`Allocate` の処理はほぼ全部がクリティカルセクションなので、**並列化率はゼロ** です。

アムダールの法則によれば、並列化できない部分が 100% なら、スレッドを増やしても速度は変わりません。**「変わらない」で済めばまだしも、実際には遅くなります。**

**理由2:ロックの奪い合いそのものがコストになる。**

複数のスレッドが同じキャッシュライン(ロック変数)を書き換えようとします。キャッシュラインは、**書き込むコアが排他所有権を持たなければなりません**。

```
コア0がロックを取る → キャッシュラインをコア0へ
コア1がロックを試す → キャッシュラインをコア1へ
コア2がロックを試す → キャッシュラインをコア2へ
...
```

**キャッシュラインが、コア間を行ったり来たりします。** これを **キャッシュラインのピンポン** と呼びます。1回の移動に数十〜百ナノ秒かかります。

**理由3:待機と起床。**

ロックが取れなかったスレッドは、OS に「待つ」と伝えて眠ります。起こされるときにはコンテキストスイッチが発生し、**マイクロ秒単位のコスト** がかかります。

### 対照:共有しない場合

**右の列は、8スレッドで 3,760 M ops/s。ほぼ完璧に線形です。**

第34章 34.7 節で述べた「共有しない」という方針が、いかに強力かが分かります。

> **ロックを付けるかどうかを議論する前に、共有しない設計にできないかを検討してください。**
> 8スレッドで **210 倍** の差です。どんな最適化も、この差には勝てません。

---

## 35.4 スピンロックを書く

ロックが取れなかったとき、眠るのではなく **その場で待ち続ける** 方式があります。

**クリティカルセクションが非常に短い場合**、眠って起きるコストより、待ったほうが安く済みます。

```cpp
// ga/Lock.h
#include <atomic>

#if defined(_MSC_VER)
#  include <immintrin.h>
#  define GA_CPU_PAUSE() _mm_pause()
#else
#  define GA_CPU_PAUSE() ((void)0)
#endif

namespace ga
{
    class SpinLock
    {
    public:
        void lock() noexcept
        {
            for (;;)
            {
                // ① まず取りにいく
                if (!flag_.test_and_set(std::memory_order_acquire)) { return; }

                // ② 取れなかったら、空くまで「読むだけ」で待つ
                while (flag_.test(std::memory_order_relaxed))
                {
                    GA_CPU_PAUSE();
                }
            }
        }

        void unlock() noexcept
        {
            flag_.clear(std::memory_order_release);
        }

    private:
        std::atomic_flag flag_;
    };
}
```

### 2段構えにする理由

素朴に書くと、こうなります。

```cpp
    while (flag_.test_and_set(std::memory_order_acquire)) { }   // ← 悪い
```

`test_and_set` は **書き込み** です。待っている全スレッドが書き込み続けると、**キャッシュラインを奪い合い続けます**。35.3 節のピンポンが、最悪の形で起きます。

**②のループでは `test()`(読むだけ)を使っています。** 読むだけなら、複数のコアが同時にキャッシュを共有できます。ラインの奪い合いが起きません。

**これは「test-and-test-and-set」と呼ばれる定石です。**

### `_mm_pause()`

CPU に「いまスピン待ちしている」と伝える命令です。

- パイプラインへの投機的な命令投入を抑え、**消費電力を下げる**
- ハイパースレッディングで、**同じ物理コアの別スレッドに実行資源を譲る**
- ロックが解放されたときの、メモリ順序違反によるペナルティを減らす

**1命令ですが、入れると入れないとで大きく違います。**

### `memory_order` の指定

```cpp
flag_.test_and_set(std::memory_order_acquire);   // ロック取得
flag_.clear(std::memory_order_release);          // ロック解放
```

**acquire / release は、この用途のためにあります。**

- **acquire**:これ以降の読み書きが、この操作より前に移動しない
- **release**:これ以前の読み書きが、この操作より後に移動しない

つまり、**クリティカルセクションの中身が、外に漏れ出さない** ことを保証します。

`memory_order_seq_cst`(既定)でも正しく動きますが、より強い保証のぶん、わずかに遅くなります。第37章で詳しく扱います。

### 測る

```
                      1スレッド    2       4       8
Bump(NullLock)         476 M      —       —       —
Bump(SpinLock)         161 M     52 M    31 M    19 M
Bump(SRWLOCK)           78 M     41 M    26 M    21 M
Bump(std::mutex)        42.7 M   28 M    21 M    17.9 M
```

### 読み取れること

**低競合(1〜2スレッド)では、スピンロックが圧倒的です。** `std::mutex` の 3.8 倍。

**高競合(8スレッド)では、差がなくなります。** むしろ `SRWLOCK` に負けています。

**スピンロックの弱点:**

- 待っている間、**CPU を消費し続けます**(他のスレッドの邪魔をする)
- **スレッド数がコア数を超えると崩壊します**。ロックを持ったスレッドが OS に止められると、待っているスレッドは無駄に回り続けます
- 優先度の低いスレッドがロックを持ったまま止まると、高優先度のスレッドが永久に待つ(**優先度逆転**)

> **「クリティカルセクションが数十ナノ秒で、スレッド数がコア数以下」** という条件が揃ったときだけ、スピンロックは有効です。
>
> 私たちの `Allocate` は数ナノ秒なので条件を満たしますが、**8スレッドで叩けば競合が激しくなり、利点が消えます。**

---

## 35.5 偽共有

ロックとは別に、**もっと見えにくい形の競合** があります。

### 実演

8つのスレッドが、**それぞれ自分専用のカウンタ** を増やします。**共有していません。** 競合するはずがありません。

```cpp
struct Counter
{
    std::atomic<std::uint64_t> value{ 0 };
};

struct alignas(std::hardware_destructive_interference_size) PaddedCounter
{
    std::atomic<std::uint64_t> value{ 0 };
};

template <class C>
double MeasureCounters(int threadCount, int opsPerThread)
{
    std::vector<C> counters(threadCount);
    std::vector<std::thread> threads;

    const auto t0 = std::chrono::steady_clock::now();

    for (int t = 0; t < threadCount; ++t)
    {
        threads.emplace_back([&, t] {
            for (int i = 0; i < opsPerThread; ++i)
            {
                counters[t].value.fetch_add(1, std::memory_order_relaxed);
            }
        });
    }

    for (auto& th : threads) { th.join(); }

    const auto t1 = std::chrono::steady_clock::now();

    return std::chrono::duration<double, std::nano>(t1 - t0).count()
         / (threadCount * static_cast<double>(opsPerThread));
}

int main()
{
    std::println("hardware_destructive_interference_size = {}",
                 std::hardware_destructive_interference_size);

    std::println("詰めて配置   : {:.2f} ns/op", MeasureCounters<Counter>(8, 5'000'000));
    std::println("64B ずつ分離 : {:.2f} ns/op", MeasureCounters<PaddedCounter>(8, 5'000'000));
}
```

```
hardware_destructive_interference_size = 64
詰めて配置   : 24.80 ns/op
64B ずつ分離 :  1.15 ns/op
```

**21 倍の差。**

### 何が起きているのか

`Counter` は 8 バイトです。`std::vector<Counter>` に 8 個並べると、**64 バイト——ちょうど1本のキャッシュライン** に全部収まります。

```
┌────────────────── キャッシュライン(64 バイト)──────────────────┐
│ c[0] │ c[1] │ c[2] │ c[3] │ c[4] │ c[5] │ c[6] │ c[7] │
└────────────────────────────────────────────────────────────┘
   ↑      ↑      ↑      ↑      ↑      ↑      ↑      ↑
 スレッド0 1     2      3      4      5      6      7
```

**キャッシュの一貫性は、キャッシュライン単位で管理されます。**

スレッド 0 が `c[0]` に書き込むと、**そのキャッシュラインの排他所有権** が必要です。他のコアが持っている同じラインは無効化されます。

**変数は別々でも、ラインが同じなら、コア間でラインが奪い合われます。**

**論理的には競合していないのに、ハードウェアのレベルで競合する。** これを **偽共有**(false sharing)と呼びます。

### `std::hardware_destructive_interference_size`

C++17 で追加された定数です(`<new>` にあります)。

```cpp
std::hardware_destructive_interference_size   // 64:これ以上離せば偽共有が起きない
std::hardware_constructive_interference_size  // 64:これ以内なら同じラインに乗る
```

**名前が長いのは、意味を正確に表そうとした結果です。**

- **destructive**(破壊的干渉):**離すべき** 距離。偽共有を避けたいとき
- **constructive**(建設的干渉):**まとめるべき** 距離。一緒に使うデータを1ラインに収めたいとき

MSVC ではどちらも 64 を返します。

> **移植性の注意:** この定数の値は実装依存で、ABI に影響します。GCC ではこの定数を使うと警告が出ることがあります(将来値が変わると ABI が壊れるため)。
>
> **迷ったら 64 をハードコードするか、自分で定数を定義しても構いません。** 大事なのは「離す」ことであって、正確な値ではありません。

### アロケーターへの適用

**どこに効くのか。**

**1. ロック変数と、それが守るデータ。**

これは **まとめるべき** です(constructive)。ロックを取ったスレッドは、すぐにデータを触ります。同じラインにあれば、1回のライン転送で済みます。

**2. スレッドごとの統計。**

第15章で作った統計を、スレッドごとに持つとします。

```cpp
struct ThreadStats { std::size_t count, bytes; };
std::vector<ThreadStats> perThread;    // ← 偽共有が起きる
```

**必ず分離してください。**

```cpp
struct alignas(64) ThreadStats { std::size_t count, bytes; };
```

**3. 第36章のスレッドローカルなフリーリスト。**

スレッドごとのキャッシュを配列で持つなら、**各要素を 64 バイト境界に置く必要があります。**

**次章の設計に、直接効いてきます。**

---

## 35.6 なぜ細粒度ロックが難しいのか

「1本の大きなロックが問題なら、細かく分ければいい」——**汎用アロケーターでは、これが難しい。**

### 第25章のビンごとにロックする?

```cpp
std::mutex binLocks_[128];    // ビンごとにロック
```

確保だけなら、うまくいきそうです。ビン4を触るときはビン4のロックだけ取ればいい。

**しかし、合体で破綻します。**

```cpp
void Free(void* p)
{
    Header* h = HeaderOf(p);

    // 後ろのブロックが空きなら、そのビンからも外す
    if (IsFree(next)) { RemoveBlock(next); }   // ← next がどのビンかは実行時に決まる

    // 前のブロックが空きなら、そのビンからも外す
    if (PrevIsFree(h)) { RemoveBlock(prev); }  // ← これも

    // 合体後のサイズで、また別のビンに入れる
    InsertBlock(h);                            // ← さらに別のビン
}
```

**1回の `Free` で、最大4つのビンを触ります。** しかも、どのビンかは実行時まで分かりません。

複数のロックを取る場合、**取得順序を一定にしないとデッドロックします**。

```
スレッド A: ビン3 を取得 → ビン7 を待つ
スレッド B: ビン7 を取得 → ビン3 を待つ
                → 永久に待つ
```

**「番号の小さいビンから取る」という規則にすればデッドロックは防げます** が、実装は複雑になり、ロックの取得回数も増えます。

### ブロックのヘッダ自体が共有データ

もっと根本的な問題があります。

第24章の合体では、**隣接するブロックのヘッダを読み書きします**。隣のブロックが、他のスレッドが使用中のブロックかもしれません。

**「どのロックがこのヘッダを守っているか」が定義できません。** ブロックの隣接関係は、確保と解放のたびに変わるからです。

### だから、方向を変えた

**現代の高性能アロケーターは、「ロックを細かくする」のではなく「共有しない」方向に進みました。**

```
tcmalloc  : スレッドごとのキャッシュ
jemalloc  : スレッドをアリーナに割り当てる
mimalloc  : スレッドごとのヒープ + リモート解放のキュー
```

**第36章で、同じ道を進みます。**

---

## 35.7 それでもロックが正解な場面

否定的なことばかり書きましたが、**ロックが適切な場面もあります。**

### 確保の頻度が低い

シーンのロード時に数百回確保するだけなら、1回 23 ns が 100 回でも 2.3 マイクロ秒です。**まったく問題になりません。**

**最適化すべきは、ホットパスだけです。**

### 単純さが重要

ロックは、**正しく書くのが簡単** です。`std::lock_guard` を置くだけで、例外が飛んでも確実に解放されます。

第37章で扱うロックフリーの実装は、**桁違いに難しい** です。

### 測ってから決める

第4章から一貫している方針です。

```
1. まず NullLock で書く(共有しない設計)
2. 共有が必要になったら、mutex を付ける
3. プロファイルを取る
4. ロックが本当にボトルネックなら、次の手を打つ
```

**3を飛ばして4に進むのが、最もよくある失敗です。**

---

## 35.8 この章の完成コード

```cpp
// ga/Lock.h
#pragma once

#include <atomic>
#include <mutex>

#if defined(_MSC_VER)
#  include <immintrin.h>
#  define GA_CPU_PAUSE() _mm_pause()
#else
#  define GA_CPU_PAUSE() ((void)0)
#endif

namespace ga
{
    struct NullLock
    {
        void lock()   noexcept {}
        void unlock() noexcept {}
    };

    class SpinLock
    {
    public:
        SpinLock() noexcept = default;

        SpinLock(const SpinLock&)            = delete;
        SpinLock& operator=(const SpinLock&) = delete;

        void lock() noexcept
        {
            for (;;)
            {
                if (!flag_.test_and_set(std::memory_order_acquire)) { return; }

                while (flag_.test(std::memory_order_relaxed))
                {
                    GA_CPU_PAUSE();
                }
            }
        }

        void unlock() noexcept
        {
            flag_.clear(std::memory_order_release);
        }

    private:
        std::atomic_flag flag_;
    };
}
```

```cpp
// ga/Bump.h
template <class Lock = NullLock>
class BasicBump { /* すべての可変メソッドで std::lock_guard */ };

using Bump           = BasicBump<NullLock>;
using ConcurrentBump = BasicBump<SpinLock>;
```

**`Pool`、`FreeList`、`Tlsf` にも同じテンプレートパラメータを足します。** 変更は機械的です。

---

## 演習

**演習35-1** `Pool` と `Tlsf` にロック方針のテンプレートパラメータを足してください。単一スレッドでの速度は変わりませんか。

**演習35-2** `Used()` のような読み取り専用のメソッドで、ロックを外してみてください。第34章の実験で問題は起きますか。

**演習35-3** `SpinLock` の②のループを `test_and_set` に戻してください。8スレッドでの性能はどう変わりますか。

**演習35-4** `GA_CPU_PAUSE()` を空にして測ってください。差はありますか。

**演習35-5** スレッド数を、論理コア数の 2 倍・4 倍にしてください。`SpinLock` と `std::mutex` の差はどうなりますか。

**演習35-6** 35.5 節の偽共有の実験で、`alignas` の値を 8 / 16 / 32 / 64 / 128 と変えてください。どこで改善しますか。

**演習35-7** 第15章の統計を、スレッドごとの配列に分離してください。`alignas` を付けた場合と付けない場合で、8スレッドの性能を比べてください。

**演習35-8** 読み書きロック(`std::shared_mutex`)を使うと、`Used()` のような読み取りが多い場合に有利ですか。実測してください。

---

## 章末チェックリスト

- [ ] ロック方針をテンプレートパラメータにし、`NullLock` のコストがゼロであることを確認した
- [ ] **状態を変えるすべてのメソッド** にロックが必要なことを理解した
- [ ] 読み取り専用のメソッドも競合することを理解した
- [ ] 単一スレッドで `new` より遅くなることを測った
- [ ] **スレッドを増やすほど遅くなる** ことを測った
- [ ] 「共有しない」方式との差(8スレッドで 210 倍)を確認した
- [ ] test-and-test-and-set を実装し、なぜ2段構えなのかを説明できる
- [ ] **偽共有** を実演し、21 倍の差を確認した
- [ ] `hardware_destructive_interference_size` の意味を説明できる
- [ ] アロケーターで細粒度ロックが難しい理由を説明できる

---

## 次章の予告

35.6 節で述べたとおり、**現代の高性能アロケーターは「共有しない」方向に進みました。**

第36章で、その設計を実装します。

```
各スレッド: 自分専用の小さなアリーナ(ロック不要)
中央     : 大きなメモリの供給元(たまにしか触らない)
```

**確保のほとんどは、スレッドローカルなキャッシュで完結します。** ロックが要りません。足りなくなったときだけ、中央から **まとめて** もらってきます。

`thread_local` キーワードで、驚くほど簡単に書けます。**しかし、大きな問題が1つあります。**

```cpp
// スレッド A で確保
void* p = Allocate(64);

// スレッド B で解放
Free(p);        // ← どのスレッドのキャッシュに返すのか?
```

**クロススレッド解放** と呼ばれる問題です。生産者・消費者パターンでは日常的に起きます。

tcmalloc、jemalloc、mimalloc が、それぞれ違う答えを出しています。私たちも1つ選んで実装します。

---

> **コラム:ミューテックスの中身**
>
> 35.2 節で「非競合でも 23 ns かかる」と書きました。**何にそれだけかかるのか。**
>
> ---
>
> **最も単純な実装**
>
> ```cpp
> void lock()
> {
>     while (flag.exchange(true)) { }   // 空くまで回る
> }
> ```
>
> スピンロックです。非競合なら **アトミック交換1回** で済みます。10 ナノ秒程度。
>
> **問題は、待つときです。** ロックが1ミリ秒保持されるなら、その間ずっと CPU を焼き続けます。
>
> ---
>
> **眠る仕組み**
>
> そこで、取れなかったら OS に「起こしてくれ」と頼んで眠ります。
>
> Linux では **futex**(fast userspace mutex)、Windows では **`WaitOnAddress` / `WakeByAddressSingle`** という仕組みがあります。
>
> **重要な工夫は、「競合しないときはカーネルに入らない」** ことです。
>
> ```
> 非競合  : ユーザーモードのアトミック操作だけ。カーネルに入らない
> 競合時  : システムコールで眠る
> ```
>
> かつての Windows の `CRITICAL_SECTION` も、同じ思想で設計されていました。「クリティカルセクションはミューテックスより速い」と言われたのは、**カーネルオブジェクトを使わずに済むから** です。
>
> ---
>
> **適応的スピン**
>
> さらに、多くの実装は「少しだけスピンしてから眠る」という戦略を取ります。
>
> ```
> 数十〜数百回スピンする
>   → 取れたら勝ち(眠るコストを回避)
>   → 取れなければ眠る(CPU を無駄にしない)
> ```
>
> **クリティカルセクションが短ければスピンで済み、長ければ眠る。** 両方の利点を取ろうとしています。
>
> `CRITICAL_SECTION` には、スピン回数を設定する API(`InitializeCriticalSectionAndSpinCount`)まで用意されていました。
>
> ---
>
> **では、なぜ 23 ns なのか**
>
> 非競合の `std::mutex` でも、次のことが起きます。
>
> - **アトミック RMW が2回**(lock と unlock)。それぞれ 5〜15 ns
> - **メモリバリア**。これ以前の書き込みを完了させる
> - 所有スレッドの記録(再帰的ロックの検出や、規格が要求する検査のため)
> - 関数呼び出し(インライン展開されない)
>
> **これらが積み重なった結果です。**
>
> ---
>
> **アロケーターにとっての意味**
>
> `Bump::Allocate` は **2.1 ナノ秒** です。**ロックのほうが 10 倍重い。**
>
> 「守るべき処理」より「守る仕組み」のほうが高価なとき、**根本的に設計を見直すべきです。**
>
> それが第36章と第37章のテーマです。
>
> - **第36章**:そもそもロックを取らずに済む構造にする
> - **第37章**:ロックを1回のアトミック操作に置き換える
>
> どちらも「ロックを速くする」のではなく、**「ロックを減らす」** というアプローチです。
