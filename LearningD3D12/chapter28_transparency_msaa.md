# 第28章 半透明とアンチエイリアス

第3部の最終章です。**3 つの章で先送りにした宿題を、ここで片付けます。**

| 宿題 | 出典 |
|---|---|
| **`DEPTH_WRITE_MASK_ZERO` の使いどころ** | 第19章 19.4.2 節 |
| **半透明は奥からソートする** | 第25章 25.5 節 |
| **FLIP モデルは MSAA をサポートしない** | 第11章 11.1.3 節 |

本章で扱うのは、**「完全に不透明で、完全に塗りつぶされる」という前提を崩したときに何が起こるか**です。

ガラス、水、煙、そして物体の輪郭 —— **どれも「部分的にしか覆わない」ものです。** そこにこれまでの手法が通用しなくなります。

**本章のゴール**
アルファブレンドで半透明を描画し、正しい順序でソートする。MSAA を有効にし、リゾルブしてバックバッファへ出力する。

---

## 28.1 ブレンドステート

### 28.1.1 何を計算しているか

**ブレンドは、「これから描く色」と「すでに描かれている色」を混ぜる処理です。**

```
最終色 = SrcColor × SrcBlend  (BlendOp)  DestColor × DestBlend
```

| 記号 | 意味 |
|---|---|
| `SrcColor` | ピクセルシェーダーが出力した色 |
| `DestColor` | レンダーターゲットに既にある色 |
| `SrcBlend` | 出力色に掛ける係数 |
| `DestBlend` | 既存色に掛ける係数 |
| `BlendOp` | 組み合わせ方(通常は加算) |

**第14章 14.4.4 節で作った `DefaultBlendDesc()` は、ブレンドを無効にしていました。**

```cpp
rt.BlendEnable = FALSE;
rt.SrcBlend    = D3D12_BLEND_ONE;
rt.DestBlend   = D3D12_BLEND_ZERO;
```

`Src × 1 + Dest × 0` = `Src` です。**既存の色を完全に上書きします。**

### 28.1.2 主要なブレンドモード

#### アルファブレンド(通常の半透明)

```cpp
rt.BlendEnable = TRUE;
rt.SrcBlend    = D3D12_BLEND_SRC_ALPHA;
rt.DestBlend   = D3D12_BLEND_INV_SRC_ALPHA;
rt.BlendOp     = D3D12_BLEND_OP_ADD;
```

```
最終色 = Src × α + Dest × (1 - α)
```

**α = 0.5 なら、半分ずつ混ざります。** ガラスや水の表現に使います。

#### 加算合成

```cpp
rt.SrcBlend  = D3D12_BLEND_SRC_ALPHA;
rt.DestBlend = D3D12_BLEND_ONE;
```

```
最終色 = Src × α + Dest
```

**光るものに使います。** 炎、爆発、レンズフレア。

**重要な性質があります。加算は順序に依存しません。**

```
A + B + C = C + B + A
```

**つまり、ソートが不要です。** 28.2 節で扱うソート問題を、加算合成なら回避できます。

#### 乗算合成

```cpp
rt.SrcBlend  = D3D12_BLEND_ZERO;
rt.DestBlend = D3D12_BLEND_SRC_COLOR;
```

```
最終色 = Dest × Src
```

**暗くする効果に使います。** 影、汚れ、着色ガラス。**これも順序に依存しません。**

#### プリマルチプライドアルファ

```cpp
rt.SrcBlend  = D3D12_BLEND_ONE;              // ← ALPHA ではない
rt.DestBlend = D3D12_BLEND_INV_SRC_ALPHA;
```

**テクスチャの RGB に、あらかじめ α を掛けておく方式です。**

```
保存時:  RGB × α を保存
描画時:  Src × 1 + Dest × (1 - α)
```

**2 つの利点があります。**

| 利点 | 説明 |
|---|---|
| **フィルタリングが正しくなる** | 通常のアルファでは、透明部分の色が滲み出る |
| **通常合成と加算合成を統一できる** | α = 0 なら加算、α = 1 なら不透明 |

**通常のアルファでバイリニア補間をすると、透明な黒(0,0,0,0)と不透明な白(1,1,1,1)の中間が、暗い半透明になります。** 輪郭が黒く縁取られる原因です。

**プリマルチプライドなら、この問題が起きません。** DirectXTK など多くのライブラリが既定でこの方式を採るのは、そのためです。

### 28.1.3 独立ブレンド

**レンダーターゲットが複数ある場合、それぞれに別のブレンド設定を与えられます。**

```cpp
desc.IndependentBlendEnable = TRUE;
desc.RenderTarget[0] = /* 通常合成 */;
desc.RenderTarget[1] = /* ブレンドなし */;
```

**`FALSE` の場合、`RenderTarget[0]` の設定が全部に適用されます。**

第14章 14.4.4 節の `DefaultBlendDesc()` では `FALSE` にして、全要素に同じ設定を入れていました。**本章でも変更しません。**

**MRT(Multiple Render Targets)を使うのは、遅延レンダリングなどの応用です。** 本書では扱いません。

### 28.1.4 ヘルパーを追加する

```cpp
// src/Graphics/D3D12Helpers.h に追加

//---------------------------------------------------------------
// アルファブレンド用のブレンド設定。
//---------------------------------------------------------------
[[nodiscard]] inline D3D12_BLEND_DESC AlphaBlendDesc() noexcept
{
    auto desc = DefaultBlendDesc();

    auto& rt = desc.RenderTarget[0];
    rt.BlendEnable    = TRUE;
    rt.SrcBlend       = D3D12_BLEND_SRC_ALPHA;
    rt.DestBlend      = D3D12_BLEND_INV_SRC_ALPHA;
    rt.BlendOp        = D3D12_BLEND_OP_ADD;
    rt.SrcBlendAlpha  = D3D12_BLEND_ONE;
    rt.DestBlendAlpha = D3D12_BLEND_INV_SRC_ALPHA;
    rt.BlendOpAlpha   = D3D12_BLEND_OP_ADD;

    //--- IndependentBlendEnable が FALSE なので [0] が全体に適用される ---
    return desc;
}

//---------------------------------------------------------------
// 加算合成。順序に依存しない。
//---------------------------------------------------------------
[[nodiscard]] inline D3D12_BLEND_DESC AdditiveBlendDesc() noexcept
{
    auto desc = DefaultBlendDesc();

    auto& rt = desc.RenderTarget[0];
    rt.BlendEnable    = TRUE;
    rt.SrcBlend       = D3D12_BLEND_SRC_ALPHA;
    rt.DestBlend      = D3D12_BLEND_ONE;
    rt.BlendOp        = D3D12_BLEND_OP_ADD;
    rt.SrcBlendAlpha  = D3D12_BLEND_ZERO;
    rt.DestBlendAlpha = D3D12_BLEND_ONE;
    rt.BlendOpAlpha   = D3D12_BLEND_OP_ADD;

    return desc;
}
```

**アルファ成分の扱いにも注意が必要です。**

`SrcBlendAlpha` / `DestBlendAlpha` は、**RGB とは独立して指定します。** レンダーターゲットのアルファ値をどう扱うかの設定です。

**バックバッファではアルファが使われないので、あまり気にする必要はありません。** ただし、第26章のオフスクリーンターゲットで合成に使う場合は重要になります。

---

## 28.2 半透明オブジェクトのソート

### 28.2.1 なぜソートが必要か

**アルファブレンドは、順序に依存します。**

```
青いガラス(α=0.5)の後ろに、赤い物体がある場合

正しい順序(奥→手前):
  赤を描く       → Dest = 赤
  青ガラスを描く → 赤 × 0.5 + 青 × 0.5 = 紫

誤った順序(手前→奥):
  青ガラスを描く → 背景 × 0.5 + 青 × 0.5
  赤を描く       → 赤(ガラスを塗りつぶす)  ❌
```

**手前から描くと、後ろのものが見えなくなります。**

### 28.2.2 深度書き込みを止める

**第19章 19.4.2 節で予告した内容です。**

> `ZERO` は半透明描画で使います(第28章)。半透明のものは「奥のものに隠されるか」は判定したいが、「後ろのものを隠す」ことはしたくないからです。

```cpp
auto desc = DefaultGraphicsPipelineStateDesc();
desc.BlendState = AlphaBlendDesc();

//--- 深度テストは行うが、書き込まない ---
desc.DepthStencilState.DepthEnable    = TRUE;                        // 読む
desc.DepthStencilState.DepthWriteMask = D3D12_DEPTH_WRITE_MASK_ZERO; // 書かない
desc.DepthStencilState.DepthFunc      = kDepthFunc;
```

**なぜ書き込んではいけないのか。**

半透明のガラスが深度を書き込むと、**その後ろにある別の半透明物体が、深度テストで捨てられます。**

```
ガラス A(手前)が深度を書く
        ↓
ガラス B(奥)を描こうとする
        ↓
深度テストで落ちる → A の向こうに B が見えない  ❌
```

**深度を書かなければ、B も描かれます。**

**ただし、読むのは必要です。** 不透明な壁の向こうにあるガラスは、隠れるべきだからです。

### 28.2.3 ソートを実装する

**第25章 25.2.3 節で `distanceToCamera` を計算していました。** ここで使います。

```cpp
void Renderer::SortRenderQueues(
    std::vector<const RenderObject*>& opaque,
    std::vector<const RenderObject*>& transparent)
{
    //--- 不透明:状態優先、次に手前から(第25章 25.5 節)---
    std::ranges::sort(opaque,
        [](const RenderObject* a, const RenderObject* b)
        {
            if (a->psoIndex  != b->psoIndex)  return a->psoIndex  < b->psoIndex;
            if (a->meshIndex != b->meshIndex) return a->meshIndex < b->meshIndex;
            return a->distanceToCamera < b->distanceToCamera;   // 手前から
        });

    //--- 半透明:距離が絶対優先、奥から手前へ ---
    std::ranges::sort(transparent,
        [](const RenderObject* a, const RenderObject* b)
        {
            return a->distanceToCamera > b->distanceToCamera;   // 奥から
        });
}
```

**半透明では、状態によるソートを諦めます。**

**距離の順序が絵の正しさを決めるので、性能より優先されます。** PSO の切り替えが増えても、順序を崩すわけにはいきません。

### 28.2.4 描画の順序

```cpp
void Renderer::DrawScene(const Camera& camera)
{
    //--- ① 不透明を描く(深度書き込みあり)---
    m_commandList->SetPipelineState(m_opaquePso.Get());
    DrawObjects(m_opaqueQueue);

    //--- ② 半透明を描く(深度書き込みなし、奥から)---
    m_commandList->SetPipelineState(m_transparentPso.Get());
    DrawObjects(m_transparentQueue);
}
```

**不透明を先に描くのには、2 つの理由があります。**

| 理由 | 説明 |
|---|---|
| **深度バッファを埋める** | 半透明の深度テストが正しく機能する |
| **Early-Z が効く**(第19章 19.1.3 節) | 隠れる半透明ピクセルを早期に捨てられる |

### 28.2.5 ソートでは解決しない場合

**正直に書いておきます。オブジェクト単位のソートには限界があります。**

**問題 1:貫通するオブジェクト**

```
   A          B
  ／＼      ／
 ／  ＼  ／
／     ×
       ＼
```

**第19章 19.1.2 節で深度バッファの必要性を説明したときと同じ図です。** どちらを先に描いても正しくなりません。

**問題 2:1 つのオブジェクトの中での前後**

半透明の球は、手前の面と奥の面が両方描かれます。**オブジェクト単位のソートでは、内部の順序を制御できません。**

**実用的な回避策があります。**

```cpp
//--- 裏面 → 表面 の 2 パスで描く ---
descBack.RasterizerState.CullMode  = D3D12_CULL_MODE_FRONT;   // 裏面だけ
descFront.RasterizerState.CullMode = D3D12_CULL_MODE_BACK;    // 表面だけ
```

**閉じた形状なら、これで正しい順序になります。**

> **順序独立透明性(OIT)**
>
> ソートを不要にする手法群があります。
>
> | 手法 | 仕組み |
> |---|---|
> | **Weighted Blended OIT** | 深度に応じた重みで近似。**実装が簡単** |
> | **Per-Pixel Linked List** | ピクセルごとにフラグメントを記録してソート。**正確だが重い** |
> | **Depth Peeling** | 層ごとに複数回描画 |
>
> **Weighted Blended OIT は、追加のレンダーターゲット 2 枚と数行のシェーダーで実装できます。** 完全に正確ではありませんが、多くの場合で十分な品質が得られます。
>
> 本書では扱いませんが、**第26章のマルチパス構造の上に素直に乗ります。**

---

## 28.3 アルファテストという別解

### 28.3.1 半透明と切り抜きは違う

**葉、金網、フェンス —— これらは「半透明」ではありません。**

```
半透明:  向こうが透けて見える       → ブレンドが必要
切り抜き: 完全に不透明か、完全に透明 → ブレンド不要
```

**切り抜きなら、ブレンドもソートも不要です。**

```hlsl
float4 PSMain(VSOutput input) : SV_Target
{
    const float4 color = gTexture.Sample(gSampler, input.uv);

    //--- α がしきい値未満なら、このピクセルを破棄 ---
    clip(color.a - 0.5f);

    // ... 通常のライティング ...
}
```

**`clip()` は、引数が負ならピクセルを破棄します。** `discard` と同じです。

**不透明として扱えるので、深度書き込みも Early-Z も使えます。**

### 28.3.2 `clip` のコスト

**ただし、代償があります。**

**第19章 19.1.3 節で書いた通り、`discard` / `clip` を使うと Early-Z が無効になります。**

```
Early-Z:  深度テスト → ピクセルシェーダー    ← 使えない
通常:     ピクセルシェーダー → 深度テスト
```

**ピクセルが残るかどうかがシェーダーの実行後まで分からない**からです。

**それでも、ソートが不要になる利点のほうが大きい場面が多くあります。** 葉が数千枚あるような植生では、ソートは現実的ではありません。

### 28.3.3 アルファトゥカバレッジ

**MSAA を使う場合(28.4 節)、切り抜きの境界を滑らかにできます。**

```cpp
desc.BlendState.AlphaToCoverageEnable = TRUE;
```

**α の値を、カバレッジ(何サンプルを覆うか)に変換します。**

```
α = 0.25 → 4 サンプル中 1 つを覆う
α = 0.50 → 4 サンプル中 2 つを覆う
```

**結果として、境界が階調を持ちます。** 葉の輪郭がギザギザにならなくなります。

**MSAA が有効でなければ効果がありません。**

---

## 28.4 MSAA

### 28.4.1 エイリアシングとは

**三角形の輪郭が階段状になる現象です。**

```
理想:   ╱
実際:  ▛▘
       ▛
```

**原因は、ピクセルが「覆われているか、いないか」の二択で判定されるからです。** 三角形の縁では、実際には部分的にしか覆っていないのに、全部か無かで決まります。

### 28.4.2 MSAA の仕組み

**1 ピクセルに複数のサンプル点を置きます。**

```
1×MSAA:      ●          カバレッジ = 0 or 1

4×MSAA:    ●   ●        カバレッジ = 0, 0.25, 0.5, 0.75, 1
           ●   ●
```

**重要な点:ピクセルシェーダーは 1 回しか実行されません。**

| 処理 | 実行単位 |
|---|---|
| ラスタライズ(カバレッジ判定) | **サンプルごと** |
| 深度テスト | **サンプルごと** |
| **ピクセルシェーダー** | **ピクセルごと(1 回)** |
| 出力の書き込み | 覆われたサンプルにのみ |

**これが MSAA と SSAA の違いです。**

| | SSAA | **MSAA** |
|---|---|---|
| シェーダーの実行 | サンプルごと | **ピクセルごと** |
| コスト | 非常に高い | **中程度** |
| 効果 | すべてに効く | **輪郭のみ** |

**MSAA はテクスチャ内部のエイリアシングには効きません。** 輪郭専用の手法です。

### 28.4.3 FLIP モデルの制約

**第11章 11.1.3 節で書いた通りです。**

> FLIP モデルのスワップチェーンは MSAA をサポートしません。`Count` は必ず `1` です。MSAA を使いたい場合は、別途 MSAA 付きのレンダーターゲットに描いてから、解決(リゾルブ)した結果をバックバッファにコピーします。**第28章で扱います。**

**しかし、第26章で既にオフスクリーン描画へ移行しています。**

```
第26章:  シーン → SceneColor → ポストエフェクト → バックバッファ
```

**SceneColor を MSAA 付きにするだけです。** バックバッファには最終合成の結果を書くので、制約に触れません。

```
第28章:  シーン → SceneColorMS(MSAA)
                     ↓ リゾルブ
                  SceneColor(通常)
                     ↓
                  ポストエフェクト → バックバッファ
```

**第26章でオフスクリーンへ移行しておいたことが、ここで効いています。**

### 28.4.4 対応サンプル数を確認する

**すべての組み合わせが使えるわけではありません。**

```cpp
UINT QueryMaxSampleCount(ID3D12Device* device, DXGI_FORMAT format)
{
    for (UINT count = 8; count >= 1; count /= 2)
    {
        D3D12_FEATURE_DATA_MULTISAMPLE_QUALITY_LEVELS levels{};
        levels.Format      = format;
        levels.SampleCount = count;
        levels.Flags       = D3D12_MULTISAMPLE_QUALITY_LEVELS_FLAG_NONE;

        //--- 第7章 7.5.1 節の QueryFeature を使う ---
        if (QueryFeature(device,
                D3D12_FEATURE_MULTISAMPLE_QUALITY_LEVELS, levels))
        {
            if (levels.NumQualityLevels > 0)
            {
                LOG_INFO(L"MSAA {}x supported ({} quality levels)",
                         count, levels.NumQualityLevels);
                return count;
            }
        }
    }
    return 1;
}
```

**`NumQualityLevels` が 0 なら非対応です。** 戻り値が成功でも、この値の確認が必要です。

**第7章 7.5.1 節で作った `QueryFeature` が、ここでも使えます。**

**NVIDIA の GPU では、通常 8× まで対応しています。** ただし帯域とメモリの消費が大きいので、**4× が実用的な選択です。**

### 28.4.5 MSAA レンダーターゲットを作る

```cpp
auto desc = MakeTexture2DDesc(
    kSceneColorFormat, width, height, 1, 1,
    D3D12_RESOURCE_FLAG_ALLOW_RENDER_TARGET);

desc.SampleDesc.Count   = m_sampleCount;    // ← 4
desc.SampleDesc.Quality = 0;
```

**深度バッファも同じサンプル数にする必要があります。**

```cpp
auto depthDesc = MakeTexture2DDesc(
    DXGI_FORMAT_D32_FLOAT, width, height, 1, 1,
    D3D12_RESOURCE_FLAG_ALLOW_DEPTH_STENCIL);

depthDesc.SampleDesc.Count = m_sampleCount;   // ← 一致させる
```

**食い違うと `OMSetRenderTargets` でエラーになります。**

**ビューの次元も変わります。**

```cpp
//--- RTV ---
rtvDesc.ViewDimension = D3D12_RTV_DIMENSION_TEXTURE2DMS;   // ← MS が付く

//--- DSV ---
dsvDesc.ViewDimension = D3D12_DSV_DIMENSION_TEXTURE2DMS;
```

**`Texture2DMS` には `MipSlice` がありません。** MSAA テクスチャはミップマップを持てないからです。

**PSO も合わせます。**

```cpp
desc.SampleDesc.Count   = m_sampleCount;
desc.SampleDesc.Quality = 0;
```

**忘れると、デバッグレイヤーが「レンダーターゲットとサンプル数が一致しない」とエラーを出します。**

### 28.4.6 リゾルブ

**MSAA テクスチャは、そのままではシェーダーから普通に読めません。** サンプルを 1 つの値にまとめる必要があります。

```cpp
void Renderer::ResolveMsaa()
{
    //--- ① 状態遷移 ---
    TransitionTo(m_commandList.Get(), m_sceneColorMS,
                 D3D12_RESOURCE_STATE_RESOLVE_SOURCE);
    TransitionTo(m_commandList.Get(), m_sceneColor,
                 D3D12_RESOURCE_STATE_RESOLVE_DEST);

    //--- ② リゾルブ ---
    m_commandList->ResolveSubresource(
        m_sceneColor.Get(),     // 出力(通常のテクスチャ)
        0,
        m_sceneColorMS.Get(),   // 入力(MSAA テクスチャ)
        0,
        kSceneColorFormat);

    //--- ③ 読み取り可能な状態へ ---
    TransitionTo(m_commandList.Get(), m_sceneColor,
                 D3D12_RESOURCE_STATE_PIXEL_SHADER_RESOURCE);
}
```

**`RESOLVE_SOURCE` と `RESOLVE_DEST` という専用の状態があります。** 第26章 26.1.4 節で作った状態追跡の仕組みが、そのまま使えます。

**リゾルブは、サンプルの平均を取ります。** 単純な平均なので、**HDR の値では問題が起こることがあります**(28.4.7 節)。

### 28.4.7 HDR との相性問題

**第26章 26.1.2 節で、シーンカラーを HDR フォーマットにしました。**

**MSAA のリゾルブは線形平均です。** そして輝度が極端に高いサンプルがあると、平均が引きずられます。

```
サンプル 1: 輝度 100(明るい光源の縁)
サンプル 2〜4: 輝度 0.1(暗い背景)

平均 = 25.05  →  トーンマッピング後はほぼ白
```

**輪郭が滑らかになるどころか、明るい点が広がります。** これを「ファイアフライ」と呼びます。

**対策があります。**

```hlsl
//--- トーンマップしてから平均し、逆変換する ---
float3 ResolveHdr(float3 samples[4])
{
    float3 sum = 0.0f;
    for (int i = 0; i < 4; ++i)
    {
        //--- 輝度で重み付け(明るいサンプルの影響を抑える)---
        const float luminance = dot(samples[i], float3(0.2126, 0.7152, 0.0722));
        const float weight = 1.0f / (1.0f + luminance);
        sum += samples[i] * weight;
    }
    return sum / 4.0f;
}
```

**これを行うには、`ResolveSubresource` ではなく自前のシェーダーでリゾルブします。**

```hlsl
Texture2DMS<float4> gSourceMS : register(t0);

float4 PSMain(VSOutput input) : SV_Target
{
    uint width, height, sampleCount;
    gSourceMS.GetDimensions(width, height, sampleCount);

    const int2 pixel = int2(input.position.xy);

    float3 sum = 0.0f;
    float  weightSum = 0.0f;

    for (uint i = 0; i < sampleCount; ++i)
    {
        const float3 sample = gSourceMS.Load(pixel, i).rgb;

        const float luminance =
            dot(sample, float3(0.2126f, 0.7152f, 0.0722f));
        const float weight = 1.0f / (1.0f + luminance);

        sum += sample * weight;
        weightSum += weight;
    }

    return float4(sum / weightSum, 1.0f);
}
```

**`Texture2DMS<float4>` と `Load(pixel, sampleIndex)` で、個別のサンプルにアクセスできます。**

**フルスクリーン三角形(第26章 26.2 節)で描けます。** 構造は既にできています。

**本書は、まず `ResolveSubresource` で実装し、必要なら自前リゾルブに切り替える形にします。**

---

## 28.5 パイプライン全体

**第27章までのパイプラインに、MSAA と半透明が加わります。**

```
①  シャドウマップ生成      → ShadowMap
        ↓
②  不透明を描く            → SceneColorMS (MSAA, HDR)
        ↓
③  半透明を描く(奥から)   → 同じターゲット
        ↓
④  リゾルブ                → SceneColor (通常)
        ↓
⑤  ブルーム                → Bloom[0..2]
        ↓
⑥  合成 + トーンマップ     → バックバッファ (_SRGB)
```

```cpp
Core::Status Renderer::RenderFrame(const Camera& camera)
{
    // ... フレームリソースの準備 ...

    //=== ① シャドウマップ(第27章)===
    RenderShadowPass(m_shadowCasters);

    //=== ② ③ シーン(MSAA ターゲットへ)===
    TransitionTo(m_commandList.Get(), m_sceneColorMS,
                 D3D12_RESOURCE_STATE_RENDER_TARGET);

    const auto rtv = m_sceneColorMS.Rtv();
    const auto dsv = m_depthMS.Dsv();

    m_commandList->OMSetRenderTargets(1, &rtv, FALSE, &dsv);
    m_commandList->ClearRenderTargetView(rtv, kSceneClearColor, 0, nullptr);
    m_commandList->ClearDepthStencilView(
        dsv, D3D12_CLEAR_FLAG_DEPTH, kDepthClearValue, 0, 0, nullptr);

    DrawOpaque(camera);          // ②
    DrawTransparent(camera);     // ③

    //=== ④ リゾルブ ===
    ResolveMsaa();

    //=== ⑤ ⑥ ポストエフェクト(第26章)===
    RenderBloom();
    Composite();
}
```

**第26章で作った構造に、パスを 2 つ足しただけです。**

---

## ✅ 本章のゴール:ガラスのような表現ができる

### Step 1:半透明を描く

**マテリアルにアルファを設定します。**

```cpp
material.diffuse = { 0.3f, 0.6f, 0.9f };
material.alpha   = 0.4f;
```

**後ろの物体が透けて見えます。**

### Step 2:ソートを無効にしてみる

```cpp
// std::ranges::sort(transparent, ...);   ← コメントアウト
```

**カメラを回すと、半透明の重なりが不正になります。**

手前のガラスが奥のガラスを塗りつぶしたり、**視点によって見え方が急に変わったりします。**

**28.2.1 節で説明した順序依存性の実物です。**

**確認したら元に戻してください。**

### Step 3:深度書き込みを有効にしてみる

```cpp
desc.DepthStencilState.DepthWriteMask = D3D12_DEPTH_WRITE_MASK_ALL;   // ❌
```

**半透明オブジェクトが重なった部分で、奥のものが消えます。**

**第19章 19.4.2 節で `ZERO` を用意した理由が、実物で確認できます。**

**確認したら元に戻してください。**

### Step 4:加算合成を試す

```cpp
desc.BlendState = AdditiveBlendDesc();
```

**光っているように見えます。**

**ソートを無効にしても、結果が変わらないことを確認してください。** 加算は順序に依存しません(28.1.2 節)。

**第26章のブルームと組み合わせると、より効果的です。**

### Step 5:MSAA を有効にする

```cpp
m_sampleCount = 4;
```

```
[Info ] Renderer.cpp(88): MSAA 4x supported (1 quality levels)
```

**輪郭が滑らかになります。**

**細い線や、遠くの物体の縁で違いが顕著です。**

### Step 6:サンプル数を比べる

| 設定 | 見た目 | メモリ |
|---|---|---|
| 1×(オフ) | ギザギザ | 基準 |
| 2× | やや改善 | 2 倍 |
| **4×** | **十分滑らか** | **4 倍** |
| 8× | わずかに改善 | 8 倍 |

**4× から 8× への改善は、コストに見合いません。**

**フレームレートも比べてください。** 帯域の消費が大きいので、解像度が高いほど差が出ます。

### Step 7:PSO のサンプル数を食い違わせる

```cpp
desc.SampleDesc.Count = 1;   // ❌ レンダーターゲットは 4
```

```
D3D12 ERROR: The sample count of the render target does not match
  the sample count specified in the pipeline state.
```

**デバッグレイヤーが検出します。**

**PSO・レンダーターゲット・深度バッファの 3 つを揃える必要があります。**

**確認したら元に戻してください。**

### Step 8:HDR でのファイアフライを観察する

**明るい光源をシーンに置きます。**

```cpp
material.diffuse = { 20.0f, 18.0f, 15.0f };
```

**その縁で、明るい点が広がって見えることがあります。**

**28.4.7 節の自前リゾルブに切り替えると、改善します。**

**この症状は、シーンの内容によっては目立たないこともあります。** 明るさのコントラストが大きいほど顕著になります。

### Step 9:アルファテストを試す

**葉のようなテクスチャがあれば試せます。**

```hlsl
clip(color.a - 0.5f);
```

**ソートなしで、切り抜きが機能します。**

```cpp
desc.BlendState.AlphaToCoverageEnable = TRUE;
```

**MSAA と組み合わせると、境界が滑らかになります**(28.3.3 節)。

---

### 本章の達成状態

- [ ] `AlphaBlendDesc` と `AdditiveBlendDesc` を自作した
- [ ] 半透明の PSO で `DEPTH_WRITE_MASK_ZERO` を使っている
- [ ] 不透明と半透明でキューを分けた
- [ ] 半透明を奥からソートしている
- [ ] 不透明を先に描いている
- [ ] `QueryFeature` で MSAA の対応を確認した
- [ ] レンダーターゲットと深度バッファのサンプル数を揃えた
- [ ] ビューの次元を `TEXTURE2DMS` にした
- [ ] PSO の `SampleDesc` を合わせた
- [ ] `ResolveSubresource` でリゾルブした
- [ ] `RESOLVE_SOURCE` / `RESOLVE_DEST` へ遷移させた
- [ ] `Resize` で MSAA ターゲットも作り直している
- [ ] **半透明が正しく描画された**
- [ ] **輪郭が滑らかになった**

---

## トラブルシューティング

| 症状 | 原因 | 対処 |
|---|---|---|
| 半透明の重なりが不正 | ソートしていない | 28.2.3 |
| 奥の半透明が消える | 深度を書き込んでいる | `ZERO` に(28.2.2) |
| 半透明が壁を透過する | 深度テストを無効にした | `DepthEnable = TRUE` |
| 輪郭が黒く縁取られる | プリマルチプライド未使用 | 28.1.2 |
| MSAA が効かない | PSO のサンプル数 | 28.4.5 |
| `OMSetRenderTargets` でエラー | RT と DSV のサンプル数不一致 | 28.4.5 |
| リゾルブでエラー | 状態遷移を忘れた | 28.4.6 |
| 明るい点が広がる | HDR のリゾルブ | 自前リゾルブ(28.4.7) |
| アルファテストで Early-Z が効かない | **仕様** | 28.3.2 |
| `AlphaToCoverage` が効かない | MSAA が無効 | 28.3.3 |
| 貫通する半透明が正しくない | ソートの限界 | 28.2.5 |

---

## まとめ

**1. ブレンドは「これから描く色」と「既にある色」を混ぜる。**
アルファブレンドは順序に依存し、加算と乗算は依存しません。**加算合成を選べる場面では、ソート問題を回避できます。**

**2. 半透明は深度を読むが、書かない。**
`DEPTH_WRITE_MASK_ZERO`。第19章 19.4.2 節で予告した用途です。

**3. 半透明では、性能より順序が優先される。**
状態によるソートを諦め、距離だけで並べます。

**4. オブジェクト単位のソートには限界がある。**
貫通する物体、1 つのオブジェクト内の前後。**完全な解決には OIT が必要です。**

**5. 切り抜きは半透明ではない。**
`clip()` で処理すれば、ブレンドもソートも不要です。**代償は Early-Z の無効化です。**

**6. MSAA はピクセルシェーダーを 1 回しか実行しない。**
だから SSAA より安く、輪郭にだけ効きます。

**7. FLIP モデルの制約は、既に回避されていた。**
第26章でオフスクリーン描画へ移行したので、MSAA ターゲットを挟むだけで済みました。**第11章 11.1.3 節の宿題が、構造の変更によって自然に解決しています。**

---

## 第3部を終えて

第1部で三角形を出し、第2部で 3D の基礎を積み、**第3部で実用的なレンダラの形になりました。**

| 章 | 得たもの |
|---|---|
| 第22章 | カメラ操作と、確認する手段 |
| 第23章 | モデルの読み込みと頂点処理 |
| 第24章 | ライティングとガンマ補正 |
| 第25章 | 複数オブジェクトと描画の最適化 |
| 第26章 | オフスクリーン描画とポストエフェクト |
| 第27章 | シャドウマップ |
| 第28章 | 半透明とアンチエイリアス |

**レンダリングパスは、こうなりました。**

```
シャドウマップ → 不透明 → 半透明 → リゾルブ → ブルーム → 合成
```

**そして、リソースの状態遷移が複雑になりました。**

第11章では `PRESENT ↔ RENDER_TARGET` の 2 状態だけでした。**今は 6 つのリソースが、それぞれ複数の状態を行き来しています。**

**第4部では、この複雑さに対処する道具を揃えます。** Nsight Graphics でフレームを解剖し、Enhanced Barriers でバリアを整理し、そして Aftermath で GPU クラッシュの原因を特定します。

**次章から、いよいよ本書の中核であるデバッグ手法に入ります。**

---

## 参考リンク

| 内容 | URL |
|---|---|
| `D3D12_BLEND_DESC` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/ns-d3d12-d3d12_blend_desc |
| `D3D12_BLEND` 列挙型 | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/ne-d3d12-d3d12_blend |
| `ResolveSubresource` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/nf-d3d12-id3d12graphicscommandlist-resolvesubresource |
| マルチサンプリング | https://learn.microsoft.com/ja-jp/windows/win32/direct3d11/d3d10-graphics-programming-guide-rasterizer-stage-rules |
| Weighted Blended OIT | https://jcgt.org/published/0002/02/09/ |
| `Texture2DMS` | https://learn.microsoft.com/ja-jp/windows/win32/direct3dhlsl/sm5-object-texture2dms |
