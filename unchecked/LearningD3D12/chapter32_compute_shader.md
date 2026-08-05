# 第32章 コンピュートシェーダー

第5部に入ります。ここからは、モダンな Direct3D 12 の機能を扱います。

**最初はコンピュートシェーダーです。**

これまでのシェーダーは、すべて描画パイプラインの一部でした。頂点シェーダーは頂点ごとに、ピクセルシェーダーはピクセルごとに実行されます。**入力も出力も、パイプラインが決めていました。**

**コンピュートシェーダーは違います。**

```
入力:  自分で決めたバッファ
処理:  自分で決めたスレッド数
出力:  自分で決めたバッファ
```

**GPU を「大量の並列計算装置」として使います。**

そして本章では、**第2章 2.3.1 節で説明した warp、SM、occupancy が、設計上の判断材料として実際に使われます。**

> **本書のルール:32 という数字をコードに埋め込まない。**
> **NVIDIA を前提にするのは判断のためであり、依存するためではありません。**

**その「判断」を、ここで行います。**

**本章のゴール**
コンピュートパイプラインを構築し、UAV でデータを読み書きする。輝度ヒストグラムと GPU パーティクルを実装し、Nsight Graphics で occupancy を確認する。

---

## 32.1 コンピュートパイプライン

### 32.1.1 描画パイプラインとの違い

| | 描画 | **コンピュート** |
|---|---|---|
| PSO | `CreateGraphicsPipelineState` | **`CreateComputePipelineState`** |
| ルートシグネチャの設定 | `SetGraphicsRootSignature` | **`SetComputeRootSignature`** |
| 定数の設定 | `SetGraphicsRoot*` | **`SetComputeRoot*`** |
| 実行 | `DrawInstanced` | **`Dispatch`** |
| 固定機能 | ラスタライザ、深度テスト等 | **なし** |

**すべての設定関数に `Compute` 版があります。**

**`Graphics` 版と `Compute` 版は、独立した状態を持ちます。** 片方を設定しても、もう片方には影響しません。

```cpp
commandList->SetGraphicsRootSignature(graphicsRootSig);   // 描画用
commandList->SetComputeRootSignature(computeRootSig);     // コンピュート用
// 両方が同時に設定されている
```

### 32.1.2 コンピュート PSO

**描画用と比べると、驚くほど単純です。**

```cpp
typedef struct D3D12_COMPUTE_PIPELINE_STATE_DESC {
    ID3D12RootSignature*        pRootSignature;
    D3D12_SHADER_BYTECODE       CS;
    UINT                        NodeMask;
    D3D12_CACHED_PIPELINE_STATE CachedPSO;
    D3D12_PIPELINE_STATE_FLAGS  Flags;
} D3D12_COMPUTE_PIPELINE_STATE_DESC;
```

**5 つだけです。**

**第14章 14.4.2 節の `D3D12_GRAPHICS_PIPELINE_STATE_DESC` は 22 個でした。** ラスタライザもブレンドも深度も、コンピュートには存在しません。

**したがって、第14章 14.4.4 節で作った「既定値を持つ自作関数」も不要です。** ゼロ初期化で問題が起きるフィールドがありません。

```cpp
Core::Result<ComPtr<ID3D12PipelineState>> CreateComputePso(
    ID3D12Device*        device,
    ID3D12RootSignature* rootSignature,
    const ShaderBlob&    cs,
    std::wstring_view    name)
{
    D3D12_COMPUTE_PIPELINE_STATE_DESC desc{};   // {} で十分
    desc.pRootSignature = rootSignature;
    desc.CS             = cs.Bytecode();
    desc.NodeMask       = 0;
    desc.Flags          = D3D12_PIPELINE_STATE_FLAG_NONE;

    ComPtr<ID3D12PipelineState> pso;
    HR_TRY(device->CreateComputePipelineState(&desc, IID_PPV_ARGS(&pso)));

    Core::SetDebugName(pso.Get(), name);
    return pso;
}
```

**ルートシグネチャのフラグも単純になります。**

```cpp
versioned.Desc_1_1.Flags = D3D12_ROOT_SIGNATURE_FLAG_NONE;
```

**第14章 14.2.3 節で使った `DENY_*` フラグは、コンピュートでは意味を持ちません。** 頂点シェーダーもピクセルシェーダーも存在しないからです。

**`ALLOW_INPUT_ASSEMBLER_INPUT_LAYOUT` も不要です。** 第26章 26.2.3 節のフルスクリーン三角形と同じく、入力アセンブラを使いません。

### 32.1.3 コンピュートキュー

**第9章 9.2.2 節で、コマンドリストの種類を挙げました。**

| 種類 | できること |
|---|---|
| DIRECT | 描画・コンピュート・コピー、すべて |
| **COMPUTE** | **コンピュートとコピー** |
| COPY | コピーのみ |

**DIRECT キューでもコンピュートは実行できます。** 本章は当面 DIRECT で進めます。

**専用の COMPUTE キューを使う利点は、描画と並行して実行できることです。** これを「非同期コンピュート」と呼びます。**第35章で扱います。**

---

## 32.2 UAV と構造化バッファ

### 32.2.1 UAV とは

**Unordered Access View — 「順序なしアクセスビュー」。**

**これまで扱ってきた SRV との違いは、書き込めることです。**

| ビュー | 読み | 書き |
|---|---|---|
| SRV(第20章) | ○ | × |
| **UAV** | **○** | **○** |

**「順序なし」という名前の意味は、複数のスレッドが同時に書き込んだとき、順序が保証されないということです。**

```
スレッド 0 が buffer[5] に書く
スレッド 1 が buffer[5] に書く
   → どちらが残るかは不定
```

**同じ場所への書き込みを避けるか、アトミック操作を使う必要があります**(32.4 節)。

### 32.2.2 構造化バッファ

**HLSL では、いくつかのバッファ型が使えます。**

| 型 | 用途 |
|---|---|
| `StructuredBuffer<T>` | 構造体の配列(読み取り専用) |
| **`RWStructuredBuffer<T>`** | **構造体の配列(読み書き)** |
| `ByteAddressBuffer` | バイト単位のアクセス(読み取り専用) |
| `RWByteAddressBuffer` | バイト単位(読み書き)。**アトミック操作用** |
| `RWTexture2D<T>` | テクスチャへの書き込み |

**構造化バッファが最も使いやすい**ので、本書はこれを中心に使います。

```hlsl
struct Particle
{
    float3 position;
    float  life;
    float3 velocity;
    float  size;
};

RWStructuredBuffer<Particle> gParticles : register(u0);

[numthreads(64, 1, 1)]
void CSMain(uint3 id : SV_DispatchThreadID)
{
    Particle p = gParticles[id.x];
    p.position += p.velocity * deltaTime;
    gParticles[id.x] = p;
}
```

**C++ 側の構造体と一致させる必要があります。**

```cpp
struct Particle
{
    Math::Vector3 position;
    float         life;
    Math::Vector3 velocity;
    float         size;
};
static_assert(sizeof(Particle) == 32);
```

> **定数バッファのパッキング規則は適用されない**
>
> **第18章 18.2.3 節で、`cbuffer` の 16 バイト境界の問題を扱いました。**
>
> **構造化バッファには、この規則が適用されません。** C++ の構造体と同じレイアウトになります。
>
> **`float3` を自由に使えます。** ただし、アラインメントの一般規則(4 バイト境界)は守る必要があります。

### 32.2.3 UAV を作る

**バッファの生成は、第16章と同じです。** フラグだけが違います。

```cpp
auto desc = MakeBufferDesc(
    elementCount * sizeof(Particle),
    D3D12_RESOURCE_FLAG_ALLOW_UNORDERED_ACCESS);   // ← 追加
```

**`ALLOW_UNORDERED_ACCESS` フラグが必須です。** 第19章の `ALLOW_DEPTH_STENCIL`、第26章の `ALLOW_RENDER_TARGET` と同じ位置づけです。

```cpp
D3D12_UNORDERED_ACCESS_VIEW_DESC uavDesc{};
uavDesc.Format                      = DXGI_FORMAT_UNKNOWN;   // 構造化バッファ
uavDesc.ViewDimension               = D3D12_UAV_DIMENSION_BUFFER;
uavDesc.Buffer.FirstElement         = 0;
uavDesc.Buffer.NumElements          = elementCount;
uavDesc.Buffer.StructureByteStride  = sizeof(Particle);
uavDesc.Buffer.CounterOffsetInBytes = 0;
uavDesc.Buffer.Flags                = D3D12_BUFFER_UAV_FLAG_NONE;

device->CreateUnorderedAccessView(
    buffer, nullptr, &uavDesc, handle.cpu);
```

**`Format = DXGI_FORMAT_UNKNOWN` と `StructureByteStride` の組み合わせが、構造化バッファを表します。**

**第 2 引数の `nullptr` は、カウンタバッファです。** `AppendStructuredBuffer` を使う場合に指定します。本書では使いません。

### 32.2.4 UAV バリア

**第30章 30.9 節で扱った内容です。**

**同じリソースに対する書き込みが完了してから、次の読み書きを始める**ことを保証します。

```cpp
//--- ディスパッチ 1:バッファに書く ---
commandList->SetPipelineState(m_writePso.Get());
commandList->Dispatch(groupCount, 1, 1);

//--- UAV バリア ---
const auto barrier = MakeUavBarrier(m_buffer.Get());
commandList7->Barrier(1, &barrierGroup);

//--- ディスパッチ 2:そのバッファを読む ---
commandList->SetPipelineState(m_readPso.Get());
commandList->Dispatch(groupCount, 1, 1);
```

**バリアがないと、ディスパッチ 2 が古い値を読む可能性があります。**

**Enhanced Barriers での書き方は、第30章 30.9.2 節の通りです。** ヘルパーを追加しておきます。

```cpp
// src/Graphics/D3D12Helpers.h に追加

//---------------------------------------------------------------
// UAV バリア。前後でアクセスとレイアウトを変えない。
//---------------------------------------------------------------
[[nodiscard]] inline D3D12_BUFFER_BARRIER MakeUavBufferBarrier(
    ID3D12Resource* resource) noexcept
{
    return MakeBufferBarrier(
        resource,
        D3D12_BARRIER_SYNC_COMPUTE_SHADING,
        D3D12_BARRIER_SYNC_COMPUTE_SHADING,
        D3D12_BARRIER_ACCESS_UNORDERED_ACCESS,
        D3D12_BARRIER_ACCESS_UNORDERED_ACCESS);
}
```

**自作ヘルパーが 14 個になりました。**

---

## 32.3 スレッドグループ

### 32.3.1 3 段階の構造

**コンピュートシェーダーのスレッドは、3 段階で構成されます。**

```
Dispatch(gx, gy, gz)           ← スレッドグループの数
  └─ [numthreads(tx, ty, tz)]  ← グループ内のスレッド数
       └─ 個々のスレッド
```

**総スレッド数は、両者の積です。**

```
Dispatch(64, 1, 1) + [numthreads(64, 1, 1)]
  → 64 × 64 = 4096 スレッド
```

### 32.3.2 システム値セマンティクス

**自分がどのスレッドかを知る手段が、4 つあります。**

```hlsl
[numthreads(8, 8, 1)]
void CSMain(uint3 groupId       : SV_GroupID,
            uint3 groupThreadId : SV_GroupThreadID,
            uint3 dispatchId    : SV_DispatchThreadID,
            uint  groupIndex    : SV_GroupIndex)
{
    // ...
}
```

| セマンティクス | 意味 |
|---|---|
| `SV_GroupID` | **グループの番号**(Dispatch の範囲) |
| `SV_GroupThreadID` | **グループ内での位置**(numthreads の範囲) |
| **`SV_DispatchThreadID`** | **全体での通し位置**(最もよく使う) |
| `SV_GroupIndex` | グループ内の一次元インデックス |

**関係式です。**

```
DispatchThreadID = GroupID × numthreads + GroupThreadID

GroupIndex = GroupThreadID.z × (tx × ty)
           + GroupThreadID.y × tx
           + GroupThreadID.x
```

**1 次元のデータを処理するなら、`SV_DispatchThreadID.x` を配列の添字にするだけです。**

### 32.3.3 範囲チェックを忘れない

**要素数がグループサイズで割り切れないとき、余分なスレッドが起動します。**

```
要素数 1000、numthreads(64,1,1)
  → 必要なグループ数 = ceil(1000 / 64) = 16
  → 総スレッド数 = 16 × 64 = 1024
  → 24 スレッドが余る
```

**余ったスレッドが配列の範囲外にアクセスすると、ページフォルトです。**

**第31章 31.5.2 節の実験 B が、まさにこれでした。**

```hlsl
[numthreads(64, 1, 1)]
void CSMain(uint3 id : SV_DispatchThreadID)
{
    //--- 必ず範囲チェックを入れる ---
    if (id.x >= elementCount)
    {
        return;
    }

    // ... 処理 ...
}
```

**グループ数の計算にも注意が必要です。**

```cpp
//--- 切り上げ除算 ---
const UINT groupCount = (elementCount + kThreadGroupSize - 1)
                      / kThreadGroupSize;

commandList->Dispatch(groupCount, 1, 1);
```

**第18章 18.2.2 節の `AlignUp` と同じ考え方です。** 定数化しておきます。

```cpp
[[nodiscard]] constexpr UINT DivideRoundUp(UINT value, UINT divisor) noexcept
{
    return (value + divisor - 1) / divisor;
}

static_assert(DivideRoundUp(1000, 64) == 16);
static_assert(DivideRoundUp(1024, 64) == 16);
static_assert(DivideRoundUp(1025, 64) == 17);
```

### 32.3.4 スレッド数を 32 の倍数にする

**第2章 2.3.1 節で説明した内容が、ここで設計判断になります。**

> **NVIDIA GPU は、32 スレッドをひとまとまりにして実行します。** この 32 スレッドの束を warp と呼びます。
> `[numthreads(10, 10, 1)]` は 100 スレッドです。100 ÷ 32 = 3.125 なので、GPU は 4 warp = 128 レーンを起動し、**28 レーンを無駄にします。**

**具体例で確認します。**

| `numthreads` | スレッド数 | warp 数 | 無駄 |
|---|---|---|---|
| `(32, 1, 1)` | 32 | 1 | **0%** |
| `(64, 1, 1)` | 64 | 2 | **0%** |
| `(8, 8, 1)` | 64 | 2 | **0%** |
| `(10, 10, 1)` | 100 | 4 | **22%** |
| `(16, 16, 1)` | 256 | 8 | **0%** |
| `(100, 1, 1)` | 100 | 4 | **22%** |

**`(8, 8, 1)` が 2D の定番**なのは、これが理由です。64 スレッド = 2 warp ちょうどになります。

**第7章で取得した実測値を使います。**

```cpp
LOG_INFO(L"wave lanes  : {} .. {} (total {})",
         caps.waveLaneCountMin,
         caps.waveLaneCountMax,
         caps.totalLaneCount);
```

```
[Info ] GraphicsDevice.cpp(291): wave lanes  : 32 .. 32  (total 5888)
```

**第7章 7.5.4 節で「第32章でスレッドグループサイズを決める根拠にする」と書きました。** ここで使います。

```cpp
//---------------------------------------------------------------
// スレッドグループサイズを、実行環境の wave サイズに合わせる。
//
// NVIDIA では 32、AMD では 32 または 64。
// 本書は 64(NVIDIA で 2 warp)を既定とする。
//---------------------------------------------------------------
UINT ChooseThreadGroupSize(const DeviceCaps& caps)
{
    const UINT waveSize = (caps.waveLaneCountMin > 0)
        ? caps.waveLaneCountMin : 32;

    //--- wave の 2 倍を既定とする ---
    return waveSize * 2;
}
```

**ただし、`[numthreads]` はコンパイル時定数です。** 実行時に変えられません。

**実用的には、いくつかのバリエーションをコンパイルしておき、実行時に選びます。**

```cpp
//--- シェーダーのコンパイル時に定義を渡す ---
// dxc -D THREAD_GROUP_SIZE=64 ...
```

```hlsl
#ifndef THREAD_GROUP_SIZE
    #define THREAD_GROUP_SIZE 64
#endif

[numthreads(THREAD_GROUP_SIZE, 1, 1)]
void CSMain(uint3 id : SV_DispatchThreadID) { ... }
```

**本書は 64 固定で進めます。** NVIDIA でも AMD でも無駄が出ない値だからです。

### 32.3.5 グループ共有メモリ

**同じグループ内のスレッドは、高速な共有メモリを使えます。**

```hlsl
groupshared float sharedData[64];

[numthreads(64, 1, 1)]
void CSMain(uint3 dispatchId  : SV_DispatchThreadID,
            uint  groupIndex  : SV_GroupIndex)
{
    //--- 各スレッドが 1 要素を読む ---
    sharedData[groupIndex] = gInput[dispatchId.x];

    //--- 全スレッドの書き込みを待つ ---
    GroupMemoryBarrierWithGroupSync();

    //--- 共有メモリから読む ---
    const float sum = sharedData[0] + sharedData[1];
    // ...
}
```

**第2章 2.3.1 節で説明した通り、共有メモリは SM 内部にあります。**

> HLSL のスレッドグループは、必ず 1 つの SM 上で実行されます。**だからこそ `groupshared` メモリによるスレッド間の高速な情報共有が可能になります。**

**同期が必須です。**

| 関数 | 動作 |
|---|---|
| `GroupMemoryBarrier()` | 共有メモリの書き込み完了を待つ |
| **`GroupMemoryBarrierWithGroupSync()`** | **上記 + 全スレッドの到達を待つ** |
| `DeviceMemoryBarrierWithGroupSync()` | UAV への書き込みも待つ |

**`WithGroupSync` を使うのが基本です。** 同期しないと、他のスレッドが書き込む前のデータを読む可能性があります。

> **サイズの上限は 32KB**
>
> D3D12 では、1 グループあたり **32KB** が上限です。
>
> **使いすぎると occupancy が下がります**(第2章 2.3.1 節)。SM が同時に抱えられるグループ数が減るためです。
>
> 32KB 使い切ると、SM に 1 グループしか載らなくなり、メモリレイテンシを隠蔽できません。

---

## 32.4 Wave Intrinsics

### 32.4.1 warp 内での協調

**Shader Model 6.0 で追加された機能です。**

**同じ warp 内のスレッド同士が、共有メモリを介さずに直接データをやり取りできます。**

```hlsl
//--- warp 内の全スレッドの値を合計する ---
const float sum = WaveActiveSum(myValue);
```

**共有メモリを使う場合と比べて、はるかに高速です。**

| 手法 | コスト |
|---|---|
| `groupshared` + 同期 | 共有メモリへの往復、同期のオーバーヘッド |
| **Wave Intrinsics** | **レジスタ間の直接転送** |

### 32.4.2 主な関数

| 関数 | 動作 |
|---|---|
| `WaveGetLaneCount()` | **warp のサイズを返す** |
| `WaveGetLaneIndex()` | warp 内での自分の位置 |
| `WaveIsFirstLane()` | 最初の有効なレーンか |
| `WaveActiveSum(x)` | 全レーンの合計 |
| `WaveActiveMin(x)` / `Max(x)` | 最小 / 最大 |
| `WaveActiveCountBits(b)` | 条件を満たすレーン数 |
| `WaveActiveAllTrue(b)` | 全レーンが真か |
| `WaveActiveAnyTrue(b)` | いずれかが真か |
| `WavePrefixSum(x)` | 自分より前のレーンの合計 |
| `WaveReadLaneFirst(x)` | 最初のレーンの値を全員が読む |

**`WaveGetLaneCount()` が、第2章の注意書きに対応します。**

> **32 という数字をコードに埋め込まない。**
> HLSL には `WaveGetLaneCount()` という組み込み関数があり、実行時のレーン数を取得できます。**Wave Intrinsics を使うコードでは、この関数を使ってください。**

### 32.4.3 対応を確認する

**第7章 7.5.3 節で `OPTIONS1` を問い合わせました。**

```cpp
D3D12_FEATURE_DATA_D3D12_OPTIONS1 options1{};
if (QueryFeature(device, D3D12_FEATURE_D3D12_OPTIONS1, options1))
{
    m_caps.waveOps          = options1.WaveOps;
    m_caps.waveLaneCountMin = options1.WaveLaneCountMin;
    m_caps.waveLaneCountMax = options1.WaveLaneCountMax;
    m_caps.totalLaneCount   = options1.TotalLaneCount;
}
```

**`WaveOps` が偽なら、Wave Intrinsics は使えません。**

**Turing 以降では、必ず対応しています**(第2章 2.1.2 節)。

### 32.4.4 縮約の例

**「大量の値を合計する」処理を、3 段階で最適化します。**

**段階 1:素朴な実装(アトミック操作)**

```hlsl
RWByteAddressBuffer gResult : register(u0);

[numthreads(64, 1, 1)]
void CSNaive(uint3 id : SV_DispatchThreadID)
{
    if (id.x >= elementCount) return;

    const uint value = gInput[id.x];

    //--- 全スレッドが同じ場所に加算 ---
    uint original;
    gResult.InterlockedAdd(0, value, original);
}
```

**動きますが、非常に遅いです。** 数千スレッドが同じアドレスを取り合います。

**段階 2:共有メモリで縮約**

```hlsl
groupshared uint sharedSum[64];

[numthreads(64, 1, 1)]
void CSSharedMemory(uint3 id : SV_DispatchThreadID,
                    uint groupIndex : SV_GroupIndex)
{
    sharedSum[groupIndex] = (id.x < elementCount) ? gInput[id.x] : 0;
    GroupMemoryBarrierWithGroupSync();

    //--- 木構造で縮約 ---
    for (uint stride = 32; stride > 0; stride >>= 1)
    {
        if (groupIndex < stride)
        {
            sharedSum[groupIndex] += sharedSum[groupIndex + stride];
        }
        GroupMemoryBarrierWithGroupSync();
    }

    //--- グループ代表が加算 ---
    if (groupIndex == 0)
    {
        uint original;
        gResult.InterlockedAdd(0, sharedSum[0], original);
    }
}
```

**アトミック操作の回数が、グループ数まで減りました。**

**段階 3:Wave Intrinsics**

```hlsl
[numthreads(64, 1, 1)]
void CSWaveIntrinsics(uint3 id : SV_DispatchThreadID,
                      uint groupIndex : SV_GroupIndex)
{
    const uint value = (id.x < elementCount) ? gInput[id.x] : 0;

    //--- warp 内で合計(同期不要)---
    const uint waveSum = WaveActiveSum(value);

    //--- warp の代表だけが共有メモリに書く ---
    groupshared uint waveSums[2];   // 64 / 32 = 2 warp

    if (WaveIsFirstLane())
    {
        waveSums[groupIndex / WaveGetLaneCount()] = waveSum;
    }

    GroupMemoryBarrierWithGroupSync();

    //--- グループ代表が最終合計 ---
    if (groupIndex == 0)
    {
        const uint groupSum = waveSums[0] + waveSums[1];
        uint original;
        gResult.InterlockedAdd(0, groupSum, original);
    }
}
```

**同期の回数が、6 回から 1 回に減りました。**

**段階 2 のループは 6 回の同期を含んでいました**(64 → 32 → 16 → 8 → 4 → 2 → 1)。**Wave Intrinsics なら、warp 内の縮約に同期が不要です。**

> **`waveSums` のサイズをハードコードしている**
>
> 上のコードは `waveSums[2]` と書いており、**warp サイズ 32 を前提としています。**
>
> 厳密には、次のように書くべきです。
>
> ```hlsl
> #define MAX_WAVES_PER_GROUP (THREAD_GROUP_SIZE / 32)
> groupshared uint waveSums[MAX_WAVES_PER_GROUP];
> ```
>
> **`groupshared` のサイズはコンパイル時定数でなければならない**ので、`WaveGetLaneCount()` は使えません。**最小の wave サイズ(32)を前提に、余裕を持って確保します。**
>
> **これが「32 を前提にせざるを得ない」数少ない場面です。**

---

## 32.5 実例 1:輝度ヒストグラム

### 32.5.1 何に使うか

**シーンの明るさの分布を求めます。**

**用途は自動露出です。** 第26章 26.5.5 節でトーンマッピングを実装しましたが、露出は固定値でした。

```cpp
color = ToneMap(color * exposure);   // exposure は定数
```

**ヒストグラムから平均輝度を求めれば、シーンに応じて露出を自動調整できます。**

```
明るいシーン → 露出を下げる
暗いシーン   → 露出を上げる
```

### 32.5.2 実装

```hlsl
//=====================================================
// shaders/LuminanceHistogram.hlsl
//=====================================================

#define HISTOGRAM_BINS 256
#define THREAD_GROUP_X 16
#define THREAD_GROUP_Y 16

cbuffer HistogramConstants : register(b0)
{
    uint  inputWidth;
    uint  inputHeight;
    float minLogLuminance;
    float logLuminanceRange;    // 1 / (max - min)
};

Texture2D<float4>       gInput     : register(t0);
RWStructuredBuffer<uint> gHistogram : register(u0);

groupshared uint sharedHistogram[HISTOGRAM_BINS];

//-----------------------------------------------------
// 輝度をビン番号に変換する。
// 対数スケールを使うのは、人間の知覚に近いため。
//-----------------------------------------------------
uint LuminanceToBin(float luminance)
{
    if (luminance < 1e-5f)
    {
        return 0;      // 黒はビン 0 に集める
    }

    const float logLuminance =
        saturate((log2(luminance) - minLogLuminance) * logLuminanceRange);

    //--- ビン 0 は黒専用なので、1 〜 255 に割り当てる ---
    return (uint)(logLuminance * 254.0f + 1.0f);
}

[numthreads(THREAD_GROUP_X, THREAD_GROUP_Y, 1)]
void CSMain(uint3 dispatchId : SV_DispatchThreadID,
            uint  groupIndex : SV_GroupIndex)
{
    //--- ① 共有メモリを初期化 ---
    // 256 ビンを 256 スレッドで分担
    sharedHistogram[groupIndex] = 0;

    GroupMemoryBarrierWithGroupSync();

    //--- ② 各スレッドが 1 ピクセルを処理 ---
    if (dispatchId.x < inputWidth && dispatchId.y < inputHeight)
    {
        const float3 color = gInput.Load(int3(dispatchId.xy, 0)).rgb;

        //--- Rec.709 の輝度(第26章 26.3.2 節)---
        const float luminance =
            dot(color, float3(0.2126f, 0.7152f, 0.0722f));

        const uint bin = LuminanceToBin(luminance);

        //--- 共有メモリへのアトミック加算 ---
        InterlockedAdd(sharedHistogram[bin], 1);
    }

    GroupMemoryBarrierWithGroupSync();

    //--- ③ グローバルへ集約 ---
    InterlockedAdd(gHistogram[groupIndex], sharedHistogram[groupIndex]);
}
```

**スレッドグループを `16 × 16 = 256` にしているのには理由があります。**

- **256 スレッド = 8 warp**(32 の倍数。無駄なし)
- **ビン数 256 と一致**(初期化と集約が 1 スレッド 1 ビンで済む)

**共有メモリへのアトミック操作を使うのが要点です。**

**グローバルメモリへのアトミック操作は遅い**(32.4.4 節の段階 1)ので、**まずグループ内で集約し、最後に 1 回だけグローバルへ書きます。**

### 32.5.3 平均輝度を求める

**ヒストグラムから平均を計算する、2 つ目のコンピュートシェーダーです。**

```hlsl
//=====================================================
// shaders/AverageLuminance.hlsl
//=====================================================

RWStructuredBuffer<uint>  gHistogram   : register(u0);
RWStructuredBuffer<float> gExposure    : register(u1);

groupshared uint sharedHistogram[HISTOGRAM_BINS];

[numthreads(HISTOGRAM_BINS, 1, 1)]
void CSMain(uint groupIndex : SV_GroupIndex)
{
    const uint count = gHistogram[groupIndex];

    //--- ビン番号で重み付け ---
    sharedHistogram[groupIndex] = count * groupIndex;

    GroupMemoryBarrierWithGroupSync();

    //--- 木構造で縮約 ---
    for (uint stride = HISTOGRAM_BINS / 2; stride > 0; stride >>= 1)
    {
        if (groupIndex < stride)
        {
            sharedHistogram[groupIndex] += sharedHistogram[groupIndex + stride];
        }
        GroupMemoryBarrierWithGroupSync();
    }

    if (groupIndex == 0)
    {
        //--- 黒ピクセル(ビン 0)を除いた平均 ---
        const uint blackCount = gHistogram[0];
        const uint totalCount = max(pixelCount - blackCount, 1u);

        const float weightedAverage =
            (float)sharedHistogram[0] / (float)totalCount - 1.0f;

        //--- ビン番号を輝度へ戻す ---
        const float averageLuminance = exp2(
            weightedAverage / 254.0f * logLuminanceRange + minLogLuminance);

        //--- 時間方向に平滑化(急激な変化を避ける)---
        const float previous = gExposure[0];
        const float adapted = previous +
            (averageLuminance - previous) * adaptationRate;

        gExposure[0] = adapted;
    }

    //--- 次フレームのためにクリア ---
    gHistogram[groupIndex] = 0;
}
```

**時間方向の平滑化が重要です。**

**急に明るい場所へ移動したとき、瞬間的に露出が変わると不自然です。** 人間の目の順応を模して、徐々に変化させます。

**最後にヒストグラムをクリアしている**ので、別途クリア処理が不要になります。

### 32.5.4 パイプラインへの統合

**第26章のパイプラインに、2 つのディスパッチを追加します。**

```
①  シャドウマップ
②  不透明 / 半透明
③  リゾルブ
④  ★ ヒストグラム生成(コンピュート)
⑤  ★ 平均輝度の計算(コンピュート)
⑥  ブルーム
⑦  合成 + トーンマップ  ← 露出バッファを参照
```

```cpp
void Renderer::ComputeAutoExposure()
{
    GPU_MARKER(m_commandList.Get(), m_aftermathContext, "Auto Exposure");

    //--- 入力を読み取り可能に ---
    TransitionTo(m_commandList7.Get(), m_sceneColor,
                 BarrierStates::kNonPixelShaderResource);

    m_commandList->SetComputeRootSignature(m_histogramRootSig.Get());

    //--- ① ヒストグラム生成 ---
    m_commandList->SetPipelineState(m_histogramPso.Get());
    m_commandList->SetComputeRootDescriptorTable(0, m_sceneColorSrv.gpu);
    m_commandList->SetComputeRootDescriptorTable(1, m_histogramUav.gpu);

    m_commandList->Dispatch(
        DivideRoundUp(m_width,  16),
        DivideRoundUp(m_height, 16),
        1);

    //--- UAV バリア(32.2.4 節)---
    const auto barrier = MakeUavBufferBarrier(m_histogramBuffer.Get());
    D3D12_BARRIER_GROUP group{
        .Type = D3D12_BARRIER_TYPE_BUFFER,
        .NumBarriers = 1,
        .pBufferBarriers = &barrier };
    m_commandList7->Barrier(1, &group);

    //--- ② 平均輝度 ---
    m_commandList->SetPipelineState(m_averageLuminancePso.Get());
    m_commandList->Dispatch(1, 1, 1);
}
```

**`kNonPixelShaderResource` という新しい状態が出てきました。**

**コンピュートシェーダーは「ピクセルシェーダー以外」に分類されます。** 第30章 30.4.1 節の対応表を参照してください。

```cpp
inline constexpr BarrierState kNonPixelShaderResource{
    D3D12_BARRIER_SYNC_COMPUTE_SHADING,
    D3D12_BARRIER_ACCESS_SHADER_RESOURCE,
    D3D12_BARRIER_LAYOUT_SHADER_RESOURCE,
};
```

---

## 32.6 実例 2:GPU パーティクル

### 32.6.1 なぜ GPU で処理するか

**数万個のパーティクルを CPU で更新すると、それだけでフレームを使い切ります。**

```
10 万個 × (位置更新 + 速度更新 + 寿命管理) = 数ミリ秒
```

**GPU なら、数万スレッドが並列に処理します。**

**さらに、データが GPU 上に留まる**という利点があります。CPU で計算すると、毎フレーム数 MB を転送することになります。

### 32.6.2 データ構造

```cpp
struct Particle
{
    Math::Vector3 position;
    float         life;          // 残り寿命(秒)
    Math::Vector3 velocity;
    float         size;
    Math::Vector4 color;
};
static_assert(sizeof(Particle) == 48);
```

**`life <= 0` を「死んでいる」と表現します。** 別途フラグを持つより効率的です。

### 32.6.3 更新シェーダー

```hlsl
//=====================================================
// shaders/ParticleUpdate.hlsl
//=====================================================

#define THREAD_GROUP_SIZE 64

struct Particle
{
    float3 position;
    float  life;
    float3 velocity;
    float  size;
    float4 color;
};

cbuffer ParticleConstants : register(b0)
{
    float3 gravity;
    float  deltaTime;
    float3 emitterPosition;
    uint   particleCount;
    float  emitRate;
    float  initialLife;
    float2 padding;
};

RWStructuredBuffer<Particle> gParticles : register(u0);

//-----------------------------------------------------
// 単純な疑似乱数(ハッシュベース)
//-----------------------------------------------------
uint WangHash(uint seed)
{
    seed = (seed ^ 61u) ^ (seed >> 16u);
    seed *= 9u;
    seed = seed ^ (seed >> 4u);
    seed *= 0x27d4eb2du;
    seed = seed ^ (seed >> 15u);
    return seed;
}

float RandomFloat(inout uint state)
{
    state = WangHash(state);
    return (float)(state & 0x00FFFFFFu) / (float)0x01000000u;
}

float3 RandomDirection(inout uint state)
{
    const float theta = RandomFloat(state) * 6.28318530718f;
    const float z     = RandomFloat(state) * 2.0f - 1.0f;
    const float r     = sqrt(1.0f - z * z);

    return float3(r * cos(theta), r * sin(theta), z);
}

[numthreads(THREAD_GROUP_SIZE, 1, 1)]
void CSMain(uint3 id : SV_DispatchThreadID,
            uint  seed : SV_GroupIndex)
{
    //--- 範囲チェック(32.3.3 節)---
    if (id.x >= particleCount)
    {
        return;
    }

    Particle p = gParticles[id.x];

    //--- 乱数の種を作る ---
    uint randomState = WangHash(id.x + (uint)(deltaTime * 1000000.0f));

    if (p.life > 0.0f)
    {
        //--- 生きている:更新 ---
        p.velocity += gravity * deltaTime;
        p.position += p.velocity * deltaTime;
        p.life     -= deltaTime;

        //--- 寿命に応じてフェードアウト ---
        p.color.a = saturate(p.life / initialLife);
    }
    else
    {
        //--- 死んでいる:再生成 ---
        p.position = emitterPosition;
        p.velocity = RandomDirection(randomState) *
                     (2.0f + RandomFloat(randomState) * 3.0f);
        p.life     = initialLife * (0.5f + RandomFloat(randomState) * 0.5f);
        p.size     = 0.05f + RandomFloat(randomState) * 0.05f;
        p.color    = float4(1.0f, 0.6f, 0.2f, 1.0f);
    }

    gParticles[id.x] = p;
}
```

**乱数生成に注意が必要です。**

**GPU には `rand()` がありません。** 各スレッドが独立して乱数を生成する必要があります。

**Wang Hash は単純で高速な手法です。** スレッド ID と時刻を種にすることで、スレッドごと・フレームごとに異なる値が得られます。

### 32.6.4 描画する

**パーティクルバッファを、頂点シェーダーから読みます。**

```hlsl
//=====================================================
// shaders/ParticleRender.hlsl
//=====================================================

StructuredBuffer<Particle> gParticles : register(t0);

struct VSOutput
{
    float4 position : SV_Position;
    float2 uv       : TEXCOORD;
    float4 color    : COLOR;
};

//-----------------------------------------------------
// 頂点バッファなしで、パーティクルごとに四角形を作る。
// 第26章 26.2.2 節と同じ手法。
//-----------------------------------------------------
VSOutput VSMain(uint vertexId   : SV_VertexID,
                uint instanceId : SV_InstanceID)
{
    const Particle p = gParticles[instanceId];

    VSOutput output;

    //--- 死んでいるパーティクルは画面外へ ---
    if (p.life <= 0.0f)
    {
        output.position = float4(0, 0, -10, 1);   // クリップされる
        output.uv       = 0;
        output.color    = 0;
        return output;
    }

    //--- 四角形の 4 隅(トライアングルストリップ)---
    const float2 corners[4] = {
        float2(-1, -1), float2(1, -1),
        float2(-1,  1), float2(1,  1),
    };
    const float2 corner = corners[vertexId];

    //--- ビルボード:カメラに正対させる ---
    const float3 cameraRight = float3(view[0][0], view[1][0], view[2][0]);
    const float3 cameraUp    = float3(view[0][1], view[1][1], view[2][1]);

    const float3 worldPos = p.position
                          + cameraRight * (corner.x * p.size)
                          + cameraUp    * (corner.y * p.size);

    output.position = mul(float4(worldPos, 1.0f), viewProjection);
    output.uv       = corner * 0.5f + 0.5f;
    output.color    = p.color;

    return output;
}

float4 PSMain(VSOutput input) : SV_Target
{
    //--- 円形にする ---
    const float2 centered = input.uv * 2.0f - 1.0f;
    const float  distSq   = dot(centered, centered);

    if (distSq > 1.0f)
    {
        discard;
    }

    const float alpha = (1.0f - distSq) * input.color.a;
    return float4(input.color.rgb, alpha);
}
```

**描画は 1 回のドローコールです。**

```cpp
m_commandList->IASetPrimitiveTopology(D3D_PRIMITIVE_TOPOLOGY_TRIANGLESTRIP);
m_commandList->DrawInstanced(4, particleCount, 0, 0);
```

**頂点 4 個 × インスタンス数。** 第25章 25.3.5 節で触れたインスタンシングです。

**ビルボードの計算に、ビュー行列の列を使っています。** 第17章 17.5.2 節で `LookAtLH` を実装したときの 3 軸が、ここで再利用されます。

**加算合成で描くと、火花のように見えます**(第28章 28.1.2 節)。

```cpp
desc.BlendState = AdditiveBlendDesc();
desc.DepthStencilState.DepthWriteMask = D3D12_DEPTH_WRITE_MASK_ZERO;
```

**深度を書かないのは、第28章 28.2.2 節の通りです。**

---

## 32.7 Nsight Graphics で確認する

### 32.7.1 Occupancy を測る

**第29章 29.3.5 節で扱った内容を、実際に使います。**

**GPU Trace でディスパッチを選択すると、occupancy が表示されます。**

```
Dispatch: LuminanceHistogram
  Theoretical Occupancy: 100%
  Achieved Occupancy:     87%
  Registers per Thread:   24
  Shared Memory:          1024 bytes
```

| 指標 | 意味 |
|---|---|
| **Theoretical** | リソース使用量から計算した上限 |
| **Achieved** | 実測値 |

**両者に差がある場合、warp の実行時間にばらつきがあります。**

### 32.7.2 スレッドグループサイズを変えて比べる

**32.3.4 節の表を、実測で確認します。**

```cpp
// dxc -D THREAD_GROUP_SIZE=32 ...
// dxc -D THREAD_GROUP_SIZE=64 ...
// dxc -D THREAD_GROUP_SIZE=100 ...
```

| サイズ | 予想 | 実測(GPU Trace) |
|---|---|---|
| 32 | 1 warp。occupancy は上がるが起動オーバーヘッド | |
| **64** | **2 warp。バランスが良い** | |
| 100 | 4 warp のうち 28 レーンが無駄 | |
| 256 | 8 warp。共有メモリを多く使える | |

**実測値を埋めてみてください。** 環境やシェーダーの内容によって最適値が変わります。

**「32 の倍数にする」という原則は共通ですが、その中でどれが最速かは測らなければ分かりません。**

### 32.7.3 共有メモリの影響

**`groupshared` のサイズを増やして、occupancy の変化を見ます。**

```hlsl
groupshared float dummy[1024];   // 4KB
groupshared float dummy[4096];   // 16KB
groupshared float dummy[8192];   // 32KB(上限)
```

**32KB 使うと、SM に 1 グループしか載りません。**

**第2章 2.3.1 節で説明した「共有メモリが occupancy を制限する」という関係が、数値で確認できます。**

---

## ✅ 本章のゴール:GPU 計算の結果が描画に反映される

### Step 1:コンピュート PSO を作る

```
[Info ] PipelineState.cpp(122): compute PSO created in 1.87 ms
```

**描画用より速く生成されます**(第14章 14.5 節)。ラスタライザや出力の設定がないためです。

### Step 2:輝度ヒストグラムを確認する

**デバッグ表示を用意すると分かりやすくなります。**

```cpp
if (input.WasKeyPressed('H'))
{
    m_showHistogram = !m_showHistogram;
}
```

**ヒストグラムを棒グラフとして描画します。**

- 暗いシーン → 左側に偏る
- 明るいシーン → 右側に偏る

**カメラを動かすと、分布が変化することを確認してください。**

### Step 3:自動露出を有効にする

**明るい場所から暗い場所へ移動してください。**

**露出が徐々に変化します。**

```cpp
constexpr float kAdaptationRate = 1.5f;   // 1 秒あたりの変化率
```

**値を変えて、順応の速さを調整してみてください。**

| 値 | 印象 |
|---|---|
| 0.5 | ゆっくり。自然 |
| **1.5** | **バランスが良い** |
| 10.0 | ほぼ瞬時。不自然 |

### Step 4:範囲チェックを外してみる

```hlsl
// if (id.x >= particleCount) return;   ← コメントアウト
```

**パーティクル数を、グループサイズで割り切れない値にします。**

```cpp
m_particleCount = 10000;   // 10000 / 64 = 156.25
```

**GPU がクラッシュします。**

```
[Fatal] Renderer.cpp(412): === DEVICE LOST ===
[Info ] Aftermath.cpp(185): crash dump written: crash-20260731-153042.nv-gpudmp
```

**第31章の Aftermath で解析してください。**

```
Page Fault:
  GPU Virtual Address: 0x0000000204B4E200
  Resource: 'ParticleBuffer'
    Size: 480000 bytes
    Note: Access beyond the end of the resource.

Active Shaders:
  Compute Shader
    Source: ParticleUpdate.hlsl
    Line 78:  gParticles[id.x] = p;
              Crash location
```

**第31章の実験 B と同じ形ですが、今回は「実際に起こりうるバグ」です。**

**確認したら元に戻してください。**

### Step 5:UAV バリアを外してみる

```cpp
// m_commandList7->Barrier(1, &group);   ← コメントアウト
```

**ヒストグラムの結果が不安定になります。**

**第30章 30.7.1 節で説明した「たまに壊れる」の実例です。**

**GPU-Based Validation を有効にすると検出されます**(第30章 30.6 節)。

**確認したら元に戻してください。**

### Step 6:アトミック操作の性能を比べる

**32.4.4 節の 3 段階を、それぞれ実装して測定します。**

| 実装 | 予想される時間 |
|---|---|
| 段階 1(グローバルアトミック) | 最も遅い |
| 段階 2(共有メモリ縮約) | 大幅に改善 |
| **段階 3(Wave Intrinsics)** | **さらに改善** |

**GPU Trace で測ってください。**

**要素数を増やすほど、差が大きくなります。**

### Step 7:スレッドグループサイズを変える

**32.7.2 節の実験です。**

**`(10, 10, 1)` にすると、22% のレーンが無駄になります**(32.3.4 節)。

**GPU Trace の SM Throughput で、その分の低下が見えるはずです。**

### Step 8:GPU パーティクルを動かす

**数万個のパーティクルが、GPU だけで更新・描画されます。**

```
[Info ] Renderer.cpp(455): particles: 100000, dispatch groups: 1563
```

**CPU 側の処理は、定数バッファの更新だけです。**

**パーティクル数を増やして、フレームレートの変化を見てください。**

| 数 | 予想 |
|---|---|
| 1 万 | 影響なし |
| 10 万 | わずかに低下 |
| 100 万 | 描画がボトルネックになる |

**GPU Trace で、更新と描画のどちらが重いかを確認してください。**

### Step 9:Wave Intrinsics の対応を確認する

```
[Info ] GraphicsDevice.cpp(288): wave ops    : yes
[Info ] GraphicsDevice.cpp(291): wave lanes  : 32 .. 32  (total 5888)
```

**`WaveGetLaneCount()` の値をシェーダーから出力して、この値と一致することを確認してください。**

```hlsl
gDebugOutput[0] = WaveGetLaneCount();
```

---

### 本章の達成状態

- [ ] コンピュート PSO を作った
- [ ] `SetComputeRoot*` を使っている(`SetGraphicsRoot*` ではなく)
- [ ] `ALLOW_UNORDERED_ACCESS` フラグを付けた
- [ ] 構造化バッファの UAV を作った
- [ ] **範囲チェックを入れている**
- [ ] `DivideRoundUp` でグループ数を計算している
- [ ] スレッド数を 32 の倍数にした
- [ ] `groupshared` の同期に `WithGroupSync` を使っている
- [ ] UAV バリアを入れている
- [ ] Wave Intrinsics で縮約を実装した
- [ ] `WaveGetLaneCount()` を使っている(32 をハードコードしていない)
- [ ] 輝度ヒストグラムから自動露出を実装した
- [ ] GPU パーティクルを実装した
- [ ] Nsight Graphics で occupancy を確認した
- [ ] **GPU 計算の結果が描画に反映された**

---

## トラブルシューティング

| 症状 | 原因 | 対処 |
|---|---|---|
| ディスパッチしても何も起きない | `SetGraphicsRoot*` を使った | `SetComputeRoot*` に(32.1.1) |
| UAV 作成に失敗 | `ALLOW_UNORDERED_ACCESS` 忘れ | 32.2.3 |
| ページフォルト | 範囲チェックがない | 32.3.3 |
| 結果が不安定 | UAV バリアがない | 32.2.4 |
| 同上 | `groupshared` の同期不足 | `WithGroupSync`(32.3.5) |
| 極端に遅い | グローバルアトミックの多用 | 共有メモリで縮約(32.4.4) |
| occupancy が低い | 共有メモリの使いすぎ | 32.7.3 |
| 同上 | レジスタの使いすぎ | Shader Profiler で確認 |
| 一部のスレッドが無駄 | スレッド数が 32 の倍数でない | 32.3.4 |
| Wave Intrinsics が使えない | `WaveOps` が偽 | 第7章 7.5.3 節 |
| 乱数が同じ値になる | 種がスレッド間で同一 | ID を種に含める(32.6.3) |
| パーティクルが見えない | 深度書き込み | `ZERO` に(第28章 28.2.2) |

---

## まとめ

**1. コンピュート PSO は驚くほど単純。**
5 フィールドだけです。第14章で苦労した既定値の問題も起きません。

**2. UAV は書き込める SRV。**
「順序なし」とは、複数スレッドの書き込み順序が保証されないという意味です。

**3. 範囲チェックは必須。**
要素数がグループサイズで割り切れないとき、余分なスレッドが起動します。**第31章の実験と同じページフォルトが、実際のバグとして起こります。**

**4. スレッド数は 32 の倍数にする。**
第2章 2.3.1 節の warp が、ここで設計判断になりました。`(8, 8, 1)` が 2D の定番である理由も、これで説明できます。

**5. アトミック操作は階層的に。**
グローバルへの直接アトミックは遅い。**共有メモリで集約し、Wave Intrinsics でさらに減らします。**

**6. `WaveGetLaneCount()` を使う。**
第2章の注意書き通りです。**ただし `groupshared` のサイズだけは、32 を前提にせざるを得ません。**

**7. occupancy は測って判断する。**
共有メモリとレジスタが制限要因です。**高ければ良いという単純な指標ではありません**(第2章 2.3.1 節)。

次章ではバインドレスを扱います。**第21章 21.1.3 節で `DescriptorHandle` に `index` を持たせた理由が、そこで明らかになります。** テクスチャの数だけディスクリプタテーブルを切り替えるのをやめ、**インデックスだけをシェーダーに渡す**設計へ移行します。

---

## 参考リンク

| 内容 | URL |
|---|---|
| コンピュートシェーダーの概要 | https://learn.microsoft.com/ja-jp/windows/win32/direct3d11/direct3d-11-advanced-stages-compute-shader |
| `Dispatch` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/nf-d3d12-id3d12graphicscommandlist-dispatch |
| 順序なしアクセスビュー | https://learn.microsoft.com/ja-jp/windows/win32/direct3d12/uav |
| Wave Intrinsics | https://learn.microsoft.com/ja-jp/windows/win32/direct3dhlsl/hlsl-shader-model-6-0-features-for-direct3d-12 |
| `groupshared` | https://learn.microsoft.com/ja-jp/windows/win32/direct3dhlsl/dx-graphics-hlsl-variable-syntax |
| CUDA Occupancy の考え方 | https://docs.nvidia.com/nsight-compute/ProfilingGuide/ |
