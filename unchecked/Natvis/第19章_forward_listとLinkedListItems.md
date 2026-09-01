# 第19章 forward_list と LinkedListItems

第5部に入ります。ここからは、**次の要素の位置が添字から計算できない**型を扱います。

配列系では「先頭 ＋ i × 要素サイズ」でアドレスが求まりました。リストではポインタを1つずつ辿るしかありません。`<LinkedListItems>` は、その辿り方をデバッガに教える要素です。

## 19.1 型を作る

まずは最も素直な形、NULL 終端の単方向リストから始めます。

`vizkit\include\vizkit\node\forward_list.hpp`

```cpp
#pragma once

#include <cstddef>
#include <utility>

namespace vizkit {

// NULL 終端の単方向リスト。ノードが値を所有する。
template <class T>
class forward_list {
public:
    struct node {
        node* next = nullptr;
        T     value{};
    };

    forward_list() = default;
    ~forward_list() { clear(); }

    forward_list(const forward_list&) = delete;
    forward_list& operator=(const forward_list&) = delete;

    void push_front(const T& v) {
        node* n = new node{ _head, v };
        _head = n;
        ++_size;
    }

    void clear() noexcept {
        while (_head) { node* n = _head->next; delete _head; _head = n; }
        _size = 0;
    }

    [[nodiscard]] std::size_t size() const noexcept { return _size; }

private:
    node*       _head = nullptr;
    std::size_t _size = 0;
};

} // namespace vizkit
```

`main.cpp`：

```cpp
#include <vizkit/node/forward_list.hpp>
// ...
    vizkit::forward_list<int> fl;
    fl.push_front(30);
    fl.push_front(20);
    fl.push_front(10);      // 10 -> 20 -> 30

    vizkit::forward_list<int> fl_empty;
```

素の表示は、第1章で予告したとおりの見えなさです。

```
▷ fl            {_head=0x000001f4a2c33a80 {next=0x000001f4a2c33a50 {...} value=10 } _size=3 }
   ▷ _head      0x000001f4a2c33a80 {next=... value=10 }
      ▷ next    0x000001f4a2c33a50 {next=... value=20 }
         ▷ next 0x000001f4a2c33a20 {next=0x0 value=30 }
```

要素を見るには、`next` を1回クリックするごとに1つ進むしかありません。100 要素のリストを目視で追うことは不可能です。

## 19.2 LinkedListItems ―― 4つの要素

```xml
<Type Name="vizkit::forward_list&lt;*&gt;">
  <DisplayString>{{ size={_size} }}</DisplayString>
  <Expand>
    <Item Name="[size]" ExcludeView="simple">_size</Item>
    <LinkedListItems>
      <Size>_size</Size>
      <HeadPointer>_head</HeadPointer>
      <NextPointer>next</NextPointer>
      <ValueNode>value</ValueNode>
    </LinkedListItems>
  </Expand>
</Type>
```

```
▷ fl             { size=3 }              vizkit::forward_list<int>
     [size]      3
     [0]         10
     [1]         20
     [2]         30
   ▷ [Raw View]  {_head=... _size=3 }
```

4つの要素の役割は次のとおりです。

| 要素 | 意味 | 評価される文脈 |
|---|---|---|
| `<Size>` | リストの長さ | **コンテナ**（`forward_list`） |
| `<HeadPointer>` | 最初のノードを指す式 | **コンテナ** |
| `<NextPointer>` | 次のノードを指す式 | **ノード** |
| `<ValueNode>` | ノードの値を指す式 | **ノード** |

## 19.3 評価の文脈が途中で変わる

前節の表で最も重要なのは、右の列です。**`NextPointer` と `ValueNode` は、コンテナではなくノードの文脈で評価されます。**

```xml
<HeadPointer>_head</HeadPointer>       <!-- this は forward_list -->
<NextPointer>next</NextPointer>        <!-- this は node -->
<ValueNode>value</ValueNode>           <!-- this は node -->
```

`<NextPointer>` に `_head->next` と書いてしまうのが典型的なミスです。これは「コンテナのつもり」で書いた式で、実際にはノードの文脈で評価されるため、`node` に `_head` というメンバーがないとしてエラーになります。

```xml
<!-- 誤り: node には _head がない -->
<NextPointer>_head->next</NextPointer>

<!-- 正しい: node のメンバー next -->
<NextPointer>next</NextPointer>
```

公式ドキュメントも明示しています。「この式はリンクリストノードのコンテキストで評価され、親のリンクリスト型ではありません」。

`ArrayItems` までは、`Expand` の中のすべての式がコンテナの文脈でした。第5部から**文脈が2つになる**という点が、リストと木を書くときの最大の切り替えポイントです。

## 19.4 ValueNode に this を書く

`<ValueNode>` は「ノードの値を指す式」ですが、**空にするか `this` と書くと、ノードそのものが値になります**。

```xml
<ValueNode>this</ValueNode>
```

```
     [0]        {next=0x... value=10 }        vizkit::forward_list<int>::node
     [1]        {next=0x... value=20 }
     [2]        {next=0x0 value=30 }
```

ノードそのものが並ぶようになりました。この形が必要になるのは、**ノードが複数の情報を持っている場合**です。連想コンテナのノード（キーと値）や、赤黒木のノード（色や親ポインタ）がそれにあたります。第21章で実際に使います。

`forward_list` では値だけが見たいので、`value` を書くのが正解です。

## 19.5 Size は省略できる

`<Size>` は必須ではありません。**省略すると、デバッガはリストを最後まで辿って要素数を数えます。**

```xml
<LinkedListItems>
  <HeadPointer>_head</HeadPointer>
  <NextPointer>next</NextPointer>
  <ValueNode>value</ValueNode>
</LinkedListItems>
```

要素数のメンバーを持たないリスト（`std::forward_list` がまさにそうです）では、これが唯一の方法になります。

ただし、**書けるなら `Size` を書くべき**です。理由は2つあります。

**(1) 速い。** デバッガは要素数を先に知っていれば、表示すべき行数を即座に決められます。省略した場合は毎回リスト全体を辿ります。

**(2) 安全。** リストが破損して循環していた場合、`Size` があればそこで止まります。省略していると、デバッガが延々と辿り続けます（第20章で詳しく扱います）。

`Size` を書く場合は、**それがリストの実態と一致していることが前提**になる点に注意してください。`_size` の更新を忘れているバグを追っているときは、逆に `Size` を消したビューを用意すると実態が見えます。

```xml
<Synthetic Name="[traverse]" IncludeView="detail">
  <DisplayString>_size を信用せず実際に辿った結果</DisplayString>
  <Expand>
    <LinkedListItems>
      <HeadPointer>_head</HeadPointer>
      <NextPointer>next</NextPointer>
      <ValueNode>value</ValueNode>
    </LinkedListItems>
  </Expand>
</Synthetic>
```

`fl,view(detail)` で、`[size]` が 3 なのに `[traverse]` には 4 個並ぶ、といった不整合を目で見つけられます。**同じデータを2つの根拠で表示して突き合わせる**という、第17章の論理／物理ビューと同じ発想です。

### Size は複数書ける

`<Size>` は `Condition` を付けて複数並べられます。この場合、**条件が真になった最初のもの**（または条件なしのもの）が使われます。`DisplayString` と同じルールです。

```xml
<Size Condition="_size &lt;= 10000">_size</Size>
<Size>10000</Size>                        <!-- 異常値のときは 10000 で打ち切る -->
```

破損したリストで `_size` が巨大になっていても、表示が 10000 件で止まります。実務的な防御策です。

## 19.6 侵入型リスト ―― フックから要素本体へ戻る

`forward_list` はノードが値を所有していました。**侵入型（intrusive）リスト**は逆で、要素自身がリンク用のメンバーを持ちます。

`vizkit\include\vizkit\node\intrusive_list.hpp`

```cpp
#pragma once

#include <cstddef>

namespace vizkit {

// リンク用のフック。要素型はこれを public 継承する。
struct list_hook {
    list_hook* next = nullptr;
};

// 要素自身がフックを持つ単方向リスト。ノードを別途確保しない。
template <class T>
class intrusive_list {
public:
    void push_front(T* item) noexcept {
        item->next = _head;
        _head = item;
        ++_size;
    }

    [[nodiscard]] std::size_t size() const noexcept { return _size; }

private:
    list_hook*  _head = nullptr;
    std::size_t _size = 0;
};

} // namespace vizkit
```

`main.cpp`：

```cpp
#include <vizkit/node/intrusive_list.hpp>
// ...
    struct task : vizkit::list_hook {
        int         id = 0;
        vizkit::name32 label{};
    };

    task t1; t1.id = 1; t1.label.assign("load");
    task t2; t2.id = 2; t2.label.assign("parse");
    task t3; t3.id = 3; t3.label.assign("render");

    vizkit::intrusive_list<task> queue;
    queue.push_front(&t3);
    queue.push_front(&t2);
    queue.push_front(&t1);
```

### 問題：辿るのは list_hook、見たいのは task

`_head` の型は `list_hook*` です。素直に書くとこうなります。

```xml
<LinkedListItems>
  <Size>_size</Size>
  <HeadPointer>_head</HeadPointer>
  <NextPointer>next</NextPointer>
  <ValueNode>this</ValueNode>
</LinkedListItems>
```

```
     [0]        {next=0x... }        vizkit::list_hook
     [1]        {next=0x... }
     [2]        {next=0x0 }
```

**フックしか見えません。** `id` も `label` も、`list_hook` のメンバーではないからです。

### 解決：要素型にキャストする

`ValueNode` はノードの文脈で評価される式なので、そこでキャストします。`$T1` は `task` です（第14章 14.4節）。

```xml
<Type Name="vizkit::intrusive_list&lt;*&gt;">
  <DisplayString>{{ size={_size} }}</DisplayString>
  <Expand>
    <Item Name="[size]" ExcludeView="simple">_size</Item>
    <LinkedListItems>
      <Size>_size</Size>
      <HeadPointer>_head</HeadPointer>
      <NextPointer>next</NextPointer>
      <ValueNode>*($T1*)this</ValueNode>
    </LinkedListItems>
  </Expand>
</Type>
```

```
▷ queue          { size=3 }                     vizkit::intrusive_list<task>
   ▷ [0]         {next=0x... id=1 label="load" }        task
   ▷ [1]         {next=0x... id=2 label="parse" }       task
   ▷ [2]         {next=0x... id=3 label="render" }      task
```

`*($T1*)this` は「いまのノード（`list_hook`）を要素型として読み直し、その実体を値とする」という意味です。

`task` は `list_hook` を単一継承しているので、`list_hook` の先頭アドレスと `task` の先頭アドレスが一致します。したがって単純なキャストで済みます。

### フックが先頭にない場合

実務では、フックがメンバーとして途中に置かれる設計もあります。

```cpp
struct task {
    int id;
    vizkit::list_hook hook;    // 途中にある
    name32 label;
};
```

この場合、フックのアドレスから要素の先頭に戻るには、オフセットを引く必要があります。C++ なら `offsetof` を使うところですが、Natvis では手で書きます。

```xml
<ValueNode>*($T1*)((char*)this - (__int64)&amp;(($T1*)0)->hook)</ValueNode>
```

`&(($T1*)0)->hook` が `offsetof(T, hook)` に相当します。読みにくいので `Intrinsic` に逃がしましょう。

```xml
<Intrinsic Name="hook_offset" Expression="(__int64)&amp;(($T1*)0)->hook" />
<Intrinsic Name="owner_of" Expression="($T1*)((char*)this - hook_offset())" />
```

ただし `Intrinsic` はコンテナの文脈で定義されているので、`this` がノードを指す `ValueNode` の中で `owner_of()` を使えるかどうかは環境に依存します。**確実にしたいなら、式を `ValueNode` に直接書いてください。**

本書の `vizkit` は「フックを継承する」設計を採っているので、単純な `*($T1*)this` で済みます。**Natvis を書きやすい設計を選ぶ**というのは、こういうところにも効いてきます（第6章 6.8節でも同じ話をしました）。

## 19.7 現時点の Natvis

```xml
  <!-- ===== node ===== -->

  <Type Name="vizkit::forward_list&lt;*&gt;">
    <DisplayString Condition="_head == nullptr">[empty]</DisplayString>
    <DisplayString>{{ size={_size} }}</DisplayString>
    <Expand>
      <Item Name="[size]" ExcludeView="simple">_size</Item>
      <Synthetic Name="[traverse]" IncludeView="detail">
        <DisplayString>_size を信用せず実際に辿った結果</DisplayString>
        <Expand>
          <LinkedListItems>
            <HeadPointer>_head</HeadPointer>
            <NextPointer>next</NextPointer>
            <ValueNode>value</ValueNode>
          </LinkedListItems>
        </Expand>
      </Synthetic>
      <LinkedListItems>
        <Size Condition="_size &lt;= 10000">_size</Size>
        <Size>10000</Size>
        <HeadPointer>_head</HeadPointer>
        <NextPointer>next</NextPointer>
        <ValueNode>value</ValueNode>
      </LinkedListItems>
    </Expand>
  </Type>

  <Type Name="vizkit::intrusive_list&lt;*&gt;">
    <DisplayString Condition="_head == nullptr">[empty]</DisplayString>
    <DisplayString>{{ size={_size} }}</DisplayString>
    <Expand>
      <Item Name="[size]" ExcludeView="simple">_size</Item>
      <LinkedListItems>
        <Size Condition="_size &lt;= 10000">_size</Size>
        <Size>10000</Size>
        <HeadPointer>_head</HeadPointer>
        <NextPointer>next</NextPointer>
        <ValueNode>*($T1*)this</ValueNode>
      </LinkedListItems>
    </Expand>
  </Type>
```

## 19.8 この章のまとめ

- `<LinkedListItems>` の要素は `Size` / `HeadPointer` / `NextPointer` / `ValueNode` の4つ。
- **`NextPointer` と `ValueNode` はノードの文脈で評価される。** `Size` と `HeadPointer` はコンテナの文脈。第5部で最も間違えやすい点。
- `<ValueNode>` を空にするか `this` と書くと、**ノードそのものが値**になる。ノードが複数の情報を持つときに使う。
- **`<Size>` は省略できる**（デバッガがリストを辿って数える）が、速度と安全性のために書くべき。
- `<Size>` は `Condition` 付きで複数書ける。異常値のときに打ち切る防御策が作れる。
- `Size` を信じるビューと、辿った結果のビューを並べると、`_size` の更新漏れを見つけられる。
- 侵入型リストでは `ValueNode` で **`*($T1*)this`** と要素型にキャストする。フックを継承させる設計にしておくと、この式が単純になる。

## 19.9 演習

1. `<NextPointer>_head->next</NextPointer>` と書いて、どんなエラーになるか確認してください。評価の文脈が違うことの確認です。
2. `<ValueNode>value</ValueNode>` を `<ValueNode>this</ValueNode>` に変え、表示がどう変わるか確認してください。
3. `<Size>_size</Size>` を消して、要素数がどう決まるかを確認してください。そのうえで `_size` をデバッガから `99` に書き換え、`Size` あり／なしで表示がどう変わるか比べてください。
4. `forward_list` のノードに `<Type Name="vizkit::forward_list&lt;*&gt;::node">` で Natvis を書き、`ValueNode` を `this` にしたときの表示を整えてください。入れ子の型に Natvis を当てられることの確認です。
5. `intrusive_list` の `ValueNode` から `*($T1*)` を外し、`list_hook` しか見えなくなることを確認してください。
