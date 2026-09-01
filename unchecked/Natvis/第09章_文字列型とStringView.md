# 第9章 文字列型と StringView

文字列は、Natvis において特別扱いが必要な唯一のデータ型です。

理由は単純で、**デバッガが特別扱いしているのは `const char*` と `wchar_t*` だけ**だからです。あなたが書いた文字列型は、たとえ中身が `char` の配列であっても、その恩恵を受けません。

この章では、自作の文字列型を「文字列らしく」表示させ、さらに**テキストビジュアライザ**（虫めがねアイコン）に接続します。

## 9.1 デバッガが文字列に見えるもの・見えないもの

第3章で観察したとおりです。

```
  name       0x00007ff6a1b2c3d0 "vizkit"      const char *
```

`const char*` は NUL 終端文字列として表示されます。これは型が `char` へのポインタだから、というだけの理由による特別扱いです。

一方、これを構造体に包んだ瞬間に失われます。

```cpp
struct wrapped { const char* p; };
```

```
▷ w        {p=0x00007ff6a1b2c3d0 "vizkit" }    wrapped
```

メンバーとしては文字列表示が残りますが、`wrapped` 自身の値列は `{p=...}` という構造体表示です。100 個の `wrapped` が並んだとき、これは読めません。

そして、**長さを別に持つ文字列**（NUL 終端でない）になると、完全に手に負えなくなります。

## 9.2 題材：2種類の文字列型

自作ライブラリで現れる文字列型は、だいたい次の2種類です。両方作ります。

`vizkit\include\vizkit\core\str_view.hpp`

```cpp
#pragma once

#include <cstddef>

namespace vizkit {

// 非所有・非 NUL 終端の文字列ビュー。
// 部分文字列を切り出しても新しいバッファを作らない代わりに、
// 「どこまでが文字列か」はデバッガには分からない。
struct str_view {
    const char* _data = nullptr;
    std::size_t _size = 0;
};

} // namespace vizkit
```

`vizkit\include\vizkit\core\name32.hpp`

```cpp
#pragma once

#include <cstddef>
#include <cstdint>
#include <cstring>

namespace vizkit {

// 固定長バッファに文字列を持つ型。ヒープを使わない代わりに上限がある。
// 本書では NUL 終端も維持する設計にしてある（_len は冗長だが速い）。
struct name32 {
    static constexpr std::size_t capacity = 31;

    char          _buf[32] = {};
    std::uint8_t  _len     = 0;

    void assign(const char* s) noexcept {
        std::size_t n = std::strlen(s);
        if (n > capacity) n = capacity;
        std::memcpy(_buf, s, n);
        _buf[n] = '\0';
        _len = static_cast<std::uint8_t>(n);
    }
};

} // namespace vizkit
```

`main.cpp` に確認用の値を用意します。

```cpp
#include <vizkit/core/str_view.hpp>
#include <vizkit/core/name32.hpp>
// ...
    const char* sentence = "hello, natvis world";

    vizkit::str_view sv_all{ sentence, 19 };
    vizkit::str_view sv_part{ sentence + 7, 6 };   // "natvis" ―― NUL 終端していない
    vizkit::str_view sv_empty{ sentence, 0 };
    vizkit::str_view sv_null{};

    vizkit::name32 label;
    label.assign("frame_buffer");
    vizkit::name32 blank;
```

素の表示：

```
▷ sv_all    {_data=0x00007ff6... "hello, natvis world" _size=19 }   vizkit::str_view
▷ sv_part   {_data=0x00007ff6... "natvis world" _size=6 }           vizkit::str_view
▷ label     {_buf=0x00000078abcff5e0 "frame_buffer" _len=12 }       vizkit::name32
```

`sv_part` に注目してください。**`"natvis world"` と表示されています。** 正しくは `"natvis"` の 6 文字です。デバッガは `_size` を知らないので、NUL が来るまで読み続けてしまいます。**これは単に読みにくいのではなく、嘘を表示している**という点で、より深刻な問題です。

## 9.3 長さ指定つき文字列 ―― `,[式]na`

第6章 6.5節で扱ったサイズ指定子を、文字列に適用します。

```xml
<Type Name="vizkit::str_view">
  <DisplayString>{_data,[_size]na}</DisplayString>
</Type>
```

```
  sv_all      "hello, natvis world"      vizkit::str_view
  sv_part     "natvis"                   vizkit::str_view
  sv_empty    ""                         vizkit::str_view
```

`sv_part` が正しく 6 文字になりました。

### なぜ `na` なのか

`na` は「ポインタのアドレスを表示しない」という指定子です。`_data` は `const char*` なので、素のままだと `0x00007ff6... "natvis"` のようにアドレスが前に付きます。`na` でそれを落とすと、文字列部分だけが残ります。

MSVC 同梱の `STL.natvis` も `std::string_view` に対してまったく同じ書き方をしています。

```xml
<Type Name="std::basic_string_view&lt;*,*&gt;">
  <Intrinsic Name="size" Expression="_Mysize" />
  <Intrinsic Name="data" Expression="_Mydata" />
  <DisplayString>{_Mydata,[_Mysize]na}</DisplayString>
  <StringView>_Mydata,[_Mysize]na</StringView>
  ...
```

`na` を選んでいるのは、`basic_string_view<*,*>` が `char` にも `wchar_t` にも `char8_t` にも当たるためです。`na` なら、要素の型を見てデバッガが適切な文字列表現を選びます。

自作型で文字の型が `char` に固定されているなら、`s8`（UTF-8、引用符あり）を明示しても構いません。

| 書き方 | 表示 | 使いどころ |
|---|---|---|
| `{_data,[_size]na}` | `"natvis"` | 文字型が可変。汎用。**迷ったらこれ** |
| `{_data,[_size]s8}` | `"natvis"` | UTF-8 と分かっている |
| `{_data,[_size]s8b}` | `natvis` | 他のテキストと連結するとき（引用符が邪魔） |
| `{_data,[_size]su}` | `L"natvis"` | UTF-16 |

## 9.4 null と巨大サイズをガードする

第7章のルールをここでも適用します。文字列型では**特に重要**です。

```
  sv_null     <エラー: ...>            vizkit::str_view
```

null ポインタから読もうとすればエラーになります。さらに危険なのは、**`_size` が壊れている場合**です。未初期化のスタック領域に `_size = 0xCCCCCCCCCCCCCCCC` のような値が入っていると、デバッガは 1800 京文字を読もうとします。ウォッチウィンドウが数秒固まる、あるいは応答しなくなることがあります。

上限つきのガードを入れます。

```xml
<Type Name="vizkit::str_view">
  <DisplayString Condition="_data == nullptr">[null]</DisplayString>
  <DisplayString Condition="_size == 0">""</DisplayString>
  <DisplayString Condition="_size &gt; 4096">[suspicious size: {_size}]</DisplayString>
  <DisplayString>{_data,[_size]na}</DisplayString>
</Type>
```

```
  sv_all      "hello, natvis world"        vizkit::str_view
  sv_part     "natvis"                     vizkit::str_view
  sv_empty    ""                           vizkit::str_view
  sv_null     [null]                       vizkit::str_view
```

`4096` という上限は恣意的です。扱うデータに応じて決めてください。要点は、**「そんなに長いはずがない」という知識を Natvis に書き込んでおくと、壊れたオブジェクトを踏んでもデバッガが止まらない**ということです。この防御は第4部以降のコンテナでも同じ形で繰り返し登場します。

## 9.5 name32 ―― 埋め込みバッファ

`name32` は `char _buf[32]` を持っています。配列なので、デバッガは既定で文字列として表示してくれます。したがって最小の Natvis はこうです。

```xml
<Type Name="vizkit::name32">
  <DisplayString>{_buf,s8}</DisplayString>
</Type>
```

```
  label       "frame_buffer"       vizkit::name32
  blank       ""                   vizkit::name32
```

ただしこれは「NUL 終端が維持されている」という前提に依存しています。`_len` を持っているのですから、そちらを信じるほうが堅牢です。

```xml
<Type Name="vizkit::name32">
  <DisplayString Condition="_len == 0">[empty]</DisplayString>
  <DisplayString Condition="_len &gt; 31">[corrupt: len={_len,d}]</DisplayString>
  <DisplayString>{_buf,[_len]na}</DisplayString>
</Type>
```

`_len > 31` のチェックは、`capacity` を超えた値が入っていれば構造体が壊れていることを意味します。**不変条件を Natvis に書いておくと、デバッガが不変条件チェッカーになります。** これは Natvis の隠れた利点です。第22章では赤黒木の不変条件で同じことをします。

## 9.6 StringView ―― テキストビジュアライザに繋ぐ

`DisplayString` は1行です。長い文字列、改行を含む文字列、JSON や SQL のような構造を持つ文字列は、1行では読めません。

`<StringView>` 要素を書くと、その値の右端に**虫めがねアイコン**が現れ、**テキストビジュアライザ**で全文を開けるようになります。

```xml
<Type Name="vizkit::str_view">
  <DisplayString Condition="_data == nullptr">[null]</DisplayString>
  <DisplayString Condition="_size == 0">""</DisplayString>
  <DisplayString Condition="_size &gt; 4096">[suspicious size: {_size}]</DisplayString>
  <DisplayString>{_data,[_size]na}</DisplayString>
  <StringView Condition="_data != nullptr">_data,[_size]na</StringView>
</Type>
```

`StringView` の中身は `DisplayString` と違って**式だけ**を書きます。中括弧は要りません。書式指定子は付けられます。

```xml
<!-- 正しい -->
<StringView>_data,[_size]na</StringView>

<!-- 誤り -->
<StringView>{_data,[_size]na}</StringView>
```

`Condition` も付けられるので、null のときは虫めがねを出さない、といった制御ができます。

`name32` にも同様に付けます。

```xml
<Type Name="vizkit::name32">
  <DisplayString Condition="_len == 0">[empty]</DisplayString>
  <DisplayString Condition="_len &gt; 31">[corrupt: len={_len,d}]</DisplayString>
  <DisplayString>{_buf,[_len]na}</DisplayString>
  <StringView>_buf,[_len]na</StringView>
</Type>
```

### テキストビジュアライザの実用性

`name32` のような 31 文字の型では、正直あまりありがたみがありません。効いてくるのは次のような場面です。

- **改行を含む文字列**。`DisplayString` では `\r\n` がエスケープされて 1 行に潰れますが、テキストビジュアライザは改行して表示します。
- **数 KB のログやレスポンスボディ**。ウォッチウィンドウの値列は途中で切れます。
- **JSON / XML / HTML**。テキストビジュアライザにはこれらの形式を整形して表示するモードがあります。値の右の虫めがねの▼から選べます。

ライブラリが文字列らしきものを保持する型を持つなら、**`StringView` は書いておいて損がありません**。1 行の追加で、あとで必ず助かります。

## 9.7 実装が違えば書き方も違う ―― STL の string を読む

参考までに、`std::string` に対する `STL.natvis` の書き方を見ておきます。SSO（小さい文字列をヒープではなく構造体内部に置く最適化）のため、`Condition` で分岐しています。

```xml
<DisplayString Condition="isShortString()">{_Mypair._Myval2._Bx._Buf,na}</DisplayString>
<DisplayString Condition="isLongString()">{_Mypair._Myval2._Bx._Ptr,na}</DisplayString>
```

構造としては、この章でやってきたことと同じです。

- `Intrinsic` で判定条件に名前を付ける（`isShortString` / `isLongString`）
- `Condition` で状態を分岐する
- `,na` で文字列として表示する

違うのは、対象が union であることだけです。同じパターンを、第18章の `small_vector`（配列版の SSO）と第34章の `variant_like` で再訪します。

## 9.8 現時点の vizkit.natvis（文字列部分）

```xml
  <!-- ===== strings ===== -->

  <Type Name="vizkit::str_view">
    <DisplayString Condition="_data == nullptr">[null]</DisplayString>
    <DisplayString Condition="_size == 0">""</DisplayString>
    <DisplayString Condition="_size &gt; 4096">[suspicious size: {_size}]</DisplayString>
    <DisplayString>{_data,[_size]na}</DisplayString>
    <StringView Condition="_data != nullptr &amp;&amp; _size &lt;= 4096">_data,[_size]na</StringView>
  </Type>

  <Type Name="vizkit::name32">
    <DisplayString Condition="_len == 0">[empty]</DisplayString>
    <DisplayString Condition="_len &gt; 31">[corrupt: len={_len,d}]</DisplayString>
    <DisplayString>{_buf,[_len]na}</DisplayString>
    <StringView Condition="_len &lt;= 31">_buf,[_len]na</StringView>
  </Type>
```

## 9.9 この章のまとめ

- デバッガが文字列として扱うのは `const char*` / `wchar_t*` だけ。**自作型は自分で書く必要がある**。
- **長さを別に持つ文字列は、Natvis なしでは嘘を表示する。** NUL まで読み続けてしまうため。
- 基本形は **`{_data,[_size]na}`**。`STL.natvis` の `std::string_view` と同じ書き方。
- 文字型が固定なら `s8` / `su` を明示してもよい。引用符が邪魔なら `s8b` / `sub`。
- **null と巨大サイズは必ずガードする。** 壊れた `_size` はデバッガを固まらせる。
- 不変条件（`_len <= capacity` など）を `Condition` に書いておくと、Natvis が不変条件チェッカーとして働く。
- **`<StringView>` は式だけを書く。中括弧は不要。** 1 行では読めない文字列に対して効く。

## 9.10 演習

1. `sv_part` の Natvis を一時的に `{_data,s8}`（長さ指定なし）に変え、`"natvis world"` と嘘が表示されることを確認してください。この章の存在理由です。
2. `str_view` の `StringView` を削除し、虫めがねアイコンが消えることを確認してください。そのうえで、改行を含む長い文字列を `str_view` に入れ、`DisplayString` だけでどこまで読めるか試してください。
3. `name32` の `_len` にデバッガから直接 `200` を書き込み（ウォッチウィンドウで値を編集できます）、`[corrupt: len=200]` と表示されることを確認してください。
4. `name32` の `DisplayString` を `{_buf,s8}` 版と `{_buf,[_len]na}` 版で切り替え、`_buf` の途中に手動で `'\0'` を書き込んだときの表示の差を観察してください。どちらの実装を信じるべきか、`name32` の設計から考えてみてください。
