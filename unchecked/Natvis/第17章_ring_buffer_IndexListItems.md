# 第17章 ring_buffer<T, N> ―― IndexListItems

ここまでの3章では、**論理的な順序と物理的な配置が一致している**型を扱ってきました。「i 番目の要素は先頭から i × 要素サイズの位置にある」という関係です。

リングバッファでは、この関係が崩れます。`ArrayItems` は使えません。`<IndexListItems>` の出番です。

## 17.1 型を作る

`vizkit\include\vizkit\seq\ring_buffer.hpp`

```cpp
#pragma once

#include <cstddef>

namespace vizkit {

// 固定容量のリングバッファ（FIFO）。
//   _head  : 論理的な先頭が置かれている物理インデックス
//   _count : 現在の要素数
// 物理位置 = (_head + 論理インデックス) % N
template <class T, std::size_t N>
class ring_buffer {
public:
    [[nodiscard]] std::size_t size()     const noexcept { return _count; }
    [[nodiscard]] std::size_t capacity() const noexcept { return N; }
    [[nodiscard]] bool        full()     const noexcept { return _count == N; }

    void push_back(const T& v) noexcept {
        if (full()) {
            _buf[_head] = v;                 // 最古を上書き
            _head = (_head + 1) % N;
        } else {
            _buf[(_head + _count) % N] = v;
            ++_count;
        }
    }

    void pop_front() noexcept {
        if (_count == 0) return;
        _head = (_head + 1) % N;
        --_count;
    }

    [[nodiscard]] T& operator[](std::size_t i) noexcept { return _buf[(_head + i) % N]; }

private:
    T           _buf[N]{};
    std::size_t _head  = 0;
    std::size_t _count = 0;
};

} // namespace vizkit
```

`main.cpp`：

```cpp
#include <vizkit/seq/ring_buffer.hpp>
// ...
    vizkit::ring_buffer<int, 5> rb;
    for (int i = 1; i <= 7; ++i) rb.push_back(i * 100);
    // 7 個入れたので 2 回上書きされ、_head は 2 に進んでいる
    // 論理的な中身: 300, 400, 500, 600, 700
    // 物理バッファ: [600, 700, 300, 400, 500]

    vizkit::ring_buffer<int, 5> rb_empty;
```

## 17.2 なぜ ArrayItems では無理か

素の表示を見ます。

```
▷ rb            {_buf=0x... {600, 700, 300, 400, 500} _head=2 _count=5 }
```

物理バッファには `600, 700, 300, 400, 500` が入っています。しかし**論理的には `300, 400, 500, 600, 700` の順**です。

`ArrayItems` で書くと、こうなります。

```xml
<!-- 間違い -->
<ArrayItems>
  <Size>_count</Size>
  <ValuePointer>_buf</ValuePointer>
</ArrayItems>
```

```
     [0]        600     ← 本当は 300
     [1]        700     ← 本当は 400
     [2]        300
     ...
```

**エラーは出ません。順序が違うだけです。** これは第16章の `Direction` の間違いと同じ種類の危険です。Natvis は表示を作るだけで、正しさを保証しません。

`ArrayItems` の制約は、`ValuePointer` と `Size` しか与えられないことです。デバッガは「先頭 ＋ i × サイズ」という計算しかできません。剰余を挟む余地がないのです。

## 17.3 IndexListItems

`<IndexListItems>` は、**i 番目の要素を求める式を、こちらが完全に書く**方式です。

```xml
<Type Name="vizkit::ring_buffer&lt;*,*&gt;">
  <DisplayString>{{ size={_count} }}</DisplayString>
  <Expand>
    <Item Name="[size]"     ExcludeView="simple">_count</Item>
    <Item Name="[capacity]" ExcludeView="simple">$T2</Item>
    <IndexListItems>
      <Size>_count</Size>
      <ValueNode>_buf[(_head + $i) % $T2]</ValueNode>
    </IndexListItems>
  </Expand>
</Type>
```

```
▷ rb            { size=5 }              vizkit::ring_buffer<int,5>
     [size]     5
     [capacity] 5
     [0]        300
     [1]        400
     [2]        500
     [3]        600
     [4]        700
   ▷ [Raw View] {_buf=... _head=2 _count=5 }
```

論理順に並びました。C++ 側の `operator[]` に書いた式 `_buf[(_head + i) % N]` を、そのまま Natvis に写しただけです。

### ArrayItems との違いは1点だけ

| | `ArrayItems` | `IndexListItems` |
|---|---|---|
| 要素数 | `<Size>` | `<Size>`（同じ） |
| 要素の求め方 | `<ValuePointer>`（先頭ポインタを渡す） | `<ValueNode>`（**i 番目の式を書く**） |

公式ドキュメントも「唯一の違いは `ValueNode` である」と述べています。

`<ValueNode>` に書くのは、**要素そのものへの完全な式**です。ポインタではありません。

```xml
<!-- 正しい: 要素そのもの -->
<ValueNode>_buf[(_head + $i) % $T2]</ValueNode>

<!-- 誤り: ポインタになっている -->
<ValueNode>&amp;_buf[(_head + $i) % $T2]</ValueNode>
```

間違えると、要素の代わりにアドレスが並びます。`ArrayItems` から書き換えるときにやりがちなミスです。

### `$i` の意味が違う

前章の `Rank` で使った `$i` は「次元のインデックス」でした。`IndexListItems` の `$i` は「**要素のインデックス**」です（0 から `Size - 1` まで）。

同じ `$i` という記号が文脈によって別のものを指します。混乱しやすい部分なので、意識しておいてください。

## 17.4 剰余の罠

`(_head + $i) % $T2` という式には、いくつか注意点があります。

**(1) `$T2` が 0 の場合。** `ring_buffer<int, 0>` は意味のない型ですが、テンプレートとしては書けてしまいます。ゼロ除算でデバッガがエラーを出します。実害は小さいので、通常は無視して構いません。

**(2) 符号。** `_head` と `$i` がどちらも符号なしなら問題ありません。しかし `_head` を `int` にしている実装では、負の値が入ったときに `%` の結果が負になり、範囲外アクセスになります。**符号なし型で持つ設計にしておくと、Natvis も安全になります。**

**(3) `_head` の破損。** `_head >= N` になっていれば構造体は壊れています。ガードを入れます。

```xml
<DisplayString Condition="_head &gt;= $T2 || _count &gt; $T2">[corrupt: head={_head} count={_count}]</DisplayString>
```

第15章と同じ、不変条件を Natvis に書き込むパターンです。

## 17.5 物理バッファも見せる

論理順で見えるようになった一方で、「実際のメモリ配置」が見えなくなりました。リングバッファのバグは `_head` の進め方にあることが多いので、物理配置も見たくなります。

第13章のビューと、第11章の `Synthetic` を組み合わせます。

```xml
<Synthetic Name="[physical]" IncludeView="detail">
  <DisplayString>head={_head}, {_count}/{$T2} used</DisplayString>
  <Expand>
    <Item Name="[head]">_head</Item>
    <ArrayItems>
      <Size>$T2</Size>
      <ValuePointer>_buf</ValuePointer>
    </ArrayItems>
  </Expand>
</Synthetic>
```

```
rb,view(detail)
  ├ [size]        5
  ├ [capacity]    5
  ├ [physical]    head=2, 5/5 used
  │    [head]     2
  │    [0]        600
  │    [1]        700
  │    [2]        300      ← ここが論理的な先頭
  │    [3]        400
  │    [4]        500
  ├ [0]           300
  ├ [1]           400
  ...
```

論理ビューと物理ビューを並べて見られるようになりました。**同じデータを2つの見方で提示する**のは、Natvis の実用的な使い方のひとつです。

## 17.6 未使用スロットに残る古い値

`ring_buffer` は要素を破棄しません。`pop_front()` は `_head` を進めるだけなので、**取り出したはずの値がバッファに残ります**。

```cpp
vizkit::ring_buffer<int, 5> rb2;
rb2.push_back(1);
rb2.push_back(2);
rb2.push_back(3);
rb2.pop_front();
rb2.pop_front();
// 論理的な中身: 3 のみ
// 物理バッファ: [1, 2, 3, 0, 0]
```

論理ビューでは正しく `[0] 3` だけが出ます。物理ビューには `1` と `2` が残って見えます。

これは**バグではなく設計どおり**ですが、`[physical]` を見た人が混乱する可能性があります。第11章の `Synthetic` による説明行を添えておくと親切です。

```xml
<Synthetic Name="[physical]" IncludeView="detail">
  <DisplayString>head={_head}, {_count}/{$T2} used (未使用スロットには古い値が残ります)</DisplayString>
  ...
```

**Natvis は、実装の意図を伝えるドキュメントにもなります**（第11章 11.2節）。

## 17.7 ArrayItems と IndexListItems の使い分け

| 状況 | 使う要素 |
|---|---|
| 要素が連続して並んでいる | **`ArrayItems`** |
| 添字の変換が必要（剰余、ストライド、間接参照） | `IndexListItems` |
| ポインタの配列で、要素は指し先（`T**`） | `IndexListItems`（`*_array[$i]`） |
| 要素が飛び飛び（空きスロットがある） | どちらも不可 → `CustomListItems`（第23章） |
| 要素数が事前に分からない | どちらも不可 → `CustomListItems` |

### 性能差を意識する

第14章 14.8節で触れた点を、ここで明確にします。

- **`ArrayItems`** ―― デバッガは「先頭 ＋ i × サイズ」でアドレスを計算する。式の評価は不要
- **`IndexListItems`** ―― **要素ごとに `ValueNode` の式を評価する**

要素数が数十なら差は感じません。しかし数万要素で `IndexListItems` を使うと、スクロールが目に見えて重くなります。

したがって原則はこうです。**添字の変換が本当に必要なときだけ `IndexListItems` を使う。** ストライド付きのビュー（`_data[$i * _stride]`）のような単純な変換であっても、可能なら `ArrayItems` で表現できる設計に寄せるほうが、デバッグ体験は良くなります。

公式ドキュメントの `IndexListItems` の例も、まさに「ポインタの配列の指し先を見せる」という、`ArrayItems` では表現できないケースです。

```xml
<IndexListItems>
  <Size>_M_vector._M_index</Size>
  <ValueNode>*(_M_vector._M_array[$i])</ValueNode>
</IndexListItems>
```

## 17.8 現時点の ring_buffer の Natvis

```xml
  <Type Name="vizkit::ring_buffer&lt;*,*&gt;">
    <DisplayString Condition="_head &gt;= $T2 || _count &gt; $T2">[corrupt: head={_head} count={_count}]</DisplayString>
    <DisplayString Condition="_count == 0">[empty] (capacity={$T2})</DisplayString>
    <DisplayString Condition="_count == $T2">{{ size={_count} (full) }}</DisplayString>
    <DisplayString>{{ size={_count} }}</DisplayString>

    <Expand>
      <Item Name="[size]"     ExcludeView="simple">_count</Item>
      <Item Name="[capacity]" ExcludeView="simple">$T2</Item>

      <Synthetic Name="[physical]" IncludeView="detail">
        <DisplayString>head={_head}, {_count}/{$T2} used (未使用スロットには古い値が残ります)</DisplayString>
        <Expand>
          <Item Name="[head]">_head</Item>
          <ArrayItems>
            <Size>$T2</Size>
            <ValuePointer>_buf</ValuePointer>
          </ArrayItems>
        </Expand>
      </Synthetic>

      <IndexListItems>
        <Size>_count</Size>
        <ValueNode>_buf[(_head + $i) % $T2]</ValueNode>
      </IndexListItems>
    </Expand>
  </Type>
```

## 17.9 この章のまとめ

- 論理順と物理配置が一致しない型には `ArrayItems` を使えない。**エラーにならず、順序だけが狂う**ので危険。
- `<IndexListItems>` は `<Size>` と `<ValueNode>` の2つ。`ArrayItems` との違いは `ValueNode` だけ。
- **`<ValueNode>` には「i 番目の要素そのもの」の式を書く。** ポインタではない。
- `IndexListItems` の `$i` は**要素のインデックス**。`Rank` の `$i`（次元のインデックス）とは別物。
- C++ 側の `operator[]` の式を、そのまま Natvis に写せることが多い。
- 剰余を使うときは、符号なし型であることと、`_head < N` の不変条件を確認する。
- 論理ビューと物理ビューを両方用意すると、リングバッファ特有のバグを追いやすい。
- **`IndexListItems` は要素ごとに式を評価するので `ArrayItems` より遅い。** 添字変換が本当に必要なときだけ使う。

## 17.10 演習

1. `IndexListItems` を `ArrayItems` に書き換え（`ValuePointer` を `_buf` に）、順序が狂うことを確認してください。エラーが出ないことが要点です。
2. `<ValueNode>&amp;_buf[(_head + $i) % $T2]</ValueNode>` と書いて、アドレスが並ぶことを確認してください。
3. デバッガから `_head` に `99` を書き込み、`[corrupt]` が出ることを確認してください。ガードを外した場合に何が表示されるかも見てください。
4. `rb2` を作り、`pop_front()` を2回呼んだ状態で `rb2,view(detail)` を開き、物理バッファに古い値が残っていることを確認してください。
5. `ring_buffer<vizkit::point2, 4>` を作り、要素の Natvis が `IndexListItems` 経由でも効くことを確認してください。
