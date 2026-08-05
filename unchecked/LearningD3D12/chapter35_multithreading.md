# 第35章 マルチスレッド化

第12章でフレームをパイプライン化し、CPU と GPU を並行して働かせました。

```
CPU: [記録 N][記録 N+1][記録 N+2]
GPU:         [実行 N  ][実行 N+1]
```

**しかし、CPU 側は 1 スレッドのままです。**

第34章で GPU カリングを実装したので、CPU の負荷は大きく減りました。**それでも、コマンドの記録は逐次的に行われています。**

**本章では、これを並列化します。**

そして、もう 1 つの並列化にも取り組みます。**キューを分けることで、GPU 内部でも処理を重ねます。**

```
描画キュー:  [シャドウ][不透明][ポスト]
コピーキュー:      [テクスチャ転送]        ← 並行して実行
```

**本章のゴール**
コマンドリストを複数スレッドで並列記録する。バンドルで記録コストを削減する。コピーキューで非同期転送を実装する。

---

## 35.1 何がスレッドセーフか

### 35.1.1 D3D12 の設計

**第9章 9.2.3 節で、規則の 3 番目としてこう書きました。**

> **規則 3:マルチスレッドで記録するなら、スレッドごとにアロケータとリストを分ける**
> コマンドリストもアロケータも、**スレッドセーフではありません。** 第35章で並列記録を扱いますが、そこでの原則は「共有しない」です。

**改めて整理します。**

| オブジェクト | スレッドセーフか |
|---|---|
| **`ID3D12Device`** | **○**(生成は並列に呼べる) |
| **`ID3D12CommandQueue`** | **○**(`ExecuteCommandLists` は並列に呼べる) |
| `ID3D12CommandAllocator` | **×** |
| `ID3D12GraphicsCommandList` | **×** |
| `ID3D12Resource::Map` | ○(同じ範囲への書き込みは自分で管理) |
| `ID3D12Fence` | ○ |

**「デバイスとキューは安全、記録に使うものは危険」**と覚えてください。

**この設計は意図的です。** D3D11 ではドライバが内部でロックを取っていましたが、それが並列化の障害になっていました。**D3D12 はロックを廃止し、代わりに「共有しない」ことをアプリに要求します。**

### 35.1.2 何を分ければよいか

```
スレッド 0:  Allocator[0] + CommandList[0]
スレッド 1:  Allocator[1] + CommandList[1]
スレッド 2:  Allocator[2] + CommandList[2]
スレッド 3:  Allocator[3] + CommandList[3]
```

**そして、フレームごとにも分ける必要があります**(第12章 12.2.1 節)。

```
必要な数 = スレッド数 × フレーム数
```

**4 スレッド × 3 フレーム = 12 組**になります。

**コマンドリストも 12 本必要でしょうか。**

**第12章 12.2.2 節では、こう書きました。**

> **コマンドリストは 1 本だけ**にします。理由は第9章 9.2.1 節の通りです。コマンドリストは記録の道具にすぎず、`ExecuteCommandLists` の直後に別のアロケータで `Reset` できます。

**この論理は、スレッドをまたぐと成立しません。**

**同時に記録するので、スレッド数だけ必要です。** ただしフレームごとには不要なので、**4 本**で足ります。

```
アロケータ:      4 スレッド × 3 フレーム = 12 個
コマンドリスト:  4 スレッド              =  4 本
```

---

## 35.2 並列記録の設計

### 35.2.1 分割の方針

**何を並列化するかには、いくつかの方針があります。**

| 方針 | 説明 |
|---|---|
| **A. パス単位** | シャドウ、不透明、半透明を別スレッドで |
| **B. オブジェクト単位** | 描画対象を N 分割 |
| **C. 混合** | 重いパスだけをさらに分割 |

**方針 A は単純ですが、負荷が偏ります。**

```
スレッド 0:  シャドウパス(48 ドロー)
スレッド 1:  不透明パス(312 ドロー)   ← 6 倍以上
スレッド 2:  半透明パス(24 ドロー)
```

**方針 B は負荷が均等になりますが、パスの境界を扱いにくくなります。**

**本書は C を採ります。** パスごとに分け、重い不透明パスをさらに分割します。

### 35.2.2 記録の順序と実行の順序

**重要な点です。**

**`ExecuteCommandLists` に渡す配列の順序が、実行順序になります**(第9章 9.5.2 節)。

**記録の順序は関係ありません。**

```cpp
//--- 並列に記録(順序は不定)---
std::vector<std::jthread> threads;
for (int i = 0; i < kThreadCount; ++i)
{
    threads.emplace_back([this, i] { RecordThread(i); });
}
threads.clear();   // jthread のデストラクタが join する

//--- 順序を指定して投入 ---
ID3D12CommandList* lists[] = {
    m_shadowList.Get(),        // ① 先に実行される
    m_opaqueLists[0].Get(),    // ②
    m_opaqueLists[1].Get(),    // ③
    m_transparentList.Get(),   // ④
    m_postList.Get(),          // ⑤
};
queue->ExecuteCommandLists(5, lists);
```

**C++20 の `std::jthread` を使っています。** デストラクタで自動的に `join` するので、明示的な待機が不要です。

### 35.2.3 状態はリストごとに独立

**第15章 15.4.2 節で書いた内容が、ここで問題になります。**

> `Reset()` を呼ぶと、パイプラインステート、ルートシグネチャ、ビューポート、シザー矩形、頂点バッファ —— **すべての設定が失われます。**

**各コマンドリストは、独立した状態を持ちます。**

**したがって、すべてのリストで設定を繰り返す必要があります。**

```cpp
void Renderer::RecordOpaqueRange(int threadIndex,
                                 std::size_t begin, std::size_t end)
{
    auto* list = m_opaqueLists[threadIndex].Get();

    //--- 各リストで、すべて設定し直す ---
    ID3D12DescriptorHeap* heaps[] = { m_descriptorHeap.Get() };
    list->SetDescriptorHeaps(1, heaps);
    list->SetGraphicsRootSignature(m_rootSignature.Get());
    list->SetGraphicsRootConstantBufferView(0, m_sceneConstantsAddress);

    list->RSSetViewports(1, &m_viewport);
    list->RSSetScissorRects(1, &m_scissor);

    const auto rtv = m_sceneColorMS.Rtv();
    const auto dsv = m_depthMS.Dsv();
    list->OMSetRenderTargets(1, &rtv, FALSE, &dsv);

    list->IASetPrimitiveTopology(D3D_PRIMITIVE_TOPOLOGY_TRIANGLELIST);

    //--- 担当範囲を描画 ---
    for (std::size_t i = begin; i < end; ++i)
    {
        DrawObject(list, *m_visibleObjects[i]);
    }
}
```

**設定のコストが、スレッド数だけ増えます。**

**分割数が多すぎると、この重複が並列化の利得を上回ります。**

### 35.2.4 バリアとクリアはどこで行うか

**これが設計上、最も注意を要する点です。**

**バリアやクリアは、複数のリストにまたがってはいけません。**

```cpp
//--- ❌ 危険 ---
// リスト 0:バリアを張る
// リスト 1:描画          ← リスト 0 の完了が保証されない?
```

**実際には、同じキューに順番に投入されるので保証されます。** しかし、**設計として分かりにくくなります。**

**本書は、専用のリストを前後に置きます。**

```
[Pre] バリア + クリア
  [Opaque 0] [Opaque 1] [Opaque 2] [Opaque 3]   ← 並列記録
[Post] バリア
```

```cpp
ID3D12CommandList* lists[] = {
    m_preList.Get(),           // バリア、クリア
    m_opaqueLists[0].Get(),
    m_opaqueLists[1].Get(),
    m_opaqueLists[2].Get(),
    m_opaqueLists[3].Get(),
    m_postList.Get(),          // バリア
};
```

**`Pre` と `Post` はメインスレッドで記録します。** 軽い処理なので、並列化する意味がありません。

---

## 35.3 実装する

### 35.3.1 スレッド数を決める

```cpp
UINT DetermineThreadCount()
{
    const UINT hardware = std::thread::hardware_concurrency();

    if (hardware == 0)
    {
        return 1;      // 取得できない場合
    }

    //--- メインスレッドの分を残す ---
    const UINT workers = (hardware > 2) ? (hardware - 1) : 1;

    //--- 上限を設ける ---
    return std::min(workers, kMaxRecordingThreads);
}
```

**`hardware_concurrency()` は論理コア数を返します。**

**すべてを使い切るべきではありません。** OS やドライバのスレッドも動いています。

**上限も設けます。**

```cpp
inline constexpr UINT kMaxRecordingThreads = 8;
```

**分割数を増やしても、35.2.3 節の設定コストが増えるだけの場面があります。** 実測で決めるべき値です。

### 35.3.2 スレッドプール

**毎フレームスレッドを作るのは無駄です。**

```cpp
// src/Core/ThreadPool.h
#pragma once
#include "std_import.h"

namespace Core
{
    class ThreadPool
    {
    public:
        explicit ThreadPool(UINT threadCount);
        ~ThreadPool();

        ThreadPool(const ThreadPool&)            = delete;
        ThreadPool& operator=(const ThreadPool&) = delete;

        //--- タスクを投入する ---
        void Submit(std::function<void()> task);

        //--- すべてのタスクの完了を待つ ---
        void WaitAll();

        UINT ThreadCount() const noexcept { return m_threadCount; }

    private:
        void WorkerLoop(std::stop_token stopToken);

        UINT m_threadCount = 0;
        std::vector<std::jthread> m_threads;

        std::mutex              m_mutex;
        std::condition_variable m_taskAvailable;
        std::condition_variable m_allDone;
        std::queue<std::function<void()>> m_tasks;

        std::size_t m_activeTasks = 0;
        bool        m_stopping    = false;
    };
}
```

```cpp
void ThreadPool::WorkerLoop(std::stop_token stopToken)
{
    while (!stopToken.stop_requested())
    {
        std::function<void()> task;

        {
            std::unique_lock lock(m_mutex);

            m_taskAvailable.wait(lock, stopToken,
                [this] { return !m_tasks.empty(); });

            if (stopToken.stop_requested()) return;
            if (m_tasks.empty()) continue;

            task = std::move(m_tasks.front());
            m_tasks.pop();
            ++m_activeTasks;
        }

        task();

        {
            std::lock_guard lock(m_mutex);
            --m_activeTasks;

            if (m_tasks.empty() && m_activeTasks == 0)
            {
                m_allDone.notify_all();
            }
        }
    }
}
```

**`std::stop_token` は C++20 の機能です。** `std::jthread` と組み合わせると、終了要求を安全に扱えます。

**`condition_variable::wait` の `stop_token` を取るオーバーロード**も C++20 で追加されました。**停止要求で待機を抜けられます。**

> **より高度な選択肢**
>
> 実用的なエンジンでは、**ワークスティーリング**を実装します。手空きのスレッドが、他のスレッドのタスクを奪う方式です。
>
> **本書は単純なキュー方式に留めます。** 描画コマンドの記録という用途では、タスクの粒度が大きく揃っているため、スティーリングの効果は限定的です。

### 35.3.3 フレームリソースを拡張する

**第12章 12.2.2 節の `FrameResource` に、スレッド分を追加します。**

```cpp
struct FrameResource
{
    //--- メインスレッド用 ---
    Microsoft::WRL::ComPtr<ID3D12CommandAllocator> mainAllocator;

    //--- ワーカースレッド用 ---
    std::vector<Microsoft::WRL::ComPtr<ID3D12CommandAllocator>> workerAllocators;

    std::uint64_t fenceValue = 0;
};
```

**コマンドリストは、フレームをまたいで再利用します**(35.1.2 節)。

```cpp
class Renderer
{
    // ...
    FrameResource m_frames[kBackBufferCount];

    //--- コマンドリストはフレームごとに不要 ---
    ComPtr<ID3D12GraphicsCommandList> m_preList;
    ComPtr<ID3D12GraphicsCommandList> m_postList;
    std::vector<ComPtr<ID3D12GraphicsCommandList>> m_workerLists;
};
```

**名前を付けるのを忘れないでください**(第6章 6.5 節)。

```cpp
Core::SetDebugNameF(m_workerLists[i].Get(), L"WorkerList[{}]", i);
Core::SetDebugNameF(frame.workerAllocators[i].Get(),
                    L"WorkerAllocator[{}][{}]", frameIndex, i);
```

**第31章の Aftermath でクラッシュを解析するとき、どのスレッドのリストで落ちたかが分かります。**

### 35.3.4 Aftermath のコンテキストを分ける

**第31章 31.2.2 節で、コンテキストハンドルを 1 つだけ作りました。**

> **コマンドリストと 1 対 1 で対応します。**
> **本書はコマンドリストが 1 本なので、ハンドルも 1 つです。** 第35章で並列記録を導入したら、リストごとに作ります。

**ここで、リストごとに作ります。**

```cpp
struct WorkerContext
{
    ComPtr<ID3D12GraphicsCommandList> commandList;
    GFSDK_Aftermath_ContextHandle     aftermathContext{};
};
```

```cpp
for (UINT i = 0; i < m_threadCount; ++i)
{
    // ... コマンドリストを作る ...

    GFSDK_Aftermath_DX12_CreateContextHandle(
        m_workers[i].commandList.Get(),
        &m_workers[i].aftermathContext);
}
```

**第31章 31.2.4 節で `MarkerRegistry` に `std::mutex` を入れておいたのは、このためでした。**

> **`std::mutex` で保護しているのは、第35章の並列記録に備えてのことです。** 現時点では 1 スレッドですが、後で書き換えずに済みます。

**予告通り、書き換えが不要でした。**

### 35.3.5 記録を並列化する

```cpp
Core::Status Renderer::RecordFrameParallel(const Camera& camera)
{
    const UINT index = m_swapChain.CurrentIndex();
    FrameResource& frame = m_frames[index];

    //--- ① すべてのアロケータをリセット ---
    HR_TRY(frame.mainAllocator->Reset());
    for (auto& allocator : frame.workerAllocators)
    {
        HR_TRY(allocator->Reset());
    }

    //--- ② Pre リスト:バリアとクリア ---
    HR_TRY(m_preList->Reset(frame.mainAllocator.Get(), nullptr));
    {
        GPU_MARKER(m_preList.Get(), m_preAftermathContext, "Frame Begin");

        TransitionTo(m_preList7.Get(), m_sceneColorMS,
                     BarrierStates::kRenderTarget, true);

        const auto rtv = m_sceneColorMS.Rtv();
        const auto dsv = m_depthMS.Dsv();

        m_preList->ClearRenderTargetView(rtv, kSceneClearColor, 0, nullptr);
        m_preList->ClearDepthStencilView(
            dsv, D3D12_CLEAR_FLAG_DEPTH, kDepthClearValue, 0, 0, nullptr);
    }
    HR_TRY(m_preList->Close());

    //--- ③ ワーカーに分配 ---
    const std::size_t objectCount = m_visibleObjects.size();
    const std::size_t perThread =
        DivideRoundUp(objectCount, m_threadCount);

    for (UINT i = 0; i < m_threadCount; ++i)
    {
        const std::size_t begin = i * perThread;
        const std::size_t end   = std::min(begin + perThread, objectCount);

        m_threadPool->Submit([this, i, begin, end, &frame]()
        {
            RecordOpaqueRange(i, frame, begin, end);
        });
    }

    //--- ④ 完了を待つ ---
    m_threadPool->WaitAll();

    //--- ⑤ Post リスト ---
    HR_TRY(m_postList->Reset(frame.mainAllocator.Get(), nullptr));
    {
        GPU_MARKER(m_postList.Get(), m_postAftermathContext, "Frame End");

        TransitionTo(m_postList7.Get(), m_sceneColorMS,
                     BarrierStates::kResolveSource);
        // ... リゾルブ、ポストエフェクト ...
    }
    HR_TRY(m_postList->Close());

    return {};
}
```

**アロケータのリセットは、メインスレッドでまとめて行います。**

**理由は、`Reset` が安全なタイミングを一箇所で判断したいからです。** 第12章 12.3.1 節でフェンスを待った直後です。

### 35.3.6 投入する

```cpp
void Renderer::SubmitFrame()
{
    std::vector<ID3D12CommandList*> lists;
    lists.reserve(m_threadCount + 2);

    lists.push_back(m_preList.Get());

    for (UINT i = 0; i < m_threadCount; ++i)
    {
        lists.push_back(m_workers[i].commandList.Get());
    }

    lists.push_back(m_postList.Get());

    m_queue.Execute(lists);
}
```

**第9章 9.6.1 節で作った `Execute(std::span<...>)` が、ここで役立ちます。**

**1 回の呼び出しでまとめて投入します**(第9章 9.5.2 節)。

---

## 35.4 バンドル

### 35.4.1 何をするものか

**記録済みのコマンドを、再生できる形で保存します。**

```
バンドル:  [PSO 設定][頂点バッファ][描画]
              ↓ 何度でも実行できる
コマンドリスト:  ExecuteBundle(bundle)
```

**利点は、記録コストが 1 回で済むことです。**

**毎フレーム同じコマンドを記録している場合、その分が節約できます。**

### 35.4.2 制約

**バンドルには、多くの制約があります。**

| できないこと | 理由 |
|---|---|
| `ExecuteCommandLists` に直接渡す | バンドルはキューに投入できない |
| レンダーターゲットの設定 | 親リストの設定を引き継ぐ |
| バリア | 同上 |
| クリア | 同上 |
| `Dispatch` の一部 | 制限あり |
| **デスクリプタヒープの変更** | **親と同じでなければならない** |
| ルートシグネチャの変更 | 親と異なる場合は再設定が必要 |

**「状態を変える操作」の多くが禁止されています。**

**バンドルは「描画コマンドの塊」であって、パスの記述ではありません。**

### 35.4.3 作る

```cpp
//--- 専用のアロケータが必要 ---
ComPtr<ID3D12CommandAllocator> bundleAllocator;
HR_TRY(device->CreateCommandAllocator(
    D3D12_COMMAND_LIST_TYPE_BUNDLE,        // ← BUNDLE
    IID_PPV_ARGS(&bundleAllocator)));

//--- バンドルを作る ---
ComPtr<ID3D12GraphicsCommandList> bundle;
HR_TRY(device4->CreateCommandList1(
    0,
    D3D12_COMMAND_LIST_TYPE_BUNDLE,        // ← BUNDLE
    D3D12_COMMAND_LIST_FLAG_NONE,
    IID_PPV_ARGS(&bundle)));

//--- 記録 ---
HR_TRY(bundle->Reset(bundleAllocator.Get(), m_opaquePso.Get()));

bundle->SetGraphicsRootSignature(m_rootSignature.Get());
bundle->IASetPrimitiveTopology(D3D_PRIMITIVE_TOPOLOGY_TRIANGLELIST);
bundle->IASetVertexBuffers(0, 1, &m_mergedVbv);
bundle->IASetIndexBuffer(&m_mergedIbv);

for (const auto& batch : m_staticBatches)
{
    bundle->DrawIndexedInstanced(
        batch.indexCount, batch.instanceCount,
        batch.startIndex, batch.baseVertex, batch.startInstance);
}

HR_TRY(bundle->Close());
```

**実行します。**

```cpp
//--- 親リストで、状態を設定してから ---
commandList->SetDescriptorHeaps(1, heaps);
commandList->OMSetRenderTargets(1, &rtv, FALSE, &dsv);
commandList->RSSetViewports(1, &viewport);
commandList->RSSetScissorRects(1, &scissor);

//--- バンドルを実行 ---
commandList->ExecuteBundle(bundle.Get());
```

### 35.4.4 いつ使うか

**正直に書きます。バンドルの出番は限られています。**

**理由は 3 つです。**

**理由 1:内容が固定でなければならない**

**動的なシーンでは使えません。** 毎フレーム描画対象が変わるなら、記録し直すことになります。

**理由 2:第34章の `ExecuteIndirect` が上位互換**

**GPU が生成したコマンドを実行するほうが、柔軟です。**

**理由 3:CPU の負荷は既に低い**

**第34章で GPU カリングを実装したので、CPU が記録するコマンドは大幅に減りました。**

**バンドルが有効な場面を挙げます。**

| 場面 | 理由 |
|---|---|
| **UI の描画** | 内容が固定的 |
| **静的な背景** | 動かない |
| **同じ描画を複数回**(キューブマップの 6 面など) | 記録が 1 回で済む |

**本書は紹介に留めます。** 実装しなくても、これまでのコードは正しく動きます。

---

## 35.5 コピーキューによる非同期転送

### 35.5.1 なぜ分けるか

**第16章と第20章で、`ResourceUploader` を作りました。**

```cpp
uploader.Begin();
uploader.UploadBuffer(...);
uploader.UploadTexture(...);
uploader.End(queue);      // ← GPU の完了を待つ
```

**初期化時なら問題ありません。**

**しかし、実行中にテクスチャを読み込みたい場合があります。**

```
プレイヤーが移動 → 新しい領域のテクスチャが必要 → 転送
```

**描画キューで転送すると、描画が止まります。**

**専用のコピーキューを使えば、並行して転送できます。**

```
描画キュー:  [フレーム N][フレーム N+1][フレーム N+2]
コピーキュー:    [テクスチャ転送 ────────────]
```

### 35.5.2 コピーキューの特性

**第9章 9.2.2 節の表を、改めて確認します。**

| 種類 | できること |
|---|---|
| DIRECT | 描画・コンピュート・コピー |
| COMPUTE | コンピュートとコピー |
| **COPY** | **コピーのみ** |

**COPY キューには、専用のハードウェアがあります。**

**NVIDIA の GPU には「コピーエンジン」が搭載されており、描画とは独立して動作します。** PCIe 越しの転送に最適化されています。

**制約もあります。**

| 制約 | 説明 |
|---|---|
| **状態遷移が限定的** | `COMMON`、`COPY_SOURCE`、`COPY_DEST` のみ |
| クリアができない | `ClearRenderTargetView` などは不可 |
| **リソースの初期状態** | `COMMON` から始める必要がある |

### 35.5.3 キュー間の同期

**第10章 10.1.4 節で挙げた 4 つの操作を、思い出してください。**

| 操作 | 誰が値を変える/待つか | 使う章 |
|---|---|---|
| `CommandQueue::Signal` | GPU が値を変える | 第10章 |
| `Fence::SetEventOnCompletion` | CPU が待つ | 第10章 |
| `Fence::Signal` | CPU が値を変える | **本章** |
| **`CommandQueue::Wait`** | **GPU が待つ** | **本章** |

**「GPU が GPU を待つ」を実現します。**

```cpp
//--- コピーキューで転送 ---
copyQueue->ExecuteCommandLists(1, copyLists);
const auto copyFenceValue = m_copyFence.Signal(copyQueue);

//--- 描画キューは、転送の完了を待ってから実行 ---
directQueue->Wait(m_copyFence.Get(), copyFenceValue);
directQueue->ExecuteCommandLists(1, drawLists);
```

**`Wait` は CPU をブロックしません。** 「このキューの実行を、フェンスが指定値に達するまで止める」という指示を積むだけです。

**CPU は先へ進めます。**

### 35.5.4 非同期アップローダ

```cpp
// src/Graphics/AsyncUploader.h
#pragma once
#include "std_import.h"
#include "Core/Error.h"
#include "Graphics/Fence.h"

namespace Graphics
{
    //-----------------------------------------------------------
    // コピーキューで非同期に転送する。
    //
    // 第16章の ResourceUploader は完了を待つが、
    // こちらは待たない。転送中も描画が進む。
    //-----------------------------------------------------------
    class AsyncUploader
    {
    public:
        struct PendingUpload
        {
            Microsoft::WRL::ComPtr<ID3D12Resource> resource;
            Microsoft::WRL::ComPtr<ID3D12Resource> staging;
            std::uint64_t fenceValue = 0;
            std::function<void()> onComplete;
        };

        Core::Status Initialize(ID3D12Device* device);
        void         Shutdown();

        //--- 転送を要求する。完了は待たない ---
        Core::Status RequestUpload(const DdsImage& image,
                                   std::wstring_view name,
                                   std::function<void(ComPtr<ID3D12Resource>)> onComplete);

        //--- 毎フレーム呼ぶ。完了したものを処理する ---
        void Update();

        //--- 描画キューに待機を指示する ---
        void InsertWait(ID3D12CommandQueue* targetQueue);

        bool HasPendingWork() const noexcept;

    private:
        ID3D12Device* m_device = nullptr;

        CommandQueue m_copyQueue;
        Fence        m_copyFence;

        Microsoft::WRL::ComPtr<ID3D12CommandAllocator>    m_allocator;
        Microsoft::WRL::ComPtr<ID3D12GraphicsCommandList> m_commandList;

        std::mutex m_mutex;
        std::vector<PendingUpload> m_pending;
        std::uint64_t m_lastSignaledValue = 0;
    };
}
```

**転送の実装は、第20章 20.4.3 節とほぼ同じです。**

**違いは、状態遷移を行わないことです。**

```cpp
//--- ❌ コピーキューではできない ---
// const auto barrier = MakeTransitionBarrier(
//     texture, COPY_DEST, PIXEL_SHADER_RESOURCE);

//--- リソースは COMMON 状態のまま残す ---
```

**35.5.2 節の制約です。**

**遷移は、描画キューで行います。**

```cpp
void Renderer::PromoteUploadedResources()
{
    for (auto& resource : m_newlyUploaded)
    {
        //--- 描画キューで、使える状態へ ---
        TransitionTo(m_commandList7.Get(), resource,
                     BarrierStates::kPixelShaderResource);
    }
    m_newlyUploaded.clear();
}
```

> **状態の暗黙的昇格**
>
> 第16章 16.3.5 節で触れた仕組みが、ここで役立ちます。
>
> > **バッファと一部のテクスチャは、`COMMON` 状態から任意の状態へ、バリアなしで自動的に遷移する。**
>
> **`COMMON` のまま残したリソースは、描画キューで使われるときに自動で昇格します。**
>
> **明示的なバリアは不要な場合もあります。** ただし本書は、第16章 16.3.5 節の方針通り明示的に書きます。

### 35.5.5 フレームループへの組み込み

```cpp
Core::Status Renderer::RenderFrame(const Camera& camera)
{
    //--- ① 完了した転送を処理 ---
    m_asyncUploader.Update();

    //--- ② フレームリソースの準備(第12章)---
    const UINT index = m_swapChain.CurrentIndex();
    FrameResource& frame = m_frames[index];

    if (auto r = m_fence.Wait(frame.fenceValue); !r) { return r; }

    // ...

    //--- ③ 記録と投入 ---
    RecordFrameParallel(camera);

    //--- ④ 転送中なら、描画キューを待たせる ---
    if (m_asyncUploader.HasPendingWork())
    {
        m_asyncUploader.InsertWait(m_queue.Get());
    }

    SubmitFrame();

    // ...
}
```

**④ が、35.5.3 節の「GPU が GPU を待つ」です。**

**転送が完了していれば、待機は即座に通過します。**

---

## 35.6 非同期コンピュート

### 35.6.1 何が並行するか

**COMPUTE キューを使うと、描画とコンピュートを重ねられます。**

```
描画キュー:      [シャドウ][不透明  ][ポスト]
コンピュートキュー:    [パーティクル更新]
```

**GPU の異なるユニットが使われる場合に、効果があります。**

| 組み合わせ | 効果 |
|---|---|
| **シャドウパス(ROP 中心)+ コンピュート(SM 中心)** | **大きい** |
| 不透明パス(SM 中心)+ コンピュート(SM 中心) | 小さい |
| ポストエフェクト(帯域中心)+ コンピュート(SM 中心) | 中程度 |

**第29章 29.3.4 節で測ったパスごとの特性が、ここで判断材料になります。**

> ```
> Shadow Pass    ████░░░░░░  ROP 中心(深度書き込みのみ)
> Opaque         ████████░░  SM とテクスチャのバランス
> ```

**シャドウパスは ROP 中心なので、SM が空いています。** ここにコンピュートを重ねるのが効果的です。

### 35.6.2 実装

```cpp
void Renderer::DispatchAsyncCompute()
{
    //--- コンピュートキューで記録 ---
    HR_TRY(m_computeAllocator->Reset());
    HR_TRY(m_computeList->Reset(m_computeAllocator.Get(), nullptr));

    {
        GPU_MARKER(m_computeList.Get(), m_computeAftermathContext,
                   "Async Particle Update");

        m_computeList->SetComputeRootSignature(m_particleRootSig.Get());
        m_computeList->SetPipelineState(m_particleUpdatePso.Get());
        // ...
        m_computeList->Dispatch(groupCount, 1, 1);
    }

    HR_TRY(m_computeList->Close());

    //--- 前フレームの描画完了を待つ ---
    m_computeQueue.Get()->Wait(m_fence.Get(), m_previousFrameFence);

    m_computeQueue.Execute(m_computeList.Get());
    m_computeFenceValue = m_computeFence.Signal(m_computeQueue.Get());
}
```

**描画側では、パーティクルを描く直前に待ちます。**

```cpp
//--- パーティクル描画の前に、更新の完了を待つ ---
m_queue.Get()->Wait(m_computeFence.Get(), m_computeFenceValue);
```

### 35.6.3 効果は測らなければ分からない

**非同期コンピュートは、必ず速くなるわけではありません。**

**理由は 3 つです。**

| 理由 | 説明 |
|---|---|
| **リソースの競合** | 同じキャッシュや帯域を奪い合う |
| **同期のオーバーヘッド** | フェンスの待機にもコストがある |
| **スケジューリング** | ドライバの判断で並行しないこともある |

**第29章 29.3 節の GPU Trace で測ってください。**

**タイムライン上で、2 つのキューが重なっているかが見えます。**

**重なっていなければ、効果はありません。**

---

## 35.7 デバッグ

### 35.7.1 何が難しくなるか

**マルチスレッドは、バグの再現性を失わせます。**

| 症状 | 原因 |
|---|---|
| **たまに落ちる** | データ競合 |
| **環境によって違う** | スレッド数の違い |
| **デバッガを通すと直る** | タイミングが変わる |

**第30章 30.7.1 節で書いた「バリア不足」と、症状が似ています。**

**切り分けが必要です。**

### 35.7.2 スレッド数を 1 にする

**最も確実な切り分け手段です。**

```cpp
config.recordingThreadCount = 1;
```

**絵が直るなら、並列化が原因です。**

**第30章 30.7.2 節の「強制全同期」と、同じ発想です。**

### 35.7.3 同期検証を有効にする

**第7章 7.1.3 節と第30章 30.6.2 節で触れた設定です。**

```cpp
debug1->SetEnableSynchronizedCommandQueueValidation(TRUE);
```

**複数キューを使う場合の検証が有効になります。**

**リソースの状態が、キューをまたいで矛盾していないかを確認できます。**

### 35.7.4 名前でスレッドを識別する

**第6章 6.5 節の習慣が、ここでも効きます。**

```
D3D12 ERROR: ID3D12CommandAllocator::Reset:
  A command allocator 'WorkerAllocator[1][3]' is still in use
```

**どのフレームの、どのスレッドのアロケータかが分かります。**

**第31章の Aftermath でも同様です。**

```
Markers:
  Queue 0, CommandList 'WorkerList[2]':
    [Executing]  Opaque Range 2
```

**どのスレッドで落ちたかが特定できます。**

### 35.7.5 Nsight Systems を使う

**第2章 2.4.2 節でインストールしたまま、使っていませんでした。**

> こちらは**システム全体**のプロファイラです。Nsight Graphics が「1 フレームの中身」を見るのに対し、Nsight Systems は「CPU スレッドと GPU の仕事が時間軸上でどう噛み合っているか」を見ます。
> **本書では第35章(マルチスレッド化)と第38章(最適化)で使います。**

**ここで出番です。**

**見えるもの:**

| 情報 | 用途 |
|---|---|
| **CPU スレッドごとの実行状況** | 並列化が効いているか |
| **スレッドの待機時間** | 負荷の偏り |
| **キューごとの GPU 実行** | 非同期コンピュートの効果 |
| API 呼び出しのタイムライン | どこで時間を使っているか |

**理想的な状態:**

```
Main:     [Pre][待機          ][Post][Submit]
Worker 0:      [Record ────────]
Worker 1:      [Record ────────]
Worker 2:      [Record ────────]
Worker 3:      [Record ────────]
```

**すべてのワーカーが同じ長さなら、負荷が均等です。**

**偏っている場合:**

```
Worker 0:      [Record ────────────────]
Worker 1:      [Record ──]
Worker 2:      [Record ──]
Worker 3:      [Record ──]
```

**分割の方法を見直す必要があります。**

---

## ✅ 本章のゴール:CPU と GPU の並列度を上げる

### Step 1:スレッド数を確認する

```
[Info ] Renderer.cpp(122): recording threads: 7 (hardware concurrency: 8)
[Info ] Renderer.cpp(128): command allocators: 21 (7 threads x 3 frames)
[Info ] Renderer.cpp(129): command lists: 9 (7 workers + pre + post)
```

**35.1.2 節の計算と一致していることを確認してください。**

### Step 2:並列記録を有効にする

**絵が変わらないことを確認します。**

```cpp
config.recordingThreadCount = 1;   // 逐次
config.recordingThreadCount = 7;   // 並列
```

**まったく同じ絵になるはずです。**

**違う場合、記録の順序に依存する処理が混ざっています。**

### Step 3:CPU の記録時間を測る

**第22章 22.2 節のタイマを使います。**

```cpp
const auto start = std::chrono::steady_clock::now();
RecordFrameParallel(camera);
const auto elapsed = std::chrono::steady_clock::now() - start;
```

| スレッド数 | 予想される記録時間 |
|---|---|
| 1 | 基準 |
| 2 | 約 60% |
| 4 | 約 40% |
| 8 | 約 35%(頭打ち) |

**線形には減りません。** 35.2.3 節の設定コストが重複するためです。

**オブジェクト数を増やすと、並列化の効果が大きくなります。**

### Step 4:分割数を変える

```cpp
config.recordingThreadCount = 2;
config.recordingThreadCount = 4;
config.recordingThreadCount = 8;
config.recordingThreadCount = 16;   // コア数を超える
```

**最適値を探してください。**

**コア数を超えると、かえって遅くなります。**

### Step 5:アロケータを共有してみる

```cpp
//--- ❌ すべてのスレッドが同じアロケータを使う ---
m_workerLists[i]->Reset(frame.mainAllocator.Get(), nullptr);
```

**デバッグレイヤーが検出します。**

```
D3D12 ERROR: ID3D12GraphicsCommandList::Reset:
  The command allocator is currently in use by another command list.
```

**第9章 9.2.3 節の規則 1 に違反しています。**

**検出されない場合もあります。** タイミング次第です。**「たまに壊れる」の典型です。**

**確認したら元に戻してください。**

### Step 6:Nsight Systems で確認する

**35.7.5 節の手順で、スレッドのタイムラインを見てください。**

**確認すべき点:**

- [ ] すべてのワーカーが同時に動いている
- [ ] 待機時間が短い
- [ ] 負荷が均等

**偏っている場合、分割方法を見直します。**

### Step 7:非同期転送を試す

**実行中にテクスチャを読み込みます。**

```cpp
if (input.WasKeyPressed('L'))
{
    m_asyncUploader.RequestUpload(
        LoadDds("assets/large_texture.dds").value(),
        L"LargeTexture",
        [this](auto resource) { m_dynamicTexture = resource; });
}
```

**転送中もフレームレートが落ちないことを確認してください。**

**同期版(第16章の `ResourceUploader`)と比較すると、差が明確になります。**

```cpp
//--- 同期版:フレームが止まる ---
uploader.Begin();
uploader.UploadTexture(image, L"LargeTexture");
uploader.End(m_queue.Get());      // ← ここで待つ
```

### Step 8:非同期コンピュートを試す

**第32章のパーティクル更新を、COMPUTE キューへ移します。**

**GPU Trace で、2 つのキューのタイムラインを確認してください。**

```
Direct Queue:   [Shadow][Opaque      ][Post]
Compute Queue:  [Particle Update]
```

**重なっていれば成功です。**

**重なっていない場合、依存関係が強すぎるか、ドライバが並行させていません。**

### Step 9:バンドルを試す(任意)

**静的なオブジェクトだけをバンドルにします。**

**記録時間の差を測ってください。**

**動的なシーンでは効果が薄いことも、実測で確認できます。**

---

### 本章の達成状態

- [ ] スレッドごとにアロケータを分けた
- [ ] スレッドごとにコマンドリストを分けた
- [ ] フレーム × スレッドでアロケータを確保した
- [ ] スレッドプールを実装した
- [ ] Pre / Post リストでバリアとクリアを分離した
- [ ] 各リストで状態を設定し直している
- [ ] Aftermath のコンテキストをリストごとに作った
- [ ] `ExecuteCommandLists` の順序を制御している
- [ ] コピーキューで非同期転送を実装した
- [ ] `CommandQueue::Wait` でキュー間を同期している
- [ ] 同期検証を有効にした
- [ ] Nsight Systems でタイムラインを確認した
- [ ] **並列記録で CPU 時間が短縮された**

---

## トラブルシューティング

| 症状 | 原因 | 対処 |
|---|---|---|
| アロケータのエラー | スレッド間で共有している | 35.1.2 |
| たまに落ちる | データ競合 | スレッド数 1 で切り分け(35.7.2) |
| 絵が並列時だけ違う | 記録順序に依存 | 依存を排除する |
| 描画が抜ける | リストで状態を設定していない | 35.2.3 |
| 並列化しても速くならない | 設定コストの重複 | 分割数を減らす(Step 4) |
| ワーカーの負荷が偏る | 分割が不均等 | Nsight Systems で確認 |
| コピーキューでエラー | 状態遷移を試みた | 35.5.2 |
| 転送したテクスチャが使えない | 状態遷移を忘れた | 35.5.4 |
| 非同期コンピュートが重ならない | 依存関係が強い | GPU Trace で確認 |
| キュー間で不整合 | 同期検証を有効に | 35.7.3 |

---

## まとめ

**1. デバイスとキューは安全、記録に使うものは危険。**
D3D12 はドライバのロックを廃止し、代わりに「共有しない」ことをアプリに要求します。

**2. アロケータは スレッド × フレーム、リストは スレッドのみ。**
リストはフレームをまたいで再利用できます。

**3. 記録の順序と実行の順序は別物。**
`ExecuteCommandLists` に渡す配列が、実行順序を決めます。

**4. バリアとクリアは専用リストに分ける。**
複数のリストにまたがると、設計が分かりにくくなります。

**5. 各リストで状態を設定し直す。**
第15章 15.4.2 節の「`Reset` で状態が消える」が、リストごとに適用されます。

**6. バンドルの出番は限られている。**
第34章の `ExecuteIndirect` が、多くの場面で上位互換です。

**7. `CommandQueue::Wait` で GPU が GPU を待つ。**
第10章 10.1.4 節で挙げた 4 操作のうち、残り 2 つがここで使われました。

**8. 効果は測らなければ分からない。**
並列化は必ず速くなるわけではありません。**Nsight Systems で、実際に重なっているかを確認してください。**

次章ではメッシュシェーダーを扱います。**第34章 34.4.5 節で「インスタンスのまとめは、第36章がより良い解を提供する」と書きました。** 頂点シェーダーとインデックスバッファという 20 年来の枠組みを離れ、**GPU 上でジオメトリを生成する**方式に移行します。

---

## 参考リンク

| 内容 | URL |
|---|---|
| マルチスレッドと D3D12 | https://learn.microsoft.com/ja-jp/windows/win32/direct3d12/multi-engine |
| バンドル | https://learn.microsoft.com/ja-jp/windows/win32/direct3d12/recording-command-lists-and-bundles |
| `ID3D12CommandQueue::Wait` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/nf-d3d12-id3d12commandqueue-wait |
| 複数エンジンの同期 | https://learn.microsoft.com/ja-jp/windows/win32/direct3d12/user-mode-heap-synchronization |
| `std::jthread` | https://ja.cppreference.com/w/cpp/thread/jthread |
| Nsight Systems | https://docs.nvidia.com/nsight-systems/ |
