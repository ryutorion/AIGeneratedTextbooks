# 第23章 可変サイズを解放する 〔v0.16:フリーリストアロケーター〕

---

## この章のゴール

プールは強力でしたが、サイズが1種類でした。**ついに、任意のサイズを個別に解放できるアロケーターを書きます。**

第20章で紙の上でやったことを、そのままコードにします。

- ブロックにヘッダを付け、サイズを記録する
- 空きブロックをフリーリストで繋ぐ
- **first fit** で探す
- 大きすぎる穴は **分割** する 〔**v0.16**〕
- ブロックを先頭から歩いて、断片化を測る
- そして **第19章の可視化が、ついに面白くなります**

この章では **合体(coalescing)を実装しません**。第20章の紙のシミュレーションで、合体なしがどれほど悲惨かを見ました。それを実物で再現します。

**壊れたものを見てから直す。** それが第24章です。

---

## 23.1 何が必要か

第20章で洗い出した4つの問題のうち、プールでは3つが消えました。**可変サイズに戻すと、全部復活します。**

| 問題 | 対応 |
|---|---|
| サイズが分からない | **ヘッダにサイズを記録する** |
| 空きをどこに記録するか | **空きブロックにフリーリストを埋め込む**(第21章と同じ) |
| 探索コスト | **first fit で走査する**(O(n) を受け入れる) |
| 断片化 | **この章では対策しない** |

さらに、プールにはなかった処理が1つ増えます。

**分割。** 100 バイトの穴に 32 バイトを置いたら、残り 68 バイトを新しい空きブロックとして登録しなければなりません。

---

## 23.2 ヘッダを設計する

### レイアウト

```
       ブロック全体
┌──────────────┬──────────────────────────────┐
│   ヘッダ      │      ユーザーに返す領域        │
│  (16 バイト)  │                              │
└──────────────┴──────────────────────────────┘
 ▲              ▲
 h              Allocate が返すアドレス
```

ヘッダはユーザー領域の **直前** に置きます。`Free(p)` されたとき、`p - 16` を見ればヘッダが読めます。

### ヘッダの中身

```cpp
namespace ga::detail
{
    struct FreeListHeader
    {
        std::size_t     size;       // ブロック全体のサイズ(ヘッダ込み)
        FreeListHeader* nextFree;   // 空きのときだけ意味を持つ
    };
}
```

x64 で 16 バイトちょうど。**これは偶然ではありません。**

第6章で決めた既定のアラインメントは 16 でした。ユーザー領域が 16 バイト境界に乗るためには、ヘッダのサイズも 16 の倍数である必要があります。16 バイトなら、ちょうど1つ分です。

### `nextFree` は使用中のブロックでは無駄では?

そのとおりです。使用中のブロックでは、この8バイトは使われません。

**しかし、削っても得はありません。** アラインメントの都合でどのみち 16 バイト必要だからです。ヘッダを 8 バイトにしても、パディングで 8 バイト捨てるだけです。

**空いている場所は、使えるうちに使っておきます。** 第24章で境界タグを実装するとき、この余白が役に立ちます。

### サイズの下位ビットをフラグに使う

ブロックサイズは必ず 16 の倍数です。ということは、**下位4ビットは常に 0** です。

**そこにフラグを詰め込みます。**

```cpp
    inline constexpr std::size_t kFlagFree = 1;
    inline constexpr std::size_t kSizeMask = ~std::size_t{15};

    constexpr std::size_t SizeOf(const FreeListHeader* h) noexcept
    {
        return h->size & kSizeMask;
    }

    constexpr bool IsFree(const FreeListHeader* h) noexcept
    {
        return (h->size & kFlagFree) != 0;
    }
```

**これは実物のアロケーターが使っている技法です。** dlmalloc も、その系譜を引く glibc の malloc も、同じことをしています。

「フラグのために `bool` を1つ足す」と、構造体は 24 バイトになります(アラインメントの都合で)。**16 バイトのままフラグを3つまで持てる** のは、大きな差です。

第24章では、ここにもう1つフラグを足します。

### オーバーヘッドを計算する

```
32 バイトの確保:
  ヘッダ         16 バイト
  ユーザー領域   32 バイト
  ─────────────────────
  合計           48 バイト   → オーバーヘッド 50%

256 バイトの確保:
  合計          272 バイト   → オーバーヘッド 6.3%
```

**小さい確保ほど割高です。**

第21章のプールは、オーバーヘッドが実質ゼロでした(検査用ビットマップの 0.4% のみ)。**サイズの自由には、これだけの値段が付いています。**

小さいオブジェクトを大量に扱うなら、プールのほうが有利です。第28章の比較で、この差を数字で確認します。

---

## 23.3 分割の規則

100 バイトの空きに 32 バイトを置いたとします。残りは 68 バイトです。

これを新しい空きブロックとして登録したい。しかし、**残りが小さすぎると登録できません**。

```
残り 8 バイト → ヘッダ(16 バイト)すら入らない
```

そこで最小ブロックサイズを決めます。

```cpp
    inline constexpr std::size_t kHeaderSize   = sizeof(detail::FreeListHeader);   // 16
    inline constexpr std::size_t kMinBlockSize = kHeaderSize + kDefaultAlignment;  // 32
```

**分割後の残りが 32 バイト未満なら、分割しません。** その場合、要求より大きなブロックを丸ごと渡すことになります。

```
要求 32 バイト(必要 48 バイト)
空きブロック 60 バイト

  分割すると残り 12 バイト → 小さすぎる
  → 60 バイト全部を渡す。12 バイトは内部断片化として捨てる
```

**内部断片化が生まれます。** 第6章で見たパディングと同じ種類の無駄です。

> **外部断片化を減らそうとすると、内部断片化が増える。** この交換関係は、第3部を通してずっと付きまといます。第26章のバディシステムで、これが極端な形で現れます。

---

## 23.4 実装する 〔v0.16〕

```cpp
// ga/FreeList.h
#pragma once

#include "ga/Core.h"
#include "ga/AllocError.h"

#include <cstddef>
#include <vector>

namespace ga
{
    namespace detail
    {
        struct FreeListHeader
        {
            std::size_t     size;
            FreeListHeader* nextFree;
        };

        inline constexpr std::size_t kFlagFree = 1;
        inline constexpr std::size_t kSizeMask = ~std::size_t{ 15 };

        constexpr std::size_t SizeOf(const FreeListHeader* h) noexcept { return h->size & kSizeMask; }
        constexpr bool        IsFree(const FreeListHeader* h) noexcept { return (h->size & kFlagFree) != 0; }
    }

    class FreeList
    {
    public:
        using Header = detail::FreeListHeader;

        static constexpr std::size_t kHeaderSize   = sizeof(Header);
        static constexpr std::size_t kMinBlockSize = kHeaderSize + kDefaultAlignment;

        explicit FreeList(std::size_t capacity)
            : buffer_(capacity + kDefaultAlignment)
        {
            const auto raw = reinterpret_cast<std::uintptr_t>(buffer_.data());
            base_ = reinterpret_cast<std::byte*>(AlignUp(raw, kDefaultAlignment));

            capacity_ = (capacity / kDefaultAlignment) * kDefaultAlignment;

            // 板全体を1つの空きブロックにする
            Header* h  = reinterpret_cast<Header*>(base_);
            h->size    = capacity_ | detail::kFlagFree;
            h->nextFree = nullptr;

            freeHead_ = h;
        }

        // --- 確保 ---
        [[nodiscard]] void* Allocate(std::size_t size) noexcept
        {
            if (size == 0) { return nullptr; }

            const std::size_t payload = AlignUpSize(size, kDefaultAlignment);
            if (payload < size) { return nullptr; }                 // 桁溢れ

            const std::size_t need = payload + kHeaderSize;
            if (need < payload) { return nullptr; }

            Header* prev = nullptr;

            for (Header* h = freeHead_; h != nullptr; prev = h, h = h->nextFree)
            {
                ++searchSteps_;

                const std::size_t blockSize = detail::SizeOf(h);
                if (blockSize < need) { continue; }

                if (blockSize - need >= kMinBlockSize)
                {
                    // --- 分割する ---
                    Header* rest = reinterpret_cast<Header*>(
                        reinterpret_cast<std::byte*>(h) + need);

                    rest->size     = (blockSize - need) | detail::kFlagFree;
                    rest->nextFree = h->nextFree;

                    Relink(prev, rest);
                    h->size = need;                     // フラグ 0 = 使用中
                }
                else
                {
                    // --- 丸ごと使う ---
                    Relink(prev, h->nextFree);
                    h->size = blockSize;
                    internalWaste_ += blockSize - need;
                }

                h->nextFree = nullptr;
                ++liveCount_;
                ++searches_;

                return reinterpret_cast<std::byte*>(h) + kHeaderSize;
            }

            ++searches_;
            ++failures_;
            return nullptr;
        }

        // --- 解放 ---
        void Free(void* p) noexcept
        {
            if (p == nullptr) { return; }

            if (!Owns(p)) { ReportError(AllocError::OutOfMemory, p); return; }

            Header* h = HeaderOf(p);

            if (detail::IsFree(h)) { ReportError(AllocError::OutOfMemory, p); return; }

            h->size    |= detail::kFlagFree;
            h->nextFree = freeHead_;
            freeHead_   = h;

            --liveCount_;
        }

        // --- ブロックを先頭から順に歩く ---
        template <class F>
        void ForEachBlock(F&& fn) const
        {
            const std::byte* end = base_ + capacity_;

            for (const std::byte* p = base_; p < end; )
            {
                const Header* h = reinterpret_cast<const Header*>(p);
                const std::size_t s = detail::SizeOf(h);

                fn(h, s, detail::IsFree(h));

                if (s == 0) { break; }      // 壊れている。無限ループを避ける
                p += s;
            }
        }

        std::size_t Capacity()    const noexcept { return capacity_; }
        std::size_t LiveCount()   const noexcept { return liveCount_; }
        std::size_t Failures()    const noexcept { return failures_; }
        double      AverageSearchSteps() const noexcept
        {
            return (searches_ == 0) ? 0.0
                 : static_cast<double>(searchSteps_) / static_cast<double>(searches_);
        }

    private:
        Header* HeaderOf(void* p) const noexcept
        {
            return reinterpret_cast<Header*>(static_cast<std::byte*>(p) - kHeaderSize);
        }

        bool Owns(const void* p) const noexcept
        {
            const auto* b = static_cast<const std::byte*>(p);

            if (b < base_ + kHeaderSize || b >= base_ + capacity_) { return false; }
            return (static_cast<std::size_t>(b - base_) % kDefaultAlignment) == 0;
        }

        void Relink(Header* prev, Header* next) noexcept
        {
            if (prev == nullptr) { freeHead_    = next; }
            else                 { prev->nextFree = next; }
        }

        void ReportError(AllocError, const void*) const noexcept
        {
            assert(false && "FreeList の使い方が誤っています");
        }

        std::vector<std::byte> buffer_;
        std::byte*             base_      = nullptr;
        Header*                freeHead_  = nullptr;
        std::size_t            capacity_  = 0;
        std::size_t            liveCount_ = 0;
        std::size_t            failures_  = 0;
        std::size_t            searches_  = 0;
        std::size_t            searchSteps_   = 0;
        std::size_t            internalWaste_ = 0;
    };
}
```

### 実装の注意点

**`Relink` の使い分け。** 分割したときは、フリーリストの `h` の位置に `rest` を差し込みます。分割しないときは、`h` を外して `h->nextFree` で繋ぎ直します。この2ケースを間違えると、フリーリストが切れるか輪になります。

**`ForEachBlock` が成立する理由。** すべてのブロックが、隙間なく敷き詰められているからです。先頭から `size` ずつ進めば、必ず次のブロックの先頭に着きます。

**この「暗黙のリスト」が、第24章の鍵になります。** 合体するには「隣のブロック」を知る必要がありますが、それはこの歩き方で分かります。

**所属チェックが不完全。** `Owns` は範囲と整列しか見ません。ブロックの先頭でないアドレスを渡されても検出できません。第22章のプールでは `% kBlockSize` で完全に判定できましたが、可変サイズではブロック境界が一定でないためです。

> **完全な検証は、`ForEachBlock` で全ブロックを歩けば可能です。** ただし O(n) です。Debug 構成専用の検査として用意するのが現実的でしょう(演習23-3)。

---

## 23.5 動かす

```cpp
int main()
{
    ga::FreeList heap(1024);

    void* a = heap.Allocate(100);
    void* b = heap.Allocate(200);
    void* c = heap.Allocate(50);

    std::println("a={} b={} c={}", a, b, c);
    std::println("live={}", heap.LiveCount());

    DumpBlocks(heap);

    heap.Free(b);

    std::println("--- b を解放 ---");
    DumpBlocks(heap);

    void* d = heap.Allocate(80);    // b の穴に入るはず
    std::println("d={} (b の穴に入った? {})", d, d == b);

    DumpBlocks(heap);
}
```

`DumpBlocks` は `ForEachBlock` を使った表示です。

```cpp
void DumpBlocks(const ga::FreeList& heap)
{
    std::size_t offset = 0;

    heap.ForEachBlock([&](const void*, std::size_t size, bool isFree) {
        std::println("  +{:<6} {:>6} バイト  {}",
                     offset, size, isFree ? "空き" : "使用中");
        offset += size;
    });
}
```

```
  +0        112 バイト  使用中
  +112      224 バイト  使用中
  +336       64 バイト  使用中
  +400      624 バイト  空き

--- b を解放 ---
  +0        112 バイト  使用中
  +112      224 バイト  空き
  +336       64 バイト  使用中
  +400      624 バイト  空き

d=... (b の穴に入った? true)
  +0        112 バイト  使用中
  +112       96 バイト  使用中
  +208      128 バイト  空き      ← 分割された残り
  +336       64 バイト  使用中
  +400      624 バイト  空き
```

**穴が再利用され、余りが分割されました。** 第20章で紙の上でやったことが、そのまま動いています。

### テスト

```cpp
void Test_FreeListBasics()
{
    ga::FreeList heap(4096);

    void* a = heap.Allocate(100);
    void* b = heap.Allocate(100);

    assert(a != nullptr && b != nullptr);
    assert(a != b);
    assert(heap.LiveCount() == 2);

    // 領域が重ならない(アロケーターの最も基本的な契約)
    const auto* pa = static_cast<std::byte*>(a);
    const auto* pb = static_cast<std::byte*>(b);
    assert(pa + 100 <= pb || pb + 100 <= pa);

    heap.Free(a);
    assert(heap.LiveCount() == 1);

    std::println("[ OK ] Test_FreeListBasics");
}

void Test_FreeListReusesHole()
{
    ga::FreeList heap(4096);

    void* a = heap.Allocate(64);
    void* b = heap.Allocate(64);
    void* c = heap.Allocate(64);

    heap.Free(b);

    void* d = heap.Allocate(64);
    assert(d == b);        // 同じ穴が返る

    (void)a; (void)c;
    std::println("[ OK ] Test_FreeListReusesHole");
}

void Test_FreeListAlignment()
{
    ga::FreeList heap(4096);

    for (std::size_t size : { 1u, 7u, 15u, 16u, 17u, 100u })
    {
        void* p = heap.Allocate(size);
        assert(p != nullptr);
        assert(ga::IsAlignedTo(p, ga::kDefaultAlignment));
    }

    std::println("[ OK ] Test_FreeListAlignment");
}
```

---

## 23.6 断片化を測る

`ForEachBlock` があれば、第19章で定義した指標を実際に計算できます。

```cpp
    FragmentationStats GetFragmentation() const noexcept
    {
        FragmentationStats f;
        f.capacity      = capacity_;
        f.internalWaste = internalWaste_;

        ForEachBlock([&](const void*, std::size_t size, bool isFree) {
            if (isFree)
            {
                f.freeTotal += size;
                ++f.freeBlockCount;
                if (size > f.freeLargest) { f.freeLargest = size; }
            }
            else
            {
                f.used += size;
            }
        });

        return f;
    }
```

**第19章では常に 0.00 だった外部断片化が、いよいよ動き出します。**

---

## 23.7 可視化:穴が空く

### 描画を対応させる

第19章の `Render` を、`FreeList` 用に書きます。タグではなく、使用中か空きかで塗り分けます。

```cpp
inline img::Image RenderFreeList(const ga::FreeList& heap, const viz::MapConfig& cfg = {})
{
    const std::size_t pixelCount = (heap.Capacity() + cfg.bytesPerPixel - 1) / cfg.bytesPerPixel;
    const int rows = static_cast<int>((pixelCount + cfg.width - 1) / cfg.width);

    img::Image image(cfg.width * cfg.scale, rows * cfg.scale);

    // まず全部を「空き」で塗る
    // 次に使用中ブロックを塗る
    std::size_t offset = 0;

    heap.ForEachBlock([&](const void*, std::size_t size, bool isFree) {
        const img::Rgb color = isFree ? img::Rgb{ 30, 30, 34 }
                                      : img::Rgb{ 80, 200, 100 };

        const std::size_t first = offset / cfg.bytesPerPixel;
        const std::size_t last  = (offset + size - 1) / cfg.bytesPerPixel;

        for (std::size_t i = first; i <= last && i < pixelCount; ++i)
        {
            const int px = static_cast<int>(i % cfg.width);
            const int py = static_cast<int>(i / cfg.width);

            for (int sy = 0; sy < cfg.scale; ++sy)
            {
                for (int sx = 0; sx < cfg.scale; ++sx)
                {
                    image.Set(px * cfg.scale + sx, py * cfg.scale + sy, color);
                }
            }
        }

        offset += size;
    });

    return image;
}
```

### 実験:ランダムな確保と解放

第20章のシミュレーターと同じ負荷を、**実物のアロケーター** にかけます。

```cpp
int main()
{
    constexpr std::size_t kCapacity = 1 * 1024 * 1024;   // 1 MB
    ga::FreeList heap(kCapacity);

    std::mt19937 rng(12345);
    std::discrete_distribution<int> sizeClass({ 60, 25, 10, 4, 1 });
    const std::size_t sizes[] = { 32, 128, 512, 4096, 65536 };
    std::uniform_real_distribution<double> action(0.0, 1.0);

    std::vector<void*> live;

    for (int step = 0; step <= 20'000; ++step)
    {
        if (step % 5000 == 0)
        {
            const auto f = heap.GetFragmentation();

            std::println("step {:>6}: 空き {:>5} 個  最大 {:>8}  外部断片化 {:.3f}  失敗 {}",
                         step, f.freeBlockCount, f.freeLargest,
                         f.ExternalFragmentation(), heap.Failures());

            RenderFreeList(heap).WritePng(std::format("freelist_{:05}.png", step).c_str());
        }

        if (!live.empty() && action(rng) < 0.45)
        {
            std::uniform_int_distribution<std::size_t> pick(0, live.size() - 1);
            const std::size_t k = pick(rng);

            heap.Free(live[k]);
            live[k] = live.back();
            live.pop_back();
        }
        else
        {
            void* p = heap.Allocate(sizes[sizeClass(rng)]);
            if (p != nullptr) { live.push_back(p); }
        }
    }
}
```

### 結果

```
step      0: 空き     1 個  最大  1048576  外部断片化 0.000  失敗 0
step   5000: 空き   612 個  最大    41248  外部断片化 0.847  失敗 0
step  10000: 空き  1489 個  最大     8192  外部断片化 0.961  失敗 38
step  15000: 空き  2201 個  最大     4096  外部断片化 0.983  失敗 412
step  20000: 空き  2874 個  最大     4096  外部断片化 0.986  失敗 1108
```

### 絵の変化

```
step 0:
████████████████████████████████████████   ← 全部空き(暗い)

step 5000:
███░██░███░█░███░██░███░█░██░████░██░███   ← 穴が散り始める

step 10000:
█░█░██░█░█░██░░█░█░█░██░█░█░██░█░░█░█░██   ← 穴だらけ

step 20000:
█░█░█░█░█░█░░█░█░█░█░█░█░█░█░░█░█░█░█░█░   ← ほぼ市松模様
```

**第20章の紙のシミュレーションが、そのまま起きています。**

### 何が起きているのか

`step 20000` の時点を見てください。

```
空きブロック : 2874 個
空きの合計   : 約 380 KB
最大の空き   : 4096 バイト
```

**380 KB 空いているのに、8 KB の確保が失敗します。**

第20章で紙の上で見た「合計 10 マス空いているのに 5 マスが取れない」が、100 倍の規模で起きています。

そして失敗回数は加速しています。0 → 38 → 412 → 1108。**断片化は自己増殖します。** 穴が細かくなるほど新しい確保が入らず、大きな穴だけが消費され、さらに細かい穴が増える。

---

## 23.8 探索コストの増大

もう1つの問題も測ります。

```
step      0: 平均探索ステップ    1.0    確保 1回あたり     18 ns
step   5000: 平均探索ステップ  184.3    確保 1回あたり    412 ns
step  10000: 平均探索ステップ  521.7    確保 1回あたり   1180 ns
step  20000: 平均探索ステップ 1348.9    確保 1回あたり   2940 ns
```

**確保が 160 倍遅くなりました。**

first fit は先頭から探します。フリーリストの先頭に小さな穴が溜まると、大きな要求のたびに、その全部を読み飛ばすことになります。

しかもフリーリストはポインタで繋がっているので、**キャッシュに乗りません**。1ノード読むたびにメモリアクセスが発生します。

比較のために:

| | 1回の確保 |
|---|---|
| `Bump`(第5章) | 1.8 ns |
| `Pool`(第22章) | 2.9 ns |
| **`FreeList`(この章、20000 step 後)** | **2940 ns** |
| `new` | 17.6 ns |

**`new` より 167 倍遅い。** 第5章で 10 倍勝っていた相手に、完敗しています。

---

## 23.9 まとめ:何を得て、何を失ったか

| | `Bump` | `Pool` | `FreeList` |
|---|---|---|---|
| 任意サイズ | ○ | × | **○** |
| 個別解放 | × | ○ | **○** |
| 確保 | O(1) | O(1) | **O(n)** |
| 解放 | — | O(1) | O(1) |
| 外部断片化 | 0 | 0 | **深刻** |
| オーバーヘッド | 0 | ほぼ0 | **16 バイト/確保** |

**欲しかった自由は手に入りました。代償も、想定どおりに払っています。**

第20章で示した第3部の地図を思い出してください。「制約を外していく」のではなく「別の制約に付け替える」章でした。ここで付け替えた先が、あまりに高くついた——それが現状です。

**しかし、まだ何もしていません。** この章では、第20章で「これを入れると劇的に改善する」と分かっていた処理を、あえて入れませんでした。

**合体です。**

---

## 演習

**演習23-1** 第20章 20.3 節の10ステップの操作列を、この `FreeList` で実行してください。紙の上の結果と一致しますか。

**演習23-2** first fit を **best fit** に変更してください。断片化と探索コストはどう変わりますか。第20章のシミュレーション結果と傾向は一致しますか。

**演習23-3** `ForEachBlock` を使って、ポインタが正しいブロックの先頭かを完全に検証する `ValidatePointer` を実装してください。Debug 構成でのみ有効にします。

**演習23-4** 任意のアラインメント(32、64 バイトなど)に対応させてください。ヘッダの位置をどう見つけますか。(ヒント:ユーザー領域の直前に、ヘッダまでのオフセットを書く)

**演習23-5** ヘッダを 8 バイトに減らすことはできますか。何が犠牲になりますか。

**演習23-6** サイズの下位ビットにフラグを入れる技法を使わず、`bool` メンバを足した場合、`sizeof(FreeListHeader)` はいくつになりますか。1万ブロックで何バイトの差ですか。

**演習23-7** 23.7 の実験を、確保サイズをすべて 64 バイトに固定して実行してください。断片化は起きますか。この結果は何を意味しますか。

---

## 章末チェックリスト

- [ ] ヘッダにサイズを持たせ、`Free(p)` から `p - 16` で読める仕組みを実装した
- [ ] サイズの下位ビットをフラグに使う技法を理解した
- [ ] 分割と、最小ブロックサイズによる分割の抑制を実装した
- [ ] `ForEachBlock` で全ブロックを歩けることと、その理由を説明できる
- [ ] 外部断片化が 0.98 まで上がるのを観測した
- [ ] **合計では足りているのに確保が失敗する** ことを実物で再現した
- [ ] 探索コストが 160 倍に増えることを測った
- [ ] 可視化で穴だらけの絵を出力した

---

## 次章の予告

第24章では、たった1つの処理を足します。

> **解放するとき、隣が空きなら、くっつける。**

第20章の紙のシミュレーションで、これが失敗を成功に変えるのを見ました。第20章のシミュレーターでは、失敗が 9821 回から 412 回に減りました。

問題は、**「隣」をどうやって知るか** です。

後ろの隣は簡単です。`h + SizeOf(h)` が次のブロックの先頭ですから。

**前の隣が難しい。** ブロックは可変サイズなので、「16 バイト戻れば前のブロック」とはいきません。前のブロックの先頭がどこにあるか、いまの `FreeList` では分かりません。

先頭から `ForEachBlock` で歩けば見つかりますが、それでは O(n) です。**解放が O(1) でなくなります。**

Knuth が示した答えが **境界タグ** です。ブロックの末尾にもサイズを書いておけば、前のブロックの末尾を読むだけで、その大きさが分かります。

そして、この章のヘッダに残しておいた「使用中では無駄になる 8 バイト」が、ついに役目を果たします。

---

> **コラム:ヘッダを削る職人技**
>
> 23.2 節で、16 バイトのヘッダがオーバーヘッド 50% を生むことを見ました。**アロケーターの設計者にとって、ヘッダは削るべき税金です。**
>
> どこまで削れるか、実際の実装が使っている工夫を見てみましょう。
>
> ---
>
> **工夫1:フラグをサイズに埋め込む**
>
> この章で採用した技法です。サイズが必ず 8 や 16 の倍数であることを利用して、下位ビットを使います。
>
> dlmalloc は下位2ビットを使っています。1つは「前のブロックが使用中か」、もう1つは「この領域は mmap で取ったか」です。**構造体を大きくせずにフラグを増やせます。**
>
> ---
>
> **工夫2:フッタを使用中ブロックでは省く**
>
> 第24章で境界タグを実装しますが、素朴にやるとヘッダとフッタで 24 バイト以上必要になります。
>
> ここに巧妙な観察があります。**フッタが必要なのは、そのブロックが空きのときだけ** です。使用中のブロックのサイズを、後ろから調べる必要はありません(前のブロックを合体しようとするのは、前が空きのときだけだから)。
>
> だから dlmalloc は、**「前のブロックが空きのときだけ、その末尾にサイズを書く」** という設計にしています。使用中のブロックでは、その 8 バイトはユーザーデータとして使われます。
>
> **フッタの領域を、隣のブロックと共有している** わけです。これでオーバーヘッドは実質 8 バイトになります。
>
> ---
>
> **工夫3:ヘッダを持たない**
>
> もっと過激な方法もあります。**サイズの情報を、ブロックの中ではなく別の場所に持つ** のです。
>
> 第21章のプールがそうでした。すべて同じサイズなので、ヘッダが要りません。
>
> 現代の高性能アロケーター(mimalloc、tcmalloc、jemalloc)は、この発想を可変サイズに拡張しています。メモリを大きな「ページ」に区切り、**1ページ内はすべて同じサイズクラス** にします。ポインタからページを計算し、ページの情報を見ればサイズが分かる——**個々のブロックにヘッダが要りません。**
>
> ```
> ポインタ → 上位ビットを取り出す → ページ情報 → サイズクラス
> ```
>
> これは第25章のサイズ別ビンと、第31章のスレッドローカル化を組み合わせた先にある設計です。第15章で見たとおり、実際のプログラムは少数のサイズを繰り返し要求します。**その規則性を、レイアウトそのものに埋め込んだ** のが現代のアロケーターです。
>
> ---
>
> 私たちの 16 バイトのヘッダは、これらに比べれば素朴です。しかし、**なぜ削りたいのか** を体感するには十分でした。第28章で全実装を比較するとき、このオーバーヘッドが数字として効いてくるのを見ることになります。
