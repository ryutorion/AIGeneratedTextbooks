# 第5章 DisplayString の式構文

第4章で `<DisplayString>` を書きました。この章では、その中括弧の中に**何を書けるのか**を正確に押さえます。

Natvis の学習でつまずく箇所の大半は、「書けると思ったものが書けない」ことに起因します。境界線を先に引いておけば、以降の章で悩む時間が減ります。

## 5.1 題材を1つ増やす

ビット演算とキャストを試したいので、パックされた整数型を追加します。

`vizkit\include\vizkit\core\version.hpp`

```cpp
#pragma once

#include <cstdint>

namespace vizkit {

// major:10 bit | minor:10 bit | patch:12 bit を 1 つの uint32 に詰めた型。
// 「生のメンバーは 1 つしかないが、意味は 3 つある」型の典型。
struct version {
    std::uint32_t packed = 0;
};

[[nodiscard]] constexpr version make_version(std::uint32_t major,
                                             std::uint32_t minor,
                                             std::uint32_t patch) noexcept {
    return version{ (major << 22) | ((minor & 0x3FFu) << 12) | (patch & 0xFFFu) };
}

[[nodiscard]] constexpr std::uint32_t major_of(version v) noexcept { return v.packed >> 22; }
[[nodiscard]] constexpr std::uint32_t minor_of(version v) noexcept { return (v.packed >> 12) & 0x3FFu; }
[[nodiscard]] constexpr std::uint32_t patch_of(version v) noexcept { return v.packed & 0xFFFu; }

} // namespace vizkit
```

`main.cpp` に追加します。

```cpp
#include <vizkit/core/version.hpp>
// ...
    vizkit::version ver = vizkit::make_version(3, 12, 7);
    vizkit::version unset{};
```

素の表示はこうなります。

```
▷ ver       {packed=13111303}       vizkit::version
▷ unset     {packed=0}              vizkit::version
```

`13111303` を見て「3.12.7 だな」と分かる人はいません。Natvis の出番です。

## 5.2 DisplayString はテンプレート文字列である

`DisplayString` の中身は、**リテラル文字列と `{式}` が交互に並んだもの**として解釈されます。

```xml
<DisplayString>v{packed >> 22}.{(packed >> 12) &amp; 0x3FF}.{packed &amp; 0xFFF}</DisplayString>
```

```
  ver       v3.12.7                 vizkit::version
  unset     v0.0.0                  vizkit::version
```

`&` を `&amp;` と書いている点に注意してください。XML なので必須です（第4章 4.7節）。ビット演算を多用する Natvis は `&amp;` だらけになりますが、慣れます。

### 中括弧を出力したい

コンテナの Natvis では `{ size=3 }` のように中括弧そのものを出したくなります。**`{{` と `}}` でエスケープ**します。

```xml
<DisplayString>{{ v{packed >> 22}.{(packed >> 12) &amp; 0x3FF}.{packed &amp; 0xFFF} }}</DisplayString>
```

```
  ver       { v3.12.7 }             vizkit::version
```

MSVC 同梱の `STL.natvis` が `std::vector` を `{ size=3 }` と表示しているのも、この記法によるものです。

## 5.3 式が評価される文脈

`{}` の中に書くのは C++ の式ですが、評価の文脈が普通のコードとは違います。

**(1) `this` が暗黙に存在する。** メンバー名をそのまま書けば、そのオブジェクトのメンバーになります。`packed` は `this->packed` の略記です。明示的に `this->packed` と書いても構いません。

**(2) アクセス指定子は無視される。** private メンバーもそのまま書けます（第3章 3.3節）。

**(3) 名前解決はその型のスコープから始まる。** グローバル変数や別の名前空間の名前も、完全修飾すれば書けます。ただし「デバッガがシンボルを解決できる範囲」に限られるので、ヘッダーの `constexpr` 定数などは解決できないことがあります。

**(4) 副作用は許されない。** デバッガはこの式を、ウィンドウの再描画のたびに何度でも評価します。ステップ実行のたびに評価されると考えてください。副作用があれば、デバッグ対象の状態が壊れます。

## 5.4 書けるもの

| 分類 | 例 | 備考 |
|---|---|---|
| メンバーアクセス | `_min.x`, `this->packed` | 入れ子も可 |
| ポインタ経由 | `_head->_next->_value` | null なら評価エラーになる |
| 配列添字 | `_buf[0]`, `_data[i]` | |
| ポインタ演算 | `_begin + 3`, `_end - _begin` | コンテナで多用する |
| 算術 | `+ - * / %` | 整数除算の切り捨てに注意 |
| ビット演算 | `& \| ^ ~ << >>` | XML エスケープを忘れずに |
| 比較・論理 | `== != < > <= >=`, `&& \|\| !` | `&&` は短絡評価される |
| 三項演算子 | `_size == 0 ? 0 : 1` | 条件分岐を式内で完結できる |
| キャスト | `(int)packed`, `(node*)_storage` | C形式キャストを使う |
| `sizeof` | `sizeof(*this)` | レイアウト確認に便利 |
| アドレス・間接 | `&_value`, `*_ptr` | |
| リテラル | `0x3FF`, `1`, `true`, `nullptr` | 文字列リテラルは比較に使えない |

### キャストの実例

Natvis でキャストが要るのは、生バイト列を型として読むときです。

```xml
<!-- alignas したストレージを T として読む（第18章で本格的に使う） -->
<DisplayString>{*($T1*)_storage}</DisplayString>
```

C++ の `static_cast` / `reinterpret_cast` は使えません。**C形式のキャスト**を書きます。

### 三項演算子の実例

```xml
<DisplayString>v{packed >> 22}.{(packed >> 12) &amp; 0x3FF}.{packed &amp; 0xFFF}{packed == 0 ? " (unset)" : ""}</DisplayString>
```

ただし、この書き方は読みにくくなりがちです。表示の出し分けは次章以降で扱う `Condition` 属性のほうが素直です。三項演算子は「1つの式の中で小さく分岐したい」ときに留めるのが実務的です。

## 5.5 書けないもの

| 分類 | 例 | 代替 |
|---|---|---|
| **関数呼び出し** | `size()`, `empty()` | メンバーから式を組む。または `Intrinsic`（第38章） |
| 演算子オーバーロード | `a + b`（自作の `operator+`） | 中身を展開して書く |
| 代入・インクリメント | `i = 0`, `++p` | `CustomListItems` の `Exec` 内なら可（第23章） |
| `new` / `delete` | | 不可 |
| ラムダ・テンプレート実体化 | | 不可 |
| 文字列リテラルの比較 | `_name == "abc"` | 不可。先頭数バイトを整数として比較するなどの回避策はあるが非推奨 |
| `dynamic_cast` / RTTI | | 不可。`,nd` 指定子や `Inheritable` で代替（第32章） |

**「関数を呼べない」の唯一の例外**が `<Intrinsic>` です。Natvis 内で「名前付きの式」を定義すると、それを関数呼び出しの構文で使えます。

```xml
<Type Name="vizkit::version">
  <Intrinsic Name="major" Expression="packed >> 22" />
  <Intrinsic Name="minor" Expression="(packed >> 12) &amp; 0x3FF" />
  <Intrinsic Name="patch" Expression="packed &amp; 0xFFF" />
  <DisplayString>v{major()}.{minor()}.{patch()}</DisplayString>
</Type>
```

ずっと読みやすくなりました。`STL.natvis` の `std::basic_string_view` が `{data(),[size()]na}` と書けているのも、`Intrinsic` で `data` と `size` を定義しているからです。

`Intrinsic` は第38章で正式に扱いますが、**「長い式に名前を付ける道具」として今から使って構いません**。Natvis が読みにくくなる最大の原因は式の長さなので、早めに使う癖をつけたほうが得です。

## 5.6 ネストした型の DisplayString は再帰的に使われる

第4章で `rect` の `{_min}` が `(0, 0)` になったのを見ました。改めて言語化しておきます。

**`{式}` の評価結果が「Natvis を持つ型」だった場合、その型の `DisplayString` が使われます。**

```xml
<Type Name="vizkit::rect">
  <DisplayString>[{_min}, {_max}]</DisplayString>   <!-- point2 の DisplayString が入る -->
</Type>
```

これは強力ですが、副作用もあります。`point2` の表示を変えると `rect` の表示も変わる。設計としては、**小さい型ほど短く表示する**のが安全です。`point2` を `point2{x=12.5, y=-4.0}` のように冗長にすると、それを含むすべての型が読みにくくなります。

再帰を止めたいときは書式指定子 `,na`（アドレスを出さない）や `,!`（Natvis を無効化）を使います。詳しくは次章です。

## 5.7 1行に何を詰めるか

`DisplayString` は**1行**です。ウォッチウィンドウの値列に収まる幅しかありません。設計指針を挙げます。

- **一目で同一性が判別できる情報を優先する。** `id`、`name`、`size` など。
- **中身の全部を詰め込まない。** 要素の列挙は `<Expand>`（第3部）の仕事です。
- **状態の異常を目立たせる。** `[empty]`、`[null]`、`[invalid]` のような短い印は、100 個並んだときに効きます。
- **単位や記号を添える。** `640x480`、`v3.12.7`、`RGBA(...)` のように、人間が意味を思い出せる形にします。

## 5.8 現時点の vizkit.natvis

```xml
<?xml version="1.0" encoding="utf-8"?>
<AutoVisualizer xmlns="http://schemas.microsoft.com/vstudio/debugger/natvis/2010">

  <Type Name="vizkit::point2">
    <DisplayString>({x,g}, {y,g})</DisplayString>
  </Type>

  <Type Name="vizkit::rgba">
    <DisplayString>RGBA({r,d}, {g,d}, {b,d}, {a,d})</DisplayString>
  </Type>

  <Type Name="vizkit::rect">
    <DisplayString Condition="_max.x &lt;= _min.x || _max.y &lt;= _min.y">[empty]</DisplayString>
    <DisplayString>[{_min}, {_max}] {_max.x - _min.x,g}x{_max.y - _min.y,g}</DisplayString>
  </Type>

  <Type Name="vizkit::version">
    <Intrinsic Name="major" Expression="packed >> 22" />
    <Intrinsic Name="minor" Expression="(packed >> 12) &amp; 0x3FF" />
    <Intrinsic Name="patch" Expression="packed &amp; 0xFFF" />
    <DisplayString>v{major()}.{minor()}.{patch()}</DisplayString>
  </Type>

</AutoVisualizer>
```

## 5.9 この章のまとめ

- `DisplayString` は「リテラル ＋ `{式}`」のテンプレート文字列。`{{` `}}` で中括弧をエスケープする。
- 式の中では `this` が暗黙、private も見える、副作用は禁止。
- 算術・ビット演算・比較・三項演算子・C形式キャスト・`sizeof`・ポインタ演算が使える。
- **関数は呼べない。** 唯一の例外が `<Intrinsic>` で、これは「長い式に名前を付ける道具」として今すぐ使ってよい。
- ネストした型は `DisplayString` が再帰的に適用される。だから小さい型ほど短く表示する。

## 5.10 演習

1. `version` の `Intrinsic` を使って、`{{ major={major()} minor={minor()} patch={patch()} }}` と表示させてください。中括弧のエスケープが正しく効くことを確認します。
2. ウォッチウィンドウで `ver.packed >> 22` と `major_of(ver)` を両方入力し、後者が何と表示されるか確認してください。「関数を呼べない」の意味が体感できます。
3. `rect` の `DisplayString` に `sizeof(*this)` を足し、`32` と表示されることを確認してください。
4. `point2` の `DisplayString` を `point2{x={x,g}, y={y,g}}` に変え、`box` と `pts` の表示がどれだけ読みにくくなるか観察してください。確認したら元に戻してください。
