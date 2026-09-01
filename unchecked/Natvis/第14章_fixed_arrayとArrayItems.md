# 第14章 fixed_array<T, N> と ArrayItems

第4部に入ります。ここからは**要素数が実行時に決まる型**、つまりコンテナを扱います。

とはいえ最初の一歩は固定長です。`<ArrayItems>` の書き方と、テンプレートクラスへのマッチのさせ方を、同時に1つずつ覚えます。

## 14.1 型を作る

`vizkit\include\vizkit\seq\fixed_array.hpp`

```cpp
#pragma once

#include <cstddef>

namespace vizkit {

// 要素数がコンパイル時に決まる配列。std::array の最小版。
template <class T, std::size_t N>
struct fixed_array {
    T _data[N];

    [[nodiscard]] constexpr std::size_t size() const noexcept { return N; }
    [[nodiscard]] constexpr T&       operator[](std::size_t i)       noexcept { return _data[i]; }
    [[nodiscard]] constexpr const T& operator[](std::size_t i) const noexcept { return _data[i]; }
};

} // namespace vizkit
```

`main.cpp`：

```cpp
#include <vizkit/seq/fixed_array.hpp>
// ...
    vizkit::fixed_array<int, 5> nums{ { 10, 20, 30, 40, 50 } };

    vizkit::fixed_array<vizkit::point2, 3> triangle{ {
        vizkit::point2{ 0.0, 0.0 },
        vizkit::point2{ 1.0, 0.0 },
        vizkit::point2{ 0.5, 1.0 },
    } };
```

素の表示はこうなります。

```
▷ nums          {_data=0x00000078abcff5a0 {10, 20, 30, 40, 50} }        vizkit::fixed_array<int,5>
   ▷ _data      0x00000078abcff5a0 {10, 20, 30, 40, 50}
        [0]     10
        [1]     20
        ...
```

実は、**素のままでもそれほど悪くありません**。`_data` が生の配列（`int[5]`）であり、要素数が型情報に含まれているためです（第3章 3.5節）。

しかし `_data` という余計な階層が1段挟まっています。`fixed_array` は概念的には配列そのものであって、「配列を持つ何か」ではありません。それに、この後の章で扱う型はどれも要素数が型情報に含まれないので、ここで基本形を身につけておきます。

## 14.2 テンプレートにマッチさせる

`Type` の `Name` 属性では、**アスタリスク `*` がワイルドカード**として使えます。

```xml
<Type Name="vizkit::fixed_array&lt;*,*&gt;">
```

`<` と `>` は XML なのでエスケープが必要です。`&lt;` と `&gt;` になります。ここが Natvis で最も打ち間違えやすい箇所です。

テンプレート引数が2つなので `*` を2つ書きます。`fixed_array<*>` では**マッチしません**。引数の個数は一致している必要があります（可変長テンプレートの扱いなど、詳細は第29章）。

### 型名を確認する確実な方法

ウォッチウィンドウの「型」列に出ている文字列が、そのまま正解です。

```
nums     vizkit::fixed_array<int,5>
```

ここから具体的な引数を `*` に置き換えれば、`vizkit::fixed_array<*,*>` が得られます。空白の有無も含めて、型列の表記に合わせるのが安全です。

## 14.3 ArrayItems ―― 3行で書ける

```xml
<Type Name="vizkit::fixed_array&lt;*,*&gt;">
  <DisplayString>{{ size={$T2} }}</DisplayString>
  <Expand>
    <ArrayItems>
      <Size>$T2</Size>
      <ValuePointer>_data</ValuePointer>
    </ArrayItems>
  </Expand>
</Type>
```

```
▷ nums          { size=5 }              vizkit::fixed_array<int,5>
     [0]        10
     [1]        20
     [2]        30
     [3]        40
     [4]        50
   ▷ [Raw View] {_data=0x... {10, 20, ...} }
```

`_data` の階層が消え、要素が直接並びました。

`<ArrayItems>` の必須要素は2つだけです。

| 要素 | 意味 | 制約 |
|---|---|---|
| `<Size>` | 要素数 | **整数**に評価される式 |
| `<ValuePointer>` | 先頭要素へのポインタ | **要素型のポインタ**。`void*` は不可 |

「ポインタ ＋ 要素数」という組み合わせは、第3章 3.5節で見た「デバッガが理解できない約束」そのものです。`ArrayItems` は、その約束を明文化する要素だと考えてください。第6章の書式指定子 `,[n]` を、型ごとに自動化したものとも言えます。

`ValuePointer` が `void*` を拒否するのは、要素のサイズと型が分からなければ配列として解釈できないからです。生バイト列を配列として見せたい場合は、キャストして型を与える必要があります（第18章）。

## 14.4 $T1 / $T2 ―― テンプレート引数を参照する

`$T1`、`$T2`、… は、`Type` の `Name` に書いたワイルドカードに対応するテンプレート引数を参照するマクロです。

```
vizkit::fixed_array<int, 5>
                    ↑    ↑
                   $T1  $T2
```

**型引数と非型引数で、使い方が違います。**

| | 中身 | 使い方 |
|---|---|---|
| `$T1` = `int` | **型** | キャストや `sizeof` に使う。`($T1*)ptr`、`sizeof($T1)` |
| `$T2` = `5` | **値** | 式の中でそのまま使う。`<Size>$T2</Size>` |

`<Size>$T2</Size>` と書けているのは、`N` が非型テンプレート引数（値）だからです。もし `$T1` を `Size` に書けば、型を整数として評価しようとして失敗します。

第16章では `($T1*)` のキャストを、第18章では `sizeof($T1)` を使います。ここでは「型と値の区別がある」ことだけ押さえてください。

## 14.5 Size を書く3つの方法

`fixed_array` の要素数は、実は3通りに書けます。

```xml
<!-- (a) テンプレート引数から -->
<Size>$T2</Size>

<!-- (b) 配列のサイズから計算 -->
<Size>sizeof(_data) / sizeof(_data[0])</Size>

<!-- (c) メンバーがあればそこから（この型にはない） -->
<Size>_count</Size>
```

どれを選ぶべきでしょうか。

**(a) が第一候補**です。意図が最も明確で、実装を変えても壊れにくい。

**(b) は保険として優秀**です。テンプレート引数の順序を変えたり、引数を1つ増やしたりしても壊れません。ただし `_data` が配列である（ポインタではない）ことに依存します。

実務では、両方を `Optional` で並べる手もあります（第8章 8.4節の応用）。

```xml
<Intrinsic Optional="true" Name="count" Expression="$T2" />
<Intrinsic Optional="true" Name="count" Expression="sizeof(_data) / sizeof(_data[0])" />
```

ただし `fixed_array` のように単純な型で、しかも自分のライブラリなら、`$T2` 一本で十分です。**保険は、壊れる可能性が現実にある場所にだけ掛けます**（第8章 8.5節）。

## 14.6 要素にも Natvis が効く

`fixed_array<point2, 3>` を開いてみてください。

```
▷ triangle      { size=3 }                    vizkit::fixed_array<vizkit::point2,3>
   ▷ [0]        (0, 0)
   ▷ [1]        (1, 0)
   ▷ [2]        (0.5, 1)
   ▷ [Raw View] {_data=... }
```

要素の `point2` には、第4章で書いた Natvis がそのまま適用されています。**コンテナの Natvis と要素の Natvis は独立していて、自動的に合成されます。**

これが Natvis の投資対効果を決定的にしている性質です。要素型の Natvis を1つ書けば、その型を格納するあらゆるコンテナの表示が改善されます。逆に言えば、**基本的な小さい型の Natvis を先に書くべき**ということでもあります。本書が第2部を `point2` や `rgba` から始めたのは、この順序に意味があるからです。

## 14.7 [size] を出すか

`STL.natvis` は `std::array` に `[size]` の `Item` を持たせています。`fixed_array` でも同様にできます。

```xml
<Expand>
  <Item Name="[size]" ExcludeView="simple">$T2</Item>
  <ArrayItems>
    <Size>$T2</Size>
    <ValuePointer>_data</ValuePointer>
  </ArrayItems>
</Expand>
```

ただし `fixed_array` の場合、要素数は**型名に書いてあります**（`fixed_array<int,5>`）。冗長です。

判断基準はこうです。**要素数が型から読み取れるなら `[size]` は不要、実行時にしか分からないなら必要。** 次章の `dynamic_array` では必須になります。

ここでは `ExcludeView="simple"` を付けたうえで残しておきます（第13章）。既定では出るが、大量に並べるときは消える、という形です。

## 14.8 大きな N と表示コスト

`fixed_array<double, 100000>` に対しても、この Natvis は動きます。しかしウォッチウィンドウを開いた瞬間にデバッガが数秒固まることがあります。

デバッガは、展開された行に対して**実際にメモリを読みに行きます**。10万要素なら10万回分の読み取りが発生しうるということです。

対策は第43章でまとめて扱いますが、いま知っておくべき点を2つ挙げます。

1. **デバッガは画面に見えている範囲だけを評価する**（仮想化されている）。ので、10万要素でも開いた直後は先頭数十件しか読まない。スクロールすると読み進む。
2. **`Size` の式自体が重いと、行数分だけ重くなる**。`Size` には単純な式を書くこと。

`ArrayItems` は、この点で最も高速な列挙方法です。デバッガが「先頭ポインタ ＋ 要素サイズ × i」という単純な計算で各要素のアドレスを求められるためです。第17章で扱う `IndexListItems` や第23章の `CustomListItems` は、要素ごとに式を評価するので、これより遅くなります。

**使えるときは `ArrayItems` を使う。** これが第4部を通じての原則です。

## 14.9 現時点の vizkit.natvis（追加分）

```xml
  <!-- ===== seq ===== -->

  <Type Name="vizkit::fixed_array&lt;*,*&gt;">
    <DisplayString>{{ size={$T2} }}</DisplayString>
    <Expand>
      <Item Name="[size]" ExcludeView="simple">$T2</Item>
      <ArrayItems>
        <Size>$T2</Size>
        <ValuePointer>_data</ValuePointer>
      </ArrayItems>
    </Expand>
  </Type>
```

## 14.10 この章のまとめ

- `Type` の `Name` では `*` がワイルドカード。`&lt;` `&gt;` のエスケープを忘れない。**引数の個数は一致している必要がある**。
- 型名はウォッチウィンドウの「型」列からコピーするのが確実。
- `<ArrayItems>` の必須要素は `<Size>`（整数式）と `<ValuePointer>`（要素型のポインタ、`void*` 不可）の2つだけ。
- **`$T1` は型、`$T2` は値。** 型はキャストと `sizeof` に、値は式にそのまま使う。
- 要素数の書き方は複数ある。第一候補は `$T2`、保険は `sizeof(_data)/sizeof(_data[0])`。
- **コンテナの Natvis と要素の Natvis は自動的に合成される。** 小さい型の Natvis を先に書くほど効率がよい。
- 要素数が型名から読めるなら `[size]` は冗長。
- `ArrayItems` は最も高速な列挙方法。**使えるときは必ずこれを使う。**

## 14.11 演習

1. `Type Name` を `vizkit::fixed_array&lt;*&gt;`（アスタリスク1つ）に変え、マッチしなくなることを確認してください。第4章 4.5節の「沈黙」の再現です。
2. `<Size>$T1</Size>` と書いて `.natvisreload` し、どんなエラーになるか確認してください。型と値の区別を体感できます。
3. `<ValuePointer>&amp;_data[0]</ValuePointer>` と書き換えて、同じ結果になることを確認してください。配列名とその先頭要素のアドレスが同じであることの確認です。
4. `fixed_array<vizkit::rgba, 4>` を `main.cpp` に追加し、第6章で書いた `#RRGGBB` 表示が要素にも効くことを確認してください。
5. `<Size>$T2 + 3</Size>` と書いて、配列の範囲外まで表示させてみてください。デバッガが何を表示するか観察したうえで、元に戻してください。Natvis は範囲チェックをしてくれない、ということの確認です。
