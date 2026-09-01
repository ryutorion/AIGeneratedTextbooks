# 第10章 Expand と Item

ここまでの9章で扱ってきた `DisplayString` は、ウォッチウィンドウの**値の列（1行）** を作るものでした。

第3部では、行の左の三角形を開いたときに現れる**子行**を作ります。担当するのは `<Expand>` 要素です。コンテナの要素を並べるのも、内部状態を整理して見せるのも、すべてここから始まります。

## 10.1 既定の展開を置き換える

`rect` を開くと、いまはこう見えています。

```
▷ box              [(0, 0), (640, 480)] 640x480      vizkit::rect
   ▷ _min          (0, 0)
   ▷ _max          (640, 480)
```

値の列は Natvis が作ったものですが、**子行は素のまま**です。`_min` と `_max` というメンバー名がそのまま並んでいます。

`<Expand>` を書くと、この子行を丸ごと差し替えられます。

```xml
<Type Name="vizkit::rect">
  <Intrinsic Name="w" Expression="_max.x - _min.x" />
  <Intrinsic Name="h" Expression="_max.y - _min.y" />
  <DisplayString Condition="w() &lt;= 0 || h() &lt;= 0">[empty]</DisplayString>
  <DisplayString>[{_min}, {_max}] {w(),g}x{h(),g}</DisplayString>
  <Expand>
    <Item Name="[width]">w()</Item>
    <Item Name="[height]">h()</Item>
    <Item Name="min">_min</Item>
    <Item Name="max">_max</Item>
  </Expand>
</Type>
```

```
▷ box              [(0, 0), (640, 480)] 640x480      vizkit::rect
     [width]       640
     [height]      480
   ▷ min           (0, 0)
   ▷ max           (640, 480)
   ▷ [Raw View]    {_min=(0, 0) _max=(640, 480) }
```

3つの変化が起きています。

**(1) 子行が定義したとおりに並んだ。** 順序も名前も、`Expand` に書いたままです。

**(2) 存在しないメンバーを子行にできた。** `[width]` と `[height]` は `rect` のメンバーではありません。式の評価結果です。

**(3) `[Raw View]` が現れた。** 素のメンバー展開はここに退避しました。第3章 3.4節で予告した動作です。

## 10.2 Item 要素

`Expand` の中で最も基本的な要素です。

```xml
<Item Name="表示名">式</Item>
```

- **`Name`** ―― 名前の列に出る文字列。リテラルですが、後述のとおり式も埋め込めます
- **要素の中身** ―― 値の列に出す**式**。`DisplayString` と違って中括弧は不要です

```xml
<!-- 正しい -->
<Item Name="[width]">_max.x - _min.x</Item>

<!-- 誤り: これは「_max.x - _min.x」という文字列を評価しようとして失敗する -->
<Item Name="[width]">{_max.x - _min.x}</Item>
```

`StringView`（第9章）と同じで、**式を書く場所には中括弧を書かない**というのが Natvis の一貫したルールです。中括弧が必要なのは `DisplayString` と `Item` の `Name` 属性だけだと覚えてください。

### Name には式を埋め込める

`Name` 属性の中では、`DisplayString` と同じ `{式}` 記法が使えます。

```xml
<Item Name="[{_len,d} chars]">_buf,[_len]na</Item>
```

これは連想コンテナで真価を発揮します。キーを名前に、値を値の列に置くことで、`["apple"] 3` という表示が作れます（第24章）。

### 書式指定子も使える

`Item` の中身は式なので、書式指定子が付けられます。

```xml
<Item Name="[packed]">packed,X</Item>
```

## 10.3 角括弧の慣習

`[width]` のように角括弧で囲む命名は、Natvis の世界の慣習です。

**「これは実際のメンバーではなく、デバッガのために合成された情報である」** という印になっています。`STL.natvis` も `[size]`、`[capacity]`、`[allocator]` という名前を使っています。

実メンバーをそのまま見せる `Item` には角括弧を付けません。上の例で `min` / `max` に括弧を付けなかったのはそのためです。

この区別は好みの問題ではなく、実務上の意味があります。デバッグ中に「この行はコードから触れるのか、Natvis の作り物なのか」が一目で分かるからです。

## 10.4 Condition と Optional

第7章・第8章で学んだ属性は、`Item` にもそのまま付きます。

```xml
<Expand>
  <Item Name="[width]">w()</Item>
  <Item Name="[height]">h()</Item>
  <Item Name="[warning]" Condition="w() &lt;= 0 || h() &lt;= 0">"degenerate rect"</Item>
  <Item Name="[origin]" Optional="true">_origin,s8</Item>
</Expand>
```

- `Condition` が `false` の `Item` は**行そのものが現れません**
- `Optional` が付いた `Item` は、式が解析できなければ黙って消えます

`Condition` 付きの `Item` は、**異常時にだけ現れる警告行**として使うと効果的です。正常なオブジェクトでは邪魔にならず、壊れたオブジェクトでは目に飛び込んできます。

なお `Item` の値に文字列リテラルを書けます（上の `"degenerate rect"`）。定型のメッセージを出したいときに使えます。

## 10.5 Expand を書くと既定の展開は消える

重要な性質です。**`<Expand>` を1つでも書いた瞬間、デバッガの既定のメンバー展開は使われなくなります。**

したがって、極端な例ですが次のように書くと子行が空になります。

```xml
<Expand>
</Expand>
```

```
▷ box              [(0, 0), (640, 480)] 640x480      vizkit::rect
   ▷ [Raw View]    {_min=(0, 0) _max=(640, 480) }
```

`[Raw View]` だけが残ります。これは「見せたいものが値の列だけで完結している型」では、むしろ望ましい形です。`version` のような型が該当します。

```xml
<Type Name="vizkit::version">
  <Intrinsic Name="major" Expression="packed >> 22" />
  <Intrinsic Name="minor" Expression="(packed >> 12) &amp; 0x3FF" />
  <Intrinsic Name="patch" Expression="packed &amp; 0xFFF" />
  <DisplayString Condition="packed == 0">[unset]</DisplayString>
  <DisplayString>v{major()}.{minor()}.{patch()} ({packed,X})</DisplayString>
  <Expand>
    <Item Name="[major]">major()</Item>
    <Item Name="[minor]">minor()</Item>
    <Item Name="[patch]">patch()</Item>
  </Expand>
</Type>
```

```
▷ ver              v3.12.7 (0X00C8C007)      vizkit::version
     [major]       3
     [minor]       12
     [patch]       7
   ▷ [Raw View]    {packed=13111303 }
```

生のメンバーは `packed` ひとつしかない型が、3つの意味あるフィールドを持つように見えるようになりました。**`Expand` は「実装の構造」ではなく「概念の構造」を見せるための道具です。**

## 10.6 順序に意味を持たせる

`Expand` の中の要素は、書いた順に上から並びます。デバッグ中に何度も見るものなので、順序は設計対象です。

指針を挙げます。

1. **要約情報を上に。** `[size]`、`[capacity]`、`[state]` など、まず知りたいもの
2. **異常時の警告をその次に。** `Condition` 付きの `Item`
3. **実データを下に。** 要素の列挙（第4部以降）
4. **付随情報を最後に。** `[allocator]`、`[origin]` など、めったに見ないもの

`[Raw View]` は常に最後に自動追加されるので、この順序の外側にあります。

## 10.7 byte_span に Expand を書く

第7章の `byte_span` を仕上げます。まだ要素の列挙（`ArrayItems`）は使えませんが、先頭数バイトくらいなら `Item` で並べられます。

```xml
<Type Name="vizkit::byte_span">
  <Intrinsic Name="ptr" Expression="_data" />
  <Intrinsic Name="len" Expression="_size" />
  <DisplayString Condition="ptr() == nullptr">[null]</DisplayString>
  <DisplayString Condition="len() == 0">[empty]</DisplayString>
  <DisplayString>{len()} bytes, first={ptr()[0],Xb}</DisplayString>
  <Expand>
    <Item Name="[size]">len()</Item>
    <Item Name="[data]">ptr()</Item>
    <Item Name="[0]" Condition="len() &gt; 0">ptr()[0],Xb</Item>
    <Item Name="[1]" Condition="len() &gt; 1">ptr()[1],Xb</Item>
    <Item Name="[2]" Condition="len() &gt; 2">ptr()[2],Xb</Item>
    <Item Name="[3]" Condition="len() &gt; 3">ptr()[3],Xb</Item>
    <Item Name="[...]" Condition="len() &gt; 4">len() - 4</Item>
  </Expand>
</Type>
```

```
▷ whole             6 bytes, first=DE        vizkit::byte_span
     [size]         6
   ▷ [data]         0x00000078abcff610
     [0]            DE
     [1]            AD
     [2]            BE
     [3]            EF
     [...]          2
   ▷ [Raw View]     {_data=... _size=6 }
```

見られるようになりましたが、明らかに不格好です。要素数が可変なのに `Item` を手で並べているからです。**これが `<ArrayItems>` が必要な理由**で、第14章でこの `Expand` を4行に書き換えます。

いまは「`Item` の手作業では限界がある」ことを体験するための踏み台と考えてください。ただし `Condition` で行の有無を制御する技法自体は、以降も繰り返し使います。

## 10.8 現時点の vizkit.natvis

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
    <Expand>
      <Item Name="[r]">r,d</Item>
      <Item Name="[g]">g,d</Item>
      <Item Name="[b]">b,d</Item>
      <Item Name="[a]">a,d</Item>
    </Expand>
  </Type>

  <Type Name="vizkit::rect">
    <Intrinsic Name="w" Expression="_max.x - _min.x" />
    <Intrinsic Name="h" Expression="_max.y - _min.y" />
    <DisplayString Condition="w() &lt;= 0 || h() &lt;= 0">[empty]</DisplayString>
    <DisplayString>[{_min}, {_max}] {w(),g}x{h(),g}</DisplayString>
    <Expand>
      <Item Name="[width]">w()</Item>
      <Item Name="[height]">h()</Item>
      <Item Name="[warning]" Condition="w() &lt;= 0 || h() &lt;= 0">"degenerate rect"</Item>
      <Item Name="min">_min</Item>
      <Item Name="max">_max</Item>
    </Expand>
  </Type>

  <Type Name="vizkit::version">
    <Intrinsic Name="major" Expression="packed >> 22" />
    <Intrinsic Name="minor" Expression="(packed >> 12) &amp; 0x3FF" />
    <Intrinsic Name="patch" Expression="packed &amp; 0xFFF" />
    <DisplayString Condition="packed == 0">[unset]</DisplayString>
    <DisplayString>v{major()}.{minor()}.{patch()} ({packed,X})</DisplayString>
    <Expand>
      <Item Name="[major]">major()</Item>
      <Item Name="[minor]">minor()</Item>
      <Item Name="[patch]">patch()</Item>
    </Expand>
  </Type>

  <!-- byte_span / str_view / name32 は前章までのものに Expand を追加（本文参照） -->

</AutoVisualizer>
```

## 10.9 この章のまとめ

- `<Expand>` は子行を再定義する。**1つでも書けば既定のメンバー展開は使われなくなり、`[Raw View]` に退避する**。
- `<Item Name="表示名">式</Item>` が基本単位。**中身は式なので中括弧を書かない**。
- `Name` 属性の中では `{式}` が使える。連想コンテナでキーを名前にするときに効く。
- **角括弧付きの名前 `[size]` は「合成された情報」の印**という慣習がある。実メンバーには付けない。
- `Condition` で行の有無を制御できる。異常時にだけ出る警告行が作れる。
- `Expand` は「実装の構造」ではなく「概念の構造」を見せるためのもの。`version` のように、生メンバー1つを3つの意味に開ける。
- 要素数が可変のものを `Item` で並べるのは限界がある。第14章の `ArrayItems` へ続く。

## 10.10 演習

1. `rect` の `Expand` から `min` / `max` の `Item` を削除し、`[width]` / `[height]` だけにしてください。実メンバーが見たくなったときにどこを開けばよいか確認してください。
2. `rect` に `<Expand></Expand>`（中身が空）を書き、子行が `[Raw View]` だけになることを確認してください。
3. `rgba` の `Expand` に `<Item Name="[luminance]">0.299 * r + 0.587 * g + 0.114 * b</Item>` を足してください。整数と浮動小数点の混在で結果がどうなるか観察し、必要なら `,g` を付けてください。
4. `name32` に `<Item Name="[{_len,d}/31]">_buf,[_len]na</Item>` を書き、`Name` の中で式が展開されることを確認してください。
