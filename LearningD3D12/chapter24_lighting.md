# 第24章 ライティング

前章でモデルが表示されました。**しかし、平坦に見えます。**

テクスチャの模様はあるものの、立体感がありません。凹凸も、光の当たり方も、影も何もない。**そこにあるのは「色のついた形」であって、「3D の物体」ではありません。**

本章でライティングを実装します。追加するのは、シェーダー 30 行ほどと定数バッファの数フィールドだけです。**それだけで、絵は劇的に変わります。**

そして本章には、**多くの人が一度は必ず踏む 2 つの罠**が登場します。

| 罠 | 症状 |
|---|---|
| **法線変換に誤った行列を使う** | スケールを掛けると陰影が崩れる |
| **ガンマ補正を無視する** | 全体的に暗い、または白飛びする |

**どちらも「なんとなく動いている」ように見えるのが厄介です。** 第17章 17.4.4 節で `InverseAffine` を用意しておいたのは、前者への備えでした。

**本章のゴール**
Lambert と Blinn-Phong を実装し、3 種類の光源に対応する。法線変換とガンマ補正を正しく扱い、陰影のついた立体的な絵を得る。

---

## 24.1 Lambert 反射

### 24.1.1 拡散反射のモデル

**もっとも単純な照明モデルです。**

> **面が光をどれだけ受けるかは、光の向きと面の向きのなす角で決まる。**

```
    光
     ↓        ↘
  ───────   ───────
  正面から      斜めから
  = 明るい     = 暗い
```

**数式は 1 行です。**

```
diffuse = max(dot(N, L), 0)
```

| 記号 | 意味 |
|---|---|
| `N` | 面の法線(正規化済み) |
| `L` | **面から光源へ向かう**単位ベクトル |

**`L` の向きに注意してください。** 「光が進む方向」ではなく、**「光源のほうを向くベクトル」**です。逆にすると、明るいところと暗いところが反転します。

**`max(..., 0)` が必要な理由**は、内積が負になる場合(光が裏側から当たっている)を切り捨てるためです。負のまま使うと、**色が負になって描画が壊れます。**

### 24.1.2 環境光を足す

**Lambert だけでは、光が当たらない面が真っ黒になります。**

現実には、壁や床で反射した光が回り込みます。**その効果を、単純な定数で近似します。**

```
color = albedo * (ambient + diffuse * lightColor)
```

**`ambient` は物理的な根拠のない値です。** 0.1〜0.2 程度にしておくと、暗部が潰れずに済みます。

> **アンビエントオクルージョンとの違い**
>
> 定数のアンビエントは、**すべての面を等しく明るくします。** 現実には、隅や窪んだ場所は回り込む光も少なくなります。
>
> それを表現するのがアンビエントオクルージョン(AO)です。本書では扱いませんが、**第26章のポストエフェクトの応用として実装できます**(SSAO)。

---

## 24.2 Blinn-Phong 反射

### 24.2.1 鏡面反射

**Lambert は「どの方向から見ても同じ明るさ」です。** 布や紙のような、光を拡散する素材を表します。

**金属やプラスチックには、視線の方向によって変わる光沢があります。**

```
     光      視線
      ↘     ↗
   ─────●─────
        ↑ ここが光る
```

**Phong モデル**は、反射ベクトルと視線ベクトルの一致度で計算します。

```
R = reflect(-L, N)
specular = pow(max(dot(R, V), 0), shininess)
```

**Blinn-Phong モデル**は、ハーフベクトルを使います。

```
H = normalize(L + V)
specular = pow(max(dot(N, H), 0), shininess)
```

### 24.2.2 なぜ Blinn-Phong なのか

**本書は Blinn-Phong を使います。**

| | Phong | **Blinn-Phong** |
|---|---|---|
| 計算量 | `reflect` が必要 | **加算と正規化のみ** |
| 浅い角度 | ハイライトが途切れる | **自然に伸びる** |
| 業界での採用 | 少数派 | **多数派** |

**「浅い角度で途切れる」問題が決定的です。**

Phong では、視線と反射ベクトルの角度が 90 度を超えると内積が負になり、ハイライトが**唐突に消えます。** 床を浅い角度で見たときなど、実際によく起こります。

**Blinn-Phong はハーフベクトルを使うので、この問題がありません。**

> **`shininess` の値が違う**
>
> 同じ見た目を得るには、**Blinn-Phong の `shininess` を Phong の 2〜4 倍**にする必要があります。
>
> OBJ の `Ns` は Phong を前提とした値です。そのまま使うとハイライトが広がりすぎるので、**4 倍程度に変換します。**
>
> ```cpp
> const float blinnShininess = material.shininess * 4.0f;
> ```

### 24.2.3 エネルギー保存を意識する

**素朴に実装すると、明るくなりすぎます。**

```hlsl
// △ 正規化されていない
float3 color = albedo * diffuse + specularColor * specular;
```

**鏡面反射の強さは、`shininess` によって変わります。** 鋭いハイライト(大きい `shininess`)は面積が狭く、緩いハイライトは広い。**同じ係数を掛けると、後者が明るくなりすぎます。**

**正規化係数を掛けます。**

```hlsl
const float normalization = (shininess + 8.0f) / (8.0f * PI);
float3 specular = specularColor * pow(nDotH, shininess) * normalization;
```

**厳密な物理モデルではありませんが、`shininess` を変えても明るさの印象が保たれます。**

---

## 24.3 光源の種類

### 24.3.1 3 種類

| 種類 | 特徴 | 例 |
|---|---|---|
| **平行光源** | 向きだけを持つ。減衰なし | 太陽 |
| **点光源** | 位置を持つ。距離で減衰 | 電球 |
| **スポットライト** | 位置と向き。円錐状 | 懐中電灯 |

**平行光源から実装します。** 最も単純で、シーン全体の基本照明になります。

```cpp
struct DirectionalLight
{
    Math::Vector3 direction{ 0.0f, -1.0f, 0.0f };   // 光が進む方向
    Math::Vector3 color{ 1.0f, 1.0f, 1.0f };
    float         intensity = 1.0f;
};
```

**`direction` は「光が進む方向」です。** シェーダー内で `L = -direction` として使います。

**混乱しやすいので、名前で区別します。**

```hlsl
const float3 L = -normalize(lightDirection);   // 面から光源へ
```

### 24.3.2 点光源の減衰

**光の強さは距離の 2 乗に反比例します。**

```
attenuation = 1 / (distance * distance)
```

**素直に実装すると、2 つの問題が起きます。**

**問題 1:光源に近づくと発散する**

`distance` が 0 に近づくと、値が無限大になります。

**問題 2:遠くまで影響が残る**

理論上は無限遠まで届くので、**すべての光源をすべてのピクセルで計算することになります。**

**実用的な形はこうです。**

```hlsl
float ComputeAttenuation(float distance, float range)
{
    //--- 逆二乗 ---
    const float attenuation = 1.0f / max(distance * distance, 0.0001f);

    //--- 範囲の端で滑らかに 0 にする ---
    const float t = saturate(distance / range);
    const float window = 1.0f - t * t * t * t;   // (1 - t^4)

    return attenuation * window * window;
}
```

**`range` を設けることで、影響範囲が有限になります。** 第25章で複数の光源を扱うとき、**どの光源がどのオブジェクトに影響するか**を判定できるようになります。

**`(1-t⁴)²` という窓関数は、端で値と微分が両方 0 になります。** 単純に打ち切ると、境界に線が見えます。

### 24.3.3 スポットライト

**点光源に、円錐状の制限を加えたものです。**

```cpp
struct SpotLight
{
    Math::Vector3 position;
    Math::Vector3 direction;
    Math::Vector3 color;
    float intensity  = 1.0f;
    float range      = 10.0f;
    float innerAngle = Math::ToRadians(20.0f);   // 完全に明るい
    float outerAngle = Math::ToRadians(30.0f);   // ここで 0 になる
};
```

```hlsl
float ComputeSpotFactor(float3 L, float3 spotDirection,
                        float cosInner, float cosOuter)
{
    const float cosAngle = dot(-L, spotDirection);

    // cosOuter <= cosAngle <= cosInner の範囲で 0→1 に補間
    return smoothstep(cosOuter, cosInner, cosAngle);
}
```

**角度そのものではなく、コサインで比較しています。** `acos` を呼ばずに済むので高速です。

**CPU 側で `cos` を計算して渡します。**

```cpp
constants.spotCosInner = std::cos(light.innerAngle);
constants.spotCosOuter = std::cos(light.outerAngle);
```

---

## 24.4 法線変換の落とし穴

### 24.4.1 ワールド行列では変換できない

**本章で最も重要な節です。**

頂点位置は、ワールド行列で変換します。

```hlsl
float3 positionWS = mul(float4(position, 1.0f), world).xyz;
```

**法線に同じ行列を使うと、非一様スケールで壊れます。**

```
元の形状:        Y 方向に 2 倍:

   ／|            ／|
  ／ | ← 法線     ／ | ← 法線は?
 ／__|          ／  |
                ／   |
```

**面は縦に伸びますが、法線は縦に伸ばしてはいけません。** むしろ逆方向に潰れる必要があります。

**正しい変換行列は、逆転置行列です。**

```
法線用の行列 = transpose(inverse(world))
```

### 24.4.2 なぜ逆転置なのか

**法線の定義から導けます。**

法線 `N` は、面上の任意のベクトル `T` と直交します。

```
dot(N, T) = 0
```

変換後もこの関係が保たれる必要があります。`T` はワールド行列 `M` で変換されるので、

```
dot(N', T * M) = 0
```

**行ベクトル流儀(第17章 17.3.3 節)で書くと、内積は次のように表せます。**

```
(N * A) · (T * M) = N * A * Mᵀ * Tᵀ
```

これが 0 になるには、`A * Mᵀ = I` すなわち **`A = (Mᵀ)⁻¹ = (M⁻¹)ᵀ`** です。

**これが逆転置行列です。**

### 24.4.3 いつ必要か

**すべての場合に必要なわけではありません。**

| 変換の内容 | ワールド行列で足りるか |
|---|---|
| 平行移動のみ | **足りる**(w=0 なら影響しない) |
| 回転のみ | **足りる**(回転行列は直交行列) |
| 一様スケール | **足りる**(正規化すれば同じ) |
| **非一様スケール** | **足りない** |
| **せん断** | **足りない** |

**「回転と一様スケールだけなら不要」**というのは正しい判断です。多くのエンジンが、この前提で最適化しています。

**本書は逆転置行列を使います。** 理由は 2 つです。

- **第25章で任意のスケールを掛けたくなる**
- **一度正しく実装しておけば、以後考えなくて済む**

### 24.4.4 実装する

**第17章 17.4.4 節で `InverseAffine` を用意した理由が、ここで明らかになります。**

```cpp
struct ObjectConstants
{
    Math::Matrix4x4 world;
    Math::Matrix4x4 worldViewProj;
    Math::Matrix4x4 normalMatrix;      // ← 逆転置行列
};
```

```cpp
void UpdateObjectConstants(const Math::Matrix4x4& world,
                           const Camera& camera,
                           ObjectConstants& out)
{
    out.world         = world;
    out.worldViewProj = world * camera.ViewMatrix() * camera.ProjectionMatrix();

    //--- 第17章 17.4.4 節の InverseAffine を使う ---
    out.normalMatrix  = Math::Transpose(Math::InverseAffine(world));
}
```

**`InverseAffine` は、平行移動・回転・スケールのみの行列を前提としています。** 本書の用途には十分です。一般的な逆行列(余因子展開)は不要でした。

シェーダー側です。

```hlsl
VSOutput VSMain(VSInput input)
{
    VSOutput output;

    output.position   = mul(float4(input.position, 1.0f), worldViewProj);
    output.positionWS = mul(float4(input.position, 1.0f), world).xyz;

    //--- 法線は逆転置行列で。w=0 で平行移動を無効化 ---
    output.normalWS = mul(float4(input.normal, 0.0f), normalMatrix).xyz;

    output.uv = input.uv;
    return output;
}
```

**`w = 0` を忘れないでください。** 第17章 17.2.3 節で書いた通り、**方向ベクトルは平行移動の影響を受けてはいけません。**

**`w = 1` にすると、モデルを動かすたびに法線まで移動します。** 陰影が場所によって変わるという、原因の分かりにくいバグになります。

> **正規化はピクセルシェーダーで行う**
>
> 頂点シェーダーで正規化しても、**ラスタライザが補間した結果は正規化されていません**(第15章 15.6 節)。
>
> ```hlsl
> float4 PSMain(VSOutput input) : SV_Target
> {
>     const float3 N = normalize(input.normalWS);   // ← ここで正規化
>     // ...
> }
> ```
>
> **これを忘れると、面の中央付近が暗くなります。** 補間されたベクトルの長さが 1 未満になるためです。

---

## 24.5 ガンマと sRGB

### 24.5.1 ディスプレイは線形ではない

**もう 1 つの、見過ごされやすい問題です。**

ディスプレイに `0.5` という値を送っても、**輝度は最大の 50% にはなりません。** おおよそ 22% 程度です。

```
表示輝度 ≈ 入力値 ^ 2.2
```

これは CRT の物理特性に由来し、**現在も sRGB 規格として維持されています。** 人間の目が暗部の変化に敏感なため、限られたビット数を有効に使えるという利点もあります。

**問題は、光の計算が線形空間で行われることです。**

```
明るさ 2 倍の光 → 計算結果も 2 倍
```

**この 2 つが噛み合っていないと、絵が破綻します。**

### 24.5.2 何が起きるか

**ガンマを無視すると、次の症状が出ます。**

| 症状 | 原因 |
|---|---|
| 全体的に暗い | 線形の計算結果をそのまま出力している |
| **中間色が濁る** | **sRGB のテクスチャを線形として扱っている** |
| ハイライトが白飛びする | 同上 |
| 2 つの光を足すと不自然 | 非線形空間で加算している |

**とくに 2 番目が厄介です。** テクスチャは通常 sRGB 空間で保存されています(画像編集ソフトがそう出力します)。それを線形の値として計算に使うと、**明るい部分が過剰に明るくなります。**

### 24.5.3 正しい流れ

```
① テクスチャ読み込み:  sRGB → 線形へ変換
        ↓
② ライティング計算:     線形空間で行う
        ↓
③ 出力:                線形 → sRGB へ変換
```

**両端で変換し、間は線形で扱う**のが原則です。

### 24.5.4 ハードウェアに任せる

**手で `pow(2.2)` を書く必要はありません。** GPU が自動でやってくれます。

**① テクスチャ側:`_SRGB` フォーマットを使う**

```cpp
DXGI_FORMAT_R8G8B8A8_UNORM_SRGB     // ← _SRGB を付ける
DXGI_FORMAT_BC7_UNORM_SRGB
```

**サンプリング時に、自動で線形へ変換されます。** シェーダーには線形の値が届きます。

**③ レンダーターゲット側:RTV を `_SRGB` で作る**

**第11章 11.1.3 節で予告した内容が、ここで回収されます。**

> FLIP モデルのスワップチェーンは `_SRGB` 形式のフォーマットを受け付けません。バッファは `_UNORM` で作り、**RTV を `_SRGB` で作ります。** 第24章 24.4 節でこれを実装します。

```cpp
//--- スワップチェーンは _UNORM のまま ---
desc.Format = DXGI_FORMAT_R8G8B8A8_UNORM;

//--- RTV は _SRGB で作る ---
D3D12_RENDER_TARGET_VIEW_DESC rtvDesc{};
rtvDesc.Format        = DXGI_FORMAT_R8G8B8A8_UNORM_SRGB;   // ← ここ
rtvDesc.ViewDimension = D3D12_RTV_DIMENSION_TEXTURE2D;

device->CreateRenderTargetView(backBuffer, &rtvDesc, handle);
```

**第11章 11.4 節では、第 2 引数に `nullptr` を渡していました。** ここで初めて記述子を指定します。

**第11章 11.2.1 節で「デスクリプタは解釈の指示書」と書いた**ことが、ここでも効いています。同じメモリを、書き込み時に sRGB 変換するビューとして解釈しています。

**PSO も合わせます。**

```cpp
desc.RTVFormats[0] = DXGI_FORMAT_R8G8B8A8_UNORM_SRGB;   // RTV と一致させる
```

**食い違うとデバッグレイヤーがエラーを出します**(第14章 14.4.6 節)。

### 24.5.5 どのテクスチャを `_SRGB` にするか

**すべてではありません。**

| 用途 | フォーマット |
|---|---|
| **色(アルベド、ディフューズ)** | **`_SRGB`** |
| 法線マップ | `_UNORM`(**方向データであって色ではない**) |
| 粗さ、金属度 | `_UNORM` |
| 高さマップ | `_UNORM` |
| AO マップ | `_UNORM` |

**法線マップを `_SRGB` にすると、法線の向きが歪みます。** 数値データを色として解釈することになるからです。

**判断基準は単純です。「人間が見て色として認識するもの」だけが `_SRGB` です。**

### 24.5.6 クリア値にも影響する

**見落としやすい点があります。**

```cpp
const float clearColor[4] = { 0.1f, 0.2f, 0.4f, 1.0f };
m_commandList->ClearRenderTargetView(rtv, clearColor, 0, nullptr);
```

**RTV が `_SRGB` の場合、この値は線形空間として扱われ、書き込み時に sRGB へ変換されます。**

つまり、**これまでと同じ値を指定すると、明るく見えます。**

第11章から使ってきた `(0.1, 0.2, 0.4)` は、そのままだとかなり明るい青になります。**線形空間での値に直すなら、おおよそ `pow(値, 2.2)` です。**

```cpp
// sRGB で (0.1, 0.2, 0.4) 相当の色を、線形空間で指定する
const float clearColor[4] = { 0.010f, 0.033f, 0.133f, 1.0f };
```

**「sRGB を有効にしたら背景が明るくなった」のは、正常な動作です。**

---

## 24.6 実装する

### 24.6.1 定数バッファ

**第18章 18.2.3 節のルールを守ります。** `float4` と `float4x4` だけで構成します。

```cpp
struct SceneConstants
{
    //--- カメラ ---
    Math::Matrix4x4 view;
    Math::Matrix4x4 projection;
    Math::Vector4   cameraPositionWS;      // w は未使用

    //--- 平行光源 ---
    Math::Vector4   lightDirectionWS;      // xyz = 向き, w = 強度
    Math::Vector4   lightColor;            // xyz = 色, w = 未使用

    //--- 環境光 ---
    Math::Vector4   ambientColor;          // xyz = 色, w = 未使用
};
static_assert(sizeof(SceneConstants) % 16 == 0);

struct ObjectConstants
{
    Math::Matrix4x4 world;
    Math::Matrix4x4 worldViewProj;
    Math::Matrix4x4 normalMatrix;
};

struct MaterialConstants
{
    Math::Vector4 diffuse;                 // xyz = 色, w = アルファ
    Math::Vector4 specular;                // xyz = 色, w = shininess
};
```

**3 つに分けた**のには理由があります。

| 定数 | 更新頻度 |
|---|---|
| Scene | **フレームに 1 回** |
| Object | オブジェクトごと |
| Material | マテリアルごと |

**更新頻度で分けるのが定石です。** 第25章で複数オブジェクトを描くとき、シーン定数を何度も書き直さずに済みます。

**`w` に値を詰め込んでいる**のも、パディングを無駄にしない工夫です。`shininess` のために `float4` を 1 つ増やすより、`specular.w` に入れるほうが効率的です。

### 24.6.2 ルートシグネチャ

**パラメータが 3 つに増えます。**

```cpp
D3D12_ROOT_PARAMETER1 params[4]{};

//--- b0: シーン定数(頂点・ピクセル両方)---
params[0].ParameterType = D3D12_ROOT_PARAMETER_TYPE_CBV;
params[0].Descriptor.ShaderRegister = 0;
params[0].Descriptor.Flags =
    D3D12_ROOT_DESCRIPTOR_FLAG_DATA_STATIC_WHILE_SET_AT_EXECUTE;
params[0].ShaderVisibility = D3D12_SHADER_VISIBILITY_ALL;

//--- b1: オブジェクト定数 ---
params[1].ParameterType = D3D12_ROOT_PARAMETER_TYPE_CBV;
params[1].Descriptor.ShaderRegister = 1;
params[1].Descriptor.Flags =
    D3D12_ROOT_DESCRIPTOR_FLAG_DATA_STATIC_WHILE_SET_AT_EXECUTE;
params[1].ShaderVisibility = D3D12_SHADER_VISIBILITY_ALL;

//--- b2: マテリアル定数 ---
params[2].ParameterType = D3D12_ROOT_PARAMETER_TYPE_CBV;
params[2].Descriptor.ShaderRegister = 2;
params[2].Descriptor.Flags =
    D3D12_ROOT_DESCRIPTOR_FLAG_DATA_STATIC_WHILE_SET_AT_EXECUTE;
params[2].ShaderVisibility = D3D12_SHADER_VISIBILITY_PIXEL;

//--- t0: テクスチャ(テーブル)---
params[3].ParameterType = D3D12_ROOT_PARAMETER_TYPE_DESCRIPTOR_TABLE;
params[3].DescriptorTable.NumDescriptorRanges = 1;
params[3].DescriptorTable.pDescriptorRanges   = &srvRange;
params[3].ShaderVisibility = D3D12_SHADER_VISIBILITY_PIXEL;
```

**定数バッファをルートディスクリプタにしました。**

第18章 18.4.5 節のコラムで触れた通りです。**第21章のリングバッファと組み合わせると、デスクリプタの割り当ても CBV の生成も不要になります。**

```cpp
const auto allocation = m_uploadBuffer.AllocateConstants(sizeof(ObjectConstants));
allocation.Write(objectConstants);

m_commandList->SetGraphicsRootConstantBufferView(1, allocation.gpuAddress);
```

**予算の確認をしておきます**(第14章 14.1.3 節)。

```
ルートディスクリプタ × 3 = 6 DWORD
ディスクリプタテーブル × 1 = 1 DWORD
─────────────────────────
合計 7 DWORD / 64 DWORD
```

**まだ十分な余裕があります。**

### 24.6.3 シェーダー

```hlsl
//=====================================================
// shaders/Lit.hlsl
//=====================================================

static const float PI = 3.14159265359f;

cbuffer SceneConstants : register(b0)
{
    row_major float4x4 view;
    row_major float4x4 projection;
    float4 cameraPositionWS;
    float4 lightDirectionWS;      // xyz = 進む向き, w = 強度
    float4 lightColor;
    float4 ambientColor;
};

cbuffer ObjectConstants : register(b1)
{
    row_major float4x4 world;
    row_major float4x4 worldViewProj;
    row_major float4x4 normalMatrix;    // 逆転置行列(24.4 節)
};

cbuffer MaterialConstants : register(b2)
{
    float4 materialDiffuse;       // xyz = 色, w = アルファ
    float4 materialSpecular;      // xyz = 色, w = shininess
};

Texture2D    gDiffuseTexture : register(t0);
SamplerState gSampler        : register(s0);

struct VSInput
{
    float3 position : POSITION;
    float3 normal   : NORMAL;
    float4 tangent  : TANGENT;
    float2 uv       : TEXCOORD;
};

struct VSOutput
{
    float4 position   : SV_Position;
    float3 positionWS : POSITION_WS;
    float3 normalWS   : NORMAL_WS;
    float2 uv         : TEXCOORD;
};

//-----------------------------------------------------
// 頂点シェーダー
//-----------------------------------------------------
VSOutput VSMain(VSInput input)
{
    VSOutput output;

    output.position   = mul(float4(input.position, 1.0f), worldViewProj);
    output.positionWS = mul(float4(input.position, 1.0f), world).xyz;

    // 法線は逆転置行列で変換する。w = 0(24.4.4 節)
    output.normalWS   = mul(float4(input.normal, 0.0f), normalMatrix).xyz;

    output.uv = input.uv;
    return output;
}

//-----------------------------------------------------
// Blinn-Phong の鏡面反射
//-----------------------------------------------------
float3 ComputeSpecular(float3 N, float3 L, float3 V,
                       float3 specularColor, float shininess)
{
    const float3 H = normalize(L + V);
    const float nDotH = saturate(dot(N, H));

    // エネルギー保存のための正規化係数(24.2.3 節)
    const float normalization = (shininess + 8.0f) / (8.0f * PI);

    return specularColor * pow(nDotH, shininess) * normalization;
}

//-----------------------------------------------------
// ピクセルシェーダー
//-----------------------------------------------------
float4 PSMain(VSOutput input) : SV_Target
{
    //--- 補間された法線を正規化する(24.4.4 節のコラム)---
    const float3 N = normalize(input.normalWS);

    //--- 面から光源へ向かうベクトル ---
    const float3 L = -normalize(lightDirectionWS.xyz);

    //--- 面から視点へ向かうベクトル ---
    const float3 V = normalize(cameraPositionWS.xyz - input.positionWS);

    //--- アルベド(_SRGB のテクスチャなので線形で届く。24.5 節)---
    const float4 texColor = gDiffuseTexture.Sample(gSampler, input.uv);
    const float3 albedo   = texColor.rgb * materialDiffuse.rgb;

    //--- Lambert ---
    const float nDotL = saturate(dot(N, L));
    const float3 diffuse = albedo * nDotL;

    //--- Blinn-Phong ---
    const float shininess = max(materialSpecular.w, 1.0f);
    float3 specular = ComputeSpecular(
        N, L, V, materialSpecular.rgb, shininess);

    // 光が当たっていない面ではハイライトも出さない
    specular *= nDotL;

    //--- 合成 ---
    const float3 lit = (diffuse + specular)
                     * lightColor.rgb * lightDirectionWS.w;

    const float3 ambient = albedo * ambientColor.rgb;

    return float4(lit + ambient, texColor.a * materialDiffuse.a);
}
```

**`specular *= nDotL` を入れている**のが要点です。

これがないと、**光が当たっていない裏面にハイライトが出ます。** ハーフベクトルは視線と光の中間なので、面の向きに関係なく計算されてしまうためです。

---

## ✅ 本章のゴール:陰影がつき、立体的に見える

### Step 1:実行する

**前章と同じモデルが、立体的に見えます。**

- 光の当たる面が明るく、反対側が暗い
- 曲面に滑らかな階調がある
- 光沢のある部分にハイライトが出る

**第22章のカメラで回してみてください。** ハイライトが視点に追従して動きます。**これが Blinn-Phong の効果です。**

### Step 2:法線行列を壊す

**24.4 節の内容を確かめます。**

```cpp
out.normalMatrix = world;   // ❌ 逆転置ではなくワールド行列
```

**一様スケールなら、見た目は変わりません。** 正しく見えます。

**非一様スケールを掛けてください。**

```cpp
const auto world = Math::Scaling(1.0f, 3.0f, 1.0f);
```

**陰影が明らかに崩れます。** 縦に伸びた面の明るさが、本来と違う値になります。

**逆転置行列に戻すと、正しくなります。**

**この症状は「なんとなくおかしい」程度にしか見えないことがあります。** だからこそ、最初から正しく実装しておく価値があります。

### Step 3:`w = 1` にしてみる

```hlsl
output.normalWS = mul(float4(input.normal, 1.0f), normalMatrix).xyz;   // ❌
```

**モデルを原点から離すと、陰影が壊れます。**

法線に平行移動が加わるためです。**第17章 17.2.3 節で「`w=0` は方向、`w=1` は点」と書いた理由が、ここで実感できます。**

**確認したら元に戻してください。**

### Step 4:正規化を忘れてみる

```hlsl
const float3 N = input.normalWS;   // ❌ normalize しない
```

**面の中央付近が暗くなります。**

補間されたベクトルの長さが 1 未満になるためです。**曲面で顕著に現れます。**

**確認したら元に戻してください。**

### Step 5:sRGB を有効にする

**RTV とテクスチャを `_SRGB` に切り替えます。**

**変化を観察してください。**

| | sRGB なし | **sRGB あり** |
|---|---|---|
| 全体の明るさ | 暗い | **適切** |
| 中間調 | 潰れている | **豊か** |
| ハイライト | 白飛びしやすい | **自然** |
| 暗部 | 黒く潰れる | **階調が残る** |

**背景のクリア色が明るくなります。** 24.5.6 節の通り、これは正常です。

**切り替えて見比べると、違いは歴然としています。** 「なんとなく絵がきれいになった」ではなく、**中間調の情報量が明確に増えます。**

### Step 6:法線マップを `_SRGB` にしてみる(任意)

法線マップを使っている場合のみ試せます。

```cpp
DXGI_FORMAT_BC5_UNORM  →  ❌ BC7_UNORM_SRGB に変更
```

**陰影が不自然に歪みます。** 方向データを色として変換したためです。

**24.5.5 節の判断基準が、実物で確認できます。**

### Step 7:Phong と比較する(任意)

```hlsl
// Phong 版
const float3 R = reflect(-L, N);
const float rDotV = saturate(dot(R, V));
float3 specular = specularColor * pow(rDotV, shininess);
```

**床を浅い角度から見てください。**

**Phong ではハイライトが唐突に途切れます。** Blinn-Phong では自然に伸びます。

**24.2.2 節で述べた差が、実際に確認できます。**

---

### 本章の達成状態

- [ ] Lambert 反射を実装した
- [ ] 環境光を加えた
- [ ] Blinn-Phong の鏡面反射を実装した
- [ ] エネルギー保存の正規化係数を入れた
- [ ] `specular *= nDotL` を入れた
- [ ] OBJ の `Ns` を 4 倍に変換した
- [ ] **法線を逆転置行列で変換している**
- [ ] `w = 0` で変換している
- [ ] ピクセルシェーダーで正規化している
- [ ] 定数バッファを更新頻度で 3 つに分けた
- [ ] ルートディスクリプタで定数を渡している
- [ ] **色テクスチャを `_SRGB` にした**
- [ ] **RTV を `_SRGB` で作った**
- [ ] 法線マップは `_UNORM` のまま
- [ ] PSO の `RTVFormats` を RTV と一致させた
- [ ] **陰影のついた立体的な絵になった**

---

## トラブルシューティング

| 症状 | 原因 | 対処 |
|---|---|---|
| 明暗が反転する | `L` の向きが逆 | `L = -direction`(24.1.1) |
| 裏面にハイライト | `nDotL` を掛けていない | 24.6.3 |
| スケールで陰影が崩れる | ワールド行列で法線を変換 | **逆転置行列**(24.4) |
| モデルを動かすと陰影が変わる | `w = 1` で変換 | `w = 0` に(24.4.4) |
| 面の中央が暗い | 補間後に正規化していない | 24.4.4 のコラム |
| 全体が暗い | sRGB の RTV がない | 24.5.4 |
| 中間色が濁る | テクスチャが `_UNORM` | `_SRGB` に(24.5.5) |
| 法線マップが歪む | `_SRGB` にした | `_UNORM` に戻す(24.5.5) |
| 背景が明るくなった | **正常**(線形空間) | クリア値を調整(24.5.6) |
| PSO 生成でエラー | RTV とフォーマット不一致 | 24.5.4 |
| ハイライトが広すぎる | `Ns` をそのまま使った | 4 倍する(24.2.2) |
| 浅い角度でハイライトが消える | Phong を使っている | Blinn-Phong に(24.2.2) |
| 明るくなりすぎる | 正規化係数がない | 24.2.3 |

---

## まとめ

**1. Lambert は `dot(N, L)` だけ。**
`L` は「面から光源へ向かう」ベクトルです。逆にすると明暗が反転します。

**2. Blinn-Phong はハーフベクトルを使う。**
Phong より速く、浅い角度でも自然です。**ただし `shininess` は 2〜4 倍にする必要があります。**

**3. 法線はワールド行列では変換できない。**
非一様スケールで崩れます。**逆転置行列を使ってください。** 第17章 17.4.4 節の `InverseAffine` は、このために用意しました。

**4. `w = 0` で変換する。**
方向ベクトルは平行移動の影響を受けません。**`w = 1` にすると、モデルを動かすたびに陰影が変わります。**

**5. 補間された法線は正規化されていない。**
ピクセルシェーダーで `normalize` してください。忘れると面の中央が暗くなります。

**6. ガンマは両端で変換し、間は線形で扱う。**
テクスチャは `_SRGB`、RTV も `_SRGB`。**ハードウェアが自動でやってくれるので、`pow` を書く必要はありません。**

**7. 法線マップは `_SRGB` にしない。**
「人間が色として見るもの」だけが `_SRGB` です。方向データを変換すると歪みます。

**8. 第11章の予告が回収された。**
「バッファは `_UNORM` で作り、RTV を `_SRGB` で作る」。**デスクリプタが解釈の指示書であることの、明快な実例です。**

次章では、複数のオブジェクトを描きます。定数バッファをオブジェクトごとに持たせ、ドローコールを減らす工夫を考え、最小限のシーングラフを作ります。**第21章で作ったリングバッファが、ようやく本領を発揮します。**

---

## 参考リンク

| 内容 | URL |
|---|---|
| ガンマ補正について | https://learn.microsoft.com/ja-jp/windows/win32/direct3ddxgi/converting-data-color-space |
| sRGB フォーマット | https://learn.microsoft.com/ja-jp/windows/win32/api/dxgiformat/ne-dxgiformat-dxgi_format |
| Blinn-Phong モデル | https://en.wikipedia.org/wiki/Blinn%E2%80%93Phong_reflection_model |
| 法線変換の数学 | https://www.scratchapixel.com/lessons/mathematics-physics-for-computer-graphics/geometry/transforming-normals.html |
| 物理ベースの光の減衰 | https://google.github.io/filament/Filament.html#lighting/directlighting |
