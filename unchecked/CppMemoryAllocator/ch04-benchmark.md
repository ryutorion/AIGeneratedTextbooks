# 第4章 速さを測る道具を先に作る

---

## この章のゴール

`Bump` は正しく動いています。次に知りたいのは「速いのか」です。

しかし、`new` と比べる前に、**測る道具**を作らなければなりません。この章ではアロケーターに一切触れず、道具作りに専念します。

- `std::chrono::steady_clock` を使った30行のベンチマークヘルパー
- 時計そのものの精度と、呼び出しコストを実測する
- 中央値・95パーセンタイル・最大値を出す(**平均値を信じない**)
- 最適化でコードが消える問題と、その回避策
- Debug と Release でどれだけ違うかを見る

遠回りに見えるかもしれません。しかし、間違った道具で測った数字は、正しい数字より **有害** です。「速くなった」と信じて進んだ先で、実は何も変わっていなかった——これが最悪の展開です。

道具を先に作りましょう。

---

## 4.1 いきなり測ってみる(そして失敗する)

まず素朴に書いてみます。配列の合計を求める処理の時間を測ります。

```cpp
#include <chrono>
#include <print>
#include <vector>

int main()
{
    std::vector<int> data(1024, 1);

    const auto begin = std::chrono::steady_clock::now();

    int sum = 0;
    for (int v : data)
    {
        sum += v;
    }

    const auto end = std::chrono::steady_clock::now();

    const auto ns = std::chrono::duration<double, std::nano>(end - begin).count();
    std::println("所要時間: {:.1f} ns", ns);
}
```

**Release 構成** で実行してください。

```
所要時間: 0.0 ns
```

0ナノ秒。1024回の足し算が、時間ゼロで終わりました。

もちろん嘘です。この結果には、これから説明する **3つの問題** が同時に現れています。

1. 時計の分解能が足りない
2. 時計を呼ぶこと自体にコストがある
3. **そもそも処理が実行されていない**(最適化で消された)

1つずつ潰していきます。

---

## 4.2 時計の分解能を実測する

`std::chrono::steady_clock` は「ナノ秒単位」の型を持っていますが、それは表現できる単位であって、実際に測れる細かさとは違います。

実際の分解能を測ってみましょう。時刻を取り続け、**値が変わった瞬間** の差を見ます。

```cpp
#include <algorithm>
#include <chrono>
#include <print>

void PrintClockResolution()
{
    using clock = std::chrono::steady_clock;

    double smallest = 1e18;

    for (int i = 0; i < 100'000; ++i)
    {
        const auto a = clock::now();
        clock::time_point b;

        do {
            b = clock::now();
        } while (b == a);

        const double d = std::chrono::duration<double, std::nano>(b - a).count();
        smallest = std::min(smallest, d);
    }

    std::println("時計の実測分解能: {:.1f} ns", smallest);
}
```

典型的な Windows 環境での結果です。

```
時計の実測分解能: 100.0 ns
```

**100ナノ秒。** これが Windows における時計の目盛りです。

MSVC の `steady_clock` は内部で `QueryPerformanceCounter`(QPC)を使っています。近年の Windows では QPC の周波数が 10 MHz に固定されていることが多く、1目盛り = 1/10,000,000 秒 = 100 ns になります。

つまり、**100ns より短い時間は原理的に測れません**。測ろうとすると、0 ns か 100 ns のどちらかになります。

これが 4.1 節で `0.0 ns` が出た理由の1つです。

---

## 4.3 時計を呼ぶコストを実測する

もう1つ。`clock::now()` を呼ぶこと自体にも時間がかかります。「測る行為が、測られる対象より重い」状況では、何を測っているのか分かりません。

空の処理を測ってみます。

```cpp
void PrintClockOverhead()
{
    using clock = std::chrono::steady_clock;

    // 何もしない区間を1万回測って、その分布を見る
    int zeroCount = 0;
    int hundredCount = 0;
    int otherCount = 0;

    for (int i = 0; i < 10'000; ++i)
    {
        const auto a = clock::now();
        const auto b = clock::now();

        const double d = std::chrono::duration<double, std::nano>(b - a).count();

        if (d == 0.0)        ++zeroCount;
        else if (d == 100.0) ++hundredCount;
        else                 ++otherCount;
    }

    std::println("now() 2回の差: 0ns={} / 100ns={} / それ以外={}",
                 zeroCount, hundredCount, otherCount);
}
```

```
now() 2回の差: 0ns=6832 / 100ns=3105 / それ以外=63
```

`now()` を2回呼ぶと、だいたい3割の確率で目盛りが1つ進みます。つまり `now()` 1回のコストは、おおむね **20〜30 ns 程度** と推定できます。

### ここから導かれる結論

| 測りたいもの | 1回ずつ測れるか |
|---|---|
| 数ナノ秒の処理(1回の確保など) | **測れない** |
| マイクロ秒級の処理 | 測れる |
| ミリ秒級の処理 | 余裕で測れる |

私たちがこれから測ろうとしている「1回のメモリ確保」は、うまくいけば数ナノ秒です。**1回ずつ測るのは無理** だと分かりました。

---

## 4.4 2つの測り方を使い分ける

解決策は「まとめて測って割る」ことです。1回の確保が測れないなら、10万回まとめて測って10万で割ります。

ただし、この方法には重大な欠点があります。**最悪値が消える** のです。

10万回のうち1回だけ 50 マイクロ秒かかったとしても、10万で割れば 0.5 ns の上乗せにしかなりません。平均に埋もれて見えなくなります。

ゲームにとって、その1回こそが問題なのに。

そこで本書では、**目的の違う2つの測定** を使い分けます。

| 測定 | 方法 | 分かること | 分からないこと |
|---|---|---|---|
| **バッチ測定** | N回まとめて測って割る | 典型的なコスト(スループット) | 個々のばらつき、最悪値 |
| **個別測定** | 1回ずつ測る | **µs級のスパイク**、最悪値 | 100ns 未満の細かい差 |

個別測定は 100ns 未満を見分けられませんが、それでいいのです。私たちが恐れているのは「たまに数十マイクロ秒かかる」ことであって、「3ns か 5ns か」ではありません。**µs級のスパイクは、100ns の目盛りでもはっきり見えます。**

---

## 4.5 ベンチマークヘルパーを書く

方針が決まったので、道具を作ります。`Playground.cpp` の上のほうに、名前空間ごと追加してください。

```cpp
#include <algorithm>
#include <chrono>
#include <cstdint>
#include <print>
#include <vector>

namespace bench
{
    // -------------------------------------------------------
    // 測定結果(単位はすべてナノ秒)
    // -------------------------------------------------------
    struct Result
    {
        std::size_t samples = 0;
        double      min     = 0.0;
        double      median  = 0.0;   // 典型値
        double      p95     = 0.0;   // 95パーセンタイル
        double      max     = 0.0;   // 最悪値
        double      mean    = 0.0;   // 平均(参考)
    };

    // -------------------------------------------------------
    // 個別測定:body() を1回ずつ測り、分布を返す
    // -------------------------------------------------------
    template <class F>
    Result Measure(std::size_t sampleCount, F&& body)
    {
        for (int i = 0; i < 3; ++i) body();          // ウォームアップ

        std::vector<double> ns;
        ns.reserve(sampleCount);

        for (std::size_t i = 0; i < sampleCount; ++i)
        {
            const auto a = std::chrono::steady_clock::now();
            body();
            const auto b = std::chrono::steady_clock::now();
            ns.push_back(std::chrono::duration<double, std::nano>(b - a).count());
        }

        std::ranges::sort(ns);

        Result r;
        r.samples = sampleCount;
        r.min     = ns.front();
        r.median  = ns[sampleCount / 2];
        r.p95     = ns[std::min<std::size_t>(sampleCount - 1,
                       static_cast<std::size_t>(sampleCount * 0.95))];
        r.max     = ns.back();

        double total = 0.0;
        for (double v : ns) total += v;
        r.mean = total / static_cast<double>(sampleCount);

        return r;
    }
}
```

`Measure` の本体は 25 行ほどです。やっていることは単純です。

1. ウォームアップに数回空回しする
2. サンプル数ぶん、時間を測って記録する
3. ソートする
4. 端と中央を取り出す

### なぜソートするのか

中央値もパーセンタイルも、並べ替えなしには求められません。ソートは測定が全部終わったあとに行うので、測定時間には影響しません。

### ウォームアップの意味

最初の数回は、次のような理由で不当に遅くなります。

- 命令キャッシュに乗っていない
- 分岐予測が学習していない
- 触っていないメモリページのページフォルト
- CPU の動作周波数がまだ上がっていない

これらは測りたい対象ではないので、捨てます。

### バッチ測定を足す

続けて、まとめて測るほうも書きます。

```cpp
namespace bench
{
    // -------------------------------------------------------
    // バッチ測定:body() を opsPerSample 回まとめて測り、
    //             1回あたりの時間に直す
    // -------------------------------------------------------
    template <class F>
    Result MeasureBatch(std::size_t sampleCount, std::size_t opsPerSample, F&& body)
    {
        const double divisor = static_cast<double>(opsPerSample);

        return Measure(sampleCount, [&] {
            for (std::size_t i = 0; i < opsPerSample; ++i)
            {
                body();
            }
        }) / divisor;   // ← これはこのままでは動かない
    }
}
```

`Result` を割り算する演算子がないので、これでは動きません。素直に書き直します。

```cpp
    template <class F>
    Result MeasureBatch(std::size_t sampleCount, std::size_t opsPerSample, F&& body)
    {
        Result r = Measure(sampleCount, [&] {
            for (std::size_t i = 0; i < opsPerSample; ++i)
            {
                body();
            }
        });

        const double d = static_cast<double>(opsPerSample);
        r.min /= d;  r.median /= d;  r.p95 /= d;  r.max /= d;  r.mean /= d;

        return r;
    }
```

### 表示関数

```cpp
namespace bench
{
    inline void Print(const char* label, const Result& r)
    {
        std::println("{:<22} median={:>9.1f}  p95={:>9.1f}  max={:>11.1f}   (mean={:>9.1f})",
                     label, r.median, r.p95, r.max, r.mean);
    }

    inline void PrintHeader()
    {
        std::println("{:<22} {:>16} {:>14} {:>16} {:>18}",
                     "測定対象", "中央値[ns]", "p95[ns]", "最大[ns]", "平均[ns]");
    }
}
```

平均値をいちばん右に、括弧付きで置いています。これは意図的です。**平均は参考値であって、判断の材料ではない** ということを、レイアウトで表現しています。

---

## 4.6 最適化で消える問題を解決する

さて、4.1 節の3つ目の問題に戻ります。**処理が実行されていなかった** 件です。

```cpp
int sum = 0;
for (int v : data) sum += v;
```

`sum` はこのあと誰にも使われません。コンパイラはこう考えます。

> この計算結果は誰も見ない。ならばループごと消してよい。

これは正当な最適化です(as-if ルール)。結果、測定対象そのものが消滅します。

### 対策:結果を「観測される」ようにする

コンパイラに「この値は外から見られる」と思わせれば、消せなくなります。`volatile` な変数への書き込みを使います。

```cpp
namespace bench
{
    // 最適化除去よけの捨て場
    inline volatile std::uintptr_t g_sink = 0;

    // 値を「使われたこと」にする
    inline void Escape(std::uintptr_t value) noexcept
    {
        g_sink = g_sink + value;
    }

    inline void Escape(const void* p) noexcept
    {
        Escape(reinterpret_cast<std::uintptr_t>(p));
    }
}
```

`g_sink` は `volatile` なので、コンパイラは「読み書きのたびに実際にメモリアクセスが必要」と判断します。したがって、`value` を計算する処理も消せません。

`Escape` はメモリストア1回ぶんのコスト(数 ns)を伴います。測定対象が数 ns の場合、この分がノイズとして乗ることは意識しておいてください。ただし、比較したい2つの実装に同じように乗るので、**相対比較には影響しません**。

> **`#pragma optimize("", off)` ではダメなのか**
> 関数単位で最適化を切る方法もあります。しかしそれでは「最適化された状態での速度」が測れません。私たちが知りたいのは製品ビルドでの性能なので、最適化は効かせたまま、消されないようにする——という方針を取ります。

### 効果を確認する

```cpp
int main()
{
    std::vector<int> data(1024, 1);

    bench::PrintHeader();

    // 結果を捨てる版
    auto r1 = bench::MeasureBatch(1000, 100, [&] {
        int sum = 0;
        for (int v : data) sum += v;
    });
    bench::Print("合計(結果を捨てる)", r1);

    // 結果を逃がす版
    auto r2 = bench::MeasureBatch(1000, 100, [&] {
        int sum = 0;
        for (int v : data) sum += v;
        bench::Escape(static_cast<std::uintptr_t>(sum));
    });
    bench::Print("合計(Escape あり)", r2);
}
```

Release 構成での結果例です。

```
測定対象                     中央値[ns]        p95[ns]         最大[ns]           平均[ns]
合計(結果を捨てる)       median=      0.0  p95=      0.0  max=       10.0   (mean=      0.1)
合計(Escape あり)        median=     58.0  p95=     61.0  max=      940.0   (mean=     59.3)
```

**上は消えています。下は実際に計算しています。**

1024 個の `int` の合計に 58 ns。1要素あたり 0.057 ns。CPU が SIMD 命令を使って一度に複数要素を足しているので、これは妥当な数字です。

`Escape` を書き忘れると `0.0 ns` という「夢のような結果」が出ます。**測定結果が異様に速いときは、まず消されていないか疑ってください。** これはベンチマークで最も頻繁に起きる事故です。

---

## 4.7 なぜ平均値を信じないのか

上の結果をもう一度見てください。

```
median=58.0   p95=61.0   max=940.0   (mean=59.3)
```

中央値 58 ns に対して、**最大値は 940 ns**。16倍です。

平均値は 59.3 で、中央値とほとんど同じです。**平均値は、この 940 ns の存在を教えてくれません。**

### 何が起きているのか

たまに現れる遅いサンプルの正体は、たいてい次のどれかです。

- OS のスケジューラが別のスレッドに CPU を渡した
- 割り込みが入った
- CPU がコアを移動し、キャッシュが冷えた
- 動作周波数が変動した

つまり、測定対象のコードのせいではありません。ノイズです。

### ではなぜ最大値を見るのか

ノイズなら無視していいのでは、と思うかもしれません。しかし、実際のアプリケーションでも同じノイズは起きます。「実行環境のせいだから許される」わけではありません。フレームが落ちれば、プレイヤーには等しくカクつきとして見えます。

そして重要なのは、これから測る **アロケーターの最悪値には、ノイズではない本物のスパイクが混ざる** ことです。

- OS に新しいメモリを要求しに行った
- 空き領域を探すのに時間がかかった
- ページフォルトが起きた

これらは実装によって発生頻度が変わります。**アロケーターを設計で改善できる部分**です。

だから本書では、常にこの4つを並べて見ます。

| 指標 | 何を教えてくれるか |
|---|---|
| **中央値** | 普段のコスト。最適化の効果はここに出る |
| **p95** | 「たまに」がどれくらい悪いか |
| **最大値** | いちばん悪い日に何が起きるか |
| 平均 | (参考) |

---

## 4.8 Debug と Release の差を見る

第1章で「計測は必ず Release」と書きました。その理由を数字で確認しましょう。

同じプログラムを Debug 構成でビルドして実行してください。

```
合計(結果を捨てる)       median=   1180.0  p95=   1250.0  max=     8900.0
合計(Escape あり)        median=   1210.0  p95=   1280.0  max=     9100.0
```

Release で 58 ns だったものが、Debug では 1210 ns。**約 20 倍** です。

さらに注目すべきは、Debug では「結果を捨てる版」も消えていないことです。Debug 構成では最適化そのものが行われないため、書いたコードがそのまま実行されます。

Debug 構成のこの遅さは、次のような要因によります。

- インライン展開されない
- レジスタに置かず、毎回メモリに書き戻す
- イテレータの境界チェックが入る(`_ITERATOR_DEBUG_LEVEL`)
- SIMD 化されない

**Debug で測った数字には、意味がありません。** 20倍のノイズの中で「10%速くなった」を議論しても無駄です。

> **ただし Debug も無価値ではありません**
> 第3章のテストは Debug で走らせます(`assert` が Release では消えるため)。役割分担です。
> **テストは Debug、計測は Release。**

---

## 4.9 測定環境を整える

同じコードでも、環境によって数字は変わります。信頼できる測定のために、次を守ってください。

**必須**

- Release 構成、x64
- **Ctrl + F5**(デバッグなしで開始)で実行する
  - F5 でデバッガを付けると、それだけで数割遅くなります
- 重いアプリケーション(ブラウザ、ビルド、ウイルススキャン)を閉じる

**できれば**

- Windows の電源プランを「高パフォーマンス」にする
  - 省電力設定では CPU の周波数が上下し、測定がばらつきます
- 同じ測定を3回実行し、中央値どうしを比べる
- ノートPCなら電源に接続する

**心構え**

絶対値は環境依存です。「私の環境では 58 ns だった」という数字自体には、あまり意味がありません。

本書で意味があるのは、**同じ環境で測った2つの実装の比**です。「`new` の 1/10 になった」なら、それはどの環境でもおおむね再現します。数字を追いかけるときは、常に比を見てください。

---

## 4.10 この章の完成コード

```cpp
#include <algorithm>
#include <chrono>
#include <cstdint>
#include <print>
#include <vector>

// =========================================================
// ベンチマークヘルパー
// =========================================================
namespace bench
{
    // --- 最適化除去よけ ---------------------------------
    inline volatile std::uintptr_t g_sink = 0;

    inline void Escape(std::uintptr_t value) noexcept
    {
        g_sink = g_sink + value;
    }

    inline void Escape(const void* p) noexcept
    {
        Escape(reinterpret_cast<std::uintptr_t>(p));
    }

    // --- 測定結果(単位:ナノ秒) ------------------------
    struct Result
    {
        std::size_t samples = 0;
        double      min     = 0.0;
        double      median  = 0.0;
        double      p95     = 0.0;
        double      max     = 0.0;
        double      mean    = 0.0;
    };

    // --- 個別測定 ---------------------------------------
    template <class F>
    Result Measure(std::size_t sampleCount, F&& body)
    {
        for (int i = 0; i < 3; ++i) body();

        std::vector<double> ns;
        ns.reserve(sampleCount);

        for (std::size_t i = 0; i < sampleCount; ++i)
        {
            const auto a = std::chrono::steady_clock::now();
            body();
            const auto b = std::chrono::steady_clock::now();
            ns.push_back(std::chrono::duration<double, std::nano>(b - a).count());
        }

        std::ranges::sort(ns);

        Result r;
        r.samples = sampleCount;
        r.min     = ns.front();
        r.median  = ns[sampleCount / 2];
        r.p95     = ns[std::min<std::size_t>(sampleCount - 1,
                       static_cast<std::size_t>(sampleCount * 0.95))];
        r.max     = ns.back();

        double total = 0.0;
        for (double v : ns) total += v;
        r.mean = total / static_cast<double>(sampleCount);

        return r;
    }

    // --- バッチ測定 -------------------------------------
    template <class F>
    Result MeasureBatch(std::size_t sampleCount, std::size_t opsPerSample, F&& body)
    {
        Result r = Measure(sampleCount, [&] {
            for (std::size_t i = 0; i < opsPerSample; ++i)
            {
                body();
            }
        });

        const double d = static_cast<double>(opsPerSample);
        r.min /= d;  r.median /= d;  r.p95 /= d;  r.max /= d;  r.mean /= d;

        return r;
    }

    // --- 表示 -------------------------------------------
    inline void Print(const char* label, const Result& r)
    {
        std::println("{:<22} median={:>9.1f}  p95={:>9.1f}  max={:>11.1f}   (mean={:>9.1f})",
                     label, r.median, r.p95, r.max, r.mean);
    }
}
```

`Bump` クラスとテスト関数は前章のまま残しておいてください。次章で合流します。

---

## 演習

**演習4-1** `Escape` を使わずに、戻り値のある関数として書く方法を考えてください。たとえば `sum` を `main` の戻り値にすると、最適化は消せなくなります。どちらが測定手段として使いやすいでしょうか。

**演習4-2** `MeasureBatch` の `opsPerSample` を 1、10、100、10000 と変えて、同じ処理を測ってみてください。中央値はどう変化しますか。最大値はどうですか。なぜそうなるか説明してください。

**演習4-3** `std::vector<int>` のサイズを 1024 → 1,000,000 に増やして測ってください。1要素あたりの時間はどう変わりますか。(答えは第32章で扱います)

**演習4-4** `Result` に「標準偏差」を足してみてください。追加したうえで、それが中央値・p95・最大値より役に立つ場面があるか考えてください。

**演習4-5** わざと重い処理(1ミリ秒かかるループなど)を作り、`Measure` で個別測定してください。分解能 100 ns の制約は、この場合問題になりますか。

---

## 章末チェックリスト

- [ ] 時計の実測分解能を自分の環境で確認した(おそらく 100 ns)
- [ ] `now()` の呼び出しコストがゼロでないことを確認した
- [ ] `Measure` と `MeasureBatch` を実装した
- [ ] `Escape` なしだと測定結果が 0.0 になることを **自分の目で見た**
- [ ] 中央値と最大値が大きく違うことを確認した
- [ ] Debug と Release で桁が違うことを確認した
- [ ] 「テストは Debug、計測は Release」を覚えた

---

## 次章の予告

道具が揃いました。次章で、いよいよ `Bump` と `new` を戦わせます。

20行のクラスが、何十年も磨かれてきた `new` に勝てるのか。勝てるとしたら、なぜなのか。負ける部分はどこなのか。

そして、中央値だけでなく **最大値** を並べたとき、両者の差がどう見えるか。ここに、この本全体を貫く主題が現れます。

---

> **コラム:3つの時計と、その使い分け**
>
> C++ の標準ライブラリには時計が3種類あります。用途が違います。
>
> **`std::system_clock`** は「壁時計」です。現在の日時を知るためのもので、`time_t` に変換できます。ただし、NTP による時刻同期やユーザーの操作で **巻き戻ることがあります**。時間の測定には使えません。
>
> **`std::steady_clock`** は「ストップウォッチ」です。単調増加が保証されており、決して巻き戻りません。起点に意味はなく、差だけが意味を持ちます。**測定に使うべきはこれです。**
>
> **`std::high_resolution_clock`** は名前に惹かれますが、罠があります。規格上は「最も分解能の高い時計」ですが、実装によって `system_clock` の別名だったり `steady_clock` の別名だったりします。MSVC では `steady_clock` の別名なので実害はありませんが、移植を考えると使わないほうが無難です。
>
> ---
>
> MSVC の `steady_clock` が内部で使っている `QueryPerformanceCounter` にも歴史があります。
>
> かつて QPC の実装は混乱を極めていました。マルチコア環境で CPU ごとに TSC(タイムスタンプカウンタ)がずれ、スレッドがコアを移動すると **時間が逆行する** ことがありました。省電力機能で CPU の周波数が変わると、TSC の刻む速度まで変わってしまう問題もありました。当時のゲーム開発者は、`SetThreadAffinityMask` でスレッドを特定のコアに固定するといった対策を強いられていました。
>
> 現在は、周波数変動の影響を受けない **invariant TSC** をハードウェアが提供し、Windows 側も QPC の周波数を 10 MHz に正規化しています。おかげで、私たちは `steady_clock::now()` と書くだけで済みます。
>
> アロケーターの話とは直接関係ありませんが、「単純に見える API の裏に、解決済みの厄介な歴史がある」という構図は、この本で何度も出てきます。第12章から始まるアロケーターの歴史も、まったく同じ構図です。
