# 第15章 三角形を描く

**第1部のゴールです。**

14 章分の準備がすべてここに集まります。デバイス、コマンドキュー、フェンス、スワップチェーン、フレームループ、シェーダー、ルートシグネチャ、PSO —— そのすべてが、たった 3 つの頂点を画面に出すためにありました。

残っているのは 3 つだけです。**頂点データを GPU が読める場所に置くこと。描画範囲を指定すること。そして「描け」と言うこと。**

本章には、D3D12 で最も有名な「何も描かれない」の原因がいくつも登場します。裏面カリング、シザー矩形の設定漏れ、そして前章で予告した `SampleMask = 0`。**三角形が出ないときにどこを疑うかを、実際に壊しながら覚えます。**

**本章のゴール**
頂点バッファを作り、`Map` でデータを書き込み、ビューポートとシザー矩形を設定して `DrawInstanced` を呼ぶ。**グラデーションのかかった三角形が画面に表示される。**

---

## 15.1 頂点構造体の定義

### 15.1.1 入力レイアウトと一致させる

第14章 14.4.5 節で、入力レイアウトをこう宣言しました。

```cpp
{ "POSITION", 0, DXGI_FORMAT_R32G32B32_FLOAT,    0, 0,      ... },
{ "COLOR",    0, DXGI_FORMAT_R32G32B32A32_FLOAT, 0, APPEND, ... },
```

**C++ 側の構造体は、これと同じ並びでなければなりません。**

```cpp
// src/Graphics/Vertex.h
#pragma once
#include "std_import.h"

namespace Graphics
{
    struct Vertex
    {
        float position[3];   // POSITION : R32G32B32_FLOAT    (12 バイト)
        float color[4];      // COLOR    : R32G32B32A32_FLOAT (16 バイト)
    };

    // 入力レイアウトの想定と食い違っていないことを確かめる。
    // パディングが入ると静かに壊れるので、明示的に検査する。
    static_assert(sizeof(Vertex) == 28,
                  "Vertex のサイズが入力レイアウトの想定と一致しません");
    static_assert(offsetof(Vertex, position) == 0);
    static_assert(offsetof(Vertex, color)    == 12);
}

```

**`static_assert` を入れておく価値は大きいです。**

構造体にメンバを足したとき、パディングが入ったとき、`float` を `double` に変えたとき —— こうした変更は、**入力レイアウトを直さなければ静かに壊れます。** 三角形が変な形になったり、色がおかしくなったりしますが、エラーは出ません。

コンパイル時に止まるほうが、何倍も速く原因にたどり着けます。

> **第17章で書き直します**
>
> `float position[3]` は素朴すぎる書き方です。第17章で `Vector3` / `Vector4` を自作したら、そちらに置き換えます。
>
> 今の段階で数学ライブラリを持ち出すと、話が二つに割れます。**まず三角形を出すことを優先します。**

### 15.1.2 頂点の座標と巻き順

第13章 13.5.2 節で説明した通り、**本章のシェーダーは座標変換をしません。** 頂点座標はそのままクリップ空間の値として扱われます。

```
        (0, 0.5)  赤
           /\
          /  \
         /    \
        /______\
(-0.5,-0.5)  (0.5,-0.5)
    青           緑
```

**問題は、3 つの頂点をどの順序で並べるかです。**

第14章 14.4.4 節の `DefaultRasterizerDesc()` は、次のようになっていました。

```cpp
desc.CullMode              = D3D12_CULL_MODE_BACK;   // 裏面を捨てる
desc.FrontCounterClockwise = FALSE;                  // 表 = 時計回り
```

つまり、**画面上で時計回りに見える三角形が「表」**です。逆順に並べると、裏面と判定されて**消えます。**

#### 時計の文字盤で考える

**画面座標では、Y 軸が下向きです。** 上が 12 時、右下が 5 時、左下が 7 時にあたります。

```
        12時 (上)
          ↓
   7時 ←──── → 5時
 (左下)        (右下)
```

**12 時 → 5 時 → 7 時** の順にたどると、時計回りです。したがって:

```cpp
constexpr Vertex kTriangleVertices[] = {
    // 位置                        色 (RGBA)
    { {  0.0f,  0.5f, 0.0f }, { 1.0f, 0.0f, 0.0f, 1.0f } },  // 上   : 赤
    { {  0.5f, -0.5f, 0.0f }, { 0.0f, 1.0f, 0.0f, 1.0f } },  // 右下 : 緑
    { { -0.5f, -0.5f, 0.0f }, { 0.0f, 0.0f, 1.0f, 1.0f } },  // 左下 : 青
};
```

**この順序を逆にすると、三角形は表示されません。** 15.7 節の実験で確かめます。

> **クリップ空間と画面座標で Y の向きが違う**
>
> クリップ空間では Y の +1 が**上**、画面座標では Y = 0 が**上**です。ビューポート変換で Y が反転します。
>
> 巻き順が判定されるのは**変換後の画面座標**なので、混乱しやすい箇所です。**「画面を時計の文字盤だと思う」**のが、最も間違えにくい覚え方です。

---

## 15.2 バッファを作る

### 15.2.1 ヒープの種類

**GPU が読めるメモリを確保します。** そのとき最初に決めるのが、**どこに置くか**です。

```cpp
typedef enum D3D12_HEAP_TYPE {
    D3D12_HEAP_TYPE_DEFAULT    = 1,
    D3D12_HEAP_TYPE_UPLOAD     = 2,
    D3D12_HEAP_TYPE_READBACK   = 3,
    D3D12_HEAP_TYPE_CUSTOM     = 4,
    D3D12_HEAP_TYPE_GPU_UPLOAD = 5,   // 第21章
} D3D12_HEAP_TYPE;
```

| 種類 | CPU から | GPU から | 用途 |
|---|---|---|---|
| **DEFAULT** | **書けない** | **速い** | 変わらないデータの置き場所 |
| **UPLOAD** | 書ける | 遅い | CPU → GPU の受け渡し |
| **READBACK** | 読める | 書ける | GPU → CPU の受け渡し |

**性能を考えれば DEFAULT が正解です。** GPU に直結したメモリ(VRAM)に置かれるからです。

しかし、**CPU から直接書き込めません。** データを入れるには、いったん UPLOAD ヒープに書いてから、コピーコマンドで転送する必要があります。**その方法は第16章で扱います。**

**本章は UPLOAD ヒープを使います。**

- 頂点が 3 つしかなく、性能は問題にならない
- `Map` して `memcpy` するだけで済み、コピーコマンドが要らない
- **第16章で DEFAULT ヒープへの転送を学ぶとき、比較対象になる**

> **UPLOAD ヒープが遅い理由**
>
> UPLOAD ヒープのメモリは、**システムメモリ側**に置かれます。GPU はこれを PCIe バス越しに読みます。VRAM への直接アクセスに比べて、帯域も遅延も大幅に劣ります。
>
> 毎フレーム更新するデータ(定数バッファ、第18章)には向いていますが、**動かないモデルの頂点データを置き続けるのは無駄です。**

### 15.2.2 `D3D12_HEAP_PROPERTIES` を埋める

```cpp
typedef struct D3D12_HEAP_PROPERTIES {
    D3D12_HEAP_TYPE         Type;
    D3D12_CPU_PAGE_PROPERTY CPUPageProperty;
    D3D12_MEMORY_POOL       MemoryPoolPreference;
    UINT                    CreationNodeMask;
    UINT                    VisibleNodeMask;
} D3D12_HEAP_PROPERTIES;
```

**`Type` が `CUSTOM` 以外のとき、`CPUPageProperty` と `MemoryPoolPreference` は `UNKNOWN` でなければなりません。**

「よく分からないから細かく指定しておこう」は逆効果です。`CUSTOM` 以外で具体的な値を入れると、`E_INVALIDARG` になります。

ヘルパーを追加します。

```cpp
// src/Graphics/D3D12Helpers.h に追加

//---------------------------------------------------------------
// ヒープ プロパティ。
// CD3DX12_HEAP_PROPERTIES の代替。
//---------------------------------------------------------------
[[nodiscard]] inline D3D12_HEAP_PROPERTIES MakeHeapProperties(
    D3D12_HEAP_TYPE type) noexcept
{
    D3D12_HEAP_PROPERTIES props{};
    props.Type                 = type;
    props.CPUPageProperty      = D3D12_CPU_PAGE_PROPERTY_UNKNOWN;
    props.MemoryPoolPreference = D3D12_MEMORY_POOL_UNKNOWN;
    props.CreationNodeMask     = 1;
    props.VisibleNodeMask      = 1;
    return props;
}
```

### 15.2.3 `D3D12_RESOURCE_DESC` を埋める

```cpp
typedef struct D3D12_RESOURCE_DESC {
    D3D12_RESOURCE_DIMENSION Dimension;
    UINT64                   Alignment;
    UINT64                   Width;
    UINT                     Height;
    UINT16                   DepthOrArraySize;
    UINT16                   MipLevels;
    DXGI_FORMAT              Format;
    DXGI_SAMPLE_DESC         SampleDesc;
    D3D12_TEXTURE_LAYOUT     Layout;
    D3D12_RESOURCE_FLAGS     Flags;
} D3D12_RESOURCE_DESC;
```

**この構造体は、テクスチャとバッファの両方を表現します。** そのため、バッファを作るときには「テクスチャ用のフィールドをどう埋めるか」という問題が生じます。

**バッファの場合、埋め方は完全に決まっています。**

| フィールド | 値 | ゼロ埋めだと |
|---|---|---|
| `Dimension` | `BUFFER` | `UNKNOWN`。**不正** |
| `Alignment` | `0`(既定に任せる) | OK |
| `Width` | **バイト数** | サイズ 0 |
| `Height` | **`1`** | **`0`。不正** |
| `DepthOrArraySize` | **`1`** | **`0`。不正** |
| `MipLevels` | **`1`** | **`0`。不正** |
| `Format` | `UNKNOWN` | OK(バッファは必ず `UNKNOWN`) |
| `SampleDesc.Count` | **`1`** | **`0`。不正** |
| `Layout` | **`ROW_MAJOR`** | **`UNKNOWN`。不正** |
| `Flags` | `NONE` | OK |

**10 フィールド中 5 つが、ゼロでは不正です。** 第14章 14.4.3 節と同じ状況です。ヘルパーを作ります。

```cpp
//---------------------------------------------------------------
// バッファ用のリソース記述子。
// CD3DX12_RESOURCE_DESC::Buffer() の代替。
//---------------------------------------------------------------
[[nodiscard]] inline D3D12_RESOURCE_DESC MakeBufferDesc(
    UINT64 sizeInBytes,
    D3D12_RESOURCE_FLAGS flags = D3D12_RESOURCE_FLAG_NONE) noexcept
{
    D3D12_RESOURCE_DESC desc{};
    desc.Dimension          = D3D12_RESOURCE_DIMENSION_BUFFER;
    desc.Alignment          = 0;
    desc.Width              = sizeInBytes;
    desc.Height             = 1;
    desc.DepthOrArraySize   = 1;
    desc.MipLevels          = 1;
    desc.Format             = DXGI_FORMAT_UNKNOWN;   // バッファは必ず UNKNOWN
    desc.SampleDesc.Count   = 1;
    desc.SampleDesc.Quality = 0;
    desc.Layout             = D3D12_TEXTURE_LAYOUT_ROW_MAJOR;  // バッファは必ず ROW_MAJOR
    desc.Flags              = flags;
    return desc;
}
```

**`Format` が `UNKNOWN` である点に注意してください。** 「頂点は float3 だから `R32G32B32_FLOAT` では?」と考えたくなりますが、違います。**バッファは型のないバイト列**です。どう解釈するかは、頂点バッファビュー(15.3.3 節)や入力レイアウトが決めます。

### 15.2.4 `CreateCommittedResource`

```cpp
const auto heapProps = MakeHeapProperties(D3D12_HEAP_TYPE_UPLOAD);
const auto bufferDesc = MakeBufferDesc(sizeof(kTriangleVertices));

ComPtr<ID3D12Resource> vertexBuffer;
HR_TRY(device->CreateCommittedResource(
    &heapProps,
    D3D12_HEAP_FLAG_NONE,
    &bufferDesc,
    D3D12_RESOURCE_STATE_GENERIC_READ,   // ← UPLOAD ヒープでは必須
    nullptr,                              // 最適化クリア値(テクスチャ用)
    IID_PPV_ARGS(&vertexBuffer)));

Core::SetDebugName(vertexBuffer.Get(), L"TriangleVertexBuffer");
```

**第 4 引数の初期状態には、ヒープごとに決まりがあります。**

| ヒープ | 必要な初期状態 | 状態遷移 |
|---|---|---|
| UPLOAD | **`GENERIC_READ`** | **できない。ずっとこのまま** |
| READBACK | **`COPY_DEST`** | できない |
| DEFAULT | 用途に応じて自由 | できる |

**UPLOAD ヒープのリソースは、状態を変えられません。** 第11章 11.5 節でバリアを書きましたが、UPLOAD ヒープのバッファに対してバリアを張ろうとすると、デバッグレイヤーに叱られます。

「`Commited`(コミット済み)」という名前は、**ヒープとリソースを同時に確保する**という意味です。ヒープを自分で管理して、そこにリソースを配置する `CreatePlacedResource` という方式もあります。第21章でリソース管理を設計する際に触れます。

---

## 15.3 `Map` / `Unmap` とメモリ配置

### 15.3.1 書き込む

```cpp
void* mapped = nullptr;

// CPU は読まないので、読み取り範囲を空にする
const D3D12_RANGE readRange{ 0, 0 };

HR_TRY(vertexBuffer->Map(0, &readRange, &mapped));
std::memcpy(mapped, kTriangleVertices, sizeof(kTriangleVertices));
vertexBuffer->Unmap(0, nullptr);
```

**引数の意味を押さえておきます。**

| 引数 | 意味 |
|---|---|
| 第1(`Subresource`) | バッファは常に `0` |
| 第2(`pReadRange`) | **CPU が読む範囲。`{0,0}` は「読まない」** |
| 第3(`ppData`) | 書き込み先のポインタが返る |

`Unmap` の第2引数(`pWrittenRange`)に `nullptr` を渡すと、「全体を書いたかもしれない」という意味になります。

### 15.3.2 UPLOAD ヒープのメモリから読んではいけない

**`pReadRange` を `{0, 0}` にするのは、単なる作法ではありません。**

UPLOAD ヒープのメモリは、通常 **ライトコンバイン(write-combined)** として割り当てられます。この方式には、明確な非対称性があります。

| 操作 | 速度 |
|---|---|
| 書き込み | 速い(まとめてバースト転送される) |
| **読み込み** | **壊滅的に遅い** |

キャッシュが効かず、1 バイト読むたびにバス越しのアクセスが発生します。**数十倍から数百倍遅くなることがあります。**

```cpp
// ❌ 絶対にやってはいけない
float* data = static_cast<float*>(mapped);
data[0] += 1.0f;        // 読んでから書いている
```

```cpp
// ✅ 書くだけ
std::memcpy(mapped, source, size);
```

**「CPU 側にもデータを持っておき、完成したものを memcpy する」** のが正しい形です。マップした領域を計算用のメモリとして使ってはいけません。

> **常時マップしたままにしてよい**
>
> D3D12 では、`Map` したまま `Unmap` せずに使い続けても構いません。GPU が読んでいる最中でも問題ありません(内容の整合性は自分で管理する必要がありますが)。
>
> 毎フレーム更新するバッファでは、**`Map` / `Unmap` の往復コストを避けるため常時マップが定石です。** 第18章の定数バッファでこの形にします。
>
> 本章は初期化時に一度書くだけなので、素直に `Unmap` します。

### 15.3.3 頂点バッファビュー

**バッファはただのバイト列です。** 「28 バイトごとに 1 頂点」という解釈を、別途伝える必要があります。

```cpp
D3D12_VERTEX_BUFFER_VIEW vbv{};
vbv.BufferLocation = vertexBuffer->GetGPUVirtualAddress();
vbv.SizeInBytes    = sizeof(kTriangleVertices);   // 84 バイト
vbv.StrideInBytes  = sizeof(Vertex);              // 28 バイト
```

**注目すべき点があります。これは「デスクリプタ」ではありません。**

第11章 11.2 節で作った RTV は、デスクリプタヒープに置く必要がありました。**頂点バッファビューは違います。** デスクリプタヒープを使わず、構造体をそのままコマンドリストに渡します。

| ビューの種類 | 置き場所 |
|---|---|
| RTV / DSV / SRV / CBV / UAV | **デスクリプタヒープ** |
| **頂点バッファビュー** | **直接コマンドリストへ** |
| **インデックスバッファビュー** | 同上(第16章) |

理由は、入力アセンブラが固定機能であり、シェーダーから任意にアクセスされるものではないからです。**扱いが単純なぶん、覚えることが少なくて助かります。**

`GetGPUVirtualAddress()` は、**GPU から見たアドレス**を返します。CPU のポインタとは別物です。バッファ以外(テクスチャ)では呼べません。

---

## 15.4 ビューポートとシザー矩形

### 15.4.1 2 つの矩形

```
ビューポート : クリップ空間 [-1,1] を、画面のどこに対応させるか
シザー矩形   : そのうち、どの範囲だけを実際に描くか
```

**両方とも必須です。片方でも設定を忘れると、何も描かれません。**

```cpp
D3D12_VIEWPORT viewport{};
viewport.TopLeftX = 0.0f;
viewport.TopLeftY = 0.0f;
viewport.Width    = static_cast<float>(clientWidth);
viewport.Height   = static_cast<float>(clientHeight);
viewport.MinDepth = D3D12_MIN_DEPTH;   // 0.0f
viewport.MaxDepth = D3D12_MAX_DEPTH;   // 1.0f

D3D12_RECT scissor{};
scissor.left   = 0;
scissor.top    = 0;
scissor.right  = static_cast<LONG>(clientWidth);
scissor.bottom = static_cast<LONG>(clientHeight);
```

**型が違う点に注意してください。** ビューポートは `float`、シザー矩形は `LONG` です。ビューポートが浮動小数点なのは、サブピクセル単位のオフセットを扱えるようにするためです(第26章のポストエフェクトなどで使います)。

`MinDepth` / `MaxDepth` は、深度値の書き込み範囲です。**D3D の深度範囲は [0, 1] です**(第13章 13.5.2 節)。OpenGL の [-1, 1] とは違います。

### 15.4.2 D3D12 には既定値がない

**Direct3D 12 は、ビューポートもシザー矩形も、既定値を持ちません。**

設定しなければ、幅も高さも 0 の状態です。**当然、何も描かれません。**

さらに厄介なことがあります。

> **コマンドリストの状態は `Reset` で消えます。**

`Reset()` を呼ぶと、パイプラインステート、ルートシグネチャ、ビューポート、シザー矩形、頂点バッファ —— **すべての設定が失われます。**

つまり、**毎フレーム、全部設定し直す必要があります。**

```cpp
// 毎フレーム必要
commandList->SetGraphicsRootSignature(...);
commandList->SetPipelineState(...);
commandList->RSSetViewports(1, &viewport);
commandList->RSSetScissorRects(1, &scissor);
commandList->IASetPrimitiveTopology(...);
commandList->IASetVertexBuffers(...);
```

「初期化時に一度設定すればいい」と考えると、**何も描かれません。** D3D11 の感覚が残っているとはまりやすい箇所です。

### 15.4.3 リサイズへの対応

ウィンドウのサイズが変わったら、ビューポートとシザー矩形も更新が必要です。

**第12章 12.4.5 節の `Renderer::Resize` に追加します。**

```cpp
Core::Status Renderer::Resize(UINT width, UINT height)
{
    // ... スワップチェーンのリサイズ ...

    m_width  = width;
    m_height = height;

    //--- ビューポートとシザー矩形を更新 ---
    m_viewport.TopLeftX = 0.0f;
    m_viewport.TopLeftY = 0.0f;
    m_viewport.Width    = static_cast<float>(width);
    m_viewport.Height   = static_cast<float>(height);
    m_viewport.MinDepth = D3D12_MIN_DEPTH;
    m_viewport.MaxDepth = D3D12_MAX_DEPTH;

    m_scissor.left   = 0;
    m_scissor.top    = 0;
    m_scissor.right  = static_cast<LONG>(width);
    m_scissor.bottom = static_cast<LONG>(height);

    return {};
}
```

第12章で「第19章で深度バッファを追加したら、ここで一緒に作り直す」とコメントを残しました。**そこに追加する最初の項目が、これです。**

---

## 15.5 描画する

### 15.5.1 コマンドの並び

第12章 12.3.2 節の `RenderFrame` の ⑤ に、描画コマンドを入れます。

```cpp
//--- ⑤ 描画 ---
m_commandList->OMSetRenderTargets(1, &rtv, FALSE, nullptr);

float clearColor[4]{};
GetAnimatedColor(clearColor);
m_commandList->ClearRenderTargetView(rtv, clearColor, 0, nullptr);

//===== ここから本章で追加 =====

// パイプラインの設定
m_commandList->SetGraphicsRootSignature(m_rootSignature.Get());
m_commandList->SetPipelineState(m_pso.Get());

// ラスタライザの設定
m_commandList->RSSetViewports(1, &m_viewport);
m_commandList->RSSetScissorRects(1, &m_scissor);

// 入力アセンブラの設定
m_commandList->IASetPrimitiveTopology(D3D_PRIMITIVE_TOPOLOGY_TRIANGLELIST);
m_commandList->IASetVertexBuffers(0, 1, &m_vertexBufferView);

// 描画
m_commandList->DrawInstanced(
    3,    // 1 インスタンスあたりの頂点数
    1,    // インスタンス数
    0,    // 開始頂点
    0);   // 開始インスタンス

//===== ここまで =====
```

### 15.5.2 ルートシグネチャは PSO とは別に設定する

**見落としやすい点です。**

```cpp
m_commandList->SetGraphicsRootSignature(m_rootSignature.Get());
m_commandList->SetPipelineState(m_pso.Get());
```

PSO はルートシグネチャを内部に持っています(第14章 14.4.6 節)。**しかし、`SetPipelineState` はルートシグネチャを設定しません。**

**両方を明示的に設定する必要があります。**

これは奇妙に見えますが、理由があります。**ルートシグネチャが同じで PSO だけ違う、という描画が非常に多いから**です。同じルートシグネチャを使い続けるなら、`SetGraphicsRootSignature` は 1 回で済みます。分離しておくと、無駄な再設定を避けられます。

なお、`SetGraphicsRootSignature` を呼ぶと、**それまでに設定したルートパラメータの値はすべて消えます。** 第18章で定数バッファを設定するようになったら、この順序が問題になります。

### 15.5.3 2 つの「トポロジ」を混同しない

**紛らわしい名前の列挙型が 2 つあります。**

```cpp
// PSO に指定するもの
D3D12_PRIMITIVE_TOPOLOGY_TYPE_TRIANGLE

// コマンドリストに指定するもの
D3D_PRIMITIVE_TOPOLOGY_TRIANGLELIST
```

| | 型 | 粒度 |
|---|---|---|
| PSO | `D3D12_PRIMITIVE_TOPOLOGY_TYPE` | **大分類**(点 / 線 / 三角形 / パッチ) |
| コマンドリスト | `D3D_PRIMITIVE_TOPOLOGY` | **具体的な並べ方**(リスト / ストリップ) |

PSO 側は「三角形を描く」とだけ言い、コマンドリスト側が「3 頂点で 1 枚(リスト)なのか、頂点を共有していく(ストリップ)のか」を指定します。

**この分離のおかげで、リストとストリップを切り替えるのに PSO を作り直さずに済みます。** PSO の生成は数ミリ秒かかる(第14章 14.5 節)ので、これは意味のある設計です。

| `D3D_PRIMITIVE_TOPOLOGY` | 意味 |
|---|---|
| `TRIANGLELIST` | 3 頂点ごとに 1 三角形。**本書の既定** |
| `TRIANGLESTRIP` | 直前 2 頂点を共有して次の三角形を作る |
| `LINELIST` | 2 頂点ごとに 1 線分 |
| `POINTLIST` | 1 頂点で 1 点 |

### 15.5.4 `DrawInstanced` の引数

```cpp
void DrawInstanced(
    UINT VertexCountPerInstance,   // 3
    UINT InstanceCount,            // 1
    UINT StartVertexLocation,      // 0
    UINT StartInstanceLocation);   // 0
```

「インスタンス」という名前ですが、**インスタンシングを使わない場合でもこの関数を呼びます。** `InstanceCount = 1` にするだけです。

D3D11 には `Draw` と `DrawInstanced` が別々にありましたが、D3D12 では統合されました。**インスタンス数 1 は特別扱いされず、同じ経路を通ります。**

インスタンシングの本格的な利用は第34章です。

> **頂点バッファなしで描く方法**
>
> 第14章 14.2.3 節で「入力アセンブラを使わない描画方式がある」と書きました。頂点シェーダーで `SV_VertexID` を受け取り、そこから座標を計算する方法です。
>
> ```hlsl
> VSOutput VSMain(uint vertexId : SV_VertexID)
> {
>     float2 pos[3] = { float2(0, 0.5), float2(0.5, -0.5), float2(-0.5, -0.5) };
>     // ...
> }
> ```
>
> 頂点バッファも入力レイアウトも不要になります。**第26章のフルスクリーン三角形で、この方式を使います。** そのときルートシグネチャから `ALLOW_INPUT_ASSEMBLER_INPUT_LAYOUT` を外せます。

---

## ✅ 本章のゴール:三角形が表示される

### Step 1:実行する

`F5` で実行してください。

**背景色がゆっくり変化する中に、グラデーションのかかった三角形が表示されます。**

```
        赤
        /\
       /  \
      / 補間 \
     /______\
   青        緑
```

**第1部のゴール到達です。**

出ない場合は、次の実験を順に試すと原因が絞れます。

### Step 2:巻き順を逆にする —— 裏面カリング

15.1.2 節で「順序を逆にすると消える」と書きました。**確かめます。**

```cpp
constexpr Vertex kTriangleVertices[] = {
    { {  0.0f,  0.5f, 0.0f }, { 1.0f, 0.0f, 0.0f, 1.0f } },  // 上
    { { -0.5f, -0.5f, 0.0f }, { 0.0f, 0.0f, 1.0f, 1.0f } },  // 左下 ← 入れ替え
    { {  0.5f, -0.5f, 0.0f }, { 0.0f, 1.0f, 0.0f, 1.0f } },  // 右下 ← 入れ替え
};
```

**三角形が消えます。** そして **デバッグレイヤーは何も言いません。** 正しく動作した結果、裏面が捨てられただけだからです。

続けて、カリングを無効にしてみます。

```cpp
// CreateTrianglePso の中で
desc.RasterizerState.CullMode = D3D12_CULL_MODE_NONE;
```

**三角形が戻ってきます。** 裏面カリングが原因だったことが確定します。

**確認したら両方とも元に戻してください。**

> **実務では `CULL_MODE_NONE` に逃げない**
>
> 「消えたからカリングを切る」で済ませると、**表と裏が逆のモデルを抱えたまま進む**ことになります。第22章でライティングを実装したとき、法線の向きが逆で真っ黒になります。
>
> **巻き順を正しく直してください。**

### Step 3:シザー矩形を消す

```cpp
// m_commandList->RSSetScissorRects(1, &m_scissor);   ← コメントアウト
```

**三角形が消えます。** 15.4.2 節で説明した通り、D3D12 には既定値がないためです。

デバッグレイヤーが警告を出す場合もありますが、**出ないこともあります。** 「何も描かれない」ときの定番の容疑者として覚えておいてください。

**確認したら元に戻してください。**

### Step 4:`SampleMask = 0` —— 前章の宿題

**第14章 14.4.3 節「罠 1」で予告したものです。**

```cpp
auto desc = DefaultGraphicsPipelineStateDesc();
desc.SampleMask = 0;      // ❌ わざと壊す
```

- PSO の生成は**成功します**
- デバッグレイヤーは**何も言いません**
- 描画コマンドも**成功します**
- **三角形だけが消えます**

**これが「黙って壊れる」の実物です。**

第14章で `DefaultGraphicsPipelineStateDesc()` を自作した理由が、ここで実感できます。`d3dx12.h` を使っていれば `CD3DX12_DEFAULT` が黙って `UINT_MAX` を入れてくれていました。**使わないと決めた以上、自分で知っていなければなりません。**

**確認したら元に戻してください。**

### Step 5:書き込みマスクを消す

同じく第14章「罠 2」です。

```cpp
desc.BlendState.RenderTarget[0].RenderTargetWriteMask = 0;   // ❌
```

**これも黙って消えます。** ピクセルシェーダーは実行されているのに、どのチャンネルにも書き込まれません。

**確認したら元に戻してください。**

---

## 15.6 頂点カラーで補間を確認する

三角形が出たら、**色に注目してください。**

3 つの頂点にはそれぞれ赤・緑・青を指定しただけです。それなのに、**三角形の内部は滑らかなグラデーション**になっています。

### 15.6.1 誰が補間しているのか

**ラスタライザです。**

```
頂点シェーダー   3 回実行(頂点ごと)
      ↓
   ラスタライザ   三角形をピクセルに分解しながら、
                  頂点の出力値を各ピクセル用に補間する
      ↓
ピクセルシェーダー  数万回実行(ピクセルごと)
```

第13章 13.5 節のピクセルシェーダーは、こう書かれていました。

```hlsl
float4 PSMain(VSOutput input) : SV_Target
{
    return input.color;   // 受け取った色をそのまま返しているだけ
}
```

**補間するコードは一行も書いていません。** それでもグラデーションになるのは、固定機能のラスタライザが `SV_Position` 以外のすべての出力を自動的に補間するからです。

**この仕組みが、この先すべての描画の土台になります。**

| 補間されるもの | 使う章 |
|---|---|
| 色 | **本章** |
| テクスチャ座標(UV) | 第20章 |
| 法線 | 第22章 |
| ワールド座標 | 第22章 |

### 15.6.2 補間を止めてみる

HLSL には、補間方法を指定する修飾子があります。

```hlsl
struct VSOutput
{
    float4 position : SV_Position;
    nointerpolation float4 color : COLOR;   // ← 補間しない
};
```

**三角形が単色になります。** 3 頂点のうち「代表頂点」(既定では最初の頂点)の値が、面全体に使われます。

| 修飾子 | 意味 |
|---|---|
| (なし) | パースペクティブ補正つきで補間。**既定** |
| `nointerpolation` | 補間しない。代表頂点の値 |
| `linear` | 既定と同じ |
| `noperspective` | 画面空間で線形に補間(パースペクティブ補正なし) |
| `centroid` | MSAA 時にサンプル位置を調整(第28章) |

**確認したら修飾子を外してください。**

> **「パースペクティブ補正」とは**
>
> 奥行きのある三角形を画面に投影すると、**画面上で等間隔でも、3D 空間では等間隔ではありません。** 単純に画面座標で線形補間すると、テクスチャが歪みます。
>
> GPU は `w` 成分を使ってこれを補正します。本章の三角形は変換していないので `w = 1` であり、補正の有無で結果は変わりません。**第18章で射影変換を入れると、違いが見えるようになります。**

---

### 本章の達成状態

- [ ] `Vertex` 構造体に `static_assert` を入れた
- [ ] 頂点の順序が画面上で時計回りになっている
- [ ] `MakeHeapProperties` / `MakeBufferDesc` を自作した
- [ ] `Format = UNKNOWN`、`Layout = ROW_MAJOR` になっている
- [ ] UPLOAD ヒープを `GENERIC_READ` で作成した
- [ ] `Map` の `pReadRange` を `{0,0}` にした
- [ ] マップした領域から読んでいない
- [ ] 頂点バッファビューをデスクリプタヒープに置いていない
- [ ] ビューポートとシザー矩形を**毎フレーム**設定している
- [ ] `Resize` でビューポートとシザー矩形を更新している
- [ ] ルートシグネチャと PSO を両方設定している
- [ ] **三角形が表示された**
- [ ] Step 2 〜 5 で、消える原因を 4 通り確認した
- [ ] 色が補間されていることを確認した

---

## トラブルシューティング

### 三角形が出ないときのチェック順

**上から順に確認してください。頻度の高い順に並べてあります。**

| # | 疑うところ | 確認方法 |
|---|---|---|
| 1 | **巻き順** | `CullMode` を `NONE` にして出るか(Step 2) |
| 2 | **シザー矩形** | `RSSetScissorRects` を呼んでいるか(Step 3) |
| 3 | **ビューポート** | `RSSetViewports` を呼んでいるか |
| 4 | **`SampleMask`** | `UINT_MAX` になっているか(Step 4) |
| 5 | **書き込みマスク** | `COLOR_WRITE_ENABLE_ALL` か(Step 5) |
| 6 | ルートシグネチャ | `SetGraphicsRootSignature` を呼んでいるか(15.5.2) |
| 7 | 毎フレーム設定 | `Reset` 後に全部設定し直しているか(15.4.2) |
| 8 | 頂点データ | クリップ空間 [-1,1] の範囲に入っているか |
| 9 | ストライド | `StrideInBytes` が `sizeof(Vertex)` か |
| 10 | トポロジ | `IASetPrimitiveTopology` を呼んでいるか |

### その他

| 症状 | 原因 | 対処 |
|---|---|---|
| `CreateCommittedResource` が `E_INVALIDARG` | `Layout` が `UNKNOWN` | `ROW_MAJOR` に(15.2.3) |
| 同上 | `Height` / `MipLevels` が 0 | `MakeBufferDesc` を使う |
| 同上 | UPLOAD なのに `GENERIC_READ` でない | 15.2.4 |
| 同上 | `CUSTOM` 以外で `CPUPageProperty` を指定した | `UNKNOWN` に(15.2.2) |
| バリアで警告 | UPLOAD ヒープに状態遷移を試みた | UPLOAD は遷移不可(15.2.4) |
| 三角形が歪む | ウィンドウが正方形でない | 仕様。第18章の射影行列で解決 |
| 色がおかしい | 頂点構造体と入力レイアウトの不一致 | `static_assert`(15.1.1) |
| 極端に遅い | マップした領域から読んでいる | 15.3.2 |
| リサイズで描画位置がずれる | ビューポートを更新していない | 15.4.3 |

---

## まとめ

**1. UPLOAD ヒープは「書くだけ」。**
ライトコンバインメモリなので、読み込みは壊滅的に遅くなります。CPU 側にデータを持ち、完成したものを `memcpy` してください。

**2. バッファの `Format` は `UNKNOWN`、`Layout` は `ROW_MAJOR`。**
バッファは型のないバイト列です。解釈を与えるのは頂点バッファビューと入力レイアウトです。

**3. 頂点バッファビューはデスクリプタではない。**
デスクリプタヒープを使わず、構造体を直接コマンドリストに渡します。

**4. D3D12 にはビューポートもシザー矩形も既定値がない。**
そして `Reset` で状態が消えるため、**毎フレーム全部設定し直す**必要があります。

**5. ルートシグネチャは PSO とは別に設定する。**
PSO が内部に持っていても、`SetPipelineState` では設定されません。

**6. 「何も描かれない」の原因は限られている。**
巻き順、シザー矩形、ビューポート、`SampleMask`、書き込みマスク。**Step 2 〜 5 で 4 つとも自分の手で再現したので、次に遭遇したときは 1 分で切り分けられます。**

**7. 補間はラスタライザがやっている。**
`SV_Position` 以外のすべての頂点出力が、自動的に補間されてピクセルシェーダーに届きます。第20章の UV も、第22章の法線も、この仕組みに乗ります。

---

## 第1部を終えて

ここまでで、**Direct3D 12 の骨格をひと通り自分の手で書いたことになります。**

| 章 | 得たもの |
|---|---|
| 第7章 | デバイスと、機能を問い合わせる手段 |
| 第9章 | コマンドの記録と投入 |
| 第10章 | CPU と GPU の同期 |
| 第11章 | 画面への出力、リソースバリア、デスクリプタ |
| 第12章 | パイプライン化されたフレームループ |
| 第13章 | シェーダーのコンパイルとデバッグ情報 |
| 第14章 | ルートシグネチャと PSO |
| 第15章 | リソースの確保と描画 |

**そして、`d3dx12.h` の代わりに自作したヘルパーは 8 つになりました。**

```
OffsetHandle(CPU)          第11章
OffsetHandle(GPU)          第11章
MakeTransitionBarrier      第11章
DefaultRasterizerDesc      第14章
DefaultBlendDesc           第14章
DefaultDepthStencilDesc    第14章
DefaultGraphicsPipelineStateDesc  第14章
MakeHeapProperties         第15章
MakeBufferDesc             第15章
```

**これが「ライブラリを使わない」の代償のすべてです。** ファイル 1 枚に収まる量でした。そして代わりに、リソースバリアが何を指定する構造体なのか、デスクリプタハンドルが何なのか、`SampleMask` が何をしているのかを、我々は知っています。

第2部からは、この土台の上に「3D らしさ」を積んでいきます。次章はインデックスバッファと立方体、そして DEFAULT ヒープへの転送です。**本章で使った UPLOAD ヒープの、正しい使い方を学びます。**

---

## 参考リンク

| 内容 | URL |
|---|---|
| `ID3D12Device::CreateCommittedResource` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/nf-d3d12-id3d12device-createcommittedresource |
| `D3D12_HEAP_PROPERTIES` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/ns-d3d12-d3d12_heap_properties |
| `D3D12_RESOURCE_DESC` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/ns-d3d12-d3d12_resource_desc |
| `ID3D12Resource::Map` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/nf-d3d12-id3d12resource-map |
| `D3D12_VERTEX_BUFFER_VIEW` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/ns-d3d12-d3d12_vertex_buffer_view |
| 座標系とジオメトリ | https://learn.microsoft.com/ja-jp/windows/win32/direct3d12/coordinate-systems-and-geometry |
| HLSL の補間修飾子 | https://learn.microsoft.com/ja-jp/windows/win32/direct3dhlsl/dx-graphics-hlsl-struct |
