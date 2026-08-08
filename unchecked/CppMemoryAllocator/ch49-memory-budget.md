# 第49章 メモリ予算を決める 〔v1.0〕

---

## この章のゴール

**第7部の最後は、すべてをまとめる章です。**

ここまで、たくさんのアリーナを作ってきました。

```
永続アリーナ、シーンアリーナ × 2、
フレームアリーナ × 面数 × ワーカー数、
汎用ヒープ、GPU ヒープ(DEFAULT / UPLOAD)…
```

**合計でどれだけ使っているのか。そして、どれだけ使ってよいのか。**

```
=== メモリ予算 ===
  カテゴリ      上限      現在      ピーク    使用率   状態
  ──────────────────────────────────────────────────────
  Texture      512.0 MB  412.3 MB  468.1 MB   80.5%   OK
  Audio         96.0 MB   88.4 MB   94.2 MB   92.1%   ⚠ 警告
  Script        32.0 MB   41.8 MB   41.8 MB  130.6%   ✗ 超過
```

- 予算表を設計し、**カテゴリ別の上限** を決める
- すべてのアリーナから **横断的に集計** する 〔**v1.0**〕
- 超過を **4段階** で扱う(第7章の選択肢の回収)
- ImGui で **実行中に表示** する
- ビルドごとに記録し、**回帰を検出** する

**そして、このコードは v1.0 になります。**

第8章のピーク、第15章のタグ、第19章のタグ境界——**すべてがここに集まります。**

---

## 49.1 なぜ予算が要るのか

**第15章のコラムで書いたことを、実装に落とします。**

> **メモリ予算表は、チーム間の契約として機能します。**
> テクスチャ担当は 512 MB の中でやりくりする。超えそうなら、他のチームと交渉して枠を融通してもらうか、圧縮率を上げる。

### ゼロサムである

**総メモリは固定です。**

```
誰かが 10 MB 増やしたら、誰かが 10 MB 減らす
```

**「少しくらいいいだろう」が積み重なると、破綻します。**

### 検出が遅れると、手遅れになる

```
開発初期  : メモリに余裕がある。誰も気にしない
中盤      : だんだん増える。まだ動く
終盤      : 突然「メモリが足りない」
出荷直前  : 全チームで削減作業。品質を落とす判断を迫られる
```

**最後の段階で削るのは、最も高くつきます。** すでに作り込まれたアセットを削り直し、テストをやり直すことになります。

> **予算管理の目的は、「早く気づくこと」です。**
>
> 10 MB 超えた時点で気づけば、原因はその日の変更に絞られます。**200 MB 超えてから気づけば、原因は数か月分の変更に散らばっています。**

---

## 49.2 予算表を設計する

### 2つの軸がある

**メモリを分類する軸は、2つあります。**

```
軸① どこに置かれているか(アリーナ)
    永続 / シーン / フレーム / 汎用 / GPU

軸② 何のためのものか(タグ)
    Texture / Mesh / Audio / Script / Physics / UI
```

**予算は、軸②で決めます。**

**理由:** 「テクスチャ担当」「オーディオ担当」というチームの分担に対応するからです。**「シーンアリーナ担当」という役割は存在しません。**

**ただし、軸①も記録します。** 「テクスチャの 412 MB のうち、380 MB は GPU ヒープ」といった内訳が、削減の手がかりになります。

### 予算の定義

```cpp
// ga/Budget.h
#pragma once

#include "ga/MemoryTag.h"

#include <array>
#include <cstddef>

namespace ga
{
    enum class BudgetState
    {
        Ok,        // 警告閾値未満
        Warning,   // 警告閾値を超えた
        Over,      // 上限を超えた
    };

    struct BudgetStatus
    {
        MemoryTag   tag        = MemoryTag::General;
        std::size_t limit      = 0;
        std::size_t warnLimit  = 0;
        std::size_t current    = 0;
        std::size_t peak       = 0;
        BudgetState state      = BudgetState::Ok;

        double Ratio() const noexcept
        {
            return (limit == 0) ? 0.0
                 : static_cast<double>(current) / static_cast<double>(limit);
        }

        double PeakRatio() const noexcept
        {
            return (limit == 0) ? 0.0
                 : static_cast<double>(peak) / static_cast<double>(limit);
        }
    };
}
```

### 警告閾値を設ける

```
上限     : 512 MB
警告閾値 : 435 MB(85%)
```

**上限に達してから気づくのでは遅い。** 余裕があるうちに警告します。

**85% という数字に根拠はありません。** プロジェクトの余裕度に応じて決めてください。**厳しめのほうが安全です。**

---

## 49.3 横断的に集計する 〔v1.0〕

**問題は、「いま何バイト使っているか」をどう知るかです。**

### 第15章の統計は、そのままでは使えない

**第15章 15.2 節で、こう書きました。**

> `Stats::bytes` は「このリセット周期で **のべ何バイト確保したか**」を意味します。`Used()` とは一致しません。

**予算に必要なのは「いま生きているバイト数」です。**

### 第19章のタグ境界が使える

**第19章 19.3 節で、バンプアロケーターのタグ境界を記録しました。**

```cpp
struct TagSpan { std::size_t begin; MemoryTag tag; };
```

**バンプアロケーターは単調増加なので、境界の間はすべて同じタグです。**

**したがって、タグごとの「いま生きているバイト数」が正確に計算できます。**

```cpp
    // Bump に追加
    std::size_t LiveBytesForTag(MemoryTag tag) const noexcept
    {
        std::size_t total = 0;

        for (std::size_t i = 0; i < tagSpans_.size(); ++i)
        {
            if (tagSpans_[i].tag != tag) { continue; }

            const std::size_t begin = tagSpans_[i].begin;
            const std::size_t end   = (i + 1 < tagSpans_.size())
                                    ? tagSpans_[i + 1].begin
                                    : offset_;

            total += (end - begin);
        }

        return total;
    }
```

**第19章で可視化のために作ったものが、予算管理に転用できました。**

### プールは、まるごと1タグ

**`Pool<Particle>` は、パーティクル専用です。** 確保ごとにタグを記録する必要はありません。

```cpp
    void Register(const char* name, const Pool<T>* pool, MemoryTag tag);
```

**プール全体を1つのタグに割り当てます。** 実用上、これで十分です。

### 集計器

```cpp
// ga/BudgetTracker.h
namespace ga
{
    class BudgetTracker
    {
    public:
        static constexpr std::size_t kTagCount =
            static_cast<std::size_t>(MemoryTag::Count);

        using ViolationCallback = void (*)(const BudgetStatus&, void* user) noexcept;

        // --- 設定 ---
        void SetLimit(MemoryTag tag, std::size_t bytes, double warnRatio = 0.85) noexcept
        {
            auto& e = entries_[static_cast<std::size_t>(tag)];
            e.tag       = tag;
            e.limit     = bytes;
            e.warnLimit = static_cast<std::size_t>(bytes * warnRatio);
        }

        void SetViolationCallback(ViolationCallback cb, void* user = nullptr) noexcept
        {
            callback_ = cb;
            callbackUser_ = user;
        }

        // --- 登録 ---
        void RegisterArena(const char* name, const Bump* arena)
        {
            arenas_.push_back(ArenaEntry{ name, arena });
        }

        void RegisterFixed(const char* name, MemoryTag tag, const std::size_t* bytesPtr)
        {
            fixed_.push_back(FixedEntry{ name, tag, bytesPtr });
        }

        // --- 更新(毎フレーム、またはシーン切り替え時)---
        void Update()
        {
            for (auto& e : entries_) { e.current = 0; }

            // ① アリーナから、タグごとに集める
            for (const auto& a : arenas_)
            {
                for (std::size_t i = 0; i < kTagCount; ++i)
                {
                    entries_[i].current +=
                        a.arena->LiveBytesForTag(static_cast<MemoryTag>(i));
                }
            }

            // ② プールや GPU ヒープなど、まるごと1タグのもの
            for (const auto& f : fixed_)
            {
                entries_[static_cast<std::size_t>(f.tag)].current += *f.bytesPtr;
            }

            // ③ 判定
            for (auto& e : entries_)
            {
                if (e.current > e.peak) { e.peak = e.current; }

                const BudgetState previous = e.state;

                if      (e.limit == 0)            { e.state = BudgetState::Ok; }
                else if (e.current > e.limit)     { e.state = BudgetState::Over; }
                else if (e.current > e.warnLimit) { e.state = BudgetState::Warning; }
                else                              { e.state = BudgetState::Ok; }

                // 状態が悪化したときだけ通知する(毎フレーム鳴らさない)
                if (callback_ != nullptr && e.state > previous)
                {
                    callback_(e, callbackUser_);
                }
            }
        }

        // --- 参照 ---
        const BudgetStatus& Status(MemoryTag tag) const noexcept
        {
            return entries_[static_cast<std::size_t>(tag)];
        }

        template <class F>
        void ForEachStatus(F&& fn) const
        {
            for (const auto& e : entries_)
            {
                if (e.limit > 0 || e.current > 0) { fn(e); }
            }
        }

        std::size_t TotalLimit()   const noexcept;
        std::size_t TotalCurrent() const noexcept;
        std::size_t TotalPeak()    const noexcept;

    private:
        struct ArenaEntry { const char* name; const Bump* arena; };
        struct FixedEntry { const char* name; MemoryTag tag; const std::size_t* bytesPtr; };

        std::array<BudgetStatus, kTagCount> entries_{};
        std::vector<ArenaEntry>             arenas_;
        std::vector<FixedEntry>             fixed_;

        ViolationCallback callback_     = nullptr;
        void*             callbackUser_ = nullptr;
    };
}
```

### 設定する

```cpp
void SetupBudget(ga::BudgetTracker& budget)
{
    using ga::MemoryTag;

    budget.SetLimit(MemoryTag::Texture, 512 * 1024 * 1024);
    budget.SetLimit(MemoryTag::Mesh,    256 * 1024 * 1024);
    budget.SetLimit(MemoryTag::Audio,    96 * 1024 * 1024);
    budget.SetLimit(MemoryTag::Script,   32 * 1024 * 1024);
    budget.SetLimit(MemoryTag::Physics,  64 * 1024 * 1024);
    budget.SetLimit(MemoryTag::UI,       32 * 1024 * 1024);
    budget.SetLimit(MemoryTag::General,  48 * 1024 * 1024);

    budget.RegisterArena("永続",       &g_persistentArena);
    budget.RegisterArena("シーン0",    &g_sceneArenas[0].Arena());
    budget.RegisterArena("シーン1",    &g_sceneArenas[1].Arena());

    for (std::size_t i = 0; i < g_workers.size(); ++i)
    {
        budget.RegisterArena("フレーム", &g_workers[i].frame.Current());
    }

    budget.RegisterFixed("GPU テクスチャ", MemoryTag::Texture, &g_gpuTextureBytes);
    budget.RegisterFixed("GPU メッシュ",   MemoryTag::Mesh,    &g_gpuMeshBytes);
}
```

---

## 49.4 超過をどう扱うか

**第7章で、エラーの伝え方に4つの選択肢がありました。**

> `nullptr` / 例外 / `std::expected` / 失敗を許さない(即死)

**予算超過にも、段階があります。**

| 使用率 | 状態 | 開発ビルド | 製品ビルド |
|---|---|---|---|
| 〜85% | **OK** | 何もしない | 何もしない |
| 85〜100% | **警告** | ログ + 画面表示 | ログのみ |
| 100%超 | **超過** | **画面に大きく表示** + ログ | ログのみ |
| 確保失敗 | **致命** | **即座に停止** | フォールバック |

### 開発ビルドでは、うるさくする

**第43章 43.8 節で書いたとおりです。**

> **警告を無視できないようにしてください。** ログに1行流すだけでは、誰も見ません。

```cpp
void OnBudgetViolation(const ga::BudgetStatus& s, void*) noexcept
{
    if (s.state == ga::BudgetState::Over)
    {
        // 画面に大きく赤い文字で表示する
        g_screenWarning.Show(std::format("メモリ予算超過: {} ({} / {})",
                                         ToString(s.tag),
                                         FormatBytes(s.current),
                                         FormatBytes(s.limit)),
                             ScreenWarning::Severity::Critical);
    }

    Log(LogLevel::Error, "予算超過: {} {:.1f}%", ToString(s.tag), s.Ratio() * 100.0);
}
```

### 自動テストでは、失敗させる

**最も効果的な手段です。**

```cpp
int RunAutomatedTest()
{
    PlayThroughAllStages();

    int violations = 0;

    budget.ForEachStatus([&](const ga::BudgetStatus& s) {
        if (s.PeakRatio() > 1.0)
        {
            std::println("✗ {} がピークで {:.1f}% に達しました",
                         ToString(s.tag), s.PeakRatio() * 100.0);
            ++violations;
        }
    });

    return (violations == 0) ? 0 : 1;      // ← CI が失敗する
}
```

**「予算を超えたビルドは、マージできない」というルールにできます。**

**これが、49.1 節で述べた「早く気づく」を実現する最も強い手段です。**

### 製品ビルドでは、落とさない

**プレイヤーの前でクラッシュさせるわけにはいきません。**

**第43章 43.8 節の緊急脱出路と同じ考え方です。**

```
警告 → ログに記録(クラッシュレポートに含める)
確保失敗 → 品質を落として続行(テクスチャの解像度を下げる、など)
```

---

## 49.5 レポートを出す

```cpp
// ga/BudgetReport.h
namespace ga
{
    inline void PrintBudgetReport(const BudgetTracker& budget, std::uint64_t frameNumber)
    {
        std::println("=== メモリ予算(フレーム {})===", frameNumber);
        std::println("");
        std::println("  {:<10} {:>10} {:>10} {:>10} {:>8}  {}",
                     "カテゴリ", "上限", "現在", "ピーク", "使用率", "状態");
        std::println("  {}", std::string(64, '-'));

        std::size_t totalLimit = 0, totalCurrent = 0, totalPeak = 0;
        int warnings = 0, overs = 0;

        budget.ForEachStatus([&](const BudgetStatus& s) {
            const char* mark = (s.state == BudgetState::Over)    ? "✗ 超過"
                             : (s.state == BudgetState::Warning) ? "⚠ 警告"
                                                                 : "OK";

            std::println("  {:<10} {:>10} {:>10} {:>10} {:>7.1f}%  {}",
                         ToString(s.tag),
                         FormatBytes(s.limit),
                         FormatBytes(s.current),
                         FormatBytes(s.peak),
                         s.Ratio() * 100.0,
                         mark);

            totalLimit   += s.limit;
            totalCurrent += s.current;
            totalPeak    += s.peak;

            if (s.state == BudgetState::Over)    { ++overs; }
            if (s.state == BudgetState::Warning) { ++warnings; }
        });

        std::println("  {}", std::string(64, '-'));
        std::println("  {:<10} {:>10} {:>10} {:>10} {:>7.1f}%",
                     "合計",
                     FormatBytes(totalLimit),
                     FormatBytes(totalCurrent),
                     FormatBytes(totalPeak),
                     100.0 * totalCurrent / static_cast<double>(totalLimit));

        if (overs > 0 || warnings > 0)
        {
            std::println("");
            std::println("  ⚠ {} 件の超過、{} 件の警告", overs, warnings);
        }
    }
}
```

```
=== メモリ予算(フレーム 18432)===

  カテゴリ         上限       現在     ピーク    使用率  状態
  ----------------------------------------------------------------
  Texture      512.00 MB  412.30 MB  468.10 MB   80.5%  OK
  Mesh         256.00 MB  148.20 MB  201.40 MB   57.9%  OK
  Audio         96.00 MB   88.40 MB   94.20 MB   92.1%  ⚠ 警告
  Script        32.00 MB   41.80 MB   41.80 MB  130.6%  ✗ 超過
  Physics       64.00 MB   22.10 MB   28.00 MB   34.5%  OK
  UI            32.00 MB   12.80 MB   14.20 MB   40.0%  OK
  General       48.00 MB   31.20 MB   35.80 MB   65.0%  OK
  ----------------------------------------------------------------
  合計        1040.00 MB  756.80 MB  883.50 MB   72.8%

  ⚠ 1 件の超過、1 件の警告
```

---

## 49.6 実行中に表示する

**レポートをログに出すだけでは、見られません。**

**画面に常に出しておくのが、最も効果的です。**

```cpp
void DrawBudgetWindow(const ga::BudgetTracker& budget)
{
    if (!ImGui::Begin("メモリ予算")) { ImGui::End(); return; }

    budget.ForEachStatus([](const ga::BudgetStatus& s) {
        const float ratio = static_cast<float>(std::min(s.Ratio(), 1.2));

        ImVec4 color;
        switch (s.state)
        {
        case ga::BudgetState::Over:    color = ImVec4(0.90f, 0.20f, 0.20f, 1.0f); break;
        case ga::BudgetState::Warning: color = ImVec4(0.90f, 0.70f, 0.20f, 1.0f); break;
        default:                       color = ImVec4(0.30f, 0.70f, 0.40f, 1.0f); break;
        }

        const std::string label = std::format("{} / {}  ({:.1f}%)",
                                              ga::FormatBytes(s.current),
                                              ga::FormatBytes(s.limit),
                                              s.Ratio() * 100.0);

        ImGui::Text("%s", ToString(s.tag));
        ImGui::SameLine(100.0f);

        ImGui::PushStyleColor(ImGuiCol_PlotHistogram, color);
        ImGui::ProgressBar(ratio, ImVec2(-1.0f, 0.0f), label.c_str());
        ImGui::PopStyleColor();

        // ピークを示す線を、別途描く
        if (s.PeakRatio() > s.Ratio())
        {
            ImGui::SameLine();
            ImGui::TextDisabled("(ピーク %.1f%%)", s.PeakRatio() * 100.0);
        }
    });

    ImGui::Separator();
    ImGui::Text("合計: %s / %s",
                ga::FormatBytes(budget.TotalCurrent()).c_str(),
                ga::FormatBytes(budget.TotalLimit()).c_str());

    ImGui::End();
}
```

### なぜ実行中の表示が効くのか

**「自分の作業がメモリに与える影響が、その場で見える」からです。**

```
アーティストが高解像度のテクスチャを追加
    → 画面のバーが伸びる
    → 「あ、増えた」と気づく
```

**レポートを見に行く必要がありません。**

**問題が起きてから調べるのではなく、作りながら気づける。** これが最大の価値です。

> **開発ビルドでは、常に表示しておくことを推奨します。** 邪魔なら小さく畳んでおき、色だけ見えるようにします。

---

## 49.7 ビルドごとに記録し、比較する

**「いつ増えたか」が分かると、原因の特定が劇的に楽になります。**

### 記録する

```cpp
void WriteBudgetCsv(const ga::BudgetTracker& budget, const char* path)
{
    std::FILE* fp = nullptr;
    if (fopen_s(&fp, path, "w") != 0) { return; }

    std::fprintf(fp, "tag,limit,current,peak\n");

    budget.ForEachStatus([fp](const ga::BudgetStatus& s) {
        std::fprintf(fp, "%s,%zu,%zu,%zu\n",
                     ToString(s.tag), s.limit, s.current, s.peak);
    });

    std::fclose(fp);
}
```

**自動テストの最後に呼び、結果を保存します。**

### 前回と比較する

```
=== 前回のビルドとの差分 ===
  Texture    +18.40 MB  (393.90 → 412.30)   ⚠
  Mesh        -2.10 MB  (150.30 → 148.20)
  Audio       +0.20 MB  ( 88.20 →  88.40)
  Script     +11.20 MB  ( 30.60 →  41.80)   ✗ 予算超過
  ────────────────────────────────────────
  合計       +27.70 MB
```

**「Script が 11 MB 増えて、予算を超えた」**

**この情報があれば、その日のコミットを調べるだけで原因にたどり着けます。**

### 自動化する

```
1. 自動テストで全ステージをプレイ
2. 予算レポートを CSV に出力
3. 前回の結果と比較
4. 一定以上の増加、または予算超過があれば、ビルドを失敗させる
```

**これが、メモリ管理の「回帰テスト」です。**

> **第4章で「測れないものは速くできない」と書きました。**
>
> **記録しないものは、管理できません。**

---

## 49.8 実際の運用

**技術より、運用のほうが難しい部分です。**

### 誰が決めるのか

**一人では決められません。**

```
テクニカルディレクター : 全体の枠を決める
各チームのリード       : 自分の枠の中で配分する
プログラマ             : 実測値を提供する
```

**予算は「技術的な事実」ではなく「合意」です。**

### いつ見直すのか

| タイミング | やること |
|---|---|
| プロジェクト開始時 | 概算で枠を決める |
| **プロトタイプ完成時** | **実測に基づいて見直す** |
| 各マイルストーン | 進捗と照らして調整 |
| 終盤 | **凍結する**(以降は交渉のみ) |

**最初の見積もりは、必ず外れます。** 早い段階で実測に置き換えることが重要です。

### 超えたときの選択肢

```
① 削る       : 品質を落とす(解像度、ポリゴン数、音質)
② 枠を移す   : 他のカテゴリから融通してもらう
③ 圧縮する   : 形式を変える(テクスチャ圧縮、音声のビットレート)
④ 遅延する   : 必要になるまで読み込まない(第47章のストリーミング)
⑤ 枠を広げる : 全体の予算を見直す(最後の手段)
```

**②が最も多く使われます。** そして、**②には交渉が必要です。**

**「テクスチャに 20 MB 欲しい。どこから持ってくるか」** ——この会話が成立するには、**全カテゴリの数字が見えている必要があります。**

**この章で作った表は、その会話のための道具です。**

---

## 49.9 v1.0 到達

**第2章の 20 行から始めて、49 章。ついに v1.0 です。**

### 到達点

| 機能 | 章 |
|---|---|
| バンプ確保、アラインメント、エラー処理 | 2, 6, 7 |
| 一括解放、マーカー、スコープ | 8, 9 |
| オブジェクト構築、破棄リスト、配列 | 10, 11, 12 |
| 確保元の記録、統計、タグ | 14, 15 |
| 塗りつぶし、ガードバイト、スタックトレース、可視化 | 16, 17, 18, 19 |
| プール、フリーリスト、合体、ビン、バディ、TLSF | 21〜27 |
| 仮想メモリ、成長する配列、ガードページ | 29, 30, 31 |
| ロック、スレッドキャッシュ、ロックフリー | 35, 36, 37 |
| 標準コンテナ、`std::pmr`、スマートポインタ、`operator new` | 38〜41 |
| フレーム、シーン、ハンドル、コンパクション | 43, 44, 45, 46 |
| ベイクデータ、GPU メモリ | 47, 48 |
| **メモリ予算** | **49** |

### バージョンの歩み

```
v0.1  (第2章)   20 行。ポインタを進めるだけ
v0.5  (第9章)   マーカーで巻き戻せる
v0.10 (第15章)  何がどれだけ使っているか見える
v0.20 (第27章)  最悪時間を保証する
v0.26 (第37章)  ロックフリー
v0.31 (第43章)  フレーム単位で回る
v0.35 (第47章)  ファイルをそのまま載せる
v1.0  (第49章)  予算の中で動いていることを保証する
```

**「20 行のバンプアロケーター」が、「予算管理された、観測可能な、マルチスレッド対応のメモリシステム」になりました。**

---

## 演習

**演習49-1** `LiveBytesForTag` を実装し、`Used()` の合計と一致することを確認してください。

**演習49-2** 予算の警告閾値を 70% / 85% / 95% と変えて、警告の頻度を比べてください。

**演習49-3** 状態が改善したとき(超過 → 警告)にも通知するようにしてください。有用ですか、うるさいですか。

**演習49-4** ImGui のウィンドウに、アリーナ別の内訳(軸①)も表示してください。

**演習49-5** 予算レポートを JSON で出力し、差分を計算するスクリプトを書いてください。

**演習49-6** 自動テストで予算超過を検出し、終了コードを返すようにしてください。

**演習49-7** ピークが記録された「瞬間」のスタックトレース(第18章)を保存してください。原因の特定に役立ちますか。

**演習49-8** 予算をシーンごとに変えられるようにしてください(戦闘シーンはエフェクトに多く割り当てる、など)。

---

## 章末チェックリスト

- [ ] 予算管理の目的が「早く気づくこと」であることを説明できる
- [ ] 2つの軸(アリーナ / タグ)の違いと、予算をタグで決める理由を説明できる
- [ ] **第19章のタグ境界が、予算集計に使える** 理由を説明できる
- [ ] `BudgetTracker` を実装し、横断的に集計した 〔v1.0〕
- [ ] 超過の4段階と、開発ビルド / 製品ビルドでの扱いの違いを説明できる
- [ ] 自動テストで予算超過を検出し、ビルドを失敗させる仕組みを作った
- [ ] ImGui で実行中に表示した
- [ ] ビルドごとの記録と差分を実装した
- [ ] 予算が「技術的事実」ではなく「合意」であることを理解した

---

## 次章の予告

**第8部が始まります。仕上げの4章です。**

**第50章は、テストとファジングです。**

ここまで、各章でテストを書いてきました。**しかし、断片的です。**

```cpp
void Test_CoalesceBoth() { /* 特定の状況を手で作る */ }
```

**手で書いたテストは、自分が思いついた状況しか検査できません。**

**ファジング** は、ランダムな操作列を大量に流し、**不変条件が破れないかを検査します。**

```
不変条件:
  ・確保した領域どうしは、絶対に重ならない
  ・すべてのブロックの合計が、容量を超えない
  ・フリーリストが輪になっていない
  ・全部解放したら、最初の状態に戻る
```

**人間が思いつかない操作列を、機械が見つけます。**

そして、失敗した操作列を **最小の再現手順に縮める** 方法を扱います。10 万回の操作で壊れたとき、そのうち本当に必要なのは 5 回かもしれません。

---

> **コラム:予算表は、技術ではなく合意である**
>
> **この章で作ったものは、100 行程度のコードです。技術的には簡単です。**
>
> **難しいのは、運用です。**
>
> ---
>
> **よくある失敗①:誰も見ない**
>
> レポートを出力するようにした。CSV も保存している。**しかし、誰も見ていない。**
>
> **原因は、見る動機がないことです。**
>
> 「メモリが足りなくなったら困る」と分かっていても、**目の前の作業のほうが優先されます。**
>
> **対策は、49.6 節の「常時表示」と、49.4 節の「ビルドを失敗させる」です。**
>
> **見に行かなくても目に入る。無視すると作業が進まない。** この2つが揃って、初めて機能します。
>
> ---
>
> **よくある失敗②:予算が現実離れしている**
>
> プロジェクト開始時に決めた枠が、実装が進むにつれて意味をなさなくなる。
>
> ```
> 「テクスチャ 512 MB」と決めたが、実際には 800 MB 必要だと判明
>     → 予算表を無視するようになる
>     → 予算管理そのものが形骸化する
> ```
>
> **予算は、定期的に見直さなければなりません。**
>
> **ただし、見直しは「合意の更新」です。** 誰かが勝手に数字を書き換えたら、意味がありません。
>
> ---
>
> **よくある失敗③:超過したときに何も起きない**
>
> **警告が出る。しかし、誰も対処しない。**
>
> ```
> 「まだ動いているから大丈夫」
> 「後で最適化する」
> 「他のチームも超えている」
> ```
>
> **超過が常態化すると、警告は無視されます。**
>
> **対策は、超過を「例外的な状態」に保つことです。** そのためには、
>
> - 予算が現実的であること(失敗②の対策)
> - **超過したら、その日のうちに対処する** というルール
> - 対処できないなら、**予算を見直して合意し直す**
>
> **「超えたまま放置する」という選択肢を、なくすことが重要です。**
>
> ---
>
> **誰が旗を振るのか**
>
> **多くの現場では、「メモリを見ている人」が1人います。**
>
> 定期的にレポートを確認し、増えていれば原因を調べ、担当者に伝える。予算の見直しを提案する。
>
> **技術的な役割というより、調整の役割です。**
>
> **この章で作った道具は、その人の仕事を楽にするためのものです。** 数字を集める作業が自動化されれば、**調整と判断に時間を使えます。**
>
> ---
>
> **最後に**
>
> **第15章のコラムで、こう書きました。**
>
> > メモリ予算表は、チーム間の契約として機能します。
>
> **契約は、守られて初めて意味を持ちます。**
>
> **そして契約が守られるためには、「守られているかどうかが見える」必要があります。**
>
> **それが、この章で作ったものです。**
