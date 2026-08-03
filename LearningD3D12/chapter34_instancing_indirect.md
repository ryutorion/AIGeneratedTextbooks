# 第34章 インスタンシングと ExecuteIndirect

第25章 25.3 節で、ドローコールの周辺コストを減らしました。第33章で、マテリアル切り替えをゼロにしました。

**しかし、ドローコールの数そのものは減っていません。**

```
objects 336/336, draws 336
```

**本章では、これを 1 回にします。**

そして、さらに先へ進みます。**「何を描くか」の判断を、CPU から GPU へ移します。**

```
これまで:  CPU がカリング → 可視オブジェクトのリストを作る → 描画
これから:  GPU がカリング → GPU が描画コマンドを生成 → 実行
```

**CPU は「全部のオブジェクトを渡す」だけになります。**

**本章のゴール**
インスタンシングで複数オブジェクトを 1 回のドローコールで描画する。コマンドシグネチャを構築し、`ExecuteIndirect` で GPU が生成したコマンドを実行する。

---

## 34.1 インスタンシング

### 34.1.1 第25章で保留した内容

**第25章 25.3.5 節では、概要だけを示しました。**

> **本書ではインスタンシングを第34章で本格的に扱います。** ここでは「そういう手段がある」ことだけ示します。
> **制約もあります。**
> - 同じメッシュ、同じマテリアルでなければならない
> - インスタンスごとに異なるテクスチャは使えない(**第33章のバインドレスで解決します**)

**前章でバインドレスを導入したので、2 つ目の制約が消えました。**

**インスタンスごとにマテリアルインデックスを持たせれば、別々のテクスチャを使えます。**

### 34.1.2 2 つの方式

**インスタンスごとのデータを渡す方法は、2 通りあります。**

| 方式 | 仕組み |
|---|---|
| **A. 入力アセンブラ** | 2 本目の頂点バッファをインスタンス単位で読む |
| **B. 構造化バッファ** | `SV_InstanceID` で配列を引く |

**方式 A は、第14章 14.4.5 節で触れた `InputSlotClass` を使います。**

```cpp
constexpr D3D12_INPUT_ELEMENT_DESC kInputElements[] = {
    //--- スロット 0:頂点ごと ---
    { "POSITION", 0, DXGI_FORMAT_R32G32B32_FLOAT, 0, 0,
      D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA, 0 },
    { "NORMAL",   0, DXGI_FORMAT_R32G32B32_FLOAT, 0, APPEND,
      D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA, 0 },

    //--- スロット 1:インスタンスごと ---
    { "WORLD", 0, DXGI_FORMAT_R32G32B32A32_FLOAT, 1, 0,
      D3D12_INPUT_CLASSIFICATION_PER_INSTANCE_DATA, 1 },   // ← 行列の 1 行目
    { "WORLD", 1, DXGI_FORMAT_R32G32B32A32_FLOAT, 1, 16,
      D3D12_INPUT_CLASSIFICATION_PER_INSTANCE_DATA, 1 },
    { "WORLD", 2, DXGI_FORMAT_R32G32B32A32_FLOAT, 1, 32,
      D3D12_INPUT_CLASSIFICATION_PER_INSTANCE_DATA, 1 },
    { "WORLD", 3, DXGI_FORMAT_R32G32B32A32_FLOAT, 1, 48,
      D3D12_INPUT_CLASSIFICATION_PER_INSTANCE_DATA, 1 },
};
```

**行列 1 つに 4 要素必要です。** `float4x4` を直接指定する手段がないためです。

**最後の引数 `InstanceDataStepRate` は、「何インスタンスごとに次のデータへ進むか」を指定します。** `1` なら毎インスタンス、`2` なら 2 インスタンスごとです。

**本書は方式 B を使います。**

**理由は 3 つです。**

- **入力レイアウトが単純なまま**(第26章 26.2.3 節と同じく、入力アセンブラへの依存を減らす)
- **構造体をそのまま使える**(4 分割が不要)
- **34.4 節の GPU カリングで、同じバッファを使い回せる**

### 34.1.3 構造化バッファ方式

```hlsl
struct InstanceData
{
    row_major float4x4 world;
    row_major float4x4 normalMatrix;    // 第24章 24.4 節
    uint   materialIndex;               // 第33章
    uint3  padding;
};

StructuredBuffer<InstanceData> gInstances : register(t0);

VSOutput VSMain(VSInput input, uint instanceId : SV_InstanceID)
{
    const InstanceData instance = gInstances[instanceId];

    VSOutput output;
    output.positionWS = mul(float4(input.position, 1.0f), instance.world).xyz;
    output.position   = mul(float4(output.positionWS, 1.0f), viewProjection);
    output.normalWS   = mul(float4(input.normal, 0.0f), instance.normalMatrix).xyz;
    output.uv         = input.uv;

    //--- マテリアルインデックスをピクセルシェーダーへ渡す ---
    output.materialIndex = instance.materialIndex;

    return output;
}
```

**`SV_InstanceID` は自動で渡されます**(第13章 13.1.3 節)。

**ピクセルシェーダー側で、バインドレスを使います。**

```hlsl
struct VSOutput
{
    float4 position      : SV_Position;
    float3 positionWS    : POSITION_WS;
    float3 normalWS      : NORMAL_WS;
    float2 uv            : TEXCOORD;
    nointerpolation uint materialIndex : MATERIAL_INDEX;   // ← 補間しない
};

float4 PSMain(VSOutput input) : SV_Target
{
    const MaterialData material = gMaterials[input.materialIndex];

    Texture2D diffuse = ResourceDescriptorHeap[
        NonUniformResourceIndex(material.diffuseTextureIndex)];

    // ...
}
```

**`nointerpolation` が重要です。**

**第15章 15.6.2 節で扱った修飾子です。** インデックスは整数なので、補間されては困ります。

**付け忘れると、三角形の内部で値が変化し、でたらめなマテリアルを参照します。**

### 34.1.4 インスタンスをまとめる

**同じメッシュを使うオブジェクトを、グループ化します。**

```cpp
struct InstanceBatch
{
    std::uint32_t meshIndex     = 0;
    std::uint32_t psoIndex      = 0;
    std::uint32_t instanceOffset = 0;    // バッファ内の開始位置
    std::uint32_t instanceCount  = 0;
};

void Renderer::BuildInstanceBatches(
    const std::vector<const RenderObject*>& visible,
    std::vector<InstanceBatch>& batches,
    std::vector<InstanceData>& instances)
{
    batches.clear();
    instances.clear();
    instances.reserve(visible.size());

    //--- メッシュと PSO でソート済みである前提(第25章 25.3.2 節)---
    std::uint32_t lastMesh = 0xFFFFFFFF;
    std::uint32_t lastPso  = 0xFFFFFFFF;

    for (const RenderObject* object : visible)
    {
        //--- 新しいバッチを開始するか ---
        if (object->meshIndex != lastMesh || object->psoIndex != lastPso)
        {
            InstanceBatch batch{};
            batch.meshIndex      = object->meshIndex;
            batch.psoIndex       = object->psoIndex;
            batch.instanceOffset = static_cast<std::uint32_t>(instances.size());
            batch.instanceCount  = 0;
            batches.push_back(batch);

            lastMesh = object->meshIndex;
            lastPso  = object->psoIndex;
        }

        //--- インスタンスデータを積む ---
        InstanceData data{};
        data.world         = object->worldMatrix;
        data.normalMatrix  = Math::Transpose(
                                 Math::InverseAffine(object->worldMatrix));
        data.materialIndex = object->materialIndex;
        instances.push_back(data);

        ++batches.back().instanceCount;
    }
}
```

**描画は、バッチごとに 1 回です。**

```cpp
for (const auto& batch : batches)
{
    if (batch.psoIndex != lastPso)
    {
        m_commandList->SetPipelineState(m_psos[batch.psoIndex].Get());
        lastPso = batch.psoIndex;
    }

    const auto& mesh = m_meshes[batch.meshIndex];
    m_commandList->IASetVertexBuffers(0, 1, &mesh.vbv);
    m_commandList->IASetIndexBuffer(&mesh.ibv);

    //--- インスタンスバッファ内の開始位置を渡す ---
    m_commandList->SetGraphicsRoot32BitConstant(
        1, batch.instanceOffset, 0);

    m_commandList->DrawIndexedInstanced(
        mesh.indexCount,
        batch.instanceCount,       // ← 1 ではない
        0, 0,
        0);
}
```

**`StartInstanceLocation`(第 5 引数)を使う方法もあります。**

```cpp
m_commandList->DrawIndexedInstanced(
    mesh.indexCount, batch.instanceCount, 0, 0,
    batch.instanceOffset);         // ← SV_InstanceID に加算される
```

**この場合、シェーダー側でオフセットを足す必要がありません。**

**ただし `SV_InstanceID` の値が変わるので、他の用途で使っている場合は注意が必要です。**

### 34.1.5 効果を測る

**第25章 25.3.3 節の統計に、バッチ数を追加します。**

```cpp
struct RenderStats
{
    // ...
    std::uint32_t instanceBatches = 0;
    std::uint32_t totalInstances  = 0;
};
```

```
インスタンシングなし:
  objects 336/336, draws 336, pso 2, mesh 3

インスタンシングあり:
  objects 336/336, draws 3, batches 3, instances 336, pso 2, mesh 3
```

**ドローコールが 336 回から 3 回になりました。**

**メッシュが 3 種類なので、バッチも 3 つです。**

---

## 34.2 コマンドシグネチャ

### 34.2.1 何をするものか

**GPU が生成したデータを、描画コマンドとして解釈する仕組みです。**

```
バッファの中身:  [引数][引数][引数] [引数][引数][引数] ...
                 └── コマンド 1 ──┘ └── コマンド 2 ──┘
                        ↓
              コマンドシグネチャが「これは DrawIndexed だ」と解釈
```

**`ExecuteIndirect` は、このバッファを読んでコマンドを実行します。**

**利点は 2 つです。**

| 利点 | 説明 |
|---|---|
| **CPU が介在しない** | GPU が決めた内容をそのまま実行 |
| **可変個数** | 何個実行するかも GPU が決められる |

### 34.2.2 引数の種類

```cpp
typedef enum D3D12_INDIRECT_ARGUMENT_TYPE {
    D3D12_INDIRECT_ARGUMENT_TYPE_DRAW                  = 0,
    D3D12_INDIRECT_ARGUMENT_TYPE_DRAW_INDEXED          = 1,
    D3D12_INDIRECT_ARGUMENT_TYPE_DISPATCH              = 2,
    D3D12_INDIRECT_ARGUMENT_TYPE_VERTEX_BUFFER_VIEW    = 3,
    D3D12_INDIRECT_ARGUMENT_TYPE_INDEX_BUFFER_VIEW     = 4,
    D3D12_INDIRECT_ARGUMENT_TYPE_CONSTANT              = 5,
    D3D12_INDIRECT_ARGUMENT_TYPE_CONSTANT_BUFFER_VIEW  = 6,
    D3D12_INDIRECT_ARGUMENT_TYPE_SHADER_RESOURCE_VIEW  = 7,
    D3D12_INDIRECT_ARGUMENT_TYPE_UNORDERED_ACCESS_VIEW = 8,
    D3D12_INDIRECT_ARGUMENT_TYPE_DISPATCH_RAYS         = 9,
    D3D12_INDIRECT_ARGUMENT_TYPE_DISPATCH_MESH         = 10,
} D3D12_INDIRECT_ARGUMENT_TYPE;
```

**描画コマンドは、必ず最後に置きます。**

```
[ルート定数] [頂点バッファビュー] [DRAW_INDEXED]
                                       ↑ 最後
```

**それ以前の要素は、コマンドの実行前に設定される引数です。**

### 34.2.3 バインドレスとの相性

**第33章でバインドレスにしたことが、ここで効きます。**

**コマンドシグネチャに `DESCRIPTOR_TABLE` は含められません。**

**従来方式なら、テクスチャの切り替えが必要になり、`ExecuteIndirect` では表現できませんでした。**

**バインドレスなら、マテリアルはインスタンスデータの中の番号にすぎません。** コマンドに含める必要がありません。

**したがって、最も単純な形が使えます。**

```cpp
D3D12_INDIRECT_ARGUMENT_DESC args[2]{};

//--- ① インスタンスバッファ内のオフセット ---
args[0].Type = D3D12_INDIRECT_ARGUMENT_TYPE_CONSTANT;
args[0].Constant.RootParameterIndex      = 1;
args[0].Constant.DestOffsetIn32BitValues = 0;
args[0].Constant.Num32BitValuesToSet     = 1;

//--- ② 描画 ---
args[1].Type = D3D12_INDIRECT_ARGUMENT_TYPE_DRAW_INDEXED;

D3D12_COMMAND_SIGNATURE_DESC desc{};
desc.ByteStride       = sizeof(IndirectDrawCommand);
desc.NumArgumentDescs = 2;
desc.pArgumentDescs   = args;
desc.NodeMask         = 0;

ComPtr<ID3D12CommandSignature> signature;
HR_TRY(device->CreateCommandSignature(
    &desc,
    m_rootSignature.Get(),     // ルート定数を含むので必要
    IID_PPV_ARGS(&signature)));
```

**第 2 引数のルートシグネチャは、`CONSTANT` などのルートパラメータを含む場合にのみ必要です。**

**`DRAW_INDEXED` だけなら `nullptr` で構いません。**

### 34.2.4 コマンド構造体

**`ByteStride` と、バッファ内のレイアウトを一致させる必要があります。**

```cpp
struct IndirectDrawCommand
{
    //--- ① ルート定数(args[0] に対応)---
    std::uint32_t instanceOffset;

    //--- ② 描画引数(args[1] に対応)---
    D3D12_DRAW_INDEXED_ARGUMENTS drawArguments;
};

static_assert(sizeof(IndirectDrawCommand) == 24);
```

```cpp
typedef struct D3D12_DRAW_INDEXED_ARGUMENTS {
    UINT IndexCountPerInstance;
    UINT InstanceCount;
    UINT StartIndexLocation;
    INT  BaseVertexLocation;
    UINT StartInstanceLocation;
} D3D12_DRAW_INDEXED_ARGUMENTS;
```

**HLSL 側にも、同じ構造体を定義します。**

```hlsl
struct IndirectDrawCommand
{
    uint instanceOffset;

    uint indexCountPerInstance;
    uint instanceCount;
    uint startIndexLocation;
    int  baseVertexLocation;
    uint startInstanceLocation;
};
```

**`static_assert` でサイズを確認しておく**のは、第15章 15.1.1 節の頂点構造体と同じ理由です。

> **アラインメントの規則**
>
> 引数の型ごとに、**バッファ内でのアラインメント要件があります。**
>
> 多くは 4 バイト境界ですが、`CONSTANT_BUFFER_VIEW` などの GPU アドレスを含むものは **8 バイト境界**が必要です。
>
> **`ByteStride` は、構造体全体のアラインメントを満たす値にしてください。**

---

## 34.3 `ExecuteIndirect`

### 34.3.1 呼び方

```cpp
void ExecuteIndirect(
    ID3D12CommandSignature* pCommandSignature,
    UINT                    MaxCommandCount,
    ID3D12Resource*         pArgumentBuffer,
    UINT64                  ArgumentBufferOffset,
    ID3D12Resource*         pCountBuffer,          // 任意
    UINT64                  CountBufferOffset);
```

| 引数 | 意味 |
|---|---|
| `MaxCommandCount` | **最大何個実行するか** |
| `pArgumentBuffer` | コマンドが入ったバッファ |
| **`pCountBuffer`** | **実際の個数が入ったバッファ(任意)** |

**`pCountBuffer` が肝心です。**

```cpp
//--- ① 固定個数 ---
commandList->ExecuteIndirect(
    signature, 100, argumentBuffer, 0, nullptr, 0);
// → 必ず 100 個実行

//--- ② GPU が決めた個数 ---
commandList->ExecuteIndirect(
    signature, 1000, argumentBuffer, 0, countBuffer, 0);
// → countBuffer の値だけ実行(最大 1000)
```

**② が、GPU カリングを可能にします。**

**カウンタバッファには、`UINT` が 1 つ入っていれば十分です。**

### 34.3.2 バッファの状態

**引数バッファには、専用の状態があります。**

```cpp
D3D12_RESOURCE_STATE_INDIRECT_ARGUMENT
```

**Enhanced Barriers では、次のように対応します**(第30章 30.4.1 節)。

```cpp
inline constexpr BarrierState kIndirectArgument{
    D3D12_BARRIER_SYNC_EXECUTE_INDIRECT,
    D3D12_BARRIER_ACCESS_INDIRECT_ARGUMENT,
    D3D12_BARRIER_LAYOUT_UNDEFINED,        // バッファなのでレイアウトなし
};
```

**GPU がコマンドを書いた後、この状態へ遷移させます。**

```cpp
//--- コンピュートで書く ---
TransitionBuffer(m_commandList7.Get(), m_indirectBuffer,
                 BarrierStates::kUnorderedAccess);
DispatchCulling();

//--- 実行前に遷移 ---
TransitionBuffer(m_commandList7.Get(), m_indirectBuffer,
                 BarrierStates::kIndirectArgument);
ExecuteIndirect();
```

**カウンタバッファも同様です。**

---

## 34.4 GPU カリング

### 34.4.1 何が変わるか

**第25章 25.3.4 節では、CPU で視錐台カリングを行いました。**

```cpp
object.visible = frustum.Intersects(worldSphere);
```

**問題は、オブジェクト数に比例して CPU の負荷が増えることです。**

```
1,000 個  →  1,000 回の判定
100,000 個 →  100,000 回の判定
```

**GPU なら、数万スレッドが並列に判定します。**

### 34.4.2 データの流れ

```
① CPU:全オブジェクトのデータを GPU へ転送
        ↓
② GPU:コンピュートシェーダーでカリング
        ↓
③ GPU:可視オブジェクトのコマンドを生成
        ↓
④ GPU:ExecuteIndirect で実行
```

**CPU は ① だけです。**

**さらに、オブジェクトが動かないなら ① も毎フレーム不要です。**

### 34.4.3 入力データ

```cpp
struct GpuObjectData
{
    Math::Matrix4x4 world;
    Math::Matrix4x4 normalMatrix;

    Math::Vector4   boundingSphere;   // xyz = 中心, w = 半径
    std::uint32_t   meshIndex;
    std::uint32_t   materialIndex;
    std::uint32_t   flags;
    std::uint32_t   padding;
};
static_assert(sizeof(GpuObjectData) == 160);
```

**境界球は、第25章 25.3.4 節で計算したものです。**

**メッシュ情報も GPU へ渡します。**

```cpp
struct GpuMeshData
{
    std::uint32_t indexCount;
    std::uint32_t startIndexLocation;
    std::int32_t  baseVertexLocation;
    std::uint32_t padding;
};
```

**第16章 16.1.3 節で説明した `BaseVertexLocation` が、ここで役立ちます。**

> これは、**複数のメッシュを 1 本のバッファにまとめる**ときに効きます。
> **インデックスを書き換えずに統合できる**わけです。第25章で複数オブジェクトを扱うときに使います。

**すべてのメッシュを 1 本のバッファに統合しておけば、頂点バッファの切り替えも不要になります。**

### 34.4.4 カリングシェーダー

```hlsl
//=====================================================
// shaders/GpuCulling.hlsl
//=====================================================

#define THREAD_GROUP_SIZE 64

struct GpuObjectData
{
    row_major float4x4 world;
    row_major float4x4 normalMatrix;
    float4 boundingSphere;
    uint   meshIndex;
    uint   materialIndex;
    uint   flags;
    uint   padding;
};

struct GpuMeshData
{
    uint indexCount;
    uint startIndexLocation;
    int  baseVertexLocation;
    uint padding;
};

struct InstanceData
{
    row_major float4x4 world;
    row_major float4x4 normalMatrix;
    uint   materialIndex;
    uint3  padding;
};

struct IndirectDrawCommand
{
    uint instanceOffset;
    uint indexCountPerInstance;
    uint instanceCount;
    uint startIndexLocation;
    int  baseVertexLocation;
    uint startInstanceLocation;
};

cbuffer CullingConstants : register(b0)
{
    float4 frustumPlanes[6];      // 第25章 25.3.4 節
    uint   objectCount;
    uint3  padding;
};

StructuredBuffer<GpuObjectData>   gObjects  : register(t0);
StructuredBuffer<GpuMeshData>     gMeshes   : register(t1);

RWStructuredBuffer<InstanceData>        gInstances : register(u0);
RWStructuredBuffer<IndirectDrawCommand> gCommands  : register(u1);
RWByteAddressBuffer                     gCounter   : register(u2);

//-----------------------------------------------------
// 視錐台との交差判定(第25章 25.3.4 節と同じ)
//-----------------------------------------------------
bool IsVisible(float4 sphere)
{
    [unroll]
    for (int i = 0; i < 6; ++i)
    {
        const float distance =
              dot(frustumPlanes[i].xyz, sphere.xyz)
            + frustumPlanes[i].w;

        if (distance < -sphere.w)
        {
            return false;
        }
    }
    return true;
}

[numthreads(THREAD_GROUP_SIZE, 1, 1)]
void CSMain(uint3 id : SV_DispatchThreadID)
{
    //--- 範囲チェック(第32章 32.3.3 節)---
    if (id.x >= objectCount)
    {
        return;
    }

    const GpuObjectData object = gObjects[id.x];

    //--- ワールド空間の境界球 ---
    const float3 center = mul(
        float4(object.boundingSphere.xyz, 1.0f), object.world).xyz;

    //--- スケールを考慮(第25章 25.3.4 節)---
    const float sx = length(float3(object.world[0][0],
                                   object.world[0][1],
                                   object.world[0][2]));
    const float sy = length(float3(object.world[1][0],
                                   object.world[1][1],
                                   object.world[1][2]));
    const float sz = length(float3(object.world[2][0],
                                   object.world[2][1],
                                   object.world[2][2]));

    const float radius = object.boundingSphere.w * max(sx, max(sy, sz));

    if (!IsVisible(float4(center, radius)))
    {
        return;
    }

    //--- 出力位置を確保する ---
    uint outputIndex;
    gCounter.InterlockedAdd(0, 1, outputIndex);

    //--- インスタンスデータを書く ---
    InstanceData instance;
    instance.world         = object.world;
    instance.normalMatrix  = object.normalMatrix;
    instance.materialIndex = object.materialIndex;
    instance.padding       = uint3(0, 0, 0);

    gInstances[outputIndex] = instance;

    //--- 描画コマンドを書く ---
    const GpuMeshData mesh = gMeshes[object.meshIndex];

    IndirectDrawCommand command;
    command.instanceOffset        = outputIndex;
    command.indexCountPerInstance = mesh.indexCount;
    command.instanceCount         = 1;
    command.startIndexLocation    = mesh.startIndexLocation;
    command.baseVertexLocation    = mesh.baseVertexLocation;
    command.startInstanceLocation = 0;

    gCommands[outputIndex] = command;
}
```

**`InterlockedAdd` で出力位置を確保しています。**

**第32章 32.4.4 節で「グローバルアトミックは遅い」と書きました。** ここでは可視オブジェクトの数だけなので、許容範囲です。

**より高速にするなら、Wave Intrinsics を使えます。**

```hlsl
//--- warp 内で何個可視かを数える ---
const uint visibleInWave = WaveActiveCountBits(true);

//--- warp の代表が、まとめて確保 ---
uint waveOffset = 0;
if (WaveIsFirstLane())
{
    gCounter.InterlockedAdd(0, visibleInWave, waveOffset);
}
waveOffset = WaveReadLaneFirst(waveOffset);

//--- warp 内での自分の位置 ---
const uint outputIndex = waveOffset + WavePrefixCountBits(true);
```

**アトミック操作が warp あたり 1 回に減ります。**

**第32章 32.4.4 節の段階 3 と、同じ考え方です。**

### 34.4.5 インスタンスがまとまらない問題

**上のシェーダーには、明らかな非効率があります。**

**可視オブジェクトごとに、`instanceCount = 1` のコマンドを 1 つ生成しています。**

```
オブジェクト 100 個が可視 → コマンド 100 個
```

**インスタンシングの利点が失われています。**

**解決策は 2 段階の処理です。**

```
① カリング:可視オブジェクトをメッシュごとに分類
        ↓
② コマンド生成:メッシュごとに 1 コマンドにまとめる
```

**実装は複雑になります。** メッシュごとのカウンタが必要で、出力位置も事前に確保する必要があります。

**本書では、単純な形に留めます。**

**理由は 2 つです。**

- **コマンド 1 個あたりのコストは小さい**(ドローコールの CPU 側コストが消えている)
- **第36章のメッシュシェーダーが、より良い解を提供する**

**「まず動く形を作り、必要なら最適化する」**という方針です(第25章 25.4.4 節と同じ)。

### 34.4.6 カウンタのリセット

**毎フレーム、カウンタを 0 に戻す必要があります。**

**方法は 3 つあります。**

| 方法 | 説明 |
|---|---|
| **A. コピー** | ゼロで埋めたバッファから `CopyBufferRegion` |
| **B. クリア** | `ClearUnorderedAccessViewUint` |
| **C. コンピュート** | 1 スレッドのディスパッチで書く |

**本書は B を使います。**

```cpp
void Renderer::ResetIndirectCounter()
{
    const UINT clearValues[4] = { 0, 0, 0, 0 };

    m_commandList->ClearUnorderedAccessViewUint(
        m_counterUavGpu,      // シェーダー可視ヒープの GPU ハンドル
        m_counterUavCpu,      // 非シェーダー可視ヒープの CPU ハンドル
        m_counterBuffer.Get(),
        clearValues,
        0, nullptr);
}
```

> **UAV のクリアには 2 つのハンドルが必要**
>
> **これは D3D12 の特殊な要件です。**
>
> | ハンドル | 用途 |
> |---|---|
> | GPU ハンドル | **シェーダー可視ヒープ**から |
> | CPU ハンドル | **非シェーダー可視ヒープ**から |
>
> **同じ UAV を、2 つのヒープに作る必要があります。**
>
> 第21章 21.1 節で作ったヒープはシェーダー可視です。**クリア専用の非可視ヒープを、別途用意します。**
>
> ```cpp
> D3D12_DESCRIPTOR_HEAP_DESC desc{};
> desc.Type           = D3D12_DESCRIPTOR_HEAP_TYPE_CBV_SRV_UAV;
> desc.NumDescriptors = 16;
> desc.Flags          = D3D12_DESCRIPTOR_HEAP_FLAG_NONE;   // ← 非可視
> ```
>
> **見落としやすい要件です。** デバッグレイヤーが指摘してくれます。

### 34.4.7 パイプラインへの統合

```cpp
void Renderer::RenderSceneIndirect(const Camera& camera)
{
    GPU_MARKER(m_commandList.Get(), m_aftermathContext, "GPU Culling");

    //--- ① カウンタをリセット ---
    ResetIndirectCounter();

    //--- ② カリング ---
    TransitionBuffer(m_commandList7.Get(), m_instanceBuffer,
                     BarrierStates::kUnorderedAccess);
    TransitionBuffer(m_commandList7.Get(), m_indirectBuffer,
                     BarrierStates::kUnorderedAccess);

    m_commandList->SetComputeRootSignature(m_cullingRootSig.Get());
    m_commandList->SetPipelineState(m_cullingPso.Get());

    //--- 視錐台を渡す(第25章 25.3.4 節)---
    const auto frustum = ExtractFrustum(
        camera.ViewMatrix() * camera.ProjectionMatrix());

    CullingConstants constants{};
    std::memcpy(constants.frustumPlanes, frustum.planes.data(),
                sizeof(constants.frustumPlanes));
    constants.objectCount = static_cast<std::uint32_t>(m_objects.size());

    const auto alloc = m_uploadBuffer.AllocateConstants(sizeof(constants));
    alloc.Write(constants);
    m_commandList->SetComputeRootConstantBufferView(0, alloc.gpuAddress);

    m_commandList->Dispatch(
        DivideRoundUp(static_cast<UINT>(m_objects.size()), 64), 1, 1);

    //--- ③ 実行可能な状態へ ---
    TransitionBuffer(m_commandList7.Get(), m_instanceBuffer,
                     BarrierStates::kNonPixelShaderResource);
    TransitionBuffer(m_commandList7.Get(), m_indirectBuffer,
                     BarrierStates::kIndirectArgument);
    TransitionBuffer(m_commandList7.Get(), m_counterBuffer,
                     BarrierStates::kIndirectArgument);

    //--- ④ 描画 ---
    {
        GPU_MARKER(m_commandList.Get(), m_aftermathContext, "Indirect Draw");

        m_commandList->SetGraphicsRootSignature(m_rootSignature.Get());
        m_commandList->SetPipelineState(m_opaquePso.Get());

        //--- メッシュを統合してあるので、1 回だけ設定 ---
        m_commandList->IASetVertexBuffers(0, 1, &m_mergedVbv);
        m_commandList->IASetIndexBuffer(&m_mergedIbv);

        m_commandList->ExecuteIndirect(
            m_commandSignature.Get(),
            static_cast<UINT>(m_objects.size()),   // 最大数
            m_indirectBuffer.Get(), 0,
            m_counterBuffer.Get(), 0);             // 実際の数
    }
}
```

**`IASetVertexBuffers` が 1 回だけです。**

**すべてのメッシュを 1 本のバッファに統合してあるためです**(34.4.3 節)。

---

## 34.5 デバッグの難しさ

### 34.5.1 何が見えなくなるか

**`ExecuteIndirect` は、デバッグを著しく困難にします。**

| | 通常の描画 | **ExecuteIndirect** |
|---|---|---|
| ドローコールの数 | **コードから分かる** | 実行時まで不明 |
| 描画の内容 | コードから分かる | **バッファの中身次第** |
| Nsight Graphics | 各ドローが個別に表示 | **1 つの `ExecuteIndirect` として表示** |

**第29章 29.2.2 節でイベントリストを読みました。**

**`ExecuteIndirect` は、そこに 1 行しか現れません。**

### 34.5.2 対策

**対策 1:Nsight Graphics で展開する**

**Nsight Graphics は、`ExecuteIndirect` の内容を展開して表示できます。**

```
▼ ExecuteIndirect (87 commands)
    [0] DrawIndexedInstanced (indexCount: 5832, instanceCount: 1)
    [1] DrawIndexedInstanced (indexCount: 1296, instanceCount: 1)
    ...
```

**引数バッファの中身を読んで、実際のコマンドを表示してくれます。**

**対策 2:CPU 版と比較する**

**同じシーンを、CPU カリング + 通常描画でも描けるようにしておきます。**

```cpp
if (m_useGpuCulling)
{
    RenderSceneIndirect(camera);
}
else
{
    RenderSceneDirect(camera);      // 第25章の方式
}
```

**絵が同じであることを確認できます。**

**第33章 33.2.2 節でフォールバックを用意したのと、同じ発想です。**

**対策 3:結果を読み戻す**

**デバッグ時だけ、カウンタと引数バッファを CPU へ読み戻します。**

```cpp
#if defined(_DEBUG)
void Renderer::DebugReadbackIndirectBuffer()
{
    //--- READBACK ヒープへコピー ---
    // 第15章 15.2.1 節で触れた D3D12_HEAP_TYPE_READBACK
    TransitionBuffer(m_commandList7.Get(), m_counterBuffer,
                     BarrierStates::kCopySource);

    m_commandList->CopyBufferRegion(
        m_readbackBuffer.Get(), 0,
        m_counterBuffer.Get(), 0,
        sizeof(std::uint32_t));

    //--- 次のフレームで読む(即座には読めない)---
    m_readbackPending = true;
    m_readbackFenceValue = m_fence.LastSignaledValue();
}

void Renderer::CheckReadback()
{
    if (!m_readbackPending) return;
    if (!m_fence.IsComplete(m_readbackFenceValue)) return;

    void* mapped = nullptr;
    const D3D12_RANGE readRange{ 0, sizeof(std::uint32_t) };
    m_readbackBuffer->Map(0, &readRange, &mapped);

    const auto visibleCount = *static_cast<std::uint32_t*>(mapped);
    LOG_TRACE(L"GPU culling: {} / {} visible",
              visibleCount, m_objects.size());

    m_readbackBuffer->Unmap(0, nullptr);
    m_readbackPending = false;
}
#endif
```

**READBACK ヒープを使います。** 第15章 15.2.1 節の表で挙げた 3 種類目です。

**GPU の完了を待つ必要があります。** 即座に読むと、まだ書かれていません。

**第10章のフェンスを使って、非同期に読み取ります。**

### 34.5.3 Aftermath での見え方

**第31章 31.2 節で、マーカーを仕込みました。**

**`ExecuteIndirect` の中でクラッシュした場合、マーカーは `Indirect Draw` としか教えてくれません。**

**「何番目のコマンドで落ちたか」は分かりません。**

**ただし、ページフォルトのリソース名は表示されます。**

```
Page Fault:
  Resource: 'MergedVertexBuffer'
    Note: Access beyond the end of the resource.
```

**`BaseVertexLocation` の計算ミスなど、原因の絞り込みには使えます。**

---

## ✅ 本章のゴール:GPU が描画コマンドを生成する

### Step 1:インスタンシングを有効にする

```
インスタンシングなし:
  draws 336, pso 2, mesh 3

インスタンシングあり:
  draws 3, batches 3, instances 336
```

**フレームレートを比較してください。**

**オブジェクト数が多いほど、差が大きくなります。**

| オブジェクト数 | 予想される改善 |
|---|---|
| 100 | わずか |
| 1,000 | 明確 |
| 10,000 | **劇的** |

### Step 2:`nointerpolation` を外してみる

```hlsl
uint materialIndex : MATERIAL_INDEX;   // ❌ nointerpolation なし
```

**三角形の内部で、マテリアルが変化します。**

**でたらめなテクスチャが貼られるか、範囲外アクセスでクラッシュします。**

**34.1.3 節で説明した理由です。**

**確認したら元に戻してください。**

### Step 3:コマンドシグネチャを作る

```
[Info ] Renderer.cpp(388): command signature created (stride 24 bytes)
```

**`ByteStride` が構造体のサイズと一致していることを確認してください。**

**食い違うと、コマンドの解釈がずれます。**

### Step 4:固定個数で `ExecuteIndirect` を試す

**まず、CPU 側でコマンドを作って実行します。**

```cpp
//--- CPU で引数バッファを埋める ---
std::vector<IndirectDrawCommand> commands;
for (const auto& batch : batches)
{
    IndirectDrawCommand cmd{};
    cmd.instanceOffset = batch.instanceOffset;
    cmd.drawArguments.IndexCountPerInstance = mesh.indexCount;
    cmd.drawArguments.InstanceCount = batch.instanceCount;
    // ...
    commands.push_back(cmd);
}

//--- アップロードして実行 ---
m_commandList->ExecuteIndirect(
    m_commandSignature.Get(),
    static_cast<UINT>(commands.size()),
    m_indirectBuffer.Get(), 0,
    nullptr, 0);
```

**通常の描画と同じ絵になることを確認してください。**

**この段階で、コマンドシグネチャの構造が正しいかが分かります。**

### Step 5:GPU カリングを有効にする

```
[Trace] Renderer.cpp(455): GPU culling: 87 / 2500 visible
```

**カメラを回すと、可視数が変化します。**

**34.5.2 節の対策 3(読み戻し)で確認できます。**

### Step 6:CPU 版と比較する

```cpp
m_useGpuCulling = false;   // CPU カリング(第25章)
```

**絵がまったく同じであることを確認してください。**

**統計も比較します。**

```
CPU カリング:  objects 87/2500, draws 87
GPU カリング:  objects 2500/2500(全部渡す), indirect commands 87
```

**CPU 側の処理時間を測ると、差が見えます。**

### Step 7:オブジェクト数を増やす

```cpp
BuildTestScene(m_objects, 100);   // 10,000 個
```

**CPU カリングでは、判定だけで時間がかかります。**

**GPU カリングでは、CPU 側はほぼ何もしません。**

**第29章の GPU Trace で、カリングのディスパッチにかかる時間を測ってください。**

### Step 8:カウンタのリセットを忘れてみる

```cpp
// ResetIndirectCounter();   ← コメントアウト
```

**カウンタが増え続け、やがて `MaxCommandCount` に達します。**

**そして、初期化されていない領域のコマンドが実行されます。**

**多くの場合、クラッシュします。**

**第31章の Aftermath で解析すると、範囲外アクセスとして現れます。**

**確認したら元に戻してください。**

### Step 9:UAV クリアのハンドルを間違えてみる

```cpp
//--- 両方ともシェーダー可視ヒープから取る ---
m_commandList->ClearUnorderedAccessViewUint(
    m_counterUavGpu,
    m_counterUavGpuAsCpu,   // ❌ 非可視ヒープであるべき
    // ...
);
```

```
D3D12 ERROR: ClearUnorderedAccessViewUint: The CPU descriptor handle
  must be from a non-shader-visible descriptor heap.
```

**34.4.6 節のコラムで説明した要件です。**

**確認したら元に戻してください。**

### Step 10:Nsight Graphics で展開する

**キャプチャして、`ExecuteIndirect` を選択してください。**

**引数バッファの中身が展開表示されます。**

**各コマンドの `InstanceCount` や `StartIndexLocation` が、期待通りかを確認できます。**

---

### 本章の達成状態

- [ ] 構造化バッファ方式でインスタンシングを実装した
- [ ] `nointerpolation` でマテリアルインデックスを渡している
- [ ] インスタンスをメッシュごとにバッチ化した
- [ ] コマンドシグネチャを作った
- [ ] `ByteStride` と構造体サイズが一致している
- [ ] `ExecuteIndirect` で描画した
- [ ] カウンタバッファで個数を制御している
- [ ] `INDIRECT_ARGUMENT` 状態へ遷移させている
- [ ] GPU カリングを実装した
- [ ] 毎フレームカウンタをリセットしている
- [ ] UAV クリア用の非可視ヒープを用意した
- [ ] CPU 版と絵が一致することを確認した
- [ ] **GPU が生成したコマンドで描画された**

---

## トラブルシューティング

| 症状 | 原因 | 対処 |
|---|---|---|
| マテリアルがおかしい | `nointerpolation` 忘れ | 34.1.3 |
| コマンドが正しく解釈されない | `ByteStride` の不一致 | 34.2.4 |
| `ExecuteIndirect` でエラー | 状態遷移を忘れた | 34.3.2 |
| 描画されない | カウンタが 0 | カリングの判定を確認 |
| 描画数が増え続ける | カウンタ未リセット | 34.4.6 |
| UAV クリアでエラー | ハンドルが可視ヒープ | 34.4.6 のコラム |
| 一部のメッシュがおかしい | `BaseVertexLocation` の誤り | 第16章 16.1.3 節 |
| Nsight で中身が見えない | 対応バージョンを確認 | 34.5.2 |
| CPU 版と絵が違う | 視錐台の抽出が誤り | 第25章 25.3.4 節 |
| Reversed-Z で手前が消える | 平面の抽出 | 第25章 25.3.4 節のコラム |

---

## まとめ

**1. 構造化バッファ方式のインスタンシングが扱いやすい。**
入力レイアウトが単純なまま、構造体をそのまま使えます。**GPU カリングでも同じバッファを使い回せます。**

**2. `nointerpolation` を忘れない。**
整数のインデックスは補間されては困ります。**第15章 15.6.2 節の修飾子が、ここで必須になります。**

**3. バインドレスが `ExecuteIndirect` を可能にした。**
コマンドシグネチャにディスクリプタテーブルは含められません。**第33章でマテリアルをインデックス化したことが、前提条件でした。**

**4. カウンタバッファが GPU カリングの鍵。**
「最大 N 個まで、実際は M 個」という指定ができます。**M を GPU が決められることが本質です。**

**5. UAV のクリアには 2 つのヒープが必要。**
シェーダー可視と非可視。**D3D12 の特殊な要件で、見落としやすい部分です。**

**6. デバッグが困難になる。**
ドローコールの数も内容も、実行時まで分かりません。**CPU 版と比較できるようにしておくことが、最も確実な検証手段です。**

**7. インスタンスのまとめは、あえて省略した。**
コマンド 1 個あたりのコストは小さく、**第36章のメッシュシェーダーがより良い解を提供します。**

次章ではマルチスレッド化を扱います。**コマンドリストの並列記録、バンドル、そしてコピーキューによる非同期転送。** 第9章 9.2.3 節で「スレッドごとにアロケータとリストを分ける」と書いた原則が、そこで実装されます。

---

## 参考リンク

| 内容 | URL |
|---|---|
| `ExecuteIndirect` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/nf-d3d12-id3d12graphicscommandlist-executeindirect |
| コマンドシグネチャ | https://learn.microsoft.com/ja-jp/windows/win32/direct3d12/indirect-drawing |
| `D3D12_INDIRECT_ARGUMENT_DESC` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/ns-d3d12-d3d12_indirect_argument_desc |
| `ClearUnorderedAccessViewUint` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/nf-d3d12-id3d12graphicscommandlist-clearunorderedaccessviewuint |
| `SV_InstanceID` | https://learn.microsoft.com/ja-jp/windows/win32/direct3dhlsl/dx-graphics-hlsl-semantics |
