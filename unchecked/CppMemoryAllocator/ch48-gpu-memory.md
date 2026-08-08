# 第48章 GPU メモリという別世界

---

## この章のゴール

ここまで扱ってきたのは、**CPU が読み書きするメモリ** でした。**GPU 側のメモリには、まったく違う制約があります。**

- **CPU から自由に読み書きできない**(ヘッダを置けない)
- **アラインメント要求が 64 KB、4 MB といった単位**
- **解放しても、GPU がまだ使っているかもしれない**
- **確保が桁違いに高価**

この章では、これらに対応したアロケーターを設計します。

- CPU メモリとの5つの違いを整理する
- Direct3D 12 のメモリモデル(ヒープと配置リソース)
- **第26章のバディが、なぜ GPU に最適なのか** ——3条件が揃う
- **第26章の実装を、GPU 用に1か所だけ変える**(重要)
- **フェンスによる遅延解放**(第43章の回収)
- アップロード用のリングバッファ

**実装は擬似的なものです。** 特定の API の詳細ではなく、**構造** を扱います。

---

## 48.1 CPU メモリとの5つの違い

### ① CPU から自由に読み書きできない

**GPU 専用のメモリ(VRAM)は、CPU からは見えません。**

```cpp
// これができない
struct Header { std::size_t size; Header* next; };
Header* h = reinterpret_cast<Header*>(gpuMemory);
h->size = 1024;      // ← 書き込めない
```

**第23章で作ったヘッダ方式が、まるごと使えません。**

**第21章の侵入的フリーリストも使えません。** 空きブロックの中にポインタを書けないからです。

> **すべての管理情報を、CPU 側に持つ必要があります。**

### ② アラインメント要求が大きい

```
通常のテクスチャ・バッファ : 64 KB 境界
マルチサンプルのテクスチャ : 4 MB 境界
定数バッファ              : 256 バイト境界
```

**第6章で扱った 16 バイトとは、桁が違います。**

**しかも、すべて 2 の冪です。**

### ③ 解放のタイミングが遅れる

```cpp
DrawMesh(vertexBuffer);      // GPU に描画を指示
FreeGpuMemory(vertexBuffer); // ← GPU はまだ描いていない!
```

**第43章の 43.3 節で見た問題です。** CPU と GPU は非同期に動きます。

**「いつ解放してよいか」を、GPU に聞く必要があります。**

### ④ ヒープの種類が複数ある

| 種類 | 場所 | CPU からの書き込み | GPU からの読み込み |
|---|---|---|---|
| **DEFAULT** | VRAM | 不可 | **最速** |
| **UPLOAD** | システムメモリ | **可能** | 遅い(PCIe 経由) |
| **READBACK** | システムメモリ | 読み取り可 | 書き込み用 |

**「CPU が書いて、GPU が読む」データは、UPLOAD に置いてから DEFAULT へコピーします。**

**それぞれに別のアロケーターが必要です。**

### ⑤ 確保が桁違いに高価

```
                              典型的なオーダー(環境依存)
CPU:  ga::Bump::Allocate           2 ns
CPU:  ::operator new              18 ns
GPU:  配置リソースの作成          数 µs
GPU:  ヒープの作成                数十〜数百 µs
```

**ドライバの呼び出しと、カーネルへの遷移が発生します。**

> **だから「大きく確保して、自分で切り分ける」しかありません。**
>
> **サブアロケーション** と呼ばれます。**本書が第2章からやってきたことと、まったく同じ発想です。**

---

## 48.2 Direct3D 12 のメモリモデル

**構造だけ、簡単に整理します。**

### ヒープとリソース

```
┌──────────────────────────────────────────┐
│  ID3D12Heap(大きなメモリの塊、例 64 MB)  │
│  ┌──────┬──────┬──────────┬────────────┐│
│  │リソースA│リソースB│  リソースC │   空き    ││
│  └──────┴──────┴──────────┴────────────┘│
└──────────────────────────────────────────┘
      ↑ 配置リソース(Placed Resource)
```

**ヒープの中の任意のオフセットに、リソースを「配置」できます。**

**これが、サブアロケーションを可能にしている仕組みです。**

### 3つのリソースの作り方

| 方式 | 特徴 |
|---|---|
| **Committed** | リソースごとにヒープを作る。**楽だが高価** |
| **Placed** | 既存のヒープの中に配置する。**サブアロケーション向き** |
| **Reserved** | タイル単位で物理メモリを割り当てる(スパース) |

**Committed が最も単純です。** しかし、リソースを 1000 個作れば、ヒープが 1000 個できます。

```
                          典型的なオーダー
Committed × 1000         数十〜数百 ms
Placed × 1000(1ヒープ)  数 ms
```

**桁違いです。** 実務では Placed が基本になります。

### アラインメントの定数

```cpp
D3D12_DEFAULT_RESOURCE_PLACEMENT_ALIGNMENT      // 64 KB
D3D12_DEFAULT_MSAA_RESOURCE_PLACEMENT_ALIGNMENT // 4 MB
D3D12_CONSTANT_BUFFER_DATA_PLACEMENT_ALIGNMENT  // 256 バイト
```

**Vulkan にも、対応する概念があります。** `VkDeviceMemory` がヒープに、`vkBindBufferMemory` が配置に相当します。

**構造は同じです。** 以降、API に依存しない形で進めます。

---

## 48.3 なぜバディなのか

**第26章の 26.9 節で、3つの条件を挙げました。**

> **① 管理情報をブロックに書き込めない**
> **② 大きなアラインメントが必要で、しかも 2 の冪**
> **③ 要求サイズがもともと 2 の冪に近い**

**GPU メモリでは、3つとも揃います。**

### ①:ヘッダを置けない

**48.1 節の制約①そのものです。**

**バディは、すべての管理情報をビットツリーとして別に持ちます。** ブロックの中には何も書きません。

### ②:自然整列がタダで手に入る

**第26章 26.5 節で触れた性質です。**

> `base_` が容量と同じ大きさに整列していれば、**すべてのブロックが自分のサイズに整列** します。

```
64 KB のブロック → 必ず 64 KB 境界
4 MB のブロック  → 必ず 4 MB 境界
```

**D3D12 が要求するアラインメントが、構造的に満たされます。** パディングが一切要りません。

**第23章のフリーリスト方式で 4 MB 境界を満たそうとすると、最悪 4 MB のパディングが発生します。**

### ③:サイズが 2 の冪に近い

```
テクスチャ: 256×256、512×512、1024×1024
ミップマップ: 半分ずつ縮小
バッファ: ページ単位に丸められる
```

**第26章 26.7 節で測った「負荷 P(2の冪ぴったり)」に近い状況です。** 内部断片化がほとんど発生しません。

---

## 48.4 第26章の実装を、1か所だけ変える

**ここが、この章で最も重要な実装上の指摘です。**

**第26章の `Buddy` は、空きブロックの中にリンクを書いていました。**

```cpp
    void PushFree(std::byte* p, std::size_t order) noexcept
    {
        Link* l = reinterpret_cast<Link*>(p);      // ← ブロックの中に書き込む
        l->prev = nullptr;
        l->next = freeLists_[order];
        // ...
    }
```

**GPU メモリでは、これができません。**

### 解決:リンクを CPU 側の配列に移す

**ブロック番号でリンクを表現します。**

```cpp
// ga/gpu/VirtualBuddy.h
#pragma once

#include "ga/Core.h"

#include <bit>
#include <cstdint>
#include <vector>

namespace ga::gpu
{
    // 実際のメモリを持たず、オフセット空間だけを管理するバディアロケーター。
    // すべての管理情報が CPU 側にあるので、GPU メモリにも使える。
    class VirtualBuddy
    {
    public:
        static constexpr std::uint64_t kInvalid = ~std::uint64_t{ 0 };

        VirtualBuddy(std::uint64_t capacity, std::uint64_t minBlock)
            : capacity_(capacity)
            , minBlock_(minBlock)
        {
            assert(std::has_single_bit(capacity));
            assert(std::has_single_bit(minBlock));

            minShift_ = static_cast<std::uint32_t>(std::countr_zero(minBlock));
            maxOrder_ = static_cast<std::uint32_t>(std::bit_width(capacity)) - 1 - minShift_;

            const std::size_t nodeCount = std::size_t{ 2 } << maxOrder_;

            splitBits_.assign((nodeCount + 63) / 64, 0);
            freeBits_ .assign((nodeCount + 63) / 64, 0);

            // ★ リンクを CPU 側の配列で持つ(第26章との唯一の違い)
            const std::size_t leafCount = std::size_t{ 1 } << maxOrder_;
            nextFree_.assign(leafCount * 2, kNoNode);
            prevFree_.assign(leafCount * 2, kNoNode);

            heads_.assign(maxOrder_ + 1, kNoNode);

            PushFree(0, maxOrder_);
        }

        [[nodiscard]] std::uint64_t Allocate(std::uint64_t size) noexcept
        {
            if (size == 0) { return kInvalid; }

            const std::uint64_t need = std::bit_ceil(size < minBlock_ ? minBlock_ : size);
            if (need > capacity_) { return kInvalid; }

            const std::uint32_t order =
                static_cast<std::uint32_t>(std::bit_width(need)) - 1 - minShift_;

            std::uint32_t k = order;
            while (k <= maxOrder_ && heads_[k] == kNoNode) { ++k; }
            if (k > maxOrder_) { return kInvalid; }

            const std::uint32_t node = heads_[k];
            UnlinkFree(node, k);

            std::uint64_t offset = OffsetOf(node, k);

            while (k > order)
            {
                SetBit(splitBits_, node);
                --k;

                const std::uint64_t buddyOffset = offset + (minBlock_ << k);
                PushFree(NodeOf(buddyOffset, k), k);
            }

            return offset;
        }

        void Free(std::uint64_t offset) noexcept
        {
            // ビットツリーを降りて order を求める(第26章と同じ)
            std::uint32_t order = maxOrder_;
            std::uint32_t node  = 1;

            while (TestBit(splitBits_, node))
            {
                --order;
                node = node * 2 +
                       static_cast<std::uint32_t>((offset >> (minShift_ + order)) & 1);
            }

            // 相方と統合(第26章と同じ)
            while (order < maxOrder_)
            {
                const std::uint64_t size        = minBlock_ << order;
                const std::uint64_t buddyOffset = offset ^ size;
                const std::uint32_t buddyNode   = NodeOf(buddyOffset, order);

                if (!TestBit(freeBits_, buddyNode)) { break; }

                UnlinkFree(buddyNode, order);

                offset &= ~size;
                ++order;

                ClearBit(splitBits_, NodeOf(offset, order));
            }

            PushFree(NodeOf(offset, order), order);
        }

    private:
        static constexpr std::uint32_t kNoNode = 0xFFFF'FFFFu;

        std::uint32_t NodeOf(std::uint64_t offset, std::uint32_t order) const noexcept
        {
            const std::uint32_t level = maxOrder_ - order;
            return (std::uint32_t{ 1 } << level) +
                   static_cast<std::uint32_t>(offset >> (minShift_ + order));
        }

        std::uint64_t OffsetOf(std::uint32_t node, std::uint32_t order) const noexcept
        {
            const std::uint32_t level = maxOrder_ - order;
            const std::uint32_t indexInLevel = node - (std::uint32_t{ 1 } << level);
            return static_cast<std::uint64_t>(indexInLevel) << (minShift_ + order);
        }

        // ★ リンク操作が、すべて CPU 側の配列で完結する
        void PushFree(std::uint32_t node, std::uint32_t order) noexcept
        {
            prevFree_[node] = kNoNode;
            nextFree_[node] = heads_[order];

            if (heads_[order] != kNoNode) { prevFree_[heads_[order]] = node; }

            heads_[order] = node;
            SetBit(freeBits_, node);
        }

        void UnlinkFree(std::uint32_t node, std::uint32_t order) noexcept
        {
            const std::uint32_t p = prevFree_[node];
            const std::uint32_t n = nextFree_[node];

            if (p != kNoNode) { nextFree_[p] = n; }
            else              { heads_[order] = n; }

            if (n != kNoNode) { prevFree_[n] = p; }

            ClearBit(freeBits_, node);
        }

        static bool TestBit (const std::vector<std::uint64_t>& v, std::uint32_t i) noexcept
        { return (v[i >> 6] >> (i & 63)) & 1u; }

        static void SetBit  (std::vector<std::uint64_t>& v, std::uint32_t i) noexcept
        { v[i >> 6] |=  (std::uint64_t{1} << (i & 63)); }

        static void ClearBit(std::vector<std::uint64_t>& v, std::uint32_t i) noexcept
        { v[i >> 6] &= ~(std::uint64_t{1} << (i & 63)); }

        std::uint64_t capacity_ = 0;
        std::uint64_t minBlock_ = 0;
        std::uint32_t minShift_ = 0;
        std::uint32_t maxOrder_ = 0;

        std::vector<std::uint64_t> splitBits_;
        std::vector<std::uint64_t> freeBits_;
        std::vector<std::uint32_t> nextFree_;
        std::vector<std::uint32_t> prevFree_;
        std::vector<std::uint32_t> heads_;
    };
}
```

### 変わったのは、この1点だけ

```
第26章 : リンクを、空きブロックの中に置く
この章 : リンクを、CPU 側の配列に置く
```

**アルゴリズムは、まったく同じです。** XOR で相方を求め、ビットツリーで状態を持ち、分割と統合を対称に行う。

**代償:CPU 側のメモリが増えます。**

```
64 MB のヒープ、最小ブロック 64 KB:
  葉の数     = 1024
  ノード数   = 2048
  ビット列   = 2048 × 2 ビット = 512 バイト
  リンク配列 = 2048 × 2 × 4 バイト = 16 KB
  ──────────────────────────────────
  合計 ≈ 16.5 KB(GPU ヒープの 0.025%)
```

**無視できる量です。**

> **「メモリの中に管理情報を置けない」という制約は、こんなに単純に回避できます。**
>
> **第21章から使ってきた侵入的リストが、初めて使えない場面でした。** そして、代替手段は「番号で表現し直す」だけでした。

---

## 48.5 フェンスによる遅延解放

**48.1 節の制約③に対処します。**

### フェンスとは

**GPU の処理の進み具合を、番号で表すものです。**

```
CPU: コマンドを積む
CPU: 「ここまで終わったら 42 番」と指示(Signal)
CPU: 次のフレームへ
...
GPU: 処理が進む
GPU: 42 番まで完了 → フェンスの値が 42 になる
```

**CPU 側から「いまいくつまで終わったか」を読めます。**

### 遅延解放キュー

```cpp
// ga/gpu/DeferredFreeQueue.h
#pragma once

#include <cstdint>
#include <deque>

namespace ga::gpu
{
    struct Allocation
    {
        std::uint32_t heapIndex = 0;
        std::uint64_t offset    = 0;
        std::uint64_t size      = 0;
    };

    class DeferredFreeQueue
    {
    public:
        // 「このフェンス値に到達したら、解放してよい」
        void Enqueue(const Allocation& alloc, std::uint64_t fenceValue)
        {
            pending_.push_back(Pending{ alloc, fenceValue });
        }

        // GPU の完了状況を渡して、解放できるものを回収する
        template <class F>
        std::size_t Collect(std::uint64_t completedFence, F&& release)
        {
            std::size_t count = 0;

            while (!pending_.empty() && pending_.front().fence <= completedFence)
            {
                release(pending_.front().alloc);
                pending_.pop_front();
                ++count;
            }

            return count;
        }

        std::size_t PendingCount() const noexcept { return pending_.size(); }

        std::uint64_t PendingBytes() const noexcept
        {
            std::uint64_t total = 0;
            for (const auto& p : pending_) { total += p.alloc.size; }
            return total;
        }

    private:
        struct Pending
        {
            Allocation    alloc;
            std::uint64_t fence;
        };

        std::deque<Pending> pending_;
    };
}
```

**`std::deque` を使っているのは、フェンス値が単調増加するため、先頭から順に回収できるからです。** 探索が要りません。

### 使う

```cpp
void GpuAllocator::Free(const Allocation& alloc)
{
    // すぐには解放しない
    deferred_.Enqueue(alloc, currentFenceValue_);
}

void GpuAllocator::EndFrame(std::uint64_t completedFence)
{
    deferred_.Collect(completedFence, [this](const Allocation& a) {
        heaps_[a.heapIndex].buddy.Free(a.offset);      // ← ここで本当に解放
    });
}
```

### 「N フレーム後」方式との比較

**より単純な方法もあります。**

```cpp
// フレーム番号で管理する
if (frameNumber_ - alloc.frameFreed >= kFrameLatency)
{
    Release(alloc);
}
```

| | フェンス方式 | N フレーム方式 |
|---|---|---|
| 正確さ | **正確** | **推測** |
| 実装 | やや複雑 | 単純 |
| メモリの回収 | **最速** | 余分に待つ |
| 危険 | フェンスの取り違え | **N が足りないと破滅** |

**フェンス方式を推奨します。** 「N フレームで足りるはず」という推測は、負荷が高いときに外れます。

**第43章のダブルバッファでも、同じ選択がありました。** 面数を推測するより、フェンスで確認するほうが安全です。

---

## 48.6 アップロード用のリングバッファ

**48.1 節の制約④に対処します。**

### 用途

```
定数バッファ(行列、ライトのパラメータ)
動的な頂点データ(パーティクル、UI、デバッグ描画)
テクスチャの転送元
```

**どれも「毎フレーム、CPU が書いて、GPU が読む」データです。**

### 構造:フレームアロケーター + フェンス

**第43章のフレームアロケーターと、ほぼ同じです。**

```cpp
namespace ga::gpu
{
    class UploadRing
    {
    public:
        UploadRing(std::byte* mappedPtr, std::uint64_t capacity)
            : base_(mappedPtr), capacity_(capacity)
        {
        }

        struct Slice
        {
            std::byte*    cpuPtr = nullptr;
            std::uint64_t offset = 0;
            std::uint64_t size   = 0;
        };

        // 書き込み領域を切り出す
        [[nodiscard]] Slice Allocate(std::uint64_t size, std::uint64_t alignment)
        {
            const std::uint64_t aligned = AlignUpU64(head_, alignment);

            if (aligned + size > capacity_)
            {
                // 末尾で折り返す
                if (size > tail_) { return {}; }      // 一周して追いついた
                head_ = 0;
                return Allocate(size, alignment);
            }

            if (head_ < tail_ && aligned + size > tail_) { return {}; }   // 追いついた

            Slice s{ base_ + aligned, aligned, size };
            head_ = aligned + size;

            return s;
        }

        // フレームの終わりに、その時点の head を記録する
        void EndFrame(std::uint64_t fenceValue)
        {
            marks_.push_back(Mark{ head_, fenceValue });
        }

        // GPU が読み終わった分だけ、tail を進める
        void Collect(std::uint64_t completedFence)
        {
            while (!marks_.empty() && marks_.front().fence <= completedFence)
            {
                tail_ = marks_.front().head;
                marks_.pop_front();
            }
        }

    private:
        struct Mark { std::uint64_t head; std::uint64_t fence; };

        std::byte*    base_     = nullptr;
        std::uint64_t capacity_ = 0;
        std::uint64_t head_     = 0;     // 次に書く位置
        std::uint64_t tail_     = 0;     // GPU が読み終わった位置

        std::deque<Mark> marks_;
    };
}
```

**バンプアロケーターを、円環状にしたものです。**

```
[  読み終わった  ][ GPU が読んでいる ][ CPU が書いている ][  未使用  ]
 ↑tail                                ↑head
```

**`head` が `tail` に追いつくと、確保できません。** GPU の完了を待つか、バッファを大きくします。

### 定数バッファのアラインメント

```cpp
constexpr std::uint64_t kConstantBufferAlignment = 256;

auto slice = ring.Allocate(sizeof(SceneConstants), kConstantBufferAlignment);
```

**256 バイト境界が要求されます。** 小さな定数バッファ(たとえば 64 バイト)でも、**256 バイト消費します。**

**第6章で扱った内部断片化が、ここでは 4 倍という大きさで現れます。**

**対策:複数のオブジェクトの定数を、1つのバッファにまとめる。**

```cpp
struct alignas(256) PerObjectConstants { Matrix world; };     // 256 バイトに揃える
auto slice = ring.Allocate(sizeof(PerObjectConstants) * count, 256);
```

---

## 48.7 全体を組み立てる

```cpp
namespace ga::gpu
{
    class GpuAllocator
    {
    public:
        GpuAllocator(HeapBackend& backend, HeapType type, std::uint64_t heapSize)
            : backend_(&backend), type_(type), heapSize_(heapSize)
        {
        }

        [[nodiscard]] Allocation Allocate(std::uint64_t size, std::uint64_t alignment,
                                          MemoryTag tag = MemoryTag::General)
        {
            // ① 既存のヒープから探す
            for (std::size_t i = 0; i < heaps_.size(); ++i)
            {
                const std::uint64_t offset = heaps_[i].buddy.Allocate(size);
                if (offset != VirtualBuddy::kInvalid)
                {
                    RecordAllocation(tag, size);
                    return Allocation{ static_cast<std::uint32_t>(i), offset, size };
                }
            }

            // ② 足りなければ、新しいヒープを作る
            if (!AddHeap()) { return {}; }

            const std::uint64_t offset = heaps_.back().buddy.Allocate(size);
            if (offset == VirtualBuddy::kInvalid) { return {}; }

            RecordAllocation(tag, size);
            return Allocation{ static_cast<std::uint32_t>(heaps_.size() - 1), offset, size };
        }

        void Free(const Allocation& alloc, std::uint64_t fenceValue)
        {
            deferred_.Enqueue(alloc, fenceValue);
        }

        void Collect(std::uint64_t completedFence)
        {
            deferred_.Collect(completedFence, [this](const Allocation& a) {
                heaps_[a.heapIndex].buddy.Free(a.offset);
            });
        }

    private:
        struct HeapEntry
        {
            std::uint32_t backendIndex = 0;
            VirtualBuddy  buddy;
        };

        HeapBackend*           backend_;
        HeapType               type_;
        std::uint64_t          heapSize_;
        std::vector<HeapEntry> heaps_;
        DeferredFreeQueue      deferred_;
    };
}
```

### 第45章のハンドルと組み合わせる

```cpp
Handle<GpuBuffer> h = resourceManager.CreateBuffer(size);

// 使うとき
if (const GpuBuffer* buf = resourceManager.Resolve(h))
{
    commandList.SetVertexBuffer(buf->gpuAddress);
}
```

**GPU リソースは、外部から生ポインタで参照すべきではありません。** ハンドルが自然です。

**Direct3D や Vulkan の API 自体が、ハンドル的な設計になっています**(第45章のコラムで触れたとおりです)。

---

## 48.8 CPU 側の道具を、そのまま使う

**第2部で作った観測手段が、GPU メモリにも適用できます。**

### タグ別集計(第15章)

```cpp
{
    GA_TAG(ga::MemoryTag::Texture);
    LoadTextures(gpuAllocator);
}
```

```
=== GPU メモリ(DEFAULT ヒープ)===
  ヒープ数     : 12(768 MB)
  使用中       : 612 MB (79.7%)
  内部断片化   : 8.2%(バディの丸め)
  遅延解放待ち : 24 MB(38 件)

  タグ         使用量     割合
  ------------------------------
  Texture     412 MB    67.3%
  Mesh        148 MB    24.2%
  UI           31 MB     5.1%
  Debug        21 MB     3.4%
```

**「VRAM が足りない」と言われたときに、内訳が分かります。**

### 可視化(第19章)

**ヒープごとの使用状況を、そのまま絵にできます。**

**バディなので、2 の冪のブロックが並んだ規則的な模様になります。**

### 断片化指標(第19章)

**そのまま使えます。**

**注意:** 48.3 節で見たとおり、バディの内部断片化は 2 の冪への丸めで発生します。**第26章 26.8 節の「外部断片化は低いが、内部断片化でメモリが足りない」という現象に注意してください。**

---

## 48.9 落とし穴

### ① GPU が使用中のメモリを解放する

**最も危険です。**

```
症状: 稀に画面が乱れる / デバイスが失われる / ドライバがクラッシュする
```

**再現しません。** 負荷が高いときだけ起きます。

**対策:** フェンスを正しく使う。**デバッグレイヤを常に有効にする。**

### ② アップロードバッファを毎フレーム作る

```cpp
// 悪い例
auto uploadBuffer = CreateUploadBuffer(size);    // ← 毎フレーム数十 µs
```

**48.6 節のリングバッファを使ってください。**

### ③ Committed リソースを大量に作る

**48.2 節で見たとおり、桁違いに高価です。**

**リソースが数百を超えるなら、必ずサブアロケーションに移行してください。**

### ④ ヒープの種類を間違える

```cpp
// DEFAULT ヒープに CPU から書き込もうとする → 失敗、または極端に遅い
```

**UPLOAD → コピー → DEFAULT という流れを守ってください。**

### ⑤ デバッグレイヤを使わない

**Direct3D 12 も Vulkan も、開発用の検証レイヤを提供しています。**

- リソースの状態遷移の誤り
- 使用中のリソースの解放
- アラインメント違反
- API の使い方の誤り

**第31章で AddressSanitizer を扱ったのと、同じ位置づけです。**

> **「開発中は常に有効、出荷時は無効」** が原則です。**性能は落ちますが、それで見つかるバグの価値のほうが、はるかに大きい。**

---

## 演習

**演習48-1** `VirtualBuddy` を実装し、第26章の `Buddy` と同じテストを通してください。

**演習48-2** `VirtualBuddy` の CPU 側メモリ消費を、ヒープサイズと最小ブロックの組み合わせで計算してください。

**演習48-3** `DeferredFreeQueue` に、フェンス値が単調でない場合の検証を追加してください。

**演習48-4** 「N フレーム後に解放」方式を実装し、N が足りない場合に何が起きるか(擬似的に)確認してください。

**演習48-5** `UploadRing` で、`head` が `tail` に追いつく状況を作ってください。どう対処すべきですか。

**演習48-6** 定数バッファのアラインメントを 256 として、64 バイトの定数を 1000 個確保してください。無駄はどれだけですか。

**演習48-7** GPU アロケーターに第15章のタグ集計を組み込み、レポートを出力してください。

**演習48-8** 第19章の可視化を GPU ヒープに適用してください。バディ特有の模様は見えますか。

---

## 章末チェックリスト

- [ ] CPU メモリとの5つの違いを挙げられる
- [ ] サブアロケーションが必要な理由(確保コスト)を説明できる
- [ ] Committed と Placed の違いを説明できる
- [ ] **バディが GPU に適する3条件** を説明できる
- [ ] **第26章の実装で変えるべき1点** を説明できる
- [ ] リンクを CPU 側の配列に移す方法を実装した
- [ ] フェンスによる遅延解放を実装した
- [ ] 「N フレーム後」方式との違いを説明できる
- [ ] アップロード用のリングバッファを実装した
- [ ] 定数バッファのアラインメントによる無駄を計算した
- [ ] デバッグレイヤの位置づけを説明できる

---

## 次章の予告

**第7部の最後は、すべてをまとめる章です。**

ここまで、たくさんのアリーナを作ってきました。

```
永続アリーナ、シーンアリーナ × 2、フレームアリーナ × 面数 × ワーカー数、
汎用ヒープ、GPU ヒープ(DEFAULT / UPLOAD)…
```

**合計で、どれだけ使っているのか。** そして、**どれだけ使ってよいのか。**

第49章では **メモリ予算** を扱います。

```
=== メモリ予算 ===
  カテゴリ      上限      現在      ピーク    状態
  ─────────────────────────────────────────────
  Texture      512 MB    412 MB    468 MB    OK
  Mesh         256 MB    148 MB    201 MB    OK
  Audio         96 MB     88 MB     94 MB    ⚠ 警告
  Script        32 MB     41 MB     41 MB    ✗ 超過
```

- カテゴリごとの上限を設定する
- 超過を **検出して、止める**
- ピークを記録し、開発チームで共有する
- ImGui で実行中に内訳を表示する

**そして、この本のコードは v1.0 になります。**

第15章で作ったタグ、第8章で作ったピーク、第19章で作った可視化——**すべてがここに集まります。**

---

> **コラム:D3D12MA と VMA の立ち位置**
>
> **GPU メモリのアロケーターには、優れた既製品があります。**
>
> ---
>
> **D3D12 Memory Allocator(D3D12MA)と Vulkan Memory Allocator(VMA)**
>
> どちらも、GPU ベンダーが公開しているオープンソースのライブラリです。**Vulkan 版のほうが歴史が長く、Direct3D 12 版はその設計を引き継いでいます。**
>
> **やっていることは、この章で作ったものと同じです。**
>
> - 大きなヒープを確保し、サブアロケーションする
> - リソースの種類とサイズに応じて、適切なヒープを選ぶ
> - 断片化を管理する
> - 統計を取る
>
> **加えて、既製品ならではの機能があります。**
>
> - **メモリの種類の自動選択。** ハードウェアによって、使えるヒープの構成が違います。「CPU から書けて GPU から速く読める」メモリがある環境と、ない環境がある。**その差を吸収してくれます**
> - **デフラグ機能。** 第46章のコンパクションに相当します
> - **予算の管理。** OS が報告する VRAM の残量を見て、確保を調整します
> - **豊富な統計と可視化**
>
> ---
>
> **興味深い事実:この本のアルゴリズムが、選択肢として入っている**
>
> **VMA には、アロケーションのアルゴリズムを選ぶ機能があります。**
>
> - **既定のアルゴリズム。** 近年のバージョンでは **TLSF ベース** です(第27章)
> - **線形(リニア)アルゴリズム。** バンプアロケーター(第2章)です。リングバッファやスタックとしても使えます(第9章)
>
> **この本の第2章、第9章、第27章で作ったものが、そのまま選択肢として存在しています。**
>
> **偶然ではありません。** 第20章で見たとおり、この問題には決まった解法の集合があり、**用途に応じて選ぶ** という構造が共通しているからです。
>
> ---
>
> **では、自作すべきか**
>
> **結論から言えば、多くの場合は既製品を使うべきです。**
>
> - **実績がある。** 多数のタイトルで使われ、多様なハードウェアで検証されている
> - **ハードウェア差の吸収は、自分でやると大変**
> - **オープンソースなので、必要なら改造できる**
>
> **第53章で扱う「いつ自作すべきでないか」の、代表例です。**
>
> ---
>
> **それでも、この章に意味がある理由**
>
> **① 中で何が起きているか分かる。**
>
> D3D12MA の統計を見て「内部断片化が 15%」と出たとき、**なぜそうなるかが分かります。** 第26章を読んでいれば、バディの丸めが原因だと推測できます。
>
> **② 設定を選べる。**
>
> VMA でアルゴリズムを選ぶとき、**「この用途にはリニアが合う」と判断できます。** 第28章の決定表が、そのまま使えます。
>
> **③ 既製品が使えない場面がある。**
>
> 独自のプラットフォーム、特殊な要件、ライセンスの制約。**そのときに、自分で書けることに意味があります。**
>
> **④ 問題を切り分けられる。**
>
> 「メモリが足りない」と言われたとき、**それが断片化なのか、単純な使いすぎなのかを判断できます。** 第19章の指標と、第15章の内訳が、その材料になります。
>
> ---
>
> **第17章のコラムで、こう書きました。**
>
> > 自作アロケーターを使うということは、既存のデバッグ支援を1つ失うということでもある。
>
> **逆もまた真です。**
>
> **既製品を使うということは、中身を理解する機会を1つ失うことでもあります。**
>
> **この本を読み終えた読者は、既製品を使いながら、中で何が起きているかを理解できます。** それが、この本の最終的な価値です。
