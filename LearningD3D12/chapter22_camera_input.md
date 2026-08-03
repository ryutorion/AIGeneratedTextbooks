# 第22章 カメラ操作と入力

第3部に入ります。ここからは「実用的なレンダラ」を組み上げていきます。

**最初にやるのは、動き回れるようにすることです。**

理由は単純で、**動かせないと何も確認できない**からです。第23章でモデルを読み込んだとき、法線が裏返っていないか。第24章でライティングを入れたとき、光の当たり方が正しいか。第27章で影を落としたとき、位置がずれていないか。**すべて、視点を変えられて初めて分かります。**

固定カメラのまま先へ進むと、「たぶん合っている」を積み上げることになります。**それは第31章で Aftermath を使うより前に、自分でバグを作り込む道です。**

**本章のゴール**
Raw Input による滑らかなマウス操作と、フレームレートに依存しない移動を実装する。軌道カメラと FPS カメラを切り替えられるようにする。

---

## 22.1 入力の取り方

### 22.1.1 なぜ `WM_MOUSEMOVE` では駄目なのか

**第5章で作った `WndProc` に、こう書けば済むように見えます。**

```cpp
case WM_MOUSEMOVE:
    const int x = GET_X_LPARAM(lParam);
    const int y = GET_Y_LPARAM(lParam);
    // 前回位置との差分を取る
```

**3 つの問題があります。**

**問題 1:カーソルが画面端で止まる**

FPS カメラでは、マウスを右へ動かし続けたい場面があります。しかし**カーソルは画面の端で止まります。** そこから先は座標が変わらないので、視点も回りません。

**問題 2:OS の加速が掛かっている**

Windows の「ポインターの精度を高める」設定は、**マウスの移動速度に応じて移動量を変化させます。** デスクトップ操作には便利ですが、視点操作では**同じ距離動かしても回転量が変わる**という結果になります。

**問題 3:解像度が失われている**

`WM_MOUSEMOVE` の座標は**ピクセル単位**です。高 DPI のマウス(数千 DPI)が持つ細かい動きは、ピクセルに丸められた時点で失われます。

### 22.1.2 Raw Input

**Raw Input は、デバイスからの生の移動量を受け取る仕組みです。**

| | `WM_MOUSEMOVE` | **Raw Input** |
|---|---|---|
| 取得できるもの | カーソルの座標 | **デバイスの移動量** |
| 画面端 | 止まる | **無関係** |
| OS の加速 | 掛かる | **掛からない** |
| 解像度 | ピクセル | **デバイスの分解能** |

**視点操作には Raw Input を使います。**

ただし、**UI のクリック判定などには `WM_LBUTTONDOWN` などの通常のメッセージを使います。** 用途で使い分けます。

### 22.1.3 登録する

```cpp
// src/Input/InputSystem.cpp

Core::Status InputSystem::Initialize(HWND hwnd)
{
    RAWINPUTDEVICE device{};
    device.usUsagePage = 0x01;      // Generic Desktop Controls
    device.usUsage     = 0x02;      // Mouse
    device.dwFlags     = 0;         // 後述
    device.hwndTarget  = hwnd;

    if (!::RegisterRawInputDevices(&device, 1, sizeof(device)))
    {
        return std::unexpected(Core::MakeError(
            HRESULT_FROM_WIN32(::GetLastError()),
            L"RegisterRawInputDevices"));
    }

    m_hwnd = hwnd;
    LOG_INFO(L"raw input registered for mouse");
    return {};
}
```

**`usUsagePage` と `usUsage` は HID の規格で決まった値です。**

| デバイス | UsagePage | Usage |
|---|---|---|
| マウス | 0x01 | 0x02 |
| キーボード | 0x01 | 0x06 |
| ゲームパッド | 0x01 | 0x05 |

**`dwFlags` について。**

| フラグ | 意味 |
|---|---|
| `0` | ウィンドウがフォアグラウンドのときだけ受け取る |
| `RIDEV_INPUTSINK` | **バックグラウンドでも受け取る**(`hwndTarget` が必須) |
| `RIDEV_NOLEGACY` | `WM_MOUSEMOVE` などを**送らせない** |

**`RIDEV_NOLEGACY` は使いません。** 通常のマウスメッセージが完全に止まるので、ウィンドウのタイトルバーをドラッグすることすらできなくなります。**第29章で ImGui のような UI を組み込む場合も困ります。**

**キーボードは Raw Input にしません。** `WM_KEYDOWN` で十分だからです。キーの押下は離散的な事象で、加速も解像度も関係ありません。

### 22.1.4 `WM_INPUT` を処理する

```cpp
LRESULT Window::WndProc(HWND hwnd, UINT msg, WPARAM wParam, LPARAM lParam)
{
    switch (msg)
    {
    case WM_INPUT:
        if (OnRawInput)
        {
            OnRawInput(reinterpret_cast<HRAWINPUT>(lParam));
        }
        // 既定の処理も必要(クリーンアップのため)
        return ::DefWindowProcW(hwnd, msg, wParam, lParam);

    // ... 以下、これまでのまま ...
    }
}
```

**`DefWindowProcW` を呼ぶ必要があります。** Raw Input のバッファを OS が解放するためです。**`return 0;` で終わらせると、リークします。**

受け取り側です。

```cpp
void InputSystem::ProcessRawInput(HRAWINPUT handle)
{
    UINT size = 0;

    //--- 必要なサイズを問い合わせる ---
    if (::GetRawInputData(handle, RID_INPUT, nullptr, &size,
                          sizeof(RAWINPUTHEADER)) != 0)
    {
        return;
    }

    //--- 十分なバッファを用意する ---
    // 毎回確保しないよう、メンバとして持ち回す
    if (m_rawBuffer.size() < size)
    {
        m_rawBuffer.resize(size);
    }

    if (::GetRawInputData(handle, RID_INPUT, m_rawBuffer.data(), &size,
                          sizeof(RAWINPUTHEADER)) != size)
    {
        return;
    }

    const auto* raw = reinterpret_cast<const RAWINPUT*>(m_rawBuffer.data());

    if (raw->header.dwType != RIM_TYPEMOUSE)
    {
        return;
    }

    //--- 相対移動か絶対座標かを確認する ---
    if (raw->data.mouse.usFlags & MOUSE_MOVE_ABSOLUTE)
    {
        // タブレットやリモートデスクトップではこちらになる。
        // 前回位置との差分を取る必要があるが、本書では扱わない。
        return;
    }

    m_mouseDelta.x += static_cast<float>(raw->data.mouse.lLastX);
    m_mouseDelta.y += static_cast<float>(raw->data.mouse.lLastY);

    //--- ホイール ---
    if (raw->data.mouse.usButtonFlags & RI_MOUSE_WHEEL)
    {
        const auto delta = static_cast<SHORT>(raw->data.mouse.usButtonData);
        m_wheelDelta += static_cast<float>(delta) / WHEEL_DELTA;
    }
}
```

**`MOUSE_MOVE_ABSOLUTE` の確認は省略しないでください。**

リモートデスクトップ、仮想マシン、ペンタブレットでは、**移動量ではなく絶対座標が入ります。** そのまま差分として扱うと、**カメラが画面外まで一瞬で吹き飛びます。**

**`m_mouseDelta` に `+=` している**のも重要です。1 フレームの間に `WM_INPUT` が複数回届くことがあるためです。**上書きすると、動きが取りこぼされます。**

### 22.1.5 フレーム単位で使う

**入力は「フレームの状態」として扱います。**

```cpp
// src/Input/InputSystem.h
class InputSystem
{
public:
    // フレームの先頭で呼ぶ
    void BeginFrame();

    //--- マウス ---
    Math::Vector2 MouseDelta() const noexcept { return m_mouseDeltaThisFrame; }
    float         WheelDelta() const noexcept { return m_wheelThisFrame; }
    bool          IsMouseDown(int button) const noexcept;
    bool          WasMousePressed(int button) const noexcept;

    //--- キーボード ---
    bool IsKeyDown(int vk)     const noexcept { return m_keys[vk]; }
    bool WasKeyPressed(int vk) const noexcept
    {
        return m_keys[vk] && !m_prevKeys[vk];
    }

private:
    std::array<bool, 256> m_keys{};
    std::array<bool, 256> m_prevKeys{};
    // ...
};
```

```cpp
void InputSystem::BeginFrame()
{
    m_prevKeys = m_keys;               // 前フレームの状態を保存
    m_prevMouseButtons = m_mouseButtons;

    m_mouseDeltaThisFrame = m_mouseDelta;   // 溜まった分を確定
    m_wheelThisFrame      = m_wheelDelta;

    m_mouseDelta = {};                 // リセット
    m_wheelDelta = 0.0f;
}
```

**「押されている」と「今押された」を区別できるようにしています。**

| 用途 | 使うもの |
|---|---|
| 移動(押している間ずっと) | `IsKeyDown` |
| 切り替え(押した瞬間だけ) | **`WasKeyPressed`** |

**第16章 16.4.1 節で、ワイヤーフレームの切り替えを `WM_KEYDOWN` で暫定的に実装しました。** ここで正式な形に置き換えます。

```cpp
// 第16章の暫定版
case WM_KEYDOWN:
    if (wParam == 'W' && OnToggleWireframe) { OnToggleWireframe(); }
```

```cpp
// 第22章の形
if (input.WasKeyPressed('F'))       // Wireframe → Fill 切り替え
{
    m_wireframe = !m_wireframe;
}
```

> **キーリピートの問題が消える**
>
> `WM_KEYDOWN` は、キーを押し続けると**繰り返し届きます。** 暫定版では、`W` を押しっぱなしにするとワイヤーフレームが高速に点滅していました。
>
> `WasKeyPressed` は前フレームとの比較なので、**押した瞬間に一度だけ真になります。**

---

## 22.2 デルタタイムと高精度タイマ

### 22.2.1 なぜ必要か

**フレームレートは一定ではありません。**

```cpp
// ❌ フレームレートに依存する
cameraPosition.z += 0.1f;
```

このコードは、60fps なら毎秒 6.0、144fps なら毎秒 14.4 進みます。**環境によって移動速度が変わります。**

```cpp
// ✅ 時間に基づく
cameraPosition.z += speed * deltaTime;
```

**`deltaTime` は、前フレームからの経過秒数です。** これを掛ければ、フレームレートに関係なく同じ速度になります。

### 22.2.2 `QueryPerformanceCounter` を使う

**時間の測り方には選択肢があります。**

| 方法 | 分解能 | 備考 |
|---|---|---|
| `GetTickCount64` | 約 16ms | **粗すぎる** |
| `timeGetTime` | 1ms(要設定) | `winmm.lib` が必要 |
| `std::chrono::steady_clock` | 環境依存 | **Windows では QPC を使う** |
| **`QueryPerformanceCounter`** | **1μs 未満** | **本書はこれ** |

**`std::chrono::steady_clock` でも構いません。** MSVC の実装は内部で `QueryPerformanceCounter` を呼んでいます。第11章 11.8 節では実際にそちらを使いました。

**本書のタイマは `QueryPerformanceCounter` を直接使います。** 理由は、**第38章で GPU タイムスタンプと突き合わせるとき、同じ時間基準にしたいから**です。

```cpp
// src/Core/Timer.h
#pragma once
#include "std_import.h"

namespace Core
{
    class Timer
    {
    public:
        Timer();

        // フレームの先頭で呼ぶ
        void Tick();

        float DeltaSeconds() const noexcept { return m_delta; }
        float TotalSeconds() const noexcept { return m_total; }
        std::uint64_t FrameCount() const noexcept { return m_frameCount; }

        // 直近 0.5 秒の平均
        float Fps() const noexcept { return m_fps; }

    private:
        std::int64_t m_frequency = 0;
        std::int64_t m_lastCount = 0;

        float m_delta = 0.0f;
        float m_total = 0.0f;
        float m_fps   = 0.0f;

        std::uint64_t m_frameCount = 0;

        // FPS 計算用
        float         m_accumulated = 0.0f;
        std::uint64_t m_framesInWindow = 0;
    };
}
```

```cpp
Timer::Timer()
{
    LARGE_INTEGER frequency{};
    ::QueryPerformanceFrequency(&frequency);
    m_frequency = frequency.QuadPart;

    LARGE_INTEGER counter{};
    ::QueryPerformanceCounter(&counter);
    m_lastCount = counter.QuadPart;
}

void Timer::Tick()
{
    LARGE_INTEGER counter{};
    ::QueryPerformanceCounter(&counter);

    const std::int64_t elapsed = counter.QuadPart - m_lastCount;
    m_lastCount = counter.QuadPart;

    m_delta = static_cast<float>(
        static_cast<double>(elapsed) / static_cast<double>(m_frequency));

    //--- 異常値を弾く ---
    // ブレークポイントで止まった後や、
    // ウィンドウのドラッグ中(第12章 12.5 節)に巨大な値になる
    constexpr float kMaxDelta = 0.1f;    // 10fps 相当
    if (m_delta > kMaxDelta)
    {
        m_delta = kMaxDelta;
    }

    m_total += m_delta;
    ++m_frameCount;

    //--- FPS の平均 ---
    m_accumulated += m_delta;
    ++m_framesInWindow;

    if (m_accumulated >= 0.5f)
    {
        m_fps = static_cast<float>(m_framesInWindow) / m_accumulated;
        m_accumulated = 0.0f;
        m_framesInWindow = 0;
    }
}
```

### 22.2.3 デルタタイムの上限は必須

**`kMaxDelta` によるクランプは、飾りではありません。**

デバッガでブレークポイントに止まると、**再開したときのデルタタイムは数十秒になります。** その値で移動計算をすると、**カメラが宇宙の彼方へ飛びます。**

第12章 12.5 節で扱った「ウィンドウのドラッグ中」も同じです。モーダルループの間、`WM_TIMER` による描画は動きますが、時間は普通に流れています。

**10fps 相当(0.1 秒)で頭打ちにしておけば、実害はありません。** 実際にそこまで遅いなら、カメラの飛び方より先に別の問題があります。

> **物理シミュレーションでは固定タイムステップを使う**
>
> 本書は扱いませんが、物理演算では**デルタタイムを直接使ってはいけません。** フレームレートによって挙動が変わり、再現性がなくなります。
>
> 「固定の刻み幅で、必要な回数だけ更新する」という方式が定石です。カメラ操作程度なら、可変で問題ありません。

---

## 22.3 カメラを実装する

### 22.3.1 共通の部分

**カメラが持つべきものは、ビュー行列と射影行列です。**

```cpp
// src/Graphics/Camera.h
#pragma once
#include "std_import.h"
#include "Math/Matrix.h"

namespace Graphics
{
    class Camera
    {
    public:
        virtual ~Camera() = default;

        virtual void Update(const Input::InputSystem& input, float deltaTime) = 0;

        //--- 行列 ---
        Math::Matrix4x4 ViewMatrix() const;
        Math::Matrix4x4 ProjectionMatrix() const;

        //--- 射影の設定 ---
        void SetPerspective(float fovY, float aspect,
                            float nearZ, float farZ);
        void SetAspectRatio(float aspect);

        //--- 状態 ---
        Math::Vector3 Position() const noexcept { return m_position; }
        Math::Vector3 Forward()  const noexcept;

    protected:
        Math::Vector3 m_position{ 0.0f, 0.0f, -5.0f };
        Math::Vector3 m_target{ 0.0f, 0.0f, 0.0f };

        float m_fovY   = Math::ToRadians(60.0f);
        float m_aspect = 16.0f / 9.0f;
        float m_nearZ  = 0.5f;
        float m_farZ   = 500.0f;
    };
}
```

```cpp
Math::Matrix4x4 Camera::ProjectionMatrix() const
{
    // 第19章 19.5.5 節の MakeProjection を使う。
    // Reversed-Z の有無を意識しなくてよい。
    return MakeProjection(m_fovY, m_aspect, m_nearZ, m_farZ);
}
```

**第19章で作った `MakeProjection` が、ここで効きます。** Reversed-Z を切り替えても、カメラ側のコードは変わりません。

### 22.3.2 軌道カメラ(オービット)

**注視点を中心に、球面上を移動するカメラです。**

モデルビューアに適しています。**第23章でモデルを読み込むとき、これがあると全方向から確認できます。**

```
        ●  カメラ
       /|
      / |  distance
     /  |
    /   θ (pitch)
   +----●  target
    \   |
     φ (yaw)
```

```cpp
class OrbitCamera : public Camera
{
public:
    void Update(const Input::InputSystem& input, float deltaTime) override;

    void SetTarget(const Math::Vector3& target) { m_target = target; }
    void SetDistance(float distance) { m_distance = distance; }

private:
    void UpdatePosition();

    float m_yaw      = 0.0f;
    float m_pitch    = Math::ToRadians(20.0f);
    float m_distance = 5.0f;

    float m_rotateSpeed = 0.005f;   // ラジアン / ピクセル
    float m_zoomSpeed   = 0.1f;
    float m_panSpeed    = 0.002f;
};
```

```cpp
void OrbitCamera::Update(const Input::InputSystem& input, float deltaTime)
{
    const auto delta = input.MouseDelta();

    //--- 左ドラッグ:回転 ---
    if (input.IsMouseDown(0))
    {
        m_yaw   += delta.x * m_rotateSpeed;
        m_pitch += delta.y * m_rotateSpeed;

        //--- 真上・真下で破綻しないよう制限する ---
        // 第17章 17.5.2 節のコラムで予告した対処
        constexpr float kLimit = Math::HalfPi - 0.01f;
        m_pitch = std::clamp(m_pitch, -kLimit, kLimit);
    }

    //--- 中ドラッグ:平行移動 ---
    if (input.IsMouseDown(2))
    {
        // カメラの右方向と上方向へ動かす
        const auto view = ViewMatrix();
        const Math::Vector3 right{ view.m[0][0], view.m[1][0], view.m[2][0] };
        const Math::Vector3 up   { view.m[0][1], view.m[1][1], view.m[2][1] };

        const float scale = m_distance * m_panSpeed;
        m_target -= right * (delta.x * scale);
        m_target += up    * (delta.y * scale);
    }

    //--- ホイール:ズーム ---
    const float wheel = input.WheelDelta();
    if (wheel != 0.0f)
    {
        // 距離に比例させると、近くでは細かく、遠くでは大きく動く
        m_distance *= std::pow(1.0f - m_zoomSpeed, wheel);
        m_distance = std::clamp(m_distance, 0.1f, 1000.0f);
    }

    UpdatePosition();
}

void OrbitCamera::UpdatePosition()
{
    //--- 球面座標から直交座標へ ---
    const float cosPitch = std::cos(m_pitch);

    const Math::Vector3 offset{
        m_distance * cosPitch * std::sin(m_yaw),
        m_distance * std::sin(m_pitch),
        m_distance * cosPitch * -std::cos(m_yaw),
    };

    m_position = m_target + offset;
}
```

**ピッチの制限が重要です。**

第17章 17.5.2 節のコラムで予告した通り、**視線と `up` ベクトルが平行になると `LookAtLH` が壊れます。** 外積がゼロベクトルになり、正規化で 0 除算が起きます。

**±90 度のわずかに手前で止めるのが、最も簡単で確実な対処です。**

**ズームを距離に比例させている**のも実用上の工夫です。固定量だと、遠くにいるときは遅すぎ、近づくと行き過ぎます。

### 22.3.3 FPS カメラ

**視点そのものを動かすカメラです。**

```cpp
class FpsCamera : public Camera
{
public:
    void Update(const Input::InputSystem& input, float deltaTime) override;

private:
    float m_yaw   = 0.0f;
    float m_pitch = 0.0f;

    float m_moveSpeed   = 5.0f;      // 単位 / 秒
    float m_boostFactor = 4.0f;      // Shift 押下時
    float m_lookSpeed   = 0.003f;    // ラジアン / ピクセル
};
```

```cpp
void FpsCamera::Update(const Input::InputSystem& input, float deltaTime)
{
    //--- 右ドラッグ中だけ視点を回す ---
    if (input.IsMouseDown(1))
    {
        const auto delta = input.MouseDelta();
        m_yaw   += delta.x * m_lookSpeed;
        m_pitch += delta.y * m_lookSpeed;

        constexpr float kLimit = Math::HalfPi - 0.01f;
        m_pitch = std::clamp(m_pitch, -kLimit, kLimit);
    }

    //--- 向きベクトルを求める ---
    const float cosPitch = std::cos(m_pitch);
    const Math::Vector3 forward{
        cosPitch * std::sin(m_yaw),
        std::sin(-m_pitch),          // 画面の上下と一致させる
        cosPitch * std::cos(m_yaw),
    };

    const Math::Vector3 worldUp{ 0.0f, 1.0f, 0.0f };
    const Math::Vector3 right = Math::Normalize(Math::Cross(worldUp, forward));

    //--- 移動 ---
    Math::Vector3 move{};
    if (input.IsKeyDown('W')) move += forward;
    if (input.IsKeyDown('S')) move -= forward;
    if (input.IsKeyDown('D')) move += right;
    if (input.IsKeyDown('A')) move -= right;
    if (input.IsKeyDown('E')) move += worldUp;
    if (input.IsKeyDown('Q')) move -= worldUp;

    if (Math::LengthSquared(move) > 0.0f)
    {
        //--- 正規化しないと斜め移動が速くなる ---
        move = Math::Normalize(move);

        float speed = m_moveSpeed;
        if (input.IsKeyDown(VK_SHIFT)) speed *= m_boostFactor;
        if (input.IsKeyDown(VK_CONTROL)) speed /= m_boostFactor;

        m_position += move * (speed * deltaTime);
    }

    m_target = m_position + forward;
}
```

**斜め移動の正規化を忘れないでください。**

`W` と `D` を同時に押すと、正規化しない場合の移動量は √2 倍になります。**斜めに走ると速い**という、古典的なバグです。

**`speed * deltaTime` を掛けている**のが、22.2 節の成果です。フレームレートが変わっても、移動速度は一定です。

### 22.3.4 カメラの切り替え

```cpp
if (input.WasKeyPressed(VK_TAB))
{
    m_useFpsCamera = !m_useFpsCamera;
    LOG_INFO(L"camera: {}", m_useFpsCamera ? L"FPS" : L"Orbit");
}

Camera& camera = m_useFpsCamera
    ? static_cast<Camera&>(m_fpsCamera)
    : static_cast<Camera&>(m_orbitCamera);

camera.Update(input, timer.DeltaSeconds());
```

**`WasKeyPressed` を使っている**ので、押しっぱなしでも 1 回しか切り替わりません(22.1.5 節)。

---

## 22.4 フレームループへの組み込み

### 22.4.1 更新の順序

```cpp
while (window.ProcessMessages())
{
    //--- ① リサイズ要求の処理(第12章 12.4.4 節)---
    if (g_pendingResize)
    {
        const auto [w, h] = *g_pendingResize;
        g_pendingResize.reset();
        renderer.Resize(w, h);
        camera.SetAspectRatio(static_cast<float>(w) / static_cast<float>(h));
    }

    if (window.IsMinimized()) continue;

    //--- ② 時間を進める ---
    timer.Tick();

    //--- ③ 入力を確定する ---
    input.BeginFrame();

    //--- ④ 更新 ---
    camera.Update(input, timer.DeltaSeconds());

    //--- ⑤ 描画 ---
    if (auto r = renderer.RenderFrame(camera, timer); !r)
    {
        Core::ReportError(r.error());
        break;
    }
}
```

**順序に理由があります。**

- ① が先なのは、アスペクト比の更新が描画より前に必要だから
- ② が ③ より先なのは、入力処理でデルタタイムを使う可能性があるから
- ③ が ④ より先なのは、`BeginFrame` で入力を確定させてから読むため

**`camera.SetAspectRatio` を忘れないでください。** リサイズしても射影行列を更新しないと、**縦横比が崩れたまま**になります。第18章 18.5.2 節で `m_width` / `m_height` から毎フレーム計算していた部分を、カメラへ移します。

### 22.4.2 定数バッファを更新する

```cpp
void Renderer::UpdateSceneConstants(const Camera& camera)
{
    const auto world = Math::Matrix4x4::Identity();   // 立方体は動かさない
    const auto view  = camera.ViewMatrix();
    const auto proj  = camera.ProjectionMatrix();

    SceneConstants constants{};
    constants.worldViewProj = world * view * proj;

    //--- 第21章のリングバッファから借りる ---
    const auto allocation = m_uploadBuffer.AllocateConstants(sizeof(constants));
    allocation.Write(constants);

    //--- 一時デスクリプタに CBV を作る ---
    const auto handle = m_descriptorHeap.Dynamic().Allocate();

    D3D12_CONSTANT_BUFFER_VIEW_DESC cbvDesc{};
    cbvDesc.BufferLocation = allocation.gpuAddress;
    cbvDesc.SizeInBytes    = static_cast<UINT>(
        AlignUp(sizeof(constants),
                D3D12_CONSTANT_BUFFER_DATA_PLACEMENT_ALIGNMENT));

    m_device->CreateConstantBufferView(&cbvDesc, handle.cpu);
    m_cbvHandleThisFrame = handle;
}
```

**立方体の回転を止めました。** カメラが動くようになったので、モデル側を回す必要がありません。**第23章でモデルを読み込むときも、この形のままです。**

---

## 22.5 情報を表示する

**カメラを操作していると、「今どこにいるか」を知りたくなります。**

```cpp
void UpdateWindowTitle(HWND hwnd, const Camera& camera,
                       const Core::Timer& timer, bool wireframe)
{
    // 0.5 秒ごとに更新する。毎フレームだと重い
    static float accumulated = 0.0f;
    accumulated += timer.DeltaSeconds();
    if (accumulated < 0.5f) return;
    accumulated = 0.0f;

    const auto pos = camera.Position();

    const std::wstring title = std::format(
        L"D3D12Book - Chapter 22   "
        L"[{:.1f} fps  {:.2f} ms]   "
        L"pos ({:.1f}, {:.1f}, {:.1f})   "
        L"{}",
        timer.Fps(), 1000.0f / timer.Fps(),
        pos.x, pos.y, pos.z,
        wireframe ? L"wireframe" : L"solid");

    ::SetWindowTextW(hwnd, title.c_str());
}
```

**第17章 17.2.4 節で `std::formatter` を書いたので、こう書くこともできます。**

```cpp
LOG_INFO(L"camera at {}", camera.Position());
```

```
[Info ] main.cpp(128): camera at (  2.500,   1.800,  -4.200)
```

**位置がおかしいときに、数値で確認できるのは大きな助けになります。**

---

## ✅ 本章のゴール:自由に見回せる

### Step 1:軌道カメラを試す

| 操作 | 動作 |
|---|---|
| **左ドラッグ** | 立方体の周りを回る |
| **中ドラッグ** | 注視点を平行移動 |
| **ホイール** | ズーム |

**立方体を全方向から観察できることを確認してください。**

- 6 面すべてにテクスチャが正しく貼られている(第20章)
- 真上・真下から見ても破綻しない(ピッチ制限)
- 近づいても遠ざかっても正しく描かれる

### Step 2:FPS カメラを試す

`Tab` で切り替えます。

| 操作 | 動作 |
|---|---|
| **右ドラッグ** | 視点を回す |
| **W / S** | 前後 |
| **A / D** | 左右 |
| **Q / E** | 上下 |
| **Shift** | 加速 |
| **Ctrl** | 減速 |

**画面端でカーソルが止まっても、視点は回り続けます。** これが Raw Input の効果です(22.1.1 節)。

### Step 3:ピッチ制限を外してみる

```cpp
// m_pitch = std::clamp(m_pitch, -kLimit, kLimit);   ← コメントアウト
```

**真上を見上げようとすると、画面が激しく回転して破綻します。**

第17章 17.5.2 節で予告した「`up` と視線が平行になると壊れる」の実物です。**行列側で対処するのではなく、入力側で防ぐのが定石**だと書いた理由が分かります。

**確認したら元に戻してください。**

### Step 4:デルタタイムを外してみる

```cpp
m_position += move * speed;      // ❌ deltaTime を掛けない
```

**移動が異常に速くなります。**

60fps なら毎秒 300 単位、144fps なら 720 単位です。**環境によって速度が変わることも確認できます。**

**確認したら元に戻してください。**

### Step 5:デルタタイムの上限を外す

```cpp
// if (m_delta > kMaxDelta) { m_delta = kMaxDelta; }   ← コメントアウト
```

**FPS カメラで移動しながら、ブレークポイントで止めてください。**

再開した瞬間、**カメラが遥か彼方へ飛びます。** ウィンドウのタイトルバーを長くドラッグしても同じことが起きます。

**22.2.3 節でクランプが必須だと書いた理由です。**

**確認したら元に戻してください。**

### Step 6:斜め移動の正規化を外す

```cpp
// move = Math::Normalize(move);   ← コメントアウト
```

**`W` + `D` の斜め移動が、単独より約 1.41 倍速くなります。**

体感で分かりにくい場合は、タイトルバーの座標表示を見てください。

**確認したら元に戻してください。**

### Step 7:`MOUSE_MOVE_ABSOLUTE` の確認を外す(任意)

**リモートデスクトップ環境がある場合のみ試せます。**

```cpp
// if (raw->data.mouse.usFlags & MOUSE_MOVE_ABSOLUTE) { return; }
```

**カーソルを少し動かしただけで、カメラが吹き飛びます。** 絶対座標(数千)を移動量として扱うためです。

**通常のローカル環境では再現しません。** だからこそ、**「自分の環境で動いた」で済ませてはいけない**箇所です。

---

### 本章の達成状態

- [ ] Raw Input を登録した(`RIDEV_NOLEGACY` は使わない)
- [ ] `WM_INPUT` で `DefWindowProcW` を呼んでいる
- [ ] `MOUSE_MOVE_ABSOLUTE` を確認している
- [ ] マウス移動量を `+=` で累積している
- [ ] `BeginFrame` で入力を確定している
- [ ] `IsKeyDown` と `WasKeyPressed` を使い分けている
- [ ] 第16章のワイヤーフレーム切り替えを置き換えた
- [ ] `QueryPerformanceCounter` でタイマを実装した
- [ ] デルタタイムに上限を設けた
- [ ] ピッチを ±90 度手前で制限した
- [ ] 斜め移動を正規化した
- [ ] 移動速度に `deltaTime` を掛けた
- [ ] リサイズ時にアスペクト比を更新している
- [ ] **軌道カメラと FPS カメラの両方が動く**
- [ ] Step 3 〜 6 で、それぞれの破綻を確認した

---

## トラブルシューティング

| 症状 | 原因 | 対処 |
|---|---|---|
| `WM_INPUT` が来ない | 登録に失敗している | `RegisterRawInputDevices` の戻り値を確認 |
| マウスメッセージが止まる | `RIDEV_NOLEGACY` を指定した | 外す(22.1.3) |
| メモリが増え続ける | `DefWindowProcW` を呼んでいない | 22.1.4 |
| 動きが取りこぼされる | `=` で上書きしている | `+=` にする(22.1.4) |
| リモートデスクトップで暴走 | `MOUSE_MOVE_ABSOLUTE` 未確認 | 22.1.4 |
| 真上を見ると破綻 | ピッチ制限がない | 22.3.2 |
| 環境で速度が変わる | `deltaTime` を掛けていない | 22.2.1 |
| デバッグ再開で飛ぶ | デルタタイムの上限がない | 22.2.3 |
| 斜めが速い | 正規化していない | 22.3.3 |
| リサイズで歪む | アスペクト比を更新していない | 22.4.1 |
| ワイヤーフレームが点滅 | `IsKeyDown` を使っている | `WasKeyPressed`(22.1.5) |
| ズームが遅い / 行き過ぎる | 固定量で変化させている | 距離に比例させる(22.3.2) |

---

## まとめ

**1. 視点操作には Raw Input を使う。**
`WM_MOUSEMOVE` は画面端で止まり、OS の加速が掛かり、解像度が失われます。**ただし `RIDEV_NOLEGACY` は使いません。** 通常のマウス操作ができなくなります。

**2. `MOUSE_MOVE_ABSOLUTE` を確認する。**
リモートデスクトップや仮想マシンでは絶対座標が届きます。**自分の環境では再現しないバグの典型です。**

**3. デルタタイムには上限を設ける。**
ブレークポイントやウィンドウのドラッグで、値が数十秒になります。**クランプがなければカメラが宇宙へ飛びます。**

**4. ピッチは ±90 度の手前で止める。**
第17章 17.5.2 節で予告した `LookAtLH` の破綻を、**入力側で防ぎます。** 行列を複雑にする必要はありません。

**5. 斜め移動は正規化する。**
古典的なバグですが、実際によく見かけます。

**6. 「押されている」と「今押された」を区別する。**
移動には `IsKeyDown`、切り替えには `WasKeyPressed`。**第16章の暫定実装にあったキーリピートの問題が、これで消えます。**

**7. 動かせるようになったことが、この先の検証手段になる。**
第23章の法線、第24章のライティング、第27章の影 —— **すべて、視点を変えられて初めて正しさが分かります。**

次章ではモデルを読み込みます。OBJ / MTL のパーサを自作し、頂点の重複排除、法線と接空間の計算まで行います。**第16章 16.1.1 節で予告した「法線があると 8 頂点では足りない」問題に、正面から取り組みます。**

---

## 参考リンク

| 内容 | URL |
|---|---|
| Raw Input について | https://learn.microsoft.com/ja-jp/windows/win32/inputdev/about-raw-input |
| `RAWINPUTDEVICE` | https://learn.microsoft.com/ja-jp/windows/win32/api/winuser/ns-winuser-rawinputdevice |
| `RAWMOUSE` | https://learn.microsoft.com/ja-jp/windows/win32/api/winuser/ns-winuser-rawmouse |
| `QueryPerformanceCounter` | https://learn.microsoft.com/ja-jp/windows/win32/api/profileapi/nf-profileapi-queryperformancecounter |
| 高解像度タイムスタンプの取得 | https://learn.microsoft.com/ja-jp/windows/win32/sysinfo/acquiring-high-resolution-time-stamps |
| 仮想キーコード | https://learn.microsoft.com/ja-jp/windows/win32/inputdev/virtual-key-codes |
