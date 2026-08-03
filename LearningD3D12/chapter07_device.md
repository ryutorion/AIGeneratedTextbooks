# 第7章 デバッグレイヤーとデバイスの生成

ここから Direct3D 12 が始まります。

本章でやることは 4 つです。**デバッグレイヤーを有効にし、NVIDIA のアダプタを確実に選び、デバイスを作り、そのデバイスに何ができるかを問い合わせる。** どれも一度書いたら以後ほとんど触らないコードですが、順序を間違えると静かに効かなくなる箇所がいくつもあります。

とりわけ強調したいのは **7.1 節のデバッグレイヤー**です。第4章で Agility SDK が「黙って失敗する」と書きました。デバッグレイヤーも同じ性質を持っています。`D3D12CreateDevice` の**後**に有効化しようとしても、何のエラーも出ないまま無効のままになります。そして本書のスタイル —— 構造体を手で埋める —— では、デバッグレイヤーは最大の味方です。これを失った状態で第11章以降に進むのは、目隠しで配線するようなものです。

本章の終わりには、第2章 2.1.2 節の機能対応表と、**自分の環境で実際に取得した値**を突き合わせられるようになります。

**本章のゴール**
GPU 名・VRAM 量・機能レベル・シェーダーモデル・対応機能の一覧が、起動時にログへ出力される。デバッグレイヤーのエラーで自動的にブレークする。

---

## 7.1 `D3D12GetDebugInterface` と EnableDebugLayer

### 7.1.1 順序が絶対

```cpp
ComPtr<ID3D12Debug> debug;
if (SUCCEEDED(::D3D12GetDebugInterface(IID_PPV_ARGS(&debug))))
{
    debug->EnableDebugLayer();
}
```

**この 4 行は、`D3D12CreateDevice` より前になければなりません。**

デバッグレイヤーは、デバイス生成時に「検証つきの実装」を差し込むことで機能します。デバイスができた後で有効化しても、そのデバイスには適用されません。

そして —— **エラーは出ません。** `EnableDebugLayer()` は戻り値すら返さない `void` 関数です。呼んだつもりで効いていない、という状態が成立します。

> **効いているかを確認する方法**
>
> 一番確実なのは、**わざとエラーを起こすこと**です。第11章で最初のリソースを作る際、意図的に不正な値を渡してみてください。デバッグレイヤーが有効なら、「出力」ウィンドウに具体的な指摘が出ます。無効なら `E_INVALIDARG` だけです。
>
> 本章では 7.2 節で、モジュールの読み込み状況から確認します。

### 7.1.2 `ID3D12Debug` のバージョン

デバッグ関連のインターフェースも、COM の流儀で版を重ねています。

| インターフェース | 主なメソッド |
|---|---|
| `ID3D12Debug` | `EnableDebugLayer` |
| `ID3D12Debug1` | `SetEnableGPUBasedValidation`<br>`SetEnableSynchronizedCommandQueueValidation` |
| `ID3D12Debug3` | `ID3D12Debug1` と同じ機能を `ID3D12Debug` から辿れるようにしたもの |
| `ID3D12Debug5` | `SetEnableAutoName` |
| `ID3D12Debug6` | `SetForceLegacyBarrierValidation` |

`ID3D12Debug5::SetEnableAutoName(TRUE)` は、名前を付けていないオブジェクトに自動で名前を割り当てます。便利ですが、**第6章 6.5 節で作った手動の名前付けを置き換えるものではありません。** 自動生成される名前は `Unnamed ID3D12Resource Object` のような機械的なもので、`"InstanceBuffer (frame 2)"` の情報量には遠く及びません。

本書は自動命名を有効にしません。**名前を付け忘れたオブジェクトが、名前のないまま目立つほうが良い**からです。

### 7.1.3 GPU-Based Validation は既定で無効にする

```cpp
ComPtr<ID3D12Debug1> debug1;
if (SUCCEEDED(debug.As(&debug1)))
{
    debug1->SetEnableGPUBasedValidation(TRUE);   // 既定では呼ばない
}
```

GPU-Based Validation(GBV)は、シェーダーの実行中にディスクリプタの妥当性やリソースの状態を検証します。通常のデバッグレイヤーが CPU 側の API 呼び出しを検証するのに対し、**GBV は GPU 側で起きることを検証します。**

強力ですが、**10〜100 倍遅くなります。** 常時有効にしていては開発が進みません。

**本書の方針:設定で切り替えられるようにし、既定は無効。** 具体的なバグを追うときにだけ有効にします(第30章)。

> **Aftermath との併用について**
>
> 第8章で Aftermath を組み込みますが、GBV を有効にすると GPU の実行タイミングが大きく変わります。**「GBV を有効にしたら再現しなくなった」という現象は珍しくありません。**
>
> クラッシュを追うときは、まず素の状態で Aftermath のダンプを取り、それでも原因が絞れないときに GBV を試す、という順序が合理的です。

### 7.1.4 DRED もここで有効にする

DRED(Device Removed Extended Data)は、デバイスロスト時に「GPU が最後にどこまで実行したか」を記録する仕組みです。第38章で扱いますが、**有効化の位置は本節と同じ、デバイス生成の前**です。

```cpp
ComPtr<ID3D12DeviceRemovedExtendedDataSettings> dred;
if (SUCCEEDED(::D3D12GetDebugInterface(IID_PPV_ARGS(&dred))))
{
    dred->SetAutoBreadcrumbsEnablement(D3D12_DRED_ENABLEMENT_FORCED_ON);
    dred->SetPageFaultEnablement(D3D12_DRED_ENABLEMENT_FORCED_ON);
}
```

本書は次章で Aftermath を導入し、そちらを主たる解析手段とします。役割が重なるため DRED は第38章まで保留しますが、**「有効化するならここ」ということだけ覚えておいてください。** 後から追加するときに、初期化順序を思い出せます。

---

## 7.2 Agility SDK 版デバッグレイヤーの確認

第4章 4.7 節で、`D3D12Core.dll` がどこから読み込まれたかを調べました。デバッグレイヤーにも同じ手が使えます。

`EnableDebugLayer()` を呼ぶと、`d3d12SDKLayers.dll` が読み込まれます。そのパスを見れば、Agility SDK 版か OS 標準版かがわかります。

```cpp
void LogDebugLayerSource()
{
    const HMODULE layers = ::GetModuleHandleW(L"d3d12SDKLayers.dll");
    if (layers == nullptr)
    {
        LOG_WARN(L"d3d12SDKLayers.dll is not loaded (debug layer disabled?)");
        return;
    }

    wchar_t path[MAX_PATH]{};
    ::GetModuleFileNameW(layers, path, MAX_PATH);
    LOG_INFO(L"debug layer : {}", path);
}
```

期待される出力:

```
[Info ] GraphicsDevice.cpp(45): debug layer : C:\dev\...\build\x64\Debug\D3D12\d3d12SDKLayers.dll
```

`System32` になっていたら、**`D3D12Core.dll` は差し替わったのに検証レイヤーだけ OS 版**という状態です。第4章 4.4.3 節で警告した「バージョン不一致」がまさにこれで、誤った警告が出たりクラッシュしたりします。

> **`DXGI_ERROR_SDK_COMPONENT_MISSING` が出る場合**
>
> `D3D12GetDebugInterface` がこのエラーを返すのは、デバッグレイヤーの実体が見つからないときです。Agility SDK を正しく導入していれば起こりません(第4章 4.6.3 節)。
>
> 出てしまった場合は、`build/x64/Debug/D3D12/` に `d3d12SDKLayers.dll` があるかを確認してください。

---

## 7.3 DXGI ファクトリとアダプタ列挙

### 7.3.1 ファクトリを作る

```cpp
UINT factoryFlags = 0;
#if defined(_DEBUG)
    factoryFlags |= DXGI_CREATE_FACTORY_DEBUG;
#endif

ComPtr<IDXGIFactory6> factory;
HR_TRY(::CreateDXGIFactory2(factoryFlags, IID_PPV_ARGS(&factory)));
```

`DXGI_CREATE_FACTORY_DEBUG` は、DXGI 側のデバッグレイヤーを有効にします。D3D12 のデバッグレイヤーとは別物で、スワップチェーンまわりの誤用を検出してくれます(第11章で効きます)。

`IDXGIFactory6` を要求しているのは、次項の `EnumAdapterByGpuPreference` を使うためです。

> **フラグつきの生成が失敗したら、フラグなしで作り直す**
>
> 環境によっては DXGI のデバッグ機能が利用できず、`DXGI_ERROR_SDK_COMPONENT_MISSING` で失敗することがあります。**デバッグ機能がないだけでアプリが起動しないのは行き過ぎです。** 実装ではフォールバックを入れます(7.7 節)。

### 7.3.2 `EnumAdapterByGpuPreference` を使う

アダプタの列挙には 2 つの方法があります。

**古い方法:`EnumAdapters1`**

```cpp
for (UINT i = 0; factory->EnumAdapters1(i, &adapter) != DXGI_ERROR_NOT_FOUND; ++i)
{
    // 順序は「システムが決めた順」。用途に最適とは限らない
}
```

**新しい方法:`EnumAdapterByGpuPreference`**

```cpp
for (UINT i = 0;
     factory->EnumAdapterByGpuPreference(
         i, DXGI_GPU_PREFERENCE_HIGH_PERFORMANCE,
         IID_PPV_ARGS(&adapter)) != DXGI_ERROR_NOT_FOUND;
     ++i)
{
    // 高性能な順に並んだ状態で返ってくる
}
```

**本書は後者を使います。** ハイブリッドグラフィックス環境(次項)で、内蔵 GPU を先に掴んでしまう事故が起きにくくなります。

`DXGI_GPU_PREFERENCE` には 3 つの値があります。

| 値 | 意味 |
|---|---|
| `DXGI_GPU_PREFERENCE_UNSPECIFIED` | 指定なし(`EnumAdapters1` と同じ順) |
| `DXGI_GPU_PREFERENCE_MINIMUM_POWER` | 省電力優先(内蔵 GPU が先) |
| `DXGI_GPU_PREFERENCE_HIGH_PERFORMANCE` | **性能優先(ディスクリート GPU が先)** |

**列挙のループはどちらの方式でも `DXGI_ERROR_NOT_FOUND` で終端**します。第6章の ✅ で意図的に発生させたエラーがこれです。**列挙の終端は「異常」ではないので、`HR_TRY` で伝搬させてはいけません。**

### 7.3.3 ベンダ ID で判定する

アダプタの情報は `DXGI_ADAPTER_DESC1` で取得します。

```cpp
DXGI_ADAPTER_DESC1 desc{};
adapter->GetDesc1(&desc);
```

| 主なフィールド | 内容 |
|---|---|
| `Description` | 製品名(`WCHAR[128]`) |
| `VendorId` | ベンダ ID |
| `DeviceId` | デバイス ID |
| `DedicatedVideoMemory` | 専用 VRAM(バイト) |
| `SharedSystemMemory` | 共有メモリ(バイト) |
| `AdapterLuid` | このアダプタの一意な識別子 |
| `Flags` | `DXGI_ADAPTER_FLAG_SOFTWARE` など |

主なベンダ ID:

| ベンダ | ID |
|---|---|
| **NVIDIA** | **0x10DE** |
| AMD | 0x1002 |
| Intel | 0x8086 |
| Microsoft(WARP) | 0x1414 |

```cpp
constexpr UINT kVendorNvidia    = 0x10DE;
constexpr UINT kVendorAmd       = 0x1002;
constexpr UINT kVendorIntel     = 0x8086;
constexpr UINT kVendorMicrosoft = 0x1414;
```

**ソフトウェアアダプタは必ず除外します。**

```cpp
if (desc.Flags & DXGI_ADAPTER_FLAG_SOFTWARE)
{
    continue;   // WARP はここでは選ばない(7.3.5 で明示的に選ぶ)
}
```

これを忘れると、GPU があるのに WARP を掴んでしまうことがあります。**「なぜか異常に遅い」の典型的な原因**です。

本書は NVIDIA を前提としますが、**NVIDIA 以外を拒否はしません。** 見つからなければ警告を出したうえで、利用可能なものを使います。第2章 2.5 節で述べた通り、NVIDIA は「対象」であって「依存先」ではありません。

### 7.3.4 ハイブリッドグラフィックスの落とし穴

第2章 2.1.3 節のコラムで触れた問題に、ここで正面から対処します。

ノート PC では、内蔵 GPU と NVIDIA GPU の両方が列挙されます。どちらが使われるかは、次の 3 つで決まります。

```
① exe に埋め込まれたエクスポートシンボル(ドライバへの指示)
② Windows の「グラフィックスの設定」(アプリごとの設定)
③ アプリ自身のアダプタ選択コード
```

**③ だけでは不十分です。** ① と ② は、そもそもどのドライバが有効になるかを決めています。

#### `NvOptimusEnablement` を宣言する

NVIDIA の Optimus ドライバは、exe から `NvOptimusEnablement` というシンボルがエクスポートされていれば、**そのアプリでは必ずディスクリート GPU を使います。**

第4章で `D3D12SDKVersion` をエクスポートしたのと、まったく同じ仕組みです。

```cpp
// src/GpuPreference.cpp
//
// ハイブリッドグラフィックス環境で、ディスクリート GPU を
// 使うようドライバに指示する。
//
// 【重要】このファイルの内容を他へ移動しないこと。
//         extern の省略も不可(第4章 4.3.3 節を参照)。

#include "pch.h"

extern "C"
{
    // NVIDIA Optimus
    __declspec(dllexport) extern const DWORD NvOptimusEnablement = 0x00000001;
}

extern "C"
{
    // AMD PowerXpress(本書の対象外だが 1 行なので入れておく)
    __declspec(dllexport) extern const int AmdPowerXpressRequestHighPerformance = 1;
}
```

**第4章 4.3.3 節で学んだ `extern` の話が、そのまま効いてきます。** `extern` を落とすと内部リンケージになり、エクスポートされず、**何のエラーも出ないまま内蔵 GPU で動きます。**

確認方法も同じです。

```
dumpbin /exports build\x64\Debug\D3D12Book.exe
```

```
    ordinal hint RVA      name

          1    0 0002B000 AmdPowerXpressRequestHighPerformance
          2    1 0002B004 D3D12SDKPath
          3    2 0002B00C D3D12SDKVersion
          4    3 0002B010 NvOptimusEnablement
```

#### それでも `EnumAdapterByGpuPreference` を使う理由

エクスポートシンボルはドライバへの指示であり、**DXGI の列挙順を保証するものではありません。** 両方やるのが確実です。

- エクスポートシンボル → どのドライバを起動するか
- `HIGH_PERFORMANCE` 列挙 → 列挙されたうちどれを選ぶか
- ベンダ ID の確認 → 選んだものが本当に意図通りか

**3 つとも実装し、最後にログで確認する。** これが本書の方針です。

### 7.3.5 WARP へのフォールバック

WARP(Windows Advanced Rasterization Platform)は、Microsoft が提供する**ソフトウェア実装の Direct3D 12** です。CPU で動くので極端に遅いのですが、診断用として価値があります。

```cpp
ComPtr<IDXGIAdapter1> warp;
HR_TRY(factory->EnumWarpAdapter(IID_PPV_ARGS(&warp)));
```

**WARP が役に立つ場面は 2 つです。**

**1. GPU ドライバのバグかどうかを切り分ける**
自分のコードが正しいのに絵が壊れる。WARP で動かして正しく出れば、ドライバ側の問題である可能性が高まります。逆に WARP でも壊れるなら、自分のコードのバグです。**第2章 4.5.3 節の切り分け手順に、もう一つ手札が加わります。**

**2. GPU がない環境で動作を確認する**
CI 環境や仮想マシンなど。

WARP は機能レベル 12_1 まで対応し、レイトレーシングやメッシュシェーダーも(非常に遅いながら)動きます。

本書では、**設定で明示的に指定したときだけ** WARP を使います。自動フォールバックはしません。「気づかないうちに WARP で動いていた」ほうが害が大きいからです。

### 7.3.6 VRAM の量を取る

`DXGI_ADAPTER_DESC1::DedicatedVideoMemory` は、**アダプタが持つ VRAM の総量**です。実際に使える量ではありません。

より実用的な情報は `IDXGIAdapter3::QueryVideoMemoryInfo` で取れます。

```cpp
ComPtr<IDXGIAdapter3> adapter3;
if (SUCCEEDED(adapter.As(&adapter3)))
{
    DXGI_QUERY_VIDEO_MEMORY_INFO info{};
    adapter3->QueryVideoMemoryInfo(
        0, DXGI_MEMORY_SEGMENT_GROUP_LOCAL, &info);

    // info.Budget       … OS が今このプロセスに割り当ててよいと考えている量
    // info.CurrentUsage … 現在使っている量
}
```

`Budget` は、**他のアプリケーションの状況に応じて動的に変わります。** ブラウザで動画を再生し始めると減ります。第21章でリソース管理を設計する際、この値を監視する話が出てきます。

本章では起動時に一度だけ記録します。

---

## 7.4 `D3D12CreateDevice` と機能レベル

### 7.4.1 最小機能レベルという考え方

```cpp
ComPtr<ID3D12Device> device;
HR_TRY(::D3D12CreateDevice(
    adapter.Get(),
    D3D_FEATURE_LEVEL_11_0,      // ← 最小要求
    IID_PPV_ARGS(&device)));
```

第 2 引数は「**最低でもこのレベルは必要**」という要求です。アダプタがそれを満たさなければ失敗します。満たしていれば成功し、**実際のデバイスはそれ以上の能力を持ちうる**ということです。

つまり、`D3D_FEATURE_LEVEL_11_0` を渡して作ったデバイスが、実は 12_2 相当の能力を持っていることは普通にあります。**引数の値と、できあがったデバイスの能力は別物です。**

**本書は最小要求を 11_0 にします。** 「まず動かして、能力は後で問い合わせる」ほうが、失敗時の切り分けが簡単だからです。12_2 を要求して失敗すると、「アダプタが悪いのか、ドライバか、コードか」がわかりません。

### 7.4.2 実際の上限を問い合わせる

作ったデバイスが実際にどこまで対応しているかは、`CheckFeatureSupport` で尋ねます。

```cpp
constexpr D3D_FEATURE_LEVEL kLevels[] = {
    D3D_FEATURE_LEVEL_12_2,
    D3D_FEATURE_LEVEL_12_1,
    D3D_FEATURE_LEVEL_12_0,
    D3D_FEATURE_LEVEL_11_1,
    D3D_FEATURE_LEVEL_11_0,
};

D3D12_FEATURE_DATA_FEATURE_LEVELS data{};
data.NumFeatureLevels        = static_cast<UINT>(std::size(kLevels));
data.pFeatureLevelsRequested = kLevels;

if (SUCCEEDED(device->CheckFeatureSupport(
        D3D12_FEATURE_FEATURE_LEVELS, &data, sizeof(data))))
{
    // data.MaxSupportedFeatureLevel に上限が入る
}
```

**渡した配列の中で最も高いものが返ります。** 配列に含まれていないレベルは返ってきません。

### 7.4.3 12_2 は DirectX 12 Ultimate

`D3D_FEATURE_LEVEL_12_2` は、次の機能をすべて備えることを意味します。

- レイトレーシング Tier 1.1
- メッシュシェーダー Tier 1
- 可変レートシェーディング Tier 2
- サンプラーフィードバック Tier 0.9

これは「DirectX 12 Ultimate」というブランド名で呼ばれるものと同じです。そして第2章で見た通り、**NVIDIA では Turing 世代以降がこれを満たします。**

つまり、起動時に `12_2` が出れば、**本書の第5部まで一通り動く環境**だということです。GTX 16 シリーズのようにレイトレーシング回路を持たない機種では、ここが `12_1` になります。**第2章のチェックシートとの、最初の答え合わせがこれです。**

---

## 7.5 `CheckFeatureSupport` で機能を問い合わせる

### 7.5.1 呼び方の型

`CheckFeatureSupport` は、**「どの機能グループか」を表す列挙値と、対応する構造体**の組で呼びます。

```cpp
HRESULT CheckFeatureSupport(
    D3D12_FEATURE Feature,       // どのグループか
    void*         pFeatureSupportData,
    UINT          FeatureSupportDataSize);
```

型情報が失われる `void*` 版なので、**列挙値と構造体の対応を間違えてもコンパイルは通ります。** 実行時に `E_INVALIDARG` が返るか、最悪の場合は誤ったサイズで読み書きされます。

そこで、小さなヘルパーを書いておきます。

```cpp
// 対応していない機能は E_INVALIDARG が返る。
// 「対応していない」は異常ではないので、bool で返す。
template <typename T>
bool QueryFeature(ID3D12Device* device, D3D12_FEATURE feature, T& data)
{
    return SUCCEEDED(device->CheckFeatureSupport(feature, &data, sizeof(T)));
}
```

**戻り値を必ず確認してください。** 新しい機能グループは、古いランタイムやドライバでは `E_INVALIDARG` を返します。そのとき構造体の中身は**未初期化のまま**です。チェックせずに読むと、乱数を機能フラグとして扱うことになります。

> **`CD3DX12FeatureSupport` は使わない**
>
> `d3dx12.h` には、すべての機能を一括で問い合わせる `CD3DX12FeatureSupport` というクラスがあります。とても便利です。
>
> **本書は使いません**(第1章 1.3.1 節)。上の `QueryFeature` は 4 行です。この 4 行を書くことで、「機能グループと構造体が 1 対 1 で対応している」という構造が頭に入ります。

### 7.5.2 シェーダーモデル

```cpp
constexpr D3D_SHADER_MODEL kShaderModels[] = {
    D3D_SHADER_MODEL_6_9,
    D3D_SHADER_MODEL_6_8,
    D3D_SHADER_MODEL_6_7,
    D3D_SHADER_MODEL_6_6,
    D3D_SHADER_MODEL_6_5,
    D3D_SHADER_MODEL_6_0,
};

D3D_SHADER_MODEL result = D3D_SHADER_MODEL_6_0;
for (const D3D_SHADER_MODEL model : kShaderModels)
{
    D3D12_FEATURE_DATA_SHADER_MODEL data{ model };
    if (QueryFeature(device, D3D12_FEATURE_SHADER_MODEL, data))
    {
        result = data.HighestShaderModel;
        break;
    }
}
```

**この API は独特な作りをしています。** `HighestShaderModel` は入力でも出力でもあります。「これ以下で最も高いものを教えてほしい」と要求し、同じフィールドに答えが返ります。

そして、**ランタイムが知らない値を渡すと `E_INVALIDARG` になります。** 将来 Shader Model 6.10 が出たとき、古いランタイムに `6_10` を渡すと失敗します。だから高いほうから順に試すループが必要です。

本書で重要になるのは次の 2 つです。

| バージョン | 必要な章 |
|---|---|
| Shader Model 6.6 | 第33章(バインドレス / Dynamic Resources) |
| Shader Model 6.9 | 第37章(DXR 1.2 の SER / OMM) |

### 7.5.3 `D3D12_OPTIONS` 系の読み方

D3D12 の機能クエリは、**`OPTIONS`、`OPTIONS1`、`OPTIONS2`……と番号を増やしながら追加されてきました。** 新しい機能がどの番号に入るかに規則性はありません。ヘッダを見るしかありません。

本書で使う主なものを挙げます。

| 構造体 | 主なフィールド | 使う章 |
|---|---|---|
| `OPTIONS` | `ResourceBindingTier`<br>`ResourceHeapTier` | 第19章、第33章 |
| `OPTIONS1` | `WaveOps`<br>`WaveLaneCountMin` / `Max`<br>`TotalLaneCount` | **第32章** |
| `OPTIONS5` | `RaytracingTier` | 第37章 |
| `OPTIONS6` | `VariableShadingRateTier` | (本書では未使用) |
| `OPTIONS7` | `MeshShaderTier`<br>`SamplerFeedbackTier` | 第36章 |
| `OPTIONS12` | `EnhancedBarriersSupported` | 第30章 |
| `OPTIONS16` | `GPUUploadHeapSupported` | 第21章 |
| `OPTIONS22` | `ShaderExecutionReorderingActuallyReorders` | 第37章 |
| `ARCHITECTURE1` | `UMA` / `CacheCoherentUMA` | 第21章 |

**レイトレーシングの Tier に注目してください。**

| 値 | 意味 |
|---|---|
| `D3D12_RAYTRACING_TIER_1_0` | DXR 1.0 |
| `D3D12_RAYTRACING_TIER_1_1` | DXR 1.1(インラインレイトレーシング) |
| **`D3D12_RAYTRACING_TIER_1_2`** | **DXR 1.2(SER と Opacity Micromap の両方)** |

第2章 2.1.2 節の表では SER / OMM を「Ada 以降」としましたが、**この判定は `OPTIONS5.RaytracingTier` という従来の場所で行います。** 新しい `OPTIONS22` が必要になるのは、「SER の API は使えるが、このデバイスは実際に並べ替えを行うのか」というより細かい問いに答えるときだけです。

なお、NVIDIA では **OMM は Ada 以降でハードウェア支援、それ以前はソフトウェアエミュレーション**という扱いになっています。「動くが速くない」という状態がありうるので、第2章の表の「×」は「実用的な速度では動かない」と読んでください。

### 7.5.4 warp = 32 を実測する

第2章 2.3.1 節で、「NVIDIA GPU は 32 スレッドをひとまとまりにして実行する」と書きました。**その値を、いま実際に取得できます。**

```cpp
D3D12_FEATURE_DATA_D3D12_OPTIONS1 options1{};
if (QueryFeature(device, D3D12_FEATURE_D3D12_OPTIONS1, options1))
{
    // NVIDIA なら 32 / 32 が返るはず
    LOG_INFO(L"wave lanes  : {} .. {} (total {})",
             options1.WaveLaneCountMin,
             options1.WaveLaneCountMax,
             options1.TotalLaneCount);
}
```

期待される出力(NVIDIA の場合):

```
[Info ] GraphicsDevice.cpp(210): wave lanes  : 32 .. 32 (total 5888)
```

`TotalLaneCount` は GPU 全体の同時実行レーン数で、おおむね「SM 数 × SM あたりのレーン数」に対応します。**第2章で調べた GPU の規模が、数値として現れます。**

そして第2章 2.3.1 節の注意書き —— **「32 という数字をコードに埋め込まない」** —— が、ここで具体的な形になります。ハードコードするのではなく、**この値を起動時に取得して保持する**のが正しい姿です。第32章でスレッドグループサイズを決める際、この値を根拠にします。

### 7.5.5 機能一覧を構造体にまとめる

問い合わせた結果は、後の章で何度も参照します。まとめて保持しておきます。

```cpp
// src/Graphics/DeviceCaps.h
#pragma once
#include "std_import.h"

namespace Graphics
{
    struct DeviceCaps
    {
        // 全体
        D3D_FEATURE_LEVEL maxFeatureLevel  = D3D_FEATURE_LEVEL_11_0;
        D3D_SHADER_MODEL  maxShaderModel   = D3D_SHADER_MODEL_6_0;

        // OPTIONS
        D3D12_RESOURCE_BINDING_TIER resourceBindingTier{};
        D3D12_RESOURCE_HEAP_TIER    resourceHeapTier{};

        // OPTIONS1 —— 第32章で使う
        bool waveOps          = false;
        UINT waveLaneCountMin = 0;
        UINT waveLaneCountMax = 0;
        UINT totalLaneCount   = 0;

        // 各機能
        D3D12_RAYTRACING_TIER            raytracingTier{};
        D3D12_MESH_SHADER_TIER           meshShaderTier{};
        D3D12_SAMPLER_FEEDBACK_TIER      samplerFeedbackTier{};
        D3D12_VARIABLE_SHADING_RATE_TIER vrsTier{};

        bool enhancedBarriers = false;   // 第30章
        bool gpuUploadHeap    = false;   // 第21章
        bool serReorders      = false;   // 第37章

        // アーキテクチャ
        bool uma              = false;
        bool cacheCoherentUma = false;

        //--- 本書の各章が動くかの判定 ---
        bool SupportsBindless()      const noexcept;  // 第33章
        bool SupportsMeshShader()    const noexcept;  // 第36章
        bool SupportsRaytracing()    const noexcept;  // 第37章
        bool SupportsDxr12()         const noexcept;  // 第37章
    };
}
```

判定関数の実装:

```cpp
bool DeviceCaps::SupportsBindless() const noexcept
{
    return resourceBindingTier >= D3D12_RESOURCE_BINDING_TIER_3
        && maxShaderModel      >= D3D_SHADER_MODEL_6_6;
}

bool DeviceCaps::SupportsMeshShader() const noexcept
{
    return meshShaderTier >= D3D12_MESH_SHADER_TIER_1;
}

bool DeviceCaps::SupportsRaytracing() const noexcept
{
    return raytracingTier >= D3D12_RAYTRACING_TIER_1_1;
}

bool DeviceCaps::SupportsDxr12() const noexcept
{
    return raytracingTier >= D3D12_RAYTRACING_TIER_1_2
        && maxShaderModel >= D3D_SHADER_MODEL_6_9;
}
```

**第2章 2.1.2 節の表を、コードにしたものです。** 表は「買い替えが必要か」を判断する目安でしたが、こちらは実行時の真実です。以後の章では、**必ずこちらを見ます。**

---

## 7.6 `ID3D12InfoQueue1` で警告時にブレークする

### 7.6.1 深刻度でブレークする

デバッグレイヤーは、問題を見つけると「出力」ウィンドウにメッセージを書きます。しかし、**メッセージだけでは「どのコードが原因か」がわかりません。**

`SetBreakOnSeverity` を使うと、**その場でデバッガが停止します。**

```cpp
ComPtr<ID3D12InfoQueue> infoQueue;
if (SUCCEEDED(device.As(&infoQueue)))
{
    infoQueue->SetBreakOnSeverity(D3D12_MESSAGE_SEVERITY_CORRUPTION, TRUE);
    infoQueue->SetBreakOnSeverity(D3D12_MESSAGE_SEVERITY_ERROR,      TRUE);
    infoQueue->SetBreakOnSeverity(D3D12_MESSAGE_SEVERITY_WARNING,    TRUE);
}
```

**これが本節で最も価値のある 4 行です。**

停止したとき、Visual Studio の呼び出し履歴を見てください。**D3D12 の内部から、自分のコードまで遡れます。** 「どのファイルの何行目で、どんな引数を渡したときに、この警告が出たのか」が一目でわかります。

| 深刻度 | ブレークすべきか |
|---|---|
| `CORRUPTION` | **必ず**。メモリ破壊が起きている |
| `ERROR` | **必ず**。API の誤用 |
| `WARNING` | **本書では有効にする**。詳細は下記 |
| `INFO` | 不要 |
| `MESSAGE` | 不要 |

**`WARNING` でもブレークする、というのは強めの設定です。** 実際、無害な警告で止まることがあります。それでも本書がこれを勧めるのは、**警告の多くが「今は動くが、後で壊れる」ことを示している**からです。

たとえば「クリア値がリソース生成時の指定と一致しない」という警告は、性能を落とすだけで絵は正しく出ます。放置しても気づきません。しかし、こういう小さな不一致の積み重ねが、第20章あたりで原因不明の不具合として現れます。

うるさすぎると感じたら、7.6.3 節の方法で個別に抑制してください。**「全部黙らせる」ではなく「これは調べたうえで無視する」という判断を残すことが大事です。**

### 7.6.2 メッセージを自作ログへ流す

`ID3D12InfoQueue1` は、デバッグレイヤーのメッセージを**コールバックで受け取る**機能を持っています。第6章で作ったログ機能に流し込めます。

```cpp
static void CALLBACK OnDebugLayerMessage(
    D3D12_MESSAGE_CATEGORY category,
    D3D12_MESSAGE_SEVERITY severity,
    D3D12_MESSAGE_ID       id,
    LPCSTR                 description,
    void*                  context)
{
    (void)category;
    (void)context;

    // description はナロー文字列。第6章の ToWide を使う。
    const std::wstring text = Core::ToWide(
        description != nullptr ? description : "");

    const std::wstring line =
        std::format(L"[D3D12] (id {}) {}", static_cast<int>(id), text);

    switch (severity)
    {
    case D3D12_MESSAGE_SEVERITY_CORRUPTION:
    case D3D12_MESSAGE_SEVERITY_ERROR:
        Core::Log::WriteRaw(Core::LogLevel::Error, {}, line);
        break;
    case D3D12_MESSAGE_SEVERITY_WARNING:
        Core::Log::WriteRaw(Core::LogLevel::Warning, {}, line);
        break;
    default:
        Core::Log::WriteRaw(Core::LogLevel::Trace, {}, line);
        break;
    }
}
```

登録します。

```cpp
ComPtr<ID3D12InfoQueue1> infoQueue1;
if (SUCCEEDED(device.As(&infoQueue1)))
{
    infoQueue1->RegisterMessageCallback(
        &OnDebugLayerMessage,
        D3D12_MESSAGE_CALLBACK_FLAG_NONE,
        nullptr,
        &m_messageCallbackCookie);
}
```

終了時に解除します。

```cpp
if (m_infoQueue1 && m_messageCallbackCookie != 0)
{
    m_infoQueue1->UnregisterMessageCallback(m_messageCallbackCookie);
    m_messageCallbackCookie = 0;
}
```

> **`description` がナロー文字列である点**
>
> 第6章で `ToWide` を作った理由の一つがこれです。デバッグレイヤーのメッセージ、DXC のエラー(第13章)、Aftermath の出力(第31章)—— **D3D 周辺の文字列は、たいていナローで返ってきます。**

> **ログの発生位置について**
>
> コールバック内で `std::source_location::current()` を取っても、**このコールバック関数の位置**しか得られません。原因となった API 呼び出しの位置ではありません。
>
> そこで役割を分担します。**メッセージの内容はコールバックが、発生位置は `SetBreakOnSeverity` によるブレークが教えてくれます。** 両方を有効にしておくのが正解です。

> **`ID3D12InfoQueue1` が取れない場合**
>
> Windows 11 または Agility SDK が必要です。本書は Agility SDK を導入済みなので通常は取得できますが、**失敗しても致命的ではありません。** `SetBreakOnSeverity` だけでも十分に機能します。実装ではフォールバックを入れます。

### 7.6.3 ノイズを抑制する —— と、その危険

特定のメッセージだけを黙らせることができます。

```cpp
D3D12_MESSAGE_ID denyIds[] = {
    D3D12_MESSAGE_ID_CLEARRENDERTARGETVIEW_MISMATCHINGCLEARVALUE,
    D3D12_MESSAGE_ID_CLEARDEPTHSTENCILVIEW_MISMATCHINGCLEARVALUE,
};

D3D12_INFO_QUEUE_FILTER filter{};
filter.DenyList.NumIDs  = static_cast<UINT>(std::size(denyIds));
filter.DenyList.pIDList = denyIds;

infoQueue->AddStorageFilterEntries(&filter);
```

**この機能は、慎重に使ってください。**

抑制リストは、放っておくと増えます。「よくわからないけど止まるから消しておこう」を繰り返した結果、**本当に見るべき警告まで消えている**プロジェクトを、実務では時々見かけます。

**本書のルール:抑制する場合は、必ずコメントで理由を書く。**

```cpp
D3D12_MESSAGE_ID denyIds[] = {
    // スワップチェーンのバックバッファは最適化クリア値を持てないため、
    // ClearRenderTargetView のたびにこの警告が出る。仕様上避けられない。
    D3D12_MESSAGE_ID_CLEARRENDERTARGETVIEW_MISMATCHINGCLEARVALUE,
};
```

理由を書けないなら、それは抑制すべきではないメッセージです。

**本章の時点では、抑制リストは空にしておきます。** 必要になったときに、上のルールに従って追加してください。

---

## 7.7 実装をまとめる

`src/Graphics/GraphicsDevice.h` / `.cpp` を作ります。

### 7.7.1 ヘッダ

```cpp
// src/Graphics/GraphicsDevice.h
#pragma once
#include "std_import.h"
#include "Core/Error.h"
#include "Graphics/DeviceCaps.h"

namespace Graphics
{
    struct AdapterInfo
    {
        std::wstring  description;
        UINT          vendorId = 0;
        UINT          deviceId = 0;
        std::uint64_t dedicatedVideoMemory = 0;
        std::uint64_t sharedSystemMemory   = 0;
        std::uint64_t videoMemoryBudget    = 0;
        bool          isSoftware = false;
    };

    class GraphicsDevice
    {
    public:
        struct Config
        {
            bool enableDebugLayer          = true;
            bool enableGpuBasedValidation  = false;   // 遅い(7.1.3)
            bool breakOnWarning            = true;    // 7.6.1
            bool useWarp                   = false;   // 7.3.5
        };

        GraphicsDevice() = default;
        ~GraphicsDevice();

        GraphicsDevice(const GraphicsDevice&)            = delete;
        GraphicsDevice& operator=(const GraphicsDevice&) = delete;

        Core::Status Initialize(const Config& config);
        void         Shutdown();

        ID3D12Device*  Device()  const noexcept { return m_device.Get(); }
        IDXGIFactory6* Factory() const noexcept { return m_factory.Get(); }
        IDXGIAdapter1* Adapter() const noexcept { return m_adapter.Get(); }

        const AdapterInfo& AdapterDesc() const noexcept { return m_adapterInfo; }
        const DeviceCaps&  Caps()        const noexcept { return m_caps; }

        void LogSummary() const;

    private:
        Core::Status EnableDebugLayer(const Config& config);
        Core::Status CreateFactory(bool enableDebug);
        Core::Status SelectAdapter(bool useWarp);
        Core::Status CreateDevice();
        void         QueryCaps();
        void         SetupInfoQueue(bool breakOnWarning);

        Microsoft::WRL::ComPtr<IDXGIFactory6>    m_factory;
        Microsoft::WRL::ComPtr<IDXGIAdapter1>    m_adapter;
        Microsoft::WRL::ComPtr<ID3D12Device>     m_device;
        Microsoft::WRL::ComPtr<ID3D12InfoQueue1> m_infoQueue1;

        AdapterInfo m_adapterInfo{};
        DeviceCaps  m_caps{};
        DWORD       m_messageCallbackCookie = 0;
    };
}
```

**メンバの宣言順序に注目してください。** 第6章 6.2.6 節で決めた通り、`m_device` は子オブジェクトより先に宣言しています。`m_infoQueue1` はデバイスから取得したものなので、デバイスより後です。破棄はこの逆順に起こります。

### 7.7.2 アダプタ選択

主要部分だけ抜き出します。

```cpp
Core::Status GraphicsDevice::SelectAdapter(bool useWarp)
{
    constexpr UINT kVendorNvidia = 0x10DE;

    if (useWarp)
    {
        HR_TRY(m_factory->EnumWarpAdapter(IID_PPV_ARGS(&m_adapter)));
        LOG_WARN(L"using WARP (software rasterizer)");
    }
    else
    {
        // 高性能な順に列挙し、最初に見つかった
        // ハードウェアアダプタを採用する
        for (UINT index = 0; ; ++index)
        {
            Microsoft::WRL::ComPtr<IDXGIAdapter1> candidate;

            const HRESULT hr = m_factory->EnumAdapterByGpuPreference(
                index,
                DXGI_GPU_PREFERENCE_HIGH_PERFORMANCE,
                IID_PPV_ARGS(&candidate));

            if (hr == DXGI_ERROR_NOT_FOUND)
            {
                break;              // 列挙の終端。異常ではない
            }
            if (FAILED(hr))
            {
                return std::unexpected(
                    Core::MakeError(hr, L"EnumAdapterByGpuPreference"));
            }

            DXGI_ADAPTER_DESC1 desc{};
            candidate->GetDesc1(&desc);

            LOG_TRACE(L"adapter[{}] {} (vendor 0x{:04X}){}",
                      index, desc.Description, desc.VendorId,
                      (desc.Flags & DXGI_ADAPTER_FLAG_SOFTWARE)
                          ? L" [software]" : L"");

            if (desc.Flags & DXGI_ADAPTER_FLAG_SOFTWARE)
            {
                continue;           // WARP は明示指定でのみ使う
            }

            // D3D12 デバイスを作れるかを、作らずに確認する
            // (第4章 4.7.2 節と同じ手法)
            const HRESULT check = ::D3D12CreateDevice(
                candidate.Get(), D3D_FEATURE_LEVEL_11_0,
                __uuidof(ID3D12Device), nullptr);

            if (SUCCEEDED(check))
            {
                m_adapter = candidate;
                break;
            }
        }
    }

    if (!m_adapter)
    {
        return std::unexpected(Core::MakeError(
            DXGI_ERROR_NOT_FOUND, L"no usable adapter found"));
    }

    //--- 情報を記録する ---
    DXGI_ADAPTER_DESC1 desc{};
    m_adapter->GetDesc1(&desc);

    m_adapterInfo.description          = desc.Description;
    m_adapterInfo.vendorId             = desc.VendorId;
    m_adapterInfo.deviceId             = desc.DeviceId;
    m_adapterInfo.dedicatedVideoMemory = desc.DedicatedVideoMemory;
    m_adapterInfo.sharedSystemMemory   = desc.SharedSystemMemory;
    m_adapterInfo.isSoftware =
        (desc.Flags & DXGI_ADAPTER_FLAG_SOFTWARE) != 0;

    Microsoft::WRL::ComPtr<IDXGIAdapter3> adapter3;
    if (SUCCEEDED(m_adapter.As(&adapter3)))
    {
        DXGI_QUERY_VIDEO_MEMORY_INFO memory{};
        if (SUCCEEDED(adapter3->QueryVideoMemoryInfo(
                0, DXGI_MEMORY_SEGMENT_GROUP_LOCAL, &memory)))
        {
            m_adapterInfo.videoMemoryBudget = memory.Budget;
        }
    }

    if (!useWarp && m_adapterInfo.vendorId != kVendorNvidia)
    {
        LOG_WARN(L"selected adapter is not NVIDIA (vendor 0x{:04X}). "
                 L"Nsight Aftermath and some chapters may not work.",
                 m_adapterInfo.vendorId);
    }

    return {};
}
```

**`D3D12CreateDevice` に `nullptr` を渡して「作れるか」だけ確かめる手法**を、第4章に続いてここでも使っています。列挙したアダプタが D3D12 に対応しているとは限らないので、採用前に確認します。

### 7.7.3 名前を付ける

デバイスを作ったら、その場で名前を付けます(第6章 6.5 節)。

```cpp
Core::Status GraphicsDevice::CreateDevice()
{
    HR_TRY(::D3D12CreateDevice(
        m_adapter.Get(),
        D3D_FEATURE_LEVEL_11_0,
        IID_PPV_ARGS(&m_device)));

    D3D_NAME(m_device);       // → L"m_device"
    return {};
}
```

**第7章から最終章まで、オブジェクトを作るたびにこの 1 行が続きます。**

---

## ✅ 本章のゴール:環境の全容をログに出す

### `main.cpp`

```cpp
int WINAPI wWinMain(...)
{
    ::SetProcessDpiAwarenessContext(DPI_AWARENESS_CONTEXT_PER_MONITOR_AWARE_V2);

    Window window;
    if (!window.Create(L"D3D12Book - Chapter 7", 1280, 720))
    {
        return 1;
    }

    Graphics::GraphicsDevice device;

    Graphics::GraphicsDevice::Config config{};
    config.enableDebugLayer         = true;
    config.enableGpuBasedValidation = false;
    config.breakOnWarning           = true;
    config.useWarp                  = false;

    if (auto result = device.Initialize(config); !result)
    {
        Core::ReportError(result.error());
        ::MessageBoxW(nullptr, L"Direct3D 12 の初期化に失敗しました。",
                      L"D3D12Book", MB_OK | MB_ICONERROR);
        return 1;
    }

    device.LogSummary();

    while (window.ProcessMessages())
    {
        if (window.IsMinimized()) continue;
        // 第9章以降、ここにコマンドの記録と実行が入る
    }

    device.Shutdown();
    return 0;
}
```

### 期待される出力

`F5` で実行し、「出力」ウィンドウを確認します。

```
[Info ] GraphicsDevice.cpp(52): debug layer : C:\dev\D3D12Book\build\x64\Debug\D3D12\d3d12SDKLayers.dll
[Trace] GraphicsDevice.cpp(118): adapter[0] NVIDIA GeForce RTX 4070 (vendor 0x10DE)
[Trace] GraphicsDevice.cpp(118): adapter[1] Intel(R) UHD Graphics (vendor 0x8086)
[Trace] GraphicsDevice.cpp(118): adapter[2] Microsoft Basic Render Driver (vendor 0x1414) [software]

===== Graphics Device =====
[Info ] GraphicsDevice.cpp(280): adapter       : NVIDIA GeForce RTX 4070
[Info ] GraphicsDevice.cpp(281): vendor        : NVIDIA (0x10DE)  device 0x2786
[Info ] GraphicsDevice.cpp(283): video memory  : 12282 MB (budget 11460 MB)
[Info ] GraphicsDevice.cpp(285): shared memory : 16297 MB
[Info ] GraphicsDevice.cpp(288): feature level : 12_2  (DirectX 12 Ultimate)
[Info ] GraphicsDevice.cpp(289): shader model  : 6.9
[Info ] GraphicsDevice.cpp(291): wave lanes    : 32 .. 32  (total 5888)
[Info ] GraphicsDevice.cpp(293): binding tier  : 3
[Info ] GraphicsDevice.cpp(294): heap tier     : 2
[Info ] GraphicsDevice.cpp(296): raytracing    : Tier 1.2
[Info ] GraphicsDevice.cpp(297): mesh shader   : Tier 1
[Info ] GraphicsDevice.cpp(298): sampler fb    : Tier 1.0
[Info ] GraphicsDevice.cpp(300): enhanced barriers : yes
[Info ] GraphicsDevice.cpp(301): GPU upload heap   : yes
[Info ] GraphicsDevice.cpp(302): SER reorders      : yes
[Info ] GraphicsDevice.cpp(304): UMA           : no

----- Chapter availability -----
[Info ] GraphicsDevice.cpp(310): ch.33 bindless      : OK
[Info ] GraphicsDevice.cpp(311): ch.36 mesh shader   : OK
[Info ] GraphicsDevice.cpp(312): ch.37 raytracing    : OK
[Info ] GraphicsDevice.cpp(313): ch.37 DXR 1.2       : OK
===========================
```

### 確認すべきこと

**1. デバッグレイヤーが Agility SDK 版か**
`d3d12SDKLayers.dll` のパスが `build/` 以下であること。`System32` なら第4章 4.4 節を見直してください。

**2. 選ばれたアダプタが NVIDIA か**
`vendor : NVIDIA (0x10DE)` であること。ノート PC で内蔵 GPU が選ばれていたら、7.3.4 節の `NvOptimusEnablement` を確認してください。

**3. `wave lanes : 32 .. 32`**
**第2章 2.3.1 節で「NVIDIA の warp は 32」と書いた、その実測値です。**

**4. 機能レベルが 12_2 か**
`12_2` なら第5部まで動きます。`12_1` の場合、レイトレーシングまたはメッシュシェーダーが使えません。第2章 2.1.1 節のコラム(GTX 16 シリーズ)を確認してください。

**5. 第2章のチェックシートと突き合わせる**
本章の出力が、第2章 Step 4 で記入した予想と一致しているはずです。**一致していなければ、どちらかが間違っています。**

### ブレークの動作を確認する(任意)

`SetBreakOnSeverity` が効いているかを試すには、わざとエラーを起こします。デバイス生成の後に次の 1 行を入れてみてください。

```cpp
// わざと不正な引数を渡す
D3D12_COMMAND_QUEUE_DESC bad{};
bad.Type = static_cast<D3D12_COMMAND_LIST_TYPE>(999);

Microsoft::WRL::ComPtr<ID3D12CommandQueue> queue;
device.Device()->CreateCommandQueue(&bad, IID_PPV_ARGS(&queue));
```

デバッグレイヤーが有効なら、**この行でデバッガが停止し**、「出力」ウィンドウに具体的な指摘が出ます。

```
[Error] Log.cpp(60): [D3D12] (id 708) D3D12 ERROR: ID3D12Device::CreateCommandQueue:
        Specified CommandListType 999 is invalid. [ EXECUTION ERROR #708 ]
```

**確認したらこのコードは削除してください。**

---

### 本章の達成状態

- [ ] `D3D12GetDebugInterface` を `D3D12CreateDevice` より前に呼んでいる
- [ ] `d3d12SDKLayers.dll` が Agility SDK 版から読み込まれている
- [ ] `src/GpuPreference.cpp` を作り、`NvOptimusEnablement` をエクスポートした
- [ ] `dumpbin /exports` で 4 つのシンボルが確認できる
- [ ] `EnumAdapterByGpuPreference` で NVIDIA アダプタが選ばれている
- [ ] `wave lanes : 32 .. 32` が出力される
- [ ] 機能レベルとシェーダーモデルがログに出る
- [ ] `SetBreakOnSeverity` でエラー時にデバッガが停止する
- [ ] デバッグレイヤーのメッセージが自作ログに流れている
- [ ] `D3D_NAME(m_device)` で名前を付けた
- [ ] 第2章のチェックシートと一致している

---

## トラブルシューティング

| 症状 | 原因 | 対処 |
|---|---|---|
| デバッグレイヤーが効かない | `D3D12CreateDevice` の後に呼んでいる | 順序を入れ替える(7.1.1) |
| `DXGI_ERROR_SDK_COMPONENT_MISSING` | `d3d12SDKLayers.dll` がない | 第4章 4.4 節を確認 |
| 内蔵 GPU が選ばれる | `NvOptimusEnablement` 未エクスポート | 7.3.4 節。`dumpbin` で確認 |
| 同上 | Windows のグラフィックス設定 | 設定 → システム → 画面 → グラフィックス |
| 異常に遅い | WARP を掴んでいる | `DXGI_ADAPTER_FLAG_SOFTWARE` を除外(7.3.3) |
| `CheckFeatureSupport` が `E_INVALIDARG` | 古いランタイム/ドライバ | 対応していない機能。戻り値を確認する(7.5.1) |
| 機能フラグの値がおかしい | 戻り値を確認していない | 失敗時の構造体は未初期化(7.5.1) |
| シェーダーモデルが 6.0 のまま | 高い値でループを抜けている | 高いほうから順に試す(7.5.2) |
| `ID3D12InfoQueue1` が取れない | Windows 10 かつ Agility SDK 未適用 | 第4章。取れなくても致命的ではない |
| 警告で止まりすぎる | `breakOnWarning` | 一時的に `false`、または個別に抑制(7.6.3) |
| 終了時に警告が出る | コールバック未解除 | `UnregisterMessageCallback` を呼ぶ(7.6.2) |

---

## まとめ

**1. デバッグレイヤーは、デバイス生成の「前」に有効化する。**
後から呼んでも効きません。しかもエラーは出ません。Agility SDK と同じ「黙って失敗する」パターンです。

**2. アダプタ選択は 3 段構えで確実にする。**
`NvOptimusEnablement` のエクスポート、`HIGH_PERFORMANCE` での列挙、ベンダ ID の確認。どれか一つでは不十分です。そして最後にログで確かめます。

**3. `CheckFeatureSupport` の戻り値を必ず見る。**
失敗したとき構造体は未初期化のままです。チェックせずに読むと、乱数を機能フラグとして扱うことになります。

**4. 機能レベルは「要求」と「実際」が別物。**
`D3D12CreateDevice` に渡すのは最小要求です。実際の上限は `D3D12_FEATURE_FEATURE_LEVELS` で問い合わせます。

**5. `SetBreakOnSeverity` が、この先 30 章の相棒になる。**
メッセージの内容はコールバックが、発生位置はブレークが教えてくれます。**構造体を手で埋める本書のスタイルでは、これが最大の防御線です。**

次章では Nsight Aftermath を組み込みます。まだ三角形すら出ていない段階でクラッシュ解析の準備をするのは奇妙に見えますが、**Aftermath の初期化はデバイス生成を挟む形で行う必要があり、後付けは初期化コードの再設計を意味します。** 本章で書いた `GraphicsDevice::Initialize` に、次章で数行を差し込みます。

---

## 参考リンク

| 内容 | URL |
|---|---|
| `ID3D12Debug` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12sdklayers/nn-d3d12sdklayers-id3d12debug |
| `ID3D12Device::CheckFeatureSupport` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/nf-d3d12-id3d12device-checkfeaturesupport |
| `D3D12_FEATURE` 列挙型 | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/ne-d3d12-d3d12_feature |
| `IDXGIFactory6::EnumAdapterByGpuPreference` | https://learn.microsoft.com/ja-jp/windows/win32/api/dxgi1_6/nf-dxgi1_6-idxgifactory6-enumadapterbygpupreference |
| `ID3D12InfoQueue1::RegisterMessageCallback` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12sdklayers/nf-d3d12sdklayers-id3d12infoqueue1-registermessagecallback |
| DXR 機能仕様(Tier 1.2 / SER) | https://microsoft.github.io/DirectX-Specs/d3d/Raytracing.html |
| D3D12 Shader Execution Reordering | https://devblogs.microsoft.com/directx/ser/ |
| D3D12 Opacity Micromaps | https://devblogs.microsoft.com/directx/omm/ |

> **本章の記述の基準日:2026 年 7 月 31 日**
>
> Agility SDK 1.619(retail)のヘッダを前提としています。`OPTIONS22` のように番号の大きい機能グループは、SDK のバージョンによって存在しないことがあります。**`QueryFeature` の戻り値を確認する設計にしてあるので、存在しない環境でも安全に動作します。**
