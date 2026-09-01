# 第7章 Condition で表示を出し分ける

同じ型でも、状態によって「見たいもの」は変わります。空のコンテナに要素を並べても意味がないし、null ポインタを持つビューの中身を表示しようとすれば評価エラーになります。

`Condition` 属性は、その出し分けを担当します。第4章で `rect` に先取りで使いましたが、ここで正式に扱います。

## 7.1 題材：ポインタと長さを持つ型

`Condition` の必要性が最もはっきり出るのは、**ポインタを持つ型**です。ポインタが null のときと、そうでないときで、書ける式が変わるからです。

`vizkit\include\vizkit\core\byte_span.hpp`

```cpp
#pragma once

#include <cstddef>

namespace vizkit {

// 生バイト列への非所有ビュー。
// ポインタと長さの組は「Natvis なしでは読めない型」の最小形。
struct byte_span {
    const std::byte* data = nullptr;
    std::size_t      size = 0;
};

} // namespace vizkit
```

`main.cpp` に3つの状態を用意します。

```cpp
#include <vizkit/core/byte_span.hpp>
// ...
    const std::byte payload[6] = {
        std::byte{0xDE}, std::byte{0xAD}, std::byte{0xBE},
        std::byte{0xEF}, std::byte{0x00}, std::byte{0x2A},
    };

    vizkit::byte_span whole{ payload, 6 };
    vizkit::byte_span empty_span{ payload, 0 };   // 有効なポインタ、長さ 0
    vizkit::byte_span null_span{};                // null ポインタ
```

素の表示：

```
▷ whole       {data=0x00000078abcff610 size=6 }     vizkit::byte_span
▷ empty_span  {data=0x00000078abcff610 size=0 }     vizkit::byte_span
▷ null_span   {data=0x0000000000000000 size=0 }     vizkit::byte_span
```

## 7.2 Condition の基本

`Condition` 属性に**真偽値を返す式**を書きます。式が `false` になったノードは、存在しないものとして扱われます。

```xml
<Type Name="vizkit::byte_span">
  <DisplayString Condition="data == nullptr">[null]</DisplayString>
  <DisplayString Condition="size == 0">[empty]</DisplayString>
  <DisplayString>{size} bytes @ {data}</DisplayString>
</Type>
```

```
  whole       6 bytes @ 0x00000078abcff610      vizkit::byte_span
  empty_span  [empty]                           vizkit::byte_span
  null_span   [null]                            vizkit::byte_span
```

3つの状態が一目で区別できるようになりました。

## 7.3 評価順序 ―― 上から、最初に成立したもの

これが `Condition` の中核的なルールです。

**デバッガは `DisplayString` を書かれた順に上から評価し、`Condition` が最初に `true` になったものを採用します。** 以降のものは評価されません。

したがって次の2点が設計上の約束になります。

**(1) 具体的な条件を上に、一般的な条件を下に置く。**

先ほどの例で `size == 0` を `data == nullptr` より上に書くと、`null_span` は `size` も 0 なので `[empty]` と表示されてしまい、null であることが見えなくなります。**より限定的な条件を先に書く**のが原則です。

**(2) 条件なしのものを必ず最後に1つ置く。**

条件なしの `DisplayString` は常に成立するので、フォールバックとして機能します。これがないと、どの条件にも当てはまらない状態のときに表示が素の形に戻り、「Natvis が効いていないのか、条件が全部外れたのか」が分からなくなります。

```xml
<!-- 悪い例：フォールバックがない -->
<DisplayString Condition="size &gt; 0">{size} bytes</DisplayString>
<!-- size == 0 のとき、素の {data=... size=0 } に戻ってしまう -->
```

## 7.4 Condition は式の評価を守る盾でもある

`Condition` の役割は「見た目を変える」だけではありません。**危険な式を評価させないためのガード**でもあります。

```xml
<!-- 危険：data が null でも先頭バイトを読もうとする -->
<DisplayString>first={data[0]}</DisplayString>
```

`null_span` に対してこれを評価すると、デバッガは null 参照を試みてエラーになります。

```
  null_span   <エラー: アクセス違反>       vizkit::byte_span
```

先に null を捕まえておけば、この式は null でない場合にしか評価されません。

```xml
<DisplayString Condition="data == nullptr">[null]</DisplayString>
<DisplayString Condition="size == 0">[empty]</DisplayString>
<DisplayString>first={data[0],Xb} ({size} bytes)</DisplayString>
```

この「**危険な式の前にガード条件を置く**」パターンは、以降のコンテナの章で繰り返し使います。特に `LinkedListItems`（第19章）や `CustomListItems`（第23章）でポインタを辿るときは、これを怠るとデバッガが不正なメモリを延々と読み続けることになります。

## 7.5 短絡評価は効くが、頼りすぎない

Natvis の式でも `&&` と `||` は短絡評価されます。したがって1つの `Condition` にまとめることは可能です。

```xml
<DisplayString Condition="data != nullptr &amp;&amp; size &gt; 0">{size} bytes</DisplayString>
```

書けますが、実務ではおすすめしません。理由は2つあります。

- `&amp;&amp;` という表記が読みにくい
- 条件を1つにまとめると「どの状態なのか」が表示から分からなくなる

`[null]` と `[empty]` を別々に出したほうが、デバッグ時の情報量が多くなります。**Natvis の目的は表示の簡潔さではなく、状態の判別しやすさです。**

## 7.6 Condition の中でも関数は呼べない

当然ですが、`Condition` も式なので第5章の制約がそのまま適用されます。

```xml
<!-- 動かない -->
<DisplayString Condition="empty()">[empty]</DisplayString>
```

`Intrinsic` で定義した名前なら使えます。

```xml
<Type Name="vizkit::byte_span">
  <Intrinsic Name="is_null"  Expression="data == nullptr" />
  <Intrinsic Name="is_empty" Expression="size == 0" />
  <DisplayString Condition="is_null()">[null]</DisplayString>
  <DisplayString Condition="is_empty()">[empty]</DisplayString>
  <DisplayString>{size} bytes, first={data[0],Xb}</DisplayString>
</Type>
```

条件が複雑になるほど、`Intrinsic` で名前を付ける価値が上がります。

## 7.7 rgba と rect を整理する

第4章と第6章で場当たり的に書いた条件を、この章のルールに沿って整理します。

```xml
<Type Name="vizkit::rgba">
  <DisplayString Condition="a == 0">[transparent]</DisplayString>
  <DisplayString Condition="a == 255">#{r,Xb}{g,Xb}{b,Xb}</DisplayString>
  <DisplayString>#{r,Xb}{g,Xb}{b,Xb} (a={a,d})</DisplayString>
</Type>
```

完全透明を最上位に置きました。`transparent`（すべて 0）は `a == 0` で捕まり、`[transparent]` になります。

```xml
<Type Name="vizkit::rect">
  <Intrinsic Name="w" Expression="_max.x - _min.x" />
  <Intrinsic Name="h" Expression="_max.y - _min.y" />
  <DisplayString Condition="w() &lt;= 0 || h() &lt;= 0">[empty]</DisplayString>
  <DisplayString>[{_min}, {_max}] {w(),g}x{h(),g}</DisplayString>
</Type>
```

`Intrinsic` を入れたことで、条件と表示の両方から同じ式を参照できるようになりました。幅の定義を変えたくなったときに直す場所が1か所で済みます。

## 7.8 Condition が使える場所

`Condition` は `DisplayString` 専用ではありません。**多くの視覚化要素に付けられます**。

| 要素 | `Condition` の意味 |
|---|---|
| `DisplayString` | この表示を採用するか（本章） |
| `StringView` | テキストビジュアライザに渡すか（第9章） |
| `Item` | この子行を出すか（第10章） |
| `ArrayItems` / `IndexListItems` など | この列挙方法を使うか（第4部） |
| `ExpandedItem` | この平坦化を行うか（第12章） |
| `Type` の内側の各ノード全般 | ― |

第18章の `small_vector` では、「インライン格納なら `ArrayItems` を A、ヒープ格納なら B」という分岐を `Condition` 付きの `ArrayItems` を2つ並べて実現します。この章のルールがそのまま効きます。

## 7.9 現時点の vizkit.natvis

```xml
<?xml version="1.0" encoding="utf-8"?>
<AutoVisualizer xmlns="http://schemas.microsoft.com/vstudio/debugger/natvis/2010">

  <Type Name="vizkit::point2">
    <DisplayString>({x,g}, {y,g})</DisplayString>
  </Type>

  <Type Name="vizkit::rgba">
    <DisplayString Condition="a == 0">[transparent]</DisplayString>
    <DisplayString Condition="a == 255">#{r,Xb}{g,Xb}{b,Xb}</DisplayString>
    <DisplayString>#{r,Xb}{g,Xb}{b,Xb} (a={a,d})</DisplayString>
  </Type>

  <Type Name="vizkit::rect">
    <Intrinsic Name="w" Expression="_max.x - _min.x" />
    <Intrinsic Name="h" Expression="_max.y - _min.y" />
    <DisplayString Condition="w() &lt;= 0 || h() &lt;= 0">[empty]</DisplayString>
    <DisplayString>[{_min}, {_max}] {w(),g}x{h(),g}</DisplayString>
  </Type>

  <Type Name="vizkit::version">
    <Intrinsic Name="major" Expression="packed >> 22" />
    <Intrinsic Name="minor" Expression="(packed >> 12) &amp; 0x3FF" />
    <Intrinsic Name="patch" Expression="packed &amp; 0xFFF" />
    <DisplayString Condition="packed == 0">[unset]</DisplayString>
    <DisplayString>v{major()}.{minor()}.{patch()} ({packed,X})</DisplayString>
  </Type>

  <Type Name="vizkit::byte_span">
    <Intrinsic Name="is_null"  Expression="data == nullptr" />
    <Intrinsic Name="is_empty" Expression="size == 0" />
    <DisplayString Condition="is_null()">[null]</DisplayString>
    <DisplayString Condition="is_empty()">[empty]</DisplayString>
    <DisplayString>{size} bytes, first={data[0],Xb}</DisplayString>
  </Type>

</AutoVisualizer>
```

## 7.10 この章のまとめ

- `Condition` は真偽値の式。`false` になったノードは存在しないものとして扱われる。
- **上から順に評価し、最初に `true` になったものを採用する。** 具体的な条件ほど上に置く。
- **条件なしのフォールバックを必ず最後に置く。** これがないと素の表示に戻り、原因の切り分けができなくなる。
- `Condition` は見た目のためだけでなく、**危険な式を評価させないガード**でもある。null チェックは必ず先に置く。
- 状態は分けて表示する。`[null]` と `[empty]` を区別することに価値がある。
- `Condition` は `DisplayString` 以外の多くの要素にも付けられる。

## 7.11 演習

1. `byte_span` の `DisplayString` の順序を入れ替え、`size == 0` を `data == nullptr` より上に置いてください。`null_span` の表示がどう変わるか確認し、なぜそうなるか説明してください。
2. フォールバックの `DisplayString`（条件なしのもの）を削除し、`whole` 以外がどう表示されるか確認してください。
3. `byte_span` に `Condition="size &gt; 1024"` の `DisplayString` を足し、`[too large: {size} bytes]` と表示させてください。異常値をデバッグ時に目立たせる実務パターンです。
4. `rect` の `Intrinsic` を消して、条件と表示の両方に `_max.x - _min.x` を直接書いてください。式が2か所に重複することの保守性を比べてみてください。
