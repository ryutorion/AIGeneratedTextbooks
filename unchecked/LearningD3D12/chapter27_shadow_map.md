# 第27章 シャドウマップ

**影がないと、物体が地面に浮いて見えます。**

第24章でライティングを実装しました。しかし、あれは「その面が光を向いているか」だけを見ています。**手前に別の物体があって光が遮られている、という判定は一切していません。**

本章で影を落とします。手法はシャドウマップ —— **ライトの視点から見て、一番手前にあるものだけが照らされている**という、単純な発想に基づきます。

そして本章では、これまでの伏線が 2 つ回収されます。

| 伏線 | 出典 |
|---|---|
| **`R32_TYPELESS` による 2 通りのビュー** | 第19章 19.3.2 節 |
| **`BORDER` アドレスモード** | 第20章 20.6.3 節 |

**シャドウマップは「動くところまで」は簡単ですが、「きれいに見せる」のが難しい**技術です。本章の後半は、そのアーティファクトとの戦いになります。

**本章のゴール**
ライト視点から深度を描き、それを参照して影を落とす。シャドウアクネとピーターパン現象を理解し、PCF で境界を滑らかにする。

---

## 27.1 シャドウマップの原理

### 27.1.1 二段階の描画

```
第 1 パス:  ライトの視点から、深度だけを描く
              → シャドウマップ(深度テクスチャ)

第 2 パス:  カメラの視点から、通常どおり描く
              → 各ピクセルについて、シャドウマップと比較する
```

**判定の考え方は単純です。**

```
そのピクセルをライト視点の座標へ変換する
        ↓
シャドウマップの、その位置に記録された深度と比較する
        ↓
記録されている深度のほうが手前 → 何かに遮られている → 影
記録されている深度と同じ       → 自分が一番手前     → 照らされている
```

```
       ライト
         ↓
    ┌────────┐
    │  遮蔽物  │ ← シャドウマップにはここの深度が記録される
    └────────┘
         ↓
    ─────────────  ← このピクセルは、記録より奥にある = 影
```

### 27.1.2 何が難しいのか

**原理は単純ですが、実装すると必ず 2 つの問題にぶつかります。**

| 問題 | 症状 |
|---|---|
| **シャドウアクネ** | 照らされているはずの面に縞模様の影 |
| **ピーターパン現象** | 物体と影が離れて、浮いて見える |

**この 2 つはトレードオフの関係にあります。** 片方を消そうとすると、もう片方が出ます。27.4 節で扱います。

さらに、**シャドウマップは有限の解像度を持つテクスチャ**です。カメラが近づけば、1 テクセルが画面上の何ピクセルにも広がり、**影の境界がギザギザになります。** これは 27.5 節の PCF で緩和します。

---

## 27.2 深度バッファをテクスチャとして読む

### 27.2.1 `D32_FLOAT` では SRV を作れない

**第19章 19.2.2 節で、深度バッファを `D32_FLOAT` で作りました。**

**このフォーマットでは、SRV を作れません。**

```
D3D12 ERROR: CreateShaderResourceView: The format (D32_FLOAT)
  cannot be used with a shader resource view.
```

`D` で始まるフォーマットは**深度専用**です。深度テストとステンシルのために特別扱いされており、汎用のテクスチャとしては読めません。

### 27.2.2 `TYPELESS` で作り、ビューで解釈を与える

**第19章 19.3.2 節のコラムで予告した解決策です。**

> 型を持たない `R32_TYPELESS` でリソースを作り、DSV は `D32_FLOAT`、SRV は `R32_FLOAT` として別々のビューを作ります。

```
リソース:  DXGI_FORMAT_R32_TYPELESS      ← 型を持たない 32bit
   ├─ DSV: DXGI_FORMAT_D32_FLOAT         ← 深度として書く
   └─ SRV: DXGI_FORMAT_R32_FLOAT         ← テクスチャとして読む
```

**第11章 11.2.1 節で「デスクリプタはリソースではなく解釈の指示書」と書きました。** 同じメモリを 2 通りに解釈する、その最も明快な実例です。

第24章 24.5.4 節で `_UNORM` のバックバッファに `_SRGB` の RTV を作ったのも、同じ仕組みでした。

```cpp
//--- リソースは TYPELESS ---
auto desc = MakeTexture2DDesc(
    DXGI_FORMAT_R32_TYPELESS,          // ← ここ
    kShadowMapSize, kShadowMapSize,
    1, 1,
    D3D12_RESOURCE_FLAG_ALLOW_DEPTH_STENCIL);

//--- 最適化クリア値は D32_FLOAT で指定する ---
D3D12_CLEAR_VALUE clearValue{};
clearValue.Format             = DXGI_FORMAT_D32_FLOAT;   // ← TYPELESS ではない
clearValue.DepthStencil.Depth = kShadowClearValue;
clearValue.DepthStencil.Stencil = 0;

HR_TRY(device->CreateCommittedResource(
    &heapProps, D3D12_HEAP_FLAG_NONE, &desc,
    D3D12_RESOURCE_STATE_DEPTH_WRITE,
    &clearValue,
    IID_PPV_ARGS(&m_shadowMap)));
```

**最適化クリア値には `D32_FLOAT` を指定します。** `TYPELESS` のままだとエラーになります。**「どう使うか」が確定していなければ、クリアの最適化はできません。**

```cpp
//--- DSV:深度として書く ---
D3D12_DEPTH_STENCIL_VIEW_DESC dsvDesc{};
dsvDesc.Format             = DXGI_FORMAT_D32_FLOAT;
dsvDesc.ViewDimension      = D3D12_DSV_DIMENSION_TEXTURE2D;
dsvDesc.Flags              = D3D12_DSV_FLAG_NONE;
dsvDesc.Texture2D.MipSlice = 0;

device->CreateDepthStencilView(m_shadowMap.Get(), &dsvDesc, m_dsv);

//--- SRV:テクスチャとして読む ---
D3D12_SHADER_RESOURCE_VIEW_DESC srvDesc{};
srvDesc.Format                  = DXGI_FORMAT_R32_FLOAT;   // ← 違う
srvDesc.ViewDimension           = D3D12_SRV_DIMENSION_TEXTURE2D;
srvDesc.Shader4ComponentMapping = D3D12_DEFAULT_SHADER_4_COMPONENT_MAPPING;
srvDesc.Texture2D.MipLevels     = 1;

device->CreateShaderResourceView(m_shadowMap.Get(), &srvDesc, m_srv.cpu);
```

**第20章 20.5.1 節で強調した `Shader4ComponentMapping` を、ここでも忘れないでください。**

### 27.2.3 状態遷移

**第26章 26.1.4 節で作った仕組みを、そのまま使います。**

```
DEPTH_WRITE                → 第 1 パス(深度を書く)
        ↓
PIXEL_SHADER_RESOURCE      → 第 2 パス(テクスチャとして読む)
        ↓
DEPTH_WRITE                → 次のフレーム
```

**第19章 19.4.4 節で「深度バッファにはバリアが不要」と書きました。** そこにはこう続けています。

> 第26章でポストエフェクトから深度を読むようになると、事情が変わります。`DEPTH_WRITE → PIXEL_SHADER_RESOURCE → DEPTH_WRITE` という往復が必要になります。

**シャドウマップが、まさにその形です。**

---

## 27.3 ライト視点の行列

### 27.3.1 平行光源には平行投影を使う

**第24章 24.3.1 節で、平行光源は「向きだけを持つ」と書きました。** 太陽のように、光線が平行に降り注ぐモデルです。

**したがって、射影も平行投影を使います。**

**第17章 17.6.5 節で `OrthographicLH` を用意しておいたのは、このためでした。**

```cpp
Math::Matrix4x4 BuildLightViewProjection(
    const Math::Vector3& lightDirection,
    const Math::Vector3& sceneCenter,
    float sceneRadius)
{
    //--- ライトの位置を、シーンの外側に取る ---
    const auto direction = Math::Normalize(lightDirection);
    const auto lightPos  = sceneCenter - direction * (sceneRadius * 2.0f);

    //--- up が視線と平行にならないよう選ぶ ---
    const Math::Vector3 up =
        (std::abs(direction.y) > 0.99f)
        ? Math::Vector3{ 0.0f, 0.0f, 1.0f }
        : Math::Vector3{ 0.0f, 1.0f, 0.0f };

    const auto view = Math::LookAtLH(lightPos, sceneCenter, up);

    //--- シーン全体を覆う平行投影 ---
    const float size  = sceneRadius * 2.0f;
    const float nearZ = 0.1f;
    const float farZ  = sceneRadius * 4.0f;

    const auto proj = MakeOrthographic(size, size, nearZ, farZ);

    return view * proj;
}
```

**`up` の選び方に注目してください。**

第17章 17.5.2 節のコラム、そして第22章 22.3.2 節で扱った問題です。**視線と `up` が平行になると `LookAtLH` が壊れます。**

第22章では入力側でピッチを制限しましたが、**ライトの向きは制限できません。** 真上からの光は普通にありえます。**そこで、視線がほぼ真上・真下なら `up` を切り替えます。**

### 27.3.2 Reversed-Z との整合

**第19章 19.5 節で Reversed-Z を採用した場合、シャドウマップも合わせる必要があります。**

```cpp
// src/Graphics/ShadowConfig.h

#if USE_REVERSED_Z
    inline constexpr float kShadowClearValue = 0.0f;
    inline constexpr D3D12_COMPARISON_FUNC kShadowDepthFunc =
        D3D12_COMPARISON_FUNC_GREATER;
    // 比較サンプラーも反転する
    inline constexpr D3D12_COMPARISON_FUNC kShadowCompareFunc =
        D3D12_COMPARISON_FUNC_GREATER_EQUAL;
#else
    inline constexpr float kShadowClearValue = 1.0f;
    inline constexpr D3D12_COMPARISON_FUNC kShadowDepthFunc =
        D3D12_COMPARISON_FUNC_LESS;
    inline constexpr D3D12_COMPARISON_FUNC kShadowCompareFunc =
        D3D12_COMPARISON_FUNC_LESS_EQUAL;
#endif
```

**第19章 19.5.4 節で「4 箇所すべてを変える必要がある」と書きました。** シャドウマップを追加すると、**さらに 3 箇所増えます。**

| 箇所 | 内容 |
|---|---|
| ① 射影行列 | `MakeOrthographic` で入れ替え |
| ② クリア値 | `kShadowClearValue` |
| ③ 深度テストの比較関数 | `kShadowDepthFunc` |
| ④ **比較サンプラーの関数** | `kShadowCompareFunc`(27.5.2 節) |
| ⑤ **バイアスの符号** | 27.4.2 節 |

**定数を 1 箇所にまとめておく設計が、ここで効きます。**

### 27.3.3 シーンの範囲を求める

**平行投影の範囲は、シーン全体を覆う必要があります。**

```cpp
struct SceneBounds
{
    Math::Vector3 center;
    float         radius = 0.0f;
};

SceneBounds ComputeSceneBounds(const std::vector<RenderObject>& objects,
                               const std::vector<Mesh>& meshes)
{
    Math::Vector3 minBounds{  1e30f,  1e30f,  1e30f };
    Math::Vector3 maxBounds{ -1e30f, -1e30f, -1e30f };

    for (const auto& object : objects)
    {
        const auto& mesh = meshes[object.meshIndex];

        //--- 境界ボックスの 8 隅をワールド空間へ ---
        for (int i = 0; i < 8; ++i)
        {
            const Math::Vector3 corner{
                (i & 1) ? mesh.boundsMax.x : mesh.boundsMin.x,
                (i & 2) ? mesh.boundsMax.y : mesh.boundsMin.y,
                (i & 4) ? mesh.boundsMax.z : mesh.boundsMin.z,
            };

            const auto world =
                Math::TransformPoint(corner, object.worldMatrix);

            minBounds.x = std::min(minBounds.x, world.x);
            minBounds.y = std::min(minBounds.y, world.y);
            minBounds.z = std::min(minBounds.z, world.z);
            maxBounds.x = std::max(maxBounds.x, world.x);
            maxBounds.y = std::max(maxBounds.y, world.y);
            maxBounds.z = std::max(maxBounds.z, world.z);
        }
    }

    SceneBounds bounds{};
    bounds.center = (minBounds + maxBounds) * 0.5f;
    bounds.radius = Math::Length(maxBounds - bounds.center);
    return bounds;
}
```

**シーンが広いほど、シャドウマップの解像度が相対的に落ちます。**

```
シーン半径 10m、シャドウマップ 2048×2048  →  1 テクセル ≈ 1cm
シーン半径 500m、同じ解像度               →  1 テクセル ≈ 50cm
```

**後者では、影の境界が非常に粗くなります。**

> **カスケードシャドウマップ**
>
> 広いシーンでは、**視錐台を距離で分割し、それぞれに別のシャドウマップを割り当てる**手法が使われます。近くは高解像度、遠くは低解像度にすることで、限られたメモリを有効に使えます。
>
> 本書では扱いませんが、**本章の実装を 3〜4 回繰り返すだけ**なので、拡張としては素直です。

---

## 27.4 シャドウアクネとピーターパン現象

### 27.4.1 なぜアクネが出るのか

**照らされているはずの面に、縞模様の影が現れます。**

**原因は、シャドウマップが離散的だからです。**

```
実際の面:   ────────────────
シャドウマップ: ┌─┐ ┌─┐ ┌─┐
              └─┘ └─┘ └─┘   ← テクセル単位の階段状
```

**1 テクセルは、面上のある範囲を代表する 1 つの深度値です。** その範囲内で面が傾いていれば、**手前側は「記録より奥」と判定されます。**

```
        テクセルの記録値
             ↓
    ────────●────────
   ／                ＼
  ／  ここは奥と判定    ＼
     = 誤って影になる
```

**面が光に対して斜めなほど、この誤差が大きくなります。**

### 27.4.2 深度バイアス

**対策は、深度を少しずらすことです。**

```
比較する深度 = 実際の深度 - バイアス
```

**少し手前にあることにすれば、自分自身を影と判定しなくなります。**

**方法は 2 つあります。**

#### 方法 A:ラスタライザステートで設定する

```cpp
auto desc = DefaultRasterizerDesc();
desc.DepthBias            = 1000;      // 整数。深度の最小単位の倍数
desc.SlopeScaledDepthBias = 2.0f;      // 傾きに比例する成分
desc.DepthBiasClamp       = 0.01f;     // 上限
```

**`SlopeScaledDepthBias` が重要です。** 27.4.1 節で見た通り、**誤差は面の傾きに比例します。** 傾いた面ほど大きなバイアスが必要です。

**GPU が自動で計算してくれるので、こちらのほうが優れています。**

#### 方法 B:シェーダーで手動で引く

```hlsl
const float bias = 0.005f;
if (currentDepth - bias > shadowMapDepth) { /* 影 */ }
```

**単純ですが、傾きを考慮できません。**

**本書は A を主に使い、B を補助的に併用します。**

> **Reversed-Z ではバイアスの符号が逆**
>
> 深度が「大きいほど手前」になるので、**バイアスは加算します。**
>
> ```cpp
> #if USE_REVERSED_Z
>     desc.DepthBias = -1000;    // 符号を反転
> #endif
> ```
>
> **符号を間違えると、アクネが悪化するだけです。** 第19章 19.5.4 節で「4 箇所すべて」と書いた項目に、これが加わります。

### 27.4.3 ピーターパン現象

**バイアスを大きくしすぎると、今度は別の問題が出ます。**

```
     物体
      ▮
   ────────
      ↑ 影がここから始まる = 物体が浮いて見える
```

**影が物体から離れます。** ピーターパンが影と離れていた童話にちなんだ名前です。

**アクネとピーターパンはトレードオフです。**

```
バイアス小 → アクネ
バイアス大 → ピーターパン
```

**「ちょうどいい値」を探すしかありません。** シーンのスケールや光の角度によって変わるので、**調整可能にしておくのが実用的です。**

### 27.4.4 表面カリングという手法

**もう一つの対策があります。**

**シャドウマップを描くとき、表面ではなく裏面を描きます。**

```cpp
//--- シャドウパス用の PSO ---
auto desc = DefaultRasterizerDesc();
desc.CullMode = D3D12_CULL_MODE_FRONT;   // ← 表面を捨てる
```

**アクネは「面が自分自身を影と判定する」ことで起きます。** 裏面を記録すれば、**表面は必ず記録より手前**になるので、アクネが原理的に消えます。

**ただし制約があります。**

| 条件 | 結果 |
|---|---|
| 閉じた形状(球、立方体など) | **うまく機能する** |
| 板ポリゴン(葉、旗など) | **裏面がないので影が出ない** |
| 薄い物体 | 影がずれる |

**シーンの内容によって使い分けます。** 本書は既定でバイアスを使い、切り替えられるようにしておきます。

---

## 27.5 PCF で境界を滑らかにする

### 27.5.1 なぜギザギザになるのか

**シャドウマップの 1 テクセルが、画面上の複数ピクセルに対応するためです。**

```
シャドウマップ:  [影][影][光][光]
                  ↓
画面:        ████████░░░░░░░░     ← 階段状の境界
```

**解像度を上げれば緩和されますが、メモリを消費します。** 根本的な解決にはなりません。

### 27.5.2 比較サンプラー

**通常のサンプリングでは、深度値そのものが返ります。**

```hlsl
const float depth = gShadowMap.Sample(gSampler, uv).r;
const bool inShadow = (currentDepth > depth);
```

**比較サンプラーを使うと、比較の結果が返ります。**

```hlsl
const float lit = gShadowMap.SampleCmpLevelZero(
    gShadowSampler, uv, currentDepth);
// 0.0 = 影、1.0 = 照らされている
```

**重要なのは、これがバイリニア補間されることです。**

```
4 テクセルの比較結果:  [0][1]
                       [0][1]
        ↓ 補間
返り値:  0.0 〜 1.0 の連続値
```

**1 回のサンプリングで、2×2 の平均が得られます。** ハードウェアが専用の回路で処理するので、非常に高速です。

**サンプラーの設定です。**

```cpp
D3D12_STATIC_SAMPLER_DESC shadowSampler{};
shadowSampler.Filter =
    D3D12_FILTER_COMPARISON_MIN_MAG_LINEAR_MIP_POINT;   // ← COMPARISON
shadowSampler.AddressU = D3D12_TEXTURE_ADDRESS_MODE_BORDER;
shadowSampler.AddressV = D3D12_TEXTURE_ADDRESS_MODE_BORDER;
shadowSampler.AddressW = D3D12_TEXTURE_ADDRESS_MODE_BORDER;
shadowSampler.ComparisonFunc = kShadowCompareFunc;      // 27.3.2 節
shadowSampler.BorderColor =
    D3D12_STATIC_BORDER_COLOR_OPAQUE_WHITE;             // ← 範囲外は「影でない」
shadowSampler.MaxLOD = D3D12_FLOAT32_MAX;
shadowSampler.ShaderRegister = 1;                       // s1
shadowSampler.ShaderVisibility = D3D12_SHADER_VISIBILITY_PIXEL;
```

**`BORDER` アドレスモードを使っている**のが、第20章 20.6.3 節で予告した内容です。

> `BORDER` は、第27章のシャドウマップで使います。影の範囲外を「影がない」扱いにするためです。

**シャドウマップの範囲外に出たピクセルを、どう扱うか。**

| モード | 結果 |
|---|---|
| `CLAMP` | 端の値が引き伸ばされる → **縞模様が伸びる** |
| `WRAP` | 反対側の値を使う → **でたらめな影** |
| **`BORDER` + 白** | **「影でない」として扱う** |

**白(1.0)を境界色にすると、比較結果が常に「照らされている」になります。**

HLSL 側では `SamplerComparisonState` として宣言します。

```hlsl
SamplerComparisonState gShadowSampler : register(s1);
```

### 27.5.3 複数サンプルの PCF

**比較サンプラー 1 回では 2×2 の平均しか得られません。** より滑らかにするには、複数箇所をサンプリングします。

```hlsl
float SampleShadowPCF(float2 uv, float compareDepth, float2 texelSize)
{
    float sum = 0.0f;

    //--- 3×3 のサンプリング(実質 6×6 テクセル)---
    [unroll]
    for (int y = -1; y <= 1; ++y)
    {
        [unroll]
        for (int x = -1; x <= 1; ++x)
        {
            const float2 offset = float2(x, y) * texelSize;
            sum += gShadowMap.SampleCmpLevelZero(
                gShadowSampler, uv + offset, compareDepth);
        }
    }

    return sum / 9.0f;
}
```

**`[unroll]` を付けている**のは、ループを展開させるためです。**固定回数のループでは、分岐のコストを避けられます。**

| カーネル | サンプル数 | 実質の範囲 | コスト |
|---|---|---|---|
| 1×1 | 1 | 2×2 | 最小 |
| **3×3** | **9** | **6×6** | **実用的** |
| 5×5 | 25 | 10×10 | 重い |

**3×3 が実用と品質のバランス点です。**

> **より高度な手法**
>
> **PCSS(Percentage-Closer Soft Shadows)** は、遮蔽物までの距離に応じてぼかし幅を変えます。**接地部分は鋭く、離れるほど柔らかい影**という、現実に近い表現ができます。
>
> **VSM / ESM** は、深度の分布を統計量として保存し、通常のフィルタリングを可能にします。ミップマップや異方性フィルタが使えるようになります。
>
> 本書では扱いませんが、**PCF の実装を土台にすれば拡張できます。**

---

## 27.6 実装

### 27.6.1 第 1 パス:深度だけを描く

**専用の PSO を作ります。**

```cpp
auto desc = DefaultGraphicsPipelineStateDesc();

desc.pRootSignature = m_shadowRootSignature.Get();
desc.VS             = shadowVs.Bytecode();
desc.PS             = D3D12_SHADER_BYTECODE{};     // ← ピクセルシェーダーなし

desc.InputLayout.pInputElementDescs = kMeshInputElements;
desc.InputLayout.NumElements =
    static_cast<UINT>(std::size(kMeshInputElements));

//--- 深度だけ ---
desc.NumRenderTargets = 0;                          // ← RTV なし
desc.DSVFormat        = DXGI_FORMAT_D32_FLOAT;

desc.DepthStencilState.DepthEnable    = TRUE;
desc.DepthStencilState.DepthWriteMask = D3D12_DEPTH_WRITE_MASK_ALL;
desc.DepthStencilState.DepthFunc      = kShadowDepthFunc;

//--- バイアス(27.4.2 節)---
desc.RasterizerState.DepthBias            = kShadowDepthBias;
desc.RasterizerState.SlopeScaledDepthBias = kShadowSlopeScaledBias;
desc.RasterizerState.DepthBiasClamp       = kShadowDepthBiasClamp;
```

**ピクセルシェーダーを指定していません。**

深度を書くだけなら不要です。**空の `D3D12_SHADER_BYTECODE` を渡すと、ピクセルシェーダーなしの PSO になります。**

**これは大きな最適化です。** ピクセルシェーダーがない場合、GPU は深度だけを高速に書き込む専用の経路を使えます。**スループットが 2 倍程度になることもあります。**

**`NumRenderTargets = 0` も忘れないでください。** カラーを書かないので、RTV は不要です。

**シェーダーは 10 行で済みます。**

```hlsl
//=====================================================
// shaders/ShadowDepth.hlsl
//=====================================================

cbuffer ShadowConstants : register(b0)
{
    row_major float4x4 lightViewProjection;
};

cbuffer ObjectConstants : register(b1)
{
    row_major float4x4 world;
};

float4 VSMain(float3 position : POSITION) : SV_Position
{
    const float4 worldPos = mul(float4(position, 1.0f), world);
    return mul(worldPos, lightViewProjection);
}
```

**入力は位置だけです。** 法線も UV も必要ありません。**ただし入力レイアウトは頂点バッファの構造に合わせる必要があります**(使わない要素は宣言だけしておきます)。

### 27.6.2 描画

```cpp
void Renderer::RenderShadowPass(const std::vector<RenderObject*>& casters)
{
    //--- ① 状態遷移 ---
    TransitionTo(m_commandList.Get(), m_shadowMap,
                 D3D12_RESOURCE_STATE_DEPTH_WRITE);

    //--- ② レンダーターゲットは設定しない ---
    m_commandList->OMSetRenderTargets(0, nullptr, FALSE, &m_shadowDsv);

    m_commandList->ClearDepthStencilView(
        m_shadowDsv, D3D12_CLEAR_FLAG_DEPTH,
        kShadowClearValue, 0, 0, nullptr);

    //--- ③ ビューポートはシャドウマップのサイズ ---
    D3D12_VIEWPORT viewport{};
    viewport.Width    = static_cast<float>(kShadowMapSize);
    viewport.Height   = static_cast<float>(kShadowMapSize);
    viewport.MinDepth = D3D12_MIN_DEPTH;
    viewport.MaxDepth = D3D12_MAX_DEPTH;

    D3D12_RECT scissor{ 0, 0, kShadowMapSize, kShadowMapSize };

    m_commandList->RSSetViewports(1, &viewport);
    m_commandList->RSSetScissorRects(1, &scissor);

    //--- ④ 描画 ---
    m_commandList->SetGraphicsRootSignature(m_shadowRootSignature.Get());
    m_commandList->SetPipelineState(m_shadowPso.Get());
    m_commandList->SetGraphicsRootConstantBufferView(
        0, m_shadowConstantsAddress);

    m_commandList->IASetPrimitiveTopology(D3D_PRIMITIVE_TOPOLOGY_TRIANGLELIST);

    for (const RenderObject* object : casters)
    {
        const auto allocation =
            m_uploadBuffer.AllocateConstants(sizeof(Math::Matrix4x4));
        allocation.Write(object->worldMatrix);

        m_commandList->SetGraphicsRootConstantBufferView(
            1, allocation.gpuAddress);

        const auto& mesh = m_meshes[object->meshIndex];
        m_commandList->IASetVertexBuffers(0, 1, &mesh.vbv);
        m_commandList->IASetIndexBuffer(&mesh.ibv);
        m_commandList->DrawIndexedInstanced(mesh.indexCount, 1, 0, 0, 0);
    }

    //--- ⑤ 読み取り可能な状態へ ---
    TransitionTo(m_commandList.Get(), m_shadowMap,
                 D3D12_RESOURCE_STATE_PIXEL_SHADER_RESOURCE);
}
```

**`OMSetRenderTargets(0, nullptr, FALSE, &dsv)` に注目してください。** レンダーターゲットの数がゼロで、深度だけを指定しています。

**ビューポートをシャドウマップのサイズにする**のを忘れないでください。**画面サイズのままだと、シャドウマップの一部にしか描かれません。**

### 27.6.3 第 2 パス:影を参照する

**定数バッファに、ライトのビュー射影行列を追加します。**

```cpp
struct SceneConstants
{
    Math::Matrix4x4 view;
    Math::Matrix4x4 projection;
    Math::Matrix4x4 lightViewProjection;    // ← 追加
    Math::Vector4   cameraPositionWS;
    Math::Vector4   lightDirectionWS;
    Math::Vector4   lightColor;
    Math::Vector4   ambientColor;
    Math::Vector4   shadowParams;           // x = テクセルサイズ, y = バイアス
};
```

**シェーダー側です。**

```hlsl
Texture2D                gShadowMap     : register(t1);
SamplerComparisonState   gShadowSampler : register(s1);

//-----------------------------------------------------
// ワールド座標から影の判定値を求める。
//   戻り値: 1.0 = 完全に照らされている
//           0.0 = 完全に影
//-----------------------------------------------------
float ComputeShadow(float3 positionWS, float nDotL)
{
    //--- ライト空間へ変換 ---
    const float4 lightClip =
        mul(float4(positionWS, 1.0f), lightViewProjection);

    //--- 透視除算(平行投影では w = 1 だが、統一のため)---
    float3 lightNdc = lightClip.xyz / lightClip.w;

    //--- NDC [-1,1] を UV [0,1] へ ---
    float2 shadowUv;
    shadowUv.x = lightNdc.x * 0.5f + 0.5f;
    shadowUv.y = lightNdc.y * -0.5f + 0.5f;     // Y を反転

    //--- 範囲外は影なし ---
    // BORDER + 白の設定で自動的に処理されるが、
    // 深度方向は自分で判定する必要がある
#if USE_REVERSED_Z
    if (lightNdc.z <= 0.0f) return 1.0f;
#else
    if (lightNdc.z >= 1.0f) return 1.0f;
#endif

    //--- 傾きに応じたバイアス(27.4.2 節)---
    const float slopeBias = shadowParams.y * (1.0f - nDotL);

#if USE_REVERSED_Z
    const float compareDepth = lightNdc.z + slopeBias;
#else
    const float compareDepth = lightNdc.z - slopeBias;
#endif

    //--- PCF ---
    return SampleShadowPCF(shadowUv, compareDepth, shadowParams.xx);
}
```

**Y の反転が必要です。** NDC は Y の +1 が上、UV は 0 が上だからです(第20章 20.6.1 節)。

**`nDotL` に応じたバイアス**は、方法 B(27.4.2 節)の実装です。ラスタライザのバイアスと併用します。**光に対して斜めな面ほど、大きくずらします。**

**ライティングに適用します。**

```hlsl
float4 PSMain(VSOutput input) : SV_Target
{
    const float3 N = normalize(input.normalWS);
    const float3 L = -normalize(lightDirectionWS.xyz);
    const float3 V = normalize(cameraPositionWS.xyz - input.positionWS);

    const float nDotL = saturate(dot(N, L));

    //--- 影の判定 ---
    const float shadow = ComputeShadow(input.positionWS, nDotL);

    const float3 albedo =
        gDiffuseTexture.Sample(gSampler, input.uv).rgb * materialDiffuse.rgb;

    const float3 diffuse  = albedo * nDotL;
    float3 specular = ComputeSpecular(N, L, V,
                                      materialSpecular.rgb,
                                      max(materialSpecular.w, 1.0f));
    specular *= nDotL;

    //--- 直接光にだけ影を掛ける ---
    const float3 lit = (diffuse + specular)
                     * lightColor.rgb * lightDirectionWS.w
                     * shadow;                    // ← ここ

    //--- 環境光には掛けない ---
    const float3 ambient = albedo * ambientColor.rgb;

    return float4(lit + ambient, 1.0f);
}
```

**環境光に影を掛けてはいけません。**

環境光は「あらゆる方向から回り込む光」の近似です(第24章 24.1.2 節)。**直接光が遮られていても、回り込む光は届きます。**

**掛けてしまうと、影の部分が真っ黒になります。**

### 27.6.4 パイプライン全体

**第26章のパイプラインに、シャドウパスを追加します。**

```
①  シャドウマップ生成   → ShadowMap (深度のみ)
        ↓
②  シーン描画           → SceneColor (HDR)   ← ShadowMap を参照
        ↓
③  ブルーム             → Bloom[0..2]
        ↓
④  合成 + トーンマップ  → バックバッファ
```

```cpp
Core::Status Renderer::RenderFrame(const Camera& camera)
{
    // ... フレームリソースの準備 ...

    //=== ① シャドウマップ ===
    RenderShadowPass(m_shadowCasters);

    //=== ② シーン ===
    TransitionTo(m_commandList.Get(), m_sceneColor,
                 D3D12_RESOURCE_STATE_RENDER_TARGET);
    // ... 第26章 26.6.2 節と同じ ...

    //=== ③④ ポストエフェクト ===
    // ... 第26章と同じ ...
}
```

**第26章で作った構造が、そのまま拡張できました。** パスを 1 つ足すだけです。

---

## ✅ 本章のゴール:影が落ちる

### Step 1:シャドウマップを可視化する

**まず、深度が正しく描かれているかを確認します。**

```cpp
if (input.WasKeyPressed('M'))
{
    m_debugShowShadowMap = !m_debugShowShadowMap;
}
```

画面の隅にシャドウマップを表示するパスを追加します。

```hlsl
float4 PSMain(VSOutput input) : SV_Target
{
    const float depth = gShadowMap.Sample(gSampler, input.uv).r;

    //--- 値の範囲が狭いので拡大して見る ---
    const float visualized = pow(depth, 50.0f);
    return float4(visualized.xxx, 1.0f);
}
```

**物体のシルエットが見えれば成功です。**

| 症状 | 原因 |
|---|---|
| 真っ白 / 真っ黒 | クリア値または比較関数(27.3.2) |
| 一部にしか描かれていない | ビューポートが画面サイズのまま(27.6.2) |
| 何も見えない | ライト行列の範囲(27.3.3) |

### Step 2:影を有効にする

**物体の下に影が落ちます。**

**第22章のカメラで確認してください。** カメラを動かしても影の位置が変わらないこと、物体を動かせば影も動くことを確認します。

### Step 3:バイアスをゼロにする

```cpp
constexpr int   kShadowDepthBias        = 0;
constexpr float kShadowSlopeScaledBias  = 0.0f;
```

**照らされている面に、縞模様の影が現れます。**

**27.4.1 節で説明したシャドウアクネの実物です。**

**カメラの角度を変えると、模様が動きます。** これはシャドウマップのテクセルと面の交差パターンが変わるためです。

**確認したら元に戻してください。**

### Step 4:バイアスを大きくしすぎる

```cpp
constexpr int kShadowDepthBias = 50000;
```

**影が物体から離れて、浮いて見えます。**

**27.4.3 節のピーターパン現象です。**

**アクネとピーターパンの間で、ちょうどいい値を探してください。** シーンによって最適値が変わることも体感できます。

### Step 5:PCF を無効にする

```hlsl
// 1 サンプルだけ
return gShadowMap.SampleCmpLevelZero(gShadowSampler, uv, compareDepth);
```

**影の境界がギザギザになります。**

**3×3 に戻すと滑らかになります。** サンプル数を 5×5 に増やすと、さらに柔らかくなります。

### Step 6:アドレスモードを変える

```cpp
shadowSampler.AddressU = D3D12_TEXTURE_ADDRESS_MODE_CLAMP;   // ❌
shadowSampler.AddressV = D3D12_TEXTURE_ADDRESS_MODE_CLAMP;
```

**シャドウマップの範囲外に、縞模様が伸びます。**

端のテクセルの値が引き伸ばされるためです。

**`BORDER` + 白に戻すと消えます。** 第20章 20.6.3 節で予告した用途が、これで確認できました。

**確認したら元に戻してください。**

### Step 7:環境光に影を掛けてみる

```hlsl
const float3 ambient = albedo * ambientColor.rgb * shadow;   // ❌
```

**影の部分が真っ黒になります。**

**27.6.3 節で説明した理由が、実物で分かります。**

**確認したら元に戻してください。**

### Step 8:表面カリングを試す

```cpp
desc.RasterizerState.CullMode = D3D12_CULL_MODE_FRONT;
```

**バイアスをゼロにしても、アクネが出なくなります。**

**ただし、板ポリゴンがあれば影が消えます**(27.4.4 節)。シーンの内容によって使い分けてください。

### Step 9:シャドウマップの解像度を下げる

```cpp
constexpr UINT kShadowMapSize = 512;
```

**影の境界が粗くなります。**

**シーンの広さとの関係を確認してください。** 27.3.3 節で説明した「1 テクセルあたりの実距離」が、品質を決めています。

---

### 本章の達成状態

- [ ] `R32_TYPELESS` でリソースを作った
- [ ] DSV を `D32_FLOAT`、SRV を `R32_FLOAT` で作った
- [ ] 最適化クリア値を `D32_FLOAT` で指定した
- [ ] 状態遷移を実装した(`DEPTH_WRITE ↔ PIXEL_SHADER_RESOURCE`)
- [ ] 平行投影でライト視点の行列を作った
- [ ] `up` が視線と平行にならないよう対処した
- [ ] Reversed-Z との整合を取った(5 箇所)
- [ ] シーンの境界からライトの範囲を決めている
- [ ] シャドウパスでピクセルシェーダーを省略した
- [ ] `NumRenderTargets = 0` にした
- [ ] ビューポートをシャドウマップのサイズにした
- [ ] 深度バイアスを設定した
- [ ] 比較サンプラーを使っている
- [ ] `BORDER` + 白でアドレスモードを設定した
- [ ] 3×3 の PCF を実装した
- [ ] 環境光には影を掛けていない
- [ ] **影が落ちた**

---

## トラブルシューティング

| 症状 | 原因 | 対処 |
|---|---|---|
| SRV 作成に失敗 | `D32_FLOAT` で作った | `R32_TYPELESS` に(27.2.2) |
| クリア値でエラー | `TYPELESS` を指定した | `D32_FLOAT` に(27.2.2) |
| 縞模様の影 | バイアス不足 | 27.4.2 |
| 影が浮いている | バイアス過剰 | 27.4.3 |
| 影が全く出ない | ライト行列の範囲 | 27.3.3 |
| 同上 | 比較関数が逆 | Reversed-Z の整合(27.3.2) |
| シャドウマップが一部だけ | ビューポート | 27.6.2 |
| 影の境界がギザギザ | PCF なし | 27.5.3 |
| 範囲外に縞模様 | アドレスモード | `BORDER` に(27.5.2) |
| 影が真っ黒 | 環境光にも掛けた | 27.6.3 |
| 影が上下逆 | UV の Y 反転忘れ | 27.6.3 |
| カメラを動かすと影が動く | ライト行列にカメラが混入 | 27.3.1 |
| 板ポリゴンの影が消える | 表面カリング | 裏面に戻す(27.4.4) |

---

## まとめ

**1. 同じメモリを 2 通りに解釈する。**
`R32_TYPELESS` で作り、DSV は `D32_FLOAT`、SRV は `R32_FLOAT`。**第19章 19.3.2 節の予告と、第11章 11.2.1 節の「デスクリプタは解釈の指示書」が、ここで結びつきました。**

**2. アクネとピーターパンはトレードオフ。**
バイアスを小さくすれば縞模様が出て、大きくすれば影が浮きます。**`SlopeScaledDepthBias` で傾きに応じた補正をするのが基本です。**

**3. 深度だけを描くパスでは、ピクセルシェーダーを省略する。**
`NumRenderTargets = 0` と組み合わせると、GPU が高速な経路を使えます。

**4. 比較サンプラーは、比較結果をバイリニア補間する。**
1 回のサンプリングで 2×2 の平均が得られます。ハードウェアの専用回路なので高速です。

**5. `BORDER` + 白で範囲外を「影なし」にする。**
第20章 20.6.3 節で予告した用途です。`CLAMP` では縞模様が伸びます。

**6. 環境光に影を掛けない。**
環境光は回り込む光の近似なので、直接光が遮られていても届きます。

**7. Reversed-Z の影響箇所が 5 つに増えた。**
射影、クリア値、深度テスト、比較サンプラー、バイアスの符号。**定数を 1 箇所にまとめておく設計が効いています。**

次章では半透明とアンチエイリアスを扱います。**第19章 19.4.2 節で予告した `DEPTH_WRITE_MASK_ZERO`、第25章 25.5 節で触れた奥からのソート、そして第11章 11.1.3 節で「FLIP モデルは MSAA をサポートしない」と書いた問題への対処**が、そこで登場します。

---

## 参考リンク

| 内容 | URL |
|---|---|
| Common Techniques to Improve Shadow Depth Maps | https://learn.microsoft.com/ja-jp/windows/win32/dxtecharts/common-techniques-to-improve-shadow-depth-maps |
| `SampleCmpLevelZero` | https://learn.microsoft.com/ja-jp/windows/win32/direct3dhlsl/texture2d-samplecmplevelzero |
| `D3D12_STATIC_SAMPLER_DESC` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/ns-d3d12-d3d12_static_sampler_desc |
| 深度バイアスについて | https://learn.microsoft.com/ja-jp/windows/win32/direct3d11/d3d10-graphics-programming-guide-output-merger-stage-depth-bias |
| カスケードシャドウマップ | https://learn.microsoft.com/ja-jp/windows/win32/dxtecharts/cascaded-shadow-maps |
