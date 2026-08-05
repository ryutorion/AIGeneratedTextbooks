# 第27章 最悪時間を保証する 〔v0.20:TLSF〕

---

## この章のゴール

第3部の最後のアロケーターです。

ここまでの実装には、共通の弱点があります。**最悪実行時間が読めない。**

| | 最悪ケースで起きること |
|---|---|
| FreeList v0.18 | 大きいビンの中を線形に探索する |
| Buddy | 空きのある order を線形に走査し、統合が最大15段連鎖する |

平均は十分に速いのですが、**「絶対にこの時間内に終わる」という保証がありません**。第2章から繰り返している「平均ではなく最悪値」という視点で見ると、まだ不十分です。

**TLSF**(Two-Level Segregated Fit)は、この保証を与えます。

- 第25章の一段構造を **二段** に拡張する
- **要求を切り上げてクラスを決める** という、TLSF の核心的な工夫
- 二段ビットマップを **ビットスキャン2回** で走査する 〔**v0.20**〕
- 分岐もループもない、**厳密に O(1) の確保**
- 第5章のヒストグラムを再実行し、最悪値がどこまで平らになるかを見る

---

## 27.1 なぜ第25章は O(1) でないのか

第25章のビン分割を思い出してください。

**小さいビンは完璧でした。** すべてのブロックサイズが 16 の倍数なので、`size / 16` がサイズを一意に決めます。ビン4に入っているブロックは、すべてちょうど 64 バイト。**先頭を取れば必ず合います。**

**大きいビンが問題です。**

```
ビン65: [2048, 4095] の範囲
  → 中身は 2048, 2560, 3000, 4000 …とばらばら
  → 3000 バイトが欲しいとき、中を探さなければならない
```

第25章では、この探索を「大きいブロックは数が少ないから実害は小さい」として受け入れました。**平均で 1.3 回**。悪くない数字です。

しかし、**最悪では何回になるか分かりません**。ビン65に空きブロックが 1000 個溜まっていたら、1000 回走査することになります。

### 素朴な解決策と、その問題

「大きいビンも細かく分ければいい」——そのとおりです。

しかし、2048〜4095 を 16 バイト刻みにすると、この範囲だけで 128 個のビンが必要です。全体では数千個。ビン配列が大きくなりすぎますし、**空でないビンを探す走査** も長くなります。

**細かさと、走査の速さを両立させる必要があります。**

---

## 27.2 二段に分ける

TLSF の第一の工夫です。サイズクラスを2つの階層で表します。

**第一段(FL):2の冪** — どの桁か

**第二段(SL):その範囲を等分** — 桁の中のどこか

```
サイズ 3000 の場合:

  第一段: 2048 〜 4095 の範囲     (fl)
  第二段: その範囲を32等分した何番目 (sl)
          範囲の幅 2048 ÷ 32 = 64 バイト刻み
          (3000 - 2048) / 64 = 14 番目
```

```
fl=11 (2048〜4095)
 ├── sl=0  : 2048〜2111
 ├── sl=1  : 2112〜2175
 ├── ...
 ├── sl=14 : 2944〜3007   ← 3000 はここ
 └── sl=31 : 4032〜4095
```

### なぜ二段が効くのか

**刻みが、サイズに比例します。**

| 範囲 | 1クラスの幅 | 相対的な粒度 |
|---|---|---|
| 512〜1023 | 16 バイト | 1/32 |
| 2048〜4095 | 64 バイト | 1/32 |
| 1M〜2M | 32 KB | 1/32 |

**どの桁でも、常に 1/32 の精度です。**

第25章では「小さいところは 16 バイト刻み、大きいところは 2 の冪刻み」と、粒度が桁によって変わっていました。TLSF は **全域で一定の相対精度** を保ちます。

そして、クラスの総数は `FL の数 × 32` です。FL が 25 個なら 800 個。**多いようですが、次節の仕掛けで走査コストは変わりません。**

### 写像を実装する

```cpp
    static constexpr int kAlignShift = 4;                      // 16 バイト
    static constexpr int kSlShift    = 5;                      // 32 分割
    static constexpr int kSlCount    = 1 << kSlShift;          // 32
    static constexpr int kFlShift    = kSlShift + kAlignShift; // 9
    static constexpr std::size_t kSmallBlock = std::size_t{1} << kFlShift;  // 512
    static constexpr int kFlCount    = 25;

    static void MappingInsert(std::size_t size, int& fl, int& sl) noexcept
    {
        if (size < kSmallBlock)
        {
            // 512 バイト未満は、16 バイト刻みで 32 クラス
            fl = 0;
            sl = static_cast<int>(size >> kAlignShift);
        }
        else
        {
            const int w = static_cast<int>(std::bit_width(size)) - 1;   // floor(log2)

            sl = static_cast<int>((size >> (w - kSlShift)) & (kSlCount - 1));
            fl = w - (kFlShift - 1);
        }
    }
```

小さいサイズを特別扱いしているのは、512 バイト未満では「32等分」が 16 バイトを下回ってしまうためです。アラインメントより細かく分けても意味がありません。

**この特別扱いにより、512 バイト未満は第25章の小さいビンとまったく同じ動作になります。**

---

## 27.3 切り上げるという発明

TLSF の核心は、実はこちらです。

### 問題

`fl`, `sl` を計算してクラスを特定しても、**そのクラスの中のブロックが必ず入るとは限りません**。

```
3000 バイトが欲しい → クラス (fl=11, sl=14) = [2944, 3007]
  このクラスには 2944 バイトのブロックが入っているかもしれない
  → 3000 は入らない!
```

**結局、中を探すことになります。**

### 解決策

**要求サイズを、そのクラスの上端まで切り上げてから、クラスを決めます。**

```cpp
    static void MappingSearch(std::size_t size, int& fl, int& sl) noexcept
    {
        if (size >= kSmallBlock)
        {
            const int w = static_cast<int>(std::bit_width(size)) - 1;

            // このクラスの幅 - 1 を足す
            size += (std::size_t{1} << (w - kSlShift)) - 1;
        }

        MappingInsert(size, fl, sl);
    }
```

3000 の場合:

```
w = 11、クラスの幅 = 2^(11-5) = 64
3000 + 63 = 3063
  → MappingInsert(3063) → fl=11, sl=15 (範囲 [3008, 3071])
```

**クラス (11, 15) に入っているブロックは、すべて 3008 バイト以上です。3000 は必ず入ります。**

### 2つの写像

TLSF は、**用途によって写像を使い分けます**。

| 写像 | 用途 | 動作 |
|---|---|---|
| `MappingInsert` | ブロックをクラスに **入れる** | 切り下げ(そのサイズが属するクラス) |
| `MappingSearch` | 要求に合うクラスを **探す** | 切り上げ(必ず入るクラス) |

**この非対称性が、O(1) を実現しています。**

### 代償は何か

「切り上げるなら、内部断片化が増えるのでは?」——**増えません。**

切り上げは **「どのクラスを探すか」を決めるだけ** です。実際に取ったブロックは、第24章と同じように分割します。3000 バイトの要求に 3500 バイトのブロックが当たれば、3000 と 500 に分けます。**消費するのは 3000 バイトのままです。**

**本当の代償は別のところにあります。**

```
3000 バイトの要求
  → クラス (11, 14) = [2944, 3007] を飛ばして (11, 15) から探す
  → もし (11, 14) に 3007 バイトのブロックがあれば、それは入るのに見送られる
```

**入るはずのブロックを、見送ることがあります。** 見送る範囲は最大でクラス1つ分、つまり **サイズの 1/32(約 3.1%)** です。

バディが「最悪 50% の内部断片化」を払ったのに対し、TLSF は「最悪 3.1% の機会損失」で済んでいます。**同じ切り上げの発想を、はるかに細かい粒度で行った結果です。**

---

## 27.4 二段ビットマップを走査する

クラスが 800 個あっても、走査は速くなければなりません。

**ビットマップを二段にします。**

```cpp
    std::uint32_t flBitmap_ = 0;              // どの fl に空きがあるか(25 ビット)
    std::uint32_t slBitmap_[kFlCount] = {};   // 各 fl の中で、どの sl に空きがあるか(32 ビット)
```

**探索の手順は、たった2回のビットスキャンです。**

```cpp
    Header* FindBlock(int& fl, int& sl) const noexcept
    {
        // ① 同じ fl の中で、sl 以上の空きクラスを探す
        std::uint32_t slMap = slBitmap_[fl] & (~std::uint32_t{0} << sl);

        if (slMap == 0)
        {
            // ② なければ、fl より大きい空きの段を探す
            const std::uint32_t flMap = flBitmap_ & (~std::uint32_t{0} << (fl + 1));

            if (flMap == 0) { return nullptr; }       // 空きなし

            fl    = std::countr_zero(flMap);
            slMap = slBitmap_[fl];
        }

        sl = std::countr_zero(slMap);
        return blocks_[fl][sl];
    }
```

**ループがありません。分岐は1つだけです。**

- `~0u << sl` で、`sl` 未満のビットを消す
- `std::countr_zero` で、最下位の 1 の位置を得る

**これで 800 個のクラスから、条件に合う最小のクラスが一発で見つかります。**

第25章では128個のビンを最悪2ワード走査していました。TLSF は、クラス数が6倍以上に増えたのに、**走査回数は減っています**。

### `kFlCount` が 32 未満である理由

`~0u << (fl + 1)` という式に注意してください。`fl + 1` が 32 以上になると、シフト量が型の幅を超えて **未定義動作** になります。

`kFlCount = 25` としているので `fl + 1 <= 25` であり、この問題は起きません。**制約が実装の安全性を担保している** 例です。

### MSVC の組み込み関数

`std::countr_zero` と `std::bit_width` は C++20 の標準機能です。MSVC では、対応する CPU 命令に落ちます。

```cpp
#include <intrin.h>

unsigned long index;
_BitScanForward64(&index, value);   // 最下位の 1 のビット位置
_BitScanReverse64(&index, value);   // 最上位の 1 のビット位置
```

これらが、標準関数の裏で使われている命令に相当します(`bsf` / `bsr`、あるいは新しい CPU では `tzcnt` / `lzcnt`)。

**しかし、本書では標準関数を使います。** 理由が2つあります。

**1. ゼロの扱いが明確。** `_BitScanForward` は、値が 0 のとき戻り値が 0 になり、`index` は **未定義のまま** です。使う側が毎回チェックしなければなりません。`std::countr_zero(0)` は、型のビット数を返すと定められています。

**2. 移植性。** `_BitScan*` は MSVC 固有です。標準関数なら、そのまま他の環境でも動きます。

> **生成されるコードは同じです。** 逆アセンブルを見れば、どちらも `tzcnt` 命令1つになっているはずです(演習27-3)。**移植性と安全性を、性能を犠牲にせずに手に入れられます。**

---

## 27.5 実装する 〔v0.20〕

ブロックの構造は **第24章とまったく同じ** です。境界タグ、`PREV_FREE` フラグ、空きブロック内の `prevFree`、フッタ。合体のロジックもそのまま使えます。

**変わるのは、フリーリストの持ち方と探し方だけです。**

```cpp
// ga/Tlsf.h
#pragma once

#include "ga/Core.h"
#include "ga/detail/BlockHeader.h"    // 第24章のヘッダ定義を共有

#include <bit>
#include <cstdint>

namespace ga
{
    class Tlsf
    {
    public:
        static constexpr int kAlignShift = 4;
        static constexpr int kSlShift    = 5;
        static constexpr int kSlCount    = 1 << kSlShift;
        static constexpr int kFlShift    = kSlShift + kAlignShift;
        static constexpr int kFlCount    = 25;

        static constexpr std::size_t kSmallBlock   = std::size_t{1} << kFlShift;
        static constexpr std::size_t kHeaderSize   = 16;
        static constexpr std::size_t kMinBlockSize = 32;

        explicit Tlsf(std::size_t capacity);

        [[nodiscard]] void* Allocate(std::size_t size) noexcept
        {
            if (size == 0) { return nullptr; }

            const std::size_t payload = AlignUpSize(size, kDefaultAlignment);
            if (payload < size) { return nullptr; }

            std::size_t need = payload + kHeaderSize;
            if (need < payload)       { return nullptr; }
            if (need < kMinBlockSize) { need = kMinBlockSize; }

            int fl = 0, sl = 0;
            MappingSearch(need, fl, sl);

            Header* h = FindBlock(fl, sl);
            if (h == nullptr) { ++failures_; return nullptr; }

            assert(detail::SizeOf(h) >= need);   // 切り上げの効果:必ず入る

            RemoveBlock(h, fl, sl);

            const std::size_t blockSize = detail::SizeOf(h);

            if (blockSize - need >= kMinBlockSize)
            {
                MarkUsed(h, need);

                Header* rest = NextOf(h);
                MarkFree(rest, blockSize - need);
                InsertBlock(rest);
            }
            else
            {
                MarkUsed(h, blockSize);
                internalWaste_ += blockSize - need;
            }

            ++liveCount_;
            return BytesOf(h) + kHeaderSize;
        }

        void Free(void* p) noexcept
        {
            if (p == nullptr) { return; }
            if (!Owns(p))     { ReportError(p); return; }

            Header* h = HeaderOf(p);
            if (detail::IsFree(h)) { ReportError(p); return; }

            --liveCount_;

            std::size_t size = detail::SizeOf(h);

            // --- 第24章とまったく同じ合体処理 ---
            if (Header* next = NextOf(h); next != nullptr && detail::IsFree(next))
            {
                RemoveBlock(next);
                size += detail::SizeOf(next);
            }

            if (detail::PrevIsFree(h))
            {
                const std::size_t prevSize = detail::PrevFooterOf(h);
                Header* prev = reinterpret_cast<Header*>(BytesOf(h) - prevSize);

                RemoveBlock(prev);
                size += prevSize;
                h = prev;
            }

            MarkFree(h, size);
            InsertBlock(h);
        }

    private:
        // ---- 2つの写像 ----

        static void MappingInsert(std::size_t size, int& fl, int& sl) noexcept
        {
            if (size < kSmallBlock)
            {
                fl = 0;
                sl = static_cast<int>(size >> kAlignShift);
            }
            else
            {
                const int w = static_cast<int>(std::bit_width(size)) - 1;
                sl = static_cast<int>((size >> (w - kSlShift)) & (kSlCount - 1));
                fl = w - (kFlShift - 1);
            }
        }

        static void MappingSearch(std::size_t size, int& fl, int& sl) noexcept
        {
            if (size >= kSmallBlock)
            {
                const int w = static_cast<int>(std::bit_width(size)) - 1;
                size += (std::size_t{1} << (w - kSlShift)) - 1;
            }
            MappingInsert(size, fl, sl);
        }

        // ---- 二段ビットマップ ----

        Header* FindBlock(int& fl, int& sl) const noexcept
        {
            std::uint32_t slMap = slBitmap_[fl] & (~std::uint32_t{0} << sl);

            if (slMap == 0)
            {
                const std::uint32_t flMap = flBitmap_ & (~std::uint32_t{0} << (fl + 1));
                if (flMap == 0) { return nullptr; }

                fl    = std::countr_zero(flMap);
                slMap = slBitmap_[fl];
            }

            sl = std::countr_zero(slMap);
            return blocks_[fl][sl];
        }

        void InsertBlock(Header* h) noexcept
        {
            int fl = 0, sl = 0;
            MappingInsert(detail::SizeOf(h), fl, sl);

            h->nextFree = blocks_[fl][sl];
            detail::PrevFreeOf(h) = nullptr;

            if (blocks_[fl][sl] != nullptr)
            {
                detail::PrevFreeOf(blocks_[fl][sl]) = h;
            }

            blocks_[fl][sl] = h;

            flBitmap_     |= (std::uint32_t{1} << fl);
            slBitmap_[fl] |= (std::uint32_t{1} << sl);
        }

        void RemoveBlock(Header* h) noexcept
        {
            int fl = 0, sl = 0;
            MappingInsert(detail::SizeOf(h), fl, sl);
            RemoveBlock(h, fl, sl);
        }

        void RemoveBlock(Header* h, int fl, int sl) noexcept
        {
            Header* prev = detail::PrevFreeOf(h);
            Header* next = h->nextFree;

            if (prev != nullptr) { prev->nextFree = next; }
            else                 { blocks_[fl][sl] = next; }

            if (next != nullptr) { detail::PrevFreeOf(next) = prev; }

            if (blocks_[fl][sl] == nullptr)
            {
                slBitmap_[fl] &= ~(std::uint32_t{1} << sl);

                if (slBitmap_[fl] == 0)
                {
                    flBitmap_ &= ~(std::uint32_t{1} << fl);
                }
            }
        }

        Header*       blocks_[kFlCount][kSlCount] = {};
        std::uint32_t flBitmap_ = 0;
        std::uint32_t slBitmap_[kFlCount] = {};
        // ... buffer_, base_, capacity_ などは第24章と同じ ...
    };
}
```

### 実装のポイント

**`RemoveBlock` の2つの版。** 合体のときはクラスを再計算する版を、確保のときは既に分かっている `fl`, `sl` を渡す版を使います。**確保の経路から計算を1つ減らせます。**

**ビットマップの更新順序。** `slBitmap_[fl]` が全部 0 になったときだけ、`flBitmap_` のビットを降ろします。**この二段の連動を間違えると、「空きがあるのに見つからない」という最悪のバグになります。**

**`assert(SizeOf(h) >= need)`。** 切り上げが正しく働いているかを検査します。**この不変条件が破れたら、TLSF の O(1) 保証そのものが崩れています。** 必ず入れておくべきアサートです。

**ブロック構造は第24章と共有。** 合体のロジックを書き直していません。第13章でヘッダを分割しておいた効果が、ここで出ています。

---

## 27.6 測る:最悪値が平らになる

### 平均性能

第23章から使っている同じ負荷をかけます。

```
                     Allocate   Free    平均探索
FreeList v0.18        16.8 ns   8.1 ns    1.3
Buddy    v0.19        11.4 ns  18.7 ns    2.1
TLSF     v0.20        10.2 ns   8.6 ns    1.0
new                   17.6 ns  15.2 ns     —
```

**確保が最速です。** しかも「平均探索 1.0」——**常に1回で見つかっています**。

### 本題:最悪値

第5章で作った個別測定とヒストグラムを、そのまま使います。100 万回の確保を1回ずつ測ります。

```
--- TLSF (サンプル数 1000000) ---
       < 100 ns |   999983 ##################################
100 ns –   1 us |       17 ############
  1 us –  10 us |        0
 10 us – 100 us |        0
100 us –   1 ms |        0
        > 1 ms  |        0
  最大値 : 300 ns

--- FreeList v0.18 (サンプル数 1000000) ---
       < 100 ns |   999412 ##################################
100 ns –   1 us |      562 ####################
  1 us –  10 us |       26 ############
 10 us – 100 us |        0
100 us –   1 ms |        0
        > 1 ms  |        0
  最大値 : 4200 ns

--- new (サンプル数 1000000) ---
       < 100 ns |   996213 ##################################
100 ns –   1 us |     3402 ########################
  1 us –  10 us |      338 ##################
 10 us – 100 us |       44 ############
100 us –   1 ms |        3 ######
        > 1 ms  |        0
  最大値 : 412300 ns
```

### この表が語ること

| | 最大値 | 中央値との比 |
|---|---|---|
| **TLSF** | **300 ns** | **約 30 倍** |
| FreeList v0.18 | 4,200 ns | 約 420 倍 |
| `new` | 412,300 ns | 約 23,000 倍 |

**TLSF に残った 300 ns は、OS のスケジューラによる中断です。** 第4章と第5章で見たノイズと同じ性質のもので、何度実行しても値が変わります。**アロケーターの中で起きたことではありません。**

FreeList の 4,200 ns は違います。大きいビンの中を長く走査したときに、再現性をもって発生します。**実装に起因するスパイクです。**

### 16.6 ms 予算で考える

第5章と同じ物差しを当てます。1フレームで1万回の確保を行うとして:

| | 合計 | 予算に対して |
|---|---|---|
| TLSF | 0.102 ms | 0.6% |
| `new` | 0.176 ms | 1.1% |

平均だけ見れば、大差ありません。

**問題はスパイクです。**

```
new  : 100 万回に3回、100 µs 超のスパイク
       → 1フレーム1万回なら、33フレームに1回
       → 16.6 ms 予算の 2.5% を単発で消費

TLSF : そのようなスパイクは観測されない
```

**これが「最悪時間を保証する」ことの意味です。**

平均性能では `new` に 1.7 倍勝っただけですが、**最悪値では 1,374 倍勝っています**。第2章で「平均ではなく最悪値で考える文化」と書いたことの、最終的な答えがこれです。

### 断片化

```
                   外部断片化   内部断片化   失敗
FreeList v0.18       0.521       0.9%        0
Buddy    v0.19       0.310      44.4%      412
TLSF     v0.20       0.498       0.9%        0
```

**FreeList とほぼ同じです。** 27.3 節で述べたとおり、切り上げは「探すクラス」を変えるだけで、消費量を変えません。

わずかに良い(0.521 → 0.498)のは、クラスが細かいぶん、より適切な大きさのブロックが選ばれているためです。**近似 best fit の精度が上がりました。**

---

## 27.7 何を得て、何を失ったか

```
                Allocate   Free    最大値    外部断片化  内部断片化  実装行数
Bump              1.8 ns     —     — (O(1))     0.000      0.06%      約 40
Pool              2.9 ns   2.0 ns  — (O(1))       —         0%        約 80
FreeList v0.18   16.8 ns   8.1 ns   4200 ns     0.521       0.9%      約 240
Buddy    v0.19   11.4 ns  18.7 ns    900 ns     0.310      44.4%      約 200
TLSF     v0.20   10.2 ns   8.6 ns    300 ns     0.498       0.9%      約 300
new              17.6 ns  15.2 ns 412300 ns       —          —          —
```

**TLSF は、汎用アロケーターとしてほぼ最良の選択肢です。**

- 確保も解放も速い
- 最悪時間が保証されている
- 断片化は FreeList 並み
- バディのような極端な内部断片化がない

**失ったものは、実装の単純さだけです。**

300 行。`Bump` の 7.5 倍、`Pool` の 4 倍です。二段の写像、二段のビットマップ、切り上げと切り下げの使い分け——**どれか1つ間違えると、静かに壊れます。**

> **だからこそ、第2部で作った道具が要ります。** ガードバイト、塗りつぶし、可視化、断片化指標。この章の実装を書くとき、それらを全部有効にしてください。私自身、`slBitmap_` と `flBitmap_` の連動を1か所間違えて、「空きがあるのに確保が失敗する」バグに数時間費やしました。

---

## 演習

**演習27-1** `kSlShift` を 4(16分割)や 6(64分割)に変えてください。断片化、速度、メモリ消費はどう変わりますか。

**演習27-2** `MappingSearch` の切り上げを外し、`MappingInsert` と同じにしてください。`assert(SizeOf(h) >= need)` はどのタイミングで失敗しますか。

**演習27-3** `std::countr_zero` と `_BitScanForward64` の生成コードを、逆アセンブルで比較してください。違いはありますか。

**演習27-4** `slBitmap_[fl]` が 0 になったときに `flBitmap_` を更新し忘れるバグを入れてください。どんな症状が出ますか。第2部のどの道具で見つけられますか。

**演習27-5** クラスごとのブロック数を表示してください。20,000 ステップ後、どのクラスが混んでいますか。第25章のビン分布と比べてください。

**演習27-6** `kFlCount` を 32 以上にすると、`~0u << (fl + 1)` が未定義動作になります。安全に扱うにはどう書き換えますか。

**演習27-7** TLSF の `Free` は、合体のたびに `MappingInsert` を計算します。これを減らす方法を考えてください。

**演習27-8** 第5章のヒストグラムを、Buddy と Pool についても実行してください。最大値はどうなりますか。

---

## 章末チェックリスト

- [ ] 第25章の実装が O(1) でない理由を説明できる
- [ ] 二段のサイズクラスが「常に 1/32 の相対精度」を与えることを説明できる
- [ ] **切り上げる写像と切り下げる写像** を使い分ける理由を説明できる
- [ ] 切り上げが内部断片化を増やさない理由を説明できる
- [ ] 二段ビットマップをビットスキャン2回で走査する仕組みを実装した 〔v0.20〕
- [ ] `std::countr_zero` を `_BitScanForward` より優先する理由を2つ挙げられる
- [ ] 確保時間のヒストグラムを取り、最大値が 300 ns 程度に収まることを確認した
- [ ] 平均で 1.7 倍、最悪値で 1,374 倍の差がつくことの意味を説明できる

---

## 次章の予告

第3部の最後は、**答え合わせ** です。

第20章から、5つのアロケーターを作ってきました。

```
Pool → FreeList(合体なし) → 合体 → ビン → Buddy → TLSF
```

第28章では、これらを **共通のベンチマーク** に流します。速度、断片化、メモリオーバーヘッドの3軸で表を作ります。

そして、最も重要な問いに答えます。

> **どれが最強か、ではない。どれをどこに使うか。**

第20章で立てた「制約を付け替える」という枠組みに、すべての実装を配置し直します。パーティクルにはプール、フレームにはバンプ、シーンにはアリーナ、そして「どうしても分類できないもの」に TLSF。

第3部で作ったものが、第7部(ゲームの形にする)でどう使われるかの見取り図を描いて、第3部を締めくくります。

---

> **コラム:リアルタイムシステムと O(1) 保証**
>
> TLSF は 2004 年に、Masmano、Ripoll、Crespo、Real らによって発表されました。目的は明確でした。**リアルタイムシステムで使える動的メモリ管理** です。
>
> ---
>
> **「リアルタイム」とは速いことではない**
>
> よくある誤解です。リアルタイムシステムに求められるのは、**速さではなく、時間の予測可能性** です。
>
> | | 求められること |
> |---|---|
> | ハードリアルタイム | 締め切りを1度でも破ると **システムの失敗** |
> | ソフトリアルタイム | 締め切りを破ると品質が下がる |
>
> エアバッグの展開、産業ロボットの制御、航空機のフライトコントロール——これらはハードリアルタイムです。「平均 1 マイクロ秒、まれに 10 ミリ秒」という部品は、**平均がどれだけ速くても使えません**。
>
> だから、これらの分野では長らく **動的メモリ確保が禁止** されてきました。`malloc` の最悪時間が保証されないからです。すべてのメモリを起動時に確保し、実行中は一切確保しない——それが安全な作法でした。
>
> **TLSF は、その禁を解こうとする試みでした。**
>
> ---
>
> **ゲームはソフトリアルタイム**
>
> ゲームは、締め切りを破っても人は死にません。フレームが落ちるだけです。
>
> しかし第5章で見たとおり、**プレイヤーは平均フレームレートより、フレーム間隔の不均一さに敏感** です。「平均 59 fps」でも、16.6 ms と 33.3 ms が交互に来れば、はっきりカクついて見えます。
>
> **求めているものは、リアルタイム制御と同じです。** 最悪値の保証。
>
> だから、リアルタイムシステムのために設計された TLSF が、ゲーム開発でも使われることになりました。
>
> ---
>
> **どこで使われているか**
>
> TLSF は、リファレンス実装がオープンソースで公開されており、組み込みシステム、RTOS、ゲームエンジン、仮想マシンなど、幅広い場所で採用されています。
>
> 近年では **GPU メモリの管理** にも使われています。Vulkan や Direct3D 12 のメモリ管理ライブラリで、サブアロケーションのアルゴリズムとして TLSF が選択できるようになっています。第26章で見たとおり、GPU メモリは長らくバディの独壇場でしたが、内部断片化の少なさから TLSF が使われる場面が増えています。
>
> ---
>
> **O(1) の意味を、正確に理解しておく**
>
> TLSF が O(1) だというとき、それは **確保と解放の手続きが、ブロック数に依存しない一定回数の操作で終わる** という意味です。
>
> **メモリが足りなくなることは防げません。** 断片化が進んで大きなブロックが取れなくなれば、確保は失敗します。O(1) で失敗するだけです。
>
> **キャッシュミスも防げません。** ビットマップとブロックヘッダへのアクセスは、キャッシュに乗っていなければメモリアクセスになります。27.6 節で観測した 300 ns のばらつきには、これも含まれています。
>
> **O(1) は「命令数の保証」であって、「実時間の保証」ではありません。** 本当のハードリアルタイムシステムでは、キャッシュの挙動やメモリバスの競合まで含めた最悪時間解析が必要になります。
>
> ---
>
> それでも、**アルゴリズムに起因するスパイクを消せる** ことの価値は大きい。27.6 節のヒストグラムが、それを示しています。
>
> `new` の 412 µs は、アルゴリズムとシステムコールに起因します。TLSF の 300 ns は、OS のスケジューラに起因します。**前者は設計で消せますが、後者は消せません。**
>
> **自分で消せるものを消しておく。** それが、この章でやったことです。
