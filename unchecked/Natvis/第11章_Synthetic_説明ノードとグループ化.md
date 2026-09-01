# 第11章 Synthetic ―― 説明ノードとグループ化

`Item` は「式の評価結果」を1行として見せる要素でした。しかし、デバッグ中に見たいものの中には**式に対応しないもの**があります。

- 「このオブジェクトは GPU 上にあるので中身を見られません」という**説明**
- 内部状態をまとめる**見出し行**
- 統計値や診断結果を集めた**グループ**

`<Synthetic>` は、そうした「実体のない子行」を作る要素です。

## 11.1 Synthetic の基本形

```xml
<Synthetic Name="表示名">
  <DisplayString>値の列に出す文字列</DisplayString>
</Synthetic>
```

`Item` との違いは、**値が式ではなく `DisplayString` で与えられる**ことです。`Synthetic` 自体は何かのオブジェクトではないので、評価すべき式がありません。その代わり、`DisplayString` の中には `{式}` を埋め込めます。

公式ドキュメントの例が、この要素の性格をよく表しています。

```xml
<Synthetic Name="Array" Condition="_M_buffer_descriptor._M_data_ptr == 0">
  <DisplayString>Array members can be viewed only under the GPU debugger</DisplayString>
</Synthetic>
```

「見られない理由」を表示する。`Item` では書けない類の行です。

## 11.2 説明行を出す

`str_view` に応用します。第9章で `_size > 4096` を「疑わしい」として弾きましたが、値の列に `[suspicious size: ...]` と出るだけでは、なぜそう判断されたのかが読み手に伝わりません。

```xml
<Type Name="vizkit::str_view">
  <DisplayString Condition="_data == nullptr">[null]</DisplayString>
  <DisplayString Condition="_size == 0">""</DisplayString>
  <DisplayString Condition="_size &gt; 4096">[suspicious size: {_size}]</DisplayString>
  <DisplayString>{_data,[_size]na}</DisplayString>
  <StringView Condition="_data != nullptr &amp;&amp; _size &lt;= 4096">_data,[_size]na</StringView>
  <Expand>
    <Synthetic Name="[!]" Condition="_size &gt; 4096">
      <DisplayString>_size が 4096 を超えています。未初期化か破損の可能性があります。</DisplayString>
    </Synthetic>
    <Item Name="[size]">_size</Item>
    <Item Name="[data]">_data</Item>
  </Expand>
</Type>
```

正常な `str_view` では `[!]` の行は現れません。壊れたものを開いたときにだけ、先頭に説明が出ます。

**Natvis はチームの知識を配布する手段でもある**、という側面がここにあります。「この値がこうなっていたら何を疑うべきか」を、コメントではなくデバッガの画面に置いておける。ライブラリを他人に使ってもらうときに効きます。

## 11.3 Synthetic に Expand を入れる ―― グループ化

`Synthetic` の真価は、**中に `<Expand>` を入れられる**ことです。これによって、子行を階層化できます。

```xml
<Synthetic Name="[グループ名]">
  <DisplayString>要約</DisplayString>
  <Expand>
    <Item Name="...">...</Item>
    <Item Name="...">...</Item>
  </Expand>
</Synthetic>
```

`rect` に幾何情報のグループを作ってみます。

```xml
<Type Name="vizkit::rect">
  <Intrinsic Name="w" Expression="_max.x - _min.x" />
  <Intrinsic Name="h" Expression="_max.y - _min.y" />
  <DisplayString Condition="w() &lt;= 0 || h() &lt;= 0">[empty]</DisplayString>
  <DisplayString>[{_min}, {_max}] {w(),g}x{h(),g}</DisplayString>
  <Expand>
    <Item Name="min">_min</Item>
    <Item Name="max">_max</Item>
    <Synthetic Name="[geometry]">
      <DisplayString>{w(),g} x {h(),g}, area={w() * h(),g}</DisplayString>
      <Expand>
        <Item Name="width">w()</Item>
        <Item Name="height">h()</Item>
        <Item Name="area">w() * h()</Item>
        <Item Name="aspect" Condition="h() != 0">w() / h()</Item>
        <Item Name="center x">(_min.x + _max.x) / 2</Item>
        <Item Name="center y">(_min.y + _max.y) / 2</Item>
      </Expand>
    </Synthetic>
  </Expand>
</Type>
```

```
▷ box                    [(0, 0), (640, 480)] 640x480      vizkit::rect
   ▷ min                 (0, 0)
   ▷ max                 (640, 480)
   ▷ [geometry]          640 x 480, area=307200
        width            640
        height           480
        area             307200
        aspect           1.3333333333333333
        center x         320
        center y         240
   ▷ [Raw View]          {_min=(0, 0) _max=(640, 480) }
```

**畳んでおけるのが利点です。** 派生情報は普段見たくないが、必要なときには手が届く。`Item` を6つ直接並べると、`rect` を開くたびに6行が視界を占領します。`Synthetic` でまとめれば1行です。

さらに `Synthetic` の `DisplayString` に要約を書いておけば、**開かなくても中身の見当がつく**。これが `Synthetic` を使う最大の理由です。

## 11.4 Synthetic の中で式はどう評価されるか

重要な注意点です。

**`Synthetic` の内側の `Expand` に書く式は、`Synthetic` ではなく「その `Synthetic` を含む型」の文脈で評価されます。**

上の例で `<Item Name="width">w()</Item>` と書けているのがその証拠です。`w()` は `rect` に定義した `Intrinsic` であり、`this` は `rect` を指しています。`Synthetic` は値を持たないので、`this` を持ちようがありません。

つまり、`Synthetic` は**見た目の階層を作るだけで、評価の文脈は変えない**ということです。ここは直感に反する部分なので、実際に手を動かして確かめておくことをおすすめします（演習1）。

なお、評価の文脈まで移したい場合は `Item` を使います。

```xml
<!-- こちらは _min という「オブジェクト」の子として point2 の Expand が使われる -->
<Item Name="min">_min</Item>
```

## 11.5 診断グループを作る

`Synthetic` のもうひとつの定番用途が、**診断・統計のグループ**です。

`byte_span` に付けてみます。

```xml
<Type Name="vizkit::byte_span">
  <Intrinsic Name="ptr" Expression="_data" />
  <Intrinsic Name="len" Expression="_size" />
  <DisplayString Condition="ptr() == nullptr">[null]</DisplayString>
  <DisplayString Condition="len() == 0">[empty]</DisplayString>
  <DisplayString>{len()} bytes, first={ptr()[0],Xb}</DisplayString>
  <Expand>
    <Item Name="[size]">len()</Item>
    <Synthetic Name="[range]" Condition="ptr() != nullptr">
      <DisplayString>{ptr()} .. {ptr() + len()}</DisplayString>
      <Expand>
        <Item Name="begin">ptr()</Item>
        <Item Name="end">ptr() + len()</Item>
        <Item Name="end (exclusive)">ptr() + len() - 1</Item>
      </Expand>
    </Synthetic>
    <Synthetic Name="[alignment]" Condition="ptr() != nullptr">
      <DisplayString>{(unsigned __int64)ptr() % 16 == 0 ? "16-byte aligned" : "unaligned"}</DisplayString>
      <Expand>
        <Item Name="address">(unsigned __int64)ptr(),X</Item>
        <Item Name="mod 16">(unsigned __int64)ptr() % 16</Item>
      </Expand>
    </Synthetic>
  </Expand>
</Type>
```

```
▷ whole                  6 bytes, first=DE
     [size]              6
   ▷ [range]             0x00000078abcff610 .. 0x00000078abcff616
   ▷ [alignment]         16-byte aligned
   ▷ [Raw View]          {_data=... _size=6 }
```

アラインメントの確認をデバッガ上で一発でできるようになりました。SIMD を扱うライブラリや、アロケータを自作している場合には、こういう行が実際に時間を節約します。

第25章では、ハッシュマップに `[統計]` グループを作り、負荷率・最長チェーン長・空バケット数を並べます。構造はここでやったものと同じです。

## 11.6 Synthetic と Item の使い分け

| やりたいこと | 使う要素 |
|---|---|
| メンバーや式の結果を1行で見せたい | `Item` |
| 式に対応しないメッセージを出したい | `Synthetic` |
| 子行をグループにまとめて畳みたい | `Synthetic` + 内側の `Expand` |
| あるメンバーを、その型の Natvis で展開したい | `Item`（型の `Expand` が適用される） |

判断に迷ったら「**その行を開いたとき、何かのオブジェクトの中身が見えるべきか**」を考えてください。見えるべきなら `Item`、そうでなく単なるまとまりなら `Synthetic` です。

## 11.7 使いすぎない

`Synthetic` は便利ですが、階層を深くしすぎると逆効果です。

デバッグ中の操作は「三角形をクリックする」の繰り返しです。3階層下にある情報は、実質的に見られません。**目安として、階層は2段まで**にしてください。

```
コンテナ
  ├─ [size]                   ← 1段目：すぐ見える
  ├─ [統計]                    ← 1段目（畳まれている）
  │    ├─ 負荷率               ← 2段目：1クリックで見える
  │    └─ 最長チェーン
  └─ [0] .. [n]               ← 1段目：実データ
```

また、`Synthetic` を10個並べるくらいなら、**ビューを分ける**ほうが適切です。次章の次、第13章で扱う `IncludeView` / `ExcludeView` の出番になります。

## 11.8 この章のまとめ

- `<Synthetic Name="...">` は**式に対応しない子行**を作る。値は内側の `<DisplayString>` で与える。
- 中に `<Expand>` を入れられるので、**子行をグループ化して畳める**。要約は `Synthetic` の `DisplayString` に書く。
- **`Synthetic` の内側の式は、親の型の文脈で評価される。** `this` は変わらない。
- 用途は大きく3つ ―― 説明行、診断・統計グループ、階層の整理。
- 「この値が異常なら何を疑うか」を書いておくと、Natvis がチームへの知識配布手段になる。
- **階層は2段まで。** それ以上必要になったら、`Synthetic` を増やすのではなくビューを分ける（第13章）。

## 11.9 演習

1. `rect` の `[geometry]` グループの中に `<Item Name="min.x">_min.x</Item>` を書き、正しく評価されることを確認してください。`Synthetic` が評価の文脈を変えないことの確認です。
2. 同じ場所に `<Item Name="self">this</Item>` を書き、`rect` 自身が表示されることを確認してください。無限に開けてしまうので、確認したら削除してください。
3. `name32` に `[capacity check]` という `Synthetic` を作り、`_len` と上限 31 の関係を要約表示させてください。`_len > 31` のときだけ現れるようにします。
4. `version` に `[bits]` グループを作り、`major` / `minor` / `patch` それぞれのビット幅とシフト量を並べてください。パック形式の仕様書をデバッガ上に置くことになります。
