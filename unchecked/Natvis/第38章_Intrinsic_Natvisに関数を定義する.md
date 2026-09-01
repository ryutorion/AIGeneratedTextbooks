# 第38章 Intrinsic ―― Natvis に関数を定義する

第5章 5.5節で `<Intrinsic>` を「長い式に名前を付ける道具」として導入し、以降ずっと使ってきました。第8部の締めくくりとして、この要素を正式に整理します。

あわせて、公式ドキュメントの記述が誤解を招きやすい点 ―― **「`Intrinsic` には VSIX 拡張が必要」という説明の正しい読み方** ―― を明確にします。

## 38.1 ここまでの使い方

本書で `Intrinsic` を使ってきた場面を振り返ります。

| 章 | 用途 |
|---|---|
| 第5章 | `version` のビット分解に名前を付ける（`major()` / `minor()` / `patch()`） |
| 第7章 | 判定条件に名前を付ける（`is_null()` / `is_empty()`） |
| 第8章 | メンバー名の揺れを吸収する（`Optional="true"` を並べる） |
| 第15章 | `size()` / `capacity()` ―― C++ 側と同じ名前で |
| 第18章 | SBO の判定（`is_inline()`） |
| 第22章 | 赤黒木の不変条件（`bad_red_child()` など） |
| 第34章 | variant のタグ（`index()`） |
| 第37章 | フラグ判定（`has(mask)`） |

一貫しているのは、**式に名前を付けて、複数箇所から参照する**ことです。

理由は3つあります。

**(1) 可読性。** `packed >> 22` より `major()` のほうが意図が明確です。

**(2) 保守性。** 判定式を1か所に定義しておけば、実装を変えたときに直す場所が1つで済みます。`Condition` と `DisplayString` と `Item` の3か所に同じ式が散らばっている Natvis は、必ず不整合を起こします。

**(3) 実装との対応。** C++ 側のメンバー関数と同じ名前を付けておくと、Natvis を読む人が対応を追えます。`STL.natvis` も `size()` / `capacity()` / `isShortString()` と、この流儀です。

## 38.2 属性の一覧

```xml
<Intrinsic Name="名前" Expression="式" ReturnType="型" SideEffect="false" Optional="true" Category="Method">
  <Parameter Name="引数名" Type="型" />
</Intrinsic>
```

| 属性 | 意味 |
|---|---|
| `Name` | **必須。** 有効な C++ 識別子 |
| `Expression` | 関数の戻り値になる式。省略した場合は 38.5節を参照 |
| `ReturnType` | 戻り値の型。省略時は `Expression` から推論される |
| `SideEffect` | `true` ならこの関数が副作用を持ちうることを示す。**既定は `false`** |
| `Optional` | 解析に失敗したら黙って飛ばす（第8章） |
| `Category` | 表示アイコンの分類。既定は `Method` |

子要素として `<Parameter Name="..." Type="..." />` を並べると、引数を取る関数になります。

## 38.3 引数を取る Intrinsic

これが本章の新しい要素です。

```xml
<Type Name="vizkit::render_state">
  <Intrinsic Name="has" Expression="((unsigned)_flags &amp; mask) != 0">
    <Parameter Name="mask" Type="unsigned" />
  </Intrinsic>

  <Expand>
    <Item Name="depth_test"  Condition="has(0x01)">true</Item>
    <Item Name="depth_write" Condition="has(0x02)">true</Item>
    <Item Name="cull_back"   Condition="has(0x04)">true</Item>
    <Item Name="wireframe"   Condition="has(0x08)">true</Item>
    <Item Name="cast_shadow" Condition="has(0x10)">true</Item>
  </Expand>
</Type>
```

`Parameter` の `Name` が `Expression` の中で参照でき、呼び出し側は `has(0x01)` と関数呼び出しの構文で書きます。

引数は複数取れます。

```xml
<Intrinsic Name="at" Expression="_blocks[(i + _offset) / $T2][(i + _offset) % $T2]">
  <Parameter Name="i" Type="int" />
</Intrinsic>
```

```xml
<IndexListItems>
  <Size>_size</Size>
  <ValueNode>at($i)</ValueNode>
</IndexListItems>
```

第28章の `deque` の長い添字計算が、1行に収まりました。**`IndexListItems` の `$i` を、引数付き `Intrinsic` に渡せる**という点に注目してください。

### 実例：ハッシュマップのバケット計算

第26章の `chain_map` に適用します。

```xml
<Type Name="vizkit::chain_map&lt;*,*&gt;">
  <Intrinsic Name="bucket_count" Expression="(int)_bucket_count" />
  <Intrinsic Name="size"         Expression="(int)_size" />
  <Intrinsic Name="head_of" Expression="_buckets[b].head">
    <Parameter Name="b" Type="int" />
  </Intrinsic>

  <DisplayString>{{ size={size()}, buckets={bucket_count()} }}</DisplayString>

  <Expand>
    <Item Name="[size]"         ExcludeView="simple">size()</Item>
    <Item Name="[bucket_count]" ExcludeView="simple">bucket_count()</Item>
    <Item Name="[load_factor]"  ExcludeView="simple">(float)size() / (float)bucket_count()</Item>

    <CustomListItems>
      <Variable Name="b" InitialValue="0" />
      <Variable Name="p" InitialValue="_buckets[0].head" />
      <Size>size()</Size>
      <Loop Condition="b &lt; bucket_count()">
        <Exec>p = head_of(b)</Exec>
        <Loop Condition="p != nullptr">
          <Item Name="[{p->key}]">p->value</Item>
          <Exec>p = p->next</Exec>
        </Loop>
        <Exec>b++</Exec>
      </Loop>
    </CustomListItems>
  </Expand>
</Type>
```

**`CustomListItems` の中でも `Intrinsic` を呼べます。** `<Exec>p = head_of(b)</Exec>` のように、`Exec` の中でも使えます。

第23章 23.4節で「`Exec` では関数を呼べない」と述べましたが、その例外が「デバッガ組み込み関数」でした。**`Intrinsic` で定義した関数も、この例外に入ります。** `__findnonnull`（第26章 26.5節）と同じ扱いです。

これは `CustomListItems` の可読性を大きく改善します。長い式に名前を付けられるので、ループ本体が C++ のコードに近い見た目になります。

## 38.4 Optional との組み合わせ

第8章 8.4節で扱った、レイアウトの差異を吸収するパターンです。

```xml
<Intrinsic Optional="true" Name="ptr" Expression="_data" />
<Intrinsic Optional="true" Name="ptr" Expression="_pointer" />
<Intrinsic Optional="true" Name="len" Expression="_size" />
<Intrinsic Optional="true" Name="len" Expression="_length" />
```

**同じ名前の `Intrinsic` を複数書き、解析できたものが使われます。** 表示ロジックは `ptr()` / `len()` を呼ぶだけになり、レイアウトの差異が上の4行に閉じ込められます。

`STL.natvis` も同じ手法を使っています。

```xml
<Intrinsic Optional="true" Name="allocator" Expression="*((_Mybase *) this)"/>
<Intrinsic Optional="true" Name="allocator" Expression="_Myval"/>
```

## 38.5 Expression を省略した場合

公式ドキュメントには、次の記述があります。

> `<Intrinsic>` 要素には、`IDkmIntrinsicFunctionEvaluator140` インターフェイスを通じて関数を実装するデバッガコンポーネントが伴っていなければなりません。

これを読んで「`Intrinsic` を使うには VSIX 拡張が必要なのか」と考えてしまうと、本書でずっと使ってきた書き方と矛盾します。

正しくはこうです。

| 書き方 | 必要なもの |
|---|---|
| **`Expression` を書く** | **何も要らない。** 純粋な式の別名として動く |
| `Expression` を省略する | デバッガ拡張（VSIX）で実装を提供する必要がある |

本書で使ってきたのは前者です。`STL.natvis` も、すべての `Intrinsic` に `Expression` を書いています。

`Expression` を省略する使い方は、**デバッガ拡張を書いて、Natvis から呼び出せるネイティブ関数を追加する**という高度な用途です。式では表現できない処理（外部プロセスへの問い合わせ、複雑なデコードなど）を Natvis から使いたい場合に登場します。本書の範囲外ですが、**そういう拡張ポイントがある**ことは知っておいてください。

`SideEffect` 属性が存在するのも、この文脈です。式の別名なら副作用はありえませんが、拡張が実装する関数は副作用を持ちうるためです。

## 38.6 制約と注意点

### IncludeView 付きの Type では使えない

報告されている制約として、**`IncludeView` 属性を持つ `<Type>` の中では `<Intrinsic>` が使えない**ことがあります。

```xml
<!-- 動かない可能性がある -->
<Type Name="vizkit::foo" IncludeView="detail">
  <Intrinsic Name="x" Expression="_a + _b" />
  ...
</Type>
```

第13章 13.5節で扱った「`Type` 自体をビューで分ける」書き方と、`Intrinsic` は相性が悪いということです。

回避策は、**ビューで分けない `Type` に `Intrinsic` を置き、ビュー分岐は `Expand` の中の `IncludeView` / `ExcludeView` で行う**ことです。本書がほとんどの章でそうしてきた形です。

### 評価の文脈

`Intrinsic` はコンテナ（`Type` が対象とする型）の文脈で定義されます。したがって、**ノードの文脈で評価される場所では使えないことがあります**。

```xml
<LinkedListItems>
  <NextPointer>next</NextPointer>        <!-- ノードの文脈 -->
  <ValueNode>owner_of()</ValueNode>      <!-- ← コンテナの Intrinsic は呼べない可能性 -->
</LinkedListItems>
```

第19章 19.6節で「確実にしたいなら式を直接書いてください」と述べたのは、この理由です。

**ノードの文脈で使いたい `Intrinsic` は、ノードの `Type` に定義します。**

```xml
<Type Name="vizkit::rb_tree&lt;*,*&gt;::node">
  <Intrinsic Name="is_red" Expression="color == vizkit::rb_color::red" />
  ...
</Type>
```

第22章の赤黒木で、判定をノード型に置いたのはこの設計です。

### 名前の衝突

`Intrinsic` の名前が、その型の実際のメンバーと衝突するとどうなるかは環境依存です。

```cpp
class dynamic_array {
    std::size_t size;      // メンバー変数
};
```

```xml
<Intrinsic Name="size" Expression="(int)(_end - _begin)" />   <!-- 衝突 -->
```

**メンバー関数と同じ名前にするのは推奨されますが、メンバー変数と同じ名前は避けてください。** C++ 側が `size()` というメンバー関数を持ち、Natvis が `size()` という `Intrinsic` を持つ ―― この対応は安全で、読みやすくなります。

## 38.7 設計指針

`Intrinsic` をいつ使うべきかの基準です。

| 状況 | `Intrinsic` にする |
|---|---|
| 同じ式が2か所以上に出てくる | **する** |
| 式が1行に収まらない | **する** |
| 判定条件（`Condition` に書く式） | **する**。名前が付くと意図が伝わる |
| レイアウトの差異を吸収したい | **する**（`Optional` と組み合わせる） |
| 添字計算を `IndexListItems` に渡す | **する**（`Parameter` 付き） |
| 単純なメンバー参照（`_size` など） | しない。かえって遠回りになる |
| 1か所でしか使わない短い式 | しない |

**「Natvis が読みにくいと感じたら、まず `Intrinsic` に切り出せないかを考える。」** これが実務的な指針です。第5章で早めに導入したのも、この効果が大きいからでした。

## 38.8 第8部のまとめ

第34章から第38章までで、個別の型の見せ方を扱いました。

| 型 | 中心となる手法 |
|---|---|
| `optional` / `expected` / `variant` | タグを `Condition` で判定 ＋ `Optional` で存在しない `$Tn` を飛ばす |
| スマートポインタ | `<SmartPointer Usage="Minimal">` で式の中での振る舞いを変える |
| ハンドル / ID | 構造体でラップ ＋ グローバル参照（`Optional` 必須）＋ `[watch]` の提示 |
| enum / ビットフラグ | `en` / `fe` 指定子 ＋ 未知の値の検出 |
| エラーコード | `hr` 指定子 ＋ `<HResult>` の登録 |
| 共通の道具 | `<Intrinsic>`（`Parameter` で引数も取れる） |

第8部を貫く原則です。

1. **無効な側を絶対に読ませない。** `DisplayString` にも `Item` にも `Condition` を付ける（第34章）。
2. **`SmartPointer` は表示ではなく式の振る舞いを変える。** 実際にポインタとして使える型にだけ書く（第35章）。
3. **グローバル参照は失敗しうる。** `Optional` とフォールバック、そして引き方の提示（第36章）。
4. **enum に定義外の値は入る。** 未知のビット・未知の値を検出する行を足す（第37章）。
5. **式に名前を付ける。** 2か所以上に出る式は `Intrinsic` に切り出す（第38章）。

そして第8部でも、第33章 33.5節の結論が繰り返し顔を出しました。**共用体のほうが生バイト配列より書きやすい。名前空間スコープのグローバルのほうが Meyers singleton より引きやすい。型情報を残す設計のほうが型消去より見える。** ―― Natvis で見たいなら、見えるように作る。

次の第9部では、書いた Natvis を**届ける**段階に入ります。ファイルの配置場所、ビルドへの組み込み、PDB への埋め込み、そして最適化ビルドや大規模データとの付き合い方です。

## 38.9 この章のまとめ

- `<Intrinsic Name="..." Expression="...">` は**式に名前を付ける**要素。可読性・保守性・実装との対応の3つが目的。
- 属性は `Name`（必須）/ `Expression` / `ReturnType` / `SideEffect` / `Optional` / `Category`。
- **`<Parameter Name="..." Type="..."/>` で引数を取れる。** 呼び出しは `has(0x01)` のように関数呼び出しの構文。
- **`CustomListItems` の `Exec` や `Condition` の中でも `Intrinsic` を呼べる。** ループ本体が C++ に近い見た目になる。
- 同名の `Intrinsic` を `Optional="true"` で複数並べると、**レイアウトの差異を数行に閉じ込められる**。
- **`Expression` を書けば VSIX は不要。** 「`IDkmIntrinsicFunctionEvaluator140` が必要」という記述は、`Expression` を省略した場合の話。
- `IncludeView` 付きの `Type` では `Intrinsic` が使えないことがある。**ビュー分岐は `Expand` の中で行う。**
- `Intrinsic` はコンテナの文脈で定義される。**ノードの文脈で使いたければ、ノードの `Type` に定義する。**
- メンバー関数と同名にするのは推奨。**メンバー変数と同名は避ける。**
- **Natvis が読みにくいと感じたら、まず `Intrinsic` に切り出せないかを考える。**

## 38.10 演習

1. 第37章の `render_state` に引数付き `Intrinsic` の `has(mask)` を実装し、5つのフラグ判定がすべて `has()` 経由になることを確認してください。
2. `deque` に `<Intrinsic Name="at" ...><Parameter Name="i" Type="int"/></Intrinsic>` を定義し、`IndexListItems` の `ValueNode` を `at($i)` に短縮してください。
3. `chain_map` の `CustomListItems` の中で `head_of(b)` が動くことを確認してください。動かない場合は診断メッセージを読み、直接式に戻して比較してください。
4. `Intrinsic` の名前を、実際のメンバー変数と同じ名前（たとえば `_size` を持つ型で `Name="_size"`）にして、何が起きるか確認してください。
5. `IncludeView="detail"` を持つ `Type` の中に `Intrinsic` を書き、あなたの環境で動作するかを確認してください。動かない場合、`Expand` の中の `IncludeView` に書き換えて解決することも確かめてください。
