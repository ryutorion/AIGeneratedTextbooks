# 第18章 コールスタックを保存する 〔v0.13〕

---

## この章のゴール

前章のガード違反レポートは、こう報告しました。

```
[ガード違反] main.cpp:42 で確保した 16 バイトの領域の 後ろ が壊れています
```

しかし 42 行目が `LoadMesh()` の中で、その関数が10か所から呼ばれていたら? **どの経路で呼ばれたときの確保なのか、特定できません。**

この章では、確保時の **コールスタック全体** を記録します。

```
[ガード違反] 16 バイトの領域が壊れています
  確保元:
    ga::Bump::NewArray<int>    ga\Bump.h(243)
    LoadMesh                   MeshLoader.cpp(88)
    LoadStage                  StageLoader.cpp(34)
    main                       main.cpp(12)
```

扱うこと。

- C++23 の `std::stacktrace` を使う 〔**v0.13**〕
- **捕捉のコスト** と **記号化のコスト** を分けて考える
- アロケーターの中でメモリを確保するという、自己再帰の危険
- 全部記録するのは無理だと認め、**サイズ閾値** と **サンプリング** で折り合う
- Release ビルドでスタックトレースが役に立たなくなる理由

---

## 18.1 `std::stacktrace`

C++23 で追加されました。`<stacktrace>` をインクルードします。

```cpp
#include <print>
#include <stacktrace>

void Inner()
{
    const auto trace = std::stacktrace::current();

    for (const auto& entry : trace)
    {
        std::println("{}  {}({})",
                     entry.description(),
                     entry.source_file(),
                     entry.source_line());
    }
}

void Middle() { Inner(); }
void Outer()  { Middle(); }

int main() { Outer(); }
```

```
Inner   C:\dev\...\main.cpp(8)
Middle  C:\dev\...\main.cpp(18)
Outer   C:\dev\...\main.cpp(19)
main    C:\dev\...\main.cpp(21)
invoke_main  d:\a\_work\...\exe_common.inl(78)
...
```

| 要素 | 内容 |
|---|---|
| `std::stacktrace` | フレームの列。コンテナとして扱える |
| `entry.description()` | 関数名 |
| `entry.source_file()` | ファイル名 |
| `entry.source_line()` | 行番号 |

### 深さを制限する

そのまま使うと、CRT の起動コードまで全部含まれます。制御できます。

```cpp
std::stacktrace::current(skip, maxDepth);
```

| 引数 | 意味 |
|---|---|
| `skip` | 手前から何フレーム捨てるか |
| `maxDepth` | 最大何フレーム取るか |

私たちのアロケーターでは、`Bump::Allocate` や `AllocateGuarded` といった **内部のフレームは不要** です。`skip` で飛ばします。

```cpp
// 自分自身と、呼び出し元のラッパを飛ばして、8フレームだけ取る
auto trace = std::stacktrace::current(2, 8);
```

### ビルド設定の注意

スタックトレースを取るには、**シンボル情報(PDB ファイル)が必要** です。

- Debug 構成:既定で PDB が生成されます
- Release 構成:Visual Studio の既定では PDB は生成されます。生成されないよう変更している場合は、`リンカー → デバッグ → デバッグ情報の生成` を確認してください

環境やバージョンによっては、リンカの「追加の依存ファイル」に `Dbghelp.lib` や `Advapi32.lib` が必要になることがあります。未解決の外部シンボルのエラーが出たら追加してください。

---

## 18.2 2つのコストを分けて考える

`std::stacktrace` を使うとき、**まったく重さの違う2つの処理** が存在します。ここを理解しないと、性能を見誤ります。

### コスト1:捕捉(capture)

`std::stacktrace::current()` が行うのは、**戻りアドレスの列を集めること** です。数値の配列を作るだけで、関数名もファイル名もまだ分かりません。

x64 の Windows では、スタックの巻き戻し情報を使ってフレームをたどります。それなりの処理ですが、マイクロ秒のオーダーです。

### コスト2:記号化(symbolization)

`entry.description()` や `entry.source_file()` を呼んだ瞬間に、**アドレスから関数名とファイル名を引く** 処理が走ります。

PDB ファイルを読み、シンボルテーブルを検索します。**桁違いに重い処理** です。

### 実測

```cpp
int main()
{
    // 捕捉だけ
    auto rCapture = bench::Measure(1000, [] {
        auto t = std::stacktrace::current(0, 16);
        bench::Escape(t.size());
    });

    // 捕捉 + 記号化
    auto rSymbol = bench::Measure(100, [] {
        auto t = std::stacktrace::current(0, 16);
        for (const auto& e : t) { bench::Escape(e.description().size()); }
    });

    bench::Print("捕捉のみ", rCapture);
    bench::Print("捕捉+記号化", rSymbol);
}
```

```
捕捉のみ       median=   1840.0  p95=   2100.0  max=    9800.0
捕捉+記号化    median= 412000.0  p95= 480000.0  max= 2100000.0
```

**220 倍以上の差** です。しかも初回はさらに重く、PDB の読み込みで数百ミリ秒かかることもあります。

### 設計の指針

> **捕捉は確保時に行い、記号化は表示時にだけ行う。**

`std::stacktrace` オブジェクトを保存しておけば、記号化は後回しにできます。**アドレスだけ持っておいて、レポートを出すときに初めて名前を引く。** これが唯一まともに使える形です。

もし確保のたびに `description()` を呼んでいたら、1回の確保に 0.4 ミリ秒かかります。1000 回で 0.4 秒。実用になりません。

---

## 18.3 落とし穴:アロケーターの中でメモリを確保する

`std::stacktrace` は、内部にフレームの列を持つ **コンテナ** です。当然、メモリを確保します。

```cpp
template <class Allocator>
class basic_stacktrace;

using stacktrace = basic_stacktrace<std::allocator<std::stacktrace_entry>>;
```

アロケーターを差し替えられる設計になっています。**私たちのアロケーターを渡したくなります。**

```cpp
// これは非常に危険
using MyTrace = std::basic_stacktrace<BumpAllocatorAdapter<std::stacktrace_entry>>;
```

**やってはいけません。**

`Bump::Allocate()` の中でスタックトレースを捕捉し、その捕捉が `Bump::Allocate()` を呼ぶ。**無限再帰** です。

### 一般則

> **アロケーターのデバッグ機能は、そのアロケーター自身を使ってはならない。**

第14章でリングバッファを固定長の配列にしたのも、第17章でガードのヘッダを板の中に置いたのも、この原則に沿っています。

この章では、`std::stacktrace` を **システムのヒープ**(`std::allocator`、つまり `new`)から確保させます。デバッグ機能が本番の経路とは別のメモリを使うのは、むしろ望ましい性質です。

> **第38章で `std::vector` に自作アロケーターを差し込むとき**、この落とし穴をもう一度扱います。「差し込めるからといって、差し込んでよいとは限らない」という話です。

---

## 18.4 全部は記録できない

正直に言うと、**すべての確保でスタックトレースを取るのは非現実的です**。

```
v0.12 (ガードあり)                median=     26.4 ns
v0.13 (毎回スタックトレース)      median=   1920.0 ns
```

**73 倍。** 加えて、1件あたり数百バイトのメモリを消費します。10 万回の確保で数十メガバイトです。

だから **絞ります**。方針は3つあります。

### 方針A:サイズの閾値

```cpp
if (size >= config_.minSizeForTrace) { /* 記録する */ }
```

大きな確保だけを記録します。「4 MB も確保しているのは誰か」を調べるには、これで十分です。

**実務ではこれが最もよく使われます。** メモリを食っているのは、たいてい少数の大きな確保だからです。

### 方針B:サンプリング

```cpp
if (++counter_ % config_.sampleRate == 0) { /* 記録する */ }
```

N 回に1回だけ記録します。統計的な傾向をつかむのに向いています。

「小さい確保が100万回あるが、どこから来ているのか」を調べるときに使います。全部は要りません。**分布が分かればいい。**

### 方針C:タグで絞る

```cpp
if (currentTag_ == config_.traceTag) { /* 記録する */ }
```

第15章のタグを使い、特定のカテゴリだけ記録します。「Texture が想定より多い」と分かった後、Texture だけ追跡する、という使い方です。

### 3つを組み合わせる

実際には、これらを併用できるようにします。

```cpp
struct TraceConfig
{
    bool        enabled     = false;
    std::size_t minSize     = 0;                 // これ以上のサイズだけ
    unsigned    sampleRate  = 1;                 // N 回に1回
    std::size_t maxDepth    = 12;
    std::size_t maxRecords  = 1024;              // 記録の上限
    MemoryTag   onlyTag     = MemoryTag::Count;  // Count なら全タグ
};
```

**既定では無効** にします。必要なときだけ有効にする、という位置づけです。

---

## 18.5 実装する 〔v0.13〕

### 記録の形

```cpp
// ga/detail/TraceRecord.h
#pragma once

#include <cstddef>
#include <stacktrace>

#include "ga/MemoryTag.h"

namespace ga::detail
{
    struct TraceRecord
    {
        const void*     address = nullptr;
        std::size_t     size    = 0;
        MemoryTag       tag     = MemoryTag::General;
        std::stacktrace trace{};
    };
}
```

### `Bump` への追加

```cpp
public:
    void SetTraceConfig(const TraceConfig& config)
    {
        traceConfig_ = config;
        traces_.reserve(config.maxRecords);
    }

    // 記録されたトレースを列挙する
    void ForEachTrace(auto&& fn) const
    {
        for (const auto& r : traces_) { fn(r); }
    }

    // 特定のアドレスに対応するトレースを探す
    const detail::TraceRecord* FindTrace(const void* address) const noexcept
    {
        for (const auto& r : traces_)
        {
            if (r.address == address) { return &r; }
        }
        return nullptr;
    }

private:
    bool ShouldTrace(std::size_t size) noexcept
    {
        if (!traceConfig_.enabled)                     { return false; }
        if (traces_.size() >= traceConfig_.maxRecords) { return false; }
        if (size < traceConfig_.minSize)               { return false; }

        if (traceConfig_.onlyTag != MemoryTag::Count &&
            traceConfig_.onlyTag != currentTag_)       { return false; }

        if (traceConfig_.sampleRate > 1)
        {
            if (++traceCounter_ % traceConfig_.sampleRate != 0) { return false; }
        }

        return true;
    }

    void CaptureTrace(const void* address, std::size_t size)
    {
#if GA_ENABLE_STACKTRACE
        if (!ShouldTrace(size)) { return; }

        traces_.push_back(detail::TraceRecord{
            address, size, currentTag_,
            std::stacktrace::current(kTraceSkipFrames, traceConfig_.maxDepth)
        });
#else
        (void)address; (void)size;
#endif
    }

    static constexpr std::size_t kTraceSkipFrames = 3;

    TraceConfig                       traceConfig_{};
    std::vector<detail::TraceRecord>  traces_;
    std::size_t                       traceCounter_ = 0;
```

`traces_` は `std::vector` なので、**システムのヒープ** から確保されます。18.3 節の原則どおりです。`reserve` しておけば、確保中の再確保も避けられます。

### `Marker` と `Reset()` への対応

第11章のファイナライザ、第17章のガードと同じパターンです。マーカーに記録数を持たせます。

```cpp
    struct Marker
    {
        // ... 既存のフィールド ...
        std::size_t traceCount = 0;   // v0.13
    };

    Marker Mark() noexcept
    {
        return Marker{ offset_, padding_, depth_++, epoch_,
                       finalizers_, guardBlocks_, traces_.size() };
    }

    void Rewind(const Marker& m) noexcept
    {
        // ... 検査、ガード確認、ファイナライザ、塗りつぶし ...

        if (traces_.size() > m.traceCount)
        {
            traces_.resize(m.traceCount);
        }
    }

    void Reset() noexcept
    {
        // ... 従来の処理 ...

        traces_.clear();     // 容量は保持されるので、再確保は起きない
    }
```

`clear()` は容量を解放しません。次の周期でも、確保なしで使い回せます。

---

## 18.6 使ってみる

### 表示関数

```cpp
// ga/Report.h に追加
namespace ga
{
    inline void PrintTrace(const std::stacktrace& trace, const char* indent = "    ")
    {
        for (const auto& entry : trace)
        {
            std::string_view file = entry.source_file();
            if (auto pos = file.find_last_of("\\/"); pos != std::string_view::npos)
            {
                file = file.substr(pos + 1);
            }

            if (file.empty())
            {
                std::println("{}{}", indent, entry.description());
            }
            else
            {
                std::println("{}{}  {}({})", indent, entry.description(), file, entry.source_line());
            }
        }
    }
}
```

### 用途1:ガード違反に確保元を添える

第17章のコールバックを強化します。

```cpp
ga::Bump* g_arena = nullptr;

void OnGuardViolation(const ga::Bump::GuardViolation& v, void*) noexcept
{
    std::println("[ガード違反] {} バイトの領域の {} が壊れています",
                 v.userSize, v.isFront ? "手前" : "後ろ");

    if (const auto* rec = g_arena->FindTrace(v.userBegin))
    {
        std::println("  確保元:");
        ga::PrintTrace(rec->trace, "    ");
    }
}
```

```
[ガード違反] 64 バイトの領域の 後ろ が壊れています
  確保元:
    ga::Bump::NewArray<Vertex>   Bump.h(312)
    LoadSubMesh                  MeshLoader.cpp(88)
    LoadMesh                     MeshLoader.cpp(140)
    LoadStage                    StageLoader.cpp(34)
    main                         main.cpp(12)
```

**呼び出し経路が全部見えます。** 同じ `LoadSubMesh` でも、どのメッシュの読み込み中かが `LoadMesh` の行番号から分かります。

### 用途2:大きな確保の犯人探し

```cpp
int main()
{
    ga::Bump arena(64 * 1024 * 1024);

    ga::TraceConfig cfg;
    cfg.enabled  = true;
    cfg.minSize  = 1024 * 1024;    // 1 MB 以上だけ
    cfg.maxDepth = 8;
    arena.SetTraceConfig(cfg);

    LoadStage(arena);

    std::println("=== 1 MB 以上の確保 ===");
    arena.ForEachTrace([](const ga::detail::TraceRecord& r) {
        std::println("{} ({})", ga::FormatBytes(r.size), ToString(r.tag));
        ga::PrintTrace(r.trace, "    ");
        std::println("");
    });
}
```

```
=== 1 MB 以上の確保 ===
4.00 MB (Texture)
    ga::Bump::Allocate            Bump.h(198)
    LoadTexture                   TextureLoader.cpp(52)
    LoadStage                     StageLoader.cpp(41)
    main                          main.cpp(12)

2.50 MB (Mesh)
    ga::Bump::NewArray<Vertex>    Bump.h(312)
    LoadSubMesh                   MeshLoader.cpp(88)
    ...
```

**第15章のレポートで「Texture が 3.0 MB」と分かった後、その中身を追う** という流れになります。

- 第15章:**どのカテゴリ** が食っているか
- 第18章:**どのコード** が食っているか

2段構えです。

### 用途3:サンプリングで傾向を見る

```cpp
    ga::TraceConfig cfg;
    cfg.enabled    = true;
    cfg.sampleRate = 1000;         // 1000 回に1回
    cfg.maxRecords = 200;
    arena.SetTraceConfig(cfg);
```

100 万回の小さな確保があるとき、そのうち 200 件を記録します。同じスタックが繰り返し現れれば、そこが主要な発生源です。

**統計的な調査には、全件は要りません。**

---

## 18.7 Release ビルドでの現実

Debug では美しく動きます。Release では、事情が変わります。

### インライン展開でフレームが消える

```
Debug 構成:
    ga::Bump::NewArray<Vertex>   Bump.h(312)
    LoadSubMesh                  MeshLoader.cpp(88)
    LoadMesh                     MeshLoader.cpp(140)
    LoadStage                    StageLoader.cpp(34)
    main                         main.cpp(12)

Release 構成:
    LoadStage                    StageLoader.cpp(34)
    main                         main.cpp(12)
```

`NewArray`、`LoadSubMesh`、`LoadMesh` が消えました。**インライン展開されて、独立したフレームでなくなった** ためです。

これは最適化の当然の帰結であり、バグではありません。しかし調査の役には立ちません。

### 対処

| 手段 | 効果 | 代償 |
|---|---|---|
| 調査したい関数に `[[msvc::noinline]]` を付ける | フレームが残る | その関数だけ遅くなる |
| `/Ob1`(インライン展開を制限) | フレームが増える | 全体が遅くなる |
| Debug 構成で再現させる | 完全な情報 | 再現しない場合がある |
| PDB を保存しておく | 後から解析できる | 管理の手間 |

**実務では「PDB を必ず保存する」が基本になります。** 出荷したビルドと同じ PDB があれば、クラッシュレポートのアドレスを後から記号化できます。

### 行番号は残る

関数フレームは消えても、`source_line()` はインライン展開後の位置を返してくれることがあります。完全に無価値になるわけではありません。

---

## 18.8 コストを測る

### 記録が有効な場合

```
v0.12 (トレースなし)              median=     26.4 ns
v0.13 (全件トレース、深さ12)      median=   1920.0 ns
v0.13 (1 MB 以上のみ)             median=     27.1 ns
v0.13 (1000 回に1回)              median=     28.3 ns
```

**閾値やサンプリングを使えば、実質的なコストはほぼゼロ** です。判定は比較数回だけなので、記録しない確保は従来と変わりません。

### メモリ

1件あたりのおおよその消費です。

```
TraceRecord            = 8 + 8 + 1(+詰め物) + std::stacktrace
std::stacktrace の中身 = 12 フレーム × 8 バイト + コンテナの管理領域
                       ≈ 130 バイト
```

`maxRecords = 1024` なら、約 130 KB。許容範囲です。

**上限を設けているのは重要です。** 無制限にすると、長時間動かすうちにデバッグ機能が本体よりメモリを食うことになります。

---

## 18.9 この章の完成コード

```cpp
// ga/TraceConfig.h(新規)
#pragma once

#include <cstddef>
#include "ga/MemoryTag.h"

namespace ga
{
    struct TraceConfig
    {
        bool        enabled    = false;
        std::size_t minSize    = 0;
        unsigned    sampleRate = 1;
        std::size_t maxDepth   = 12;
        std::size_t maxRecords = 1024;
        MemoryTag   onlyTag    = MemoryTag::Count;
    };
}
```

```cpp
// ga/Core.h に追加
#ifndef GA_ENABLE_STACKTRACE
#  ifdef NDEBUG
#    define GA_ENABLE_STACKTRACE 0
#  else
#    define GA_ENABLE_STACKTRACE 1
#  endif
#endif
```

`ga/Bump.h` への追加は 18.5 節のとおりです。`Marker` に `traceCount` が加わり、フィールドは7個になりました。

```cpp
    struct Marker
    {
        std::size_t          offset     = 0;
        std::size_t          padding    = 0;
        std::uint32_t        depth      = 0;
        std::uint32_t        epoch      = kInvalidEpoch;
        detail::Finalizer*   finalizers = nullptr;
        detail::GuardBlock*  guards     = nullptr;
        std::size_t          traceCount = 0;
    };
```

**マーカーが「その時点のアロケーターの全状態」を表す** という設計が、ここまで一貫しています。機能を追加するたびにフィールドが1つ増え、`Rewind()` がそれを戻す。この規則性のおかげで、追加が機械的に行えています。

---

## 演習

**演習18-1** `kTraceSkipFrames` を 0 にすると、レポートの先頭に何が現れますか。適切な値はどうやって決めればよいでしょうか。

**演習18-2** `std::stacktrace` を `std::basic_stacktrace` に変え、自作アロケーターを渡してみてください。何が起きますか(スタックオーバーフローで落ちるので、デバッガで確認してください)。

**演習18-3** 同じスタックトレースが繰り返し記録される場合、ハッシュで重複を排除できます。実装して、記録件数がどれだけ減るか確認してください。

**演習18-4** `FindTrace` は線形探索です。記録が 1024 件あるとき、どれくらいかかりますか。ハッシュ表にすると改善しますか。

**演習18-5** Release 構成でトレースを取り、Debug 構成と比べてください。どのフレームが消えますか。`[[msvc::noinline]]` を付けると戻りますか。

**演習18-6** サンプリングを「N 回に1回」ではなく「確率的に」行うよう変更してください。周期的な確保パターンがあるとき、どちらが正確ですか。

**演習18-7** 捕捉した `std::stacktrace` をファイルに保存し、別のプロセスで記号化することはできますか。何が必要でしょうか。

---

## 章末チェックリスト

- [ ] `std::stacktrace::current(skip, maxDepth)` を使った
- [ ] **捕捉と記号化のコストの差**(200倍以上)を実測した
- [ ] 「捕捉は確保時、記号化は表示時」の原則を説明できる
- [ ] アロケーターのデバッグ機能が自分自身を使ってはいけない理由を説明できる
- [ ] サイズ閾値・サンプリング・タグの3つの絞り込みを実装した 〔v0.13〕
- [ ] ガード違反レポートに確保元スタックを添えた
- [ ] Release 構成でフレームが消えることを確認した
- [ ] 記録件数の上限を設ける理由を説明できる

---

## 次章の予告

第2部もいよいよ最後です。ここまでで集めた情報は、すべて **文字** でした。数字、表、スタックトレース。

第19章では **絵にします**。

板の使用状況を、1ピクセル = 1KB として PNG 画像に出力します。使用中は色、未使用は黒。タグごとに色を変えれば、メモリの中で何がどこに置かれているかが一目で分かります。

```
████████░░░░████░░░░░░░░░░░░░░░░░░░░
Mesh     空き Tex  空き
```

バンプアロケーターでは、絵は単純な帯になります。**面白くなるのは第20章以降です。** 個別解放を導入した瞬間、この帯に穴が空きはじめます。穴が散らばり、合計では足りているのに確保できなくなる——**断片化** が、絵として目に見えるようになります。

第19章で作る道具は、第3部を通してずっと使い続けることになります。

---

> **コラム:スタックトレースを取るのは、なぜ難しいのか**
>
> 「呼び出し元をたどる」というだけの処理が、なぜマイクロ秒もかかるのでしょうか。
>
> ---
>
> 素朴な方法は、**フレームポインタ** をたどることです。x86 の伝統的な関数呼び出しでは、`ebp` レジスタが現在のフレームの底を指し、そこに1つ前の `ebp` が保存されています。数珠つなぎになっているので、たどるだけでスタックが復元できます。
>
> ところが、この方式は **最適化で消えます**。フレームポインタは便利ですが、レジスタを1本占有します。コンパイラは「使えるレジスタが1本増えるなら」と、フレームポインタを省略します(FPO: Frame Pointer Omission)。x64 では、これが既定の動作です。
>
> ---
>
> では x64 でどうやってたどるのか。**テーブルを引きます。**
>
> x64 の Windows では、実行ファイルに **アンワインド情報** のテーブルが埋め込まれています。「この関数はスタックを何バイト使い、どのレジスタを保存したか」という情報です。もともとは例外処理(SEH)のために用意されたものですが、スタックトレースにも使えます。
>
> つまり、1フレームたどるたびに **バイナリサーチでテーブルを引く** ことになります。フレームポインタをたどるより、はるかに重い。これが捕捉コストの正体です。
>
> ---
>
> そして記号化はさらに重い。アドレスから関数名を引くには、**PDB ファイル** を読む必要があります。PDB は実行ファイルとは別のファイルで、数十メガバイトから数百メガバイトになることもあります。初回のアクセスでは、これを開いてインデックスを構築します。
>
> だから「記号化は表示時にだけ」なのです。
>
> ---
>
> **ゲーム機での事情は、さらに複雑です。**
>
> 出荷するバイナリにシンボル情報を含めるわけにはいきません(解析されてしまいます)。かといって、クラッシュレポートにアドレスだけ入っていても読めません。
>
> 一般的な解は、**アドレスだけを送り、開発側で記号化する** という分業です。ビルドごとに PDB を保管し、レポートのアドレスと突き合わせます。そのために、どのビルドのバイナリかを識別する情報をレポートに含めます。
>
> 演習18-7 は、この仕組みを自分で考えてもらう問題です。実際の運用では、これが「クラッシュレポート基盤」と呼ばれるものになります。
>
> ---
>
> ちなみに、`std::stacktrace` が C++23 まで標準化されなかったのは、**このあたりが徹底的に環境依存だから** です。Windows は PDB とアンワインドテーブル、Linux は DWARF と `libbacktrace`、macOS は dSYM。共通化できるのは API の形だけでした。
>
> 逆に言えば、標準化されたことで **移植可能なコードで書けるようになった** のは大きな進歩です。かつては環境ごとに `#ifdef` で分岐し、`DbgHelp` や `backtrace_symbols` を直接呼ぶ必要がありました。私たちが数行で済ませられるのは、その苦労が標準の裏側に隠されたおかげです。
