# 第24章 穴をくっつける 〔v0.17〕

---

## この章のゴール

第23章のアロケーターは、20,000 ステップで壊滅しました。

```
空きブロック : 2874 個
外部断片化   : 0.986
確保の失敗   : 1108 回
確保のコスト : 2940 ns(new の 167 倍)
```

**この章で、たった1つの処理を足します。**

> 解放するとき、隣が空きなら、くっつける。

第20章の紙のシミュレーションで、これが失敗を成功に変えるのを見ました。実装すると、上の数字がどうなるか——それがこの章です。

- 「前の隣」を O(1) で見つける方法(**境界タグ**)
- フラグをもう1つ足して、フッタを省く工夫
- フリーリストからの O(1) 取り外し
- 合体を実装する 〔**v0.17**〕
- 第23章と同じ実験を走らせ、数字と絵を比べる
- 即時合体と遅延合体という選択肢

---

## 24.1 「前の隣」が分からない

合体には、隣接するブロックが必要です。

**後ろの隣は簡単です。**

```cpp
Header* next = reinterpret_cast<Header*>(
    reinterpret_cast<std::byte*>(h) + SizeOf(h));
```

自分のサイズだけ進めば、次のブロックの先頭です。第23章の `ForEachBlock` で使った理屈と同じです。

**前の隣が難しい。**

```
┌────────────┬──────────┬─────────────┐
│    ???     │    h     │    next     │
└────────────┴──────────┴─────────────┘
      ▲            ▲
   どこから始まる?   ここは分かる
```

ブロックは可変サイズです。「16 バイト戻れば前のブロック」とはいきません。前のブロックが 32 バイトかもしれないし、64 KB かもしれない。

先頭から `ForEachBlock` で歩けば見つかりますが、**それでは O(n) です**。解放が O(1) でなくなります。

---

## 24.2 境界タグ

Knuth が示した答えは、拍子抜けするほど単純です。

> **ブロックの末尾にも、サイズを書いておく。**

```
┌──────────────┬──────────────────────┬──────────┐
│  ヘッダ       │      内容             │  フッタ   │
│ (size, ...)  │                      │ (size)   │
└──────────────┴──────────────────────┴──────────┘
```

こうすると、`h` の 8 バイト手前を読むだけで、**前のブロックのサイズが分かります**。

```cpp
const std::size_t prevSize = *reinterpret_cast<std::size_t*>(
    reinterpret_cast<std::byte*>(h) - sizeof(std::size_t));

Header* prev = reinterpret_cast<Header*>(
    reinterpret_cast<std::byte*>(h) - prevSize);
```

**O(1)。**

これが **境界タグ**(boundary tag)です。ブロックの両端にサイズという「タグ」を置くことで、どちらの方向からもブロックの境界が分かるようになります。

### 素朴にやるとオーバーヘッドが増える

フッタは 8 バイトです。第23章のヘッダ 16 バイトと合わせると、**確保のたびに 24 バイト**。

```
32 バイトの確保:
  ヘッダ 16 + 内容 32 + フッタ 8 = 56 バイト  → オーバーヘッド 75%
```

第23章の 50% からさらに悪化しました。**削れないでしょうか。**

---

## 24.3 フッタは、空きブロックにだけあればいい

ここで、第23章のコラムで予告した観察が効きます。

> **フッタを読むのは、前のブロックと合体したいときだけ。**
> **合体したいのは、前のブロックが空きのときだけ。**

つまり、**使用中のブロックにフッタは要りません**。

```
空きブロック:
┌──────────────┬──────────────────────┬──────────┐
│  ヘッダ       │      (空き)           │  フッタ   │
└──────────────┴──────────────────────┴──────────┘

使用中のブロック:
┌──────────────┬─────────────────────────────────┐
│  ヘッダ       │      ユーザーの領域(末尾まで)     │
└──────────────┴─────────────────────────────────┘
```

**フッタの領域を、ユーザーデータとして使い回します。**

### ただし、問題が1つ

`h` の手前 8 バイトを読んでも、それが **フッタなのかユーザーデータなのか分かりません**。

前のブロックが使用中なら、そこにはユーザーが書いた値が入っています。それをサイズだと思って解釈すれば、でたらめなアドレスに飛びます。

**だから、前のブロックが空きかどうかを、先に知る必要があります。**

### フラグをもう1つ

第23章で、サイズの下位4ビットが空いていることを利用しました。**そこにもう1つフラグを足します。**

```cpp
    inline constexpr std::size_t kFlagFree     = 1;   // このブロックは空き
    inline constexpr std::size_t kFlagPrevFree = 2;   // 前のブロックが空き
    inline constexpr std::size_t kSizeMask     = ~std::size_t{ 15 };
```

`kFlagPrevFree` が立っていれば、手前 8 バイトはフッタです。立っていなければ、読んではいけません。

**この設計は dlmalloc とその系譜が採用しているものです。** 実物のアロケーターと同じ構造を、私たちも持つことになります。

### フラグの維持

フラグは、状態が変わるたびに更新しなければなりません。

| 操作 | やること |
|---|---|
| ブロックが空きになった | 自分に `FREE` を立て、**フッタを書く**、**次のブロックに `PREV_FREE` を立てる** |
| ブロックが使用中になった | 自分の `FREE` を降ろし、**次のブロックの `PREV_FREE` を降ろす** |

**「次のブロックに知らせる」のを忘れるのが、典型的なバグです。** 忘れると、使用中のブロックのユーザーデータをフッタとして読み、任意のアドレスへ飛びます。

---

## 24.4 フリーリストからの O(1) 取り外し

もう1つ、実装上の課題があります。

合体するとき、**隣のブロックはすでにフリーリストに入っています**。合体して1つになるので、片方をリストから外さなければなりません。

第23章のフリーリストは **単方向** でした。途中のノードを外すには、前のノードを探して O(n) です。

**双方向にします。**

### どこにポインタを置くか

ヘッダに `prevFree` を足すと、ヘッダが 24 バイト(実質 32 バイト)になります。避けたい。

**第21章の発想を思い出してください。**

> 空いているブロックは、誰も使っていません。ならば、そこに何を書いても構いません。

`prevFree` を、**空きブロックのユーザー領域の先頭** に置きます。

```
空きブロック:
┌────────┬────────┬────────┬─────────────┬────────┐
│  size  │nextFree│prevFree│   (未使用)    │ フッタ  │
└────────┴────────┴────────┴─────────────┴────────┘
 ← ヘッダ(16 バイト) →  ← ユーザー領域 →
```

使用中のブロックでは、`prevFree` の位置もユーザーデータです。**空きのときだけ意味を持ちます。**

### 最小ブロックサイズ

空きブロックには、少なくとも次が必要です。

```
ヘッダ(16) + prevFree(8) + フッタ(8) = 32 バイト
```

第23章と同じ 32 バイトで足ります。**アラインメントの都合で確保していた余白が、ちょうど収まりました。**

```cpp
    inline constexpr std::size_t kMinBlockSize = 32;
```

---

## 24.5 実装する 〔v0.17〕

補助関数から書きます。ここを整理しておくと、本体が読みやすくなります。

```cpp
namespace ga::detail
{
    struct FreeListHeader
    {
        std::size_t     size;       // 下位 4 ビットはフラグ
        FreeListHeader* nextFree;   // 空きのときのみ
    };

    inline constexpr std::size_t kFlagFree     = 1;
    inline constexpr std::size_t kFlagPrevFree = 2;
    inline constexpr std::size_t kSizeMask     = ~std::size_t{ 15 };

    using Header = FreeListHeader;

    constexpr std::size_t SizeOf(const Header* h)     noexcept { return h->size & kSizeMask; }
    constexpr bool        IsFree(const Header* h)     noexcept { return (h->size & kFlagFree) != 0; }
    constexpr bool        PrevIsFree(const Header* h) noexcept { return (h->size & kFlagPrevFree) != 0; }

    inline std::byte* BytesOf(Header* h) noexcept
    {
        return reinterpret_cast<std::byte*>(h);
    }

    // 空きブロックのユーザー領域の先頭に置く prevFree
    inline Header*& PrevFreeOf(Header* h) noexcept
    {
        return *reinterpret_cast<Header**>(BytesOf(h) + sizeof(Header));
    }

    // 空きブロックの末尾 8 バイトに書くフッタ
    inline std::size_t& FooterOf(Header* h) noexcept
    {
        return *reinterpret_cast<std::size_t*>(
            BytesOf(h) + SizeOf(h) - sizeof(std::size_t));
    }

    // h の手前にあるフッタ(前が空きのときだけ有効)
    inline std::size_t PrevFooterOf(Header* h) noexcept
    {
        return *reinterpret_cast<std::size_t*>(BytesOf(h) - sizeof(std::size_t));
    }
}
```

### 本体

```cpp
class FreeList
{
public:
    // ... 第23章と同じ構築処理 ...

    [[nodiscard]] void* Allocate(std::size_t size) noexcept
    {
        if (size == 0) { return nullptr; }

        const std::size_t payload = AlignUpSize(size, kDefaultAlignment);
        if (payload < size) { return nullptr; }

        std::size_t need = payload + kHeaderSize;
        if (need < payload)        { return nullptr; }
        if (need < kMinBlockSize)  { need = kMinBlockSize; }

        for (Header* h = freeHead_; h != nullptr; h = h->nextFree)
        {
            ++searchSteps_;

            const std::size_t blockSize = detail::SizeOf(h);
            if (blockSize < need) { continue; }

            UnlinkFree(h);

            if (blockSize - need >= kMinBlockSize)
            {
                // --- 分割:後半を空きブロックにする ---
                MarkUsed(h, need);

                Header* rest = NextOf(h);
                MarkFree(rest, blockSize - need);
                PushFree(rest);
            }
            else
            {
                MarkUsed(h, blockSize);
                internalWaste_ += blockSize - need;
            }

            ++liveCount_;
            ++searches_;
            return BytesOf(h) + kHeaderSize;
        }

        ++searches_;
        ++failures_;
        return nullptr;
    }

    void Free(void* p) noexcept
    {
        if (p == nullptr) { return; }
        if (!Owns(p))     { ReportError(); return; }

        Header* h = HeaderOf(p);
        if (detail::IsFree(h)) { ReportError(); return; }

        --liveCount_;

        std::size_t size = detail::SizeOf(h);

        // --- ① 後ろと合体 ---
        Header* next = NextOf(h);
        if (next != nullptr && detail::IsFree(next))
        {
            UnlinkFree(next);
            size += detail::SizeOf(next);
            ++coalesceCount_;
        }

        // --- ② 前と合体 ---
        if (detail::PrevIsFree(h))
        {
            const std::size_t prevSize = detail::PrevFooterOf(h);
            Header* prev = reinterpret_cast<Header*>(BytesOf(h) - prevSize);

            UnlinkFree(prev);
            size += prevSize;
            h = prev;
            ++coalesceCount_;
        }

        // --- ③ 空きとして登録 ---
        MarkFree(h, size);
        PushFree(h);
    }

private:
    // ---- ブロックの状態を書き換える ----

    void MarkUsed(Header* h, std::size_t size) noexcept
    {
        const bool prevFree = detail::PrevIsFree(h);
        h->size = size | (prevFree ? detail::kFlagPrevFree : 0);

        // 次のブロックに「前は使用中だ」と伝える
        if (Header* n = NextOf(h); n != nullptr)
        {
            n->size &= ~detail::kFlagPrevFree;
        }
    }

    void MarkFree(Header* h, std::size_t size) noexcept
    {
        const bool prevFree = detail::PrevIsFree(h);
        h->size = size | detail::kFlagFree | (prevFree ? detail::kFlagPrevFree : 0);

        detail::FooterOf(h) = size;          // フッタを書く

        // 次のブロックに「前は空きだ」と伝える
        if (Header* n = NextOf(h); n != nullptr)
        {
            n->size |= detail::kFlagPrevFree;
        }
    }

    // ---- 隣のブロック ----

    Header* NextOf(Header* h) const noexcept
    {
        std::byte* p = BytesOf(h) + detail::SizeOf(h);
        return (p < base_ + capacity_) ? reinterpret_cast<Header*>(p) : nullptr;
    }

    // ---- フリーリスト(双方向)----

    void PushFree(Header* h) noexcept
    {
        h->nextFree = freeHead_;
        detail::PrevFreeOf(h) = nullptr;

        if (freeHead_ != nullptr) { detail::PrevFreeOf(freeHead_) = h; }
        freeHead_ = h;
    }

    void UnlinkFree(Header* h) noexcept
    {
        Header* prev = detail::PrevFreeOf(h);
        Header* next = h->nextFree;

        if (prev != nullptr) { prev->nextFree = next; }
        else                 { freeHead_ = next; }

        if (next != nullptr) { detail::PrevFreeOf(next) = prev; }
    }
};
```

### 順序が重要な箇所

**`Free` の中で、①後ろ → ②前 の順に処理しています。** 逆でも動きますが、前と合体した後に `h` が移動するため、後ろの合体を先にやるほうが素直に書けます。

**`MarkFree` は必ず最後に呼びます。** サイズが確定してからでないと、フッタを間違った位置に書いてしまいます。

**`UnlinkFree` を合体の前に呼びます。** 合体してサイズを変えた後だと、フッタの位置が変わって整合が取れなくなります。

---

## 24.6 動かして確かめる

```cpp
int main()
{
    ga::FreeList heap(1024);

    void* a = heap.Allocate(100);
    void* b = heap.Allocate(100);
    void* c = heap.Allocate(100);
    void* d = heap.Allocate(100);

    std::println("--- 4個確保 ---");
    DumpBlocks(heap);

    heap.Free(b);
    std::println("--- b 解放 ---");
    DumpBlocks(heap);

    heap.Free(c);
    std::println("--- c 解放(b と合体するはず)---");
    DumpBlocks(heap);

    heap.Free(a);
    std::println("--- a 解放(bc と合体するはず)---");
    DumpBlocks(heap);

    (void)d;
}
```

```
--- 4個確保 ---
  +0        112 バイト  使用中
  +112      112 バイト  使用中
  +224      112 バイト  使用中
  +336      112 バイト  使用中
  +448      576 バイト  空き

--- b 解放 ---
  +0        112 バイト  使用中
  +112      112 バイト  空き
  +224      112 バイト  使用中
  +336      112 バイト  使用中
  +448      576 バイト  空き

--- c 解放(b と合体するはず)---
  +0        112 バイト  使用中
  +112      224 バイト  空き      ← 112 + 112 = 224
  +336      112 バイト  使用中
  +448      576 バイト  空き

--- a 解放(bc と合体するはず)---
  +0        336 バイト  空き      ← 112 + 224 = 336
  +336      112 バイト  使用中
  +448      576 バイト  空き
```

**合体しています。**

`c` を解放したときは **前と合体**(b と)、`a` を解放したときは **後ろと合体**(bc と)しました。両方向が動いています。

### テスト

```cpp
void Test_CoalesceBoth()
{
    ga::FreeList heap(4096);

    void* a = heap.Allocate(64);
    void* b = heap.Allocate(64);
    void* c = heap.Allocate(64);
    void* d = heap.Allocate(64);

    heap.Free(a);
    heap.Free(c);

    // この時点で:空き(a) 使用中(b) 空き(c) 使用中(d) 空き(残り)
    assert(CountFreeBlocks(heap) == 3);

    heap.Free(b);   // 前(a)とも後ろ(c)とも合体する

    // 空き(a+b+c) 使用中(d) 空き(残り)
    assert(CountFreeBlocks(heap) == 2);

    (void)d;
    std::println("[ OK ] Test_CoalesceBoth");
}

void Test_CoalesceRestoresFullBlock()
{
    ga::FreeList heap(4096);

    std::vector<void*> ptrs;
    for (int i = 0; i < 20; ++i) { ptrs.push_back(heap.Allocate(100)); }

    for (void* p : ptrs) { heap.Free(p); }

    // すべて解放したら、板全体が1つの空きブロックに戻るはず
    assert(CountFreeBlocks(heap) == 1);

    // 最初と同じ大きさの確保ができる
    void* big = heap.Allocate(4000);
    assert(big != nullptr);

    std::println("[ OK ] Test_CoalesceRestoresFullBlock");
}
```

**2つ目のテストが重要です。** 「全部解放したら元に戻る」——当たり前に思えますが、合体がなければ成り立ちません。第23章の実装では、20 個の穴が残ったままでした。

---

## 24.7 実験:断片化はどうなったか

第23章とまったく同じ負荷をかけます。乱数の種も同じです。

### 第23章(合体なし)

```
step      0: 空き     1 個  最大  1048576  外部断片化 0.000  失敗    0
step   5000: 空き   612 個  最大    41248  外部断片化 0.847  失敗    0
step  10000: 空き  1489 個  最大     8192  外部断片化 0.961  失敗   38
step  15000: 空き  2201 個  最大     4096  外部断片化 0.983  失敗  412
step  20000: 空き  2874 個  最大     4096  外部断片化 0.986  失敗 1108
```

### 第24章(合体あり)

```
step      0: 空き     1 個  最大  1048576  外部断片化 0.000  失敗    0
step   5000: 空き    38 個  最大   204800  外部断片化 0.412  失敗    0
step  10000: 空き    57 個  最大   151552  外部断片化 0.487  失敗    0
step  15000: 空き    72 個  最大   118784  外部断片化 0.531  失敗    0
step  20000: 空き    84 個  最大    98304  外部断片化 0.558  失敗    0
```

### 何が変わったか

| | 合体なし | 合体あり | 比 |
|---|---|---|---|
| 空きブロック数 | 2874 | **84** | 1/34 |
| 最大の空き | 4 KB | **96 KB** | 24 倍 |
| 外部断片化 | 0.986 | **0.558** | — |
| 確保の失敗 | 1108 回 | **0 回** | — |

**失敗がゼロになりました。**

第20章のシミュレーターで見た傾向(失敗 9821 → 412)と同じです。紙の上、シミュレーター、実物——3段階すべてで、同じ結論が出ました。

### 絵の変化

```
合体なし(step 20000):
█░█░█░█░█░█░░█░█░█░█░█░█░█░█░░█░█░█░█░█░

合体あり(step 20000):
███████░░░░░░████████░░░░░░░░░░████░░░░░
```

**穴が塊になりました。** 細かい市松模様が消え、まとまった空き領域が見えます。

### 安定している

もう1つ注目すべき点があります。合体ありの数字は、**時間とともに悪化しますが、頭打ちになっています**。

```
空きブロック数: 38 → 57 → 72 → 84
外部断片化    : 0.412 → 0.487 → 0.531 → 0.558
```

増え方が鈍っています。合体なしでは 612 → 1489 → 2201 → 2874 と、ほぼ線形に増え続けていました。

**これは第20章のコラムで触れた「50パーセント則」に関係します。** 解放が穴を作り、確保と合体が穴を消す。両者が釣り合う点で、系が定常状態に落ち着きます。

---

## 24.8 コストを測る

合体は無料ではありません。

### 解放が重くなる

```
Free (合体なし)   median=      3.1 ns
Free (合体あり)   median=      7.4 ns
```

**2.4 倍。** やっていることは、

- 次のブロックのヘッダを読む(メモリアクセス1回)
- フラグを見る
- 場合によってフッタを読む(メモリアクセス1回)
- フリーリストから外す(双方向なので O(1))
- フッタを書き、次のブロックのフラグを更新する

**すべて O(1) です。** ブロック数がいくら増えても、この時間は変わりません。

### 確保が劇的に軽くなる

```
                     合体なし    合体あり
step  5000:            412 ns      31 ns
step 10000:           1180 ns      38 ns
step 20000:           2940 ns      45 ns
```

**65 倍速くなりました。**

理由は明快です。フリーリストの長さが 2874 から 84 に減ったので、first fit の走査が短くなりました。

```
平均探索ステップ:
  合体なし: 1348.9
  合体あり:   11.2
```

**解放で 4.3 ns 払って、確保で 2895 ns 取り返しています。**

### 他の実装との比較

```
Bump   (第5章)          1.8 ns
Pool   (第22章)         2.9 ns
FreeList 合体あり       45.0 ns   ← 今回
new                    17.6 ns
FreeList 合体なし     2940.0 ns
```

**`new` にはまだ負けています。** 2.5 倍遅い。

first fit の走査が残っているためです。空きブロックが 84 個あれば、平均で 11 個は見ることになります。

**次章で、ここに手を入れます。**

---

## 24.9 即時合体と遅延合体

私たちは `Free` の中で即座に合体しました。これを **即時合体**(immediate coalescing)と呼びます。

別の方針もあります。

### 遅延合体(deferred coalescing)

解放時には合体せず、リストに繋ぐだけ。**確保が失敗しそうになったときに、まとめて合体します。**

| | 即時合体 | 遅延合体 |
|---|---|---|
| `Free` のコスト | やや高い | **最小** |
| 直前に解放したブロックの再利用 | 合体されて分割し直し | **そのまま返せる** |
| 実装 | 単純 | 複雑 |

### 遅延が有利な場面

**同じサイズを確保・解放し続ける場合** です。

```cpp
for (int i = 0; i < 1000000; ++i)
{
    void* p = heap.Allocate(64);
    // 使う
    heap.Free(p);
}
```

即時合体では、解放のたびに隣と合体して大きなブロックになり、次の確保で再び 64 バイトに分割されます。**合体と分割を延々と繰り返します。**

遅延合体なら、64 バイトのブロックがそのまま次の確保に返ります。**無駄がありません。**

### 実物のアロケーターは併用している

glibc の malloc には **fastbins** という仕組みがあります。小さいブロックは解放しても合体せず、専用のリストに置いておきます。同じサイズの要求が来たら、そのまま返します。

そして、大きな確保が失敗しそうになったときに、fastbins をまとめて合体します。**即時と遅延のいいとこ取り** です。

本書では即時合体のまま進めます。ただし、**次章のサイズ別ビンが、遅延合体と似た効果を持つ** ことになります。同じサイズのブロックが、同じビンに戻ってくるからです。

---

## 演習

**演習24-1** `MarkFree` の中で「次のブロックに `PREV_FREE` を立てる」処理を消してください。何が起きますか。どのタイミングで壊れますか。

**演習24-2** フッタを使用中ブロックにも書くようにしてください。オーバーヘッドはどれだけ増えますか。`PREV_FREE` フラグは不要になりますか。

**演習24-3** 24.7 の実験を、確保サイズをすべて 64 バイトに固定して実行してください。合体の効果はありますか。

**演習24-4** 遅延合体を実装してください。24.9 のループ(同じサイズの確保・解放を繰り返す)で、どれだけ速くなりますか。

**演習24-5** フリーリストを **アドレス順** に保つよう変更してください。挿入が O(n) になりますが、合体は簡単になりますか。(これは K&R malloc の設計です)

**演習24-6** 合体の回数を数える `coalesceCount_` を表示してください。20,000 ステップで何回合体が起きますか。解放回数との比はどうですか。

**演習24-7** `Free` の中で、後ろと前の合体の順序を入れ替えてください。正しく動きますか。動かないなら、どこを直す必要がありますか。

---

## 章末チェックリスト

- [ ] 「前の隣」を O(1) で見つける必要性を説明できる
- [ ] 境界タグの仕組みを図で説明できる
- [ ] **使用中ブロックにフッタが不要** な理由を説明できる
- [ ] `PREV_FREE` フラグの役割と、更新を忘れたときの症状を説明できる
- [ ] フリーリストを双方向にし、`prevFree` を空きブロック内に置いた
- [ ] 前後両方向の合体を実装した 〔v0.17〕
- [ ] 「全部解放したら1つの空きブロックに戻る」テストが通った
- [ ] 断片化と失敗回数が劇的に改善することを測った
- [ ] 解放が 2.4 倍重くなり、確保が 65 倍軽くなることを確認した

---

## 次章の予告

合体で、断片化は制御下に入りました。**残る問題は探索です。**

```
FreeList(合体あり) : 45 ns
new                : 17.6 ns
```

まだ 2.5 倍負けています。原因は first fit の線形探索です。空きブロックが 84 個あれば、平均 11 個を辿ることになります。しかもポインタを追うので、キャッシュに乗りません。

第25章では、フリーリストを **1本から複数本に分けます**。

```
32〜63 バイト   → ビン0
64〜127 バイト  → ビン1
128〜255 バイト → ビン2
...
```

64 バイトが欲しければ、ビン1の先頭を見るだけ。**探索がほぼ O(1) になります。**

代償は内部断片化です。ビンの粒度が粗ければ、要求より大きなブロックを返すことになります。第15章で作ったサイズ分布のヒストグラムが、ここで役に立ちます。

---

> **コラム:K&R malloc と `sbrk` の時代**
>
> C 言語の教科書として名高い『プログラミング言語C』(K&R)の最後の節に、`malloc` の実装が載っています。8.7 節「記憶割当ての例」です。わずか 60 行ほどのコードですが、本章までに扱った要素がほとんど詰まっています。
>
> ---
>
> **ヘッダの定義が美しい**
>
> ```c
> typedef long Align;    /* 最も厳しい境界調整のための型 */
>
> union header {
>     struct {
>         union header *ptr;   /* 空きブロックの循環リスト */
>         unsigned size;       /* このブロックの大きさ */
>     } s;
>     Align x;                 /* ブロックの境界調整を強制 */
> };
> ```
>
> `union` に `Align x` を入れているのが工夫です。**この共用体全体が `long` と同じ境界に整列される** ことを保証しています。C++ の `alignas` に相当するものが、当時はまだありませんでした。
>
> 私たちが第6章で `alignof` / `alignas` を使ったところを、当時は型システムの副作用で実現していたわけです。
>
> ---
>
> **アドレス順の循環リスト**
>
> K&R malloc の最大の特徴は、フリーリストを **アドレス順に並べた循環リスト** にしていることです。
>
> ```
> free → [0x1000] → [0x3000] → [0x5000] ─┐
>          ▲                              │
>          └──────────────────────────────┘
> ```
>
> この設計だと、**合体が自然に書けます**。解放したブロックを挿入する位置を探す過程で、前後の隣がすでに分かるからです。境界タグが要りません。
>
> 代償は挿入コストです。O(n) の探索が必要になります。私たちは境界タグを使い、挿入を O(1) にして、代わりにフッタとフラグを持ちました。**どちらを取るかは設計判断です**(演習24-5)。
>
> ---
>
> **`sbrk` という原始的な仕組み**
>
> K&R malloc がメモリを OS から得る手段は `sbrk` でした。
>
> ```
>        ┌──────────────┐
>        │   スタック    │  ↓ 下へ伸びる
>        │              │
>        │    (空き)     │
>        │              │
>        │   ヒープ      │  ↑ sbrk で上へ伸ばす
>        ├──────────────┤
>        │   データ      │
>        │   コード      │
>        └──────────────┘
> ```
>
> `sbrk(n)` は、ヒープの末端(break)を `n` バイト押し上げ、その領域を使えるようにします。**1本の連続した領域が、一方向に伸びるだけ** です。
>
> 何かに似ていませんか。**私たちの `Bump` です。**
>
> そして、`sbrk` には重大な制約がありました。**縮められるのは末端だけ** です。ヒープの真ん中が空いても、OS に返せません。末端に使用中のブロックが1つでもあれば、その手前の空きは永久に返却できない。
>
> ---
>
> **`mmap` の登場**
>
> この制約を解いたのが `mmap`(Windows なら `VirtualAlloc`)です。任意のアドレスに、任意のサイズの領域を、独立して確保・解放できます。
>
> 現代のアロケーターは、大きな確保を `mmap` で個別に取り、解放時にそのまま OS へ返します。glibc の malloc も、しきい値(既定 128 KB)を超える要求は `mmap` を使います。
>
> **第29章で、私たちも同じ転換を行います。** `std::vector<std::byte>` という「一度確保したら固定」の土台を捨てて、`VirtualAlloc` による予約とコミットの分離に移ります。
>
> ---
>
> **60行のコードが教えてくれること**
>
> K&R malloc は、1978年の本に載ったコードです。40年以上前のものです。
>
> しかし、そこにあるのは——ヘッダ、フリーリスト、アラインメント、合体、OS からの追加取得——**この本の第3部で扱っている要素そのもの** です。
>
> 変わったのは規模と要求水準です。マルチスレッド、キャッシュ、断片化への耐性、セキュリティ。それらに応えるために実装は複雑になりましたが、**骨格は変わっていません**。
>
> もし手元に K&R があるなら、8.7 節を読んでみてください。第23章と第24章を書き終えたいま、あのコードが以前とは違って見えるはずです。
