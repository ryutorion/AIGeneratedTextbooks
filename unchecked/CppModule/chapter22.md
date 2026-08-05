# 第22章 実際のゲームコードから使う

## この章について

最終章です。`GameMath` を、実際に動くゲームらしいプログラムから使います。

作るのは、**回転する立方体のワイヤーフレーム表示**です。Windows のウィンドウを開き、GDI で線を引くだけの素朴なものですが、`Vector3`、`Matrix4x4`、`Quaternion`、投影行列 ── 本書で作ったものが一通り登場します。

そして、この章には**本書全体の総決算**があります。

第1章 1.1.5 で、こういう問題を扱いました。

```cpp
#include <Windows.h>
#include "GameMath/Vector3.h"      // min / max マクロで壊れる
```

**モジュールにしたいま、この問題は本当に消えたのか。**

22.2 で、4通りの書き方を試して確かめます。`<Windows.h>` をどこに置いたときに何が守られ、何が守られないのか ── 第6章のグローバルモジュールフラグメントの理解が、ここで実地に試されます。

最後の 22.3 で、完成した `GameMath` の全体像と、本書で下してきた判断を振り返ります。

---

## 22.1 簡単な描画サンプルから `GameMath` を import する

### 22.1.1 作るもの

構成はこうします。

```
Sample/
├── Platform.ixx    export module sample.platform;   ← ウィンドウと線描画
├── Platform.cpp    module sample.platform;          ← ここに <Windows.h>
├── Renderer.ixx    export module sample.renderer;   ← 立方体の描画
├── Renderer.cpp    module sample.renderer;
└── SampleMain.cpp  import ...;
```

**役割をはっきり分けます。**

- `sample.platform` ── OS に触る唯一の層。`<Windows.h>` を**ここだけに閉じ込める**
- `sample.renderer` ── `GameMath` を使って計算する層。**Windows のことを何も知らない**
- `SampleMain.cpp` ── 両者をつなぐ

この分離が、22.2 の実験の土台になります。

### 22.1.2 `GameMath` に投影行列を足す

描画には、まだ `GameMath` にないものが2つ必要です。**透視投影行列**と**ビュー行列**です。

置き場所は考えるまでもありません。**`gamemath.core:matrix`** です。第12章で構成を決めておいたおかげで、追加のたびに設計を考え直す必要がありません。

`Core/Matrix.ixx` に宣言を足します。

```cpp
// Core/Matrix.ixx(namespace gamemath の中)

// 同次座標での変換(w 成分を保つ)
export constexpr Vector4 Transform(const Matrix4x4& a, const Vector4& v)
{
    Vector4 r{};
    for (std::size_t i = 0; i < 4; ++i) {
        r[i] = a.m[i][0] * v.x() + a.m[i][1] * v.y()
             + a.m[i][2] * v.z() + a.m[i][3] * v.w();
    }
    return r;
}

// 透視投影行列(右手系。カメラは -Z 方向を向く)
export Matrix4x4 Perspective(float fovYRadians, float aspect,
                             float nearZ, float farZ);

// ビュー行列
export Matrix4x4 LookAt(const Vector3& eye, const Vector3& target,
                        const Vector3& up);
```

**`Vector4` が、ようやく本来の用途で使われます。** 第8章 8.1.4 でエイリアスを作ったときは「同次座標や RGBA に使う」と書いただけでした。同次座標というのは、まさにこれのことです。

定義は `Core/Matrix.cpp` に書きます。どちらも1フレームに1回程度しか呼ばれず、行数もあるので、実装単位が適切です(第5章 5.6.5)。

```cpp
// Core/Matrix.cpp(namespace gamemath の中)

Matrix4x4 Perspective(float fovYRadians, float aspect, float nearZ, float farZ)
{
    const float f = 1.0f / std::tan(fovYRadians * 0.5f);
    const float d = nearZ - farZ;

    Matrix4x4 r{};
    r.m[0][0] = f / aspect;
    r.m[1][1] = f;
    r.m[2][2] = (farZ + nearZ) / d;
    r.m[2][3] = 2.0f * farZ * nearZ / d;
    r.m[3][2] = -1.0f;
    return r;
}

Matrix4x4 LookAt(const Vector3& eye, const Vector3& target, const Vector3& up)
{
    const Vector3 zAxis = (eye - target).Normalized();      // 後ろ向き
    const Vector3 xAxis = Cross(up, zAxis).Normalized();
    const Vector3 yAxis = Cross(zAxis, xAxis);

    Matrix4x4 r = Matrix4x4::Identity();

    r.m[0][0] = xAxis.x();  r.m[0][1] = xAxis.y();  r.m[0][2] = xAxis.z();
    r.m[1][0] = yAxis.x();  r.m[1][1] = yAxis.y();  r.m[1][2] = yAxis.z();
    r.m[2][0] = zAxis.x();  r.m[2][1] = zAxis.y();  r.m[2][2] = zAxis.z();

    r.m[0][3] = -Dot(xAxis, eye);
    r.m[1][3] = -Dot(yAxis, eye);
    r.m[2][3] = -Dot(zAxis, eye);

    return r;
}
```

> **引数名について**
>
> `nearZ` / `farZ` という名前にしました。`near` / `far` と書きたいところです。
>
> 第1章 1.1.5 で触れたとおり、`near` と `far` はかつて `<Windows.h>` がマクロとして定義していました。そのせいで、世界中のカメラクラスが `float near;` と書けずに苦しみました。
>
> 現在の Windows SDK では整理されていますが、**慣習として避けられ続けています。** これは、マクロ汚染がコードベースに残す傷跡の一例です。モジュールを使えば新しい傷は増えませんが、**過去の傷は消えません。**

### 22.1.3 プラットフォーム層 ── `<Windows.h>` を閉じ込める

`Sample/Platform.ixx` を作ります。**ここには `<Windows.h>` を書きません。**

```cpp
// Sample/Platform.ixx
export module sample.platform;

import std;

namespace sample {

export struct Color
{
    int r;
    int g;
    int b;
};

export class Window
{
public:
    Window(const char* title, int width, int height);
    ~Window();

    Window(const Window&) = delete;
    Window& operator=(const Window&) = delete;

    // false が返ったら終了
    bool ProcessMessages();

    void Clear(Color c);
    void DrawLine(int x0, int y0, int x1, int y1, Color c);
    void Present();

    int Width() const  { return width_; }
    int Height() const { return height_; }

private:
    void* handle_   = nullptr;    // HWND
    void* memoryDc_ = nullptr;    // HDC
    void* bitmap_   = nullptr;    // HBITMAP
    int   width_    = 0;
    int   height_   = 0;
};

} // namespace sample
```

**`HWND` も `HDC` も現れません。** `void*` で持っています。

これは第16章 16.1.6 で `__m128` について、第15章 15.3.3 で `std::optional` について決めたのと同じ判断です。

> 公開する宣言のシグネチャに、外部の型を出さない。

`HWND` を公開すれば、`sample.platform` を `import` した人は `<Windows.h>` を自分で取り込まなければ型名を書けません。それでは閉じ込めた意味がありません。

**代償**もあります。`void*` は型安全ではなく、実装の中でキャストが必要です。より安全な方法として、第11章 11.2.6 の**不完全型**を公開する手もあります。

```cpp
export struct WindowHandle;              // 定義は隠す
```

`void*` より安全ですが、扱いはやや複雑になります。本書では単純さを優先しました。

### 22.1.4 実装単位に `<Windows.h>` を置く

`Sample/Platform.cpp` です。**ここが唯一、Windows を知っている場所です。**

```cpp
// Sample/Platform.cpp
module;

#include <Windows.h>

module sample.platform;

import std;

namespace {

LRESULT CALLBACK SampleWndProc(HWND hwnd, UINT msg, WPARAM wp, LPARAM lp)
{
    if (msg == WM_DESTROY) {
        PostQuitMessage(0);
        return 0;
    }
    return DefWindowProcA(hwnd, msg, wp, lp);
}

COLORREF ToColorRef(sample::Color c)
{
    return RGB(c.r, c.g, c.b);
}

} // namespace

namespace sample {

Window::Window(const char* title, int width, int height)
    : width_(width), height_(height)
{
    const HINSTANCE inst = GetModuleHandleA(nullptr);

    WNDCLASSA wc{};
    wc.lpfnWndProc   = SampleWndProc;
    wc.hInstance     = inst;
    wc.lpszClassName = "GameMathSampleWindow";
    wc.hCursor       = LoadCursorA(nullptr, IDC_ARROW);
    RegisterClassA(&wc);

    RECT rc{ 0, 0, width, height };
    AdjustWindowRect(&rc, WS_OVERLAPPEDWINDOW, FALSE);

    const HWND hwnd = CreateWindowExA(
        0, wc.lpszClassName, title, WS_OVERLAPPEDWINDOW,
        CW_USEDEFAULT, CW_USEDEFAULT,
        rc.right - rc.left, rc.bottom - rc.top,
        nullptr, nullptr, inst, nullptr);

    ShowWindow(hwnd, SW_SHOW);

    // 描画用のオフスクリーンバッファ
    const HDC dc  = GetDC(hwnd);
    const HDC mem = CreateCompatibleDC(dc);
    SelectObject(mem, CreateCompatibleBitmap(dc, width, height));
    ReleaseDC(hwnd, dc);

    handle_   = hwnd;
    memoryDc_ = mem;
}

Window::~Window()
{
    if (memoryDc_ != nullptr) {
        DeleteDC(static_cast<HDC>(memoryDc_));
    }
    if (handle_ != nullptr) {
        DestroyWindow(static_cast<HWND>(handle_));
    }
}

bool Window::ProcessMessages()
{
    MSG msg{};
    while (PeekMessageA(&msg, nullptr, 0, 0, PM_REMOVE)) {
        if (msg.message == WM_QUIT) {
            return false;
        }
        TranslateMessage(&msg);
        DispatchMessageA(&msg);
    }
    return true;
}

void Window::Clear(Color c)
{
    const HDC mem = static_cast<HDC>(memoryDc_);
    RECT rc{ 0, 0, width_, height_ };
    const HBRUSH brush = CreateSolidBrush(ToColorRef(c));
    FillRect(mem, &rc, brush);
    DeleteObject(brush);
}

void Window::DrawLine(int x0, int y0, int x1, int y1, Color c)
{
    const HDC mem = static_cast<HDC>(memoryDc_);
    const HPEN pen = CreatePen(PS_SOLID, 1, ToColorRef(c));
    const HGDIOBJ old = SelectObject(mem, pen);

    MoveToEx(mem, x0, y0, nullptr);
    LineTo(mem, x1, y1);

    SelectObject(mem, old);
    DeleteObject(pen);
}

void Window::Present()
{
    const HWND hwnd = static_cast<HWND>(handle_);
    const HDC dc = GetDC(hwnd);
    BitBlt(dc, 0, 0, width_, height_, static_cast<HDC>(memoryDc_), 0, 0, SRCCOPY);
    ReleaseDC(hwnd, dc);
}

} // namespace sample
```

**`module;` から始まり、`#include <Windows.h>` を経て `module sample.platform;` に至る形**です。第6章 6.3.5 で扱った、実装単位のグローバルモジュールフラグメントです。

そして注目してほしいのは、**`#define NOMINMAX` を書いていない**ことです。

ヘッダの時代なら、これは危険な省略でした。`min` / `max` マクロが漏れて、あちこちが壊れたはずです。書かなくてよい理由は 22.2 で確認します。

### 22.1.5 描画層 ── `GameMath` だけを使う

`Sample/Renderer.ixx` です。

```cpp
// Sample/Renderer.ixx
export module sample.renderer;

import gamemath;
import sample.platform;

namespace sample {

export class CubeRenderer
{
public:
    void Update(float deltaSeconds);
    void Draw(Window& window) const;

private:
    float angle_ = 0.0f;
};

} // namespace sample
```

**`import gamemath;` と `import sample.platform;` の2行だけです。** `<Windows.h>` はもちろん、`import std;` すら書いていません(宣言だけなので不要です)。

`Sample/Renderer.cpp` です。

```cpp
// Sample/Renderer.cpp
module sample.renderer;

import std;
import gamemath;
import sample.platform;

namespace {

using namespace gamemath;

// 単位立方体の8頂点
constexpr Vector3 kVertices[8] = {
    { -1.0f, -1.0f, -1.0f }, {  1.0f, -1.0f, -1.0f },
    {  1.0f,  1.0f, -1.0f }, { -1.0f,  1.0f, -1.0f },
    { -1.0f, -1.0f,  1.0f }, {  1.0f, -1.0f,  1.0f },
    {  1.0f,  1.0f,  1.0f }, { -1.0f,  1.0f,  1.0f },
};

// 12本の稜線
constexpr int kEdges[12][2] = {
    {0,1},{1,2},{2,3},{3,0},
    {4,5},{5,6},{6,7},{7,4},
    {0,4},{1,5},{2,6},{3,7},
};

} // namespace

namespace sample {

void CubeRenderer::Update(float deltaSeconds)
{
    angle_ += deltaSeconds;
}

void CubeRenderer::Draw(Window& window) const
{
    const int w = window.Width();
    const int h = window.Height();
    const float aspect = static_cast<float>(w) / static_cast<float>(h);

    // 2軸まわりの回転をクォータニオンで合成する
    const Quaternion qy = FromAxisAngle(Vector3{ 0.0f, 1.0f, 0.0f }, angle_);
    const Quaternion qx = FromAxisAngle(Vector3{ 1.0f, 0.0f, 0.0f }, angle_ * 0.6f);

    const Matrix4x4 model = ToMatrix((qy * qx).Normalized());

    const Matrix4x4 view = LookAt(
        Vector3{ 0.0f, 0.0f, 6.0f },
        Vector3{ 0.0f, 0.0f, 0.0f },
        Vector3{ 0.0f, 1.0f, 0.0f });

    const Matrix4x4 proj = Perspective(
        60.0f * std::numbers::pi_v<float> / 180.0f, aspect, 0.1f, 100.0f);

    const Matrix4x4 mvp = proj * view * model;

    // 頂点をスクリーン座標へ
    int sx[8]{};
    int sy[8]{};

    // 投影後の範囲を求める(min / max マクロが漏れていたら、ここが壊れる)
    float minX = std::numeric_limits<float>::max();
    float maxX = std::numeric_limits<float>::lowest();

    for (int i = 0; i < 8; ++i) {
        const Vector4 p{ kVertices[i].x(), kVertices[i].y(), kVertices[i].z(), 1.0f };
        const Vector4 clip = Transform(mvp, p);

        const float invW = 1.0f / clip.w();
        const float ndcX = clip.x() * invW;
        const float ndcY = clip.y() * invW;

        minX = std::min(minX, ndcX);
        maxX = std::max(maxX, ndcX);

        sx[i] = static_cast<int>((ndcX * 0.5f + 0.5f) * static_cast<float>(w));
        sy[i] = static_cast<int>((0.5f - ndcY * 0.5f) * static_cast<float>(h));
    }

    const Color line{ 80, 220, 160 };
    for (const auto& e : kEdges) {
        window.DrawLine(sx[e[0]], sy[e[0]], sx[e[1]], sy[e[1]], line);
    }
}

} // namespace sample
```

`minX` / `maxX` の計算は、実は描画に使っていません。**マクロ漏れの検出器として置いてあります。**

`std::numeric_limits<float>::max()` と `std::min` / `std::max` は、`<Windows.h>` の `min` / `max` マクロが漏れていたら**必ず壊れます**。第1章 1.1.5 で見たとおりです。

### 22.1.6 組み立てて動かす

`Sample/SampleMain.cpp` です。

```cpp
// Sample/SampleMain.cpp
import std;
import sample.platform;
import sample.renderer;

int main()
{
    sample::Window window{ "GameMath Sample", 800, 600 };
    sample::CubeRenderer renderer;

    const sample::Color background{ 16, 16, 24 };

    auto previous = std::chrono::steady_clock::now();

    while (window.ProcessMessages()) {
        const auto now = std::chrono::steady_clock::now();
        const float dt = std::chrono::duration<float>(now - previous).count();
        previous = now;

        renderer.Update(dt);

        window.Clear(background);
        renderer.Draw(window);
        window.Present();

        std::this_thread::sleep_for(std::chrono::milliseconds(16));
    }

    return 0;
}
```

`main.cpp` と同居させると `main` が2つになるので、これまでの `main.cpp` はプロジェクトから外すか、別のプロジェクトにしてください。

ビルドして実行すると、暗い背景の中で緑色の立方体が回ります。

**`import` の行を見返してください。**

```cpp
import std;
import sample.platform;
import sample.renderer;
```

**`#include` が1行もありません。** ゲームのメインループが、プリプロセッサ指令なしで書けています。第1章 1.1.2 でプリプロセス結果を見たときのことを思い出してください。あのときの数万行は、どこにも展開されていません。

---

## 22.2 他ライブラリのヘッダと混ぜたときに起きたこと

いま `<Windows.h>` は、`Platform.cpp` の**グローバルモジュールフラグメント**に置かれています。

置き場所を4通り変えて、何が起きるかを確かめます。**本書の総復習になります。**

### 22.2.1 【実験1】実装単位のグローバルモジュールフラグメント(現状)

```cpp
// Sample/Platform.cpp
module;
#include <Windows.h>
module sample.platform;
```

ビルドは通り、動きます。そして `Renderer.cpp` の `std::min` / `std::max` / `std::numeric_limits` も無事です。

**確認**

`Renderer.cpp` で、`min` がマクロとして定義されているか調べてみてください。

```cpp
// Renderer.cpp の先頭あたり
#ifdef min
#error "min macro leaked!"
#endif
```

**エラーになりません。** マクロは漏れていません。

第6章 6.3.3 で確認したとおりです。

> GMF で定義したマクロも、`import` した側に漏れない。

`NOMINMAX` を書かなくても大丈夫だったのは、これが理由です。

**`.ifc` への影響もありません。** 実装単位は `.ifc` を生成しないからです(第5章 5.2.5)。`sample.platform` の `.ifc` には、`Window` クラスの宣言しか入っていません。

**これが最良の形です。**

### 22.2.2 【実験2】インターフェイス単位のグローバルモジュールフラグメント

`Platform.ixx` を書き換えてみます。

```cpp
// Sample/Platform.ixx
module;

#include <Windows.h>          // ← インターフェイス側に移した

export module sample.platform;

import std;

namespace sample {

export class Window
{
public:
    // ...
private:
    HWND  handle_ = nullptr;      // ← 本物の型が使える
    HDC   memoryDc_ = nullptr;
    // ...
};

}
```

ビルドは通ります。動きます。**`min` / `max` マクロも漏れません。** GMF に置いているので、第6章 6.3.3 の性質がそのまま効きます。

**では、何が悪いのか。**

**問題1: `.ifc` が重くなる**

`<Windows.h>` の宣言のうち、`Window` クラスから到達できるものが `.ifc` に取り込まれます(第6章 6.3.4)。`HWND` や `HDC` がメンバの型として現れるので、それに関連するものが入ってきます。

`sample.platform` を `import` するすべての翻訳単位が、これを読むことになります。

**問題2: `HWND` が到達可能になる**

`Renderer.cpp` で、こう書けてしまいます。

```cpp
auto h = /* Window から HWND を取り出す何か */;    // auto なら通る
```

第7章 7.4.4 の「到達可能だが見えない」状態です。**設計として意図していない依存が生まれる余地があります。**

**問題3: ABI が Windows の型に縛られる**

`sizeof(Window)` が `HWND` のサイズに依存します。第11章 11.3.4 で扱った問題です。

**実験が終わったら、22.1.3 / 22.1.4 の形に戻してください。**

### 22.2.3 【実験3】購買域に置く

もっと素直に書いてみます。

```cpp
// Sample/Platform.cpp
module sample.platform;

#include <Windows.h>          // ← モジュール宣言の後ろ

import std;
```

ビルドすると、**警告 C5244** が出ます。第6章 6.1.2 で見たものです。

```
warning C5244: '#include <Windows.h>' in the purview of module 'sample.platform'
               appears erroneous.
```

**そして、ビルドは通ってしまう可能性があります。**

しかし、第6章 6.1.5 で説明したとおり、`<Windows.h>` の宣言がすべて `sample.platform` モジュールに属することになります。`HWND` も `RECT` も `MSG` も、他所の同名の型とは別物になります。

**別の翻訳単位が `#include <Windows.h>` していて、そこと型をやり取りしようとした瞬間に壊れます。**

いまのサンプルでは他に `<Windows.h>` を使う場所がないので、たまたま動きます。**「たまたま動く」がいちばん危険です。**

**必ず元に戻してください。**

### 22.2.4 【実験4】利用側で `#include` する

最後に、`SampleMain.cpp` で直接 `<Windows.h>` を取り込んでみます。

```cpp
// Sample/SampleMain.cpp
#include <Windows.h>          // ← 追加

import std;
import sample.platform;
import sample.renderer;

int main()
{
    // ...
    const int m = std::numeric_limits<int>::max();     // ← 壊れる
}
```

**ここは壊れます。**

```
error C2589: '(': '::' に続くトークンが正しくありません。
```

第1章 1.1.5 で見たエラーそのものです。`max` がマクロとして展開されました。

**しかし、被害はこのファイルだけです。**

- `Renderer.cpp` は無傷です
- `Platform.cpp` も無傷です
- `GameMath` のどのファイルも無傷です

ヘッダの時代なら、`<Windows.h>` を `#include` したファイルから、その先の `#include` チェーンすべてに影響が及びました。**モジュールでは、マクロは1つの翻訳単位に閉じ込められます。**

これが第1章 1.3.1 で書いた「マクロは越えてこない」の実際の姿です。

**問題が起きたファイルを見れば、原因がそこにある** ── 第17章 17.3.1 で「モジュールのエラーは出た場所と原因の場所が違う」と書きましたが、マクロ汚染に関しては逆です。**同じファイルの中に必ず原因があります。**

**追加した行を削除してください。**

### 22.2.5 何が守られ、何が守られないのか

4つの実験を整理します。

| `<Windows.h>` の置き場所 | マクロ漏れ | `.ifc` への影響 | 型の同一性 | 評価 |
|---|---|---|---|---|
| **実装単位の GMF** | なし | **なし** | 正しい | **最良** |
| インターフェイスの GMF | なし | あり | 正しい | 必要なら |
| 購買域 | なし | あり | **壊れる** | **禁止** |
| 利用側で `#include` | **そのファイルだけ** | ── | 正しい | 局所的 |

**守られたもの**

- **マクロ汚染がモジュール境界を越えない。** `NOMINMAX` を書かなくてよくなった
- **被害が翻訳単位に閉じ込められる**
- 型の同一性が保たれる(GMF を正しく使えば)

**守られなかったもの**

- **1つの翻訳単位の中では、マクロは相変わらず暴れます。** `#include <Windows.h>` と書いたファイルの中では、何も変わっていません
- **ヘッダを使う側の作法は変わっていません。** `NOMINMAX` は、`#include` する側では依然として必要です

つまり、**モジュールが守るのは境界だけです。** 境界の内側は従来どおりです。

第1章 1.4.4 で「マクロがなくなるわけではない」と書いたのは、こういうことでした。

### 22.2.6 実務への含意

このサンプルから得られる実践的な指針は、1つです。

> **OS や外部ライブラリのヘッダは、それを必要とする実装単位のグローバルモジュールフラグメントに閉じ込める。**
> **公開インターフェイスには、外部の型を出さない。**

これは本書で繰り返してきた原則の、最後の適用例です。

| 章 | 出さないと決めたもの | 手段 |
|---|---|---|
| 6.5.4 | 標準ライブラリの型 | 自前の型で返す |
| 15.3.5 | `std::optional` | `RayHit` 構造体 |
| 16.1.6 | `__m128` | 関数の中だけで使う |
| **22.1.3** | **`HWND` / `HDC`** | **`void*`(または不完全型)** |

**外部の型を公開インターフェイスに出さなければ、利用者はその外部ライブラリを知らずに済みます。** `sample.renderer` は Windows を一切知りません。仮に描画層を Linux に移植するとしても、`Renderer.cpp` は1行も変わりません。

これはモジュール特有の話ではなく、昔からある良い設計です。**モジュールは、それを言語機能として支えてくれるようになっただけです。** ヘッダの時代には、`detail` 名前空間と同じで「お願い」にすぎませんでした(第7章 7.2.4)。

---

## 22.3 完成したライブラリの全体像を振り返る

### 22.3.1 ファイル構成

```
GameMath/
├── GameMath.ixx            export module gamemath;              傘(第12章)
│
├── Core/
│   ├── Core.ixx            export module gamemath.core;
│   ├── Vector.ixx              :vector       Vector<T,N> / 演算子 / Dot / Cross
│   ├── Detail.ixx              :detail       [内部] MinorAxis / SafeAcos
│   ├── Basis.ixx               :basis        BasisPair / Orthogonal / AngleBetween
│   ├── Matrix.ixx              :matrix       Matrix4x4 / 変換行列 / SIMD
│   ├── Quaternion.ixx          :quaternion   Quaternion / Slerp
│   ├── Transform.ixx           :transform    ToMatrix / ToQuaternion
│   ├── Basis.cpp           module gamemath.core;
│   ├── Matrix.cpp
│   ├── Quaternion.cpp
│   └── Transform.cpp
│
├── Geometry/
│   ├── Geometry.ixx        export module gamemath.geometry;
│   ├── Shapes.ixx              :shapes       AABB / Sphere / Ray / Plane
│   ├── Intersect.ixx           :intersect    RayHit / Overlaps / Intersect
│   └── Intersect.cpp
│
├── Random/
│   └── Random.ixx          export module gamemath.random;       [private fragment]
│
└── Debug/
    ├── Debug.ixx           export module gamemath.debug;        [export import std]
    └── Debug.cpp
```

**モジュール単位が20個。** 5種類すべてが登場しています。

| 種類 | 例 | 章 |
|---|---|---|
| プライマリインターフェイス単位 | `Core.ixx` | 3 |
| インターフェイスパーティション | `Vector.ixx` | 9 |
| 内部パーティション | `Detail.ixx` | 10 |
| 実装単位 | `Basis.cpp` | 5 |
| プライベートモジュールフラグメント | `Random.ixx` | 11 |

### 22.3.2 依存グラフ

```
                        gamemath (傘)
              ┌──────────────┼──────────────┐
              ↓              ↓              ↓
      gamemath.geometry  gamemath.core  gamemath.random
              │              │
              │      ┌───────┴────────┐
              ↓      ↓                ↓
         :intersect  :transform
              ↓      ↓      ↓
          :shapes  :matrix  :quaternion
              └──────┼──────┘
                     ↓
                  :basis
                     ↓
                  :detail
                     ↓
                  :vector          ← 土台

      gamemath.debug (傘に入れない。export import std)
```

**一方向です。** 循環はありません(第10章 10.4)。

`gamemath.random` が独立していること、`gamemath.debug` が傘に入っていないこと ── どちらも設計上の判断でした(第11章 11.1.5、第17章 17.2.6)。

### 22.3.3 使った道具の一覧

| 道具 | どこで | 章 |
|---|---|---|
| `export module` / `import` | すべて | 3 |
| 名前空間との分離 | 全モジュールが `gamemath` | 4 |
| 実装単位 | `Orthogonal` など | 5 |
| グローバルモジュールフラグメント | `<immintrin.h>` / `<cassert>` / `<Windows.h>` | 6 |
| `import std;` | すべてのモジュール単位 | 6 |
| モジュールリンケージ | `kEpsilon`(のちに公開)など | 7 |
| コンセプト | `FloatingPoint` / `HasBounds` | 8 |
| 明示的インスタンス化 | `Orthogonal<float/double>` | 8 |
| インターフェイスパーティション | `:vector` など7つ | 9 |
| 内部パーティション | `:detail` | 10 |
| プライベートモジュールフラグメント | `gamemath.random` | 11 |
| `export import` | 傘とプライマリ | 12 |
| 隠しフレンド | 演算子すべて | 13 |
| ブリッジパーティション | `:transform` | 14 |
| `if consteval` | `operator*`(行列) | 16 |

### 22.3.4 決めた方針の一覧

本書で下した設計判断です。**理由とともに覚えてください。**

| 方針 | 理由 | 章 |
|---|---|---|
| `import std;` を使う | 混在を避け、マクロ汚染を防ぐ | 6.6.1 |
| **公開インターフェイスに外部の型を出さない** | 利用者の環境を選ばない | 6.5.4 / 15.3.5 / 16.1.6 / 22.1.3 |
| 短くて頻繁に呼ばれるものはインターフェイスに書く | インライン展開・`constexpr`・テンプレート | 5.6.5 / 8.5.5 |
| **記号で呼ぶものは隠しフレンド、名前で呼ぶものは自由関数** | 修飾して呼べるかどうか | 13.4.6 |
| 演算子は意味が1つに決まるときだけ作る | `operator*(M, V)` の曖昧さ | 13.5.4 |
| 拡張点はメンバ関数とコンセプトで作る | ADL は境界を越えない | 13.3.4 / 15.4.6 |
| マクロによる切り替えを型に置き換える | マクロは境界を越えない | 16.5.1 |
| `assert` は実装単位に書く | `NDEBUG` はモジュールのビルド時に確定する | 17.1.6 |
| 傘は「全部入り」ではなく「既定で欲しいもの」 | `gamemath.debug` を入れない | 17.2.6 |
| 迷ったらパーティション | 昇格はできるが降格はできない | 12.5 |

### 22.3.5 使わなかった道具と、その理由

**使わなかったことも設計判断です。**

| 道具 | 使わなかった理由 | 章 |
|---|---|---|
| ヘッダユニット | `import std;` があり、外部依存もない | 18.4.6 |
| `/translateInclude` | 移行する既存ヘッダがない | 18.2.5 |
| `export namespace` | 公開は明示的な行為であるべき | 4.4.3 |
| `export import std;`(本体) | 利用者の環境を選ばない | 6.5.4 |
| `alignas(16)` | ABI を固くする代償が大きい | 16.2.3 |
| `std::optional` を返す | 情報が複数あり、外部型を出したくない | 15.3.5 |
| プライベートモジュールフラグメント(本体) | モジュール単位が複数ある | 11.4.3 |

**道具を知ることと、使うべき場面を見極めることは別です。** 本書が第5章、第11章、第18章で「この道具は使わない」と明言してきたのは、そのためでした。

---

## 22.4 本書のまとめ

### 22.4.1 モジュールが本当に変えたこと

第1章 1.3 で、3つの変化を挙げました。振り返ります。

**1. テキストではなく「意味」を取り込む**

`import` の行はプリプロセスされません。`.ifc` から解析済みのデータを読みます。だから ──

- マクロが越えません(第3章 3.6.4、第22章 22.2)
- 順序が意味を持ちません(第3章 3.4.4)
- 依存が伝染しません(第6章 6.2.3)

**2. 公開するものを自分で選べる**

`export` しないものは、名前として存在しません。`detail` 名前空間は「お願い」でしたが、モジュールリンケージは言語による保証です(第7章 7.2)。

ただし、**隠れるのは名前であって情報ではありません**(第7章 7.4.5)。ここを取り違えると、`.ifc` の肥大化と再コンパイルの連鎖に悩まされます。

**3. 解析は1回だけ**

掛け算が足し算に変わりました(第1章 1.1.4)。ただし、**効果は分割設計に依存します**(第1章 1.4.1、第21章)。

### 22.4.2 変えなかったこと

第1章 1.4 で挙げたことは、すべてそのまま残りました。

- ABI 問題は解決していません(第11章 11.3.4、第20章 20.4)
- テンプレートの実体化コストは変わりません(第8章 8.2.6、第21章 21.5.3)
- パッケージ管理は別の問題です(第20章 20.5)
- マクロは残ります。ただし境界を越えません(第16章、第22章 22.2.5)

そして、**新しい制約が加わりました。**

- **循環依存が許されません**(第10章 10.4)
- **ビルド順序に制約が生まれます**(第20章 20.1)
- **翻訳単位ごとに違う設定でコンパイルできません**(第16章 16.4.2)

最後の制約は、**ODR 違反が構造的に起こらなくなる**という利点の裏返しでした(第16章 16.4.5)。得たものと失ったものが同じ性質から来ている ── モジュールの本質をよく表しています。

### 22.4.3 これから

本書を書いている時点で、C++ のモジュールはまだ発展の途上にあります。

- コンパイラ間の差異が残っています(付録E)
- ビルドシステムとパッケージマネージャの対応は進行中です(第20章)
- サードパーティライブラリの多くは、まだヘッダです(第18章)

そして、確実に変わっていく部分もあります。

- C++26 で数学関数が `constexpr` になれば、第14章 14.3.5 の表は書き換わります
- C++26 の契約(Contracts)が入れば、第17章 17.1 の制約は緩みます
- パッケージマネージャが対応すれば、第20章 20.5 は過去の話になります

**しかし、本書で扱った判断の枠組みは変わりません。**

- 何を公開し、何を隠すか
- どこに定義を置くか(到達可能性)
- 依存をどちらの向きにするか
- 境界をまたぐ仕掛けを、どう型で表現するか

これらは、モジュールという機能の細部が変わっても残る問いです。そして、モジュールが登場する前から、良い C++ の設計が扱ってきた問いでもあります。

**モジュールは、その問いに答えるための道具を、言語の中に用意してくれました。**

第1章の最後に、こう書きました。

> それでもモジュールを学ぶ理由 ── ビルド時間は、正しく設計すれば確かに改善します。それ以上に価値があるのは、カプセル化が言語機能として保証されることです。

22章を経て、この評価は変わりません。**`GameMath` の内部は、本当に外から見えません。** `detail` 名前空間も、命名規則も、レビューでの注意も要りませんでした。書かなかったものは、存在しないのです。

ここから先は、あなたのコードで確かめてください。

---

## 付録について

本書の内容を、実務で参照しやすい形にまとめたものが付録です。

- **付録A** モジュール構文チートシート
- **付録B** よく出るコンパイルエラーと対処(エラーコード別)
- **付録C** 用語集
- **付録D** MSVC 固有の挙動と制限事項
- **付録E** 他コンパイラ(Clang / GCC)との差異
- **付録F** 完成版 `GameMath` ソースコード一覧

とくに**付録B** は、第17章 17.3 と対にして使ってください。第17章が症状から原因をたどる手順、付録B がエラーコードから引く索引です。

お疲れさまでした。
