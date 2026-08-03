# 第12章 フレームループを正しく回す

前章で画面が青くなりました。しかし、そのループには 3 つの宿題が残っています。

| 宿題 | 出典 |
|---|---|
| 毎フレーム GPU を待っているので CPU と GPU が交互にしか働かない | 第10章 10.4.3 節 |
| ウィンドウをリサイズすると絵が崩れる | 第11章 11.1.3 節 |
| ウィンドウの枠をドラッグしている間、描画が止まる | 第5章 5.5.2 節 |

**本章で 3 つとも片付けます。**

そして、ここで完成させるループは**最後まで変わりません**。第37章でレイトレーシングを実装するときも、骨格はこのままです。中身が増えるだけです。だから、いま正しい形にしておく価値があります。

**本章のゴール**
フレームをパイプライン化し、CPU と GPU を並行して働かせる。ウィンドウのリサイズに対応し、ドラッグ中も描画を続ける。リサイズしても落ちず、絵が崩れない状態にする。

---

## 12.1 バッファ数をいくつにするか

### 12.1.1 2 つの「数」を区別する

紛らわしいのですが、**別々の概念が 2 つあります。**

| 用語 | 意味 |
|---|---|
| **バックバッファ数** | スワップチェーンが持つバッファの枚数 |
| **同時進行フレーム数**(frames in flight) | CPU が GPU より何フレーム先行できるか |

前者は DXGI の設定、後者はこちらの設計です。**本書ではこの 2 つを同じ値にします**が、理屈のうえでは別物です。

第11章のループでは、同時進行フレーム数は **1** でした。毎フレーム `WaitForGpuIdle` していたので、CPU は先行できません。

```
バッファ 2 枚 / 同時進行 1 フレーム(第11章)

CPU: [記録][投入][──── 待機 ────][記録][投入][──── 待機 ────]
GPU:              [── 実行 ──]                  [── 実行 ──]
         ↑                    ↑
      GPU が遊ぶ            CPU が遊ぶ
```

**バッファを増やしても、待ち方を変えなければ意味がありません。** 12.2 節で待ち方を変えて初めて、バッファ数が効いてきます。

### 12.1.2 レイテンシとスループットのトレードオフ

```
バッファ 3 枚 / 同時進行 3 フレーム

CPU: [記録 N][記録 N+1][記録 N+2][記録 N+3]
GPU:         [実行 N  ][実行 N+1][実行 N+2]
```

**両方が常に働いています。** これがパイプライン化です。

ただし、代償があります。

| バッファ数 | 利点 | 欠点 |
|---|---|---|
| 2(ダブル) | 遅延が小さい | 1 フレームでも時間がかかると即座にコマ落ちする |
| 3(トリプル) | たまに重いフレームがあっても吸収できる | **遅延が 1 フレーム分増える** |

垂直同期を有効にしている場合、この差は顕著です。60Hz でバッファが 2 枚のとき、あるフレームが 16.6ms をわずかに超えただけで、**次の垂直同期を逃して 30fps に落ちます。** 3 枚あれば、その 1 フレームを吸収できます。

一方、遅延はバッファが増えるほど大きくなります。マウスを動かしてから画面に反映されるまでのフレーム数が増えるからです。競技性の高いゲームや VR では、これが問題になります。

> **NVIDIA Reflex について**
>
> この遅延とスループットのトレードオフを、より賢く制御する仕組みが NVIDIA Reflex です。「CPU が走りすぎないように、必要な時刻まで待たせる」という発想で、バッファを増やしつつ遅延を抑えます。
>
> NVAPI が必要なので本書では扱いません(第2章 2.5 節)。入り口は付録 H にまとめます。

### 12.1.3 本書の選択

```cpp
inline constexpr UINT kBackBufferCount = 3;   // 第11章の 2 から変更
```

**3 枚にします。**

- メモリ増加は 1080p の `R8G8B8A8` で約 8MB。無視できる量です
- 学習中は 1 フレームだけ極端に重くなることが頻繁にあります(シェーダーの再コンパイル、リソースの初回アクセスなど)。それを吸収できたほうが観察しやすくなります
- 遅延が問題になる用途は、本書の範囲外です

第11章の `kBackBufferCount` を `3` に変更してください。**スワップチェーン生成時の `BufferCount` と、次節のフレームリソースの数が、これで一致します。**

---

## 12.2 フレームごとのリソース

### 12.2.1 何をフレームごとに持つか

**判断基準は単純です。「GPU がまだ読んでいる可能性があるものは、フレームごとに持つ」。**

| リソース | フレームごとに必要か | 理由 |
|---|---|---|
| **コマンドアロケータ** | **必要** | GPU が記録内容を読んでいる(第9章 9.2.1 節) |
| コマンドリスト | 不要 | メモリを持たない。投入直後に `Reset` できる |
| コマンドキュー | 不要 | 状態を持たない |
| フェンス | 不要 | 1 つのカウンタで全フレームを表現できる |
| **フェンス値** | **必要** | 「このスロットは、どの値まで進めば空くか」 |
| 定数バッファ | 必要(第18章) | GPU が読んでいる |
| バックバッファ | (DXGI が管理) | |

**本章で必要なのは、アロケータとフェンス値の 2 つだけです。** 定数バッファやアップロードバッファは、第18章と第21章で同じ枠組みに加わります。

### 12.2.2 `FrameResource` 構造体

```cpp
// src/Graphics/Renderer.h
namespace Graphics
{
    struct FrameResource
    {
        Microsoft::WRL::ComPtr<ID3D12CommandAllocator> allocator;

        // このスロットで最後に投入した作業のフェンス値。
        // 0 は「まだ一度も投入していない」を意味する。
        std::uint64_t fenceValue = 0;
    };
}
```

**コマンドリストは 1 本だけ**にします。理由は第9章 9.2.1 節の通りです。コマンドリストは記録の道具にすぎず、`ExecuteCommandLists` の直後に別のアロケータで `Reset` できます。

```cpp
FrameResource                     m_frames[kBackBufferCount];  // 3 セット
ComPtr<ID3D12GraphicsCommandList> m_commandList;               // 1 本
```

**この非対称が、第9章で説明した「ノートとペン」の比喩そのものです。** ノートは 3 冊必要で、ペンは 1 本で足ります。

### 12.2.3 フェンス値 `0` が効いてくる

```cpp
std::uint64_t fenceValue = 0;
```

第10章 10.3.2 節で、**最初のシグナル値を 1 にする**と決めました。その理由が、ここで回収されます。

起動直後、すべてのスロットの `fenceValue` は `0` です。1 周目のループでは、まだ何も投入されていないので、待つ必要がありません。

```cpp
fence.Wait(frame.fenceValue);   // Wait(0)
```

フェンスの現在値は `0` 以上なので、`GetCompletedValue() >= 0` は**常に真**です。第10章 10.2.4 節で入れた早期リターンにより、**この呼び出しは即座に返ります。**

**特別扱いのコードが 1 行も要りません。** もしフェンス値を 0 から使い始めていたら、「初回かどうか」を判定する分岐が必要になっていました。

### 12.2.4 バックバッファ番号をスロット番号として使う

```cpp
const UINT index = m_swapChain.CurrentIndex();   // GetCurrentBackBufferIndex()
FrameResource& frame = m_frames[index];
```

**バックバッファの番号を、そのままフレームリソースの添字として使います。**

これが安全な理由は 2 つあります。

**1. スワップチェーンが順番を保証している**
DXGI は、バッファ *i* が表示中または表示待ちの間、そのバッファを描画用に返しません。だから、同じ番号が返ってきたということは、そのバッファが解放されたということです。

**2. どちらにせよフェンスで待っている**
仮に DXGI が予想外の順序を返しても、そのスロットの `fenceValue` を待つので安全です。**番号の並びに依存した実装にはなっていません。**

第11章 11.1.2 節で「番号を自分で計算してはいけない」と書きました。**その原則を守っていれば、この設計は自動的に安全になります。**

---

## 12.3 レンダリングループの完成形

### 12.3.1 待機を「必要な分だけ」にする

**第11章との違いは、たった 1 行です。**

```cpp
// 第11章:投入済みの全作業を待つ
WaitForGpuIdle(queue, fence);

// 第12章:これから使うスロットが空くのだけを待つ
fence.Wait(m_frames[index].fenceValue);
```

**これだけでパイプライン化が完成します。**

考え方はこうです。3 スロットあるので、CPU がスロット 0 を使おうとするとき、GPU はスロット 1 か 2 を処理しているはずです。スロット 0 の**前回**の作業さえ終わっていれば、そのアロケータを `Reset` できます。**GPU が今何をしているかは、関係ありません。**

```
             スロット 0    スロット 1    スロット 2
フレーム 0:  CPU 記録
フレーム 1:  GPU 実行      CPU 記録
フレーム 2:                GPU 実行      CPU 記録
フレーム 3:  CPU 記録  ←── ここでスロット 0 の
             (待つのは     フレーム 0 の完了だけを待てばよい
              これだけ)
```

### 12.3.2 実装

第9章から第11章まで、グローバル変数で書いてきました。状態が増えてきたので、ここで `Renderer` クラスにまとめます。

```cpp
// src/Graphics/Renderer.h
#pragma once
#include "std_import.h"
#include "Core/Error.h"
#include "Graphics/CommandQueue.h"
#include "Graphics/Fence.h"
#include "Graphics/SwapChain.h"

namespace Graphics
{
    struct FrameResource
    {
        Microsoft::WRL::ComPtr<ID3D12CommandAllocator> allocator;
        std::uint64_t fenceValue = 0;
    };

    class Renderer
    {
    public:
        Core::Status Initialize(ID3D12Device* device,
                                IDXGIFactory6* factory,
                                HWND hwnd, UINT width, UINT height);
        void Shutdown();

        Core::Status RenderFrame();
        Core::Status Resize(UINT width, UINT height);

        std::uint32_t FrameCount() const noexcept { return m_frameCount; }

    private:
        Core::Status WaitForGpuIdle();

        ID3D12Device* m_device = nullptr;   // 所有しない

        CommandQueue m_queue;
        Fence        m_fence;
        SwapChain    m_swapChain;

        FrameResource m_frames[kBackBufferCount];
        Microsoft::WRL::ComPtr<ID3D12GraphicsCommandList> m_commandList;

        UINT          m_width  = 0;
        UINT          m_height = 0;
        std::uint32_t m_frameCount = 0;
        bool          m_occluded   = false;
    };
}
```

初期化のうち、本章で新しいのはフレームリソースの部分だけです。

```cpp
for (UINT i = 0; i < kBackBufferCount; ++i)
{
    HR_TRY(device->CreateCommandAllocator(
        D3D12_COMMAND_LIST_TYPE_DIRECT,
        IID_PPV_ARGS(&m_frames[i].allocator)));

    Core::SetDebugNameF(m_frames[i].allocator.Get(),
                        L"FrameAllocator[{}]", i);

    m_frames[i].fenceValue = 0;      // まだ何も投入していない
}
```

**名前に添字を入れる**ことを忘れないでください(第6章 6.5.3 節)。デバッグレイヤーが「`FrameAllocator[1]` が使用中に Reset された」と言ってくれるのと、`<unnamed>` と言うのでは、調査の手間がまるで違います。

そして本体です。

```cpp
Core::Status Renderer::RenderFrame()
{
    //--- ① 今回使うスロットを決める(12.2.4) ---
    const UINT index = m_swapChain.CurrentIndex();
    FrameResource& frame = m_frames[index];

    //--- ② このスロットの前回の作業だけを待つ(12.3.1) ---
    if (auto r = m_fence.Wait(frame.fenceValue); !r)
    {
        return r;
    }

    //--- ③ 記録の準備 ---
    HR_TRY(frame.allocator->Reset());
    HR_TRY(m_commandList->Reset(frame.allocator.Get(), nullptr));

    //--- ④ PRESENT → RENDER_TARGET ---
    ID3D12Resource* backBuffer = m_swapChain.CurrentBackBuffer();
    const D3D12_CPU_DESCRIPTOR_HANDLE rtv = m_swapChain.CurrentRtv();
    {
        const auto barrier = MakeTransitionBarrier(
            backBuffer,
            D3D12_RESOURCE_STATE_PRESENT,
            D3D12_RESOURCE_STATE_RENDER_TARGET);
        m_commandList->ResourceBarrier(1, &barrier);
    }

    //--- ⑤ 描画 ---
    m_commandList->OMSetRenderTargets(1, &rtv, FALSE, nullptr);

    float clearColor[4]{};
    GetAnimatedColor(clearColor);          // 第11章 11.8 節
    m_commandList->ClearRenderTargetView(rtv, clearColor, 0, nullptr);

    // 第13章から、ここに描画コマンドが入る

    //--- ⑥ RENDER_TARGET → PRESENT ---
    {
        const auto barrier = MakeTransitionBarrier(
            backBuffer,
            D3D12_RESOURCE_STATE_RENDER_TARGET,
            D3D12_RESOURCE_STATE_PRESENT);
        m_commandList->ResourceBarrier(1, &barrier);
    }

    //--- ⑦ 投入 ---
    HR_TRY(m_commandList->Close());
    m_queue.Execute(m_commandList.Get());

    //--- ⑧ 表示 ---
    if (auto r = m_swapChain.Present(m_device, 1); !r)
    {
        return r;
    }

    //--- ⑨ このスロットの完了を表すフェンス値を記録 ---
    auto value = m_fence.Signal(m_queue.Get());
    if (!value)
    {
        return std::unexpected(value.error());
    }
    frame.fenceValue = *value;

    ++m_frameCount;
    return {};
}
```

**⑨ が本章の核心です。** シグナルした値を、そのスロットに覚えさせています。次に同じスロットが回ってきたとき、②でこの値を待ちます。

### 12.3.3 パイプラインが効いていることを確認する

**フレームレートをタイトルバーに表示します。** これが最も直接的な確認方法です。

```cpp
namespace
{
    void UpdateFpsTitle(HWND hwnd, std::uint32_t& frames,
                        std::chrono::steady_clock::time_point& lastUpdate)
    {
        ++frames;

        const auto now = std::chrono::steady_clock::now();
        const auto elapsed =
            std::chrono::duration<double>(now - lastUpdate).count();

        if (elapsed < 0.5)
        {
            return;
        }

        const double fps = frames / elapsed;
        const std::wstring title = std::format(
            L"D3D12Book - Chapter 12   [{:.1f} fps  {:.2f} ms]",
            fps, 1000.0 / fps);

        ::SetWindowTextW(hwnd, title.c_str());

        frames = 0;
        lastUpdate = now;
    }
}
```

垂直同期を有効にしているので、**ディスプレイのリフレッシュレートに一致した値**が出るはずです。60Hz なら約 60fps、144Hz なら約 144fps です。

**ここで重要な確認があります。** `Present(1, 0)` の第 1 引数を `0` にして、垂直同期を切ってみてください。

```cpp
m_swapChain.Present(m_device, 0);   // 一時的に vsync オフ
```

- **第11章のループ**(全体待ち)では、数百〜数千 fps 程度
- **第12章のループ**(パイプライン化)では、さらに高い値

空のシーンなので差が小さく見えることもありますが、**両者の差はフレームあたりの同期コストそのものです。** 第13章以降で実際の描画が入ると、この差は大きく開きます。

**確認したら `Present(m_device, 1)` に戻してください。** 垂直同期を切ったまま進めると、GPU が無駄に全力で回り続けます。

---

## 12.4 ウィンドウリサイズと `ResizeBuffers`

### 12.4.1 手順

**順序が厳密です。1 つでも飛ばすと失敗します。**

```
① GPU の全作業が終わるのを待つ
       ↓
② バックバッファへの参照をすべて解放する
       ↓
③ ResizeBuffers を呼ぶ
       ↓
④ バックバッファを取り直し、RTV を作り直す
```

### 12.4.2 参照をすべて解放する

**`ResizeBuffers` が失敗する原因の 9 割がこれです。**

```cpp
for (auto& buffer : m_backBuffers)
{
    buffer.Reset();      // ComPtr を空にする = Release する
}
```

第11章 11.4 節で `GetBuffer` から受け取った `ComPtr` が、参照を保持しています。**1 つでも残っていると `ResizeBuffers` は `DXGI_ERROR_INVALID_CALL` を返します。**

見落としやすい参照:

- `ComPtr<ID3D12Resource> m_backBuffers[N]` — これは明らか
- ローカル変数に残っている一時的な `ComPtr`
- **コマンドリストが参照している可能性** → だから ① で GPU を待つ

なお、**RTV(デスクリプタ)はリソースへの参照を保持しません。** デスクリプタは「解釈の指示書」にすぎないからです(第11章 11.2.1 節)。ただしリサイズ後は指す先が無効になるので、作り直す必要があります。

### 12.4.3 `Flags` を引き継ぐ

```cpp
DXGI_SWAP_CHAIN_DESC1 desc{};
HR_TRY(m_swapChain->GetDesc1(&desc));

HR_TRY(m_swapChain->ResizeBuffers(
    kBackBufferCount,
    width, height,
    desc.Format,      // DXGI_FORMAT_UNKNOWN なら現状維持
    desc.Flags));     // ← ここを 0 にしてはいけない
```

**`Flags` に `0` を渡すと、`DXGI_SWAP_CHAIN_FLAG_ALLOW_TEARING` が消えます。**

第11章 11.1.6 節で設定したティアリング許可フラグが、リサイズのたびに失われることになります。症状は「起動直後は G-SYNC が効くのに、ウィンドウをリサイズすると効かなくなる」という、極めて追いにくいものです。

**現在の記述子を `GetDesc1` で取得し、そこから引き継ぐのが確実です。**

### 12.4.4 `WndProc` から直接呼ばない

**設計上、ここが重要です。**

第5章で `OnResize` コールバックを用意しました。素直に考えると、そこで直接リサイズしたくなります。

```cpp
// △ 動くが、望ましくない
window.OnResize = [&](int w, int h)
{
    renderer.Resize(w, h);    // WndProc の中で GPU を待つことになる
};
```

このコールバックが呼ばれる場所を追ってみてください。

```
メインループ
  └─ window.ProcessMessages()
       └─ DispatchMessageW()
            └─ WndProc()
                 └─ OnResize()
                      └─ renderer.Resize()
                           └─ WaitForGpuIdle()   ← ここで数ミリ秒止まる
```

**メッセージ処理の途中で、GPU の完了を待つことになります。** OS のメッセージ処理を長時間ブロックするのは避けるべきですし、リサイズ処理の中でさらにメッセージが飛んでくると、再入の危険もあります。

**フラグを立てるだけにして、ループの先頭で処理します。**

```cpp
std::optional<std::pair<int, int>> g_pendingResize;

window.OnResize = [](int w, int h)
{
    g_pendingResize = { w, h };     // 記録するだけ
};
```

```cpp
while (window.ProcessMessages())
{
    if (g_pendingResize)
    {
        const auto [w, h] = *g_pendingResize;
        g_pendingResize.reset();

        if (auto r = renderer.Resize(w, h); !r)
        {
            Core::ReportError(r.error());
            break;
        }
    }

    if (window.IsMinimized()) continue;

    if (auto r = renderer.RenderFrame(); !r) { /* ... */ }
}
```

**この形なら、リサイズは常に「フレームとフレームの間」で起こります。** 描画の途中に割り込むことがありません。

### 12.4.5 実装

```cpp
Core::Status SwapChain::Resize(ID3D12Device* device,
                               ID3D12CommandQueue* queue,
                               Fence& fence,
                               UINT width, UINT height)
{
    //--- 0 サイズは無視(最小化。第5章 5.5.2 節) ---
    if (width == 0 || height == 0)
    {
        return {};
    }

    //--- ① GPU の完了を待つ ---
    if (auto r = WaitForGpuIdle(queue, fence); !r)
    {
        return r;
    }

    //--- ② 参照をすべて解放する(12.4.2) ---
    for (auto& buffer : m_backBuffers)
    {
        buffer.Reset();
    }

    //--- ③ リサイズ。Flags を引き継ぐ(12.4.3) ---
    DXGI_SWAP_CHAIN_DESC1 desc{};
    HR_TRY(m_swapChain->GetDesc1(&desc));

    HR_TRY(m_swapChain->ResizeBuffers(
        kBackBufferCount, width, height, desc.Format, desc.Flags));

    //--- ④ RTV を作り直す ---
    if (auto r = CreateRenderTargetViews(device); !r)
    {
        return r;
    }

    LOG_INFO(L"swap chain resized: {} x {}", width, height);
    return {};
}
```

```cpp
Core::Status Renderer::Resize(UINT width, UINT height)
{
    if (width == 0 || height == 0) return {};
    if (width == m_width && height == m_height) return {};   // 変化なし

    if (auto r = m_swapChain.Resize(
            m_device, m_queue.Get(), m_fence, width, height); !r)
    {
        return r;
    }

    m_width  = width;
    m_height = height;

    // WaitForGpuIdle 済みなので、全スロットの作業は完了している。
    // 記録済みのフェンス値はすべて到達済みなので、そのままでよい。

    // 第19章で深度バッファ、第26章でオフスクリーンターゲットを
    // 追加したら、ここで一緒に作り直す。

    return {};
}
```

**最後のコメントが重要です。** 第19章で深度バッファを、第26章でポストエフェクト用のレンダーターゲットを追加します。**それらはすべて、この関数の中で作り直す必要があります。** 忘れると、リサイズ後にサイズの食い違いでクラッシュします。

---

## 12.5 ドラッグ中も描画する

### 12.5.1 モーダルループの問題

第5章 5.5.2 節で予告した問題です。

ウィンドウの枠をドラッグしている間、Windows は `DefWindowProcW` の内部で**独自のメッセージループ**を回します。その間、我々の

```cpp
while (window.ProcessMessages()) { ... }
```

は**一度も実行されません。** `DispatchMessageW` から制御が戻ってこないからです。

結果として、ドラッグ中は画面が更新されません。前のフレームが残り続けるか、環境によっては真っ黒になります。

### 12.5.2 `WM_TIMER` で描画する

**Windows のモーダルループも、タイマーメッセージは配送します。** これを利用します。

```cpp
// Window.h に追加
std::function<void()> OnIdleRender;    // モーダルループ中の描画
```

```cpp
// Window.cpp
namespace
{
    constexpr UINT_PTR kResizeTimerId = 1;
}

LRESULT Window::WndProc(HWND hwnd, UINT msg, WPARAM wParam, LPARAM lParam)
{
    switch (msg)
    {
    case WM_ENTERSIZEMOVE:
        m_resizing = true;
        // モーダルループ中も動くタイマーを仕掛ける
        ::SetTimer(hwnd, kResizeTimerId, USER_TIMER_MINIMUM, nullptr);
        return 0;

    case WM_EXITSIZEMOVE:
        ::KillTimer(hwnd, kResizeTimerId);
        m_resizing = false;
        NotifyResize();
        return 0;

    case WM_TIMER:
        if (wParam == kResizeTimerId && OnIdleRender)
        {
            OnIdleRender();
        }
        return 0;

    // ... 以下、第5章のまま ...
    }
    return ::DefWindowProcW(hwnd, msg, wParam, lParam);
}
```

`USER_TIMER_MINIMUM` は Windows が定める最小のタイマー間隔(10 ミリ秒)です。**ドラッグ中は最大 100fps 程度で描画が続きます。**

`main.cpp` 側で接続します。

```cpp
window.OnIdleRender = [&]()
{
    if (window.IsMinimized()) return;
    if (auto r = renderer.RenderFrame(); !r)
    {
        Core::ReportError(r.error());
    }
};
```

**これでウィンドウが「応答なし」にならず、ドラッグ中も色の変化が続きます。**

> **再入に注意**
>
> `OnIdleRender` は `WndProc` の中から呼ばれます。つまり `RenderFrame` が `WndProc` の内側で実行されます。
>
> 本書の `RenderFrame` はメッセージを処理しないので問題ありませんが、**将来ここで何かを待つコードを足す場合は、再入の可能性を意識してください。**

### 12.5.3 残る見た目の問題

**ドラッグ中の見た目は、完全には正しくなりません。**

第5章 5.5.2 節の設計により、ドラッグ中は `WM_SIZE` が来ても `NotifyResize()` を呼びません。したがってバックバッファのサイズは古いままです。

第11章で `DXGI_SCALING_NONE` を選んだので、**ウィンドウを大きくすると、バッファが届かない領域が現れます。**

**選択肢は 3 つあります。**

| 方針 | 利点 | 欠点 |
|---|---|---|
| **A. このまま**(本書の既定) | リサイズ処理が 1 回だけ。実装が単純 | ドラッグ中に未描画領域が見える |
| B. `DXGI_SCALING_STRETCH` にする | ドラッグ中も全面が埋まる | 引き伸ばしでぼやける |
| C. ドラッグ中も毎回リサイズする | 見た目が常に正しい | リサイズ処理が毎回走る |

**本書は A を採ります。** ドラッグ中の一時的な見た目より、リサイズ処理の回数を抑えることを優先します。

理由は先を見据えてのことです。第19章で深度バッファ、第26章でポストエフェクト用のレンダーターゲット、第27章でシャドウマップが加わると、**1 回のリサイズで作り直すリソースが 10 個近くになります。** それをドラッグ中に毎秒 100 回行うのは現実的ではありません。

B は 1 行の変更で試せます。気になる場合は `DXGI_SCALING_STRETCH` にしてみてください。**どれが正解ということはなく、作るものによって選ぶ性質の判断です。**

---

## 12.6 `Present` の例外的な戻り値

第11章 11.6.3 節で `Present` の戻り値を確認するようにしました。ループが本格的に回り始めたので、**残りのケースも扱います。**

### 12.6.1 `DXGI_STATUS_OCCLUDED`

ウィンドウが他のウィンドウに完全に隠されると、`Present` は `DXGI_STATUS_OCCLUDED` を返します。**これは成功コードです。**

隠れている間も全力で描画し続けるのは、電力の無駄です。**描画を止めて、状態が変わるのを待ちます。**

```cpp
Core::Status SwapChain::Present(ID3D12Device* device, UINT syncInterval)
{
    const UINT flags =
        (syncInterval == 0 && m_allowTearing) ? DXGI_PRESENT_ALLOW_TEARING : 0;

    const HRESULT hr = m_swapChain->Present(syncInterval, flags);

    //--- デバイスロスト(第11章 11.6.3 節) ---
    if (hr == DXGI_ERROR_DEVICE_REMOVED || hr == DXGI_ERROR_DEVICE_RESET)
    {
        OnDeviceLost(device->GetDeviceRemovedReason());
        return std::unexpected(Core::MakeError(hr, L"Present: device lost"));
    }

    //--- 隠れている ---
    m_occluded = (hr == DXGI_STATUS_OCCLUDED);

    HR_TRY(hr);
    return {};
}
```

ループ側では、隠れている間は描画を控えます。

```cpp
if (renderer.IsOccluded())
{
    // 描画せず、状態が変わったかだけ確認する
    if (!renderer.TestPresent())
    {
        ::Sleep(50);
        continue;
    }
}
```

`DXGI_PRESENT_TEST` フラグを使うと、**実際には表示せずに「今 `Present` したらどうなるか」だけを確認できます。**

```cpp
bool SwapChain::TestPresent()
{
    const HRESULT hr = m_swapChain->Present(0, DXGI_PRESENT_TEST);
    if (hr == DXGI_STATUS_OCCLUDED)
    {
        return false;      // まだ隠れている
    }
    m_occluded = false;
    return true;
}
```

> **なぜ `Sleep` を挟むのか**
>
> 挟まないと、隠れている間ずっとタイトなループが回り、CPU を 1 コア使い切ります。第10章 10.2.1 節でポーリングを禁じたのと同じ理屈です。
>
> 50 ミリ秒であれば、ウィンドウが再び見えるようになったときの反応も十分速く感じられます。

### 12.6.2 戻り値の一覧

| 戻り値 | 意味 | 対応 |
|---|---|---|
| `S_OK` | 成功 | — |
| `DXGI_STATUS_OCCLUDED` | 隠れている(**成功コード**) | 描画を止めて待つ |
| `DXGI_ERROR_DEVICE_REMOVED` | デバイスロスト | 第8章・第10章の処理へ |
| `DXGI_ERROR_DEVICE_RESET` | 同上 | 同上 |
| `DXGI_ERROR_INVALID_CALL` | 引数の誤り | ティアリングフラグの整合を確認 |

---

## ✅ 本章のゴール:リサイズしても落ちない・崩れない

### Step 1:フレームレートを確認する

実行して、タイトルバーを見てください。

```
D3D12Book - Chapter 12   [60.0 fps  16.67 ms]
```

ディスプレイのリフレッシュレートと一致していれば、垂直同期とパイプラインが機能しています。

### Step 2:リサイズする

ウィンドウの端をドラッグしてサイズを変えます。

**確認すること:**

- [ ] ウィンドウが「応答なし」にならない
- [ ] **ドラッグ中も色が変化し続ける**(12.5 節の成果)
- [ ] マウスを離した瞬間に、バッファのサイズが追従する
- [ ] デバッグレイヤーが何も言わない
- [ ] 落ちない

```
[Info ] SwapChain.cpp(148): swap chain resized: 1024 x 600
```

ドラッグ中に未描画の領域が見えるのは、12.5.3 節で説明した既知の挙動です。**離した瞬間に正しくなれば成功です。**

### Step 3:最小化と復帰

最小化ボタンを押し、タスクバーから戻します。

- [ ] 最小化中にログが出ない(0×0 が通知されない)
- [ ] 復帰時に正しいサイズで描画される
- [ ] 落ちない

**第5章 5.5.2 節で入れた `m_minimized` フラグが、ここで効いています。** これがないと `ResizeBuffers(3, 0, 0, ...)` が呼ばれ、失敗します。

### Step 4:最大化・復元

最大化ボタンとダブルクリックを試します。**`WM_ENTERSIZEMOVE` が来ない経路**なので、`WM_SIZE` 側の通知が働いているかの確認になります(第5章 5.5.2 節)。

### Step 5:隠してみる

他のウィンドウを最大化して、完全に隠します。

- [ ] CPU 使用率が下がる(タスクマネージャで確認)
- [ ] 再び見えるようにすると、描画が再開する

### Step 6:リサイズの規則を破る

**`ResizeBuffers` の前提が本当に検証されているかを確かめます。**

12.4.5 節の ② —— バックバッファの解放 —— をコメントアウトしてください。

```cpp
//for (auto& buffer : m_backBuffers)
//{
//    buffer.Reset();
//}
```

リサイズすると失敗します。

```
[Error] SwapChain.cpp(140): DXGI_ERROR_INVALID_CALL (0x887A0001) ...
    -> m_swapChain->ResizeBuffers(kBackBufferCount, width, height, desc.Format, desc.Flags)
```

**第6章で作ったエラー機構が、失敗した式そのものを表示しています。** どこで何が起きたかを探す必要がありません。

**確認したら元に戻してください。**

### Step 7:GPU がパイプライン化されていることを確認する(任意)

Nsight Graphics(第2章 2.4.1 節)でフレームをキャプチャすると、CPU の記録と GPU の実行が重なっている様子が見られます。本格的な解析は第29章で行いますが、**「本当に並行しているのか」を目で確かめたい場合はここで試せます。**

---

### 本章の達成状態

- [ ] `kBackBufferCount` を 3 に変更した
- [ ] フレームごとにコマンドアロケータを持たせた
- [ ] コマンドリストは 1 本のまま
- [ ] 各スロットにフェンス値を記録している
- [ ] 待機を `WaitForGpuIdle` から `Wait(frame.fenceValue)` に変えた
- [ ] アロケータに `FrameAllocator[n]` という名前を付けた
- [ ] `ResizeBuffers` の前に GPU を待ち、参照を解放している
- [ ] `ResizeBuffers` に `Flags` を引き継いでいる
- [ ] リサイズを `WndProc` から直接呼ばず、ループの先頭で処理している
- [ ] `WM_TIMER` によりドラッグ中も描画される
- [ ] `DXGI_STATUS_OCCLUDED` を成功コードとして扱っている
- [ ] リサイズ・最小化・最大化のいずれでも落ちない

---

## トラブルシューティング

| 症状 | 原因 | 対処 |
|---|---|---|
| `ResizeBuffers` が `DXGI_ERROR_INVALID_CALL` | バックバッファの参照が残っている | 12.4.2 |
| 同上 | GPU がまだ使用中 | 先に `WaitForGpuIdle`(12.4.1) |
| リサイズ後に G-SYNC が効かない | `Flags` に 0 を渡した | `GetDesc1` から引き継ぐ(12.4.3) |
| 最小化でクラッシュ | 0×0 でリサイズしている | 早期リターン(12.4.5) |
| ドラッグ中に固まる | `WM_TIMER` を処理していない | 12.5.2 |
| ドラッグ中に絵が欠ける | `DXGI_SCALING_NONE` の既知の挙動 | 12.5.3(仕様) |
| アロケータ Reset の警告 | フェンス値の記録漏れ | ⑨ を確認(12.3.2) |
| フレームレートが上がらない | まだ `WaitForGpuIdle` を呼んでいる | 12.3.1 |
| 隠すと CPU 使用率が高い | `DXGI_STATUS_OCCLUDED` 未処理 | 12.6.1 |
| リサイズ後に深度バッファのサイズが合わない | **第19章で対応する** | `Resize` に追加する |

---

## まとめ

**1. バッファ数と同時進行フレーム数は別の概念。**
バッファを増やしても、待ち方を変えなければパイプライン化されません。本書は両方を 3 にしました。

**2. 「GPU が読んでいる可能性があるもの」だけをフレームごとに持つ。**
コマンドアロケータは必要、コマンドリストは不要。第9章の「ノートとペン」の比喩がそのまま設計になります。

**3. 待機を「スロット単位」にすることでパイプラインが完成する。**
`WaitForGpuIdle` を `Wait(frame.fenceValue)` に変えるだけです。**変更は 1 行ですが、意味はまったく違います。**

**4. フェンス値を 1 から始めた判断が、ここで回収された。**
初期値 0 が「まだ何も投入していない」を自然に表現するので、初回の特別扱いが不要になりました。

**5. `ResizeBuffers` は前提条件が厳しい。**
GPU の完了を待ち、すべての参照を解放し、`Flags` を引き継ぐ。1 つでも欠けると失敗します。

**6. リサイズを `WndProc` から直接呼ばない。**
フラグを立て、フレームの切れ目で処理します。メッセージ処理を長時間ブロックせず、再入も避けられます。

**7. これがフレームループの完成形。**
第37章まで、この骨格は変わりません。増えるのは ⑤ の中身と、`Resize` で作り直すリソースだけです。

次章から、いよいよ「絵を描く」段階に入ります。まず HLSL のシェーダーを書き、DXC でコンパイルします。**第31章で Aftermath にクラッシュ位置を HLSL の行番号で教えてもらうための伏線が、次章に登場します。**

---

## 参考リンク

| 内容 | URL |
|---|---|
| `IDXGISwapChain::ResizeBuffers` | https://learn.microsoft.com/ja-jp/windows/win32/api/dxgi/nf-dxgi-idxgiswapchain-resizebuffers |
| `IDXGISwapChain::Present` | https://learn.microsoft.com/ja-jp/windows/win32/api/dxgi/nf-dxgi-idxgiswapchain-present |
| DXGI のベストプラクティス | https://learn.microsoft.com/ja-jp/windows/win32/direct3ddxgi/d3d10-graphics-programming-guide-dxgi |
| `SetTimer` | https://learn.microsoft.com/ja-jp/windows/win32/api/winuser/nf-winuser-settimer |
| フレームのレイテンシ | https://learn.microsoft.com/ja-jp/windows/win32/direct3ddxgi/dxgi-flip-model#avoiding-detecting-and-recovering-from-latency |
