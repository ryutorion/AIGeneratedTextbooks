# 第14章 ルートシグネチャとPSO

シェーダーができました。しかし、**シェーダーだけでは絵は出ません。**

GPU に「描け」と言うには、シェーダー以外にも大量の情報が要ります。三角形を塗りつぶすのか線で描くのか。裏向きの面を捨てるのか。深度テストをするのか。出力先のフォーマットは何か。頂点データはどう並んでいるのか。

Direct3D 11 では、これらを個別の API で設定していました。`RSSetState`、`OMSetBlendState`、`IASetInputLayout` —— **描画のたびに、ばらばらの設定を積み上げていく**方式です。

Direct3D 12 は、これを**一つの塊に固めます**。それがパイプラインステートオブジェクト(PSO)です。

そして本章は、**本書でもっとも大きな構造体**と対面する章でもあります。`D3D12_GRAPHICS_PIPELINE_STATE_DESC` はフィールドが 22 個あり、そのうち 4 つは入れ子の構造体です。`d3dx12.h` の `CD3DX12_*` ヘルパーは使えません。

第9章 9.3.3 節で「D3D12 の構造体は必ず `{}` を付けて宣言する」と決めました。**本章で、その習慣だけでは足りないことが分かります。** ゼロで埋めると壊れるフィールドがあるからです。しかも、**エラーを出さずに壊れます。**

**本章のゴール**
空のルートシグネチャと、三角形を描くための PSO を生成する。あわせて、既定値を持つ自作関数を用意し、以後のすべての PSO の土台とする。

---

## 14.1 ルートシグネチャ = シェーダーの「引数の型宣言」

### 14.1.1 何を宣言するのか

C++ の関数を考えてみてください。

```cpp
float4 PixelShader(Texture2D tex, SamplerState smp, float4x4 matrix);
```

呼び出す側は、この宣言を見て「テクスチャと、サンプラーと、行列を渡せばいい」と分かります。

**ルートシグネチャは、これのシェーダー版です。**

```
ルートシグネチャ = 「このパイプラインは、どんなリソースを受け取るか」の宣言
```

宣言するのは**型と個数と並び**であって、実際のリソースではありません。実際に何を渡すかは、描画のたびにコマンドリストで指定します(第18章)。

**この分離が D3D12 の設計の核心です。** 宣言は PSO と一緒に固めておき、実データだけを毎フレーム差し替えます。ドライバは事前に最適化でき、描画時のオーバーヘッドが下がります。

### 14.1.2 3 種類のルートパラメータ

シェーダーに渡せるものは、大きく 3 種類あります。**詳細は第18章で扱いますが、全体像だけ先に示します。**

| 種類 | 何を置くか | 間接参照 | コスト |
|---|---|---|---|
| **ルート定数** | 32bit の値を直接 | なし | **最速** |
| **ルートディスクリプタ** | リソースの GPU アドレス | 1 段 | 速い |
| **ディスクリプタテーブル** | デスクリプタヒープ上の範囲 | 2 段 | 柔軟 |

```
ルート定数:
  [ルートシグネチャ] ──► 値そのもの

ルートディスクリプタ:
  [ルートシグネチャ] ──► リソース

ディスクリプタテーブル:
  [ルートシグネチャ] ──► デスクリプタヒープ ──► リソース
```

**下に行くほど柔軟で、上に行くほど速い**という関係です。第18章でこの使い分けを設計します。

加えて、**静的サンプラー**という枠があります。サンプラー(テクスチャの読み取り方)は種類が少なく、実行時に変わらないことが多いため、ルートシグネチャに直接埋め込めます。第20章で使います。

### 14.1.3 64 DWORD の予算

**ルートシグネチャには、厳しいサイズ制限があります。**

```
合計 64 DWORD(= 256 バイト)まで
```

| 種類 | 消費する DWORD 数 |
|---|---|
| ルート定数 | **1 つにつき 1** |
| ルートディスクリプタ | **1 つにつき 2** |
| ディスクリプタテーブル | **1 つにつき 1** |
| 静的サンプラー | **0**(別枠) |

たとえば `float4x4` の行列をルート定数で渡すと、16 個の 32bit 値なので **16 DWORD** を消費します。予算の 4 分の 1 です。

**この制限が、リソースの渡し方を設計する際の主要な制約になります。** ディスクリプタテーブルが 1 DWORD で済むのは、「ヒープ上の位置」だけを持つからです。だからテーブル経由が基本になり、第33章のバインドレスは**テーブルすら使わない**ところまで進みます。

### 14.1.4 本章では空のものを作る

第15章で描く三角形は、頂点バッファだけを使います。**定数バッファもテクスチャも使いません。**

したがって、**ルートパラメータはゼロ**です。

「それならルートシグネチャは要らないのでは」と思うかもしれませんが、**必須です。** PSO は必ずルートシグネチャを持たなければなりません。空でも構いませんが、`nullptr` は許されません。

ただし、**フラグだけは正しく設定する必要があります**(14.2.3 節)。ここが本節最大の落とし穴です。

---

## 14.2 ルートシグネチャを作る

### 14.2.1 `D3D12_ROOT_SIGNATURE_DESC`

まず 1.0 版の構造体を見ます。

```cpp
typedef struct D3D12_ROOT_SIGNATURE_DESC {
    UINT                            NumParameters;
    const D3D12_ROOT_PARAMETER*     pParameters;
    UINT                            NumStaticSamplers;
    const D3D12_STATIC_SAMPLER_DESC* pStaticSamplers;
    D3D12_ROOT_SIGNATURE_FLAGS      Flags;
} D3D12_ROOT_SIGNATURE_DESC;
```

**5 つだけです。** 「パラメータの配列」「静的サンプラーの配列」「フラグ」という素直な構成です。

### 14.2.2 バージョン 1.1 を使う

ルートシグネチャには 1.0 と 1.1 があります。

**1.1 で追加されたのは、「揮発性」の宣言です。**

```
このディスクリプタは、コマンドリストの記録後に変更されない
このデータは、実行中に変わらない
```

こうした情報をドライバに伝えると、**より積極的な最適化が可能になります。** たとえば、変わらないと分かっているデータをレジスタに載せておく、といったことです。

**パラメータがゼロの本章では、1.0 と 1.1 に違いはありません。** それでも 1.1 で書くのは、**第18章で書き直したくないから**です。

- 1.0 → `D3D12_ROOT_SIGNATURE_DESC` / `D3D12_ROOT_PARAMETER`
- 1.1 → `D3D12_VERSIONED_ROOT_SIGNATURE_DESC` / `D3D12_ROOT_PARAMETER1`

**構造体の型が違うので、後から切り替えると全部書き直しになります。**

対応状況は `CheckFeatureSupport` で問い合わせます。第7章 7.5.1 節で作った `QueryFeature` を使います。

```cpp
D3D12_FEATURE_DATA_ROOT_SIGNATURE data{};
data.HighestVersion = D3D_ROOT_SIGNATURE_VERSION_1_1;

if (!QueryFeature(device, D3D12_FEATURE_ROOT_SIGNATURE, data))
{
    data.HighestVersion = D3D_ROOT_SIGNATURE_VERSION_1_0;
}
```

**この API も `HighestVersion` が入力兼出力です。** 第13章 13.1.2 節のシェーダーモデル問い合わせと同じ作りです(第7章 7.5.2 節)。

問い合わせた結果は `DeviceCaps`(第7章 7.5.5 節)に加えておきます。

```cpp
struct DeviceCaps
{
    // ...
    D3D_ROOT_SIGNATURE_VERSION rootSignatureVersion
        = D3D_ROOT_SIGNATURE_VERSION_1_0;
};
```

**Turing 世代以降の環境なら、まず 1.1 が返ります。**

### 14.2.3 フラグ —— 最大の落とし穴

```cpp
typedef enum D3D12_ROOT_SIGNATURE_FLAGS {
    D3D12_ROOT_SIGNATURE_FLAG_NONE                               = 0,
    D3D12_ROOT_SIGNATURE_FLAG_ALLOW_INPUT_ASSEMBLER_INPUT_LAYOUT = 0x1,
    D3D12_ROOT_SIGNATURE_FLAG_DENY_VERTEX_SHADER_ROOT_ACCESS     = 0x2,
    D3D12_ROOT_SIGNATURE_FLAG_DENY_HULL_SHADER_ROOT_ACCESS       = 0x4,
    D3D12_ROOT_SIGNATURE_FLAG_DENY_DOMAIN_SHADER_ROOT_ACCESS     = 0x8,
    D3D12_ROOT_SIGNATURE_FLAG_DENY_GEOMETRY_SHADER_ROOT_ACCESS   = 0x10,
    D3D12_ROOT_SIGNATURE_FLAG_DENY_PIXEL_SHADER_ROOT_ACCESS      = 0x20,
    // ... 以下、ストリーム出力やレイトレーシング用 ...
} D3D12_ROOT_SIGNATURE_FLAGS;
```

#### `ALLOW_INPUT_ASSEMBLER_INPUT_LAYOUT` は必須

**頂点バッファを使うなら、このフラグが必要です。**

付け忘れると、ルートシグネチャの生成は成功します。**失敗するのは PSO の生成です。**

```
D3D12 ERROR: ID3D12Device::CreateGraphicsPipelineState:
  The Root Signature does not allow the Input Assembler,
  but the Pipeline State Object contains an Input Layout.
```

**症状と原因が離れているタイプのバグ**です。デバッグレイヤーが明確に教えてくれるのが救いですが、これがなければ延々と悩むところです(第7章の設定が効いています)。

なぜこんなフラグがあるのか。**入力アセンブラを使わない描画方式が存在するから**です。第15章 15.5 節で触れますが、頂点バッファなしで `SV_VertexID` だけを使って三角形を描く方法があり、そのときこのフラグは不要です。使わないと宣言すれば、ドライバは少し最適化できます。

#### `DENY_*` は積極的に付ける

「このステージからはルートシグネチャにアクセスしない」という宣言です。**付けるほどドライバの最適化余地が増えます。**

本章のシェーダーは頂点とピクセルだけなので、残り 3 つを拒否できます。

```cpp
desc.Flags =
      D3D12_ROOT_SIGNATURE_FLAG_ALLOW_INPUT_ASSEMBLER_INPUT_LAYOUT
    | D3D12_ROOT_SIGNATURE_FLAG_DENY_HULL_SHADER_ROOT_ACCESS
    | D3D12_ROOT_SIGNATURE_FLAG_DENY_DOMAIN_SHADER_ROOT_ACCESS
    | D3D12_ROOT_SIGNATURE_FLAG_DENY_GEOMETRY_SHADER_ROOT_ACCESS;
```

**ただし、拒否したステージからアクセスするとエラーになります。** 第36章でメッシュシェーダーを使うときは、対応する `DENY` を外す必要があります。

### 14.2.4 シリアライズとエラー Blob

ルートシグネチャの生成は、**2 段階**です。

```
① D3D12SerializeVersionedRootSignature  記述子 → バイナリ表現
                ↓
② ID3D12Device::CreateRootSignature     バイナリ表現 → オブジェクト
```

なぜ 2 段階なのか。**バイナリ表現をファイルに保存できるから**です。ビルド時にシリアライズしておき、実行時は読み込むだけ、という運用が可能になります。第13章でシェーダーをビルド時にコンパイルしたのと同じ発想です。

**本書は実行時にシリアライズします。** ルートシグネチャは小さく、生成コストも無視できるためです。

```cpp
ComPtr<ID3DBlob> blob;
ComPtr<ID3DBlob> errorBlob;

const HRESULT hr = ::D3D12SerializeVersionedRootSignature(
    &versionedDesc, &blob, &errorBlob);
```

**`errorBlob` を無視しないでください。**

失敗したとき、**なぜ失敗したかは `errorBlob` の中にしか書かれていません。** `HRESULT` は `E_INVALIDARG` としか言いません。

```cpp
if (FAILED(hr))
{
    if (errorBlob)
    {
        // ナロー文字列で返る。第6章の ToWide を使う
        const std::string_view message(
            static_cast<const char*>(errorBlob->GetBufferPointer()),
            errorBlob->GetBufferSize());

        LOG_ERROR(L"root signature: {}", Core::ToWide(message));
    }
    return std::unexpected(Core::MakeError(
        hr, L"D3D12SerializeVersionedRootSignature"));
}
```

**D3D 周辺の文字列がナローで返る例が、また一つ増えました。** デバッグレイヤーのメッセージ(第7章)、DXC のエラー(第13章)、そしてこれです。第6章 6.3.4 節で `ToWide` を作っておいた甲斐があります。

### 14.2.5 実装

```cpp
// src/Graphics/RootSignature.cpp(抜粋)

Core::Result<ComPtr<ID3D12RootSignature>> CreateEmptyRootSignature(
    ID3D12Device* device,
    D3D_ROOT_SIGNATURE_VERSION version,
    std::wstring_view name)
{
    //--- 記述子を組み立てる ---
    D3D12_VERSIONED_ROOT_SIGNATURE_DESC versioned{};
    versioned.Version = version;

    if (version >= D3D_ROOT_SIGNATURE_VERSION_1_1)
    {
        versioned.Desc_1_1.NumParameters     = 0;
        versioned.Desc_1_1.pParameters       = nullptr;
        versioned.Desc_1_1.NumStaticSamplers = 0;
        versioned.Desc_1_1.pStaticSamplers   = nullptr;
        versioned.Desc_1_1.Flags =
              D3D12_ROOT_SIGNATURE_FLAG_ALLOW_INPUT_ASSEMBLER_INPUT_LAYOUT
            | D3D12_ROOT_SIGNATURE_FLAG_DENY_HULL_SHADER_ROOT_ACCESS
            | D3D12_ROOT_SIGNATURE_FLAG_DENY_DOMAIN_SHADER_ROOT_ACCESS
            | D3D12_ROOT_SIGNATURE_FLAG_DENY_GEOMETRY_SHADER_ROOT_ACCESS;
    }
    else
    {
        versioned.Desc_1_0.NumParameters     = 0;
        versioned.Desc_1_0.pParameters       = nullptr;
        versioned.Desc_1_0.NumStaticSamplers = 0;
        versioned.Desc_1_0.pStaticSamplers   = nullptr;
        versioned.Desc_1_0.Flags =
              D3D12_ROOT_SIGNATURE_FLAG_ALLOW_INPUT_ASSEMBLER_INPUT_LAYOUT
            | D3D12_ROOT_SIGNATURE_FLAG_DENY_HULL_SHADER_ROOT_ACCESS
            | D3D12_ROOT_SIGNATURE_FLAG_DENY_DOMAIN_SHADER_ROOT_ACCESS
            | D3D12_ROOT_SIGNATURE_FLAG_DENY_GEOMETRY_SHADER_ROOT_ACCESS;
    }

    //--- ① シリアライズ ---
    ComPtr<ID3DBlob> blob;
    ComPtr<ID3DBlob> errorBlob;

    const HRESULT hr = ::D3D12SerializeVersionedRootSignature(
        &versioned, &blob, &errorBlob);

    if (FAILED(hr))
    {
        if (errorBlob)
        {
            LOG_ERROR(L"root signature: {}", Core::ToWide(
                std::string_view(
                    static_cast<const char*>(errorBlob->GetBufferPointer()),
                    errorBlob->GetBufferSize())));
        }
        return std::unexpected(Core::MakeError(
            hr, L"D3D12SerializeVersionedRootSignature"));
    }

    //--- ② オブジェクト生成 ---
    ComPtr<ID3D12RootSignature> rootSignature;
    HR_TRY(device->CreateRootSignature(
        0,                            // NodeMask
        blob->GetBufferPointer(),
        blob->GetBufferSize(),
        IID_PPV_ARGS(&rootSignature)));

    Core::SetDebugName(rootSignature.Get(), name);
    LOG_INFO(L"root signature created: {} ({} bytes)",
             name, blob->GetBufferSize());

    return rootSignature;
}
```

---

## 14.3 HLSL 側に書く方法との比較

**ルートシグネチャは、HLSL の中に書くこともできます。**

```hlsl
#define TriangleRS \
    "RootFlags(ALLOW_INPUT_ASSEMBLER_INPUT_LAYOUT | " \
    "          DENY_HULL_SHADER_ROOT_ACCESS | " \
    "          DENY_DOMAIN_SHADER_ROOT_ACCESS | " \
    "          DENY_GEOMETRY_SHADER_ROOT_ACCESS)"

[RootSignature(TriangleRS)]
VSOutput VSMain(VSInput input) { ... }
```

こう書くと、**コンパイル済みシェーダーの中にルートシグネチャが埋め込まれます。** C++ 側では、シェーダーバイナリをそのまま `CreateRootSignature` に渡せます。

```cpp
device->CreateRootSignature(0, vs.Bytecode().pShaderBytecode,
                            vs.Bytecode().BytecodeLength,
                            IID_PPV_ARGS(&rootSignature));
```

**この方式には明確な利点があります。**

| | HLSL に書く | **C++ に書く(本書)** |
|---|---|---|
| シェーダーとの不一致 | **起こりえない** | 起こりうる |
| 記述の量 | 少ない | 多い |
| 複数シェーダーでの共有 | しにくい | **しやすい** |
| 実行時に組み立てる | できない | **できる** |
| 型チェック | 文字列なのでなし | 構造体なのであり |

**「不一致が起こりえない」は大きな利点です。** シェーダーが `t0` を読んでいるのにルートシグネチャに宣言がない、という食い違いは、実際によく起きるバグです。

それでも本書が C++ 側を選ぶ理由は 3 つです。

**1. 構造体を見せたい**
第1章 1.3.1 節の方針です。文字列で書くと、`D3D12_ROOT_PARAMETER1` の中身を知らないまま進めてしまいます。

**2. 実行時に組み立てる必要が出てくる**
第33章のバインドレスでは、対応状況に応じてルートシグネチャの構成を変えます。文字列では書けません。

**3. 共有したい**
第25章で複数のオブジェクトを描くようになると、同じルートシグネチャを複数の PSO で使い回します。

**どちらが正解ということはありません。** 小規模なプロジェクトや、シェーダーごとに専用のルートシグネチャを持つ設計なら、HLSL に書くほうが安全で簡潔です。

---

## 14.4 パイプラインステートオブジェクト

### 14.4.1 PSO とは何か

**描画に必要な設定を、まとめて一つのオブジェクトに固めたもの**です。

```
PSO = ルートシグネチャ
    + 頂点シェーダー + ピクセルシェーダー
    + 入力レイアウト
    + ラスタライザの設定
    + ブレンドの設定
    + 深度ステンシルの設定
    + 出力先のフォーマット
    + プリミティブの種類
```

生成すると、**ドライバがこれらを一括で検証し、最適化し、GPU の実行形式に変換します。** 第13章 13.2.2 節で「DXIL は実行時にドライバが機械語へ変換する」と書きました。**その変換が起きるのが、この瞬間です。**

**だから PSO の生成は遅いです。** 1 つあたり数ミリ秒かかることも珍しくありません。

代わりに、描画時のコストは劇的に下がります。

```cpp
commandList->SetPipelineState(pso.Get());   // これだけ
```

D3D11 のように 10 個の API を呼ぶ必要はありません。**「事前に固める」ことで「本番を速くする」** —— D3D12 全体を貫く思想が、ここに最も明確に現れています。

> **PSO のキャッシュについて**
>
> `D3D12_CACHED_PIPELINE_STATE` を使うと、一度生成した PSO の結果をバイト列として取り出し、次回の起動時に再利用できます。起動時間を短縮する定番の手法です。
>
> ただし、**ドライバのバージョンが変わるとキャッシュは無効になります。** 検証と再生成の仕組みが必要で、それなりの実装量になります。本書では扱いません。

### 14.4.2 `D3D12_GRAPHICS_PIPELINE_STATE_DESC` の全フィールド

**本書で最大の構造体です。**

```cpp
typedef struct D3D12_GRAPHICS_PIPELINE_STATE_DESC {
    ID3D12RootSignature*               pRootSignature;
    D3D12_SHADER_BYTECODE              VS;
    D3D12_SHADER_BYTECODE              PS;
    D3D12_SHADER_BYTECODE              DS;
    D3D12_SHADER_BYTECODE              HS;
    D3D12_SHADER_BYTECODE              GS;
    D3D12_STREAM_OUTPUT_DESC           StreamOutput;
    D3D12_BLEND_DESC                   BlendState;
    UINT                               SampleMask;
    D3D12_RASTERIZER_DESC              RasterizerState;
    D3D12_DEPTH_STENCIL_DESC           DepthStencilState;
    D3D12_INPUT_LAYOUT_DESC            InputLayout;
    D3D12_INDEX_BUFFER_STRIP_CUT_VALUE IBStripCutValue;
    D3D12_PRIMITIVE_TOPOLOGY_TYPE      PrimitiveTopologyType;
    UINT                               NumRenderTargets;
    DXGI_FORMAT                        RTVFormats[8];
    DXGI_FORMAT                        DSVFormat;
    DXGI_SAMPLE_DESC                   SampleDesc;
    UINT                               NodeMask;
    D3D12_CACHED_PIPELINE_STATE        CachedPSO;
    D3D12_PIPELINE_STATE_FLAGS         Flags;
} D3D12_GRAPHICS_PIPELINE_STATE_DESC;
```

**22 フィールド。** うち 4 つは、それ自体が構造体です。

本章で実際に触るのは、このうち 7 つだけです。**残りの 15 個は既定値のままにします。** そして —— **その「既定値」が問題です。**

### 14.4.3 `{}` では足りない

第9章 9.3.3 節で「必ず `{}` を付けろ」と決めました。**しかしゼロ初期化では動かないフィールドがあります。**

しかも、種類が 2 つに分かれます。

| 種類 | 例 | 見つけやすさ |
|---|---|---|
| **ゼロが不正な値** | `RasterizerState.FillMode` | エラーになるので**気づく** |
| **ゼロが「無効」を意味する有効な値** | `SampleMask` | **黙って何も描かれない** |

**後者が本当に危険です。** 順に見ます。

#### 罠 1:`SampleMask = 0`

```cpp
UINT SampleMask;   // {} で 0 になる
```

**サンプルマスクは、ラスタライズするサンプルをビットで指定します。** `0` は「どのサンプルも描かない」という意味です。

つまり、**すべてのピクセルが捨てられます。**

```
PSO の生成は成功する
デバッグレイヤーは何も言わない
描画コマンドも成功する
そして、何も描かれない
```

**Direct3D 12 初心者が最初にはまる罠として、これは有名です。** 正しい値は「全部のサンプルを使う」を意味する `UINT_MAX` です。

```cpp
desc.SampleMask = UINT_MAX;   // 0xFFFFFFFF
```

`d3dx12.h` の `CD3DX12_DEFAULT` は、これを `UINT_MAX` に設定します。**ヘルパーを使わない我々は、自分で知っている必要があります。**

#### 罠 2:`RenderTargetWriteMask = 0`

```cpp
D3D12_BLEND_DESC BlendState;
// → RenderTarget[0].RenderTargetWriteMask が 0 になる
```

**書き込むカラーチャンネルをビットで指定します。** `0` は「どのチャンネルにも書かない」です。

これも**黙って何も描かれません。**

```cpp
desc.BlendState.RenderTarget[0].RenderTargetWriteMask =
    D3D12_COLOR_WRITE_ENABLE_ALL;   // = 15 (RGBA すべて)
```

#### 罠 3:`SampleDesc.Count = 0`

```cpp
DXGI_SAMPLE_DESC SampleDesc;   // Count が 0 になる
```

**`0` は不正です。** MSAA を使わない場合でも `1` を指定します。こちらはエラーになるので気づけます。

#### 罠 4:`RasterizerState` のゼロ埋め

```cpp
typedef enum D3D12_FILL_MODE {
    D3D12_FILL_MODE_WIREFRAME = 2,
    D3D12_FILL_MODE_SOLID     = 3
} D3D12_FILL_MODE;

typedef enum D3D12_CULL_MODE {
    D3D12_CULL_MODE_NONE  = 1,
    D3D12_CULL_MODE_FRONT = 2,
    D3D12_CULL_MODE_BACK  = 3
} D3D12_CULL_MODE;
```

**どちらも 0 は定義されていません。** ゼロ埋めすると不正な値になります。これはエラーになるので気づけます。

また `DepthClipEnable` はゼロ埋めで `FALSE` になりますが、**通常は `TRUE` にすべき**です。`FALSE` にすると、視錐台の手前・奥をはみ出した部分がクリップされなくなります。

#### 罠 5:`PrimitiveTopologyType = 0`

`D3D12_PRIMITIVE_TOPOLOGY_TYPE_UNDEFINED` です。**三角形を描くなら `TRIANGLE` を指定します。**

### 14.4.4 既定値を持つ自作関数を用意する

**罠を毎回思い出すのは無理です。仕組みで解決します。**

`d3dx12.h` は `CD3DX12_RASTERIZER_DESC(D3D12_DEFAULT)` のようなコンストラクタでこれを解決しています。**本書は、同じ既定値を返す関数を自作します。**

```cpp
// src/Graphics/D3D12Helpers.h に追加

//---------------------------------------------------------------
// 既定のラスタライザ設定。
// CD3DX12_RASTERIZER_DESC(D3D12_DEFAULT) と同じ値。
//---------------------------------------------------------------
[[nodiscard]] inline D3D12_RASTERIZER_DESC DefaultRasterizerDesc() noexcept
{
    D3D12_RASTERIZER_DESC desc{};
    desc.FillMode              = D3D12_FILL_MODE_SOLID;
    desc.CullMode              = D3D12_CULL_MODE_BACK;
    desc.FrontCounterClockwise = FALSE;
    desc.DepthBias             = D3D12_DEFAULT_DEPTH_BIAS;
    desc.DepthBiasClamp        = D3D12_DEFAULT_DEPTH_BIAS_CLAMP;
    desc.SlopeScaledDepthBias  = D3D12_DEFAULT_SLOPE_SCALED_DEPTH_BIAS;
    desc.DepthClipEnable       = TRUE;
    desc.MultisampleEnable     = FALSE;
    desc.AntialiasedLineEnable = FALSE;
    desc.ForcedSampleCount     = 0;
    desc.ConservativeRaster    =
        D3D12_CONSERVATIVE_RASTERIZATION_MODE_OFF;
    return desc;
}

//---------------------------------------------------------------
// 既定のブレンド設定(ブレンドなし、RGBA すべて書き込む)。
//---------------------------------------------------------------
[[nodiscard]] inline D3D12_BLEND_DESC DefaultBlendDesc() noexcept
{
    D3D12_RENDER_TARGET_BLEND_DESC rt{};
    rt.BlendEnable           = FALSE;
    rt.LogicOpEnable         = FALSE;
    rt.SrcBlend              = D3D12_BLEND_ONE;
    rt.DestBlend             = D3D12_BLEND_ZERO;
    rt.BlendOp               = D3D12_BLEND_OP_ADD;
    rt.SrcBlendAlpha         = D3D12_BLEND_ONE;
    rt.DestBlendAlpha        = D3D12_BLEND_ZERO;
    rt.BlendOpAlpha          = D3D12_BLEND_OP_ADD;
    rt.LogicOp               = D3D12_LOGIC_OP_NOOP;
    rt.RenderTargetWriteMask = D3D12_COLOR_WRITE_ENABLE_ALL;  // ← 罠 2

    D3D12_BLEND_DESC desc{};
    desc.AlphaToCoverageEnable  = FALSE;
    desc.IndependentBlendEnable = FALSE;
    for (auto& target : desc.RenderTarget)
    {
        target = rt;
    }
    return desc;
}

//---------------------------------------------------------------
// 既定の深度ステンシル設定。
// CD3DX12_DEPTH_STENCIL_DESC(D3D12_DEFAULT) と同じく
// DepthEnable は TRUE。深度バッファがない場合は
// 呼び出し側で FALSE にすること。
//---------------------------------------------------------------
[[nodiscard]] inline D3D12_DEPTH_STENCIL_DESC
DefaultDepthStencilDesc() noexcept
{
    D3D12_DEPTH_STENCILOP_DESC op{};
    op.StencilFailOp      = D3D12_STENCIL_OP_KEEP;
    op.StencilDepthFailOp = D3D12_STENCIL_OP_KEEP;
    op.StencilPassOp      = D3D12_STENCIL_OP_KEEP;
    op.StencilFunc        = D3D12_COMPARISON_FUNC_ALWAYS;

    D3D12_DEPTH_STENCIL_DESC desc{};
    desc.DepthEnable      = TRUE;
    desc.DepthWriteMask   = D3D12_DEPTH_WRITE_MASK_ALL;
    desc.DepthFunc        = D3D12_COMPARISON_FUNC_LESS;
    desc.StencilEnable    = FALSE;
    desc.StencilReadMask  = D3D12_DEFAULT_STENCIL_READ_MASK;
    desc.StencilWriteMask = D3D12_DEFAULT_STENCIL_WRITE_MASK;
    desc.FrontFace        = op;
    desc.BackFace         = op;
    return desc;
}

//---------------------------------------------------------------
// 既定のグラフィックス PSO 記述子。
// 危険なゼロ値をすべて安全な既定値に置き換えてある。
// 呼び出し側は、必要なフィールドだけを上書きすればよい。
//---------------------------------------------------------------
[[nodiscard]] inline D3D12_GRAPHICS_PIPELINE_STATE_DESC
DefaultGraphicsPipelineStateDesc() noexcept
{
    D3D12_GRAPHICS_PIPELINE_STATE_DESC desc{};
    desc.BlendState            = DefaultBlendDesc();
    desc.SampleMask            = UINT_MAX;              // ← 罠 1
    desc.RasterizerState       = DefaultRasterizerDesc();
    desc.DepthStencilState     = DefaultDepthStencilDesc();
    desc.IBStripCutValue       =
        D3D12_INDEX_BUFFER_STRIP_CUT_VALUE_DISABLED;
    desc.PrimitiveTopologyType =
        D3D12_PRIMITIVE_TOPOLOGY_TYPE_TRIANGLE;         // ← 罠 5
    desc.SampleDesc.Count      = 1;                     // ← 罠 3
    desc.SampleDesc.Quality    = 0;
    desc.NodeMask              = 0;
    desc.Flags                 = D3D12_PIPELINE_STATE_FLAG_NONE;
    return desc;
}
```

**これで、罠 1 〜 5 がすべて封じられました。**

`d3dx12.h` を使わない代償として自作したヘルパーは、これで 6 つになりました(第11章の 2 つと合わせて)。**付録 A にまとめます。**

> **`D3D12_DEFAULT_DEPTH_BIAS` などの定数について**
>
> `d3dx12.h` を使わなくても、これらの既定値定数は `d3d12.h` に定義されています。**魔法の数字を書く必要はありません。**
>
> `D3D12_DEFAULT_STENCIL_READ_MASK`、`D3D12_DEFAULT_STENCIL_WRITE_MASK` なども同様です。

### 14.4.5 入力レイアウト

**頂点バッファの中身が、どう並んでいるかの宣言**です。

```cpp
constexpr D3D12_INPUT_ELEMENT_DESC kTriangleInputElements[] = {
    { "POSITION", 0, DXGI_FORMAT_R32G32B32_FLOAT,    0,
      0,                            D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA, 0 },
    { "COLOR",    0, DXGI_FORMAT_R32G32B32A32_FLOAT, 0,
      D3D12_APPEND_ALIGNED_ELEMENT, D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA, 0 },
};
```

フィールドの意味です。

| 位置 | フィールド | 値の意味 |
|---|---|---|
| 1 | `SemanticName` | **HLSL のセマンティクスと一致させる**(第13章 13.1.3 節) |
| 2 | `SemanticIndex` | `TEXCOORD0` / `TEXCOORD1` の数字部分 |
| 3 | `Format` | この要素のデータ形式 |
| 4 | `InputSlot` | 何番目の頂点バッファから読むか |
| 5 | `AlignedByteOffset` | 頂点構造体の先頭からのバイト位置 |
| 6 | `InputSlotClass` | 頂点ごとか、インスタンスごとか(第34章) |
| 7 | `InstanceDataStepRate` | インスタンシング用。頂点データなら 0 |

**`D3D12_APPEND_ALIGNED_ELEMENT` は便利です。** 「直前の要素の続きに詰める」という意味で、オフセットを自分で計算しなくて済みます。

`SemanticName` は `LPCSTR`(ナロー文字列)です。**HLSL 側の名前と一文字でも違えば、値が渡りません。**

```
HLSL:            float3 position : POSITION;
入力レイアウト:  { "POSITION", ... }
                    ↑ 一致していること
```

大文字小文字は区別されません(`"position"` でも通ります)が、**揃えておいたほうが読みやすくなります。**

### 14.4.6 組み立てる

```cpp
Core::Result<ComPtr<ID3D12PipelineState>> CreateTrianglePso(
    ID3D12Device*        device,
    ID3D12RootSignature* rootSignature,
    const ShaderBlob&    vs,
    const ShaderBlob&    ps)
{
    //--- 既定値から始める(14.4.4) ---
    auto desc = DefaultGraphicsPipelineStateDesc();

    //--- 必要なフィールドだけを上書きする ---
    desc.pRootSignature = rootSignature;
    desc.VS             = vs.Bytecode();
    desc.PS             = ps.Bytecode();

    desc.InputLayout.pInputElementDescs = kTriangleInputElements;
    desc.InputLayout.NumElements =
        static_cast<UINT>(std::size(kTriangleInputElements));

    desc.NumRenderTargets = 1;
    desc.RTVFormats[0]    = DXGI_FORMAT_R8G8B8A8_UNORM;  // スワップチェーンと一致

    //--- 深度バッファはまだない(第19章で有効にする) ---
    desc.DepthStencilState.DepthEnable = FALSE;
    desc.DSVFormat                     = DXGI_FORMAT_UNKNOWN;

    //--- 生成 ---
    const auto start = std::chrono::steady_clock::now();

    ComPtr<ID3D12PipelineState> pso;
    HR_TRY(device->CreateGraphicsPipelineState(&desc, IID_PPV_ARGS(&pso)));

    const auto elapsed = std::chrono::duration<double, std::milli>(
        std::chrono::steady_clock::now() - start).count();

    Core::SetDebugName(pso.Get(), L"TrianglePSO");
    LOG_INFO(L"PSO created in {:.2f} ms", elapsed);

    return pso;
}
```

**上書きするのは 8 行だけです。** 残り 15 フィールドは既定値のままで正しく動きます。

**`RTVFormats[0]` に注意してください。** スワップチェーンのフォーマット(第11章 11.1.3 節)と一致していなければなりません。食い違うと、PSO の生成は成功して、**描画時にデバッグレイヤーがエラーを出します。**

第24章で sRGB 対応を入れるとき、ここも変更が必要になります。

---

## 14.5 生成時に何が起きているか

`CreateGraphicsPipelineState` の中では、次のことが行われます。

```
① 記述子の整合性を検証
     ・ルートシグネチャとシェーダーの宣言が一致しているか
     ・入力レイアウトとシェーダーの入力が一致しているか
     ・出力フォーマットとシェーダーの出力が一致しているか
        ↓
② DXIL を、このハードウェア向けの機械語にコンパイル
        ↓
③ 固定機能部の設定を、GPU のレジスタ設定に変換
        ↓
④ 全部まとめて、一つの不変オブジェクトにする
```

**② が時間のかかる処理です。** 実測すると、単純なシェーダーでも数ミリ秒かかります。

**「PSO の生成は初期化時にまとめてやる」のが鉄則です。** 描画中に生成すると、その瞬間だけフレームが飛びます。ゲームで「初回のエフェクト発動時にカクつく」のは、たいていこれが原因です。

> **本書では、当面 PSO は 1 つだけです**
>
> 第25章で複数オブジェクトを描くようになり、第26章でポストエフェクトが入り、第27章でシャドウマップが加わると、PSO は 10 個以上になります。そのとき、PSO の管理をどう設計するかが問題になります。
>
> 現時点では 1 つなので、素直に持っておきます。**第9章 9.6.2 節と同じく、抽象化は必要になってからです。**

---

## ✅ 本章のゴール:ルートシグネチャと PSO の生成が成功する

### Step 1:実行する

```cpp
//--- 初期化時 ---
Graphics::ShaderBlob vs, ps;
const auto dir = Graphics::ShaderDirectory();

if (auto r = vs.LoadFromFile(dir / L"Triangle.VS.cso"); !r) { /* ... */ }
if (auto r = ps.LoadFromFile(dir / L"Triangle.PS.cso"); !r) { /* ... */ }

auto rootSig = Graphics::CreateEmptyRootSignature(
    device.Device(), device.Caps().rootSignatureVersion, L"TriangleRS");
if (!rootSig) { Core::ReportError(rootSig.error()); return 1; }

auto pso = Graphics::CreateTrianglePso(
    device.Device(), rootSig->Get(), vs, ps);
if (!pso) { Core::ReportError(pso.error()); return 1; }
```

**期待される出力**

```
[Info ] GraphicsDevice.cpp(295): root signature ver : 1.1
[Info ] ShaderBlob.cpp(48): shader loaded: Triangle.VS.cso (3216 bytes)
[Info ] ShaderBlob.cpp(48): shader loaded: Triangle.PS.cso (2984 bytes)
[Info ] RootSignature.cpp(62): root signature created: TriangleRS (44 bytes)
[Info ] PipelineState.cpp(58): PSO created in 3.42 ms
```

**確認すべき点が 3 つあります。**

1. **ルートシグネチャのバージョンが 1.1 か**(14.2.2 節)
2. **ルートシグネチャのバイナリが数十バイトしかない**。パラメータがゼロなので、フラグくらいしか情報がありません
3. **PSO の生成に数ミリ秒かかっている**。これがドライバによる DXIL のコンパイル時間です(14.5 節)

画面は相変わらず色が変わるだけです。**PSO はまだ使っていません。** 第15章で使います。

### Step 2:入力アセンブラのフラグを外す

**14.2.3 節で説明した罠を、実際に踏んでみます。**

```cpp
versioned.Desc_1_1.Flags =
    // D3D12_ROOT_SIGNATURE_FLAG_ALLOW_INPUT_ASSEMBLER_INPUT_LAYOUT |  ← 外す
      D3D12_ROOT_SIGNATURE_FLAG_DENY_HULL_SHADER_ROOT_ACCESS
    | D3D12_ROOT_SIGNATURE_FLAG_DENY_DOMAIN_SHADER_ROOT_ACCESS
    | D3D12_ROOT_SIGNATURE_FLAG_DENY_GEOMETRY_SHADER_ROOT_ACCESS;
```

**ルートシグネチャの生成は成功します。** 失敗するのは PSO のほうです。

```
[Error] Log.cpp(60): [D3D12] (id 651) D3D12 ERROR:
  ID3D12Device::CreateGraphicsPipelineState: The Root Signature
  does not allow the Input Assembler, but the Pipeline State Object
  contains an Input Layout.
```

**症状と原因が別の場所にある**という、D3D12 でよくあるパターンです。**確認したら元に戻してください。**

### Step 3:セマンティクスを食い違わせる

入力レイアウトの `"COLOR"` を `"COLOUR"` に変えてみます。

```cpp
{ "COLOUR", 0, DXGI_FORMAT_R32G32B32A32_FLOAT, 0, ... },   // ❌
```

```
D3D12 ERROR: ID3D12Device::CreateGraphicsPipelineState:
  The input-assembler expects a semantic 'COLOR' but it was
  not provided by the input layout.
```

**HLSL と C++ の対応が、PSO 生成時に検証されている**ことが分かります。**確認したら元に戻してください。**

### Step 4:既定値の効果を確かめる

`DefaultGraphicsPipelineStateDesc()` の代わりに、素の `{}` を使ってみます。

```cpp
D3D12_GRAPHICS_PIPELINE_STATE_DESC desc{};   // ❌ 既定値なし
desc.pRootSignature = rootSignature;
desc.VS = vs.Bytecode();
desc.PS = ps.Bytecode();
// ... 以下同じ ...
```

**ラスタライザの不正な値でエラーになります。**

```
D3D12 ERROR: ID3D12Device::CreateGraphicsPipelineState:
  FillMode(0) is not a valid D3D12_FILL_MODE.
```

ここで `FillMode` と `CullMode` だけを手で埋めてみてください。**今度は生成が成功します。** しかし `SampleMask` は `0` のままです。

**この PSO は、第15章で使っても何も描きません。** そしてデバッグレイヤーは何も言いません。

**これが 14.4.3 節で「後者が本当に危険」と書いた意味です。** 第15章で三角形が出なかったら、まずここを疑ってください。

**確認したら `DefaultGraphicsPipelineStateDesc()` に戻してください。**

---

### 本章の達成状態

- [ ] ルートシグネチャのバージョンを問い合わせ、`DeviceCaps` に加えた
- [ ] `D3D12_VERSIONED_ROOT_SIGNATURE_DESC` で 1.1 を使っている
- [ ] `ALLOW_INPUT_ASSEMBLER_INPUT_LAYOUT` を指定している
- [ ] 使わないステージに `DENY_*` を指定している
- [ ] `errorBlob` の内容をログに出している
- [ ] `DefaultRasterizerDesc` などの既定値関数を自作した
- [ ] `SampleMask = UINT_MAX` になっている
- [ ] `RenderTargetWriteMask = ALL` になっている
- [ ] `SampleDesc.Count = 1` になっている
- [ ] `RTVFormats[0]` がスワップチェーンと一致している
- [ ] 深度を明示的に無効化している
- [ ] ルートシグネチャと PSO に名前を付けた
- [ ] PSO の生成時間をログに出した
- [ ] Step 2 / 3 / 4 で、それぞれの失敗を確認した

---

## トラブルシューティング

| 症状 | 原因 | 対処 |
|---|---|---|
| PSO が「Input Assembler を許可していない」 | ルートシグネチャのフラグ | 14.2.3 |
| `FillMode(0) is not valid` | `{}` のまま使った | `DefaultRasterizerDesc()`(14.4.4) |
| `SampleDesc.Count` のエラー | 同上 | `Count = 1` |
| セマンティクスが見つからない | HLSL と入力レイアウトの不一致 | 14.4.5 |
| シリアライズが `E_INVALIDARG` | 記述子の誤り | **`errorBlob` を読む**(14.2.4) |
| 描画しても何も出ない | `SampleMask = 0` | **14.4.3 罠 1** |
| 同上 | `RenderTargetWriteMask = 0` | **14.4.3 罠 2** |
| 描画時に RTV フォーマット不一致 | `RTVFormats[0]` の設定誤り | スワップチェーンと合わせる |
| 深度に関するエラー | `DepthEnable = TRUE` なのに DSV がない | `FALSE` にする(14.4.6) |
| PSO 生成が遅い | 仕様。ドライバがコンパイルしている | 初期化時にまとめて生成する(14.5) |
| ルートシグネチャが 1.0 になる | 環境が古い | 動作はする。1.1 の最適化が効かないだけ |

---

## まとめ

**1. ルートシグネチャはシェーダーの引数宣言。**
何を渡すかではなく、**どんな型のものをいくつ渡すか**を宣言します。実データは描画時に指定します。

**2. 予算は 64 DWORD。**
ルート定数は 1、ルートディスクリプタは 2、テーブルは 1。この制約が、第18章以降のリソース設計を決めます。

**3. `ALLOW_INPUT_ASSEMBLER_INPUT_LAYOUT` を忘れない。**
忘れるとルートシグネチャは成功し、**PSO が失敗します。** 症状と原因が別の場所に出る典型例です。

**4. `errorBlob` にしか理由は書かれていない。**
`HRESULT` は `E_INVALIDARG` としか言いません。D3D 周辺の文字列がナローで返る例が、また一つ増えました。

**5. PSO は「事前に固めて、本番を速くする」思想の結晶。**
生成には数ミリ秒かかりますが、描画時は `SetPipelineState` の 1 行で済みます。**必ず初期化時に生成してください。**

**6. `{}` だけでは足りない。**
`SampleMask = 0` と `RenderTargetWriteMask = 0` は、**エラーを出さずに描画結果を消します。** 既定値を返す関数を自作することが、`d3dx12.h` を使わない場合の必須の対策です。

次章で、ついに三角形が出ます。頂点バッファを作り、`Map` でデータを書き込み、ビューポートとシザー矩形を設定して `DrawInstanced` を呼びます。**第1部のゴールです。**

---

## 参考リンク

| 内容 | URL |
|---|---|
| ルートシグネチャの概要 | https://learn.microsoft.com/ja-jp/windows/win32/direct3d12/root-signatures |
| ルートシグネチャの制限 | https://learn.microsoft.com/ja-jp/windows/win32/direct3d12/root-signature-limits |
| ルートシグネチャ バージョン 1.1 | https://learn.microsoft.com/ja-jp/windows/win32/direct3d12/root-signature-version-1-1 |
| HLSL でのルートシグネチャ記述 | https://learn.microsoft.com/ja-jp/windows/win32/direct3d12/specifying-root-signatures-in-hlsl |
| `D3D12_GRAPHICS_PIPELINE_STATE_DESC` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/ns-d3d12-d3d12_graphics_pipeline_state_desc |
| `D3D12_INPUT_ELEMENT_DESC` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/ns-d3d12-d3d12_input_element_desc |
| パイプラインステートオブジェクト | https://learn.microsoft.com/ja-jp/windows/win32/direct3d12/managing-graphics-pipeline-state-in-direct3d-12 |
