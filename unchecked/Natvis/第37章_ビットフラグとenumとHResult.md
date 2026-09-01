# 第37章 ビットフラグ・enum・HResult

数値そのものは意味を持ちません。`6` が何を表すのかは、文脈が決めます。

この章では、数値に意味を与える3つの仕組み ―― enum、ビットフラグ、エラーコード ―― の見せ方を扱います。第6章 6.7節で表だけ示した `en` / `fe` / `hr` の書式指定子が、ここで主役になります。

## 37.1 enum と `en` 指定子

まず基本です。

```cpp
namespace vizkit {

enum class blend_mode : std::uint8_t {
    opaque      = 0,
    alpha       = 1,
    additive    = 2,
    multiply    = 3,
};

} // namespace vizkit
```

素の表示では、デバッガが自動的に列挙子名を出してくれます。

```
  bm         alpha (1)          vizkit::blend_mode
```

`en` 指定子を使うと、名前だけになります。

```xml
<Item Name="[mode]">_mode,en</Item>
```

```
     [mode]     alpha
```

`DisplayString` の中で他のテキストと連結するときは、`,en` を付けたほうが読みやすくなります。

```xml
<DisplayString>material {_name} ({_mode,en})</DisplayString>
```

```
  mat        material "wall" (alpha)
```

### 定義されていない値

enum の変数に、列挙子にない値が入ることがあります。ファイルから読んだ値をそのままキャストした場合などです。

```
  bm         (7)                vizkit::blend_mode
```

デバッガは括弧付きの数値だけを表示します。**これを異常として目立たせる**には、`Condition` で範囲を確認します。

```xml
<Type Name="vizkit::material">
  <DisplayString Condition="_mode &gt; vizkit::blend_mode::multiply">[!] material {_name} (unknown mode {_mode,d})</DisplayString>
  <DisplayString>material {_name} ({_mode,en})</DisplayString>
</Type>
```

第15章以降続けてきた「不変条件を Natvis に書く」の、enum への適用です。**enum は「定義された値しか入らない」と思い込みがちですが、C++ ではそうではありません。**

## 37.2 ビットフラグ ―― `fe` 指定子

複数のフラグを OR で組み合わせる型です。

```cpp
namespace vizkit {

enum class render_flags : std::uint32_t {
    none         = 0,
    depth_test   = 1u << 0,
    depth_write  = 1u << 1,
    cull_back    = 1u << 2,
    wireframe    = 1u << 3,
    cast_shadow  = 1u << 4,
};

[[nodiscard]] constexpr render_flags operator|(render_flags a, render_flags b) noexcept {
    return static_cast<render_flags>(static_cast<std::uint32_t>(a) | static_cast<std::uint32_t>(b));
}

} // namespace vizkit
```

素の表示は数値です。

```
  f          25                 vizkit::render_flags
```

`25` が `depth_test | wireframe | cast_shadow` であることを、頭の中で二進数に直して読む ―― これは避けたい作業です。

**`fe` 指定子**（ビットフラグ enum）が、これを解決します。

```xml
<Item Name="[flags]">_flags,fe</Item>
```

```
     [flags]    depth_test | wireframe | cast_shadow (25)
```

**1行で読めるようになりました。** `fe` は、値を構成するビットに対応する列挙子を探して `|` で連結してくれます。

`DisplayString` にも書けます。

```xml
<Type Name="vizkit::render_state">
  <DisplayString>{_flags,fe}</DisplayString>
</Type>
```

`fe` はデバッガのバージョンによって使えないことがあります。動かない場合は、次節の方法に切り替えてください。

## 37.3 フラグを1つずつ Item に出す

`fe` が使えない場合、あるいは**どのフラグが立っているかを個別に確認したい**場合の書き方です。

```xml
<Type Name="vizkit::render_state">
  <Intrinsic Name="bits" Expression="(unsigned)_flags" />
  <DisplayString Condition="bits() == 0">[none]</DisplayString>
  <DisplayString>{_flags,fe}</DisplayString>
  <Expand>
    <Item Name="[raw]" ExcludeView="simple">bits(),X</Item>
    <Synthetic Name="[flags]">
      <DisplayString>{_flags,fe}</DisplayString>
      <Expand>
        <Item Name="depth_test"  Condition="(bits() &amp; 0x01) != 0">true</Item>
        <Item Name="depth_write" Condition="(bits() &amp; 0x02) != 0">true</Item>
        <Item Name="cull_back"   Condition="(bits() &amp; 0x04) != 0">true</Item>
        <Item Name="wireframe"   Condition="(bits() &amp; 0x08) != 0">true</Item>
        <Item Name="cast_shadow" Condition="(bits() &amp; 0x10) != 0">true</Item>
        <Item Name="[!] unknown bits" Condition="(bits() &amp; ~0x1Fu) != 0">bits() &amp; ~0x1Fu,X</Item>
      </Expand>
    </Synthetic>
  </Expand>
</Type>
```

```
▷ st                 depth_test | wireframe | cast_shadow (25)
     [raw]           0X00000019
   ▷ [flags]         depth_test | wireframe | cast_shadow (25)
        depth_test   true
        wireframe    true
        cast_shadow  true
```

**立っているフラグだけが並びます。** `Condition` が偽の `Item` は行ごと現れないので（第10章 10.4節）、一覧が短くなります。

最後の `[!] unknown bits` に注目してください。**定義されていないビットが立っていたら警告を出します。** `~0x1Fu` は「定義済みの5ビット以外」のマスクです。

未知のビットは、次のような原因で立ちます。

- 古いバージョンで保存したデータを読んだ
- 別の enum の値を誤って代入した
- 構造体が破損している

**`fe` 指定子は未知のビットを黙って数値として出すだけ**なので、この警告行を足しておく価値があります。

### 定義を1か所にまとめる

ビットの値を Natvis に直書きするのは、実装との二重管理になります。`Intrinsic` で名前を付けておくと、いくらか改善します。

```xml
<Intrinsic Name="has" Expression="((unsigned)_flags &amp; mask) != 0">
  <Parameter Name="mask" Type="unsigned" />
</Intrinsic>
```

```xml
<Item Name="depth_test" Condition="has(0x01)">true</Item>
<Item Name="wireframe"  Condition="has(0x08)">true</Item>
```

**引数付きの `Intrinsic`** です。次章で詳しく扱います。

## 37.4 強い型 ―― 単位を持つ数値

物理量を扱うコードでは、単位を型で区別する設計がよく使われます。

```cpp
namespace vizkit {

template <class Tag, class Rep = double>
struct quantity {
    Rep value{};
};

struct meters_tag {};
struct seconds_tag {};
struct degrees_tag {};

using meters  = quantity<meters_tag>;
using seconds = quantity<seconds_tag>;
using degrees = quantity<degrees_tag>;

} // namespace vizkit
```

第36章 36.1節の `strong_id` と同じ構造です。タグごとに Natvis を書きます。

```xml
<Type Name="vizkit::quantity&lt;vizkit::meters_tag,*&gt;">
  <DisplayString>{value,g} m</DisplayString>
</Type>

<Type Name="vizkit::quantity&lt;vizkit::seconds_tag,*&gt;">
  <DisplayString>{value,g} s</DisplayString>
</Type>

<Type Name="vizkit::quantity&lt;vizkit::degrees_tag,*&gt;">
  <DisplayString>{value,g}°</DisplayString>
</Type>

<Type Name="vizkit::quantity&lt;*,*&gt;" Priority="Low">
  <DisplayString>{value,g}</DisplayString>
</Type>
```

```
  d          42.5 m             vizkit::quantity<vizkit::meters_tag,double>
  t          0.016 s            vizkit::quantity<vizkit::seconds_tag,double>
  a          90°                vizkit::quantity<vizkit::degrees_tag,double>
```

**単位が表示に出ます。** ラジアンと度の取り違え、秒とミリ秒の取り違えといったバグは、この表示があれば一目で見つかります。

`°` のような非 ASCII 文字も、`.natvis` を UTF-8 で保存していれば使えます（XML 宣言の `encoding="utf-8"` を確認してください。第2章 2.5節）。

## 37.5 HResult 要素

Windows API を扱うライブラリでは、`HRESULT` を保持する型が出てきます。

```cpp
namespace vizkit {

struct hr_result {
    long hr = 0;   // HRESULT
};

} // namespace vizkit
```

`hr` 指定子を使えば、既知のエラーコードは名前で表示されます。

```xml
<Type Name="vizkit::hr_result">
  <DisplayString Condition="hr &gt;= 0">OK ({hr,hr})</DisplayString>
  <DisplayString>[FAILED] {hr,hr}</DisplayString>
</Type>
```

```
  ok         OK (S_OK)
  err        [FAILED] E_OUTOFMEMORY
```

### 独自のエラーコードを登録する

ライブラリが独自の `HRESULT` を定義している場合、`<HResult>` 要素で名前と説明を登録できます。これは `Type` ではなく、`AutoVisualizer` の直下に書く要素です。

```xml
<AutoVisualizer xmlns="http://schemas.microsoft.com/vstudio/debugger/natvis/2010">

  <HResult Name="VIZKIT_E_INVALID_FORMAT">
    <HRValue>0x88760001</HRValue>
    <HRDescription>画像フォーマットが不正です</HRDescription>
  </HResult>

  <HResult Name="VIZKIT_E_TOO_LARGE">
    <HRValue>0x88760002</HRValue>
    <HRDescription>画像サイズが上限を超えています</HRDescription>
  </HResult>

  <!-- Type 定義が続く -->
</AutoVisualizer>
```

| 要素 | 意味 |
|---|---|
| `Name` 属性 | デバッガに表示される名前（必須） |
| `<HRValue>` | 32 ビットの `HRESULT` 値（必須） |
| `<HRDescription>` | 説明文（省略可） |
| `<AlternativeHResult>` | 同じ表示を共有する別の値（複数可） |

登録しておくと、`hr` 指定子がその名前を使うようになります。

```
  err        [FAILED] VIZKIT_E_INVALID_FORMAT
```

**エラーコードの意味を、デバッガの画面に置いておける。** 第11章 11.2節で述べた「Natvis はチームの知識を配布する手段」の、最も直接的な形です。ヘッダーのコメントを探しに行かずに済みます。

### HRESULT でないエラーコード

自作のエラー enum には `HResult` は使えません。`en` 指定子で列挙子名を出すのが基本です。

```cpp
enum class error_code : std::int32_t {
    ok = 0,
    invalid_format = -1,
    too_large = -2,
};
```

```xml
<Type Name="vizkit::error_result">
  <DisplayString Condition="_code == vizkit::error_code::ok">OK</DisplayString>
  <DisplayString>[FAILED] {_code,en}</DisplayString>
  <Expand>
    <Item Name="[code]">_code,en</Item>
    <Item Name="[numeric]" IncludeView="detail">(int)_code</Item>
    <Item Name="[hint]" Condition="_code == vizkit::error_code::too_large">"画像サイズの上限は 8192x8192 です"</Item>
  </Expand>
</Type>
```

`[hint]` のように、**エラーごとの対処方法を書いておく**のは実用的です。`HResult` の `HRDescription` を、enum で自前に再現した形になります。

## 37.6 この章のまとめ

- `en` 指定子は enum の列挙子名を表示する。**他のテキストと連結するときは必須。**
- **enum に定義外の値が入ることはある。** `Condition` で範囲外を検出し、警告を出す。
- `fe` 指定子はビットフラグ enum を `A | B | C (25)` の形で表示する。**1行で読める。**
- `fe` が使えない、または個別に確認したい場合は、**`Condition` 付きの `Item` を並べる**。立っているものだけが現れる。
- **未知のビット（`~mask`）を検出する行を足す。** `fe` は未知ビットを黙って数値にするだけ。
- 単位を持つ強い型は、タグごとに `Type` を書いて `42.5 m` のように**単位を表示に出す**。`Priority="Low"` の汎用定義をフォールバックに。
- `<HResult Name="...">` は `AutoVisualizer` の直下に書き、`<HRValue>` と `<HRDescription>` で独自エラーコードを登録する。`hr` 指定子がその名前を使うようになる。
- 自作の error enum では、`[hint]` の `Item` で**対処方法を書いておく**と `HRDescription` の代わりになる。

## 37.7 演習

1. `blend_mode` の変数に `(vizkit::blend_mode)7` を代入し、デバッガの表示を確認してください。そのうえで `[!]` の警告を出す `Condition` を書いてください。
2. `render_flags` に `25` を設定し、`,fe` と `,b`（二進数）と `,X`（16進数）の3つを並べて比べてください。どれが最も読みやすいか確認します。
3. `render_flags` に `0x100`（未定義のビット）を OR で足し、`[!] unknown bits` が出ることを確認してください。`,fe` の表示がどうなるかも見てください。
4. `quantity<degrees_tag>` の `DisplayString` に `°` を使い、`.natvis` を ANSI で保存した場合に文字化けすることを確認してください。UTF-8 で保存し直して直ることも確認します。
5. 独自の `HResult` を1つ登録し、`hr` 指定子でその名前が出ることを確認してください。`HRValue` を実在する `HRESULT`（`0x80070005` = `E_ACCESSDENIED` など）にした場合に何が起きるかも試してください。
