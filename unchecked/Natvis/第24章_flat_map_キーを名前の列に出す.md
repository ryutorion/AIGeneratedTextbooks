# 第24章 flat_map ―― キーを名前の列に出す

連想コンテナで最も見たいのは「どのキーにどの値が入っているか」です。

第21章の `bst` では `[0] 20 => "twenty"` という表示が精一杯でした。`TreeItems` にはキーを名前の列に出す手段がないためです。`CustomListItems` を使うと、それができます。

この章では最も単純な連想コンテナ ―― ソート済み配列 ―― で、その手法を確立します。

## 24.1 型を作る

`vizkit\include\vizkit\assoc\flat_map.hpp`

```cpp
#pragma once

#include <cstddef>
#include <new>
#include <utility>

namespace vizkit {

// キー昇順にソートされた配列で連想を表現する。
// 検索は二分探索、挿入は O(n)。キャッシュ効率がよい。
template <class K, class V>
class flat_map {
public:
    struct entry {
        K key{};
        V value{};
    };

    flat_map() = default;
    ~flat_map() { clear(); ::operator delete(static_cast<void*>(_data)); }

    flat_map(const flat_map&) = delete;
    flat_map& operator=(const flat_map&) = delete;

    void insert(const K& k, const V& v) {
        std::size_t pos = lower_bound(k);
        if (pos < _size && !(k < _data[pos].key)) { _data[pos].value = v; return; }
        if (_size == _cap) grow();
        for (std::size_t i = _size; i > pos; --i) _data[i] = _data[i - 1];
        _data[pos] = entry{ k, v };
        ++_size;
    }

    [[nodiscard]] std::size_t size() const noexcept { return _size; }

private:
    [[nodiscard]] std::size_t lower_bound(const K& k) const noexcept {
        std::size_t lo = 0, hi = _size;
        while (lo < hi) { std::size_t mid = (lo + hi) / 2;
                          if (_data[mid].key < k) lo = mid + 1; else hi = mid; }
        return lo;
    }

    void clear() noexcept { for (std::size_t i = 0; i < _size; ++i) _data[i].~entry(); _size = 0; }

    void grow() {
        std::size_t nc = _cap == 0 ? 4 : _cap * 2;
        entry* p = static_cast<entry*>(::operator new(sizeof(entry) * nc));
        for (std::size_t i = 0; i < _size; ++i) { new (p + i) entry(std::move(_data[i])); _data[i].~entry(); }
        ::operator delete(static_cast<void*>(_data));
        _data = p; _cap = nc;
    }

    entry*      _data = nullptr;
    std::size_t _size = 0;
    std::size_t _cap  = 0;
};

} // namespace vizkit
```

`main.cpp`：

```cpp
#include <vizkit/assoc/flat_map.hpp>
// ...
    vizkit::flat_map<int, vizkit::name32> fm;
    auto put = [&](int k, const char* s) {
        vizkit::name32 n; n.assign(s); fm.insert(k, n);
    };
    put(30, "thirty");
    put(10, "ten");
    put(20, "twenty");

    vizkit::flat_map<vizkit::name32, int> fm_str;   // キーが文字列（演習用）
```

## 24.2 まず ArrayItems 版を書く

`flat_map` の要素は連続した配列です。**`ArrayItems` で書けます。**

```xml
<Type Name="vizkit::flat_map&lt;*,*&gt;">
  <DisplayString>{{ size={_size} }}</DisplayString>
  <Expand>
    <Item Name="[size]"     ExcludeView="simple">_size</Item>
    <Item Name="[capacity]" ExcludeView="simple">_cap</Item>
    <ArrayItems>
      <Size>_size</Size>
      <ValuePointer>_data</ValuePointer>
    </ArrayItems>
  </Expand>
</Type>

<Type Name="vizkit::flat_map&lt;*,*&gt;::entry">
  <DisplayString>{key} =&gt; {value}</DisplayString>
  <Expand>
    <Item Name="key">key</Item>
    <Item Name="value">value</Item>
  </Expand>
</Type>
```

```
▷ fm             { size=3 }
     [size]      3
     [capacity]  4
   ▷ [0]         10 => "ten"
   ▷ [1]         20 => "twenty"
   ▷ [2]         30 => "thirty"
```

第21章の `bst` と同じ水準の表示です。**そしてこれは、`std::map` の表示水準と同じ**でもあります。

まず専用要素で書けるところまで書く。これが第23章 23.1節で立てた原則です。

## 24.3 CustomListItems でキーを名前の列に出す

さらに上を目指します。`CustomListItems` の `<Item>` には `Name` 属性があり、そこには式を埋め込めます（第10章 10.2節）。

```xml
<Type Name="vizkit::flat_map&lt;*,*&gt;">
  <DisplayString>{{ size={_size} }}</DisplayString>
  <Expand>
    <Item Name="[size]"     ExcludeView="simple">_size</Item>
    <Item Name="[capacity]" ExcludeView="simple">_cap</Item>
    <CustomListItems>
      <Variable Name="i" InitialValue="0" />
      <Size>_size</Size>
      <Loop Condition="i &lt; (int)_size">
        <Item Name="[{_data[i].key}]">_data[i].value</Item>
        <Exec>i++</Exec>
      </Loop>
    </CustomListItems>
  </Expand>
</Type>
```

```
▷ fm             { size=3 }
     [size]      3
     [capacity]  4
     [10]        "ten"
     [20]        "twenty"
     [30]        "thirty"
```

**キーが名前の列に出ました。** 第1章 1.3節で先出しした `["apple"] 3` という表示に到達しています。

`Name` の中の `{_data[i].key}` は、第10章で学んだ「`Name` 属性の中では `{式}` が使える」という性質そのものです。違いは、その式の中で `CustomListItems` のループ変数 `i` を参照できることです。

### `(int)` キャストについて

`<Loop Condition="i &lt; (int)_size">` でキャストしています。`i` は `InitialValue="0"` から `int` と推論され、`_size` は `std::size_t`（符号なし 64 ビット）です。符号の異なる比較は環境によって挙動が読みにくくなるので、明示的に揃えておくのが安全です。

`<Variable Name="i" InitialValue="(int)0" />` のように初期値側で型を明示する書き方もあります。どちらでも構いませんが、**符号付きと符号なしを混ぜない**ことを意識してください。`CustomListItems` のループ変数は、この種のバグが最も出やすい場所です。

## 24.4 キーが文字列の場合

`flat_map<name32, int>` のように、キーが構造体の場合はどうなるでしょうか。

```xml
<Item Name="[{_data[i].key}]">_data[i].value</Item>
```

`name32` には Natvis を書いてあるので（第9章）、`{_data[i].key}` は `"frame_buffer"` と展開されます。

```
     ["alpha"]      1
     ["beta"]       2
```

引用符が付いています。名前の列としては、引用符がないほうが読みやすいこともあります。第6章 6.4節の「引用符なし」指定子を使います。

```xml
<Item Name="[{_data[i].key._buf,[_data[i].key._len]s8b}]">_data[i].value</Item>
```

長すぎます。`Intrinsic` に逃がしましょう……と言いたいところですが、`Intrinsic` はコンテナの文脈で定義されるため、ループ変数 `i` を渡せません。

現実的な解決策は、**キー型の側にビューを用意する**ことです。

```xml
<Type Name="vizkit::name32" IncludeView="bare">
  <DisplayString>{_buf,[_len]s8b}</DisplayString>
</Type>
```

```xml
<Item Name="[{_data[i].key,view(bare)}]">_data[i].value</Item>
```

```
     [alpha]       1
     [beta]        2
```

第13章 13.6節で「ビューは再帰的な表示を制御する道具でもある」と述べました。ここがその実例です。**表示のバリエーションはキー型の側に持たせ、コンテナ側からはビュー名で選ぶ。** 複数の連想コンテナで同じキー型を使うなら、この形が最も保守しやすくなります。

## 24.5 Item に Condition を書かない

`CustomListItems` の `<Item>` については、実装によって `Condition` 属性が効かないことが報告されています。**分岐が必要なら `<If>` で `<Item>` を囲むのが確実**です。

```xml
<!-- 推奨しない -->
<Item Name="..." Condition="...">...</Item>

<!-- 確実 -->
<If Condition="...">
  <Item Name="...">...</Item>
</If>
```

`If` で囲む形は、`CustomListItems` の手続き的な性格にも合っています。次章の「空スロットを飛ばす」処理は、まさにこの形になります。

同じ理由で、**表示形式が2通りある場合は、`If` で分けてブロックごと複製する**のが定石です。

```xml
<Loop Condition="i &lt; (int)_size">
  <If Condition="_data[i].value &lt; 0">
    <Item Name="[!{_data[i].key}]">_data[i].value</Item>
  </If>
  <Else>
    <Item Name="[{_data[i].key}]">_data[i].value</Item>
  </Else>
  <Exec>i++</Exec>
</Loop>
```

冗長ですが、これが `CustomListItems` の作法です。

## 24.6 どちらを既定にするか

`ArrayItems` 版と `CustomListItems` 版の両方が書けました。どちらを既定にすべきでしょうか。

| | `ArrayItems` 版 | `CustomListItems` 版 |
|---|---|---|
| 表示 | `[0] 10 => "ten"` | `[10] "ten"` |
| 速度 | 速い | 遅い（要素ごとに複数の式を評価） |
| キーで探しやすいか | 添字順に読む必要がある | **キーが目に入る** |
| 要素数が多いとき | 実用的 | 数千件で重くなる |

実務的な結論は、**要素数によって使い分ける**です。

`flat_map` のような「数十〜数百件」を想定した型なら、`CustomListItems` 版を既定にする価値があります。数万件を扱う型なら `ArrayItems` を既定にして、キー表示は別ビューに回します。

```xml
<Expand>
  <ArrayItems IncludeView="fast">
    <Size>_size</Size>
    <ValuePointer>_data</ValuePointer>
  </ArrayItems>

  <CustomListItems ExcludeView="fast">
    <Variable Name="i" InitialValue="0" />
    <Size>_size</Size>
    <Loop Condition="i &lt; (int)_size">
      <Item Name="[{_data[i].key}]">_data[i].value</Item>
      <Exec>i++</Exec>
    </Loop>
  </CustomListItems>
</Expand>
```

既定ではキー表示、`fm,view(fast)` で軽い表示。第13章の3ビュー設計（既定 / `simple` / `detail`）に、性能のための `fast` を足した形です。

## 24.7 二分探索の様子を見せる

`flat_map` の特徴はソート済みであることです。それが崩れていないかを確認できると、デバッグが楽になります。

```xml
<Synthetic Name="[sorted?]" IncludeView="detail" Condition="_size &gt; 1">
  <DisplayString>キー順序の検証</DisplayString>
  <Expand>
    <CustomListItems>
      <Variable Name="i"   InitialValue="1" />
      <Variable Name="bad" InitialValue="-1" />
      <Loop Condition="i &lt; (int)_size">
        <If Condition="bad &lt; 0 &amp;&amp; !(_data[i-1].key &lt; _data[i].key)">
          <Exec>bad = i</Exec>
        </If>
        <Exec>i++</Exec>
      </Loop>
      <If Condition="bad &lt; 0">
        <Item Name="[result]">"ソート済み"</Item>
      </If>
      <Else>
        <Item Name="[!] 順序違反">bad</Item>
      </Else>
    </CustomListItems>
  </Expand>
</Synthetic>
```

第23章 23.9節で使った「ループを回して結果を1件だけ出す」パターンです。第15章以降続けてきた「不変条件を Natvis に書く」方針が、`CustomListItems` によって**全要素にわたる条件にまで拡張された**ことになります。

## 24.8 現時点の flat_map の Natvis

```xml
  <!-- ===== assoc ===== -->

  <Type Name="vizkit::flat_map&lt;*,*&gt;">
    <DisplayString Condition="_size &gt; _cap">[corrupt: size={_size} &gt; capacity={_cap}]</DisplayString>
    <DisplayString Condition="_size == 0">[empty]</DisplayString>
    <DisplayString>{{ size={_size} }}</DisplayString>

    <Expand>
      <Item Name="[size]"     ExcludeView="simple">_size</Item>
      <Item Name="[capacity]" ExcludeView="simple">_cap</Item>

      <Synthetic Name="[sorted?]" IncludeView="detail" Condition="_size &gt; 1">
        <DisplayString>キー順序の検証</DisplayString>
        <Expand>
          <CustomListItems>
            <Variable Name="i"   InitialValue="1" />
            <Variable Name="bad" InitialValue="-1" />
            <Loop Condition="i &lt; (int)_size">
              <If Condition="bad &lt; 0 &amp;&amp; !(_data[i-1].key &lt; _data[i].key)">
                <Exec>bad = i</Exec>
              </If>
              <Exec>i++</Exec>
            </Loop>
            <If Condition="bad &lt; 0"><Item Name="[result]">"ソート済み"</Item></If>
            <Else><Item Name="[!] 順序違反">bad</Item></Else>
          </CustomListItems>
        </Expand>
      </Synthetic>

      <ArrayItems IncludeView="fast">
        <Size>_size</Size>
        <ValuePointer>_data</ValuePointer>
      </ArrayItems>

      <CustomListItems ExcludeView="fast">
        <Variable Name="i"     InitialValue="0" />
        <Variable Name="guard" InitialValue="0" />
        <Size>_size</Size>
        <Loop Condition="i &lt; (int)_size">
          <Break Condition="guard &gt; 100000" />
          <Item Name="[{_data[i].key}]">_data[i].value</Item>
          <Exec>i++</Exec>
          <Exec>guard++</Exec>
        </Loop>
      </CustomListItems>
    </Expand>
  </Type>

  <Type Name="vizkit::flat_map&lt;*,*&gt;::entry">
    <DisplayString>{key} =&gt; {value}</DisplayString>
    <Expand>
      <Item Name="key">key</Item>
      <Item Name="value">value</Item>
    </Expand>
  </Type>
```

## 24.9 この章のまとめ

- **キーを名前の列に出せるのは `CustomListItems` だけ。** `<Item Name="[{式}]">値の式</Item>` と書く。
- `Name` の中の式では、ループ変数を参照できる。これが `TreeItems` / `ArrayItems` との決定的な差。
- ループ変数は `int` に、比較相手は `(int)` にキャストして**符号を揃える**。
- キー型の表示を調整したいときは、**キー型の側に `IncludeView` でビューを用意し、`,view(bare)` で選ぶ**。式をコンテナ側に書き下すより保守しやすい。
- `CustomListItems` の `<Item>` に `Condition` を書くのは避け、**`<If>` で囲む**。表示形式が複数あるならブロックごと複製する。
- `ArrayItems` 版（速い・添字表示）と `CustomListItems` 版（遅い・キー表示）を**ビューで使い分ける**。要素数の想定で既定を決める。
- ループを回して結果を1件出すパターンで、**全要素にわたる不変条件**（ソート済みかどうか）を検証できる。

## 24.10 演習

1. `<Loop Condition="i &lt; _size">`（`(int)` キャストなし）に変え、動作が変わるかどうか確認してください。環境によっては動きますが、なぜ揃えるべきかを考えてください。
2. `<Item Name="[{_data[i].key}]">` の `Name` を `[{i}] {_data[i].key}` に変え、添字とキーを両方出す形にしてください。
3. `flat_map<name32, int>` を作り、24.4節の `bare` ビューを使ってキーを引用符なしで表示させてください。
4. デバッガから `_data[1].key` を `_data[2].key` より大きい値に書き換え、`[sorted?]` が `[!] 順序違反` を出すことを確認してください。
5. `fm,view(fast)` と `fm` を並べ、要素数を 1000 件に増やしたときの応答速度の差を体感してください。
