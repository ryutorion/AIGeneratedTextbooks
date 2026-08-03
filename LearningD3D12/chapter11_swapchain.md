# 第11章 スワップチェーンと画面クリア

**本章の終わりに、画面が青くなります。**

10 章かけてここまで来ました。デバイスを作り、コマンドの投入路を敷き、GPU との同期を確立しました。その全部が、この 1 枚の青い画面のためにありました。

そして本章は、**本書がもっとも「フルスクラッチらしく」なる章**でもあります。`d3dx12.h` を使えば、

```cpp
auto barrier = CD3DX12_RESOURCE_BARRIER::Transition(
    backBuffer, D3D12_RESOURCE_STATE_PRESENT,
    D3D12_RESOURCE_STATE_RENDER_TARGET);
```

の 1 行で済むところを、構造体の全フィールドを自分で埋めます。デスクリプタハンドルのオフセット計算も、`CD3DX12_CPU_DESCRIPTOR_HANDLE` に頼らず自分で書きます。

**面倒です。** しかしこの面倒を通ることで、「リソースバリアとは、実は 4 つの値を持つ構造体を並べているだけだ」「デスクリプタハンドルとは、実は生のアドレスだ」ということが腹に落ちます。第30章で Enhanced Barriers に移行するとき、第19章でデスクリプタアロケータを設計するとき、この理解が土台になります。

**本章のゴール**
スワップチェーンを作り、バックバッファに RTV を作り、リソースバリアを手で書いて、`ClearRenderTargetView` で画面を塗る。最後に、色を時間で変化させて「本当に毎フレーム描いている」ことを確認する。

---

## 11.1 `CreateSwapChainForHwnd` と FLIP モデル

### 11.1.1 スワップチェーンとは何か

**描いている途中の絵を人に見せてはいけません。**

画面に直接描くと、描画途中の状態がユーザーに見えてしまいます。上半分は新しいフレーム、下半分は古いフレーム —— これが「ティアリング(裂け)」です。

そこで、**見えていない場所に描いて、描き終わってから入れ替える**という方法をとります。

```
┌──────────────┐      ┌──────────────┐
│ フロントバッファ│      │ バックバッファ │
│  (画面に表示)  │◄────►│  (描画中)     │
└──────────────┘ 入替  └──────────────┘
```

この「入れ替わる複数のバッファの組」がスワップチェーンです。DXGI が管理します。

### 11.1.2 FLIP モデル

バッファの入れ替え方には歴史があります。

| 方式 | `DXGI_SWAP_EFFECT` | D3D12 で使えるか |
|---|---|---|
| BLT モデル(コピー) | `DISCARD` / `SEQUENTIAL` | **使えない** |
| FLIP モデル(付け替え) | `FLIP_SEQUENTIAL` | 使える |
| FLIP モデル(破棄あり) | **`FLIP_DISCARD`** | **使える。本書はこれ** |

**BLT モデル**は、バックバッファの内容をデスクトップ合成用のサーフェスに**コピー**していました。フル HD なら毎フレーム 8MB 前後の転送が発生します。

**FLIP モデル**は、コピーせず**表示先のポインタを付け替える**だけです。帯域を消費せず、条件が揃えばデスクトップ合成器(DWM)を経由せず直接ディスプレイへ出す「独立フリップ」も可能になり、遅延が下がります。

**Direct3D 12 では FLIP モデルが必須です。** BLT モデルの `DXGI_SWAP_EFFECT_DISCARD` を指定すると、スワップチェーンの生成が失敗します。

#### `FLIP_DISCARD` の重要な帰結

**`Present` した後、そのバッファの内容は未定義になります。**

「前のフレームの絵が残っているから、変わったところだけ描けばいい」という発想は使えません。**毎フレーム、画面全体を描き直す必要があります。**

本章で毎フレーム `ClearRenderTargetView` を呼ぶのは、これが理由です。

#### バックバッファの番号は循環しない

BLT モデルでは、バックバッファは常に 1 つで、インデックスを気にする必要がありませんでした。FLIP モデルでは複数のバッファが入れ替わります。

そして、**次にどのバッファに描くべきかは、こちらで計算してはいけません。**

```cpp
// ❌ 間違い。0, 1, 0, 1 と循環するとは限らない
m_frameIndex = (m_frameIndex + 1) % kBackBufferCount;

// ✅ 正しい。DXGI に聞く
m_frameIndex = m_swapChain->GetCurrentBackBufferIndex();
```

`IDXGISwapChain3::GetCurrentBackBufferIndex()` を使ってください。DXGI は状況に応じてバッファの順序を変えることがあります。

### 11.1.3 `DXGI_SWAP_CHAIN_DESC1` を埋める

```cpp
DXGI_SWAP_CHAIN_DESC1 desc{};              // ← {} を忘れない(第9章 9.3.3)
desc.Width              = width;
desc.Height             = height;
desc.Format             = DXGI_FORMAT_R8G8B8A8_UNORM;
desc.Stereo             = FALSE;
desc.SampleDesc.Count   = 1;
desc.SampleDesc.Quality = 0;
desc.BufferUsage        = DXGI_USAGE_RENDER_TARGET_OUTPUT;
desc.BufferCount        = kBackBufferCount;      // 2
desc.Scaling            = DXGI_SCALING_NONE;
desc.SwapEffect         = DXGI_SWAP_EFFECT_FLIP_DISCARD;
desc.AlphaMode          = DXGI_ALPHA_MODE_UNSPECIFIED;
desc.Flags              = tearingFlag;
```

**注意が必要なフィールドを 4 つ**説明します。

#### `Format`

**FLIP モデルのスワップチェーンは、`_SRGB` 形式のフォーマットを受け付けません。**

使えるのは主に次の 4 つです。

| フォーマット | 用途 |
|---|---|
| `DXGI_FORMAT_R8G8B8A8_UNORM` | **本書の既定** |
| `DXGI_FORMAT_B8G8R8A8_UNORM` | 同上(チャンネル順が違うだけ) |
| `DXGI_FORMAT_R10G10B10A2_UNORM` | HDR / 広色域 |
| `DXGI_FORMAT_R16G16B16A16_FLOAT` | HDR |

「では sRGB のガンマ補正はどうするのか」という疑問が出ます。答えは、**バッファは `_UNORM` で作り、レンダーターゲットビュー(RTV)を `_SRGB` で作る**というものです。ビューはリソースの解釈方法にすぎないので、これが可能です。

**第24章 24.4 節でこれを実装します。** 本章では素の `_UNORM` のまま進めます。色が「なんとなく暗い/明るい」と感じても、今は正常です。

#### `SampleDesc`

マルチサンプルアンチエイリアス(MSAA)の設定です。

**FLIP モデルのスワップチェーンは MSAA をサポートしません。** `Count` は必ず `1` です。

MSAA を使いたい場合は、別途 MSAA 付きのレンダーターゲットに描いてから、解決(リゾルブ)した結果をバックバッファにコピーします。第28章で扱います。

#### `BufferCount`

FLIP モデルでは **2 以上**が必須です。

本章では `2`(ダブルバッファリング)にします。**3 にすべきかどうかは第12章で検討します。** 現在の実装は毎フレーム GPU を待つ(第10章 10.4.3 節)ので、バッファを増やしても意味がありません。

#### `Scaling`

ウィンドウのサイズとバッファのサイズが食い違ったときの挙動です。

| 値 | 挙動 |
|---|---|
| `DXGI_SCALING_STRETCH` | 引き伸ばす |
| **`DXGI_SCALING_NONE`** | **引き伸ばさない(左上に原寸で表示)** |
| `DXGI_SCALING_ASPECT_RATIO_STRETCH` | 比率を保って引き伸ばす |

本書は `NONE` を使います。**リサイズ時にバッファも作り直す方針**(第12章)なので、引き伸ばしは不要です。

> **本章ではリサイズに対応しません**
>
> ウィンドウのサイズを変えると、**バッファのサイズが追従せず、画面の一部だけが塗られた状態になります。** これは第12章 12.4 節で `ResizeBuffers` を実装するまでの既知の挙動です。
>
> 本章の確認では、ウィンドウのサイズを変えないでください。

### 11.1.4 デバイスではなく「キュー」を渡す

**D3D12 で最も驚かれる箇所の一つです。**

```cpp
HRESULT CreateSwapChainForHwnd(
    IUnknown*                             pDevice,   // ← ここ
    HWND                                  hWnd,
    const DXGI_SWAP_CHAIN_DESC1*          pDesc,
    const DXGI_SWAP_CHAIN_FULLSCREEN_DESC* pFullscreenDesc,
    IDXGIOutput*                          pRestrictToOutput,
    IDXGISwapChain1**                     ppSwapChain);
```

第 1 引数の名前は `pDevice` です。しかし **Direct3D 12 では、ここに `ID3D12CommandQueue` を渡します。**

```cpp
ComPtr<IDXGISwapChain1> swapChain1;
HR_TRY(factory->CreateSwapChainForHwnd(
    queue,          // ← ID3D12Device ではなく ID3D12CommandQueue
    hwnd,
    &desc,
    nullptr,        // ウィンドウモード
    nullptr,
    &swapChain1));
```

理由は、**`Present` がどのキューの作業の後に実行されるべきかを、DXGI が知る必要があるから**です。D3D11 にはコンテキストが 1 つしかなかったのでデバイスで十分でしたが、D3D12 では複数のキューがありえます。

**デバイスを渡すと `E_INVALIDARG` になります。** エラーメッセージからは理由がわかりにくいので、覚えておいてください。

生成後、`IDXGISwapChain3` に問い合わせます(`GetCurrentBackBufferIndex` を使うため)。

```cpp
HR_TRY(swapChain1.As(&m_swapChain));   // ComPtr<IDXGISwapChain3>
```

### 11.1.5 Alt+Enter を無効化する

```cpp
HR_TRY(factory->MakeWindowAssociation(hwnd, DXGI_MWA_NO_ALT_ENTER));
```

DXGI は既定で、**Alt+Enter を押すと勝手にフルスクリーンへ切り替えます。**

これは親切に見えて厄介です。DXGI が独自にウィンドウのサイズと状態を変更するため、第5章で作った `WM_SIZE` の処理と衝突します。「なぜかリサイズ処理が二重に走る」「フルスクリーンから戻れない」といった問題の原因になります。

**無効化して、必要ならば自分で実装します。** フルスクリーンの扱いは付録 G にまとめます。

なお、`MakeWindowAssociation` は**スワップチェーンを作った後**に呼んでください。前に呼んでも効果がありません。

### 11.1.6 ティアリング許可(可変リフレッシュレート)

G-SYNC や FreeSync といった可変リフレッシュレート技術を使うには、**垂直同期を待たずに `Present` できる**必要があります。そのためのフラグです。

対応状況を問い合わせます。

```cpp
BOOL allowTearing = FALSE;
if (FAILED(factory->CheckFeatureSupport(
        DXGI_FEATURE_PRESENT_ALLOW_TEARING,
        &allowTearing, sizeof(allowTearing))))
{
    allowTearing = FALSE;
}

const UINT tearingFlag =
    allowTearing ? DXGI_SWAP_CHAIN_FLAG_ALLOW_TEARING : 0u;
```

**このフラグは、スワップチェーンの生成時に指定しなければなりません。** 後から付けることはできず、付けたければスワップチェーンを作り直すことになります。だから、使うかどうか未定でも**指定だけはしておきます。**

実際にティアリングを許可して `Present` するには、次の 2 つが**両方**必要です。

```cpp
swapChain->Present(0, DXGI_PRESENT_ALLOW_TEARING);
//                 ↑ SyncInterval は 0 でなければならない
```

`SyncInterval` が 1 以上のときに `DXGI_PRESENT_ALLOW_TEARING` を渡すと失敗します。

**本章では垂直同期あり(`Present(1, 0)`)で進めます。** 可変リフレッシュレートの実際の利用は付録 G で扱います。

---

## 11.2 デスクリプタヒープ

### 11.2.1 デスクリプタとは何か

バックバッファができました。しかし、**`ID3D12Resource*` をそのまま「ここに描け」と指定することはできません。**

GPU がリソースを扱うには、「このアドレスのメモリを、この形式の、この大きさの、こういう用途のものとして解釈せよ」という記述が必要です。**これがデスクリプタです。**

```
ID3D12Resource      デスクリプタ            GPU
(メモリの実体)  ←──  (解釈の指示書)  ←──  参照する
```

**重要な点:デスクリプタはリソースではありません。** 同じリソースに対して、異なる解釈のデスクリプタを複数作れます。11.1.3 節で触れた「`_UNORM` のバッファに `_SRGB` の RTV を作る」は、まさにこれです。

デスクリプタには種類があります。

| 種類 | 意味 | 略称 |
|---|---|---|
| Render Target View | 描画先として使う | **RTV** |
| Depth Stencil View | 深度バッファとして使う | DSV(第19章) |
| Shader Resource View | シェーダーから読む | SRV(第20章) |
| Constant Buffer View | 定数バッファとして読む | CBV(第18章) |
| Unordered Access View | シェーダーから読み書きする | UAV(第32章) |
| Sampler | サンプリング方法 | (第20章) |

**本章で使うのは RTV だけです。**

### 11.2.2 デスクリプタヒープ

デスクリプタは、**「デスクリプタヒープ」という配列の中に置かれます。** 単体で存在することはできません。

```
デスクリプタヒープ(RTV 用、要素数 2)
┌──────────┬──────────┐
│ RTV [0]  │ RTV [1]  │
└──────────┴──────────┘
     ↓           ↓
BackBuffer[0]  BackBuffer[1]
```

```cpp
D3D12_DESCRIPTOR_HEAP_DESC desc{};
desc.Type           = D3D12_DESCRIPTOR_HEAP_TYPE_RTV;
desc.NumDescriptors = kBackBufferCount;
desc.Flags          = D3D12_DESCRIPTOR_HEAP_FLAG_NONE;
desc.NodeMask       = 0;

ComPtr<ID3D12DescriptorHeap> rtvHeap;
HR_TRY(device->CreateDescriptorHeap(&desc, IID_PPV_ARGS(&rtvHeap)));
```

**4 つのフィールドしかありません。** 第9章のコマンドキューと同じ大きさです。

### 11.2.3 RTV ヒープはシェーダーから見えない

```cpp
desc.Flags = D3D12_DESCRIPTOR_HEAP_FLAG_NONE;
```

`D3D12_DESCRIPTOR_HEAP_FLAG_SHADER_VISIBLE` というフラグがありますが、**RTV と DSV のヒープには指定できません。** 指定すると生成が失敗します。

| ヒープの種類 | シェーダー可視にできるか |
|---|---|
| CBV_SRV_UAV | **できる** |
| SAMPLER | **できる** |
| RTV | できない |
| DSV | できない |

理由は用途にあります。RTV は「出力先を指定する」ためのもので、パイプラインの固定機能部(Output Merger)が使います。シェーダーのコードから直接読むものではありません。

**この区別は第19章でデスクリプタ管理を設計するときに効いてきます。** シェーダー可視ヒープには数やサイズの制約があり、扱いが違ってきます。

---

## 11.3 ハンドルの計算

### 11.3.1 増分サイズはハードウェア依存

ヒープの中の N 番目のデスクリプタを指すには、**アドレスを自分で計算します。**

```cpp
D3D12_CPU_DESCRIPTOR_HANDLE start =
    rtvHeap->GetCPUDescriptorHandleForHeapStart();
```

`D3D12_CPU_DESCRIPTOR_HANDLE` の中身は、驚くほど単純です。

```cpp
typedef struct D3D12_CPU_DESCRIPTOR_HANDLE {
    SIZE_T ptr;
} D3D12_CPU_DESCRIPTOR_HANDLE;
```

**メンバはアドレス 1 つだけです。** 型安全のためだけに構造体で包んであります。

N 番目を指すには、増分サイズを掛けて足します。

```cpp
const UINT increment =
    device->GetDescriptorHandleIncrementSize(D3D12_DESCRIPTOR_HEAP_TYPE_RTV);

D3D12_CPU_DESCRIPTOR_HANDLE handle = start;
handle.ptr += static_cast<SIZE_T>(index) * increment;
```

**増分サイズは、絶対にハードコードしてはいけません。**

| やってはいけないこと | 理由 |
|---|---|
| `handle.ptr += index * 32;` | 値は GPU/ドライバによって違う |
| `handle.ptr += index * sizeof(...);` | そもそも構造体のサイズとは無関係 |

`GetDescriptorHandleIncrementSize` は**ヒープの種類ごとに違う値**を返し、同じ種類でも GPU が違えば値が違います。**必ず実行時に問い合わせ、結果をキャッシュしてください。**

> **`static_cast<SIZE_T>` を忘れない**
>
> ```cpp
> handle.ptr += index * increment;              // ❌ 危険
> handle.ptr += static_cast<SIZE_T>(index) * increment;   // ✅
> ```
>
> `index` も `increment` も `UINT`(32 ビット)です。掛け算が 32 ビットで行われてから 64 ビットに拡張されるため、**大きなヒープでは桁あふれします。**
>
> RTV が 2 個なら問題になりませんが、第19章で数万個のデスクリプタを扱うようになると現実の問題になります。**習慣として今から書いておきます。**

### 11.3.2 `CD3DX12_CPU_DESCRIPTOR_HANDLE` を使わずに書く

`d3dx12.h` を使えば、こう書けます。

```cpp
// 本書では使わない
CD3DX12_CPU_DESCRIPTOR_HANDLE handle(start, index, increment);
```

**本書では自作します。** 第1章 1.3.1 節の方針通りです。

```cpp
// src/Graphics/D3D12Helpers.h
#pragma once
#include "std_import.h"

namespace Graphics
{
    //-----------------------------------------------------------
    // デスクリプタハンドルを index 個分ずらす。
    // CD3DX12_CPU_DESCRIPTOR_HANDLE の代替。
    //-----------------------------------------------------------
    [[nodiscard]] inline D3D12_CPU_DESCRIPTOR_HANDLE OffsetHandle(
        D3D12_CPU_DESCRIPTOR_HANDLE handle,
        UINT index,
        UINT incrementSize) noexcept
    {
        handle.ptr += static_cast<SIZE_T>(index) * incrementSize;
        return handle;
    }

    [[nodiscard]] inline D3D12_GPU_DESCRIPTOR_HANDLE OffsetHandle(
        D3D12_GPU_DESCRIPTOR_HANDLE handle,
        UINT index,
        UINT incrementSize) noexcept
    {
        handle.ptr += static_cast<std::uint64_t>(index) * incrementSize;
        return handle;
    }
}
```

**この 2 つの関数が、`CD3DX12_CPU_DESCRIPTOR_HANDLE` の代わりを務めます。**

`ptr` の型が CPU 側は `SIZE_T`、GPU 側は `UINT64` と異なる点に注意してください。x64 では同じ大きさですが、意味が違います。CPU ハンドルは CPU から書き込むためのアドレス、GPU ハンドルは GPU が参照するためのアドレスです。

> **本書のヘルパーは付録 A にまとまります**
>
> 以後、`d3dx12.h` の代替として書いた関数はすべて `D3D12Helpers.h` に集めます。付録 A に、`d3dx12.h` の何に対応するかの一覧を載せます。**最終的に 20 個ほどになりますが、それが `d3dx12.h` を使わない代償のすべてです。**

---

## 11.4 バックバッファの取得と RTV 作成

```cpp
for (UINT i = 0; i < kBackBufferCount; ++i)
{
    //--- ① スワップチェーンからバッファを取り出す ---
    HR_TRY(m_swapChain->GetBuffer(i, IID_PPV_ARGS(&m_backBuffers[i])));

    Core::SetDebugNameF(m_backBuffers[i].Get(), L"BackBuffer[{}]", i);

    //--- ② そのバッファを指す RTV を作る ---
    const D3D12_CPU_DESCRIPTOR_HANDLE handle =
        OffsetHandle(m_rtvHeapStart, i, m_rtvIncrement);

    device->CreateRenderTargetView(
        m_backBuffers[i].Get(),
        nullptr,            // 記述子(後述)
        handle);
}
```

**3 つの補足があります。**

#### `GetBuffer` は参照カウントを増やす

`ComPtr` で受けているので自動的に管理されます。ただし、**第12章で `ResizeBuffers` を呼ぶ前には、これらをすべて解放しなければなりません。** バックバッファへの参照が 1 つでも残っていると `ResizeBuffers` は失敗します。

#### `CreateRenderTargetView` の第 2 引数

`D3D12_RENDER_TARGET_VIEW_DESC*` です。`nullptr` を渡すと、**リソース自身のフォーマットと次元がそのまま使われます。**

11.1.3 節で触れた「`_UNORM` のバッファに `_SRGB` の RTV を作る」をやりたい場合は、ここに記述子を渡します(第24章)。

#### `CreateRenderTargetView` は失敗を返さない

戻り値は `void` です。**引数が不正でもその場ではわかりません。** デバッグレイヤーが指摘してくれるので、第7章の `SetBreakOnSeverity` が効いてきます。

**名前を付けるのを忘れないでください。** `BackBuffer[0]` という名前は、この先ずっとデバッグレイヤーの警告や Nsight の画面に現れます。第31章で Aftermath のダンプを読むときにも役立ちます(第6章 6.5.2 節)。

---

## 11.5 リソースバリアを手で書く

### 11.5.1 状態は自分で覚えておく

**D3D12 のリソースには「状態」があります。**

| 状態 | 意味 |
|---|---|
| `D3D12_RESOURCE_STATE_RENDER_TARGET` | 描画先として書き込み中 |
| `D3D12_RESOURCE_STATE_PRESENT` | 表示可能 |
| `D3D12_RESOURCE_STATE_PIXEL_SHADER_RESOURCE` | シェーダーから読み取り中 |
| `D3D12_RESOURCE_STATE_COPY_DEST` | コピー先 |
| …ほか多数 | |

GPU は、状態に応じて内部のキャッシュ構成やメモリ圧縮の仕方を変えます。**用途を変えるときには、明示的に「遷移」を宣言しなければなりません。** それがリソースバリアです。

**そして最も重要な点:D3D12 は、リソースが今どの状態かを教えてくれません。**

```cpp
// ❌ こんな API は存在しない
auto state = resource->GetCurrentState();
```

**現在の状態は、アプリケーションが自分で覚えておく必要があります。** バリアには「遷移前の状態」を書く欄があり、そこに実際と違う値を書くと、デバッグレイヤーが指摘します。実機では絵が壊れたり、GPU がハングしたりします。

この「状態を自分で追跡する」という負担が、D3D12 でもっとも面倒な部分の一つです。第30章で導入する Enhanced Barriers は、まさにこの設計を見直したものです。

**本章では、バックバッファの状態が 2 つしかないので手で追えます。**

```
PRESENT ──バリア──► RENDER_TARGET ──描画──► ──バリア──► PRESENT
```

### 11.5.2 `D3D12_RESOURCE_BARRIER` の全フィールド

```cpp
typedef struct D3D12_RESOURCE_BARRIER {
    D3D12_RESOURCE_BARRIER_TYPE  Type;
    D3D12_RESOURCE_BARRIER_FLAGS Flags;
    union {
        D3D12_RESOURCE_TRANSITION_BARRIER Transition;
        D3D12_RESOURCE_ALIASING_BARRIER   Aliasing;
        D3D12_RESOURCE_UAV_BARRIER        UAV;
    };
} D3D12_RESOURCE_BARRIER;
```

**共用体です。** `Type` で指定した種類に対応するメンバだけが有効です。

| `Type` | 有効なメンバ | 用途 |
|---|---|---|
| `TRANSITION` | `Transition` | **状態遷移。本章で使う** |
| `ALIASING` | `Aliasing` | 同じメモリを別リソースが使い始めるとき(第19章) |
| `UAV` | `UAV` | UAV への書き込み完了を待つ(第32章) |

`Flags` には 3 つの値があります。

| 値 | 意味 |
|---|---|
| `NONE` | **通常。本章で使う** |
| `BEGIN_ONLY` | 分割バリアの開始(第30章) |
| `END_ONLY` | 分割バリアの終了(第30章) |

そして遷移バリアの中身です。

```cpp
typedef struct D3D12_RESOURCE_TRANSITION_BARRIER {
    ID3D12Resource*       pResource;
    UINT                  Subresource;
    D3D12_RESOURCE_STATES StateBefore;
    D3D12_RESOURCE_STATES StateAfter;
} D3D12_RESOURCE_TRANSITION_BARRIER;
```

`Subresource` は、テクスチャのミップレベルや配列要素を個別に遷移させるための指定です。全部まとめて遷移させるなら定数を使います。

```cpp
D3D12_RESOURCE_BARRIER_ALL_SUBRESOURCES   // = 0xffffffff
```

バックバッファはミップもない単純な 2D テクスチャなので、これで構いません。

### 11.5.3 `PRESENT` と `COMMON` は同じ値

```cpp
D3D12_RESOURCE_STATE_COMMON  = 0
D3D12_RESOURCE_STATE_PRESENT = 0     // ← 同じ
```

**この 2 つは別名です。値はどちらも 0 です。**

`COMMON` は「特別な状態にない、汎用の状態」を意味し、`PRESENT` は「表示できる状態」を意味します。**スワップチェーンのバックバッファにおいては、両者は同一です。**

どちらを書いても動きますが、**意図が伝わるほうを選んでください。** 本書ではバックバッファに対しては `PRESENT` と書きます。

### 11.5.4 自作ヘルパー `MakeTransitionBarrier`

素直に書くと、こうなります。

```cpp
D3D12_RESOURCE_BARRIER barrier{};
barrier.Type  = D3D12_RESOURCE_BARRIER_TYPE_TRANSITION;
barrier.Flags = D3D12_RESOURCE_BARRIER_FLAG_NONE;
barrier.Transition.pResource   = backBuffer;
barrier.Transition.Subresource = D3D12_RESOURCE_BARRIER_ALL_SUBRESOURCES;
barrier.Transition.StateBefore = D3D12_RESOURCE_STATE_PRESENT;
barrier.Transition.StateAfter  = D3D12_RESOURCE_STATE_RENDER_TARGET;

commandList->ResourceBarrier(1, &barrier);
```

**7 行です。** これを毎回書くのは現実的ではありません。ヘルパーを作ります。

```cpp
// src/Graphics/D3D12Helpers.h に追加

//---------------------------------------------------------------
// 状態遷移バリアを作る。
// CD3DX12_RESOURCE_BARRIER::Transition() の代替。
//---------------------------------------------------------------
[[nodiscard]] inline D3D12_RESOURCE_BARRIER MakeTransitionBarrier(
    ID3D12Resource*              resource,
    D3D12_RESOURCE_STATES        before,
    D3D12_RESOURCE_STATES        after,
    UINT                         subresource
        = D3D12_RESOURCE_BARRIER_ALL_SUBRESOURCES,
    D3D12_RESOURCE_BARRIER_FLAGS flags
        = D3D12_RESOURCE_BARRIER_FLAG_NONE) noexcept
{
    D3D12_RESOURCE_BARRIER barrier{};
    barrier.Type  = D3D12_RESOURCE_BARRIER_TYPE_TRANSITION;
    barrier.Flags = flags;
    barrier.Transition.pResource   = resource;
    barrier.Transition.Subresource = subresource;
    barrier.Transition.StateBefore = before;
    barrier.Transition.StateAfter  = after;
    return barrier;
}
```

使い方:

```cpp
const auto toRenderTarget = MakeTransitionBarrier(
    m_backBuffers[index].Get(),
    D3D12_RESOURCE_STATE_PRESENT,
    D3D12_RESOURCE_STATE_RENDER_TARGET);

commandList->ResourceBarrier(1, &toRenderTarget);
```

**`d3dx12.h` の 1 行と、ほぼ同じ書き心地になりました。** 違いは、その 1 行の中身を我々が全部知っていることです。

> **バリアはまとめて発行する**
>
> ```cpp
> // △ 2 回に分けている
> commandList->ResourceBarrier(1, &barrierA);
> commandList->ResourceBarrier(1, &barrierB);
>
> // ✅ まとめる
> const D3D12_RESOURCE_BARRIER barriers[] = { barrierA, barrierB };
> commandList->ResourceBarrier(2, barriers);
> ```
>
> バリアは GPU のパイプラインを一時的に止める操作です。**まとめて発行すれば、停止は 1 回で済みます。** 本章では 1 つずつですが、第26章のポストエフェクトあたりから効いてきます。

---

## 11.6 `ClearRenderTargetView` と `Present`

### 11.6.1 1 フレームの流れ

ここまでの部品が揃いました。全体の流れを整理します。

```
① 前フレームの GPU 完了を待つ           (第10章)
② アロケータとコマンドリストを Reset     (第9章)
③ 現在のバックバッファ番号を取得         GetCurrentBackBufferIndex
④ バリア: PRESENT → RENDER_TARGET       (11.5)
⑤ レンダーターゲットを設定               OMSetRenderTargets
⑥ 塗りつぶす                             ClearRenderTargetView
⑦ バリア: RENDER_TARGET → PRESENT       (11.5)
⑧ コマンドリストを Close                 (第9章)
⑨ 投入                                   ExecuteCommandLists
⑩ 表示                                   Present
⑪ フェンスに Signal                      (第10章)
```

**この 11 ステップが、本書の最後まで骨格として残ります。** 第13章では ⑥ の後に描画コマンドが増え、第26章では ④〜⑦ が複数回繰り返されるようになります。

### 11.6.2 クリアする

```cpp
const D3D12_CPU_DESCRIPTOR_HANDLE rtv =
    OffsetHandle(m_rtvHeapStart, index, m_rtvIncrement);

//--- レンダーターゲットを設定 ---
commandList->OMSetRenderTargets(
    1,          // レンダーターゲットの数
    &rtv,       // ハンドルの配列
    FALSE,      // 連続した範囲かどうか
    nullptr);   // 深度ステンシルビュー(第19章)

//--- 塗りつぶす ---
const float clearColor[4] = { 0.1f, 0.2f, 0.4f, 1.0f };
commandList->ClearRenderTargetView(rtv, clearColor, 0, nullptr);
```

`OMSetRenderTargets` の第 3 引数は、**「複数の RTV を渡すとき、それらがヒープ上で連続しているか」**を示します。`FALSE` なら、第 2 引数はハンドルの配列です。1 個しかないので、どちらでも同じです。

`ClearRenderTargetView` の後ろ 2 つは、クリアする矩形の指定です。`0, nullptr` で全面をクリアします。

**色は RGBA の 0.0〜1.0 です。** `{ 0.1f, 0.2f, 0.4f, 1.0f }` は落ち着いた青になります。

> **なぜ真っ青(0, 0, 1)にしないのか**
>
> 好みの問題ですが、**真っ黒や真っ白、原色は避けたほうが便利です。** 「描画されていない領域」と「意図的に塗った領域」が区別できなくなるからです。
>
> 少し濁った色にしておくと、第13章で三角形が出たとき、背景と図形が明確に分かれて見えます。

### 11.6.3 `Present` の戻り値を確認する

```cpp
const HRESULT hr = m_swapChain->Present(1, 0);
```

| 引数 | 意味 |
|---|---|
| 第 1 (`SyncInterval`) | `0` = 待たない、`1` = 垂直同期を 1 回待つ |
| 第 2 (`Flags`) | `DXGI_PRESENT_ALLOW_TEARING` など |

**`Present` の戻り値は必ず確認してください。**

D3D12 において、**デバイスロストが最初に表面化する場所は、たいてい `Present` です。** `ExecuteCommandLists` は `void` を返し(第9章 9.5.3 節)、記録メソッドも `void` を返します。どこかで GPU が死んでも、それが伝わってくるのはここです。

```cpp
const HRESULT hr = m_swapChain->Present(1, 0);

if (hr == DXGI_ERROR_DEVICE_REMOVED || hr == DXGI_ERROR_DEVICE_RESET)
{
    // 第10章 10.5.4 節の処理へ
    OnDeviceLost(device->GetDeviceRemovedReason());
    return std::unexpected(Core::MakeError(hr, L"Present: device lost"));
}
HR_TRY(hr);
```

> **`DXGI_STATUS_OCCLUDED` について**
>
> ウィンドウが完全に隠れているとき、`Present` は `DXGI_STATUS_OCCLUDED` を返します。**これは成功コードです**(`SUCCEEDED` が真になります)。
>
> 第6章 6.3.1 節で「`hr == S_OK` で判定してはいけない」と書いた理由が、ここでも当てはまります。`S_OK` と比較していると、ウィンドウを隠しただけでエラー扱いになります。

---

## 11.7 実装

### 11.7.1 `SwapChain` クラス

```cpp
// src/Graphics/SwapChain.h
#pragma once
#include "std_import.h"
#include "Core/Error.h"

namespace Graphics
{
    inline constexpr UINT kBackBufferCount = 2;   // 第12章で再検討する

    class SwapChain
    {
    public:
        SwapChain() = default;
        ~SwapChain() = default;

        SwapChain(const SwapChain&)            = delete;
        SwapChain& operator=(const SwapChain&) = delete;

        Core::Status Initialize(ID3D12Device* device,
                                IDXGIFactory6* factory,
                                ID3D12CommandQueue* queue,
                                HWND hwnd,
                                UINT width, UINT height);
        void Shutdown();

        UINT CurrentIndex() const
        {
            return m_swapChain->GetCurrentBackBufferIndex();
        }

        ID3D12Resource* CurrentBackBuffer() const
        {
            return m_backBuffers[CurrentIndex()].Get();
        }

        D3D12_CPU_DESCRIPTOR_HANDLE CurrentRtv() const;

        Core::Status Present(ID3D12Device* device, UINT syncInterval = 1);

    private:
        Core::Status CreateRenderTargetViews(ID3D12Device* device);

        Microsoft::WRL::ComPtr<IDXGISwapChain3>       m_swapChain;
        Microsoft::WRL::ComPtr<ID3D12DescriptorHeap>  m_rtvHeap;
        Microsoft::WRL::ComPtr<ID3D12Resource>        m_backBuffers[kBackBufferCount];

        D3D12_CPU_DESCRIPTOR_HANDLE m_rtvHeapStart{};
        UINT m_rtvIncrement = 0;
        UINT m_presentFlags = 0;
        bool m_allowTearing = false;
    };
}
```

### 11.7.2 初期化

```cpp
// src/Graphics/SwapChain.cpp(抜粋)

Core::Status SwapChain::Initialize(
    ID3D12Device* device,
    IDXGIFactory6* factory,
    ID3D12CommandQueue* queue,
    HWND hwnd,
    UINT width, UINT height)
{
    //--- ① ティアリング対応の確認(11.1.6) ---
    BOOL allowTearing = FALSE;
    if (FAILED(factory->CheckFeatureSupport(
            DXGI_FEATURE_PRESENT_ALLOW_TEARING,
            &allowTearing, sizeof(allowTearing))))
    {
        allowTearing = FALSE;
    }
    m_allowTearing = (allowTearing != FALSE);

    LOG_INFO(L"allow tearing : {}", m_allowTearing ? L"yes" : L"no");

    //--- ② スワップチェーンの記述子(11.1.3) ---
    DXGI_SWAP_CHAIN_DESC1 desc{};
    desc.Width              = width;
    desc.Height             = height;
    desc.Format             = DXGI_FORMAT_R8G8B8A8_UNORM;
    desc.Stereo             = FALSE;
    desc.SampleDesc.Count   = 1;          // FLIP モデルは MSAA 不可
    desc.SampleDesc.Quality = 0;
    desc.BufferUsage        = DXGI_USAGE_RENDER_TARGET_OUTPUT;
    desc.BufferCount        = kBackBufferCount;
    desc.Scaling            = DXGI_SCALING_NONE;
    desc.SwapEffect         = DXGI_SWAP_EFFECT_FLIP_DISCARD;
    desc.AlphaMode          = DXGI_ALPHA_MODE_UNSPECIFIED;
    desc.Flags              = m_allowTearing
                            ? DXGI_SWAP_CHAIN_FLAG_ALLOW_TEARING : 0u;

    //--- ③ 生成。第1引数はキュー(11.1.4) ---
    Microsoft::WRL::ComPtr<IDXGISwapChain1> swapChain1;
    HR_TRY(factory->CreateSwapChainForHwnd(
        queue,          // ← ID3D12Device ではない
        hwnd,
        &desc,
        nullptr,        // ウィンドウモード
        nullptr,
        &swapChain1));

    HR_TRY(swapChain1.As(&m_swapChain));

    //--- ④ Alt+Enter を無効化(11.1.5) ---
    HR_TRY(factory->MakeWindowAssociation(hwnd, DXGI_MWA_NO_ALT_ENTER));

    //--- ⑤ RTV ヒープ(11.2.2) ---
    D3D12_DESCRIPTOR_HEAP_DESC heapDesc{};
    heapDesc.Type           = D3D12_DESCRIPTOR_HEAP_TYPE_RTV;
    heapDesc.NumDescriptors = kBackBufferCount;
    heapDesc.Flags          = D3D12_DESCRIPTOR_HEAP_FLAG_NONE;
    heapDesc.NodeMask       = 0;

    HR_TRY(device->CreateDescriptorHeap(&heapDesc, IID_PPV_ARGS(&m_rtvHeap)));
    Core::SetDebugName(m_rtvHeap.Get(), L"RtvHeap");

    m_rtvHeapStart = m_rtvHeap->GetCPUDescriptorHandleForHeapStart();
    m_rtvIncrement = device->GetDescriptorHandleIncrementSize(
        D3D12_DESCRIPTOR_HEAP_TYPE_RTV);

    LOG_INFO(L"RTV descriptor size : {} bytes", m_rtvIncrement);

    //--- ⑥ バックバッファと RTV(11.4) ---
    if (auto r = CreateRenderTargetViews(device); !r)
    {
        return r;
    }

    LOG_INFO(L"swap chain created: {} x {}, {} buffers",
             width, height, kBackBufferCount);
    return {};
}

Core::Status SwapChain::CreateRenderTargetViews(ID3D12Device* device)
{
    for (UINT i = 0; i < kBackBufferCount; ++i)
    {
        HR_TRY(m_swapChain->GetBuffer(i, IID_PPV_ARGS(&m_backBuffers[i])));
        Core::SetDebugNameF(m_backBuffers[i].Get(), L"BackBuffer[{}]", i);

        device->CreateRenderTargetView(
            m_backBuffers[i].Get(),
            nullptr,
            OffsetHandle(m_rtvHeapStart, i, m_rtvIncrement));
    }
    return {};
}

D3D12_CPU_DESCRIPTOR_HANDLE SwapChain::CurrentRtv() const
{
    return OffsetHandle(m_rtvHeapStart, CurrentIndex(), m_rtvIncrement);
}
```

### 11.7.3 フレームの描画

```cpp
Core::Status RenderFrame(ID3D12Device* device)
{
    //--- ① 前フレームの完了を待つ(第10章) ---
    if (auto r = Graphics::WaitForGpuIdle(g_queue.Get(), g_fence); !r)
    {
        return r;
    }

    //--- ② 記録の準備(第9章) ---
    HR_TRY(g_allocator->Reset());
    HR_TRY(g_commandList->Reset(g_allocator.Get(), nullptr));

    //--- ③ 今回のバックバッファ ---
    ID3D12Resource* backBuffer = g_swapChain.CurrentBackBuffer();
    const D3D12_CPU_DESCRIPTOR_HANDLE rtv = g_swapChain.CurrentRtv();

    //--- ④ PRESENT → RENDER_TARGET ---
    {
        const auto barrier = Graphics::MakeTransitionBarrier(
            backBuffer,
            D3D12_RESOURCE_STATE_PRESENT,
            D3D12_RESOURCE_STATE_RENDER_TARGET);
        g_commandList->ResourceBarrier(1, &barrier);
    }

    //--- ⑤⑥ 設定してクリア ---
    g_commandList->OMSetRenderTargets(1, &rtv, FALSE, nullptr);

    const float clearColor[4] = { 0.1f, 0.2f, 0.4f, 1.0f };
    g_commandList->ClearRenderTargetView(rtv, clearColor, 0, nullptr);

    // 第13章から、ここに描画コマンドが入る

    //--- ⑦ RENDER_TARGET → PRESENT ---
    {
        const auto barrier = Graphics::MakeTransitionBarrier(
            backBuffer,
            D3D12_RESOURCE_STATE_RENDER_TARGET,
            D3D12_RESOURCE_STATE_PRESENT);
        g_commandList->ResourceBarrier(1, &barrier);
    }

    //--- ⑧⑨ 記録終了と投入 ---
    HR_TRY(g_commandList->Close());
    g_queue.Execute(g_commandList.Get());

    //--- ⑩ 表示 ---
    if (auto r = g_swapChain.Present(device, 1); !r)
    {
        return r;
    }

    //--- ⑪ フェンスに Signal ---
    auto value = g_fence.Signal(g_queue.Get());
    if (!value)
    {
        return std::unexpected(value.error());
    }

    return {};
}
```

**④ と ⑦ のバリアが対になっている**ことを確認してください。片方を書き忘れると、次のフレームで「状態が一致しない」とデバッグレイヤーに指摘されます。

---

## ✅ 本章のゴール:画面が青一色になる

### Step 1:実行する

`F5` で実行します。

**ウィンドウのクライアント領域が、落ち着いた青(RGB 約 26, 51, 102)で塗られます。**

```
[Info ] SwapChain.cpp(30): allow tearing : yes
[Info ] SwapChain.cpp(78): RTV descriptor size : 32 bytes
[Info ] SwapChain.cpp(92): swap chain created: 1280 x 720, 2 buffers
```

**`RTV descriptor size` の値に注目してください。** 11.3.1 節で「ハードウェア依存だからハードコードするな」と書いた、その実測値です。NVIDIA では 32 バイトになることが多いですが、**この数字を信じてコードに書いてはいけません。**

### Step 2:デバッグレイヤーが黙っていることを確認する

第7章で `SetBreakOnSeverity(WARNING, TRUE)` を設定しました。**何も出ず、止まらないこと**を確認してください。

### Step 3:バリアをわざと間違える

**状態追跡が本当に検証されているかを確かめます。**

④ のバリアの `StateBefore` を、わざと間違えてみてください。

```cpp
const auto barrier = Graphics::MakeTransitionBarrier(
    backBuffer,
    D3D12_RESOURCE_STATE_COPY_DEST,      // ❌ 実際は PRESENT
    D3D12_RESOURCE_STATE_RENDER_TARGET);
```

デバッガが停止し、次のような指摘が出ます。

```
[Error] Log.cpp(60): [D3D12] (id 527) D3D12 ERROR:
  ID3D12CommandList::ResourceBarrier: Before state (0x400: D3D12_RESOURCE_STATE_COPY_DEST)
  of resource (0x...:'BackBuffer[0]') doesn't match with the state (0x0:
  D3D12_RESOURCE_STATE_[COMMON|PRESENT]) ...
```

**`'BackBuffer[0]'` という名前が出ていることに注目してください。** 第6章 6.5 節で名前付けを習慣にした成果です。名前がなければ `0x000001F2A4B3C120` という数字だけで、どのバッファか特定できません。

**確認したら元に戻してください。**

### Step 4:片方のバリアを消してみる

⑦ のバリア(`RENDER_TARGET → PRESENT`)をコメントアウトします。

1 フレーム目は通りますが、**2 フレーム目の ④ で状態が一致せず**、デバッグレイヤーが指摘します。バックバッファが `RENDER_TARGET` のまま残っているからです。

**バリアが対でなければならない理由**が、実際の挙動として確認できます。

**確認したら元に戻してください。**

---

## 11.8 時間で色を変える

**青一色の画面では、本当に毎フレーム描かれているのか判断できません。** 1 回描いて止まっていても同じに見えます。

色を時間で変化させて確かめます。

```cpp
namespace
{
    std::chrono::steady_clock::time_point g_startTime =
        std::chrono::steady_clock::now();

    void GetAnimatedColor(float (&out)[4])
    {
        const auto now = std::chrono::steady_clock::now();
        const float t = std::chrono::duration<float>(now - g_startTime).count();

        // 位相をずらした 3 つの正弦波で色を作る
        constexpr float kTau = 6.283185307f;
        out[0] = 0.5f + 0.5f * std::sin(t * 0.7f);
        out[1] = 0.5f + 0.5f * std::sin(t * 0.7f + kTau / 3.0f);
        out[2] = 0.5f + 0.5f * std::sin(t * 0.7f + kTau * 2.0f / 3.0f);
        out[3] = 1.0f;
    }
}
```

`RenderFrame` の ⑥ を差し替えます。

```cpp
float clearColor[4]{};
GetAnimatedColor(clearColor);
g_commandList->ClearRenderTargetView(rtv, clearColor, 0, nullptr);
```

**画面の色がゆっくりと循環すれば成功です。**

- 色が滑らかに変化する → 毎フレーム描画され、`Present` が機能している
- 色が変わらない → ループが回っていない
- 色がカクつく → 垂直同期の設定を確認(`Present(1, 0)`)

> **`std::sin` はどこから来たのか**
>
> `import std;` に含まれています。第3章で決めた通り、`<cmath>` を `#include` する必要はありません。
>
> ただし `std::sin` であって `::sin` ではない点に注意してください。名前付きモジュールの `std` は、グローバル名前空間の C 関数を提供しません(第6章 6.3.4 節)。

---

### 本章の達成状態

- [ ] `CreateSwapChainForHwnd` の第 1 引数にコマンドキューを渡した
- [ ] `DXGI_SWAP_EFFECT_FLIP_DISCARD` を指定した
- [ ] `MakeWindowAssociation` で Alt+Enter を無効化した
- [ ] ティアリング対応を問い合わせ、フラグを設定した
- [ ] RTV ヒープを `SHADER_VISIBLE` なしで作った
- [ ] `GetDescriptorHandleIncrementSize` の値を実行時に取得している
- [ ] `OffsetHandle` を自作した(`SIZE_T` へのキャストつき)
- [ ] `MakeTransitionBarrier` を自作した
- [ ] バリアが PRESENT ↔ RENDER_TARGET の対になっている
- [ ] `Present` の戻り値を確認し、デバイスロストを処理している
- [ ] バックバッファに `BackBuffer[n]` という名前を付けた
- [ ] **画面が青くなった**
- [ ] 色が時間で変化することを確認した
- [ ] Step 3 / Step 4 でデバッグレイヤーが検出することを確認した

---

## トラブルシューティング

| 症状 | 原因 | 対処 |
|---|---|---|
| `CreateSwapChainForHwnd` が `E_INVALIDARG` | 第 1 引数にデバイスを渡した | キューを渡す(11.1.4) |
| 同上 | `DXGI_SWAP_EFFECT_DISCARD` を指定した | FLIP モデルにする(11.1.2) |
| 同上 | `SampleDesc.Count` が 1 でない | FLIP は MSAA 不可(11.1.3) |
| 同上 | `BufferCount` が 1 | 2 以上にする |
| `CreateDescriptorHeap` が失敗 | RTV に `SHADER_VISIBLE` を指定した | `NONE` にする(11.2.3) |
| 画面が真っ黒のまま | バリアの向きが逆 | 11.5 節を確認 |
| 同上 | `Present` を呼んでいない | 11.6.3 |
| 2 フレーム目で状態不一致 | 戻すバリアがない | 対で書く(Step 4) |
| RTV の位置がずれる | 増分サイズをハードコードした | 11.3.1 |
| 大きなヒープで異常なアドレス | `SIZE_T` へのキャスト忘れ | 11.3.1 |
| リサイズすると絵が崩れる | **第12章で対応する** | 今は正常 |
| Alt+Enter でおかしくなる | `MakeWindowAssociation` 未呼び出し | 11.1.5 |
| 色が変化しない | ループが回っていない | 第10章 10.6 節を確認 |
| ウィンドウを隠すとエラー | `DXGI_STATUS_OCCLUDED` を失敗扱い | `SUCCEEDED` で判定(11.6.3) |

---

## まとめ

**1. D3D12 では FLIP モデルが必須。**
`FLIP_DISCARD` では `Present` 後のバッファ内容が未定義になるため、毎フレーム全画面を描き直します。バックバッファの番号は `GetCurrentBackBufferIndex()` に聞きます。

**2. スワップチェーンにはキューを渡す。**
`CreateSwapChainForHwnd` の第 1 引数は `pDevice` という名前ですが、D3D12 では `ID3D12CommandQueue` です。

**3. デスクリプタはリソースではない。**
「このメモリをこう解釈せよ」という指示書です。同じリソースに複数の解釈を作れます。

**4. デスクリプタの増分サイズはハードウェア依存。**
必ず `GetDescriptorHandleIncrementSize` で問い合わせます。そして掛け算の前に `SIZE_T` へキャストします。

**5. リソースの状態は自分で覚える。**
D3D12 は現在の状態を教えてくれません。バリアの `StateBefore` が実際と違えば、デバッグレイヤーが指摘し、実機では絵が壊れます。

**6. `Present` の戻り値を確認する。**
デバイスロストが最初に表面化する場所です。`DXGI_STATUS_OCCLUDED` は成功コードである点にも注意してください。

**7. 自作ヘルパーが 2 つ増えた。**
`OffsetHandle` と `MakeTransitionBarrier`。`d3dx12.h` を使わない代償は、最終的にこの程度のファイル 1 枚に収まります(付録 A)。

次章では、このフレームループを正しく回します。ダブル/トリプルバッファリング、フレームごとのアロケータ、そして第10章 10.4.3 節で先送りにした「毎フレーム全部待つのは遅い」問題の解決。あわせて、本章で対応しなかったウィンドウのリサイズも扱います。

---

## 参考リンク

| 内容 | URL |
|---|---|
| DXGI フリップモデル | https://learn.microsoft.com/ja-jp/windows/win32/direct3ddxgi/dxgi-flip-model |
| `IDXGIFactory2::CreateSwapChainForHwnd` | https://learn.microsoft.com/ja-jp/windows/win32/api/dxgi1_2/nf-dxgi1_2-idxgifactory2-createswapchainforhwnd |
| `DXGI_SWAP_CHAIN_DESC1` | https://learn.microsoft.com/ja-jp/windows/win32/api/dxgi1_2/ns-dxgi1_2-dxgi_swap_chain_desc1 |
| デスクリプタヒープ | https://learn.microsoft.com/ja-jp/windows/win32/direct3d12/descriptor-heaps |
| リソースバリアと状態遷移 | https://learn.microsoft.com/ja-jp/windows/win32/direct3d12/using-resource-barriers-to-synchronize-resource-states-in-direct3d-12 |
| `D3D12_RESOURCE_STATES` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/ne-d3d12-d3d12_resource_states |
| 可変リフレッシュレートへの対応 | https://learn.microsoft.com/ja-jp/windows/win32/direct3ddxgi/variable-refresh-rate-displays |
