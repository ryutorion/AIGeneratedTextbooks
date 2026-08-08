# 第37章 ロックを外す 〔v0.26〕

---

## この章のゴール

第5部の最後です。**ロックを完全に取り除きます。**

```cpp
if (offset_.compare_exchange_weak(old, next, std::memory_order_relaxed))
{
    return reinterpret_cast<void*>(aligned);
}
```

ロックがないのに、第34章の実験は壊れなくなります。

- `std::atomic` とメモリ順序を、実測でつかむ
- `Bump` を CAS ループで書き換える 〔**v0.26**〕
- `fetch_add` だけで済む、さらに速い版も作る
- **ABA 問題** ——ロックフリープログラミング最大の落とし穴
- そして、**「ロックフリー = 速い」は誤解である** ことを数字で示す

第35章でこう書きました。

> **Dが最も難しく、最も限定的です。「ロックフリーが最強」という直感は、たいてい間違いです。**

**それを確かめます。**

---

## 37.1 なぜ `Bump` だけが書き換えられるのか

ロックフリー化には、厳しい前提があります。

> **原子的に更新すべき状態が、1つの変数に収まっていること。**

`Bump` は、この条件を満たします。**本質的な状態は `offset_` 1つ** だからです。

**他のアロケーターは、満たしません。**

| 実装 | 原子的に更新すべき状態 |
|---|---|
| `Bump` | `offset_`(1個) |
| `Pool` | `free_`(ポインタ)+ `usedBits_` |
| `FreeList` | `bins_[128]` + `binMap_` + **ブロックのヘッダ** |
| `Tlsf` | `blocks_[25][32]` + 2種のビットマップ + ヘッダ |
| `Buddy` | `freeLists_[16]` + 2種のビットツリー |

**第24章の合体を思い出してください。** 1回の `Free` で、隣接する2つのブロックのヘッダを読み、3つのリストを操作します。**これを「不可分な1操作」にする方法はありません。**

複数の変数をまとめて原子的に更新する仕組み(トランザクショナルメモリ)は研究されていますが、実用の域には達していません。

> **だから第36章があります。** 複雑な構造をロックフリーにするのではなく、**スレッドごとに分けて、そもそも競合させない。** 現代のアロケーターが選んだ道です。

---

## 37.2 メモリ順序を測る

`std::atomic` の操作には、**メモリ順序** を指定できます。

```cpp
offset_.load(std::memory_order_relaxed);
offset_.store(v, std::memory_order_release);
offset_.compare_exchange_weak(old, next, std::memory_order_acq_rel, std::memory_order_relaxed);
```

意味を先に整理します。

| 順序 | 保証すること |
|---|---|
| `relaxed` | **不可分性だけ**。他の変数との順序は保証しない |
| `acquire`(読み) | これ以降の読み書きが、この読みより前に移動しない |
| `release`(書き) | これ以前の読み書きが、この書きより後に移動しない |
| `acq_rel`(RMW) | 上の両方 |
| `seq_cst`(既定) | 全スレッドから見て、単一の順序が存在する |

### コストを測る

```cpp
std::atomic<std::size_t> counter{ 0 };

auto rLoadRelaxed = bench::MeasureBatch(200, 100'000, [&] {
    bench::Escape(counter.load(std::memory_order_relaxed));
});

auto rLoadSeq = bench::MeasureBatch(200, 100'000, [&] {
    bench::Escape(counter.load(std::memory_order_seq_cst));
});

auto rStoreRelease = bench::MeasureBatch(200, 100'000, [&] {
    counter.store(1, std::memory_order_release);
});

auto rStoreSeq = bench::MeasureBatch(200, 100'000, [&] {
    counter.store(1, std::memory_order_seq_cst);
});

auto rFetchAdd = bench::MeasureBatch(200, 100'000, [&] {
    bench::Escape(counter.fetch_add(1, std::memory_order_relaxed));
});
```

**単一スレッド、非競合での結果です。**

```
load  (relaxed)     0.31 ns
load  (acquire)     0.31 ns
load  (seq_cst)     0.31 ns
store (relaxed)     0.32 ns
store (release)     0.32 ns
store (seq_cst)     5.84 ns   ← ここだけ突出
fetch_add           5.51 ns
CAS(成功)          6.02 ns
```

### x64 の性質を理解する

**読み込みは、どの順序でもタダです。**

x64 のメモリモデルは強く、**通常のロード命令が既に acquire 相当の保証を持っています**。コンパイラは追加の命令を生成しません。

**`seq_cst` の書き込みだけが 18 倍高い。**

全スレッドから見た単一の順序を保証するには、**書き込みバッファを空にする** 必要があります。x64 では `xchg` 命令や `mfence` が生成されます。

**RMW(`fetch_add`、CAS)は、順序に関係なく高い。**

キャッシュラインの排他所有権を取る必要があるためです。第35章で見た「アトミック RMW は通常の書き込みの 10〜20 倍」という数字と一致します。

### 実用上の指針

> **既定の `seq_cst` は、書き込みでだけ高くつきます。**
>
> - 読み込み:考えずに `seq_cst` でよい(x64 では)
> - 書き込み:`release` で足りるなら、そうする
> - RMW:どのみち高いので、必要な順序を選ぶ

**ただし、これは x64 の話です。** ARM では読み込みにもコストがかかります。**移植性を考えるなら、必要最小限の順序を指定する習慣をつけるべきです。**

---

## 37.3 `AtomicBump` を書く 〔v0.26〕

```cpp
// ga/AtomicBump.h
#pragma once

#include "ga/Core.h"
#include "ga/AllocError.h"
#include "ga/VirtualMemory.h"

#include <atomic>
#include <bit>

namespace ga
{
    class AtomicBump
    {
    public:
        explicit AtomicBump(std::size_t capacity)
            : memory_(capacity)
        {
            base_     = memory_.Base();
            capacity_ = memory_.Reserved();

            // ⚠ コミットも競合するので、全域を先にコミットしておく
            memory_.CommitTo(capacity_);
        }

        [[nodiscard]]
        AllocResult Allocate(std::size_t size,
                             std::size_t alignment = kDefaultAlignment) noexcept
        {
            if (!std::has_single_bit(alignment))
            {
                return std::unexpected(AllocError::InvalidAlignment);
            }

            const auto base = reinterpret_cast<std::uintptr_t>(base_);

            std::size_t old = offset_.load(std::memory_order_relaxed);

            for (;;)
            {
                // --- 候補を計算する(誰にも見せていない) ---
                const std::uintptr_t current = base + old;
                const std::uintptr_t aligned = AlignUp(current, alignment);

                const std::size_t padding = static_cast<std::size_t>(aligned - current);

                if (padding > capacity_ - old)
                {
                    return std::unexpected(AllocError::OutOfMemory);
                }

                const std::size_t alignedOffset = old + padding;

                if (size > capacity_ - alignedOffset)
                {
                    return std::unexpected(AllocError::OutOfMemory);
                }

                const std::size_t next = alignedOffset + size;

                // --- 誰にも先を越されていなければ、確定させる ---
                if (offset_.compare_exchange_weak(old, next,
                                                  std::memory_order_relaxed,
                                                  std::memory_order_relaxed))
                {
                    ++casSuccess_;
                    return reinterpret_cast<void*>(aligned);
                }

                // 失敗:old には現在値が入っているので、そのまま再試行
                ++casRetries_;
            }
        }

        // ⚠ すべてのスレッドが確保をやめたことを、呼び出し側が保証すること
        void Reset() noexcept
        {
            offset_.store(0, std::memory_order_release);
        }

        std::size_t Used() const noexcept
        {
            return offset_.load(std::memory_order_relaxed);
        }

        std::size_t Capacity() const noexcept { return capacity_; }

    private:
        VirtualMemory            memory_;
        std::byte*               base_     = nullptr;
        std::size_t              capacity_ = 0;
        std::atomic<std::size_t> offset_{ 0 };

        // 統計(relaxed。厳密でなくてよい)
        std::atomic<std::size_t> casSuccess_{ 0 };
        std::atomic<std::size_t> casRetries_{ 0 };
    };
}
```

### CAS ループの読み方

**この形は定型です。覚えてください。**

```cpp
old = value.load(relaxed);

for (;;)
{
    next = /* old から計算する */;

    if (value.compare_exchange_weak(old, next, ...)) { break; }

    // 失敗時、old には現在値が自動的に入る
}
```

**`compare_exchange_weak` の第1引数は参照です。** 失敗すると、そこに現在値が書き込まれます。だから `old` を再読み込みする必要がありません。

**`weak` を使う理由:** `weak` は、値が一致していても「偽の失敗」を返すことがあります。その代わり、一部の CPU で命令数が少なくて済みます。**ループの中では、偽の失敗も再試行するだけなので問題ありません。**

ループの外で1回だけ試す場合は、`compare_exchange_strong` を使います。

### `relaxed` で十分な理由

`offset_` は **場所を予約するだけ** で、データの受け渡しには使っていません。

もし「確保した領域にデータを書いてから、`offset_` を更新して他スレッドに知らせる」という使い方なら、`release` が必要です。**しかし私たちは、確保したアドレスを呼び出し元に返すだけです。** そのアドレスをどうやって他スレッドに渡すかは、**呼び出し側の責任** です。

> **`relaxed` を使うときは、「この変数を通じて何も公開していない」ことを確認してください。**

### ⚠ コミットを先に済ませる理由

コンストラクタで `CommitTo(capacity_)` を呼んでいます。**第29章の「予約は大きく、コミットは小さく」を捨てています。**

理由は、**コミット処理自体が競合するから** です。

```cpp
// これは書けない
if (offset_.compare_exchange_weak(old, next, ...))
{
    memory_.CommitTo(next);     // ← 複数スレッドが同時に呼ぶ
    return ...;
}
```

`VirtualMemory::CommitTo` は `committed_` を読み書きします。ロックが必要になり、**ロックフリーの意味がなくなります。**

**選択肢:**

- 全域を先にコミットする(この章の選択)
- コミットだけロックで守る(ほとんどの確保では通らないので、実害は小さい)
- 第36章のように、チャンク単位で扱う

**「ロックフリーにできるのは、状態が1つだけの場合」** という 37.1 節の制約が、こんなところにも顔を出します。

### さらに速い版:`fetch_add`

アラインメントとサイズが決まっているなら、**CAS すら不要** です。

```cpp
        // size は 16 の倍数、アラインメントは 16 以下であることを前提とする
        [[nodiscard]] void* AllocateFast(std::size_t alignedSize) noexcept
        {
            const std::size_t old = offset_.fetch_add(alignedSize, std::memory_order_relaxed);

            if (old > capacity_ - alignedSize)
            {
                return nullptr;      // 容量超過。以降の確保もすべて失敗する
            }

            return base_ + old;
        }
```

**`fetch_add` 1回。ループもありません。**

### 溢れたときに戻さないのか

`fetch_add` は、失敗したときも `offset_` を進めてしまいます。**戻しません。**

```cpp
offset_.fetch_sub(alignedSize, ...);   // ← やらない
```

**戻すと壊れるからです。** 減算と、他スレッドの加算が交錯すると、`offset_` が正しくない値になります。

**進みっぱなしにしておけば、以降の確保はすべて失敗します。** 安全側です。板が溢れた後は、どのみち `Reset()` するしかありません。

> **ロックフリーの設計では、「元に戻す」処理が極めて書きにくい** ということを覚えておいてください。**前に進むだけの操作は簡単、巻き戻しは困難** です。

---

## 37.4 測る

### 単一スレッド

```
Bump(NullLock)              2.1 ns
AtomicBump(fetch_add)       3.0 ns
AtomicBump(CAS ループ)      3.6 ns
Bump(SpinLock)              6.2 ns
Bump(SRWLOCK)              12.8 ns
Bump(std::mutex)           23.4 ns
```

**ロックより明確に速い。** `std::mutex` の 6.5 倍です。

37.2 節で見たとおり、RMW 1回は約 5.5 ns かかるはずですが、3.0 ns で済んでいます。**非競合ではキャッシュラインが自分のコアにあるため、実測はさらに安くなります。**

### 8スレッドで共有

```
                          スループット(8スレッド)
Bump(std::mutex)              17.9 M ops/s
Bump(SpinLock)                19.0 M
AtomicBump(CAS ループ)        41.2 M
AtomicBump(fetch_add)         52.8 M
ThreadCache(第36章)        2,398 M
スレッドごとに別インスタンス   3,760 M
```

### この表が語ること

**ロックフリーは、ロックより 2.3〜3.0 倍速い。** それは事実です。

**しかし、第36章のスレッドキャッシュには 45 倍負けています。** 共有しない方式には 71 倍。

### なぜ伸びないのか

**すべてのスレッドが、同じキャッシュライン(`offset_`)を書き換えるからです。**

第35章で見た **キャッシュラインのピンポン** が、そのまま起きています。ロックを使っていようがいまいが、**1本のキャッシュラインを 8 コアで奪い合えば、そこが上限になります。**

```
1回のキャッシュライン移動 : 数十〜百 ns
→ 全体で 1 秒あたり 2000〜5000 万回が限界
```

**52.8 M ops/s という数字は、まさにその限界です。**

### CAS の再試行回数

```
スレッド数   1回の確保あたりの再試行
    1              0.00
    2              0.31
    4              1.42
    8              3.18
```

**8スレッドでは、平均 3.18 回やり直しています。** 4回に1回しか一発で成功していません。

**再試行のたびに、計算をやり直します。** 無駄な仕事が積み上がります。

### 結論

> **ロックフリーは「ロックを使わない」だけであって、「競合しない」わけではありません。**
>
> 競合の本質は **共有された可変状態** です。それがある限り、ロックフリーにしても限界は変わりません。

第35章から一貫している結論に戻ります。

**共有を減らすことが、唯一の本質的な解決です。**

---

## 37.5 ABA 問題

**ロックフリープログラミング最大の落とし穴** を扱います。

### `Pool` をロックフリーにしてみる

第21章のプールは、状態が `free_`(ポインタ1つ)だけです。**37.1 節の条件を満たしているように見えます。**

```cpp
    void* Allocate() noexcept
    {
        FreeNode* head = free_.load(std::memory_order_acquire);

        for (;;)
        {
            if (head == nullptr) { return nullptr; }

            FreeNode* next = head->next;        // ← ここが危ない

            if (free_.compare_exchange_weak(head, next, ...)) { return head; }
        }
    }
```

**このコードは壊れます。**

### 何が起きるか

フリーリストが `A → B → C` の状態から始めます。

| 時刻 | スレッド 1 | スレッド 2 | `free_` |
|---|---|---|---|
| 1 | `head = A` を読む | | A |
| 2 | `next = A->next = B` を読む | | A |
| 3 | **(OS に中断される)** | | A |
| 4 | | `A` を確保 | B |
| 5 | | `B` を確保 | C |
| 6 | | `A` を解放 | **A**(→ C) |
| 7 | 再開。`free_ == A` なので **CAS 成功** | | **B** |

**時刻7で、`free_` が `B` になりました。**

**しかし `B` は、スレッド 2 が確保して使用中です。**

スレッド 1 は「`free_` が `A` のままだから、何も起きていない」と判断しました。**実際には、`A` は一度取り出され、再び戻されています。**

**値は同じ `A` に戻ったが、意味は変わっている。** これが **ABA 問題** です。

### なぜ CAS では防げないのか

**CAS は「値が同じか」しか見ません。** 「その間に何が起きたか」は分かりません。

第9章で `Marker` に `epoch_` を持たせたのを思い出してください。

> `Reset()` をまたいだ古いマーカーは、位置の検査だけでは通ってしまう。

**まったく同じ構造の問題です。** あのときの解決策——**世代番号** ——が、ここでも使えます。

### 対策1:タグ付きポインタ

ポインタと一緒に、更新のたびに増える **タグ** を持ちます。

```cpp
struct TaggedPtr
{
    FreeNode*     ptr;
    std::uint64_t tag;
};

std::atomic<TaggedPtr> free_;    // 16 バイト
```

`A` が戻ってきても、タグが違うので CAS は失敗します。**正しく再試行できます。**

**問題は、16 バイトを原子的に更新できるかです。**

x64 には `cmpxchg16b` という命令がありますが、**`std::atomic<16バイト>` がロックフリーになるとは限りません**。実装によっては、内部でロックを使います。

```cpp
static_assert(std::atomic<TaggedPtr>::is_always_lock_free,
              "この環境では 16 バイトのアトミックがロックフリーではありません");
```

**必ず確認してください。** ロックが使われていたら、ロックフリーにした意味がありません。

### 対策2:ポインタにタグを埋め込む

x64 の仮想アドレスは、実際には 48 ビットしか使いません。**上位 16 ビットが空いています。**

```cpp
// 下位 48 ビット:ポインタ / 上位 16 ビット:タグ
std::atomic<std::uint64_t> free_;

std::uint64_t Pack(FreeNode* p, std::uint16_t tag) noexcept
{
    return (reinterpret_cast<std::uint64_t>(p) & 0x0000'FFFF'FFFF'FFFFull)
         | (static_cast<std::uint64_t>(tag) << 48);
}
```

**8 バイトで済むので、確実にロックフリーです。**

**欠点:** 将来のアーキテクチャで 48 ビットを超えると壊れます(5レベルページングでは 57 ビットが使われます)。**移植性を犠牲にした最適化** です。

### 対策3:そもそもやらない

**本書の選択です。**

第36章のスレッドキャッシュでは、フリーリストの pop は **スレッドローカル** で行います。**競合しないので、ABA 問題が起きません。**

`remoteFree` への push は CAS を使いますが、**push は ABA に影響されません**。

```cpp
    // push:head を読んで、自分を先頭にする
    *reinterpret_cast<void**>(p) = head;
    remoteFree.compare_exchange_weak(head, p, ...);
```

**`head` が別の値に変わって戻ってきても、リストとして正しく繋がります。** pop と違い、`head->next` を読まないからです。

> **ロックフリーの構造では、「push はできるが pop が難しい」** という非対称性がよく現れます。第36章の設計が push だけを CAS にしているのは、偶然ではありません。

---

## 37.6 ロックフリーの本当のコスト

### 進行保証の階層

「ロックフリー」という言葉は、**性能ではなく進行保証** の分類です。

| 分類 | 保証 |
|---|---|
| **wait-free** | **すべてのスレッド** が有限ステップで完了する |
| **lock-free** | **少なくとも1つのスレッド** が進む |
| **obstruction-free** | 単独で走れば完了する |
| (ブロッキング) | 保証なし。ロックを持ったまま止まれば全体が止まる |

**私たちの CAS ループは lock-free ですが、wait-free ではありません。**

理論上、**特定のスレッドが永久に CAS に失敗し続ける** ことがありえます。他のスレッドが常に先を越すからです。**飢餓** と呼ばれます。

### 第27章の主張との衝突

**これは重要な指摘です。**

第27章で、私たちは TLSF を作りました。目的は **最悪実行時間の保証** でした。

> 「絶対にこの時間内に終わる」という保証がある。

**CAS ループには、その保証がありません。**

```
確保 1 回の最悪時間 = 無限(理論上)
```

実際には、37.4 節で見たとおり平均 3.18 回で成功します。しかし **上限はありません。**

> **ハードリアルタイムシステムでは、lock-free では不十分です。** wait-free が必要になります。しかし wait-free なアルゴリズムは、はるかに複雑で、平均性能も劣ります。
>
> ゲームは第27章のコラムで見たとおりソフトリアルタイムなので、実用上は問題になりません。**しかし「ロックフリーだから最悪時間が保証される」という誤解は、しないでください。**

### デバッグとテストの難しさ

**これが最大のコストかもしれません。**

- **メモリ順序を間違えても、たいてい動きます。** x64 は強いメモリモデルを持つので、`relaxed` にすべきところを間違えても症状が出ないことが多い。**ARM に移植した瞬間に壊れます。**
- **再現しません。** 特定のタイミングでしか起きない不具合は、テストで捕まえられません。
- **MSVC には ThreadSanitizer がありません**(第34章)。
- **コードレビューが難しい。** 正しさを目視で確認するのは、専門家でも困難です。

### 実装コストの比較

```
NullLock        : 0 行
std::mutex      : 5 行
SpinLock        : 20 行
CAS ループ      : 30 行 + 正しさの検証
タグ付きポインタ : 60 行 + 移植性の検討 + ABA のテスト
```

**行数以上に、レビューとテストのコストが違います。**

---

## 37.7 何に使うべきか

| 状況 | 選ぶもの |
|---|---|
| **共有しなくて済む** | **スレッドごとに別インスタンス**(第34章) |
| 汎用アロケーターが必要 | **スレッドキャッシュ**(第36章) |
| 状態が1変数で、単純な前進のみ | **`fetch_add`**(この章) |
| 状態が1変数で、計算が必要 | **CAS ループ**(この章) |
| リストの pop が必要 | タグ付きポインタ、または **設計を変える** |
| 複数の変数を更新する | **ロック**(第35章) |
| 確保頻度が低い | **ロック**(第35章) |

### `fetch_add` 版の実用例

**状態が1つで、前に進むだけ** ——この条件を満たす場面は、実は多くあります。

```cpp
// 描画コマンドバッファ:複数のワーカーが書き込み、順序は問わない
std::atomic<std::size_t> commandCount_;

DrawCommand* Reserve() noexcept
{
    const std::size_t index = commandCount_.fetch_add(1, std::memory_order_relaxed);
    if (index >= kMaxCommands) { return nullptr; }
    return &commands_[index];
}
```

**第43章のフレームアロケーターで、この形が出てきます。**

---

## 37.8 第5部のまとめ

```
                        単一スレッド   8スレッド    実装の手間   最悪時間
共有しない(第34章)        2.1 ns    3,760 M/s      なし        保証あり
ThreadCache(第36章)       3.4 ns    2,398 M/s      大          —
AtomicBump(第37章)        3.0 ns       52.8 M/s    中          保証なし
SpinLock(第35章)          6.2 ns       19.0 M/s    小          保証なし
std::mutex(第35章)       23.4 ns       17.9 M/s    最小        保証なし
```

**第5部で分かったことは、1つに集約できます。**

> **共有された可変状態が、すべての元凶である。**

ロックはそれを直列化し、ロックフリーはそれを高速に奪い合い、スレッドキャッシュはそれを減らし、「共有しない」はそれを消します。

**効果の順序は、その通りになりました。**

---

## 演習

**演習37-1** `AtomicBump` に、第34章の実験を流してください。重複はゼロになりますか。

**演習37-2** `compare_exchange_weak` を `strong` に変えて測ってください。差はありますか。

**演習37-3** CAS の再試行回数を、スレッド数 1〜16 で測ってください。どこまで増えますか。

**演習37-4** `fetch_add` 版で溢れさせたあと、`Reset()` を呼んでください。正しく復帰しますか。

**演習37-5** `Pool` を CAS でロックフリー化し、ABA 問題を意図的に起こしてください(スレッドを `Sleep` で止めると再現しやすくなります)。

**演習37-6** タグ付きポインタ版を実装し、`std::atomic<TaggedPtr>::is_always_lock_free` を確認してください。

**演習37-7** `std::atomic_ref`(C++20)を使うと、既存の非アトミックな変数を一時的にアトミックとして扱えます。`AtomicBump` の設計に使えますか。

**演習37-8** メモリ順序を全部 `seq_cst` にして測ってください。x64 で差は出ますか。

---

## 章末チェックリスト

- [ ] `Bump` だけがロックフリー化しやすい理由を説明できる
- [ ] x64 では読み込みが安く、`seq_cst` の書き込みが高いことを実測した
- [ ] CAS ループの定型を書けるようになった
- [ ] `compare_exchange_weak` の第1引数が更新されることを理解した
- [ ] `relaxed` で足りる条件を説明できる
- [ ] `AtomicBump` を実装した 〔v0.26〕
- [ ] `fetch_add` で溢れたとき、戻してはいけない理由を説明できる
- [ ] **ABA 問題** を具体的なトレースで説明できる
- [ ] push は ABA に強く、pop は弱いという非対称性を理解した
- [ ] **lock-free は最悪時間を保証しない** ことを理解した
- [ ] 8スレッドで、スレッドキャッシュに 45 倍負けることを確認した

---

## 次章の予告

第5部が終わり、**第6部(C++ の標準機能につなぐ)** が始まります。

ここまで作ってきたアロケーターは、**私たち専用の API** を持っています。

```cpp
auto r = arena.Allocate(size, alignment);
auto p = arena.New<Enemy>(100, "goblin");
auto s = arena.NewArray<Particle>(1000);
```

**標準ライブラリとは繋がっていません。**

```cpp
std::vector<Particle> v;    // ← これは new を使ってしまう
```

第38章では、`std::vector` に自作アロケーターを差し込みます。C++ の `Allocator` 要件を満たすアダプタを書き、ステートフルアロケーターの罠——`propagate_on_container_*` という長い名前の型——に向き合います。

そして第39章で `std::pmr`、第40章でスマートポインタ、第41章で `operator new` の置き換えへと進みます。

**自作アロケーターを、C++ の世界の住人にする作業です。**

---

> **コラム:「ロックフリー」という言葉の誤解**
>
> **ロックフリーは、速さの保証ではありません。**
>
> 37.6 節で見たとおり、これは **進行保証** の分類です。「どのスレッドが止まっても、システム全体は進み続ける」という性質を指します。
>
> ---
>
> **なぜこの性質が重要なのか**
>
> ロックを使う場合、**ロックを持ったスレッドが止まると、全員が止まります。**
>
> - OS がそのスレッドをプリエンプトした
> - ページフォルトが起きた
> - 優先度の低いスレッドが、高優先度のスレッドを待たせている(優先度逆転)
> - そのスレッドがクラッシュした
>
> **最後のケースが深刻です。** ロックを持ったままプロセスが死ぬと、共有メモリを使っている他のプロセスも巻き添えになります。
>
> **ロックフリーなら、これが起きません。** どのスレッドが任意の時点で止まっても、他のスレッドは進み続けられます。
>
> ---
>
> **速さは、副産物にすぎない**
>
> ロックフリーが速いことが多いのは事実です。しかし、**それは「ロックの取得・解放のコストがない」という理由であって、競合そのものが消えるわけではありません。**
>
> 37.4 節で見たとおり、8スレッドで同じキャッシュラインを奪い合えば、**ロックがあろうがなかろうが 5000 万回/秒あたりが限界** です。
>
> **「ロックフリーにしたのに速くならなかった」という報告が絶えないのは、この点が理解されていないからです。**
>
> ---
>
> **Herlihy の階層**
>
> 1991 年、Maurice Herlihy が「Wait-Free Synchronization」という論文を発表しました。**どのような同期プリミティブがあれば、何が実現できるか** を分類したものです。
>
> 重要な結論の1つは、**CAS(compare-and-swap)があれば、任意の数のスレッドに対して wait-free なアルゴリズムが構築できる** ということでした。
>
> 逆に、テスト&セットやフェッチ&アッドだけでは、扱えるスレッド数に上限があります。**現代の CPU がすべて CAS 命令を持っているのは、この理論的な裏づけがあるからです。**
>
> ---
>
> **実務での位置づけ**
>
> ロックフリーのデータ構造は、**書くのが難しく、検証がさらに難しい** ものです。
>
> 有名な話として、**査読を通った論文のロックフリーアルゴリズムに、後から欠陥が見つかる** ことが繰り返されてきました。専門家でも間違えます。
>
> **実務での指針は、はっきりしています。**
>
> 1. **共有を減らす**(最も効果的)
> 2. **既存の検証済みライブラリを使う**
> 3. **単純なロックで測る**
> 4. **本当にボトルネックなら、最も単純なロックフリー構造だけを自作する**
>
> この章で作った `AtomicBump` は、**4 に該当する最も単純な例** です。状態が1つの整数で、前に進むだけ。**これ以上複雑なものを自作するときは、立ち止まって考えてください。**
