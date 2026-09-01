# 第12章 ExpandedItem ―― 階層を平坦化する

`Synthetic` が「階層を作る」要素だとすれば、`<ExpandedItem>` はその逆、**階層を潰す**要素です。

ラッパー型や継承を使うと、意味的には1つのものが、デバッガ上では2段3段の階層になってしまいます。`ExpandedItem` は、それを親のレベルに引き上げます。

## 12.1 題材：ラッパーと継承

2つのパターンを用意します。

### ラッパー型

`vizkit\include\vizkit\core\named_rect.hpp`

```cpp
#pragma once

#include <vizkit/core/name32.hpp>
#include <vizkit/core/rect.hpp>

namespace vizkit {

// 「rect に名前を付けただけ」の型。
// 意味的には rect そのものだが、デバッガ上では 1 段深くなる。
struct named_rect {
    name32 _name{};
    rect   _bounds{};
};

} // namespace vizkit
```

### 継承

`vizkit\include\vizkit\core\shape.hpp`

```cpp
#pragma once

#include <vizkit/core/point2.hpp>
#include <vizkit/core/rgba.hpp>
#include <vizkit/core/version.hpp>

namespace vizkit {

struct shape_base {
    rgba    color{};
    version created_with{};
    int     layer = 0;
};

struct circle : shape_base {
    point2 center{};
    double radius = 0.0;
};

} // namespace vizkit
```

`main.cpp`：

```cpp
#include <vizkit/core/named_rect.hpp>
#include <vizkit/core/shape.hpp>
// ...
    vizkit::named_rect viewport;
    viewport._name.assign("viewport");
    viewport._bounds = vizkit::rect{ {0.0, 0.0}, {1920.0, 1080.0} };

    vizkit::circle c;
    c.color = vizkit::from_argb(0xFF3366CCu);
    c.created_with = vizkit::make_version(3, 12, 7);
    c.layer = 2;
    c.center = vizkit::point2{ 100.0, 50.0 };
    c.radius = 12.5;
```

## 12.2 何が問題なのか

Natvis を書かずに `circle` を開くと、こうなります。

```
▷ c                        {color=#CC6633 created_with=v3.12.7 ... }   vizkit::circle
   ▷ vizkit::shape_base    {color=#CC6633 created_with=v3.12.7 layer=2 }
        ▷ color            #CC6633
        ▷ created_with     v3.12.7
          layer            2
   ▷ center                (100, 50)
     radius                12.500000000000000
```

基底クラスの部分が `vizkit::shape_base` という1行に押し込められ、`color` を見るには**2回クリック**する必要があります。多重継承や CRTP を使うと、これが3段4段になります。

`named_rect` も同様です。

```
▷ viewport                 {_name="viewport" _bounds=[(0, 0), (1920, 1080)] ... }
   ▷ _name                 "viewport"
   ▷ _bounds               [(0, 0), (1920, 1080)] 1920x1080
        [width]            1920
        [height]           1080
```

幅を知るのに2クリックです。`named_rect` は概念的には「名前付きの矩形」であって、「矩形を内部に持つ何か」ではありません。表示もそうあるべきです。

## 12.3 ExpandedItem の基本

```xml
<ExpandedItem>式</ExpandedItem>
```

式が評価され、**その結果の子行が、この位置に直接展開されます**。1行を消費しません。

`named_rect` に適用します。

```xml
<Type Name="vizkit::named_rect">
  <DisplayString>{_name} {_bounds}</DisplayString>
  <Expand>
    <Item Name="[name]">_name</Item>
    <ExpandedItem>_bounds</ExpandedItem>
  </Expand>
</Type>
```

```
▷ viewport                 "viewport" [(0, 0), (1920, 1080)] 1920x1080
     [name]                "viewport"
     [width]               1920
     [height]              1080
   ▷ min                   (0, 0)
   ▷ max                   (1920, 1080)
   ▷ [geometry]            1920 x 1080, area=2073600
   ▷ [Raw View]            {_name=... _bounds=... }
```

`_bounds` という行が消え、その中身（第11章で `rect` に定義した `Expand` の内容）が `named_rect` の直下に並びました。**`rect` の Natvis を書き直す必要はありません。** `ExpandedItem` は、対象の型に定義された `Expand` をそのまま持ってきます。

## 12.4 基底クラスを引き上げる ―― `,nd` が要る理由

継承のほうが本題です。素直に書くと次のようになりますが、**これは動きません**。

```xml
<!-- 危険 -->
<Type Name="vizkit::circle">
  <Expand>
    <ExpandedItem>*(vizkit::shape_base*)this</ExpandedItem>
    <Item Name="center">center</Item>
    <Item Name="radius">radius</Item>
  </Expand>
</Type>
```

問題は、デバッガが**「最も派生した型」を自動で解決しようとする**ことです。`shape_base*` にキャストしても、デバッガはそのオブジェクトが実際には `circle` であることを知っています。したがって `circle` の Natvis を再び適用し、その中でまた `shape_base*` にキャストし……と無限に潜ろうとします。

これを止めるのが、第6章で紹介した書式指定子 **`nd`**（派生クラス情報を出さず、基底クラスとして表示する）です。

```xml
<Type Name="vizkit::circle">
  <DisplayString>circle r={radius,g} @ {center}</DisplayString>
  <Expand>
    <Item Name="center">center</Item>
    <Item Name="radius">radius</Item>
    <ExpandedItem>*(vizkit::shape_base*)this,nd</ExpandedItem>
  </Expand>
</Type>
```

```
▷ c                        circle r=12.5 @ (100, 50)      vizkit::circle
   ▷ center                (100, 50)
     radius                12.500000000000000
   ▷ color                 #CC6633
   ▷ created_with          v3.12.7
     layer                 2
   ▷ [Raw View]            {shape_base={...} center=... radius=... }
```

基底クラスのメンバーが、派生クラスのメンバーと同じ高さに並びました。

公式ドキュメントの例も、まったく同じ形をしています。

```xml
<Type Name="CPanel">
  <DisplayString>{{Name = {*(m_pstrName)}}}</DisplayString>
  <Expand>
    <Item Name="IsItemsHost">(bool)m_bItemsHost</Item>
    <ExpandedItem>*(CFrameworkElement*)this,nd</ExpandedItem>
  </Expand>
</Type>
```

**`ExpandedItem` で基底クラスを引き上げるときは `,nd` を付ける。** これは定型句として覚えてしまって構いません。

## 12.5 Optional との併用

`ExpandedItem` は `Optional` と相性がよい要素です。基底クラスの構成が条件付きコンパイルやテンプレート引数で変わる場合、キャストが解析できないことがあるからです。

`STL.natvis` にも実例があります。

```xml
<ExpandedItem Optional="true">*((_Mybase::_Mybase::_Mybase::_Mybase::_Mybase *) this)</ExpandedItem>
```

継承の深さが実体によって変わるため、`Optional` で「あれば使う、なければ飛ばす」としています。自作ライブラリでも、ポリシークラスやミックスインを使っていると同じ状況になります（第33章）。

## 12.6 名前の衝突に注意する

`ExpandedItem` を複数並べたり、`Item` と混ぜたりすると、**同じ名前の子行が2つ並ぶ**ことがあります。

```xml
<Expand>
  <ExpandedItem>*(base_a*)this,nd</ExpandedItem>   <!-- id を持つ -->
  <ExpandedItem>*(base_b*)this,nd</ExpandedItem>   <!-- こちらも id を持つ -->
</Expand>
```

```
     id        1        ← base_a の
     id        7        ← base_b の
```

デバッガはエラーにしません。同じ名前の行が2つ並ぶだけです。どちらがどちらか区別できず、かえって混乱します。

多重継承で衝突が起きる場合は、平坦化をあきらめて `Item` で名前を付けるほうが親切です。

```xml
<Expand>
  <Item Name="[base_a]">*(base_a*)this,nd</Item>
  <Item Name="[base_b]">*(base_b*)this,nd</Item>
</Expand>
```

**平坦化は「1つの基底が主役である」ときにだけ効く技法**だと考えてください。第32章で多重継承・仮想継承の扱いを詳しく見ます。

## 12.7 どこまで平坦化するか

`ExpandedItem` を使うかどうかは、**その型が概念的に何であるか**で決めます。

| 型の性格 | 扱い |
|---|---|
| ラッパー（`named_rect`、スマートポインタ、`optional`） | 平坦化する。中身が主役 |
| 「is-a」の継承（`circle` は `shape_base` である） | 平坦化する |
| 「has-a」の合成（`window` が `rect` を持つ） | 平坦化しない。`Item` で見せる |
| 多重継承で衝突がある | 平坦化しない |
| 基底が実装詳細（アロケータ基底など） | 平坦化しない。むしろ隠す |

最後の行は重要です。`STL.natvis` が `[allocator]` という `Item` を作って、アロケータ基底を1行に押し込めているのがこの判断です。**すべてを平坦化すると、実装詳細まで表に出てきて逆に読みにくくなります。**

## 12.8 現時点の vizkit.natvis（第3部の追加分）

```xml
  <Type Name="vizkit::named_rect">
    <DisplayString>{_name} {_bounds}</DisplayString>
    <Expand>
      <Item Name="[name]">_name</Item>
      <ExpandedItem>_bounds</ExpandedItem>
    </Expand>
  </Type>

  <Type Name="vizkit::shape_base">
    <DisplayString>layer={layer} {color}</DisplayString>
    <Expand>
      <Item Name="color">color</Item>
      <Item Name="layer">layer</Item>
      <Item Name="[created with]">created_with</Item>
    </Expand>
  </Type>

  <Type Name="vizkit::circle">
    <DisplayString>circle r={radius,g} @ {center}</DisplayString>
    <Expand>
      <Item Name="center">center</Item>
      <Item Name="radius">radius</Item>
      <ExpandedItem>*(vizkit::shape_base*)this,nd</ExpandedItem>
    </Expand>
  </Type>
```

`shape_base` にも Natvis を書いてある点に注目してください。`circle` の `ExpandedItem` は `shape_base` の `Expand` を持ってくるので、**基底クラス側の定義がそのまま効きます**。基底の見せ方を変えれば、すべての派生型の表示が同時に変わります。

## 12.9 この章のまとめ

- `<ExpandedItem>式</ExpandedItem>` は、式の結果の**子行をこの位置に直接展開する**。1行を消費しない。
- ラッパー型の中身を引き上げる、基底クラスのメンバーを派生と同じ高さに並べる、という2つが主な用途。
- **基底クラスを引き上げるときは `,nd` を必ず付ける。** 付けないと最派生型の解決が繰り返され、無限に潜る。
- 対象の型に定義された `Expand` がそのまま使われるので、基底側の Natvis を書けば派生すべてに効く。
- 構成が変わりうる基底には `Optional="true"` を併用する。
- **名前が衝突する多重継承では平坦化しない。** `Item` で名前を付けて分ける。
- 判断基準は「その型が概念的に何であるか」。ラッパーと is-a は平坦化、has-a と実装詳細は平坦化しない。

## 12.10 演習

1. `circle` の `ExpandedItem` から `,nd` を外して `.natvisreload` してください。どうなるか観察し（環境によってはデバッガが重くなるので、確認したらすぐ戻してください）、なぜそうなるか説明してください。
2. `named_rect` の `ExpandedItem` を `<Item Name="bounds">_bounds</Item>` に変え、どちらの表示が読みやすいか比べてください。
3. `circle` の `Expand` で `ExpandedItem` を先頭に移動し、基底のメンバーが上に来る形にしてください。どちらの順序が実務で便利か考えてみてください。
4. `shape_base` の `Expand` に `[created with]` を残したまま、`circle` 側にも同名の `Item` を追加してください。名前が衝突したときの表示を確認してください。
