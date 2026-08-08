# 第50章 テストとファジング

---

## この章のゴール

**第8部が始まります。仕上げの4章です。**

ここまで、各章でテストを書いてきました。**しかし、断片的です。**

```cpp
void Test_CoalesceBoth()
{
    heap.Free(a);
    heap.Free(c);
    heap.Free(b);      // ← この順序だから通る
}
```

**手で書いたテストは、自分が思いついた状況しか検査できません。**

この章では、**機械に探させます。**

- アロケーターが常に満たすべき **不変条件** を洗い出す
- ランダムな操作列を大量に流す(**ファジング**)
- **参照実装と突き合わせる**(モデルベーステスト)
- 書き込んだ内容が壊れていないかを検証する
- **失敗した操作列を、最小の再現手順に縮める**

**10 万操作で見つかったバグが、5 操作で再現できるようになります。**

---

## 50.1 手で書いたテストの限界

### 組み合わせ爆発

**5個のブロックを確保し、任意の順序で解放する。**

```
確保の順序 : 5! = 120 通り
解放の順序 : 5! = 120 通り
サイズの組み合わせ : サイズが 3 種類なら 3^5 = 243 通り
────────────────────────────────────────────
合計 : 120 × 120 × 243 = 3,499,200 通り
```

**5個でこれです。**

**手で書けるのは、せいぜい数十通りです。**

### 実際のバグは、思いつかなかった順序で起きる

**第25章で、こんな警告を書きました。**

> **`UnlinkFree` は、サイズが変わる前に呼ばなければなりません。**

**この順序を間違えるバグは、次の条件が揃ったときだけ表面化します。**

```
① 合体が起きる
② 合体前と合体後で、ブロックが別のビンに移る
③ そのビンに他のブロックがある
```

**3つが揃う操作列を、手で思いつけるでしょうか。**

**ファジングなら、数秒で見つけます。**

---

## 50.2 不変条件を洗い出す

**「アロケーターが、いつでも満たしているべき性質」を列挙します。**

| # | 不変条件 | 破れるとどうなるか |
|---|---|---|
| ① | **確保した領域どうしは重ならない** | 2つのオブジェクトが同じメモリを共有 |
| ② | 確保した領域は、板の中に収まる | 板の外を破壊 |
| ③ | アラインメント要求を満たす | 第6章の問題 |
| ④ | ブロックの合計サイズ = 容量 | ブロックが失われている |
| ⑤ | フリーリストが輪になっていない | 無限ループ |
| ⑥ | フリーリストの要素は、すべて空きフラグが立っている | 使用中ブロックが配られる |
| ⑦ | 隣接する空きブロックが存在しない(合体後) | 合体の漏れ |
| ⑧ | ビンの中のブロックは、そのビンの範囲内 | 探索が失敗する |
| ⑨ | 全部解放したら、1つの空きブロックに戻る | 断片化が残っている |
| ⑩ | `PREV_FREE` フラグが実際と一致する | 第24章の合体が壊れる |

**①が最も重要です。** 第3章で「アロケーターの最も基本的な契約」と書いたものです。

### `Validate()` を実装する

```cpp
    // すべての不変条件を検査する。Debug 専用の重い処理。
    bool Validate(std::string* outError = nullptr) const
    {
        auto Fail = [outError](std::string_view msg) {
            if (outError != nullptr) { *outError = msg; }
            return false;
        };

        // --- ④⑦⑩:ブロックを先頭から歩く ---
        std::size_t total = 0;
        bool        prevWasFree = false;
        const std::byte* p = base_;
        const std::byte* end = base_ + capacity_;

        while (p < end)
        {
            const auto* h = reinterpret_cast<const Header*>(p);
            const std::size_t size = detail::SizeOf(h);
            const bool isFree = detail::IsFree(h);

            if (size == 0)                 { return Fail("サイズが 0 のブロック"); }
            if (size % kDefaultAlignment)  { return Fail("サイズが整列していない"); }
            if (p + size > end)            { return Fail("ブロックが板をはみ出している"); }

            // ⑩ PREV_FREE が実際と一致するか
            if (p != base_ && detail::PrevIsFree(h) != prevWasFree)
            {
                return Fail("PREV_FREE フラグが実際と一致しない");
            }

            // ⑦ 隣接する空きが並んでいないか
            if (isFree && prevWasFree)
            {
                return Fail("隣接する空きブロックが合体されていない");
            }

            // 空きブロックのフッタが正しいか
            if (isFree && detail::FooterOf(const_cast<Header*>(h)) != size)
            {
                return Fail("フッタがサイズと一致しない");
            }

            total      += size;
            prevWasFree = isFree;
            p          += size;
        }

        if (total != capacity_) { return Fail("ブロックの合計が容量と一致しない"); }

        // --- ⑤⑥⑧:フリーリストを検査する ---
        for (std::size_t bin = 0; bin < kBinCount; ++bin)
        {
            // Floyd の循環検出
            const Header* slow = bins_[bin];
            const Header* fast = bins_[bin];

            while (fast != nullptr && fast->nextFree != nullptr)
            {
                slow = slow->nextFree;
                fast = fast->nextFree->nextFree;

                if (slow == fast) { return Fail("フリーリストが輪になっている"); }
            }

            for (const Header* h = bins_[bin]; h != nullptr; h = h->nextFree)
            {
                if (!detail::IsFree(h))          { return Fail("使用中ブロックがフリーリストにある"); }
                if (BinOf(detail::SizeOf(h)) != bin) { return Fail("ブロックが違うビンにある"); }
            }

            // ビットマップとの整合
            const bool bitSet = (binMap_[bin >> 6] >> (bin & 63)) & 1u;
            if (bitSet != (bins_[bin] != nullptr))
            {
                return Fail("ビットマップとフリーリストが一致しない");
            }
        }

        return true;
    }
```

**⑤の循環検出には、Floyd のアルゴリズム(亀とうさぎ)を使っています。** 2つのポインタを別の速度で進め、追いついたら循環です。**追加のメモリを使いません。**

> **`Validate()` は O(n) です。** 毎回呼ぶと非常に遅くなります。50.7 節で、呼ぶ頻度を扱います。

---

## 50.3 ファジングの基本形

```cpp
// Playground/Fuzz.h
#pragma once

#include "ga/Allocator.h"

#include <random>
#include <vector>

namespace fuzz
{
    struct Op
    {
        enum class Kind { Allocate, Free };

        Kind        kind = Kind::Allocate;
        std::size_t value = 0;    // Allocate ならサイズ、Free なら「何番目の生存ブロックか」
    };

    // 操作列を生成する
    inline std::vector<Op> GenerateSequence(std::uint32_t seed, std::size_t opCount)
    {
        std::mt19937 rng(seed);

        std::discrete_distribution<int> sizeClass({ 50, 30, 15, 4, 1 });
        const std::size_t sizes[] = { 32, 128, 512, 4096, 65536 };

        std::uniform_real_distribution<double> action(0.0, 1.0);
        std::uniform_int_distribution<std::size_t> pick(0, 1'000'000);

        std::vector<Op> ops;
        ops.reserve(opCount);

        for (std::size_t i = 0; i < opCount; ++i)
        {
            if (action(rng) < 0.45)
            {
                ops.push_back(Op{ Op::Kind::Free, pick(rng) });
            }
            else
            {
                ops.push_back(Op{ Op::Kind::Allocate, sizes[sizeClass(rng)] });
            }
        }

        return ops;
    }
}
```

### 操作列を実行する

```cpp
    struct RunResult
    {
        bool        ok       = true;
        std::string error;
        std::size_t failedAt = 0;
    };

    inline RunResult RunSequence(const std::vector<Op>& ops,
                                 std::size_t validateEvery = 100)
    {
        ga::FreeList heap(1 * 1024 * 1024);
        std::vector<void*> live;

        for (std::size_t i = 0; i < ops.size(); ++i)
        {
            const Op& op = ops[i];

            if (op.kind == Op::Kind::Allocate)
            {
                if (void* p = heap.Allocate(op.value)) { live.push_back(p); }
            }
            else
            {
                if (!live.empty())
                {
                    const std::size_t index = op.value % live.size();

                    heap.Free(live[index]);
                    live[index] = live.back();
                    live.pop_back();
                }
            }

            if ((i % validateEvery) == 0)
            {
                std::string err;
                if (!heap.Validate(&err))
                {
                    return RunResult{ false, err, i };
                }
            }
        }

        // 最後に、全部解放して ⑨ を検査する
        for (void* p : live) { heap.Free(p); }

        std::string err;
        if (!heap.Validate(&err)) { return RunResult{ false, err, ops.size() }; }

        if (heap.FreeBlockCount() != 1)
        {
            return RunResult{ false, "全部解放したのに、空きブロックが1つでない", ops.size() };
        }

        return RunResult{};
    }
```

### ⚠ 種を記録する

**再現できなければ、意味がありません。**

```cpp
int main()
{
    for (std::uint32_t seed = 0; seed < 100'000; ++seed)
    {
        auto ops = fuzz::GenerateSequence(seed, 10'000);
        auto result = fuzz::RunSequence(ops);

        if (!result.ok)
        {
            std::println("✗ 失敗:seed={} 操作 {} 番目", seed, result.failedAt);
            std::println("  {}", result.error);
            return 1;
        }

        if (seed % 1000 == 0) { std::println("  seed {} まで通過", seed); }
    }

    std::println("✓ すべて通過");
}
```

**「seed = 42731 で失敗」と分かれば、いつでも同じ状況を再現できます。**

---

## 50.4 参照実装と突き合わせる

**`Validate()` は「内部が壊れていないか」を見ます。**

**しかし、「返された領域が正しいか」は見ていません。**

### モデルベーステスト

**単純な参照実装を用意し、突き合わせます。**

```cpp
    // 使用中の領域を、単純に記録するだけのモデル
    class ReferenceModel
    {
    public:
        // 新しい領域が、既存のものと重ならないか
        bool TryAdd(std::size_t offset, std::size_t size, std::string* err)
        {
            auto it = live_.upper_bound(offset);

            // 直前の領域と重なるか
            if (it != live_.begin())
            {
                auto prev = std::prev(it);
                if (prev->first + prev->second > offset)
                {
                    *err = "確保した領域が、既存の領域と重なっている";
                    return false;
                }
            }

            // 直後の領域と重なるか
            if (it != live_.end() && offset + size > it->first)
            {
                *err = "確保した領域が、既存の領域と重なっている";
                return false;
            }

            live_[offset] = size;
            return true;
        }

        void Remove(std::size_t offset) { live_.erase(offset); }

        std::size_t TotalBytes() const
        {
            std::size_t total = 0;
            for (const auto& [off, size] : live_) { total += size; }
            return total;
        }

    private:
        std::map<std::size_t, std::size_t> live_;
    };
```

**`std::map` で使用中の領域を管理するだけです。** 遅いですが、**正しさは自明です。**

### 突き合わせる

```cpp
            if (op.kind == Op::Kind::Allocate)
            {
                void* p = heap.Allocate(op.value);
                if (p == nullptr) { continue; }

                const std::size_t offset =
                    static_cast<const std::byte*>(p) - heap.Base();

                // ① 重なっていないか
                if (!model.TryAdd(offset, op.value, &err))
                {
                    return RunResult{ false, err, i };
                }

                // ② 板の中に収まっているか
                if (offset + op.value > heap.Capacity())
                {
                    return RunResult{ false, "板をはみ出している", i };
                }

                // ③ アラインメント
                if (offset % ga::kDefaultAlignment != 0)
                {
                    return RunResult{ false, "アラインメント違反", i };
                }

                live.push_back(Live{ p, offset, op.value });
            }
```

**不変条件①②③が、確保のたびに検査されます。**

> **これが、最も強力な検査です。**
>
> **「単純だが確実に正しい実装」と、「複雑だが速い実装」を突き合わせる。** アルゴリズムのテストにおける定石です。

---

## 50.5 内容の破壊を検出する

**「重なっていない」ことは検査しました。**

**しかし、他の経路でメモリが壊れるかもしれません。** 第24章の合体処理が、隣のブロックのヘッダを書き換えるかもしれない。

**書き込んだ内容を、読み返して検証します。**

```cpp
    struct Live
    {
        void*         ptr;
        std::size_t   offset;
        std::size_t   size;
        std::uint32_t token;     // このブロックの識別子
    };

    // 確保直後に埋める
    void Fill(const Live& b)
    {
        auto* p = static_cast<std::uint32_t*>(b.ptr);
        const std::size_t count = b.size / sizeof(std::uint32_t);

        for (std::size_t i = 0; i < count; ++i)
        {
            p[i] = b.token ^ static_cast<std::uint32_t>(i);
        }
    }

    // 解放直前に検証する
    bool Check(const Live& b, std::string* err)
    {
        const auto* p = static_cast<const std::uint32_t*>(b.ptr);
        const std::size_t count = b.size / sizeof(std::uint32_t);

        for (std::size_t i = 0; i < count; ++i)
        {
            if (p[i] != (b.token ^ static_cast<std::uint32_t>(i)))
            {
                *err = std::format("領域が破壊されている(オフセット +{})",
                                   i * sizeof(std::uint32_t));
                return false;
            }
        }
        return true;
    }
```

**トークンに `i` を XOR しているのは、位置ごとに違う値にするためです。**

**ブロック全体が同じ値だと、「隣のブロックからずれてコピーされた」ことを検出できません。**

> **第34章のマルチスレッド実験で、同じ手法を使いました。** 各スレッドが自分の ID を書き、読み返す。**壊れ方を特定するための、汎用的な技法です。**

### 全生存ブロックを毎回検証すると遅い

```
生存ブロックが 1000 個、平均 512 バイト
→ 1回の検証で 512 KB の走査
→ 1万操作で 5 GB の走査
```

**間引きます。**

```cpp
            // 100 操作に1回、全部を検証する
            if ((i % 100) == 0)
            {
                for (const Live& b : live)
                {
                    if (!Check(b, &err)) { return RunResult{ false, err, i }; }
                }
            }
```

---

## 50.6 最小再現に縮める

**ファジングで見つかったバグは、たいてい巨大です。**

```
seed = 42731、10,000 操作の 8,432 番目で失敗
```

**8,432 操作をデバッガで追うのは、現実的ではありません。**

**しかし、本当に必要なのは、そのうち数操作かもしれません。**

### デルタデバッギング

**考え方は単純です。**

> **操作列の一部を削って、まだ失敗するなら、削った部分は不要だった。**

```cpp
    // 失敗する操作列を、できるだけ短くする
    inline std::vector<Op> Shrink(std::vector<Op> ops)
    {
        auto Fails = [](const std::vector<Op>& candidate) {
            return !RunSequence(candidate, 1).ok;     // 毎回検証する
        };

        if (!Fails(ops)) { return ops; }              // そもそも失敗しない

        bool changed = true;

        while (changed)
        {
            changed = false;

            // 大きな塊から順に、削れるか試す
            for (std::size_t chunk = ops.size() / 2; chunk >= 1; chunk /= 2)
            {
                for (std::size_t i = 0; i + chunk <= ops.size(); )
                {
                    std::vector<Op> candidate;
                    candidate.reserve(ops.size() - chunk);

                    candidate.insert(candidate.end(), ops.begin(), ops.begin() + i);
                    candidate.insert(candidate.end(), ops.begin() + i + chunk, ops.end());

                    if (Fails(candidate))
                    {
                        ops     = std::move(candidate);
                        changed = true;
                        // i はそのまま(削れた位置から再試行)
                    }
                    else
                    {
                        i += chunk;
                    }
                }

                if (chunk == 1) { break; }
            }
        }

        return ops;
    }
```

**大きな塊から削り、だんだん細かくします。** 二分探索に似た戦略です。

### 実行例

```
=== 縮小 ===
  元の操作列 : 10,000 操作
  ステップ 1 : 5,000 操作(失敗する)
  ステップ 2 : 2,500 操作(失敗する)
  ステップ 3 : 1,250 操作(失敗する)
  ...
  最終       : 6 操作

  1. Allocate(1008)  → A
  2. Allocate(64)    → B
  3. Allocate(1008)  → C
  4. Free(A)
  5. Free(B)
  6. Allocate(1024)  → ここで Validate が失敗
     「ブロックが違うビンにある」
```

**10,000 操作が 6 操作になりました。**

### この 6 操作が語ること

**第25章の警告が、まさにこれです。**

```
① 1008 バイト → ビン 63(小さいビンの最大)
② 64 バイト
③ 1008 バイト

④ A を解放 → ビン 63 に入る
⑤ B を解放 → A と B が合体して 1072 バイト
             → ビン 64(大きいビン)に移るべき
             → しかし UnlinkFree を呼ぶ順序を間違えていると…
```

**小さいビンと大きいビンの境界という、境界値のバグです。**

**手で書いたテストで、これを思いつくのは困難です。**

> **縮小は、デバッグの前段階として極めて有効です。**
>
> **6 操作なら、デバッガで1つずつ追えます。** 8,432 操作では不可能でした。

---

## 50.7 実行時間との折り合い

### 検証の頻度

```
                    1000万操作の実行時間
検証なし                    2.1 秒
1000 操作に1回              4.8 秒
100 操作に1回              22.4 秒
10 操作に1回              184.0 秒
毎回                     1,820.0 秒(30 分)
```

**`Validate()` は O(n) なので、頻度が上がると急激に遅くなります。**

### 使い分け

| 場面 | 頻度 | 目的 |
|---|---|---|
| **開発中** | 1000 操作に1回 | 素早く回す |
| **CI(毎回)** | 100 操作に1回 | 数分で終わる範囲 |
| **CI(夜間)** | 100 操作に1回 × 大量の seed | 広く探す |
| **縮小するとき** | **毎回** | 正確な位置を特定する |

**縮小のときだけ、毎回検証します。** 「どの操作で壊れたか」を正確に知る必要があるからです。

### AddressSanitizer と併用する

**第31章の ASan を有効にして、ファジングを走らせます。**

```
ファジング単体      : 不変条件の違反を検出
ファジング + ASan   : + 範囲外アクセス、解放後使用、リーク
```

**組み合わせると、検出力が大きく上がります。**

**速度は 2〜3 倍落ちますが、CI の夜間実行なら問題ありません。**

---

## 50.8 実際に見つかるバグ

**この手法で見つかりやすいバグを、3つ挙げます。**

### ① 境界値

```
size == kSmallMax ちょうど(1024)
size == kMinBlockSize ちょうど(32)
分割後の残りが kMinBlockSize ちょうど
容量ぴったりの確保
```

**「ちょうど」の値で、条件式の `<` と `<=` を間違えます。**

**乱数だけでは、境界値になかなか当たりません。**

**対策:境界値を、意図的に混ぜます。**

```cpp
    const std::size_t sizes[] = {
        32, 128, 512, 4096, 65536,          // 通常
        1008, 1016, 1024, 1032,             // 小さいビンの境界(1024)
        16, 24, 32, 40,                     // 最小ブロックの境界
    };
```

**「乱数 + 境界値」が、実用的な組み合わせです。**

### ② 順序依存

**50.6 節で見た例です。**

```
UnlinkFree をサイズ変更の後に呼ぶ
MarkFree をフッタ書き込みの前に呼ぶ
Reset で塗りつぶしを破棄の前に行う(第16章)
```

**「手順が正しい順序で書かれているか」は、特定の状況でしか表面化しません。**

### ③ 統計のずれ

```
liveCount_ が実際のブロック数と合わない
internalWaste_ が二重に加算される
Peak() が更新されない経路がある
```

**動作には影響しませんが、第49章の予算管理が狂います。**

**`Validate()` に統計の検査を足すと見つかります。**

```cpp
        // 統計の整合を検査する
        std::size_t actualLive = 0;
        ForEachBlock([&](const Header*, std::size_t, bool isFree) {
            if (!isFree) { ++actualLive; }
        });

        if (actualLive != liveCount_) { return Fail("liveCount_ が実際と一致しない"); }
```

---

## 50.9 テストの階層

**5つの層があります。**

| 層 | 何を見つけるか | コスト |
|---|---|---|
| **単体テスト**(手書き) | 基本的な動作、意図した仕様 | 低 |
| **不変条件検査** | 内部構造の破壊 | 中 |
| **ファジング** | 思いつかなかった操作列 | 中 |
| **モデルベース** | 出力の正しさ | 高 |
| **実ワークロード** | 現実の使われ方での性能と正しさ | 高 |

**どれか1つでは足りません。**

- **単体テストは、仕様を文書化します。** 「この関数はこう動くべき」を残します
- **ファジングは、仕様外の穴を見つけます**
- **モデルベースは、正しさの基準を与えます**
- **実ワークロードは、第52章で扱います**

### 推奨する構成

```
コミット前  : 単体テスト(数秒)
プルリクエスト: 単体テスト + ファジング 10 万操作(1 分)
毎晩        : ファジング 1 億操作 + ASan(数時間)
```

**「毎晩、大量に流す」ことが、最も費用対効果が高い。** 開発者を待たせずに、広く探せます。

---

## 演習

**演習50-1** `Validate()` を実装し、意図的にバグを入れて検出されることを確認してください。

**演習50-2** `Tlsf`(第27章)用の `Validate()` を書いてください。二段ビットマップの整合を、どう検査しますか。

**演習50-3** `Pool`(第22章)用のファジングを書いてください。不変条件は何ですか。

**演習50-4** 縮小を実装し、10,000 操作の失敗を何操作まで縮められるか試してください。

**演習50-5** 境界値をサイズの候補に混ぜて、見つかるバグが増えるか確認してください。

**演習50-6** ASan を有効にしてファジングを走らせ、実行時間の増加を測ってください。

**演習50-7** マルチスレッド版のファジングを書いてください。何が難しくなりますか(ヒント:再現性)。

**演習50-8** `Validate()` の統計検査を追加し、意図的に `liveCount_` を狂わせて検出させてください。

---

## 章末チェックリスト

- [ ] 手で書いたテストの限界を、組み合わせ数で説明できる
- [ ] 不変条件を 10 個挙げられる
- [ ] `Validate()` を実装し、Floyd の循環検出を使った
- [ ] ファジングを実装し、**種を記録** した
- [ ] 参照実装と突き合わせるモデルベーステストを実装した
- [ ] 書き込んだ内容の破壊を検出する仕組みを作った
- [ ] **失敗を最小再現に縮めた**
- [ ] 検証の頻度と実行時間の関係を測った
- [ ] 境界値を意図的に混ぜる必要性を理解した
- [ ] テストの5つの層と、それぞれの役割を説明できる

---

## 次章の予告

**第51章は、デバッグ機能の整理です。**

この本を通して、たくさんの検査機能を作ってきました。

```
統計、タグ、確保元の記録、スタックトレース、
塗りつぶし、ガードバイト、ガードページ、
スレッドの所有者検査、不変条件検査…
```

**すべてを有効にすると、どれだけ遅くなるのか。**

そして、**それぞれをどう切り替えるのか。**

これまで `#if` マクロを使ってきましたが、**第9章と第14章で警告したとおり、「メンバを消すと ODR 違反になる」** という問題がありました。

第51章では、これを **テンプレートパラメータによる設定** に整理します。

```cpp
using DebugBump   = BasicBump<DebugConfig>;
using ReleaseBump = BasicBump<ReleaseConfig>;
```

**型が別物になるので、混ぜてもリンクエラーで気づけます。**

そして、Debug と Release の中間——**「開発ビルド」** という第3の構成を設計します。実務では、これが最も長く使われる構成です。

---

> **コラム:ファジングの発見**
>
> **1988 年、ウィスコンシン大学の Barton Miller は、嵐の夜に自宅からモデム経由で Unix にログインしていました。**
>
> 回線のノイズで、入力にランダムな文字が混ざります。**そして、いくつものプログラムがクラッシュしました。**
>
> **「ランダムな入力を与えるだけで、これほど壊れるのか」**
>
> 彼はこれを研究テーマにしました。学生に課題を出し、**ランダムな文字列を Unix のユーティリティに与える** という実験を行いました。
>
> **結果は衝撃的でした。** 標準的なユーティリティの相当数が、クラッシュまたはハングしました。
>
> **これが「ファジング」という言葉と手法の始まりです。**
>
> ---
>
> **なぜ、そこまで壊れたのか**
>
> **プログラマは、正しい入力しかテストしていなかったからです。**
>
> 「ファイル名を渡す」「数値を渡す」という想定の下で書かれたコードに、**ランダムなバイト列を渡す** ことを、誰も試していませんでした。
>
> **50.1 節で書いたとおりです。** 手で書いたテストは、**自分が思いついた状況しか検査しません。**
>
> ---
>
> **その後の発展**
>
> **カバレッジ誘導型ファジング。** ランダムに入力を作るだけでなく、**「まだ実行されていないコードに到達する入力」を優先的に生成します。**
>
> 入力を少し変異させ、新しいコードパスに到達したら、その入力を種として保存する。**進化的アルゴリズムのような動作です。**
>
> `AFL` や `libFuzzer` といったツールが、この方式を実用化しました。**多数の重大な脆弱性が、この手法で発見されています。**
>
> **プロパティベーステスト。** Haskell の `QuickCheck` が広めた考え方です。
>
> 「入力を大量に生成し、**性質(プロパティ)が成り立つか** を検査する」——**50.2 節の不変条件と同じ発想です。**
>
> **そして `QuickCheck` は、失敗した入力を自動的に縮小する機能を持っていました。** 50.6 節で実装したものです。
>
> ---
>
> **アロケーターは、ファジングに向いている**
>
> **理由が3つあります。**
>
> **① 入力が単純。** 「サイズを指定して確保する」「解放する」だけです。ランダムな操作列を作るのが容易です。
>
> **② 不変条件が明確。** 50.2 節で 10 個挙げました。**「正しさ」を機械的に検査できます。**
>
> **③ 状態が複雑。** フリーリスト、ビン、ビットマップ、境界タグ。**組み合わせが爆発するので、手で網羅できません。**
>
> **入力が単純で、正しさが明確で、状態が複雑。** ファジングが最も効く条件が揃っています。
>
> ---
>
> **教訓**
>
> **Miller の実験から、40 年近く経ちました。**
>
> **しかし、教訓は変わっていません。**
>
> > **自分が思いつく範囲だけをテストしていては、自分が思いつかないバグは見つからない。**
>
> **機械に探させましょう。** 機械は、私たちより辛抱強く、私たちより偏見がありません。
