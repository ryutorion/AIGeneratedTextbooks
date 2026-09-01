# 第35章 スマートポインタと SmartPointer 要素

スマートポインタは、Natvis で扱う型の中でも特殊な位置にあります。

`DisplayString` と `Expand` を書けば表示は整います。しかしそれだけでは足りません。**ウォッチウィンドウで `sp->member` と打てるようにしたい**からです。`<SmartPointer>` 要素は、そのための専用要素です。

## 35.1 何が問題なのか

まず、素の型を作ります。

`vizkit\include\vizkit\util\ref_ptr.hpp`

```cpp
#pragma once

#include <cstddef>

namespace vizkit {

// 侵入型の参照カウントを持つオブジェクトの基底。
struct ref_counted {
    mutable long _refs = 0;
    virtual ~ref_counted() = default;
};

// 侵入型参照カウントのスマートポインタ。
template <class T>
class ref_ptr {
public:
    ref_ptr() = default;
    explicit ref_ptr(T* p) noexcept : _p(p) { if (_p) ++_p->_refs; }
    ref_ptr(const ref_ptr& o) noexcept : _p(o._p) { if (_p) ++_p->_refs; }
    ~ref_ptr() { if (_p && --_p->_refs == 0) delete _p; }

    [[nodiscard]] T* get()        const noexcept { return _p; }
    [[nodiscard]] T* operator->() const noexcept { return _p; }
    [[nodiscard]] T& operator*()  const noexcept { return *_p; }
    explicit operator bool()      const noexcept { return _p != nullptr; }

private:
    T* _p = nullptr;
};

} // namespace vizkit
```

`main.cpp`：

```cpp
#include <vizkit/util/ref_ptr.hpp>
// ...
    struct texture : vizkit::ref_counted {
        vizkit::name32 name{};
        int width = 0, height = 0;
    };

    auto* raw = new texture;
    raw->name.assign("diffuse");
    raw->width = 1024; raw->height = 512;

    vizkit::ref_ptr<texture> t1{ raw };
    vizkit::ref_ptr<texture> t2 = t1;      // 参照カウント 2
    vizkit::ref_ptr<texture> t_null;
```

`DisplayString` と `Expand` だけを書いた場合：

```xml
<Type Name="vizkit::ref_ptr&lt;*&gt;">
  <DisplayString Condition="_p == nullptr">empty</DisplayString>
  <DisplayString>{*_p} [refs={_p->_refs}]</DisplayString>
  <Expand>
    <Item Name="[refs]">_p->_refs</Item>
    <ExpandedItem>*_p</ExpandedItem>
  </Expand>
</Type>
```

```
▷ t1        {name="diffuse" width=1024 height=512 } [refs=2]
     [refs] 2
     name   "diffuse"
     width  1024
     height 512
```

表示としては十分です。しかし、ウォッチウィンドウでこう打つと失敗します。

```
t1->width          ← エラー: ポインタではない
```

`ref_ptr` はクラス型なので、`->` は `operator->()` の呼び出しになります。**Natvis からは関数を呼べない**ので（第5章 5.5節）、デバッガも呼べません。

`t1._p->width` と書けば動きますが、内部メンバー名を知っている必要があります。デバッグのたびにこれを打つのは苦痛です。

## 35.2 SmartPointer 要素

```xml
<Type Name="vizkit::ref_ptr&lt;*&gt;">
  <SmartPointer Usage="Minimal">_p</SmartPointer>
  <DisplayString Condition="_p == nullptr">empty</DisplayString>
  <DisplayString>{*_p} [refs={_p->_refs}]</DisplayString>
  <Expand>
    <Item Name="[refs]">_p->_refs</Item>
    <ExpandedItem>*_p</ExpandedItem>
  </Expand>
</Type>
```

要素の中身は、**基礎となるポインタを返す式**です。`Type` の直下、`DisplayString` より前に書きます。

これで、ウォッチウィンドウで次が使えるようになります。

```
t1->width          → 1024
*t1                → {name="diffuse" width=1024 ...}
t1 == t2           → true
```

**デバッガが `ref_ptr` をポインタとして扱うようになりました。** これが `SmartPointer` 要素の役割です。表示ではなく、**式の中での振る舞い**を変えます。

### Usage 属性

どこまでポインタとして扱うかを指定します。

| 値 | サポートする演算 |
|---|---|
| `Minimal` | `operator*()`、`operator->()`、`operator==()`、`operator!=()` |
| `Indexable` | `Minimal` に加えて `operator+()`、`operator-()`、`operator[]` |
| `Full` | 基礎ポインタへの変換演算子を含む。ポインタとして有効なすべての用法 |

**スマートポインタなら `Minimal`。** `std::shared_ptr` も `std::unique_ptr` も `Minimal` です。

```xml
<Type Name="std::shared_ptr&lt;*&gt;">
  <SmartPointer Usage="Minimal">_Ptr</SmartPointer>
  ...
</Type>

<Type Name="std::unique_ptr&lt;*&gt;">
  <SmartPointer Usage="Minimal">_Mypair._Myval2</SmartPointer>
  ...
</Type>
```

## 35.3 イテレータも「スマートポインタ」

`STL.natvis` を調べると、`SmartPointer` 要素の使用箇所の大半が**イテレータ**であることに気づきます。

```xml
<Type Name="std::_Vector_iterator&lt;*&gt;" Priority="Medium">
  <SmartPointer Usage="Indexable">_Ptr,na</SmartPointer>
</Type>

<Type Name="std::_List_iterator&lt;*&gt;">
  <SmartPointer Usage="Minimal">&amp;_Ptr->_Myval,na</SmartPointer>
</Type>

<Type Name="std::_Tree_iterator&lt;*&gt;">
  <SmartPointer Usage="Minimal">_Ptr->_Isnil ? nullptr : &amp;_Ptr->_Myval</SmartPointer>
</Type>
```

これは非常に実用的です。デバッグ中にイテレータを見つけたとき、`it->member` と打てる。ランダムアクセスイテレータなら `it[3]` や `it + 1` まで書ける（`Usage="Indexable"`）。

3つの読みどころがあります。

**(1) `_List_iterator` は `&_Ptr->_Myval`。** ノードそのものではなく、**ノードの中の値のアドレス**を返しています。イテレータが指しているのは値であって、リンクではないからです。

**(2) `_Tree_iterator` は番兵で `nullptr` を返す。** `end()` イテレータを間接参照しようとしたら、null になる。**不正なメモリを読むより安全**です（第7章 7.4節の考え方）。

**(3) `,na` が付いている。** アドレスを表示しない指定子です（第6章 6.6節）。`SmartPointer` の式にも書式指定子を書けます。

**自作ライブラリでイテレータを実装しているなら、`SmartPointer` を書く価値は非常に高い**です。表示を整えるより、`it->member` が打てることのほうが日々のデバッグに効きます。

## 35.4 DefaultExpansion 属性

`SmartPointer` を書くと、既定では**指し先の内容が展開されます**。`Expand` を書いていなくても、指し先のメンバーが子行として並びます。

これを抑制するのが `DefaultExpansion="false"` です。

```xml
<Type Name="std::vector&lt;wchar_t,*&gt;">
  <SmartPointer Usage="Indexable" DefaultExpansion="false">data()</SmartPointer>
  ...
</Type>
```

`std::vector<wchar_t>` は「先頭要素へのポインタとしても振る舞ってほしいが、展開は `ArrayItems` に任せたい」という要求なので、既定展開を切っています。

自作型でも、**`Expand` を明示的に書いているなら `DefaultExpansion="false"` を検討**してください。既定展開と `Expand` が二重になることを防げます。

## 35.5 参照カウントを見せる

スマートポインタのデバッグで最も見たいのは、**参照カウント**です。リークや二重解放の原因が、たいていここにあります。

```xml
<Type Name="vizkit::ref_ptr&lt;*&gt;">
  <SmartPointer Usage="Minimal" DefaultExpansion="false">_p</SmartPointer>

  <DisplayString Condition="_p == nullptr">empty</DisplayString>
  <DisplayString Condition="_p->_refs &lt;= 0">[!] refs={_p->_refs} {*_p}</DisplayString>
  <DisplayString>{*_p} [refs={_p->_refs}]</DisplayString>

  <Expand>
    <Item Name="[refs]" ExcludeView="simple">_p->_refs</Item>
    <Item Name="[ptr]"  ExcludeView="simple">_p</Item>
    <Item Name="[!]" Condition="_p != nullptr &amp;&amp; _p->_refs &lt;= 0">"参照カウントが 0 以下です。解放済みの可能性があります。"</Item>
    <ExpandedItem Condition="_p != nullptr">*_p</ExpandedItem>
  </Expand>
</Type>
```

```
  t1        {name="diffuse" ...} [refs=2]
  t_null    empty
  dangling  [!] refs=0 {...}
```

`_refs <= 0` は `ref_ptr` の不変条件違反です。**不変条件を Natvis に書く**という、第15章以降続けてきた方針の適用です。解放済みオブジェクトを指しているポインタが、一目で分かります。

`[ptr]` を出しておくのも実用的です。**同じオブジェクトを指す `ref_ptr` を見分ける**には、アドレスの一致を見るのが最も速いからです。

## 35.6 循環参照を見つける

参照カウント方式の最大の弱点が循環参照です。Natvis で直接検出することはできませんが、**手がかりを出す**ことはできます。

```cpp
struct node : vizkit::ref_counted {
    vizkit::name32          name{};
    vizkit::ref_ptr<node>   child;
    node*                   parent = nullptr;   // 弱い参照（カウントしない）
};
```

```xml
<Type Name="vizkit::ref_ptr&lt;*&gt;">
  ...
  <Expand>
    <Item Name="[refs]" ExcludeView="simple">_p->_refs</Item>
    <Item Name="[ptr]"  ExcludeView="simple">_p</Item>
    <Synthetic Name="[!] 参照カウントが高い" IncludeView="detail" Condition="_p != nullptr &amp;&amp; _p->_refs &gt; 8">
      <DisplayString>refs={_p->_refs}。循環参照の可能性があります。</DisplayString>
    </Synthetic>
    <ExpandedItem Condition="_p != nullptr">*_p</ExpandedItem>
  </Expand>
</Type>
```

閾値は恣意的ですが、**「そんなに多いはずがない」という知識を Natvis に書き込む**という点で、第9章 9.4節の巨大サイズ検出と同じ発想です。

そして、循環を辿るときには `ExpandedItem` が助けになります。`t1` を開くと `child` が出て、その `child` を開くと……と辿っていけば、同じアドレスに戻ることで循環が確認できます。`[ptr]` を表示しておく意味がここにあります。

## 35.7 非所有ポインタ / weak 参照

`weak_ptr` 風の型（所有せず、対象が生きているかを確認できる）はどうでしょうか。

`STL.natvis` の `std::weak_ptr` には、**`SmartPointer` 要素がありません**。

理由は明快です。`weak_ptr` は間接参照できない型だからです。`lock()` して `shared_ptr` を得なければ使えません。**デバッガでポインタとして扱えるようにすると、実際のコードでできないことができてしまい、誤解を招きます。**

> **`SmartPointer` は、その型が実際にポインタとして使える場合にだけ書く。**

自作の弱参照型でも同じです。`SmartPointer` は書かず、`DisplayString` で状態を示します。

```xml
<Type Name="vizkit::weak_ref&lt;*&gt;">
  <DisplayString Condition="_p == nullptr">[expired]</DisplayString>
  <DisplayString Condition="_p->_refs == 0">[expired] (was {*_p})</DisplayString>
  <DisplayString>[weak] {*_p}</DisplayString>
  <Expand>
    <Item Name="[expired]">_p == nullptr || _p->_refs == 0</Item>
    <Item Name="[ptr]">_p</Item>
  </Expand>
</Type>
```

`[weak]` という印を付けておくと、所有しているポインタと区別できます。これも「状態は分けて表示する」（第7章 7.5節）の適用です。

## 35.8 この章のまとめ

- `<SmartPointer>` は**表示ではなく、式の中での振る舞い**を変える要素。ウォッチウィンドウで `sp->member` と打てるようになる。
- 中身は**基礎となるポインタを返す式**。`Type` の直下、`DisplayString` より前に書く。書式指定子も使える。
- `Usage` は `Minimal`（`*` `->` `==` `!=`）/ `Indexable`（＋ `+` `-` `[]`）/ `Full`（変換演算子まで）。**スマートポインタなら `Minimal`。**
- **`STL.natvis` の `SmartPointer` の大半はイテレータ。** 自作イテレータに書く価値は非常に高い。
- `_Tree_iterator` が番兵で `nullptr` を返しているように、**間接参照できない状態では null を返す**のが安全。
- `DefaultExpansion="false"` で既定展開を切れる。`Expand` を明示的に書いているなら検討する。
- **参照カウントは必ず表示する。** `_refs <= 0` は不変条件違反なので `[!]` を出す。`[ptr]` も同じオブジェクトの識別に役立つ。
- 循環参照は直接検出できないが、**閾値を超えた参照カウントに印を付ける**ことで手がかりになる。
- **`SmartPointer` は、その型が実際にポインタとして使える場合にだけ書く。** `weak_ptr` 風の型には書かない。

## 35.9 演習

1. `SmartPointer` 要素を消して `.natvisreload` し、ウォッチウィンドウで `t1->width` がエラーになることを確認してください。書き戻して動くようになることも確認します。
2. `Usage="Minimal"` を `Usage="Indexable"` に変え、`t1[0]` が何を表示するか確認してください。`ref_ptr` に `Indexable` が不適切な理由を考えてください。
3. `t1` と `t2` の `[ptr]` が同じアドレスであること、`[refs]` が 2 であることを確認してください。`t2` をスコープから外し（別の関数に切り出すなど）、`[refs]` が 1 に戻る様子を追ってください。
4. `raw->_refs` をデバッガから `0` に書き換え、`[!]` の警告が出ることを確認してください。
5. `STL.natvis` で `std::_Tree_iterator` の `SmartPointer` を探し、`std::map` の `end()` イテレータをウォッチウィンドウで間接参照してみてください。`nullptr` が返ることを確認します。
