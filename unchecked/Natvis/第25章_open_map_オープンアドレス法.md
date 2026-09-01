# 第25章 open_map ―― オープンアドレス法と要素のスキップ

ハッシュマップに入ります。まずはオープンアドレス法 ―― 要素をバケット配列に直接置き、衝突したら別のスロットを探す方式です。

この方式の Natvis で本質的なのは、**表示すべきでないスロットを飛ばす**ことです。`ArrayItems` にはその能力がありません。

## 25.1 型を作る

`vizkit\include\vizkit\assoc\open_map.hpp`

```cpp
#pragma once

#include <cstddef>
#include <cstdint>
#include <new>

namespace vizkit {

enum class slot_state : std::uint8_t {
    empty     = 0,   // 一度も使われていない
    occupied  = 1,   // 有効な要素が入っている
    tombstone = 2,   // 削除済み（探索は通過する）
};

// 線形探索によるオープンアドレス法ハッシュマップ。
template <class K, class V>
class open_map {
public:
    struct slot {
        slot_state  state = slot_state::empty;
        std::size_t hash  = 0;
        K           key{};
        V           value{};
    };

    explicit open_map(std::size_t cap = 8) { rehash(cap); }
    ~open_map() { ::operator delete(static_cast<void*>(_slots)); }

    open_map(const open_map&) = delete;
    open_map& operator=(const open_map&) = delete;

    void insert(std::size_t h, const K& k, const V& v) {
        if ((_size + _tombstones) * 4 >= _cap * 3) rehash(_cap * 2);
        std::size_t i = h & (_cap - 1);
        while (_slots[i].state == slot_state::occupied) {
            if (_slots[i].hash == h && _slots[i].key == k) { _slots[i].value = v; return; }
            i = (i + 1) & (_cap - 1);
        }
        if (_slots[i].state == slot_state::tombstone) --_tombstones;
        _slots[i] = slot{ slot_state::occupied, h, k, v };
        ++_size;
    }

    void erase(std::size_t h, const K& k) {
        std::size_t i = h & (_cap - 1);
        while (_slots[i].state != slot_state::empty) {
            if (_slots[i].state == slot_state::occupied &&
                _slots[i].hash == h && _slots[i].key == k) {
                _slots[i].state = slot_state::tombstone;
                --_size; ++_tombstones; return;
            }
            i = (i + 1) & (_cap - 1);
        }
    }

    [[nodiscard]] std::size_t size() const noexcept { return _size; }

private:
    void rehash(std::size_t nc);   // 実装省略

    slot*       _slots      = nullptr;
    std::size_t _cap        = 0;
    std::size_t _size       = 0;
    std::size_t _tombstones = 0;
};

} // namespace vizkit
```

`main.cpp`：

```cpp
#include <vizkit/assoc/open_map.hpp>
// ...
    vizkit::open_map<int, vizkit::name32> om;
    auto put = [&](int k, const char* s) {
        vizkit::name32 n; n.assign(s);
        om.insert(static_cast<std::size_t>(k) * 2654435761u, k, n);
    };
    put(1, "one");
    put(2, "two");
    put(3, "three");
    om.erase(static_cast<std::size_t>(2) * 2654435761u, 2);   // 墓石を作る
    put(4, "four");
```

## 25.2 ArrayItems では読めない

素直に `ArrayItems` を当てるとどうなるか、まず見ておきます。

```xml
<!-- 不十分 -->
<ArrayItems>
  <Size>_cap</Size>
  <ValuePointer>_slots</ValuePointer>
</ArrayItems>
```

```
▷ om             { size=3 }
   ▷ [0]         {state=empty (0) hash=0 key=0 value="" }
   ▷ [1]         {state=empty (0) ... }
   ▷ [2]         {state=occupied (1) hash=... key=3 value="three" }
   ▷ [3]         {state=tombstone (2) hash=... key=2 value="two" }
   ▷ [4]         {state=empty (0) ... }
   ▷ [5]         {state=occupied (1) ... key=1 value="one" }
   ▷ [6]         {state=empty (0) ... }
   ▷ [7]         {state=occupied (1) ... key=4 value="four" }
```

要素は 3 つなのに 8 行が並び、そのうち有効なのは 3 行だけ。しかも `[3]` は削除済みなのに値が残っています。**`size=3` という表示と、目に見える行数が一致しません。**

要素数が 1024 スロットになれば、有効な要素を探すためにスクロールし続けることになります。

`ArrayItems` にできないのは、**条件によって要素を飛ばすこと**です。列挙するかしないかを制御する手段がありません。

## 25.3 CustomListItems でスキップする

```xml
<Type Name="vizkit::open_map&lt;*,*&gt;">
  <DisplayString>{{ size={_size} }}</DisplayString>
  <Expand>
    <Item Name="[size]"     ExcludeView="simple">_size</Item>
    <Item Name="[capacity]" ExcludeView="simple">_cap</Item>

    <CustomListItems>
      <Variable Name="i" InitialValue="0" />
      <Size>_size</Size>
      <Loop Condition="i &lt; (int)_cap">
        <If Condition="_slots[i].state == vizkit::slot_state::occupied">
          <Item Name="[{_slots[i].key}]">_slots[i].value</Item>
        </If>
        <Exec>i++</Exec>
      </Loop>
    </CustomListItems>
  </Expand>
</Type>
```

```
▷ om             { size=3 }
     [size]      3
     [capacity]  8
     [3]         "three"
     [1]         "one"
     [4]         "four"
```

**有効な 3 要素だけが並びました。** 順序はハッシュ値によるので、挿入順でもキー順でもありません。それがハッシュマップの性質です。

構造を確認しておきます。

- **ループはスロット全体を回る**（`i < _cap`）
- **`Item` は `If` の中にある**（第24章 24.5節）
- **`Size` は要素数** ―― ループの回数（`_cap`）ではない

最後の点が重要です。`<Size>` は「最終的に何件出力されるか」を宣言するものであり、ループが何回回るかとは無関係です。ここを `_cap` にすると、デバッガは 8 件出ると思って待ち、実際には 3 件しか出ないので不整合が起きます。

**`<Size>` は出力件数。** `CustomListItems` を書くときに最も間違えやすい点です。

## 25.4 墓石と空スロットを見分ける

削除済みスロットの数は、ハッシュマップの性能に直結します。墓石が増えると探索が長くなるからです。既定ビューには出さず、詳細ビューで見せます。

```xml
<Synthetic Name="[storage]" IncludeView="detail">
  <DisplayString>{_size} used, {_tombstones} tombstones, {_cap} slots</DisplayString>
  <Expand>
    <Item Name="[load factor]">(float)(_size + _tombstones) / (float)_cap</Item>
    <Item Name="[tombstones]">_tombstones</Item>
    <ArrayItems>
      <Size>_cap</Size>
      <ValuePointer>_slots</ValuePointer>
    </ArrayItems>
  </Expand>
</Synthetic>
```

スロット型にも Natvis を書いて、状態を一目で分かるようにします。

```xml
<Type Name="vizkit::open_map&lt;*,*&gt;::slot">
  <DisplayString Condition="state == vizkit::slot_state::empty">·</DisplayString>
  <DisplayString Condition="state == vizkit::slot_state::tombstone">× (was {key})</DisplayString>
  <DisplayString>{key} =&gt; {value}</DisplayString>
  <Expand>
    <Item Name="state">state,en</Item>
    <Item Name="hash" IncludeView="detail">hash,X</Item>
    <Item Name="key"   Condition="state != vizkit::slot_state::empty">key</Item>
    <Item Name="value" Condition="state == vizkit::slot_state::occupied">value</Item>
  </Expand>
</Type>
```

```
om,view(detail)
  └ [storage]      3 used, 1 tombstones, 8 slots
       [load factor]  0.5
       [tombstones]   1
       [0]            ·
       [1]            ·
       [2]            3 => "three"
       [3]            × (was 2)
       [4]            ·
       [5]            1 => "one"
       [6]            ·
       [7]            4 => "four"
```

**バケット配列の使われ方が視覚的に分かるようになりました。** クラスタリング（連続したスロットが埋まる現象）が起きているかどうかも一目です。

`·` と `×` という短い記号にしているのは、第22章 22.3節と同じ理由です。**一覧性を上げるには、行の先頭に短く違いを出す**のが効きます。

## 25.5 統計を計算する

墓石数はメンバーとして持っていますが、「最長の探索距離」のような値は持っていません。デバッグ時にだけ知りたい値です。

第23章 23.9節の「ループを回して1件出す」パターンで計算します。

```xml
<Synthetic Name="[stats]" IncludeView="detail">
  <DisplayString>探索距離の統計</DisplayString>
  <Expand>
    <CustomListItems>
      <Variable Name="i"       InitialValue="0" />
      <Variable Name="run"     InitialValue="0" />
      <Variable Name="longest" InitialValue="0" />
      <Loop Condition="i &lt; (int)_cap">
        <If Condition="_slots[i].state == vizkit::slot_state::empty">
          <Exec>run = 0</Exec>
        </If>
        <Else>
          <Exec>run++</Exec>
          <If Condition="run &gt; longest">
            <Exec>longest = run</Exec>
          </If>
        </Else>
        <Exec>i++</Exec>
      </Loop>
      <Item Name="[longest run]">longest</Item>
    </CustomListItems>
  </Expand>
</Synthetic>
```

`[longest run]` は、連続して埋まっているスロットの最大長です。これが大きいほど、探索が長くなります。ハッシュ関数の質を評価する目安になります。

**この値はプログラムのどこにも存在しません。** Natvis がデバッグ時にだけ計算しています。第11章 11.5節で述べた「Natvis は診断ツールでもある」という性格が、`CustomListItems` によって本格的になったことになります。

## 25.6 制御バイト方式への応用

近年のハッシュマップ（SwissTable 系）は、状態を要素と分離した**制御バイト配列**に持ちます。

```cpp
template <class K, class V>
class swiss_map {
    std::int8_t* _ctrl;    // 各スロットの状態＋ハッシュ上位ビット
    slot*        _slots;   // 要素本体
    std::size_t  _cap;
    std::size_t  _size;
};
```

制御バイトの規約はおおむね次のようなものです。

- 負の値（最上位ビットが 1）＝ 空または削除済み
- 0 以上 ＝ 使用中（値はハッシュの上位 7 ビット）

Natvis の書き方は本章とほとんど同じで、判定条件が変わるだけです。

```xml
<CustomListItems>
  <Variable Name="i" InitialValue="0" />
  <Size>_size</Size>
  <Loop Condition="i &lt; (int)_cap">
    <If Condition="_ctrl[i] &gt;= 0">
      <Item Name="[{_slots[i].key}]">_slots[i].value</Item>
    </If>
    <Exec>i++</Exec>
  </Loop>
</CustomListItems>
```

**判定の場所が別配列になっても、構造は変わりません。** `CustomListItems` の汎用性がここに現れています。`ArrayItems` は「要素の並び」しか扱えませんが、`CustomListItems` は「2つの配列を突き合わせる」ことができます。

制御バイト方式で追加したくなる診断は、状態の内訳です。

```xml
<CustomListItems>
  <Variable Name="i"     InitialValue="0" />
  <Variable Name="full"  InitialValue="0" />
  <Variable Name="del"   InitialValue="0" />
  <Variable Name="empty" InitialValue="0" />
  <Loop Condition="i &lt; (int)_cap">
    <If Condition="_ctrl[i] &gt;= 0"><Exec>full++</Exec></If>
    <Elseif Condition="_ctrl[i] == -2"><Exec>del++</Exec></Elseif>
    <Else><Exec>empty++</Exec></Else>
    <Exec>i++</Exec>
  </Loop>
  <Item Name="[full]">full</Item>
  <Item Name="[deleted]">del</Item>
  <Item Name="[empty]">empty</Item>
</CustomListItems>
```

`<Elseif>` を使った3分岐です。**ループの外に `Item` を3つ並べれば、3行の統計が出ます。**

## 25.7 現時点の open_map の Natvis

```xml
  <Type Name="vizkit::open_map&lt;*,*&gt;">
    <DisplayString Condition="_slots == nullptr">[unallocated]</DisplayString>
    <DisplayString Condition="_size == 0">[empty] (capacity={_cap})</DisplayString>
    <DisplayString>{{ size={_size} }}</DisplayString>

    <Expand>
      <Item Name="[size]"     ExcludeView="simple">_size</Item>
      <Item Name="[capacity]" ExcludeView="simple">_cap</Item>

      <Synthetic Name="[storage]" IncludeView="detail" Condition="_slots != nullptr">
        <DisplayString>{_size} used, {_tombstones} tombstones, {_cap} slots</DisplayString>
        <Expand>
          <Item Name="[load factor]">(float)(_size + _tombstones) / (float)_cap</Item>
          <Item Name="[tombstones]">_tombstones</Item>
          <ArrayItems>
            <Size>_cap</Size>
            <ValuePointer>_slots</ValuePointer>
          </ArrayItems>
        </Expand>
      </Synthetic>

      <Synthetic Name="[stats]" IncludeView="detail" Condition="_slots != nullptr">
        <DisplayString>探索距離の統計</DisplayString>
        <Expand>
          <CustomListItems>
            <Variable Name="i"       InitialValue="0" />
            <Variable Name="run"     InitialValue="0" />
            <Variable Name="longest" InitialValue="0" />
            <Loop Condition="i &lt; (int)_cap">
              <If Condition="_slots[i].state == vizkit::slot_state::empty">
                <Exec>run = 0</Exec>
              </If>
              <Else>
                <Exec>run++</Exec>
                <If Condition="run &gt; longest"><Exec>longest = run</Exec></If>
              </Else>
              <Exec>i++</Exec>
            </Loop>
            <Item Name="[longest run]">longest</Item>
          </CustomListItems>
        </Expand>
      </Synthetic>

      <CustomListItems>
        <Variable Name="i"     InitialValue="0" />
        <Variable Name="guard" InitialValue="0" />
        <Size>_size</Size>
        <Loop Condition="i &lt; (int)_cap">
          <Break Condition="guard &gt; 1000000" />
          <If Condition="_slots[i].state == vizkit::slot_state::occupied">
            <Item Name="[{_slots[i].key}]">_slots[i].value</Item>
          </If>
          <Exec>i++</Exec>
          <Exec>guard++</Exec>
        </Loop>
      </CustomListItems>
    </Expand>
  </Type>

  <Type Name="vizkit::open_map&lt;*,*&gt;::slot">
    <DisplayString Condition="state == vizkit::slot_state::empty">·</DisplayString>
    <DisplayString Condition="state == vizkit::slot_state::tombstone">× (was {key})</DisplayString>
    <DisplayString>{key} =&gt; {value}</DisplayString>
    <Expand>
      <Item Name="state">state,en</Item>
      <Item Name="hash" IncludeView="detail">hash,X</Item>
      <Item Name="key"   Condition="state != vizkit::slot_state::empty">key</Item>
      <Item Name="value" Condition="state == vizkit::slot_state::occupied">value</Item>
    </Expand>
  </Type>
```

## 25.8 この章のまとめ

- `ArrayItems` には**要素を飛ばす能力がない**。空スロットや墓石を含む配列には使えない。
- `CustomListItems` では、**ループはスロット全体を回り、`Item` を `If` の中に置く**。これが「スキップ」の実現形。
- **`<Size>` は出力件数であって、ループの回数ではない。** `_cap` ではなく `_size` を書く。
- 有効要素の一覧（既定ビュー）と、スロット配列そのもの（詳細ビュー）を分けると、両方の見方ができる。
- スロット型に Natvis を当て、`·` `×` のような**短い記号**で状態を示すと、クラスタリングが視覚的に分かる。
- ループを回して統計を計算し、`Item` を1〜数件出すパターンで、**プログラムのどこにも存在しない診断値**を作れる。
- 制御バイト方式（SwissTable 系）でも構造は同じ。**`CustomListItems` は2つの配列を突き合わせられる**点が `ArrayItems` との本質的な差。

## 25.9 演習

1. `<Size>_size</Size>` を `<Size>_cap</Size>` に変え、表示にどんな不整合が出るか観察してください。
2. `If` の条件を `state != vizkit::slot_state::empty` に変え、墓石も一覧に出るようにしてください。削除済み要素が見えることの利点と欠点を考えてください。
3. `om` に 20 件ほど挿入し、`om,view(detail)` の `[longest run]` がどう変化するか観察してください。ハッシュ関数を `k` そのもの（乗算なし）に変えると、値がどう変わるかも試してください。
4. `slot` の `DisplayString` から `·` の行を削除し、空スロットが `{key} => {value}` として表示されることを確認してください。未初期化領域を表示することの危険が分かります。
5. 25.6節の制御バイト方式の `swiss_map` を実際に実装し、`CustomListItems` がそのまま流用できることを確認してください。
