# 第27章 STL.natvis を読む

ここまでの章で、Natvis の主要な要素をひととおり使いました。**他人が書いた Natvis を読める**段階に来ています。

読む対象として最良の教材が、MSVC に同梱されている `STL.natvis` です。標準ライブラリのすべてのコンテナに対する、実戦的な Natvis がまとまっています。この章では、その読み方と、そこから学べる設計判断を扱います。

## 27.1 ファイルの場所

`STL.natvis` は、Visual Studio のインストールフォルダにあります。

```
<Visual Studio インストール先>\Common7\Packages\Debugger\Visualizers\STL.natvis
```

第39章で詳しく扱いますが、ここは**マシン全体の Natvis ディレクトリ**です。すべてのプロジェクトに自動的に適用されます。

同じディレクトリには、`concurrency.natvis`、`atlmfc.natvis`、`winrt.natvis` など、他のライブラリ用のファイルも置かれています。どれも読む価値があります。

最新版は GitHub の [microsoft/STL](https://github.com/microsoft/STL) リポジトリの `stl/debugger/STL.natvis` でも公開されています。手元のバージョンと差異があることもあるので、実際に動いているものを読むなら、インストールフォルダのほうを開いてください。

> **編集しないこと**
> このファイルを直接書き換えると、Visual Studio の更新で上書きされますし、他のプロジェクトにも影響します。**読むだけ**にしてください。特定の型の表示を変えたいなら、自分のプロジェクトに `.natvis` を追加し、`Priority` 属性で優先させます（第31章）。

## 27.2 読み方の作法

Natvis を読むときは、必ず**実物と突き合わせます**。

1. その型の変数を作ってブレークポイントで止める
2. ウォッチウィンドウで `[Raw View]` を開く（第3章 3.4節）
3. `STL.natvis` の該当箇所を開き、式に出てくる名前を `[Raw View]` で確認する

`_Mypair._Myval2._Myfirst` のような長い名前は、読んでいるだけでは意味が分かりません。実物を開けば、それがどこにある何なのかが一目で分かります。

**Natvis は「読解」ではなく「照合」で読む。** これがこの章の唯一の作法です。

## 27.3 std::vector ―― 三ポインタ表現

第15章で書いた `dynamic_array` と比べてください。

```xml
<Item Name="[size]" ExcludeView="simple">_Mylast - _Myfirst</Item>
<Item Name="[capacity]" ExcludeView="simple">_Myend - _Myfirst</Item>
```

| | `STL.natvis` | 本書の `dynamic_array` |
|---|---|---|
| 先頭 | `_Myfirst` | `_begin` |
| 終端 | `_Mylast` | `_end` |
| 容量端 | `_Myend` | `_cap` |
| `[size]` | `_Mylast - _Myfirst` | `size()`（`Intrinsic`） |
| `ExcludeView="simple"` | あり | あり |

**構造は同一です。** 名前が違うだけです。三ポインタ表現に対する `ArrayItems` の当て方は、実装が誰であっても変わりません。

`_Mypair._Myval2.` という前置きが省かれているのは、`Type` の直下で `ExpandedItem` や `Intrinsic` を使って文脈を移しているためです。`_Mypair` は「アロケータと値を空基底最適化で1つに詰め込む」ための入れ物で、標準ライブラリの実装都合です。

## 27.4 std::list ―― 番兵の次

```xml
<Type Name="std::list&lt;*&gt;">
  <DisplayString>{{ size={_Mypair._Myval2._Mysize} }}</DisplayString>
  <Expand>
    <Item Name="[allocator]" ExcludeView="simple">_Mypair</Item>
    <LinkedListItems>
      <Size>_Mypair._Myval2._Mysize</Size>
      <HeadPointer>_Mypair._Myval2._Myhead-&gt;_Next</HeadPointer>
      <NextPointer>_Next</NextPointer>
      <ValueNode>_Myval</ValueNode>
    </LinkedListItems>
  </Expand>
</Type>
```

第20章で書いた `list` と照合してください。

- **`HeadPointer` が `_Myhead->_Next`** ―― 番兵ではなく「番兵の次」（第20章 20.3節）
- **`Size` が必ず書かれている** ―― 循環リストなので省略できない（第20章 20.3節）
- `NextPointer` は `_Next`、`ValueNode` は `_Myval` ―― どちらもノードの文脈（第19章 19.3節）

そして `[allocator]` の `Item` に注目してください。第12章 12.7節で述べた「実装詳細の基底は平坦化せず、1行に押し込める」という判断が、ここで実際に行われています。アロケータは見たいことがまれなので、`ExcludeView="simple"` も付いています。

## 27.5 std::map ―― 番兵つきの木

第21章 21.7節で引用したものです。

```xml
<TreeItems>
  <Size>_Mysize</Size>
  <HeadPointer>_Myhead-&gt;_Parent</HeadPointer>
  <LeftPointer>_Left</LeftPointer>
  <RightPointer>_Right</RightPointer>
  <ValueNode Condition="!((bool)_Isnil)">_Myval</ValueNode>
</TreeItems>
```

`std::map` は赤黒木ですが、色の表示はありません。**ユーザーが知りたいのは中身であって、実装の内部状態ではない**という判断です。

第22章で `rb_tree` に色や不変条件の表示を付けたのは、**自分がその木の実装をデバッグする立場だから**です。ライブラリの利用者に見せる Natvis と、実装者が自分のために書く Natvis は、目的が違います。

この使い分けは、第13章のビューで両立できます。既定は利用者向け、`detail` ビューは実装者向け、という設計です。

## 27.6 std::unordered_map ―― 意外な実装

本書の第26章で二重ループを苦労して書いたので、`std::unordered_map` の定義を見ると驚くはずです。

```xml
<Type Name="std::unordered_map&lt;*&gt;" Priority="Medium">
  <AlternativeType Name="std::unordered_multimap&lt;*&gt;" />
  <AlternativeType Name="std::unordered_set&lt;*&gt;" />
  <AlternativeType Name="std::unordered_multiset&lt;*&gt;" />
  <DisplayString>{_List}</DisplayString>
  <Expand>
    <Item Name="[bucket_count]" IncludeView="detailed">_Maxidx</Item>
    <Item Name="[load_factor]" IncludeView="detailed">
      ((float)_List._Mypair._Myval2._Mysize) / ((float)_Maxidx)
    </Item>
    <Item Name="[max_load_factor]" IncludeView="detailed">_Traitsobj._Mypair._Myval2._Myval2</Item>
    <Item Name="[hash_function]" ExcludeView="simple">_Traitsobj._Mypair</Item>
    <Item Name="[key_eq]" ExcludeView="simple">_Traitsobj._Mypair._Myval2</Item>
    <Item Name="[allocator]" ExcludeView="simple">_List._Mypair</Item>
    <ExpandedItem>_List,view(simple)</ExpandedItem>
  </Expand>
</Type>
```

**`CustomListItems` がありません。** 二重ループもありません。

理由は実装にあります。MSVC の `std::unordered_map` は、**全要素を1本の `std::list` に格納**し、バケット配列にはそのリストへのイテレータを持たせる、という設計です。

```
_List  ──▶ 全要素が1本のリストに並んでいる（同じバケットの要素は隣接）
_Vec   ──▶ 各バケットの範囲を示すイテレータの配列
```

したがって、要素を列挙するだけなら `_List` をそのまま見せればよい。`<ExpandedItem>_List,view(simple)</ExpandedItem>` の1行で済みます。

### ここから学べること

**(1) データ構造の設計が、Natvis の書きやすさを決める。**

同じ「チェイン法のハッシュマップ」でも、要素を1本のリストに集める設計なら Natvis は1行、バケットごとにリストを分ける設計なら二重ループが要ります。第6章 6.8節・第19章 19.6節でも触れた「Natvis を書きやすい設計」という観点が、最も大きく効いた例です。

**(2) 委譲できるなら委譲する。**

`std::unordered_map` の Natvis は、要素の列挙を `std::list` の Natvis に丸ごと任せています。`DisplayString` すら `{_List}` です。**自分で書かずに済ませられる部分は、他の型の Natvis に委ねる。** 保守の量が減ります。

本書の `chain_map` でも、バケット型に `LinkedListItems` を書いて委譲する形を取りました（第26章 26.4節）。同じ発想です。

**(3) `AlternativeType` で4つの型をまとめている。**

`unordered_map` / `unordered_multimap` / `unordered_set` / `unordered_multiset` は内部構造が同じなので、1つの定義を共有しています。`<AlternativeType>` は「この定義を別の型にも適用する」という要素です（第31章で詳しく扱います）。

**(4) `Priority="Medium"` が付いている。**

複数の定義がマッチしうる場合の優先度です。より特殊化された定義（たとえば特定のキー型に対するもの）を上書きさせるために、あえて中優先度にしてあります（第31章）。

**(5) ビュー名が `detailed`。**

本書では `detail` を使ってきましたが、`STL.natvis` は `detailed` です。**ビュー名に標準はありません。** `simple` だけが事実上の慣習になっています。他人の Natvis を使うときは、ビュー名を確認する必要があります。

## 27.7 std::deque ―― 二段構造を IndexListItems で

```xml
<Type Name="std::deque&lt;*&gt;">
  <Intrinsic Name="size" Expression="_Mypair._Myval2._Mysize" />
  <DisplayString>{{ size={size()} }}</DisplayString>
  <Expand>
    <Item Name="[allocator]" ExcludeView="simple">_Mypair</Item>
    <IndexListItems>
      <Size>size()</Size>
      <ValueNode>
        _Mypair._Myval2._Map[(($i + _Mypair._Myval2._Myoff) / _EEN_DS) % _Mypair._Myval2._Mapsize]
                            [($i + _Mypair._Myval2._Myoff) % _EEN_DS]
      </ValueNode>
    </IndexListItems>
  </Expand>
</Type>
```

`std::deque` はブロック（チャンク）の配列で構成される二段構造です。第28章で自作します。

注目点は、**`CustomListItems` を使っていない**ことです。二段構造でありながら、`IndexListItems` で書けています。

```
論理インデックス $i
  ↓  + _Myoff（先頭のオフセット）
  ↓  / _EEN_DS  → ブロック番号
  ↓  % _EEN_DS  → ブロック内オフセット
```

**添字の変換で位置が求まるなら `IndexListItems` で足りる。** 第17章 17.7節の使い分け表のとおりです。ブロックサイズ `_EEN_DS` が定数だからこそ、この計算が成立しています。

`_EEN_DS` はブロックあたりの要素数を表す、デバッガから見える定数です。実装内部の enum として定義されており、Natvis から名前で参照できます。**Natvis からは、クラス内の enum 定数も参照できる**ということです。

第26章の `chain_map` で `CustomListItems` が必要だったのは、チェーンの長さが可変だからでした。**可変なものを辿るときだけ `CustomListItems` が要る。** この境界がはっきりします。

## 27.8 STL.natvis から学べる設計判断

読み込んで見えてくる、実戦的な判断をまとめます。

| 判断 | `STL.natvis` の実例 |
|---|---|
| 実装詳細は1行に押し込める | `[allocator]` を `Item` にして `ExcludeView="simple"` |
| 診断情報は別ビューに置く | `[bucket_count]` `[load_factor]` を `IncludeView="detailed"` |
| 委譲できるなら委譲する | `unordered_map` → `{_List}` と `ExpandedItem` |
| 同構造の型はまとめる | `AlternativeType` で4型を共有 |
| 環境差は `Optional` で吸収 | `<Intrinsic Optional="true" Name="allocator" ...>` を2つ並べる |
| 長い式には名前を付ける | `<Intrinsic Name="size" ...>` |
| 利用者に内部状態は見せない | `std::map` に色の表示はない |
| 番兵は `HeadPointer` を一段間接に | `_Myhead->_Next`、`_Myhead->_Parent` |
| 専用要素で書けるなら使う | `deque` は `IndexListItems`、`CustomListItems` ではない |

このリストは、本書がここまでの章で少しずつ立ててきた原則と、ほぼ一致しています。**Natvis の実務的な作法は、標準ライブラリの Natvis を読めば裏付けが取れる**ということです。

## 27.9 他人のライブラリに Natvis を書くとき

`STL.natvis` を読む経験は、そのまま**サードパーティのライブラリに Natvis を書く**作業に応用できます。手順は次のとおりです。

1. **対象の型の変数を作り、`[Raw View]` を開く。** 内部構造を把握する。
2. **`sizeof` とメモリウィンドウでレイアウトを確認する**（第3章 3.7節）。ポインタと長さの対応を推測する。
3. **ウォッチウィンドウで式を試す**（第4章 4.3節）。`ptr,[len]` で配列として見えるかを確かめる。
4. **`Optional="true"` を厚めに付ける。** 他人のライブラリはバージョンで内部が変わります。自分のライブラリとは判断が逆になります（第8章 8.5節）。
5. **バージョンを限定する。** `<Version Min="..." Max="..."/>` で、想定バージョン以外には適用しないようにできます（第31章）。

そして最も大事な点。**その Natvis はあなたのプロジェクトの `.natvis` に書く。** ライブラリ側のファイルを書き換えてはいけません。

## 27.10 この章のまとめ

- `STL.natvis` は `<VS インストール先>\Common7\Packages\Debugger\Visualizers\` にある。**読むだけで、編集しない。**
- 読み方は「読解」ではなく **「`[Raw View]` との照合」**。
- `std::vector` / `std::list` / `std::map` は、本書で書いてきたものと**構造が同一**。名前が違うだけ。
- **`std::unordered_map` には `CustomListItems` がない。** 全要素を1本の `std::list` に持つ実装なので、`ExpandedItem` 1行で済んでいる。
- ここから学べる最大の教訓は、**データ構造の設計が Natvis の書きやすさを決める**こと、そして**委譲できるなら委譲する**こと。
- `std::deque` は二段構造だが `IndexListItems` で書ける。**添字の変換で位置が求まるなら `CustomListItems` は不要。**
- `ExcludeView="simple"` / `IncludeView="detailed"` / `AlternativeType` / `Priority` / `Optional` / `Intrinsic` ―― 本書で扱ってきた要素の実戦例がすべて詰まっている。
- ビュー名に標準はない。`simple` だけが事実上の慣習。
- 他人のライブラリに Natvis を書くときは `Optional` を厚めに、`Version` で限定し、**自分のプロジェクトのファイルに書く**。

## 27.11 演習

1. インストールフォルダの `STL.natvis` を開き、`std::shared_ptr` の定義を探して読んでください。`<SmartPointer>` 要素が出てきます（第35章で扱います）。
2. `std::unordered_map<std::string, int>` の変数を作り、既定表示・`,view(simple)`・`,view(detailed)` の3つを比べてください。
3. 同じ変数の `[Raw View]` を開き、`_List` と `_Vec` の関係を確認してください。全要素が1本のリストに並んでいることを目で確かめます。
4. `std::deque<int>` を作り、ウォッチウィンドウで `_EEN_DS` を評価してください。ブロックあたりの要素数が分かります。要素型を `double` や大きな構造体に変えると値がどう変わるかも試してください。
5. `std::vector<bool>` の定義を探してください。他の `vector` とまったく別の書き方になっているはずです。理由を説明してください。
