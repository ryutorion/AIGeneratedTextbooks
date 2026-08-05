# 第18章 定数バッファ(CBV)

第16章の立方体は、**CPU 側で座標を焼き込んだにせもの**でした。

```cpp
// 【暫定】第17章で Matrix4x4 として整理し、
//         第18章で頂点シェーダーへ移す。
void ProjectPoint(const float in[3], float out[3]) { ... }
```

前章で行列が揃いました。**本章でこれを GPU へ送り、頂点シェーダーで変換します。** 暫定コードは削除され、立方体は回り出します。

送るデータは、たった 64 バイトの行列 1 つです。それだけのために、本章では次を扱います。

- シェーダーから見えるデスクリプタヒープ
- 256 バイトアラインメントの制約
- HLSL のパッキング規則
- ルートパラメータの 3 つの形態
- フレームごとのデータ更新

**「たった 64 バイト」に対して大げさに見えますが、ここで作る仕組みが第20章のテクスチャ、第25章の複数オブジェクト、第33章のバインドレスまで続きます。**

**本章のゴール**
定数バッファで行列を GPU へ送り、頂点シェーダーで変換する。**立方体が回転する。**

---

## 18.1 デスクリプタヒープ再訪

### 18.1.1 シェーダーから見えるヒープ

第11章 11.2 節で RTV 用のデスクリプタヒープを作りました。そのとき、こう書きました。

> RTV と DSV のヒープには `SHADER_VISIBLE` フラグを指定できません。

**今回作るのは、指定できるほうのヒープです。**

| ヒープの種類 | シェーダー可視にできるか |
|---|---|
| **CBV_SRV_UAV** | **できる。本章で使う** |
| **SAMPLER** | できる(第20章) |
| RTV | できない |
| DSV | できない |

「シェーダーから見える」とは、文字どおり **GPU 上のシェーダーコードが、そのヒープの中身を参照できる**という意味です。RTV は出力先の指定であってシェーダーが読むものではないので、可視にする必要がありませんでした。

**定数バッファは違います。** 頂点シェーダーが実際に読みます。だから可視ヒープに置きます。

```cpp
D3D12_DESCRIPTOR_HEAP_DESC desc{};
desc.Type           = D3D12_DESCRIPTOR_HEAP_TYPE_CBV_SRV_UAV;
desc.NumDescriptors = kBackBufferCount;      // フレームごとに 1 つ
desc.Flags          = D3D12_DESCRIPTOR_HEAP_FLAG_SHADER_VISIBLE;   // ← 今回は指定する
desc.NodeMask       = 0;
```

### 18.1.2 同時にバインドできるのは 1 つだけ

**シェーダー可視ヒープには、強い制約があります。**

> **コマンドリストに同時にバインドできるのは、CBV_SRV_UAV ヒープが 1 つ、SAMPLER ヒープが 1 つまで。**

用途ごとにヒープを分けて、描画のたびに切り替える —— という設計はできません。**すべての CBV / SRV / UAV は、1 つのヒープに同居する必要があります。**

そして、**ヒープの切り替えは重い操作です。** GPU によってはパイプラインのフラッシュを伴います。

したがって、現実的な設計は 1 つに定まります。

> **起動時に十分大きなヒープを 1 つ作り、その中を自分で切り分けて使う。**

**これが第19章でデスクリプタアロケータを設計する動機です。** 本章ではフレーム数ぶん(3 個)しか要らないので、素直に確保します。

バインドは `SetDescriptorHeaps` で行います。

```cpp
ID3D12DescriptorHeap* heaps[] = { m_cbvHeap.Get() };
commandList->SetDescriptorHeaps(1, heaps);
```

**`Reset` のたびに呼び直す必要があります**(第15章 15.4.2 節)。コマンドリストの状態は毎回消えます。

### 18.1.3 CPU ハンドルと GPU ハンドル

**シェーダー可視ヒープには、2 種類のハンドルがあります。**

```cpp
D3D12_CPU_DESCRIPTOR_HANDLE cpuStart = heap->GetCPUDescriptorHandleForHeapStart();
D3D12_GPU_DESCRIPTOR_HANDLE gpuStart = heap->GetGPUDescriptorHandleForHeapStart();
```

| ハンドル | 用途 |
|---|---|
| **CPU ハンドル** | デスクリプタを**書き込む**とき(`CreateConstantBufferView` など) |
| **GPU ハンドル** | シェーダーに**参照させる**とき(`SetGraphicsRootDescriptorTable`) |

**同じデスクリプタを、書く側と読む側で別のアドレスで指します。** シェーダー可視でないヒープ(RTV など)では、GPU ハンドルは取得できません。

第11章 11.3.2 節で `OffsetHandle` を CPU 版と GPU 版の 2 つ作ったのは、このためでした。**ここで両方を使います。**

```cpp
const UINT increment = device->GetDescriptorHandleIncrementSize(
    D3D12_DESCRIPTOR_HEAP_TYPE_CBV_SRV_UAV);

// i 番目のデスクリプタを指す 2 つのハンドル
const auto cpu = OffsetHandle(cpuStart, i, increment);
const auto gpu = OffsetHandle(gpuStart, i, increment);
```

**増分サイズは RTV とは別の値です。** ヒープの種類ごとに問い合わせてください(第11章 11.3.1 節)。

---

## 18.2 256 バイトアラインメントの罠

### 18.2.1 2 つの制約

**定数バッファには、256 バイト単位の制約があります。**

```cpp
D3D12_CONSTANT_BUFFER_DATA_PLACEMENT_ALIGNMENT   // = 256
```

| 制約 | 内容 |
|---|---|
| **アドレス** | CBV が指すアドレスは 256 の倍数でなければならない |
| **サイズ** | CBV のサイズは 256 の倍数でなければならない |

本章の定数は行列 1 つ、**64 バイト**です。

```cpp
struct SceneConstants
{
    Math::Matrix4x4 worldViewProj;   // 4×4×4 = 64 バイト
};
static_assert(sizeof(SceneConstants) == 64);
```

**しかし、256 バイトを確保しなければなりません。** 192 バイトが無駄になります。

複数のフレーム分を 1 本のバッファに詰める場合も、**256 バイト刻み**にする必要があります。

```
バッファ(768 バイト)
┌──────────────┬──────────────┬──────────────┐
│ frame 0      │ frame 1      │ frame 2      │
│ 64B + 192B空 │ 64B + 192B空 │ 64B + 192B空 │
└──────────────┴──────────────┴──────────────┘
0            256            512            768
```

**この無駄が気になるなら、定数を増やせばよいのです。** 第22章でライティングを実装すると、ライトの位置・色・カメラ位置などが加わり、200 バイト前後になります。**そのころには「256 バイトを使い切る」ほうが普通になります。**

### 18.2.2 `AlignUp` ヘルパー

切り上げ計算を毎回書くのは面倒なので、ヘルパーにします。

```cpp
// src/Graphics/D3D12Helpers.h に追加

//---------------------------------------------------------------
// value を alignment の倍数へ切り上げる。
// alignment は 2 のべき乗であること。
//---------------------------------------------------------------
[[nodiscard]] constexpr UINT64 AlignUp(UINT64 value, UINT64 alignment) noexcept
{
    return (value + alignment - 1) & ~(alignment - 1);
}

static_assert(AlignUp(0,   256) == 0);
static_assert(AlignUp(1,   256) == 256);
static_assert(AlignUp(64,  256) == 256);
static_assert(AlignUp(256, 256) == 256);
static_assert(AlignUp(257, 256) == 512);
```

**`static_assert` で確認しておきます。** ビット演算は書き間違えやすく、しかも間違えても「だいたい動く」ので気づきにくい部類です(第17章 17.7 節と同じ方針です)。

```cpp
inline constexpr UINT kSceneConstantsSize =
    static_cast<UINT>(AlignUp(sizeof(SceneConstants),
                              D3D12_CONSTANT_BUFFER_DATA_PLACEMENT_ALIGNMENT));

static_assert(kSceneConstantsSize == 256);
```

### 18.2.3 HLSL 側のパッキング規則 —— もう一つの罠

**256 バイトの話より厄介な問題があります。**

**HLSL の `cbuffer` は、C++ の構造体と同じレイアウトになるとは限りません。**

HLSL は、定数を **16 バイト(`float4` 1 個分)単位のレジスタ**に詰めます。そして、

> **1 つのメンバが 16 バイト境界をまたぐことはない。**

という規則があります。

**具体例で見ます。**

```hlsl
cbuffer Bad : register(b0)
{
    float3 a;    // オフセット 0  (12 バイト)
    float2 b;    // ← 12〜20 は境界をまたぐので、16 へ送られる
};
```

```cpp
// C++ 側の素朴な対応
struct Bad
{
    float a[3];   // オフセット 0  (12 バイト)
    float b[2];   // オフセット 12  ← ずれている!
};
```

**`b` の位置が 4 バイトずれます。** そしてこのずれは、**コンパイルエラーにも実行時エラーにもなりません。** 値が壊れるだけです。

一方、次は一致します。

```hlsl
cbuffer Good : register(b0)
{
    float3 a;    // オフセット 0
    float  b;    // オフセット 12  ← 16 バイトに収まるので詰められる
};
```

**規則を覚えるより、事故が起きない書き方をするほうが確実です。**

> **本書のルール:定数バッファの構造体は、`float4` と `float4x4` だけで構成する。**
>
> 3 成分しか要らない場合も `float4` にして、余りをパディングとして使います。

```hlsl
cbuffer SceneConstants : register(b0)
{
    row_major float4x4 worldViewProj;   //  0 〜  63
    float4             lightDirection;  // 64 〜  79  (w は未使用)
    float4             cameraPosition;  // 80 〜  95  (w は未使用)
};
```

```cpp
struct SceneConstants
{
    Math::Matrix4x4 worldViewProj;
    Math::Vector4   lightDirection;
    Math::Vector4   cameraPosition;
};
static_assert(sizeof(SceneConstants) % 16 == 0);
```

**`static_assert` は 16 の倍数であることしか保証しません。** 内部のずれは検出できません。**規則を守ることが唯一の対策です。**

> **`float3` を使いたくなったら**
>
> どうしても使いたい場合は、後ろに `float` を 1 つ置いてください。
>
> ```hlsl
> float3 position;
> float  radius;      // ← 一緒に 16 バイトに収まる
> ```
>
> この形なら C++ 側と一致します。**「3 + 1 で 1 セット」を守るのがコツです。**

---

## 18.3 CBV を作る

### 18.3.1 バッファを確保する

**定数は毎フレーム書き換えるので、UPLOAD ヒープを使います。**

第15章 15.2.1 節で「動かないデータは DEFAULT ヒープへ」と書きました。**定数バッファは逆で、UPLOAD ヒープが正解です。** 毎フレーム CPU から書くのに、いちいち転送していては本末転倒です。

```cpp
const UINT64 totalSize = kSceneConstantsSize * kBackBufferCount;   // 768

const auto heapProps = MakeHeapProperties(D3D12_HEAP_TYPE_UPLOAD);
const auto bufferDesc = MakeBufferDesc(totalSize);

HR_TRY(device->CreateCommittedResource(
    &heapProps,
    D3D12_HEAP_FLAG_NONE,
    &bufferDesc,
    D3D12_RESOURCE_STATE_GENERIC_READ,    // UPLOAD は必ずこれ
    nullptr,
    IID_PPV_ARGS(&m_constantBuffer)));

Core::SetDebugName(m_constantBuffer.Get(), L"SceneConstantBuffer");
```

**リソースの先頭アドレスは、D3D12 が 64KB 境界に確保します。** したがって 256 の倍数でもあり、**先頭は必ずアラインメント条件を満たします。** あとは 256 刻みでオフセットすれば、すべての区画が条件を満たします。

### 18.3.2 CBV を作る

```cpp
const auto gpuAddress = m_constantBuffer->GetGPUVirtualAddress();

for (UINT i = 0; i < kBackBufferCount; ++i)
{
    D3D12_CONSTANT_BUFFER_VIEW_DESC cbvDesc{};
    cbvDesc.BufferLocation = gpuAddress + i * kSceneConstantsSize;
    cbvDesc.SizeInBytes    = kSceneConstantsSize;   // 256 の倍数

    device->CreateConstantBufferView(
        &cbvDesc,
        OffsetHandle(m_cbvHeapCpuStart, i, m_cbvIncrement));
}
```

**フィールドは 2 つだけです。** これまで見てきた D3D12 の構造体の中では最小です。

```cpp
typedef struct D3D12_CONSTANT_BUFFER_VIEW_DESC {
    D3D12_GPU_VIRTUAL_ADDRESS BufferLocation;
    UINT                      SizeInBytes;
} D3D12_CONSTANT_BUFFER_VIEW_DESC;
```

`CreateConstantBufferView` は `void` を返します(第11章 11.4 節の RTV と同じです)。**アラインメント違反はデバッグレイヤーが指摘します。**

### 18.3.3 常時マップする

第15章 15.3.2 節で予告した形にします。

```cpp
void* mapped = nullptr;
const D3D12_RANGE readRange{ 0, 0 };
HR_TRY(m_constantBuffer->Map(0, &readRange, &mapped));
m_cbvMapped = static_cast<std::byte*>(mapped);

// Unmap しない。アプリケーションが終わるまでマップしたまま使う。
```

**毎フレーム `Map` / `Unmap` を往復させる必要はありません。** D3D12 では、マップしたまま GPU に読ませて構いません。

**ただし、書き換えるタイミングは自分で管理する必要があります。** 18.5 節で扱います。

**`Map` した領域から読んではいけない**という制約は変わりません(第15章 15.3.2 節)。書くだけです。

---

## 18.4 ルートパラメータの 3 形態

### 18.4.1 3 つの選択肢

第14章 14.1.2 節で概観だけ示したものを、ここで実際に使います。

```cpp
typedef struct D3D12_ROOT_PARAMETER1 {
    D3D12_ROOT_PARAMETER_TYPE ParameterType;
    union {
        D3D12_ROOT_DESCRIPTOR_TABLE1 DescriptorTable;
        D3D12_ROOT_CONSTANTS         Constants;
        D3D12_ROOT_DESCRIPTOR1       Descriptor;
    };
    D3D12_SHADER_VISIBILITY ShaderVisibility;
} D3D12_ROOT_PARAMETER1;
```

**共用体です。** `ParameterType` で選んだものだけが有効になります(第11章 11.5.2 節の `D3D12_RESOURCE_BARRIER` と同じ形です)。

#### ① ルート定数(32BIT_CONSTANTS)

**値そのものをルートシグネチャに埋め込みます。**

```cpp
param.ParameterType             = D3D12_ROOT_PARAMETER_TYPE_32BIT_CONSTANTS;
param.Constants.ShaderRegister  = 0;    // b0
param.Constants.RegisterSpace   = 0;
param.Constants.Num32BitValues  = 16;   // float4x4 = 16 個
```

```cpp
commandList->SetGraphicsRoot32BitConstants(
    0,                  // ルートパラメータの番号
    16,                 // 値の個数
    &matrix,            // データ
    0);                 // 書き込み開始位置
```

**バッファもデスクリプタも要りません。** 最速です。

**代わりに、予算を大量に消費します。** 行列 1 つで **16 DWORD**、全体の 4 分の 1 です(第14章 14.1.3 節)。

#### ② ルートディスクリプタ(CBV / SRV / UAV)

**リソースの GPU アドレスを直接埋め込みます。**

```cpp
param.ParameterType             = D3D12_ROOT_PARAMETER_TYPE_CBV;
param.Descriptor.ShaderRegister = 0;    // b0
param.Descriptor.RegisterSpace  = 0;
param.Descriptor.Flags          = D3D12_ROOT_DESCRIPTOR_FLAG_DATA_STATIC_WHILE_SET_AT_EXECUTE;
```

```cpp
commandList->SetGraphicsRootConstantBufferView(
    0,
    m_constantBuffer->GetGPUVirtualAddress() + index * kSceneConstantsSize);
```

**デスクリプタヒープも CBV も要りません。** アドレスを渡すだけです。

消費は **2 DWORD**。アドレスが 64bit だからです。

**注意点があります。** ここに渡すアドレスも **256 バイト境界**でなければなりません。

#### ③ ディスクリプタテーブル

**デスクリプタヒープ上の範囲を指します。**

```cpp
D3D12_DESCRIPTOR_RANGE1 range{};
range.RangeType                         = D3D12_DESCRIPTOR_RANGE_TYPE_CBV;
range.NumDescriptors                    = 1;
range.BaseShaderRegister                = 0;    // b0
range.RegisterSpace                     = 0;
range.Flags                             = D3D12_DESCRIPTOR_RANGE_FLAG_DATA_STATIC_WHILE_SET_AT_EXECUTE;
range.OffsetInDescriptorsFromTableStart = D3D12_DESCRIPTOR_RANGE_OFFSET_APPEND;

param.ParameterType                       = D3D12_ROOT_PARAMETER_TYPE_DESCRIPTOR_TABLE;
param.DescriptorTable.NumDescriptorRanges = 1;
param.DescriptorTable.pDescriptorRanges   = &range;
```

```cpp
commandList->SetGraphicsRootDescriptorTable(
    0,
    OffsetHandle(m_cbvHeapGpuStart, index, m_cbvIncrement));
```

消費は **1 DWORD** だけです。**ヒープ上の位置しか持たないから**です。

**そのぶん、参照が 2 段になります。** ルートシグネチャ → ヒープ上のデスクリプタ → リソース。

### 18.4.2 コストと使い分け

| | 消費 DWORD | 間接参照 | ヒープ | 適する用途 |
|---|---|---|---|---|
| ルート定数 | **要素数ぶん** | なし | 不要 | **少数の値。オブジェクト ID、インデックスなど** |
| ルートディスクリプタ | 2 | 1 段 | 不要 | **1 つのリソース。フレーム定数など** |
| ディスクリプタテーブル | **1** | 2 段 | 必要 | **複数のリソースをまとめる。テクスチャ群など** |

**選び方の目安です。**

```
値が数個 (16 バイト以下)         → ルート定数
リソースが 1 つ                  → ルートディスクリプタ
リソースが複数、または数が可変    → ディスクリプタテーブル
```

第25章で複数のオブジェクトを描くとき、**ルート定数でオブジェクト番号だけを渡す**という手を使います。第33章のバインドレスでは、**テーブルすら使わずインデックスだけを渡す**ところまで進みます。

### 18.4.3 ルートシグネチャ 1.1 のフラグが効いてくる

**第14章 14.2.2 節で「1.1 で書き始める」と決めた理由が、ここで回収されます。**

```cpp
range.Flags = D3D12_DESCRIPTOR_RANGE_FLAG_DATA_STATIC_WHILE_SET_AT_EXECUTE;
```

1.1 では、**データやデスクリプタが「いつ変わらないか」をドライバに宣言できます。**

| フラグ | 意味 |
|---|---|
| `DATA_STATIC` | ルートシグネチャに設定した後、**一切変わらない** |
| **`DATA_STATIC_WHILE_SET_AT_EXECUTE`** | **設定してから実行するまでの間は変わらない** |
| `DATA_VOLATILE` | いつ変わるか分からない(既定・最も制約が緩い) |
| `DESCRIPTORS_VOLATILE` | デスクリプタ自体が変わりうる |

**本書のケースを考えます。**

```
① 定数バッファに行列を書く       ← ここで書く
② コマンドリストを記録する
③ SetGraphicsRootDescriptorTable
④ Close, ExecuteCommandLists     ← ここまで変えない
```

**③ から ④ の間、データは変わりません。** したがって `DATA_STATIC_WHILE_SET_AT_EXECUTE` が正しい宣言です。

これを伝えると、ドライバは**データをレジスタにあらかじめ載せておく**といった最適化を選べます。`DATA_VOLATILE` のままだと、毎回メモリから読み直す必要があります。

**宣言と実際の挙動が食い違うと、静かに壊れます。** 「変わらない」と言っておいて変えると、古い値が使われることがあります。**GPU-Based Validation(第7章 7.1.3 節)を有効にすると検出できます。**

### 18.4.4 `ShaderVisibility`

```cpp
param.ShaderVisibility = D3D12_SHADER_VISIBILITY_VERTEX;
```

**どのシェーダーステージから見えるかを指定します。**

| 値 | 意味 |
|---|---|
| `ALL` | 全ステージから見える(既定) |
| **`VERTEX`** | **頂点シェーダーのみ** |
| `PIXEL` | ピクセルシェーダーのみ |
| `HULL` / `DOMAIN` / `GEOMETRY` | 各ステージのみ |
| `MESH` / `AMPLIFICATION` | 第36章 |

**本章の行列は頂点シェーダーしか使わないので、`VERTEX` に絞ります。**

第14章 14.2.3 節で `DENY_*` フラグを付けたのと同じ発想です。**使わないと宣言するほど、ドライバの最適化余地が増えます。**

**絞りすぎると、そのステージから読めなくなります。** 第22章でライティングを実装すると、ピクセルシェーダーからも定数を読みたくなるので、そのときに `ALL` へ広げるか、パラメータを分けるかを判断します。

### 18.4.5 本書の選択

**本章はディスクリプタテーブルを使います。**

正直に書けば、**行列 1 つだけならルートディスクリプタのほうが単純です。** デスクリプタヒープも CBV も要らず、コードが 20 行ほど減ります。

それでもテーブルを選ぶ理由は 2 つです。

**1. 第20章でどのみち必要になる**
テクスチャの SRV は、**必ず**シェーダー可視ヒープに置く必要があります。ルートディスクリプタで SRV を渡すこともできますが、テクスチャの場合は制約が多く現実的ではありません。**ヒープの仕組みを先に作っておくほうが、後の章がなめらかになります。**

**2. デスクリプタ管理を考える動機になる**
18.1.2 節で述べた「ヒープは 1 つ、切り分けて使う」という設計課題に、早い段階で触れておきたいからです。

> **ルートディスクリプタ版も試してください**
>
> 切り替えは簡単です。ルートパラメータを `D3D12_ROOT_PARAMETER_TYPE_CBV` にして、描画側を次に変えるだけです。
>
> ```cpp
> commandList->SetGraphicsRootConstantBufferView(
>     0, m_constantBuffer->GetGPUVirtualAddress() + index * kSceneConstantsSize);
> ```
>
> **`SetDescriptorHeaps` も CBV の生成も不要になります。** どれだけ単純になるかを体験しておくと、第25章で使い分けを判断するときの助けになります。

### 18.4.6 ルートシグネチャを作り直す

第14章の `CreateEmptyRootSignature` を、パラメータつきに置き換えます。

```cpp
Core::Result<ComPtr<ID3D12RootSignature>> CreateSceneRootSignature(
    ID3D12Device* device, D3D_ROOT_SIGNATURE_VERSION version)
{
    //--- b0 の CBV を 1 つ持つテーブル ---
    D3D12_DESCRIPTOR_RANGE1 range{};
    range.RangeType                         = D3D12_DESCRIPTOR_RANGE_TYPE_CBV;
    range.NumDescriptors                    = 1;
    range.BaseShaderRegister                = 0;      // b0
    range.RegisterSpace                     = 0;
    range.Flags                             =
        D3D12_DESCRIPTOR_RANGE_FLAG_DATA_STATIC_WHILE_SET_AT_EXECUTE;
    range.OffsetInDescriptorsFromTableStart =
        D3D12_DESCRIPTOR_RANGE_OFFSET_APPEND;

    D3D12_ROOT_PARAMETER1 params[1]{};
    params[0].ParameterType = D3D12_ROOT_PARAMETER_TYPE_DESCRIPTOR_TABLE;
    params[0].DescriptorTable.NumDescriptorRanges = 1;
    params[0].DescriptorTable.pDescriptorRanges   = &range;
    params[0].ShaderVisibility = D3D12_SHADER_VISIBILITY_VERTEX;

    D3D12_VERSIONED_ROOT_SIGNATURE_DESC versioned{};
    versioned.Version = version;
    versioned.Desc_1_1.NumParameters     = 1;
    versioned.Desc_1_1.pParameters       = params;
    versioned.Desc_1_1.NumStaticSamplers = 0;
    versioned.Desc_1_1.pStaticSamplers   = nullptr;
    versioned.Desc_1_1.Flags =
          D3D12_ROOT_SIGNATURE_FLAG_ALLOW_INPUT_ASSEMBLER_INPUT_LAYOUT
        | D3D12_ROOT_SIGNATURE_FLAG_DENY_HULL_SHADER_ROOT_ACCESS
        | D3D12_ROOT_SIGNATURE_FLAG_DENY_DOMAIN_SHADER_ROOT_ACCESS
        | D3D12_ROOT_SIGNATURE_FLAG_DENY_GEOMETRY_SHADER_ROOT_ACCESS
        | D3D12_ROOT_SIGNATURE_FLAG_DENY_PIXEL_SHADER_ROOT_ACCESS;   // ← 追加

    // ... 以下、第14章 14.2.5 節と同じ ...
}
```

**`DENY_PIXEL_SHADER_ROOT_ACCESS` を追加しました。** ピクセルシェーダーは定数を読まないからです。第22章で読むようになったら外します。

> **`range` の寿命に注意**
>
> `params[0].DescriptorTable.pDescriptorRanges` は **ポインタ**です。`D3D12SerializeVersionedRootSignature` を呼ぶまで、`range` が生きている必要があります。
>
> ローカル変数のアドレスを渡して、関数を抜けてからシリアライズする —— という書き方をすると壊れます。**同じスコープ内で完結させてください。**

---

## 18.5 フレームごとの定数バッファ更新

### 18.5.1 なぜフレームごとに要るのか

**第12章 12.2.1 節の判断基準を思い出してください。**

> **「GPU がまだ読んでいる可能性があるものは、フレームごとに持つ。」**

定数バッファは、GPU が頂点シェーダーの実行中に読みます。**第12章でパイプライン化したので、CPU はフレーム N+2 を記録している最中に、GPU はまだフレーム N を実行しているかもしれません。**

```
CPU: [記録 N][記録 N+1][記録 N+2]
GPU:         [実行 N  ][実行 N+1]
                ↑
        まだ N の定数を読んでいる
```

**1 本の定数バッファを使い回すと、実行中のデータを上書きします。** 立方体が一瞬おかしな向きになる、といった症状が出ます。**しかもエラーは出ません。**

**だからフレーム数ぶん用意します。** 第12章のコマンドアロケータとまったく同じ理屈です。

### 18.5.2 書き込みのタイミング

```cpp
Core::Status Renderer::RenderFrame()
{
    const UINT index = m_swapChain.CurrentIndex();
    FrameResource& frame = m_frames[index];

    //--- ② このスロットの前回の作業を待つ(第12章)---
    if (auto r = m_fence.Wait(frame.fenceValue); !r) { return r; }

    //--- ★ 待った後に定数を書く ★ ---
    UpdateSceneConstants(index);

    //--- ③ 記録の準備 ---
    HR_TRY(frame.allocator->Reset());
    // ...
}
```

**`m_fence.Wait()` の後に書く**ことが重要です。

待つ前に書いてしまうと、まだ GPU が読んでいる領域を上書きする可能性があります。**フレームリソースの待機は、アロケータのためだけでなく、定数バッファのためでもあります。**

```cpp
void Renderer::UpdateSceneConstants(UINT frameIndex)
{
    const float t = ElapsedSeconds();

    //--- ワールド行列:回転させる ---
    const auto world = Math::RotationY(t * 0.8f)
                     * Math::RotationX(t * 0.5f);

    //--- ビュー行列:カメラを後ろに置く ---
    const auto view = Math::LookAtLH(
        Math::Vector3{ 0.0f, 0.0f, -3.0f },
        Math::Vector3{ 0.0f, 0.0f,  0.0f },
        Math::Vector3{ 0.0f, 1.0f,  0.0f });

    //--- 射影行列:縦横比を反映 ---
    const float aspect = static_cast<float>(m_width)
                       / static_cast<float>(m_height);

    const auto proj = Math::PerspectiveFovLH(
        Math::ToRadians(60.0f),
        aspect,
        0.5f,        // nearZ。小さくしすぎない(第17章 17.6.4)
        100.0f);

    //--- 合成 ---
    SceneConstants constants{};
    constants.worldViewProj = world * view * proj;

    //--- 該当スロットへ書き込む ---
    std::memcpy(m_cbvMapped + frameIndex * kSceneConstantsSize,
                &constants, sizeof(constants));
}
```

**`world * view * proj` の順序**は、第17章 17.3.3 節で決めた通りです。適用したい順に左から並べます。

**縦横比が射影行列に入った**ので、第15章・第16章で「立方体が横に伸びる」と書いた問題が解決します。ウィンドウをリサイズしても正しく見えるようになります。

### 18.5.3 描画コード

```cpp
//--- ⑤ 描画 ---
m_commandList->OMSetRenderTargets(1, &rtv, FALSE, nullptr);
m_commandList->ClearRenderTargetView(rtv, clearColor, 0, nullptr);

//--- デスクリプタヒープをバインド(Reset のたびに必要)---
ID3D12DescriptorHeap* heaps[] = { m_cbvHeap.Get() };
m_commandList->SetDescriptorHeaps(1, heaps);

m_commandList->SetGraphicsRootSignature(m_rootSignature.Get());

//--- ★ 定数バッファを渡す ★ ---
m_commandList->SetGraphicsRootDescriptorTable(
    0,                                              // ルートパラメータ番号
    OffsetHandle(m_cbvHeapGpuStart, index, m_cbvIncrement));

m_commandList->SetPipelineState(m_pso.Get());
m_commandList->RSSetViewports(1, &m_viewport);
m_commandList->RSSetScissorRects(1, &m_scissor);

m_commandList->IASetPrimitiveTopology(D3D_PRIMITIVE_TOPOLOGY_TRIANGLELIST);
m_commandList->IASetVertexBuffers(0, 1, &m_vertexBufferView);
m_commandList->IASetIndexBuffer(&m_indexBufferView);

m_commandList->DrawIndexedInstanced(36, 1, 0, 0, 0);
```

**順序に 2 つの決まりがあります。**

**1. `SetDescriptorHeaps` は、テーブルを設定する前に呼ぶ**

ヒープをバインドせずに `SetGraphicsRootDescriptorTable` を呼ぶと、デバッグレイヤーがエラーを出します。

**2. `SetGraphicsRootSignature` は、ルートパラメータの設定より前に呼ぶ**

**ルートシグネチャを設定すると、それまでに設定したルートパラメータの値がすべて消えます**(第15章 15.5.2 節)。

```cpp
// ❌ 間違い
m_commandList->SetGraphicsRootDescriptorTable(0, handle);
m_commandList->SetGraphicsRootSignature(m_rootSignature.Get());  // ← ここで消える
```

**この間違いをすると、シェーダーが読む定数がゴミになります。** 立方体が消えるか、画面いっぱいに引き伸ばされます。

---

## 18.6 シェーダーと立方体を直す

### 18.6.1 シェーダー

```hlsl
//=====================================================
// shaders/Cube.hlsl
//=====================================================

//-----------------------------------------------------
// 定数バッファ。
//
// row_major を指定して、C++ 側の行優先レイアウトと
// 一致させる(第17章 17.3.4 節)。
// これにより、転置が一切不要になる。
//-----------------------------------------------------
cbuffer SceneConstants : register(b0)
{
    row_major float4x4 worldViewProj;
};

struct VSInput
{
    float3 position : POSITION;
    float4 color    : COLOR;
};

struct VSOutput
{
    float4 position : SV_Position;
    float4 color    : COLOR;
};

VSOutput VSMain(VSInput input)
{
    VSOutput output;

    // C++ 側の Transform(v, M) と同じ形
    output.position = mul(float4(input.position, 1.0f), worldViewProj);
    output.color    = input.color;

    return output;
}

float4 PSMain(VSOutput input) : SV_Target
{
    return input.color;
}
```

**`mul(v, M)` の順序**が、C++ 側の `Transform(v, M)` と一致していることを確認してください。**転置は登場しません**(第17章 17.3.4 節)。

第13章で設定したビルド設定に、このファイルを追加します。エントリポイントとターゲットは同じです。

### 18.6.2 立方体データを素の形に戻す

**第16章 16.2.4 節の暫定コードを削除します。**

```cpp
// ❌ 削除する
void ProjectPoint(const float in[3], float out[3]) { ... }
```

頂点は、**モデル空間の座標をそのまま**入れます。

```cpp
constexpr Math::Vector3 kCubeCorners[8] = {
    { -0.5f, -0.5f, -0.5f },   // 0
    {  0.5f, -0.5f, -0.5f },   // 1
    {  0.5f,  0.5f, -0.5f },   // 2
    { -0.5f,  0.5f, -0.5f },   // 3
    { -0.5f, -0.5f,  0.5f },   // 4
    {  0.5f, -0.5f,  0.5f },   // 5
    {  0.5f,  0.5f,  0.5f },   // 6
    { -0.5f,  0.5f,  0.5f },   // 7
};

std::vector<Vertex> BuildCubeVertices()
{
    std::vector<Vertex> vertices;
    vertices.reserve(8);

    for (const auto& c : kCubeCorners)
    {
        Vertex v{};
        v.position[0] = c.x;
        v.position[1] = c.y;
        v.position[2] = c.z;
        v.color[0] = c.x + 0.5f;
        v.color[1] = c.y + 0.5f;
        v.color[2] = c.z + 0.5f;
        v.color[3] = 1.0f;
        vertices.push_back(v);
    }
    return vertices;
}
```

**回転も投影も、頂点データには含まれません。** すべて行列が担当します。

**このコードは 30 行以上短くなりました。** しかも、視点を変えるのに頂点バッファを作り直す必要がなくなりました。**「正しい設計は短い」の実例です。**

---

## ✅ 本章のゴール:立方体が回転する

### Step 1:実行する

**色のついた立方体が、ゆっくり回転します。**

- 縦横比が正しい(引き伸ばされていない)
- ウィンドウをリサイズしても形が保たれる
- 手前の面と奥の面が入れ替わりながら回る

### Step 2:深度テストがないことを確認する

**回転させると、面の前後関係がおかしいことが明確に分かります。**

奥にあるはずの面が手前を隠したり、回転の途中で急に描画順が入れ替わったりします。**インデックスの並び順がそのまま描画順になっている**からです。

**これが第19章の動機です。** 深度バッファを入れると解決します。

「まだ壊れている」ことを、**今のうちに目に焼き付けておいてください。** 第19章で直したときの違いが分かります。

### Step 3:定数バッファを 1 本にしてみる

**18.5.1 節の警告を確かめます。**

フレームごとではなく、常に同じ領域を使うようにします。

```cpp
std::memcpy(m_cbvMapped, &constants, sizeof(constants));   // ❌ index を無視

// 描画側も
m_commandList->SetGraphicsRootDescriptorTable(
    0, m_cbvHeapGpuStart);                                 // ❌ 常に 0 番
```

**症状は環境によって変わります。**

- 何も起きない
- 回転がわずかにカクつく
- 立方体が一瞬おかしな向きになる

**「何も起きない」が最も危険です。** 第16章 16.3.4 節の中間バッファと同じ構図です。定数が増え、GPU の負荷が上がると、必ず表面化します。

**確認したら元に戻してください。**

### Step 4:設定の順序を入れ替える

```cpp
m_commandList->SetGraphicsRootDescriptorTable(0, handle);
m_commandList->SetGraphicsRootSignature(m_rootSignature.Get());   // ❌ 逆
```

**立方体が消えるか、画面いっぱいに広がります。** 行列がゴミになり、頂点がクリップ範囲の外へ飛ぶためです。

デバッグレイヤーが警告を出す場合もあります。**確認したら元に戻してください。**

### Step 5:`row_major` を外す

```hlsl
cbuffer SceneConstants : register(b0)
{
    float4x4 worldViewProj;   // ❌ row_major を外す
};
```

**行列が転置された状態で解釈されます。** 立方体は激しく歪むか、完全に消えます。

**第17章 17.3.4 節で説明した内容の実物です。** 転置しない設計を選んだ以上、`row_major` は必須です。

**確認したら元に戻してください。**

### Step 6:アラインメントを壊す

```cpp
cbvDesc.SizeInBytes = sizeof(SceneConstants);   // ❌ 64 バイト
```

```
D3D12 ERROR: ID3D12Device::CreateConstantBufferView:
  SizeInBytes must be a multiple of 256.
```

**これはデバッグレイヤーが確実に検出します。** 黙って壊れる系ではありません。

**確認したら元に戻してください。**

### Step 7:ルートディスクリプタ版を試す(任意)

18.4.5 節のコラムの通りに書き換えると、**デスクリプタヒープと CBV が丸ごと不要になります。**

- `SetDescriptorHeaps` が消える
- `CreateConstantBufferView` が消える
- `m_cbvHeap` が消える

**同じ結果が、20 行少ないコードで得られます。** 第20章でテクスチャを追加すると事情が変わりますが、**「今の要件だけならこちらが正しい」ことを体験しておく価値があります。**

---

### 本章の達成状態

- [ ] `SHADER_VISIBLE` な CBV_SRV_UAV ヒープを作った
- [ ] CPU ハンドルと GPU ハンドルを使い分けている
- [ ] `AlignUp` を自作し、`static_assert` で検証した
- [ ] 定数バッファのサイズが 256 の倍数になっている
- [ ] 定数バッファの構造体を `float4` / `float4x4` だけで構成した
- [ ] UPLOAD ヒープに確保し、常時マップしている
- [ ] ルートパラメータの 3 形態を理解した
- [ ] ルートシグネチャ 1.1 のフラグを指定した
- [ ] `ShaderVisibility` を `VERTEX` に絞った
- [ ] フレームごとに別の領域へ書いている
- [ ] `Wait` の後に定数を書いている
- [ ] `SetGraphicsRootSignature` をパラメータ設定より前に呼んでいる
- [ ] HLSL で `row_major` を指定している
- [ ] 第16章の暫定コードを削除した
- [ ] **立方体が回転した**
- [ ] Step 2 で深度テストがないことを確認した

---

## トラブルシューティング

| 症状 | 原因 | 対処 |
|---|---|---|
| `CreateConstantBufferView` でエラー | サイズが 256 の倍数でない | 18.2.1 |
| 同上 | アドレスが 256 境界にない | オフセットを 256 刻みに |
| ヒープ生成で失敗 | `SHADER_VISIBLE` を RTV に付けた | 種類を確認(18.1.1) |
| 立方体が消える / 巨大化する | 行列がゴミ | 設定順序(Step 4)、`row_major`(Step 5) |
| 立方体が歪む | `row_major` の指定漏れ | 18.6.1 |
| 定数の値がずれる | HLSL のパッキング規則 | `float4` で揃える(18.2.3) |
| 回転がカクつく | 定数バッファを共有している | フレームごとに分ける(18.5.1) |
| テーブル設定でエラー | `SetDescriptorHeaps` を忘れた | 18.5.3 |
| GBV でデータ不整合の警告 | フラグの宣言と挙動が違う | 18.4.3 |
| 面の前後関係がおかしい | 深度テストがない | **第19章で解決** |
| ピクセルシェーダーから定数が読めない | `ShaderVisibility` が `VERTEX` | `ALL` にする(18.4.4) |

---

## まとめ

**1. シェーダー可視ヒープは 1 つしかバインドできない。**
用途ごとに分けることはできません。**大きなヒープを 1 つ作り、切り分けて使う**のが唯一の現実的な設計です。第19章でアロケータを作ります。

**2. 定数バッファは 256 バイト単位。**
アドレスもサイズも 256 の倍数です。行列 1 つ(64 バイト)でも 256 バイト確保します。

**3. HLSL のパッキング規則のほうが危険。**
16 バイト境界をまたがない、という規則により、C++ 側と食い違うことがあります。**エラーは出ず、値が壊れるだけです。** `float4` と `float4x4` だけで構成するのが最も安全です。

**4. ルートパラメータには 3 形態がある。**
値そのもの、アドレス、ヒープ上の範囲。**予算 64 DWORD の中で、速度と柔軟性を天秤にかけます。**

**5. ルートシグネチャ 1.1 は「変わらなさ」を宣言する仕組み。**
第14章でパラメータがゼロなのに 1.1 で書き始めた理由が、ここで回収されました。

**6. 定数バッファもフレームごとに持つ。**
GPU がまだ読んでいる可能性があるものは、すべてフレーム単位です。コマンドアロケータとまったく同じ理屈です。

**7. 正しい設計は短くなった。**
第16章の暫定コードは 30 行以上ありましたが、行列を GPU へ送る形にしたら消えました。しかも視点を自由に変えられるようになりました。

次章では深度バッファを導入します。**Step 2 で見た「面の前後関係の破綻」が解決します。** あわせて DSV ヒープ、深度フォーマット、そして第17章 17.6.4 節で予告した Z ファイティングと Reversed-Z を扱います。

---

## 参考リンク

| 内容 | URL |
|---|---|
| 定数バッファ ビュー | https://learn.microsoft.com/ja-jp/windows/win32/direct3d12/constant-buffer-view--cbv- |
| ルート シグネチャの制限 | https://learn.microsoft.com/ja-jp/windows/win32/direct3d12/root-signature-limits |
| ルート シグネチャ バージョン 1.1 | https://learn.microsoft.com/ja-jp/windows/win32/direct3d12/root-signature-version-1-1 |
| `D3D12_DESCRIPTOR_RANGE_FLAGS` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/ne-d3d12-d3d12_descriptor_range_flags |
| HLSL のパッキング規則 | https://learn.microsoft.com/ja-jp/windows/win32/direct3dhlsl/dx-graphics-hlsl-packing-rules |
| `SetDescriptorHeaps` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/nf-d3d12-id3d12graphicscommandlist-setdescriptorheaps |
