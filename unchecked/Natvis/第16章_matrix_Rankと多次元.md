# 第16章 matrix<T, Rows, Cols> ―― Rank と多次元

多次元配列は、C++ では1次元の連続領域として持つのが普通です。行優先か列優先か、という「約束」だけが違います。

デバッガはその約束を知りません。`ArrayItems` の `<Rank>` と `<Direction>` は、その約束を伝えるための要素です。

## 16.1 型を作る

`vizkit\include\vizkit\seq\matrix.hpp`

```cpp
#pragma once

#include <cstddef>

namespace vizkit {

// 行優先（row-major）の固定サイズ行列。
// _data[row * Cols + col] が (row, col) 要素。
template <class T, std::size_t Rows, std::size_t Cols>
struct matrix {
    T _data[Rows * Cols];

    [[nodiscard]] constexpr T&       at(std::size_t r, std::size_t c)       noexcept { return _data[r * Cols + c]; }
    [[nodiscard]] constexpr const T& at(std::size_t r, std::size_t c) const noexcept { return _data[r * Cols + c]; }
};

} // namespace vizkit
```

`main.cpp`：

```cpp
#include <vizkit/seq/matrix.hpp>
// ...
    vizkit::matrix<int, 3, 4> grid{};
    for (std::size_t r = 0; r < 3; ++r)
        for (std::size_t c = 0; c < 4; ++c)
            grid.at(r, c) = static_cast<int>(r * 10 + c);

    vizkit::matrix<double, 2, 2> m2{ { 1.0, 2.0, 3.0, 4.0 } };
```

素の表示は「12 個の int が一列に並んだもの」です。

```
▷ grid       {_data=0x... {0, 1, 2, 3, 10, 11, 12, 13, 20, 21, 22, 23} }
```

値の並びから 3×4 を読み取ることはできますが、行の境目を目で数える作業になります。要素数が増えれば非現実的です。

## 16.2 Rank と $i

多次元として見せるには、`ArrayItems` に3つの要素を足します。

```xml
<Type Name="vizkit::matrix&lt;*,*,*&gt;">
  <DisplayString>{{ {$T2}x{$T3} }}</DisplayString>
  <Expand>
    <ArrayItems>
      <Direction>Forward</Direction>
      <Rank>2</Rank>
      <Size>$i == 0 ? $T2 : $T3</Size>
      <ValuePointer>_data</ValuePointer>
    </ArrayItems>
  </Expand>
</Type>
```

```
▷ grid              { 3x4 }                     vizkit::matrix<int,3,4>
   ▷ [0]            {0, 1, 2, 3}
        [0]         0
        [1]         1
        [2]         2
        [3]         3
   ▷ [1]            {10, 11, 12, 13}
   ▷ [2]            {20, 21, 22, 23}
   ▷ [Raw View]     {_data=... }
```

行と列の構造が現れました。

新しく出てきた要素を整理します。

| 要素 | 意味 |
|---|---|
| `<Rank>` | **次元数**。`2` なら2次元 |
| `<Size>` | 各次元の長さ。**`$i` に次元のインデックスが入る** |
| `<Direction>` | `Forward`（行優先）または `Backward`（列優先） |

### `$i` は「次元の番号」

ここが `Rank` を使うときの要です。`<Size>` の中の `$i` には、**次元のインデックス**（0, 1, 2, …）が入ります。要素のインデックスではありません。

```
$i == 0  →  0 次元目（行）の長さを返せ  →  $T2（= 3）
$i == 1  →  1 次元目（列）の長さを返せ  →  $T3（= 4）
```

デバッガは `Rank` の回数だけ `Size` を評価し、各次元の長さを集めてから、配列全体の形を決めます。

3次元以上でも同じです。

```xml
<Rank>3</Rank>
<Size>$i == 0 ? $T2 : ($i == 1 ? $T3 : $T4)</Size>
```

三項演算子が入れ子になって読みにくいので、`Intrinsic` に逃がすのも手です。

```xml
<Intrinsic Name="dim" Expression="$i == 0 ? $T2 : ($i == 1 ? $T3 : $T4)" />
```

ただし `$i` は `Size` の文脈でのみ意味を持つ暗黙パラメータなので、`Intrinsic` に切り出すと環境によって挙動が変わることがあります。**素直に `Size` の中に書くのが確実**です。

### すべての次元が同じ長さなら

正方行列のような場合は、条件分岐すら要りません。

```xml
<Rank>2</Rank>
<Size>$T2</Size>
```

`$i` を使わなければ、全次元が同じ長さとして扱われます。

## 16.3 Direction ―― 行優先と列優先

`<Direction>` は、1次元の連続領域をどう畳むかを指定します。

| 値 | 意味 | `_data[i]` の対応 |
|---|---|---|
| `Forward` | **行優先**（row-major）。C/C++ の多次元配列と同じ | `_data[row * Cols + col]` |
| `Backward` | **列優先**（column-major）。Fortran、OpenGL、多くの数学ライブラリ | `_data[col * Rows + row]` |

`vizkit::matrix` は行優先なので `Forward` です。省略した場合の既定も `Forward` ですが、**明示的に書いておくことを勧めます**。行列は列優先の実装も同じくらい多く、後から読む人にとって「どちらの約束か」が Natvis に書いてあること自体に価値があります。

列優先の実装に切り替えた場合は、`Direction` を `Backward` に変え、`Size` の条件も入れ替えます。

```cpp
// 列優先版: _data[col * Rows + row]
```

```xml
<Direction>Backward</Direction>
<Rank>2</Rank>
<Size>$i == 0 ? $T2 : $T3</Size>
```

`Direction` を間違えると、**エラーにならずに転置された表示になります**。3×4 の行列が 3×4 のまま、中身だけ入れ替わって見える。気づきにくい種類のバグなので、値を目で確認する習慣をつけてください。演習3で実際に間違えてもらいます。

## 16.4 行を1つの型として見る ―― 別解

`Rank` による表示は、`[0]` `[1]` `[2]` という行の階層を作ります。ただし各行の値の列は `{0, 1, 2, 3}` という素朴な表示で、要素型の Natvis は行レベルには効きません。

「各行を `fixed_array<T, Cols>` として見せる」という別解があります。前章までに `fixed_array` の Natvis を書いてあるので、それを再利用できます。

```xml
<Type Name="vizkit::matrix&lt;*,*,*&gt;">
  <DisplayString>{{ {$T2}x{$T3} }}</DisplayString>
  <Expand>
    <IndexListItems Optional="true">
      <Size>$T2</Size>
      <ValueNode>*(vizkit::fixed_array&lt;$T1,$T3&gt;*)(_data + $i * $T3)</ValueNode>
    </IndexListItems>
  </Expand>
</Type>
```

行の先頭アドレスを `fixed_array<T, Cols>*` にキャストしています。レイアウトが同一（どちらも `T` が `Cols` 個連続しているだけ）なので、これは成立します。

### ただし重大な制約がある

この書き方には落とし穴があります。**`vizkit::fixed_array<int,4>` という型が、デバッグ情報に存在しなければキャストが失敗します。**

テンプレートは実際に使われて初めて実体化されます。プログラムのどこかで `fixed_array<int,4>` を使っていなければ、その型は PDB に存在せず、Natvis からは「知らない型」です。

```
▷ grid       { 3x4 }
   ▷ [Raw View]     ...           ← 子行が出ない
```

`Optional="true"` を付けているのは、この失敗を黙って飛ばすためです（第8章）。

**Natvis がキャストできるのは、デバッグ情報に存在する型だけ。** これは第4部以降で何度か問題になります。特に「ノードの型」や「内部実装の型」にキャストする場面で、テンプレートが実体化されていないと同じことが起きます。

したがってこの別解は、「その型を確実に使っている」と分かっている場合の選択肢と考えてください。汎用性では `Rank` に劣ります。

## 16.5 転置ビューを作る

第13章のビューを使うと、同じデータを行優先・列優先の両方で見られます。

```xml
<Type Name="vizkit::matrix&lt;*,*,*&gt;">
  <DisplayString>{{ {$T2}x{$T3} }}</DisplayString>
  <Expand>
    <Item Name="[rows]" ExcludeView="simple">$T2</Item>
    <Item Name="[cols]" ExcludeView="simple">$T3</Item>

    <ArrayItems ExcludeView="transposed">
      <Direction>Forward</Direction>
      <Rank>2</Rank>
      <Size>$i == 0 ? $T2 : $T3</Size>
      <ValuePointer>_data</ValuePointer>
    </ArrayItems>

    <ArrayItems IncludeView="transposed">
      <Direction>Backward</Direction>
      <Rank>2</Rank>
      <Size>$i == 0 ? $T3 : $T2</Size>
      <ValuePointer>_data</ValuePointer>
    </ArrayItems>
  </Expand>
</Type>
```

`grid,view(transposed)` で 4×3 として見えます。行列演算のバグを追っているとき、「転置し忘れではないか」を確かめるのに使えます。

`ArrayItems` に `ExcludeView` / `IncludeView` が付けられることも、ここで確認しておいてください。第13章で扱った属性は、`Expand` の中のあらゆる要素に付きます。

## 16.6 現時点の matrix の Natvis

```xml
  <Type Name="vizkit::matrix&lt;*,*,*&gt;">
    <DisplayString>{{ {$T2}x{$T3} }}</DisplayString>
    <Expand>
      <Item Name="[rows]" ExcludeView="simple">$T2</Item>
      <Item Name="[cols]" ExcludeView="simple">$T3</Item>

      <ArrayItems ExcludeView="transposed">
        <Direction>Forward</Direction>
        <Rank>2</Rank>
        <Size>$i == 0 ? $T2 : $T3</Size>
        <ValuePointer>_data</ValuePointer>
      </ArrayItems>

      <ArrayItems IncludeView="transposed">
        <Direction>Backward</Direction>
        <Rank>2</Rank>
        <Size>$i == 0 ? $T3 : $T2</Size>
        <ValuePointer>_data</ValuePointer>
      </ArrayItems>
    </Expand>
  </Type>
```

## 16.7 この章のまとめ

- 多次元は `<Rank>` `<Size>` `<Direction>` の3点セットで表現する。
- **`<Size>` の中の `$i` は「次元のインデックス」**。要素のインデックスではない。`Rank` の回数だけ評価される。
- 全次元が同じ長さなら `$i` を使わず `<Size>$T2</Size>` でよい。
- `<Direction>` は `Forward`（行優先）/ `Backward`（列優先）。**既定は `Forward` だが明示する**。実装の約束を Natvis に残す価値がある。
- **`Direction` を間違えてもエラーにならない。** 転置された表示になるだけなので、値を目で確認する。
- 行を別の型としてキャストする別解もあるが、**その型がデバッグ情報に存在しなければ失敗する**。`Optional="true"` で保護する。
- `IncludeView` / `ExcludeView` は `ArrayItems` にも付けられる。転置ビューのような切り替えが作れる。

## 16.8 演習

1. `<Rank>2</Rank>` を `<Rank>1</Rank>` に変えて、表示がどうなるか確認してください。`Size` の `$i` が 0 のときしか評価されなくなります。
2. `<Size>$i == 0 ? $T3 : $T2</Size>`（行と列を逆に）と書いて `.natvisreload` し、どんな表示になるか観察してください。範囲外アクセスが起きうることも確認してください。
3. `<Direction>Backward</Direction>` に変えて、`grid` の値がどう並び替わるか確認してください。`(0,1)` の位置に何が来るかを目で追ってください。
4. `main.cpp` に `vizkit::fixed_array<int, 4> dummy{};` を追加してから、16.4節の `IndexListItems` 版を試してください。デバッグ情報に型が現れたことで動くようになるはずです。これが「実体化されていない型にはキャストできない」ことの証明になります。
5. `matrix<vizkit::rgba, 2, 2>` を作り、要素の `#RRGGBB` 表示が2次元展開の中でも効くことを確認してください。
