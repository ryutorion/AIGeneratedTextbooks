# 第45章 Natvis のテストと CI

本編の最終章です。

Natvis は「壊れても気づきにくい」という性質を持っています。表示が素っ気なくなるだけで、ビルドは通り、テストも通ります。気づくのは、誰かがデバッグ中に「あれ、前はもっと見やすかったような」と思ったときです。

この章では、その状況を減らすための手段を扱います。**完全な自動テストは現時点では難しい**という現実も含めて、率直に整理します。

## 45.1 なぜテストが難しいのか

Natvis のテストとは、「デバッガに特定の状態のオブジェクトを見せたとき、期待した文字列が表示されるか」を確認することです。

必要な要素が多すぎます。

1. デバッグ対象のプログラムをビルドする
2. デバッガで起動し、特定の場所で止める
3. 変数の**表示文字列**を取得する
4. 期待値と比較する

3番目が壁です。通常のプログラムからは、デバッガが作った表示文字列を取得する手段がありません。

Visual Studio の GUI を自動操作するアプローチ（AutoHotkey や PowerShell）も考えられますが、ビルド完了の待機やエラー処理が不安定で、実用に耐えません。

したがって、**段階的に現実的な手段から積み上げる**ことになります。

```
【段階1】構文の検証        XSD による妥当性チェック        → CI に組み込める
【段階2】読み込みの検証     診断メッセージにエラーがないか   → CI に組み込める
【段階3】表示の検証        CDB + .nvload + dx              → 制約あり
【段階4】目視確認          チェックリストとレビュー         → 人間が行う
```

## 45.2 段階1：XSD による構文検証

**最も費用対効果が高い施策です。**

Natvis には公式の XML スキーマ（XSD）があります。Microsoft の C/C++ 拡張機能のリポジトリで公開されており、Visual Studio のインストールフォルダにも含まれています。

これで検証すると、次のような誤りをビルド前に検出できます。

- 要素名の綴り間違い（`<DisplayStrng>`、`<ArrayItem>`）
- 属性名の綴り間違い（`Conditon`、`IncludeVeiw`）
- 置ける場所の間違い（`<Expand>` の外に `<Item>` を書いた、など）
- 必須要素の欠落（`<ArrayItems>` に `<Size>` がない）
- `xmlns` の誤り

**これらはすべて、実行時には「沈黙」として現れます**（第42章 症状1・症状2）。事前に検出できる価値は大きいです。

### Visual Studio の XML エディタで検証する

最も手軽な方法です。`.natvis` を Visual Studio で開き、**XML > スキーマ** からスキーマを選択します。以後、編集中にリアルタイムで検証されます。

`.natvis` ファイル自体にスキーマを指定することもできます。

```xml
<AutoVisualizer xmlns="http://schemas.microsoft.com/vstudio/debugger/natvis/2010"
                xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                xsi:noNamespaceSchemaLocation="natvis.xsd">
```

ただしこの書き方はデバッガ側の解釈に影響しうるので、**エディタの設定で行うほうが無難**です。

### CI で検証する

コマンドラインから検証するには、XML 検証ツールを使います。PowerShell なら標準機能で書けます。

```powershell
# validate-natvis.ps1
param(
    [string]$SchemaPath = "tools/natvis.xsd",
    [string]$NatvisGlob = "**/*.natvis"
)

$schemas = New-Object System.Xml.Schema.XmlSchemaSet
$schemas.Add("http://schemas.microsoft.com/vstudio/debugger/natvis/2010", $SchemaPath) | Out-Null

$failed = $false
Get-ChildItem -Recurse -Filter *.natvis | ForEach-Object {
    $doc = New-Object System.Xml.XmlDocument
    $doc.Schemas = $schemas
    try {
        $doc.Load($_.FullName)
        $doc.Validate({
            param($sender, $e)
            Write-Host "$($args[1].Exception.LineNumber): $($args[1].Message)" -ForegroundColor Red
            $script:failed = $true
        })
        if (-not $failed) { Write-Host "OK: $($_.Name)" -ForegroundColor Green }
    } catch {
        Write-Host "PARSE ERROR: $($_.Name) - $_" -ForegroundColor Red
        $failed = $true
    }
}

if ($failed) { exit 1 }
```

`xmllint`（libxml2）が使える環境なら、より簡潔に書けます。

```bash
xmllint --noout --schema tools/natvis.xsd vizkit/vizkit.natvis
```

**この1行を CI に入れるだけで、`.natvis` の構文崩れがマージされなくなります。**

XSD ファイルはリポジトリに含めておいてください。Visual Studio のバージョンによってスキーマが更新される可能性があるので、**ビルド環境に依存しない場所に置く**のが安全です。

## 45.3 段階2：読み込みの検証

構文が正しくても、読み込まれるとは限りません（第42章 症状2）。

診断メッセージにエラーが出ていないことを確認できれば理想的ですが、Visual Studio の出力ウィンドウをスクリプトから読むのは困難です。

**CDB（コンソール版 WinDbg）なら可能です。**

```
cdb -c ".nvload C:\dev\vizkit\vizkit\vizkit.natvis; q" playground.exe
```

読み込みに失敗すればエラーメッセージが標準出力に出るので、それを検査できます。

```powershell
$out = & cdb -c ".nvload $natvis; q" $exe 2>&1 | Out-String
if ($out -match "error|failed|No .natvis") {
    Write-Error "natvis の読み込みに失敗しました"
    exit 1
}
```

**PDB 埋め込みの検証にも使えます**（第40章 40.6節）。

```
cdb -c ".nvload vizkit; q" playground.exe
```

モジュール名を渡すと、そのモジュールの PDB に埋め込まれた Natvis を読み込みます。埋め込みが失敗していればエラーになります。**リリース前のチェックとして CI に入れる価値があります。**

## 45.4 段階3：表示の検証とその限界

理想は「`dx v` の出力が期待値と一致するか」を自動で確認することです。

```
cdb -c ".nvload vizkit; bm playground!breakpoint_here; g; dx v; q" playground.exe
```

| コマンド | 役割 |
|---|---|
| `.sympath` | シンボルパスの設定 |
| `.nvload` | Natvis の読み込み |
| `bm` | ブレークポイントの設定 |
| `g` | 実行 |
| `dx` | Natvis を適用した表示を取得 |

出力をパースして期待値と比較すれば、回帰テストになります。

### ただし限界がある

第44章 44.4節で述べたとおり、**WinDbg / CDB の Natvis 実装は Visual Studio と同一ではありません**。

- `IncludeView` / `ExcludeView` が効かない
- `Intrinsic` のオーバーロードが効かない

つまり、**CDB でのテストが通っても Visual Studio での表示を保証しません**し、逆に **Visual Studio で正しい表示が CDB では再現しません**。

この分野で実際にテスト自動化に取り組んだ報告でも、この2点が障壁となり、「これらの機能が実装されるまで自動テストの実現は困難」という結論に至っています。

### それでも部分的には有効

全体をテストできなくても、**基本層（第44章 44.5節）に限れば有効です**。

```
テスト対象にする:
  DisplayString の文字列（ビューを使っていないもの）
  ArrayItems / LinkedListItems / TreeItems による要素の列挙
  Condition による状態の出し分け（[empty] / [null] など）

テスト対象にしない:
  ,view(...) を使った表示
  Intrinsic の Optional 重ねに依存する部分
```

**「壊れたら困る中核部分だけをテストする」**という割り切りが現実的です。すべてをテストしようとして頓挫するより、`point2` が `(12.5, -4)` と表示されることを CI で守るほうが価値があります。

## 45.5 段階4：目視確認とレビュー

自動化できない部分は、人間が行います。**チェックリストがあれば、抜けは大きく減ります。**

### ショーケースプログラム

第44章 44.5節で提案した、全型の代表的な状態を作るプログラムです。

```cpp
// natvis_showcase.cpp
int main() {
    // --- core ---
    vizkit::point2 p{ 12.5, -4.0 };
    vizkit::rgba color = vizkit::from_argb(0xFF3366CCu);
    vizkit::rgba transparent{ 0, 0, 0, 0 };
    vizkit::rect box{ {0,0}, {640,480} }, degenerate{ {5,5}, {5,9} };
    vizkit::version ver = vizkit::make_version(3, 12, 7), unset{};

    // --- 文字列 ---
    vizkit::str_view sv_all{ "hello", 5 }, sv_empty{ "hello", 0 }, sv_null{};

    // --- コンテナ: 空 / 通常 / 大量 の3状態を必ず作る ---
    vizkit::dynamic_array<int> v_empty, v_small, v_large;
    // ...

    // --- 壊れた状態（デバッガから手で壊す用のコメント付き） ---
    // v_small._end を _begin より小さくすると [corrupt] が出るはず

    breakpoint_here();
    return 0;
}
```

**「空・通常・境界・壊れた状態」の4つを、すべての型について用意する**のがポイントです。Natvis のバグは、たいてい空か境界か壊れた状態で出ます。

### レビュー観点チェックリスト

`.natvis` の変更をレビューするときの観点です。

```
□ 型名
  □ ウォッチウィンドウの「型」列と一致しているか
  □ 名前空間は完全修飾か
  □ &lt; &gt; &amp; のエスケープは正しいか

□ DisplayString
  □ 条件なしのフォールバックが最後にあるか（第8章 8.7節）
  □ 具体的な条件が上、一般的な条件が下になっているか（第7章 7.3節）
  □ 空・null・破損の状態が区別されているか
  □ 1行に収まる長さか

□ Expand
  □ [size] などの角括弧の慣習に従っているか（第10章 10.3節）
  □ 危険な式に Condition のガードがあるか（第7章 7.4節、第34章 34.5節）
  □ ExpandedItem で基底を引き上げるとき ,nd が付いているか（第12章 12.4節）

□ 列挙
  □ 専用要素で書けるものに CustomListItems を使っていないか（第23章 23.1節）
  □ CustomListItems の Size は「出力件数」になっているか（第25章 25.3節）
  □ 無限ループ防止のカウンタと Break があるか（第23章 23.7節）
  □ 循環リストで Size を省略していないか（第20章 20.3節）
  □ 要素数が多いとき、打ち切りまたはビュー分割があるか（第28章 28.4節）

□ 堅牢性
  □ 不変条件が [corrupt] などとして書かれているか（第15章 15.4節）
  □ Optional を付けすぎていないか（第8章 8.5節）
  □ [Raw View] を隠していないか（第13章 13.8節）

□ 保守性
  □ 同じ式が2か所以上に出ていないか（Intrinsic に切り出す。第38章 38.7節）
  □ ビューは「あれば嬉しい」程度の位置づけか（第44章 44.5節）
```

**このチェックリストを CI では代替できません。** レビューで守る部分です。

### 変更したら showcase を目視する

`.natvis` を変更したら、ショーケースプログラムをデバッグして全型を目視で確認します。所要時間は数分です。**「Natvis を変えたら showcase を開く」をチームの習慣にする**だけで、事故は大きく減ります。

## 45.6 本編のまとめ

第1章から第45章までで、Natvis の全体を扱いました。最後に、本書を貫いてきた原則を並べます。

**書き方の原則**

1. **型名はウォッチウィンドウの「型」列からコピーする。** 推測しない（第29章）。
2. **`.natvis` に書く前に、ウォッチウィンドウで式を試す。**（第4章）
3. **`[Raw View]` が唯一の情報源。** 隠さない（第3章・第13章）。
4. **フォールバックを必ず置く。** 素の表示に戻るより印が出るほうがよい（第8章）。
5. **危険な式の前にガード条件を置く。** `Condition` は表示だけでなく評価の制御でもある（第7章）。
6. **不変条件を書けば、Natvis が不変条件チェッカーになる。**（第15章・第22章）
7. **式に名前を付ける。** 2か所以上に出る式は `Intrinsic` へ（第38章）。
8. **専用要素で書けるなら `CustomListItems` を使わない。**（第23章・第28章）
9. **委譲できるなら委譲する。**（第27章）

**設計の原則**

10. **データ構造の設計が、Natvis の書きやすさを決める。**（第27章）
11. **Natvis で見たいなら、見えるように作る。**（第33章）

10 と 11 が、本書の最も実務的な結論です。`std::unordered_map` の Natvis が1行で済むのは実装が優れているからであり、型消去型の Natvis が書けないのは型情報を捨てているからです。共用体は生バイト配列より書きやすく、名前空間スコープのグローバルは Meyers singleton より引きやすく、クラス内の `enum` 定数は `$T2` より堅牢です。

デバッグしやすさを設計判断の材料に加えてよい ―― これを持ち帰っていただければ、本書の目的は達成されます。

**残る作業**

付録では、本編で扱いきれなかった参照情報をまとめています。

- **付録A** Natvis 要素・属性 完全リファレンス
- **付録B** 書式指定子リファレンス
- **付録C** vizkit 全ソースコード一覧
- **付録D** 完成版 `vizkit.natvis` 全文
- **付録E** `.natstepfilter` と `.natjmc`
- **付録F** `STL.natvis` 読みどころ抜粋
- **付録G** エラーメッセージ逆引き集

## 45.7 この章のまとめ

- Natvis の完全な自動テストは、**現時点では困難**。表示文字列を取得する手段が限られるため。
- **段階1（XSD 検証）が最も費用対効果が高い。** 要素名・属性名の誤り、必須要素の欠落を、実行前に検出できる。`xmllint` か PowerShell で CI に入れる。
- **段階2（読み込みの検証）は CDB の `.nvload` で可能。** PDB 埋め込みの検証にも使える。
- **段階3（表示の検証）は CDB + `dx` でできるが、WinDbg の Natvis 実装が Visual Studio と同一でないため保証にならない。** `IncludeView` / `ExcludeView` と `Intrinsic` のオーバーロードが効かない。
- **中核部分だけをテストする**割り切りが現実的。すべてをテストしようとして頓挫するより価値がある。
- **ショーケースプログラム**を用意し、全型について「空・通常・境界・壊れた状態」の4つを作る。
- **レビュー観点チェックリスト**で人間が守る部分を明確にする。
- **「Natvis を変えたら showcase を開く」をチームの習慣にする。**

## 45.8 演習

1. Natvis の XSD を入手してリポジトリに置き、`xmllint` または PowerShell で `vizkit.natvis` を検証してください。意図的に要素名を間違えて、検出されることを確認します。
2. その検証を CI（GitHub Actions、Azure Pipelines など）のステップに追加してください。
3. `natvis_showcase.cpp` を作り、`vizkit` の全型について「空・通常・境界・壊れた状態」を用意してください。
4. CDB で `.nvload` と `dx` を使い、`point2` の表示が `(12.5, -4)` になることを確認するスクリプトを書いてください。
5. 45.5節のレビュー観点チェックリストを、本書でここまで書いてきた `vizkit.natvis` に対して実際に適用してください。**いくつ違反が見つかりましたか。** 見つかった項目を修正して、付録D の完成版に反映してください。
