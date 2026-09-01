# 第26章 chain_map ―― バケット配列 × ノードリストの二重走査

第1章の冒頭で、「何ひとつ分からない」表示の例として挙げた型です。

```
▷ m      {_buckets=0x000001f4a2c33a80 {_head=0x0...} _bucket_count=8 _size=3 ...}
```

`apple` も `banana` も `cherry` も画面上のどこにもない、というあの状態を、この章で解決します。本書の技術的な山場です。

## 26.1 型を作る

`vizkit\include\vizkit\assoc\chain_map.hpp`

```cpp
#pragma once

#include <cstddef>

namespace vizkit {

// チェイン法（separate chaining）のハッシュマップ。
// バケット配列の各要素が、単方向リストの先頭を指す。
template <class K, class V>
class chain_map {
public:
    struct node {
        node*       next = nullptr;
        std::size_t hash = 0;
        K           key{};
        V           value{};
    };

    struct bucket {
        node* head = nullptr;
    };

    explicit chain_map(std::size_t n = 8) : _bucket_count(n) {
        _buckets = new bucket[n]{};
    }
    ~chain_map() {
        for (std::size_t b = 0; b < _bucket_count; ++b) {
            node* p = _buckets[b].head;
            while (p) { node* nx = p->next; delete p; p = nx; }
        }
        delete[] _buckets;
    }

    chain_map(const chain_map&) = delete;
    chain_map& operator=(const chain_map&) = delete;

    void insert(std::size_t h, const K& k, const V& v) {
        std::size_t b = h % _bucket_count;
        for (node* p = _buckets[b].head; p; p = p->next)
            if (p->hash == h && p->key == k) { p->value = v; return; }
        _buckets[b].head = new node{ _buckets[b].head, h, k, v };
        ++_size;
    }

    [[nodiscard]] std::size_t size()         const noexcept { return _size; }
    [[nodiscard]] std::size_t bucket_count() const noexcept { return _bucket_count; }

private:
    bucket*     _buckets      = nullptr;
    std::size_t _bucket_count = 0;
    std::size_t _size         = 0;
};

} // namespace vizkit
```

`main.cpp`：

```cpp
#include <vizkit/assoc/chain_map.hpp>
// ...
    vizkit::chain_map<vizkit::name32, int> cm;
    auto put = [&](const char* s, int v) {
        vizkit::name32 n; n.assign(s);
        std::size_t h = 1469598103934665603ull;
        for (const char* p = s; *p; ++p) { h ^= (unsigned char)*p; h *= 1099511628211ull; }
        cm.insert(h, n, v);
    };
    put("apple", 3);
    put("banana", 7);
    put("cherry", 1);
    put("date", 9);
    put("elderberry", 4);
```

## 26.2 なぜ既存の要素では書けないのか

構造を整理します。

```
_buckets ─┬─ [0] head ──▶ node("banana", 7) ──▶ node("date", 9) ──▶ null
          ├─ [1] head ──▶ null
          ├─ [2] head ──▶ null
          ├─ [3] head ──▶ node("apple", 3) ──▶ null
          ├─ [4] head ──▶ null
          ├─ [5] head ──▶ node("cherry", 1) ──▶ null
          ├─ [6] head ──▶ node("elderberry", 4) ──▶ null
          └─ [7] head ──▶ null
```

- `ArrayItems` ―― バケット配列は連続だが、**要素はバケットそのものではなくノードの中にある**
- `IndexListItems` ―― `$i` 番目の要素を式で書けない。「i 番目の要素がどのバケットの何番目にあるか」は、辿ってみないと分からない
- `LinkedListItems` ―― リストは 8 本ある。1本しか辿れない
- `TreeItems` ―― 木ではない

**外側の配列を回りながら、内側のリストを辿る。** 二重の走査が要ります。これができるのは `CustomListItems` だけです。

## 26.3 二重ループを書く

```xml
<Type Name="vizkit::chain_map&lt;*,*&gt;">
  <DisplayString>{{ size={_size}, buckets={_bucket_count} }}</DisplayString>
  <Expand>
    <Item Name="[size]"         ExcludeView="simple">_size</Item>
    <Item Name="[bucket_count]" ExcludeView="simple">_bucket_count</Item>

    <CustomListItems>
      <Variable Name="b" InitialValue="0" />
      <Variable Name="p" InitialValue="_buckets[0].head" />
      <Size>_size</Size>

      <Loop Condition="b &lt; (int)_bucket_count">
        <Exec>p = _buckets[b].head</Exec>
        <Loop Condition="p != nullptr">
          <Item Name="[{p->key}]">p->value</Item>
          <Exec>p = p->next</Exec>
        </Loop>
        <Exec>b++</Exec>
      </Loop>
    </CustomListItems>
  </Expand>
</Type>
```

```
▷ cm                 { size=5, buckets=8 }        vizkit::chain_map<vizkit::name32,int>
     [size]          5
     [bucket_count]  8
     ["banana"]      7
     ["date"]        9
     ["apple"]       3
     ["cherry"]      1
     ["elderberry"]  4
   ▷ [Raw View]      {_buckets=... }
```

**第1章 1.3節で先出しした表示に到達しました。**

C++ で書けば、こうです。

```cpp
for (size_t b = 0; b < _bucket_count; ++b) {
    node* p = _buckets[b].head;
    while (p != nullptr) {
        yield { p->key, p->value };
        p = p->next;
    }
}
```

Natvis の XML は冗長ですが、**構造は完全に対応しています**。難しいのは XML の書き方ではなく、「イテレータの内部状態を、基本型とポインタの組に分解する」という発想のほうです（第23章 23.4節）。

### 変数の初期値に注意

```xml
<Variable Name="p" InitialValue="_buckets[0].head" />
```

`p` の初期値に `_buckets[0].head` を使っているのは、**変数の型を推論させるため**です。`InitialValue="nullptr"` と書くと、型が定まらず後の代入で失敗することがあります。

実際の値はループの先頭で `<Exec>p = _buckets[b].head</Exec>` によって上書きされるので、初期値そのものは意味を持ちません。**「型を決めるためのダミー初期値」**という書き方は、`CustomListItems` でよく使う技法です。

ただし `_buckets` が NULL の場合、この初期値の評価に失敗します。ガードを入れておきます。

```xml
<DisplayString Condition="_buckets == nullptr">[unallocated]</DisplayString>
```

`DisplayString` で捕まえても `CustomListItems` は実行されるので、外側のループ条件でも守ります。

```xml
<Loop Condition="_buckets != nullptr &amp;&amp; b &lt; (int)_bucket_count">
```

## 26.4 バケットごとに見るビュー

フラットな一覧が既定ですが、ハッシュマップのデバッグでは「どのバケットに何が入っているか」を見たいことがあります。衝突が偏っていないかを確認するためです。

バケット型に Natvis を書き、バケット配列を `ArrayItems` で並べます。

```xml
<Type Name="vizkit::chain_map&lt;*,*&gt;::bucket">
  <DisplayString Condition="head == nullptr">·</DisplayString>
  <DisplayString>{head->key} ...</DisplayString>
  <Expand>
    <LinkedListItems>
      <HeadPointer>head</HeadPointer>
      <NextPointer>next</NextPointer>
      <ValueNode>this</ValueNode>
    </LinkedListItems>
  </Expand>
</Type>

<Type Name="vizkit::chain_map&lt;*,*&gt;::node">
  <DisplayString>{key} =&gt; {value}</DisplayString>
  <Expand>
    <Item Name="key">key</Item>
    <Item Name="value">value</Item>
    <Item Name="hash" IncludeView="detail">hash,X</Item>
  </Expand>
</Type>
```

```xml
<!-- chain_map の Expand に追加 -->
<Synthetic Name="[buckets]" IncludeView="detail" Condition="_buckets != nullptr">
  <DisplayString>{_bucket_count} buckets</DisplayString>
  <Expand>
    <ArrayItems>
      <Size>_bucket_count</Size>
      <ValuePointer>_buckets</ValuePointer>
    </ArrayItems>
  </Expand>
</Synthetic>
```

```
cm,view(detail)
  └ [buckets]        8 buckets
       ▷ [0]         "banana" ...
            [0]      "banana" => 7
            [1]      "date" => 9
         [1]         ·
         [2]         ·
       ▷ [3]         "apple" ...
            [0]      "apple" => 3
         [4]         ·
       ▷ [5]         "cherry" ...
         [6]         "elderberry" ...
         [7]         ·
```

**衝突しているバケットが一目で分かります。** `[0]` に 2 件入っていること、`[1]` `[2]` `[4]` `[7]` が空であることが見えます。

ここで使っているのは、第5部の `LinkedListItems` です。**バケット単体を見れば、それは単なる単方向リスト**なので、専用要素で書けます。二重ループが必要なのは「全バケットを横断して1つの列にする」場合だけです。

`ValueNode` に `this` を書き、ノード型の `DisplayString` で `key => value` に整えるのは、第21章 21.3節で確立した連想コンテナの定石です。

## 26.5 空バケットを速く飛ばす ―― `__findnonnull`

バケット数が大きく、要素が少ない場合（負荷率が低い場合）、二重ループの外側は空バケットばかりを回ることになります。1024 バケットに 10 件なら、1014 回は空振りです。

Visual Studio のデバッガには、**`__findnonnull` という組み込み関数**があります。ポインタ配列の中から最初の非 NULL を探すものです。

公式ドキュメントの `CAtlMap` の例が、まさにこれを使っています。

```xml
<Exec>iBucketIncrement = __findnonnull(m_ppBins + iBucket, m_nBins - iBucket)</Exec>
<Break Condition="iBucketIncrement == -1" />
<Exec>iBucket += iBucketIncrement</Exec>
```

`__findnonnull(先頭ポインタ, 個数)` が、最初の非 NULL 要素までのオフセットを返します。見つからなければ `-1` です。

第23章 23.4節で「`Exec` では関数を呼べない」と述べましたが、**「C++ 式エバリュエーターがサポートするデバッガ組み込み関数」は例外**でした。`__findnonnull` はその1つです。

`vizkit::chain_map` の `bucket` は `node*` を1つ持つだけの構造体なので、レイアウト上は `node*` の配列と同じです。適用できます。

```xml
<CustomListItems>
  <Variable Name="b"    InitialValue="0" />
  <Variable Name="step" InitialValue="0" />
  <Variable Name="p"    InitialValue="_buckets[0].head" />
  <Size>_size</Size>
  <Exec>p = nullptr</Exec>

  <Loop>
    <If Condition="p == nullptr">
      <Exec>step = __findnonnull((node**)(_buckets + b), (int)_bucket_count - b)</Exec>
      <Break Condition="step == -1" />
      <Exec>b += step</Exec>
      <Exec>p = _buckets[b].head</Exec>
      <Exec>b++</Exec>
    </If>
    <Item Name="[{p->key}]">p->value</Item>
    <Exec>p = p->next</Exec>
  </Loop>
</CustomListItems>
```

構造は `CAtlMap` の例と同じです。「チェーンを使い切ったら次の非空バケットへ跳ぶ」という形になっています。

**ただし、これは最適化です。** まずは 26.3 節の素直な二重ループで正しく動くものを書き、性能が問題になってから置き換えてください。`__findnonnull` はデバッガ実装に依存する機能なので、環境によっては使えない可能性もあります。動かない場合は素直な二重ループに戻せば済みます。

## 26.6 統計を出す

ハッシュマップのデバッグで最も知りたいのは、分布の偏りです。第25章 25.5節と同じパターンで計算します。

```xml
<Synthetic Name="[stats]" IncludeView="detail" Condition="_buckets != nullptr">
  <DisplayString>load={(float)_size / (float)_bucket_count}</DisplayString>
  <Expand>
    <Item Name="[load factor]">(float)_size / (float)_bucket_count</Item>
    <CustomListItems>
      <Variable Name="b"       InitialValue="0" />
      <Variable Name="p"       InitialValue="_buckets[0].head" />
      <Variable Name="len"     InitialValue="0" />
      <Variable Name="longest" InitialValue="0" />
      <Variable Name="used"    InitialValue="0" />
      <Variable Name="guard"   InitialValue="0" />

      <Loop Condition="b &lt; (int)_bucket_count">
        <Break Condition="guard &gt; 1000000" />
        <Exec>p = _buckets[b].head</Exec>
        <Exec>len = 0</Exec>
        <If Condition="p != nullptr"><Exec>used++</Exec></If>
        <Loop Condition="p != nullptr">
          <Exec>len++</Exec>
          <Exec>p = p->next</Exec>
          <Exec>guard++</Exec>
        </Loop>
        <If Condition="len &gt; longest"><Exec>longest = len</Exec></If>
        <Exec>b++</Exec>
      </Loop>

      <Item Name="[longest chain]">longest</Item>
      <Item Name="[used buckets]">used</Item>
      <Item Name="[empty buckets]">(int)_bucket_count - used</Item>
    </CustomListItems>
  </Expand>
</Synthetic>
```

```
cm,view(detail)
  └ [stats]            load=0.625
       [load factor]   0.625
       [longest chain] 2
       [used buckets]  4
       [empty buckets] 4
```

**`[longest chain]` が要素数に近づいていたら、ハッシュ関数が壊れています。** 全要素が1本のチェーンに繋がっている、つまり `chain_map` が連結リストとして動作している状態です。これはハッシュマップの典型的な性能事故で、Natvis なしでは気づきにくいものです。

内側のループにも `guard` を効かせている点に注意してください。チェーンが循環していると、内側で無限ループになります。**二重ループでは、内側にこそ保険が要ります。**

## 26.7 _size を信用しない検証

第19章 19.5節で `forward_list` に対してやったことを、ここでも用意します。実際に辿った件数と `_size` を突き合わせます。

```xml
<Synthetic Name="[verify]" IncludeView="detail" Condition="_buckets != nullptr">
  <DisplayString>_size と実際の件数を突き合わせる</DisplayString>
  <Expand>
    <CustomListItems>
      <Variable Name="b"     InitialValue="0" />
      <Variable Name="p"     InitialValue="_buckets[0].head" />
      <Variable Name="n"     InitialValue="0" />
      <Variable Name="bad"   InitialValue="0" />
      <Variable Name="guard" InitialValue="0" />

      <Loop Condition="b &lt; (int)_bucket_count">
        <Break Condition="guard &gt; 1000000" />
        <Exec>p = _buckets[b].head</Exec>
        <Loop Condition="p != nullptr">
          <Exec>n++</Exec>
          <If Condition="p->hash % _bucket_count != b"><Exec>bad++</Exec></If>
          <Exec>p = p->next</Exec>
          <Exec>guard++</Exec>
        </Loop>
        <Exec>b++</Exec>
      </Loop>

      <Item Name="[counted]">n</Item>
      <Item Name="[_size]">(int)_size</Item>
      <Item Name="[misplaced nodes]">bad</Item>
    </CustomListItems>
  </Expand>
</Synthetic>
```

`[misplaced nodes]` は、**「ハッシュ値から計算されるバケット番号と、実際に置かれているバケット番号が食い違うノード」**の数です。リハッシュのバグを検出できます。0 でなければ、そのマップは壊れています。

第15章から続けてきた「不変条件を Natvis に書く」方針が、ここで**ハッシュマップ全体の整合性検証**にまで届きました。`CustomListItems` があれば、この水準の検証をデバッガ上で実行できます。

## 26.8 この章のまとめ

- バケット配列 × チェーンという二重構造は、**`CustomListItems` の二重ループでしか書けない**。
- 外側ループの先頭で `<Exec>p = _buckets[b].head</Exec>` と内側ポインタを初期化し、内側ループでチェーンを辿る。C++ の二重ループとそのまま対応する。
- `<Variable>` の `InitialValue` は**型を推論させるためのダミー**でよい。`nullptr` ではなく実際のメンバー式を書く。
- **バケット単体は単なる単方向リスト**なので、`LinkedListItems` で書ける。バケット型に Natvis を当てれば、衝突の分布が視覚的に分かる。
- `__findnonnull(ptr, count)` はデバッガ組み込み関数で、空バケットを高速に飛ばせる。**ただし最適化なので、素直な二重ループが動いてから導入する。**
- ループを回して統計を出すパターンで、`[longest chain]` `[used buckets]` などプログラムに存在しない診断値を作れる。**`[longest chain]` が要素数に近ければハッシュ関数が壊れている。**
- **二重ループでは内側にこそ `guard` が要る。** チェーンの循環で無限ループになる。
- `p->hash % _bucket_count != b` の検出で、リハッシュのバグを見つけられる。

## 26.9 演習

1. 26.3節の二重ループから `<Exec>p = _buckets[b].head</Exec>` を削除し、何が起きるか観察してください。外側ループの先頭で内側の状態をリセットする必要性が分かります。
2. `put()` のハッシュ計算を `h = 0`（全要素が同じバケットに入る）に変え、`[longest chain]` と `[buckets]` の表示がどうなるか確認してください。ハッシュ関数の事故が Natvis でどう見えるかの実験です。
3. `<Size>_size</Size>` を削除し、`[verify]` の `[counted]` と表示件数が一致するか確認してください。
4. デバッガから任意のノードの `hash` を書き換え、`[misplaced nodes]` が 1 になることを確認してください。
5. 26.5節の `__findnonnull` 版を実装し、バケット数を 4096、要素数を 10 にした状態で、素直な二重ループ版との応答速度を比べてください。環境によっては `__findnonnull` が使えないので、その場合は診断メッセージを確認してください。
