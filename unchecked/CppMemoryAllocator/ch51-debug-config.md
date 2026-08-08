# 第51章 デバッグ機能を切り離す

---

## この章のゴール

**この本を通して、たくさんの検査機能を作ってきました。**

```
統計、タグ、確保元の記録、スタックトレース、
塗りつぶし、ガードバイト、ガードページ、
スレッドの所有者検査、不変条件検査…
```

**すべてを有効にすると、`Allocate` は 2.1 ns から 28.4 ns になります。13 倍です。**

この章では、これを整理します。

- 積み上がった機能と、そのコストを一覧にする
- **`#if` マクロの限界** ——第9章と第14章で警告した ODR 違反
- **設定を型にする** ——テンプレートパラメータによる構成
- 空のメンバを、本当にゼロバイトにする
- **3つの構成**(Debug / 開発 / Release)を設計する
- そして、**「開発ビルド」こそが最も長く使われる** という主張

---

## 51.1 積み上がったものを、一覧にする

| 章 | 機能 | `Allocate` への追加 | メモリへの影響 |
|---|---|---|---|
| — | 基本(v1.0 の骨格) | **2.1 ns** | — |
| 14 | 確保元の記録(`source_location`) | +1.3 ns | 640 B + 実行ファイル +7 KB |
| 15 | 統計・タグ・サイズ分布 | +0.9 ns | 約 400 B |
| 16 | 塗りつぶし(`0xCD` / `0xDD`) | +2.6 ns | 0(`Reset` が O(n) に) |
| 17 | ガードバイト | +19.5 ns | **+96 B / 確保** |
| 18 | スタックトレース(全件) | +1893 ns | 130 B / 件 |
| 19 | タグ境界の記録 | +0.2 ns | 境界数 × 16 B |
| 34 | スレッド所有者の検査 | +0.3 ns | 4 B |
| 50 | 不変条件検査(`Reset` 時) | — | `Reset` が O(n) に |

### 全部有効にすると

```
                     Allocate    Reset(1 MB 使用時)
基本                    2.1 ns          0 ns
全部有効               28.4 ns        412 µs
```

**13 倍遅く、`Reset` は O(1) から O(n) へ。**

**第2章から積み上げてきた性能が、跡形もありません。**

### しかし、必要な機能である

**第16章で塗りつぶしを作ったとき、こう書きました。**

> 「たまたま動いていた」コードが壊れる。それは退化ではなく、発見です。

**第17章のガードバイトは、はみ出しを検出しました。第18章のスタックトレースは、確保元を特定しました。**

**どれも、開発中には必要です。**

> **問題は「必要かどうか」ではなく、「いつ有効にするか」です。**

---

## 51.2 `#if` マクロの限界

**これまで、こう書いてきました。**

```cpp
#if GA_ENABLE_ALLOC_TRACKING
    ++allocCount_;
    recent_[recentHead_] = info;
#endif
```

**3つの問題があります。**

### 問題1:ODR 違反の危険

**第9章と第14章で、繰り返し警告しました。**

> **メンバは常に持ち、コードだけを消します。** 構成によってクラスのサイズが変わると、ODR 違反の温床になります。

**具体的に、何が起きるか。**

```cpp
// ライブラリ側を Debug でビルド
class Bump
{
    std::array<AllocationInfo, 16> recent_;   // ← 640 バイト
    std::size_t offset_;
};   // sizeof = 1096

// アプリケーション側を Release でビルド
class Bump
{
    std::size_t offset_;                       // ← recent_ がない
};   // sizeof = 72
```

**同じ名前のクラスが、2つの異なる定義を持ちます。**

**リンカは気づきません。** 名前が同じなので、そのまま繋がります。

**結果:** アプリケーション側が `sizeof(Bump)` を 72 だと思って確保し、ライブラリ側が 1096 バイト目に書き込む。**メモリ破壊です。**

**しかも、症状は不定です。**

### だから、妥協していた

**「メンバは残し、コードだけ消す」** という方針を取ってきました。

**安全ですが、Release でも無駄なメモリを持ちます。**

```
sizeof(ga::Bump)(Release、全機能無効)  : 1096 バイト
実際に必要なもの                        :   72 バイト
```

**1024 バイトが無駄です。** アリーナが 100 個あれば 100 KB。

### 問題2:粒度が粗い

```cpp
#define GA_ENABLE_ALLOC_TRACKING 1
```

**「統計だけ欲しいが、スタックトレースは要らない」** といった組み合わせが作りにくい。

**マクロを増やせば対応できますが、組み合わせが爆発します。**

### 問題3:設定がグローバル

**プロジェクト全体で1つの設定になります。**

```cpp
// フレームアリーナ:速度が critical → 検査を最小限に
// シーンアリーナ  :ロード時のみ → 検査を厚く
```

**同じプロセス内で、アリーナごとに設定を変えられません。**

---

## 51.3 設定を型にする

**解決策は、テンプレートパラメータです。**

```cpp
// ga/Config.h
#pragma once

namespace ga
{
    // すべての検査を有効にする(開発中の詳細な調査用)
    struct DebugConfig
    {
        static constexpr bool kTrackStats      = true;   // 第15章
        static constexpr bool kTrackTagSpans   = true;   // 第19章
        static constexpr bool kTrackSource     = true;   // 第14章
        static constexpr bool kStackTrace      = true;   // 第18章
        static constexpr bool kFillPattern     = true;   // 第16章
        static constexpr bool kGuardBytes      = true;   // 第17章
        static constexpr bool kThreadCheck     = true;   // 第34章
        static constexpr bool kValidateOnReset = true;   // 第50章
        static constexpr bool kOwnershipCheck  = true;   // 第22章
    };

    // 実機に近い速度で、必要最小限の検査を残す
    struct DevelopmentConfig
    {
        static constexpr bool kTrackStats      = true;   // 予算管理(第49章)に必要
        static constexpr bool kTrackTagSpans   = true;   // 同上
        static constexpr bool kTrackSource     = false;
        static constexpr bool kStackTrace      = false;
        static constexpr bool kFillPattern     = false;
        static constexpr bool kGuardBytes      = false;
        static constexpr bool kThreadCheck     = true;   // 0.3 ns。安い
        static constexpr bool kValidateOnReset = false;
        static constexpr bool kOwnershipCheck  = true;
    };

    // 出荷用
    struct ReleaseConfig
    {
        static constexpr bool kTrackStats      = false;
        static constexpr bool kTrackTagSpans   = false;
        static constexpr bool kTrackSource     = false;
        static constexpr bool kStackTrace      = false;
        static constexpr bool kFillPattern     = false;
        static constexpr bool kGuardBytes      = false;
        static constexpr bool kThreadCheck     = false;
        static constexpr bool kValidateOnReset = false;
        static constexpr bool kOwnershipCheck  = true;   // ★ 残す(51.9 節)
    };
}
```

### 型が別物になる

```cpp
template <class Config = DefaultConfig>
class BasicBump { /* ... */ };

using Bump            = BasicBump<DefaultConfig>;
using DebugBump       = BasicBump<DebugConfig>;
using DevelopmentBump = BasicBump<DevelopmentConfig>;
using ReleaseBump     = BasicBump<ReleaseConfig>;
```

**`BasicBump<DebugConfig>` と `BasicBump<ReleaseConfig>` は、まったく別の型です。**

**名前マングリングが違うので、混ぜたらリンクエラーになります。**

```
error LNK2019: 未解決の外部シンボル
  "void __cdecl Process(class ga::BasicBump<struct ga::ReleaseConfig> &)"
```

> **ODR 違反が、リンクエラーに変わりました。** 静かに壊れるのではなく、ビルドが止まります。
>
> **第9章から警告してきた問題への、根本的な答えです。**

---

## 51.4 メンバを本当に消す

**`if constexpr` でコードは消せますが、メンバは残ります。**

```cpp
class BasicBump
{
    std::array<AllocationInfo, 16> recent_;   // ← Config に関係なく存在する
};
```

### 解決:機能をクラスに切り出す

**「機能を持つ版」と「何もしない版」を用意し、`std::conditional_t` で選びます。**

```cpp
// ga/detail/StatsHolder.h
namespace ga::detail
{
    // 統計を持つ版
    class StatsHolder
    {
    public:
        void RecordAllocation(std::size_t size, std::size_t padding, MemoryTag tag) noexcept
        {
            ++stats_.count;
            stats_.bytes   += size;
            stats_.padding += padding;

            auto& t = tagStats_[static_cast<std::size_t>(tag)];
            ++t.count;
            t.bytes += size;

            ++sizeBuckets_[SizeBucketOf(size)];
        }

        void CycleAll() noexcept { /* Reset 時の処理 */ }

        const Stats& GetStats() const noexcept { return stats_; }
        const Stats& GetTagStats(MemoryTag t) const noexcept
        {
            return tagStats_[static_cast<std::size_t>(t)];
        }

    private:
        Stats                                     stats_{};
        std::array<Stats, kTagCount>              tagStats_{};
        std::array<std::size_t, kSizeBucketCount> sizeBuckets_{};
    };

    // 何もしない版(空のクラス)
    class NoStats
    {
    public:
        void RecordAllocation(std::size_t, std::size_t, MemoryTag) noexcept {}
        void CycleAll() noexcept {}

        const Stats& GetStats() const noexcept { return kEmpty; }
        const Stats& GetTagStats(MemoryTag)  const noexcept { return kEmpty; }

    private:
        static inline const Stats kEmpty{};
    };

    template <class Config>
    using StatsFor = std::conditional_t<Config::kTrackStats, StatsHolder, NoStats>;
}
```

**`NoStats` は、メンバを持ちません。空のクラスです。**

### `[[msvc::no_unique_address]]` で 0 バイトにする

**第40章で学んだ属性です。**

```cpp
template <class Config = DefaultConfig>
class BasicBump
{
    // ...

private:
    VirtualMemory memory_;
    std::byte*    base_   = nullptr;
    std::size_t   offset_ = 0;

    GA_NO_UNIQUE_ADDRESS detail::StatsFor<Config>      stats_;
    GA_NO_UNIQUE_ADDRESS detail::TagSpansFor<Config>   tagSpans_;
    GA_NO_UNIQUE_ADDRESS detail::RecentFor<Config>     recent_;
    GA_NO_UNIQUE_ADDRESS detail::GuardFor<Config>      guard_;
    GA_NO_UNIQUE_ADDRESS detail::ThreadGuardFor<Config> threadGuard_;
};
```

**空のクラスなら、本当に 0 バイトになります。**

**第40章で確認したとおり、MSVC では `[[msvc::no_unique_address]]` が必要です。**

```cpp
#if defined(_MSC_VER)
#  define GA_NO_UNIQUE_ADDRESS [[msvc::no_unique_address]]
#else
#  define GA_NO_UNIQUE_ADDRESS [[no_unique_address]]
#endif
```

### 呼び出し側は変わらない

```cpp
    AllocResult Allocate(std::size_t size, std::size_t alignment,
                         const std::source_location& loc = std::source_location::current()) noexcept
    {
        // ... 検査と計算 ...

        // ★ 分岐がない。NoStats なら空の関数呼び出し → 完全に消える
        stats_.RecordAllocation(size, padding, currentTag_);

        if constexpr (Config::kFillPattern)
        {
            FillPattern(result, size, kPatternAllocated);
        }

        return result;
    }
```

**`if constexpr` と「何もしないクラス」を併用します。**

- **状態を持つ機能** → クラスに切り出して `conditional_t`
- **状態を持たない処理** → `if constexpr`

### `sizeof` を確認する

```cpp
int main()
{
    std::println("Debug       : {} バイト", sizeof(ga::BasicBump<ga::DebugConfig>));
    std::println("Development : {} バイト", sizeof(ga::BasicBump<ga::DevelopmentConfig>));
    std::println("Release     : {} バイト", sizeof(ga::BasicBump<ga::ReleaseConfig>));
}
```

```
Debug       : 1096 バイト
Development :  480 バイト
Release     :   72 バイト
```

**Release では、本当に必要なものだけになりました。**

---

## 51.5 3つの構成

**「Debug と Release の2つ」では足りません。**

### Debug 構成の問題

```
Allocate     : 28.4 ns(13 倍)
Reset(1 MB) : 412 µs
最適化       : なし(第4章で見たとおり、20 倍遅い)
```

**ゲームが 60 fps で動きません。**

**動かないビルドでは、ゲームプレイのバグを見つけられません。**

### Release 構成の問題

```
統計       : なし → 第49章の予算管理ができない
確保元     : なし → 問題の原因が分からない
検査       : なし → 壊れても気づかない
```

**バグが起きても、何も分かりません。**

### だから、中間が要る

| 構成 | 用途 | 期間 |
|---|---|---|
| **Debug** | 特定のバグを追い詰める | 短い(数時間〜数日) |
| **開発(Development)** | **日常の開発、QA、プレイテスト** | **最も長い** |
| **Release** | 出荷 | 最後 |

> **開発ビルドこそが、最も長く使われる構成です。**
>
> プログラマも、デザイナーも、QA も、**日常的に触るのはこの構成です。**
>
> **「実機に近い速度で動き、問題が起きたら手がかりが残る」** ——これが要件です。

### 開発ビルドの設計方針

**残すもの:**

- **統計とタグ**(第15章)。第49章の予算管理に必須。**コストは 0.9 ns**
- **タグ境界**(第19章)。予算の集計に必要。**0.2 ns**
- **スレッド検査**(第34章)。**0.3 ns で、深刻なバグを防ぐ**
- **所有権検査**(第22章)。**1 ns 未満**

**外すもの:**

- 塗りつぶし(+2.6 ns、`Reset` が O(n))
- ガードバイト(+19.5 ns、メモリ 96 B/確保)
- スタックトレース(+1893 ns)
- 確保元の記録(+1.3 ns、実行ファイル +7 KB)

**合計:2.1 → 3.5 ns。67% の増加で済みます。**

### Visual Studio に構成を追加する

**「構成マネージャー」で、Release をコピーして `Development` を作ります。**

```
プロパティ → C/C++ → プリプロセッサ → プリプロセッサの定義
    GA_CONFIG_DEVELOPMENT
```

```cpp
// ga/Config.h
#if defined(GA_CONFIG_DEBUG)
    using DefaultConfig = DebugConfig;
#elif defined(GA_CONFIG_DEVELOPMENT)
    using DefaultConfig = DevelopmentConfig;
#else
    using DefaultConfig = ReleaseConfig;
#endif
```

**第13章で作ったプロパティシートに、この定義を追加します。**

---

## 51.6 アリーナごとに設定を変える

**型が設定を持つので、アリーナごとに変えられます。**

```cpp
// フレームアリーナ:毎フレーム大量に確保 → 最小限の検査
ga::BasicBump<ga::ReleaseConfig> g_frameArena(64 * 1024 * 1024);

// シーンアリーナ:ロード時のみ → 厚い検査でも問題ない
ga::BasicBump<ga::DebugConfig> g_sceneArena(512 * 1024 * 1024);
```

**51.2 節の問題3が解決しました。**

### 型が違うと、関数に渡せない

```cpp
void Process(ga::Bump& arena);       // ← どちらか一方しか渡せない
```

**これは意図した動作です。** 混ぜたらリンクエラー、という保護が働いています。

**両方を扱いたい場合は、テンプレートにします。**

```cpp
template <class Config>
void Process(ga::BasicBump<Config>& arena);
```

**あるいは、第39章の `std::pmr::memory_resource` で型消去します。**

```cpp
void Process(std::pmr::memory_resource* resource);   // ← どちらでも渡せる
```

> **第39章で「型消去には仮想呼び出しのコストがある」と書きました。**
>
> **その代わりに得られるのが、この柔軟性です。** 用途に応じて選んでください。

---

## 51.7 測る

### 速度

```
構成            Allocate    Reset(1 MB)   New<Vec3>
Debug            28.4 ns      412 µs       31.2 ns
Development       3.5 ns        0 µs        4.0 ns
Release           2.1 ns        0 µs        2.5 ns
```

**開発構成は、Release の 1.67 倍。** 実用的な範囲です。

### メモリ

```
構成          sizeof(Bump)   1万確保あたりの追加メモリ
Debug            1096 B         960 KB(ガードバイト)
Development       480 B           0 B
Release            72 B           0 B
```

### 実行ファイルのサイズ

```
構成          実行ファイル    増分
Debug            412 KB      +78 KB
Development      348 KB      +14 KB
Release          334 KB        ±0
```

**Debug の +78 KB は、主に `source_location` の文字列(第14章)と、`std::stacktrace` の実体化です。**

### 1フレームあたりの影響(第43章の負荷)

```
構成          10,000 確保 + Reset
Debug              298 µs   ← 16.6 ms 予算の 1.8%
Development         35 µs   ← 0.21%
Release             21 µs   ← 0.13%
```

**Debug でも、アロケーターだけなら 1.8% です。**

**しかし、第4章で見たとおり、Debug 構成では最適化が無効なので、ゲーム全体が 20 倍遅くなります。** アロケーターだけの問題ではありません。

---

## 51.8 実行時に切り替える機能

**コンパイル時に決められないものもあります。**

```cpp
// 第14章:ログのコールバック
arena.SetLogCallback(&PrintAllocation);

// 第18章:トレースの設定
ga::TraceConfig cfg;
cfg.enabled = true;
cfg.minSize = 1024 * 1024;
arena.SetTraceConfig(cfg);
```

**「問題が起きたときだけ、実行中に有効にする」** という使い方です。

### コスト

```cpp
    if (logCallback_ != nullptr) { logCallback_(info, logUser_); }
```

**分岐1回です。** そして、**ほぼ常に同じ結果になるので、分岐予測がほぼ 100% 当たります。**

```
分岐なし              2.1 ns
分岐あり(常に false)  2.2 ns
```

**0.1 ns。実質無料です。**

### 使い分け

| 判断のタイミング | 手段 | 例 |
|---|---|---|
| **コンパイル時** | テンプレートパラメータ | 統計、ガードバイト、塗りつぶし |
| **起動時** | 設定ファイル、コマンドライン | トレースの有効化 |
| **実行中** | ImGui、コンソールコマンド | ログの一時的な有効化 |

**構造を変える機能はコンパイル時、動作を変える機能は実行時。** これが目安です。

---

## 51.9 Release でも残すもの

**第22章で、こう判断しました。**

> **所属チェックと二重解放チェックは、Release でも常時有効にします。**
> コストは 1 ナノ秒未満。そして防げる事故の深刻さは、はみ出し検出と同等かそれ以上。

**この判断基準を、一般化します。**

### 判断の基準

```
残すべき :  コストが 1 ns 未満  かつ  防げる被害がメモリ破壊
外すべき :  コストが 数 ns 以上  または  被害が限定的
```

| 検査 | コスト | 被害 | 判断 |
|---|---|---|---|
| **所属チェック**(第22章) | 0.4 ns | メモリ破壊 | **残す** |
| **二重解放**(第22章) | 0.3 ns | メモリ破壊 | **残す** |
| **アラインメント検査**(第7章) | 0.1 ns | 未定義動作 | **残す** |
| **容量超過**(第7章) | 0.2 ns | メモリ破壊 | **残す** |
| 巻き戻しの順序(第9章) | 0.3 ns | メモリ破壊 | **残す**(最低限の防衛) |
| スレッド所有者(第34章) | 0.3 ns | データ競合 | 開発まで |
| 塗りつぶし(第16章) | 2.6 ns | 検出のみ | 外す |
| ガードバイト(第17章) | 19.5 ns | 検出のみ | 外す |

### 「検出」と「防止」を区別する

**重要な区別です。**

- **防止する検査**(不正な操作を、実際に止める)→ **残す**
- **検出する検査**(壊れたことを、後から知らせる)→ **外してよい**

**第22章の所属チェックは、「別のプールに返す」操作を実際に止めます。** 外すと、その場でメモリが壊れます。

**第17章のガードバイトは、壊れたことを後から知らせるだけです。** 外しても、壊れ方は変わりません。

> **Release で残すべきなのは、「防止」する検査です。**

---

## 演習

**演習51-1** `NoStats` を使い、`sizeof(BasicBump<ReleaseConfig>)` が 72 バイトになることを確認してください。

**演習51-2** `[[msvc::no_unique_address]]` を外して `sizeof` を測ってください。何バイト増えますか。

**演習51-3** `BasicBump<DebugConfig>` を受け取る関数に `BasicBump<ReleaseConfig>` を渡してください。エラーメッセージは読めますか。

**演習51-4** 独自の構成(たとえば「統計とガードバイトだけ有効」)を作ってください。

**演習51-5** 開発構成の `Allocate` を測り、Release の何倍か確認してください。

**演習51-6** `Pool` と `Tlsf` にも、同じ設定機構を導入してください。

**演習51-7** 51.9 節の表に従い、`ReleaseConfig` で残す検査を実装してください。コストを測ってください。

**演習51-8** Visual Studio に `Development` 構成を追加し、第43章のフレームループを3構成で測ってください。

---

## 章末チェックリスト

- [ ] 積み上がった機能とコストを一覧にした
- [ ] **`#if` によるメンバの削除が ODR 違反になる** 理由を、具体的な症状で説明できる
- [ ] 設定をテンプレートパラメータにし、型が別物になることを確認した
- [ ] **混ぜたらリンクエラーになる** ことを確認した
- [ ] `std::conditional_t` と空のクラスで、メンバを本当に消した
- [ ] `[[msvc::no_unique_address]]` の効果を測った
- [ ] 3つの構成を設計し、**開発ビルドが最も長く使われる** 理由を説明できる
- [ ] アリーナごとに設定を変えられることを確認した
- [ ] コンパイル時と実行時の切り替えを使い分けられる
- [ ] **「防止」と「検出」の区別** で、Release に残す検査を判断できる

---

## 次章の予告

**第52章は、この本の実践編です。**

小さな 2D ゲームを用意し、**`new` 版から自作アロケーター版へ、段階的に移行します。**

```
段階0: すべて new / delete            → 基準を測る
段階1: フレーム単位のデータをアリーナへ  → 第43章
段階2: パーティクルと弾をプールへ       → 第21章
段階3: シーンのロードをアリーナへ       → 第44章
段階4: 残りを TLSF へ                 → 第27章
```

**各段階で必ず測ります。** フレーム時間、最悪フレーム、ピークメモリ。

**そして、移行中に踏んだバグを、第2部で作った道具で潰します。**

- リセット後のポインタ使用 → 第16章の塗りつぶし
- はみ出し → 第17章のガードバイト
- スレッドの取り違え → 第34章の所有者検査

**「本に書いてある手順どおりにやったら、すんなり動いた」という話にはなりません。**

**実際に起きることを、正直に書きます。**

---

> **コラム:ゼロオーバーヘッド原則の、実践と限界**
>
> **C++ の設計原則として、しばしば引用される言葉があります。**
>
> > **使わない機能に対して、対価を払う必要はない。**
> > **使う機能については、手で書くよりも良いコードは書けない。**
>
> **Bjarne Stroustrup が挙げた、C++ の設計指針です。**
>
> ---
>
> **この章でやったこと**
>
> **51.4 節の設計は、この原則の実践です。**
>
> ```
> 統計を使わない → sizeof が増えない、コードが生成されない
> ガードを使わない → メモリも時間も一切消費しない
> ```
>
> **`if constexpr` と `[[no_unique_address]]` と `std::conditional_t`。** これらは、**原則を実現するための道具** です。
>
> **興味深いのは、これらがすべて比較的新しい機能だということです。**
>
> - `std::conditional_t` : C++11
> - `if constexpr` : C++17
> - `[[no_unique_address]]` : C++20
>
> **原則は最初からありましたが、それを実現する道具が揃うまでに、数十年かかりました。**
>
> ---
>
> **原則が守られていない例**
>
> **公平のために、C++ 自身が原則を破っている箇所も挙げます。**
>
> **仮想関数。** 1つでも仮想関数を持つクラスは、`vptr` を持ちます。**そのオブジェクトを多態的に使わなくても、8 バイト増えます。**
>
> **例外。** 例外を投げないコードでも、巻き戻しのためのテーブルが実行ファイルに含まれます。**`/EHsc` を外せば消せますが、標準ライブラリが例外を使うため、現実には難しい。**
>
> **RTTI。** `dynamic_cast` を使わなくても、型情報が生成されます。
>
> **これらが、ゲーム業界で例外や RTTI が敬遠されてきた理由の1つです。** 第7章で例外を選ばなかったのも、この文脈にあります。
>
> ---
>
> **原則の限界**
>
> **「使わないものに払わない」は、理想です。**
>
> **現実には、次の妥協が発生します。**
>
> - **ABI 互換性。** 一度公開したクラスのレイアウトは変えられない。MSVC が `[[no_unique_address]]` を無視するのも、この理由です(第40章)
> - **コンパイル時間。** テンプレートで everything を解決すると、ビルドが遅くなります
> - **可読性。** 51.4 節のコードは、素直な実装より読みにくい
>
> **この本の第39章で `std::pmr` を扱ったとき、同じ話がありました。**
>
> ```
> テンプレート : 速いが、型が増え、コンパイルが遅い
> 型消去       : 遅いが、型が1つで、柔軟
> ```
>
> **どちらが「正しい」ということはありません。**
>
> ---
>
> **私たちの選択**
>
> **この本では、両方を用意しました。**
>
> - **テンプレート版**(第38章、この章):性能が critical な場所
> - **`std::pmr` 版**(第39章):柔軟性が必要な場所
>
> **そして、選べることに価値があります。**
>
> **第53章で、この本の締めくくりとして「いつ自作すべきか」を扱います。** その判断も、結局は同じ構図です。
>
> **何かを得るために、何かを諦める。** 第20章で「制約の付け替え」と呼んだものが、性能の話だけでなく、**設計の柔軟性についても当てはまります。**
