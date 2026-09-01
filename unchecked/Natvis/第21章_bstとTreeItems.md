# 第21章 bst<K, V> と TreeItems

リストの次は木です。`<TreeItems>` は `LinkedListItems` の兄弟のような要素で、**左右2本のポインタを辿って中順（in-order）に列挙**します。

構文はほとんど同じですが、「ノードが2つの情報（キーと値）を持つ」という点が新しく、これが連想コンテナ全般の見せ方につながります。

## 21.1 型を作る

`vizkit\include\vizkit\node\bst.hpp`

```cpp
#pragma once

#include <cstddef>

namespace vizkit {

// 平衡化しない素朴な二分探索木。
template <class K, class V>
class bst {
public:
    struct node {
        node* left  = nullptr;
        node* right = nullptr;
        K     key{};
        V     value{};
    };

    bst() = default;
    ~bst() { destroy(_root); }

    bst(const bst&) = delete;
    bst& operator=(const bst&) = delete;

    void insert(const K& k, const V& v) {
        node** slot = &_root;
        while (*slot) {
            if (k < (*slot)->key)       slot = &(*slot)->left;
            else if ((*slot)->key < k)  slot = &(*slot)->right;
            else { (*slot)->value = v; return; }
        }
        *slot = new node{ nullptr, nullptr, k, v };
        ++_size;
    }

    [[nodiscard]] std::size_t size() const noexcept { return _size; }

private:
    static void destroy(node* n) noexcept {
        if (!n) return;
        destroy(n->left);
        destroy(n->right);
        delete n;
    }

    node*       _root = nullptr;
    std::size_t _size = 0;
};

} // namespace vizkit
```

`main.cpp`：

```cpp
#include <vizkit/node/bst.hpp>
// ...
    vizkit::bst<int, vizkit::name32> tree;
    auto add = [&](int k, const char* s) {
        vizkit::name32 n; n.assign(s); tree.insert(k, n);
    };
    add(50, "fifty");
    add(30, "thirty");
    add(70, "seventy");
    add(20, "twenty");
    add(40, "forty");
    add(60, "sixty");

    vizkit::bst<int, vizkit::name32> tree_empty;
```

素の表示は、リストよりさらに絶望的です。

```
▷ tree          {_root=0x000001f4a2c33b00 {left=... right=... key=50 value=... } _size=6 }
```

木を目視で辿るには、左右どちらに進むかを毎回選びながらクリックすることになります。6 要素でも苦痛です。

## 21.2 TreeItems ―― 5つの要素

```xml
<Type Name="vizkit::bst&lt;*,*&gt;">
  <DisplayString>{{ size={_size} }}</DisplayString>
  <Expand>
    <Item Name="[size]" ExcludeView="simple">_size</Item>
    <TreeItems>
      <Size>_size</Size>
      <HeadPointer>_root</HeadPointer>
      <LeftPointer>left</LeftPointer>
      <RightPointer>right</RightPointer>
      <ValueNode>this</ValueNode>
    </TreeItems>
  </Expand>
</Type>
```

```
▷ tree            { size=6 }                    vizkit::bst<int,vizkit::name32>
     [size]       6
   ▷ [0]          {left=0x... right=0x... key=20 value="twenty" }
   ▷ [1]          {left=... key=30 ... }
   ▷ [2]          {left=... key=40 ... }
   ▷ [3]          {left=... key=50 ... }
   ▷ [4]          {left=... key=60 ... }
   ▷ [5]          {left=... key=70 ... }
   ▷ [Raw View]   {_root=... _size=6 }
```

**キーの昇順に並びました。** `TreeItems` は中順走査を行うので、二分探索木ならソート済みの順序で出てきます。

要素の意味は `LinkedListItems` とほぼ同じです。

| 要素 | 意味 | 評価される文脈 |
|---|---|---|
| `<Size>` | 要素数 | **コンテナ** |
| `<HeadPointer>` | 根ノードを指す式 | **コンテナ** |
| `<LeftPointer>` | 左の子を指す式 | **ノード** |
| `<RightPointer>` | 右の子を指す式 | **ノード** |
| `<ValueNode>` | ノードの値 | **ノード** |

第19章 19.3節と同じく、**下3つはノードの文脈**です。`_root->left` と書いてはいけません。

## 21.3 ノードに Natvis を当てる

`{left=0x... right=0x... key=20 value="twenty" }` では読めません。`ValueNode` を `this` にしたので、表示はノード型の `DisplayString` に委ねられています。

第20章 20.5節で使った、入れ子クラスへの Natvis を書きます。

```xml
<Type Name="vizkit::bst&lt;*,*&gt;::node">
  <DisplayString>{key} =&gt; {value}</DisplayString>
  <Expand>
    <Item Name="key">key</Item>
    <Item Name="value">value</Item>
    <Item Name="[left]"  IncludeView="detail">left</Item>
    <Item Name="[right]" IncludeView="detail">right</Item>
  </Expand>
</Type>
```

```
▷ tree            { size=6 }
     [size]       6
   ▷ [0]          20 => "twenty"
   ▷ [1]          30 => "thirty"
   ▷ [2]          40 => "forty"
   ▷ [3]          50 => "fifty"
   ▷ [4]          60 => "sixty"
   ▷ [5]          70 => "seventy"
```

`=>` は XML では `=&gt;` と書きます（`>` はそのままでも通りますが、対称性のためエスケープしておくのが無難です。第4章 4.7節）。

`left` / `right` を `IncludeView="detail"` に閉じ込めているのは、第20章と同じ理由です。既定ビューで子ノードを開けるようにすると、木の中を何度でも潜れてしまいます。

### なぜ ValueNode を this にするのか

`bst` のノードはキーと値の2つを持っています。`<ValueNode>value</ValueNode>` と書けば値だけが並びますが、**キーが見えなくなります**。

```
     [0]        "twenty"      ← どのキーの値なのか分からない
```

連想コンテナでは、キーと値の両方が見えなければ意味がありません。したがって `ValueNode` を `this` にして、ノード全体を見せる。そのうえでノード型の `DisplayString` で `key => value` の形に整える ―― これが連想コンテナの定石です。

第24章の `flat_map` 以降、第6部でも同じ形を繰り返し使います。

## 21.4 キーを名前の列に出す

もう一段進めると、キーを「名前」の列に置けます。`STL.natvis` の `std::map` がそう見えるのは、この形だからです。

しかし `TreeItems` の `ValueNode` には `Name` 属性がありません。**`TreeItems` ではキーを名前の列に出せない**のです。

これができるのは、`Item` の `Name` 属性に式を書ける場合（第10章 10.2節）と、`CustomListItems` の `<Item Name="{key}">` です。第24章で `flat_map` を書くときに実現します。

`std::map` がどうしているかというと、実は同じ制約を受けています。

```
▷ m             { size=3 }
   ▷ [0]        {first=1 second="one"}
   ▷ [1]        {first=2 second="two"}
```

`std::map` も `[0]` `[1]` という添字表示で、キーは値の列の中に `first=...` として出ています。`TreeItems` を使う以上、これが限界です。

**要素の見せ方は、使う列挙要素によって決まる。** これは Natvis の重要な制約なので、覚えておいてください。

## 21.5 空の木とガード

`_root` が NULL のときも `TreeItems` は破綻しませんが、表示を明示しておきます。

```xml
<DisplayString Condition="_root == nullptr &amp;&amp; _size != 0">[corrupt: size={_size} but root is null]</DisplayString>
<DisplayString Condition="_root == nullptr">[empty]</DisplayString>
<DisplayString>{{ size={_size} }}</DisplayString>
```

第15章以降と同じ、不変条件を書き込むパターンです。

`Size` の上限による打ち切りも、リストと同様に有効です。

```xml
<Size Condition="_size &lt;= 10000">_size</Size>
<Size>10000</Size>
```

木が循環している（`left` が祖先を指しているような破損）場合、`Size` がなければデバッガは終わりません。**`TreeItems` でも `Size` は書くべきです。**

## 21.6 木の形を見る ―― 構造ビュー

`TreeItems` は中順に平坦化するので、**木の形が見えません**。`bst` は平衡化しないので、挿入順によっては連結リストのように偏ります。それを確認したいことがあります。

`Synthetic` と、ノード型の `Expand` を組み合わせて、根から辿れる構造ビューを作ります。

```xml
<Synthetic Name="[structure]" IncludeView="detail" Condition="_root != nullptr">
  <DisplayString>root = {*_root}</DisplayString>
  <Expand>
    <ExpandedItem>_root,view(tree)</ExpandedItem>
  </Expand>
</Synthetic>
```

```xml
<Type Name="vizkit::bst&lt;*,*&gt;::node" IncludeView="tree">
  <DisplayString>{key} =&gt; {value}</DisplayString>
  <Expand>
    <Item Name="L" Condition="left  != nullptr">left,view(tree)</Item>
    <Item Name="R" Condition="right != nullptr">right,view(tree)</Item>
  </Expand>
</Type>
```

```
tree,view(detail)
  └ [structure]     root = 50 => "fifty"
       ├ L          30 => "thirty"
       │   ├ L      20 => "twenty"
       │   └ R      40 => "forty"
       └ R          70 => "seventy"
             └ L    60 => "sixty"
```

`L` / `R` という短い名前にして、`Condition` で存在する枝だけを出しています。子がないノードは開けないので、葉が視覚的に分かります。

第13章の「`Type` 要素自体に `IncludeView` を付けられる」という性質を使って、**同じノード型に2種類の見せ方**を用意しました。既定ビューでは `key => value` の1行、`tree` ビューでは再帰的に潜れる構造。ビューは再帰表示の制御に使える、という第13章 13.6節の話が、ここで実際の道具になっています。

なお、木が深いと `[structure]` は延々と潜れます。`IncludeView="detail"` に閉じ込めているのはそのためです。

## 21.7 std::map の TreeItems を読む

公式ドキュメントに載っている `std::map` の定義を見てください。

```xml
<Type Name="std::map&lt;*&gt;">
  <DisplayString>{{size = {_Mysize}}}</DisplayString>
  <Expand>
    <Item Name="[size]">_Mysize</Item>
    <Item Name="[comp]">comp</Item>
    <TreeItems>
      <Size>_Mysize</Size>
      <HeadPointer>_Myhead-&gt;_Parent</HeadPointer>
      <LeftPointer>_Left</LeftPointer>
      <RightPointer>_Right</RightPointer>
      <ValueNode Condition="!((bool)_Isnil)">_Myval</ValueNode>
    </TreeItems>
  </Expand>
</Type>
```

2つの点に注目してください。

**(1) `HeadPointer` が `_Myhead->_Parent`。** `std::map` は番兵ノード（`_Myhead`）を持ち、その `_Parent` が実際の根を指しています。第20章 20.3節で見た「番兵の次」と同じ発想です。**番兵を使う実装では、`HeadPointer` は必ず一段間接になる**と覚えておくと、他人のコードの Natvis も読めます。

**(2) `ValueNode` に `Condition` が付いている。** `_Isnil` は「このノードは番兵である」という印です。番兵に行き当たったら値として扱わない、という指定です。

自作の `bst` は NULL 終端なので、この `Condition` は不要でした。しかし番兵方式の木を書くなら、同じ形が必要になります。次章の赤黒木では、この判断が実際に出てきます。

## 21.8 現時点の bst の Natvis

```xml
  <Type Name="vizkit::bst&lt;*,*&gt;">
    <DisplayString Condition="_root == nullptr &amp;&amp; _size != 0">[corrupt: size={_size} but root is null]</DisplayString>
    <DisplayString Condition="_root == nullptr">[empty]</DisplayString>
    <DisplayString>{{ size={_size} }}</DisplayString>
    <Expand>
      <Item Name="[size]" ExcludeView="simple">_size</Item>

      <Synthetic Name="[structure]" IncludeView="detail" Condition="_root != nullptr">
        <DisplayString>root = {*_root}</DisplayString>
        <Expand>
          <ExpandedItem>_root,view(tree)</ExpandedItem>
        </Expand>
      </Synthetic>

      <TreeItems>
        <Size Condition="_size &lt;= 10000">_size</Size>
        <Size>10000</Size>
        <HeadPointer>_root</HeadPointer>
        <LeftPointer>left</LeftPointer>
        <RightPointer>right</RightPointer>
        <ValueNode>this</ValueNode>
      </TreeItems>
    </Expand>
  </Type>

  <Type Name="vizkit::bst&lt;*,*&gt;::node" ExcludeView="tree">
    <DisplayString>{key} =&gt; {value}</DisplayString>
    <Expand>
      <Item Name="key">key</Item>
      <Item Name="value">value</Item>
      <Item Name="[left]"  IncludeView="detail">left</Item>
      <Item Name="[right]" IncludeView="detail">right</Item>
    </Expand>
  </Type>

  <Type Name="vizkit::bst&lt;*,*&gt;::node" IncludeView="tree">
    <DisplayString>{key} =&gt; {value}</DisplayString>
    <Expand>
      <Item Name="L" Condition="left  != nullptr">left,view(tree)</Item>
      <Item Name="R" Condition="right != nullptr">right,view(tree)</Item>
    </Expand>
  </Type>
```

## 21.9 この章のまとめ

- `<TreeItems>` は `Size` / `HeadPointer` / `LeftPointer` / `RightPointer` / `ValueNode` の5要素。**中順（in-order）に列挙する**ので、二分探索木ならソート順に並ぶ。
- `LeftPointer` / `RightPointer` / `ValueNode` は**ノードの文脈**で評価される。`LinkedListItems` と同じ。
- 連想コンテナでは **`ValueNode` を `this` にして、ノード型の `DisplayString` で `key => value` に整える**のが定石。
- **`TreeItems` ではキーを「名前」の列に出せない。** `std::map` も同じ制約を受けている。名前にキーを出せるのは `CustomListItems`（第24章以降）。
- 木が破損して循環していると `Size` なしでは終わらない。**`TreeItems` でも `Size` を書く。**
- `Type` 要素に `IncludeView` を付けて、同じノード型に「1行表示」と「再帰的な構造表示」の2つを用意できる。
- **番兵を使う実装では `HeadPointer` が一段間接になる**（`std::map` の `_Myhead->_Parent`）。`ValueNode` に番兵を除く `Condition` が必要になることもある。

## 21.10 演習

1. `<ValueNode>this</ValueNode>` を `<ValueNode>value</ValueNode>` に変え、キーが見えなくなることを確認してください。
2. `LeftPointer` に `_root->left` と書いて、どんなエラーになるか確認してください（第19章 19.3節の再確認）。
3. `LeftPointer` と `RightPointer` を入れ替えて、キーが降順に並ぶことを確認してください。中順走査の意味が分かります。
4. `add()` を `10, 20, 30, 40, 50, 60` の昇順で呼ぶ木を作り、`,view(detail)` の `[structure]` で右に一直線に伸びる形を確認してください。平衡化しない木の問題が目に見えます。
5. `bst<vizkit::name32, int>` を作ろうとすると `operator<` がなくてコンパイルできません。`name32` に比較演算子を足したうえで、キーが文字列の木がどう表示されるか確認してください。
