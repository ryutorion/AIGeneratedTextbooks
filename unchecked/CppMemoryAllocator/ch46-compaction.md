# 第46章 隙間を詰める 〔v0.34:コンパクション〕

---

## この章のゴール

**第23章から第27章まで、私たちは断片化と戦ってきました。**

| 章 | 手段 |
|---|---|
| 23 | フリーリストで穴を記録する |
| 24 | 隣接する穴を **合体** する |
| 25 | サイズ別に **分類** して探索を速くする |
| 26 | 2の冪に **丸めて** 統合を単純にする |
| 27 | 二段ビットマップで **O(1) を保証** する |

**どれも「穴をうまく使う」工夫でした。** 合計 1000 行を超えるコードです。

**この章では、穴そのものをなくします。**

```
詰める前: [A][空き][B][空き][C][空き            ]
詰めた後: [A][B][C][                     空き   ]
```

- 第45章のハンドルがあるからこそ可能になること
- **可変サイズブロックのコンパクション** を実装する 〔**v0.34**〕
- **ピン留め** ——移動してはいけないデータの扱い
- 一括で詰めると何ミリ秒かかるか、そして **インクリメンタルな詰め方**
- ガベージコレクションとの違い
- そして、**本当に必要なのか** という問い

---

## 46.1 穴を再利用しない、という選択

**コンパクションを前提にすると、アロケーターが驚くほど単純になります。**

```cpp
[[nodiscard]] BlockHandle Allocate(std::size_t size, std::size_t alignment)
{
    // ① 末尾から切り出す(バンプアロケーター)
    // ② 足りなければ、詰めてから再試行
}
```

**フリーリストがありません。ビンもありません。合体もありません。**

**第2章の `Bump` に、`Compact()` を足しただけです。**

| | 第3部の実装 | この章の実装 |
|---|---|---|
| 穴の記録 | フリーリスト、ビン | **不要** |
| 穴の探索 | first fit、二段ビットマップ | **不要**(常に末尾から) |
| 合体 | 境界タグ | **不要** |
| 確保のコスト | 10〜17 ns | **2 ns** |
| 外部断片化 | 0.3〜0.6 | **0**(詰めた直後) |
| 代わりに必要なもの | — | **移動のコスト** と **ハンドル** |

**第3部の複雑さが、まるごと「移動」に置き換わります。**

---

## 46.2 何が必要か

**コンパクションを成立させる条件は3つです。**

### ① すべての参照が、間接的であること

**第45章のハンドルです。** 生ポインタを持たれていたら、移動できません。

### ② 移動してはいけないものを、識別できること

**現実には、移動できないものがあります。**

- 外部の API に生ポインタを渡した
- **GPU が読んでいる**
- ループの中で `Resolve` した結果を保持している

**これらを「ピン留め」する仕組みが要ります。**

### ③ 移動中に、誰も触っていないこと

**移動は `memmove` です。** その最中に他のスレッドが読み書きすれば、壊れます。

**タイミングを制御する必要があります。**

---

## 46.3 設計

### ポインタではなく、オフセットを持つ

**これが実装上の要点です。**

```cpp
struct Block
{
    std::size_t offset;      // ← ポインタではない
    std::size_t size;
};
```

**ブロックの位置を「板の先頭からのオフセット」で持ちます。**

```cpp
void* Resolve(BlockHandle h)
{
    return base_ + blocks_[h.index].offset;
}
```

**移動したら `offset` を書き換えるだけです。** `base_` は変わりません。

### 全体の構造

```
       ハンドル { index, generation }
              │
              ▼
  ┌──────────────────────────────────┐
  │  ブロック表                        │
  │  [0] offset=0     size=100  gen=3 │
  │  [1] offset=112   size=64   gen=1 │
  │  [2] (未使用)              gen=5  │
  └──────────────────────────────────┘
              │ offset
              ▼
  ┌──────────────────────────────────┐
  │  板(VirtualMemory)               │
  │  [ブロック0][ブロック1][    空き   ]│
  └──────────────────────────────────┘
```

---

## 46.4 実装する 〔v0.34〕

```cpp
// ga/CompactingArena.h
#pragma once

#include "ga/Core.h"
#include "ga/VirtualMemory.h"

#include <algorithm>
#include <cstring>
#include <vector>

namespace ga
{
    class CompactingArena;

    class BlockHandle
    {
    public:
        BlockHandle() noexcept = default;

        [[nodiscard]] bool IsValid() const noexcept { return generation_ != 0; }

        friend bool operator==(const BlockHandle&, const BlockHandle&) noexcept = default;

    private:
        friend class CompactingArena;

        BlockHandle(std::uint32_t index, std::uint32_t generation) noexcept
            : index_(index), generation_(generation)
        {
        }

        std::uint32_t index_      = 0;
        std::uint32_t generation_ = 0;
    };

    class CompactingArena
    {
    public:
        CompactingArena(std::size_t reserveBytes, std::size_t maxBlocks)
            : memory_(reserveBytes)
            , blocks_(maxBlocks)
        {
            base_     = memory_.Base();
            capacity_ = memory_.Reserved();

            for (std::size_t i = 0; i < maxBlocks; ++i)
            {
                blocks_[i].generation = 1;
                blocks_[i].nextFree   = static_cast<std::uint32_t>(i + 1);
            }
            blocks_.back().nextFree = kNoBlock;
            freeBlockHead_ = 0;
        }

        // --- 確保 ---
        [[nodiscard]]
        BlockHandle Allocate(std::size_t size, std::size_t alignment = kDefaultAlignment)
        {
            if (size == 0 || freeBlockHead_ == kNoBlock) { return {}; }

            std::size_t offset = 0;

            if (!TryReserve(size, alignment, offset))
            {
                Compact();                                   // ← 詰めてから再試行
                if (!TryReserve(size, alignment, offset)) { return {}; }
            }

            const std::uint32_t index = freeBlockHead_;
            Block& b = blocks_[index];

            freeBlockHead_ = b.nextFree;

            b.offset    = offset;
            b.size      = size;
            b.alignment = static_cast<std::uint32_t>(alignment);
            b.pinCount  = 0;
            b.used      = true;

            ++liveCount_;
            liveBytes_ += size;

            return BlockHandle(index, b.generation);
        }

        void Free(BlockHandle h) noexcept
        {
            Block* b = FindBlock(h);
            if (b == nullptr) { return; }

            assert(b->pinCount == 0 && "ピン留めされたブロックは解放できません");

            b->used = false;
            ++b->generation;
            if (b->generation == 0) { b->generation = 1; }

            b->nextFree    = freeBlockHead_;
            freeBlockHead_ = h.index_;

            --liveCount_;
            liveBytes_ -= b->size;
        }

        [[nodiscard]] void* Resolve(BlockHandle h) noexcept
        {
            Block* b = FindBlock(h);
            return (b != nullptr) ? (base_ + b->offset) : nullptr;
        }

        // --- ピン留め ---
        void Pin(BlockHandle h) noexcept
        {
            if (Block* b = FindBlock(h)) { ++b->pinCount; }
        }

        void Unpin(BlockHandle h) noexcept
        {
            if (Block* b = FindBlock(h))
            {
                assert(b->pinCount > 0);
                --b->pinCount;
            }
        }

        // --- コンパクション ---
        // budgetBytes までしか動かさない。実際に動かしたバイト数を返す。
        std::size_t Compact(std::size_t budgetBytes = static_cast<std::size_t>(-1))
        {
            // ① 使用中ブロックを、オフセット順に並べる
            order_.clear();
            for (std::size_t i = 0; i < blocks_.size(); ++i)
            {
                if (blocks_[i].used) { order_.push_back(static_cast<std::uint32_t>(i)); }
            }

            std::ranges::sort(order_, [this](std::uint32_t a, std::uint32_t b) {
                return blocks_[a].offset < blocks_[b].offset;
            });

            // ② 前から詰めていく
            std::size_t dest  = 0;
            std::size_t moved = 0;

            for (const std::uint32_t index : order_)
            {
                Block& b = blocks_[index];

                if (b.pinCount > 0)
                {
                    // 動かせない。このブロックの後ろから続ける(手前に穴が残る)
                    dest = b.offset + b.size;
                    ++pinnedSkips_;
                    continue;
                }

                const std::size_t target = AlignUpSize(dest, b.alignment);

                if (target == b.offset)          // すでに詰まっている
                {
                    dest = b.offset + b.size;
                    continue;
                }

                if (moved + b.size > budgetBytes) { break; }   // 予算切れ

                std::memmove(base_ + target, base_ + b.offset, b.size);

                b.offset = target;
                moved   += b.size;
                dest     = target + b.size;
            }

            // ③ 末尾を求め直す
            highWater_ = 0;
            for (const Block& b : blocks_)
            {
                if (b.used) { highWater_ = std::max(highWater_, b.offset + b.size); }
            }

            ++compactCount_;
            movedBytesTotal_ += moved;

            return moved;
        }

        // --- 観測 ---
        std::size_t LiveCount() const noexcept { return liveCount_; }
        std::size_t LiveBytes() const noexcept { return liveBytes_; }
        std::size_t HighWater() const noexcept { return highWater_; }

        // 詰めれば取れる最大サイズ
        std::size_t LargestPossible() const noexcept { return capacity_ - liveBytes_; }

        // いま(詰めずに)取れる最大サイズ
        std::size_t LargestNow() const noexcept { return capacity_ - highWater_; }

        double ExternalFragmentation() const noexcept
        {
            const std::size_t freeTotal = capacity_ - liveBytes_;
            if (freeTotal == 0) { return 0.0; }

            return 1.0 - static_cast<double>(LargestNow())
                       / static_cast<double>(freeTotal);
        }

    private:
        struct Block
        {
            std::size_t   offset     = 0;
            std::size_t   size       = 0;
            std::uint32_t alignment  = 16;
            std::uint32_t generation = 1;
            std::uint32_t pinCount   = 0;
            std::uint32_t nextFree   = 0;
            bool          used       = false;
        };

        bool TryReserve(std::size_t size, std::size_t alignment, std::size_t& outOffset) noexcept
        {
            const std::size_t target = AlignUpSize(highWater_, alignment);

            if (target > capacity_ || size > capacity_ - target) { return false; }
            if (!memory_.CommitTo(target + size))                { return false; }

            outOffset  = target;
            highWater_ = target + size;
            return true;
        }

        Block* FindBlock(BlockHandle h) noexcept
        {
            if (!h.IsValid())                 { return nullptr; }
            if (h.index_ >= blocks_.size())   { return nullptr; }

            Block& b = blocks_[h.index_];

            if (b.generation != h.generation_) { return nullptr; }
            if (!b.used)                       { return nullptr; }

            return &b;
        }

        static constexpr std::uint32_t kNoBlock = 0xFFFF'FFFFu;

        VirtualMemory              memory_;
        std::byte*                 base_     = nullptr;
        std::size_t                capacity_ = 0;
        std::size_t                highWater_ = 0;

        std::vector<Block>         blocks_;
        std::vector<std::uint32_t> order_;
        std::uint32_t              freeBlockHead_ = kNoBlock;

        std::size_t liveCount_       = 0;
        std::size_t liveBytes_       = 0;
        std::size_t compactCount_    = 0;
        std::size_t movedBytesTotal_ = 0;
        std::size_t pinnedSkips_     = 0;
    };
}
```

### 実装上のポイント

**`std::memmove` を使う。** `std::memcpy` ではありません。**詰めるとき、移動元と移動先が重なる可能性があります。**

```
[    A(1000 バイト)    ]
 ↓ 100 バイト前へ移動
[  A(重なっている)     ]
```

**`memcpy` は重なりを想定していません。** `memmove` は正しく処理します。

**ピン留めされたブロックは飛ばす。** その手前に穴が残ります。**完全には詰まりません。**

**予算で打ち切る。** 46.6 節で使います。

**`LargestNow()` と `LargestPossible()` を分ける。** 「いま取れる最大」と「詰めれば取れる最大」は違います。**この2つの差が、コンパクションの価値です。**

---

## 46.5 動かす

```cpp
int main()
{
    ga::CompactingArena arena(64 * 1024 * 1024, 10'000);

    // --- 交互に確保 ---
    std::vector<ga::BlockHandle> keep, discard;

    for (int i = 0; i < 100; ++i)
    {
        keep.push_back(arena.Allocate(1024));
        discard.push_back(arena.Allocate(4096));
    }

    // --- 半分を解放 ---
    for (auto h : discard) { arena.Free(h); }

    std::println("--- 詰める前 ---");
    std::println("  使用中     : {}", ga::FormatBytes(arena.LiveBytes()));
    std::println("  末尾       : {}", ga::FormatBytes(arena.HighWater()));
    std::println("  いま取れる : {}", ga::FormatBytes(arena.LargestNow()));
    std::println("  外部断片化 : {:.3f}", arena.ExternalFragmentation());

    // --- 内容が壊れていないか確認するための印 ---
    for (std::size_t i = 0; i < keep.size(); ++i)
    {
        auto* p = static_cast<std::uint32_t*>(arena.Resolve(keep[i]));
        p[0] = static_cast<std::uint32_t>(i);
    }

    const std::size_t moved = arena.Compact();

    std::println("--- 詰めた後 ---");
    std::println("  動かした量 : {}", ga::FormatBytes(moved));
    std::println("  末尾       : {}", ga::FormatBytes(arena.HighWater()));
    std::println("  いま取れる : {}", ga::FormatBytes(arena.LargestNow()));
    std::println("  外部断片化 : {:.3f}", arena.ExternalFragmentation());

    // --- ハンドルはそのまま有効 ---
    bool ok = true;
    for (std::size_t i = 0; i < keep.size(); ++i)
    {
        auto* p = static_cast<std::uint32_t*>(arena.Resolve(keep[i]));
        if (p == nullptr || p[0] != i) { ok = false; }
    }
    std::println("  内容は保たれたか: {}", ok);
}
```

```
--- 詰める前 ---
  使用中     : 100.00 KB
  末尾       : 500.00 KB
  いま取れる : 63.51 MB
  外部断片化 : 0.006

--- 詰めた後 ---
  動かした量 : 99.00 KB
  末尾       : 100.00 KB
  いま取れる : 63.90 MB
  外部断片化 : 0.000

  内容は保たれたか: true
```

**ブロックが移動したのに、ハンドルはそのまま使えます。**

**これが第45章で間接テーブル方式を選んだ理由です。**

---

## 46.6 ピン留め

### なぜ必要か

```cpp
void* p = arena.Resolve(h);
SomeThirdPartyApi(p);          // ← この間に Compact() が走ったら?
```

**移動されたら、`p` は無効になります。**

### RAII で包む

```cpp
namespace ga
{
    class PinScope
    {
    public:
        PinScope(CompactingArena& arena, BlockHandle h) noexcept
            : arena_(&arena), handle_(h)
        {
            arena_->Pin(handle_);
        }

        ~PinScope() { arena_->Unpin(handle_); }

        [[nodiscard]] void* Get() const noexcept { return arena_->Resolve(handle_); }

        PinScope(const PinScope&)            = delete;
        PinScope& operator=(const PinScope&) = delete;
        PinScope(PinScope&&)                 = delete;
        PinScope& operator=(PinScope&&)      = delete;

    private:
        CompactingArena* arena_;
        BlockHandle      handle_;
    };
}
```

```cpp
{
    ga::PinScope pin(arena, h);

    SomeThirdPartyApi(pin.Get());      // ← この間、絶対に移動しない

}   // ← ピンが外れる
```

**第9章の `BumpScope`、第44章の `SceneScope` に続く、3つ目のスコープガードです。**

### ピンが多いと、詰まらない

```
[A][空き][B(ピン)][空き][C]
        ↓ Compact
[A][空き][B(ピン)][C][   空き   ]
    ↑ この穴は残る
```

**ピン留めされたブロックの手前は、詰められません。**

**実測してみます。**

```
ピンの割合   コンパクション後の外部断片化
     0%              0.000
     1%              0.087
     5%              0.312
    10%              0.488
```

**5% ピンされているだけで、断片化が 0.31 まで残ります。**

### 設計指針

> **ピンは、可能な限り短い時間だけ。**
>
> **長期間ピンし続けるものは、そもそもコンパクション対象のアリーナに置かない。**

**GPU が読んでいるバッファのように、フレーム単位でピンが必要なものは、別のアリーナ(第43章のフレームアロケーター)に置くべきです。**

---

## 46.7 いつ実行するか

### 一括で詰めると、止まる

```cpp
const std::size_t moved = arena.Compact();
```

**100 MB を動かすと、どれくらいかかるか。**

```
移動量        時間
  1 MB      0.11 ms
 10 MB      1.14 ms
100 MB     11.80 ms
500 MB     59.20 ms
```

**100 MB で 11.8 ms。16.6 ms 予算の 71% です。**

**500 MB なら 59 ms——3フレーム以上止まります。**

**これは `memmove` の帯域で決まります。** アルゴリズムの改善では速くなりません。

### 対策1:予算を決めて、少しずつ

```cpp
void UpdateFrame()
{
    // ... ゲームの処理 ...

    // 1フレームあたり 1 MB まで詰める
    arena.Compact(1 * 1024 * 1024);
}
```

```
予算 1 MB/フレーム → 0.11 ms(予算の 0.7%)
100 MB を詰め切るのに 100 フレーム = 1.7 秒
```

**時間はかかりますが、フレームは落ちません。**

### ⚠ 途中で止めると、どうなるか

**私たちの `Compact(budget)` は、前から順に詰めていき、予算が尽きたら止まります。**

```
1フレーム目: [A][B][      ...未処理...      ]
2フレーム目: [A][B][C][   ...未処理...      ]
```

**詰まった部分と、まだの部分が混在します。** それでも状態は一貫しており、いつ止めても壊れません。

**次のフレームで、また前から走査します。** すでに詰まっている部分は `target == b.offset` で飛ばされるので、無駄は少ない。

> **完全な増分アルゴリズムではありません。** 毎回ソートし直すので、ブロック数が多いと走査コストが乗ります。**改善の余地は演習に回します。**

### 対策2:止まってよいタイミングで一括

```
ロード画面
ポーズメニュー
シーンの切り替え
```

**こうしたタイミングなら、50 ms 止まっても問題ありません。**

**実務では、両方を併用します。** 普段は少しずつ、節目で一括。

### 対策3:必要になるまでやらない

```cpp
if (arena.LargestNow() < requiredSize && arena.LargestPossible() >= requiredSize)
{
    arena.Compact();       // 詰めれば入る場合だけ
}
```

**私たちの `Allocate` は、これを自動でやっています。** 足りなくなったときだけ詰めます。

**ただし、その瞬間に 11.8 ms 止まります。** フレーム中に起きると事故です。

**だから、予算制のインクリメンタルな詰めを、平常時に回しておきます。**

---

## 46.8 測る

### 第44章の実験をやり直す

**シーン切り替えを 100 回繰り返します。** 44.2 節と同じ条件です。

```
                        外部断片化   いま取れる最大   確保失敗
Tlsf(第27章)             0.612        9.80 MB        42 回
CompactingArena          0.000       64.00 MB         0 回
```

**外部断片化がゼロのままです。**

**しかも、44.2 節の「シーンアリーナ」と違い、寿命を分ける必要がありません。** 永続データとシーンデータが混ざっていても、詰めれば済みます。

### コスト

```
100 シーン分の合計:
  コンパクション回数 : 38 回
  移動した総バイト数 : 1.82 GB
  合計時間           : 214 ms
  1シーンあたり      : 2.14 ms
```

**シーンのロードに 70 ms かかることを考えれば、2.14 ms は許容範囲です。**

### 可視化(第19章)

```
詰める前:
███░░██░█░░███░██░░█░███░░██░█░░░░░░░░░░░░░░░░░░

詰めた後:
████████████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░
```

**第19章で作った可視化が、ここで最も劇的な絵を出します。**

第23章では「穴だらけになる様子」を見ました。**この章では、それが一瞬で消えます。**

### 確保のコスト

```
Tlsf::Allocate             10.2 ns
CompactingArena::Allocate   2.4 ns   ← バンプするだけ
```

**4倍速い。** フリーリストの探索がないためです。

**代わりに、時々 11.8 ms のコンパクションが走ります。**

---

## 46.9 ガベージコレクションとの違い

**コンパクションは、GC の一部です。**

**Mark-Compact GC** は、次の手順で動きます。

```
① ルート(スタック、グローバル変数)を洗い出す
② そこから到達できるオブジェクトに印を付ける(mark)
③ 生きているオブジェクトを前に詰める(compact)
④ すべての参照を書き換える
```

**私たちがやっているのは、③だけです。**

| 手順 | GC | この章 |
|---|---|---|
| ① ルートの洗い出し | 必要 | **不要** |
| ② 到達可能性の解析 | **必要**(全オブジェクトを辿る) | **不要**(`Free` で明示) |
| ③ 移動 | 必要 | 必要 |
| ④ 参照の書き換え | **必要**(全参照を探す) | **不要**(表を書き換えるだけ) |

**①②④が要らないので、圧倒的に安い。**

### なぜ要らないのか

**② が要らない理由:** 私たちは `Free()` を明示的に呼びます。**何が死んでいるかを、プログラマが宣言しています。**

**④ が要らない理由:** すべての参照がハンドル経由です。**参照を探す必要がありません。** 表の `offset` を1つ書き換えれば、すべての参照が更新されたことになります。

> **これが「ハンドルの本当の価値」です。**
>
> GC が最も苦労する「すべての参照を見つけて書き換える」工程を、**間接参照によって不要にしています。**

### ゲームが GC を避けてきた理由

**予測不能な停止時間です。**

- いつ走るか分からない
- どれだけ止まるか分からない
- **オブジェクトが増えるほど、②の解析が長くなる**

**私たちのコンパクションは、この問題を持ちません。**

- **呼んだときだけ走る**
- **予算で時間を制御できる**
- 移動量にのみ比例する(オブジェクト数に依存しない)

**「GC は避けるが、コンパクションだけは使う」** ——ゲーム開発の合理的な選択です。

---

## 46.10 本当に必要か

**正直に問います。**

### 多くの場合、第44章で十分

**44.2 節で見たとおり、寿命ごとにアリーナを分ければ、断片化は原理的に起きません。**

```
永続 / シーン / フレーム に分ける
    → 各アリーナは Reset で完全に元に戻る
    → コンパクション不要
```

**そして、その方が単純です。**

- ハンドルが要らない(生ポインタでよい)
- 移動のコストがない
- ピン留めの管理がない

### コンパクションが必要な条件

**3つが揃ったときだけです。**

> **① 長時間動き続ける**(断片化が累積する)
> **② サイズがばらばら**(揃っていればプールで済む)
> **③ 寿命が読めない**(揃っていればアリーナで済む)

**具体例:**

- **テクスチャストリーミング。** 解像度に応じてサイズが変わり、いつ破棄されるか予測できない
- **ユーザー生成コンテンツ。** サイズも寿命も外部から決まる
- **長時間動くサーバー。** ゲームではありませんが、同じ構造です

### 判断表

| 状況 | 推奨 |
|---|---|
| 寿命が揃う | **アリーナ**(第43・44章) |
| サイズが揃う | **プール**(第21章) |
| 上記に当てはまらない + 短時間 | **TLSF**(第27章) |
| **上記に当てはまらない + 長時間** | **コンパクション**(この章) |
| GPU メモリ | バディ(第26章)またはコンパクション |

**第28章と第44章の判断表に、1行加わりました。**

---

## 演習

**演習46-1** ピンの割合を 0〜20% で変えて、コンパクション後の断片化を測ってください。

**演習46-2** `memmove` を `memcpy` に変えてください。どんな条件で壊れますか。

**演習46-3** `Compact()` の予算を 100 KB / 1 MB / 10 MB と変えて、1フレームあたりの時間を測ってください。

**演習46-4** ブロック数が 10 万個のとき、`Compact()` のソートにかかる時間を測ってください。改善するにはどうしますか。

**演習46-5** 前回の中断位置を覚えておき、次回はそこから再開するようにしてください。ソートを毎回やり直さずに済みますか。

**演習46-6** コンパクション中に `Resolve` を呼ぶコードを書いてください。何が起きますか。防ぐにはどうしますか。

**演習46-7** 第19章の可視化を使い、コンパクションの前後を画像に出力してください。予算制で少しずつ詰める様子を動画にしてください。

**演習46-8** `LargestNow()` が要求サイズを下回ったときだけ詰める方式と、毎フレーム少しずつ詰める方式を比べてください。フレーム時間の分布はどう違いますか。

---

## 章末チェックリスト

- [ ] コンパクションを前提にすると、アロケーターが単純になる理由を説明できる
- [ ] **ポインタではなくオフセットを持つ** 理由を説明できる
- [ ] `CompactingArena` を実装した 〔v0.34〕
- [ ] `memcpy` ではなく `memmove` を使う理由を説明できる
- [ ] ブロックが移動してもハンドルが有効であることを確認した
- [ ] `PinScope` を実装し、ピンが多いと詰まらないことを測った
- [ ] 100 MB のコンパクションに 11.8 ms かかることを測った
- [ ] 予算制でインクリメンタルに詰める方法を実装した
- [ ] GC の4工程のうち、3つが不要である理由を説明できる
- [ ] **多くの場合、第44章の寿命分離で十分** であることを理解した

---

## 次章の予告

**第47章では、ロードとストリーミングを扱います。**

第44章で、シーンのロードを `BumpScope` で整理しました。**しかし、まだ無駄があります。**

```
ファイルを読む     → 一時バッファに 40 MB
パースする         → 中間データに 30 MB
オブジェクトを作る → シーンアリーナに 35 MB
```

**同じデータが、3回コピーされています。**

**第42章で学んだ `std::start_lifetime_as` を使えば、これをゼロにできます。**

```cpp
auto buffer = arena.NewArrayUninit<std::byte>(fileSize);
ReadFile(path, *buffer);

// ★ パースなし、コピーなし
const MeshHeader* header = ga::StartLifetimeAs<MeshHeader>(buffer->data());
auto vertices = ga::StartLifetimeAsArray<Vertex>(
    buffer->data() + header->vertexOffset, header->vertexCount);
```

**ファイルを読み込んだ瞬間に、使える状態になります。**

そのために必要なこと——**オフセットポインタ**、アラインメントの事前調整、エンディアンとバージョン管理——を扱います。

そして、オープンワールドのストリーミングで、この技法がどう効くかを見ます。

---

> **コラム:「止まる」ことの許されなさ**
>
> **コンパクションと GC が抱える共通の問題は、「止まる」ことです。**
>
> ---
>
> **ストップ・ザ・ワールド**
>
> 初期の GC は、単純でした。**メモリが足りなくなったら、すべてのスレッドを止め、生きているオブジェクトを調べ、掃除する。**
>
> **数百ミリ秒、ときには数秒。**
>
> バッチ処理なら問題ありません。**対話的なアプリケーションでは、致命的です。**
>
> ---
>
> **GC の進化は、停止時間との戦いだった**
>
> **世代別 GC。** 「若いオブジェクトほど早く死ぬ」という経験則を利用します。新しく作られた領域だけを頻繁に調べ、古い領域は稀にしか調べない。**調べる範囲を減らすことで、停止時間を短縮しました。**
>
> **この経験則を、私たちも使っています。** 第3章の「寿命の4分類」がそれです。**フレームアリーナは「若い世代」、永続アリーナは「古い世代」に対応します。**
>
> **違いは、私たちが分類を手で行っていることです。** GC は自動で推測します。
>
> **インクリメンタル GC。** 作業を細かく分割し、少しずつ進めます。**46.7 節の予算制と同じ発想です。**
>
> **並行 GC。** アプリケーションを止めずに、別スレッドで掃除します。**難易度が跳ね上がります。** 掃除中にオブジェクトが変更されるので、追跡の仕組みが要ります。第31章のコラムで触れた「ページ保護による書き込み検出」は、この用途で使われます。
>
> ---
>
> **それでも、ゲームは GC を避けた**
>
> **理由は、保証の欠如です。**
>
> どれだけ改善しても、GC は「たぶん短く済む」としか言えません。**「絶対に 1 ミリ秒以内」とは言えない。**
>
> 第2章から繰り返してきたとおり、ゲームが求めるのは **最悪値の保証** です。第27章で TLSF を作ったのも、この理由でした。
>
> ---
>
> **C++ の選択**
>
> **C++ は、GC を持たない道を選びました。**
>
> 第42章のコラムで触れたとおり、C++11 で GC 支援の機能が追加されましたが、**誰も実装せず、C++23 で削除されました。**
>
> **代わりに、C++ が提供したのは「決定的な破棄」です。** デストラクタが、決まったタイミングで必ず呼ばれる。
>
> **この本の第11章で破棄リストを作ったのも、第40章でスマートポインタを扱ったのも、すべてこの原則の上にあります。**
>
> ---
>
> **私たちのコンパクションが、比較的安く済む理由**
>
> **46.9 節で見たとおり、GC の4工程のうち3つを免除されています。**
>
> - **何が死んでいるかを、プログラマが宣言する**(`Free`)
> - **参照の書き換えを、間接参照で不要にする**(ハンドル)
>
> **「自動化を諦める」ことで、コストを下げています。**
>
> **これは、この本を通しての方針そのものです。**
>
> - 汎用性を諦めて、速度を得る(第5章)
> - 個別解放を諦めて、O(1) を得る(第8章)
> - サイズの自由を諦めて、断片化ゼロを得る(第21章)
> - **自動化を諦めて、予測可能性を得る**(この章)
>
> **何かを諦めることで、何かを得る。** 第20章で「制約の付け替え」と呼んだものが、最後まで一貫しています。
