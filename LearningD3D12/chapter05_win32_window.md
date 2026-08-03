# 第5章 Win32ウィンドウを作る

4 章にわたる配管工事が終わりました。本章の終わりには、ようやく画面に何かが表示されます。

とはいえ、出るのは灰色の四角いウィンドウです。Direct3D はまだ一行も呼びません。「グラフィックスプログラミングの本を読んでいるのに、なぜ Win32 API の話を?」と思うかもしれません。

理由は 2 つあります。

**1 つ目。Direct3D 12 のスワップチェーンは、`HWND` を要求します。** 第11章で `CreateSwapChainForHwnd` を呼ぶとき、その第 2 引数にウィンドウハンドルを渡します。ウィンドウなしでは画面に絵を出せません。

**2 つ目。ウィンドウのリサイズは、描画側に直接影響します。** ウィンドウの大きさが変わればバックバッファも作り直す必要があり(第12章)、最小化されたときは幅も高さも 0 になるのでスワップチェーンを作れません。**ウィンドウの状態管理を雑にすると、第12章で必ず落ちます。**

本章は「とりあえずウィンドウが出ればいい」ではなく、**後の章で困らない土台**を作ることを目的とします。

**本章のゴール**
1280×720 のクライアント領域を持つウィンドウを表示し、リサイズと終了が正しく動くことを確認する。リサイズ時にはデバッグ出力にサイズが表示される。

---

## 5.0 その前に:自作ヘッダをどこで `#include` するか

本章で初めて、自分で書いたヘッダファイル(`Window.h`)を作ります。ここで第3章のルールに補足が必要になります。

第3章 3.5.4 節では、すべての `.cpp` の書き出しをこう定めました。

```cpp
#include "pch.h"
#include "std_import.h"
#if USE_STD_MODULE
import std;
#endif
```

では `Window.h` はどこに入れるのか。**答えは `import std;` の後です。**

```cpp
#include "pch.h"          // ① Windows / D3D12(PCH)
#include "std_import.h"   // ②
#if USE_STD_MODULE
import std;               // ③ 標準ライブラリ
#endif
#include "Window.h"       // ④ 自作ヘッダ ← ここ
```

### なぜ後なのか

第3章の鉄則 1 は「`#include` が先、`import` が後」でした。これは**標準ライブラリの内容を持ち込むヘッダ**についての規則です。`<vector>` を `import std;` の後に include すると、同じ宣言が二重に現れて壊れます。

`Window.h` は標準ヘッダを一切 include しません。自分で書いた宣言しか含まないので、二重定義は起きません。むしろ `Window.h` の中で `std::wstring_view` を使うためには、**その時点で `std` が可視になっている必要があります**。だから `import std;` の後に置きます。

### 自作ヘッダの側のルール

この構成が成立するために、自作ヘッダには次の制約を課します。

> **自作ヘッダは標準ライブラリのヘッダを `#include` しない。**
> `std` は、そのヘッダを取り込む翻訳単位が用意済みであることを前提とする。

**これは明確なデメリットを伴います。** ヘッダが自己完結しなくなるからです。`Window.h` を単独で include してもコンパイルは通りません。C++ の常識からすれば行儀の悪い設計です。

**`import std;` を使う以上、これは避けられない代償です。** 標準ライブラリだけがモジュール化され、それ以外はヘッダのままという過渡期にいる限り、どこかで妥協が必要になります。本書は「翻訳単位の先頭で `std` を用意する」という一点に妥協を集中させ、それ以外の場所には持ち込まない方針を採ります。

なお、第3章で作った `std_import.h` を各ヘッダの先頭に置いておくと、退避モード(`USE_STD_MODULE = 0`)のときだけ標準ヘッダが取り込まれ、ヘッダが自己完結します。本書のヘッダはこの形で書きます。

---

## 5.1 `wWinMain` とエントリポイント

### 5.1.1 サブシステムを変更する

第3章では、コンソールアプリケーションとしてプロジェクトを構成しました。本章でウィンドウアプリケーションに切り替えます。

`プロジェクトのプロパティ → 構成プロパティ → リンカー → システム`

| 項目 | 変更前 | 変更後 |
|---|---|---|
| サブシステム | コンソール (`/SUBSYSTEM:CONSOLE`) | **Windows (`/SUBSYSTEM:WINDOWS`)** |

**すべての構成に対して**変更してください。

サブシステムを変更すると、リンカが探すエントリポイントも変わります。

| サブシステム | エントリポイント(CRT) | 呼ばれる関数 |
|---|---|---|
| CONSOLE | `mainCRTStartup` | `main` / `wmain` |
| WINDOWS | `wWinMainCRTStartup` | `WinMain` / **`wWinMain`** |

`main` のままビルドすると、次のエラーになります。

```
error LNK2019: 未解決の外部シンボル WinMain が関数 "int __cdecl
__scrt_common_main_seh(void)" で参照されました
```

### 5.1.2 `wWinMain` を書く

`src/main.cpp` を書き換えます。まずは骨組みだけです。

```cpp
// src/main.cpp
#include "pch.h"
#include "std_import.h"
#if USE_STD_MODULE
import std;
#endif

int WINAPI wWinMain(
    _In_     HINSTANCE hInstance,
    _In_opt_ HINSTANCE hPrevInstance,
    _In_     PWSTR     pCmdLine,
    _In_     int       nCmdShow)
{
    ::MessageBoxW(nullptr, L"Hello, Direct3D 12!", L"D3D12Book", MB_OK);
    return 0;
}
```

**引数の意味**

| 引数 | 内容 | 本書での扱い |
|---|---|---|
| `hInstance` | このモジュールのインスタンスハンドル | 使うが、後述の理由で引数からは受け取らない |
| `hPrevInstance` | **常に `nullptr`** | 使わない |
| `pCmdLine` | コマンドライン(プログラム名を除く) | 使わない |
| `nCmdShow` | 最初にどう表示するか | `ShowWindow` に渡す |

`hPrevInstance` は 16bit Windows の遺産で、Win32 では常に `nullptr` です。**30 年以上、誰の役にも立っていない引数**が今も残っています。

`_In_` や `_In_opt_` は SAL(Source-code Annotation Language)注釈です。静的解析ツールに「この引数は入力専用」「null を許容する」といった情報を伝えます。省略してもコンパイルは通りますが、Windows SDK の宣言と一致させておくと解析の精度が上がります。

### 5.1.3 `wWinMain` と `WinMain` の違い

`w` の有無は、コマンドライン引数の文字型です。

| 関数 | `pCmdLine` の型 |
|---|---|
| `WinMain` | `PSTR`(ANSI) |
| `wWinMain` | `PWSTR`(ワイド文字) |

第3章で `UNICODE` / `_UNICODE` を定義したので、**`wWinMain` を使います。** 本書ではコマンドライン引数を使いませんが、統一のためです。

### 5.1.4 `HINSTANCE` を引数から受け取らない理由

`hInstance` はウィンドウクラスの登録に必要です。しかし本書では、引数から受け取らずに次のように取得します。

```cpp
const HINSTANCE hInstance = ::GetModuleHandleW(nullptr);
```

`GetModuleHandleW(nullptr)` は「現在のプロセスの実行ファイルのモジュールハンドル」を返します。exe においては、これは `wWinMain` の `hInstance` と**同じ値**です。

なぜわざわざこうするのか。**`Window` クラスが `wWinMain` に依存しなくなるからです。** 引数で渡す設計にすると、`Window::Create` を呼ぶすべての場所に `hInstance` を持ち回る必要が生じます。テストコードから呼ぶときも、DLL 化したときも面倒です。

「どこからでも取れる値を、わざわざ引数で運ばない」という判断です。

---

## 5.2 ウィンドウクラス登録と `CreateWindowEx`

### 5.2.1 ウィンドウクラスとは

Win32 でウィンドウを作るには、2 段階の手順を踏みます。

```
① RegisterClassExW  … ウィンドウの「型」を OS に登録する
        ↓
② CreateWindowExW   … その型の「実体」を作る
```

ウィンドウクラスは、C++ のクラスとは無関係です。**ウィンドウの共通の性質(どの関数がメッセージを処理するか、カーソルは何か、背景をどう塗るか)をまとめた設定**だと考えてください。同じクラスから複数のウィンドウを作れます。

### 5.2.2 `WNDCLASSEXW` を埋める

```cpp
WNDCLASSEXW wc{};
wc.cbSize        = sizeof(WNDCLASSEXW);
wc.style         = CS_HREDRAW | CS_VREDRAW;
wc.lpfnWndProc   = &Window::StaticWndProc;
wc.cbClsExtra    = 0;
wc.cbWndExtra    = 0;
wc.hInstance     = hInstance;
wc.hIcon         = nullptr;
wc.hCursor       = ::LoadCursorW(nullptr, IDC_ARROW);
wc.hbrBackground = nullptr;
wc.lpszMenuName  = nullptr;
wc.lpszClassName = L"D3D12BookWindowClass";
wc.hIconSm       = nullptr;
```

**重要なフィールドを 4 つ**説明します。

#### `lpfnWndProc` —— メッセージ処理関数

このウィンドウクラスに属するウィンドウ宛のメッセージを処理する関数です。**ここには通常の関数ポインタしか入りません。** C++ のメンバ関数は入れられないので、5.4 節で橋渡しの仕組みを作ります。

#### `hCursor` —— カーソル

```cpp
wc.hCursor = ::LoadCursorW(nullptr, IDC_ARROW);
```

`nullptr` のままにすると、**マウスをウィンドウ上に動かしたとき、直前のカーソル形状がそのまま残ります。** 他のウィンドウの端でリサイズカーソルになった状態でこちらに入ると、矢印に戻りません。地味ですが目に見えるバグです。

第1引数の `nullptr` は「システム標準のカーソルを使う」という意味です。自作のカーソルリソースを使う場合はここにインスタンスハンドルを渡します。

#### `hbrBackground` —— 背景ブラシ

```cpp
wc.hbrBackground = nullptr;   // 背景を塗らせない
```

**ここは `nullptr` にします。** Direct3D で描画する領域を、Windows に塗らせる必要はないからです。

`(HBRUSH)(COLOR_WINDOW + 1)` のような値を入れると、`WM_ERASEBKGND` のたびに Windows が背景を塗ります。その直後に D3D が同じ場所を上書きするので、**リサイズ中にちらつき(フリッカー)が出ます。**

`nullptr` にすると `DefWindowProcW` は背景を塗らず、何もせずに戻ります。

#### `style` —— クラススタイル

```cpp
wc.style = CS_HREDRAW | CS_VREDRAW;
```

横幅・縦幅が変わったときに、クライアント領域全体を無効化(再描画対象に)するフラグです。D3D アプリでは毎フレーム全画面を描き直すので実害はありませんが、リサイズ時の挙動が素直になります。

### 5.2.3 `CreateWindowExW` を呼ぶ

```cpp
m_hwnd = ::CreateWindowExW(
    0,                          // 拡張スタイル
    L"D3D12BookWindowClass",    // ウィンドウクラス名
    L"D3D12Book",               // タイトルバーの文字列
    WS_OVERLAPPEDWINDOW,        // ウィンドウスタイル
    CW_USEDEFAULT,              // x 座標
    CW_USEDEFAULT,              // y 座標
    windowWidth,                // 幅(枠を含む)
    windowHeight,               // 高さ(枠を含む)
    nullptr,                    // 親ウィンドウ
    nullptr,                    // メニュー
    hInstance,                  // インスタンスハンドル
    this);                      // 追加パラメータ ← 5.4 節で使う
```

`WS_OVERLAPPEDWINDOW` は、次のスタイルの組み合わせです。

| 含まれるスタイル | 効果 |
|---|---|
| `WS_OVERLAPPED` | タイトルバーと枠 |
| `WS_CAPTION` | タイトルバー |
| `WS_SYSMENU` | システムメニューと閉じるボタン |
| `WS_THICKFRAME` | **サイズ変更可能な枠** |
| `WS_MINIMIZEBOX` | 最小化ボタン |
| `WS_MAXIMIZEBOX` | 最大化ボタン |

最後の引数 `this` が重要です。**このポインタは `WM_NCCREATE` メッセージで受け取れます。** 5.4 節でこれを使います。

### 5.2.4 クライアント領域のサイズを正確に指定する

**ここが本節でもっとも実用的な部分です。**

`CreateWindowExW` に渡す幅と高さは、**枠とタイトルバーを含んだウィンドウ全体のサイズ**です。しかし我々が欲しいのは、**中身(クライアント領域)のサイズ**です。

Direct3D のバックバッファはクライアント領域と同じ大きさで作ります。1280×720 のバックバッファが欲しいなら、クライアント領域を正確に 1280×720 にしなければなりません。

枠の太さは、Windows のバージョン、テーマ、DPI によって変わります。決め打ちはできません。**Win32 API に計算させます。**

```cpp
const UINT dpi = ::GetDpiForSystem();

RECT rect{ 0, 0, clientWidth, clientHeight };
::AdjustWindowRectExForDpi(&rect, style, FALSE, exStyle, dpi);

const int windowWidth  = rect.right  - rect.left;
const int windowHeight = rect.bottom - rect.top;
```

`AdjustWindowRectExForDpi` は、**「このクライアント領域を得るには、ウィンドウ全体をどれだけの大きさにすればよいか」**を計算します。第 3 引数の `FALSE` は「メニューバーなし」という意味です。

> **`AdjustWindowRect` ではなく `AdjustWindowRectExForDpi`**
>
> 末尾に `ForDpi` の付かない古い版もありますが、そちらは DPI を考慮しません。高 DPI 環境では枠の太さが変わるので、計算がずれます。
>
> ずれた結果どうなるか。バックバッファが 1280×720 なのにクライアント領域が 1272×713、といった状態になり、**画面の端に描かれない帯が出ます。** 目立たないぶん、原因を突き止めにくいバグです。

なお、この時点ではウィンドウがまだ存在しないので、そのウィンドウが表示されるモニタの DPI はわかりません。`GetDpiForSystem()` でシステム既定の DPI を使い、実際に別 DPI のモニタに表示されたら `WM_DPICHANGED` で調整します(5.5 節)。

---

## 5.3 メッセージループ

### 5.3.1 `GetMessage` 型と `PeekMessage` 型

Win32 のメッセージループには 2 つの型があります。

**`GetMessage` 型**

```cpp
MSG msg{};
while (::GetMessageW(&msg, nullptr, 0, 0) > 0)
{
    ::TranslateMessage(&msg);
    ::DispatchMessageW(&msg);
}
```

`GetMessageW` は、メッセージが来るまで**ブロックします**。CPU を消費しないので、テキストエディタのような「入力があったときだけ動く」アプリに適しています。

**`PeekMessage` 型**

```cpp
while (running)
{
    MSG msg{};
    while (::PeekMessageW(&msg, nullptr, 0, 0, PM_REMOVE))
    {
        ::TranslateMessage(&msg);
        ::DispatchMessageW(&msg);
    }
    // ここで毎フレームの更新と描画を行う
    Update();
    Render();
}
```

`PeekMessageW` は、メッセージがなければ**すぐに `FALSE` を返します**。溜まっているメッセージを処理し終えたら、描画に移ります。

**本書は `PeekMessage` 型を使います。** リアルタイムに描画し続けるアプリケーションでは、これ以外の選択肢がありません。「入力がないから何もしない」わけにはいかないからです。

### 5.3.2 `WM_QUIT` の扱い

**`WM_QUIT` は特別なメッセージです。** ウィンドウ宛ではなく、**スレッド宛**に送られます。したがって `DispatchMessageW` に渡しても `WndProc` には届きません。

ループの中で明示的に判定する必要があります。

```cpp
bool Window::ProcessMessages()
{
    MSG msg{};
    while (::PeekMessageW(&msg, nullptr, 0, 0, PM_REMOVE))
    {
        if (msg.message == WM_QUIT)
        {
            m_shouldClose = true;
            return false;
        }
        ::TranslateMessage(&msg);
        ::DispatchMessageW(&msg);
    }
    return !m_shouldClose;
}
```

`PeekMessageW` の第 2 引数に `nullptr` を渡しているのがポイントです。ここに `HWND` を指定すると**そのウィンドウ宛のメッセージだけ**が取得され、スレッド宛の `WM_QUIT` が永久に取れなくなります。**アプリが終了しなくなります。**

### 5.3.3 `TranslateMessage` は何をしているのか

```cpp
::TranslateMessage(&msg);
```

キーボードの `WM_KEYDOWN` を見て、文字入力に相当するものであれば `WM_CHAR` を生成してメッセージキューに追加します。「A キーが押された」を「文字 'a' が入力された」に翻訳する処理です。

本書では文字入力を扱いませんが、**将来 ImGui などを組み込むときに必要になる**ので残しておきます。

---

## 5.4 `WndProc` をクラスのメンバ関数へ橋渡しする

### 5.4.1 何が問題なのか

`WNDCLASSEXW::lpfnWndProc` の型は次の通りです。

```cpp
LRESULT (CALLBACK *WNDPROC)(HWND, UINT, WPARAM, LPARAM);
```

**通常の関数ポインタです。** C++ の非静的メンバ関数は、暗黙の `this` を持つため、この型には代入できません。

一方、我々はメンバ変数(クライアントサイズ、リサイズ中フラグなど)にアクセスしたい。**関数ポインタとオブジェクトを結びつける仕組みが必要です。**

### 5.4.2 グローバル変数を使ってはいけない

もっとも安易な解決は、グローバル変数に `Window*` を置くことです。

```cpp
Window* g_window = nullptr;   // ❌
```

動きますが、**ウィンドウが 1 つしか作れなくなります。** 本書では 1 つしか作りませんが、この制約はいずれ必ず邪魔になります。ツールウィンドウを追加したくなったとき、設計をやり直すことになります。

### 5.4.3 `GWLP_USERDATA` を使う

Win32 は、**各ウィンドウにポインタ 1 つ分の領域**を用意しています。`SetWindowLongPtrW` / `GetWindowLongPtrW` でアクセスします。

ここに `this` を保存すれば、`HWND` から `Window*` を引けます。

**問題は、いつ保存するかです。**

`CreateWindowExW` が返る前に、いくつものメッセージが `WndProc` に届きます。返り値の `HWND` を待っていては間に合いません。

そこで、`CreateWindowExW` の最後の引数に渡した `this` を使います。この値は **`WM_NCCREATE`** メッセージの `lParam` から、`CREATESTRUCTW::lpCreateParams` として取り出せます。

### 5.4.4 橋渡しの実装

```cpp
LRESULT CALLBACK Window::StaticWndProc(
    HWND hwnd, UINT msg, WPARAM wParam, LPARAM lParam)
{
    Window* self = nullptr;

    if (msg == WM_NCCREATE)
    {
        // CreateWindowExW の最後の引数がここに届く
        const auto* create = reinterpret_cast<const CREATESTRUCTW*>(lParam);
        self = static_cast<Window*>(create->lpCreateParams);

        self->m_hwnd = hwnd;
        ::SetWindowLongPtrW(hwnd, GWLP_USERDATA,
                            reinterpret_cast<LONG_PTR>(self));
    }
    else
    {
        self = reinterpret_cast<Window*>(
            ::GetWindowLongPtrW(hwnd, GWLP_USERDATA));
    }

    if (self != nullptr)
    {
        return self->WndProc(hwnd, msg, wParam, lParam);
    }

    // まだ this が結びついていないメッセージは既定の処理へ
    return ::DefWindowProcW(hwnd, msg, wParam, lParam);
}
```

**最後の `DefWindowProcW` を省略してはいけません。**

`WM_NCCREATE` は最初のメッセージではありません。実際には次の順序で届きます。

```
WM_GETMINMAXINFO   ← まだ this が結びついていない
WM_NCCREATE        ← ここで結びつける
WM_NCCALCSIZE
WM_CREATE
...
```

`WM_GETMINMAXINFO` は `WM_NCCREATE` より**先に**来ます。この時点では `GetWindowLongPtrW` は `0` を返すので、`self` は `nullptr` です。ここで `self->WndProc(...)` を呼ぶと**アクセス違反でクラッシュします。**

`nullptr` チェックと `DefWindowProcW` へのフォールバックは、必須の安全装置です。

> **`WM_NCCREATE` で `FALSE` を返すとウィンドウ作成が失敗する**
>
> `WM_NCCREATE` の戻り値には意味があります。`FALSE`(0)を返すと、`CreateWindowExW` は `nullptr` を返して失敗します。
>
> 上のコードでは `self->WndProc(...)` に処理を委ね、その中で `DefWindowProcW` を呼ぶので `TRUE` が返ります。**`WM_NCCREATE` を `case` で拾って `return 0;` などと書かないでください。** ウィンドウが作られなくなります。

---

## 5.5 リサイズ・終了処理・高DPI対応

### 5.5.1 終了処理のメッセージ連鎖

ウィンドウを閉じるとき、4 つのメッセージが順に流れます。**この連鎖を理解していないと、「閉じても終わらない」「終わったのに落ちる」といった問題を解けません。**

```
ユーザーが × をクリック
        ↓
① WM_CLOSE        「閉じてよいか?」
        ↓          DestroyWindow(hwnd) を呼ぶ
② WM_DESTROY      「ウィンドウが破棄された」
        ↓          PostQuitMessage(0) を呼ぶ
③ WM_QUIT         スレッドのメッセージキューに積まれる
        ↓
④ メッセージループが WM_QUIT を検出して終了
```

各段階での役割は次の通りです。

| メッセージ | 誰が何をするか |
|---|---|
| `WM_CLOSE` | 終了確認ダイアログを出すならここ。`DestroyWindow` を呼ぶと次へ進む |
| `WM_DESTROY` | ウィンドウはもう存在しない。`PostQuitMessage` でループに終了を伝える |
| `WM_QUIT` | `WndProc` には届かない。ループが自分で判定する(5.3.2) |

**`WM_CLOSE` を処理せずに `DefWindowProcW` に任せても動きます**(既定の処理が `DestroyWindow` を呼ぶため)。しかし明示的に書いたほうが、後で「終了前に保存しますか?」を挟むときに困りません。

### 5.5.2 リサイズを扱う

#### `WM_SIZE` の情報

```cpp
case WM_SIZE:
    m_clientWidth  = LOWORD(lParam);
    m_clientHeight = HIWORD(lParam);
    m_minimized    = (wParam == SIZE_MINIMIZED);
    return 0;
```

`lParam` に**新しいクライアント領域のサイズ**が入っています。これがそのままバックバッファの必要サイズです。

`wParam` には変化の種類が入ります。

| 値 | 意味 |
|---|---|
| `SIZE_RESTORED` | 通常のリサイズ |
| `SIZE_MINIMIZED` | 最小化された |
| `SIZE_MAXIMIZED` | 最大化された |

#### 最小化の罠

**最小化されると、クライアント領域は 0×0 になります。**

第12章でスワップチェーンの `ResizeBuffers` を呼ぶとき、0 を渡すと失敗します。デバッグレイヤーが警告を出し、場合によってはデバイスロストに至ります。

そこで、最小化状態をフラグとして保持し、**描画もリサイズ処理もスキップする**ようにします。

```cpp
m_minimized = (wParam == SIZE_MINIMIZED);
```

第12章のレンダリングループは、このフラグを見て早期リターンします。**今のうちに用意しておくことで、後の章で不可解なクラッシュに遭わずに済みます。**

#### ドラッグ中の連続リサイズ

ウィンドウの端をドラッグすると、**マウスを動かすたびに `WM_SIZE` が飛んできます。** 1 回のドラッグで数百回になることもあります。

`WM_SIZE` のたびにバックバッファを作り直すと、GPU リソースの生成と破棄が大量に発生し、目に見えて重くなります。

対策として、ドラッグの開始と終了を検出します。

```cpp
case WM_ENTERSIZEMOVE:
    m_resizing = true;
    return 0;

case WM_EXITSIZEMOVE:
    m_resizing = false;
    NotifyResize();       // ここで 1 回だけ通知
    return 0;

case WM_SIZE:
    m_clientWidth  = LOWORD(lParam);
    m_clientHeight = HIWORD(lParam);
    m_minimized    = (wParam == SIZE_MINIMIZED);
    if (!m_resizing && !m_minimized)
    {
        NotifyResize();   // ドラッグ以外(最大化など)は即座に通知
    }
    return 0;
```

`WM_ENTERSIZEMOVE` / `WM_EXITSIZEMOVE` は、**ユーザーが枠をドラッグしている間だけ**送られます。最大化ボタンや `SetWindowPos` によるサイズ変更では飛んできません。だから `WM_SIZE` 側にも通知経路を残しています。

> **既知の問題:ドラッグ中はメインループが止まる**
>
> ウィンドウの枠をドラッグしている間、Windows は `DefWindowProcW` の内部で**独自のメッセージループ**を回します。我々の `while (window.ProcessMessages())` は、その間まったく実行されません。
>
> 結果として、**ドラッグ中は画面が更新されません。** 描画したものが残り続けるか、環境によっては黒くなります。
>
> 解決策はいくつかあります(`WM_ENTERSIZEMOVE` で `SetTimer` を仕掛け、`WM_TIMER` で描画する等)。第12章で描画ループを完成させる際に扱います。**今の段階では「そういうもの」と認識しておいてください。**

### 5.5.3 最小サイズを制限する

クライアント領域が極端に小さくなると、レンダリング側で除算が発生する箇所(アスペクト比の計算など)で問題が起きます。下限を設けます。

```cpp
case WM_GETMINMAXINFO:
{
    auto* info = reinterpret_cast<MINMAXINFO*>(lParam);
    info->ptMinTrackSize.x = 320;
    info->ptMinTrackSize.y = 240;
    return 0;
}
```

`ptMinTrackSize` は**ウィンドウ全体**のサイズなので、クライアント領域はこれより小さくなります。厳密にやるなら `AdjustWindowRectExForDpi` で換算しますが、下限としては十分です。

なお 5.4.4 節で述べた通り、**このメッセージは `WM_NCCREATE` より先に届きます。** 最初の 1 回は `self` が `nullptr` のため `DefWindowProcW` が処理し、この制限は適用されません。実害はありません。

### 5.5.4 高 DPI 対応

#### なぜ必要か

Windows の表示スケールが 100% でない環境(ノート PC ではむしろこちらが普通です)で、DPI 非対応のアプリケーションを動かすとどうなるか。

**Windows がアプリの出力を拡大表示します。** 1280×720 で描いた絵が、システム側で 1920×1080 に引き伸ばされます。結果はぼやけた画面です。

Direct3D で緻密に描いたものが、最後の最後で拡大フィルタを通される。**これほど無駄なことはありません。**

#### DPI 認識を宣言する

`wWinMain` の**最初**、ウィンドウを作る前に次を呼びます。

```cpp
::SetProcessDpiAwarenessContext(DPI_AWARENESS_CONTEXT_PER_MONITOR_AWARE_V2);
```

これで「このアプリは自分で DPI を扱うので、勝手に拡大しないでほしい」と OS に伝わります。以後、クライアント領域のサイズは**物理ピクセル単位**になります。

`PER_MONITOR_AWARE_V2` は、モニタごとに異なる DPI を扱えるモードです。異なるスケールのモニタ間でウィンドウを移動したときに、正しく追従します。

> **マニフェストで宣言する方法もある**
>
> アプリケーションマニフェストに `<dpiAwareness>PerMonitorV2</dpiAwareness>` と書く方法もあり、こちらのほうが確実(プロセス起動時点から有効)とされています。
>
> 本書は API 呼び出しを採用します。**マニフェストファイルを追加せずに済み、コードを読めば何をしているかがわかる**からです。ウィンドウを作る前に呼ぶ限り、実用上の差はありません。

#### `WM_DPICHANGED` に応答する

ウィンドウが別の DPI のモニタに移動したとき、または表示スケールが変更されたときに送られます。

```cpp
case WM_DPICHANGED:
{
    // lParam に「推奨される新しいウィンドウ矩形」が入っている
    const auto* suggested = reinterpret_cast<const RECT*>(lParam);
    ::SetWindowPos(hwnd, nullptr,
        suggested->left,
        suggested->top,
        suggested->right  - suggested->left,
        suggested->bottom - suggested->top,
        SWP_NOZORDER | SWP_NOACTIVATE);
    return 0;
}
```

**自分で計算する必要はありません。** `lParam` に Windows が推奨する矩形が入っているので、そのまま `SetWindowPos` に渡します。この結果としてサイズが変われば `WM_SIZE` が続けて飛んでくるので、リサイズ処理は自動的に走ります。

`wParam` の下位/上位ワードには新しい X/Y DPI が入っています。UI のフォントサイズなどを調整する場合に使いますが、本書では使いません。

---

## 5.6 実装をまとめる

ここまでの内容を 3 つのファイルにまとめます。

### 5.6.1 `Debug.h` —— デバッグ出力

サブシステムを Windows に変えたため、`std::println` の出力先がなくなりました。**Visual Studio の「出力」ウィンドウに出す**仕組みを作ります。

```cpp
// src/Debug.h
#pragma once
#include "std_import.h"

// Visual Studio の「出力」ウィンドウに書き出す。
// 第6章で、これをログ機能として本格的に整備する。
template <typename... Args>
void DebugPrint(std::wformat_string<Args...> fmt, Args&&... args)
{
    const std::wstring text = std::format(fmt, std::forward<Args>(args)...);
    ::OutputDebugStringW(text.c_str());
}
```

`OutputDebugStringW` は Win32 の関数で、デバッガが接続されていればその出力ウィンドウに、いなければ何もしません。**CRT にも標準ライブラリにも依存しない**ので、この段階で使うのに都合が良い選択です。

`std::wformat_string` は C++23 の型で、コンパイル時に書式文字列を検証してくれます。引数の数や型が合っていなければコンパイルエラーになります。

> **コンソールが欲しい場合**
>
> `AllocConsole()` を呼べばコンソールウィンドウを作れますが、`std::println` の出力先をそこに向けるには `freopen_s` による標準出力の再接続が必要で、CRT のヘッダを持ち込むことになります。第3章の鉄則 3 と衝突するので、本書では採りません。
>
> 「出力」ウィンドウで十分です。むしろ、実行中のアプリと別ウィンドウが増えないぶん扱いやすくなります。

### 5.6.2 `Window.h`

```cpp
// src/Window.h
#pragma once
#include "std_import.h"

class Window
{
public:
    Window() = default;
    ~Window();

    Window(const Window&)            = delete;
    Window& operator=(const Window&) = delete;

    // クライアント領域のサイズを物理ピクセルで指定する
    bool Create(std::wstring_view title, int clientWidth, int clientHeight);
    void Destroy();

    // 溜まっているメッセージを処理する。
    // 終了要求を受け取ったら false を返す。
    bool ProcessMessages();

    HWND Handle()        const noexcept { return m_hwnd; }
    int  ClientWidth()   const noexcept { return m_clientWidth; }
    int  ClientHeight()  const noexcept { return m_clientHeight; }
    bool IsMinimized()   const noexcept { return m_minimized; }

    // サイズが確定したときに呼ばれる(第12章で使う)
    std::function<void(int width, int height)> OnResize;

private:
    static LRESULT CALLBACK StaticWndProc(HWND, UINT, WPARAM, LPARAM);
    LRESULT WndProc(HWND, UINT, WPARAM, LPARAM);
    void    NotifyResize();

    HWND m_hwnd         = nullptr;
    int  m_clientWidth  = 0;
    int  m_clientHeight = 0;
    bool m_minimized    = false;
    bool m_resizing     = false;
    bool m_shouldClose  = false;
};
```

**命名について**:第3章 3.4.5 節で決めた通り、Win32 のマクロと衝突する名前を避けています。`GetMessage` ではなく `ProcessMessages`、`CreateWindow` ではなく `Create` としているのはそのためです。

### 5.6.3 `Window.cpp`

```cpp
// src/Window.cpp
#include "pch.h"
#include "std_import.h"
#if USE_STD_MODULE
import std;
#endif
#include "Window.h"
#include "Debug.h"

namespace
{
    constexpr const wchar_t* kClassName = L"D3D12BookWindowClass";
    constexpr DWORD kStyle   = WS_OVERLAPPEDWINDOW;
    constexpr DWORD kExStyle = 0;

    bool g_classRegistered = false;
}

Window::~Window()
{
    Destroy();
}

bool Window::Create(std::wstring_view title, int clientWidth, int clientHeight)
{
    const HINSTANCE hInstance = ::GetModuleHandleW(nullptr);

    //--- ① ウィンドウクラスの登録(初回のみ) ---
    if (!g_classRegistered)
    {
        WNDCLASSEXW wc{};
        wc.cbSize        = sizeof(wc);
        wc.style         = CS_HREDRAW | CS_VREDRAW;
        wc.lpfnWndProc   = &Window::StaticWndProc;
        wc.hInstance     = hInstance;
        wc.hCursor       = ::LoadCursorW(nullptr, IDC_ARROW);
        wc.hbrBackground = nullptr;          // 背景を塗らせない
        wc.lpszClassName = kClassName;

        if (::RegisterClassExW(&wc) == 0)
        {
            DebugPrint(L"[Window] RegisterClassExW failed: {}\n",
                       ::GetLastError());
            return false;
        }
        g_classRegistered = true;
    }

    //--- ② クライアント領域から必要なウィンドウサイズを逆算 ---
    const UINT dpi = ::GetDpiForSystem();

    RECT rect{ 0, 0, clientWidth, clientHeight };
    ::AdjustWindowRectExForDpi(&rect, kStyle, FALSE, kExStyle, dpi);

    const int windowWidth  = rect.right  - rect.left;
    const int windowHeight = rect.bottom - rect.top;

    //--- ③ ウィンドウ生成 ---
    // wstring_view は null 終端が保証されないので wstring に変換する
    const std::wstring titleText(title);

    m_hwnd = ::CreateWindowExW(
        kExStyle,
        kClassName,
        titleText.c_str(),
        kStyle,
        CW_USEDEFAULT, CW_USEDEFAULT,
        windowWidth, windowHeight,
        nullptr, nullptr, hInstance,
        this);                               // ← WM_NCCREATE で受け取る

    if (m_hwnd == nullptr)
    {
        DebugPrint(L"[Window] CreateWindowExW failed: {}\n", ::GetLastError());
        return false;
    }

    ::ShowWindow(m_hwnd, SW_SHOW);
    ::UpdateWindow(m_hwnd);

    DebugPrint(L"[Window] created: client {} x {} (dpi {})\n",
               m_clientWidth, m_clientHeight, ::GetDpiForWindow(m_hwnd));
    return true;
}

void Window::Destroy()
{
    if (m_hwnd != nullptr)
    {
        const HWND hwnd = m_hwnd;
        m_hwnd = nullptr;
        ::DestroyWindow(hwnd);
    }
}

bool Window::ProcessMessages()
{
    MSG msg{};
    while (::PeekMessageW(&msg, nullptr, 0, 0, PM_REMOVE))
    {
        if (msg.message == WM_QUIT)
        {
            m_shouldClose = true;
            return false;
        }
        ::TranslateMessage(&msg);
        ::DispatchMessageW(&msg);
    }
    return !m_shouldClose;
}

void Window::NotifyResize()
{
    if (OnResize && m_clientWidth > 0 && m_clientHeight > 0)
    {
        OnResize(m_clientWidth, m_clientHeight);
    }
}

LRESULT CALLBACK Window::StaticWndProc(
    HWND hwnd, UINT msg, WPARAM wParam, LPARAM lParam)
{
    Window* self = nullptr;

    if (msg == WM_NCCREATE)
    {
        const auto* create = reinterpret_cast<const CREATESTRUCTW*>(lParam);
        self = static_cast<Window*>(create->lpCreateParams);
        self->m_hwnd = hwnd;
        ::SetWindowLongPtrW(hwnd, GWLP_USERDATA,
                            reinterpret_cast<LONG_PTR>(self));
    }
    else
    {
        self = reinterpret_cast<Window*>(
            ::GetWindowLongPtrW(hwnd, GWLP_USERDATA));
    }

    // this がまだ結びついていない段階のメッセージは既定処理へ
    if (self == nullptr)
    {
        return ::DefWindowProcW(hwnd, msg, wParam, lParam);
    }
    return self->WndProc(hwnd, msg, wParam, lParam);
}

LRESULT Window::WndProc(HWND hwnd, UINT msg, WPARAM wParam, LPARAM lParam)
{
    switch (msg)
    {
    case WM_CLOSE:
        ::DestroyWindow(hwnd);
        return 0;

    case WM_DESTROY:
        m_hwnd = nullptr;
        ::PostQuitMessage(0);
        return 0;

    case WM_ENTERSIZEMOVE:
        m_resizing = true;
        return 0;

    case WM_EXITSIZEMOVE:
        m_resizing = false;
        NotifyResize();
        return 0;

    case WM_SIZE:
        m_clientWidth  = LOWORD(lParam);
        m_clientHeight = HIWORD(lParam);
        m_minimized    = (wParam == SIZE_MINIMIZED);
        if (!m_resizing && !m_minimized)
        {
            NotifyResize();
        }
        return 0;

    case WM_GETMINMAXINFO:
    {
        auto* info = reinterpret_cast<MINMAXINFO*>(lParam);
        info->ptMinTrackSize.x = 320;
        info->ptMinTrackSize.y = 240;
        return 0;
    }

    case WM_DPICHANGED:
    {
        const auto* suggested = reinterpret_cast<const RECT*>(lParam);
        ::SetWindowPos(hwnd, nullptr,
            suggested->left,
            suggested->top,
            suggested->right  - suggested->left,
            suggested->bottom - suggested->top,
            SWP_NOZORDER | SWP_NOACTIVATE);
        DebugPrint(L"[Window] dpi changed: {}\n", LOWORD(wParam));
        return 0;
    }

    default:
        break;
    }

    return ::DefWindowProcW(hwnd, msg, wParam, lParam);
}
```

### 5.6.4 `main.cpp`

```cpp
// src/main.cpp
#include "pch.h"
#include "std_import.h"
#if USE_STD_MODULE
import std;
#endif
#include "Window.h"
#include "Debug.h"

int WINAPI wWinMain(
    _In_     HINSTANCE hInstance,
    _In_opt_ HINSTANCE hPrevInstance,
    _In_     PWSTR     pCmdLine,
    _In_     int       nCmdShow)
{
    // 未使用引数の警告を抑える
    (void)hInstance;
    (void)hPrevInstance;
    (void)pCmdLine;
    (void)nCmdShow;

    // ウィンドウを作る前に DPI 認識を宣言する
    ::SetProcessDpiAwarenessContext(DPI_AWARENESS_CONTEXT_PER_MONITOR_AWARE_V2);

    Window window;

    window.OnResize = [](int width, int height)
    {
        DebugPrint(L"[Window] resize: {} x {}\n", width, height);
    };

    if (!window.Create(L"D3D12Book - Chapter 5", 1280, 720))
    {
        ::MessageBoxW(nullptr, L"ウィンドウの作成に失敗しました。",
                      L"D3D12Book", MB_OK | MB_ICONERROR);
        return 1;
    }

    //--- メインループ ---
    while (window.ProcessMessages())
    {
        if (window.IsMinimized())
        {
            continue;
        }

        // 第12章以降、ここに描画処理が入る
    }

    DebugPrint(L"[Window] exit\n");
    return 0;
}
```

`OnResize` を `Create` の**前**に設定しているのは、ウィンドウ生成中に届く最初の `WM_SIZE` を取りこぼさないためです。

---

## ✅ 本章のゴール:ウィンドウを表示して閉じる

### Step 1:ビルドして実行する

`Ctrl + F5` ではなく、**`F5`(デバッグ開始)で実行してください。** `OutputDebugStringW` の出力は、デバッガが接続されていないと見えません。

### Step 2:表示を確認する

- タイトルバーに `D3D12Book - Chapter 5` と表示される
- クライアント領域が灰色または白(描画していないので未定義)
- マウスカーソルが矢印になる

### Step 3:出力ウィンドウを確認する

Visual Studio の `表示 → 出力` を開き、「出力元」を **「デバッグ」** にします。

```
[Window] created: client 1280 x 720 (dpi 96)
[Window] resize: 1280 x 720
```

DPI が `96` なら表示スケール 100%、`144` なら 150%、`192` なら 200% です。

### Step 4:リサイズを確認する

ウィンドウの端をドラッグしてサイズを変えます。**ドラッグ中は何も出力されず、マウスを離した瞬間に 1 行だけ出る**ことを確認してください。

```
[Window] resize: 1024 x 600
```

これが 5.5.2 節で実装した `WM_ENTERSIZEMOVE` / `WM_EXITSIZEMOVE` の効果です。ドラッグ中に大量の行が流れるようなら、実装を見直してください。

### Step 5:最小化を確認する

最小化ボタンを押します。**何も出力されないこと**を確認してください。0×0 のサイズが通知されていたら、`m_minimized` の判定が効いていません。

元に戻すと、再びサイズが出力されます。

### Step 6:終了を確認する

`×` ボタンでウィンドウを閉じます。

```
[Window] exit
```

が出力され、プロセスが終了すること。**Visual Studio が「デバッグを停止」の状態に戻る**ことを確認してください。戻らない場合、メッセージループが `WM_QUIT` を検出できていません。

### Step 7:マルチモニタでの確認(任意)

表示スケールの異なるモニタが 2 台ある環境なら、ウィンドウをもう一方に移動してみてください。

```
[Window] dpi changed: 144
[Window] resize: 1920 x 1080
```

DPI に応じてウィンドウが拡縮され、クライアント領域のピクセル数が変わります。

---

### 本章の達成状態

- [ ] サブシステムを `Windows` に変更した
- [ ] `wWinMain` がエントリポイントになっている
- [ ] `Window.h` / `Window.cpp` / `Debug.h` を作成した
- [ ] 自作ヘッダを `import std;` の**後**に include している
- [ ] ウィンドウが 1280×720 のクライアント領域で表示される
- [ ] リサイズがドラッグ終了時に 1 回だけ通知される
- [ ] 最小化時に 0×0 が通知されない
- [ ] `×` で正常に終了し、プロセスが残らない
- [ ] `SetProcessDpiAwarenessContext` を呼んでいる

---

## トラブルシューティング

| 症状 | 原因 | 対処 |
|---|---|---|
| `LNK2019: 未解決の外部シンボル WinMain` | サブシステムが Windows なのに `main` を書いている | `wWinMain` に変える(5.1.1) |
| `LNK2019: 未解決の外部シンボル wWinMain` | サブシステムがコンソールのまま | リンカー → システムを確認 |
| ウィンドウが出ない | `ShowWindow` の呼び忘れ | `Create` の末尾を確認 |
| 起動直後にクラッシュ | `WM_GETMINMAXINFO` で `self` が `nullptr` | `nullptr` チェックを追加(5.4.4) |
| `CreateWindowExW` が `nullptr` を返す | `WM_NCCREATE` で 0 を返している | `case WM_NCCREATE` を書かない |
| 閉じても終了しない | `PeekMessageW` の第2引数に `HWND` を渡している | `nullptr` に変える(5.3.2) |
| 同上 | `WM_QUIT` を判定していない | ループ内で明示的に判定する |
| クライアント領域が指定より小さい | `AdjustWindowRectExForDpi` を使っていない | 5.2.4 を確認 |
| 画面がぼやける | DPI 認識を宣言していない | `SetProcessDpiAwarenessContext`(5.5.4) |
| リサイズ中に大量のログ | `WM_ENTERSIZEMOVE` を処理していない | 5.5.2 を確認 |
| 出力ウィンドウに何も出ない | `Ctrl + F5` で起動した | `F5` で起動する |
| カーソルが矢印にならない | `hCursor` が `nullptr` | `LoadCursorW` を設定(5.2.2) |

---

## まとめ

**1. クライアント領域のサイズは自分で逆算する。**
`CreateWindowExW` に渡すのはウィンドウ全体のサイズです。バックバッファと一致させるため、`AdjustWindowRectExForDpi` でクライアント領域から逆算します。

**2. `WndProc` とオブジェクトは `GWLP_USERDATA` で結びつける。**
`CreateWindowExW` の最後の引数に `this` を渡し、`WM_NCCREATE` で受け取ります。それ以前のメッセージでは `nullptr` になるので、必ずチェックしてください。

**3. `WM_QUIT` は `WndProc` に届かない。**
スレッド宛のメッセージなので、ループ内で明示的に判定します。`PeekMessageW` の第2引数を `nullptr` にすることも忘れずに。

**4. 最小化とドラッグ中のリサイズは、後の章で必ず問題になる。**
0×0 のバックバッファは作れません。ドラッグ中の連続リサイズは重すぎます。**今のうちにフラグを用意しておくことが、第12章での平穏につながります。**

**5. DPI 認識を宣言しないと、描いた絵が拡大される。**
Direct3D で緻密に描いても、最後に OS の拡大フィルタを通されては台無しです。ウィンドウを作る前に一行呼ぶだけで済みます。

次章では、COM とエラー処理の土台を作ります。`ComPtr` の正しい使い方と、`std::expected` によるエラー伝搬、そして `std::source_location` を使ったログとアサートを整備します。**第7章でデバイスを作る前に、失敗を検出できる体制を整えます。**

---

## 参考リンク

| 内容 | URL |
|---|---|
| ウィンドウの作成(公式チュートリアル) | https://learn.microsoft.com/ja-jp/windows/win32/learnwin32/creating-a-window |
| ウィンドウメッセージ | https://learn.microsoft.com/ja-jp/windows/win32/winmsg/about-messages-and-message-queues |
| 高 DPI デスクトップアプリケーション開発 | https://learn.microsoft.com/ja-jp/windows/win32/hidpi/high-dpi-desktop-application-development-on-windows |
| `AdjustWindowRectExForDpi` | https://learn.microsoft.com/ja-jp/windows/win32/api/winuser/nf-winuser-adjustwindowrectexfordpi |
| `SetWindowLongPtrW` | https://learn.microsoft.com/ja-jp/windows/win32/api/winuser/nf-winuser-setwindowlongptrw |
