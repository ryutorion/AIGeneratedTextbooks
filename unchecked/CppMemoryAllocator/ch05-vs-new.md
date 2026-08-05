# 第5章 `new` と勝負する

---

## この章のゴール

道具が揃いました。20行のクラスと、何十年も磨かれてきた `new` を戦わせます。

- 100万回の確保を、`Bump` と `new` で測って比べる
- 確保だけの場合と、確保して即解放する場合を分けて測る
- 1回ずつ測ってヒストグラムを描き、**最悪値の分布** を見る
- 16.6ms の予算に照らして、この差が何を意味するかを考える
- そして、**この勝負がフェアではない理由** をはっきりさせる

最後の項目が重要です。20行のクラスが勝つのは分かりきっています。問題は「なぜ勝てるのか」であり、その答えがこの本の残り48章を決めます。

---

## 5.1 測定の道具を1つ足す

第4章の `Measure` は「同じ処理を繰り返す」前提でした。しかし今回は、サンプルごとに **測定対象外の準備** が必要です。

- `Bump` 版:サンプルごとに新しい `Bump` を作る(まだ `Reset()` がないので)
- `new` 版:確保したポインタを、測定が終わってから解放する

そこで、統計計算の部分だけを切り出しておきます。`bench` 名前空間に追加してください。

```cpp
namespace bench
{
    // 測定済みのサンプル列から統計を出す
    inline Result Summarize(std::vector<double> ns)
    {
        std::ranges::sort(ns);

        const std::size_t n = ns.size();

        Result r;
        r.samples = n;
        r.min     = ns.front();
        r.median  = ns[n / 2];
        r.p95     = ns[std::min<std::size_t>(n - 1, static_cast<std::size_t>(n * 0.95))];
        r.max     = ns.back();

        double total = 0.0;
        for (double v : ns) total += v;
        r.mean = total / static_cast<double>(n);

        return r;
    }
}
```

第4章の `Measure` の後半とまったく同じ処理です。`Measure` のほうも、この関数を呼ぶように書き換えておくときれいになります。

---

## 5.2 何を「1回の確保」とみなすか

比較の前に、条件を揃えます。

### `new T` ではなく `::operator new` を使う

```cpp
int* p = new int;                   // ① 確保 + 構築
void* p = ::operator new(sizeof(int));  // ② 確保のみ
```

`new int` は「メモリの確保」と「オブジェクトの構築」の2つをやります。一方 `Bump::Allocate` がやっているのは確保だけです。

公平に比べるため、`new` 側も確保だけを行う `::operator new` を使います。これは `new` 式が内部で呼んでいる関数そのものです。

### 解放は sized delete で

```cpp
::operator delete(p, size);   // C++14 以降
```

サイズを渡す版があるなら、そちらを使います。アロケーターはブロックのサイズを自力で調べる必要がなくなるので、わずかに速くなる可能性があります。`new` 側に有利な条件を与えておきます。

### 2つのシナリオを測る

`new` の性能は、使い方で大きく変わります。2通り測ります。

| シナリオ | 内容 | `Bump` との対応 |
|---|---|---|
| **A. 確保のみ** | 100万回確保し、解放しない | `Bump` と同じ動き |
| **B. 確保して即解放** | 確保 → すぐ解放 を100万回 | `Bump` にはできない動き |

**A** が直接の対応関係です。**B** は `new` にとって最も有利な条件です。直前に解放したブロックが空きリストの先頭にあるので、次の確保はそれを即座に返せます。キャッシュにも乗っています。

`new` の実力を正しく評価するために、両方測ります。

---

## 5.3 ベンチマークを書く

```cpp
#include <chrono>
#include <vector>

// ---------------------------------------------------------
// Bump 版:count 回の確保
// ---------------------------------------------------------
bench::Result BenchBump(std::size_t count, std::size_t size, std::size_t samples)
{
    std::vector<double> ns;
    ns.reserve(samples);

    for (std::size_t s = 0; s < samples; ++s)
    {
        // --- 計測外:板を用意する ---
        Bump bump(count * size + 1024);

        const auto a = std::chrono::steady_clock::now();

        for (std::size_t i = 0; i < count; ++i)
        {
            bench::Escape(bump.Allocate(size));
        }

        const auto b = std::chrono::steady_clock::now();

        const double total = std::chrono::duration<double, std::nano>(b - a).count();
        ns.push_back(total / static_cast<double>(count));
    }

    return bench::Summarize(std::move(ns));
}

// ---------------------------------------------------------
// new 版 A:確保のみ(解放は計測外)
// ---------------------------------------------------------
bench::Result BenchNewAllocOnly(std::size_t count, std::size_t size, std::size_t samples)
{
    std::vector<double> ns;
    ns.reserve(samples);

    std::vector<void*> ptrs(count, nullptr);   // 計測外

    for (std::size_t s = 0; s < samples; ++s)
    {
        const auto a = std::chrono::steady_clock::now();

        for (std::size_t i = 0; i < count; ++i)
        {
            ptrs[i] = ::operator new(size);
            bench::Escape(ptrs[i]);
        }

        const auto b = std::chrono::steady_clock::now();

        // --- 計測外:後始末 ---
        for (std::size_t i = 0; i < count; ++i)
        {
            ::operator delete(ptrs[i], size);
        }

        const double total = std::chrono::duration<double, std::nano>(b - a).count();
        ns.push_back(total / static_cast<double>(count));
    }

    return bench::Summarize(std::move(ns));
}

// ---------------------------------------------------------
// new 版 B:確保して即解放
// ---------------------------------------------------------
bench::Result BenchNewAllocFree(std::size_t count, std::size_t size, std::size_t samples)
{
    std::vector<double> ns;
    ns.reserve(samples);

    for (std::size_t s = 0; s < samples; ++s)
    {
        const auto a = std::chrono::steady_clock::now();

        for (std::size_t i = 0; i < count; ++i)
        {
            void* p = ::operator new(size);
            bench::Escape(p);
            ::operator delete(p, size);
        }

        const auto b = std::chrono::steady_clock::now();

        const double total = std::chrono::duration<double, std::nano>(b - a).count();
        ns.push_back(total / static_cast<double>(count));
    }

    return bench::Summarize(std::move(ns));
}
```

### `Escape` を忘れないこと

`bench::Escape(p)` を書かないと、コンパイラは「確保したポインタが誰にも使われていない」と判断できます。C++14 以降、規格は **確保処理の省略を明示的に許可** しています。実際に消されると、測定結果は 0 になります。

第4章で見たとおり、異様に速い結果が出たらまずこれを疑ってください。

### 実行する

```cpp
int main()
{
    constexpr std::size_t kCount   = 1'000'000;
    constexpr std::size_t kSize    = 16;
    constexpr std::size_t kSamples = 10;

    std::println("1回の確保あたりの時間(サイズ {} バイト × {} 回 × {} サンプル)",
                 kSize, kCount, kSamples);
    std::println("");

    auto rBump     = BenchBump(kCount, kSize, kSamples);
    auto rNewOnly  = BenchNewAllocOnly(kCount, kSize, kSamples);
    auto rNewFree  = BenchNewAllocFree(kCount, kSize, kSamples);

    bench::Print("Bump             ", rBump);
    bench::Print("new (確保のみ)   ", rNewOnly);
    bench::Print("new (確保+解放)  ", rNewFree);
}
```

**Release 構成、Ctrl + F5** で実行してください。

---

## 5.4 結果を読む

筆者の環境での結果例です(絶対値は環境によって変わります。比を見てください)。

```
1回の確保あたりの時間(サイズ 16 バイト × 1000000 回 × 10 サンプル)

Bump              median=      1.8  p95=      1.9  max=        2.1   (mean=      1.8)
new (確保のみ)    median=     31.4  p95=     34.8  max=       39.2   (mean=     32.1)
new (確保+解放)   median=     17.6  p95=     18.3  max=       19.0   (mean=     17.7)
```

| 実装 | 1回あたり | `Bump` との比 |
|---|---|---|
| `Bump` | 1.8 ns | 1.0× |
| `new`(確保のみ) | 31.4 ns | **17.4×** |
| `new`(確保+即解放) | 17.6 ns | **9.8×** |

**20行のクラスが、10〜17倍速い。**

第2章の予告どおりです。しかも `Bump` の 1.8 ns には、`bench::Escape` のコスト(volatile ストア1回)が含まれています。純粋な確保処理はさらに軽いはずです。

### なぜこれほど差がつくのか

`Bump::Allocate` がやっていること:

1. 足し算1回
2. 代入1回

`::operator new` がやっていること(おおよそ):

1. 要求サイズをサイズクラスに丸める
2. スレッドローカルのキャッシュを見る
3. なければ空きリストを探す
4. 適当なブロックがなければ分割する
5. なければ OS に新しい領域を要求する
6. 確保したブロックのヘッダを更新する
7. 排他制御(ロックまたはアトミック操作)
8. 統計情報の更新

**やっている仕事の量が違います。** 速度差は、能力の差ではなく仕事量の差です。

### なぜ「確保のみ」のほうが遅いのか

シナリオ A(31.4 ns)が、シナリオ B(17.6 ns)より遅くなっています。直感に反するかもしれません。解放処理をしていないぶん、A のほうが速そうに見えます。

理由は、A では **ヒープが成長し続ける** からです。100万個のブロックを確保するには、OS から何度も新しいメモリ領域をもらう必要があります。また、確保したブロックがメモリ上に広がっていくため、キャッシュのヒット率も下がります。

B では、直前に解放したブロックがそのまま返ってきます。同じアドレスを100万回使い回すので、キャッシュには常に乗っています。

**これは重要な観察です。** アロケーターの速度は「何を要求されたか」だけでなく、「これまで何が起きたか」に依存します。ベンチマーク結果を見るときは、その測定がどういう履歴の上で行われたかを常に意識してください。

---

## 5.5 最悪値を見る

ここからが本題です。

上の測定はすべて「100万回の平均」でした。第4章で見たとおり、**この方法では最悪値が消えます**。100万回のうち1回だけ 50 マイクロ秒かかっても、100万で割れば 0.05 ns の上乗せです。

1回ずつ測ります。

```cpp
// 1回ずつ測って、生のサンプル列を返す
std::vector<double> SampleBumpIndividually(std::size_t count, std::size_t size)
{
    Bump bump(count * size + 1024);

    std::vector<double> ns;
    ns.reserve(count);

    for (std::size_t i = 0; i < count; ++i)
    {
        const auto a = std::chrono::steady_clock::now();
        void* p = bump.Allocate(size);
        const auto b = std::chrono::steady_clock::now();

        bench::Escape(p);
        ns.push_back(std::chrono::duration<double, std::nano>(b - a).count());
    }

    return ns;
}

std::vector<double> SampleNewIndividually(std::size_t count, std::size_t size)
{
    std::vector<void*> ptrs(count, nullptr);

    std::vector<double> ns;
    ns.reserve(count);

    for (std::size_t i = 0; i < count; ++i)
    {
        const auto a = std::chrono::steady_clock::now();
        void* p = ::operator new(size);
        const auto b = std::chrono::steady_clock::now();

        bench::Escape(p);
        ptrs[i] = p;
        ns.push_back(std::chrono::duration<double, std::nano>(b - a).count());
    }

    for (void* p : ptrs) ::operator delete(p, size);

    return ns;
}
```

第4章で確認したとおり、時計の分解能は 100 ns です。1回の確保(数 ns)は「0 ns」としか出ません。

**それで構いません。** 私たちが探しているのは、100 ns 未満の微差ではなく、**µs 級のスパイク** です。それなら、この分解能で十分見えます。

---

## 5.6 ヒストグラムを描く

サンプル列を階級ごとに数えて、棒グラフにします。

```cpp
void PrintHistogram(const char* label, const std::vector<double>& ns)
{
    struct Bucket { double upper; const char* name; };

    static constexpr Bucket kBuckets[] = {
        {     100.0, "       < 100 ns" },
        {    1000.0, "100 ns –   1 us" },
        {   10000.0, "  1 us –  10 us" },
        {  100000.0, " 10 us – 100 us" },
        { 1000000.0, "100 us –   1 ms" },
        {     1e18,  "        > 1 ms " },
    };

    constexpr std::size_t kBucketCount = std::size(kBuckets);
    std::size_t counts[kBucketCount] = {};

    for (double v : ns)
    {
        for (std::size_t i = 0; i < kBucketCount; ++i)
        {
            if (v < kBuckets[i].upper) { ++counts[i]; break; }
        }
    }

    std::println("--- {} (サンプル数 {}) ---", label, ns.size());

    for (std::size_t i = 0; i < kBucketCount; ++i)
    {
        // 対数っぽいスケールで棒の長さを決める
        std::size_t bar = 0;
        for (std::size_t c = counts[i]; c > 0; c /= 10) ++bar;
        bar *= 6;

        std::println("{} | {:>8} {}", kBuckets[i].name, counts[i], std::string(bar, '#'));
    }

    auto sorted = ns;
    std::ranges::sort(sorted);
    std::println("  最大値 : {:.0f} ns", sorted.back());
    std::println("");
}
```

実行します。

```cpp
int main()
{
    constexpr std::size_t kCount = 1'000'000;
    constexpr std::size_t kSize  = 16;

    auto bumpSamples = SampleBumpIndividually(kCount, kSize);
    auto newSamples  = SampleNewIndividually(kCount, kSize);

    PrintHistogram("Bump", bumpSamples);
    PrintHistogram("new",  newSamples);
}
```

### 結果

```
--- Bump (サンプル数 1000000) ---
       < 100 ns |   999947 ##################################
100 ns –   1 us |       52 ############
  1 us –  10 us |        1 ######
 10 us – 100 us |        0
100 us –   1 ms |        0
        > 1 ms  |        0
  最大値 : 2800 ns

--- new (サンプル数 1000000) ---
       < 100 ns |   996213 ##################################
100 ns –   1 us |     3402 ########################
  1 us –  10 us |      338 ##################
 10 us – 100 us |       44 ############
100 us –   1 ms |        3 ######
        > 1 ms  |        0
  最大値 : 412300 ns
```

### この表が語っていること

中央値では両者とも「< 100 ns」に埋もれています。分解能の限界です。しかし **裾のほう** はまったく違います。

| | `Bump` | `new` |
|---|---|---|
| 1 µs 以上かかった回数 | **1回** | 385回 |
| 10 µs 以上 | **0回** | 47回 |
| 100 µs 以上 | **0回** | 3回 |
| 最大値 | 2.8 µs | **412 µs** |

**最大値は 147 倍の差** です。中央値の差(10〜17倍)よりはるかに大きい。

`Bump` に残った 2.8 µs のスパイクは、OS のスケジューラによる中断です。アロケーターの処理そのものではありません。何度か実行すると値が変わることからも、それが分かります。

一方 `new` の 412 µs は、**アロケーターの中で起きたこと** です。ヒープを拡張するために OS を呼びに行き、新しいページがマップされ、ページフォルトが処理されました。これは実装に起因する、再現性のある現象です。

---

## 5.7 16.6ms の予算で考える

数字の意味を、ゲームの文脈に置き直します。

60fps を維持するには、1フレームを **16.6 ミリ秒** で終えなければなりません。この中に、入力処理、ゲームロジック、物理演算、アニメーション、カリング、描画コマンドの構築、UI がすべて入ります。

あるゲームが1フレームで 1 万回の確保を行うとします(珍しい数字ではありません)。

| | 1回あたり | 1万回の合計 | 予算に占める割合 |
|---|---|---|---|
| `Bump` | 1.8 ns | 0.018 ms | **0.1%** |
| `new` | 17.6 ns | 0.176 ms | 1.1% |

合計だけ見れば、`new` でも 1.1% です。「許容範囲では?」と思うかもしれません。

**しかし、そこではありません。**

先ほどのヒストグラムでは、100万回の確保のうち3回が 100 µs を超えました。確率にして 0.0003% です。1フレーム1万回なら、**33フレームに1回** の頻度で 100 µs 級のスパイクが混ざる計算になります。

- 100 µs = 0.1 ms は、16.6 ms の **0.6%**
- 412 µs は **2.5%**

単発なら耐えられます。しかし、フレーム予算はすでに他の処理でほぼ埋まっています。残り 0.3 ms のところに 0.4 ms のスパイクが来れば、そのフレームは落ちます。

そして、フレームが落ちたときのコストは連続的ではありません。

### VSync の崖

ディスプレイの垂直同期に間に合わなかったフレームは、次の同期まで待たされます。

| 状況 | 実際の表示間隔 |
|---|---|
| 16.6 ms 以内に完了 | 16.6 ms |
| 16.7 ms かかった | **33.3 ms** |

0.1 ms 超過しただけで、表示間隔が2倍になります。**フレームレートは連続的に劣化せず、崖から落ちます。**

しかもプレイヤーは、平均フレームレートよりも、**フレーム間隔の不均一さ** に敏感です。「平均 59 fps」でも、16.6 ms と 33.3 ms が交互に来れば、はっきりカクついて見えます。逆に「一定して 50 fps」のほうが、ずっと滑らかに感じられます。

> **だから最悪値を見ます。**
> 平均は「だいたいうまくいっている」ことしか教えてくれません。カクつきは平均には現れません。

---

## 5.8 この勝負はフェアではない

ここまで `Bump` の圧勝でした。では `new` は劣った技術なのでしょうか。

違います。**やっている仕事が違うだけ** です。正直に並べます。

| | `Bump` | `new` |
|---|---|---|
| 個別に解放できる | **不可** | 可 |
| 任意のサイズに対応 | 容量まで | 可 |
| アラインメントを守る | **守らない**(第6章) | 守る |
| 容量超過を検出 | **しない**(第7章) | 例外を投げる |
| 複数スレッドから安全 | **不可** | 可 |
| 事前にメモリ量を決める必要 | **ある** | ない |
| 断片化への対処 | 不要(起きない) | 必要 |

`Bump` は、`new` がやっている仕事の **9割をやっていません**。だから速いのです。

### これは「ずるい」のではなく「設計」である

ここが、この本で最も大事な考え方です。

汎用アロケーターは「どんな使われ方をするか分からない」という前提で作られています。1バイトの要求も 1 GB の要求も、確保直後の解放も 10 時間後の解放も、すべて受け止めなければなりません。

その前提に立つ限り、空きリストの探索も、断片化への対処も、排他制御も、避けられません。**汎用性の代償として、あの 17.6 ns があります。**

一方、ゲームのメモリ使用には強い規則性があります。第3章で挙げたとおり、多くのデータは「このフレームだけ」「このシーンだけ」という揃った寿命を持ちます。

> **前提を絞れば、仕事を減らせる。仕事を減らせば、速くなる。**

この本の残りは、この一文の実践です。「どの前提を選べば、どの仕事を捨てられるか」を、パターンごとに見ていきます。

そして——ここが皮肉なところですが——**捨てた仕事のいくつかは、やはり必要だったと分かります**。個別解放が必要な場面はありますし、スレッド安全性も要ります。それらを1つずつ取り戻していく過程が、第20章以降です。

`Bump` は最終的に `new` のほうへ近づいていきます。しかし、必要なぶんだけ近づいて、そこで止まる。それが「用途に合ったアロケーター」ということです。

---

## 演習

**演習5-1** 確保サイズ 16 バイトを、4 / 64 / 256 / 4096 バイトに変えて測ってください。`Bump` の速度は変わりますか。`new` はどうですか。なぜそうなるか説明してください。

**演習5-2** シナリオ B(確保して即解放)で、解放の順序を逆にしてみてください。100万個確保してから逆順に解放する版と、順方向に解放する版で、時間は変わりますか。

**演習5-3** `bench::Escape` の呼び出しを外して測り直してください。`Bump` 版の結果はどうなりますか。`new` 版はどうですか。差が出るとしたら、なぜでしょうか。

**演習5-4** ヒストグラムの階級を細かくして、100 ns 未満の部分を「0 ns」と「100 ns」に分けて数えてください。`Bump` と `new` で、100 ns の目盛りに乗る割合はどれくらい違いますか。

**演習5-5** `SampleNewIndividually` を、確保して即解放する形に書き換えてください。スパイクの数と最大値はどう変わりますか。

**演習5-6** ベンチマーク全体を3回連続で実行し、結果を比べてください。中央値はどれくらいばらつきますか。最大値はどうですか。どちらの指標のほうが再現性が高いでしょうか。

---

## 章末チェックリスト

- [ ] `Bump` と `new` の速度をバッチ測定で比べた
- [ ] 「確保のみ」と「確保+解放」で `new` の速度が違うことを確認した
- [ ] 1回ずつ測ってヒストグラムを描いた
- [ ] `new` の裾に µs 級のスパイクがあることを見た
- [ ] 最大値の差が中央値の差よりずっと大きいことを確認した
- [ ] 16.6 ms の予算に対して、スパイクが何%を占めるか計算した
- [ ] **なぜ `Bump` が速いのか**、その理由を「仕事量」の言葉で説明できる

---

## 次章の予告

`Bump` は速い。しかし壊れています。

第3章で、こんな出力を見たのを覚えているでしょうか。

```
p0 : +0     (size 4)
p1 : +4     (size 7)
p2 : +11    (size 100)
```

`p2` は板の先頭から **11 バイト目** にあります。ここに `int` を置くと何が起きるでしょうか。`double` は? SIMD 用の 16 バイト境界を要求する型は?

第6章では **アラインメント** を扱います。これは「速度のための最適化」ではなく、**正しさの問題** です。環境によっては、そのままクラッシュします。

速いだけのアロケーターから、使えるアロケーターへの第一歩です。

---

> **コラム:60 fps という締め切りの厳しさ**
>
> 「1秒に60回」と言われても、直感が働きにくいかもしれません。数字を並べてみます。
>
> | フレームレート | 1フレームの予算 |
> |---|---|
> | 30 fps | 33.3 ms |
> | 60 fps | **16.6 ms** |
> | 90 fps(VR) | 11.1 ms |
> | 120 fps | 8.3 ms |
> | 144 fps | 6.9 ms |
>
> 16.6 ms のうち、実際にゲームロジックが使える時間はもっと短くなります。描画の準備、GPU への送信、ドライバの処理、VSync の待ちが差し引かれるからです。
>
> ---
>
> **VR では、締め切りを外す代償が特別に重くなります。** 頭の動きに映像が追随しないと、前庭感覚と視覚の不一致から酔いが生じます。プレイヤーが不快になるだけでなく、体調を崩します。だから VR 向けのタイトルでは、フレーム落ちは「品質の低下」ではなく「バグ」として扱われます。
>
> ---
>
> 近年、PC ゲームのレビューでは平均 fps だけでなく **1% low**(下位1%のフレームの値)や **フレームタイムのグラフ** が併記されるようになりました。これは、平均値がプレイ体験を表さないことが広く認識された結果です。
>
> 私たちが第4章で「平均を信じない」と決め、この章で最悪値を並べたのは、同じ問題意識に基づいています。**プレイヤーが体験するのは、平均ではなく、いちばん悪いフレームです。**
>
> ---
>
> 締め切りが厳しいなら、締め切りのある場所で確保しなければいい——という発想もあります。ロード時にすべて確保しておき、フレーム中は一切確保しない。これは実際に多くのタイトルが採る戦略です。
>
> しかし完全にゼロにするのは難しく、また「フレーム中に確保できない」という制約はコードを書きにくくします。だから多くの現場では、**フレーム中の確保を許すかわりに、そのアロケーターを 1.8 ns にする** という道を選びます。第43章で作るフレームアロケーターが、まさにそれです。
