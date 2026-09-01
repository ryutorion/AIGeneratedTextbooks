# 第18章 small_vector<T, N> ―― SBO を Condition で分岐

第4部の締めくくりは、**要素がどこにあるかが実行時に変わる**型です。

小容量バッファ最適化（SBO / SSO）は、要素数が少ないうちは構造体の内部に、多くなったらヒープに置く手法です。`std::string` の短い文字列最適化と同じ考え方で、実務ではよく出てきます。

Natvis 側では、`Condition` 付きの `ArrayItems` を2つ並べることで対応します。

## 18.1 型を作る

`vizkit\include\vizkit\seq\small_vector.hpp`

```cpp
#pragma once

#include <cstddef>
#include <new>
#include <utility>

namespace vizkit {

// 要素数が N 以下ならインライン領域、超えたらヒープを使う。
//   _capacity <= N   → インライン格納（_sbo を使用）
//   _capacity >  N   → ヒープ格納（_heap を使用）
template <class T, std::size_t N>
class small_vector {
public:
    small_vector() = default;

    ~small_vector() {
        clear();
        if (!inline_active()) ::operator delete(static_cast<void*>(_heap));
    }

    small_vector(const small_vector&) = delete;
    small_vector& operator=(const small_vector&) = delete;

    void push_back(const T& v) {
        if (_size == _capacity) grow();
        new (data() + _size) T(v);
        ++_size;
    }

    void clear() noexcept {
        T* p = data();
        while (_size != 0) { --_size; p[_size].~T(); }
    }

    [[nodiscard]] std::size_t size()     const noexcept { return _size; }
    [[nodiscard]] std::size_t capacity() const noexcept { return _capacity; }
    [[nodiscard]] bool inline_active()   const noexcept { return _capacity <= N; }

    [[nodiscard]] T* data() noexcept {
        return inline_active() ? reinterpret_cast<T*>(_sbo) : _heap;
    }

private:
    void grow() {
        const std::size_t new_cap = _capacity == 0 ? N : _capacity * 2;
        T* p = static_cast<T*>(::operator new(sizeof(T) * new_cap));
        T* old = data();
        for (std::size_t i = 0; i < _size; ++i) {
            new (p + i) T(std::move(old[i]));
            old[i].~T();
        }
        if (!inline_active()) ::operator delete(static_cast<void*>(old));
        _heap = p;
        _capacity = new_cap;
    }

    std::size_t _size     = 0;
    std::size_t _capacity = N;
    union {
        alignas(T) unsigned char _sbo[sizeof(T) * N];
        T* _heap;
    };
};

} // namespace vizkit
```

`main.cpp`：

```cpp
#include <vizkit/seq/small_vector.hpp>
// ...
    vizkit::small_vector<int, 4> sv_small;      // インラインのまま
    sv_small.push_back(1);
    sv_small.push_back(2);
    sv_small.push_back(3);

    vizkit::small_vector<int, 4> sv_large;      // ヒープに移行する
    for (int i = 1; i <= 7; ++i) sv_large.push_back(i);
```

## 18.2 Natvis なしでは絶望的

素の表示を見てください。

```
▷ sv_small      {_size=3 _capacity=4 {_sbo=0x... _heap=0x...} }
     _size      3
     _capacity  4
   ▷ _sbo       0x00000078abcff5c0 "\x1"
   ▷ _heap      0x0000000300000002 {???}
```

`_sbo` は `unsigned char[16]` なので、デバッガは文字列として解釈しようとします。`_heap` は同じメモリを `T*` として読んだ結果なので、**まったく無意味なポインタ**です。

union の両側が同時に表示され、しかもどちらが有効なのか分からない。第3章で見たどのケースより読めません。

## 18.3 判定を Intrinsic に閉じ込める

まず、「どちらのモードか」の判定に名前を付けます。C++ 側の `inline_active()` と同じ式です。

```xml
<Intrinsic Name="is_inline" Expression="_capacity &lt;= $T2" />
```

この式が Natvis 全体の基礎になるので、`Intrinsic` にしておく価値は高いです（第5章 5.5節）。実装の判定条件を変えたときに、直す場所が1か所で済みます。

## 18.4 Condition 付きの ArrayItems を2つ並べる

```xml
<Type Name="vizkit::small_vector&lt;*,*&gt;">
  <Intrinsic Name="is_inline" Expression="_capacity &lt;= $T2" />

  <DisplayString Condition="_size == 0">[empty]</DisplayString>
  <DisplayString Condition="is_inline()">{{ size={_size} (inline) }}</DisplayString>
  <DisplayString>{{ size={_size} (heap) }}</DisplayString>

  <Expand>
    <Item Name="[size]"     ExcludeView="simple">_size</Item>
    <Item Name="[capacity]" ExcludeView="simple">_capacity</Item>
    <Item Name="[mode]"     ExcludeView="simple">is_inline() ? "inline" : "heap"</Item>

    <ArrayItems Condition="is_inline()">
      <Size>_size</Size>
      <ValuePointer>($T1*)_sbo</ValuePointer>
    </ArrayItems>

    <ArrayItems Condition="!is_inline()">
      <Size>_size</Size>
      <ValuePointer>_heap</ValuePointer>
    </ArrayItems>
  </Expand>
</Type>
```

```
▷ sv_small      { size=3 (inline) }         vizkit::small_vector<int,4>
     [size]     3
     [capacity] 4
     [mode]     inline
     [0]        1
     [1]        2
     [2]        3
   ▷ [Raw View] {_size=3 _capacity=4 ... }

▷ sv_large      { size=7 (heap) }           vizkit::small_vector<int,4>
     [size]     7
     [capacity] 8
     [mode]     heap
     [0]        1
     ...
     [6]        7
```

読めるようになりました。しかも `[mode]` の行があるので、**いつヒープに移行したかをデバッガ上で追える**ようになっています。SBO の実装をデバッグするときには、この行が主役になります。

## 18.5 生バッファをキャストする ―― `($T1*)_sbo`

インライン側の `ValuePointer` に注目してください。

```xml
<ValuePointer>($T1*)_sbo</ValuePointer>
```

`_sbo` は `unsigned char[16]` です。`ArrayItems` の `ValuePointer` は「要素型のポインタでなければならない」（`void*` は不可）という制約がありました（第14章 14.3節）。そこで `$T1*`（この場合 `int*`）にキャストしています。

**`$T1` が「型」であることが効く場面**です。第14章 14.4節で触れた区別が、ここで実際に必要になります。

C形式キャストを使う点にも注意してください（第5章 5.4節）。`reinterpret_cast` は書けません。

### alignas との関係

`_sbo` に `alignas(T)` を付けているのは C++ 側の要請ですが、Natvis 側にも意味があります。アラインメントが合っていなければ、キャストして読んだ値が壊れます。デバッガはアラインメント違反を検出しないので、**壊れた値が「正しそうに」表示されます**。

もし表示がおかしいと感じたら、第11章 11.5節で作った `[alignment]` グループのような診断を足して、アドレスを確認してみてください。

## 18.6 条件が排他でないと二重に出る

重要な注意点です。**`Condition` が `true` になった `ArrayItems` は、すべて列挙されます。** 「最初にマッチしたものを採用して打ち切り」という `DisplayString` のルール（第7章 7.3節）とは**違います**。

試しに、2つ目の `Condition` を消してみてください。

```xml
<ArrayItems Condition="is_inline()">
  <Size>_size</Size>
  <ValuePointer>($T1*)_sbo</ValuePointer>
</ArrayItems>

<ArrayItems>                        <!-- Condition を消した -->
  <Size>_size</Size>
  <ValuePointer>_heap</ValuePointer>
</ArrayItems>
```

```
▷ sv_small      { size=3 (inline) }
     [0]        1                ← インライン側
     [1]        2
     [2]        3
     [0]        -858993460       ← ヒープ側（無効なポインタを読んでいる）
     [1]        -858993460
     [2]        -858993460
```

要素が二重に並び、しかも後半は無効なメモリの内容です。

**`Expand` の中の列挙要素は「並列に並ぶ」ものであり、「排他選択」ではありません。** 排他にしたければ、条件を自分で排他に書く必要があります。

```xml
<ArrayItems Condition="is_inline()">   ...
<ArrayItems Condition="!is_inline()">  ...
```

`is_inline()` と `!is_inline()` は定義上排他なので安全です。`_capacity <= $T2` と `_capacity > $T2` のように別々に書くよりも、**否定で書くほうが「条件を片方だけ直して不整合になる」事故を防げます**。

## 18.7 どちらも false のときの保険

`Intrinsic` の解析が失敗したり、条件式が期待どおりに評価されなかった場合、**両方の `ArrayItems` が飛ばされて要素が1つも出ません**。しかもエラーは出ないので、「空のコンテナ」に見えてしまいます。

第8章 8.7節で扱った「フォールバックを置く」考え方を、`Expand` にも適用します。

```xml
<Synthetic Name="[!]" Condition="_size != 0 &amp;&amp; !is_inline() &amp;&amp; _heap == nullptr">
  <DisplayString>heap モードなのに _heap が null です。破損の可能性があります。</DisplayString>
</Synthetic>
```

さらに慎重にするなら、モードの判定そのものが破綻していないかも見ます。

```xml
<DisplayString Condition="_size &gt; _capacity">[corrupt: size={_size} &gt; capacity={_capacity}]</DisplayString>
```

`_size <= _capacity` は `small_vector` の不変条件です。第15章・第17章と同じパターンで、Natvis に書き込んでおきます。

## 18.8 std::string の SSO と同じ構造

第9章 9.7節で紹介した `STL.natvis` の `std::string` を、あらためて見てください。

```xml
<DisplayString Condition="isShortString()">{_Mypair._Myval2._Bx._Buf,na}</DisplayString>
<DisplayString Condition="isLongString()">{_Mypair._Myval2._Bx._Ptr,na}</DisplayString>
```

構造は本章とまったく同じです。

| | `std::string` | `vizkit::small_vector` |
|---|---|---|
| 判定 | `isShortString()` / `isLongString()`（`Intrinsic`） | `is_inline()`（`Intrinsic`） |
| インライン側 | `_Bx._Buf` | `($T1*)_sbo` |
| ヒープ側 | `_Bx._Ptr` | `_heap` |
| 分岐 | `Condition` 付き `DisplayString` | `Condition` 付き `ArrayItems` |

`std::string` は「値の表示」なので `DisplayString` を分岐させ、`small_vector` は「要素の列挙」なので `ArrayItems` を分岐させている。**分岐する対象が違うだけで、考え方は同一**です。

MSVC が `isShortString` / `isLongString` という2つの `Intrinsic` を定義しているのは、`!isShortString()` と書かずに済ませるためでしょう。どちらの流儀でも構いませんが、**判定を `Intrinsic` に閉じ込める**という点は共通しています。

## 18.9 完成形

```xml
  <Type Name="vizkit::small_vector&lt;*,*&gt;">
    <Intrinsic Name="is_inline" Expression="_capacity &lt;= $T2" />

    <DisplayString Condition="_size &gt; _capacity">[corrupt: size={_size} &gt; capacity={_capacity}]</DisplayString>
    <DisplayString Condition="_size == 0">[empty] ({is_inline() ? "inline" : "heap"}, capacity={_capacity})</DisplayString>
    <DisplayString Condition="is_inline()">{{ size={_size} (inline) }}</DisplayString>
    <DisplayString>{{ size={_size} (heap) }}</DisplayString>

    <Expand>
      <Item Name="[size]"     ExcludeView="simple">_size</Item>
      <Item Name="[capacity]" ExcludeView="simple">_capacity</Item>
      <Item Name="[mode]"     ExcludeView="simple">is_inline() ? "inline" : "heap"</Item>

      <Synthetic Name="[!]" Condition="_size != 0 &amp;&amp; !is_inline() &amp;&amp; _heap == nullptr">
        <DisplayString>heap モードなのに _heap が null です。破損の可能性があります。</DisplayString>
      </Synthetic>

      <ArrayItems Condition="is_inline()">
        <Size>_size</Size>
        <ValuePointer>($T1*)_sbo</ValuePointer>
      </ArrayItems>

      <ArrayItems Condition="!is_inline()">
        <Size>_size</Size>
        <ValuePointer>_heap</ValuePointer>
      </ArrayItems>

      <Synthetic Name="[storage]" IncludeView="detail">
        <DisplayString>inline capacity={$T2}, sizeof(T)={sizeof($T1)}</DisplayString>
        <Expand>
          <Item Name="[sbo address]">($T1*)_sbo</Item>
          <Item Name="[heap pointer]" Condition="!is_inline()">_heap</Item>
        </Expand>
      </Synthetic>
    </Expand>
  </Type>
```

## 18.10 第4部のまとめ

第14章から第18章までで、配列系コンテナの可視化を一通り扱いました。

```
ArrayItems           連続領域を配列として列挙する（最速。使えるなら常にこれ）
 ├─ Size             要素数（整数式）
 ├─ ValuePointer     先頭要素へのポインタ（要素型。void* 不可）
 ├─ LowerBound       表示上の添字の開始値（既定 0）
 ├─ Rank             次元数。Size の中の $i が次元インデックスになる
 └─ Direction        Forward（行優先）/ Backward（列優先）

IndexListItems       i 番目の式を自分で書く（添字変換が必要なとき）
 ├─ Size             要素数
 └─ ValueNode        i 番目の要素そのものの式。$i は要素インデックス

共通
 ├─ Condition        排他条件で複数並べると分岐になる（並列に列挙される点に注意）
 └─ IncludeView/ExcludeView   ビューごとの出し分け
```

ここまでで扱えるのは、**要素の位置が添字から計算できる**型です。

次の第5部からは、位置が計算できない型 ―― ポインタを辿らなければ次の要素が分からない、リストや木に入ります。`LinkedListItems` と `TreeItems` の世界です。

## 18.11 この章のまとめ

- SBO 型は `Condition` 付きの `ArrayItems` を2つ並べて分岐する。
- **判定は `Intrinsic` に閉じ込める。** `is_inline()` を1か所に定義し、条件も表示もそれを参照する。
- 生バイトバッファは `($T1*)` にキャストして `ValuePointer` に渡す。**`$T1` が型であることがここで効く。**
- **`Expand` の中の列挙要素は、条件が `true` のものが「すべて」列挙される。** `DisplayString` のような打ち切りはない。条件は否定形で排他に書く。
- どちらの条件も外れると、エラーなしで「空」に見える。破損検出の `Synthetic` を保険に置く。
- `[mode]` の行は、SBO の実装をデバッグするときの主役になる。
- `std::string` の SSO と構造は同一。分岐する対象が `DisplayString` か `ArrayItems` かの違いだけ。

## 18.12 演習

1. 2つ目の `ArrayItems` から `Condition` を外し、要素が二重に並ぶことを確認してください。第7章の `DisplayString` の挙動との違いが要点です。
2. `sv_small` に `push_back` を繰り返し、`[mode]` が `inline` から `heap` に変わる瞬間をステップ実行で捉えてください。
3. `is_inline()` の定義を `_capacity &lt; $T2`（`=` を落とす）に変え、`sv_small` がどう表示されるか確認してください。オフバイワンが表示にどう現れるかの実験です。
4. `small_vector<vizkit::point2, 2>` を作り、インライン領域に `point2` が正しく並ぶことを確認してください。`sizeof(T)` が 16 になるので、`_sbo` は 32 バイトになります。
5. `sv_large,view(detail)` で `[storage]` を開き、`_sbo` のアドレスと `_heap` の値が別物であることを確認してください。ヒープモードでは `_sbo` を読んでも無意味であることが分かります。
