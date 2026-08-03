# 第37章 DirectX Raytracing

**第5部の最終章です。**

第15章で三角形を描いてから、ここまでずっとラスタライズを扱ってきました。**三角形を画面に投影し、覆われたピクセルを塗る。** これが 30 年以上、リアルタイムグラフィックスの基本でした。

**レイトレーシングは、逆から考えます。**

```
ラスタライズ:  三角形 → どのピクセルを覆うか
レイトレーシング:  ピクセル → どの三角形に当たるか
```

**この逆転が、多くのものを自然に扱えるようにします。**

| 表現 | ラスタライズ | レイトレーシング |
|---|---|---|
| 影 | シャドウマップ(第27章) | **光源へ光線を飛ばすだけ** |
| 反射 | 環境マップ、SSR | **反射方向へ光線を飛ばすだけ** |
| 屈折 | 困難 | **屈折方向へ光線を飛ばすだけ** |
| 間接光 | 事前計算、SSAO | **ランダムに光線を飛ばすだけ** |

**第27章のシャドウマップで戦ったアーティファクト —— アクネ、ピーターパン、ギザギザ —— は、レイトレーシングでは原理的に発生しません。**

**代わりに、別の難しさがあります。** ノイズ、コスト、そして**実装の複雑さ**です。

**本章のゴール**
加速構造を構築し、ステートオブジェクトとシェーダーテーブルを手で組み立てる。レイトレーシングによる反射を実装し、SER で性能を改善する。

---

## 37.1 対応を確認する

### 37.1.1 3 つの Tier

**第7章 7.5.3 節で問い合わせた `OPTIONS5` を使います。**

```cpp
D3D12_FEATURE_DATA_D3D12_OPTIONS5 options5{};
if (QueryFeature(device, D3D12_FEATURE_D3D12_OPTIONS5, options5))
{
    m_caps.raytracingTier = options5.RaytracingTier;
}
```

| 値 | 意味 | 対応世代 |
|---|---|---|
| `TIER_NOT_SUPPORTED` | 非対応 | Pascal 以前 |
| **`TIER_1_0`** | DXR 1.0 | Turing 以降 |
| **`TIER_1_1`** | + インラインレイトレーシング | Turing 以降 |
| **`TIER_1_2`** | **+ SER と Opacity Micromap** | **Ada 以降** |

**第7章 7.5.3 節で、こう書きました。**

> **レイトレーシングの Tier に注目してください。**
> | **`D3D12_RAYTRACING_TIER_1_2`** | **DXR 1.2(SER と Opacity Micromap の両方)** |

**第7章 7.5.5 節の判定関数も、既に用意してあります。**

```cpp
bool DeviceCaps::SupportsRaytracing() const noexcept
{
    return raytracingTier >= D3D12_RAYTRACING_TIER_1_1;
}

bool DeviceCaps::SupportsDxr12() const noexcept
{
    return raytracingTier >= D3D12_RAYTRACING_TIER_1_2
        && maxShaderModel >= D3D_SHADER_MODEL_6_9;
}
```

**DXR 1.2 には Shader Model 6.9 も必要です。**

### 37.1.2 GTX 16 シリーズの注意

**第2章 2.1.1 節のコラムを、改めて確認してください。**

> GTX 1650 / 1660 などの GTX 16 シリーズは、アーキテクチャとしては Turing です。しかし **RT Core と Tensor Core が搭載されていません**。メッシュシェーダーや VRS は動きますが、**レイトレーシングは Pascal と同じくソフトウェア実行になります。**
>
> **「Turing なのに第37章が動かない」という事故の典型がこれです。**

**予告した通り、本章が該当します。**

**API 上は対応していると報告されますが、実用的な速度は出ません。**

### 37.1.3 インターフェース

**レイトレーシングには、専用のインターフェースが必要です。**

```cpp
ComPtr<ID3D12Device5>             device5;
ComPtr<ID3D12GraphicsCommandList4> commandList4;

HR_TRY(device.As(&device5));
HR_TRY(commandList.As(&commandList4));
```

**第6章 6.1.4 節のインターフェースバージョンです。**

| インターフェース | 追加されたメソッド |
|---|---|
| `ID3D12Device5` | `CreateStateObject`、`GetRaytracingAccelerationStructurePrebuildInfo` |
| `ID3D12GraphicsCommandList4` | `BuildRaytracingAccelerationStructure`、`DispatchRays`、`SetPipelineState1` |

---

## 37.2 加速構造

### 37.2.1 なぜ必要か

**光線と三角形の交差判定を、素朴に行うとこうなります。**

```
1 本の光線 × 10 万三角形 = 10 万回の判定
```

**1920×1080 のピクセルすべてで行うと、2000 億回です。** 現実的ではありません。

**加速構造は、空間を階層的に分割します。**

```
        [全体]
       /      \
   [左半分]  [右半分]
   /    \      /    \
 ...    ...  ...    ...
```

**光線が箱に当たらなければ、その中の三角形はすべて無視できます。**

**判定回数は、おおよそ `log(三角形数)` に比例します。**

```
10 万三角形 → 約 17 回の階層を降りる
```

### 37.2.2 BLAS と TLAS

**2 段階の構造になっています。**

```
TLAS(Top Level)
  ├─ インスタンス 0 → BLAS A(変換行列つき)
  ├─ インスタンス 1 → BLAS A(別の変換行列)
  ├─ インスタンス 2 → BLAS B
  └─ ...

BLAS(Bottom Level)
  A: 三角形の集合(モデル空間)
  B: 三角形の集合(モデル空間)
```

| | BLAS | TLAS |
|---|---|---|
| 内容 | **ジオメトリ** | **インスタンス** |
| 座標系 | モデル空間 | ワールド空間 |
| 構築コスト | **高い** | 低い |
| 更新頻度 | モデルが変形したときのみ | **毎フレーム** |

**この分離が重要です。**

**同じモデルを 100 個配置しても、BLAS は 1 つで済みます。** TLAS に 100 個のインスタンスを登録するだけです。

**第25章のシーングラフと、対応関係があります。**

```
RenderObject.meshIndex     → BLAS
RenderObject.worldMatrix   → インスタンスの変換行列
```

### 37.2.3 BLAS を構築する

**まず、ジオメトリを記述します。**

```cpp
D3D12_RAYTRACING_GEOMETRY_DESC geometryDesc{};
geometryDesc.Type  = D3D12_RAYTRACING_GEOMETRY_TYPE_TRIANGLES;
geometryDesc.Flags = D3D12_RAYTRACING_GEOMETRY_FLAG_OPAQUE;

auto& triangles = geometryDesc.Triangles;
triangles.Transform3x4 = 0;                    // 変換なし

triangles.IndexFormat  = DXGI_FORMAT_R32_UINT;
triangles.IndexCount   = mesh.indexCount;
triangles.IndexBuffer  = mesh.indexBuffer->GetGPUVirtualAddress();

triangles.VertexFormat = DXGI_FORMAT_R32G32B32_FLOAT;
triangles.VertexCount  = mesh.vertexCount;
triangles.VertexBuffer.StartAddress =
    mesh.vertexBuffer->GetGPUVirtualAddress();
triangles.VertexBuffer.StrideInBytes = sizeof(MeshVertex);
```

**頂点フォーマットは位置だけです。**

**加速構造は、位置しか使いません。** 法線や UV は、ヒット時にシェーダーが自分で読みます(37.5.3 節)。

**`FLAG_OPAQUE` が重要です。**

| フラグ | 意味 |
|---|---|
| **`OPAQUE`** | **AnyHit シェーダーを呼ばない** |
| `NO_DUPLICATE_ANYHIT_INVOCATION` | AnyHit の重複呼び出しを避ける |

**不透明と分かっているなら `OPAQUE` を指定してください。** 大幅に速くなります。

**次に、必要なサイズを問い合わせます。**

```cpp
D3D12_BUILD_RAYTRACING_ACCELERATION_STRUCTURE_INPUTS inputs{};
inputs.Type  = D3D12_RAYTRACING_ACCELERATION_STRUCTURE_TYPE_BOTTOM_LEVEL;
inputs.Flags = D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BUILD_FLAG_PREFER_FAST_TRACE;
inputs.DescsLayout   = D3D12_ELEMENTS_LAYOUT_ARRAY;
inputs.NumDescs      = 1;
inputs.pGeometryDescs = &geometryDesc;

D3D12_RAYTRACING_ACCELERATION_STRUCTURE_PREBUILD_INFO prebuildInfo{};
device5->GetRaytracingAccelerationStructurePrebuildInfo(
    &inputs, &prebuildInfo);

LOG_INFO(L"BLAS size: {} bytes, scratch: {} bytes",
         prebuildInfo.ResultDataMaxSizeInBytes,
         prebuildInfo.ScratchDataSizeInBytes);
```

**構築フラグの選択が、性能を左右します。**

| フラグ | 用途 |
|---|---|
| **`PREFER_FAST_TRACE`** | **構築は遅いが、光線が速い。静的なジオメトリ向け** |
| `PREFER_FAST_BUILD` | 構築が速いが、光線が遅い。毎フレーム作り直す場合 |
| `ALLOW_UPDATE` | 部分的な更新を許可 |
| `MINIMIZE_MEMORY` | メモリを節約 |

**BLAS は静的なので `PREFER_FAST_TRACE`、TLAS は毎フレーム作り直すので `PREFER_FAST_BUILD` が定石です。**

**バッファを確保します。**

```cpp
//--- 結果を格納するバッファ ---
auto resultDesc = MakeBufferDesc(
    prebuildInfo.ResultDataMaxSizeInBytes,
    D3D12_RESOURCE_FLAG_ALLOW_UNORDERED_ACCESS);   // ← 必須

const auto heapProps = MakeHeapProperties(D3D12_HEAP_TYPE_DEFAULT);

HR_TRY(device->CreateCommittedResource(
    &heapProps, D3D12_HEAP_FLAG_NONE, &resultDesc,
    D3D12_RESOURCE_STATE_RAYTRACING_ACCELERATION_STRUCTURE,   // ← 専用の状態
    nullptr,
    IID_PPV_ARGS(&blas)));

Core::SetDebugNameF(blas.Get(), L"BLAS_{}", mesh.name);
```

**`ALLOW_UNORDERED_ACCESS` フラグが必須です。**

**そして、初期状態が `RAYTRACING_ACCELERATION_STRUCTURE` です。**

**この状態は特別で、他の状態へ遷移できません。** 加速構造は、生成から破棄までこの状態のままです。

**構築します。**

```cpp
D3D12_BUILD_RAYTRACING_ACCELERATION_STRUCTURE_DESC buildDesc{};
buildDesc.Inputs                          = inputs;
buildDesc.DestAccelerationStructureData   = blas->GetGPUVirtualAddress();
buildDesc.ScratchAccelerationStructureData = scratch->GetGPUVirtualAddress();

commandList4->BuildRaytracingAccelerationStructure(&buildDesc, 0, nullptr);

//--- 完了を待つ UAV バリア ---
const auto barrier = MakeUavBufferBarrier(blas.Get());
// ... Barrier グループとして発行 ...
```

**UAV バリアが必要です**(第32章 32.2.4 節)。

**構築の完了を待たずに TLAS を作ると、不正な結果になります。**

### 37.2.4 TLAS を構築する

**インスタンスを記述します。**

```cpp
struct D3D12_RAYTRACING_INSTANCE_DESC
{
    FLOAT  Transform[3][4];                  // 3×4 の変換行列
    UINT   InstanceID                  : 24;
    UINT   InstanceMask                : 8;
    UINT   InstanceContributionToHitGroupIndex : 24;
    UINT   Flags                       : 8;
    D3D12_GPU_VIRTUAL_ADDRESS AccelerationStructure;
};
```

**ビットフィールドを使った、密なレイアウトです。**

**`Transform` は 3×4 の行優先です。**

**第17章 17.3.3 節で採用した行ベクトル流儀とは、転置の関係にあります。**

```cpp
void SetInstanceTransform(D3D12_RAYTRACING_INSTANCE_DESC& desc,
                          const Math::Matrix4x4& world)
{
    //--- 行ベクトル流儀の 4×4 から、DXR の 3×4 へ ---
    // DXR は列ベクトル流儀(M * v)なので、転置して詰める
    for (int row = 0; row < 3; ++row)
    {
        for (int col = 0; col < 4; ++col)
        {
            desc.Transform[row][col] = world.m[col][row];
        }
    }
}
```

**本書で唯一、転置が必要になる箇所です。**

**第17章 17.3.4 節で「転置は一切登場しません」と書きましたが、DXR の API がそう定めているため、ここだけは避けられません。**

**各フィールドの意味です。**

| フィールド | 用途 |
|---|---|
| `InstanceID` | シェーダーから `InstanceID()` で取得できる **24bit の任意値** |
| **`InstanceMask`** | **光線のマスクと AND を取り、0 なら無視** |
| `InstanceContributionToHitGroupIndex` | シェーダーテーブルの索引(37.4.3 節) |
| `Flags` | カリングの制御など |

**`InstanceMask` が便利です。**

```cpp
constexpr UINT kMaskOpaque      = 0x01;
constexpr UINT kMaskTransparent = 0x02;
constexpr UINT kMaskShadowCaster = 0x04;
```

**影用の光線では `kMaskShadowCaster` だけを対象にする、といった使い分けができます。**

**構築は BLAS とほぼ同じです。**

```cpp
inputs.Type = D3D12_RAYTRACING_ACCELERATION_STRUCTURE_TYPE_TOP_LEVEL;
inputs.Flags = D3D12_RAYTRACING_ACCELERATION_STRUCTURE_BUILD_FLAG_PREFER_FAST_BUILD;
inputs.NumDescs     = instanceCount;
inputs.InstanceDescs = instanceBuffer->GetGPUVirtualAddress();
```

**`pGeometryDescs` ではなく `InstanceDescs` を使います。** 共用体です。

**インスタンスの記述は、GPU から読める場所に置く必要があります。** 第21章のリングバッファを使えます。

---

## 37.3 ステートオブジェクト

### 37.3.1 PSO との違い

**レイトレーシングでは、複数のシェーダーが 1 つのオブジェクトにまとまります。**

```
ステートオブジェクト
  ├─ RayGeneration シェーダー
  ├─ Miss シェーダー
  ├─ ヒットグループ 0
  │    ├─ ClosestHit シェーダー
  │    ├─ AnyHit シェーダー(任意)
  │    └─ Intersection シェーダー(任意)
  ├─ ヒットグループ 1
  ├─ ルートシグネチャ(グローバル)
  ├─ ルートシグネチャ(ローカル)
  └─ 各種設定
```

**第36章のパイプラインステートストリームと似た、サブオブジェクトの列で記述します。**

**ただし形式が違います。**

```cpp
typedef struct D3D12_STATE_OBJECT_DESC {
    D3D12_STATE_OBJECT_TYPE      Type;
    UINT                         NumSubobjects;
    const D3D12_STATE_SUBOBJECT* pSubobjects;
} D3D12_STATE_OBJECT_DESC;

typedef struct D3D12_STATE_SUBOBJECT {
    D3D12_STATE_SUBOBJECT_TYPE Type;
    const void*                pDesc;    // ← ポインタ
} D3D12_STATE_SUBOBJECT;
```

**第36章はデータを直接埋め込みましたが、こちらはポインタです。**

**したがって、参照先の寿命を管理する必要があります。**

### 37.3.2 `CD3DX12_STATE_OBJECT_DESC` を使わずに書く

**`d3dx12.h` には便利なヘルパーがありますが、使いません**(第1章 1.3.1 節)。

**寿命の管理を自分で行うため、専用のビルダーを作ります。**

```cpp
// src/Graphics/StateObjectBuilder.h
#pragma once
#include "std_import.h"

namespace Graphics
{
    //-----------------------------------------------------------
    // ステートオブジェクトの記述を組み立てる。
    //
    // サブオブジェクトはポインタで参照されるため、
    // 生成が完了するまで、すべての実体を保持する必要がある。
    //-----------------------------------------------------------
    class StateObjectBuilder
    {
    public:
        //--- DXIL ライブラリ ---
        void AddLibrary(const ShaderBlob& blob,
                        std::span<const std::wstring_view> exportNames);

        //--- ヒットグループ ---
        void AddHitGroup(std::wstring_view name,
                         std::wstring_view closestHit,
                         std::wstring_view anyHit = {},
                         std::wstring_view intersection = {});

        //--- シェーダー設定 ---
        void SetShaderConfig(UINT maxPayloadSize, UINT maxAttributeSize);

        //--- パイプライン設定 ---
        void SetPipelineConfig(UINT maxRecursionDepth);

        //--- ルートシグネチャ ---
        void SetGlobalRootSignature(ID3D12RootSignature* rootSignature);
        void AddLocalRootSignature(ID3D12RootSignature* rootSignature,
                                   std::span<const std::wstring_view> associations);

        //--- 構築 ---
        Core::Result<Microsoft::WRL::ComPtr<ID3D12StateObject>>
            Build(ID3D12Device5* device5, std::wstring_view name);

    private:
        std::vector<D3D12_STATE_SUBOBJECT> m_subobjects;

        //--- 実体を保持する(ポインタの参照先)---
        std::deque<D3D12_DXIL_LIBRARY_DESC>       m_libraries;
        std::deque<D3D12_HIT_GROUP_DESC>          m_hitGroups;
        std::deque<D3D12_RAYTRACING_SHADER_CONFIG> m_shaderConfigs;
        std::deque<D3D12_RAYTRACING_PIPELINE_CONFIG> m_pipelineConfigs;
        std::deque<D3D12_GLOBAL_ROOT_SIGNATURE>   m_globalRootSigs;
        std::deque<D3D12_LOCAL_ROOT_SIGNATURE>    m_localRootSigs;
        std::deque<D3D12_SUBOBJECT_TO_EXPORTS_ASSOCIATION> m_associations;

        //--- 文字列の実体 ---
        std::deque<std::wstring>              m_strings;
        std::deque<std::vector<LPCWSTR>>      m_stringArrays;
        std::deque<std::vector<D3D12_EXPORT_DESC>> m_exportDescs;
    };
}
```

**`std::deque` を使っているのが要点です。**

**`std::vector` では、`push_back` で再配置が起こり、既存のポインタが無効になります。**

**`std::deque` は、既存要素のアドレスが変わらないことを保証します。**

**第25章 25.4.2 節で「ポインタではなくインデックスで階層を表現する」と書いたのと、逆の判断です。** ここでは API がポインタを要求するので、安定したアドレスが必要になります。

### 37.3.3 DXIL ライブラリ

**レイトレーシングのシェーダーは、`lib_6_x` としてコンパイルします。**

**第13章 13.1.2 節のターゲット文字列の表に、既に含まれていました。**

| 接頭辞 | ステージ |
|---|---|
| `lib_` | ライブラリ(レイトレーシング) |

```
dxc -T lib_6_9 -Fo Raytracing.cso Raytracing.hlsl
```

**エントリポイントを指定しません。** すべての関数がエクスポートされます。

```cpp
void StateObjectBuilder::AddLibrary(
    const ShaderBlob& blob,
    std::span<const std::wstring_view> exportNames)
{
    //--- エクスポート名を保持 ---
    auto& descs = m_exportDescs.emplace_back();
    descs.reserve(exportNames.size());

    for (const auto name : exportNames)
    {
        auto& stored = m_strings.emplace_back(name);

        D3D12_EXPORT_DESC desc{};
        desc.Name           = stored.c_str();
        desc.ExportToRename = nullptr;
        desc.Flags          = D3D12_EXPORT_FLAG_NONE;
        descs.push_back(desc);
    }

    //--- ライブラリ記述 ---
    auto& library = m_libraries.emplace_back();
    library.DXILLibrary = blob.Bytecode();
    library.NumExports  = static_cast<UINT>(descs.size());
    library.pExports    = descs.data();

    m_subobjects.push_back({
        D3D12_STATE_SUBOBJECT_TYPE_DXIL_LIBRARY,
        &library });
}
```

**エクスポート名を明示することで、使わない関数を除外できます。**

**すべてエクスポートしたい場合は、`NumExports = 0` にします。**

### 37.3.4 シェーダー設定

**ペイロードと属性のサイズを宣言します。**

```cpp
void StateObjectBuilder::SetShaderConfig(UINT maxPayloadSize,
                                         UINT maxAttributeSize)
{
    auto& config = m_shaderConfigs.emplace_back();
    config.MaxPayloadSizeInBytes   = maxPayloadSize;
    config.MaxAttributeSizeInBytes = maxAttributeSize;

    m_subobjects.push_back({
        D3D12_STATE_SUBOBJECT_TYPE_RAYTRACING_SHADER_CONFIG,
        &config });
}
```

| 項目 | 説明 |
|---|---|
| **ペイロード** | 光線が運ぶデータ。RayGen ↔ ClosestHit/Miss |
| **属性** | 交差の情報。三角形なら重心座標(8 バイト) |

**サイズは小さいほど速くなります。** レジスタを消費するためです。

```cpp
struct RayPayload
{
    Math::Vector3 color;
    float         hitDistance;
    UINT          recursionDepth;
    UINT          padding[3];
};
static_assert(sizeof(RayPayload) == 32);
```

**32 バイト程度に抑えるのが目安です。**

### 37.3.5 再帰の深さ

```cpp
void StateObjectBuilder::SetPipelineConfig(UINT maxRecursionDepth)
{
    auto& config = m_pipelineConfigs.emplace_back();
    config.MaxTraceRecursionDepth = maxRecursionDepth;

    m_subobjects.push_back({
        D3D12_STATE_SUBOBJECT_TYPE_RAYTRACING_PIPELINE_CONFIG,
        &config });
}
```

**上限は 31 です。**

**しかし、実用的には 2〜3 に抑えるべきです。**

```
深さ 1: 一次光線のみ
深さ 2: + 反射 1 回
深さ 3: + 反射 2 回、または反射 + 影
```

**深いほどスタックを消費し、性能が落ちます。**

**宣言した深さを超えると、未定義動作です。** デバイスロストの原因になります。

---

## 37.4 シェーダーテーブル

### 37.4.1 何をするものか

**「どの光線が、どのシェーダーを実行するか」を対応づけるテーブルです。**

```
シェーダーテーブル
  ├─ RayGeneration レコード
  ├─ Miss レコード × N
  └─ HitGroup レコード × M
```

**各レコードは、次の形です。**

```
[シェーダー識別子(32 バイト)][ローカルルート引数(可変)]
```

**シェーダー識別子は、ステートオブジェクトから取得します。**

```cpp
ComPtr<ID3D12StateObjectProperties> properties;
HR_TRY(stateObject.As(&properties));

const void* rayGenId = properties->GetShaderIdentifier(L"RayGenMain");
```

**`D3D12_SHADER_IDENTIFIER_SIZE_IN_BYTES` は 32 です。**

### 37.4.2 アラインメント

**2 つの制約があります。**

```cpp
D3D12_RAYTRACING_SHADER_RECORD_BYTE_ALIGNMENT       // = 32
D3D12_RAYTRACING_SHADER_TABLE_BYTE_ALIGNMENT        // = 64
```

| 対象 | アラインメント |
|---|---|
| **各レコード** | **32 バイト** |
| **テーブルの先頭** | **64 バイト** |

**第18章 18.2 節の定数バッファ(256 バイト)、第20章 20.4.1 節のテクスチャ(256 / 512 バイト)に続く、3 つ目のアラインメント要件です。**

**第18章 18.2.2 節で作った `AlignUp` を使います。**

```cpp
const UINT recordSize = static_cast<UINT>(AlignUp(
    D3D12_SHADER_IDENTIFIER_SIZE_IN_BYTES + localRootArgumentSize,
    D3D12_RAYTRACING_SHADER_RECORD_BYTE_ALIGNMENT));
```

### 37.4.3 テーブルを構築する

```cpp
// src/Graphics/ShaderTable.h

class ShaderTable
{
public:
    struct Record
    {
        const void*            shaderIdentifier = nullptr;
        std::span<const std::byte> localRootArguments;
    };

    Core::Status Build(ID3D12Device* device,
                       std::span<const Record> records,
                       std::wstring_view name);

    D3D12_GPU_VIRTUAL_ADDRESS Address() const noexcept;
    UINT RecordSize() const noexcept { return m_recordSize; }
    UINT RecordCount() const noexcept { return m_recordCount; }

private:
    Microsoft::WRL::ComPtr<ID3D12Resource> m_buffer;
    UINT m_recordSize  = 0;
    UINT m_recordCount = 0;
};
```

```cpp
Core::Status ShaderTable::Build(ID3D12Device* device,
                                std::span<const Record> records,
                                std::wstring_view name)
{
    //--- 最大のローカル引数サイズを求める ---
    std::size_t maxArgumentSize = 0;
    for (const auto& record : records)
    {
        maxArgumentSize = std::max(maxArgumentSize,
                                   record.localRootArguments.size());
    }

    m_recordSize = static_cast<UINT>(AlignUp(
        D3D12_SHADER_IDENTIFIER_SIZE_IN_BYTES + maxArgumentSize,
        D3D12_RAYTRACING_SHADER_RECORD_BYTE_ALIGNMENT));

    m_recordCount = static_cast<UINT>(records.size());

    const UINT64 totalSize = static_cast<UINT64>(m_recordSize) * m_recordCount;

    //--- UPLOAD ヒープに作る(毎フレーム更新しないなら DEFAULT でもよい)---
    const auto heapProps = MakeHeapProperties(D3D12_HEAP_TYPE_UPLOAD);
    const auto desc      = MakeBufferDesc(totalSize);

    HR_TRY(device->CreateCommittedResource(
        &heapProps, D3D12_HEAP_FLAG_NONE, &desc,
        D3D12_RESOURCE_STATE_GENERIC_READ, nullptr,
        IID_PPV_ARGS(&m_buffer)));

    Core::SetDebugName(m_buffer.Get(), name);

    //--- 書き込む ---
    std::byte* mapped = nullptr;
    const D3D12_RANGE readRange{ 0, 0 };
    HR_TRY(m_buffer->Map(0, &readRange,
                         reinterpret_cast<void**>(&mapped)));

    for (std::size_t i = 0; i < records.size(); ++i)
    {
        std::byte* dest = mapped + i * m_recordSize;

        //--- ① シェーダー識別子 ---
        std::memcpy(dest, records[i].shaderIdentifier,
                    D3D12_SHADER_IDENTIFIER_SIZE_IN_BYTES);

        //--- ② ローカルルート引数 ---
        if (!records[i].localRootArguments.empty())
        {
            std::memcpy(
                dest + D3D12_SHADER_IDENTIFIER_SIZE_IN_BYTES,
                records[i].localRootArguments.data(),
                records[i].localRootArguments.size());
        }
    }

    m_buffer->Unmap(0, nullptr);

    LOG_INFO(L"shader table: {} records x {} bytes",
             m_recordCount, m_recordSize);

    return {};
}
```

**第15章 15.3.2 節の「書くだけ」の原則を守っています。**

### 37.4.4 索引の計算

**どのヒットグループが実行されるかは、次の式で決まります。**

```
索引 = RayContributionToHitGroupIndex
     + MultiplierForGeometryContributionToHitGroupIndex × GeometryIndex
     + InstanceContributionToHitGroupIndex
```

| 項 | どこで指定するか |
|---|---|
| `RayContributionToHitGroupIndex` | `TraceRay` の引数(光線の種類) |
| `MultiplierFor...` | `TraceRay` の引数(通常はヒットグループの種類数) |
| `GeometryIndex` | BLAS 内のジオメトリ番号 |
| `InstanceContributionToHitGroupIndex` | インスタンス記述(37.2.4 節) |

**この計算が、レイトレーシングで最も分かりにくい部分です。**

**本書のように「光線は 2 種類(通常と影)、マテリアルごとにヒットグループ」という構成なら、こうなります。**

```
ヒットグループの並び:
  [0] マテリアル 0 / 通常光線
  [1] マテリアル 0 / 影光線
  [2] マテリアル 1 / 通常光線
  [3] マテリアル 1 / 影光線
  ...
```

```cpp
//--- インスタンス側 ---
instanceDesc.InstanceContributionToHitGroupIndex = materialIndex * 2;
```

```hlsl
//--- 通常光線 ---
TraceRay(scene, flags, mask,
         0,      // RayContributionToHitGroupIndex
         2,      // MultiplierForGeometryContribution
         0,      // MissShaderIndex
         ray, payload);

//--- 影光線 ---
TraceRay(scene, flags, mask,
         1,      // ← 1 つずらす
         2,
         1,      // 影用の Miss シェーダー
         ray, shadowPayload);
```

> **バインドレスなら簡略化できる**
>
> **第33章のバインドレスを使えば、ヒットグループを 1 つに統一できます。**
>
> マテリアルの違いは、シェーダー内でインデックスを引くだけになります。
>
> ```hlsl
> const uint materialIndex = InstanceID();   // 24bit の任意値
> const MaterialData material = gMaterials[materialIndex];
> ```
>
> **`InstanceContributionToHitGroupIndex` を使う必要がなくなります。**
>
> **本書はこの方式を採ります。** 索引の計算に悩まずに済みます。

---

## 37.5 シェーダーを書く

### 37.5.1 グローバルとローカル

**ルートシグネチャが 2 種類あります。**

| 種類 | 適用範囲 | 設定方法 |
|---|---|---|
| **グローバル** | すべてのシェーダー | `SetComputeRootSignature` |
| **ローカル** | レコードごと | シェーダーテーブルに埋め込む |

**バインドレスを使うなら、グローバルだけで足ります。**

```cpp
D3D12_ROOT_PARAMETER1 params[2]{};

//--- b0: シーン定数 ---
params[0].ParameterType = D3D12_ROOT_PARAMETER_TYPE_CBV;
params[0].Descriptor.ShaderRegister = 0;

//--- t0: TLAS ---
params[1].ParameterType = D3D12_ROOT_PARAMETER_TYPE_SRV;
params[1].Descriptor.ShaderRegister = 0;

versioned.Desc_1_1.Flags =
      D3D12_ROOT_SIGNATURE_FLAG_CBV_SRV_UAV_HEAP_DIRECTLY_INDEXED
    | D3D12_ROOT_SIGNATURE_FLAG_SAMPLER_HEAP_DIRECTLY_INDEXED;
```

**加速構造は、ルートディスクリプタで渡せます。**

**出力先の UAV も、バインドレスで扱えます。**

### 37.5.2 RayGeneration

```hlsl
//=====================================================
// shaders/Raytracing.hlsl
//=====================================================

RaytracingAccelerationStructure gScene : register(t0);

cbuffer RayConstants : register(b0)
{
    row_major float4x4 invViewProjection;
    float4 cameraPosition;
    float4 lightDirection;
    uint   outputTextureIndex;
    uint   materialBufferIndex;
    uint   maxRecursion;
    uint   frameIndex;
};

struct RayPayload
{
    float3 color;
    float  hitDistance;
    uint   recursionDepth;
};

[shader("raygeneration")]
void RayGenMain()
{
    const uint2 pixel = DispatchRaysIndex().xy;
    const uint2 dimensions = DispatchRaysDimensions().xy;

    //--- ピクセルから光線を作る ---
    const float2 uv = (pixel + 0.5f) / dimensions;
    const float2 ndc = float2(uv.x * 2.0f - 1.0f, 1.0f - uv.y * 2.0f);

    //--- 逆行列でワールド空間へ ---
    const float4 nearPoint = mul(float4(ndc, 0.0f, 1.0f), invViewProjection);
    const float4 farPoint  = mul(float4(ndc, 1.0f, 1.0f), invViewProjection);

    const float3 origin    = nearPoint.xyz / nearPoint.w;
    const float3 direction = normalize(farPoint.xyz / farPoint.w - origin);

    RayDesc ray;
    ray.Origin    = origin;
    ray.Direction = direction;
    ray.TMin      = 0.001f;
    ray.TMax      = 10000.0f;

    RayPayload payload;
    payload.color          = float3(0, 0, 0);
    payload.hitDistance    = -1.0f;
    payload.recursionDepth = 0;

    TraceRay(gScene,
             RAY_FLAG_NONE,
             0xFF,            // InstanceMask
             0, 0, 0,         // ヒットグループとミスの索引
             ray, payload);

    //--- バインドレスで出力(第33章)---
    RWTexture2D<float4> output = ResourceDescriptorHeap[outputTextureIndex];
    output[pixel] = float4(payload.color, 1.0f);
}
```

**`invViewProjection` を使う点に注目してください。**

**第17章の数学ライブラリには、一般的な逆行列がありません。** `InverseAffine` は射影行列に使えません(第17章 17.4.4 節)。

**しかし、ビュー行列と射影行列を別々に逆変換すれば済みます。**

```cpp
//--- 射影の逆行列は、要素から直接構築できる ---
Math::Matrix4x4 InversePerspective(float fovY, float aspect,
                                   float nearZ, float farZ)
{
    const float h = 1.0f / std::tan(fovY * 0.5f);
    const float w = h / aspect;
    const float range = farZ / (farZ - nearZ);

    Math::Matrix4x4 result{};
    result.m[0][0] = 1.0f / w;
    result.m[1][1] = 1.0f / h;
    result.m[2][3] = -1.0f / (range * nearZ);
    result.m[3][2] = 1.0f;
    result.m[3][3] = 1.0f / nearZ;
    return result;
}

//--- ビューの逆行列は InverseAffine で足りる ---
const auto invView = Math::InverseAffine(camera.ViewMatrix());
const auto invProj = InversePerspective(...);
const auto invViewProj = invProj * invView;
```

**第17章 17.4.4 節の `InverseAffine` が、ここでも役立ちました。**

### 37.5.3 ClosestHit

```hlsl
struct MeshVertex
{
    float3 position;
    float3 normal;
    float4 tangent;
    float2 uv;
};

[shader("closesthit")]
void ClosestHitMain(inout RayPayload payload,
                    in BuiltInTriangleIntersectionAttributes attributes)
{
    //--- 重心座標 ---
    const float3 barycentrics = float3(
        1.0f - attributes.barycentrics.x - attributes.barycentrics.y,
        attributes.barycentrics.x,
        attributes.barycentrics.y);

    //--- インスタンス ID からマテリアルを引く(37.4.4 節のコラム)---
    const uint materialIndex = InstanceID();

    StructuredBuffer<MaterialData> materials =
        ResourceDescriptorHeap[materialBufferIndex];
    const MaterialData material = materials[materialIndex];

    //--- 頂点データを読む ---
    StructuredBuffer<MeshVertex> vertices =
        ResourceDescriptorHeap[NonUniformResourceIndex(material.vertexBufferIndex)];
    StructuredBuffer<uint> indices =
        ResourceDescriptorHeap[NonUniformResourceIndex(material.indexBufferIndex)];

    const uint triangleIndex = PrimitiveIndex();
    const uint3 tri = uint3(
        indices[triangleIndex * 3 + 0],
        indices[triangleIndex * 3 + 1],
        indices[triangleIndex * 3 + 2]);

    //--- 補間 ---
    const float3 normal = normalize(
          vertices[tri.x].normal * barycentrics.x
        + vertices[tri.y].normal * barycentrics.y
        + vertices[tri.z].normal * barycentrics.z);

    const float2 uv =
          vertices[tri.x].uv * barycentrics.x
        + vertices[tri.y].uv * barycentrics.y
        + vertices[tri.z].uv * barycentrics.z;

    //--- ワールド空間の法線 ---
    const float3 normalWS = normalize(
        mul(normal, (float3x3)ObjectToWorld3x4()));

    //--- アルベド(第33章のバインドレス)---
    Texture2D diffuseTexture =
        ResourceDescriptorHeap[NonUniformResourceIndex(material.diffuseTextureIndex)];
    const float3 albedo =
        diffuseTexture.SampleLevel(gSampler, uv, 0).rgb * material.diffuseColor.rgb;

    //--- 直接光(第24章と同じ Lambert)---
    const float3 L = -normalize(lightDirection.xyz);
    const float nDotL = saturate(dot(normalWS, L));

    //--- 影の判定 ---
    const float3 hitPosition = WorldRayOrigin()
                             + WorldRayDirection() * RayTCurrent();

    float shadow = 1.0f;
    if (nDotL > 0.0f)
    {
        shadow = TraceShadowRay(hitPosition + normalWS * 0.001f, L);
    }

    payload.color = albedo * nDotL * shadow;
    payload.hitDistance = RayTCurrent();
}
```

**ラスタライズとの対応が見えます。**

| 処理 | ラスタライズ | レイトレーシング |
|---|---|---|
| 頂点の補間 | **ラスタライザが自動**(第15章 15.6 節) | **手動**(重心座標) |
| 法線変換 | 頂点シェーダー(第24章 24.4 節) | `ObjectToWorld3x4()` |
| ライティング | ピクセルシェーダー | 同じ計算 |
| 影 | シャドウマップ(第27章) | **光線 1 本** |

**`SampleLevel` を使っている**点に注目してください。

**レイトレーシングでは、UV の微分が計算できません。** ミップレベルの自動選択ができないため、明示的に指定します。

**`0` を指定すると最高解像度になります。** 距離に応じたミップ選択は、自分で実装する必要があります(第20章 20.6.5 節の課題が再燃します)。

### 37.5.4 影の光線

```hlsl
struct ShadowPayload
{
    bool hit;
};

float TraceShadowRay(float3 origin, float3 direction)
{
    RayDesc ray;
    ray.Origin    = origin;
    ray.Direction = direction;
    ray.TMin      = 0.001f;
    ray.TMax      = 10000.0f;

    ShadowPayload payload;
    payload.hit = true;      // 何も当たらなければ Miss が false にする

    TraceRay(gScene,
             RAY_FLAG_ACCEPT_FIRST_HIT_AND_END_SEARCH
           | RAY_FLAG_SKIP_CLOSEST_HIT_SHADER,
             0xFF,
             0, 0,
             1,              // 影用の Miss シェーダー
             ray, payload);

    return payload.hit ? 0.0f : 1.0f;
}

[shader("miss")]
void ShadowMissMain(inout ShadowPayload payload)
{
    payload.hit = false;
}
```

**2 つのフラグが重要です。**

| フラグ | 効果 |
|---|---|
| **`ACCEPT_FIRST_HIT_AND_END_SEARCH`** | **最初のヒットで探索を打ち切る** |
| **`SKIP_CLOSEST_HIT_SHADER`** | **ClosestHit を実行しない** |

**影の判定には「当たったかどうか」しか要りません。** 最も近いヒットを探す必要がないので、大幅に速くなります。

**第27章のシャドウマップと比べてください。**

```
シャドウマップ:  解像度、バイアス、PCF、カスケード…
レイトレーシング:  光線 1 本
```

**アーティファクトが原理的に発生しません。**

**代わりに、コストは高くなります。** ピクセルごとに光線を追跡するためです。

---

## 37.6 Shader Execution Reordering

### 37.6.1 何を解決するか

**第2章 2.3.1 節で説明したダイバージェンスが、レイトレーシングでは深刻です。**

> 同じ warp に属する 32 スレッドは、原則として同じ命令を同時に実行します。ここで条件分岐が入り、16 スレッドが `if` 側、16 スレッドが `else` 側に行くとどうなるか。
> GPU は **両方の経路を順番に実行し、それぞれで不要なレーンをマスクします。**

**光線は、隣接するピクセルでも別々の場所に当たります。**

```
warp 内の 32 スレッド:
  スレッド 0  → 金属に当たった   → 金属のシェーダー
  スレッド 1  → 木に当たった     → 木のシェーダー
  スレッド 2  → 空に抜けた       → Miss シェーダー
  ...
```

**すべての経路が順番に実行されます。**

**SER は、実行前にスレッドを並べ替えます。**

```
並べ替え後:
  スレッド 0〜9   → 全部金属
  スレッド 10〜20 → 全部木
  スレッド 21〜31 → 全部 Miss
```

**同じ処理をするスレッドがまとまり、ダイバージェンスが減ります。**

### 37.6.2 `HitObject` による分離

**SER の導入とともに、`HitObject` という抽象が追加されました。**

**従来の `TraceRay` は、探索とシェーディングを一体で行います。**

```
TraceRay → 探索 → ClosestHit を実行
```

**`HitObject` は、これを分離します。**

```
HitObject::TraceRay → 探索のみ(HitObject を返す)
      ↓
MaybeReorderThread  → 並べ替え
      ↓
HitObject::Invoke   → ClosestHit を実行
```

**この分離には、SER 以外の利点もあります。**

| 利点 | 説明 |
|---|---|
| **共通処理の集約** | 頂点の取得と補間を RayGen に置ける |
| **可視判定の軽量化** | ヒットシェーダーを呼ばずに距離が分かる |
| **`RayQuery` との統合** | インライン方式でも使える |

### 37.6.3 実装

```hlsl
[shader("raygeneration")]
void RayGenSerMain()
{
    // ... 光線の生成 ...

    RayPayload payload;
    payload.color = float3(0, 0, 0);

    //--- ① 探索のみ ---
    HitObject hit = HitObject::TraceRay(
        gScene, RAY_FLAG_NONE, 0xFF,
        0, 0, 0, ray, payload);

    //--- ② 並べ替え ---
    if (hit.IsHit())
    {
        //--- マテリアル番号をソートキーにする ---
        const uint materialIndex = hit.GetInstanceID();

        //--- 下位 8bit をキーとして使う ---
        MaybeReorderThread(materialIndex, 8);
    }
    else
    {
        //--- Miss はまとめる ---
        MaybeReorderThread(0xFF, 8);
    }

    //--- ③ シェーディング ---
    HitObject::Invoke(hit, payload);

    RWTexture2D<float4> output = ResourceDescriptorHeap[outputTextureIndex];
    output[DispatchRaysIndex().xy] = float4(payload.color, 1.0f);
}
```

**`MaybeReorderThread` は RayGeneration シェーダーでのみ使えます。**

**引数は 2 通りあります。**

```hlsl
//--- ① ソートキーを明示 ---
MaybeReorderThread(uint sortKey, uint numBits);

//--- ② HitObject の性質でソート ---
MaybeReorderThread(HitObject hit);

//--- ③ 両方 ---
MaybeReorderThread(HitObject hit, uint sortKey, uint numBits);
```

**②が最も簡単です。** ヒットしたジオメトリやシェーダーで自動的に並べ替えます。

**①は、アプリケーション固有の基準を使いたい場合に有効です。**

**「Maybe」という名前が示す通り、実装は並べ替えを行わなくても構いません。** 最小の実装は何もしないことです。

### 37.6.4 実際に並べ替えるかを確認する

**第7章 7.5.3 節で触れた `OPTIONS22` を使います。**

```cpp
D3D12_FEATURE_DATA_D3D12_OPTIONS22 options22{};
if (QueryFeature(device, D3D12_FEATURE_D3D12_OPTIONS22, options22))
{
    m_caps.serReorders = options22.ShaderExecutionReorderingActuallyReorders;
}
```

**第7章 7.5.3 節で、こう書きました。**

> 新しい `OPTIONS22` が必要になるのは、「SER の API は使えるが、このデバイスは実際に並べ替えを行うのか」というより細かい問いに答えるときだけです。

**その問いに答えるのが、ここです。**

**偽の場合、`MaybeReorderThread` は何もしません。** コードは動きますが、性能改善もありません。

**NVIDIA では Ada 世代以降が真を返します**(第2章 2.1.1 節)。

### 37.6.5 効果

**Microsoft のデモでは、RTX 4090 でおよそ 40% の改善が報告されています。**

**ただし、効果はシーンに強く依存します。**

| シーン | 効果 |
|---|---|
| **マテリアルが多様** | **大きい** |
| **パストレーシング**(多数の反射) | **非常に大きい** |
| 単一マテリアル | ほぼなし |
| 一次光線のみ | 小さい |

**本書のシーンでは、限定的です。**

**それでも実装しておく価値はあります。** シーンが複雑になったときに効いてきます。

---

## 37.7 Opacity Micromap

### 37.7.1 何を解決するか

**アルファテストのあるジオメトリ(葉、金網)が、レイトレーシングでは高コストになります。**

**第28章 28.3 節で扱った内容です。**

```
光線が三角形に当たる
  → AnyHit シェーダーを実行
  → テクスチャをサンプリング
  → α < しきい値なら「当たらなかった」ことにする
  → 探索を続行
```

**AnyHit シェーダーの実行が、繰り返し発生します。**

**OMM は、三角形を細分割して不透明度を事前計算します。**

```
三角形を 4^N の小三角形に分割
  各小三角形に状態を記録:
    ・完全に不透明
    ・完全に透明
    ・不明(AnyHit が必要)
```

**「完全に不透明」または「完全に透明」なら、AnyHit を呼ばずに判定できます。**

### 37.7.2 実装の概要

**本書では実装しません。** 理由は 2 つです。

- **本書のシーンにアルファテストのジオメトリがない**
- **OMM の構築には、テクスチャの解析が必要で、量が多い**

**概要だけ示します。**

```cpp
//--- ① OMM 配列を構築 ---
D3D12_RAYTRACING_OPACITY_MICROMAP_ARRAY_DESC ommDesc{};
// ... 細分割レベル、各三角形の状態 ...

//--- ② ジオメトリに関連づける ---
D3D12_RAYTRACING_GEOMETRY_OMM_TRIANGLES_DESC triangles{};
triangles.pTriangles      = &geometryDesc.Triangles;
triangles.pOmmLinkage     = &linkage;

geometryDesc.Type = D3D12_RAYTRACING_GEOMETRY_TYPE_OMM_TRIANGLES;
```

**NVIDIA は、テクスチャから OMM を生成するライブラリを提供しています。**

**必要になったときに、そちらを参照してください。**

---

## 37.8 ハイブリッド構成

### 37.8.1 全部をレイトレーシングしない

**一次光線までラスタライズし、二次光線だけレイトレーシングする**のが実用的です。

```
① ラスタライズ:  G-Buffer(位置、法線、マテリアル)を生成
② レイトレーシング:  反射・影・間接光
③ 合成
```

**利点は 2 つです。**

| 利点 | 説明 |
|---|---|
| **一次光線が速い** | ラスタライズのほうが効率的 |
| **既存の資産が使える** | 第24章〜第28章のコードがそのまま |

### 37.8.2 反射だけを追加する

**本書のパイプラインに、反射パスを追加します。**

```
①  シャドウマップ(第27章)
②  不透明・半透明(第28章)
③  ★ レイトレーシング反射
④  リゾルブ(第28章)
⑤  ブルーム(第26章)
⑥  合成
```

**③ では、既に描画された結果を入力に使います。**

```hlsl
[shader("raygeneration")]
void ReflectionRayGenMain()
{
    const uint2 pixel = DispatchRaysIndex().xy;

    //--- G-Buffer から情報を取得 ---
    Texture2D<float4> normalBuffer =
        ResourceDescriptorHeap[normalBufferIndex];
    Texture2D<float>  depthBuffer =
        ResourceDescriptorHeap[depthBufferIndex];

    const float depth = depthBuffer[pixel];

    //--- 空なら何もしない ---
#if USE_REVERSED_Z
    if (depth <= 0.0f) return;      // 第19章 19.5 節
#else
    if (depth >= 1.0f) return;
#endif

    const float3 normalWS = normalBuffer[pixel].xyz * 2.0f - 1.0f;

    //--- 深度からワールド座標を復元 ---
    const float3 positionWS = ReconstructWorldPosition(pixel, depth);

    //--- 反射方向 ---
    const float3 viewDir = normalize(positionWS - cameraPosition.xyz);
    const float3 reflectDir = reflect(viewDir, normalWS);

    RayDesc ray;
    ray.Origin    = positionWS + normalWS * 0.01f;
    ray.Direction = reflectDir;
    ray.TMin      = 0.001f;
    ray.TMax      = 100.0f;

    // ... TraceRay ...
}
```

**Reversed-Z の判定に注意してください**(第19章 19.5 節)。

**深度バッファを読むには、`R32_TYPELESS` で作る必要があります**(第27章 27.2.2 節)。

**第27章で既にそうしているので、変更は不要です。**

### 37.8.3 加速構造の更新

**動くオブジェクトがある場合、TLAS を毎フレーム作り直します。**

```cpp
void Renderer::UpdateTlas()
{
    //--- インスタンス記述を更新 ---
    std::vector<D3D12_RAYTRACING_INSTANCE_DESC> instances;
    instances.reserve(m_objects.size());

    for (const auto& object : m_objects)
    {
        D3D12_RAYTRACING_INSTANCE_DESC desc{};
        SetInstanceTransform(desc, object.worldMatrix);   // 37.2.4 節

        desc.InstanceID            = object.materialIndex;
        desc.InstanceMask          = 0xFF;
        desc.InstanceContributionToHitGroupIndex = 0;     // 統一(37.4.4 節)
        desc.Flags                 = D3D12_RAYTRACING_INSTANCE_FLAG_NONE;
        desc.AccelerationStructure =
            m_blas[object.meshIndex]->GetGPUVirtualAddress();

        instances.push_back(desc);
    }

    //--- リングバッファへ(第21章)---
    const auto alloc = m_uploadBuffer.Allocate(
        instances.size() * sizeof(D3D12_RAYTRACING_INSTANCE_DESC),
        D3D12_RAYTRACING_INSTANCE_DESCS_BYTE_ALIGNMENT);

    std::memcpy(alloc.cpuAddress, instances.data(),
                instances.size() * sizeof(D3D12_RAYTRACING_INSTANCE_DESC));

    //--- 再構築 ---
    // ... BuildRaytracingAccelerationStructure ...
}
```

**`D3D12_RAYTRACING_INSTANCE_DESCS_BYTE_ALIGNMENT` は 16 です。**

**BLAS は再構築しません。** ジオメトリが変形しない限り、そのまま使えます。

---

## 37.9 デバッグ

### 37.9.1 何が難しいか

**レイトレーシングは、これまでで最もデバッグが困難です。**

| 課題 | 説明 |
|---|---|
| **ドローコールがない** | `DispatchRays` 1 回のみ |
| **シェーダーの選択が動的** | どのヒットグループが実行されるか不明 |
| **再帰的** | スタックの状態が追いにくい |
| **加速構造が不透明** | 内部構造を見られない |

### 37.9.2 Nsight Graphics

**第29章で扱った Frame Debugger が、レイトレーシングにも対応しています。**

**確認できるもの:**

| 項目 | 用途 |
|---|---|
| **加速構造の可視化** | BLAS / TLAS の階層を表示 |
| **シェーダーテーブルの内容** | 索引の計算を検証 |
| **光線の統計** | 発射数、ヒット数 |
| ステートオブジェクトの構成 | エクスポートとヒットグループ |

**加速構造の可視化が特に有用です。**

**インスタンスの変換行列が誤っていれば、目で見て分かります。**

**37.2.4 節で転置が必要だと書きました。** 間違えると、オブジェクトがおかしな場所に配置されます。

### 37.9.3 Aftermath

**第31章の内容が、そのまま使えます。**

**ただし、注意点があります。**

**マーカーは `DispatchRays` の前後にしか打てません。**

```cpp
{
    GPU_MARKER(m_commandList.Get(), m_aftermathContext, "Raytracing");
    m_commandList4->DispatchRays(&dispatchDesc);
}
```

**「どのシェーダーで落ちたか」は、シェーダーデバッグ情報から特定します。**

**第13章 13.6 節の PDB 出力が、ここでも効きます。**

```
Active Shaders:
  Closest Hit Shader
    Source: Raytracing.hlsl
    Line 187:  const MaterialData material = materials[materialIndex];
               Crash location
```

**`InstanceID` が範囲外だった、といった原因が特定できます。**

### 37.9.4 段階的に確認する

**一度に全部を実装せず、段階を踏んでください。**

| 段階 | 確認内容 |
|---|---|
| **① 加速構造だけ** | 構築が成功するか |
| **② 単色出力** | `DispatchRays` が動くか |
| **③ 法線の可視化** | 交差判定が正しいか |
| **④ UV の可視化** | 頂点データの読み取りが正しいか |
| **⑤ テクスチャ** | バインドレスが機能するか |
| **⑥ ライティング** | 通常の描画と一致するか |
| **⑦ 反射** | 完成 |

**③ が最も重要です。**

```hlsl
[shader("closesthit")]
void DebugNormalHit(inout RayPayload payload,
                    in BuiltInTriangleIntersectionAttributes attributes)
{
    // ... 法線を計算 ...
    payload.color = normalWS * 0.5f + 0.5f;
}
```

**第23章 23.4.3 節の法線デバッグ表示と、同じ手法です。**

**ラスタライズの結果と比較すれば、一致するはずです。**

---

## ✅ 本章のゴール:レイトレースによる反射が出る

### Step 1:対応を確認する

```
[Info ] GraphicsDevice.cpp(296): raytracing    : Tier 1.2
[Info ] GraphicsDevice.cpp(302): SER reorders      : yes
[Info ] GraphicsDevice.cpp(312): ch.37 raytracing    : OK
[Info ] GraphicsDevice.cpp(313): ch.37 DXR 1.2       : OK
```

**`Tier 1.0` や `1.1` の場合、SER と OMM は使えません。**

**GTX 16 シリーズでは、動作はしますが極端に遅くなります**(37.1.2 節)。

### Step 2:加速構造を構築する

```
[Info ] AccelerationStructure.cpp(88): BLAS 'Bunny': 2.1 MB, scratch 0.8 MB
[Info ] AccelerationStructure.cpp(88): BLAS 'Plane': 0.01 MB, scratch 0.01 MB
[Info ] AccelerationStructure.cpp(142): TLAS: 336 instances, 0.02 MB
```

**BLAS のサイズは、頂点数に比例します。**

**スクラッチバッファは構築後に解放できます。**

### Step 3:単色を出力する

```hlsl
[shader("raygeneration")]
void RayGenMain()
{
    RWTexture2D<float4> output = ResourceDescriptorHeap[outputTextureIndex];
    output[DispatchRaysIndex().xy] = float4(1, 0, 1, 1);   // マゼンタ
}
```

**画面がマゼンタになれば、`DispatchRays` とシェーダーテーブルが機能しています。**

**ここで失敗する場合の原因:**

| 症状 | 原因 |
|---|---|
| 何も出ない | シェーダーテーブルの識別子が誤り |
| デバイスロスト | ペイロードサイズの宣言不足 |
| ステートオブジェクト生成に失敗 | エクスポート名の誤り |

### Step 4:法線を可視化する

**37.9.4 節の段階 ③ です。**

**ラスタライズの法線デバッグ表示(第23章 23.4.3 節)と比較してください。**

**一致すれば、加速構造と交差判定が正しく機能しています。**

**ずれている場合、37.2.4 節の転置を疑ってください。**

### Step 5:影を追加する

**シャドウマップ(第27章)と比較してください。**

| 項目 | シャドウマップ | レイトレーシング |
|---|---|---|
| アクネ | あり(要バイアス) | **なし** |
| ピーターパン | あり | **なし** |
| 境界のギザギザ | あり(要 PCF) | **なし** |
| 範囲の制限 | あり | **なし** |
| コスト | 低い | **高い** |

**第27章で戦ったアーティファクトが、すべて消えます。**

**代わりに、フレームレートが落ちます。**

### Step 6:反射を追加する

**37.8.2 節のハイブリッド構成です。**

**金属的なマテリアルで、周囲が映り込みます。**

**再帰の深さを変えてみてください。**

```cpp
builder.SetPipelineConfig(1);   // 反射なし
builder.SetPipelineConfig(2);   // 反射 1 回
builder.SetPipelineConfig(3);   // 反射 2 回
```

**深さ 2 と 3 の違いは、鏡が向かい合った場合などに現れます。**

### Step 7:SER の効果を測る

**37.6.3 節の実装に切り替えます。**

**第29章の GPU Trace で比較してください。**

| 設定 | 予想 |
|---|---|
| SER なし | 基準 |
| **SER あり** | **マテリアルが多いほど改善** |

**本書のシーンでは、差が小さいかもしれません。**

**マテリアルの種類を増やすと、効果が見えやすくなります。**

### Step 8:`OPTIONS22` を確認する

```cpp
if (!m_caps.serReorders)
{
    LOG_WARN(L"SER API is available but this device does not reorder");
}
```

**Ada 世代未満では、この警告が出ます。**

**コードは動きますが、性能改善はありません**(37.6.4 節)。

### Step 9:ペイロードサイズを間違えてみる

```cpp
builder.SetShaderConfig(
    8,      // ❌ 実際は 32 バイト必要
    8);
```

**デバイスロストになります。**

**第31章の Aftermath で解析してください。**

**確認したら元に戻してください。**

### Step 10:再帰の深さを超えてみる

```cpp
builder.SetPipelineConfig(1);   // 深さ 1 を宣言
```

```hlsl
//--- ClosestHit で TraceRay を呼ぶ(深さ 2 になる)---
TraceRay(gScene, ...);   // ❌ 宣言を超える
```

**未定義動作です。** デバイスロストになるか、不正な結果になります。

**確認したら元に戻してください。**

---

### 本章の達成状態

- [ ] `SupportsRaytracing()` と `SupportsDxr12()` で確認した
- [ ] BLAS を `PREFER_FAST_TRACE` で構築した
- [ ] TLAS を `PREFER_FAST_BUILD` で構築した
- [ ] インスタンスの変換行列を転置した
- [ ] UAV バリアで構築の完了を待っている
- [ ] `StateObjectBuilder` で寿命を管理した
- [ ] `std::deque` でポインタの安定性を確保した
- [ ] シェーダーテーブルのアラインメントを守った
- [ ] バインドレスでヒットグループを統一した
- [ ] `SampleLevel` で明示的にミップを指定した
- [ ] 影の光線に最適化フラグを付けた
- [ ] `HitObject` と `MaybeReorderThread` を実装した
- [ ] `OPTIONS22` で実際の並べ替えを確認した
- [ ] **レイトレーシングによる反射が表示された**

---

## トラブルシューティング

| 症状 | 原因 | 対処 |
|---|---|---|
| 加速構造の構築に失敗 | `ALLOW_UNORDERED_ACCESS` 忘れ | 37.2.3 |
| 同上 | 状態が誤り | `RAYTRACING_ACCELERATION_STRUCTURE`(37.2.3) |
| オブジェクトの位置がおかしい | 転置忘れ | 37.2.4 |
| 何も描画されない | シェーダー識別子の誤り | Step 3 |
| デバイスロスト | ペイロードサイズ不足 | 37.3.4 |
| 同上 | 再帰の深さ超過 | 37.3.5 |
| ステートオブジェクト生成に失敗 | エクスポート名の誤り | 37.3.3 |
| 同上 | サブオブジェクトの寿命 | `std::deque`(37.3.2) |
| 誤ったシェーダーが実行される | 索引の計算 | 37.4.4 |
| テクスチャがぼやける | ミップの選択 | `SampleLevel`(37.5.3) |
| 影が黒すぎる | 環境光を掛けている | 第27章 27.6.3 節 |
| SER の効果がない | `OPTIONS22` が偽 | 37.6.4 |
| 極端に遅い | GTX 16 シリーズ | 37.1.2 |

---

## まとめ

**1. 加速構造は 2 段階。**
BLAS はジオメトリ、TLAS はインスタンス。**同じモデルを 100 個配置しても、BLAS は 1 つで済みます。**

**2. 本書で唯一、転置が必要な場所。**
第17章 17.3.4 節で「転置は登場しない」と書きましたが、DXR の API が列ベクトル流儀なので、ここだけは避けられません。

**3. ステートオブジェクトはポインタで参照される。**
`std::vector` では再配置でポインタが無効になります。**`std::deque` を使ってください。**

**4. バインドレスがシェーダーテーブルを単純にする。**
ヒットグループを 1 つに統一でき、索引の計算に悩まずに済みます。

**5. 影は光線 1 本。**
第27章で戦ったアクネ、ピーターパン、ギザギザが、すべて原理的に発生しません。

**6. SER は `HitObject` と組み合わせる。**
探索とシェーディングを分離し、その間に並べ替えます。**第2章 2.3.1 節のダイバージェンスへの、ハードウェアによる答えです。**

**7. `OPTIONS22` で実際の並べ替えを確認する。**
API が使えることと、実際に並べ替えることは別です。

**8. ハイブリッドが実用的。**
一次光線はラスタライズ、二次光線をレイトレーシング。**第24章〜第28章のコードがそのまま活きます。**

---

## 第5部を終えて

**モダンな Direct3D 12 の機能を、ひと通り実装しました。**

| 章 | 得たもの |
|---|---|
| 第32章 | コンピュートシェーダーと Wave Intrinsics |
| 第33章 | バインドレス |
| 第34章 | インスタンシングと GPU 駆動の描画 |
| 第35章 | マルチスレッド化 |
| 第36章 | メッシュシェーダー |
| 第37章 | レイトレーシング |

**そして、第2章で調べた GPU の性質が、すべて設計判断として使われました。**

| 性質 | 使った場所 |
|---|---|
| **warp = 32** | 第32章(`numthreads`)、第36章(メッシュレット) |
| **SM と共有メモリ** | 第32章(occupancy) |
| **ダイバージェンス** | 第33章(`NonUniformResourceIndex`)、第37章(SER) |
| **RT Core** | 第37章 |
| **メッシュシェーダー対応** | 第36章 |

**第2章 2.3.2 節で示した表の通りになりました。**

**次章では、性能を測ります。** ここまで「実測で判断する」と何度も書いてきました。その手段を、最後に整えます。

---

## 参考リンク

| 内容 | URL |
|---|---|
| DirectX Raytracing 仕様 | https://microsoft.github.io/DirectX-Specs/d3d/Raytracing.html |
| Shader Execution Reordering | https://devblogs.microsoft.com/directx/shader-execution-reordering/ |
| SER の HLSL 仕様 | https://microsoft.github.io/hlsl-specs/proposals/0027-shader-execution-reordering.html |
| Opacity Micromaps | https://devblogs.microsoft.com/directx/omm/ |
| `DispatchRays` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/nf-d3d12-id3d12graphicscommandlist4-dispatchrays |
| DXR チュートリアル(NVIDIA) | https://developer.nvidia.com/rtx/raytracing/dxr/dx12-raytracing-tutorial-part-1 |
| RTX Path Tracing | https://github.com/NVIDIA-RTX/RTXPT |
