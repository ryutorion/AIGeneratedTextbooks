# 第20章 テクスチャ

**本書で最も手強い章です。**

第16章でバッファを DEFAULT ヒープへ転送したとき、`CopyBufferRegion` の 1 行で済みました。**テクスチャはそうはいきません。**

理由は、前章 19.2.1 節で触れた 1 行にあります。

```cpp
desc.Layout = D3D12_TEXTURE_LAYOUT_UNKNOWN;   // テクスチャ
```

**GPU は、テクスチャのピクセルを独自の順序で並べます。** 2 次元のアクセス効率を上げるためのもので、その並べ方はハードウェアごとに異なり、非公開です。だから `Map` して直接書けません。

代わりに、**「行ごとに整列された中間バッファ」を経由します。** そして、その整列規則をこちらが計算する必要があります。`d3dx12.h` の `UpdateSubresources()` は、まさにこの計算を隠していた関数です。

**本章では、それを自分で書きます。**

**本章のゴール**
DDS ローダを自作し、テクスチャを DEFAULT ヒープへ転送し、SRV とサンプラーを設定して、テクスチャ付きの立方体を表示する。

---

## 20.1 DDS ファイルの構造

### 20.1.1 なぜ DDS なのか

第1章 1.3 節の線引きにより、本書は WIC(Windows Imaging Component)を使いません。したがって PNG や JPEG は読めません。**自分でデコーダを書くのは、本書の主題から外れすぎます。**

**DDS(DirectDraw Surface)は、そのために存在するような形式です。**

| 特徴 | 意味 |
|---|---|
| **ヘッダの後に生データが並ぶだけ** | デコード処理が不要 |
| **ミップマップを含められる** | 実行時に生成しなくてよい |
| **圧縮フォーマットをそのまま格納できる** | GPU が直接読める形で保持できる |
| **キューブマップ・3D テクスチャに対応** | 将来の拡張に使える |

**とくに 3 番目が重要です。** BC1〜BC7 といったブロック圧縮フォーマットは、**GPU がハードウェアで展開します。** PNG を読んで展開すると非圧縮の RGBA になりますが、DDS なら圧縮したまま VRAM に置け、容量が 4 分の 1 から 8 分の 1 になります。

DDS ファイルは、Microsoft の **DirectXTex** に含まれる `texconv` ツールで作れます。

```
texconv -f BC7_UNORM_SRGB -m 0 input.png
```

> **本章は非圧縮フォーマットから始めます**
>
> ブロック圧縮を扱うと、「1 行のバイト数」の計算が複雑になります(20.4.2 節)。まず `R8G8B8A8_UNORM` で仕組みを作り、20.7 節で圧縮に対応させます。

### 20.1.2 ヘッダの構造

DDS ファイルは、次の順に並んでいます。

```
┌────────────────────────────┐
│ マジックナンバー "DDS "     │  4 バイト
├────────────────────────────┤
│ DDS_HEADER                 │  124 バイト
├────────────────────────────┤
│ DDS_HEADER_DXT10(任意)    │  20 バイト
├────────────────────────────┤
│ ピクセルデータ              │  残り全部
└────────────────────────────┘
```

**構造体を自分で定義します。**

```cpp
// src/Graphics/DdsLoader.cpp

namespace
{
    constexpr std::uint32_t kDdsMagic = 0x20534444;   // "DDS " (リトルエンディアン)

    struct DdsPixelFormat
    {
        std::uint32_t size;
        std::uint32_t flags;
        std::uint32_t fourCC;
        std::uint32_t rgbBitCount;
        std::uint32_t rBitMask;
        std::uint32_t gBitMask;
        std::uint32_t bBitMask;
        std::uint32_t aBitMask;
    };
    static_assert(sizeof(DdsPixelFormat) == 32);

    struct DdsHeader
    {
        std::uint32_t  size;              // 常に 124
        std::uint32_t  flags;
        std::uint32_t  height;
        std::uint32_t  width;
        std::uint32_t  pitchOrLinearSize;
        std::uint32_t  depth;
        std::uint32_t  mipMapCount;
        std::uint32_t  reserved1[11];
        DdsPixelFormat pixelFormat;
        std::uint32_t  caps;
        std::uint32_t  caps2;
        std::uint32_t  caps3;
        std::uint32_t  caps4;
        std::uint32_t  reserved2;
    };
    static_assert(sizeof(DdsHeader) == 124);

    struct DdsHeaderDxt10
    {
        std::uint32_t dxgiFormat;
        std::uint32_t resourceDimension;
        std::uint32_t miscFlag;
        std::uint32_t arraySize;
        std::uint32_t miscFlags2;
    };
    static_assert(sizeof(DdsHeaderDxt10) == 20);
}
```

**`static_assert` でサイズを確認しています。** パディングが入ると、ファイルの読み取り位置が丸ごとずれます。**第15章 15.1.1 節で頂点構造体に入れたのと同じ発想です。**

### 20.1.3 DXT10 拡張を優先する

**DDS には歴史的な事情があります。**

古い DDS では、フォーマットを `DdsPixelFormat` のビットマスクで表現していました。「R は下位 8bit、G は次の 8bit……」という記述です。**これを DXGI_FORMAT に対応づけるのは、驚くほど面倒です。**

そこで後に追加されたのが `DDS_HEADER_DXT10` です。**DXGI_FORMAT の値がそのまま書かれています。**

```cpp
if (header.pixelFormat.flags & 0x4 /* DDPF_FOURCC */ &&
    header.pixelFormat.fourCC == MakeFourCC('D','X','1','0'))
{
    // DXT10 ヘッダがある → dxgiFormat をそのまま使える
}
```

**本書は DXT10 拡張つきの DDS のみを扱います。**

`texconv` に `-dx10` オプションを付ければ、必ずこの形式になります。

```
texconv -f R8G8B8A8_UNORM -m 0 -dx10 input.png
```

**古い形式への対応は、本書の主題ではありません。** ビットマスクの解釈は、DirectXTex の `DDSTextureLoader.cpp` に 100 行以上のテーブルとして書かれています。必要になったらそちらを参照してください。

### 20.1.4 ローダを書く

```cpp
// src/Graphics/DdsLoader.h
#pragma once
#include "std_import.h"
#include "Core/Error.h"

namespace Graphics
{
    //-----------------------------------------------------------
    // 1 つのミップレベルの情報
    //-----------------------------------------------------------
    struct SubresourceData
    {
        const std::byte* data     = nullptr;
        UINT             width    = 0;
        UINT             height   = 0;
        UINT64           rowPitch = 0;   // 1 行のバイト数
        UINT64           slicePitch = 0; // 1 枚のバイト数
    };

    //-----------------------------------------------------------
    // 読み込んだ DDS。
    // pixels がデータを所有し、subresources はそこを指す。
    //-----------------------------------------------------------
    struct DdsImage
    {
        DXGI_FORMAT format = DXGI_FORMAT_UNKNOWN;
        UINT        width  = 0;
        UINT        height = 0;
        UINT        mipLevels = 1;
        UINT        arraySize = 1;

        std::vector<std::byte>       pixels;
        std::vector<SubresourceData> subresources;
    };

    Core::Result<DdsImage> LoadDds(const std::filesystem::path& path);
}
```

読み込みの本体です。

```cpp
Core::Result<DdsImage> LoadDds(const std::filesystem::path& path)
{
    //--- ファイル全体を読む ---
    std::ifstream file(path, std::ios::binary | std::ios::ate);
    if (!file)
    {
        LOG_ERROR(L"dds not found: {}", path.wstring());
        return std::unexpected(Core::MakeError(
            HRESULT_FROM_WIN32(ERROR_FILE_NOT_FOUND), L"LoadDds"));
    }

    const auto fileSize = static_cast<std::size_t>(file.tellg());
    file.seekg(0);

    std::vector<std::byte> raw(fileSize);
    file.read(reinterpret_cast<char*>(raw.data()),
              static_cast<std::streamsize>(fileSize));

    //--- 最小サイズの検査 ---
    constexpr std::size_t kMinSize = 4 + sizeof(DdsHeader);
    if (fileSize < kMinSize)
    {
        return std::unexpected(Core::MakeError(E_FAIL, L"dds too small"));
    }

    //--- マジックナンバー ---
    std::uint32_t magic = 0;
    std::memcpy(&magic, raw.data(), 4);
    if (magic != kDdsMagic)
    {
        return std::unexpected(Core::MakeError(E_FAIL, L"not a DDS file"));
    }

    //--- ヘッダ ---
    DdsHeader header{};
    std::memcpy(&header, raw.data() + 4, sizeof(header));

    if (header.size != 124 || header.pixelFormat.size != 32)
    {
        return std::unexpected(Core::MakeError(E_FAIL, L"invalid DDS header"));
    }

    //--- DXT10 拡張(20.1.3 の通り、必須とする)---
    constexpr std::uint32_t kDdpfFourCC = 0x4;
    const bool hasDxt10 =
        (header.pixelFormat.flags & kDdpfFourCC) &&
        (header.pixelFormat.fourCC == MakeFourCC('D','X','1','0'));

    if (!hasDxt10)
    {
        LOG_ERROR(L"legacy DDS is not supported. "
                  L"Convert with: texconv -dx10 ...");
        return std::unexpected(Core::MakeError(
            E_NOTIMPL, L"legacy DDS format"));
    }

    DdsHeaderDxt10 dxt10{};
    std::memcpy(&dxt10, raw.data() + 4 + sizeof(DdsHeader), sizeof(dxt10));

    const std::size_t dataOffset =
        4 + sizeof(DdsHeader) + sizeof(DdsHeaderDxt10);

    //--- 結果を組み立てる ---
    DdsImage image{};
    image.format    = static_cast<DXGI_FORMAT>(dxt10.dxgiFormat);
    image.width     = header.width;
    image.height    = header.height;
    image.mipLevels = (header.mipMapCount > 0) ? header.mipMapCount : 1;
    image.arraySize = (dxt10.arraySize > 0) ? dxt10.arraySize : 1;

    image.pixels.assign(raw.begin() + dataOffset, raw.end());

    //--- 各ミップレベルの位置を求める ---
    if (auto r = BuildSubresources(image); !r)
    {
        return std::unexpected(r.error());
    }

    LOG_INFO(L"dds loaded: {} ({}x{}, {} mips, format {})",
             path.filename().wstring(),
             image.width, image.height, image.mipLevels,
             static_cast<int>(image.format));

    return image;
}
```

### 20.1.5 ミップレベルの位置を計算する

**ピクセルデータは、ミップレベルが順に詰まっているだけです。**

```
[mip 0: 256×256][mip 1: 128×128][mip 2: 64×64]...[mip 8: 1×1]
```

**各レベルの開始位置を求めるには、前のレベルのサイズを積み上げます。**

```cpp
Core::Status BuildSubresources(DdsImage& image)
{
    image.subresources.clear();
    image.subresources.reserve(image.mipLevels);

    std::size_t offset = 0;
    UINT width  = image.width;
    UINT height = image.height;

    for (UINT mip = 0; mip < image.mipLevels; ++mip)
    {
        UINT64 rowPitch   = 0;
        UINT64 slicePitch = 0;

        if (auto r = ComputePitch(image.format, width, height,
                                  rowPitch, slicePitch); !r)
        {
            return r;
        }

        if (offset + slicePitch > image.pixels.size())
        {
            LOG_ERROR(L"dds data truncated at mip {}", mip);
            return std::unexpected(Core::MakeError(E_FAIL, L"dds truncated"));
        }

        SubresourceData sub{};
        sub.data       = image.pixels.data() + offset;
        sub.width      = width;
        sub.height     = height;
        sub.rowPitch   = rowPitch;
        sub.slicePitch = slicePitch;
        image.subresources.push_back(sub);

        offset += static_cast<std::size_t>(slicePitch);

        // 次のレベルは半分。ただし 1 未満にはならない
        width  = std::max(width  / 2u, 1u);
        height = std::max(height / 2u, 1u);
    }

    return {};
}
```

**`std::max(width / 2, 1)` が要点です。** 幅が 1 になった後も、高さがまだ大きい場合があります(横長のテクスチャなど)。**片方が 1 になっても、もう片方が 1 になるまでミップは続きます。**

`ComputePitch` は、フォーマットごとに「1 行のバイト数」を求める関数です。**非圧縮なら単純ですが、圧縮フォーマットでは事情が変わります**(20.7 節)。

```cpp
Core::Status ComputePitch(DXGI_FORMAT format, UINT width, UINT height,
                          UINT64& rowPitch, UINT64& slicePitch)
{
    const UINT bpp = BitsPerPixel(format);
    if (bpp == 0)
    {
        LOG_ERROR(L"unsupported format: {}", static_cast<int>(format));
        return std::unexpected(Core::MakeError(E_NOTIMPL, L"unsupported format"));
    }

    rowPitch   = (static_cast<UINT64>(width) * bpp + 7) / 8;
    slicePitch = rowPitch * height;
    return {};
}
```

---

## 20.2 テクスチャリソースを作る

前章で作った `MakeTexture2DDesc` を、そのまま使えます。

```cpp
const auto desc = MakeTexture2DDesc(
    image.format,
    image.width,
    image.height,
    static_cast<UINT16>(image.mipLevels),
    static_cast<UINT16>(image.arraySize),
    D3D12_RESOURCE_FLAG_NONE);          // 深度バッファと違いフラグ不要

const auto heapProps = MakeHeapProperties(D3D12_HEAP_TYPE_DEFAULT);

ComPtr<ID3D12Resource> texture;
HR_TRY(device->CreateCommittedResource(
    &heapProps,
    D3D12_HEAP_FLAG_NONE,
    &desc,
    D3D12_RESOURCE_STATE_COPY_DEST,     // 第16章と同じ
    nullptr,                            // クリア値は不要
    IID_PPV_ARGS(&texture)));

Core::SetDebugName(texture.Get(), name);
```

**第19章の深度バッファとの違いは 2 つだけです。**

| | 深度バッファ | テクスチャ |
|---|---|---|
| `Flags` | `ALLOW_DEPTH_STENCIL` | `NONE` |
| クリア値 | 指定する | `nullptr` |
| 初期状態 | `DEPTH_WRITE` | `COPY_DEST` |

**前章で作ったヘルパーが、そのまま流用できました。**

---

## 20.3 サブリソースとは何か

### 20.3.1 テクスチャは複数の「面」を持つ

**バッファは 1 つの連続領域でした。テクスチャは違います。**

```
2D テクスチャ(mip 3 段、配列 2 枚)

配列 0:  [mip 0][mip 1][mip 2]     ← サブリソース 0, 1, 2
配列 1:  [mip 0][mip 1][mip 2]     ← サブリソース 3, 4, 5
```

**それぞれが独立した「サブリソース」です。** 転送も、状態遷移も、サブリソース単位で行えます。

**サブリソースの番号は、次の式で決まります。**

```cpp
subresourceIndex = mipSlice + arraySlice * mipLevels;
```

第11章 11.5.2 節で、リソースバリアに `D3D12_RESOURCE_BARRIER_ALL_SUBRESOURCES` を渡していたのは、**「全部まとめて遷移させる」**という指定でした。ここでその意味がはっきりします。

### 20.3.2 転送は「サブリソースごと」

ミップマップが 9 段あるテクスチャなら、**9 回のコピーが必要です。**

そして、**それぞれについて中間バッファ内の配置を計算する必要があります。**

---

## 20.4 サブリソース転送 —— 本章の山場

### 20.4.1 2 つのアラインメント

**中間バッファの配置には、2 つの制約があります。**

```cpp
D3D12_TEXTURE_DATA_PITCH_ALIGNMENT      // = 256
D3D12_TEXTURE_DATA_PLACEMENT_ALIGNMENT  // = 512
```

| 定数 | 対象 | 値 |
|---|---|---|
| `PITCH_ALIGNMENT` | **1 行のバイト数** | **256** |
| `PLACEMENT_ALIGNMENT` | **各サブリソースの開始位置** | **512** |

**具体例で見ます。** 256×256 の `R8G8B8A8_UNORM` テクスチャの場合。

```
元データの 1 行:  256 ピクセル × 4 バイト = 1024 バイト
                  → 1024 は 256 の倍数なので、そのまま
```

**問題は、ミップレベルが下がったときです。**

```
mip 6 (4×4):   1 行 = 4 × 4 = 16 バイト
               → 256 に切り上げ = 256 バイト

  元データ:     [16B]
  中間バッファ: [16B][240B のパディング]
```

**16 バイトのデータのために 256 バイト使います。** 無駄に見えますが、これはハードウェアの都合であり、避けられません。

**行ごとにパディングが入るということは、`memcpy` を 1 回で済ませられない**ということです。

```cpp
// ❌ できない
std::memcpy(mapped, sourceData, totalSize);

// ✅ 行ごとにコピーする
for (UINT row = 0; row < height; ++row)
{
    std::memcpy(dest + row * alignedRowPitch,
                src  + row * sourceRowPitch,
                sourceRowPitch);
}
```

**これが `UpdateSubresources()` がやっていたことの中心です。**

### 20.4.2 `GetCopyableFootprints`

**アラインメントの計算を自分でやる必要はありません。** D3D12 が計算してくれます。

```cpp
HRESULT GetCopyableFootprints(
    const D3D12_RESOURCE_DESC*          pResourceDesc,
    UINT                                FirstSubresource,
    UINT                                NumSubresources,
    UINT64                              BaseOffset,
    D3D12_PLACED_SUBRESOURCE_FOOTPRINT* pLayouts,     // 出力
    UINT*                               pNumRows,     // 出力
    UINT64*                             pRowSizeInBytes, // 出力
    UINT64*                             pTotalBytes);    // 出力
```

**返ってくる情報の意味を、正確に押さえてください。**

```cpp
struct D3D12_PLACED_SUBRESOURCE_FOOTPRINT
{
    UINT64                Offset;      // 中間バッファ内の開始位置(512 整列済み)
    D3D12_SUBRESOURCE_FOOTPRINT Footprint;
};

struct D3D12_SUBRESOURCE_FOOTPRINT
{
    DXGI_FORMAT Format;
    UINT        Width;
    UINT        Height;
    UINT        Depth;
    UINT        RowPitch;   // ← 256 に整列された 1 行のバイト数
};
```

| 出力 | 意味 |
|---|---|
| `pLayouts[i].Offset` | 中間バッファ内での開始位置 |
| `pLayouts[i].Footprint.RowPitch` | **整列後**の 1 行のバイト数 |
| `pNumRows[i]` | **コピーすべき行数** |
| `pRowSizeInBytes[i]` | **実データ**の 1 行のバイト数(整列前) |
| `pTotalBytes` | 中間バッファ全体に必要なサイズ |

**`RowPitch` と `pRowSizeInBytes` の違いが決定的です。**

```
RowPitch        = 256   ← コピー先で 1 行進むバイト数
pRowSizeInBytes =  16   ← 実際にコピーするバイト数
```

**この 2 つを取り違えると、絵が斜めにずれるか、メモリを破壊します。**

> **`pNumRows` は「高さ」とは限らない**
>
> 非圧縮テクスチャなら `pNumRows[i] == Height` です。
>
> **ブロック圧縮では違います。** BC フォーマットは 4×4 ピクセルを 1 ブロックとして扱うため、**行数はブロック単位**になります。256×256 の BC7 なら `pNumRows = 64` です。
>
> **だから `Height` ではなく `pNumRows` を使ってください。** これで 20.7 節の圧縮対応が、ほぼ自動的に成立します。

### 20.4.3 転送を実装する

```cpp
Core::Result<ComPtr<ID3D12Resource>> ResourceUploader::UploadTexture(
    const DdsImage& image, std::wstring_view name)
{
    D3D_ASSERT_MSG(m_recording, L"Begin() を先に呼ぶこと");

    const UINT subresourceCount = image.mipLevels * image.arraySize;

    //--- ① 転送先のテクスチャ ---
    const auto desc = MakeTexture2DDesc(
        image.format, image.width, image.height,
        static_cast<UINT16>(image.mipLevels),
        static_cast<UINT16>(image.arraySize));

    const auto defaultHeap = MakeHeapProperties(D3D12_HEAP_TYPE_DEFAULT);

    ComPtr<ID3D12Resource> texture;
    HR_TRY(m_device->CreateCommittedResource(
        &defaultHeap, D3D12_HEAP_FLAG_NONE, &desc,
        D3D12_RESOURCE_STATE_COPY_DEST, nullptr,
        IID_PPV_ARGS(&texture)));

    Core::SetDebugName(texture.Get(), name);

    //--- ② 配置情報を問い合わせる(20.4.2)---
    std::vector<D3D12_PLACED_SUBRESOURCE_FOOTPRINT> layouts(subresourceCount);
    std::vector<UINT>   numRows(subresourceCount);
    std::vector<UINT64> rowSizes(subresourceCount);
    UINT64 totalBytes = 0;

    m_device->GetCopyableFootprints(
        &desc, 0, subresourceCount, 0,
        layouts.data(), numRows.data(), rowSizes.data(), &totalBytes);

    //--- ③ 中間バッファ ---
    const auto uploadHeap  = MakeHeapProperties(D3D12_HEAP_TYPE_UPLOAD);
    const auto stagingDesc = MakeBufferDesc(totalBytes);

    ComPtr<ID3D12Resource> staging;
    HR_TRY(m_device->CreateCommittedResource(
        &uploadHeap, D3D12_HEAP_FLAG_NONE, &stagingDesc,
        D3D12_RESOURCE_STATE_GENERIC_READ, nullptr,
        IID_PPV_ARGS(&staging)));

    Core::SetDebugNameF(staging.Get(), L"Staging({})", name);

    //--- ④ 行ごとにコピーする ---
    std::byte* mapped = nullptr;
    const D3D12_RANGE readRange{ 0, 0 };
    HR_TRY(staging->Map(0, &readRange, reinterpret_cast<void**>(&mapped)));

    for (UINT i = 0; i < subresourceCount; ++i)
    {
        const auto& src    = image.subresources[i];
        const auto& layout = layouts[i];

        std::byte* dest = mapped + layout.Offset;

        for (UINT row = 0; row < numRows[i]; ++row)
        {
            std::memcpy(
                dest + row * layout.Footprint.RowPitch,  // 整列後のピッチ
                src.data + row * src.rowPitch,           // 元データのピッチ
                static_cast<std::size_t>(rowSizes[i]));  // 実データのサイズ
        }
    }

    staging->Unmap(0, nullptr);

    //--- ⑤ サブリソースごとにコピーを記録 ---
    for (UINT i = 0; i < subresourceCount; ++i)
    {
        D3D12_TEXTURE_COPY_LOCATION dst{};
        dst.pResource        = texture.Get();
        dst.Type             = D3D12_TEXTURE_COPY_TYPE_SUBRESOURCE_INDEX;
        dst.SubresourceIndex = i;

        D3D12_TEXTURE_COPY_LOCATION src{};
        src.pResource       = staging.Get();
        src.Type            = D3D12_TEXTURE_COPY_TYPE_PLACED_FOOTPRINT;
        src.PlacedFootprint = layouts[i];

        m_commandList->CopyTextureRegion(&dst, 0, 0, 0, &src, nullptr);
    }

    //--- ⑥ シェーダーから読める状態へ ---
    const auto barrier = MakeTransitionBarrier(
        texture.Get(),
        D3D12_RESOURCE_STATE_COPY_DEST,
        D3D12_RESOURCE_STATE_PIXEL_SHADER_RESOURCE);
    m_commandList->ResourceBarrier(1, &barrier);

    //--- 中間バッファを End() まで保持(第16章 16.3.4)---
    m_stagingBuffers.push_back(std::move(staging));

    return texture;
}
```

**④ の三重ループが、この章の核心です。**

```cpp
dest + row * layout.Footprint.RowPitch    // 進む幅は整列後
src.data + row * src.rowPitch             // 読む幅は元データ
rowSizes[i]                               // コピーする量は実データ
```

**3 つの値がすべて違います。** ここを取り違えるのが、テクスチャ転送で最も多い失敗です。

**⑤ の `D3D12_TEXTURE_COPY_LOCATION` にも注意が必要です。**

| | `Type` | 使うメンバ |
|---|---|---|
| コピー先(テクスチャ) | `SUBRESOURCE_INDEX` | `SubresourceIndex` |
| コピー元(バッファ) | `PLACED_FOOTPRINT` | `PlacedFootprint` |

**共用体なので、`Type` と一致するメンバだけが有効です。** 第11章のリソースバリアと同じ形です。

---

## 20.5 SRV とサンプラー

### 20.5.1 SRV を作る

**第18章で作った CBV_SRV_UAV ヒープに、SRV を追加します。**

```cpp
D3D12_SHADER_RESOURCE_VIEW_DESC srvDesc{};
srvDesc.Format                    = image.format;
srvDesc.ViewDimension             = D3D12_SRV_DIMENSION_TEXTURE2D;
srvDesc.Shader4ComponentMapping   = D3D12_DEFAULT_SHADER_4_COMPONENT_MAPPING;
srvDesc.Texture2D.MostDetailedMip = 0;
srvDesc.Texture2D.MipLevels       = image.mipLevels;   // 全部使う
srvDesc.Texture2D.PlaneSlice      = 0;
srvDesc.Texture2D.ResourceMinLODClamp = 0.0f;

device->CreateShaderResourceView(texture.Get(), &srvDesc, cpuHandle);
```

**`Shader4ComponentMapping` を忘れないでください。**

これはチャンネルの並べ替えを指定するもので、**ゼロにすると全チャンネルが赤成分になります。** `D3D12_DEFAULT_SHADER_4_COMPONENT_MAPPING` が「そのまま」を意味します。

**第14章 14.4.3 節で見た「ゼロが有効な値として通ってしまう」罠と同じ形です。** エラーは出ず、絵が赤くなるだけです。

**ヒープの割り当てを整理します。** 第18章では 3 個(フレーム数)でしたが、SRV が加わります。

```
CBV_SRV_UAV ヒープ(要素数 4)
┌────────┬────────┬────────┬────────┐
│ CBV[0] │ CBV[1] │ CBV[2] │  SRV   │
└────────┴────────┴────────┴────────┘
```

**第18章 18.1.2 節で述べた「1 つのヒープを切り分けて使う」の実践です。** 現時点では手で番号を管理できますが、**第21章でアロケータを作ります。**

### 20.5.2 サンプラー —— 静的サンプラーを使う

**サンプラーは「テクスチャの読み方」を決めるオブジェクトです。**

| 設定項目 | 内容 |
|---|---|
| フィルタ | 拡大・縮小時の補間方法 |
| アドレスモード | UV が 0〜1 の外に出たときの扱い |
| ミップの選び方 | LOD バイアス、上限・下限 |

**通常のサンプラーは、専用のデスクリプタヒープが必要です。** しかし D3D12 には、**ルートシグネチャに直接埋め込む方法**があります。

```cpp
D3D12_STATIC_SAMPLER_DESC sampler{};
sampler.Filter           = D3D12_FILTER_MIN_MAG_MIP_LINEAR;
sampler.AddressU         = D3D12_TEXTURE_ADDRESS_MODE_WRAP;
sampler.AddressV         = D3D12_TEXTURE_ADDRESS_MODE_WRAP;
sampler.AddressW         = D3D12_TEXTURE_ADDRESS_MODE_WRAP;
sampler.MipLODBias       = 0.0f;
sampler.MaxAnisotropy    = 1;
sampler.ComparisonFunc   = D3D12_COMPARISON_FUNC_NEVER;
sampler.BorderColor      = D3D12_STATIC_BORDER_COLOR_OPAQUE_BLACK;
sampler.MinLOD           = 0.0f;
sampler.MaxLOD           = D3D12_FLOAT32_MAX;
sampler.ShaderRegister   = 0;                              // s0
sampler.RegisterSpace    = 0;
sampler.ShaderVisibility = D3D12_SHADER_VISIBILITY_PIXEL;
```

**静的サンプラーの利点は 3 つです。**

- **予算を消費しません**(第14章 14.1.3 節)
- サンプラー用のデスクリプタヒープが不要
- 実行時に変わらないので、ドライバが最適化できる

**サンプラーの種類は多くありません。** 「線形補間 + 繰り返し」「点サンプリング + クランプ」「異方性フィルタ」など、数種類あれば足ります。**静的サンプラーで足りる場面がほとんどです。**

本書は最後まで静的サンプラーだけを使います。

### 20.5.3 ルートシグネチャを更新する

```cpp
//--- b0: CBV(第18章)---
D3D12_DESCRIPTOR_RANGE1 cbvRange{};
cbvRange.RangeType       = D3D12_DESCRIPTOR_RANGE_TYPE_CBV;
cbvRange.NumDescriptors  = 1;
cbvRange.BaseShaderRegister = 0;
cbvRange.Flags = D3D12_DESCRIPTOR_RANGE_FLAG_DATA_STATIC_WHILE_SET_AT_EXECUTE;
cbvRange.OffsetInDescriptorsFromTableStart =
    D3D12_DESCRIPTOR_RANGE_OFFSET_APPEND;

//--- t0: SRV(本章)---
D3D12_DESCRIPTOR_RANGE1 srvRange{};
srvRange.RangeType       = D3D12_DESCRIPTOR_RANGE_TYPE_SRV;
srvRange.NumDescriptors  = 1;
srvRange.BaseShaderRegister = 0;
srvRange.Flags = D3D12_DESCRIPTOR_RANGE_FLAG_DATA_STATIC;   // ← 変わらない
srvRange.OffsetInDescriptorsFromTableStart =
    D3D12_DESCRIPTOR_RANGE_OFFSET_APPEND;

D3D12_ROOT_PARAMETER1 params[2]{};

// パラメータ 0: CBV(頂点シェーダーのみ)
params[0].ParameterType = D3D12_ROOT_PARAMETER_TYPE_DESCRIPTOR_TABLE;
params[0].DescriptorTable.NumDescriptorRanges = 1;
params[0].DescriptorTable.pDescriptorRanges   = &cbvRange;
params[0].ShaderVisibility = D3D12_SHADER_VISIBILITY_VERTEX;

// パラメータ 1: SRV(ピクセルシェーダーのみ)
params[1].ParameterType = D3D12_ROOT_PARAMETER_TYPE_DESCRIPTOR_TABLE;
params[1].DescriptorTable.NumDescriptorRanges = 1;
params[1].DescriptorTable.pDescriptorRanges   = &srvRange;
params[1].ShaderVisibility = D3D12_SHADER_VISIBILITY_PIXEL;

//--- フラグ ---
versioned.Desc_1_1.NumParameters     = 2;
versioned.Desc_1_1.pParameters       = params;
versioned.Desc_1_1.NumStaticSamplers = 1;
versioned.Desc_1_1.pStaticSamplers   = &sampler;
versioned.Desc_1_1.Flags =
      D3D12_ROOT_SIGNATURE_FLAG_ALLOW_INPUT_ASSEMBLER_INPUT_LAYOUT
    | D3D12_ROOT_SIGNATURE_FLAG_DENY_HULL_SHADER_ROOT_ACCESS
    | D3D12_ROOT_SIGNATURE_FLAG_DENY_DOMAIN_SHADER_ROOT_ACCESS
    | D3D12_ROOT_SIGNATURE_FLAG_DENY_GEOMETRY_SHADER_ROOT_ACCESS;
    // DENY_PIXEL_SHADER_ROOT_ACCESS を削除(第18章 18.4.6)
```

**第18章で追加した `DENY_PIXEL_SHADER_ROOT_ACCESS` を削除します。** ピクセルシェーダーが SRV を読むようになったからです。**残したままだと、ルートシグネチャの生成が失敗します。**

**`srvRange.Flags` を `DATA_STATIC` にしています。** テクスチャの中身は転送後まったく変わらないので、最も強い宣言ができます(第18章 18.4.3 節)。

---

## 20.6 UV 座標とアドレッシング

### 20.6.1 UV 座標

**D3D の UV は、左上が原点です。**

```
(0,0) ─────── (1,0)
  │             │
  │   テクスチャ  │
  │             │
(0,1) ─────── (1,1)
```

**OpenGL は左下が原点です。** 他のエンジンからデータを持ってくるとき、**V が反転していないか確認してください。** 上下逆に貼られていたら、これが原因です。

### 20.6.2 立方体に UV を付ける

**第16章で作った 8 頂点の立方体には、UV を付けられません。**

理由は第16章 16.1.1 節で予告した通りです。立方体の角では、**隣り合う 3 つの面で UV が異なります。** 位置は同じでも UV が違うので、共有できません。

**6 面 × 4 隅 = 24 頂点**に増やします。

```cpp
struct Vertex
{
    float position[3];
    float color[4];
    float uv[2];        // ← 追加
};

static_assert(sizeof(Vertex) == 36);
```

**入力レイアウトも更新します。**

```cpp
constexpr D3D12_INPUT_ELEMENT_DESC kCubeInputElements[] = {
    { "POSITION", 0, DXGI_FORMAT_R32G32B32_FLOAT,    0,
      0,                            D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA, 0 },
    { "COLOR",    0, DXGI_FORMAT_R32G32B32A32_FLOAT, 0,
      D3D12_APPEND_ALIGNED_ELEMENT, D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA, 0 },
    { "TEXCOORD", 0, DXGI_FORMAT_R32G32_FLOAT,       0,
      D3D12_APPEND_ALIGNED_ELEMENT, D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA, 0 },
};
```

**`D3D12_APPEND_ALIGNED_ELEMENT` のおかげで、オフセットを計算し直す必要がありません**(第14章 14.4.5 節)。

**この 24 頂点の構造は、第22章でも使います。** 法線を追加するときに、また同じ理由で共有できないからです。**今回の作業は無駄になりません。**

### 20.6.3 アドレスモード

**UV が 0〜1 の外に出たときの扱いです。**

| モード | 挙動 |
|---|---|
| **`WRAP`** | **繰り返す。`1.5` → `0.5`** |
| `CLAMP` | 端の色が伸びる。`1.5` → `1.0` |
| `MIRROR` | 折り返す。`1.5` → `0.5`(反転) |
| `BORDER` | 指定した色になる |

**タイル状の模様なら `WRAP`、UI や単発の画像なら `CLAMP`** が定石です。

`BORDER` は、第27章のシャドウマップで使います。影の範囲外を「影がない」扱いにするためです。

### 20.6.4 フィルタリング

```cpp
sampler.Filter = D3D12_FILTER_MIN_MAG_MIP_LINEAR;
```

**名前の 3 つの部分が、それぞれ別の状況を指します。**

| 部分 | いつ使われるか |
|---|---|
| `MIN` | **縮小時**(テクスチャがピクセルより細かい) |
| `MAG` | **拡大時**(テクスチャがピクセルより粗い) |
| `MIP` | **ミップレベル間の補間** |

| 値 | 意味 |
|---|---|
| `POINT` | 最も近いテクセルをそのまま |
| `LINEAR` | 周囲を線形補間 |

**ドット絵を拡大するなら `MIN_MAG_MIP_POINT` です。** `LINEAR` にするとぼやけます。

**異方性フィルタ**は、斜めに見た面の品質を上げます。

```cpp
sampler.Filter        = D3D12_FILTER_ANISOTROPIC;
sampler.MaxAnisotropy = 16;    // 1〜16
```

**地面や壁など、視線と浅い角度で交わる面で劇的に効きます。** コストは上がりますが、現代の GPU では実用的な範囲です。

### 20.6.5 ミップマップ

**ミップマップは、縮小されたテクスチャをあらかじめ用意しておく仕組みです。**

```
mip 0: 256×256
mip 1: 128×128
mip 2:  64×64
   ...
mip 8:   1×1
```

**ないとどうなるか。** 遠くの面でテクスチャを 1 ピクセルおきに拾うことになり、**模様がちらつきます。** カメラが動くと激しくノイズが出ます。

**GPU は UV の変化率から適切なレベルを自動で選びます。** シェーダー側で指定する必要はありません。

```hlsl
float4 color = texture.Sample(sampler, input.uv);   // 自動でミップを選ぶ
```

**容量は 1.33 倍で済みます。** 各段が 4 分の 1 になるので、`1 + 1/4 + 1/16 + ... = 4/3` です。**ちらつきの解消に対して、非常に安い代償です。**

> **ミップを自分で生成したい場合**
>
> DDS に含めておくのが最善ですが、実行時に作る必要が出ることもあります。
>
> D3D12 には `GenerateMips` に相当する API が**ありません**(D3D11 にはありました)。**コンピュートシェーダーで自分で書くことになります。** 第32章の題材として適しています。

---

## 20.7 ブロック圧縮への対応

**20.4.2 節で `pNumRows` を使ったおかげで、対応はほぼ済んでいます。**

必要なのは、`ComputePitch` を圧縮フォーマットに対応させることだけです。

```cpp
[[nodiscard]] constexpr bool IsBlockCompressed(DXGI_FORMAT format) noexcept
{
    switch (format)
    {
    case DXGI_FORMAT_BC1_UNORM: case DXGI_FORMAT_BC1_UNORM_SRGB:
    case DXGI_FORMAT_BC2_UNORM: case DXGI_FORMAT_BC2_UNORM_SRGB:
    case DXGI_FORMAT_BC3_UNORM: case DXGI_FORMAT_BC3_UNORM_SRGB:
    case DXGI_FORMAT_BC4_UNORM: case DXGI_FORMAT_BC4_SNORM:
    case DXGI_FORMAT_BC5_UNORM: case DXGI_FORMAT_BC5_SNORM:
    case DXGI_FORMAT_BC6H_UF16: case DXGI_FORMAT_BC6H_SF16:
    case DXGI_FORMAT_BC7_UNORM: case DXGI_FORMAT_BC7_UNORM_SRGB:
        return true;
    default:
        return false;
    }
}

Core::Status ComputePitch(DXGI_FORMAT format, UINT width, UINT height,
                          UINT64& rowPitch, UINT64& slicePitch)
{
    if (IsBlockCompressed(format))
    {
        // 4×4 ピクセルで 1 ブロック
        const UINT64 blockSize = BlockSizeInBytes(format);   // BC1/BC4 は 8、他は 16
        const UINT64 blocksWide = std::max<UINT64>((width  + 3) / 4, 1);
        const UINT64 blocksHigh = std::max<UINT64>((height + 3) / 4, 1);

        rowPitch   = blocksWide * blockSize;
        slicePitch = rowPitch * blocksHigh;
        return {};
    }

    const UINT bpp = BitsPerPixel(format);
    if (bpp == 0)
    {
        return std::unexpected(Core::MakeError(E_NOTIMPL, L"unsupported format"));
    }

    rowPitch   = (static_cast<UINT64>(width) * bpp + 7) / 8;
    slicePitch = rowPitch * height;
    return {};
}
```

**転送コードは 1 文字も変えません。** `GetCopyableFootprints` が正しい `pNumRows` を返し、それをそのまま使っているからです。

**これが 20.4.2 節のコラムで「`Height` ではなく `pNumRows` を使え」と書いた理由です。**

> **ブロック圧縮は 4 の倍数サイズを前提にする**
>
> 4×4 ブロック単位なので、**幅や高さが 4 の倍数でないと端が無駄になります。** ミップの最下段(1×1、2×2)も 1 ブロックぶん(8〜16 バイト)を消費します。
>
> テクスチャは 2 のべき乗サイズにしておくのが無難です。

---

## 20.8 シェーダーを更新する

```hlsl
//=====================================================
// shaders/TexturedCube.hlsl
//=====================================================

cbuffer SceneConstants : register(b0)
{
    row_major float4x4 worldViewProj;
};

Texture2D    gTexture : register(t0);
SamplerState gSampler : register(s0);   // 静的サンプラー

struct VSInput
{
    float3 position : POSITION;
    float4 color    : COLOR;
    float2 uv       : TEXCOORD;
};

struct VSOutput
{
    float4 position : SV_Position;
    float4 color    : COLOR;
    float2 uv       : TEXCOORD;
};

VSOutput VSMain(VSInput input)
{
    VSOutput output;
    output.position = mul(float4(input.position, 1.0f), worldViewProj);
    output.color    = input.color;
    output.uv       = input.uv;
    return output;
}

float4 PSMain(VSOutput input) : SV_Target
{
    const float4 texColor = gTexture.Sample(gSampler, input.uv);
    return texColor * input.color;
}
```

**`register(t0)` と `register(s0)` が、ルートシグネチャの `BaseShaderRegister` と対応します。**

`uv` が**ラスタライザによって補間される**ことに注目してください(第15章 15.6 節)。色と同じ仕組みです。

描画側でテーブルを 2 つ設定します。

```cpp
m_commandList->SetGraphicsRootDescriptorTable(
    0, OffsetHandle(m_cbvHeapGpuStart, index, m_cbvIncrement));   // CBV

m_commandList->SetGraphicsRootDescriptorTable(
    1, OffsetHandle(m_cbvHeapGpuStart, kSrvIndex, m_cbvIncrement)); // SRV
```

---

## ✅ 本章のゴール:テクスチャ付き立方体

### Step 1:テクスチャを用意する

`texconv` で DDS を作ります。

```
texconv -f R8G8B8A8_UNORM -m 0 -dx10 -o assets test.png
```

| オプション | 意味 |
|---|---|
| `-f` | 出力フォーマット |
| `-m 0` | **ミップマップを全段生成** |
| `-dx10` | **DXT10 拡張を付ける(必須)** |

**格子模様のような、向きが分かる画像**を使ってください。上下反転や UV のずれに気づけます。

### Step 2:実行する

```
[Info ] DdsLoader.cpp(142): dds loaded: test.dds (256x256, 9 mips, format 28)
[Info ] ResourceUploader.cpp(198): uploaded 1 texture(s), 9 subresource(s)
```

**テクスチャの貼られた立方体が回転します。**

### Step 3:行ピッチを取り違えてみる

**20.4.3 節の ④ で、`RowPitch` を `rowSizes` に変えます。**

```cpp
std::memcpy(
    dest + row * rowSizes[i],          // ❌ 整列後ではなく実データのピッチ
    src.data + row * src.rowPitch,
    static_cast<std::size_t>(rowSizes[i]));
```

**絵が斜めにずれます。** 行を進む幅が足りないので、各行が少しずつ前へ食い込みます。

**これがテクスチャ転送で最も多い失敗です。** 症状を一度見ておくと、次に遭遇したとき即座に分かります。

**確認したら元に戻してください。**

### Step 4:`Shader4ComponentMapping` を忘れる

```cpp
srvDesc.Shader4ComponentMapping = 0;   // ❌
```

**絵が赤一色になります。** 全チャンネルが赤成分にマッピングされるからです。

**エラーは出ません。** 第14章 14.4.3 節の `SampleMask` と同じ、「ゼロが有効な値として通る」罠です。

**確認したら元に戻してください。**

### Step 5:ミップマップを無効にする

```cpp
srvDesc.Texture2D.MipLevels = 1;   // mip 0 だけ使う
```

**立方体を遠ざけると、模様がちらつきます。**

戻すと、なめらかになります。**ミップマップの効果が最も分かりやすい確認方法です。**

### Step 6:アドレスモードを変える

UV を 0〜2 の範囲にしてみます。

```cpp
// 各面の UV を 2 倍に
{ 0, 0 }, { 2, 0 }, { 2, 2 }, { 0, 2 }
```

| モード | 見え方 |
|---|---|
| `WRAP` | 2×2 のタイル状 |
| `CLAMP` | 端の色が伸びる |
| `MIRROR` | 鏡像で折り返す |

**確認したら元に戻してください。**

### Step 7:圧縮フォーマットを試す

```
texconv -f BC7_UNORM -m 0 -dx10 -o assets test.png
```

**コードを変えずに動きます。**

ファイルサイズを比べてください。**`R8G8B8A8` の 4 分の 1 になっています。**

---

### 本章の達成状態

- [ ] DDS ヘッダ構造体に `static_assert` を入れた
- [ ] DXT10 拡張の有無を確認している
- [ ] ミップレベルごとの位置を計算している
- [ ] `GetCopyableFootprints` で配置情報を取得している
- [ ] `RowPitch` と `rowSizes` を正しく使い分けている
- [ ] `pNumRows` を使っている(`Height` ではなく)
- [ ] `CopyTextureRegion` をサブリソースごとに呼んでいる
- [ ] `PIXEL_SHADER_RESOURCE` へ遷移させている
- [ ] `Shader4ComponentMapping` を指定した
- [ ] 静的サンプラーを使っている
- [ ] `DENY_PIXEL_SHADER_ROOT_ACCESS` を削除した
- [ ] 立方体を 24 頂点にした
- [ ] **テクスチャ付き立方体が表示された**
- [ ] Step 3 で行ピッチの取り違えを体験した
- [ ] ブロック圧縮が動くことを確認した

---

## トラブルシューティング

| 症状 | 原因 | 対処 |
|---|---|---|
| `not a DDS file` | マジックナンバー不一致 | ファイルを確認 |
| `legacy DDS format` | DXT10 拡張がない | `texconv -dx10`(20.1.3) |
| `dds truncated` | ミップ数とデータ量の不整合 | 再変換する |
| **絵が斜めにずれる** | **`RowPitch` の取り違え** | **20.4.3 の ④** |
| 絵が赤一色 | `Shader4ComponentMapping` | 20.5.1 |
| 上下が逆 | UV の V 軸 | 20.6.1 |
| 遠くでちらつく | ミップマップがない | `-m 0` で生成(20.6.5) |
| 拡大するとぼやける | `LINEAR` フィルタ | `POINT` にする(20.6.4) |
| ルートシグネチャ生成に失敗 | `DENY_PIXEL_SHADER_ROOT_ACCESS` | 削除する(20.5.3) |
| SRV が読めない | `SetDescriptorHeaps` 忘れ | 第18章 18.5.3 |
| 圧縮テクスチャで壊れる | `ComputePitch` がブロック非対応 | 20.7 |
| 転送後に描画するとエラー | 状態遷移を忘れた | 20.4.3 の ⑥ |

---

## まとめ

**1. テクスチャは `Map` して直接書けない。**
GPU が独自のスウィズル順で並べるためです。**中間バッファを経由し、行ごとにコピーします。**

**2. 2 つのアラインメントがある。**
行は 256 バイト、サブリソースの開始位置は 512 バイト。**行ごとにパディングが入るため、`memcpy` を 1 回で済ませられません。**

**3. `GetCopyableFootprints` が計算してくれる。**
自分でアラインメントを計算する必要はありません。**ただし返ってくる 3 つの値の意味を正確に区別する必要があります。**

**4. `RowPitch` と `pRowSizeInBytes` は別物。**
前者は「進む幅」、後者は「コピーする量」。**取り違えると絵が斜めにずれます。**

**5. `pNumRows` を使えば圧縮に自動対応する。**
`Height` を使うと、ブロック圧縮で壊れます。**`ComputePitch` を直すだけで BC7 が動いたのは、ここを正しく書いたからです。**

**6. 静的サンプラーで足りる。**
サンプラーの種類は多くありません。ルートシグネチャに埋め込めば、ヒープも予算も不要です。

**7. これが `UpdateSubresources()` の中身。**
`d3dx12.h` を使えば 1 行でした。**その 1 行が何をしていたかを、我々は知っています。**

次章では、リソース管理を設計します。**本章で手作業だったデスクリプタの番号管理**を、アロケータとして整理します。あわせてアップロードバッファのリングバッファ化、Resizable BAR と GPU Upload Heaps、そしてリソースの遅延解放を扱います。**第2部の締めくくりです。**

---

## 参考リンク

| 内容 | URL |
|---|---|
| DDS ファイル形式のリファレンス | https://learn.microsoft.com/ja-jp/windows/win32/direct3ddds/dx-graphics-dds-pguide |
| `DDS_HEADER_DXT10` | https://learn.microsoft.com/ja-jp/windows/win32/direct3ddds/dds-header-dxt10 |
| `GetCopyableFootprints` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/nf-d3d12-id3d12device-getcopyablefootprints |
| `CopyTextureRegion` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/nf-d3d12-id3d12graphicscommandlist-copytextureregion |
| ブロック圧縮 | https://learn.microsoft.com/ja-jp/windows/win32/direct3d11/texture-block-compression-in-direct3d-11 |
| `D3D12_STATIC_SAMPLER_DESC` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/ns-d3d12-d3d12_static_sampler_desc |
| DirectXTex(texconv) | https://github.com/microsoft/DirectXTex |
