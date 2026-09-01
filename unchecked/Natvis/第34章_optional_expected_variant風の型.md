# 第34章 optional / expected / variant 風の型

第8部に入ります。ここからは、コンテナではなく**個別の型の見せ方**を扱います。

最初は「値を持っているかもしれないし、持っていないかもしれない」型 ―― タグ付き共用体です。第33章 33.2節で「型消去に Natvis を当てるとは、実質的に variant として設計し直すこと」と述べました。その variant を、ここで正面から扱います。

## 34.1 共通の構造

`optional`、`expected`、`variant` は、Natvis から見ると同じ形をしています。

```
[ 記憶領域 ] + [ どれが有効かを示すタグ ]
```

タグの種類が違うだけです。

| 型 | タグ | 状態の数 |
|---|---|---|
| `optional<T>` | `bool` | 2（値あり / なし） |
| `expected<T,E>` | `bool` | 2（値 / エラー） |
| `variant<Ts...>` | 整数インデックス | N + 1（無効状態を含む） |

したがって Natvis の書き方も共通です。**タグを `Condition` で判定し、対応する型にキャストして表示する。** 第18章の `small_vector`（SBO）と同じ構造です。

そして最大の注意点も共通です。**無効な側を絶対に読ませてはいけません。** 未構築の領域を要素型の Natvis に渡すと、不正なメモリを読んでデバッガが不安定になります。

## 34.2 optional_like

`vizkit\include\vizkit\util\optional_like.hpp`

```cpp
#pragma once

#include <new>
#include <utility>

namespace vizkit {

template <class T>
class optional_like {
public:
    optional_like() = default;
    ~optional_like() { reset(); }

    void emplace(const T& v) { reset(); new (_storage) T(v); _has_value = true; }
    void reset() noexcept { if (_has_value) { ptr()->~T(); _has_value = false; } }

    [[nodiscard]] bool has_value() const noexcept { return _has_value; }
    [[nodiscard]] T&   value()           noexcept { return *ptr(); }

private:
    [[nodiscard]] T* ptr() noexcept { return reinterpret_cast<T*>(_storage); }

    alignas(T) unsigned char _storage[sizeof(T)]{};
    bool _has_value = false;
};

} // namespace vizkit
```

Natvis：

```xml
<Type Name="vizkit::optional_like&lt;*&gt;">
  <DisplayString Condition="!_has_value">nullopt</DisplayString>
  <DisplayString>{*($T1*)_storage}</DisplayString>
  <Expand>
    <Item Name="[has_value]" ExcludeView="simple">_has_value</Item>
    <Item Name="value" Condition="_has_value">*($T1*)_storage</Item>
  </Expand>
</Type>
```

```
  o1          42                      vizkit::optional_like<int>
  o2          nullopt                 vizkit::optional_like<int>
```

3つの要点があります。

**(1) `($T1*)_storage` のキャスト。** 第18章 18.5節・第30章 30.2節と同じです。生バイト列に型を与えます。

**(2) `Condition` が値を守っている。** `_has_value` が偽なら `*($T1*)_storage` は評価されません。第7章 7.4節で述べた「`Condition` は危険な式を評価させないガードでもある」の典型例です。

**(3) `Item` にも `Condition` を付ける。** `DisplayString` だけ守っても、展開したときに未構築領域を読んでしまいます。**両方に付ける**必要があります。

### std::optional と比べる

`STL.natvis` の定義です。

```xml
<Type Name="std::optional&lt;*&gt;">
  <DisplayString Condition="!_Has_value">nullopt</DisplayString>
  <DisplayString Condition="_Has_value">{_Value}</DisplayString>
  <Expand>
    <Item Condition="_Has_value" Name="value">_Value</Item>
  </Expand>
</Type>
```

**ほぼ同一です。** 違いは、`std::optional` が共用体メンバー `_Value` を持っているのでキャストが要らない点だけです。

ここに設計上の示唆があります。**生バイト配列より共用体のほうが、Natvis を書きやすい**のです。

```cpp
// Natvis を書きやすい設計
template <class T>
class optional_like {
    union { char _dummy; T _value; };   // 名前が付く
    bool _has_value = false;
};
```

```xml
<DisplayString Condition="_has_value">{_value}</DisplayString>
```

キャストが消え、`$T1` への依存もなくなります。第30章 30.5節で述べた「`$Tn` に頼らない書き方があるならそちら」の実例です。

## 34.3 expected_like

C++23 の `std::expected` に相当する型です。

```cpp
#pragma once

namespace vizkit {

template <class T, class E>
class expected_like {
public:
    expected_like(const T& v) : _value(v), _has_value(true) {}
    static expected_like unexpected(const E& e) { expected_like r; r._error = e; r._has_value = false; return r; }

    [[nodiscard]] bool has_value() const noexcept { return _has_value; }

private:
    expected_like() {}
    ~expected_like() {}   // 実際には _has_value を見て適切に破棄する

    union {
        T _value;
        E _error;
    };
    bool _has_value = false;
};

} // namespace vizkit
```

```xml
<Type Name="vizkit::expected_like&lt;*,*&gt;">
  <DisplayString Condition="_has_value">{_value}</DisplayString>
  <DisplayString>[error] {_error}</DisplayString>
  <Expand>
    <Item Name="value" Condition="_has_value">_value</Item>
    <Item Name="error" Condition="!_has_value">_error</Item>
  </Expand>
</Type>
```

```
  r1          42                       vizkit::expected_like<int,vizkit::name32>
  r2          [error] "file not found"  vizkit::expected_like<int,vizkit::name32>
```

**エラー側に印を付ける**のが実用上のポイントです。`[error]` という接頭辞があるだけで、100 個の `expected` が並んだときに失敗したものが目に飛び込んできます。第7章 7.5節の「状態は分けて表示する」に沿った設計です。

## 34.4 variant_like と可変長テンプレートの壁

```cpp
#pragma once

#include <cstdint>

namespace vizkit {

template <class... Ts>
class variant_like {
public:
    static constexpr std::uint8_t invalid = 0xFF;

    template <class T> void set(std::uint8_t index, const T& v);

private:
    alignas(16) unsigned char _storage[64]{};
    std::uint8_t _index = invalid;
};

} // namespace vizkit
```

第29章 29.7節で予告した問題がここで現れます。**テンプレート引数の個数が使い方によって変わるので、`Type Name` に個数を書けません。**

```xml
<Type Name="vizkit::variant_like&lt;*&gt;">
```

末尾の `*` に頼るしかありません。そして `$T2` 以降が確実に取れる保証がありません。

### 素直な解決 ―― タグで分岐する

`$Tn` に頼らず、**実行時のタグ `_index` で分岐**します。

```xml
<Type Name="vizkit::variant_like&lt;*&gt;">
  <Intrinsic Name="index" Expression="(int)_index" />

  <DisplayString Condition="_index == 0xFF">[valueless]</DisplayString>
  <DisplayString Condition="index() == 0" Optional="true">{{ index=0, value={*($T1*)_storage} }}</DisplayString>
  <DisplayString Condition="index() == 1" Optional="true">{{ index=1, value={*($T2*)_storage} }}</DisplayString>
  <DisplayString Condition="index() == 2" Optional="true">{{ index=2, value={*($T3*)_storage} }}</DisplayString>
  <DisplayString>{{ index={index()} }}</DisplayString>

  <Expand>
    <Item Name="[index]">index()</Item>
    <Item Name="value" Condition="index() == 0" Optional="true">*($T1*)_storage</Item>
    <Item Name="value" Condition="index() == 1" Optional="true">*($T2*)_storage</Item>
    <Item Name="value" Condition="index() == 2" Optional="true">*($T3*)_storage</Item>
  </Expand>
</Type>
```

**`Optional="true"` が本質的です。** `variant_like<int, double>` に対して `$T3` は存在しないので、3つ目の行は解析に失敗します。`Optional` があれば黙って飛ばされ、他の行は生きます（第8章 8.2節）。

そして最後に、条件なしのフォールバックを置きます（第8章 8.7節）。想定より多い型を持つ variant でも、少なくとも `index` は表示されます。

### std::variant の Natvis はどうしているか

驚くかもしれませんが、`STL.natvis` は**同じことを 32 回書いています**。

```xml
<Type Name="std::variant&lt;*&gt;">
  <Intrinsic Name="index" Expression="(int)_Which"/>
  <DisplayString Condition="index() &lt; 0">[valueless_by_exception]</DisplayString>
  <DisplayString Condition="index() == 0" Optional="true">{{ index=0, value={_Head} }}</DisplayString>
  <DisplayString Condition="index() == 1" Optional="true">{{ index=1, value={_Tail._Head} }}</DisplayString>
  <DisplayString Condition="index() == 2" Optional="true">{{ index=2, value={_Tail._Tail._Head} }}</DisplayString>
  <!-- ... index=31 まで続く ... -->
  <Expand>
    <Item Name="index">index()</Item>
    <!-- ... 0 から 31 まで ... -->
  </Expand>
</Type>
```

`_Tail._Tail._Head` という再帰的な構造をベタ書きし、`Optional="true"` で存在しないものを飛ばしています。**33 個目以降の型を持つ `std::variant` は、デバッガで中身が見えません。**

これは Natvis の表現力の限界を示す、非常に率直な実例です。

**(1) 再帰的な定義が書けない。** Natvis の `Type` は自分自身を参照できません。

**(2) ループで `Item` を生成できない。** `CustomListItems` は使えますが、**型の列挙はできません**（`$Tn` をループ変数で選べない）。

**(3) したがってベタ書きするしかない。** MSVC は 32 で線を引きました。

自作の `variant_like` でも、**現実的に使う個数だけ書く**のが正解です。5〜8 個も書けば足ります。書かなかった分はフォールバックの `{{ index={index()} }}` に落ちるので、少なくとも状態は分かります。

### インデックスと型の対応を残す設計

より確実なのは、第33章 33.2節 (b) と同じく、**enum のタグを持たせる**ことです。

```cpp
enum class value_kind : std::uint8_t { none, i32, f64, point, name };

class tagged_value {
    alignas(16) unsigned char _storage[32]{};
    value_kind _kind = value_kind::none;
};
```

```xml
<DisplayString Condition="_kind == vizkit::value_kind::i32">int: {*(int*)_storage}</DisplayString>
<DisplayString Condition="_kind == vizkit::value_kind::f64">double: {*(double*)_storage,g}</DisplayString>
```

`$Tn` にも `Optional` にも頼らず、**enum の名前で読める**表示になります。可変長テンプレートの制約を、設計で回避した形です。

**汎用性を捨てて、Natvis の書きやすさを取る。** 第33章 33.5節で立てた原則「Natvis で見たいなら、見えるように作る」の適用です。

## 34.5 未初期化領域を守る

タグ付き共用体で最も危険なのは、**無効な側を読ませてしまうこと**です。

```xml
<!-- 危険: _has_value を見ていない -->
<Item Name="value">*($T1*)_storage</Item>
```

`_has_value` が偽のとき、`_storage` には何が入っているか分かりません。要素型が `name32` なら `_len` に巨大な値が入っているかもしれず、そうなるとデバッガが大量のメモリを読もうとします（第9章 9.4節）。

チェックリストです。

1. **`DisplayString` に `Condition` を付ける**
2. **`Item` にも `Condition` を付ける**
3. **`ArrayItems` などの列挙要素にも `Condition` を付ける**（値がコンテナの場合）
4. **生の記憶領域は `IncludeView="detail"` に閉じ込める**

4番目は、デバッグ用に生バイトを見たいときの作法です。

```xml
<Item Name="[storage]" IncludeView="detail">_storage</Item>
```

`unsigned char[N]` として表示されるので、要素型の Natvis は適用されません。**型を与えないから安全**です。

## 34.6 現時点の Natvis

```xml
  <!-- ===== util ===== -->

  <Type Name="vizkit::optional_like&lt;*&gt;">
    <DisplayString Condition="!_has_value">nullopt</DisplayString>
    <DisplayString>{*($T1*)_storage}</DisplayString>
    <Expand>
      <Item Name="[has_value]" ExcludeView="simple">_has_value</Item>
      <Item Name="value" Condition="_has_value">*($T1*)_storage</Item>
      <Item Name="[storage]" IncludeView="detail">_storage</Item>
    </Expand>
  </Type>

  <Type Name="vizkit::expected_like&lt;*,*&gt;">
    <DisplayString Condition="_has_value">{_value}</DisplayString>
    <DisplayString>[error] {_error}</DisplayString>
    <Expand>
      <Item Name="value" Condition="_has_value">_value</Item>
      <Item Name="error" Condition="!_has_value">_error</Item>
    </Expand>
  </Type>

  <Type Name="vizkit::variant_like&lt;*&gt;">
    <Intrinsic Name="index" Expression="(int)_index" />
    <DisplayString Condition="_index == 0xFF">[valueless]</DisplayString>
    <DisplayString Condition="index() == 0" Optional="true">{{ index=0, value={*($T1*)_storage} }}</DisplayString>
    <DisplayString Condition="index() == 1" Optional="true">{{ index=1, value={*($T2*)_storage} }}</DisplayString>
    <DisplayString Condition="index() == 2" Optional="true">{{ index=2, value={*($T3*)_storage} }}</DisplayString>
    <DisplayString Condition="index() == 3" Optional="true">{{ index=3, value={*($T4*)_storage} }}</DisplayString>
    <DisplayString>{{ index={index()} }}</DisplayString>
    <Expand>
      <Item Name="[index]">index()</Item>
      <Item Name="value" Condition="index() == 0" Optional="true">*($T1*)_storage</Item>
      <Item Name="value" Condition="index() == 1" Optional="true">*($T2*)_storage</Item>
      <Item Name="value" Condition="index() == 2" Optional="true">*($T3*)_storage</Item>
      <Item Name="value" Condition="index() == 3" Optional="true">*($T4*)_storage</Item>
      <Item Name="[storage]" IncludeView="detail">_storage</Item>
    </Expand>
  </Type>
```

## 34.7 この章のまとめ

- `optional` / `expected` / `variant` は、Natvis から見ると同じ「記憶領域 ＋ タグ」の構造。**タグを `Condition` で判定し、対応する型にキャストする。**
- **`DisplayString` にも `Item` にも `Condition` を付ける。** 片方だけでは、展開時に未構築領域を読む。
- **生バイト配列より共用体のほうが Natvis を書きやすい。** メンバーに名前が付き、キャストと `$T1` への依存が消える。
- `expected` はエラー側に `[error]` の印を付ける。一覧したときに失敗が目に飛び込む。
- 可変長テンプレートは `Type Name` に個数を書けないので、**実行時のタグ `_index` で分岐する**。
- **`Optional="true"` が本質的。** 存在しない `$Tn` を参照する行を黙って飛ばす。条件なしのフォールバックも必ず置く。
- **`STL.natvis` の `std::variant` は 32 個分をベタ書きしている。** Natvis には再帰的定義もループによる型の列挙もない。自作型でも現実的な個数だけ書けばよい。
- より確実なのは **enum のタグを持たせる設計**。`$Tn` にも `Optional` にも頼らず、名前で読める表示になる。
- 生の記憶領域を見たいときは、**型を与えずに `unsigned char[N]` のまま** `IncludeView="detail"` で出す。

## 34.8 演習

1. `optional_like` の `Item` から `Condition="_has_value"` を外し、`nullopt` の状態で展開してみてください。要素型を `name32` にすると何が起きるかも確認してください（デバッガが重くなるので注意）。
2. `optional_like` を共用体版（`union { char _dummy; T _value; };`）に書き換え、Natvis からキャストが消えることを確認してください。
3. `variant_like<int, double>` に対して `index() == 2` の行から `Optional="true"` を外し、どんなエラーが出るか確認してください。
4. `_index` をデバッガから `10` に書き換え、フォールバックの `{{ index=10 }}` が表示されることを確認してください。
5. `STL.natvis` の `std::variant` の定義を実際に開き、32 個の `DisplayString` が並んでいることを目で確認してください。33 個以上の型を持つ `std::variant` を作って、表示がどうなるかも試してください。
