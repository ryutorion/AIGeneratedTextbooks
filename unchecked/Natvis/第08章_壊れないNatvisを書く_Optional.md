# 第8章 壊れない Natvis を書く ―― Optional

Natvis は**実装の内部に直接依存します**。private メンバーの名前を式に書き込むのですから、当然です。

その結果、ライブラリをリファクタリングした瞬間に Natvis が壊れます。しかも壊れ方が静かなので、しばらく気づかないことすらあります。この章では、その壊れ方を観察し、`Optional` 属性で被害を局所化する方法を扱います。

## 8.1 リファクタリングで何が起きるか

`byte_span` のメンバー名を変えてみましょう。ライブラリの命名規約を `_` 始まりに統一した、という想定です。

```cpp
struct byte_span {
    const std::byte* _data = nullptr;   // data → _data
    std::size_t      _size = 0;         // size → _size
};
```

第7章の Natvis はそのままです。`.natvisreload` して表示を見ます。

```
  whole       <エラー: 識別子 "data" が定義されていません>    vizkit::byte_span
  empty_span  <エラー: 識別子 "data" が定義されていません>    vizkit::byte_span
  null_span   <エラー: 識別子 "data" が定義されていません>    vizkit::byte_span
```

出力ウィンドウ（診断メッセージが Verbose）にも記録されます。

```
Natvis: 型 'vizkit::byte_span' の Intrinsic 'is_null' の評価に失敗しました:
        識別子 "data" が定義されていません
```

**これは良い壊れ方です。** エラーが目に見えるので、直せます。問題はもっと厄介なケースです。

## 8.2 Optional 属性の意味

公式ドキュメントの記述は簡潔です。

> `Optional` 属性は任意のノードに付けられる。optional なノードの内部の部分式が**解析に失敗した**場合、デバッガはそのノードを無視し、`Type` の残りのルールは適用する。

分解すると3点です。

1. **任意のノードに付けられる**（`DisplayString`、`Item`、`ArrayItems`、`Intrinsic`、`ExpandedItem` …）
2. 効くのは**解析（parse）の失敗**。存在しないメンバー名は解析失敗にあたる
3. 失敗したノードだけが無視され、**同じ `Type` の他のノードは生きたまま**

3点目が重要です。`Optional` は「壊れた部分だけを切り捨てて、残りは動かす」ための仕組みです。

## 8.3 使ってみる

新旧どちらのメンバー名でも動く Natvis を書きます。

```xml
<Type Name="vizkit::byte_span">
  <!-- 旧レイアウト: data / size -->
  <DisplayString Optional="true" Condition="data == nullptr">[null]</DisplayString>
  <DisplayString Optional="true" Condition="size == 0">[empty]</DisplayString>
  <DisplayString Optional="true">{size} bytes, first={data[0],Xb}</DisplayString>

  <!-- 新レイアウト: _data / _size -->
  <DisplayString Optional="true" Condition="_data == nullptr">[null]</DisplayString>
  <DisplayString Optional="true" Condition="_size == 0">[empty]</DisplayString>
  <DisplayString Optional="true">{_size} bytes, first={_data[0],Xb}</DisplayString>
</Type>
```

新しい実装（`_data` / `_size`）でこれを読み込むと、上の3つは解析に失敗して無視され、下の3つが第7章と同じように動きます。逆に古い実装に戻しても動きます。

```
  whole       6 bytes, first=DE       vizkit::byte_span
  empty_span  [empty]                 vizkit::byte_span
  null_span   [null]                  vizkit::byte_span
```

このパターンは実務で本当に使います。**複数バージョンのライブラリを1つの `.natvis` でサポートしたい**とき、`Optional` を並べるのが最も手軽な方法です（より厳密にやるなら `<Version>` 要素を使います。第31章）。

## 8.4 Intrinsic に付ける

`Intrinsic` にも `Optional` を付けられます。むしろこちらのほうが整理されます。

```xml
<Type Name="vizkit::byte_span">
  <!-- どちらか一方だけが成立する -->
  <Intrinsic Optional="true" Name="ptr" Expression="data" />
  <Intrinsic Optional="true" Name="ptr" Expression="_data" />
  <Intrinsic Optional="true" Name="len" Expression="size" />
  <Intrinsic Optional="true" Name="len" Expression="_size" />

  <DisplayString Condition="ptr() == nullptr">[null]</DisplayString>
  <DisplayString Condition="len() == 0">[empty]</DisplayString>
  <DisplayString>{len()} bytes, first={ptr()[0],Xb}</DisplayString>
</Type>
```

**レイアウトの差異を `Intrinsic` に閉じ込め、表示ロジックは1つだけ書く。** メンバー名の揺れが 4 行に局所化され、`DisplayString` は 1 組で済みます。表示を変えたくなったときに直す場所も 1 か所です。

MSVC の `STL.natvis` も同じ手法を使っています。

```xml
<Intrinsic Optional="true" Name="allocator" Expression="*((_Mybase *) this)"/>
<Intrinsic Optional="true" Name="allocator" Expression="_Myval"/>
```

## 8.5 Optional を付けるべきでない場面

`Optional` は便利ですが、無条件に付けるものではありません。**「壊れても黙る」という性質は、諸刃の剣**です。

自分のライブラリの、自分で書いた Natvis に全部 `Optional` を付けたとします。あるときメンバー名を変え、Natvis を直し忘れた。表示は素の生ビューに戻りますが、エラーは出ません。「あれ、なんか表示が素っ気ないな」と思いながら、そのままデバッグを続けることになります。

判断基準を整理します。

| 状況 | `Optional` |
|---|---|
| 自分のライブラリの、常に存在するメンバー | **付けない**。壊れたら気づきたい |
| バージョンによって存在したりしなかったりするメンバー | 付ける |
| 標準ライブラリなど、実装が環境で変わるものへの参照 | 付ける |
| デバッグビルドにだけ存在するメンバー（カナリア、割り当て元情報など） | 付ける |
| 派生型に依存する参照（`ExpandedItem` で基底を辿るなど） | 付ける |

実務での落としどころは、「**壊れてほしくない中核部分は `Optional` なし、環境差を吸収する部分だけ `Optional` あり**」です。

## 8.6 デバッグビルド専用メンバーの例

`Optional` が最も自然に効くのは、条件付きコンパイルされるメンバーです。

```cpp
struct byte_span {
    const std::byte* _data = nullptr;
    std::size_t      _size = 0;
#ifdef VIZKIT_DEBUG_TAGS
    const char*      _origin = nullptr;   // どこで作られたか
#endif
};
```

```xml
<Type Name="vizkit::byte_span">
  <DisplayString Condition="_data == nullptr">[null]</DisplayString>
  <DisplayString Condition="_size == 0">[empty]</DisplayString>
  <DisplayString>{_size} bytes, first={_data[0],Xb}</DisplayString>
  <Expand>
    <Item Name="[origin]" Optional="true">_origin,s8</Item>
  </Expand>
</Type>
```

`VIZKIT_DEBUG_TAGS` が定義されていないビルドでは `_origin` が存在しないので、この `Item` だけが黙って消えます。`DisplayString` は影響を受けません。まさに「壊れたノードだけを無視し、残りは適用する」の想定どおりの使い方です。

（`<Expand>` と `<Item>` は次章から本格的に扱います。ここでは `Optional` がどのノードにも付くことを示すためのものです。）

## 8.7 評価順序をもう一度整理する

`Condition` と `Optional` が同居すると、順序の理解が要ります。1つの `DisplayString` について、デバッガは次の順に判断します。

```
   ┌──────────────────────────────────────────────┐
   │ 1. ノードを解析できるか？                     │
   │      → 失敗 かつ Optional="true"  → 黙って飛ばす │
   │      → 失敗 かつ Optional なし    → エラー表示   │
   │ 2. Condition があるか？                        │
   │      → false → 次のノードへ                    │
   │ 3. 採用。ここで打ち切り                        │
   └──────────────────────────────────────────────┘
```

つまり **`Optional` の判定が先、`Condition` の判定が後**です。8.3節の例で、旧レイアウトの3つが「条件を評価するまでもなく」飛ばされていたのはこのためです。

そして「最初に採用されたところで打ち切り」というルール（第7章 7.3節）は変わりません。したがって**フォールバックは、`Optional` なしで最後に置く**のが定石です。

```xml
<Type Name="vizkit::byte_span">
  <DisplayString Optional="true" Condition="...">...</DisplayString>   <!-- 環境依存 -->
  <DisplayString Optional="true" Condition="...">...</DisplayString>   <!-- 環境依存 -->
  <DisplayString>[byte_span]</DisplayString>                           <!-- 常に成立する保険 -->
</Type>
```

最後の1行があれば、すべての `Optional` が飛ばされた最悪ケースでも「Natvis は読み込まれているが、中身がマッチしていない」ことが表示から分かります。素の生表示に戻るより、はるかに切り分けやすくなります。

## 8.8 現時点の vizkit.natvis

メンバー名は `_data` / `_size`（新レイアウト）に統一した前提で、実務的な形にまとめます。

```xml
<?xml version="1.0" encoding="utf-8"?>
<AutoVisualizer xmlns="http://schemas.microsoft.com/vstudio/debugger/natvis/2010">

  <Type Name="vizkit::point2">
    <DisplayString>({x,g}, {y,g})</DisplayString>
  </Type>

  <Type Name="vizkit::rgba">
    <DisplayString Condition="a == 0">[transparent]</DisplayString>
    <DisplayString Condition="a == 255">#{r,Xb}{g,Xb}{b,Xb}</DisplayString>
    <DisplayString>#{r,Xb}{g,Xb}{b,Xb} (a={a,d})</DisplayString>
  </Type>

  <Type Name="vizkit::rect">
    <Intrinsic Name="w" Expression="_max.x - _min.x" />
    <Intrinsic Name="h" Expression="_max.y - _min.y" />
    <DisplayString Condition="w() &lt;= 0 || h() &lt;= 0">[empty]</DisplayString>
    <DisplayString>[{_min}, {_max}] {w(),g}x{h(),g}</DisplayString>
  </Type>

  <Type Name="vizkit::version">
    <Intrinsic Name="major" Expression="packed >> 22" />
    <Intrinsic Name="minor" Expression="(packed >> 12) &amp; 0x3FF" />
    <Intrinsic Name="patch" Expression="packed &amp; 0xFFF" />
    <DisplayString Condition="packed == 0">[unset]</DisplayString>
    <DisplayString>v{major()}.{minor()}.{patch()} ({packed,X})</DisplayString>
  </Type>

  <Type Name="vizkit::byte_span">
    <Intrinsic Name="ptr" Expression="_data" />
    <Intrinsic Name="len" Expression="_size" />
    <DisplayString Condition="ptr() == nullptr">[null]</DisplayString>
    <DisplayString Condition="len() == 0">[empty]</DisplayString>
    <DisplayString>{len()} bytes, first={ptr()[0],Xb}</DisplayString>
    <Expand>
      <Item Name="[origin]" Optional="true">_origin,s8</Item>
    </Expand>
  </Type>

</AutoVisualizer>
```

## 8.9 この章のまとめ

- Natvis は実装の内部名に直接依存する。リファクタリングで壊れるのは避けられない。
- `Optional="true"` は「**そのノードの解析に失敗したら、黙って飛ばす。他のノードは生かす**」という属性。任意のノードに付けられる。
- 判定順は **`Optional`（解析）→ `Condition`（真偽）→ 採用して打ち切り**。
- レイアウトの差異は `Intrinsic Optional="true"` に閉じ込め、表示ロジックは1組にする。これが最も保守しやすい形。
- **全部に `Optional` を付けてはいけない。** 壊れたら気づきたい箇所には付けない。
- フォールバックの `DisplayString` は `Optional` なしで最後に置く。素の表示に戻るより、`[型名]` とだけ出るほうが切り分けやすい。

## 8.10 演習

1. `byte_span` の `Intrinsic Name="ptr"` を `Optional="true"` にしたうえで、`Expression` を存在しない `_pointer` に変えてください。`DisplayString` がどう表示されるか観察し、`Optional` なしの場合とどう違うか比べてください。
2. 8.3節の「新旧両対応」の形に書き戻し、ヘッダーのメンバー名を `data` / `size` と `_data` / `_size` の間で往復させて、どちらでも動くことを確認してください。
3. `Expand` の中の `Item Name="[origin]"` から `Optional="true"` を外し、`VIZKIT_DEBUG_TAGS` を定義していない状態でどうなるか確認してください。
4. `rect` の3つの `DisplayString` すべてに `Condition` を付け（フォールバックをなくし）、そのうえですべての条件が外れる `rect` の値を作ってみてください。何が表示されますか。
