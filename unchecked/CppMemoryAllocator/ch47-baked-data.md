# 第47章 ファイルをそのまま載せる 〔v0.35〕

---

## この章のゴール

第44章で、シーンのロードを `BumpScope` で整理しました。**しかし、まだ大きな無駄があります。**

```
ファイルを読む     → 一時バッファに 40 MB
パースする         → 中間データに 30 MB
オブジェクトを作る → シーンアリーナに 35 MB
```

**同じデータが、実質3回コピーされています。**

**この章では、それをゼロにします。**

```cpp
auto raw = arena.Allocate(fileSize, 64);
ReadFile(path, *raw, fileSize);

const MeshHeader* mesh = ga::StartLifetimeAs<MeshHeader>(*raw);   // ← これだけ
```

**ファイルを読み込んだ瞬間に、使える状態になります。**

- **オフセットポインタ** ——ファイルに保存できる参照
- ビルド時に **ベイクする** 仕組み
- 第42章の `std::start_lifetime_as` を実用の形で使う 〔**v0.35**〕
- **メモリマップトファイル** で、読み込みすら省く
- そして、この技法の **落とし穴** を漏れなく挙げる

---

## 47.1 いまのロード処理の内訳

**40 MB のメッシュデータを読み込む処理を、分解して測ります。**

```cpp
void LoadMeshTraditional(ga::Bump& arena, const char* path)
{
    // ① ファイルを読む
    auto raw = arena.NewArrayUninit<std::byte>(fileSize);
    ReadFile(path, raw->data(), fileSize);

    // ② パースする(中間表現を作る)
    ParsedMesh parsed = ParseMeshFile(*raw);

    // ③ 実行時のオブジェクトを作る
    Mesh* mesh = arena.New<Mesh>();
    mesh->vertices.assign(parsed.vertices.begin(), parsed.vertices.end());
    mesh->indices.assign(parsed.indices.begin(), parsed.indices.end());
}
```

```
① ファイル読み込み  :  38.2 ms   (40 MB / 約 1 GB/s)
② パース            :  62.4 ms
③ オブジェクト構築  :  47.1 ms
──────────────────────────────────
合計                : 147.7 ms

ピークメモリ        : 112 MB   (40 + 32 + 40)
```

**読み込み以外に 109 ms、そして 2.8 倍のメモリ。**

### 何が無駄なのか

**「ファイルの形式」と「メモリ上の形式」が違うから、変換が要ります。**

```
ファイル: テキスト、あるいは独自の可変長形式
     ↓ パース
メモリ  : 構造体の配列
```

**もし、ファイルの中身が最初からメモリ上の形式だったら?**

**読み込んだだけで、そのまま使えます。**

---

## 47.2 ベイクという発想

**ビルド時に変換を済ませておきます。**

```
[ビルド時]
  元データ(FBX、glTF、テキスト)
      ↓ パース、最適化、変換
  メモリ上のレイアウトを組み立てる
      ↓ そのまま書き出す
  .bin ファイル

[実行時]
  .bin ファイルを読む
      ↓ (何もしない)
  使える
```

**変換のコストを、実行時からビルド時へ移動させています。**

### ゲーム開発では、当たり前の手法

**この発想は、メモリだけの話ではありません。**

- ライトマップ:光の計算を事前に済ませる
- ナビメッシュ:経路探索用のグラフを事前に構築
- シェーダ:コンパイルを事前に済ませる
- テクスチャ:圧縮形式に変換しておく

**「実行時にできることは、ビルド時にやる」** ——ゲーム開発の基本方針です。

---

## 47.3 何が邪魔になるか

**メモリの内容をそのまま保存するには、4つの問題があります。**

### 問題1:ポインタが保存できない

```cpp
struct Mesh
{
    Vertex* vertices;      // ← アドレスは実行のたびに変わる
    std::size_t count;
};
```

**保存したアドレスは、次回の起動では無意味です。** 第29章のコラムで触れた ASLR もあります。

**→ オフセットポインタで解決します(47.4 節)。**

### 問題2:アラインメント

```cpp
struct Vertex { float x, y, z; };    // alignof = 4
struct alignas(16) Matrix { ... };   // alignof = 16
```

**ファイルをどこに読み込むかで、アラインメントが変わります。**

**→ ベイク時に整列させ、読み込み先も十分に整列させます。**

### 問題3:構造体のレイアウトが、コンパイラ依存

```cpp
struct Header
{
    std::uint32_t a;
    std::uint64_t b;      // ← パディングが入る位置は?
};
```

**同じコンパイラ・同じ設定なら再現します。** しかし、バージョンを変えたり、別のプラットフォーム向けにビルドすると変わりえます。

**→ `static_assert` で固定します(47.10 節)。**

### 問題4:載せられない型がある

```cpp
struct Bad
{
    std::string  name;       // ← 内部にポインタを持つ
    virtual void Update();   // ← vtable ポインタを持つ
};
```

**→ `static_assert` で弾きます。**

---

## 47.4 オフセットポインタ

**絶対アドレスの代わりに、「自分自身からの相対位置」を持ちます。**

```
┌────────────────────────────────────────┐
│  ヘッダ                                  │
│    vertices.offset = +64  ──────────┐   │
│    vertices.count  = 1000           │   │
├─────────────────────────────────────┼───┤
│  (パディング)                        │   │
├─────────────────────────────────────▼───┤
│  Vertex[1000]                           │
└─────────────────────────────────────────┘
```

**どこに読み込んでも、相対関係は変わりません。**

### 実装

```cpp
// ga/OffsetPtr.h
#pragma once

#include <cstddef>
#include <cstdint>
#include <span>
#include <type_traits>

namespace ga
{
    // 自分自身のアドレスからの相対位置でオブジェクトを指す。
    // ⚠ コピーすると意味が変わる。値渡ししてはいけない。
    template <class T>
    class OffsetPtr
    {
    public:
        OffsetPtr() noexcept = default;

        [[nodiscard]] T* Get() noexcept
        {
            if (offset_ == 0) { return nullptr; }
            return reinterpret_cast<T*>(reinterpret_cast<std::byte*>(this) + offset_);
        }

        [[nodiscard]] const T* Get() const noexcept
        {
            if (offset_ == 0) { return nullptr; }
            return reinterpret_cast<const T*>(
                reinterpret_cast<const std::byte*>(this) + offset_);
        }

        void Set(T* target) noexcept
        {
            if (target == nullptr) { offset_ = 0; return; }

            const auto diff = reinterpret_cast<std::byte*>(target)
                            - reinterpret_cast<std::byte*>(this);

            offset_ = static_cast<std::int32_t>(diff);
        }

        T*       operator->()       noexcept { return Get(); }
        const T* operator->() const noexcept { return Get(); }
        T&       operator*()        noexcept { return *Get(); }
        const T& operator*()  const noexcept { return *Get(); }

        explicit operator bool() const noexcept { return offset_ != 0; }

    private:
        std::int32_t offset_ = 0;
    };

    // 配列版
    template <class T>
    class OffsetSpan
    {
    public:
        OffsetSpan() noexcept = default;

        [[nodiscard]] std::span<T> Get() noexcept
        {
            if (count_ == 0) { return {}; }
            return { reinterpret_cast<T*>(reinterpret_cast<std::byte*>(this) + offset_),
                     count_ };
        }

        [[nodiscard]] std::span<const T> Get() const noexcept
        {
            if (count_ == 0) { return {}; }
            return { reinterpret_cast<const T*>(
                         reinterpret_cast<const std::byte*>(this) + offset_),
                     count_ };
        }

        void Set(std::span<T> target) noexcept
        {
            if (target.empty()) { offset_ = 0; count_ = 0; return; }

            const auto diff = reinterpret_cast<std::byte*>(target.data())
                            - reinterpret_cast<std::byte*>(this);

            offset_ = static_cast<std::int32_t>(diff);
            count_  = static_cast<std::uint32_t>(target.size());
        }

        std::size_t Size() const noexcept { return count_; }

    private:
        std::int32_t  offset_ = 0;
        std::uint32_t count_  = 0;
    };
}
```

### ⚠ コピーしてはいけない

**`offset_` は「自分自身のアドレス」を基準にしています。**

```cpp
OffsetPtr<Vertex> a = header.vertices;   // ← コピーした瞬間、意味が変わる
a->x = 1.0f;                             // ← まったく別の場所を指す
```

**コピーコンストラクタを `delete` したくなりますが、できません。**

**理由:** `delete` すると自明にコピー可能でなくなり、**第42章の「暗黙の生存期間を持つ型」の条件を満たさなくなります。** `std::start_lifetime_as` が使えません。

**規約で守るしかありません。**

```cpp
// ○ 参照で使う
std::span<Vertex> verts = header->vertices.Get();

// × 値としてコピーする
OffsetSpan<Vertex> copy = header->vertices;    // 禁止
```

> **これは、この技法の本質的な弱点です。** 型では守れません。**コードレビューと命名(`OffsetPtr` という名前自体が警告)で防ぎます。**

### なぜ 32 ビットか

```cpp
std::int32_t offset_;
```

- **8 バイトではなく 4 バイト。** 大量に持つのでサイズが効きます
- ±2 GB を表現できる。**1つのアセットとしては十分**
- **符号付き。** 後ろから前を指すこともできます

---

## 47.5 ベイクする側

**ビルドツールで、メモリ上に最終形を組み立ててから書き出します。**

### 美しい点:実行時と同じアロケーターを使う

```cpp
void BakeMesh(const SourceMesh& src, const char* outPath)
{
    ga::Bump arena(256 * 1024 * 1024);

    // ① ヘッダを最初に確保する(ファイルの先頭になる)
    auto* header = arena.NewTrivial<MeshHeader>().value_or(nullptr);

    header->magic     = kMeshMagic;
    header->version   = kMeshVersion;
    header->alignment = kBakedAlignment;

    // ② 頂点を確保して埋める
    auto verts = arena.NewArrayUninit<Vertex>(src.vertices.size());
    std::ranges::copy(src.vertices, verts->begin());
    header->vertices.Set(*verts);

    // ③ インデックス
    auto indices = arena.NewArrayUninit<std::uint32_t>(src.indices.size());
    std::ranges::copy(src.indices, indices->begin());
    header->indices.Set(*indices);

    // ④ サブメッシュ
    auto subs = arena.NewArrayUninit<SubMesh>(src.subMeshes.size());
    for (std::size_t i = 0; i < subs->size(); ++i)
    {
        (*subs)[i] = Convert(src.subMeshes[i]);
    }
    header->subMeshes.Set(*subs);

    header->totalSize = static_cast<std::uint32_t>(arena.Used());

    // ⑤ 先頭から Used() バイトを、そのまま書き出す
    std::FILE* fp = nullptr;
    fopen_s(&fp, outPath, "wb");
    std::fwrite(arena.Base(), 1, arena.Used(), fp);
    std::fclose(fp);
}
```

**アリーナの中身が、そのままファイルになります。**

**第2章で作ったバンプアロケーターが、シリアライザとして働いています。**

- 連続して確保されるので、**隙間がない**
- アラインメントは `Allocate` が保証する
- `Used()` が、そのままファイルサイズ

**ベイクツールと実行時が、同じコードを共有できます。**

### ヘッダの定義

```cpp
// ga/BakedMesh.h(ベイクツールと実行時で共有する)
#pragma once

#include "ga/OffsetPtr.h"

namespace ga
{
    inline constexpr std::uint32_t kMeshMagic     = 0x4D41'4741;   // "AGAM"
    inline constexpr std::uint32_t kMeshVersion   = 3;
    inline constexpr std::size_t   kBakedAlignment = 64;

    struct Vertex
    {
        float x, y, z;
        float nx, ny, nz;
        float u, v;
    };

    struct SubMesh
    {
        std::uint32_t indexStart;
        std::uint32_t indexCount;
        std::uint32_t materialId;
        std::uint32_t padding;
    };

    struct MeshHeader
    {
        std::uint32_t magic;
        std::uint32_t version;
        std::uint32_t totalSize;
        std::uint32_t alignment;

        OffsetSpan<Vertex>        vertices;
        OffsetSpan<std::uint32_t> indices;
        OffsetSpan<SubMesh>       subMeshes;
    };

    // --- レイアウトを固定する ---
    static_assert(std::is_trivially_copyable_v<MeshHeader>);
    static_assert(std::is_trivially_destructible_v<MeshHeader>);
    static_assert(sizeof(Vertex)  == 32);
    static_assert(sizeof(SubMesh) == 16);
    static_assert(sizeof(MeshHeader) == 40);
    static_assert(offsetof(MeshHeader, vertices) == 16);
}
```

---

## 47.6 読み込む側 〔v0.35〕

```cpp
// ga/BakedLoader.h
namespace ga
{
    enum class LoadError
    {
        FileNotFound,
        ReadFailed,
        OutOfMemory,
        BadMagic,
        BadVersion,
        BadSize,
    };

    [[nodiscard]]
    std::expected<const MeshHeader*, LoadError>
    LoadBakedMesh(Bump& arena, const char* path)
    {
        const std::size_t fileSize = GetFileSize(path);
        if (fileSize < sizeof(MeshHeader)) { return std::unexpected(LoadError::BadSize); }

        // ★ ベイク時のアラインメントで確保する
        auto raw = arena.Allocate(fileSize, kBakedAlignment);
        if (!raw) { return std::unexpected(LoadError::OutOfMemory); }

        if (!ReadFileInto(path, *raw, fileSize))
        {
            return std::unexpected(LoadError::ReadFailed);
        }

        // ★ ここで「型として扱う」
        const MeshHeader* header = ga::StartLifetimeAs<MeshHeader>(*raw);

        // --- 検証 ---
        if (header->magic     != kMeshMagic)                 { return std::unexpected(LoadError::BadMagic); }
        if (header->version   != kMeshVersion)               { return std::unexpected(LoadError::BadVersion); }
        if (header->totalSize != fileSize)                   { return std::unexpected(LoadError::BadSize); }
        if (header->alignment != kBakedAlignment)            { return std::unexpected(LoadError::BadSize); }

        // 配列の範囲がファイル内に収まっているか
        if (!InRange(*raw, fileSize, header->vertices))      { return std::unexpected(LoadError::BadSize); }
        if (!InRange(*raw, fileSize, header->indices))       { return std::unexpected(LoadError::BadSize); }
        if (!InRange(*raw, fileSize, header->subMeshes))     { return std::unexpected(LoadError::BadSize); }

        return header;
    }
}
```

**パースがありません。** ファイルを読み、検証し、ポインタを返すだけです。

### 使う

```cpp
auto mesh = ga::LoadBakedMesh(sceneArena, "stage01.mesh");
if (!mesh) { /* エラー処理 */ }

std::span<const Vertex> verts = (*mesh)->vertices.Get();

std::println("頂点数     : {}", verts.size());
std::println("サブメッシュ: {}", (*mesh)->subMeshes.Size());

UploadToGpu(verts);
```

### ⚠ 検証は必須

**ファイルは、壊れているかもしれません。** 改竄されているかもしれません。

**オフセットが範囲外を指していたら、任意のメモリを読み書きすることになります。**

```cpp
    template <class T>
    bool InRange(const void* base, std::size_t size, const OffsetSpan<T>& span)
    {
        if (span.Size() == 0) { return true; }

        const auto* begin = reinterpret_cast<const std::byte*>(span.Get().data());
        const auto* start = static_cast<const std::byte*>(base);

        if (begin < start) { return false; }

        const std::size_t offset = static_cast<std::size_t>(begin - start);
        const std::size_t bytes  = span.Size() * sizeof(T);

        if (offset > size)          { return false; }
        if (bytes  > size - offset) { return false; }

        // アラインメントも確認する
        return (offset % alignof(T)) == 0;
    }
```

**「自分がベイクしたファイルだから安全」とは限りません。** ビルドの不整合、ディスクの破損、ダウンロードの失敗。**必ず検証してください。**

---

## 47.7 測る

**40 MB のメッシュ、頂点 100 万個。**

```
                       ロード時間   内訳                  ピークメモリ
従来のパース方式         147.7 ms   読38 / 解析62 / 構築47    112 MB
ベイク方式                40.6 ms   読38 / 検証 2.6           40 MB
```

**3.6 倍速く、メモリは 2.8 倍少ない。**

### 時間の内訳が示すこと

**ベイク方式の 40.6 ms のうち、38 ms はファイル読み込みです。**

```
処理時間 : 2.6 ms(6.4%)
I/O 時間 : 38.0 ms(93.6%)
```

**もう、CPU 側にできることはほとんどありません。**

**ボトルネックが完全に I/O に移りました。** ここから先を速くするには、

- ストレージを速くする(SSD、NVMe)
- **データを小さくする**(圧縮)
- **非同期に読む**(47.9 節)

**「アルゴリズムの改善では、もう速くならない」ところまで来た** ということです。

### ロード中のメモリ

```
従来:  ┌─────ファイル 40MB─────┐
       ┌──中間 32MB──┐
       ┌────実行時 40MB────┐     ← 3つが同時に存在

ベイク: ┌─────ファイル 40MB─────┐   ← これだけ
```

**据置機のように総メモリが固定の環境では、この差が決定的です。**

---

## 47.8 メモリマップトファイル

**さらに一歩進めると、読み込みすら省けます。**

```cpp
HANDLE file    = CreateFileW(path, GENERIC_READ, FILE_SHARE_READ, nullptr,
                             OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, nullptr);
HANDLE mapping = CreateFileMappingW(file, nullptr, PAGE_READONLY, 0, 0, nullptr);
void*  view    = MapViewOfFile(mapping, FILE_MAP_READ, 0, 0, 0);

const MeshHeader* header = ga::StartLifetimeAs<MeshHeader>(view);
```

**ファイルの中身が、そのままアドレス空間に現れます。**

### 何が起きているか

**第31章のコラムで触れた仕組みです。**

```
MapViewOfFile   → アドレス空間に「予約」するだけ。読み込みは発生しない
実際にアクセス  → ページフォルト → OS がその部分だけ読み込む
```

**「使った部分だけ、必要になったときに読まれる」** ——第29章のデマンドページングと同じ構造です。

### 測る

```
                          時間        物理メモリ
ReadFile(全部読む)      38.2 ms      40 MB
MapViewOfFile(マップのみ) 0.3 ms       0 MB
  + 全ページに触る       41.0 ms      40 MB
  + 半分だけ触る         20.8 ms      20 MB
```

**全部使うなら、時間はほぼ同じです。**

**一部しか使わないなら、劇的に有利です。**

### 利点

- **使わない部分は読み込まれない**
- **OS のページキャッシュを利用できる。** 2回目のロードは瞬時
- **複数プロセスで共有できる**(同じ物理ページを共有)
- コードが単純

### 欠点

**① ページフォルトのタイミングが制御できない。**

```cpp
for (const Vertex& v : verts)     // ← このループの途中でページフォルトが起きる
{
    Process(v);
}
```

**フレーム中に起きると、予測できない停止時間になります。**

**第2章から繰り返している「最悪値」の観点では、望ましくありません。**

**② 書き換えられない。**

`PAGE_READONLY` でマップした領域は読み取り専用です。書き換えたい場合は `FILE_MAP_COPY`(コピーオンライト)を使いますが、**書き換えたページは物理メモリを消費します。**

**③ アドレス空間を消費する。**

64 ビットなら問題になりません。

### ゲームでの使いどころ

| 状況 | 推奨 |
|---|---|
| **巨大なデータの一部だけ使う** | メモリマップ |
| **読み取り専用のアセット** | メモリマップ |
| 起動時に一度だけ読む | どちらでも |
| **フレーム中にアクセスする** | **`ReadFile` で先に読む** |
| 書き換える | `ReadFile` |

**「ロード中に全部触っておく(プリフェッチ)」という併用もあります。**

```cpp
// マップした後、全ページに触れて物理メモリに載せておく
volatile std::byte sink{};
for (std::size_t i = 0; i < fileSize; i += 4096)
{
    sink = static_cast<const std::byte*>(view)[i];
}
```

**マップの単純さと、`ReadFile` の予測可能性を、両方得られます。**

---

## 47.9 ストリーミング

**オープンワールドでは、世界を分割して、必要な部分だけを読み込みます。**

### 構造

```
世界をセルに分割
    ↓
プレイヤーの位置に応じて、
  近いセル → ロード
  遠いセル → アンロード
```

**第44章のシーンアリーナを、セル単位にしたものです。**

```cpp
class CellStreamer
{
public:
    struct Cell
    {
        ga::SceneArena     arena;
        const MeshHeader*  mesh = nullptr;
        int                x = 0, y = 0;
        bool               loaded = false;
    };

    void Update(int playerCellX, int playerCellY)
    {
        // 範囲外のセルをアンロード
        for (auto& cell : cells_)
        {
            if (cell.loaded && Distance(cell, playerCellX, playerCellY) > kUnloadRadius)
            {
                cell.arena.Unload();       // ← 第44章
                cell.loaded = false;
            }
        }

        // 範囲内のセルをロード(1フレームに1つまで)
        for (auto& cell : cells_)
        {
            if (!cell.loaded && Distance(cell, playerCellX, playerCellY) <= kLoadRadius)
            {
                RequestLoad(cell);
                break;
            }
        }
    }
};
```

### 非同期に読む

**フレーム中に `ReadFile` を呼べば、そのフレームは止まります。**

```cpp
// 非同期 I/O(重複 I/O)
OVERLAPPED ov{};
ov.hEvent = CreateEventW(nullptr, TRUE, FALSE, nullptr);

ReadFile(file, buffer, size, nullptr, &ov);    // すぐ戻る

// ... 別のフレーム処理を続ける ...

// 完了を確認する
DWORD transferred = 0;
if (GetOverlappedResult(file, &ov, &transferred, FALSE))
{
    // 読み込み完了
}
```

**ベイク方式との相性が抜群です。**

```
読み込み完了 → 検証(2.6 ms)→ 使える
```

**パースが要らないので、完了後の処理がほぼゼロです。** 従来方式なら、読み込み後に 109 ms の CPU 処理が待っています。

> **近年は、より高速なストレージ API も提供されています。** SSD の性能を引き出し、GPU へ直接転送する仕組みも登場しています。**どの API を使うにせよ、「読んだらそのまま使える」形にしておくことが前提になります。**

### 第46章との関係

**セルを何度もロード・アンロードすると、断片化が起きます。**

**対策は2つ。**

**① セルごとにアリーナを分ける。** 第44章の方式です。セル数分のアリーナが必要ですが、断片化しません。

**② コンパクションを使う。** 第46章の方式です。1つのアリーナで済みますが、ハンドルとピン留めが必要です。

**セルのサイズが揃っているなら①、ばらばらなら②** が目安になります。

---

## 47.10 落とし穴

**この技法には、明確な危険があります。漏れなく挙げます。**

### ① 構造体のレイアウトは保証されない

```cpp
static_assert(sizeof(MeshHeader) == 40);
static_assert(offsetof(MeshHeader, vertices) == 16);
static_assert(offsetof(MeshHeader, indices)  == 24);
```

**必ず `static_assert` で固定してください。**

**コンパイラのバージョンを上げたとき、ここで止まります。** 止まらずに動いて、実行時に壊れるより、はるかにましです。

### ② `#pragma pack` を使わない

```cpp
#pragma pack(push, 1)
struct Bad { std::uint8_t a; std::uint64_t b; };    // ← 危険
#pragma pack(pop)
```

**サイズは縮みますが、`b` が 8 バイト境界に乗りません。**

**第6章で見たとおり、非アラインアクセスは遅く、SIMD では例外になります。**

**パディングを手で入れてください。**

```cpp
struct Good
{
    std::uint8_t  a;
    std::uint8_t  pad[7];
    std::uint64_t b;
};
```

### ③ 仮想関数を持つ型は載せられない

```cpp
struct Bad { virtual void Update(); };
```

**vtable ポインタが入ります。** アドレスは実行のたびに変わります。

```cpp
static_assert(!std::is_polymorphic_v<T>);
```

### ④ 標準ライブラリの型は載せられない

```cpp
struct Bad
{
    std::string        name;      // 内部にポインタ
    std::vector<int>   data;      // 内部にポインタ
};
```

**文字列は、`OffsetSpan<char>` で持ちます。**

```cpp
struct Good
{
    OffsetSpan<char> name;
};
```

### ⑤ エンディアン

**同じアーキテクチャなら問題ありません。** 異なるプラットフォーム向けにベイクする場合は、変換が必要です。

**マジックナンバーで検出できます。**

```cpp
if (header->magic == ByteSwap(kMeshMagic))
{
    // エンディアンが逆
}
```

### ⑥ バージョン管理

**フォーマットを変えたら、必ずバージョンを上げてください。**

```cpp
if (header->version != kMeshVersion)
{
    // 古いファイル → 再ベイクを促す、または変換する
}
```

**「ビルドはできたが、古いアセットで起動して壊れる」という事故が、最も多い。**

### まとめ:守るための `static_assert`

```cpp
template <class T>
constexpr bool IsBakeable()
{
    return std::is_trivially_copyable_v<T>
        && std::is_trivially_destructible_v<T>
        && !std::is_polymorphic_v<T>;
}

static_assert(IsBakeable<MeshHeader>());
static_assert(IsBakeable<Vertex>());
static_assert(IsBakeable<SubMesh>());
```

**これを、ベイクするすべての型に付けてください。**

---

## 演習

**演習47-1** `OffsetPtr` をコピーして、壊れることを確認してください。第31章のガードページで検出できますか。

**演習47-2** ベイクしたファイルのオフセットを手で書き換え、範囲外を指させてください。`InRange` の検証は働きますか。

**演習47-3** `kBakedAlignment` を 64 から 4 に変え、`alignas(32)` のメンバを持つ型をベイクしてください。何が起きますか。

**演習47-4** メモリマップ方式で、ファイルの一部だけにアクセスしてください。物理メモリの使用量を確認してください。

**演習47-5** 非同期読み込みを実装し、読み込み中も 60 fps を維持できるか確認してください。

**演習47-6** 文字列を `OffsetSpan<char>` で持つ形にベイクしてください。終端はどう扱いますか。

**演習47-7** `#pragma pack(1)` でベイクし、非アラインアクセスの速度を測ってください(第6章の測定を再利用します)。

**演習47-8** ベイクしたファイルを圧縮し、展開しながら読み込んでください。時間とメモリはどうなりますか。

---

## 章末チェックリスト

- [ ] 従来のロード処理の内訳を測り、無駄を特定した
- [ ] ベイクという発想を説明できる
- [ ] `OffsetPtr` / `OffsetSpan` を実装した
- [ ] **オフセットポインタをコピーしてはいけない** 理由と、型で守れない理由を説明できる
- [ ] バンプアロケーターがシリアライザとして働くことを理解した
- [ ] `std::start_lifetime_as` を実用の形で使った 〔v0.35〕
- [ ] **範囲の検証が必須** である理由を説明できる
- [ ] ロード時間が I/O に律速されるところまで来たことを確認した
- [ ] メモリマップトファイルの利点と、ページフォルトの制御不能性を説明できる
- [ ] 6つの落とし穴を挙げ、`static_assert` で守った

---

## 次章の予告

**第7部も終盤です。次は GPU メモリを扱います。**

ここまで扱ってきたのは、**CPU が読み書きするメモリ** でした。GPU 側のメモリには、まったく違う制約があります。

- **CPU から自由に読み書きできない。** ヘッダを書き込めない(第26章でバディを選んだ理由)
- **アラインメントの要求が大きい。** 64 KB、4 MB といった単位
- **解放のタイミングが遅れる。** GPU がまだ使っているかもしれない(第43章のダブルバッファと同じ問題)
- **転送が必要。** CPU からアップロードし、GPU が読める場所へ

第48章では、これらに対応したサブアロケーターを設計します。**第26章のバディ、第43章のフレーム、第45章のハンドルが、すべて動員されます。**

そして、D3D12MA や VMA といった既存ライブラリの立ち位置を整理します。

---

> **コラム:ゼロコピーという思想**
>
> **「読み込んだらそのまま使える」という発想は、ゲーム開発だけのものではありません。**
>
> ---
>
> **シリアライズ形式の進化**
>
> データを保存・送信する形式は、長い間「テキストか、独自バイナリ」でした。
>
> **XML、JSON。** 人間が読めますが、**パースが重い。** 数十 MB のデータを読むと、パースだけで数百ミリ秒かかります。
>
> **Protocol Buffers など。** バイナリで小さくなりましたが、**やはりパースが必要です。** バイト列から、メモリ上のオブジェクトへ変換します。
>
> ---
>
> **ゼロコピー形式の登場**
>
> **FlatBuffers**(Google)と **Cap'n Proto** は、この前提を覆しました。
>
> > **パースしない。バイト列を、そのままアクセスする。**
>
> **この章でやったことと、まったく同じ発想です。**
>
> - 相対オフセットで参照を持つ
> - 読み込んだメモリを、そのまま構造体として扱う
> - **アクセスするまで、何のコストもかからない**
>
> FlatBuffers は、実際にゲーム開発の文脈から生まれました。**「モバイルゲームで、大量のデータを高速にロードしたい」** という要求が出発点です。
>
> ---
>
> **違い:安全性と汎用性**
>
> **これらのライブラリと、この章で作ったものの違いは、安全性への配慮です。**
>
> - **スキーマ言語を持つ。** 構造をコードとは別に定義し、コードを自動生成する
> - **バージョン間の互換性を扱う。** フィールドを追加しても、古いデータが読める
> - **アクセスのたびに範囲を検証する**(オプション)
> - **プラットフォーム間の差異を吸収する**
>
> **私たちの実装は、これらを持ちません。** 47.10 節で挙げた落とし穴を、すべて手で守る必要があります。
>
> ---
>
> **では、なぜ自作するのか**
>
> **速度と制御です。**
>
> FlatBuffers のアクセスは、**フィールドごとに間接参照** が入ります。安全性と柔軟性のための設計です。
>
> **私たちは、構造体を直接触ります。** `verts[i].x` は、単なるメモリアクセスです。**第32章で見たとおり、この差が走査速度を決めます。**
>
> **そして、アラインメントを完全に制御できます。** SIMD で読むために 16 バイト境界に揃える、キャッシュラインを意識して配置する——**汎用のシリアライズ形式では、ここまでの制御は困難です。**
>
> ---
>
> **判断**
>
> | 状況 | 推奨 |
> |---|---|
> | 頂点データ、テクスチャ、アニメーション | **自作のベイク形式** |
> | 設定ファイル、セーブデータ | 汎用形式(JSON、FlatBuffers) |
> | ネットワーク通信 | 汎用形式 |
> | 外部ツールと連携する | 汎用形式 |
>
> **「性能が critical で、自分たちだけが読む」データにだけ、自作の形式を使う。**
>
> **これは、この本を通しての判断基準と同じです。** 第53章で「いつ自作すべきか」を扱うとき、同じ問いに戻ってきます。
