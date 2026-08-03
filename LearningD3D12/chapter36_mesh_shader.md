# 第36章 メッシュシェーダー

**20 年以上続いた枠組みを離れます。**

第13章 13.1.1 節で示したパイプラインを、思い出してください。

```
Input Assembler → Vertex Shader → Rasterizer → Pixel Shader
```

**この形は、Direct3D 8 の頃から本質的に変わっていません。** 頂点バッファとインデックスバッファを固定機能が読み、頂点シェーダーが 1 頂点ずつ処理する。

**メッシュシェーダーは、この前提を捨てます。**

```
Amplification Shader → Mesh Shader → Rasterizer → Pixel Shader
```

**入力アセンブラがありません。** 頂点バッファもインデックスバッファも、パイプラインの一部ではなくなります。

**メッシュシェーダーは、コンピュートシェーダーのように動き、頂点と三角形を直接出力します。**

第34章 34.4.5 節で、こう書きました。

> **本書では、単純な形に留めます。**
> - **第36章のメッシュシェーダーが、より良い解を提供する**

**その解を、本章で示します。**

**本章のゴール**
メッシュレットを生成し、メッシュシェーダーで描画する。Amplification シェーダーで GPU カリングを行う。あわせて、パイプラインステートストリームを自作する。

---

## 36.1 従来パイプラインの限界

### 36.1.1 頂点シェーダーの制約

**頂点シェーダーは、1 頂点を受け取って 1 頂点を出力します。**

```
入力 1 頂点  →  頂点シェーダー  →  出力 1 頂点
```

**この 1 対 1 の対応が、多くの制約を生みます。**

| できないこと | 理由 |
|---|---|
| **頂点を増やす** | 出力は 1 つだけ |
| **頂点を減らす** | 同上 |
| **三角形単位の処理** | 隣接頂点が見えない |
| **他の頂点との協調** | スレッド間で通信できない |

**ジオメトリシェーダーが、これを解決するはずでした。**

**第13章 13.1.1 節で、こう書きました。**

> **ジオメトリシェーダーについて。** 使えますが、本書では最後まで使いません。NVIDIA のハードウェアでは実装が非効率で、性能が大きく落ちるためです。**同じことをしたければ、第36章のメッシュシェーダーが正しい後継です。**

**ジオメトリシェーダーが遅い理由は、出力の順序を保証するためです。**

**入力された三角形の順に出力しなければならないため、GPU は中間バッファに結果を溜め、並べ替える必要があります。**

### 36.1.2 インデックスバッファの限界

**第16章でインデックスバッファを導入しました。**

**頂点の重複を排除し、頂点キャッシュを効かせる**という利点がありました。

**しかし、問題もあります。**

| 問題 | 説明 |
|---|---|
| **メモリアクセスがランダム** | インデックスが指す先が飛び飛び |
| **キャッシュの効率が読めない** | 頂点の並び順に依存 |
| **カリングの粒度が粗い** | オブジェクト単位でしか判定できない |

**最後が本章の主題です。**

**第34章の GPU カリングは、オブジェクト単位でした。**

```
オブジェクトが視錐台の中 → 全ポリゴンを描く
```

**巨大なモデルの一部だけが見えている場合、見えない部分も描画されます。**

### 36.1.3 メッシュレットという発想

**モデルを、小さな塊に分割します。**

```
モデル(10 万三角形)
  ├─ メッシュレット 0(64 頂点、124 三角形)
  ├─ メッシュレット 1
  ├─ ...
  └─ メッシュレット 1500
```

**それぞれが、独立して処理される単位です。**

**利点は 3 つです。**

| 利点 | 説明 |
|---|---|
| **細かいカリング** | メッシュレット単位で判定できる |
| **キャッシュ効率** | 局所的な頂点をまとめて処理 |
| **並列性** | メッシュレットごとに 1 スレッドグループ |

---

## 36.2 対応を確認する

### 36.2.1 必要な条件

**第7章 7.5.5 節の判定関数を思い出してください。**

```cpp
bool DeviceCaps::SupportsMeshShader() const noexcept
{
    return meshShaderTier >= D3D12_MESH_SHADER_TIER_1;
}
```

**問い合わせは `OPTIONS7` です**(第7章 7.5.3 節)。

```cpp
D3D12_FEATURE_DATA_D3D12_OPTIONS7 options7{};
if (QueryFeature(device, D3D12_FEATURE_D3D12_OPTIONS7, options7))
{
    m_caps.meshShaderTier = options7.MeshShaderTier;
}
```

**第2章 2.1.1 節で、Turing 世代の特徴として挙げました。**

> - 第1世代 RT Core によるハードウェアレイトレーシング
> - **メッシュシェーダー、Amplification シェーダー**

**そして、GTX 16 シリーズの注意も書きました。**

> GTX 1650 / 1660 などの GTX 16 シリーズは、アーキテクチャとしては Turing です。しかし **RT Core と Tensor Core が搭載されていません**。**メッシュシェーダーや VRS は動きますが**、レイトレーシングは Pascal と同じくソフトウェア実行になります。

**メッシュシェーダーは、GTX 16 シリーズでも動きます。**

### 36.2.2 シェーダーモデル

**Shader Model 6.5 以上が必要です。**

```
dxc -T ms_6_6 ...    ← メッシュシェーダー
dxc -T as_6_6 ...    ← Amplification シェーダー
```

**第13章 13.1.2 節のターゲット文字列の表に、既に含まれていました。**

| 接頭辞 | ステージ | 登場する章 |
|---|---|---|
| `ms_` | メッシュシェーダー | 第36章 |
| `as_` | Amplification シェーダー | 第36章 |

**本書のベースラインは `6_6` なので、そのまま使えます。**

---

## 36.3 パイプラインステートストリーム

### 36.3.1 `CD3DX12_PIPELINE_STATE_STREAM` が使えない

**メッシュシェーダー用の PSO を作るには、新しい仕組みが必要です。**

**`D3D12_GRAPHICS_PIPELINE_STATE_DESC` には、メッシュシェーダーのフィールドがありません。**

**構造体に追加すると、既存のコードが壊れます。** そこで、**拡張可能な形式**が導入されました。

```cpp
typedef struct D3D12_PIPELINE_STATE_STREAM_DESC {
    SIZE_T SizeInBytes;
    void*  pPipelineStateSubobjectStream;
} D3D12_PIPELINE_STATE_STREAM_DESC;
```

**「サブオブジェクトの列」を渡します。**

```
[型][データ] [型][データ] [型][データ] ...
```

**`d3dx12.h` には `CD3DX12_PIPELINE_STATE_STREAM` というヘルパーがありますが、本書では使いません**(第1章 1.3.1 節)。

**自分で書きます。**

### 36.3.2 サブオブジェクトの構造

**各サブオブジェクトは、型とデータの組です。**

```cpp
typedef enum D3D12_PIPELINE_STATE_SUBOBJECT_TYPE {
    D3D12_PIPELINE_STATE_SUBOBJECT_TYPE_ROOT_SIGNATURE        = 0,
    D3D12_PIPELINE_STATE_SUBOBJECT_TYPE_VS                    = 1,
    D3D12_PIPELINE_STATE_SUBOBJECT_TYPE_PS                    = 2,
    // ...
    D3D12_PIPELINE_STATE_SUBOBJECT_TYPE_BLEND                 = 8,
    D3D12_PIPELINE_STATE_SUBOBJECT_TYPE_DEPTH_STENCIL         = 9,
    D3D12_PIPELINE_STATE_SUBOBJECT_TYPE_INPUT_LAYOUT          = 10,
    D3D12_PIPELINE_STATE_SUBOBJECT_TYPE_RASTERIZER            = 12,
    D3D12_PIPELINE_STATE_SUBOBJECT_TYPE_RENDER_TARGET_FORMATS = 14,
    // ...
    D3D12_PIPELINE_STATE_SUBOBJECT_TYPE_AS                    = 24,
    D3D12_PIPELINE_STATE_SUBOBJECT_TYPE_MS                    = 25,
} D3D12_PIPELINE_STATE_SUBOBJECT_TYPE;
```

**メッシュシェーダーは 25、Amplification シェーダーは 24 です。**

### 36.3.3 アラインメントの規則

**ここが最も注意を要する点です。**

> **各サブオブジェクトは、`void*` のアラインメント(x64 では 8 バイト)に揃える必要があります。**

**素朴に構造体を並べると、コンパイラのパディングと食い違います。**

```cpp
//--- ❌ 危険 ---
struct BadStream
{
    D3D12_PIPELINE_STATE_SUBOBJECT_TYPE type1;   // 4 バイト
    ID3D12RootSignature* rootSignature;          // 8 バイト
    // ここに 4 バイトのパディングが入る?
};
```

**`alignas` で明示します。**

```cpp
template <typename T, D3D12_PIPELINE_STATE_SUBOBJECT_TYPE Type>
struct alignas(void*) PipelineStateSubobject
{
    D3D12_PIPELINE_STATE_SUBOBJECT_TYPE type = Type;
    T value{};

    PipelineStateSubobject() = default;

    explicit PipelineStateSubobject(const T& v)
        : value(v) {}

    PipelineStateSubobject& operator=(const T& v)
    {
        value = v;
        return *this;
    }
};
```

**テンプレートにすることで、型と列挙値の対応を強制できます。**

**`d3dx12.h` の `CD3DX12_PIPELINE_STATE_STREAM_SUBOBJECT` も、同じ構造です。**

### 36.3.4 型安全なストリームを作る

**よく使うものに別名を付けます。**

```cpp
// src/Graphics/PipelineStateStream.h
#pragma once
#include "std_import.h"

namespace Graphics
{
    template <typename T, D3D12_PIPELINE_STATE_SUBOBJECT_TYPE Type>
    struct alignas(void*) PipelineStateSubobject
    {
        D3D12_PIPELINE_STATE_SUBOBJECT_TYPE type = Type;
        T value{};

        PipelineStateSubobject& operator=(const T& v)
        {
            value = v;
            return *this;
        }
    };

    //--- よく使うもの ---
    using PssRootSignature = PipelineStateSubobject<
        ID3D12RootSignature*,
        D3D12_PIPELINE_STATE_SUBOBJECT_TYPE_ROOT_SIGNATURE>;

    using PssAmplificationShader = PipelineStateSubobject<
        D3D12_SHADER_BYTECODE,
        D3D12_PIPELINE_STATE_SUBOBJECT_TYPE_AS>;

    using PssMeshShader = PipelineStateSubobject<
        D3D12_SHADER_BYTECODE,
        D3D12_PIPELINE_STATE_SUBOBJECT_TYPE_MS>;

    using PssPixelShader = PipelineStateSubobject<
        D3D12_SHADER_BYTECODE,
        D3D12_PIPELINE_STATE_SUBOBJECT_TYPE_PS>;

    using PssBlend = PipelineStateSubobject<
        D3D12_BLEND_DESC,
        D3D12_PIPELINE_STATE_SUBOBJECT_TYPE_BLEND>;

    using PssRasterizer = PipelineStateSubobject<
        D3D12_RASTERIZER_DESC,
        D3D12_PIPELINE_STATE_SUBOBJECT_TYPE_RASTERIZER>;

    using PssDepthStencil = PipelineStateSubobject<
        D3D12_DEPTH_STENCIL_DESC,
        D3D12_PIPELINE_STATE_SUBOBJECT_TYPE_DEPTH_STENCIL>;

    using PssRenderTargetFormats = PipelineStateSubobject<
        D3D12_RT_FORMAT_ARRAY,
        D3D12_PIPELINE_STATE_SUBOBJECT_TYPE_RENDER_TARGET_FORMATS>;

    using PssDepthStencilFormat = PipelineStateSubobject<
        DXGI_FORMAT,
        D3D12_PIPELINE_STATE_SUBOBJECT_TYPE_DEPTH_STENCIL_FORMAT>;

    using PssSampleDesc = PipelineStateSubobject<
        DXGI_SAMPLE_DESC,
        D3D12_PIPELINE_STATE_SUBOBJECT_TYPE_SAMPLE_DESC>;

    using PssSampleMask = PipelineStateSubobject<
        UINT,
        D3D12_PIPELINE_STATE_SUBOBJECT_TYPE_SAMPLE_MASK>;

    using PssPrimitiveTopology = PipelineStateSubobject<
        D3D12_PRIMITIVE_TOPOLOGY_TYPE,
        D3D12_PIPELINE_STATE_SUBOBJECT_TYPE_PRIMITIVE_TOPOLOGY>;
}
```

**メッシュシェーダー用のストリームを定義します。**

```cpp
struct MeshShaderPipelineStateStream
{
    PssRootSignature       rootSignature;
    PssAmplificationShader amplificationShader;
    PssMeshShader          meshShader;
    PssPixelShader         pixelShader;
    PssBlend               blend;
    PssRasterizer          rasterizer;
    PssDepthStencil        depthStencil;
    PssRenderTargetFormats renderTargetFormats;
    PssDepthStencilFormat  depthStencilFormat;
    PssSampleDesc          sampleDesc;
    PssSampleMask          sampleMask;
    PssPrimitiveTopology   primitiveTopology;
};
```

**既定値を設定するヘルパーも作ります。**

**第14章 14.4.4 節の `DefaultGraphicsPipelineStateDesc()` と同じ発想です。**

```cpp
[[nodiscard]] inline MeshShaderPipelineStateStream
DefaultMeshShaderPipelineStateStream() noexcept
{
    MeshShaderPipelineStateStream stream{};

    //--- 第14章 14.4.3 節の罠を、ここでも回避する ---
    stream.blend         = DefaultBlendDesc();
    stream.rasterizer    = DefaultRasterizerDesc();
    stream.depthStencil  = DefaultDepthStencilDesc();
    stream.sampleMask    = UINT_MAX;                    // ← 罠 1
    stream.sampleDesc    = DXGI_SAMPLE_DESC{ 1, 0 };    // ← 罠 3
    stream.primitiveTopology =
        D3D12_PRIMITIVE_TOPOLOGY_TYPE_TRIANGLE;         // ← 罠 5

    return stream;
}
```

**第14章の罠が、ここでも同じように適用されます。**

**`InputLayout` がない**ことに注目してください。メッシュシェーダーでは不要です。

### 36.3.5 PSO を作る

```cpp
Core::Result<ComPtr<ID3D12PipelineState>> CreateMeshShaderPso(
    ID3D12Device2*       device2,
    ID3D12RootSignature* rootSignature,
    const ShaderBlob*    amplificationShader,   // nullptr 可
    const ShaderBlob&    meshShader,
    const ShaderBlob&    pixelShader,
    std::wstring_view    name)
{
    auto stream = DefaultMeshShaderPipelineStateStream();

    stream.rootSignature = rootSignature;
    stream.meshShader    = meshShader.Bytecode();
    stream.pixelShader   = pixelShader.Bytecode();

    if (amplificationShader != nullptr)
    {
        stream.amplificationShader = amplificationShader->Bytecode();
    }

    //--- 出力フォーマット(第26章の HDR ターゲット)---
    D3D12_RT_FORMAT_ARRAY formats{};
    formats.NumRenderTargets = 1;
    formats.RTFormats[0]     = kSceneColorFormat;
    stream.renderTargetFormats = formats;

    stream.depthStencilFormat = DXGI_FORMAT_D32_FLOAT;
    stream.sampleDesc         = DXGI_SAMPLE_DESC{ m_sampleCount, 0 };

    //--- ストリームとして渡す ---
    D3D12_PIPELINE_STATE_STREAM_DESC desc{};
    desc.SizeInBytes                   = sizeof(stream);
    desc.pPipelineStateSubobjectStream = &stream;

    ComPtr<ID3D12PipelineState> pso;
    HR_TRY(device2->CreatePipelineState(&desc, IID_PPV_ARGS(&pso)));

    Core::SetDebugName(pso.Get(), name);
    return pso;
}
```

**`ID3D12Device2` が必要です。** `CreatePipelineState` は、この版で追加されました。

**第6章 6.1.4 節のインターフェースバージョンです。**

> **Amplification シェーダーは省略できます**
>
> `nullptr` を渡すと、メッシュシェーダーだけのパイプラインになります。
>
> **ただし、サブオブジェクト自体は含めておく必要があります。** `D3D12_SHADER_BYTECODE` がゼロなら、「なし」と解釈されます。

---

## 36.4 メッシュレットを生成する

### 36.4.1 制約

**メッシュシェーダーが 1 回で出力できる量には、上限があります。**

```
最大頂点数:    256
最大プリミティブ数: 256
```

**そして、NVIDIA には推奨値があります。**

| 項目 | 推奨値 | 理由 |
|---|---|---|
| **頂点数** | **64** | 2 warp ぶん |
| **三角形数** | **126** | 84 バイト境界に収まる |

**64 頂点は、第2章 2.3.1 節の warp = 32 の 2 倍です。**

**第32章 32.3.4 節で `numthreads` を 64 にしたのと、同じ理由です。**

> `(64, 1, 1)` | 64 | 2 warp | **0%**

**126 という数字は、内部のデータ構造に由来します。** 128 でも動きますが、64/126 が最も効率的とされています。

### 36.4.2 データ構造

```cpp
struct Meshlet
{
    std::uint32_t vertexOffset;      // 頂点インデックス配列内の位置
    std::uint32_t vertexCount;
    std::uint32_t primitiveOffset;   // プリミティブ配列内の位置
    std::uint32_t primitiveCount;
};

struct MeshletBounds
{
    Math::Vector3 center;
    float         radius;

    //--- 法線錐(36.6.3 節)---
    Math::Vector3 coneAxis;
    float         coneCutoff;
};

//--- 三角形を 1 つの uint に詰める ---
struct PackedPrimitive
{
    std::uint32_t packed;   // 10bit × 3 + 予備

    void Set(std::uint32_t i0, std::uint32_t i1, std::uint32_t i2) noexcept
    {
        packed = (i0 & 0x3FF)
               | ((i1 & 0x3FF) << 10)
               | ((i2 & 0x3FF) << 20);
    }
};
```

**プリミティブを 10bit ずつに詰めています。**

**メッシュレット内の頂点は最大 256 個なので、8bit で足ります。** 10bit にしているのは、`uint` に 3 つ詰めるためです。

**メモリを 3 分の 1 に節約できます。**

### 36.4.3 生成アルゴリズム

**単純な貪欲法を実装します。**

```cpp
struct MeshletBuildResult
{
    std::vector<Meshlet>         meshlets;
    std::vector<std::uint32_t>   vertexIndices;    // 元の頂点への参照
    std::vector<PackedPrimitive> primitives;
    std::vector<MeshletBounds>   bounds;
};

MeshletBuildResult BuildMeshlets(
    const std::vector<MeshVertex>& vertices,
    const std::vector<std::uint32_t>& indices,
    UINT maxVertices  = 64,
    UINT maxPrimitives = 126)
{
    MeshletBuildResult result{};

    //--- 現在構築中のメッシュレット ---
    std::unordered_map<std::uint32_t, std::uint32_t> localIndexMap;
    std::vector<std::uint32_t> currentVertices;
    std::vector<PackedPrimitive> currentPrimitives;

    const auto flush = [&]()
    {
        if (currentPrimitives.empty()) return;

        Meshlet meshlet{};
        meshlet.vertexOffset =
            static_cast<std::uint32_t>(result.vertexIndices.size());
        meshlet.vertexCount =
            static_cast<std::uint32_t>(currentVertices.size());
        meshlet.primitiveOffset =
            static_cast<std::uint32_t>(result.primitives.size());
        meshlet.primitiveCount =
            static_cast<std::uint32_t>(currentPrimitives.size());

        result.meshlets.push_back(meshlet);

        result.vertexIndices.insert(result.vertexIndices.end(),
                                    currentVertices.begin(),
                                    currentVertices.end());
        result.primitives.insert(result.primitives.end(),
                                 currentPrimitives.begin(),
                                 currentPrimitives.end());

        //--- 境界を計算(36.6 節で使う)---
        result.bounds.push_back(
            ComputeMeshletBounds(vertices, currentVertices, currentPrimitives));

        localIndexMap.clear();
        currentVertices.clear();
        currentPrimitives.clear();
    };

    for (std::size_t i = 0; i + 2 < indices.size(); i += 3)
    {
        const std::uint32_t tri[3] = {
            indices[i], indices[i + 1], indices[i + 2] };

        //--- 新たに必要な頂点数を数える ---
        UINT newVertexCount = 0;
        for (const auto index : tri)
        {
            if (!localIndexMap.contains(index))
            {
                ++newVertexCount;
            }
        }

        //--- 上限を超えるなら、現在のメッシュレットを確定 ---
        if (currentVertices.size() + newVertexCount > maxVertices ||
            currentPrimitives.size() + 1 > maxPrimitives)
        {
            flush();
        }

        //--- 頂点を登録 ---
        std::uint32_t local[3]{};
        for (int k = 0; k < 3; ++k)
        {
            const auto it = localIndexMap.find(tri[k]);
            if (it != localIndexMap.end())
            {
                local[k] = it->second;
            }
            else
            {
                local[k] = static_cast<std::uint32_t>(currentVertices.size());
                localIndexMap.emplace(tri[k], local[k]);
                currentVertices.push_back(tri[k]);
            }
        }

        PackedPrimitive primitive{};
        primitive.Set(local[0], local[1], local[2]);
        currentPrimitives.push_back(primitive);
    }

    flush();

    LOG_INFO(L"meshlets built: {} meshlets from {} triangles",
             result.meshlets.size(), indices.size() / 3);

    return result;
}
```

**この実装は単純ですが、局所性を考慮していません。**

**インデックスの順序に従って詰めるだけなので、モデルの並び方によっては効率が落ちます。**

> **より良い分割**
>
> **実用的には、空間的に近い三角形をまとめるべきです。**
>
> Microsoft の **DirectXMesh** ライブラリには、`ComputeMeshlets` という関数があります。頂点キャッシュの効率を考慮した並べ替えも行います。
>
> **本書はライブラリを使わない方針なので、単純な実装に留めます**(第1章 1.3 節)。
>
> **品質を上げたい場合、次の手順が効果的です。**
>
> 1. 三角形を空間分割(Morton 順など)で並べ替える
> 2. その順序でメッシュレットを構築する
>
> **第23章のモデル読み込みに追加できます。**

### 36.4.4 境界を計算する

```cpp
MeshletBounds ComputeMeshletBounds(
    const std::vector<MeshVertex>& vertices,
    const std::vector<std::uint32_t>& meshletVertices,
    const std::vector<PackedPrimitive>& primitives)
{
    MeshletBounds bounds{};

    //--- ① 境界球 ---
    Math::Vector3 minBounds{  1e30f,  1e30f,  1e30f };
    Math::Vector3 maxBounds{ -1e30f, -1e30f, -1e30f };

    for (const auto index : meshletVertices)
    {
        const auto& p = vertices[index].position;
        minBounds.x = std::min(minBounds.x, p.x);
        minBounds.y = std::min(minBounds.y, p.y);
        minBounds.z = std::min(minBounds.z, p.z);
        maxBounds.x = std::max(maxBounds.x, p.x);
        maxBounds.y = std::max(maxBounds.y, p.y);
        maxBounds.z = std::max(maxBounds.z, p.z);
    }

    bounds.center = (minBounds + maxBounds) * 0.5f;
    bounds.radius = Math::Length(maxBounds - bounds.center);

    //--- ② 法線錐(36.6.3 節)---
    Math::Vector3 averageNormal{};

    for (const auto& primitive : primitives)
    {
        const auto i0 = meshletVertices[primitive.packed & 0x3FF];
        const auto i1 = meshletVertices[(primitive.packed >> 10) & 0x3FF];
        const auto i2 = meshletVertices[(primitive.packed >> 20) & 0x3FF];

        const auto normal = Math::Normalize(Math::Cross(
            vertices[i1].position - vertices[i0].position,
            vertices[i2].position - vertices[i0].position));

        averageNormal += normal;
    }

    bounds.coneAxis = Math::Normalize(averageNormal);

    //--- 最も外れた法線との角度 ---
    float minDot = 1.0f;
    for (const auto& primitive : primitives)
    {
        // ... 各面の法線と coneAxis の内積の最小値 ...
    }

    bounds.coneCutoff = minDot;

    return bounds;
}
```

**法線錐は「このメッシュレットの面が向いている範囲」を表します。**

**36.6.3 節で、裏面カリングに使います。**

---

## 36.5 メッシュシェーダーを書く

### 36.5.1 基本形

```hlsl
//=====================================================
// shaders/MeshletRender.hlsl
//=====================================================

#define MESHLET_MAX_VERTICES   64
#define MESHLET_MAX_PRIMITIVES 126
#define THREAD_GROUP_SIZE      128

struct MeshVertex
{
    float3 position;
    float3 normal;
    float4 tangent;
    float2 uv;
};

struct Meshlet
{
    uint vertexOffset;
    uint vertexCount;
    uint primitiveOffset;
    uint primitiveCount;
};

struct VertexOutput
{
    float4 position   : SV_Position;
    float3 positionWS : POSITION_WS;
    float3 normalWS   : NORMAL_WS;
    float2 uv         : TEXCOORD;
    nointerpolation uint meshletIndex : MESHLET_INDEX;
};

//--- バインドレスで取得(第33章)---
cbuffer DrawConstants : register(b1)
{
    uint vertexBufferIndex;
    uint meshletBufferIndex;
    uint vertexIndexBufferIndex;
    uint primitiveBufferIndex;
    uint instanceBufferIndex;
    uint3 padding;
};

//-----------------------------------------------------
// 三角形の展開
//-----------------------------------------------------
uint3 UnpackPrimitive(uint packed)
{
    return uint3(
        packed        & 0x3FF,
        (packed >> 10) & 0x3FF,
        (packed >> 20) & 0x3FF);
}

//-----------------------------------------------------
// メッシュシェーダー
//-----------------------------------------------------
[outputtopology("triangle")]
[numthreads(THREAD_GROUP_SIZE, 1, 1)]
void MSMain(
    uint groupThreadId : SV_GroupThreadID,
    uint groupId       : SV_GroupID,
    out vertices VertexOutput outVertices[MESHLET_MAX_VERTICES],
    out indices  uint3        outIndices[MESHLET_MAX_PRIMITIVES])
{
    //--- バッファを取得 ---
    StructuredBuffer<Meshlet> meshlets =
        ResourceDescriptorHeap[meshletBufferIndex];

    const Meshlet meshlet = meshlets[groupId];

    //--- ① 出力数を宣言する(必須)---
    SetMeshOutputCounts(meshlet.vertexCount, meshlet.primitiveCount);

    //--- ② 頂点を出力 ---
    if (groupThreadId < meshlet.vertexCount)
    {
        StructuredBuffer<MeshVertex> vertices =
            ResourceDescriptorHeap[vertexBufferIndex];
        StructuredBuffer<uint> vertexIndices =
            ResourceDescriptorHeap[vertexIndexBufferIndex];

        const uint globalIndex =
            vertexIndices[meshlet.vertexOffset + groupThreadId];
        const MeshVertex vertex = vertices[globalIndex];

        VertexOutput output;
        output.positionWS = mul(float4(vertex.position, 1.0f), world).xyz;
        output.position   = mul(float4(output.positionWS, 1.0f), viewProjection);
        output.normalWS   = mul(float4(vertex.normal, 0.0f), normalMatrix).xyz;
        output.uv         = vertex.uv;
        output.meshletIndex = groupId;

        outVertices[groupThreadId] = output;
    }

    //--- ③ 三角形を出力 ---
    if (groupThreadId < meshlet.primitiveCount)
    {
        StructuredBuffer<uint> primitives =
            ResourceDescriptorHeap[primitiveBufferIndex];

        const uint packed = primitives[meshlet.primitiveOffset + groupThreadId];
        outIndices[groupThreadId] = UnpackPrimitive(packed);
    }
}
```

**重要な点が 4 つあります。**

**① `[outputtopology]` は必須**

```hlsl
[outputtopology("triangle")]
```

**`"line"` や `"point"` も指定できます。**

**② `SetMeshOutputCounts` を最初に呼ぶ**

**これを呼ぶ前に出力配列へ書き込むと、未定義動作です。**

**そして、1 回しか呼べません。**

**③ `out vertices` と `out indices`**

**専用の修飾子です。** 通常の `out` とは違います。

**配列のサイズは、コンパイル時定数でなければなりません。**

**④ スレッド数と出力数は独立**

```hlsl
[numthreads(128, 1, 1)]     // 128 スレッド
out vertices ... [64]        // 64 頂点
out indices  ... [126]       // 126 三角形
```

**128 スレッドで、64 頂点と 126 三角形を処理します。**

**スレッド数を 128 にしているのは、頂点(64)と三角形(126)の両方を 1 パスで処理するためです。**

**第2章 2.3.1 節の warp = 32 の 4 倍で、無駄がありません。**

### 36.5.2 バインドレスとの組み合わせ

**第33章のバインドレスが、ここで威力を発揮します。**

**メッシュシェーダーには、入力アセンブラがありません。** すべてのデータをシェーダー側で読む必要があります。

**従来の方式なら、5 つの SRV をテーブルでバインドすることになります。**

**バインドレスなら、インデックスを渡すだけです。**

```hlsl
StructuredBuffer<MeshVertex> vertices =
    ResourceDescriptorHeap[vertexBufferIndex];
```

**第33章 33.4.4 節で示した「究極形」が、ここで自然な選択になります。**

### 36.5.3 描画する

```cpp
void Renderer::DrawMeshlets(const RenderObject& object)
{
    const auto& mesh = m_meshes[object.meshIndex];

    //--- 定数を設定 ---
    DrawConstants constants{};
    constants.vertexBufferIndex      = mesh.vertexSrvIndex;
    constants.meshletBufferIndex     = mesh.meshletSrvIndex;
    constants.vertexIndexBufferIndex = mesh.vertexIndexSrvIndex;
    constants.primitiveBufferIndex   = mesh.primitiveSrvIndex;

    const auto alloc = m_uploadBuffer.AllocateConstants(sizeof(constants));
    alloc.Write(constants);
    m_commandList->SetGraphicsRootConstantBufferView(1, alloc.gpuAddress);

    //--- メッシュレット数だけディスパッチ ---
    m_commandList6->DispatchMesh(mesh.meshletCount, 1, 1);
}
```

**`ID3D12GraphicsCommandList6` が必要です。**

**`IASetVertexBuffers` も `IASetIndexBuffer` も呼びません。**

**`DispatchMesh` の引数は、コンピュートシェーダーの `Dispatch` と同じ形です。**

```cpp
void DispatchMesh(
    UINT ThreadGroupCountX,
    UINT ThreadGroupCountY,
    UINT ThreadGroupCountZ);
```

**上限があります。** 1 次元あたり 65535 です。

**メッシュレットが 65535 を超える場合、2 次元に分割します。**

---

## 36.6 Amplification シェーダー

### 36.6.1 何をするものか

**メッシュシェーダーの前段で動きます。**

```
Amplification Shader → Mesh Shader
     (何個起動するか決める)
```

**役割は 2 つです。**

| 役割 | 説明 |
|---|---|
| **カリング** | 不要なメッシュレットを起動しない |
| **増幅** | 1 つの入力から複数のメッシュシェーダーを起動 |

**「Amplification(増幅)」という名前は、後者に由来します。** LOD の動的な分割などに使えます。

**本書では、カリングに使います。**

### 36.6.2 実装

```hlsl
//=====================================================
// shaders/MeshletCulling.hlsl
//=====================================================

#define AS_GROUP_SIZE 32

struct MeshletBounds
{
    float3 center;
    float  radius;
    float3 coneAxis;
    float  coneCutoff;
};

//--- メッシュシェーダーへ渡すデータ ---
struct MeshletPayload
{
    uint meshletIndices[AS_GROUP_SIZE];
};

groupshared MeshletPayload gPayload;

//-----------------------------------------------------
// 視錐台カリング(第25章 25.3.4 節と同じ)
//-----------------------------------------------------
bool IsInFrustum(float3 center, float radius)
{
    [unroll]
    for (int i = 0; i < 6; ++i)
    {
        if (dot(frustumPlanes[i].xyz, center) + frustumPlanes[i].w < -radius)
        {
            return false;
        }
    }
    return true;
}

//-----------------------------------------------------
// 法線錐による裏面カリング(36.6.3 節)
//-----------------------------------------------------
bool IsBackfacing(float3 center, float3 coneAxis, float coneCutoff)
{
    const float3 toCamera = normalize(cameraPosition - center);
    return dot(coneAxis, toCamera) < coneCutoff;
}

[numthreads(AS_GROUP_SIZE, 1, 1)]
void ASMain(uint groupThreadId : SV_GroupThreadID,
            uint dispatchId    : SV_DispatchThreadID)
{
    bool visible = false;

    if (dispatchId < meshletCount)
    {
        StructuredBuffer<MeshletBounds> boundsBuffer =
            ResourceDescriptorHeap[boundsBufferIndex];

        const MeshletBounds bounds = boundsBuffer[dispatchId];

        //--- ワールド空間へ ---
        const float3 center = mul(float4(bounds.center, 1.0f), world).xyz;
        const float  radius = bounds.radius * maxScale;

        const float3 coneAxis =
            normalize(mul(float4(bounds.coneAxis, 0.0f), world).xyz);

        //--- 2 段階のカリング ---
        visible = IsInFrustum(center, radius)
               && !IsBackfacing(center, coneAxis, bounds.coneCutoff);
    }

    //--- 可視のものだけを詰める ---
    // 第32章 32.4.2 節の Wave Intrinsics を使う
    const uint visibleCount = WaveActiveCountBits(visible);

    if (visible)
    {
        const uint slot = WavePrefixCountBits(visible);
        gPayload.meshletIndices[slot] = dispatchId;
    }

    //--- メッシュシェーダーを起動 ---
    DispatchMesh(visibleCount, 1, 1, gPayload);
}
```

**`DispatchMesh` は、Amplification シェーダー内でも呼びます。**

**引数の最後がペイロードです。** メッシュシェーダーへ渡すデータを指定します。

**`WavePrefixCountBits` で、可視のものを詰めています。**

**第32章 32.4.4 節で使った手法と同じです。**

**メッシュシェーダー側では、ペイロードを受け取ります。**

```hlsl
[outputtopology("triangle")]
[numthreads(THREAD_GROUP_SIZE, 1, 1)]
void MSMain(
    uint groupThreadId : SV_GroupThreadID,
    uint groupId       : SV_GroupID,
    in payload MeshletPayload payload,          // ← 追加
    out vertices VertexOutput outVertices[MESHLET_MAX_VERTICES],
    out indices  uint3        outIndices[MESHLET_MAX_PRIMITIVES])
{
    //--- ペイロードから実際のメッシュレット番号を取得 ---
    const uint meshletIndex = payload.meshletIndices[groupId];

    StructuredBuffer<Meshlet> meshlets =
        ResourceDescriptorHeap[meshletBufferIndex];

    const Meshlet meshlet = meshlets[meshletIndex];

    // ... 以下同じ ...
}
```

**ペイロードのサイズには上限があります。** 16KB です。

### 36.6.3 法線錐カリング

**メッシュレット単位の裏面カリングです。**

```
メッシュレットの全面が、カメラから見て裏を向いている
  → 描画しない
```

**36.4.4 節で計算した `coneAxis` と `coneCutoff` を使います。**

```
coneAxis:   平均法線
coneCutoff: 最も外れた法線との内積
```

**判定は 1 回の内積で済みます。**

```hlsl
dot(coneAxis, toCamera) < coneCutoff
```

**これが成り立つなら、すべての面が裏を向いています。**

**通常の裏面カリング(第16章 16.4 節)は、ラスタライザで三角形ごとに行われます。**

**法線錐なら、メッシュレット全体(126 三角形)を 1 回の判定で捨てられます。**

> **平面的なメッシュレットほど効果が高い**
>
> `coneCutoff` は、メッシュレット内の法線のばらつきを表します。
>
> | 形状 | `coneCutoff` | カリングの効果 |
> |---|---|---|
> | 平面 | 1.0 に近い | **高い** |
> | 緩やかな曲面 | 0.5 程度 | 中程度 |
> | 球状 | -1.0 に近い | **効かない** |
>
> **36.4.3 節で述べた「空間的に近い三角形をまとめる」という改善が、ここでも効きます。**

---

## 36.7 従来方式との比較

### 36.7.1 何が良くなるか

| 項目 | 従来 | **メッシュシェーダー** |
|---|---|---|
| **カリングの粒度** | オブジェクト単位 | **メッシュレット単位** |
| **裏面カリング** | 三角形ごと(ラスタライザ) | **メッシュレット単位で先に** |
| **CPU の負荷** | ドローコールごと | **同じ** |
| **柔軟性** | 固定 | **ジオメトリを生成できる** |

**最大の利点は、カリングの粒度です。**

**第34章の GPU カリングは、オブジェクト単位でした。**

```
オブジェクトが視錐台の中 → 全メッシュレットを描く
```

**メッシュシェーダーなら、メッシュレット単位で判定できます。**

```
巨大な地形の 10% だけが見える → 10% のメッシュレットだけを描く
```

### 36.7.2 何が難しくなるか

**正直に書きます。**

| 課題 | 説明 |
|---|---|
| **前処理が必要** | メッシュレットの生成 |
| **メモリが増える** | 頂点の重複、境界データ |
| **デバッグが難しい** | 入力アセンブラがないため、ツールの支援が減る |
| **対応環境が限られる** | Turing 以降 |

**メモリについて、具体的に見ます。**

```
従来:
  頂点 10 万個 × 48 バイト = 4.8 MB
  インデックス 30 万個 × 4 バイト = 1.2 MB
  合計 6.0 MB

メッシュレット:
  頂点 10 万個 × 48 バイト = 4.8 MB
  頂点インデックス 12 万個 × 4 バイト = 0.48 MB
  プリミティブ 10 万個 × 4 バイト = 0.4 MB
  メッシュレット 1600 個 × 16 バイト = 0.026 MB
  境界 1600 個 × 32 バイト = 0.051 MB
  合計 5.8 MB
```

**この例ではほぼ同等ですが、メッシュレット間で頂点が重複するため、増えることもあります。**

**プリミティブを 10bit に詰めた効果**(36.4.2 節)が、ここで効いています。

### 36.7.3 いつ使うべきか

| 場面 | 推奨 |
|---|---|
| **巨大なモデル**(数十万三角形) | **メッシュシェーダー** |
| **地形** | **メッシュシェーダー** |
| 小さなオブジェクトが多数 | 従来 + インスタンシング |
| UI、パーティクル | 従来 |
| **手続き的なジオメトリ** | **メッシュシェーダー** |

**本書のシーンでは、効果が限定的です。**

**モデルが小さく、メッシュレット数が少ないためです。**

**それでも実装する価値はあります。** 仕組みを理解しておけば、必要になったときに使えます。

---

## ✅ 本章のゴール:メッシュシェーダーで描画できる

### Step 1:対応を確認する

```
[Info ] GraphicsDevice.cpp(297): mesh shader   : Tier 1
[Info ] GraphicsDevice.cpp(311): ch.36 mesh shader   : OK
```

**第7章 7.5.5 節の判定関数が使われます。**

### Step 2:メッシュレットを生成する

```
[Info ] MeshletBuilder.cpp(118): meshlets built: 1587 meshlets from 98304 triangles
[Info ] MeshletBuilder.cpp(122):   avg vertices/meshlet: 58.3
[Info ] MeshletBuilder.cpp(123):   avg triangles/meshlet: 61.9
```

**平均が上限(64 / 126)に近いほど、効率的です。**

**三角形の平均が 62 なのは、頂点数の制約が先に来ているためです。**

**36.4.3 節で述べた「空間的な並べ替え」を行うと、改善します。**

### Step 3:PSO を作る

```
[Info ] PipelineState.cpp(188): mesh shader PSO created in 4.21 ms
```

**アラインメントの誤りがあると、ここで失敗します。**

```
D3D12 ERROR: CreatePipelineState: The subobject stream is malformed.
```

**36.3.3 節の `alignas(void*)` を確認してください。**

### Step 4:描画する

**従来方式と同じ絵になることを確認してください。**

**キー入力で切り替えられるようにしておくと便利です。**

```cpp
if (input.WasKeyPressed('M'))
{
    m_useMeshShader = !m_useMeshShader;
    LOG_INFO(L"rendering: {}",
             m_useMeshShader ? L"mesh shader" : L"traditional");
}
```

**第33章 33.2.2 節、第34章 34.5.2 節と同じ、比較による検証です。**

### Step 5:メッシュレットを可視化する

**`meshletIndex` から色を生成します。**

```hlsl
float3 MeshletColor(uint index)
{
    //--- ハッシュで擬似的な色を作る ---
    const uint hash = index * 2654435761u;
    return float3(
        ((hash      ) & 0xFF) / 255.0f,
        ((hash >>  8) & 0xFF) / 255.0f,
        ((hash >> 16) & 0xFF) / 255.0f);
}

float4 PSMain(VertexOutput input) : SV_Target
{
#if DEBUG_MESHLETS
    return float4(MeshletColor(input.meshletIndex), 1.0f);
#endif
    // ...
}
```

**モデルが色とりどりのパッチに分かれて見えます。**

**分割の品質が目で確認できます。**

| 見え方 | 評価 |
|---|---|
| **まとまったパッチ** | **良い分割** |
| 細長い帯状 | 局所性が悪い |
| 飛び飛び | 順序が悪い |

### Step 6:Amplification シェーダーを有効にする

**カリングの効果を測ります。**

```cpp
struct MeshletStats
{
    std::uint32_t totalMeshlets    = 0;
    std::uint32_t visibleMeshlets  = 0;
};
```

**GPU から読み戻すには、第34章 34.5.2 節の手法を使います。**

```
meshlets: 412 / 1587 visible (26%)
```

**カメラを動かすと、可視率が変化します。**

### Step 7:法線錐カリングの効果を見る

**視錐台カリングだけの場合と比較します。**

```cpp
visible = IsInFrustum(center, radius);
// && !IsBackfacing(...);   ← 無効化
```

| 設定 | 可視メッシュレット |
|---|---|
| カリングなし | 1587 |
| 視錐台のみ | 780 |
| **+ 法線錐** | **412** |

**閉じた形状では、およそ半分が裏を向いています。**

### Step 8:`SetMeshOutputCounts` を忘れてみる

```hlsl
// SetMeshOutputCounts(meshlet.vertexCount, meshlet.primitiveCount);
```

**何も描画されないか、不正な描画になります。**

**デバッグレイヤーが警告する場合もあります。**

**確認したら元に戻してください。**

### Step 9:GPU Trace で比較する

**第29章 29.3 節の手法で、従来方式と比較します。**

| 項目 | 予想 |
|---|---|
| 頂点処理 | メッシュシェーダーのほうが少ない(カリング後) |
| ラスタライズ | 同等 |
| **全体** | **モデルが大きいほど有利** |

**小さなモデルでは、差が出ないか、むしろ遅くなることもあります。**

**36.7.3 節の判断基準を、実測で確認してください。**

---

### 本章の達成状態

- [ ] `SupportsMeshShader()` で対応を確認した
- [ ] `PipelineStateSubobject` テンプレートを自作した
- [ ] `alignas(void*)` を指定した
- [ ] 既定値を持つストリームを用意した
- [ ] メッシュレットを生成した(64 頂点 / 126 三角形)
- [ ] プリミティブを 10bit × 3 に詰めた
- [ ] 境界球と法線錐を計算した
- [ ] `[outputtopology]` を指定した
- [ ] `SetMeshOutputCounts` を最初に呼んでいる
- [ ] バインドレスでバッファを取得している
- [ ] `DispatchMesh` で描画した
- [ ] Amplification シェーダーでカリングした
- [ ] Wave Intrinsics で可視メッシュレットを詰めた
- [ ] **メッシュシェーダーで描画できた**

---

## トラブルシューティング

| 症状 | 原因 | 対処 |
|---|---|---|
| PSO 生成に失敗 | アラインメントの誤り | 36.3.3 |
| 同上 | `ID3D12Device2` を使っていない | 36.3.5 |
| コンパイルエラー | シェーダーモデルが 6.5 未満 | `-T ms_6_6`(36.2.2) |
| 何も描画されない | `SetMeshOutputCounts` 忘れ | 36.5.1 |
| 一部の三角形が欠ける | 出力数の計算誤り | メッシュレットの構造を確認 |
| 頂点がおかしい | インデックスの展開誤り | `UnpackPrimitive`(36.5.1) |
| Amplification が効かない | ペイロードのサイズ超過 | 16KB 以内に |
| カリングしすぎ | 境界球が小さい | 余裕を持たせる(第25章 Step 6) |
| 法線錐が効かない | 形状が球状 | 36.6.3 のコラム |
| メッシュレットが多すぎる | 分割の局所性が悪い | 36.4.3 のコラム |
| 従来方式より遅い | モデルが小さい | 36.7.3 |

---

## まとめ

**1. 入力アセンブラがなくなる。**
頂点バッファもインデックスバッファも、シェーダーが自分で読みます。**バインドレス(第33章)が前提になります。**

**2. パイプラインステートストリームは、拡張可能な形式。**
`alignas(void*)` を忘れると、サブオブジェクトの解釈がずれます。

**3. 64 頂点 / 126 三角形が推奨値。**
64 は warp の 2 倍です。**第2章 2.3.1 節と第32章 32.3.4 節と、同じ根拠です。**

**4. `SetMeshOutputCounts` を最初に呼ぶ。**
出力配列へ書き込む前に、必ず 1 回だけ呼びます。

**5. Amplification シェーダーで、メッシュレット単位のカリング。**
第34章のオブジェクト単位より、はるかに細かく判定できます。

**6. 法線錐は、平面的なメッシュレットで効く。**
126 三角形をまとめて捨てられます。**球状の形状では効果がありません。**

**7. モデルが大きいほど有利。**
小さなオブジェクトでは、前処理とメモリのコストが上回ります。

**8. 第14章の罠が、ここでも適用される。**
`SampleMask`、`SampleDesc`、`PrimitiveTopology`。**既定値を持つ関数を作る理由は、形式が変わっても同じです。**

次章では、いよいよレイトレーシングを扱います。**加速構造の構築、ステートオブジェクト、シェーダーテーブル。** そして第2章 2.1.1 節で触れた **Shader Execution Reordering** と **Opacity Micromap** —— 第7章 7.5.3 節で「DXR 1.2」として確認した機能を、実際に使います。

---

## 参考リンク

| 内容 | URL |
|---|---|
| メッシュシェーダー仕様 | https://microsoft.github.io/DirectX-Specs/d3d/MeshShader.html |
| メッシュシェーダーの概要 | https://learn.microsoft.com/ja-jp/windows/win32/direct3d12/mesh-shader |
| `DispatchMesh` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/nf-d3d12-id3d12graphicscommandlist6-dispatchmesh |
| パイプラインステートストリーム | https://learn.microsoft.com/ja-jp/windows/win32/direct3d12/pipeline-state-object-interface |
| NVIDIA のメッシュシェーダー入門 | https://developer.nvidia.com/blog/introduction-turing-mesh-shaders/ |
| DirectXMesh | https://github.com/microsoft/DirectXMesh |
