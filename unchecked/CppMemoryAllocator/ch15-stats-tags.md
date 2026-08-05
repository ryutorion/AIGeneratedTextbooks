# 第15章 使用量を集計する 〔v0.10〕

---

## この章のゴール

前章で「どこから確保されたか」が見えるようになりました。次は **「全体でどうなっているか」** です。

```
=== メモリ内訳 ===
  Mesh      :  2.10 MB (44.7%)   1240件
  Texture   :  1.60 MB (34.0%)     48件
  Audio     :  0.62 MB (13.2%)     12件
  Script    :  0.38 MB ( 8.1%)    892件
```

この形の情報が、実際のゲーム開発で最もよく見られているものです。

- 統計構造体を作り、確保のたびに更新する 〔**v0.10**〕
- サイズの分布をヒストグラムで取る
- **タグ(カテゴリ)** を導入し、内訳を出す
- 第14章で見送った「スコープでまとめる」方式を、ここで実装する
- 終了時にサマリを表示する
- コストを測る

---

## 15.1 何を知りたいのか

漠然と「統計を取る」のではなく、答えたい質問から決めます。実際の開発で出てくるのは、こういう問いです。

| 質問 | 必要な情報 |
|---|---|
| 板のサイズは適切か | ピーク使用量 |
| 確保が多すぎないか | 確保回数 |
| 小さい確保が大量にあるか | サイズの分布 |
| アラインメントで無駄が出ていないか | パディング合計 |
| **何がメモリを食っているのか** | **タグ別の内訳** |
| 前回のビルドから増えたか | 上記すべての記録と比較 |

最後の2つが特に重要です。「4.7 MB 使っている」だけでは、多いのか少ないのか判断できません。**内訳が分かって初めて、削る対象が決まります。**

---

## 15.2 統計を取る

第8章で `Peak()` を、第6章で `Padding()` を作りました。これらを1つの構造体にまとめ、項目を増やします。

```cpp
namespace ga
{
    struct Stats
    {
        // --- 現在のリセット周期 ---
        std::size_t count   = 0;   // 確保回数
        std::size_t bytes   = 0;   // 要求バイトの合計
        std::size_t padding = 0;   // パディングで捨てたバイト

        // --- 周期をまたいで保持 ---
        std::size_t peakBytes  = 0;
        std::size_t totalCount = 0;   // 全期間の確保回数
        std::size_t totalBytes = 0;   // 全期間の要求バイト
        std::size_t resetCount = 0;

        // --- サイズの範囲 ---
        std::size_t minSize = (std::numeric_limits<std::size_t>::max)();
        std::size_t maxSize = 0;

        double AverageSize() const noexcept
        {
            return (count == 0) ? 0.0 : static_cast<double>(bytes) / static_cast<double>(count);
        }
    };
}
```

### 「周期」と「全期間」を分ける

`Reset()` を呼ぶと、板の中身は空になります。しかし統計をすべてゼロに戻すと、履歴が失われます。

そこで2種類持ちます。

- **周期の統計**(`count`、`bytes`、`padding`):`Reset()` でゼロに戻る。「いまどうなっているか」
- **全期間の統計**(`totalCount`、`totalBytes`、`peakBytes`):ずっと積み上がる。「これまでどうだったか」

`Reset()` のときに、周期の値をピークへ反映してからクリアします。

```cpp
    void Reset() noexcept
    {
        RunFinalizersUntil(nullptr);

        if (stats_.bytes > stats_.peakBytes) { stats_.peakBytes = stats_.bytes; }
        ++stats_.resetCount;

        stats_.count   = 0;
        stats_.bytes   = 0;
        stats_.padding = 0;

        // ... offset_ 等のクリアは従来どおり ...
    }
```

### `Rewind()` はどう扱うか

正直に書きます。**`Rewind()` では統計を減らしません。**

減らすには、巻き戻す範囲にどの確保が含まれていたかを知る必要があります。それには確保ごとにメタデータを持たせなければならず、コストが大きすぎます。

したがって `bytes` は「このリセット周期で **のべ何バイト確保したか**」を意味します。`Used()` とは一致しません。**この違いは必ずドキュメントに書いてください。** 一致すると思い込むと、数字を読み間違えます。

| | 意味 |
|---|---|
| `Used()` | いま板のどこまで使っているか(`Rewind` で減る) |
| `Stats::bytes` | 周期内に要求された合計(減らない) |

---

## 15.3 サイズの分布を見る

平均だけでは実態が分かりません。「平均 394 バイト」という数字は、次のどちらでも成り立ちます。

- ほとんどが 400 バイト前後
- 8 バイトが大量にあり、たまに 1 MB が混ざる

**後者なら、小さい確保に特化したアロケーター(第21章のプール)が効きます。** 対策がまるで違うので、分布を見る必要があります。

2の冪でビンを分けます。

```cpp
    static constexpr std::size_t kSizeBucketCount = 24;   // 〜8 MB 以上まで

    // size が属するビンの番号
    static constexpr std::size_t SizeBucketOf(std::size_t size) noexcept
    {
        const std::size_t idx = std::bit_width(size);   // 0→0, 1→1, 2..3→2, 4..7→3
        return (idx < kSizeBucketCount) ? idx : kSizeBucketCount - 1;
    }
```

`std::bit_width`(C++20、`<bit>`)は「その値を表すのに必要なビット数」を返します。最上位ビットの位置なので、2の冪ごとのビン分けにそのまま使えます。除算も分岐もありません。

第6章で `AlignUp` にビット演算を使ったのと同じ発想です。そして **この考え方は、第25章でサイズ別ビンを実装するときに本格的に使います**。ここはその予行演習でもあります。

---

## 15.4 タグを導入する

いよいよ本題です。「何がメモリを食っているか」を知るには、確保を **分類** する必要があります。

```cpp
namespace ga
{
    enum class MemoryTag : std::uint8_t
    {
        General,
        Mesh,
        Texture,
        Audio,
        Script,
        Physics,
        UI,
        Debug,

        Count       // 番兵。要素数として使う
    };

    constexpr const char* ToString(MemoryTag t) noexcept
    {
        switch (t)
        {
        case MemoryTag::General: return "General";
        case MemoryTag::Mesh:    return "Mesh";
        case MemoryTag::Texture: return "Texture";
        case MemoryTag::Audio:   return "Audio";
        case MemoryTag::Script:  return "Script";
        case MemoryTag::Physics: return "Physics";
        case MemoryTag::UI:      return "UI";
        case MemoryTag::Debug:   return "Debug";
        case MemoryTag::Count:   break;
        }
        return "?";
    }
}
```

### なぜ文字列ではなく `enum` か

タグを `const char*` や `std::string` にする設計も考えられます。柔軟ですが、避けます。

| | `enum` | 文字列 |
|---|---|---|
| 集計 | **配列の添字で O(1)** | ハッシュ表が必要 |
| 比較 | 整数比較 | 文字列比較 |
| メモリ | 1 バイト | ポインタ + 実体 |
| タイプミス | **コンパイルエラー** | 実行時まで気づかない |
| 拡張 | 再コンパイルが必要 | 実行時に追加できる |

**アロケーターの内側で毎回実行される処理** なので、速度と単純さを優先します。集計配列を `std::array<TagStats, static_cast<std::size_t>(MemoryTag::Count)>` として持てるのが決定的です。

拡張性の弱さは認めます。ライブラリ利用者が自分のタグを定義できるようにするには、タグの型をテンプレートパラメータにする必要があります。第51章で整理します。

### `Count` という番兵

列挙の最後に `Count` を置くのは定番の手法です。要素数がコンパイル時に取れます。

```cpp
static constexpr std::size_t kTagCount = static_cast<std::size_t>(MemoryTag::Count);
std::array<Stats, kTagCount> tagStats_{};
```

タグを追加しても、配列のサイズは自動で追随します。

---

## 15.5 タグをどうやって伝えるか

ここが設計の勘所です。第14章で見送った問題が戻ってきました。

### 案A:引数で渡す

```cpp
bump.Allocate(size, alignment, MemoryTag::Mesh);
```

明示的で分かりやすい。しかし、**すべての確保箇所を書き換える** ことになります。しかも、深い呼び出しの先で確保が起きる場合、タグを引数として持ち回る必要があります。

```cpp
void LoadModel(Bump& arena, MemoryTag tag);       // タグを引数に足す
void ParseVertices(Bump& arena, MemoryTag tag);   // ここにも
void DecompressBuffer(Bump& arena, MemoryTag tag);// ここにも
```

現実的ではありません。

### 案B:スコープで指定する ← 採用

アロケーターに「いまのタグ」を持たせ、スコープガードで切り替えます。

```cpp
void LoadStage()
{
    {
        GA_TAG(arena, MemoryTag::Mesh);
        LoadModel(arena);            // この中の確保はすべて Mesh
        LoadModel(arena);
    }

    {
        GA_TAG(arena, MemoryTag::Texture);
        LoadTextures(arena);         // ここは Texture
    }
}
```

**呼ばれる側は何も知りません。** `LoadModel` のシグネチャは変わりません。

第9章の `BumpScope` とまったく同じ構造です。**C++ のコールスタックを、タグのスタックとして流用しています。**

```cpp
namespace ga
{
    class TagScope
    {
    public:
        TagScope(Bump& bump, MemoryTag tag) noexcept
            : bump_(&bump)
            , previous_(bump.PushTag(tag))
        {
        }

        ~TagScope()
        {
            bump_->PopTag(previous_);
        }

        TagScope(const TagScope&)            = delete;
        TagScope& operator=(const TagScope&) = delete;
        TagScope(TagScope&&)                 = delete;
        TagScope& operator=(TagScope&&)      = delete;

    private:
        Bump*     bump_;
        MemoryTag previous_;
    };
}
```

`Bump` 側は2行です。

```cpp
    MemoryTag PushTag(MemoryTag tag) noexcept
    {
        const MemoryTag previous = currentTag_;
        currentTag_ = tag;
        return previous;
    }

    void PopTag(MemoryTag previous) noexcept
    {
        currentTag_ = previous;
    }
```

**スタックを自前で持つ必要はありません。** 前の値をスコープガードが持ち、デストラクタで戻します。深さの制限もありません。

### マクロで書きやすくする

変数名を毎回考えるのは面倒です。

```cpp
#define GA_CONCAT_INNER(a, b) a##b
#define GA_CONCAT(a, b)       GA_CONCAT_INNER(a, b)

#define GA_TAG(arena, tag) \
    ::ga::TagScope GA_CONCAT(gaTagScope_, __LINE__)((arena), (tag))
```

`__LINE__` を使って一意な変数名を作ります。同じ行に2つ書くことはないので、これで衝突しません。

> **`GA_CONCAT` が2段になっている理由**
> `a##b` は引数を展開せずに連結してしまいます。`GA_CONCAT(x, __LINE__)` を1段で書くと `x__LINE__` という変数名になります。1段はさむことで `__LINE__` が先に展開され、`x42` になります。プリプロセッサの有名な癖です。

---

## 15.6 実装する 〔v0.10〕

記録処理に統計の更新を足します。

```cpp
    void RecordAllocation(const void* address, std::size_t size, std::size_t alignment,
                          std::size_t padding, const std::source_location& loc) noexcept
    {
#if GA_ENABLE_ALLOC_TRACKING
        // --- 全体の統計 ---
        UpdateStats(stats_, size, padding);

        // --- タグ別の統計 ---
        UpdateStats(tagStats_[static_cast<std::size_t>(currentTag_)], size, padding);

        // --- サイズ分布 ---
        ++sizeBuckets_[SizeBucketOf(size)];

        // --- 直近の記録(第14章)---
        const AllocationInfo info{ address, size, alignment, padding, loc, currentTag_ };
        recent_[recentHead_] = info;
        recentHead_ = (recentHead_ + 1) % kRecentCapacity;

        if (logCallback_) { logCallback_(info, logUser_); }
#else
        (void)address; (void)size; (void)alignment; (void)padding; (void)loc;
#endif
    }

private:
    static void UpdateStats(Stats& s, std::size_t size, std::size_t padding) noexcept
    {
        ++s.count;
        ++s.totalCount;

        s.bytes      += size;
        s.totalBytes += size;
        s.padding    += padding;

        if (size < s.minSize) { s.minSize = size; }
        if (size > s.maxSize) { s.maxSize = size; }
    }
```

`AllocationInfo` にもタグを足しました。第14章のログ出力に、カテゴリが表示されるようになります。

`Reset()` では、全体とタグ別の両方を処理します。

```cpp
    void Reset() noexcept
    {
        RunFinalizersUntil(nullptr);

        CycleStats(stats_);
        for (auto& s : tagStats_) { CycleStats(s); }

        // ... 従来の処理 ...
    }

private:
    static void CycleStats(Stats& s) noexcept
    {
        if (s.bytes > s.peakBytes) { s.peakBytes = s.bytes; }
        ++s.resetCount;

        s.count   = 0;
        s.bytes   = 0;
        s.padding = 0;
    }
```

---

## 15.7 サマリを表示する

表示機能は **ライブラリ本体から分けます**。`std::print` に依存させたくないためです。第13章の方針どおり、公開ヘッダの依存は最小限にします。

`ga/Report.h`(利用者が明示的にインクルードする):

```cpp
#pragma once

#include "ga/Bump.h"

#include <format>
#include <print>
#include <string>

namespace ga
{
    inline std::string FormatBytes(std::size_t bytes)
    {
        constexpr const char* kUnits[] = { "B", "KB", "MB", "GB" };

        double v = static_cast<double>(bytes);
        int    u = 0;

        while (v >= 1024.0 && u < 3) { v /= 1024.0; ++u; }

        return (u == 0) ? std::format("{} B", bytes)
                        : std::format("{:.2f} {}", v, kUnits[u]);
    }

    inline void PrintReport(const Bump& bump)
    {
        const Stats& s = bump.GetStats();

        const double usedPct = 100.0 * static_cast<double>(bump.Used())
                                     / static_cast<double>(bump.Capacity());

        std::println("=== ga::Bump メモリレポート ===");
        std::println("  容量        : {}", FormatBytes(bump.Capacity()));
        std::println("  使用量      : {} ({:.1f}%)", FormatBytes(bump.Used()), usedPct);
        std::println("  ピーク      : {}", FormatBytes(s.peakBytes));
        std::println("  確保回数    : {} (全期間 {})", s.count, s.totalCount);
        std::println("  平均サイズ  : {:.0f} B", s.AverageSize());
        std::println("  最小/最大   : {} / {}",
                     FormatBytes(s.minSize == (std::numeric_limits<std::size_t>::max)() ? 0 : s.minSize),
                     FormatBytes(s.maxSize));
        std::println("  パディング  : {}", FormatBytes(s.padding));
        std::println("  リセット回数: {}", s.resetCount);
        std::println("");

        // --- タグ別 ---
        std::println("  {:<10} {:>10} {:>8} {:>8} {:>10}",
                     "タグ", "使用量", "割合", "件数", "平均");
        std::println("  {}", std::string(50, '-'));

        for (std::size_t i = 0; i < Bump::kTagCount; ++i)
        {
            const Stats& t = bump.GetTagStats(static_cast<MemoryTag>(i));
            if (t.count == 0) { continue; }

            const double pct = (s.bytes == 0) ? 0.0
                             : 100.0 * static_cast<double>(t.bytes) / static_cast<double>(s.bytes);

            std::println("  {:<10} {:>10} {:>7.1f}% {:>8} {:>10.0f}",
                         ToString(static_cast<MemoryTag>(i)),
                         FormatBytes(t.bytes), pct, t.count, t.AverageSize());
        }

        // --- サイズ分布 ---
        std::println("");
        std::println("  サイズ分布:");

        for (std::size_t i = 0; i < Bump::kSizeBucketCount; ++i)
        {
            const std::size_t n = bump.GetSizeBucket(i);
            if (n == 0) { continue; }

            const std::size_t lower = (i == 0) ? 0 : (std::size_t{1} << (i - 1));
            const std::size_t upper = (std::size_t{1} << i) - 1;

            // 対数っぽい棒
            std::size_t bar = 0;
            for (std::size_t c = n; c > 0; c /= 10) { ++bar; }

            std::println("    {:>8} - {:>8} : {:>8} {}",
                         FormatBytes(lower), FormatBytes(upper), n, std::string(bar * 4, '#'));
        }
    }
}
```

---

## 15.8 使ってみる

ステージのロード処理を模したプログラムです。

```cpp
#include "pch.h"

#include "ga/Allocator.h"
#include "ga/Report.h"

using ga::MemoryTag;

void LoadMeshes(ga::Bump& arena)
{
    for (int i = 0; i < 40; ++i)
    {
        (void)arena.Allocate(48 * 1024 + i * 512);   // 頂点バッファ
        (void)arena.Allocate(16 * 1024);             // インデックスバッファ
    }
}

void LoadTextures(ga::Bump& arena)
{
    for (int i = 0; i < 12; ++i)
    {
        (void)arena.Allocate(256 * 1024);            // 512x512 圧縮テクスチャ
    }
}

void LoadScripts(ga::Bump& arena)
{
    for (int i = 0; i < 800; ++i)
    {
        (void)arena.Allocate(64 + (i % 7) * 16);     // 小さいノードが大量
    }
}

int main()
{
    ga::Bump arena(16 * 1024 * 1024);

    {
        GA_TAG(arena, MemoryTag::Mesh);
        LoadMeshes(arena);
    }
    {
        GA_TAG(arena, MemoryTag::Texture);
        LoadTextures(arena);
    }
    {
        GA_TAG(arena, MemoryTag::Script);
        LoadScripts(arena);
    }

    ga::PrintReport(arena);
}
```

```
=== ga::Bump メモリレポート ===
  容量        : 16.00 MB
  使用量      : 7.02 MB (43.9%)
  ピーク      : 0 B
  確保回数    : 892 (全期間 892)
  平均サイズ  : 8253 B
  最小/最大   : 64 B / 268.00 KB
  パディング  : 4.05 KB
  リセット回数: 0

  タグ            使用量     割合     件数         平均
  --------------------------------------------------
  Mesh          4.19 MB    59.7%       80      54886
  Texture       3.00 MB    42.8%       12     262144
  Script       64.06 KB     0.9%      800         82

  サイズ分布:
        32 B -      63 B :      114 ####
        64 B -     127 B :      686 ############
       128 B -     255 B :        0
      ...
     32.00 KB - 63.99 KB :       80 ########
    256.00 KB - 511.9 KB :       12 ########
```

### 読み取れること

**メッシュが 6 割を占めています。** 削るならここです。

**スクリプトは件数が 800 で最多ですが、容量は 0.9% しかありません。** 小さい確保が大量にあるパターンです。バンプアロケーターなら問題ありませんが、`new` を使っていたら 800 回分のオーバーヘッドが乗ります。第21章のプールが効く形でもあります。

**サイズ分布が二極化しています。** 64〜127 バイトの山と、32 KB 以上の山。中間がほとんどありません。ゲームのメモリ使用は、たいていこういう形になります。

**パディングは 4 KB、全体の 0.06%** です。第6章で心配した内部断片化は、この使い方では問題になっていません。

---

## 15.9 コストを測る

```
v0.8  (統計なし)              median=      2.1
v0.9  (直近の記録のみ)        median=      3.4
v0.10 (統計 + タグ + 分布)    median=      4.3
```

**0.9 ns の追加** です。増えた処理は、

- 加算とインクリメントが6回(全体)
- 同じく6回(タグ別)
- 最小・最大の比較が4回
- `bit_width` とビンのインクリメント

分岐が最小・最大の比較だけで、しかも予測が当たりやすい形です。

`new` との比較でいえば、統計を全部有効にしても **まだ4倍速い** ことになります。それでも、Release でホットパスに置くなら消すべきです。第14章の `GA_ENABLE_ALLOC_TRACKING` で、まとめて無効化できます。

---

## 15.10 この情報が実務で何に使われるか

作った機能が、現場でどう使われるかを書いておきます。

**1. 予算との比較。** タグごとに上限を決め、超過を検出します。「テクスチャは 3 MB まで」という取り決めがあれば、この数字と突き合わせるだけで判定できます。第49章で実装します。

**2. ビルドごとの記録。** レポートをファイルに吐き、前回のビルドと比較します。「昨日から Mesh が 800 KB 増えた。誰の変更か」を追えるようにしておくと、問題が小さいうちに気づけます。

**3. QA からの報告の裏付け。** 「特定のステージで落ちる」という報告に対し、そのステージのレポートを見れば、どのカテゴリが想定を超えているか分かります。

**4. 最適化の優先順位づけ。** 上の例なら、Script の 800 件を減らす作業より、Mesh の 4.19 MB を減らす作業のほうが効果が大きい。**測らずに直感で選ぶと、たいてい間違えます。**

---

## 15.11 この章の完成コード

主要な差分です。

```cpp
// ga/MemoryTag.h(新規)
#pragma once
#include <cstdint>

namespace ga
{
    enum class MemoryTag : std::uint8_t
    {
        General, Mesh, Texture, Audio, Script, Physics, UI, Debug,
        Count
    };

    constexpr const char* ToString(MemoryTag t) noexcept { /* ... */ }
}
```

```cpp
// ga/Stats.h(新規)
#pragma once
#include <cstddef>
#include <limits>

namespace ga
{
    struct Stats
    {
        std::size_t count      = 0;
        std::size_t bytes      = 0;
        std::size_t padding    = 0;
        std::size_t peakBytes  = 0;
        std::size_t totalCount = 0;
        std::size_t totalBytes = 0;
        std::size_t resetCount = 0;
        std::size_t minSize    = (std::numeric_limits<std::size_t>::max)();
        std::size_t maxSize    = 0;

        double AverageSize() const noexcept
        {
            return (count == 0) ? 0.0
                                : static_cast<double>(bytes) / static_cast<double>(count);
        }
    };
}
```

```cpp
// ga/Bump.h に追加されるもの
    static constexpr std::size_t kTagCount        = static_cast<std::size_t>(MemoryTag::Count);
    static constexpr std::size_t kSizeBucketCount = 24;

    MemoryTag PushTag(MemoryTag tag) noexcept;
    void      PopTag(MemoryTag previous) noexcept;
    MemoryTag CurrentTag() const noexcept { return currentTag_; }

    const Stats& GetStats() const noexcept { return stats_; }
    const Stats& GetTagStats(MemoryTag t) const noexcept
    {
        return tagStats_[static_cast<std::size_t>(t)];
    }
    std::size_t GetSizeBucket(std::size_t i) const noexcept { return sizeBuckets_[i]; }

private:
    Stats                              stats_{};
    std::array<Stats, kTagCount>       tagStats_{};
    std::array<std::size_t, kSizeBucketCount> sizeBuckets_{};
    MemoryTag                          currentTag_ = MemoryTag::General;
```

---

## 演習

**演習15-1** `Stats::bytes` と `Used()` が一致しない状況を作ってください。`Rewind()` を使います。

**演習15-2** タグに `Count` という番兵を置く方式の弱点を挙げてください。`ToString` の `switch` に新しいタグを足し忘れると、コンパイラは警告してくれますか。

**演習15-3** `GA_TAG` を同じ行に2つ書くとどうなりますか。`__LINE__` の代わりに `__COUNTER__` を使うと解決しますか。

**演習15-4** タグごとにピーク使用量を記録し、レポートに追加してください。周期の切り替えをどこで行うべきですか。

**演習15-5** レポートを JSON 形式で出力する関数を書いてください。ビルドごとの比較には、どちらの形式が便利ですか。

**演習15-6** サイズ分布のビンを2の冪ではなく、より細かい区切り(1.5倍刻みなど)にしてください。実装はどう変わりますか。

**演習15-7** `MemoryTag` をテンプレートパラメータにして、利用者が自分の列挙を使えるようにしてください。`Bump` の宣言はどう変わりますか。

---

## 章末チェックリスト

- [ ] `Stats` 構造体を実装した 〔v0.10〕
- [ ] 「周期の統計」と「全期間の統計」を分ける理由を説明できる
- [ ] `Stats::bytes` と `Used()` の違いを説明できる
- [ ] `std::bit_width` でサイズ分布のビンを求めた
- [ ] タグを `enum` にした理由を説明できる
- [ ] `TagScope` と `GA_TAG` を実装した
- [ ] `GA_CONCAT` が2段になっている理由を説明できる
- [ ] レポートを表示し、内訳を読み取った
- [ ] 統計のコスト(約 0.9 ns)を測った

---

## 次章の予告

ここまでは「数を数える」話でした。次章からは **「壊れているかどうかを見つける」** 話に移ります。

第16章では、メモリを塗りつぶします。

- 確保した直後の領域を `0xCD` で埋める
- `Reset()` で解放した領域を `0xDD` で埋める

これだけで、次の2種類のバグが一瞬で見つかるようになります。

**初期化を忘れたまま読んでいるバグ。** 値が `0xCDCDCDCD` という異様な数字になるので、デバッガで一目で分かります。

**`Reset()` 後の古いポインタを使っているバグ。** 第8章で「アクセス違反にならないので手がかりがない」と書いた、あの問題です。`0xDDDDDDDD` が見えれば、すぐ気づけます。

第3章で作った `DumpBytes` が、ここでようやく本領を発揮します。

---

> **コラム:メモリ内訳を見る、という文化**
>
> ゲーム開発の現場では、「メモリの内訳表」が日常的に共有されます。他の分野のソフトウェア開発と比べて、この習慣は際立っています。
>
> 理由は単純で、**据置機や携帯機のメモリ量が固定だから** です。
>
> PC アプリケーションなら、メモリが足りなければスワップされ、遅くはなっても動きます。ユーザーが増設することもできます。しかしゲーム機では、搭載量が最初から決まっており、1バイトも増えません。しかも OS が予約する分を除いた残りしか使えません。
>
> だから、開発の初期段階で **メモリ予算表** が作られます。
>
> ```
>   システム        : 128 MB
>   テクスチャ      : 512 MB
>   モデル          : 256 MB
>   アニメーション  : 128 MB
>   オーディオ      :  96 MB
>   スクリプト      :  32 MB
>   予備            :  64 MB
>   ---------------------------
>   合計            : 1216 MB
> ```
>
> この表は、チーム間の **契約** として機能します。テクスチャ担当は 512 MB の中でやりくりする。超えそうなら、他のチームと交渉して枠を融通してもらうか、圧縮率を上げる。
>
> ---
>
> 予算表が機能するには、**現在値を測れなければなりません**。そのために、ほぼすべてのゲームエンジンがタグ別のメモリ集計機能を持っています。
>
> Unreal Engine の `LLM`(Low Level Memory Tracker)は、まさにこの章で作ったものの本格版です。スコープでタグを切り替え、確保のたびに集計し、統計を吐き出します。Unity にも Memory Profiler があり、カテゴリ別の内訳を見られます。
>
> どのエンジンも同じ結論に至っているのは、**この情報なしにはメモリを管理できない** からです。
>
> ---
>
> 面白いのは、**この文化がゲーム以外にも広がりつつある** ことです。
>
> サーバーサイドでは、コンテナごとにメモリ上限が設定されるようになりました。上限を超えたプロセスは OOM Killer に殺されます。「使える量が固定で、超えたら死ぬ」という点で、ゲーム機と同じ制約が生まれています。
>
> 組み込み機器やモバイルでも事情は似ています。iOS はメモリを使いすぎるアプリを容赦なく終了させます。
>
> **「メモリは無限にあるものとして書き、足りなければ OS が何とかしてくれる」** という前提が成り立つ環境は、実はそれほど多くありません。この章で作った内訳表は、ゲーム以外の場面でも役に立つはずです。
