# 第33章 型消去・CRTP・ポリシークラス

第7部の締めくくりです。テンプレートを使った3つの高度な技法を、Natvis からどう扱うかを見ます。

結論を先に言うと、**型消去は Natvis から最も遠い技法**です。そしてその事実から、本書が繰り返し述べてきた「Natvis を書きやすい設計」という観点が、最も鮮明な形で立ち上がります。

## 33.1 型消去 ―― なぜ書けないのか

型消去とは、異なる型を同じ静的型で扱えるようにする技法です。`std::any`、`std::function`、仮想関数を使わない多態がこれにあたります。

素朴な実装を考えます。

```cpp
namespace vizkit {

// 型消去の最小形。バッファに任意の型を格納する。
class any_like {
public:
    template <class T>
    void emplace(const T& v) {
        static_assert(sizeof(T) <= 32);
        new (_storage) T(v);
        _destroy = [](void* p) { static_cast<T*>(p)->~T(); };
    }

    ~any_like() { if (_destroy) _destroy(_storage); }

private:
    alignas(16) unsigned char _storage[32]{};
    void (*_destroy)(void*) = nullptr;
};

} // namespace vizkit
```

Natvis から見えるのは、`_storage`（32 バイトの生バイト列）と `_destroy`（関数ポインタ）だけです。

```
▷ a          {_storage=0x... "..." _destroy=0x00007ff6a1b2c3d0 {test.exe!...} }
```

**格納されている型がどこにも記録されていません。**

`_destroy` が指す関数のシンボル名には、テンプレート実体化の痕跡としてラムダの型名が含まれることがあります。デバッガはこれを表示することもありますが、**Natvis の式からその情報を取り出す手段はありません**。関数を呼ぶこともできません（第5章 5.5節）。

vtable を持つ設計（`shape_base` のような）ならデバッガが最派生型を解決できました（第32章 32.4節）。しかし型消去はまさに「型を消す」ための技法なので、その手がかりが意図的に取り除かれています。

**Natvis は、実行時に型情報が残っている場合にしか型を判別できません。**

## 33.2 型情報を残す設計

したがって、型消去型に Natvis を当てたければ、**設計側で型情報を残す**しかありません。3つの方法があります。

### (a) 型名の文字列を持つ

最も直接的です。

```cpp
template <class T>
constexpr const char* type_name_of() {
#if defined(_MSC_VER)
    return __FUNCSIG__;      // 関数シグネチャに T が含まれる
#else
    return __PRETTY_FUNCTION__;
#endif
}

class any_like {
public:
    template <class T>
    void emplace(const T& v) {
        new (_storage) T(v);
        _destroy   = [](void* p) { static_cast<T*>(p)->~T(); };
        _type_name = type_name_of<T>();       // ← 追加
        _type_size = sizeof(T);
    }

private:
    alignas(16) unsigned char _storage[32]{};
    void (*_destroy)(void*) = nullptr;
    const char* _type_name  = nullptr;        // ← 追加
    std::size_t _type_size  = 0;
};
```

```xml
<Type Name="vizkit::any_like">
  <DisplayString Condition="_destroy == nullptr">[empty]</DisplayString>
  <DisplayString>any&lt;{_type_name,sb}&gt; ({_type_size} bytes)</DisplayString>
  <Expand>
    <Item Name="[type]">_type_name,s8b</Item>
    <Item Name="[size]">_type_size</Item>
    <Item Name="[storage]" IncludeView="detail">_storage</Item>
  </Expand>
</Type>
```

```
  a       any<const char *__cdecl vizkit::type_name_of<int>(void)> (4 bytes)
```

型名が出ました。`__FUNCSIG__` の生の文字列なので冗長ですが、**何が入っているかは分かります**。

デバッグビルドでのみ持つようにすれば、リリースへの影響もありません。

```cpp
#ifdef VIZKIT_DEBUG_TYPEINFO
    const char* _type_name = nullptr;
    std::size_t _type_size = 0;
#endif
```

Natvis 側は `Optional="true"` で吸収します（第8章 8.6節）。

```xml
<Item Name="[type]" Optional="true">_type_name,s8b</Item>
```

### (b) 型タグ（enum）を持つ

格納しうる型が限られているなら、enum のほうが軽量で、**中身まで表示できます**。

```cpp
enum class any_kind : std::uint8_t { none, i32, f64, point, name };

class tagged_any {
    alignas(16) unsigned char _storage[32]{};
    any_kind _kind = any_kind::none;
};
```

```xml
<Type Name="vizkit::tagged_any">
  <DisplayString Condition="_kind == vizkit::any_kind::none">[empty]</DisplayString>
  <DisplayString Condition="_kind == vizkit::any_kind::i32">int: {*(int*)_storage}</DisplayString>
  <DisplayString Condition="_kind == vizkit::any_kind::f64">double: {*(double*)_storage,g}</DisplayString>
  <DisplayString Condition="_kind == vizkit::any_kind::point">point2: {*(vizkit::point2*)_storage}</DisplayString>
  <DisplayString Condition="_kind == vizkit::any_kind::name">name32: {*(vizkit::name32*)_storage}</DisplayString>
  <DisplayString>[unknown kind={_kind,d}]</DisplayString>

  <Expand>
    <Item Name="[kind]">_kind,en</Item>
    <Item Name="value" Condition="_kind == vizkit::any_kind::i32">*(int*)_storage</Item>
    <Item Name="value" Condition="_kind == vizkit::any_kind::f64">*(double*)_storage</Item>
    <Item Name="value" Condition="_kind == vizkit::any_kind::point">*(vizkit::point2*)_storage</Item>
    <Item Name="value" Condition="_kind == vizkit::any_kind::name">*(vizkit::name32*)_storage</Item>
  </Expand>
</Type>
```

**中身が読めるようになりました。** 型名だけでなく値まで見えます。

この形は第34章の `variant_like` とまったく同じです。**型消去に Natvis を当てるということは、実質的に variant として設計し直すこと**を意味します。

### (c) 既知の型を総当たりする

型タグがない既存のライブラリに対する、最後の手段です。

```xml
<Item Name="[as int]"     IncludeView="raw">*(int*)_storage</Item>
<Item Name="[as double]"  IncludeView="raw">*(double*)_storage</Item>
<Item Name="[as point2]"  IncludeView="raw">*(vizkit::point2*)_storage</Item>
```

すべて表示されますが、**正しいのは1つだけで、どれが正しいかは分かりません**。デバッグの手がかりにはなりますが、実用性は低いです。`IncludeView="raw"` に閉じ込め、「これは推測である」ことが分かる名前にしてください。

### 結論

> **型消去型の Natvis は、設計とセットでしか書けない。**
> 型情報を残していないなら、書けるのは「何バイトかの生データがある」ことだけ。

第27章 27.6節で `std::unordered_map` について述べた「データ構造の設計が Natvis の書きやすさを決める」という発見が、ここでは極端な形で現れます。**Natvis で見たいなら、見えるように作る。** これが本書の最も実務的な結論の1つです。

## 33.3 CRTP

CRTP（Curiously Recurring Template Pattern）は、派生クラス自身をテンプレート引数として基底に渡す技法です。

```cpp
namespace vizkit {

template <class Derived>
struct comparable {
    [[nodiscard]] bool operator!=(const Derived& o) const noexcept {
        return !(static_cast<const Derived&>(*this) == o);
    }
};

struct version_id : comparable<version_id> {
    std::uint32_t value = 0;
    [[nodiscard]] bool operator==(const version_id& o) const noexcept { return value == o.value; }
};

} // namespace vizkit
```

Natvis から見ると、`comparable<version_id>` は**メンバーを持たない空の基底**です。表示上はほぼ無害ですが、`[Raw View]` に1行増えます。

```
▷ vid                    {value=42}
   ▷ vizkit::comparable<vizkit::version_id>   {...}
     value               42
```

対処は簡単です。CRTP 基底に `Inheritable="false"` で空の定義を与え、平坦化もしません。

```xml
<Type Name="vizkit::comparable&lt;*&gt;" Inheritable="false">
  <DisplayString>(crtp base)</DisplayString>
  <Expand HideRawView="true" />
</Type>
```

**`Inheritable="false"` が効く典型例**です（第32章 32.3節）。これを付けないと、`comparable` を継承したすべての型が `(crtp base)` と表示されてしまいます。

派生側は普通に書きます。

```xml
<Type Name="vizkit::version_id">
  <DisplayString>#{value}</DisplayString>
</Type>
```

CRTP 基底が**状態を持つ**場合（カウンタやフラグなど）は、`ExpandedItem` で引き上げます。第12章 12.4節と同じで、`,nd` を忘れないでください。

```xml
<ExpandedItem>*(vizkit::counted&lt;$T1&gt;*)this,nd</ExpandedItem>
```

`$T1` を使って基底の型名を組み立てられる点に注目してください。CRTP は**派生型がテンプレート引数として現れる**ので、Natvis から基底の型名を書きやすい構造になっています。

## 33.4 ポリシークラス

テンプレート引数で実装が切り替わる設計です。

```cpp
namespace vizkit {

struct no_lock   { };
struct spin_lock { std::atomic_flag flag = ATOMIC_FLAG_INIT; };

template <class T, class LockPolicy = no_lock>
class registry {
    dynamic_array<T> _items;
    LockPolicy       _lock;
public:
    ...
};

} // namespace vizkit
```

`registry<int, no_lock>` と `registry<int, spin_lock>` は別の型です。1つの Natvis で両方に対応したいところです。

### 基本形：委譲する

ポリシーは実装の詳細なので、**表示は本体に委ねる**のが素直です。

```xml
<Type Name="vizkit::registry&lt;*,*&gt;">
  <DisplayString>{_items}</DisplayString>
  <Expand>
    <Item Name="[policy]" ExcludeView="simple">_lock</Item>
    <ExpandedItem>_items</ExpandedItem>
  </Expand>
</Type>
```

`std::unordered_map` が `{_List}` と書いているのと同じ発想です（第27章 27.6節）。**委譲できるなら委譲する。**

`[policy]` は `ExcludeView="simple"` で隠します。実装詳細を1行に押し込める、というのは `STL.natvis` の `[allocator]` と同じ判断です（第27章 27.4節）。

### ポリシーによってメンバー構成が変わる場合

より厄介なのは、ポリシーによって**本体のメンバーが変わる**場合です。

```cpp
template <class T, bool TrackStats = false>
class registry;                     // TrackStats が true のときだけ _hits を持つ
```

`Optional` で吸収します（第8章 8.6節）。

```xml
<Type Name="vizkit::registry&lt;*,*&gt;">
  <DisplayString>{_items}</DisplayString>
  <Expand>
    <ExpandedItem>_items</ExpandedItem>
    <Item Name="[hits]"   Optional="true" ExcludeView="simple">_hits</Item>
    <Item Name="[misses]" Optional="true" ExcludeView="simple">_misses</Item>
  </Expand>
</Type>
```

`TrackStats` が `false` のビルドでは、`_hits` が存在しないので、その `Item` だけが黙って消えます。**「壊れたノードだけを無視し、残りは適用する」という `Optional` の設計どおりの使い方**です。

### 非型引数で表示を変える

第30章 30.3節で触れた形です。ポリシーが `bool` や enum なら、`Condition` で分岐できます。

```xml
<Type Name="vizkit::registry&lt;*,*&gt;">
  <DisplayString Condition="$T2">{_items} (stats on)</DisplayString>
  <DisplayString>{_items}</DisplayString>
</Type>
```

**コンパイル時の選択が、デバッガの表示に現れます。** どのビルド構成のオブジェクトを見ているかを取り違えなくなります。

### ポリシーごとに特殊化する

構造がまったく違うなら、第31章の特殊化を使います。

```xml
<Type Name="vizkit::registry&lt;*,vizkit::spin_lock&gt;" Priority="High">
  <DisplayString>{_items} [locked={_lock.flag}]</DisplayString>
  ...
</Type>

<Type Name="vizkit::registry&lt;*,*&gt;">
  <DisplayString>{_items}</DisplayString>
  ...
</Type>
```

`Priority="High"` を忘れないでください（第31章 31.3節）。

## 33.5 第7部のまとめ

第29章から第33章までで、テンプレートへの対応を扱いました。

```
Type Name="ns::tmpl&lt;*,*&gt;"      ワイルドカード。引数の個数だけ書くのが確実（第29章）
                                  末尾の * は残りをまとめて受ける
 ├─ $T1 / $T2 / ...               型引数は「型」、非型引数は「値」（第30章）
 ├─ Priority="High/…/Low"         同じ場所での優先順位。既定 Medium（第31章）
 ├─ AlternativeType Name="..."    同構造の型で定義を共有（第31章）
 ├─ Version Name="..." Min/Max    モジュールのバージョンで限定（第31章）
 └─ Inheritable="true/false"      派生に適用するか。既定 true（第32章）
```

第7部を通じての原則です。

1. **型名は「型」列からコピーする。** 推測しない（第29章）。
2. **`$Tn` を使うなら引数の個数だけ `*` を書く。**（第29章・第30章）
3. **`$Tn` に頼らない書き方があるなら、そちらが堅牢。** クラス内の定数を使う（第30章）。
4. **優先度は「譲る」方向に設計する。** ライブラリ側は `Medium` のまま（第31章）。
5. **`,nd` は最派生型の解決を止める指定子。** 基底へのキャストには必須（第32章）。
6. **デバッガが型を判別できるかは vtable の有無で決まる。**（第32章・第33章）
7. **型消去型の Natvis は、設計とセットでしか書けない。**（第33章）

第8部からは、`optional` / `variant` / スマートポインタ / ハンドル型といった、**個別の型の見せ方**に踏み込みます。第33章 33.2節で見た「型タグで分岐する」形が、そこで中心的な手法になります。

## 33.6 この章のまとめ

- **型消去は Natvis から最も遠い技法。** 実行時に型情報が残っていなければ、判別する手段がない。
- 対処は3つ ―― **(a) 型名の文字列を持つ**（`__FUNCSIG__`）、**(b) 型タグ enum を持つ**（中身まで表示できる）、**(c) 既知の型を総当たり**（推測にすぎない）。
- **型消去に Natvis を当てるとは、実質的に variant として設計し直すこと。**
- デバッグ専用の型情報メンバーは `#ifdef` で囲み、Natvis 側は `Optional="true"` で吸収する。
- CRTP 基底には **`Inheritable="false"` と空の `Expand`** を与えて無害化する。状態を持つなら `ExpandedItem` ＋ `,nd`。
- CRTP は**派生型がテンプレート引数に現れる**ので、`$T1` で基底の型名を組み立てられる。
- ポリシークラスは、まず**委譲**（`{_items}` と `ExpandedItem`）。ポリシー自体は `[policy]` として `ExcludeView="simple"` に。
- ポリシーでメンバー構成が変わるなら `Optional="true"`、`bool`/enum なら `Condition` と `$T2`、構造が違うなら `Priority="High"` の特殊化。
- **Natvis で見たいなら、見えるように作る。** 本書で最も実務的な結論。

## 33.7 演習

1. 33.1節の `any_like` に `int` を格納し、`[Raw View]` から中身の `4` を見つけられるか試してください。何バイト目にあるかを特定する作業が、型情報がないことの意味を教えてくれます。
2. 33.2節 (a) の `_type_name` を追加し、`__FUNCSIG__` がどんな文字列になるか確認してください。表示を短くする方法（先頭の固定部分を読み飛ばすなど）も考えてみてください。
3. 33.2節 (b) の `tagged_any` を実装し、4種類の型を格納して、それぞれが正しく表示されることを確認してください。`_kind` をデバッガから書き換え、誤ったキャストで何が表示されるかも見てください。
4. `comparable<*>` の `Inheritable="false"` を外し、`version_id` が `(crtp base)` と表示されることを確認してください。
5. `registry<int, no_lock>` と `registry<int, spin_lock>` を両方作り、33.4節の特殊化で表示が変わることを確認してください。`Priority="High"` を外すとどうなるかも試してください。
