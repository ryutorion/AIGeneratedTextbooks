# 第26章 オフスクリーン描画とポストエフェクト

第11章から本章の直前まで、**描画先は常にバックバッファでした。**

```cpp
m_commandList->OMSetRenderTargets(1, &backBufferRtv, FALSE, &dsv);
```

**これを変えます。** 一度テクスチャに描き、そのテクスチャを加工してからバックバッファへ出す。**この「一段挟む」ことで、画面全体に対する処理が可能になります。**

```
これまで:  シーン ─────────────────────► バックバッファ

これから:  シーン ──► オフスクリーン ──► 加工 ──► バックバッファ
```

本章で実装するのは、色調変換とブラー、そしてブルームです。**しかし本当の主題は、レンダリングパスを組み立てる仕組みそのものです。** 第27章のシャドウマップも、同じ構造の上に乗ります。

**本章のゴール**
オフスクリーンレンダーターゲットを作り、フルスクリーン三角形でポストエフェクトを適用する。グレースケール、ビネット、ガウスぼかし、ブルームを実装する。

---

## 26.1 オフスクリーンレンダーターゲット

### 26.1.1 何が変わるか

**バックバッファとオフスクリーンターゲットの違いを整理します。**

| | バックバッファ | オフスクリーン |
|---|---|---|
| 作る人 | **DXGI** | **自分** |
| 枚数 | 3 枚(第12章) | **1 枚で足りる** |
| フォーマット | `_UNORM` 固定(第11章 11.1.3 節) | **自由** |
| シェーダーから読む | できない | **できる** |
| 状態遷移 | `PRESENT ↔ RENDER_TARGET` | `RENDER_TARGET ↔ PIXEL_SHADER_RESOURCE` |

**「1 枚で足りる」理由は、第19章 19.2.4 節の深度バッファと同じです。** 同じキューのコマンドは順に実行されるので、2 つのフレームが同時に書き込むことはありません。

**「フォーマットが自由」なのが最大の利点です。**

### 26.1.2 HDR フォーマットを使う

**バックバッファは `R8G8B8A8_UNORM` で、値の範囲は [0, 1] に制限されます。**

**現実の光には上限がありません。** 太陽を直視すれば、紙の白より遥かに明るい。**その差を [0, 1] に押し込めると、明るい部分がすべて 1.0 に張り付きます。**

```
実際の輝度:  0.5   2.0   8.0   50.0
UNORM:       0.5   1.0   1.0    1.0    ← 区別が消える
```

**ブルーム(26.5 節)は、この情報がなければ実装できません。** 「1.0 を超えた部分だけを光らせる」という処理ができないからです。

```cpp
DXGI_FORMAT_R16G16B16A16_FLOAT   // 本書はこれ
```

| フォーマット | ビット/ピクセル | 範囲 |
|---|---|---|
| `R8G8B8A8_UNORM_SRGB` | 32 | [0, 1] |
| **`R16G16B16A16_FLOAT`** | **64** | **広い(半精度浮動小数)** |
| `R11G11B10_FLOAT` | 32 | 広い(**アルファなし**) |

**`R11G11B10_FLOAT` は、容量と品質のバランスが良い選択です。** アルファが不要なら有力な候補になります。本書は分かりやすさを優先して `R16G16B16A16_FLOAT` を使います。

> **sRGB との関係**
>
> 第24章 24.5 節で、RTV を `_SRGB` にしました。**オフスクリーンでは不要です。**
>
> 浮動小数フォーマットは値の範囲が広く、**線形空間のまま保持できる**からです。sRGB 変換は、最後にバックバッファへ書き出すときだけ行います。
>
> つまり、**第24章で `_SRGB` にした RTV は、そのまま使い続けます。** 変換の位置がパイプラインの末尾へ移っただけです。

### 26.1.3 レンダーターゲットを作る

```cpp
// src/Graphics/RenderTexture.h
namespace Graphics
{
    class RenderTexture
    {
    public:
        Core::Status Initialize(ID3D12Device* device,
                                DescriptorHeap& srvHeap,
                                UINT width, UINT height,
                                DXGI_FORMAT format,
                                const Math::Vector4& clearColor,
                                std::wstring_view name);
        void Shutdown();

        ID3D12Resource* Get() const noexcept { return m_resource.Get(); }

        D3D12_CPU_DESCRIPTOR_HANDLE Rtv() const noexcept { return m_rtv; }
        DescriptorHandle             Srv() const noexcept { return m_srv; }

        //--- 現在の状態を追跡する(26.1.4 節)---
        D3D12_RESOURCE_STATES State() const noexcept { return m_state; }
        void SetState(D3D12_RESOURCE_STATES state) noexcept { m_state = state; }

        UINT Width()  const noexcept { return m_width; }
        UINT Height() const noexcept { return m_height; }

    private:
        Microsoft::WRL::ComPtr<ID3D12Resource> m_resource;
        D3D12_CPU_DESCRIPTOR_HANDLE m_rtv{};
        DescriptorHandle            m_srv{};

        D3D12_RESOURCE_STATES m_state = D3D12_RESOURCE_STATE_COMMON;
        UINT m_width  = 0;
        UINT m_height = 0;
        Math::Vector4 m_clearColor{};
    };
}
```

**生成部分です。** 第19章の深度バッファとほぼ同じ形になります。

```cpp
Core::Status RenderTexture::Initialize(...)
{
    //--- ① リソース記述子(第19章 19.2.1 節の MakeTexture2DDesc)---
    auto desc = MakeTexture2DDesc(format, width, height, 1, 1,
                                  D3D12_RESOURCE_FLAG_ALLOW_RENDER_TARGET);

    //--- ② 最適化クリア値(第19章 19.2.3 節)---
    D3D12_CLEAR_VALUE clearValue{};
    clearValue.Format   = format;
    clearValue.Color[0] = clearColor.x;
    clearValue.Color[1] = clearColor.y;
    clearValue.Color[2] = clearColor.z;
    clearValue.Color[3] = clearColor.w;

    const auto heapProps = MakeHeapProperties(D3D12_HEAP_TYPE_DEFAULT);

    HR_TRY(device->CreateCommittedResource(
        &heapProps, D3D12_HEAP_FLAG_NONE, &desc,
        D3D12_RESOURCE_STATE_RENDER_TARGET,   // 初期状態
        &clearValue,
        IID_PPV_ARGS(&m_resource)));

    Core::SetDebugName(m_resource.Get(), name);
    m_state = D3D12_RESOURCE_STATE_RENDER_TARGET;

    //--- ③ RTV(専用のヒープから)---
    m_rtv = rtvHeap.Allocate();
    device->CreateRenderTargetView(m_resource.Get(), nullptr, m_rtv);

    //--- ④ SRV(永続領域から。第21章 21.1.3 節)---
    m_srv = srvHeap.Static().Allocate();

    D3D12_SHADER_RESOURCE_VIEW_DESC srvDesc{};
    srvDesc.Format                  = format;
    srvDesc.ViewDimension           = D3D12_SRV_DIMENSION_TEXTURE2D;
    srvDesc.Shader4ComponentMapping = D3D12_DEFAULT_SHADER_4_COMPONENT_MAPPING;
    srvDesc.Texture2D.MipLevels     = 1;

    device->CreateShaderResourceView(m_resource.Get(), &srvDesc, m_srv.cpu);

    m_width  = width;
    m_height = height;
    m_clearColor = clearColor;

    return {};
}
```

**`ALLOW_RENDER_TARGET` フラグが必須です。** 第19章の `ALLOW_DEPTH_STENCIL` と同じ位置づけで、忘れると RTV の作成に失敗します。

**RTV 用のヒープが新たに必要になります。** 第11章で作ったヒープはバックバッファ 3 枚ぶんしかありません。**オフスクリーンターゲットの数だけ拡張します。**

### 26.1.4 状態遷移を追跡する

**第11章 11.5.1 節で書いた通り、D3D12 はリソースの現在状態を教えてくれません。**

バックバッファは `PRESENT ↔ RENDER_TARGET` の 2 状態だけだったので手で追えました。**オフスクリーンターゲットは複雑になります。**

```
シーン描画:     RENDER_TARGET
      ↓
ブラー入力:     PIXEL_SHADER_RESOURCE
      ↓
再利用:         RENDER_TARGET
      ↓
最終合成:       PIXEL_SHADER_RESOURCE
```

**クラス内に状態を保持し、遷移を関数にまとめます。**

```cpp
void TransitionTo(ID3D12GraphicsCommandList* commandList,
                  RenderTexture& texture,
                  D3D12_RESOURCE_STATES newState)
{
    if (texture.State() == newState)
    {
        return;      // 既にその状態なら何もしない
    }

    const auto barrier = MakeTransitionBarrier(
        texture.Get(), texture.State(), newState);

    commandList->ResourceBarrier(1, &barrier);
    texture.SetState(newState);
}
```

**「既にその状態なら何もしない」という判定が重要です。** 同じ状態への遷移はデバッグレイヤーがエラーとして報告します。

> **これが Enhanced Barriers の動機**
>
> 第30章で扱う Enhanced Barriers は、この「状態を自分で追跡する」という設計自体を見直したものです。
>
> **本章のコードは、その必要性を実感するための材料でもあります。** レンダーパスが増えるほど、追跡は複雑になります。

---

## 26.2 フルスクリーン三角形

### 26.2.1 四角形ではなく三角形

**画面全体を覆うには、四角形を描くのが自然に思えます。**

```
  ┌─────┐        ┌─────┐
  │╲    │        │    ╱│
  │  ╲  │   +    │  ╱  │     三角形 2 枚
  │    ╲│        │╱    │
  └─────┘        └─────┘
```

**しかし、画面より大きい三角形 1 枚のほうが優れています。**

```
      ╱╲
     ╱  ╲
    ╱┌──┐╲       画面はこの中に収まる
   ╱ │  │ ╲
  ╱  └──┘  ╲
 ╱___________╲
```

**理由は 2 つあります。**

**理由 1:対角線がなくなる**

四角形 2 枚では、中央に対角線が走ります。**その線上のピクセルは、両方の三角形で処理されることがあります。** GPU は 2×2 のクアッド単位でピクセルシェーダーを実行するため、境界付近で無駄が生じます。

**理由 2:頂点が 3 つで済む**

処理する頂点が 4 つから 3 つに減ります。**微々たる差ですが、無料の改善です。**

### 26.2.2 頂点バッファなしで描く

**第15章 15.5.4 節のコラムで予告した手法を、ここで使います。**

> 頂点シェーダーで `SV_VertexID` を受け取り、そこから座標を計算する方法です。頂点バッファも入力レイアウトも不要になります。**第26章のフルスクリーン三角形で、この方式を使います。**

```hlsl
//=====================================================
// shaders/FullscreenTriangle.hlsl
//=====================================================

struct VSOutput
{
    float4 position : SV_Position;
    float2 uv       : TEXCOORD;
};

//-----------------------------------------------------
// 頂点バッファを使わずに、画面を覆う三角形を作る。
//
//   id=0 → uv(0,0) → 位置(-1, +1)  左上
//   id=1 → uv(2,0) → 位置(+3, +1)  右上(画面外)
//   id=2 → uv(0,2) → 位置(-1, -3)  左下(画面外)
//-----------------------------------------------------
VSOutput VSMain(uint id : SV_VertexID)
{
    VSOutput output;

    output.uv = float2((id << 1) & 2, id & 2);

    output.position = float4(
        output.uv.x *  2.0f - 1.0f,
        output.uv.y * -2.0f + 1.0f,     // Y を反転(第20章 20.6.1 節)
        0.0f,
        1.0f);

    return output;
}
```

**ビット演算の意味を確認します。**

| id | `(id << 1) & 2` | `id & 2` | uv | 位置 |
|---|---|---|---|---|
| 0 | 0 | 0 | (0, 0) | (-1, +1) |
| 1 | 2 | 0 | (2, 0) | (+3, +1) |
| 2 | 0 | 2 | (0, 2) | (-1, -3) |

**UV の Y を反転している**のは、第20章 20.6.1 節の通り、D3D の UV が左上原点だからです。

**描画は 1 行です。**

```cpp
m_commandList->IASetPrimitiveTopology(D3D_PRIMITIVE_TOPOLOGY_TRIANGLELIST);
m_commandList->DrawInstanced(3, 1, 0, 0);
```

**`IASetVertexBuffers` も `IASetIndexBuffer` も呼びません。**

### 26.2.3 ルートシグネチャから入力アセンブラを外す

**第14章 14.2.3 節で予告した内容が、ここで回収されます。**

> なぜこんなフラグがあるのか。入力アセンブラを使わない描画方式が存在するからです。第15章 15.5 節で触れますが、頂点バッファなしで `SV_VertexID` だけを使って三角形を描く方法があり、**そのときこのフラグは不要です。**

```cpp
//--- ポストエフェクト用のルートシグネチャ ---
versioned.Desc_1_1.Flags =
      D3D12_ROOT_SIGNATURE_FLAG_DENY_VERTEX_SHADER_ROOT_ACCESS
    | D3D12_ROOT_SIGNATURE_FLAG_DENY_HULL_SHADER_ROOT_ACCESS
    | D3D12_ROOT_SIGNATURE_FLAG_DENY_DOMAIN_SHADER_ROOT_ACCESS
    | D3D12_ROOT_SIGNATURE_FLAG_DENY_GEOMETRY_SHADER_ROOT_ACCESS;
    // ALLOW_INPUT_ASSEMBLER_INPUT_LAYOUT を付けない
```

**`DENY_VERTEX_SHADER_ROOT_ACCESS` も付けられます。** 頂点シェーダーは `SV_VertexID` しか使わず、定数もテクスチャも読まないからです。

**PSO では入力レイアウトを空にします。**

```cpp
auto desc = DefaultGraphicsPipelineStateDesc();
desc.InputLayout.pInputElementDescs = nullptr;    // ← 空
desc.InputLayout.NumElements        = 0;
desc.DepthStencilState.DepthEnable  = FALSE;      // 深度テストは不要
desc.DSVFormat                      = DXGI_FORMAT_UNKNOWN;
```

**深度テストを無効にする**のを忘れないでください。画面全体を無条件に塗るので、深度は関係ありません。

---

## 26.3 ポストエフェクトの基本形

### 26.3.1 パスの構造

**すべてのポストエフェクトは、同じ形をしています。**

```
① 入力テクスチャを PIXEL_SHADER_RESOURCE へ遷移
② 出力先を RENDER_TARGET へ遷移
③ OMSetRenderTargets で出力先を設定
④ ビューポートとシザー矩形を設定
⑤ PSO と定数を設定
⑥ フルスクリーン三角形を描画
```

**共通処理をまとめます。**

```cpp
void Renderer::DrawFullscreenPass(
    ID3D12PipelineState* pso,
    std::span<const DescriptorHandle> inputs,
    D3D12_CPU_DESCRIPTOR_HANDLE outputRtv,
    UINT width, UINT height,
    D3D12_GPU_VIRTUAL_ADDRESS constantsAddress)
{
    //--- 出力先を設定 ---
    m_commandList->OMSetRenderTargets(1, &outputRtv, FALSE, nullptr);

    //--- ビューポート(出力先のサイズに合わせる)---
    D3D12_VIEWPORT viewport{};
    viewport.Width    = static_cast<float>(width);
    viewport.Height   = static_cast<float>(height);
    viewport.MinDepth = D3D12_MIN_DEPTH;
    viewport.MaxDepth = D3D12_MAX_DEPTH;

    D3D12_RECT scissor{ 0, 0,
        static_cast<LONG>(width), static_cast<LONG>(height) };

    m_commandList->RSSetViewports(1, &viewport);
    m_commandList->RSSetScissorRects(1, &scissor);

    //--- パイプライン ---
    m_commandList->SetGraphicsRootSignature(m_postRootSignature.Get());
    m_commandList->SetPipelineState(pso);

    if (constantsAddress != 0)
    {
        m_commandList->SetGraphicsRootConstantBufferView(0, constantsAddress);
    }

    //--- 入力テクスチャ ---
    if (!inputs.empty())
    {
        m_commandList->SetGraphicsRootDescriptorTable(1, inputs[0].gpu);
    }

    //--- 描画 ---
    m_commandList->IASetPrimitiveTopology(D3D_PRIMITIVE_TOPOLOGY_TRIANGLELIST);
    m_commandList->DrawInstanced(3, 1, 0, 0);
}
```

**ビューポートを出力先のサイズに合わせている**のが要点です。26.4 節でブラーを実装するとき、**縮小バッファを使う**ので、サイズが毎回変わります。

### 26.3.2 色調変換

**最も単純なポストエフェクトです。**

```hlsl
//=====================================================
// shaders/PostColorGrade.hlsl
//=====================================================

cbuffer PostConstants : register(b0)
{
    float2 invScreenSize;      // 1 / (幅, 高さ)
    float  exposure;
    float  time;

    float  grayscaleAmount;
    float  sepiaAmount;
    float  vignetteStrength;
    float  vignetteRadius;
};

Texture2D    gSource  : register(t0);
SamplerState gSampler : register(s0);

float3 ApplyGrayscale(float3 color, float amount)
{
    // 人間の視感度に合わせた重み(Rec.709)
    const float luminance = dot(color, float3(0.2126f, 0.7152f, 0.0722f));
    return lerp(color, luminance.xxx, amount);
}

float3 ApplySepia(float3 color, float amount)
{
    const float3x3 sepiaMatrix = float3x3(
        0.393f, 0.769f, 0.189f,
        0.349f, 0.686f, 0.168f,
        0.272f, 0.534f, 0.131f);

    return lerp(color, mul(sepiaMatrix, color), amount);
}

float ComputeVignette(float2 uv, float radius, float strength)
{
    const float2 centered = uv - 0.5f;
    const float distance = length(centered) * 1.41421356f;   // 対角で 1.0

    return 1.0f - saturate((distance - radius) * strength);
}

float4 PSMain(VSOutput input) : SV_Target
{
    float3 color = gSource.Sample(gSampler, input.uv).rgb;

    //--- 露出 ---
    color *= exposure;

    //--- 色調 ---
    color = ApplyGrayscale(color, grayscaleAmount);
    color = ApplySepia(color, sepiaAmount);

    //--- ビネット ---
    color *= ComputeVignette(input.uv, vignetteRadius, vignetteStrength);

    return float4(color, 1.0f);
}
```

**グレースケールの重みに注目してください。**

```hlsl
float3(0.2126f, 0.7152f, 0.0722f)
```

**単純平均(1/3 ずつ)ではありません。** 人間の目は緑に最も敏感で、青に鈍いためです。**平均を使うと、緑の物体が不自然に暗く見えます。**

**これは Rec.709(sRGB と同じ色域)の係数です。** 第24章 24.5 節で線形空間を扱うようにしたので、この値が正しく機能します。

---

## 26.4 ガウスぼかし

### 26.4.1 分離可能フィルタ

**2 次元のガウスぼかしを素直に実装すると、非常に重くなります。**

```
5×5 のカーネル → 1 ピクセルあたり 25 回のサンプリング
9×9 のカーネル → 81 回
```

**しかし、ガウス関数には便利な性質があります。**

```
G(x, y) = G(x) × G(y)
```

**横方向と縦方向に分けて 2 回かけると、同じ結果が得られます。**

```
9×9 = 81 回  →  9 + 9 = 18 回
```

**4 倍以上速くなります。** カーネルが大きいほど差が広がります。

### 26.4.2 バイリニア補間を使った半減

**さらに減らせます。**

**GPU のバイリニア補間は、隣接する 2 テクセルの重み付き平均を 1 回のサンプリングで返します。** サンプル位置をテクセルの中間にずらせば、**2 テクセルぶんを 1 回で読めます。**

```
通常:    [t0] [t1] [t2] [t3] [t4]     5 回サンプリング

最適化:  [t0] [t1] [t2] [t3] [t4]
           ↑     ↑     ↑
         3 回で済む
```

**重みとオフセットを事前に計算します。**

```cpp
//---------------------------------------------------------------
// バイリニア補間を利用したガウスカーネルを生成する。
// サンプル数がほぼ半分になる。
//---------------------------------------------------------------
struct GaussianKernel
{
    std::vector<float> offsets;
    std::vector<float> weights;
};

GaussianKernel BuildGaussianKernel(int radius, float sigma)
{
    //--- ① 通常の重みを計算 ---
    std::vector<float> discrete(radius + 1);
    float sum = 0.0f;

    for (int i = 0; i <= radius; ++i)
    {
        const float x = static_cast<float>(i);
        discrete[i] = std::exp(-(x * x) / (2.0f * sigma * sigma));
        sum += (i == 0) ? discrete[i] : discrete[i] * 2.0f;
    }

    for (auto& w : discrete) { w /= sum; }

    //--- ② 2 つずつまとめる ---
    GaussianKernel kernel{};
    kernel.offsets.push_back(0.0f);
    kernel.weights.push_back(discrete[0]);

    for (int i = 1; i + 1 <= radius; i += 2)
    {
        const float w0 = discrete[i];
        const float w1 = discrete[i + 1];
        const float combined = w0 + w1;

        //--- 重み付き平均の位置 ---
        const float offset =
            (i * w0 + (i + 1) * w1) / combined;

        kernel.offsets.push_back(offset);
        kernel.weights.push_back(combined);
    }

    return kernel;
}
```

**シェーダー側です。**

```hlsl
cbuffer BlurConstants : register(b0)
{
    float2 texelSize;        // 1 / テクスチャサイズ
    float2 blurDirection;    // (1,0) = 横, (0,1) = 縦
    int    sampleCount;
    float3 padding;
    float4 offsetsAndWeights[8];   // xy = offset, zw = weight
};

float4 PSMain(VSOutput input) : SV_Target
{
    //--- 中心 ---
    float3 color = gSource.Sample(gSampler, input.uv).rgb
                 * offsetsAndWeights[0].z;

    //--- 左右(または上下)対称にサンプリング ---
    for (int i = 1; i < sampleCount; ++i)
    {
        const float offset = offsetsAndWeights[i].x;
        const float weight = offsetsAndWeights[i].z;

        const float2 delta = blurDirection * texelSize * offset;

        color += gSource.Sample(gSampler, input.uv + delta).rgb * weight;
        color += gSource.Sample(gSampler, input.uv - delta).rgb * weight;
    }

    return float4(color, 1.0f);
}
```

**`blurDirection` を変えるだけで、横と縦を切り替えられます。** PSO は 1 つで済みます。

### 26.4.3 縮小バッファを使う

**ぼかしは、解像度を落としても品質がほとんど変わりません。**

```
フル解像度でぼかす:      1920×1080 × 2 パス
1/4 解像度でぼかす:       480×270  × 2 パス   ← 16 分の 1 のピクセル数
```

**ブラー自体が高周波成分を落とす処理なので、縮小による情報の損失は目立ちません。**

```cpp
//--- 縮小バッファを 2 枚用意する(ピンポン用)---
m_blurTextureA.Initialize(device, ..., width / 4, height / 4, format, ...);
m_blurTextureB.Initialize(device, ..., width / 4, height / 4, format, ...);
```

**「ピンポン」とは、2 枚のバッファを交互に使うことです。**

```
A → (横ブラー) → B → (縦ブラー) → A
```

**同じテクスチャを入力と出力に同時に使うことはできません。** デバッグレイヤーがエラーを出します。

---

## 26.5 ブルーム

### 26.5.1 仕組み

**明るい部分が滲んで見える現象を再現します。**

```
① 明るい部分を抽出       (しきい値以上だけ残す)
        ↓
② ぼかす                 (26.4 節)
        ↓
③ 元の絵に加算合成
```

**26.1.2 節で HDR フォーマットにした理由が、ここで効きます。** `_UNORM` では 1.0 を超える値が保持されないので、「明るい部分」を区別できません。

### 26.5.2 明るい部分の抽出

```hlsl
cbuffer BloomConstants : register(b0)
{
    float threshold;      // これを超えた部分だけ残す
    float softKnee;       // 境界の滑らかさ
    float intensity;
    float padding;
};

float4 PSMain(VSOutput input) : SV_Target
{
    const float3 color = gSource.Sample(gSampler, input.uv).rgb;

    //--- 輝度を計算(26.3.2 節と同じ重み)---
    const float luminance = dot(color, float3(0.2126f, 0.7152f, 0.0722f));

    //--- soft knee でしきい値付近を滑らかにする ---
    const float knee = threshold * softKnee;
    float soft = luminance - threshold + knee;
    soft = clamp(soft, 0.0f, 2.0f * knee);
    soft = soft * soft / (4.0f * knee + 0.0001f);

    const float contribution =
        max(soft, luminance - threshold) / max(luminance, 0.0001f);

    return float4(color * contribution, 1.0f);
}
```

**単純な `if (luminance > threshold)` では、境界に不自然な線が出ます。**

明るさがゆっくり変化する部分で、**しきい値をまたいだ瞬間に急に光り出す**からです。soft knee は、その境界を滑らかにします。

### 26.5.3 多段ブラー

**より自然な滲みを得るには、複数の解像度でぼかして合成します。**

```
1/2 解像度でぼかす  →  細かい滲み
1/4 解像度でぼかす  →  中程度の滲み
1/8 解像度でぼかす  →  大きな滲み
        ↓
   すべて加算
```

**単一解像度では、「小さく鋭い光」か「大きくぼんやりした光」のどちらかにしかなりません。** 多段にすることで、両方の性質を持たせられます。

**本書は 3 段にします。** 実用的なエンジンでは 5〜6 段使うこともあります。

```cpp
//--- 段階的に縮小しながらぼかす ---
for (int level = 0; level < kBloomLevels; ++level)
{
    const UINT w = m_width  >> (level + 1);
    const UINT h = m_height >> (level + 1);

    //--- 前段からダウンサンプル ---
    DrawFullscreenPass(m_downsamplePso.Get(),
                       { previousSrv }, m_bloomTextures[level].Rtv(), w, h, 0);

    //--- 横ブラー → 縦ブラー ---
    BlurTexture(m_bloomTextures[level], m_bloomTemp[level], w, h);

    previousSrv = m_bloomTextures[level].Srv();
}
```

### 26.5.4 合成

```hlsl
float4 PSMain(VSOutput input) : SV_Target
{
    float3 color = gSceneColor.Sample(gSampler, input.uv).rgb;

    //--- 各段のブルームを加算 ---
    float3 bloom = 0.0f;
    bloom += gBloom0.Sample(gSampler, input.uv).rgb;
    bloom += gBloom1.Sample(gSampler, input.uv).rgb;
    bloom += gBloom2.Sample(gSampler, input.uv).rgb;

    color += bloom * bloomIntensity;

    //--- トーンマッピング(26.5.5 節)---
    color = ToneMap(color * exposure);

    return float4(color, 1.0f);
}
```

### 26.5.5 トーンマッピング

**HDR の値を [0, 1] へ写す処理が必要です。**

**単純にクランプすると、明るい部分がすべて白飛びします。**

```hlsl
// ❌ 情報が失われる
color = saturate(color);
```

**トーンマッピング関数を使います。**

```hlsl
//--- Reinhard(単純)---
float3 ToneMapReinhard(float3 color)
{
    return color / (1.0f + color);
}

//--- ACES 近似(映画的)---
float3 ToneMapACES(float3 color)
{
    const float a = 2.51f;
    const float b = 0.03f;
    const float c = 2.43f;
    const float d = 0.59f;
    const float e = 0.14f;

    return saturate((color * (a * color + b)) /
                    (color * (c * color + d) + e));
}
```

| 手法 | 特徴 |
|---|---|
| **Reinhard** | 単純。全体が眠い印象になりやすい |
| **ACES 近似** | **コントラストが高く、映画的**。広く使われる |

**本書は ACES 近似を使います。**

> **トーンマッピングと sRGB の順序**
>
> ```
> 線形 HDR → トーンマッピング → 線形 LDR → sRGB 変換
> ```
>
> **sRGB 変換は最後です。** そして第24章 24.5.4 節の通り、**RTV を `_SRGB` にしてあるのでハードウェアが自動で行います。**
>
> シェーダーの出力は線形のままで正しいのです。

---

## 26.6 パイプライン全体

### 26.6.1 構成

```
①  シーン描画        → SceneColor (HDR, R16G16B16A16_FLOAT)
                        + 深度バッファ

②  明部抽出          → Bloom[0] (1/2)
③  ダウンサンプル     → Bloom[1] (1/4), Bloom[2] (1/8)
④  各段をブラー       → 横 → 縦

⑤  合成 + トーンマップ → バックバッファ (_SRGB RTV)
```

### 26.6.2 実装

```cpp
Core::Status Renderer::RenderFrame(const Camera& camera)
{
    // ... フレームリソースの準備(第12章)...

    //=== ① シーンを HDR ターゲットへ描く ===
    TransitionTo(m_commandList.Get(), m_sceneColor,
                 D3D12_RESOURCE_STATE_RENDER_TARGET);

    const auto sceneRtv = m_sceneColor.Rtv();
    const auto dsv = m_dsvHeap->GetCPUDescriptorHandleForHeapStart();

    m_commandList->OMSetRenderTargets(1, &sceneRtv, FALSE, &dsv);
    m_commandList->ClearRenderTargetView(sceneRtv, kSceneClearColor, 0, nullptr);
    m_commandList->ClearDepthStencilView(
        dsv, D3D12_CLEAR_FLAG_DEPTH, kDepthClearValue, 0, 0, nullptr);

    DrawScene(camera);          // 第25章の内容

    //=== ② ブルーム ===
    TransitionTo(m_commandList.Get(), m_sceneColor,
                 D3D12_RESOURCE_STATE_PIXEL_SHADER_RESOURCE);

    RenderBloom();

    //=== ③ 最終合成 ===
    ID3D12Resource* backBuffer = m_swapChain.CurrentBackBuffer();
    {
        const auto barrier = MakeTransitionBarrier(
            backBuffer,
            D3D12_RESOURCE_STATE_PRESENT,
            D3D12_RESOURCE_STATE_RENDER_TARGET);
        m_commandList->ResourceBarrier(1, &barrier);
    }

    const auto backBufferRtv = m_swapChain.CurrentRtv();
    DrawFullscreenPass(m_compositePso.Get(),
                       { m_sceneColor.Srv(), m_bloomTextures[0].Srv() },
                       backBufferRtv, m_width, m_height,
                       postConstantsAddress);

    {
        const auto barrier = MakeTransitionBarrier(
            backBuffer,
            D3D12_RESOURCE_STATE_RENDER_TARGET,
            D3D12_RESOURCE_STATE_PRESENT);
        m_commandList->ResourceBarrier(1, &barrier);
    }

    // ... Close, Execute, Present(第12章)...
}
```

**バックバッファのバリアが、これまでと同じ形で残っている**ことに注目してください。第11章 11.5 節で書いたコードが、パイプラインの末尾へ移動しただけです。

### 26.6.3 リサイズへの対応

**第12章 12.4.5 節に残したコメントを、ここでも回収します。**

```cpp
Core::Status Renderer::Resize(UINT width, UINT height)
{
    // ... スワップチェーンのリサイズ(第12章)...
    // ... 深度バッファの再作成(第19章)...

    //--- ★ オフスクリーンターゲットを作り直す ★ ---
    m_sceneColor.Shutdown();
    if (auto r = m_sceneColor.Initialize(
            m_device, m_descriptorHeap, width, height,
            kSceneColorFormat, kSceneClearColor, L"SceneColor"); !r)
    {
        return r;
    }

    //--- ブルーム用の縮小バッファ ---
    for (int i = 0; i < kBloomLevels; ++i)
    {
        const UINT w = std::max(width  >> (i + 1), 1u);
        const UINT h = std::max(height >> (i + 1), 1u);

        m_bloomTextures[i].Shutdown();
        m_bloomTextures[i].Initialize(m_device, m_descriptorHeap,
                                      w, h, kSceneColorFormat, {}, ...);
    }

    // ... ビューポートとシザー矩形(第15章)...
    return {};
}
```

**`std::max(..., 1u)` を忘れないでください。** 1/8 解像度では、小さいウィンドウでゼロになります。**サイズ 0 のテクスチャは作れません。**

**デスクリプタの解放も必要です。** `Shutdown()` の中で `srvHeap.Static().Free(m_srv)` を呼びます。**第21章 21.1.3 節でフリーリストを実装したのは、このためでした。**

---

## ✅ 本章のゴール:画面全体にエフェクトがかかる

### Step 1:オフスクリーン描画に移行する

**まずエフェクトなしで、そのままコピーするパスを作ります。**

```hlsl
float4 PSMain(VSOutput input) : SV_Target
{
    return gSource.Sample(gSampler, input.uv);
}
```

**見た目が変わらないことを確認してください。**

**変わってしまう場合の原因:**

| 症状 | 原因 |
|---|---|
| 上下が逆 | フルスクリーン三角形の Y 反転(26.2.2) |
| 全体が暗い | トーンマッピングを掛けていない |
| 全体が明るい | sRGB 変換が二重になっている |

### Step 2:グレースケールを試す

```cpp
postConstants.grayscaleAmount = 1.0f;
```

**白黒になります。**

**重みを単純平均に変えてみてください。**

```hlsl
const float luminance = dot(color, float3(0.333f, 0.333f, 0.333f));
```

**緑の物体が不自然に暗くなります。** Rec.709 の重みが必要な理由が分かります。

### Step 3:ビネットを試す

```cpp
postConstants.vignetteStrength = 1.5f;
postConstants.vignetteRadius   = 0.5f;
```

**画面の四隅が暗くなります。**

**半径を 0 にすると、中央まで暗くなります。** パラメータの意味を確認してください。

### Step 4:ブラーを確認する

**1 方向だけ有効にしてみてください。**

```cpp
blurConstants.blurDirection = { 1.0f, 0.0f };   // 横のみ
```

**横方向にだけ流れたような絵になります。**

**縦も掛けると、等方的なぼけになります。** これが分離可能フィルタの効果です(26.4.1 節)。

### Step 5:バイリニア最適化を外してみる

```cpp
// 通常のカーネル(オフセットが整数)
kernel.offsets = { 0, 1, 2, 3, 4 };
```

**見た目はほとんど変わりません。** しかしサンプリング回数が増えています。

**Nsight Graphics(第29章)で測ると、差が確認できます。** 現時点では「同じ結果がより少ないサンプルで得られている」ことを理解しておけば十分です。

### Step 6:ブルームを試す

**明るい光源をシーンに置いてください。**

```cpp
// マテリアルの色を 1.0 より大きくする
material.diffuse = { 5.0f, 4.5f, 3.0f };
```

**その部分が滲んで光ります。**

**しきい値を変えてみてください。**

| threshold | 結果 |
|---|---|
| 0.5 | ほぼ全体が光る |
| **1.0** | **1.0 を超えた部分だけ** |
| 3.0 | 極端に明るい部分だけ |

### Step 7:HDR を無効にしてみる

**シーンカラーのフォーマットを `_UNORM` に変えます。**

```cpp
constexpr DXGI_FORMAT kSceneColorFormat = DXGI_FORMAT_R8G8B8A8_UNORM;
```

**ブルームがほとんど出なくなります。**

1.0 を超える値が保持されないため、**しきい値を超える部分が存在しなくなる**からです。

**26.1.2 節で HDR フォーマットを選んだ理由が、実物で確認できます。**

**確認したら元に戻してください。**

### Step 8:同じテクスチャを入出力にしてみる

```cpp
// A → (ブラー) → A   ❌
```

```
D3D12 ERROR: Resource state conflict: the resource is used as both
  render target and shader resource in the same draw call.
```

**デバッグレイヤーが検出します。** ピンポンバッファが必要な理由です(26.4.3 節)。

**確認したら元に戻してください。**

---

### 本章の達成状態

- [ ] オフスクリーンターゲットを HDR フォーマットで作った
- [ ] `ALLOW_RENDER_TARGET` フラグを付けた
- [ ] 状態遷移を追跡する仕組みを作った
- [ ] フルスクリーン三角形を `SV_VertexID` で描いている
- [ ] 入力レイアウトを空にした
- [ ] ルートシグネチャから `ALLOW_INPUT_ASSEMBLER` を外した
- [ ] グレースケールに Rec.709 の重みを使った
- [ ] ガウスぼかしを分離可能フィルタで実装した
- [ ] バイリニア補間でサンプル数を半減させた
- [ ] 縮小バッファでブラーを掛けている
- [ ] ピンポンバッファを使っている
- [ ] soft knee で明部抽出の境界を滑らかにした
- [ ] 多段ブルームを実装した
- [ ] ACES 近似でトーンマッピングした
- [ ] `Resize` でオフスクリーンターゲットを作り直している
- [ ] **画面全体にエフェクトがかかった**

---

## トラブルシューティング

| 症状 | 原因 | 対処 |
|---|---|---|
| 上下が逆 | UV の Y 反転忘れ | 26.2.2 |
| RTV 作成に失敗 | `ALLOW_RENDER_TARGET` 忘れ | 26.1.3 |
| 状態遷移でエラー | 同じ状態への遷移 | 判定を入れる(26.1.4) |
| 入出力の競合 | 同じテクスチャを両方に使用 | ピンポン(26.4.3) |
| ブルームが出ない | フォーマットが `_UNORM` | HDR に(26.1.2) |
| 明部の境界に線が出る | soft knee がない | 26.5.2 |
| 白飛びする | トーンマッピングがない | 26.5.5 |
| 全体が明るすぎる | sRGB 変換が二重 | オフスクリーンは `_SRGB` にしない |
| 緑が不自然に暗い | 輝度の重みが単純平均 | Rec.709(26.3.2) |
| リサイズでクラッシュ | 縮小バッファがサイズ 0 | `std::max(..., 1u)`(26.6.3) |
| デスクリプタが枯渇 | 解放していない | `Free` を呼ぶ(第21章 21.1.3) |
| ブラーが片方向だけ | 2 パス目を忘れた | 26.4.1 |

---

## まとめ

**1. 一段挟むだけで、画面全体の処理が可能になる。**
バックバッファに直接描くのをやめ、テクスチャを経由します。**この構造が、第27章のシャドウマップにも使われます。**

**2. HDR フォーマットがブルームの前提。**
`_UNORM` では 1.0 を超える値が保持されず、「明るい部分」を区別できません。

**3. フルスクリーン三角形は頂点バッファなしで描ける。**
`SV_VertexID` からビット演算で座標を作ります。**第14章で予告した「入力アセンブラを使わない描画」の実例です。**

**4. ガウスぼかしは分離可能。**
横と縦に分けると、9×9 が 81 回から 18 回になります。さらにバイリニア補間で半減できます。

**5. ぼかしは縮小バッファで十分。**
高周波成分を落とす処理なので、解像度を下げても品質差は小さく、コストは 16 分の 1 になります。

**6. トーンマッピングは sRGB 変換の前。**
そして sRGB 変換は、RTV を `_SRGB` にしてハードウェアに任せます(第24章)。

**7. 状態追跡が複雑になってきた。**
パスが増えるほど、リソースの状態を手で追うのは難しくなります。**第30章の Enhanced Barriers が、この問題に答えます。**

次章ではシャドウマップを実装します。**ライト視点から深度を描き、それをテクスチャとして読む**という処理は、本章のオフスクリーン描画の応用です。第19章 19.3.2 節のコラムで予告した `R32_TYPELESS` による 2 通りのビューが、そこで実際に必要になります。

---

## 参考リンク

| 内容 | URL |
|---|---|
| フルスクリーン三角形の手法 | https://wallisc.github.io/rendering/2021/04/18/Fullscreen-Pass.html |
| ブルームの実装 | https://learnopengl.com/Advanced-Lighting/Bloom |
| ACES トーンマッピング近似 | https://knarkowicz.wordpress.com/2016/01/06/aces-filmic-tone-mapping-curve/ |
| ガウスぼかしの最適化 | https://www.rastergrid.com/blog/2010/09/efficient-gaussian-blur-with-linear-sampling/ |
| `D3D12_RESOURCE_FLAGS` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/ne-d3d12-d3d12_resource_flags |
