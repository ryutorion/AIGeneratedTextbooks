# 第32章 継承と Inheritable

第12章で `ExpandedItem` を使って基底クラスを平坦化しました。この章では継承そのものの扱いを整理します。

軸は2つです。**基底の Natvis が派生にも適用されること**と、**デバッガが「最も派生した型」を解決すること**。この2つが組み合わさると、意図しない表示になることがあります。

## 32.1 題材を広げる

第12章の `shape_base` / `circle` に、多態的な型を足します。

`vizkit\include\vizkit\core\shape.hpp`

```cpp
#pragma once

#include <vizkit/core/point2.hpp>
#include <vizkit/core/rgba.hpp>
#include <vizkit/core/rect.hpp>
#include <vizkit/core/version.hpp>

namespace vizkit {

struct shape_base {
    rgba    color{};
    version created_with{};
    int     layer = 0;

    virtual ~shape_base() = default;
    [[nodiscard]] virtual double area() const noexcept = 0;
};

struct circle : shape_base {
    point2 center{};
    double radius = 0.0;
    [[nodiscard]] double area() const noexcept override;
};

struct box : shape_base {
    rect bounds{};
    [[nodiscard]] double area() const noexcept override;
};

// タグ付けのためのミックスイン（多重継承の題材）
struct selectable {
    bool selected = false;
};

struct handle_box : box, selectable {
    int handle_id = 0;
};

} // namespace vizkit
```

`main.cpp`：

```cpp
#include <vizkit/core/shape.hpp>
#include <vizkit/seq/dynamic_array.hpp>
// ...
    vizkit::circle c;
    c.color = vizkit::from_argb(0xFF3366CCu);
    c.layer = 2;
    c.center = { 100.0, 50.0 };
    c.radius = 12.5;

    vizkit::handle_box hb;
    hb.bounds = vizkit::rect{ {0,0}, {40,20} };
    hb.selected = true;
    hb.handle_id = 7;

    vizkit::dynamic_array<vizkit::shape_base*> scene;
    scene.push_back(&c);
    scene.push_back(&hb);
```

## 32.2 基底の Natvis は派生にも適用される

`shape_base` にだけ Natvis を書いてみます。

```xml
<Type Name="vizkit::shape_base">
  <DisplayString>shape layer={layer} {color}</DisplayString>
  <Expand>
    <Item Name="color">color</Item>
    <Item Name="layer">layer</Item>
  </Expand>
</Type>
```

`circle` には何も書いていません。それでも、こう表示されます。

```
▷ c          shape layer=2 #CC6633        vizkit::circle
     color   #CC6633
     layer   2
```

**基底の Natvis が、派生クラスにも適用されています。**

これは `Inheritable` 属性の既定値が `true` だからです。

> `Inheritable` ―― この型から派生したクラスのオブジェクトがこのビジュアライザーを使ってよければ `true`、オブジェクトが厳密にこの型でなければならないなら `false`。**既定は `true`。**

便利な性質です。基底に1つ書けば、派生型すべてが最低限の表示を持つことになります。

ただし副作用もあります。**派生固有のメンバーが見えなくなります。** `circle` の `radius` も `center` も、`Expand` に書いていないので消えました。`[Raw View]` を開かなければ辿り着けません。

したがって、派生クラスにも `Type` を書き、基底を `ExpandedItem` で引き上げるのが定石でした（第12章 12.4節）。

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

**派生に定義があれば、そちらが優先されます。** 基底の定義は `ExpandedItem` から明示的に呼び出す形になります。

## 32.3 Inheritable="false"

派生に適用させたくない場合に指定します。

```xml
<Type Name="vizkit::shape_base" Inheritable="false">
  <DisplayString>shape base only</DisplayString>
</Type>
```

こうすると、`circle` にはこの定義が適用されません。`circle` 用の `Type` がなければ、素の表示になります。

使いどころは限られますが、次のような場面があります。

**(1) 基底が実装詳細で、派生ごとにまったく違う表示をしたい。**

```cpp
namespace vizkit::detail {
    struct node_base { node_base* next; };   // すべてのノードの共通基底
}
```

`node_base` に「リンクを表示する」Natvis を書くと、それを継承したすべてのノード型が同じ表示になってしまいます。`Inheritable="false"` にすれば、`node_base` そのものにだけ適用されます。

**(2) 基底の Natvis が派生では誤解を招く。**

`shape_base` の `DisplayString` に `area()` 相当の式を書いたとします。`circle` と `box` では計算式が違うので、基底の式は誤った値を出します。**間違った値を出すくらいなら、素の表示のほうがましです。**

**(3) タグ用の空クラス。**

`selectable` のようなミックスインは、単独では意味を持ちません。Natvis を書くとしても、派生に適用させる必要はありません。

## 32.4 最も派生した型の解決

デバッガは、**ポインタや参照が指しているオブジェクトの実際の型（最派生型）を解決**します。仮想関数を持つクラスなら、vtable から判定できるためです。

```cpp
vizkit::shape_base* p = &c;      // 静的な型は shape_base*
```

```
▷ p          0x00000078abcff5c0 circle r=12.5 @ (100, 50)      vizkit::shape_base *
```

**`circle` の Natvis が使われています。** 静的な型は `shape_base*` なのに、実体が `circle` であることをデバッガが知っているからです。

多態的なコンテナでも同様です。

```
▷ scene              { size=2 }                       vizkit::dynamic_array<vizkit::shape_base *>
   ▷ [0]             0x... circle r=12.5 @ (100, 50)
   ▷ [1]             0x... box [(0, 0), (40, 20)]
```

**基底ポインタのコンテナが、それぞれの派生型として表示されます。** これは Natvis の非常に便利な性質で、シーングラフやプラグイン配列のデバッグで実際に助かります。

### 副作用：無限再帰

この解決が、第12章 12.4節で扱った問題を引き起こします。

```xml
<!-- 危険 -->
<ExpandedItem>*(vizkit::shape_base*)this</ExpandedItem>
```

`shape_base*` にキャストしても、デバッガは実体が `circle` だと知っているので `circle` の Natvis を適用します。その中でまた `shape_base*` にキャストし……と無限に潜ります。

`,nd`（最派生型を解決せず、基底として表示する）で止めます。

```xml
<ExpandedItem>*(vizkit::shape_base*)this,nd</ExpandedItem>
```

**`,nd` は「最派生型の解決を止める」指定子**だと理解しておくと、なぜここで必要なのかが腑に落ちます。

### 解決を止めたいその他の場面

デバッグ中に「実際の型ではなく、宣言された型として見たい」ことがあります。

```
p,nd          0x... shape layer=2 #CC6633      ← shape_base として表示
p             0x... circle r=12.5 @ (100, 50)  ← circle として表示
```

型の取り違えを疑っているときに、この2つを並べると切り分けが速くなります。

## 32.5 多重継承

`handle_box` は `box` と `selectable` を継承しています。素直に書くとこうなります。

```xml
<Type Name="vizkit::handle_box">
  <DisplayString>handle #{handle_id} {bounds}</DisplayString>
  <Expand>
    <Item Name="handle_id">handle_id</Item>
    <ExpandedItem>*(vizkit::box*)this,nd</ExpandedItem>
    <ExpandedItem>*(vizkit::selectable*)this,nd</ExpandedItem>
  </Expand>
</Type>
```

```
▷ hb                 handle #7 [(0, 0), (40, 20)] 40x20
     handle_id       7
   ▷ bounds          [(0, 0), (40, 20)] 40x20
     color           #000000
     layer           0
     selected        true
   ▷ [Raw View]      ...
```

動きます。ただし2つの注意点があります。

**(1) 名前の衝突。** 第12章 12.6節で扱ったとおり、両方の基底に同名のメンバーがあると、同じ名前の行が2つ並びます。デバッガはエラーにしません。

衝突する場合は、平坦化をやめて名前を付けます。

```xml
<Item Name="[box]">*(vizkit::box*)this,nd</Item>
<Item Name="[selectable]">*(vizkit::selectable*)this,nd</Item>
```

**(2) キャストのアドレス調整。** 多重継承では、`(vizkit::selectable*)this` は `this` と異なるアドレスになります。デバッガはこの調整を正しく行うので、**C 形式キャストを書いておけば問題ありません**。

ただし、`(char*)this + オフセット` のような手計算をしてはいけません。オフセットはコンパイラの都合で変わります。**必ず型キャストを使ってください。**

## 32.6 仮想継承

仮想継承は、Natvis から見ると最も厄介なケースです。基底のアドレスが実行時のオフセットテーブル経由で決まるため、`(Base*)this` のキャストが期待どおりに動かないことがあります。

対処は3段階です。

**(1) まず素直にキャストを試す。** 多くの場合は動きます。

```xml
<ExpandedItem Optional="true">*(vizkit::shape_base*)this,nd</ExpandedItem>
```

`Optional="true"` を付けておくのが重要です。動かない環境で全体が壊れるのを防げます。

**(2) 動かなければ、`[Raw View]` に見えている経路を使う。** 仮想基底は `[Raw View]` の中に何らかの形で現れます。その名前を直接書きます。

**(3) それでも駄目なら平坦化をあきらめる。** `Item` で1行にまとめれば、少なくとも `[Raw View]` を開くより浅い位置に来ます。

`STL.natvis` には `Inheritable` も仮想継承の対応も出てきません。標準ライブラリが仮想継承を避けているためです。**仮想継承は Natvis を書きにくくする**というのは、設計判断の材料になります。

## 32.7 派生型ごとの分岐を1つの定義で書く

基底の Natvis の中で、派生型に応じた表示をしたくなることがあります。**`dynamic_cast` は書けません**（第5章 5.5節）。

現実的な方法は2つです。

**(1) 派生ごとに `Type` を書く。** 最も素直で、本章で採ってきた方法です。

**(2) 型タグを持たせる。** 基底に判別用のメンバーを持たせる設計です。

```cpp
enum class shape_kind : std::uint8_t { circle, box };

struct shape_base {
    shape_kind kind;          // ← 追加
    ...
};
```

```xml
<Type Name="vizkit::shape_base">
  <DisplayString Condition="kind == vizkit::shape_kind::circle">circle (base view)</DisplayString>
  <DisplayString Condition="kind == vizkit::shape_kind::box">box (base view)</DisplayString>
  <DisplayString>shape</DisplayString>
</Type>
```

(2) は冗長に見えますが、**vtable を持たない（仮想関数のない）階層では唯一の方法**です。デバッガが最派生型を解決できるのは、vtable がある場合に限られるからです。

```cpp
struct tagged_base { shape_kind kind; };     // 仮想関数なし
struct tagged_circle : tagged_base { ... };  // デバッガは最派生型を解決できない
```

この場合、`tagged_base*` のコンテナを表示しても、すべて `tagged_base` として表示されます。タグで分岐するしかありません。

**「デバッガが型を判別できるか」は、vtable の有無で決まる。** 型消去や tagged union を扱う第33章・第34章で、この論点が中心になります。

## 32.8 この章のまとめ

- **基底の Natvis は派生にも適用される**（`Inheritable` の既定は `true`）。派生固有のメンバーは見えなくなるので、派生にも `Type` を書き、基底は `ExpandedItem` で引き上げる。
- `Inheritable="false"` は、**基底が実装詳細のとき**、**基底の式が派生で誤った値を出すとき**、**タグ用の空クラスのとき**に使う。
- デバッガは**最も派生した型を解決する**（vtable がある場合）。基底ポインタのコンテナが、それぞれの派生型として表示される。
- その副作用が無限再帰。**`,nd` は「最派生型の解決を止める」指定子**。
- `p` と `p,nd` を並べると、型の取り違えを切り分けられる。
- 多重継承では**名前の衝突**に注意。衝突するなら平坦化をやめて `Item` で名前を付ける。
- **アドレス調整は型キャストに任せる。** `(char*)this + オフセット` の手計算をしない。
- 仮想継承は動かないことがある。**`Optional="true"` を付けて試し、駄目なら平坦化をあきらめる。**
- `dynamic_cast` は書けない。派生ごとに `Type` を書くか、**型タグを持たせる**。
- **デバッガが型を判別できるかは vtable の有無で決まる。** 仮想関数のない階層ではタグが必須。

## 32.9 演習

1. `circle` 用の `Type` を削除し、`shape_base` の定義だけで `circle` がどう表示されるか確認してください。`radius` がどこに行ったかも確認します。
2. `shape_base` に `Inheritable="false"` を付け、`circle` の表示がどう変わるか確認してください。
3. `scene`（`shape_base*` のコンテナ）を開き、要素がそれぞれの派生型として表示されることを確認してください。次に `scene,view(simple)` や各要素に `,nd` を付けて、基底として表示させてください。
4. `circle` の `ExpandedItem` から `,nd` を外し、無限に潜れることを確認してください（第12章の演習1の再確認）。**なぜそうなるか**を、本章 32.4節の言葉で説明してください。
5. `shape_base` から `virtual ~shape_base()` を削除して非多態にし、`scene` の表示がどう変わるか確認してください。vtable の有無が最派生型の解決を左右することの確認です。
