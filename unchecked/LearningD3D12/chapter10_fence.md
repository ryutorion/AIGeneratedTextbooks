# 第10章 フェンスによるCPU/GPU同期

前章は、宿題を残したまま終わりました。

**GPU がコマンドリストの実行を終えたかどうかを知る手段がない。** そのせいでコマンドアロケータを `Reset` できず、フレームループを回せませんでした。終了処理も、GPU が実行中かもしれない状態で押し通しました。

本章で、その手段を手に入れます。使う API は 3 つだけです。

```cpp
queue->Signal(fence, value);           // GPU に「ここまで来たら印を付けろ」
fence->GetCompletedValue();            // 「今どこまで来た?」
fence->SetEventOnCompletion(value, e); // 「そこまで来たら教えて」
```

**たったこれだけで、D3D12 のすべての同期が表現できます。** CPU が GPU を待つのも、GPU が別のキューを待つのも、フレームをパイプライン化するのも(第12章)、根っこはこの 3 つです。

そして本章には、もう一つ重要な主題があります。**待ちが返ってこなかったらどうするか。** GPU がハングしたとき、素朴に書いた待機コードはアプリを永久に凍らせます。ユーザーはタスクマネージャで強制終了するしかなく、クラッシュダンプも残りません。**第8章で組み込んだ Aftermath が働くためには、待機側の設計が正しくなければなりません。**

**本章のゴール**
`Fence` クラスを実装し、実行完了を確実に待てるようにする。フレームループを(暫定的に)動かし、終了時に GPU の完了を待ってから破棄する。あわせて、待機のタイムアウトとデバイスロスト検出を組み込む。

---

## 10.1 フェンスとは何か

### 10.1.1 単調増加する 64bit カウンタ

`ID3D12Fence` の実体は、**CPU と GPU の両方から見える 64 ビットの整数**です。それ以上でも以下でもありません。

```cpp
ComPtr<ID3D12Fence> fence;
HR_TRY(device->CreateFence(
    0,                          // 初期値
    D3D12_FENCE_FLAG_NONE,      // フラグ
    IID_PPV_ARGS(&fence)));
```

この数値の使い方が、すべてです。

```
値 = 「GPU がどこまで進んだか」を表す目盛り
```

CPU 側は「この目盛りまで進んだら教えて」と依頼し、GPU 側は「ここまで来たら目盛りを N にしろ」という命令をコマンドの流れの中で実行します。

> **なぜ 64 ビットなのか**
>
> 溢れないためです。仮に毎秒 1 万回シグナルしたとしても、`UINT64` を使い切るには 5,800 万年かかります。**折り返しを気にする必要はありません。**
>
> 32 ビットだと 1 日ももちません。値の巻き戻りを考慮した比較を書く羽目になり、そこは間違いなくバグの温床になります。64 ビットという選択は、その面倒を丸ごと消しています。

### 10.1.2 `Signal` は「積まれる」

**ここが最も重要な理解です。**

```cpp
queue->Signal(fence.Get(), 42);
```

この呼び出しは、**その場でフェンスの値を 42 にするわけではありません。** キューに「フェンスを 42 にする」という命令を**積みます**。

キューは投入された順に処理するので(第9章 9.5.2 節)、この命令が実行されるのは、**それ以前に投入されたすべてのコマンドリストが完了した後**です。

```
キューの中身:
  ┌─────────────────┐
  │ CommandList A   │  ← 先に投入
  ├─────────────────┤
  │ CommandList B   │
  ├─────────────────┤
  │ Signal(fence,42)│  ← A と B が終わってから実行される
  └─────────────────┘
```

だから、**「フェンスが 42 になった」= 「A と B の実行が完了した」** と言えるのです。フェンスが完了マーカーとして機能するのは、この順序性のおかげです。

`queue->Signal()` 自体は `ExecuteCommandLists` と同じく即座に戻ります。**戻った時点では、まだ何も終わっていません。**

### 10.1.3 `GetCompletedValue`

現在の値を CPU から読みます。

```cpp
const UINT64 completed = fence->GetCompletedValue();
```

「完了した(completed)値」という名前が示す通り、**この値以下のシグナルはすべて実行済み**です。

```cpp
if (fence->GetCompletedValue() >= 42)
{
    // Signal(fence, 42) の時点までのコマンドは完了している
}
```

比較は `==` ではなく `>=` を使ってください。フェンスの値は飛ぶことがあります。43 や 44 がすでにシグナルされていれば、`GetCompletedValue()` は 42 を通り越しています。

> **デバイスロスト時の戻り値**
>
> `GetCompletedValue()` は、**デバイスが失われている場合に `UINT64_MAX` を返します。** これは仕様として定められた挙動で、10.5 節でデバイスロストの検出に使います。
>
> 逆に言えば、`UINT64_MAX` を意図的にシグナルしてはいけません。区別が付かなくなります。

### 10.1.4 4 つの操作

フェンスに対する操作は 4 つあります。本章で使うのは上 2 つですが、全体像を先に示します。

| 操作 | 誰が値を変える/待つか | 使う章 |
|---|---|---|
| `ID3D12CommandQueue::Signal(fence, v)` | **GPU が**値を v にする(キューに積まれる) | **本章** |
| `ID3D12Fence::SetEventOnCompletion(v, e)` | **CPU が**待つ | **本章** |
| `ID3D12Fence::Signal(v)` | **CPU が**即座に値を v にする | 第35章 |
| `ID3D12CommandQueue::Wait(fence, v)` | **GPU が**待つ(キューが停止する) | 第35章 |

上 2 つの組み合わせが「CPU が GPU を待つ」、下 2 つの組み合わせが「GPU が CPU を待つ」、そして GPU の `Signal` と別キューの `Wait` を組み合わせると「GPU が GPU を待つ」になります。

**本章では「CPU が GPU を待つ」だけを扱います。** 残りは第35章で、コピーキューと描画キューを協調させるときに使います。

### 10.1.5 フェンスはキューごとに持つ

複数のキュー(DIRECT / COMPUTE / COPY)を使うようになったら、**フェンスもキューごとに用意してください。**

1 つのフェンスを複数のキューからシグナルすると、値の増加順序が保証されなくなります。「フェンスが 42 になった」が「どのキューのどこまでが終わったか」を意味しなくなり、意味を失います。

本章はキューが 1 つなので、フェンスも 1 つです。

---

## 10.2 イベントオブジェクトによる待機

### 10.2.1 ポーリングしてはいけない

「フェンスの値を読めるなら、値が上がるまで回せばいいのでは」と考えたくなります。

```cpp
// ❌ やってはいけない
while (fence->GetCompletedValue() < value)
{
    // 何もせずに回り続ける
}
```

これは **CPU コアを 1 つ、100% で焼き続けます。** ノート PC ならファンが全開になり、バッテリーが溶けます。しかも GPU の処理が速くなるわけでもありません。

正しい方法は、**OS に「起こしてくれ」と頼んで眠ること**です。そのための道具が Win32 のイベントオブジェクトです。

### 10.2.2 イベントオブジェクトを作る

```cpp
HANDLE event = ::CreateEventW(
    nullptr,    // セキュリティ属性
    FALSE,      // bManualReset : FALSE = 自動リセット
    FALSE,      // bInitialState: FALSE = 非シグナル状態
    nullptr);   // 名前(不要)

if (event == nullptr)
{
    return std::unexpected(Core::MakeError(
        HRESULT_FROM_WIN32(::GetLastError()), L"CreateEventW"));
}
```

**第 2 引数の `FALSE`(自動リセット)が重要です。**

自動リセットのイベントは、待機が解除された瞬間に自動的に非シグナル状態へ戻ります。手動リセット(`TRUE`)にすると、一度シグナルされた後ずっとシグナル状態のままになり、**次の待機が即座に通過してしまいます。**

**イベントハンドルは、必ず `CloseHandle` で閉じてください。** 閉じ忘れはハンドルリークです。RAII で管理します(10.3 節)。

### 10.2.3 `SetEventOnCompletion`

```cpp
HR_TRY(fence->SetEventOnCompletion(value, event));
::WaitForSingleObject(event, timeoutMs);
```

「フェンスの値が `value` 以上になったら、このイベントをシグナルしてほしい」という依頼です。依頼を出したら `WaitForSingleObject` で眠ります。

**すでに `value` に達している場合、イベントは即座にシグナルされます。** つまり、この 2 行はどんな状況でも安全に呼べます。

> **`nullptr` を渡すこともできる**
>
> `SetEventOnCompletion(value, nullptr)` と書くと、イベントを作らずに、その場でブロックします。手軽ですが、**タイムアウトを指定できません。** GPU がハングしたら永久に返ってきません。
>
> 本書は使いません。10.5 節で説明する通り、**タイムアウトは必須**です。

### 10.2.4 待つ前に確認する

```cpp
if (fence->GetCompletedValue() >= value)
{
    return {};    // すでに完了している。カーネルに入る必要すらない
}

HR_TRY(fence->SetEventOnCompletion(value, event));
::WaitForSingleObject(event, timeoutMs);
```

先頭の `if` は最適化です。**なくても正しく動きます**(前項の通り、`SetEventOnCompletion` は達成済みの値を扱えます)。

それでも書く価値があります。`SetEventOnCompletion` と `WaitForSingleObject` はいずれもカーネルモードへの遷移を伴い、往復で数マイクロ秒から数十マイクロ秒かかります。**GPU がすでに終わっているケースは実際によくある**ので、その分を節約できます。

第12章でフレームをパイプライン化すると、「待つ必要がなかった」がむしろ普通の状態になります。そのときこの `if` が効いてきます。

---

## 10.3 `Fence` クラスを実装する

### 10.3.1 設計

前章では「まだ抽象化しない」と決めましたが、フェンスは事情が違います。

- フェンス、イベントハンドル、次に使う値 —— この 3 つは**常に一緒に動きます**
- イベントハンドルは RAII で閉じる必要があります
- 値の管理を間違えると、**永久に返ってこない待機**が生まれます

**危険が「値の管理」に集中しており、そこを間違えないための抽象化なので、隠すことに意味があります。** 第9章のアロケータとは逆の判断です。

```cpp
// src/Graphics/Fence.h
#pragma once
#include "std_import.h"
#include "Core/Error.h"

namespace Graphics
{
    class Fence
    {
    public:
        // TDR の既定値(2 秒)より長く取る。理由は 10.5.2 節。
        static constexpr DWORD kDefaultTimeoutMs = 5000;

        Fence() = default;
        ~Fence();

        Fence(const Fence&)            = delete;
        Fence& operator=(const Fence&) = delete;

        Core::Status Initialize(ID3D12Device* device, std::wstring_view name);
        void         Shutdown();

        // キューに Signal を積み、その値を返す
        Core::Result<std::uint64_t> Signal(ID3D12CommandQueue* queue);

        // 指定の値に到達しているか(待たない)
        bool IsComplete(std::uint64_t value) const;

        // 指定の値に到達するまで待つ
        Core::Status Wait(std::uint64_t value,
                          DWORD timeoutMs = kDefaultTimeoutMs);

        std::uint64_t LastSignaledValue() const noexcept
        {
            return m_nextValue - 1;
        }

        ID3D12Fence* Get() const noexcept { return m_fence.Get(); }

    private:
        Microsoft::WRL::ComPtr<ID3D12Fence> m_fence;
        HANDLE        m_event     = nullptr;
        std::uint64_t m_nextValue = 1;      // 理由は 10.3.2 節
    };

    // 投入済みのすべての作業が完了するまで待つ
    Core::Status WaitForGpuIdle(ID3D12CommandQueue* queue, Fence& fence);
}
```

### 10.3.2 値を 1 から始める理由

```cpp
std::uint64_t m_nextValue = 1;   // 0 ではなく 1
```

フェンスは初期値 `0` で生成されます。したがって、**生成直後から `GetCompletedValue()` は 0 を返します。**

もし最初のシグナルを 0 にすると、「まだ何もシグナルしていない」と「1 回目のシグナルが完了した」が区別できません。

```cpp
fence->CreateFence(0, ...);              // 値は 0
queue->Signal(fence.Get(), 0);           // ❌ 何も意味しない
fence->GetCompletedValue() >= 0;         // 常に true
```

**1 から始めれば、この曖昧さが消えます。** 些細に見えますが、第12章でフレームごとのフェンス値を扱うとき、「このフレームはまだ一度も投入されていない」を 0 で表せることが効いてきます。

### 10.3.3 実装

```cpp
// src/Graphics/Fence.cpp
#include "pch.h"
#include "std_import.h"
#if USE_STD_MODULE
import std;
#endif
#include "Graphics/Fence.h"
#include "Core/Log.h"
#include "Core/DebugName.h"

namespace Graphics
{
    Fence::~Fence()
    {
        Shutdown();
    }

    Core::Status Fence::Initialize(ID3D12Device* device,
                                   std::wstring_view name)
    {
        HR_TRY(device->CreateFence(
            0,                          // 初期値
            D3D12_FENCE_FLAG_NONE,
            IID_PPV_ARGS(&m_fence)));

        Core::SetDebugName(m_fence.Get(), name);

        // 自動リセット・非シグナル状態(10.2.2)
        m_event = ::CreateEventW(nullptr, FALSE, FALSE, nullptr);
        if (m_event == nullptr)
        {
            return std::unexpected(Core::MakeError(
                HRESULT_FROM_WIN32(::GetLastError()), L"CreateEventW"));
        }

        m_nextValue = 1;
        LOG_INFO(L"fence created: {}", name);
        return {};
    }

    void Fence::Shutdown()
    {
        if (m_event != nullptr)
        {
            ::CloseHandle(m_event);     // ハンドルリークを防ぐ
            m_event = nullptr;
        }
        m_fence.Reset();
    }

    Core::Result<std::uint64_t> Fence::Signal(ID3D12CommandQueue* queue)
    {
        const std::uint64_t value = m_nextValue;

        // キューに「ここまで来たら value にしろ」を積む(10.1.2)
        HR_TRY(queue->Signal(m_fence.Get(), value));

        ++m_nextValue;
        return value;
    }

    bool Fence::IsComplete(std::uint64_t value) const
    {
        return m_fence->GetCompletedValue() >= value;
    }

    Core::Status Fence::Wait(std::uint64_t value, DWORD timeoutMs)
    {
        //--- ① すでに到達していれば何もしない(10.2.4) ---
        const std::uint64_t completed = m_fence->GetCompletedValue();

        if (completed == std::numeric_limits<std::uint64_t>::max())
        {
            // デバイスロストの兆候(10.1.3)
            return std::unexpected(Core::MakeError(
                DXGI_ERROR_DEVICE_REMOVED,
                L"fence returned UINT64_MAX (device removed)"));
        }
        if (completed >= value)
        {
            return {};
        }

        //--- ② 到達を通知してもらう ---
        HR_TRY(m_fence->SetEventOnCompletion(value, m_event));

        //--- ③ 眠る ---
        const DWORD result = ::WaitForSingleObject(m_event, timeoutMs);

        switch (result)
        {
        case WAIT_OBJECT_0:
            return {};

        case WAIT_TIMEOUT:
            LOG_ERROR(L"fence wait timed out: waiting {} but completed {}",
                      value, m_fence->GetCompletedValue());
            return std::unexpected(Core::MakeError(
                HRESULT_FROM_WIN32(WAIT_TIMEOUT), L"fence wait timed out"));

        default:
            return std::unexpected(Core::MakeError(
                HRESULT_FROM_WIN32(::GetLastError()),
                L"WaitForSingleObject failed"));
        }
    }

    Core::Status WaitForGpuIdle(ID3D12CommandQueue* queue, Fence& fence)
    {
        auto value = fence.Signal(queue);
        if (!value)
        {
            return std::unexpected(value.error());
        }
        return fence.Wait(*value);
    }
}
```

**`Signal` が `Result<uint64_t>` を返している**点に注目してください。`ID3D12CommandQueue::Signal` は `HRESULT` を返します。デバイスが失われていれば失敗するので、無視できません。

---

## 10.4 終了時に必ず待つ

### 10.4.1 待たないと何が起きるか

第9章の終了処理は、こうなっていました。

```cpp
// GPU がまだ実行中かもしれないが、待つ手段は第10章まで手に入らない
device.Shutdown();
```

**このコードが何をしているか、正確に考えてみます。**

1. `ComPtr` のデストラクタが `Release` を呼ぶ
2. 参照カウントが 0 になり、オブジェクトが破棄される
3. コマンドアロケータの持つメモリが解放される
4. **GPU がまだそのメモリを読んでいたら?**

D3D12 は、**GPU がリソースを使用中かどうかを参照カウントで管理しません。** これは D3D11 との決定的な違いです。D3D11 ではドライバが遅延解放してくれましたが、D3D12 では解放した瞬間に解放されます。

結果として起きること:

| 症状 | 説明 |
|---|---|
| デバッグレイヤーの警告 | 使用中のオブジェクトを破棄したという指摘 |
| デバイスロスト | GPU が無効なメモリを読んだ |
| ドライバ内でのクラッシュ | 呼び出し履歴が D3D12 の内部で終わっていて追えない |
| **何も起きない** | **一番たちが悪い。たまたま間に合っただけ** |

最後の行が問題です。空のコマンドリストなら、CPU が破棄処理を書き終える前に GPU が終わっているでしょう。**動いているように見えます。** しかし第13章で三角形を描き、第20章でテクスチャを転送するようになると、いつか間に合わなくなります。

**そして、そのとき原因がここにあるとは気づけません。**

### 10.4.2 `WaitForGpuIdle`

```cpp
Core::Status WaitForGpuIdle(ID3D12CommandQueue* queue, Fence& fence)
{
    auto value = fence.Signal(queue);   // 「全部終わったら印を付けろ」
    if (!value) return std::unexpected(value.error());
    return fence.Wait(*value);          // その印が付くまで待つ
}
```

**シンプルですが、これが「投入済みの作業がすべて終わるまで待つ」の正体です。**

キューに積まれたシグナルは、それ以前のすべてのコマンドリストの後に実行されます(10.1.2)。だから、そのシグナルを待てば、投入済みの全作業を待ったことになります。

終了処理は次のようになります。

```cpp
//--- 終了処理 ---
if (auto r = Graphics::WaitForGpuIdle(g_queue.Get(), g_fence); !r)
{
    Core::ReportError(r.error());
}

g_commandList.Reset();
g_allocator.Reset();
g_fence.Shutdown();
g_queue.Shutdown();
device.Shutdown();
```

**順序も意識してください。** 待機が最初、そのあと子オブジェクトから順に破棄し、デバイスを最後に破棄します(第6章 6.2.6 節)。

### 10.4.3 これは重い操作である

**`WaitForGpuIdle` は、正しいが遅い操作です。**

```
CPU: [記録][投入][────── 待機 ──────][記録][投入][──── 待機 ────]
GPU:              [── 実行 ──]                    [── 実行 ──]
         ↑                    ↑
      GPU が遊んでいる      CPU が遊んでいる
```

CPU と GPU が交互にしか働きません。**どちらか一方は常に遊んでいます。** 本来なら、CPU が次のフレームを記録している間に GPU が現在のフレームを描くべきです。

```
理想:
CPU: [記録 N][記録 N+1][記録 N+2][記録 N+3]
GPU:         [実行 N  ][実行 N+1][実行 N+2]
```

これを実現するのが**フレームのパイプライン化**で、第12章の主題です。そこでは「毎フレーム全部待つ」のをやめ、「2 フレーム前の完了だけを待つ」ようにします。

**`WaitForGpuIdle` を使ってよい場面は限られます。**

| 場面 | 使ってよいか |
|---|---|
| アプリケーション終了時 | **必須** |
| スワップチェーンのリサイズ前(第12章) | **必須** |
| リソースを破棄する前 | 必要(第21章でより賢い方法を作る) |
| 毎フレーム | **使ってはいけない** |

---

## 10.5 無限待ちを避ける

### 10.5.1 `INFINITE` を使ってはいけない

多くのサンプルコードは、こう書かれています。

```cpp
fence->SetEventOnCompletion(value, event);
::WaitForSingleObject(event, INFINITE);   // ← 危険
```

正常に動いている間は問題ありません。**問題は、GPU がハングしたときです。**

第32章で無限ループのコンピュートシェーダーを書いてしまったとしましょう。GPU は永久にそのシェーダーを実行し続け、フェンスは永久にシグナルされません。`WaitForSingleObject` は永久に返ってきません。

そのときアプリケーションは:

- ウィンドウが応答しなくなる(メッセージループが回らない)
- 「応答なし」と表示される
- ユーザーはタスクマネージャで強制終了するしかない
- **Aftermath のクラッシュダンプを書き出す機会がない**

最後の行が致命的です。**第8章で Aftermath を組み込んだのは、まさにこういうときのためでした。** それが、待機側の設計のせいで無駄になります。

### 10.5.2 タイムアウト値の決め方 —— TDR との関係

```cpp
static constexpr DWORD kDefaultTimeoutMs = 5000;
```

**この値は、TDR より長く設定します。**

TDR(Timeout Detection and Recovery)は、GPU が応答しなくなったことを OS が検出してドライバをリセットする仕組みです。**既定のタイムアウトは 2 秒**で、レジストリの `TdrDelay` で変更できます。

```
時刻 0.0s : GPU がハングする
時刻 2.0s : TDR が発火 → ドライバがリセットされる
            → デバイスロスト
            → Aftermath のクラッシュダンプ生成が始まる
時刻 5.0s : 我々の待機がタイムアウト
            → デバイスロストを検出できる状態になっている
```

**タイムアウトを TDR より短くすると、順序が逆転します。** まだ TDR が発火していない段階でタイムアウトし、「GPU が遅いだけ」なのか「ハングした」のか判断できません。

5 秒という値に強い根拠はありませんが、「TDR の 2 秒 + ダンプ生成の余裕」として妥当な範囲です。**重要なのは、`INFINITE` でも 100ms でもなく、TDR より少し長い有限値だということです。**

> **TDR を無効化しないこと**
>
> レジストリで `TdrLevel = 0` にすると TDR を無効化できます。**開発機で絶対にやらないでください。** ハングした瞬間に PC ごと固まり、電源ボタン以外の手段がなくなります。
>
> 第9章 9.3.2 節で `DISABLE_GPU_TIMEOUT` フラグを禁じたのと同じ理由です。

### 10.5.3 タイムアウトには 2 つの原因がある

**タイムアウトしたからといって、必ずしも GPU がハングしたわけではありません。**

| 原因 | 見分け方 |
|---|---|
| **GPU がハングした** | `GetDeviceRemovedReason()` が失敗を返す |
| **到達しない値を待った(自分のバグ)** | `GetDeviceRemovedReason()` が `S_OK` |

2 つ目は、フェンス値の管理を間違えたときに起きます。たとえば、シグナルしていない値を待ってしまった場合です。**GPU は元気に動いているのに、こちらが来ない電車を待っている**状態です。

判別するコードを書きます。

```cpp
// src/Graphics/DeviceLost.h
Core::Status HandleWaitFailure(ID3D12Device* device, const Core::Error& error);
```

```cpp
Core::Status HandleWaitFailure(ID3D12Device* device, const Core::Error& error)
{
    const HRESULT reason = device->GetDeviceRemovedReason();

    if (SUCCEEDED(reason))
    {
        // デバイスは生きている → 同期ロジックのバグ
        LOG_FATAL(L"fence wait failed but device is alive. "
                  L"Check fence value bookkeeping.");
        return std::unexpected(error);
    }

    // デバイスロスト
    LOG_FATAL(L"device removed: {}", Core::FormatHResult(reason));
    OnDeviceLost(reason);
    return std::unexpected(Core::MakeError(reason, L"device removed"));
}
```

**この分岐が非常に重要です。** 「タイムアウトしたら即デバイスロスト」と決めつけると、自分のバグを GPU のせいにして何日も探すことになります。

### 10.5.4 Aftermath へつなぐ

デバイスロストを検出したら、**Aftermath のクラッシュダンプが書き出されるのを待ってから終了します。**

第8章 8.5 節で実装した待機処理を、ここで呼びます。

```cpp
void OnDeviceLost(HRESULT reason)
{
    LOG_FATAL(L"=== DEVICE LOST ===");
    LOG_FATAL(L"reason: {}", Core::FormatHResult(reason));

    // 第8章 8.5 節:ダンプ生成の完了を待つ。
    // ここで待たずに終了すると、ダンプが書き終わる前に
    // プロセスが消えてファイルが残らない。
    Aftermath::WaitForCrashDump();

    // 第38章で DRED を導入したら、ここでその内容も出力する
}
```

**「ダンプ生成の完了を待つ」を忘れると、すべてが台無しになります。**

Aftermath のクラッシュダンプ生成は非同期に進みます。デバイスロストを検出してすぐ `return 1;` してしまうと、**書き込み途中でプロセスが終了し、ファイルが残りません。** 第8章でわざわざ `GFSDK_Aftermath_GetCrashDumpStatus` によるポーリングを実装したのは、この瞬間のためです。

第31章で「意図的にクラッシュさせて原因行を特定する」実習を行いますが、**本節のこの数行がないと、その実習は成立しません。**

---

## 10.6 フレームループを完成させる(暫定版)

第9章で書けなかったループが、ようやく書けます。

```cpp
Core::Status RenderFrame()
{
    //--- ① 前フレームの完了を待つ ---
    //     これがあるから ② のアロケータ Reset が安全になる。
    //     ただし毎フレーム全部待つのは遅い(10.4.3)。
    //     第12章でパイプライン化する。
    if (auto r = Graphics::WaitForGpuIdle(g_queue.Get(), g_fence); !r)
    {
        return r;
    }

    //--- ② 記録 ---
    HR_TRY(g_allocator->Reset());       // ← 第9章では書けなかった行
    HR_TRY(g_commandList->Reset(g_allocator.Get(), nullptr));

    // ここに描画コマンドが入る(第11章から)

    HR_TRY(g_commandList->Close());

    //--- ③ 投入 ---
    g_queue.Execute(g_commandList.Get());

    return {};
}
```

```cpp
//--- メインループ ---
while (window.ProcessMessages())
{
    if (window.IsMinimized())
    {
        continue;
    }

    if (auto r = RenderFrame(); !r)
    {
        HandleWaitFailure(device.Device(), r.error());
        break;
    }
}
```

**`allocator->Reset()` に根拠がつきました。** 直前で GPU の完了を待っているので、このアロケータを読んでいる者はもういません。

> **待機の位置について**
>
> 上の実装では、待機をフレームの**先頭**に置いています。末尾(投入の直後)に置いても正しく動きますが、先頭のほうが構造として素直です。
>
> 第12章でパイプライン化すると、「N フレーム前の完了を、フレームの先頭で待つ」という形になります。**待機を先頭に置く形は、そのまま発展させられます。**

---

## ✅ 本章のゴール:実行完了を確実に待てる

### Step 1:同期が機能していることを確認する

フェンスの値の変化を観察します。

```cpp
Core::Status DemonstrateFence()
{
    LOG_INFO(L"--- fence demonstration ---");

    HR_TRY(g_allocator->Reset());
    HR_TRY(g_commandList->Reset(g_allocator.Get(), nullptr));
    HR_TRY(g_commandList->Close());

    LOG_INFO(L"before submit : completed = {}",
             g_fence.Get()->GetCompletedValue());

    g_queue.Execute(g_commandList.Get());

    auto target = g_fence.Signal(g_queue.Get());
    if (!target) return std::unexpected(target.error());

    LOG_INFO(L"after signal  : completed = {}, target = {}",
             g_fence.Get()->GetCompletedValue(), *target);

    if (auto r = g_fence.Wait(*target); !r)
    {
        return r;
    }

    LOG_INFO(L"after wait    : completed = {}",
             g_fence.Get()->GetCompletedValue());
    return {};
}
```

**期待される出力**

```
[Info ] main.cpp(72): --- fence demonstration ---
[Info ] main.cpp(80): before submit : completed = 0
[Info ] main.cpp(88): after signal  : completed = 0, target = 1
[Info ] main.cpp(95): after wait    : completed = 1
```

**注目すべきは 3 行目です。** `Signal` を呼んだ直後なのに、`completed` はまだ `0` のままです。**シグナルは積まれただけで、実行されていません**(10.1.2)。

そして待機の後に `1` になります。これが「GPU が追いついた」ことの証拠です。

> **3 行目が `1` になることもあります**
>
> 空のコマンドリストなので、CPU がログを出力する間に GPU が終わっている可能性があります。環境によって結果が変わります。
>
> **どちらでも構いません。** 重要なのは「待った後は必ず到達している」ことであり、「待つ前は保証がない」ことです。この不確定性こそが、フェンスを必要とする理由そのものです。

### Step 2:タイムアウトの動作を確認する

**到達しない値を待って、タイムアウト処理が正しく動くことを確かめます。**

```cpp
void DemonstrateTimeout(ID3D12Device* device)
{
    LOG_INFO(L"--- timeout demonstration (this will take 1 second) ---");

    // 決してシグナルされない値を待つ
    const std::uint64_t impossible = g_fence.LastSignaledValue() + 1000;

    if (auto r = g_fence.Wait(impossible, 1000); !r)   // 1 秒でタイムアウト
    {
        HandleWaitFailure(device, r.error());
    }
}
```

**期待される出力**

```
[Info ] main.cpp(102): --- timeout demonstration (this will take 1 second) ---
[Error] Fence.cpp(96): fence wait timed out: waiting 1001 but completed 1
[Fatal] DeviceLost.cpp(18): fence wait failed but device is alive. Check fence value bookkeeping.
```

**最後の行が重要です。**

デバイスは生きています。GPU はハングしていません。**タイムアウトの原因は、こちらの論理エラーです。** 10.5.3 節で作った分岐が、正しくそう判断しています。

**確認したらこのコードは削除してください。**

### Step 3:フレームループを回す

10.6 節のループに差し替えて実行します。

- ウィンドウが表示され、応答する
- CPU 使用率が異常に高くならない(ポーリングしていない証拠)
- デバッグレイヤーの警告が出ない
- `×` で正常に終了する

**画面には何も出ません。それで正解です。** ただし前章と違い、**根拠のある正しさ**になりました。

### Step 4:終了時の待機を確認する

```cpp
//--- 終了処理 ---
LOG_INFO(L"waiting for GPU...");

if (auto r = Graphics::WaitForGpuIdle(g_queue.Get(), g_fence); !r)
{
    Core::ReportError(r.error());
}

LOG_INFO(L"GPU idle. shutting down.");

g_commandList.Reset();
g_allocator.Reset();
g_fence.Shutdown();
g_queue.Shutdown();
device.Shutdown();
```

```
[Info ] main.cpp(140): waiting for GPU...
[Info ] main.cpp(147): GPU idle. shutting down.
```

**この 2 行の間で、デバッグレイヤーの警告が出ないこと。** 出るようなら、待機が正しく機能していません。

---

### 本章の達成状態

- [ ] `Fence` クラスを実装し、イベントハンドルを RAII で管理した
- [ ] フェンスの初期値を 0、最初のシグナル値を 1 にした
- [ ] `queue->Signal()` の戻り値を確認している
- [ ] 待機前に `GetCompletedValue()` で早期リターンしている
- [ ] `WaitForSingleObject` に `INFINITE` を渡していない
- [ ] タイムアウト値が TDR(2 秒)より長い
- [ ] タイムアウト時に `GetDeviceRemovedReason()` で原因を切り分けている
- [ ] デバイスロスト時に Aftermath のダンプ生成を待っている
- [ ] `allocator->Reset()` の前に GPU の完了を待っている
- [ ] 終了時に `WaitForGpuIdle` を呼んでいる
- [ ] Step 2 のタイムアウト実験で「デバイスは生きている」と判定された

---

## トラブルシューティング

| 症状 | 原因 | 対処 |
|---|---|---|
| 待機が返ってこない | `INFINITE` を使っている | 有限のタイムアウトにする(10.5.1) |
| 待機が即座に通過する | 手動リセットのイベントを使っている | `CreateEventW` の第2引数を `FALSE` に(10.2.2) |
| CPU 使用率が 100% | ポーリングしている | イベントで待つ(10.2.1) |
| いつもタイムアウトする | シグナルしていない値を待っている | フェンス値の管理を見直す(10.5.3) |
| `GetCompletedValue()` が `UINT64_MAX` | デバイスロスト | `GetDeviceRemovedReason()` を確認(10.1.3) |
| 終了時に「使用中」の警告 | `WaitForGpuIdle` を呼んでいない | 10.4.2 を確認 |
| ダンプファイルが残らない | 生成完了を待たずに終了した | 第8章 8.5 節を確認(10.5.4) |
| ハンドルリーク | `CloseHandle` 忘れ | `Shutdown()` を確認(10.3.3) |
| フレームレートが上がらない | 毎フレーム全部待っている | **第12章で解決する** |

---

## まとめ

**1. フェンスは 64 ビットの目盛りにすぎない。**
複雑な機構ではありません。「GPU がどこまで進んだか」を表す数値が 1 つあるだけです。

**2. `Signal` は積まれる。**
呼んだ瞬間に値が変わるのではなく、キューに積まれた命令として、それ以前のコマンドリストの後に実行されます。**この順序性が、フェンスを完了マーカーたらしめています。**

**3. ポーリングせず、イベントで眠る。**
自動リセットのイベントを作り、`SetEventOnCompletion` で依頼し、`WaitForSingleObject` で待ちます。

**4. `INFINITE` を使わない。**
GPU がハングしたとき、アプリが永久に凍り、**Aftermath のダンプを書き出す機会が失われます。** タイムアウトは TDR の 2 秒より長く取ります。

**5. タイムアウトの原因は 2 つある。**
GPU のハングか、自分の値の管理ミスか。`GetDeviceRemovedReason()` が区別してくれます。この分岐がないと、自分のバグを何日も GPU のせいにします。

**6. 終了時は必ず待つ。**
D3D12 は GPU の使用状況を参照カウントで管理しません。待たずに破棄すると、「たまたま動く」状態になります。それが一番たちが悪いのです。

**7. `WaitForGpuIdle` は正しいが遅い。**
CPU と GPU が交互にしか働きません。第12章でパイプライン化します。

次章で、ついに画面に色が出ます。スワップチェーンを作り、バックバッファを取得し、リソースバリアを —— `d3dx12.h` のヘルパーなしで —— 手で書いて、青一色の画面を表示します。**本書で最初の「絵」です。**

---

## 参考リンク

| 内容 | URL |
|---|---|
| `ID3D12Fence` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/nn-d3d12-id3d12fence |
| `ID3D12Fence::SetEventOnCompletion` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/nf-d3d12-id3d12fence-seteventoncompletion |
| `ID3D12CommandQueue::Signal` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/nf-d3d12-id3d12commandqueue-signal |
| マルチエンジンの同期 | https://learn.microsoft.com/ja-jp/windows/win32/direct3d12/user-mode-heap-synchronization |
| TDR レジストリキー | https://learn.microsoft.com/ja-jp/windows-hardware/drivers/display/tdr-registry-keys |
| `WaitForSingleObject` | https://learn.microsoft.com/ja-jp/windows/win32/api/synchapi/nf-synchapi-waitforsingleobject |
