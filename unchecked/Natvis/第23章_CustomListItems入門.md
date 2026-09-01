# 第23章 CustomListItems 入門

第6部に入ります。ここが本書の中心です。

第4部・第5部で使ってきた `ArrayItems` / `IndexListItems` / `LinkedListItems` / `TreeItems` は、いずれも**決まった形のデータ構造を宣言的に辿る**要素でした。形が想定と違えば手が出ません。

`<CustomListItems>` は、**走査の手続きを自分で書く**ための要素です。変数を宣言し、ループを回し、条件で分岐し、値を更新する。Natvis の中に小さなプログラミング言語が用意されている、と考えてください。

## 23.1 公式ドキュメントの位置づけ

まず、いつ使うべきかの基準です。

> `CustomListItems` 展開を使うと、ハッシュテーブルのようなデータ構造を走査するカスタムロジックを書けます。評価に必要なすべてを C++ の式で表現できるが、`ArrayItems`、`IndexListItems`、`LinkedListItems` の型にはうまく当てはまらないデータ構造を可視化するには、`CustomListItems` を使ってください。

「うまく当てはまらないとき」という条件が明示されています。**当てはまるなら専用要素を使うべき**です。理由は第14章 14.8節・第17章 17.7節で述べた性能です。`CustomListItems` は要素ごとに複数の式を評価するので、最も遅い列挙方法になります。

第6部を通じての判断基準を先に置いておきます。

| 構造 | 使う要素 |
|---|---|
| 要素が連続している | `ArrayItems` |
| 添字の変換で位置が求まる | `IndexListItems` |
| 単一のポインタで繋がっている | `LinkedListItems` |
| 左右2本のポインタの木 | `TreeItems` |
| **飛ばす要素がある**（空スロット、墓石） | **`CustomListItems`** |
| **入れ子の走査が要る**（バケット × チェーン） | **`CustomListItems`** |
| **辿りながら計算が要る**（統計、検証） | **`CustomListItems`** |

## 23.2 最小の CustomListItems

いきなり複雑な構造に取り組む前に、第19章の `forward_list` を `CustomListItems` で書き直してみます。結果は同じですが、機構が分かります。

```xml
<Type Name="vizkit::forward_list&lt;*&gt;">
  <DisplayString>{{ size={_size} }}</DisplayString>
  <Expand>
    <CustomListItems>
      <Variable Name="p" InitialValue="_head" />
      <Size>_size</Size>
      <Loop Condition="p != nullptr">
        <Item>p->value</Item>
        <Exec>p = p->next</Exec>
      </Loop>
    </CustomListItems>
  </Expand>
</Type>
```

```
▷ fl             { size=3 }
     [0]         10
     [1]         20
     [2]         30
```

C++ で書けば、こうなります。

```cpp
node* p = _head;
while (p != nullptr) {
    yield p->value;
    p = p->next;
}
```

**ほぼそのままの対応**です。`Item` が「1つ出力する」に相当します。

`LinkedListItems` なら4行で済んだものが6行になりました。この型では `CustomListItems` を使う理由がありません。しかしこの後の章では、この手続きの中に `If` や内側の `Loop` が入ってきます。

## 23.3 実行モデル

要素を整理します。

| 要素 | 役割 |
|---|---|
| `<Variable Name="..." InitialValue="..."/>` | 変数を宣言する。**ループの外**に置く |
| `<Size>` | 要素数（省略可） |
| `<Exec>式</Exec>` | 式を実行する。**代入ができる** |
| `<Loop Condition="...">` | 条件が真の間、中身を繰り返す |
| `<Break Condition="..."/>` | 最も内側の `Loop` を抜ける |
| `<If Condition="..."> / <Elseif Condition="..."> / <Else>` | 分岐 |
| `<Item Name="...">式</Item>` | **1件出力する** |

処理の流れは上から下へ、`Loop` に入ったら条件が偽になるか `Break` に当たるまで繰り返し、`Item` に到達するたびに1行が生成されます。

### 変数の宣言はループの外

`<Variable>` は `CustomListItems` の直下に置きます。ループの内側では宣言できません。C++ でいえば、すべての変数を関数の先頭でまとめて宣言する古い流儀に近い形です。

```xml
<CustomListItems>
  <Variable Name="i" InitialValue="0" />
  <Variable Name="p" InitialValue="_head" />
  <Size>_size</Size>
  <Loop Condition="p != nullptr">
    ...
  </Loop>
</CustomListItems>
```

`InitialValue` はコンテナの文脈で評価されます。ここで `_head` のようなメンバーを参照できます。

**変数の型は `InitialValue` から推論されます。** `InitialValue="0"` なら整数、`InitialValue="_head"` ならノードへのポインタです。型を明示する構文はありません。

## 23.4 Exec で書けること・書けないこと

公式ドキュメントの記述が明快です。

> `Exec` を使って `CustomListItems` 展開の内部でコードを実行できます。展開内で定義した変数やオブジェクトを使えます。`Exec` では論理演算子、算術演算子、代入演算子を使えます。**関数を評価するために `Exec` を使うことはできません**。ただし「C++ 式エバリュエーターがサポートするデバッガ組み込み関数」は例外です。

| できる | 例 |
|---|---|
| 代入 | `<Exec>p = p->next</Exec>` |
| 複合代入・インクリメント | `<Exec>i++</Exec>`、`<Exec>total += 1</Exec>` |
| 算術・論理・ビット演算 | `<Exec>idx = (idx + 1) % cap</Exec>` |
| キャスト | `<Exec>n = ($T1*)raw</Exec>` |
| デバッガ組み込み関数 | `<Exec>k = __findnonnull(p, n)</Exec>`（第26章） |

| できない | 代替 |
|---|---|
| ユーザー定義関数の呼び出し | 中身の式を手で書く |
| `operator++` などの演算子オーバーロード | イテレータのロジックを手で再実装する |
| **クラス型オブジェクトの生成** | 基本型とポインタだけで状態を持つ |

最後の「クラス型オブジェクトを作れない」が実務上の制約になります。**イテレータをそのまま使えないので、イテレータの内部状態を整数とポインタの組に分解して自分で回す**ことになります。第26章のハッシュマップで、この作業を実際にやります。

## 23.5 Loop の2つの書き方

`Loop` には2つの流儀があります。

**(a) 条件付きループ**

```xml
<Loop Condition="p != nullptr">
  <Item>p->value</Item>
  <Exec>p = p->next</Exec>
</Loop>
```

**(b) 無限ループ ＋ Break**

```xml
<Loop>
  <Break Condition="p == nullptr" />
  <Item>p->value</Item>
  <Exec>p = p->next</Exec>
</Loop>
```

どちらでも構いません。ただし**脱出条件が複数ある場合は (b) が読みやすくなります**。公式ドキュメントの `CAtlMap` の例も (b) の形です。

```xml
<Loop>
  <If Condition="pBucket == nullptr">
    <Exec>iBucket++</Exec>
    <Exec>iBucketIncrement = __findnonnull(m_ppBins + iBucket, m_nBins - iBucket)</Exec>
    <Break Condition="iBucketIncrement == -1" />
    <Exec>iBucket += iBucketIncrement</Exec>
    <Exec>pBucket = m_ppBins[iBucket]</Exec>
  </If>
  <Item>pBucket,na</Item>
  <Exec>pBucket = pBucket->m_pNext</Exec>
</Loop>
```

「バケットを使い切ったら次の非 NULL バケットを探し、見つからなければ終了」という脱出が `Break` で表現されています。

## 23.6 ループが黙って終わる ―― 最大の落とし穴

`Loop` の停止条件は、実は3つあります。

1. `Condition` が偽になった
2. `Break` に到達した
3. **中の式の評価に失敗した**

3番目が問題です。仕様として「`Break` 要素または**式評価の失敗**まで繰り返す」と定められています。

つまり、`p->next` の評価に失敗しても（`p` が不正なアドレスを指しているなど）、**エラーは表示されず、ループが静かに終わります**。

```
     [0]        10
     [1]        20
                        ← ここで終わり。3件目以降が出ない
```

「なぜか途中までしか出ない」という症状の多くは、これが原因です。この場合の切り分け手順は次のとおりです。

1. `Size` を書いているなら、一時的に外す（`Size` と実際の件数の食い違いを見る）
2. 途中で止まった位置の変数を `Item` で出力してみる
3. ウォッチウィンドウで、その位置の式を手で評価する

デバッグ用に、ループ変数そのものを出す `Item` を仕込むのが有効です。

```xml
<Loop Condition="p != nullptr">
  <Item Name="[dbg p]" >p</Item>          <!-- 開発中だけ有効にする -->
  <Item>p->value</Item>
  <Exec>p = p->next</Exec>
</Loop>
```

**`CustomListItems` は Natvis の中で最もデバッグしにくい要素です。** 診断メッセージも限定的なので、「変数を表示して確かめる」という原始的な方法が最も確実です。

## 23.7 無限ループを防ぐ

`Exec` を書き忘れたり、リストが循環していたりすると、ループが終わりません。デバッガが応答しなくなります。

**必ずカウンタによる保険を入れてください。**

```xml
<CustomListItems>
  <Variable Name="p" InitialValue="_head" />
  <Variable Name="guard" InitialValue="0" />
  <Size>_size</Size>
  <Loop Condition="p != nullptr">
    <Break Condition="guard &gt; 100000" />
    <Item>p->value</Item>
    <Exec>p = p->next</Exec>
    <Exec>guard++</Exec>
  </Loop>
</CustomListItems>
```

`Size` を書いている場合、デバッガは必要な件数だけ生成して打ち切るので、ある程度は守られます。しかし `Size` 自体が壊れていることもあります。**カウンタの `Break` は、`CustomListItems` を書くときの定型句**として身につけてください。

本書ではこれ以降、紙面の都合で `guard` を省略した例も出しますが、実際に書くときは必ず入れてください。

## 23.8 MaxItemsPerView

`CustomListItems` には `MaxItemsPerView` 属性があります。

```xml
<CustomListItems MaxItemsPerView="5000">
```

- 一度に表示するコレクション要素の最大数
- 指定できる範囲は 1〜50000
- **既定値は 5000**
- 超えた分は末尾に特別なノードが作られ、開くと続きが見られる

巨大なコンテナで既定の 5000 が重いと感じるなら、小さくできます。逆に、どうしても一度に見たいなら増やせます。

この属性は `CustomListItems` にのみ存在します。`ArrayItems` などにはありません（そちらはデバッガ側の表示仮想化に任されています）。

## 23.9 第22章の宿題 ―― 黒高さを計算する

第22章 22.6節で「ループがないので書けない」とした黒高さを、いま書けます。

```xml
<Type Name="vizkit::rb_tree&lt;*,*&gt;">
  ...
  <Expand>
    ...
    <Synthetic Name="[invariants]" IncludeView="detail" Condition="_root != nullptr">
      <DisplayString>root is {_root->color,en}</DisplayString>
      <Expand>
        <CustomListItems>
          <Variable Name="p"  InitialValue="_root" />
          <Variable Name="bh" InitialValue="0" />
          <Variable Name="guard" InitialValue="0" />
          <Loop Condition="p != nullptr">
            <Break Condition="guard &gt; 1000" />
            <If Condition="p->color == vizkit::rb_color::black">
              <Exec>bh = bh + 1</Exec>
            </If>
            <Exec>p = p->left</Exec>
            <Exec>guard++</Exec>
          </Loop>
          <Item Name="[black height (leftmost)]">bh</Item>
        </CustomListItems>
      </Expand>
    </Synthetic>
  </Expand>
</Type>
```

```
tree,view(detail)
  └ [invariants]                 root is black
       [black height (leftmost)] 3
```

注目すべき構造が2つあります。

**(1) ループの外に `Item` を置いている。** `CustomListItems` は「複数の要素を列挙する」ためのものですが、**ループを回して計算し、結果を1件だけ出力する**という使い方もできます。統計値の算出に使う定型パターンです。第25章・第26章で多用します。

**(2) 最左パスしか見ていない。** 赤黒木の性質上、黒高さは全パスで等しいはずなので、1本調べれば値は分かります。「全パスが等しいか」を検証するには全探索が必要で、それには明示的なスタック（配列変数）が要ります。Natvis の `Variable` は配列を宣言できないため、**この検証は Natvis では現実的ではありません**。

道具の限界を見極めるのも、実務では重要です。`CustomListItems` は万能ではありません。

## 23.10 現時点のまとめ図

```
CustomListItems  MaxItemsPerView="5000"（既定 5000、範囲 1〜50000）
 ├─ Variable Name="..." InitialValue="..."   ループの外で宣言。型は推論
 ├─ Size                                     要素数（省略可）
 ├─ Exec                                     代入・算術・論理。関数呼び出し不可
 ├─ Loop Condition="..."                     繰り返し
 │   ├─ If / Elseif / Else                   分岐
 │   ├─ Break Condition="..."                最も内側の Loop を抜ける
 │   ├─ Item Name="...">式</Item             1件出力
 │   └─ Loop                                 入れ子にできる（第26章）
 └─ Item                                     ループの外にも置ける（統計値）
```

## 23.11 この章のまとめ

- `CustomListItems` は**走査の手続きを自分で書く**要素。専用要素で書けるなら、そちらを使う（性能のため）。
- 使うべき場面は3つ ―― **飛ばす要素がある**、**入れ子の走査**、**辿りながらの計算**。
- `<Variable>` はループの外で宣言。型は `InitialValue` から推論される。
- `<Exec>` では代入・算術・論理演算が使える。**関数は呼べず、クラス型オブジェクトも作れない。** イテレータは自分で分解して再実装する。
- `Loop` は「条件付き」と「無限ループ ＋ `Break`」の2通り。脱出条件が複数なら後者。
- **式の評価に失敗すると、ループは黙って終わる。** 「途中までしか出ない」の主因。デバッグは変数を `Item` で出力して行う。
- **カウンタによる `Break` を必ず入れる。** 無限ループはデバッガを止める。
- `MaxItemsPerView` は既定 5000、範囲 1〜50000。`CustomListItems` 専用の属性。
- ループを回して `Item` を1件だけ出す使い方 ―― 統計値の算出 ―― も定型パターン。

## 23.12 演習

1. 23.2節の `forward_list` 版から `<Exec>p = p->next</Exec>` を削除し、無限ループになることを確認してください（デバッガが固まるので、`Size` を小さい値に固定してから試してください）。
2. `<Item>p->value</Item>` を `<Item Name="node {p}">p->value</Item>` に変え、`Name` の中で変数が使えることを確認してください。
3. `<Size>_size</Size>` を消し、代わりに `guard` を 2 で `Break` させて、2件しか出ないことを確認してください。`Size` とループの停止条件が独立していることの確認です。
4. 23.9節の黒高さ計算で、`p = p->left` を `p = p->right` に変え、最右パスの黒高さが同じになることを確認してください。赤黒木が正しく平衡していれば一致するはずです。
5. `MaxItemsPerView="3"` を指定し、4件以上ある `forward_list` を開いて、末尾に続きを見るノードが現れることを確認してください。
