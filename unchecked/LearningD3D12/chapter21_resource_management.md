# 第21章 デスクリプタとリソース管理の設計

第2部の締めくくりです。

ここまで、リソースは行き当たりばったりに作ってきました。第18章で CBV を 3 個、第20章で SRV を 1 個。**デスクリプタの番号は、コードに直接書いてありました。**

```cpp
constexpr UINT kSrvIndex = 3;   // CBV が 0〜2 だから 3
```

**第23章でモデルを読み込み、第25章で複数のオブジェクトを描くようになると、この方式は破綻します。** テクスチャが 20 枚あれば、番号を手で管理することになります。

本章では、この先の 20 章を支える 3 つの仕組みを作ります。

| 仕組み | 解決する問題 |
|---|---|
| **デスクリプタアロケータ** | 番号の手動管理 |
| **アップロードリングバッファ** | 毎フレームのデータ転送 |
| **遅延解放キュー** | 「GPU がまだ使っているかも」問題 |

**本章のゴール**
3 つの仕組みを実装し、`ReportLiveDeviceObjects` でリークがゼロであることを確認する。

---

## 21.1 ヒープの分割設計

### 21.1.1 制約の再確認

第18章 18.1.2 節で述べた制約を思い出してください。

> **コマンドリストに同時にバインドできるのは、CBV_SRV_UAV ヒープが 1 つ、SAMPLER ヒープが 1 つまで。**

そして、ヒープの切り替えは重い操作です。**現実的な設計は 1 つに定まります。**

> **起動時に十分大きなヒープを 1 つ作り、その中を自分で切り分けて使う。**

### 21.1.2 2 種類の割り当て

**デスクリプタには、性質の異なる 2 種類があります。**

| 種類 | 例 | 寿命 |
|---|---|---|
| **永続(static)** | テクスチャの SRV | **アプリケーション終了まで** |
| **一時(dynamic)** | 毎フレーム変わる CBV | **1 フレーム** |

**これを同じ方法で管理すると、必ず破綻します。**

永続的なものを詰めた後に一時的なものを詰めると、解放されたときに穴が空きます。穴を埋めようとすると、フリーリストや断片化の管理が必要になります。**毎フレーム数百個を割り当てる用途で、それは重すぎます。**

**答えは、ヒープを領域で分けることです。**

```
CBV_SRV_UAV ヒープ(たとえば 8192 個)
┌──────────────────────┬──────────────────────────────────┐
│  永続領域(2048)     │  一時領域(2048 × 3 フレーム)     │
│  ← 前から詰める       │  ← フレームごとにリセット          │
└──────────────────────┴──────────────────────────────────┘
```

**一時領域は、さらにフレーム数で分割します。** 第12章 12.2 節のフレームリソースと同じ考え方です。GPU がまだ読んでいる可能性があるので、フレームごとに別の区画を使います。

### 21.1.3 永続領域のアロケータ

**単純なフリーリストで十分です。**

```cpp
// src/Graphics/DescriptorAllocator.h
#pragma once
#include "std_import.h"
#include "Core/Error.h"

namespace Graphics
{
    //-----------------------------------------------------------
    // ヒープ内の 1 個ぶんの位置。
    // CPU / GPU 両方のハンドルを持つ。
    //-----------------------------------------------------------
    struct DescriptorHandle
    {
        D3D12_CPU_DESCRIPTOR_HANDLE cpu{};
        D3D12_GPU_DESCRIPTOR_HANDLE gpu{};
        UINT index = kInvalidIndex;

        static constexpr UINT kInvalidIndex = 0xFFFFFFFF;

        [[nodiscard]] bool IsValid() const noexcept
        {
            return index != kInvalidIndex;
        }
    };

    //-----------------------------------------------------------
    // 永続デスクリプタの割り当て。
    // 解放されたスロットは再利用する。
    //-----------------------------------------------------------
    class StaticDescriptorAllocator
    {
    public:
        void Initialize(D3D12_CPU_DESCRIPTOR_HANDLE cpuStart,
                        D3D12_GPU_DESCRIPTOR_HANDLE gpuStart,
                        UINT baseIndex,
                        UINT capacity,
                        UINT incrementSize);

        [[nodiscard]] DescriptorHandle Allocate();
        void Free(const DescriptorHandle& handle);

        UINT Used()     const noexcept { return m_allocated - static_cast<UINT>(m_freeList.size()); }
        UINT Capacity() const noexcept { return m_capacity; }

    private:
        D3D12_CPU_DESCRIPTOR_HANDLE m_cpuStart{};
        D3D12_GPU_DESCRIPTOR_HANDLE m_gpuStart{};
        UINT m_baseIndex     = 0;
        UINT m_capacity      = 0;
        UINT m_incrementSize = 0;
        UINT m_allocated     = 0;      // 一度でも配った数
        std::vector<UINT> m_freeList;  // 返却されたローカル番号
    };
}
```

```cpp
DescriptorHandle StaticDescriptorAllocator::Allocate()
{
    UINT local = 0;

    if (!m_freeList.empty())
    {
        local = m_freeList.back();
        m_freeList.pop_back();
    }
    else if (m_allocated < m_capacity)
    {
        local = m_allocated++;
    }
    else
    {
        LOG_ERROR(L"descriptor heap exhausted ({} used)", m_capacity);
        D3D_ASSERT_MSG(false, L"デスクリプタヒープが枯渇しました");
        return {};
    }

    const UINT global = m_baseIndex + local;

    DescriptorHandle handle{};
    handle.cpu   = OffsetHandle(m_cpuStart, local, m_incrementSize);
    handle.gpu   = OffsetHandle(m_gpuStart, local, m_incrementSize);
    handle.index = global;
    return handle;
}
```

**`index` を持たせている**点に注目してください。CPU / GPU ハンドルだけでも描画はできますが、**第33章のバインドレスでは、この番号そのものをシェーダーに渡します。** そのときのために用意しておきます。

**枯渇したときに `D3D_ASSERT` で止める**のも意図的です。無効なハンドルを返して黙って進むと、描画結果がおかしくなるだけで原因が分かりません。**第6章 6.4.2 節で作ったアサートが、ここで働きます。**

### 21.1.4 一時領域のアロケータ

**フレームごとに、単に先頭へ戻すだけです。**

```cpp
class DynamicDescriptorAllocator
{
public:
    void Initialize(D3D12_CPU_DESCRIPTOR_HANDLE cpuStart,
                    D3D12_GPU_DESCRIPTOR_HANDLE gpuStart,
                    UINT baseIndex,
                    UINT capacityPerFrame,
                    UINT frameCount,
                    UINT incrementSize);

    // フレームの先頭で呼ぶ
    void BeginFrame(UINT frameIndex);

    // 連続した count 個を確保する
    [[nodiscard]] DescriptorHandle Allocate(UINT count = 1);

private:
    // ...
    UINT m_currentFrame  = 0;
    UINT m_offsetInFrame = 0;
};
```

```cpp
void DynamicDescriptorAllocator::BeginFrame(UINT frameIndex)
{
    m_currentFrame  = frameIndex;
    m_offsetInFrame = 0;     // 巻き戻すだけ
}

DescriptorHandle DynamicDescriptorAllocator::Allocate(UINT count)
{
    if (m_offsetInFrame + count > m_capacityPerFrame)
    {
        LOG_ERROR(L"dynamic descriptors exhausted this frame");
        D3D_ASSERT_MSG(false, L"1 フレームのデスクリプタ数が上限を超えました");
        return {};
    }

    const UINT local = m_currentFrame * m_capacityPerFrame + m_offsetInFrame;
    m_offsetInFrame += count;

    // ... ハンドルを組み立てて返す ...
}
```

**`Free` がありません。** フレームが終われば全部無効になるので、個別の解放という概念自体が不要です。**これが「一時」と「永続」を分けたことの最大の利点です。**

**`count` を受け取れるようにしている**のは、ディスクリプタテーブルが**連続した範囲**を指すからです(第18章 18.4.1 節)。CBV と SRV を 1 つのテーブルにまとめたい場合、隣り合った 2 個が必要になります。

### 21.1.5 まとめて `DescriptorHeap` にする

```cpp
class DescriptorHeap
{
public:
    struct Config
    {
        UINT staticCapacity        = 2048;
        UINT dynamicCapacityPerFrame = 2048;
    };

    Core::Status Initialize(ID3D12Device* device, const Config& config);

    void BeginFrame(UINT frameIndex)
    {
        m_dynamic.BeginFrame(frameIndex);
    }

    StaticDescriptorAllocator&  Static()  noexcept { return m_static; }
    DynamicDescriptorAllocator& Dynamic() noexcept { return m_dynamic; }

    ID3D12DescriptorHeap* Get() const noexcept { return m_heap.Get(); }

    void LogUsage() const;

private:
    Microsoft::WRL::ComPtr<ID3D12DescriptorHeap> m_heap;
    StaticDescriptorAllocator  m_static;
    DynamicDescriptorAllocator m_dynamic;
};
```

初期化では、1 つのヒープを 2 つの領域に切り分けます。

```cpp
Core::Status DescriptorHeap::Initialize(ID3D12Device* device,
                                        const Config& config)
{
    const UINT dynamicTotal = config.dynamicCapacityPerFrame * kBackBufferCount;
    const UINT total = config.staticCapacity + dynamicTotal;

    D3D12_DESCRIPTOR_HEAP_DESC desc{};
    desc.Type           = D3D12_DESCRIPTOR_HEAP_TYPE_CBV_SRV_UAV;
    desc.NumDescriptors = total;
    desc.Flags          = D3D12_DESCRIPTOR_HEAP_FLAG_SHADER_VISIBLE;
    desc.NodeMask       = 0;

    HR_TRY(device->CreateDescriptorHeap(&desc, IID_PPV_ARGS(&m_heap)));
    Core::SetDebugName(m_heap.Get(), L"MainDescriptorHeap");

    const auto cpuStart = m_heap->GetCPUDescriptorHandleForHeapStart();
    const auto gpuStart = m_heap->GetGPUDescriptorHandleForHeapStart();
    const UINT increment = device->GetDescriptorHandleIncrementSize(
        D3D12_DESCRIPTOR_HEAP_TYPE_CBV_SRV_UAV);

    //--- 前半:永続 ---
    m_static.Initialize(cpuStart, gpuStart, 0,
                        config.staticCapacity, increment);

    //--- 後半:一時 ---
    m_dynamic.Initialize(
        OffsetHandle(cpuStart, config.staticCapacity, increment),
        OffsetHandle(gpuStart, config.staticCapacity, increment),
        config.staticCapacity,
        config.dynamicCapacityPerFrame,
        kBackBufferCount,
        increment);

    LOG_INFO(L"descriptor heap: {} total ({} static + {} x {} dynamic)",
             total, config.staticCapacity,
             config.dynamicCapacityPerFrame, kBackBufferCount);
    return {};
}
```

**上限について。** シェーダー可視の CBV_SRV_UAV ヒープは、Resource Binding Tier 3 なら **100 万個**まで作れます。第7章 7.5.3 節で問い合わせた `ResourceBindingTier` がここで意味を持ちます。

**ただし、大きなヒープはメモリを消費します。** 1 個あたり 32〜64 バイトなので、8192 個で 256〜512KB 程度です。**必要になったら増やす、で構いません。**

---

## 21.2 アップロードバッファのリングバッファ化

### 21.2.1 何が問題か

**第18章の定数バッファは、行列 1 つぶんの領域を固定で持っていました。**

```cpp
// 768 バイト固定(256 × 3 フレーム)
UINT64 totalSize = kSceneConstantsSize * kBackBufferCount;
```

**第25章で複数のオブジェクトを描くようになると、これでは足りません。**

オブジェクトごとにワールド行列が要ります。100 個描くなら 100 個ぶん。**しかも数は毎フレーム変わります。**

**リソースを毎フレーム作り直すのは論外です。** `CreateCommittedResource` は重い操作で、しかも解放のタイミング問題(第16章 16.3.4 節)が付きまといます。

**答えは、大きなバッファを 1 本用意して、切り分けて使うことです。** デスクリプタヒープとまったく同じ発想です。

### 21.2.2 フレーム分割方式

**リングバッファには 2 つの実装方針があります。**

| 方式 | 仕組み |
|---|---|
| **A. フレーム分割** | 領域をフレーム数で割り、フレームごとに巻き戻す |
| B. 真のリング | フェンス値で「解放済みの位置」を追跡し、循環させる |

**B のほうがメモリ効率は良いのですが、実装が複雑です。** 割り当て位置と完了位置の追跡、折り返し時の処理、枯渇時の待機。**バグを入れやすい部分です。**

**本書は A を採ります。** 第12章のコマンドアロケータ、本章のデスクリプタと、**すべて同じパターンになる**のが利点です。読者が覚えることが 1 つで済みます。

```
アップロードバッファ(たとえば 16MB)
┌────────────┬────────────┬────────────┐
│  frame 0   │  frame 1   │  frame 2   │
│ ← 前から詰める          │            │
└────────────┴────────────┴────────────┘
```

### 21.2.3 実装

```cpp
// src/Graphics/UploadRingBuffer.h
#pragma once
#include "std_import.h"
#include "Core/Error.h"

namespace Graphics
{
    //-----------------------------------------------------------
    // 確保した領域への参照。
    //-----------------------------------------------------------
    struct UploadAllocation
    {
        std::byte*                cpuAddress = nullptr;
        D3D12_GPU_VIRTUAL_ADDRESS gpuAddress = 0;
        UINT64                    offset     = 0;
        UINT64                    size       = 0;

        [[nodiscard]] bool IsValid() const noexcept
        {
            return cpuAddress != nullptr;
        }

        // 型つきで書き込む
        template <typename T>
        void Write(const T& value) noexcept
        {
            D3D_ASSERT(sizeof(T) <= size);
            std::memcpy(cpuAddress, &value, sizeof(T));
        }
    };

    class UploadRingBuffer
    {
    public:
        Core::Status Initialize(ID3D12Device* device,
                                UINT64 bytesPerFrame,
                                std::wstring_view name);
        void Shutdown();

        void BeginFrame(UINT frameIndex);

        // 定数バッファ用(256 整列)
        [[nodiscard]] UploadAllocation AllocateConstants(UINT64 size);

        // 任意のアラインメントで確保
        [[nodiscard]] UploadAllocation Allocate(UINT64 size, UINT64 alignment);

        UINT64 UsedThisFrame() const noexcept { return m_offsetInFrame; }
        UINT64 CapacityPerFrame() const noexcept { return m_bytesPerFrame; }

    private:
        Microsoft::WRL::ComPtr<ID3D12Resource> m_buffer;
        std::byte*                m_mappedBase = nullptr;
        D3D12_GPU_VIRTUAL_ADDRESS m_gpuBase    = 0;

        UINT64 m_bytesPerFrame  = 0;
        UINT64 m_frameBase      = 0;
        UINT64 m_offsetInFrame  = 0;
        UINT64 m_peakUsage      = 0;   // 最大使用量を記録する
    };
}
```

```cpp
UploadAllocation UploadRingBuffer::Allocate(UINT64 size, UINT64 alignment)
{
    const UINT64 aligned = AlignUp(m_offsetInFrame, alignment);

    if (aligned + size > m_bytesPerFrame)
    {
        LOG_ERROR(L"upload buffer exhausted: need {} bytes, {} available",
                  size, m_bytesPerFrame - aligned);
        D3D_ASSERT_MSG(false, L"アップロードバッファが不足しました");
        return {};
    }

    const UINT64 absolute = m_frameBase + aligned;

    UploadAllocation allocation{};
    allocation.cpuAddress = m_mappedBase + absolute;
    allocation.gpuAddress = m_gpuBase + absolute;
    allocation.offset     = absolute;
    allocation.size       = size;

    m_offsetInFrame = aligned + size;
    m_peakUsage     = std::max(m_peakUsage, m_offsetInFrame);

    return allocation;
}

UploadAllocation UploadRingBuffer::AllocateConstants(UINT64 size)
{
    return Allocate(size, D3D12_CONSTANT_BUFFER_DATA_PLACEMENT_ALIGNMENT);
}
```

**`m_peakUsage` を記録している**のは実用的な工夫です。終了時にログへ出せば、**バッファサイズが適切かどうかが分かります。**

```cpp
void UploadRingBuffer::Shutdown()
{
    LOG_INFO(L"upload buffer peak usage: {} / {} bytes ({:.1f}%)",
             m_peakUsage, m_bytesPerFrame,
             100.0 * m_peakUsage / m_bytesPerFrame);
    // ...
}
```

**枯渇したら止める、という設計も意図的です。** 黙って `nullptr` を返すと、描画が静かに壊れます。**サイズが足りないなら、設定を増やすべきです。**

### 21.2.4 第18章の定数バッファを置き換える

**これまでのコードが、こうなります。**

```cpp
// 第18章:固定領域に書く
std::memcpy(m_cbvMapped + frameIndex * kSceneConstantsSize,
            &constants, sizeof(constants));
```

```cpp
// 第21章:リングバッファから借りる
const auto allocation = m_uploadBuffer.AllocateConstants(sizeof(SceneConstants));
allocation.Write(constants);
```

**そして、CBV を作る場所も変わります。**

第18章では初期化時に 3 個の CBV を作り、フレームごとに使い分けていました。**今は毎フレーム、一時デスクリプタとして作ります。**

```cpp
const auto handle = m_descriptorHeap.Dynamic().Allocate();

D3D12_CONSTANT_BUFFER_VIEW_DESC cbvDesc{};
cbvDesc.BufferLocation = allocation.gpuAddress;
cbvDesc.SizeInBytes    = static_cast<UINT>(
    AlignUp(sizeof(SceneConstants),
            D3D12_CONSTANT_BUFFER_DATA_PLACEMENT_ALIGNMENT));

m_device->CreateConstantBufferView(&cbvDesc, handle.cpu);

m_commandList->SetGraphicsRootDescriptorTable(0, handle.gpu);
```

**「毎フレーム CBV を作るのは無駄では?」と思うかもしれません。**

`CreateConstantBufferView` は、**ヒープのメモリに数十バイト書き込むだけ**です。リソース生成とは比べ物になりません。**数百個作っても問題になりません。**

> **ルートディスクリプタなら CBV すら不要**
>
> 第18章 18.4.5 節のコラムで触れた通り、ルートディスクリプタを使えばこうなります。
>
> ```cpp
> m_commandList->SetGraphicsRootConstantBufferView(0, allocation.gpuAddress);
> ```
>
> **デスクリプタの割り当ても CBV の生成も要りません。** リングバッファと組み合わせると、これが最も単純です。
>
> 第25章で複数オブジェクトを描くとき、**オブジェクトごとの定数はルートディスクリプタ、テクスチャはテーブル**という使い分けを検討します。

---

## 21.3 Resizable BAR と GPU Upload Heaps

### 21.3.1 従来の制約

**第15章 15.2.1 節で見た表を思い出してください。**

| ヒープ | CPU から | GPU から |
|---|---|---|
| DEFAULT | **書けない** | 速い |
| UPLOAD | 書ける | **遅い** |

**この二択が、これまでのすべての設計を決めていました。** DEFAULT に置きたければ転送が必要で(第16章)、CPU から書きたければ遅いメモリを使うしかありませんでした。

**理由は PCI Express の仕様にあります。** CPU が GPU のメモリを直接見られる窓(BAR = Base Address Register)が、**伝統的に 256MB に制限されていました。** VRAM が 12GB あっても、CPU からは 256MB しか見えなかったのです。

### 21.3.2 Resizable BAR

**この制限を取り払うのが Resizable BAR です。**

VRAM 全体を CPU のアドレス空間にマップできるようになります。**つまり、CPU が VRAM に直接書き込めます。**

有効にするには 3 つの条件が要ります。

| 条件 | 確認方法 |
|---|---|
| マザーボードの UEFI 設定 | 「Resizable BAR」「Above 4G Decoding」を有効に |
| GPU | Ampere 世代以降が確実(第2章 2.1.2 節) |
| ドライバ | 対応版 |

**NVIDIA コントロールパネルの「システム情報」で確認できます。**

### 21.3.3 `D3D12_HEAP_TYPE_GPU_UPLOAD`

**Agility SDK 1.613 以降で追加されたヒープ種別です。**

```cpp
D3D12_HEAP_TYPE_GPU_UPLOAD = 5
```

| ヒープ | 置かれる場所 | CPU から | GPU から |
|---|---|---|---|
| UPLOAD | システムメモリ | 書ける | 遅い |
| DEFAULT | VRAM | 書けない | 速い |
| **GPU_UPLOAD** | **VRAM** | **書ける** | **速い** |

**両方の利点を持ちます。** 転送が不要になり、しかも読み取りが速くなります。

**対応の確認は必須です。**

```cpp
D3D12_FEATURE_DATA_D3D12_OPTIONS16 options16{};
if (QueryFeature(device, D3D12_FEATURE_D3D12_OPTIONS16, options16))
{
    m_caps.gpuUploadHeap = options16.GPUUploadHeapSupported;
}
```

**第7章 7.5.5 節の `DeviceCaps` に、既に用意してあります。** ここで使います。

### 21.3.4 使いどころ

**すべてを GPU_UPLOAD にすればよい、というものではありません。**

| データ | 適したヒープ | 理由 |
|---|---|---|
| モデルの頂点・インデックス | **DEFAULT** | 一度書いたら変わらない。転送のコストは初期化時のみ |
| テクスチャ | **DEFAULT** | 同上 |
| 毎フレーム更新する定数 | **GPU_UPLOAD**(あれば) | 転送が消える。GPU の読み取りも速い |
| 毎フレーム更新する頂点(パーティクルなど) | **GPU_UPLOAD**(あれば) | 同上 |

**VRAM は有限です。** GPU_UPLOAD は VRAM を消費するので、**動かないデータまでそこに置くと圧迫します。**

**本書の適用先は、リングバッファです。**

```cpp
Core::Status UploadRingBuffer::Initialize(
    ID3D12Device* device, UINT64 bytesPerFrame,
    bool preferGpuUpload, std::wstring_view name)
{
    const D3D12_HEAP_TYPE heapType = preferGpuUpload
        ? D3D12_HEAP_TYPE_GPU_UPLOAD
        : D3D12_HEAP_TYPE_UPLOAD;

    const auto heapProps = MakeHeapProperties(heapType);
    const auto desc = MakeBufferDesc(bytesPerFrame * kBackBufferCount);

    // GPU_UPLOAD でも初期状態は GENERIC_READ でよい
    HRESULT hr = device->CreateCommittedResource(
        &heapProps, D3D12_HEAP_FLAG_NONE, &desc,
        D3D12_RESOURCE_STATE_GENERIC_READ, nullptr,
        IID_PPV_ARGS(&m_buffer));

    //--- 失敗したら通常の UPLOAD へ落とす ---
    if (FAILED(hr) && preferGpuUpload)
    {
        LOG_WARN(L"GPU_UPLOAD heap failed, falling back to UPLOAD");

        const auto fallback = MakeHeapProperties(D3D12_HEAP_TYPE_UPLOAD);
        hr = device->CreateCommittedResource(
            &fallback, D3D12_HEAP_FLAG_NONE, &desc,
            D3D12_RESOURCE_STATE_GENERIC_READ, nullptr,
            IID_PPV_ARGS(&m_buffer));
    }
    HR_TRY(hr);

    LOG_INFO(L"upload buffer: {} ({} bytes/frame)",
             (heapType == D3D12_HEAP_TYPE_GPU_UPLOAD) ? L"GPU_UPLOAD" : L"UPLOAD",
             bytesPerFrame);
    // ...
}
```

**フォールバックを入れている**のが実用上の要点です。対応していない環境で起動しないのは行き過ぎです。**第7章 7.3.1 節で DXGI のデバッグフラグにフォールバックを入れたのと同じ判断です。**

> **書き込み方の注意は変わらない**
>
> GPU_UPLOAD でも、**ライトコンバインメモリである点は同じ**です(第15章 15.3.2 節)。
>
> - 書くだけ。読まない
> - `memcpy` で一気に書く
> - マップした領域を計算用に使わない
>
> **VRAM に直接書けるようになっただけで、CPU 側から見た性質は変わりません。**

---

## 21.4 リソースの寿命と遅延解放

### 21.4.1 これまでの対処

**「GPU がまだ使っているかもしれない」問題は、繰り返し出てきました。**

| 章 | 場面 | 対処 |
|---|---|---|
| 第10章 | 終了時 | `WaitForGpuIdle` |
| 第12章 | リサイズ時 | `WaitForGpuIdle` |
| 第16章 | 中間バッファ | `WaitForGpuIdle` |

**すべて「全部待つ」で解決してきました。** 初期化時や終了時なら、それで構いません。

**しかし、実行中にリソースを破棄したくなったら?**

第23章でモデルを差し替える、第26章でウィンドウをリサイズしてレンダーターゲットを作り直す —— **そのたびに GPU を全部待つのは、目に見えてカクつきます。**

### 21.4.2 遅延解放キュー

**発想は単純です。「今のフェンス値を覚えて、そこまで進んだら解放する」。**

```cpp
// src/Graphics/DeferredReleaseQueue.h
#pragma once
#include "std_import.h"

namespace Graphics
{
    class DeferredReleaseQueue
    {
    public:
        // 破棄したいリソースを預ける
        void Enqueue(Microsoft::WRL::ComPtr<ID3D12Object> object,
                     std::uint64_t fenceValue);

        // 完了したものを解放する。毎フレーム呼ぶ
        void Collect(std::uint64_t completedValue);

        // 全部解放する。終了時に GPU を待ってから呼ぶ
        void Flush();

        std::size_t PendingCount() const noexcept { return m_pending.size(); }

    private:
        struct Entry
        {
            Microsoft::WRL::ComPtr<ID3D12Object> object;
            std::uint64_t fenceValue = 0;
        };
        std::vector<Entry> m_pending;
    };
}
```

```cpp
void DeferredReleaseQueue::Enqueue(ComPtr<ID3D12Object> object,
                                   std::uint64_t fenceValue)
{
    if (!object) return;
    m_pending.push_back({ std::move(object), fenceValue });
}

void DeferredReleaseQueue::Collect(std::uint64_t completedValue)
{
    // 完了したものを末尾へ寄せて、まとめて削除する
    const auto removed = std::ranges::remove_if(m_pending,
        [completedValue](const Entry& e)
        {
            return e.fenceValue <= completedValue;
        });

    if (!removed.empty())
    {
        LOG_TRACE(L"released {} deferred object(s)", removed.size());
        m_pending.erase(removed.begin(), removed.end());
    }
}
```

**`std::ranges::remove_if` は C++20 の形です。** 削除された範囲を返すので、`erase` と組み合わせます。

**`ComPtr` に預けているので、`erase` した瞬間に `Release` が呼ばれます。** 明示的な解放処理は不要です。

### 21.4.3 フレームループに組み込む

```cpp
Core::Status Renderer::RenderFrame()
{
    const UINT index = m_swapChain.CurrentIndex();
    FrameResource& frame = m_frames[index];

    //--- ② スロットの前回の作業を待つ(第12章)---
    if (auto r = m_fence.Wait(frame.fenceValue); !r) { return r; }

    //--- ★ 完了したリソースを解放する ★ ---
    m_releaseQueue.Collect(m_fence.Get()->GetCompletedValue());

    //--- ★ 一時領域を巻き戻す ★ ---
    m_descriptorHeap.BeginFrame(index);
    m_uploadBuffer.BeginFrame(index);

    //--- ③ 記録の準備 ---
    HR_TRY(frame.allocator->Reset());
    // ...
}
```

**3 行増えただけです。** そして、この 3 行で「フレーム単位で巻き戻す」仕組みがすべて揃いました。

使う側はこうなります。

```cpp
// 古いテクスチャを破棄したい
m_releaseQueue.Enqueue(oldTexture, m_fence.LastSignaledValue());
oldTexture.Reset();     // ローカルの参照を手放す

// この時点では、まだ GPU が使っているかもしれない。
// キューが参照を持っているので解放されない。
// 数フレーム後、Collect() が安全に解放する。
```

**`WaitForGpuIdle` が不要になりました。** カクつきません。

### 21.4.4 終了時の処理

```cpp
void Renderer::Shutdown()
{
    //--- ① GPU を待つ ---
    if (auto r = WaitForGpuIdle(m_queue.Get(), m_fence); !r)
    {
        Core::ReportError(r.error());
    }

    //--- ② 遅延解放キューを空にする ---
    m_releaseQueue.Flush();

    //--- ③ 各オブジェクトを解放 ---
    m_uploadBuffer.Shutdown();
    m_descriptorHeap.Shutdown();
    // ...
}
```

**順序が重要です。** GPU を待つ前に `Flush` すると、使用中のリソースを解放することになります。

---

## 21.5 リークをゼロにする

### 21.5.1 `ReportLiveDeviceObjects`

**D3D12 には、生き残っているオブジェクトを列挙する機能があります。**

```cpp
void ReportLiveObjects()
{
    ComPtr<IDXGIDebug1> dxgiDebug;
    if (SUCCEEDED(::DXGIGetDebugInterface1(0, IID_PPV_ARGS(&dxgiDebug))))
    {
        LOG_INFO(L"--- live objects ---");
        dxgiDebug->ReportLiveObjects(
            DXGI_DEBUG_ALL,
            static_cast<DXGI_DEBUG_RLO_FLAGS>(
                DXGI_DEBUG_RLO_SUMMARY | DXGI_DEBUG_RLO_IGNORE_INTERNAL));
    }
}
```

**`DXGI_DEBUG_RLO_IGNORE_INTERNAL` が重要です。** これを付けないと、D3D12 が内部的に保持しているオブジェクトまで列挙され、**「リークしていないのにリークに見える」**ことになります。

**呼ぶ場所は `main` の最後、すべてが破棄された後です。**

```cpp
int WINAPI wWinMain(...)
{
    {
        Graphics::GraphicsDevice device;
        Graphics::Renderer       renderer;
        // ... 中略 ...
        renderer.Shutdown();
        device.Shutdown();
    }   // ← ここでデストラクタが走る

#if defined(_DEBUG)
    ReportLiveObjects();
#endif
    return 0;
}
```

**スコープで囲む**のがコツです。デストラクタが走った後でなければ意味がありません。

### 21.5.2 出力の読み方

**リークがない場合、「出力」ウィンドウにこう出ます。**

```
DXGI WARNING: Live Producer at 0x..., Refcount: 0.
```

**`Refcount: 0` なら問題ありません。**

**リークがある場合は、こうなります。**

```
D3D12 WARNING: Live ID3D12Resource at 0x000001F2A4B3C120, Refcount: 1,
  Name: "CubeVertexBuffer"
D3D12 WARNING: Live ID3D12PipelineState at 0x000001F2A4B40080, Refcount: 1,
  Name: "TrianglePSO"
```

**`Name:` が出ていることに注目してください。**

**第6章 6.5 節で名前付けを習慣にした成果が、ここで最大限に発揮されます。** 名前がなければ `Live ID3D12Resource at 0x...` としか出ず、**どのリソースか特定できません。**

### 21.5.3 よくあるリークの原因

| 原因 | 対処 |
|---|---|
| **デバイスより先に他を解放していない** | メンバの宣言順(第6章 6.2.6 節) |
| `GetAddressOf()` を使った | `&` を使う(第6章 6.2.3 節) |
| 遅延解放キューを `Flush` していない | 21.4.4 |
| ローカル変数の `ComPtr` が残っている | スコープを閉じる |
| デバッグレイヤーの `InfoQueue` | **これは正常**(内部で保持される) |

**`ID3D12InfoQueue` が残るのは正常です。** 第7章 7.6.2 節で登録したコールバックを解除していれば、それ以上は増えません。

---

## ✅ 本章のゴール:リークゼロ

### Step 1:使用状況をログに出す

```cpp
void DescriptorHeap::LogUsage() const
{
    LOG_INFO(L"descriptors: {} / {} static",
             m_static.Used(), m_static.Capacity());
}
```

実行時の出力です。

```
[Info ] DescriptorHeap.cpp(52): descriptor heap: 8192 total (2048 static + 2048 x 3 dynamic)
[Info ] UploadRingBuffer.cpp(48): upload buffer: GPU_UPLOAD (1048576 bytes/frame)
[Info ] Renderer.cpp(210): descriptors: 1 / 2048 static
```

**現時点では、永続デスクリプタは SRV の 1 個だけです。** 第23章でモデルを読み込むと増えます。

### Step 2:終了時の統計を確認する

```
[Info ] UploadRingBuffer.cpp(88): upload buffer peak usage: 256 / 1048576 bytes (0.0%)
[Info ] Renderer.cpp(285): deferred release queue: 0 pending
```

**使用率が 0.0% です。** 定数バッファ 1 つしか使っていないので当然ですが、**第25章で 100 個のオブジェクトを描くと、目に見えて増えます。**

### Step 3:リークがないことを確認する

```
[Info ] main.cpp(142): --- live objects ---
DXGI WARNING: Live Producer at 0x00007FF8..., Refcount: 0.
```

**`Refcount: 0` だけなら成功です。**

### Step 4:わざとリークさせる

**Step 3 が本当に機能しているかを確かめます。**

`Renderer::Shutdown` で、何か 1 つ解放しないようにします。

```cpp
void Renderer::Shutdown()
{
    // ...
    // m_pso.Reset();   ← コメントアウト
}
```

**`Renderer` のデストラクタで結局解放されるので、`ComPtr` をリークさせるには工夫が要ります。** 手軽なのは、意図的に `AddRef` することです。

```cpp
m_pso->AddRef();   // ❌ わざと参照カウントを増やす
```

```
D3D12 WARNING: Live ID3D12PipelineState at 0x000001F2A4B40080,
  Refcount: 1, Name: "TrianglePSO"
```

**名前が出ることを確認してください。** これが第6章の投資の回収です。

**確認したら元に戻してください。**

### Step 5:デスクリプタを枯渇させる

```cpp
Config config{};
config.staticCapacity = 1;    // わざと小さく
```

SRV を 2 個作ろうとすると、アサートで止まります。

```
[Error] DescriptorAllocator.cpp(38): descriptor heap exhausted (1 used)
[Fatal] DescriptorAllocator.cpp(39): assertion failed: false (デスクリプタヒープが枯渇しました)
```

**黙って無効なハンドルを返さないので、原因が即座に分かります。**

**確認したら元に戻してください。**

### Step 6:GPU_UPLOAD の効果を確認する(任意)

**対応環境なら、ログに `GPU_UPLOAD` と出ているはずです。**

```
[Info ] UploadRingBuffer.cpp(48): upload buffer: GPU_UPLOAD (1048576 bytes/frame)
```

`UPLOAD` と出ている場合、原因は次のいずれかです。

- Resizable BAR が UEFI で無効
- GPU が Turing 世代(Ampere 以降を推奨)
- Agility SDK が 1.613 未満

**現時点では性能差は測れません。** 定数バッファ 1 つでは誤差に埋もれます。**第25章で大量のオブジェクトを描くようになったら、第38章の測定手法で比較できます。**

---

### 本章の達成状態

- [ ] ヒープを永続領域と一時領域に分割した
- [ ] 永続アロケータにフリーリストを実装した
- [ ] 一時アロケータをフレームごとに巻き戻している
- [ ] `DescriptorHandle` に `index` を持たせた(第33章用)
- [ ] アップロードリングバッファを実装した
- [ ] ピーク使用量を記録している
- [ ] 第18章の固定定数バッファを置き換えた
- [ ] `GPUUploadHeapSupported` を確認し、フォールバックを入れた
- [ ] 遅延解放キューを実装した
- [ ] フレームループで `Collect` / `BeginFrame` を呼んでいる
- [ ] 終了時に GPU を待ってから `Flush` している
- [ ] `ReportLiveObjects` を `IGNORE_INTERNAL` 付きで呼んでいる
- [ ] **リークがゼロである**
- [ ] Step 4 で名前が表示されることを確認した

---

## トラブルシューティング

| 症状 | 原因 | 対処 |
|---|---|---|
| デスクリプタが枯渇する | 容量不足、または `Free` 漏れ | 21.1.3 |
| 一時デスクリプタが上書きされる | `BeginFrame` を呼んでいない | 21.4.3 |
| アップロードバッファが不足 | `bytesPerFrame` が小さい | ピーク使用量を見て調整 |
| 定数の値が壊れる | フレーム分割ができていない | 21.2.2 |
| `GPU_UPLOAD` で生成に失敗 | 非対応環境 | フォールバック(21.3.4) |
| リソース破棄でクラッシュ | GPU がまだ使用中 | 遅延解放キューへ(21.4.2) |
| 終了時にリーク報告 | メンバの宣言順 | デバイスを最初に宣言(第6章 6.2.6) |
| 同上 | `Flush` を呼んでいない | 21.4.4 |
| 内部オブジェクトが大量に出る | `IGNORE_INTERNAL` 忘れ | 21.5.1 |
| リーク元が分からない | 名前を付けていない | 第6章 6.5 節 |

---

## まとめ

**1. すべて「フレームごとに巻き戻す」で統一した。**
コマンドアロケータ(第12章)、デスクリプタ、アップロードバッファ。**同じパターンなので、覚えることが 1 つで済みます。**

**2. 永続と一時を分ける。**
寿命の違うものを同じアロケータで管理すると、断片化の管理が必要になります。**領域を分ければ、一時領域は「巻き戻すだけ」で済みます。**

**3. 枯渇したら止める。**
黙って無効なハンドルを返すと、描画が静かに壊れます。**アサートで止めれば、原因が即座に分かります。**

**4. GPU_UPLOAD は VRAM に直接書ける。**
Resizable BAR の恩恵です。**ただし VRAM は有限なので、毎フレーム更新するデータに限って使います。**

**5. 遅延解放で `WaitForGpuIdle` が不要になる。**
「今のフェンス値を覚えて、そこまで進んだら解放する」。**実行中のリソース破棄がカクつかなくなります。**

**6. 名前がなければリークは追えない。**
`ReportLiveObjects` の出力に `Name:` が出るかどうかは、第6章からの習慣で決まります。**第31章の Aftermath でも同じことが起こります。**

---

## 第2部を終えて

第1部で三角形を出し、第2部で「3D らしさ」を積み上げました。

| 章 | 得たもの |
|---|---|
| 第16章 | インデックス描画、DEFAULT ヒープへの転送 |
| 第17章 | 数学ライブラリと、座標系の規約 |
| 第18章 | 定数バッファ、ルートパラメータの選択 |
| 第19章 | 深度テスト、Reversed-Z |
| 第20章 | テクスチャ、サブリソース転送 |
| 第21章 | リソース管理の設計 |

**自作ヘルパーは 11 個になりました。**

```
OffsetHandle (CPU / GPU)             第11章
MakeTransitionBarrier                第11章
DefaultRasterizerDesc                第14章
DefaultBlendDesc                     第14章
DefaultDepthStencilDesc              第14章
DefaultGraphicsPipelineStateDesc     第14章
MakeHeapProperties                   第15章
MakeBufferDesc                       第15章
AlignUp                              第18章
MakeTexture2DDesc                    第19章
```

**そして、`UpdateSubresources()` を自分で書きました**(第20章)。`d3dx12.h` を使わない代償は、これでほぼ出揃いました。

第3部からは、これらの土台の上に「実用的なレンダラ」を組み上げます。カメラ操作、モデル読み込み、ライティング、影、ポストエフェクト。**次章では、まず動き回れるようにします。**

---

## 参考リンク

| 内容 | URL |
|---|---|
| デスクリプタ ヒープ | https://learn.microsoft.com/ja-jp/windows/win32/direct3d12/descriptor-heaps |
| リソース バインディングの階層 | https://learn.microsoft.com/ja-jp/windows/win32/direct3d12/hardware-support |
| GPU Upload Heaps | https://devblogs.microsoft.com/directx/d3d12-gpu-upload-heaps/ |
| `IDXGIDebug::ReportLiveObjects` | https://learn.microsoft.com/ja-jp/windows/win32/api/dxgidebug/nf-dxgidebug-idxgidebug-reportliveobjects |
| Resizable BAR について | https://www.nvidia.com/ja-jp/geforce/news/geforce-rtx-30-series-resizable-bar-support/ |
