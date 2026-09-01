# 第40章 MSBuild への組み込みと PDB 埋め込み

前章で「配布は PDB 埋め込み」と結論しました。この章では、その具体的な手順を扱います。

MSBuild プロジェクトでの設定と、静的ライブラリ / DLL それぞれの事情、そしてライブラリ利用者に自動で適用されるようにする `.props` の書き方までを見ます。

## 40.1 プロジェクトへの追加（復習）

第2章 2.5節の再確認です。

1. プロジェクトを右クリック > **追加 > 新しい項目** > **デバッガー ビジュアライゼーション ファイル (.natvis)**
2. ファイルを選択して F4（プロパティ）
3. **項目の種類** が `Natvis` になっていることを確認

`.vcxproj` の中では、次のように記録されます。

```xml
<ItemGroup>
  <Natvis Include="vizkit.natvis" />
</ItemGroup>
```

`Natvis` という項目の種類が、MSBuild にとっての目印です。手で `.vcxproj` を編集する場合も、この形にします。

### ビルドから除外

```xml
<Natvis Include="vizkit.natvis">
  <ExcludedFromBuild>true</ExcludedFromBuild>
</Natvis>
```

プロパティウィンドウの **ビルドから除外 = はい** に相当します。**PDB への埋め込みだけを止めたい**ときに使います。プロジェクト内のファイルとしては引き続き読み込まれます（第39章の優先順位2）。

## 40.2 /NATVIS リンカーオプション

PDB への埋め込みは、リンカーオプション `/NATVIS` が担当します。

```
/NATVIS:filename
```

> `/NATVIS` リンカーオプションは、Natvis ファイルで定義されたデバッガ可視化情報を LINK が生成する PDB ファイルに埋め込みます。これにより、デバッガは Natvis ファイルに依存せず可視化を表示できます。

**複数指定できます。**

```
/NATVIS:core.natvis /NATVIS:containers.natvis /NATVIS:math.natvis
```

Visual Studio の IDE で設定する場合は、**構成プロパティ > リンカー > コマンド ライン > 追加オプション** に書きます。

```
/NATVIS:$(SolutionDir)vizkit\vizkit.natvis
```

### /DEBUG がなければ無視される

**重要な制限です。**

> LINK は `/DEBUG` オプションで PDB ファイルが作成されない場合、`/NATVIS` を無視します。

PDB が生成されなければ、埋め込む先がないので当然です。エラーも警告も出ずに無視されるので、**「埋め込んだつもりが埋め込まれていない」という状態になります**。

Debug 構成では既定で `/DEBUG` が有効ですが、Release 構成では設定を確認してください。**リンカー > デバッグ > デバッグ情報の生成** が `いいえ` になっていると、`/NATVIS` は効きません。

配布するライブラリなら、Release でも PDB を生成する設定にしておくのが実務的です。利用者は Release ビルドのライブラリを Debug ビルドのアプリから使うことが多いからです。

## 40.3 静的ライブラリの落とし穴

ここが最も誤解されやすい点です。

**`/NATVIS` はリンカー（`link.exe`）のオプションです。静的ライブラリを作る `lib.exe` には渡せません。**

```
vizkit.lib  ← lib.exe が作る。/NATVIS は使えない
playground.exe  ← link.exe が作る。/NATVIS が使える
```

したがって、**静的ライブラリの Natvis を PDB に埋め込むには、最終的な実行可能ファイル（または DLL）をリンクするときに `/NATVIS` を指定する必要があります**。

ライブラリの作者としては、これを利用者に手作業でやらせたくありません。解決策が次節の `.props` です。

## 40.4 .props で利用者に自動適用する

MSBuild の `.props` ファイルは、プロジェクトのプロパティを外部から追加する仕組みです。ライブラリに同梱して、参照するだけでリンカーオプションが追加されるようにできます。

`vizkit\build\vizkit.props`

```xml
<?xml version="1.0" encoding="utf-8"?>
<Project xmlns="http://schemas.microsoft.com/developer/msbuild/2003">

  <!-- このファイルからの相対でライブラリのルートを求める -->
  <PropertyGroup>
    <VizkitRoot Condition="'$(VizkitRoot)' == ''">$(MSBuildThisFileDirectory)..\</VizkitRoot>
  </PropertyGroup>

  <!-- インクルードパス -->
  <ItemDefinitionGroup>
    <ClCompile>
      <AdditionalIncludeDirectories>$(VizkitRoot)include;%(AdditionalIncludeDirectories)</AdditionalIncludeDirectories>
    </ClCompile>

    <!-- natvis を PDB に埋め込む -->
    <Link>
      <AdditionalOptions>/NATVIS:"$(VizkitRoot)vizkit.natvis" %(AdditionalOptions)</AdditionalOptions>
    </Link>
  </ItemDefinitionGroup>

</Project>
```

利用者側の `.vcxproj` で、この `.props` をインポートします。

```xml
<ImportGroup Label="PropertySheets">
  <Import Project="..\external\vizkit\build\vizkit.props" />
</ImportGroup>
```

Visual Studio の **表示 > その他のウィンドウ > プロパティ マネージャー** から、GUI で追加することもできます。

これで、**利用者は何も意識せずに Natvis の恩恵を受けられます**。インクルードパスと Natvis が同時に設定されるので、設定漏れも起きません。

### パスに空白が入る場合

`/NATVIS:` のパスは引用符で囲んでください。`C:\Program Files\...` のようなパスで失敗します。

```xml
<AdditionalOptions>/NATVIS:"$(VizkitRoot)vizkit.natvis" %(AdditionalOptions)</AdditionalOptions>
```

`%(AdditionalOptions)` を末尾に付けるのを忘れないでください。これがないと、他のオプションを上書きしてしまいます。

## 40.5 DLL の場合

DLL は `link.exe` でリンクされるので、**自分自身の PDB に埋め込めます**。

DLL プロジェクトのリンカー設定に `/NATVIS` を追加すれば完了です。利用者側の設定は不要です。

```
構成プロパティ > リンカー > コマンド ライン > 追加オプション
  /NATVIS:$(ProjectDir)vizkit.natvis
```

配布物に `.pdb` を含めることを忘れないでください。**PDB がなければ、埋め込んだ Natvis も届きません。**

| 配布形態 | 埋め込み先 | 利用者の設定 |
|---|---|---|
| ヘッダーオンリー | 埋め込めない | `.natvis` を配って各自で設定、または `.props` を同梱 |
| 静的ライブラリ | 利用者の EXE / DLL | **`.props` を同梱する**のが実務的 |
| 動的ライブラリ | DLL 自身の PDB | 不要（PDB を配布物に含める） |

**ヘッダーオンリーライブラリが最も面倒です。** リンクされる実体がないので、`.props` による `/NATVIS` の注入が唯一の現実的な手段になります。

## 40.6 埋め込まれたか確認する

`/NATVIS` が効いているかどうかは、次の方法で確認できます。

**方法1：診断メッセージ。**
プロジェクト内の `.natvis` を一時的に除外（または「項目の種類」を `なし` に）してデバッグし、**それでも表示が正しければ PDB から読み込まれています**。

**方法2：別のソリューションから使う。**
ライブラリのプロジェクトを含まない新しいソリューションを作り、ビルド済みの `.lib` / `.dll` をリンクしてデバッグします。表示が効いていれば埋め込み成功です。**これが最も確実な検証**で、実際に利用者と同じ状況を再現できます。

**方法3：WinDbg で確認する。**
`.nvload モジュール名` を実行すると、そのモジュールの PDB に埋め込まれた Natvis が読み込まれます（第44章 44.4節）。エラーが出れば埋め込まれていません。

## 40.7 開発と配布を両立させる

第39章 39.5節で述べたとおり、**PDB に埋め込まれた `.natvis` はデバッグ中に変更できません**。`.natvisreload` が効かないので、開発中は不便です。

推奨する構成は次のとおりです。

```
Debug 構成    : プロジェクト内の .natvis として扱う（/NATVIS なし）
                → .natvisreload で高速に開発できる

Release 構成  : /NATVIS で PDB に埋め込む
                → 配布物に含まれる
```

MSBuild では構成ごとに設定できます。

```xml
<ItemDefinitionGroup Condition="'$(Configuration)'=='Release'">
  <Link>
    <AdditionalOptions>/NATVIS:"$(ProjectDir)vizkit.natvis" %(AdditionalOptions)</AdditionalOptions>
  </Link>
</ItemDefinitionGroup>
```

ただしこれには落とし穴があります。**Release でしか埋め込まないと、埋め込みの不具合に気づくのが遅れます。** 40.6節の方法2（別ソリューションでの検証）を、リリース前に必ず実施してください。

より安全なのは、**Debug でも埋め込んだうえで、開発中はプロジェクト内のファイルを優先させる**ことです。第39章 39.5節で触れた仕様が効きます。

> PDB に埋め込まれた Natvis と同名のファイルがプロジェクトにあれば、**プロジェクト側が優先される**。

つまり、同じ `vizkit.natvis` という名前でプロジェクトにも置いておけば、開発中はそちらが使われ、`.natvisreload` も効きます。そして利用者の環境では PDB のものが使われます。**両立できます。**

## 40.8 この章のまとめ

- プロジェクトへの追加は **項目の種類 = `Natvis`**。`.vcxproj` では `<Natvis Include="..."/>`。
- **ビルドから除外**すると、PDB への埋め込みだけが止まる。プロジェクト内のファイルとしては読み込まれ続ける。
- 埋め込みはリンカーオプション **`/NATVIS:filename`**。複数指定できる。
- **`/DEBUG` がなければ黙って無視される。** Release で PDB を作らない設定になっていないか確認する。
- **`/NATVIS` は `link.exe` のオプション。`lib.exe` には渡せない。** 静的ライブラリは、利用者の EXE / DLL をリンクするときに指定する必要がある。
- 静的ライブラリとヘッダーオンリーライブラリでは、**`.props` を同梱して利用者に自動適用する**のが実務的。`%(AdditionalOptions)` を末尾に付け、パスは引用符で囲む。
- DLL は自分自身の PDB に埋め込める。**配布物に `.pdb` を含めることを忘れない。**
- 検証は**ライブラリのプロジェクトを含まない別ソリューションから**行うのが最も確実。
- **同名のファイルをプロジェクトにも置けば、開発中は `.natvisreload` が使え、利用者には PDB のものが届く。** 両立できる。

## 40.9 演習

1. `playground` のリンカー設定に `/NATVIS:$(SolutionDir)vizkit\vizkit.natvis` を追加してビルドし、`vizkit.natvis` の「項目の種類」を `なし` にしても表示が効くことを確認してください。
2. 続けて、リンカー > デバッグ > デバッグ情報の生成 を `いいえ` に変え、`/NATVIS` が無視されることを確認してください。エラーも警告も出ないことが要点です。
3. 40.4節の `vizkit.props` を作り、新しいソリューションから `vizkit.lib` を使うプロジェクトを作って、`.props` のインポートだけで Natvis が効くことを確認してください。
4. `vizkit` を DLL プロジェクトに変え、DLL 自身の PDB に埋め込んでみてください。`.pdb` を削除した状態でデバッグし、表示が失われることも確認してください。
5. PDB に埋め込んだ状態で `.natvisreload` を実行し、`.natvis` を編集しても反映されないことを確認してください。そのうえで同名のファイルをプロジェクトに追加し、反映されるようになることを確かめてください。
