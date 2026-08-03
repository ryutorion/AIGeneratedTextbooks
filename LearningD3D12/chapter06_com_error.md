# 第6章 COMとエラー処理の土台

次章でついに Direct3D 12 のデバイスを作ります。その前に、**失敗を検出する仕組み**を用意します。

D3D12 の API は、ほぼすべてが `HRESULT` を返します。そして本書のスタイル —— ヘルパーを使わずに構造体を手で埋める —— では、埋め忘れや値の取り違えが日常的に起こります。そのとき返ってくるのは `E_INVALIDARG` という 8 桁の数字だけです。

**どの呼び出しが、どのファイルの何行目で、なぜ失敗したのか。** これが即座にわかる状態を作っておくかどうかで、この先の 30 章の体験がまったく変わります。

あわせて、D3D12 オブジェクトの生存期間を管理する `ComPtr` の正確な挙動を押さえます。ここを曖昧にしたまま進むと、第21章で「リソースリークがゼロにならない」と何時間も悩むことになります。

**本章のゴール**
`std::expected` によるエラー伝搬、`std::source_location` を使ったログとアサート、そして COM オブジェクトへの名前付けを整備する。わざと失敗させ、発生箇所つきのエラーが表示されることを確認する。

---

## 6.1 COM と `IUnknown`、参照カウントの仕組み

### 6.1.1 D3D12 は「COM のようなもの」で作られている

`ID3D12Device`、`ID3D12CommandQueue`、`IDXGIFactory6` —— これらはすべて **COM インターフェース**です。名前が `I` で始まるのはその慣習によります。

ただし、**D3D12 はフル装備の COM を使っていません。** 次のものは登場しません。

| COM の機能 | D3D12 では |
|---|---|
| `CoInitializeEx` によるアパートメント初期化 | **不要** |
| `CoCreateInstance` によるオブジェクト生成 | 使わない(専用の生成関数を使う) |
| レジストリへのクラス登録 | なし |
| マーシャリング、プロキシ | なし |

実際に使われるのは、**`IUnknown` が定める 3 つのメソッドだけ**です。この最小限の使い方は、俗に「nano-COM」と呼ばれます。

> **`CoInitializeEx` は呼ばなくてよい**
>
> 「COM を使うなら `CoInitializeEx` が必要」というのは正しい一般論ですが、D3D12 と DXGI には当てはまりません。呼ばなくても動きます。
>
> ただし、将来 WIC(Windows Imaging Component)で PNG を読むといったことをすると必要になります。**本書は DDS ローダを自作する(第20章)ので、最後まで不要です。**

### 6.1.2 `IUnknown` の 3 つのメソッド

すべての COM インターフェースは `IUnknown` を継承しており、次の 3 つを持ちます。

```cpp
class IUnknown
{
    virtual HRESULT QueryInterface(REFIID riid, void** ppvObject) = 0;
    virtual ULONG   AddRef() = 0;
    virtual ULONG   Release() = 0;
};
```

| メソッド | 役割 |
|---|---|
| `AddRef` | 参照カウントを 1 増やす |
| `Release` | 参照カウントを 1 減らす。**0 になったらオブジェクトが自分を破棄する** |
| `QueryInterface` | 「あなたは別のインターフェースも実装していますか?」と尋ねる |

### 6.1.3 参照カウントの規約

COM の生存期間管理は、次の 2 つの規約に集約されます。

> **規約 1:オブジェクトを受け取ったら、使い終わったときに `Release` する。**
> **規約 2:ポインタを複製して長く保持するなら、`AddRef` してから保持する。**

D3D12 の生成関数(`D3D12CreateDevice` など)が返すオブジェクトは、**参照カウント 1 の状態**で渡されます。つまり、受け取った側が `Release` する責任を負います。

```cpp
ID3D12Device* device = nullptr;
::D3D12CreateDevice(nullptr, D3D_FEATURE_LEVEL_11_0, IID_PPV_ARGS(&device));
// ここで device の参照カウントは 1

// ... 使う ...

device->Release();   // 忘れるとリーク
```

**この `Release` を手で書くのが、あらゆる悲劇の元です。** 途中で `return` した経路、例外が飛んだ経路、`if` で分岐した経路 —— どれか一つでも漏れればリークします。逆に二重に呼べばクラッシュします。

だから `ComPtr` を使います。

### 6.1.4 インターフェースのバージョン

D3D12 の API は拡張されるたびに、**新しいインターフェースを追加する**という方法で進化してきました。

```
ID3D12Device
  └─ ID3D12Device1
       └─ ID3D12Device2
            └─ ...
                 └─ ID3D12Device14   (Agility SDK で追加)
```

各インターフェースは前の版を継承しており、新しいメソッドが増えています。**古いインターフェースの定義は決して変更されません。** これが COM の後方互換性の仕組みです。

生成関数は基本版(`ID3D12Device`)を返すので、新しい機能を使いたければ `QueryInterface` で問い合わせます。

```cpp
// 第30章の Enhanced Barriers は ID3D12GraphicsCommandList7 以降が必要
ComPtr<ID3D12GraphicsCommandList7> list7;
if (SUCCEEDED(list.As(&list7)))
{
    list7->Barrier(...);
}
```

**失敗したら、その機能は使えません。** ドライバか Agility SDK か GPU が対応していないということです(第4章 4.5.1 節の「4 つの条件の積」)。

### 6.1.5 `IID_PPV_ARGS` マクロ

`QueryInterface` や生成関数は、「どのインターフェースが欲しいか」を GUID で指定し、結果を `void**` で受け取ります。

```cpp
HRESULT D3D12CreateDevice(IUnknown* pAdapter,
                          D3D_FEATURE_LEVEL MinimumFeatureLevel,
                          REFIID riid,          // 欲しいインターフェースの GUID
                          void** ppDevice);     // 型情報が失われている
```

`riid` と `ppDevice` の型が食い違っていても**コンパイルは通ります**。実行時に `E_NOINTERFACE` が返るか、最悪の場合は誤った型のポインタを掴みます。

これを防ぐのが `IID_PPV_ARGS` マクロです。

```cpp
::D3D12CreateDevice(nullptr, D3D_FEATURE_LEVEL_11_0, IID_PPV_ARGS(&device));
//                                                   ^^^^^^^^^^^^^^^^^^^^^
//                                    引数 2 つ分に展開される
```

このマクロは、渡されたポインタの型から GUID を自動で導出します。**必ずこれを使ってください。** GUID を手で書く理由はありません。

なお、GUID の実体は `dxguid.lib` にあります。第4章でリンク設定を済ませています。

---

## 6.2 `Microsoft::WRL::ComPtr` の使い方

### 6.2.1 なぜ自作せず `ComPtr` を使うのか

第1章 1.3.2 節で述べた通り、本書は `ComPtr` を使います。方針の再確認をしておきます。

本書が自作するのは、**レンダラの設計判断を伴うもの**です。デスクリプタの管理方法、リソースの寿命戦略、バリアの張り方 —— これらは「どう作るか」に答えがなく、考えることに意味があります。

一方 `ComPtr` は、**COM の規約を機械的に守るだけの道具**です。自作しても、参照カウントの規約を再実装するだけで、学びは「規約を理解した」以上には広がりません。そして実務で事故が多いのは、自作の失敗よりも**既製品の誤用**です。

したがって本書は、`ComPtr` を使ったうえで、**その挙動を正確に理解すること**に紙面を割きます。

### 6.2.2 基本的な使い方

`ComPtr` は `<wrl/client.h>` にあります(第3章で `pch.h` に追加済み)。

```cpp
using Microsoft::WRL::ComPtr;

ComPtr<ID3D12Device> device;
::D3D12CreateDevice(nullptr, D3D_FEATURE_LEVEL_11_0, IID_PPV_ARGS(&device));
// ここで device の参照カウントは 1

device->CreateCommandQueue(...);   // operator-> で生のインターフェースにアクセス

// スコープを抜けると自動的に Release される
```

主なメンバは次の通りです。

| メンバ | 動作 |
|---|---|
| `Get()` | 生ポインタを返す。**`AddRef` しない** |
| `GetAddressOf()` | `T**` を返す。**現在の値を解放しない** |
| `ReleaseAndGetAddressOf()` | **解放してから** `T**` を返す |
| `operator&` | `ReleaseAndGetAddressOf()` と同じ |
| `Reset()` | 解放して null にする |
| `As(&other)` | `QueryInterface` を行う。`HRESULT` を返す |
| `Detach()` | 所有権を放棄して生ポインタを返す |
| `Attach(p)` | `AddRef` せずに所有権を受け取る |

### 6.2.3 `&` と `GetAddressOf()` と `ReleaseAndGetAddressOf()` の違い

**ここが `ComPtr` でもっとも誤解される部分です。**

WRL の `ComPtr` において、**`operator&` は `ReleaseAndGetAddressOf()` と等価です。**

```cpp
ComPtr<ID3D12Device> device;
CreateSomething(&device);                       // 中身を解放してから書き込む
CreateSomething(device.ReleaseAndGetAddressOf());  // ↑ と同じ
CreateSomething(device.GetAddressOf());         // 解放せずに書き込む ← 危険
```

3 行目が問題です。`device` がすでに何かを保持している状態でこれを呼ぶと、**古いオブジェクトの参照カウントが減らないまま、ポインタだけが上書きされます。** つまりリークです。

```cpp
ComPtr<ID3D12Fence> fence;
device->CreateFence(0, flags, IID_PPV_ARGS(fence.GetAddressOf()));  // 1 個目
device->CreateFence(0, flags, IID_PPV_ARGS(fence.GetAddressOf()));  // ❌ 1 個目がリーク
```

**本書のルール:生成関数に渡すときは常に `&` を使う。**

```cpp
device->CreateFence(0, flags, IID_PPV_ARGS(&fence));   // ✅
```

`&` なら、既存の値があっても正しく解放されてから書き込まれます。何度呼んでも安全です。

> **ATL の `CComPtr` とは挙動が違う**
>
> ATL の `CComPtr` では、`operator&` は「中身が null であること」をアサートします。null でなければデバッグビルドで停止します。
>
> WRL の `ComPtr` は黙って解放します。**同じ名前の型なのに、`&` の意味が違います。** 他のコードベースから移ってきた場合は注意してください。

### 6.2.4 `Get()` の使いどころと落とし穴

`Get()` は生ポインタを返します。**`AddRef` しません。** つまり、返ってきたポインタは `ComPtr` が生きている間しか有効ではありません。

```cpp
// ✅ 正しい:関数に一時的に渡すだけ
queue->ExecuteCommandLists(1, commandLists);
::D3D12CreateDevice(adapter.Get(), level, IID_PPV_ARGS(&device));
```

```cpp
// ❌ 誤り:ComPtr の寿命を超えて保持している
ID3D12Device* raw = nullptr;
{
    ComPtr<ID3D12Device> device;
    CreateDevice(&device);
    raw = device.Get();
}   // ここで device が解放される
raw->CreateFence(...);   // 解放済みオブジェクトへのアクセス
```

**`Get()` の戻り値をメンバ変数に保存しないでください。** 保存したいなら `ComPtr` のまま保持します。

### 6.2.5 `As()` によるインターフェース問い合わせ

`QueryInterface` の薄いラッパーです。

```cpp
ComPtr<ID3D12Device>   device;
ComPtr<ID3D12Device14> device14;

if (SUCCEEDED(device.As(&device14)))
{
    // device14 が使える(参照カウントは +1 されている)
}
else
{
    // このランタイム/ドライバでは ID3D12Device14 は使えない
}
```

**`As()` は `HRESULT` を返します。戻り値を無視しないでください。** 失敗時に `device14` は null のままなので、チェックせずに使うとアクセス違反になります。

`As()` が成功すると、`device` と `device14` は**同じオブジェクトを指す 2 本の参照**になります。参照カウントは 2 です。どちらも独立して解放できます。

> **どの版のインターフェースを要求すべきか**
>
> 「常に最新版を要求すればいい」わけではありません。要求する版が新しいほど、動く環境が狭くなります。
>
> 本書の方針は、**使う機能が要求する最小の版を要求する**です。第30章で Enhanced Barriers を使うときに `ID3D12GraphicsCommandList7` を要求し、それ以外の場所では基本版を使います。

### 6.2.6 よくある誤用と参照リーク

**誤用 1:手動で `Release` を呼ぶ**

```cpp
ComPtr<ID3D12Device> device;
CreateDevice(&device);
device.Get()->Release();   // ❌ 二重解放になる
```

デストラクタがもう一度 `Release` します。**`ComPtr` を使うなら、`Release` を書く必要は一切ありません。**

**誤用 2:`GetAddressOf()` を使う(6.2.3)**

**誤用 3:メンバの宣言順序を意識しない**

C++ のメンバは、**宣言と逆順**に破棄されます。

```cpp
class Renderer
{
    ComPtr<ID3D12Device>       m_device;      // 先に宣言 → 後で破棄
    ComPtr<ID3D12CommandQueue> m_queue;       // 後に宣言 → 先に破棄
};
```

D3D12 では、子オブジェクト(コマンドキューなど)が親のデバイスへの参照を保持しています。したがって**デバイスを先に宣言しておけば、子より後に解放され**、生存期間の逆転が起きません。

**本書のルール:デバイスをもっとも先に宣言する。**

これは第21章でリソースリークをゼロにする際に効いてきます。`ReportLiveDeviceObjects` は、デバイスが生きている状態で呼ぶ必要があるからです。

**誤用 4:`ComPtr` を `void*` にキャストして渡す**

```cpp
SomeFunction(reinterpret_cast<void**>(&comptr));   // ❌
```

`ComPtr::operator&` は生の `T**` ではなく、変換用のプロキシ型を返します。`reinterpret_cast` すると壊れます。`IID_PPV_ARGS(&comptr)` を使ってください。

---

## 6.3 `HRESULT` と `std::expected` によるエラー伝搬

### 6.3.1 `HRESULT` の読み方

`HRESULT` は 32bit の整数で、最上位ビットが成功/失敗を表します。

```
 31  30-16          15-0
┌───┬──────────────┬──────────────┐
│ S │  Facility    │     Code     │
└───┴──────────────┴──────────────┘
  0 = 成功 / 1 = 失敗
```

**判定には必ずマクロを使ってください。**

```cpp
if (SUCCEEDED(hr)) { ... }   // ✅
if (FAILED(hr))    { ... }   // ✅

if (hr == S_OK)    { ... }   // ❌ 危険
```

なぜ `hr == S_OK` が危険か。**成功を表す値は `S_OK`(0)だけではない**からです。

第4章で `D3D12CreateDevice` の第 4 引数に `nullptr` を渡したとき、成功時の戻り値は `S_OK` ではなく **`S_FALSE`(1)** でした。`hr == S_OK` で判定していたら、成功を失敗と誤認します。

### 6.3.2 D3D12 で頻出する `HRESULT`

| 定数 | 意味 | よくある原因 |
|---|---|---|
| `S_OK` | 成功 | |
| `S_FALSE` | 成功だが「条件を満たさない」 | 問い合わせ専用の呼び出し |
| `E_INVALIDARG` | 引数が不正 | **構造体の埋め忘れ、値の取り違え** |
| `E_OUTOFMEMORY` | メモリ不足 | VRAM 枯渇、巨大なリソース |
| `E_NOINTERFACE` | そのインターフェースは未実装 | `As()` で新しすぎる版を要求した |
| `E_FAIL` | 一般的な失敗 | 原因が絞れない。デバッグレイヤーを見る |
| `DXGI_ERROR_NOT_FOUND` | 見つからない | アダプタ列挙の終端 |
| `DXGI_ERROR_UNSUPPORTED` | 非対応 | フォーマットや機能レベルが未対応 |
| `DXGI_ERROR_DEVICE_REMOVED` (0x887A0005) | **デバイスロスト** | 第8章・第31章の主題 |
| `DXGI_ERROR_DEVICE_HUNG` (0x887A0006) | **GPU ハング** | 同上 |
| `DXGI_ERROR_SDK_COMPONENT_MISSING` | デバッグレイヤーがない | 第4章 4.6.3 節を参照 |

**本書のスタイルで圧倒的に多いのは `E_INVALIDARG` です。** `d3dx12.h` を使わずに構造体を埋めるので、初期化漏れや値の取り違えが起きます。そして `E_INVALIDARG` だけでは**どのフィールドが悪いのかわかりません。**

その答えを教えてくれるのがデバッグレイヤーです。第7章で有効にします。本章では「どの呼び出しが失敗したか」までを特定できるようにします。

### 6.3.3 なぜ例外ではなく `std::expected` なのか

DirectX の公式サンプルは、次のようなヘルパーを使っています。

```cpp
inline void ThrowIfFailed(HRESULT hr)
{
    if (FAILED(hr)) throw HrException(hr);
}
```

簡潔で、実務でも広く使われている良い方法です。**本書はこれを採らず、`std::expected` を使います。** 理由は 3 つあります。

**理由 1:失敗しうることが型に現れる**

```cpp
Result<ComPtr<ID3D12Device>> CreateDevice(IDXGIAdapter1* adapter);
```

この宣言を見れば、呼び出し側は失敗を扱う必要があると即座にわかります。`ThrowIfFailed` 方式では、関数の中身を読むまでわかりません。**学習用の本としては、失敗の経路が見えているほうが良い**と判断しました。

**理由 2:Win32 のコールバックと例外の相性が悪い**

第5章で書いた `WndProc` は、OS から呼ばれる C ABI の関数です。ここを例外が通り抜けると、動作は未定義です。D3D12 アプリケーションではメッセージ処理と描画が絡み合うので、例外の到達範囲を管理するのが面倒になります。

**理由 3:C++23 を使う本だから**

`std::expected` は C++23 の目玉機能の一つです。せっかく `/std:c++23preview` を選んだのですから、実戦で使ってみる価値があります。

> **例外が劣っているわけではありません**
>
> 公平のために書いておくと、`ThrowIfFailed` 方式には明確な利点があります。**コードが圧倒的に短くなります。** 初期化関数が 30 行の `HRESULT` チェックで埋まることがありません。
>
> 実務のプロジェクトでどちらを選ぶかは、チームの方針と既存コードによります。**本書の選択を唯一の正解と受け取らないでください。**

### 6.3.4 エラー型を定義する

まず、文字列変換のユーティリティを用意します。DXC のエラーメッセージ(第13章)など、UTF-8 の文字列を扱う場面が今後たびたび出てきます。

```cpp
// src/Core/StringUtil.h
#pragma once
#include "std_import.h"

namespace Core
{
    // UTF-8 → UTF-16
    std::wstring ToWide(std::string_view utf8);

    // UTF-16 → UTF-8
    std::string  ToUtf8(std::wstring_view utf16);
}
```

```cpp
// src/Core/StringUtil.cpp
#include "pch.h"
#include "std_import.h"
#if USE_STD_MODULE
import std;
#endif
#include "Core/StringUtil.h"

namespace Core
{
    std::wstring ToWide(std::string_view utf8)
    {
        if (utf8.empty()) return {};

        const int length = ::MultiByteToWideChar(
            CP_UTF8, 0, utf8.data(), static_cast<int>(utf8.size()),
            nullptr, 0);
        if (length <= 0) return {};

        std::wstring result(static_cast<std::size_t>(length), L'\0');
        ::MultiByteToWideChar(
            CP_UTF8, 0, utf8.data(), static_cast<int>(utf8.size()),
            result.data(), length);
        return result;
    }

    std::string ToUtf8(std::wstring_view utf16)
    {
        if (utf16.empty()) return {};

        const int length = ::WideCharToMultiByte(
            CP_UTF8, 0, utf16.data(), static_cast<int>(utf16.size()),
            nullptr, 0, nullptr, nullptr);
        if (length <= 0) return {};

        std::string result(static_cast<std::size_t>(length), '\0');
        ::WideCharToMultiByte(
            CP_UTF8, 0, utf16.data(), static_cast<int>(utf16.size()),
            result.data(), length, nullptr, nullptr);
        return result;
    }
}
```

次に、エラー型です。

```cpp
// src/Core/Error.h
#pragma once
#include "std_import.h"

namespace Core
{
    //-----------------------------------------------------------
    // 失敗の情報
    //-----------------------------------------------------------
    struct Error
    {
        HRESULT              hr = E_FAIL;
        std::wstring         context;   // 失敗した式、または説明
        std::source_location where;     // 発生箇所
    };

    // 成功時に値 T を返し、失敗時に Error を返す型
    template <typename T>
    using Result = std::expected<T, Error>;

    // 値を返さない場合
    using Status = std::expected<void, Error>;

    //-----------------------------------------------------------
    // HRESULT を人間が読める文字列にする
    //-----------------------------------------------------------
    std::wstring FormatHResult(HRESULT hr);

    // Error を 1 行の文字列にする
    std::wstring FormatError(const Error& error);

    // Error をログに出力する
    void ReportError(const Error& error);

    //-----------------------------------------------------------
    // Error の生成
    //-----------------------------------------------------------
    [[nodiscard]] inline Error MakeError(
        HRESULT hr,
        std::wstring_view context,
        const std::source_location& where = std::source_location::current())
    {
        return Error{ hr, std::wstring(context), where };
    }
}

//---------------------------------------------------------------
// HRESULT を返す式を評価し、失敗したら呼び出し元に伝搬する
//
//   Status Renderer::Initialize()
//   {
//       HR_TRY(::CreateDXGIFactory2(0, IID_PPV_ARGS(&m_factory)));
//       HR_TRY(::D3D12CreateDevice(...));
//       return {};
//   }
//---------------------------------------------------------------
#define HR_WIDEN_IMPL(x) L##x
#define HR_WIDEN(x)      HR_WIDEN_IMPL(x)

#define HR_TRY(expr)                                                       \
    do {                                                                   \
        const HRESULT hr_ = (expr);                                        \
        if (FAILED(hr_)) {                                                 \
            return std::unexpected(                                        \
                ::Core::MakeError(hr_, HR_WIDEN(#expr)));                  \
        }                                                                  \
    } while (false)
```

**`HR_WIDEN` の 2 段構えについて。** `#expr` はナロー文字列リテラルを作ります。これをワイドにするには `L` を前置しますが、`L#expr` と書いても展開順序の都合で意図通りになりません。いったん別のマクロに渡して展開を確定させてから連結するのが定石です。

**`std::source_location::current()` を既定引数にしている点も重要です。** マクロの展開位置で評価されるので、`HR_TRY` を書いた行が記録されます。

### 6.3.5 `HRESULT` を文字列に変換する

```cpp
// src/Core/Error.cpp
#include "pch.h"
#include "std_import.h"
#if USE_STD_MODULE
import std;
#endif
#include "Core/Error.h"
#include "Core/Log.h"
#include "Core/StringUtil.h"

namespace Core
{
    namespace
    {
        // FormatMessageW では出ない、あるいは説明が不十分なものを補う
        constexpr std::wstring_view KnownHResultName(HRESULT hr)
        {
            switch (hr)
            {
            case S_OK:                            return L"S_OK";
            case S_FALSE:                         return L"S_FALSE";
            case E_FAIL:                          return L"E_FAIL";
            case E_INVALIDARG:                    return L"E_INVALIDARG";
            case E_OUTOFMEMORY:                   return L"E_OUTOFMEMORY";
            case E_NOINTERFACE:                   return L"E_NOINTERFACE";
            case E_NOTIMPL:                       return L"E_NOTIMPL";
            case E_ACCESSDENIED:                  return L"E_ACCESSDENIED";
            case DXGI_ERROR_INVALID_CALL:         return L"DXGI_ERROR_INVALID_CALL";
            case DXGI_ERROR_NOT_FOUND:            return L"DXGI_ERROR_NOT_FOUND";
            case DXGI_ERROR_UNSUPPORTED:          return L"DXGI_ERROR_UNSUPPORTED";
            case DXGI_ERROR_DEVICE_REMOVED:       return L"DXGI_ERROR_DEVICE_REMOVED";
            case DXGI_ERROR_DEVICE_HUNG:          return L"DXGI_ERROR_DEVICE_HUNG";
            case DXGI_ERROR_DEVICE_RESET:         return L"DXGI_ERROR_DEVICE_RESET";
            case DXGI_ERROR_DRIVER_INTERNAL_ERROR:return L"DXGI_ERROR_DRIVER_INTERNAL_ERROR";
            case DXGI_ERROR_WAS_STILL_DRAWING:    return L"DXGI_ERROR_WAS_STILL_DRAWING";
            case DXGI_ERROR_SDK_COMPONENT_MISSING:return L"DXGI_ERROR_SDK_COMPONENT_MISSING";
            default:                              return {};
            }
        }

        std::wstring SystemMessage(HRESULT hr)
        {
            wchar_t* buffer = nullptr;
            const DWORD length = ::FormatMessageW(
                FORMAT_MESSAGE_ALLOCATE_BUFFER |
                FORMAT_MESSAGE_FROM_SYSTEM |
                FORMAT_MESSAGE_IGNORE_INSERTS,
                nullptr,
                static_cast<DWORD>(hr),
                MAKELANGID(LANG_NEUTRAL, SUBLANG_DEFAULT),
                reinterpret_cast<LPWSTR>(&buffer),
                0, nullptr);

            if (length == 0 || buffer == nullptr)
            {
                return {};
            }

            std::wstring message(buffer, length);
            ::LocalFree(buffer);

            // 末尾の改行を削る
            while (!message.empty() &&
                   (message.back() == L'\r' || message.back() == L'\n'))
            {
                message.pop_back();
            }
            return message;
        }
    }

    std::wstring FormatHResult(HRESULT hr)
    {
        const std::wstring_view name = KnownHResultName(hr);
        const std::wstring      text = SystemMessage(hr);

        if (!name.empty() && !text.empty())
        {
            return std::format(L"{} (0x{:08X}) {}", name,
                               static_cast<std::uint32_t>(hr), text);
        }
        if (!name.empty())
        {
            return std::format(L"{} (0x{:08X})", name,
                               static_cast<std::uint32_t>(hr));
        }
        if (!text.empty())
        {
            return std::format(L"0x{:08X} {}",
                               static_cast<std::uint32_t>(hr), text);
        }
        return std::format(L"0x{:08X}", static_cast<std::uint32_t>(hr));
    }

    std::wstring FormatError(const Error& error)
    {
        return std::format(L"{}\n    -> {}",
                           FormatHResult(error.hr), error.context);
    }

    void ReportError(const Error& error)
    {
        Log::WriteRaw(LogLevel::Error, error.where, FormatError(error));
    }
}
```

### 6.3.6 使い方

```cpp
Core::Result<ComPtr<IDXGIFactory6>> CreateFactory(bool enableDebug)
{
    const UINT flags = enableDebug ? DXGI_CREATE_FACTORY_DEBUG : 0;

    ComPtr<IDXGIFactory6> factory;
    HR_TRY(::CreateDXGIFactory2(flags, IID_PPV_ARGS(&factory)));

    return factory;
}
```

呼び出し側:

```cpp
auto factory = CreateFactory(true);
if (!factory)
{
    Core::ReportError(factory.error());
    return 1;
}

factory.value()->EnumAdapters1(0, &adapter);
```

**`std::expected` は単項演算子で真偽を判定できます。** `if (!factory)` は「失敗した」を意味します。値へのアクセスは `.value()` または `operator*` です。

### 6.3.7 モナド演算について

C++23 の `std::expected` は `and_then` / `transform` / `or_else` を備えており、成功時だけ処理を継続する記述ができます。

```cpp
auto device = CreateFactory(true)
    .and_then(SelectAdapter)
    .and_then(CreateDevice);
```

美しい書き方ですが、**本書ではあまり使いません。** D3D12 の初期化は「A を作り、A を使って B を作り、A と B を使って C を作る」という形が多く、直前の結果だけを受け渡すモナド連鎖に素直に乗らないからです。

無理にチェーンするより、`HR_TRY` で早期リターンするほうが読みやすくなります。**道具は形に合わせて選びます。**

---

## 6.4 `std::source_location` を使ったログとアサート

### 6.4.1 ログの設計

第5章で作った `Debug.h` の `DebugPrint` を、本節で正式なログ機能に置き換えます。

必要な機能は次の 4 つです。

- レベル(Trace / Info / Warning / Error / Fatal)による絞り込み
- 発生箇所(ファイル名と行番号)の自動付与
- `std::format` による型安全な書式化
- Visual Studio の「出力」ウィンドウへの出力

```cpp
// src/Core/Log.h
#pragma once
#include "std_import.h"

namespace Core
{
    // 【注意】ERROR という名前は使えない。
    // wingdi.h が #define ERROR 0 しているため
    // (第3章 3.4.5 節を参照)。
    enum class LogLevel
    {
        Trace,
        Info,
        Warning,
        Error,
        Fatal,
    };

    namespace Log
    {
        void SetMinLevel(LogLevel level);
        LogLevel MinLevel();

        // 書式化済みの文字列を出力する
        void WriteRaw(LogLevel level,
                      const std::source_location& where,
                      std::wstring_view text);

        template <typename... Args>
        void Write(LogLevel level,
                   const std::source_location& where,
                   std::wformat_string<Args...> fmt,
                   Args&&... args)
        {
            if (level < MinLevel())
            {
                return;
            }
            WriteRaw(level, where,
                     std::format(fmt, std::forward<Args>(args)...));
        }
    }
}

//---------------------------------------------------------------
// ログマクロ
//
// マクロを使う理由:
//   std::source_location::current() を「呼び出し元の位置」で
//   評価させたいが、可変長引数の後ろに既定引数は置けない。
//   クラステンプレート + 推論補助で回避する手法もあるが、
//   マクロのほうが単純で確実。
//---------------------------------------------------------------
#define LOG_TRACE(...) \
    ::Core::Log::Write(::Core::LogLevel::Trace, \
                       std::source_location::current(), __VA_ARGS__)

#define LOG_INFO(...) \
    ::Core::Log::Write(::Core::LogLevel::Info, \
                       std::source_location::current(), __VA_ARGS__)

#define LOG_WARN(...) \
    ::Core::Log::Write(::Core::LogLevel::Warning, \
                       std::source_location::current(), __VA_ARGS__)

#define LOG_ERROR(...) \
    ::Core::Log::Write(::Core::LogLevel::Error, \
                       std::source_location::current(), __VA_ARGS__)

#define LOG_FATAL(...) \
    ::Core::Log::Write(::Core::LogLevel::Fatal, \
                       std::source_location::current(), __VA_ARGS__)
```

> **`ERROR` という名前を避けた理由**
>
> 第3章 3.4.5 節で予告した通りです。`wingdi.h` が `#define ERROR 0` しているため、
>
> ```cpp
> enum class LogLevel { Trace, Info, Warning, ERROR, Fatal };
> ```
>
> と書くと `ERROR` が `0` に置換され、`Warning, 0, Fatal` という構文エラーになります。**予告した地雷を、実際に踏まずに回避できました。**

実装です。

```cpp
// src/Core/Log.cpp
#include "pch.h"
#include "std_import.h"
#if USE_STD_MODULE
import std;
#endif
#include "Core/Log.h"
#include "Core/StringUtil.h"

namespace Core
{
    namespace
    {
        LogLevel g_minLevel =
#if defined(_DEBUG)
            LogLevel::Trace;
#else
            LogLevel::Info;
#endif

        constexpr std::wstring_view LevelTag(LogLevel level)
        {
            switch (level)
            {
            case LogLevel::Trace:   return L"Trace";
            case LogLevel::Info:    return L"Info ";
            case LogLevel::Warning: return L"Warn ";
            case LogLevel::Error:   return L"Error";
            case LogLevel::Fatal:   return L"Fatal";
            }
            return L"?????";
        }

        // フルパスからファイル名だけを取り出す
        constexpr std::string_view FileNameOnly(std::string_view path)
        {
            const auto pos = path.find_last_of("/\\");
            return (pos == std::string_view::npos)
                 ? path : path.substr(pos + 1);
        }
    }

    void     Log::SetMinLevel(LogLevel level) { g_minLevel = level; }
    LogLevel Log::MinLevel()                  { return g_minLevel; }

    void Log::WriteRaw(LogLevel level,
                       const std::source_location& where,
                       std::wstring_view text)
    {
        if (level < g_minLevel)
        {
            return;
        }

        // ファイル名は const char* なのでワイドに変換する。
        // 第3章でパスに日本語を使わないと決めたので ASCII だが、
        // 変換関数は共通のものを使っておく。
        const std::wstring file =
            ToWide(FileNameOnly(where.file_name()));

        const std::wstring line = std::format(
            L"[{}] {}({}): {}\n",
            LevelTag(level), file, where.line(), text);

        ::OutputDebugStringW(line.c_str());
    }
}
```

出力例:

```
[Info ] Window.cpp(88): created: client 1280 x 720 (dpi 96)
[Error] main.cpp(52): DXGI_ERROR_NOT_FOUND (0x887A0002) ...
```

**Visual Studio の「出力」ウィンドウでは、`ファイル名(行番号):` の形式をダブルクリックすると該当行にジャンプできます。** この書式は偶然ではなく、そのために選んでいます。

### 6.4.2 アサート

```cpp
// src/Core/Assert.h
#pragma once
#include "std_import.h"
#include "Core/Log.h"

namespace Core::Detail
{
    inline void AssertFailed(std::wstring_view expression,
                             std::wstring_view message,
                             const std::source_location& where)
    {
        if (message.empty())
        {
            Log::WriteRaw(LogLevel::Fatal, where,
                std::format(L"assertion failed: {}", expression));
        }
        else
        {
            Log::WriteRaw(LogLevel::Fatal, where,
                std::format(L"assertion failed: {}  ({})",
                            expression, message));
        }

        if (::IsDebuggerPresent())
        {
            __debugbreak();
        }
    }
}

#define ASSERT_WIDEN_IMPL(x) L##x
#define ASSERT_WIDEN(x)      ASSERT_WIDEN_IMPL(x)

#if defined(_DEBUG)

    // Debug ビルドでのみ評価される
    #define D3D_ASSERT(cond)                                              \
        do {                                                              \
            if (!(cond)) {                                                \
                ::Core::Detail::AssertFailed(                             \
                    ASSERT_WIDEN(#cond), L"",                             \
                    std::source_location::current());                     \
            }                                                             \
        } while (false)

    #define D3D_ASSERT_MSG(cond, msg)                                     \
        do {                                                              \
            if (!(cond)) {                                                \
                ::Core::Detail::AssertFailed(                             \
                    ASSERT_WIDEN(#cond), (msg),                           \
                    std::source_location::current());                     \
            }                                                             \
        } while (false)

#else

    #define D3D_ASSERT(cond)          ((void)0)
    #define D3D_ASSERT_MSG(cond, msg) ((void)0)

#endif

//---------------------------------------------------------------
// Release でも式を評価する版。
// 副作用のある式を書いてしまう事故を防ぐために分けている。
//---------------------------------------------------------------
#define D3D_VERIFY(cond)                                                  \
    do {                                                                  \
        if (!(cond)) {                                                    \
            ::Core::Detail::AssertFailed(                                 \
                ASSERT_WIDEN(#cond), L"",                                 \
                std::source_location::current());                         \
        }                                                                 \
    } while (false)
```

**設計上のポイントを 3 つ。**

**1. `assert` を使わない理由**
第3章 3.4.5 節で述べた通り、`import std;` は**マクロをエクスポートしません**。`assert` を使うには `<cassert>` を `#include` する必要があり、鉄則 3 と衝突します。自作したほうが制約に悩みません。

**2. `IsDebuggerPresent()` で分岐する理由**
デバッガが接続されていないときに `__debugbreak()` を実行すると、**アプリが即座に異常終了します。** エンドユーザーの環境で発生すると、何の情報も残りません。デバッガがいるときだけ止めます。

**3. `D3D_ASSERT` と `D3D_VERIFY` を分けた理由**
`D3D_ASSERT` は Release で式ごと消えます。したがって、

```cpp
D3D_ASSERT(InitializeSomething());   // ❌ Release で呼ばれなくなる
```

という書き方は事故になります。副作用のある式を書きたい場合は `D3D_VERIFY` を使ってください。**名前を分けておくことで、レビュー時に気づけます。**

> **`std::stacktrace` は使わないのか**
>
> C++23 には `<stacktrace>` があり、`std::stacktrace::current()` で呼び出し履歴を取得できます。魅力的ですが、本書のアサートには組み込みません。
>
> 理由は、**`__debugbreak()` で止まればデバッガが呼び出し履歴を表示してくれる**からです。開発中に必要な情報は、すでに Visual Studio が提供しています。取得コストも小さくありません。
>
> エンドユーザーの環境でクラッシュ情報を集めたい場合には価値があります。そのときに導入を検討してください。

### 6.4.3 第5章の `Debug.h` を置き換える

`src/Debug.h` は役目を終えました。削除し、`Window.cpp` の `DebugPrint` を `LOG_*` に置き換えます。

```cpp
// 変更前
DebugPrint(L"[Window] created: client {} x {} (dpi {})\n", ...);

// 変更後
LOG_INFO(L"window created: client {} x {} (dpi {})", ...);
```

**末尾の `\n` が不要**になっている点に注意してください。改行はログ側で付けます。また、`[Window]` という接頭辞も不要です。ファイル名が自動で入るからです。

---

## 6.5 `SetName()` によるリソース名付けを習慣にする

### 6.5.1 名前を付けるとは

すべての D3D12 オブジェクトは `ID3D12Object` を継承しており、`SetName` を持っています。

```cpp
device->SetName(L"MainDevice");
commandQueue->SetName(L"DirectQueue");
backBuffer->SetName(L"BackBuffer[0]");
```

これは単なるメモではありません。**名前は、以下のすべてに現れます。**

| ツール | 名前が現れる場所 |
|---|---|
| D3D12 デバッグレイヤー | 警告・エラーメッセージ本文 |
| Nsight Graphics | リソース一覧、フレームデバッガ |
| PIX | イベントリスト、リソースビュー |
| DRED | デバイスロスト時の自動ブレッドクラム |
| **Nsight Aftermath** | **ページフォルトを起こしたリソースの特定** |

### 6.5.2 この習慣が Aftermath で効いてくる

第31章の内容を先取りします。GPU がクラッシュしたとき、Aftermath のクラッシュダンプにはこう書かれます。

**名前を付けていない場合**

```
Page fault at GPU virtual address 0x0000000204A00000
  Resource: <unnamed>  (size 8388608, D3D12_RESOURCE_DIMENSION_BUFFER)
```

アドレスとサイズだけです。数十のバッファを作っていたら、どれのことか特定できません。

**名前を付けている場合**

```
Page fault at GPU virtual address 0x0000000204A00000
  Resource: "InstanceBuffer (frame 2)"  (size 8388608, ...)
```

**一発でわかります。**

第31章で「意図的にクラッシュさせて原因行を特定する」実習を行いますが、そこで効いてくるのは第31章に書くコードではなく、**第7章から積み上げてきた名前付けの習慣**です。だから本章で仕組みを作り、以後すべてのオブジェクトに適用します。

### 6.5.3 ヘルパーを書く

```cpp
// src/Core/DebugName.h
#pragma once
#include "std_import.h"

//---------------------------------------------------------------
// D3D12 オブジェクトにデバッグ名を付ける。
//
// D3D12_ENABLE_DEBUG_NAMES を 0 にすると、名前付けが消える。
// Aftermath を製品版に組み込む場合は 1 のままにする(第31章)。
//---------------------------------------------------------------
#if !defined(D3D12_ENABLE_DEBUG_NAMES)
    #define D3D12_ENABLE_DEBUG_NAMES 1
#endif

namespace Core
{
#if D3D12_ENABLE_DEBUG_NAMES

    inline void SetDebugName(ID3D12Object* object, std::wstring_view name)
    {
        if (object != nullptr)
        {
            object->SetName(std::wstring(name).c_str());
        }
    }

    template <typename... Args>
    void SetDebugNameF(ID3D12Object* object,
                       std::wformat_string<Args...> fmt,
                       Args&&... args)
    {
        if (object != nullptr)
        {
            object->SetName(
                std::format(fmt, std::forward<Args>(args)...).c_str());
        }
    }

#else

    inline void SetDebugName(ID3D12Object*, std::wstring_view) {}

    template <typename... Args>
    void SetDebugNameF(ID3D12Object*, std::wformat_string<Args...>, Args&&...) {}

#endif
}

//---------------------------------------------------------------
// 変数名をそのままデバッグ名にする。
//   D3D_NAME(m_commandQueue);  →  名前は L"m_commandQueue"
//---------------------------------------------------------------
#define NAME_WIDEN_IMPL(x) L##x
#define NAME_WIDEN(x)      NAME_WIDEN_IMPL(x)

#define D3D_NAME(obj) ::Core::SetDebugName((obj).Get(), NAME_WIDEN(#obj))
```

使い方:

```cpp
// 変数名をそのまま使う
D3D_NAME(m_device);
D3D_NAME(m_commandQueue);

// 添字などを含めたい場合
Core::SetDebugNameF(m_backBuffers[i].Get(), L"BackBuffer[{}]", i);
```

### 6.5.4 Release ビルドでどうするか

名前付けにはコストがあります。文字列の保持のために、オブジェクトごとに小さなメモリ確保が発生します。

伝統的には「Debug のみ有効、Release では消す」とされてきました。**本書はこの判断を保留します。**

理由は、**Aftermath を製品版に組み込むなら、名前は Release でこそ必要だから**です(第31章 31.6 節)。エンドユーザーの環境で発生したクラッシュのダンプに `<unnamed>` しか書かれていなければ、集める意味が薄れます。

そこで `D3D12_ENABLE_DEBUG_NAMES` という独立したマクロで制御し、**既定では Release でも有効**にしました。コストが問題になったときに、意識的に切れるようにしてあります。

**本書のルール:D3D12 オブジェクトを作ったら、その場で名前を付ける。**

第7章以降、すべての生成コードの直後に `D3D_NAME` が並びます。冗長に見えるかもしれませんが、**第31章での 30 分と、第20章での 3 時間を節約する投資**だと考えてください。

---

## ✅ 本章のゴール:わざと失敗させる

用意した仕組みが動くことを確認します。**成功する処理だけを書いても、エラー処理が正しいかはわかりません。**

### `main.cpp`

```cpp
// src/main.cpp
#include "pch.h"
#include "std_import.h"
#if USE_STD_MODULE
import std;
#endif
#include "Window.h"
#include "Core/Log.h"
#include "Core/Error.h"
#include "Core/Assert.h"
#include "Core/DebugName.h"

using Microsoft::WRL::ComPtr;

namespace
{
    //-----------------------------------------------------------
    // 実験 1:成功するはずの処理
    //-----------------------------------------------------------
    Core::Result<ComPtr<IDXGIFactory6>> CreateFactory()
    {
        ComPtr<IDXGIFactory6> factory;
        HR_TRY(::CreateDXGIFactory2(0, IID_PPV_ARGS(&factory)));
        return factory;
    }

    //-----------------------------------------------------------
    // 実験 2:存在しないアダプタを要求する
    //          → DXGI_ERROR_NOT_FOUND
    //-----------------------------------------------------------
    Core::Status EnumerateMissingAdapter(IDXGIFactory6* factory)
    {
        ComPtr<IDXGIAdapter1> adapter;
        HR_TRY(factory->EnumAdapters1(999, &adapter));
        return {};
    }

    //-----------------------------------------------------------
    // 実験 3:無関係なインターフェースを問い合わせる
    //          → E_NOINTERFACE
    //-----------------------------------------------------------
    Core::Status QueryWrongInterface(IDXGIFactory6* factory)
    {
        ComPtr<ID3D12Device> device;
        HR_TRY(factory->QueryInterface(IID_PPV_ARGS(&device)));
        return {};
    }

    //-----------------------------------------------------------
    // 実験 4:Win32 の失敗を HRESULT に変換する
    //-----------------------------------------------------------
    Core::Status OpenMissingFile()
    {
        const HANDLE handle = ::CreateFileW(
            L"NoSuchFile.bin", GENERIC_READ, 0, nullptr,
            OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, nullptr);

        if (handle == INVALID_HANDLE_VALUE)
        {
            return std::unexpected(Core::MakeError(
                HRESULT_FROM_WIN32(::GetLastError()),
                L"CreateFileW(L\"NoSuchFile.bin\")"));
        }

        ::CloseHandle(handle);
        return {};
    }

    void RunErrorHandlingTests()
    {
        LOG_INFO(L"--- error handling tests ---");

        auto factory = CreateFactory();
        if (!factory)
        {
            Core::ReportError(factory.error());
            return;
        }
        LOG_INFO(L"DXGI factory created");

        // 以下はいずれも失敗するのが正しい
        if (auto r = EnumerateMissingAdapter(factory->Get()); !r)
        {
            Core::ReportError(r.error());
        }

        if (auto r = QueryWrongInterface(factory->Get()); !r)
        {
            Core::ReportError(r.error());
        }

        if (auto r = OpenMissingFile(); !r)
        {
            Core::ReportError(r.error());
        }

        LOG_INFO(L"--- assertion test ---");
        const int value = 42;
        D3D_ASSERT_MSG(value == 0, L"わざと失敗させています");

        LOG_INFO(L"--- tests finished ---");
    }
}

int WINAPI wWinMain(
    _In_     HINSTANCE hInstance,
    _In_opt_ HINSTANCE hPrevInstance,
    _In_     PWSTR     pCmdLine,
    _In_     int       nCmdShow)
{
    (void)hInstance; (void)hPrevInstance;
    (void)pCmdLine;  (void)nCmdShow;

    ::SetProcessDpiAwarenessContext(DPI_AWARENESS_CONTEXT_PER_MONITOR_AWARE_V2);

    RunErrorHandlingTests();

    Window window;
    window.OnResize = [](int width, int height)
    {
        LOG_INFO(L"resize: {} x {}", width, height);
    };

    if (!window.Create(L"D3D12Book - Chapter 6", 1280, 720))
    {
        ::MessageBoxW(nullptr, L"ウィンドウの作成に失敗しました。",
                      L"D3D12Book", MB_OK | MB_ICONERROR);
        return 1;
    }

    while (window.ProcessMessages())
    {
        if (window.IsMinimized())
        {
            continue;
        }
    }

    LOG_INFO(L"exit");
    return 0;
}
```

### 実行結果

**`F5`(デバッグ開始)で実行し、「出力」ウィンドウを見てください。**

```
[Info ] main.cpp(78): --- error handling tests ---
[Info ] main.cpp(85): DXGI factory created
[Error] main.cpp(37): DXGI_ERROR_NOT_FOUND (0x887A0002) 指定されたエンティティが見つかりませんでした。
    -> factory->EnumAdapters1(999, &adapter)
[Error] main.cpp(48): E_NOINTERFACE (0x80004002) インターフェイスがサポートされていません。
    -> factory->QueryInterface(IID_PPV_ARGS(&device))
[Error] main.cpp(66): 0x80070002 指定されたファイルが見つかりません。
    -> CreateFileW(L"NoSuchFile.bin")
[Info ] main.cpp(104): --- assertion test ---
[Fatal] main.cpp(107): assertion failed: value == 0  (わざと失敗させています)
```

**確認すべき点は 4 つです。**

1. **行番号が、失敗した `HR_TRY` の位置を指している**
   `RunErrorHandlingTests` の行ではなく、`EnumerateMissingAdapter` の中の行が出ています。`std::source_location` が正しく機能しています。

2. **失敗した式がそのまま表示されている**
   `-> factory->EnumAdapters1(999, &adapter)` の部分です。どの呼び出しかを探す必要がありません。

3. **`HRESULT` に名前と説明が付いている**
   `0x887A0002` だけでは調べる手間がかかります。

4. **アサートで `__debugbreak()` が発生し、デバッガが停止する**
   停止したら `F5` で続行してください。呼び出し履歴も確認できます。

**「出力」ウィンドウの `main.cpp(37):` をダブルクリックすると、その行にジャンプします。** これが 6.4.1 節で書式を選んだ理由です。

---

### 本章の達成状態

- [ ] `src/Core/` に `StringUtil` / `Log` / `Error` / `Assert` / `DebugName` を作成した
- [ ] 第5章の `Debug.h` を削除し、`Window.cpp` を `LOG_*` に置き換えた
- [ ] `HR_TRY` で失敗が呼び出し元に伝搬する
- [ ] エラーにファイル名・行番号・失敗した式が含まれる
- [ ] `HRESULT` が名前と説明つきで表示される
- [ ] `D3D_ASSERT` が発火し、デバッガが停止する
- [ ] `D3D_NAME` マクロを用意した(使うのは第7章から)

---

## トラブルシューティング

| 症状 | 原因 | 対処 |
|---|---|---|
| `error C2065: 'expected'` | C++23 が有効でない | `/std:c++23preview` を確認(第3章) |
| `LOG_INFO` で行番号が全部同じ | `source_location` をマクロ外で評価している | マクロ定義を確認(6.4.1) |
| `L#cond` でコンパイルエラー | ワイド化の 2 段構えを省略した | `ASSERT_WIDEN` を使う(6.4.2) |
| 日本語のログが文字化けする | `OutputDebugStringA` を使っている | ワイド版に統一する |
| `ERROR` で構文エラー | `wingdi.h` のマクロ | 名前を `Error` にする(6.4.1) |
| `unresolved external symbol IID_...` | `dxguid.lib` 未リンク | 第4章 4.2.6 節 |
| リソースが解放されない | `GetAddressOf()` を使っている | `&` に変える(6.2.3) |
| 終了時にクラッシュ | 手動で `Release` を呼んでいる | `ComPtr` に任せる(6.2.6) |
| アサートで即終了する | デバッガなしで実行した | `F5` で起動する |

---

## まとめ

**1. `ComPtr::operator&` は解放してから書き込む。**
`GetAddressOf()` は解放しません。生成関数に渡すときは常に `&` を使ってください。この違いが参照リークの主要な原因です。

**2. 成功判定に `hr == S_OK` を使わない。**
`S_FALSE` も成功です。`SUCCEEDED` / `FAILED` マクロを使ってください。

**3. `std::expected` で失敗を型に載せる。**
関数の宣言を見ただけで失敗しうることがわかります。例外方式より冗長ですが、学習用としては失敗の経路が見えているほうが良いと判断しました。

**4. `std::source_location` は既定引数として渡す。**
マクロの展開位置で評価されるので、呼び出し元の行番号が記録されます。`__FILE__` / `__LINE__` を手で書き回す必要はありません。

**5. オブジェクトを作ったら、その場で名前を付ける。**
デバッグレイヤー、Nsight、PIX、そして Aftermath のすべてで名前が使われます。**第31章でページフォルトを起こしたリソースを特定できるかどうかは、今日からの習慣で決まります。**

次章で、いよいよ Direct3D 12 のデバイスを作ります。デバッグレイヤーを有効にし、NVIDIA のアダプタを確実に選び、`CheckFeatureSupport` で対応機能を一覧出力します。**本章で作った道具が、そこから最後まで働き続けます。**

---

## 参考リンク

| 内容 | URL |
|---|---|
| COM の基礎(Learn to Program for Windows) | https://learn.microsoft.com/ja-jp/windows/win32/learnwin32/component-object-model--com--in-windows-programming |
| `ComPtr` クラス | https://learn.microsoft.com/ja-jp/cpp/cppcx/wrl/comptr-class |
| `IID_PPV_ARGS` | https://learn.microsoft.com/ja-jp/windows/win32/api/combaseapi/nf-combaseapi-iid_ppv_args |
| `ID3D12Object::SetName` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/nf-d3d12-id3d12object-setname |
| `std::expected`(cppreference) | https://ja.cppreference.com/w/cpp/utility/expected |
| `std::source_location`(cppreference) | https://ja.cppreference.com/w/cpp/utility/source_location |
