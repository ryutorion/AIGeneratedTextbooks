# Natvis入門 — 自作データ型で学ぶ Visual Studio カスタムビジュアライザー

## 本書について

| 項目 | 内容 |
|---|---|
| 環境 | Visual Studio 2026（内部バージョン 18）+ C++23 / MSVC |
| 制約 | **C++モジュールは使わない**（ヘッダー + .cpp の伝統的構成） |
| 形式 | ステップバイステップのハンズオン。全章で手を動かす |
| 読者 | C++実務経験あり／Natvis は未経験 |
| 構成 | 全45章 ＋ 付録A〜G |
| 題材 | 自作コンテナライブラリ **vizkit** を1章ずつ育てながら、対応する natvis を並行して書く |

### 通しの題材ライブラリ「vizkit」

各章で型を1つ追加し、その型を"見やすくする"natvis をその場で書く、という往復を繰り返します。

```
vizkit/
  core/     point2, rgba, fixed_string, strong_id, flags
  seq/      fixed_array, dynamic_array, matrix, ring_buffer, small_vector, deque
  node/     intrusive_list, bst, rb_tree
  assoc/    flat_map, open_map（オープンアドレス法）, chain_map（チェイン法）
  util/     optional_like, expected_like, variant_like, ref_ptr, any_like
  vizkit.natvis   ← 本書を通して育て続ける1ファイル
```

---

# 第1部 はじめの一歩

### 第1章 Natvis とは何か
- デバッガの「素の表示」と「Natvis を当てた表示」を並べて見る
- Natvis が効く場面／効かない場面（型情報・PDB・最適化）
- 本書のゴール像：完成後の vizkit のウォッチウィンドウを先に見せる

### 第2章 環境を用意する
- Visual Studio 2026 のインストール構成と C++23（`/std:c++latest`）の設定
- 題材ライブラリ vizkit のソリューション作成（静的ライブラリ + テスト用コンソール）
- `.natvis` ファイルをプロジェクトに追加し、ビルドに載せる最短手順

### 第3章 デバッガの素の表示を読む
- ローカル／ウォッチ／クイックウォッチ／メモリウィンドウの役割分担
- 生ビュー（Raw View）で見えるメンバーが「natvis から触れる名前」であること
- ポインタ・配列・参照が既定でどう表示されるか

### 第4章 はじめての Natvis
- `point2` に `<DisplayString>` を1行書く
- **`.natvisreload`** — デバッグ中に natvis を再読み込みして即座に反映する
- Natvis 診断メッセージ（Tools > Options > Debugging）を Verbose にして、読み込み・エラーを目で確認する

---

# 第2部 値の表示を作る — DisplayString

### 第5章 DisplayString の式構文
- `{式}` の埋め込みと、`{{` `}}` によるブレースのエスケープ
- 使える式・使えない式（関数呼び出し、副作用、`new`）
- `this` の扱いとメンバーへのアクセス

### 第6章 書式指定子カタログ
- 数値：`d` `x` `o` `b` `e` `hr` `en` `wc`
- 文字列：`s` `sb` `s8` `su` `sub` `bstr`
- 配列：`,[n]` `,[expr]` / ポインタ：`na` `nd` `!`
- `rgba` 型を16進で見せる、`flags` を2進で見せる実践

### 第7章 Condition で表示を出し分ける
- 空／null／不正状態を別文言で見せる
- `Condition` 付き `DisplayString` を複数並べたときの評価順（上から最初にマッチしたもの）
- `dynamic_array` の empty 表示を作る

### 第8章 壊れない natvis の書き方 — Optional
- メンバー名を変えたら natvis が丸ごと死ぬ問題
- `Optional="true"` で「解決できないノードだけ無視する」
- リファクタリング耐性を持たせる設計指針

### 第9章 文字列型と StringView
- `fixed_string<N>` / `string_view_like` の実装
- `<StringView>` とテキストビジュアライザ（虫めがねアイコン）
- 長さ付き文字列・非NUL終端文字列を正しく見せる

---

# 第3部 Expand で中身を組み立てる

### 第10章 Expand と Item
- `<Expand>` `<Item Name="...">` で論理メンバーを定義する
- 実メンバーと論理メンバーを分ける設計（`[size]` `[capacity]` 慣習）
- `Item` にも `Condition` が効く

### 第11章 Synthetic — 説明ノードとグループ化
- 式に対応しない「見出しノード」を作る
- `Synthetic` の中に `Expand` を入れ子にしてグループを作る
- ハッシュマップの `[統計]` グループを先取りで作ってみる

### 第12章 ExpandedItem — 平坦化する
- ラッパー型の中身を親の階層に直接展開する
- 基底クラスのメンバーを引き上げる
- `,nd` 指定子と組み合わせて再帰的マッチを止める

### 第13章 ビューを切り替える
- `IncludeView` / `ExcludeView` と、ウォッチ式の `,view(simple)`
- 既定ビュー／`simple` ビューの設計指針
- `HideRawView` で生ビューを隠す（隠してよい場面・隠すべきでない場面）

---

# 第4部 配列系コンテナを可視化する

### 第14章 fixed_array<T, N> と ArrayItems
- `<ArrayItems>` の `<Size>` と `<ValuePointer>`
- テンプレートの非型引数 `N` を natvis から参照する（`$T2`）
- 要素数が大きいときのデバッガの挙動

### 第15章 dynamic_array<T>
- `_begin / _end / _cap` 方式の実装に対する `Size` の書き方
- `LowerBound` で1始まり配列を表現する
- `[size]` `[capacity]` `[allocator]` の見せ方

### 第16章 matrix<T, Rows, Cols>
- `<Rank>` を使った多次元 `ArrayItems`
- `<Direction>Forward/Backward</Direction>` と行優先／列優先
- 行ごとにグループ表示する別解（`Synthetic` + `IndexListItems`）

### 第17章 ring_buffer<T, N> と IndexListItems
- 論理インデックス `$i` から物理インデックスへの写像を式で書く
- 折り返し（wrap-around）を natvis 側で吸収する
- `ArrayItems` では表現できないケースの見分け方

### 第18章 small_vector<T, N> — SBO を見せる
- インライン格納／ヒープ格納を `Condition` で分岐した2つの `ArrayItems`
- 現在どちらのモードかを `[mode]` として表示する
- union / `alignas` バッファをキャストして読む

---

# 第5部 ノード構造を辿る

### 第19章 intrusive_list と LinkedListItems
- `<HeadPointer>` `<NextPointer>` `<ValueNode>` の3点セット
- フック（`list_hook`）から要素本体へ戻る計算（`offsetof` 相当を式で書く）
- 要素数を `<Size>` で明示する／しない

### 第20章 双方向リストと番兵ノードの罠
- 番兵（sentinel）を1周分よけいに列挙してしまう典型ミス
- `ValueNode` に `this` を書くべき場面
- 循環・破損リストで natvis が固まらないようにする

### 第21章 bst<K, V> と TreeItems
- `<LeftPointer>` `<RightPointer>` `<ValueNode>` による中順走査
- key と value をまとめて `key => value` の1行に見せる
- ノード数を持たない木で `Size` をどうするか

### 第22章 赤黒木を見やすくする
- 色・親ポインタ・不変条件をデバッグ時に確認できる形にする
- `[黒高さ]` のような検証用ノードを Synthetic で足す
- `std::map` の見せ方と比較する

---

# 第6部 CustomListItems — 複雑なデータ構造に踏み込む

### 第23章 CustomListItems 入門
- `<Variable>` `<Exec>` `<Loop>` `<Break>` `<If>` `<Elseif>` `<Else>` `<Item>`
- natvis の中の「小さなプログラミング言語」としての文法
- `<Size>` を先に確定させる／させないの違い

### 第24章 flat_map — ソート済みベクタ
- `pair` の配列を `key => value` として列挙する
- `<Item Name="{...}">` で Name 側にも式を書く
- キー型が文字列のときの書式指定

### 第25章 open_map — オープンアドレス法ハッシュマップ
- 空スロット・墓石（tombstone）を `If` でスキップする
- 実要素数 `[size]` と容量 `[capacity]`、負荷率の表示
- 制御バイト方式（SwissTable 風）への応用

### 第26章 chain_map — チェイン法ハッシュマップ（unordered_map 相当）
- **バケット配列 × 各バケットのノードリスト**という二重ループを `CustomListItems` で書く
- バケット単位で見る／全要素をフラットに見る、2つのビューを用意する
- `IncludeView` で「デバッグ用の内部構造ビュー」を分ける

### 第27章 STL.natvis を読む
- MSVC 同梱 `STL.natvis` の `std::unordered_map` / `std::map` / `std::deque` 定義を読み解く
- 実装依存メンバー名（`_Mypair._Myval2...`）との付き合い方
- `<Intrinsic>` が使われている箇所の意図

### 第28章 segmented deque とループ上限
- ブロック配列 + ブロック内オフセットの二段構造を辿る
- 巨大コンテナで展開が重くなる問題と、途中打ち切り（`[...]` 表示）の作法
- `Loop Condition` の設計と無限ループ防止

---

# 第7部 テンプレートクラスへの対応

### 第29章 ワイルドカードとマッチング規則
- `Type Name="vizkit::dynamic_array&lt;*&gt;"` の書き方と名前空間・エイリアス
- 複数ワイルドカード、可変長テンプレートのマッチ
- typedef に効かない、プリミティブ型に効かないという制約

### 第30章 $T1〜$Tn を使いこなす
- 型引数・非型引数の参照、`sizeof($T1)` などの式
- テンプレート引数を使ったキャスト（`(($T1*)_data)[$i]`）
- アロケータ引数がある型を壊さずに書く

### 第31章 特殊化と優先順位
- 汎用ルールと特殊化ルールを共存させる
- `Priority`（Low / MediumLow / Medium / MediumHigh / High）の実際の効き方
- `AlternativeType` で別名型・実装型に同じルールを当てる

### 第32章 継承と Inheritable
- `Inheritable="false"` が必要になる場面
- 基底クラス部分を `ExpandedItem` で見せる／`Item Name="[base]"` で畳む
- 多重継承・仮想継承での注意点

### 第33章 型消去・CRTP・ポリシークラス
- `any_like` / `function_like` の中身を vtable ポインタ経由で覗く
- CRTP 基底を派生型として表示する
- ポリシーテンプレートで実装が切り替わる型に、1つの natvis で対応する

---

# 第8部 高度な型を扱う

### 第34章 optional / expected / variant 風の型
- タグ付き共用体を `Condition` で正しく分岐する
- 未初期化領域を読んでしまわない書き方
- C++23 `std::expected` の実表示と比較する

### 第35章 スマートポインタ
- `<SmartPointer>` 要素の効果（`Usage` 属性、ウォッチでの `->` 展開）
- `ref_ptr<T>` に参照カウントを `[refs]` として見せる
- 循環参照の検出に役立つ表示を足す

### 第36章 ハンドル／ID 型と外部テーブル参照
- `strong_id<Tag>` の生の数値を意味のある表示に変える
- グローバルなプール／レジストリを natvis から引く方法と、その限界
- 引けない場合に「引くためのウォッチ式」を `Synthetic` で提示するテクニック

### 第37章 ビットフラグ・enum・HResult
- `flags<E>` を `A|B|C` 形式で表示する
- `en` 指定子と enum の欠損値
- `<HResult>` 要素でエラーコードに説明を付ける

### 第38章 Intrinsic — natvis に関数を定義する
- `<Intrinsic Name="..." Expression="..."/>` による式の再利用
- 長い式を名前付きにして natvis の可読性を上げる
- chain_map のバケット計算を Intrinsic に切り出す／`ReturnType` と制約

---

# 第9部 運用・配布・トラブルシューティング

### 第39章 natvis ファイルの探索順序と配置場所
- PDB 埋め込み > プロジェクト内 > VSIX > ユーザー別 > マシン全体、の優先順位
- **Visual Studio 2026 のユーザー別フォルダは `Documents\Visual Studio 18\Visualizers`**（2022 までとパスが変わっている点）
- 複数の natvis が同じ型を定義したときの勝ち負け

### 第40章 MSBuild プロジェクトへの組み込みと PDB 埋め込み
- `.natvis` を `Natvis` アイテムとして扱う／`Excluded From Build` の意味
- ライブラリ利用者に natvis を届ける「PDB 埋め込み」パターン
- 静的ライブラリ／DLL それぞれでの届き方の違い

### 第41章 CMake プロジェクトへの組み込み
- `target_sources` / `INTERFACE` で natvis を配布する
- インストール時に natvis を同梱する
- vcpkg ポートに natvis を含める場合の考え方

### 第42章 診断メッセージとトラブルシューティング
- Natvis 診断メッセージの読み方（Error / Warning / Verbose）
- 典型エラー逆引き：型がマッチしない／式が評価できない／展開が空になる
- 「natvis が当たっていないのか、当たって空なのか」の切り分け手順

### 第43章 最適化ビルドと大規模データ
- Release ビルドで変数が消える・インライン化される問題
- `/Zo`、Edit and Continue、最適化と natvis の相性
- 数十万要素のコンテナで展開コストを抑える設計（`Size` 上限、遅延展開、専用ビュー）

### 第44章 他ツールでの互換性
- VS Code + cppvsdbg / cppdbg、LLDB の natvis 対応範囲
- WinDbg（`.nvload`）での利用
- Windows 以外でも壊れない書き方の指針

### 第45章 natvis のテストと CI
- XSD スキーマによる構文検証をビルドに組み込む
- 「期待する表示文字列」を自動で突き合わせる回帰テストの考え方
- チームで natvis を育てるためのレビュー観点チェックリスト

---

# 付録

- **付録A** Natvis 要素・属性 完全リファレンス
- **付録B** 書式指定子リファレンス（数値／文字列／配列／ポインタ）
- **付録C** vizkit 全ソースコード一覧
- **付録D** 完成版 `vizkit.natvis` 全文（コメント付き）
- **付録E** `.natstepfilter` と `.natjmc` — ステップ実行とコールスタックも整える
- **付録F** `STL.natvis` 読みどころ抜粋
- **付録G** エラーメッセージ逆引き集
