# 第36章 スレッドごとに持たせる 〔v0.25〕

---

## この章のゴール

第35章の結論は明快でした。

> **「守るべき処理」より「守る仕組み」のほうが高価なら、設計を見直すべき。**

この章では、**ロックを取らずに済む構造** を作ります。

```
各スレッド : 自分専用のキャッシュ(ロック不要)
中央       : 大きなチャンクの供給元(たまにしか触らない)
```

確保のほとんどはスレッドローカルで完結します。足りなくなったときだけ、中央から **64 KB まとめて** もらってきます。**ロックのコストが 1000 分の 1 に薄まります。**

- `thread_local` のコストと落とし穴を測る
- **二層構造** のアロケーターを実装する 〔**v0.25**〕
- チャンクを 64 KB 境界に置き、**ブロックごとのヘッダを不要にする**
- **クロススレッド解放** という難問に、3つの答えを比較して1つ実装する
- 8スレッドで、第35章の 134 倍のスループットを出す

---

## 36.1 二層構造

```
┌──────────────────────────────────────────────────┐
│                  中央ヒープ                        │
│         64 KB のチャンクを配る(ロックあり)         │
└───────┬──────────────┬──────────────┬────────────┘
        │              │              │
   ┌────▼────┐    ┌────▼────┐    ┌────▼────┐
   │ スレッド0 │    │ スレッド1 │    │ スレッド2 │
   │ キャッシュ│    │ キャッシュ│    │ キャッシュ│
   │(ロック無)│    │(ロック無)│    │(ロック無)│
   └─────────┘    └─────────┘    └─────────┘
```

### なぜ効くのか

64 KB のチャンクを、64 バイトのブロックに切り分けると **1024 個** になります。

```
1024 回の確保のうち、中央を触るのは 1 回だけ
→ ロックのコストが 1/1024 に薄まる
```

第35章で `SpinLock` が 1 回 6 ns 程度でした。1024 回に 1 回なら、**1確保あたり 0.006 ns**。無視できます。

**これが tcmalloc が広めた発想です。** 第33章のコラムで触れたとおり、名前そのものが "thread-caching malloc" でした。

---

## 36.2 `thread_local` のコストを測る

まず、道具の性能を確認します。

```cpp
// ① 定数で初期化されるポインタ
thread_local ga::ThreadCache* g_cachePtr = nullptr;

// ② 動的初期化されるオブジェクト
thread_local ga::ThreadCache g_cacheObject;

int main()
{
    auto r1 = bench::MeasureBatch(200, 100'000, [] {
        bench::Escape(g_cachePtr);
    });

    auto r2 = bench::MeasureBatch(200, 100'000, [] {
        bench::Escape(&g_cacheObject);
    });

    bench::Print("thread_local ポインタ ", r1);
    bench::Print("thread_local オブジェクト", r2);
}
```

```
thread_local ポインタ      median=      0.71 ns
thread_local オブジェクト   median=      2.48 ns
```

### 3.5 倍の差はどこから来るか

**動的初期化を伴う `thread_local` は、アクセスのたびに「初期化済みか」の検査が入ります。**

```
① ポインタ(ゼロ初期化)
   → TLS 領域から読むだけ

② コンストラクタを持つオブジェクト
   → 初期化フラグを見る
   → 未初期化なら初期化する
   → ラッパ関数の呼び出しが挟まる
```

しかも、**この検査はインライン展開されないことが多い** ため、関数呼び出しのコストも乗ります。

### 教訓

> **`thread_local` には、ポインタか単純な整数だけを置く。**
> **実体は別の場所に持ち、ポインタで指す。**

私たちの設計でも、この方針を採ります。

```cpp
thread_local ThreadCache* g_cache = nullptr;   // ← これだけ
```

### もう1つの落とし穴:破棄の順序

スレッドが終了するとき、`thread_local` オブジェクトのデストラクタが呼ばれます。**その順序と、他のグローバルオブジェクトとの前後関係は、注意が必要です。**

「スレッド終了時に、中央ヒープへチャンクを返す」という処理を書くとき、**中央ヒープがすでに破棄されていたら、クラッシュします。**

**対策:**

- 中央ヒープを、**プログラム終了まで生き続ける** ようにする(意図的にリークさせる、あるいは静的寿命にする)
- スレッド終了時の返却を、**あえてやらない**(チャンクは中央ヒープが保持しているので、リークにはならない)

**本書では後者を選びます。** スレッドが終了しても、そのスレッドが持っていたチャンクは中央ヒープの管理下に残ります。回収は次章の話題に譲ります。

---

## 36.3 チャンクを 64 KB 境界に置く

設計上の重要な工夫です。

**チャンクを 64 KB 境界に配置すると、任意のポインタから、そのチャンクの先頭が計算だけで求まります。**

```cpp
constexpr std::size_t kChunkSize = 64 * 1024;

ChunkHeader* ChunkOf(const void* p) noexcept
{
    return reinterpret_cast<ChunkHeader*>(
        reinterpret_cast<std::uintptr_t>(p) & ~(kChunkSize - 1));
}
```

**ビット演算1回です。**

### 何が嬉しいのか

**ブロックごとのヘッダが不要になります。**

第23章では、`Free(p)` からサイズを知るために 16 バイトのヘッダを付けました。第26章のバディはヘッダなしでしたが、ビットツリーが必要でした。

**この設計では、チャンクの先頭にある情報を見るだけです。**

```cpp
struct ChunkHeader
{
    std::uint32_t      owner;        // 所有スレッドのトークン
    std::uint32_t      sizeClass;    // このチャンクが扱うサイズクラス
    std::atomic<void*> remoteFree;   // 他スレッドからの返却リスト(36.5節)
    ChunkHeader*       nextChunk;
};
```

**1チャンク(64 KB)につき、ヘッダ1つ。** 1024 個のブロックで割れば、**1ブロックあたり 0.03 バイト** です。

第23章のコラムで、こう書きました。

> 現代の高性能アロケーターは、メモリを大きな「ページ」に区切り、1ページ内はすべて同じサイズクラスにします。ポインタからページを計算し、ページの情報を見ればサイズが分かる——個々のブロックにヘッダが要りません。

**その設計を、いま実装しています。**

### `VirtualAlloc` が味方する

第29章で確認したとおり、`VirtualAlloc` が返すアドレスは **必ず 64 KB 境界** です。

**予約領域の先頭が 64 KB 境界なので、そこから 64 KB ずつ切り出せば、すべてのチャンクが自動的に 64 KB 境界に乗ります。** 調整が一切要りません。

---

## 36.4 実装する 〔v0.25〕

### 中央ヒープ

```cpp
// ga/CentralHeap.h
#pragma once

#include "ga/Lock.h"
#include "ga/VirtualMemory.h"

namespace ga
{
    inline constexpr std::size_t kChunkSize  = 64 * 1024;
    inline constexpr std::size_t kChunkShift = 16;

    class CentralHeap
    {
    public:
        explicit CentralHeap(std::size_t reserveBytes)
            : memory_(AlignUpSize(reserveBytes, kChunkSize))
        {
        }

        // 64 KB のチャンクを1つ渡す
        std::byte* AcquireChunk() noexcept
        {
            std::lock_guard guard(lock_);

            ++acquireCount_;

            if (freeChunks_ != nullptr)          // 返却済みのものを再利用
            {
                std::byte* p = freeChunks_;
                freeChunks_ = *reinterpret_cast<std::byte**>(p);
                return p;
            }

            if (offset_ + kChunkSize > memory_.Reserved()) { return nullptr; }

            if (!memory_.CommitTo(offset_ + kChunkSize)) { return nullptr; }

            std::byte* p = memory_.Base() + offset_;
            offset_ += kChunkSize;
            return p;
        }

        void ReleaseChunk(std::byte* p) noexcept
        {
            std::lock_guard guard(lock_);

            *reinterpret_cast<std::byte**>(p) = freeChunks_;
            freeChunks_ = p;
        }

        std::size_t AcquireCount() const noexcept { return acquireCount_; }

    private:
        VirtualMemory memory_;
        SpinLock      lock_;
        std::byte*    freeChunks_   = nullptr;
        std::size_t   offset_       = 0;
        std::size_t   acquireCount_ = 0;
    };
}
```

**チャンクの再利用リストも、侵入的リストです。** 空きチャンクの先頭 8 バイトに、次のチャンクへのポインタを書きます。第21章から一貫している手法です。

### サイズクラス

```cpp
    inline constexpr std::size_t kSizeClassCount = 8;

    inline constexpr std::size_t kSizeClasses[kSizeClassCount] =
    {
        16, 32, 64, 128, 256, 512, 1024, 2048
    };

    // size に対応するクラスの番号。大きすぎれば npos
    inline std::size_t SizeClassOf(std::size_t size) noexcept
    {
        if (size == 0 || size > kSizeClasses[kSizeClassCount - 1])
        {
            return static_cast<std::size_t>(-1);
        }

        // 16 未満は 16、それ以外は 2 の冪に切り上げ
        const std::size_t rounded = (size <= 16) ? 16 : std::bit_ceil(size);

        return static_cast<std::size_t>(std::bit_width(rounded)) - 5;   // 16 → 0
    }
```

**2 の冪に丸めています。** 第26章のバディと同じで、内部断片化は最悪 50% です。

**割り切りです。** サイズクラスを細かくすると、クラスごとのチャンクが増え、メモリを食います。**スレッド数 × クラス数 × チャンクサイズ** が最低限必要になるためです。

```
8 スレッド × 8 クラス × 64 KB = 4 MB
```

**クラスを 32 個にすれば 16 MB です。** スレッドローカル化には、こういうメモリの代償が伴います。

### チャンクのヘッダ

```cpp
    struct alignas(64) ChunkHeader
    {
        std::uint32_t      owner     = 0;
        std::uint32_t      sizeClass = 0;
        std::atomic<void*> remoteFree{ nullptr };
        ChunkHeader*       nextChunk = nullptr;
    };

    inline ChunkHeader* ChunkOf(const void* p) noexcept
    {
        return reinterpret_cast<ChunkHeader*>(
            reinterpret_cast<std::uintptr_t>(p) & ~(kChunkSize - 1));
    }
```

`alignas(64)` を付けているのは、**第35章の偽共有対策** です。`remoteFree` は他スレッドが書き換えるので、他のデータと同じキャッシュラインに置きたくありません。

### スレッドキャッシュ

```cpp
// ga/ThreadCache.h
namespace ga
{
    class ThreadCache
    {
    public:
        explicit ThreadCache(CentralHeap& central, std::uint32_t token) noexcept
            : central_(&central), token_(token) {}

        [[nodiscard]] void* Allocate(std::size_t size) noexcept
        {
            const std::size_t cls = SizeClassOf(size);
            if (cls >= kSizeClassCount) { return nullptr; }   // 大きすぎる

            if (freeLists_[cls] == nullptr)
            {
                if (!Refill(cls)) { return nullptr; }
            }

            void* p = freeLists_[cls];
            freeLists_[cls] = *reinterpret_cast<void**>(p);
            return p;
        }

        void Free(void* p) noexcept
        {
            if (p == nullptr) { return; }

            ChunkHeader* chunk = ChunkOf(p);

            if (chunk->owner == token_)
            {
                // --- 自分のチャンク:ロックなしでフリーリストへ ---
                const std::size_t cls = chunk->sizeClass;
                *reinterpret_cast<void**>(p) = freeLists_[cls];
                freeLists_[cls] = p;
            }
            else
            {
                // --- 他人のチャンク:所有者宛のキューへ(36.5節)---
                PushRemote(chunk, p);
            }
        }

    private:
        bool Refill(std::size_t cls) noexcept
        {
            // ① まず、他スレッドから返ってきたブロックを回収する
            if (DrainRemote(cls)) { return true; }

            // ② それでも空なら、中央から新しいチャンクをもらう
            std::byte* raw = central_->AcquireChunk();
            if (raw == nullptr) { return false; }

            auto* chunk = std::construct_at(reinterpret_cast<ChunkHeader*>(raw));
            chunk->owner     = token_;
            chunk->sizeClass = static_cast<std::uint32_t>(cls);
            chunk->nextChunk = ownedChunks_[cls];
            ownedChunks_[cls] = chunk;

            // ③ チャンクをブロックに切り分けて、フリーリストに繋ぐ
            const std::size_t blockSize = kSizeClasses[cls];
            std::byte* first = raw + sizeof(ChunkHeader);
            const std::size_t count = (kChunkSize - sizeof(ChunkHeader)) / blockSize;

            void* head = nullptr;
            for (std::size_t i = count; i > 0; --i)
            {
                std::byte* block = first + (i - 1) * blockSize;
                *reinterpret_cast<void**>(block) = head;
                head = block;
            }

            freeLists_[cls] = head;
            return true;
        }

        CentralHeap*  central_ = nullptr;
        std::uint32_t token_   = 0;

        void*        freeLists_[kSizeClassCount]   = {};
        ChunkHeader* ownedChunks_[kSizeClassCount] = {};
    };
}
```

### 入口

```cpp
namespace ga
{
    inline CentralHeap& GlobalCentralHeap()
    {
        // 意図的に解放しない(スレッドより長生きさせる)
        static CentralHeap* heap = new CentralHeap(1024ull * 1024 * 1024);
        return *heap;
    }

    inline ThreadCache& CurrentThreadCache()
    {
        thread_local ThreadCache* cache = nullptr;   // ポインタだけ

        if (cache == nullptr)
        {
            cache = new ThreadCache(GlobalCentralHeap(), CurrentThreadToken());
        }
        return *cache;
    }

    [[nodiscard]] inline void* TcAllocate(std::size_t size) noexcept
    {
        return CurrentThreadCache().Allocate(size);
    }

    inline void TcFree(void* p) noexcept
    {
        CurrentThreadCache().Free(p);
    }
}
```

`CurrentThreadToken()` は、第34章で作った関数です。

---

## 36.5 クロススレッド解放

**この設計の最大の難所です。**

```cpp
// スレッド A で確保
void* p = TcAllocate(64);

// スレッド B で解放
TcFree(p);        // ← どうする?
```

生産者・消費者パターンでは、日常的に起きます。「ロードスレッドが確保し、メインスレッドが解放する」といった構造です。

### 3つの答え

**答え A:所有者のフリーリストに、直接返す。**

所有者のキャッシュを触ることになるので、**そのフリーリストにロックが必要** になります。「ロックを避ける」という目的が崩れます。

**答え B:解放したスレッドのキャッシュに入れてしまう。**

```cpp
// スレッド B が、スレッド A のブロックを自分のリストに繋ぐ
```

単純ですが、**致命的な問題があります。**

```
スレッド A: 確保するだけ  → チャンクを次々に消費
スレッド B: 解放するだけ  → ブロックが溜まり続ける
```

**A は中央からチャンクを取り続け、B にブロックが溜まり続けます。** メモリが片方に偏り、際限なく増えていきます。

この現象は **メモリのブローアップ** として知られています。**生産者・消費者パターンでは、必ず起きます。**

**答え C:所有者宛のキューに積む。** ← 採用

```cpp
struct ChunkHeader
{
    std::atomic<void*> remoteFree;   // 他スレッドからの返却リストの先頭
};
```

**解放するスレッドは、チャンクのヘッダにあるリストへ、アトミックに繋ぐだけです。**

```cpp
    void PushRemote(ChunkHeader* chunk, void* p) noexcept
    {
        void* head = chunk->remoteFree.load(std::memory_order_relaxed);

        for (;;)
        {
            *reinterpret_cast<void**>(p) = head;

            if (chunk->remoteFree.compare_exchange_weak(
                    head, p,
                    std::memory_order_release,
                    std::memory_order_relaxed))
            {
                return;
            }
            // 失敗したら head が更新されているので、やり直す
        }
    }
```

**所有者は、自分のフリーリストが空になったときに回収します。**

```cpp
    bool DrainRemote(std::size_t cls) noexcept
    {
        bool got = false;

        for (ChunkHeader* c = ownedChunks_[cls]; c != nullptr; c = c->nextChunk)
        {
            // リスト全体を、アトミックに横取りする
            void* list = c->remoteFree.exchange(nullptr, std::memory_order_acquire);

            while (list != nullptr)
            {
                void* next = *reinterpret_cast<void**>(list);

                *reinterpret_cast<void**>(list) = freeLists_[cls];
                freeLists_[cls] = list;

                list = next;
                got  = true;
            }
        }

        return got;
    }
```

### この方式の利点

**1. メモリが偏らない。** ブロックは必ず所有者のチャンクに戻ります。

**2. ロックが要らない。** CAS(compare-and-swap)1回だけです。

**3. 回収がまとめて行われる。** 所有者は、リスト全体を `exchange` で一度に横取りします。1個ずつ取るより、はるかに効率的です。

### CAS ループの説明

```cpp
compare_exchange_weak(head, p, ...)
```

**「`remoteFree` が `head` と同じなら `p` に書き換える。違えば `head` に現在値を入れて失敗を返す」** という操作です。

他のスレッドが同時に押し込んでいれば失敗しますが、そのときは更新された `head` で再試行します。**必ずどれかのスレッドが成功するので、全体としては進み続けます。**

`weak` を使っているのは、ループの中では「偽の失敗」が起きても再試行するだけなので、コストの低い版で十分だからです。

> **メモリ順序の指定については、第37章で詳しく扱います。** ここでは「押し込むときは `release`、取り出すときは `acquire`」という対応関係だけ覚えておいてください。**押し込む前に書いたデータが、取り出した側から見えること** を保証しています。

---

## 36.6 測る

### 単一スレッド

```
Pool(第22章)             2.9 ns
ThreadCache(この章)      3.4 ns
Bump + SpinLock(第35章)  6.2 ns
Bump + std::mutex        23.4 ns
```

**Pool より 0.5 ns 遅いだけです。** 増えたのは `thread_local` アクセス(0.7 ns)と、チャンク所有者の検査です。

### スケーリング(クロススレッド解放なし)

```
スレッド数   std::mutex(第35章)   ThreadCache      比
    1           42.7 M ops/s        294 M ops/s    6.9×
    2           28.4 M              586 M         20.6×
    4           21.2 M            1,164 M         54.9×
    8           17.9 M            2,398 M        134.0×
```

**8スレッドで 134 倍。** ほぼ線形にスケールしています。

### 中央ヒープを触る頻度

```cpp
std::println("確保回数    : {}", totalAllocations);
std::println("チャンク取得 : {}", ga::GlobalCentralHeap().AcquireCount());
```

```
確保回数    : 8000000
チャンク取得 : 8192
```

**976 回に 1 回。** ロックのコストが、そのぶん薄まっています。

### クロススレッド解放の割合を変える

```
解放が他スレッド   スループット(8スレッド)
      0%            2,398 M ops/s
     10%            1,940 M
     50%            1,180 M
    100%              612 M
```

**100% でも 612 M ops/s。** `std::mutex` の 17.9 M の **34 倍** です。

CAS 1 回のコストは、ロックの取得・解放より **はるかに安い** ということです。

### メモリ消費

```
8 スレッド × 8 サイズクラス × 64 KB = 4 MB(最悪時)
```

実際には、使われないサイズクラスのチャンクは確保されません。

**しかし、スレッド数が増えると比例して増えます。** 64 スレッドなら 32 MB です。**これがスレッドローカル化の代償です。**

---

## 36.7 偽共有への配慮

第35章で学んだことを、ここで適用します。

### `ChunkHeader` の分離

```cpp
struct alignas(64) ChunkHeader { ... };
```

`remoteFree` は **他スレッドが書き換えます**。もし他のチャンクのヘッダと同じキャッシュラインにあれば、無関係なチャンクどうしで偽共有が起きます。

**64 KB 境界にあるので、そもそも別のラインですが、明示しておきます。**

### `ThreadCache` を配列で持つ場合

本書では `thread_local` なポインタで各キャッシュを別々に確保しているので、問題は起きません。

**しかし、「スレッド番号で添字を引く配列」にすると、偽共有が起きます。**

```cpp
std::vector<ThreadCache> caches;      // ← 危ない
caches[threadIndex].Allocate(64);
```

`ThreadCache` は 128 バイト程度なので隣接します。**必ず `alignas` で分離してください。**

---

## 36.8 限界

正直に整理します。

### 大きなサイズを扱えない

サイズクラスは 2048 バイトまでです。それを超える確保は、別の経路(第27章の TLSF など)に渡す必要があります。

**実際のアロケーターもそうしています。** 小さいものはスレッドキャッシュ、大きいものは中央のヒープ、非常に大きいものは直接 `VirtualAlloc`。**第33章で見た Windows ヒープの構造と同じです。**

### メモリがスレッド数に比例する

36.6 節で見たとおりです。**スレッドが多い環境では、無視できない量になります。**

対策としては、

- 使われていないチャンクを、定期的に中央へ返す
- スレッド終了時に返す(36.2 節で触れた、破棄順序の問題に注意)

### スレッドをまたぐ寿命に弱い

クロススレッド解放は動きますが、**コストは 4 倍近くかかります**(36.6 節)。

**極端に生産者・消費者に偏った処理では、別の設計を検討すべきです。**

### そして——共有しないほうが速い

```
スレッドごとに別の Bump(第34章)   3,760 M ops/s
ThreadCache(この章)                2,398 M ops/s
```

**第34章で見た「共有しない」方式のほうが、まだ 1.6 倍速い。**

**当然です。** こちらはサイズクラスも、チャンクの管理も、所有者の検査もありません。ポインタを進めるだけです。

> **順序を守ってください。**
>
> 1. 共有しなくて済まないか(第34章)
> 2. 済まないなら、スレッドキャッシュ(この章)
> 3. それでも足りないなら、ロックフリー(第37章)
>
> **この章の設計は、「汎用アロケーターをマルチスレッド対応にする」ためのものです。** 用途が絞れるなら、1 が最良です。

---

## 演習

**演習36-1** `thread_local` にオブジェクトを直接置く版を実装し、速度を比べてください。差は 36.2 節と一致しますか。

**演習36-2** サイズクラスを 2 の冪から、第25章のような細かい刻みに変えてください。内部断片化とメモリ消費はどう変わりますか。

**演習36-3** 答え B(解放側のキャッシュに入れる)を実装し、生産者・消費者パターンで走らせてください。メモリはどれくらい増えますか。

**演習36-4** `DrainRemote` を、フリーリストが空になる前(たとえば残り 10 個を切ったら)に呼ぶようにしてください。性能は変わりますか。

**演習36-5** チャンクサイズを 16 KB / 256 KB に変えてください。中央ヒープを触る頻度と、メモリ消費はどう変わりますか。

**演習36-6** 空のチャンクを中央へ返す処理を実装してください。いつ返すべきですか。

**演習36-7** `ChunkHeader` から `alignas(64)` を外して、8スレッドで測ってください。差はありますか。ない場合、その理由を説明してください。

**演習36-8** 2048 バイトを超える確保を、第27章の `Tlsf`(ロック付き)に転送してください。全体の性能はどうなりますか。

---

## 章末チェックリスト

- [ ] 二層構造がロックのコストを薄める仕組みを説明できる
- [ ] `thread_local` にはポインタだけを置くべき理由を、実測で説明できる
- [ ] `thread_local` の破棄順序の問題を理解した
- [ ] **チャンクを 64 KB 境界に置くことで、ブロックのヘッダが不要になる** ことを説明できる
- [ ] `VirtualAlloc` の 64 KB 境界がこの設計を支えていることを理解した
- [ ] スレッドキャッシュと中央ヒープを実装した 〔v0.25〕
- [ ] クロススレッド解放の3つの答えと、それぞれの問題を説明できる
- [ ] **メモリのブローアップ** が起きる条件を説明できる
- [ ] CAS ループで、リストにアトミックに押し込む処理を書いた
- [ ] 8スレッドで 134 倍のスループットを確認した
- [ ] それでも「共有しない」方式のほうが速いことを確認した

---

## 次章の予告

第36章では、CAS を1か所だけ使いました。`remoteFree` への押し込みです。

**第37章では、アロケーターの中心部そのものをアトミックにします。**

```cpp
// v0.26:Bump をロックフリーにする
AllocResult Allocate(std::size_t size) noexcept
{
    std::size_t old = offset_.load(std::memory_order_relaxed);

    for (;;)
    {
        const std::size_t next = AlignUpSize(old, alignment) + size;
        if (next > capacity_) { return std::unexpected(AllocError::OutOfMemory); }

        if (offset_.compare_exchange_weak(old, next, ...)) { return ...; }
    }
}
```

**ロックがありません。** それでいて、第34章の実験は壊れなくなります。

扱うこと。

- `std::atomic` とメモリ順序(`relaxed` / `acquire` / `release` / `seq_cst`)
- CAS ループの正しい書き方
- **ABA 問題** ——ロックフリープログラミング最大の落とし穴
- そして、**ロックフリーは万能ではない** という結論

第35章で「ロックフリーが最強という直感はたいてい間違い」と書きました。**それを数字で確かめます。**

---

> **コラム:`thread_local` は、どう実装されているのか**
>
> 36.2 節で、`thread_local` へのアクセスに 0.71 ns、動的初期化を伴う場合は 2.48 ns かかることを測りました。**中で何が起きているのか。**
>
> ---
>
> **スレッドローカル記憶域(TLS)の基本**
>
> 各スレッドには、専用の小さな領域が割り当てられています。Windows では、スレッドごとの管理構造(TEB)から辿れる場所にあります。
>
> `thread_local` な変数は、この領域の中に置かれます。アクセスは、
>
> ```
> ① TEB から TLS 配列のポインタを読む
> ② モジュールごとのインデックスで引く
> ③ 変数のオフセットを足す
> ```
>
> **数命令で済みます。** これが 0.71 ns の内訳です。
>
> ---
>
> **静的 TLS と動的 TLS**
>
> 実行ファイルや、起動時に読み込まれる DLL の `thread_local` は、**静的 TLS** として扱われます。スレッド作成時に、まとめて領域が確保されます。**高速です。**
>
> 問題は、`LoadLibrary` で **実行中に読み込まれた DLL** です。すでに走っているスレッドの TLS 領域には、その DLL のぶんの場所がありません。
>
> Windows は、この場合のために **動的 TLS** の仕組みを持っています。初回アクセス時に領域を確保します。**そのぶん遅くなります。**
>
> 歴史的には、遅延読み込みされる DLL で `thread_local` を使うと問題が起きることがありました。プラグイン形式のアーキテクチャでは、注意が必要な領域です。
>
> ---
>
> **なぜ動的初期化は遅いのか**
>
> ```cpp
> thread_local ThreadCache g_cache;    // コンストラクタがある
> ```
>
> この変数にアクセスするたび、コンパイラは次のコードを生成します(概念)。
>
> ```cpp
> ThreadCache& GetCache()
> {
>     if (!g_initialized)        // ← スレッドごとのフラグ
>     {
>         new (&g_cache) ThreadCache();
>         RegisterDestructor(&g_cache);   // スレッド終了時に呼ぶ登録
>         g_initialized = true;
>     }
>     return g_cache;
> }
> ```
>
> **アクセスのたびに、フラグの検査が入ります。** さらに、この関数はモジュール境界をまたぐ可能性があるため、インライン展開されないことがあります。
>
> **だから「ポインタだけ置く」のです。** ポインタはゼロ初期化されるだけで、動的初期化が不要です。
>
> ---
>
> **デストラクタの登録**
>
> 上のコードにある `RegisterDestructor` も重要です。スレッドが終了するとき、登録された順の逆に呼ばれます。
>
> Windows では、DLL の `DllMain` に `DLL_THREAD_DETACH` という通知が来ます。ここで `thread_local` オブジェクトの破棄が行われます。
>
> **`DllMain` の中でできることは、厳しく制限されています。** ロックを取る、他の DLL を呼ぶ、メモリを確保する——どれも危険とされています。
>
> **「スレッド終了時に中央ヒープへ返す」処理が難しいのは、この制約のためでもあります。** 36.2 節で「あえて返さない」と決めたのは、この地雷原を避けるためです。
>
> ---
>
> **実用上の指針**
>
> - `thread_local` には、**ゼロ初期化できるもの** だけを置く
> - 実体は、初回アクセス時に `new` して、ポインタを保持する
> - **スレッド終了時の後始末は、できるだけ単純にする**
> - プラグイン形式の DLL で使う場合は、動作を確認する
>
> 単純な機能に見えて、裏側は複雑です。**「ポインタ1本」という制約を守るだけで、その複雑さのほとんどを回避できます。**
