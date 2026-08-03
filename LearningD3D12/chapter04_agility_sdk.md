# 第4章 DirectX Agility SDK を導入する

前章で C++23 のプロジェクトができました。本章では、そこに Direct3D 12 のランタイムを差し替える仕組みを組み込みます。

**なぜデバイスを作る前に、こんな配管工事をするのか。**

多くの入門書は、Agility SDK を巻末の付録に置きます。「必要になったら入れればいい」という扱いです。本書がこれを第4章、つまりデバイス生成(第7章)より前に置くのには理由があります。

Agility SDK の導入で失敗したとき、**何のエラーも出ないから**です。ビルドは通ります。実行もできます。デバイスも作れます。ただ、第30章の Enhanced Barriers を書こうとした段階で `ID3D12Device10` の QueryInterface が失敗し、「ドライバが古いのか」「GPU が対応していないのか」「コードが間違っているのか」の三択で悩むことになります。実際の原因は、`.cpp` に 2 行書き忘れたことだったりします。

この沈黙の失敗を、機能を使い始める前に潰しておく。それが本章の目的です。

**本章のゴール**
Agility SDK を導入し、**実行時に読み込まれた D3D12 ランタイムのパスとバージョン番号を表示させて**、確かに差し替わっていることを目視で確認する。

---

## 4.1 Windows SDK 同梱の D3D12 と Agility SDK の関係

### 4.1.1 ランタイムは 2 つのファイルに分かれている

Direct3D 12 のランタイムは、Windows のあるバージョンから**2 つの DLL に分割されました**。

| DLL | 役割 | 置き場所 |
|---|---|---|
| `d3d12.dll` | **ローダー**。どの実装を使うかを決めて呼び出すだけの薄い層 | `C:\Windows\System32\`(OS 固定) |
| `D3D12Core.dll` | **実装本体**。API の中身はすべてこちら | OS 標準版か、アプリ同梱版か |

アプリケーションがリンクするのは `d3d12.dll` のインポートライブラリ(`d3d12.lib`)です。ここは今までと変わりません。変わったのは、`d3d12.dll` が「どの `D3D12Core.dll` を読むか」を選べるようになった点です。

この分割こそが、Agility SDK を成立させている仕組みです。

### 4.1.2 なぜ OS 標準の実装では足りないのか

Direct3D 12 の新機能は、伝統的に **Windows のバージョンアップと一緒に**提供されてきました。この方式には構造的な問題があります。

新機能が実装されてから、それを使うゲームを作れるだけの普及率に達するまで、**1〜2 年かかります**。その間、GPU ベンダは新しいドライバも新しいハードウェアも出しているのに、API だけが足を引っ張ります。

Microsoft はこれを、ランタイムを再頒布可能にすることで解決しました。**アプリが自分専用の `D3D12Core.dll` を抱えて配布する**という方式です。

### 4.1.3 「最新機能を Windows 10 でも使える」仕組み

動作の流れは次の通りです。

```
① アプリが起動する
        ↓
② d3d12.dll(OS 標準のローダー)が読み込まれる
        ↓
③ ローダーがアプリの exe から 2 つのエクスポートシンボルを探す
     ・D3D12SDKVersion … どのバージョンを使いたいか
     ・D3D12SDKPath    … D3D12Core.dll がどこにあるか
        ↓
④-a 見つかった → 指定されたパスの D3D12Core.dll を読み込む
④-b 見つからない → OS 標準の D3D12Core.dll を読み込む
```

この方式のおかげで、**Windows 10 バージョン 1909(2019 年 11 月更新)以降**であれば、OS を更新せずに最新の Direct3D 12 機能を使えます。必要なのは、対応したドライバと(機能によっては)ハードウェアだけです。

> **バージョンが古い場合は OS 側が使われます**
>
> アプリが指定した `D3D12SDKVersion` が、OS に入っている実装より**古い**場合、ローダーは OS 側を使います。これは安全側に倒した設計です。Windows Update で配信されたバグ修正が、アプリが古い DLL を抱えているせいで無効化される、という事態を避けています。

### 4.1.4 最大の特徴:黙って失敗する

上の流れの ④-b に注目してください。**エクスポートシンボルが見つからなくても、エラーは出ません。** OS 標準のランタイムに静かにフォールバックし、アプリは正常に動き続けます。

これが本章を第4章に置いた理由です。設定の誤りは、次のいずれの形でも表面化しません。

- コンパイルエラー → 出ません(ヘッダは Agility SDK のものを見ているため)
- リンクエラー → 出ません
- 実行時エラー → 出ません
- デバイス生成の失敗 → しません

表面化するのは、**新機能を使おうとした瞬間**だけです。しかもその症状は「ドライバが対応していない」場合とまったく同じに見えます。

だから 4.7 節で、**実行時に確認する手段**を作ります。

---

## 4.2 NuGet パッケージ `Microsoft.Direct3D.D3D12` の追加

### 4.2.1 retail 版と preview 版

Agility SDK には 2 系統あります。バージョン番号で見分けられます。

| 系統 | バージョン | `D3D12SDKVersion` | 開発者モード |
|---|---|---|---|
| **retail** | 1.6xx.x | 6xx | 不要 |
| **preview** | 1.7xx.x-preview | 7xx | **必須** |

preview 版は、まだ仕様が固まっていない機能を先行公開するためのものです。**利用者の Windows が開発者モードでないと機能が有効になりません。** つまり、配布するアプリケーションで preview 版を使うことはできません。

**本書は retail 版を使います。** 執筆時点の最新は **1.619.0**(2026 年 2 月リリース)で、Shader Model 6.9 と DXR 1.2(SER / OMM)を含みます。第2章で見た通り、本書が扱う機能はすべてこれで足ります。

以降、`D3D12SDKVersion = 619` として説明します。**読者がインストールしたバージョンに読み替えてください。** この数字が実際のパッケージと一致していないと、フォールバックが起きます。

### 4.2.2 パッケージを追加する

1. ソリューションエクスプローラーで `D3D12Book` プロジェクトを右クリック
2. `NuGet パッケージの管理` を選択
3. `参照` タブで `Microsoft.Direct3D.D3D12` を検索
4. **「プレリリースを含める」のチェックは外したまま**にする(retail 版を選ぶため)
5. バージョンを確認してインストール

インストール後、`パッケージ` タブでバージョンを確認してください。`1.619.0` のように表示されます。

> **プレリリースのチェックを外す意味**
>
> チェックを入れると preview 版(1.7xx-preview)が候補に出てきます。番号が大きいので「新しいほう」を選びたくなりますが、**開発者モードが必須になり、配布もできなくなります。**
>
> preview 版を試したくなったときは、この制約を理解したうえで切り替えてください。

### 4.2.3 パッケージの中身

NuGet パッケージ(`.nupkg`)は実体が ZIP アーカイブです。展開すると次の構成になっています。

```
Microsoft.Direct3D.D3D12.1.619.0/
└─ build/native/
    ├─ include/
    │   ├─ d3d12.h              ← Windows SDK 版より新しい
    │   ├─ d3d12sdklayers.h
    │   ├─ d3d12video.h
    │   ├─ d3dx12.h             ← 本書では使わない
    │   └─ ...
    ├─ bin/x64/
    │   ├─ D3D12Core.dll        ← ランタイム本体
    │   ├─ D3D12Core.pdb
    │   ├─ d3d12SDKLayers.dll   ← デバッグレイヤー
    │   └─ d3d12SDKLayers.pdb
    └─ Microsoft.Direct3D.D3D12.targets
```

`.targets` ファイルが、MSBuild に対して次の 2 つを自動で行います。

- `include/` をインクルードパスの**先頭**に追加する
- ビルド時に `bin/x64/` の DLL を `$(OutDir)D3D12\` にコピーする

つまり、**DLL のコピーは自分で書く必要がありません。** 4.4 節で実際にコピーされていることを確認します。

### 4.2.4 ヘッダの優先順位が重要

Agility SDK の `include/` には、Windows SDK に入っているものより**新しい `d3d12.h`** が含まれています。新しい構造体や列挙値の定義は、こちらにしかありません。

`.targets` はインクルードパスの先頭に追加してくれるので、通常は正しく解決されます。しかし、プロジェクトの設定で `追加のインクルード ディレクトリ` に Windows SDK のパスを明示的に書き足していると、順序が逆転することがあります。

**症状**

```
error C2065: 'D3D12_BARRIER_TYPE_GLOBAL': 定義されていない識別子です。
error C2039: 'Barrier': 'ID3D12GraphicsCommandList' のメンバーではありません
```

新しい機能の識別子だけが見つからない、というエラーが出たら、**古い `d3d12.h` を掴んでいます。** インクルードパスの順序を確認してください。

確認方法は、コンパイラに `/showIncludes` を渡してビルド出力を見ることです。

```
プロジェクトのプロパティ → C/C++ → 詳細設定 → インクルード ファイルの一覧を表示 → はい
```

出力ウィンドウに読み込まれたヘッダのフルパスが並ぶので、`d3d12.h` がどこから来ているかがわかります。**確認したら元に戻してください。** 出力が膨大になります。

### 4.2.5 `d3dx12.h` は使わない

パッケージには `d3dx12.h`(および `d3dx12_*.h`)が同梱されています。`CD3DX12_RESOURCE_BARRIER::Transition()` などのヘルパーを提供する、便利なヘッダです。

**本書では使いません。** 第1章 1.3.1 節で述べた通り、これが本書の中核的な方針です。D3D12 の構造体を毎回自分の手で埋めることで、「バリアとは何をしている構造体なのか」が身につきます。

同梱されているからといって、うっかり `#include <d3dx12.h>` と書かないでください。**書いた瞬間に、この本を読む意味が半分なくなります。**

必要なヘルパーは第11章以降で自作し、付録 A に一覧としてまとめます。

### 4.2.6 `pch.h` を更新する

第3章で作った `pch.h` に、D3D12 のヘッダを追加します。コメントで「第7章で追加する」と書いた箇所です。

```cpp
// pch.h(第4章時点)
#pragma once

#include <Windows.h>
#include <wrl/client.h>

// ---------------------------------------------------------
// Direct3D 12 / DXGI
// ---------------------------------------------------------
// これらは Agility SDK 版のヘッダが使われる。
// NuGet の .targets がインクルードパスの先頭に追加している。
#include <d3d12.h>
#include <dxgi1_6.h>
#include <dxgidebug.h>

// d3dx12.h は本書では使わない。同梱されているが include しないこと。

// ---------------------------------------------------------
// Nsight Aftermath
// ---------------------------------------------------------
// 第8章で追加する:
// #include <GFSDK_Aftermath.h>
```

あわせて、インポートライブラリをリンクします。

`プロジェクトのプロパティ → リンカー → 入力 → 追加の依存ファイル`

```
d3d12.lib
dxgi.lib
dxguid.lib
GFSDK_Aftermath_Lib.x64.lib
%(AdditionalDependencies)
```

`dxguid.lib` は `IID_ID3D12Device` などの GUID 定数の実体を提供します。忘れると `LNK2001: 外部シンボル "IID_..." は未解決です` になります。

**✅ ここまでの確認**
NuGet パッケージがインストールされ、`pch.h` に `d3d12.h` が追加され、ビルドが通る。

---

## 4.3 エクスポートシンボルの記述

ここが本章の核心です。**2 つの定数を exe からエクスポートする**、それだけの作業ですが、細部に落とし穴が複数あります。

### 4.3.1 何を書くか

公式仕様が示す書き方は次の通りです。

```cpp
extern "C" { __declspec(dllexport) extern const UINT D3D12SDKVersion = 619; }
extern "C" { __declspec(dllexport) extern const char* D3D12SDKPath = ".\\D3D12\\"; }
```

| シンボル | 型 | 意味 |
|---|---|---|
| `D3D12SDKVersion` | `UINT` | 使用したい Agility SDK のバージョン番号 |
| `D3D12SDKPath` | `const char*` | exe からの**相対パス**。UTF-8 文字列 |

**注意点を 3 つ。**

1. `D3D12SDKVersion` は、インストールした NuGet パッケージのバージョンと**厳密に一致**しなければなりません。1.619.0 なら `619` です。
2. `D3D12SDKPath` は**必ず末尾に `\` を付けます**。`".\\D3D12"` では動きません。
3. パスは**相対パス**です。絶対パスを書くと、別の環境で動かなくなります。

### 4.3.2 C++23 での落とし穴 —— `u8` リテラルが使えない

Microsoft の公式ドキュメントは、パスをこう書いています。

```cpp
extern "C" { __declspec(dllexport) extern const char* D3D12SDKPath = u8".\\D3D12\\"; }
```

**このコードは、本書のプロジェクトではコンパイルが通りません。**

理由は C++20 での変更にあります。`u8"..."` リテラルの型が `const char[]` から **`const char8_t[]`** に変わったため、`const char*` への暗黙変換ができなくなりました。

```
error C2440: '初期化中': 'const char8_t [10]' から 'const char *' に変換できません。
```

公式ドキュメントの例は C++17 時代に書かれたものです。C++23 を使う本書では、**`u8` プレフィックスを外します**。

```cpp
// ✅ 本書での書き方
extern "C" { __declspec(dllexport) extern const char* D3D12SDKPath = ".\\D3D12\\"; }
```

問題はありません。仕様が要求しているのは「UTF-8 でエンコードされた文字列」であり、この文字列は ASCII のみで構成されています。**ASCII は UTF-8 の部分集合なので、そのまま妥当な UTF-8 です。**

> **どうしても `u8` を使いたい場合**
>
> ```cpp
> extern "C" { __declspec(dllexport) extern const char* D3D12SDKPath =
>     reinterpret_cast<const char*>(u8".\\D3D12\\"); }
> ```
>
> 動きますが、読みにくいだけで利点がありません。本書はプレフィックスなしを採用します。

### 4.3.3 なぜ `extern` が必要なのか

宣言をもう一度見てください。

```cpp
extern "C" { __declspec(dllexport) extern const UINT D3D12SDKVersion = 619; }
//                                 ^^^^^^ これ
```

`const` が付いているのに、なぜ `extern` も書くのか。

**C++ では、名前空間スコープの `const` オブジェクトは既定で内部リンケージを持つ**からです。内部リンケージのシンボルは、翻訳単位の外から見えません。`__declspec(dllexport)` を付けても、エクスポートテーブルに載らないのです。

`extern` を明示することで外部リンケージになり、初めてエクスポートが成立します。

**この `extern` を落とすと、何のエラーも出ないまま、Agility SDK が有効になりません。** 4.1.4 節で述べた「黙って失敗する」の典型例です。

> **`extern "C"` のほうは何のためか**
>
> C++ の名前マングリングを抑制するためです。ローダーは `GetProcAddress` で `"D3D12SDKVersion"` という文字列を探すので、`?D3D12SDKVersion@@3IB` のような装飾名になっていると見つけられません。

### 4.3.4 どこに書くか

**専用の `.cpp` ファイルを 1 つ作り、そこにだけ書きます。**

`D3D12Book/src/AgilitySDK.cpp` を新規作成してください。

```cpp
// src/AgilitySDK.cpp
//
// DirectX Agility SDK を有効にするためのエクスポートシンボル。
//
// 【重要】
// ・このファイルの内容を他のファイルに移動しないこと
// ・ヘッダに書いてはならない(多重定義になる)
// ・extern を省略してはならない(内部リンケージになり、
//   エクスポートされず、黙ってフォールバックする)
//
// D3D12SDKVersion は NuGet パッケージのバージョンと一致させること。
//   Microsoft.Direct3D.D3D12 1.619.0  →  619

#include "pch.h"

extern "C"
{
    __declspec(dllexport) extern const UINT  D3D12SDKVersion = 619;
}

extern "C"
{
    // 末尾の "\" は必須。u8 プレフィックスは C++20 以降では付けられない。
    __declspec(dllexport) extern const char* D3D12SDKPath    = ".\\D3D12\\";
}
```

**ヘッダに書いてはいけない理由**は、`__declspec(dllexport)` 付きの定義を複数の翻訳単位に配ると多重定義エラーになるからです。1 つの `.cpp` に閉じ込めるのが唯一の正解です。

**ファイルを分ける理由**は、実務上の事故防止です。`main.cpp` の片隅に書いておくと、リファクタリングのときに一緒に消えます。専用ファイルにして、冒頭に警告コメントを置いておけば、誤って触られる確率が下がります。

### 4.3.5 `dumpbin` でエクスポートを確認する

正しくエクスポートされたかは、ビルド後の exe を調べればわかります。実行する前に確認できるので、切り分けの手段として覚えておいてください。

Visual Studio の **「x64 Native Tools Command Prompt for VS 2026」** を起動し、次を実行します。

```
dumpbin /exports build\x64\Debug\D3D12Book.exe
```

期待される出力(抜粋):

```
    ordinal hint RVA      name

          1    0 0002B000 D3D12SDKPath
          2    1 0002B008 D3D12SDKVersion
```

**この 2 行が出なければ、エクスポートに失敗しています。** 原因はほぼ次のいずれかです。

- `extern` を書き忘れた(4.3.3)
- `extern "C"` を書き忘れた
- `AgilitySDK.cpp` がプロジェクトに追加されていない

3 つ目は意外と起こります。ファイルをエクスプローラーで作っただけでは、Visual Studio のプロジェクトには含まれません。ソリューションエクスプローラーで `既存の項目の追加` を行ったか確認してください。

**✅ ここまでの確認**
`dumpbin /exports` で `D3D12SDKVersion` と `D3D12SDKPath` が表示される。

---

## 4.4 `D3D12Core.dll` / `d3d12SDKLayers.dll` の配置

### 4.4.1 NuGet が自動でやってくれること

4.2.3 節で触れた通り、NuGet パッケージの `.targets` がビルド後のコピーを行います。ビルドして、出力ディレクトリを確認してください。

```
build/x64/Debug/
├─ D3D12Book.exe
├─ GFSDK_Aftermath_Lib.x64.dll     ← 第3章で設定したコピー
└─ D3D12/                          ← .targets が作る
    ├─ D3D12Core.dll
    ├─ D3D12Core.pdb
    ├─ d3d12SDKLayers.dll
    └─ d3d12SDKLayers.pdb
```

`D3D12/` というフォルダ名は、`D3D12SDKPath` に書いた `".\\D3D12\\"` と対応しています。**両者が一致していなければ動きません。**

### 4.4.2 手動で配置する場合

NuGet を使わない場合(社内のビルドシステムに組み込む場合など)は、`.nupkg` を ZIP として展開し、自分でコピーします。

**ビルド後のイベント**に追加します。

```
if not exist "$(OutDir)D3D12\" mkdir "$(OutDir)D3D12\"
xcopy /Y /D "$(SolutionDir)external\AgilitySDK\bin\x64\*.dll" "$(OutDir)D3D12\"
```

第3章で設定した Aftermath の DLL コピーと並べて書くことになります。

### 4.4.3 exe と同じフォルダに置いてはいけない

`D3D12SDKPath` を `".\\"`(exe と同じディレクトリ)にすることも、技術的には可能です。**しかし DirectX チームはこれを明確に非推奨としています。**

理由は、`d3d12SDKLayers.dll` のバージョン不一致です。デバッグレイヤーは OS 側にも同名のファイルが存在し、探索順序によっては OS 標準版が読み込まれることがあります。`D3D12Core.dll` は Agility SDK 版、`d3d12SDKLayers.dll` は OS 版、という組み合わせが成立してしまうと、デバッグレイヤーが誤った警告を出したり、クラッシュしたりします。

**サブフォルダに置く**ことで、この曖昧さが消えます。素直に `D3D12\` を使ってください。

---

## 4.5 Agility SDK とドライバの対応関係

### 4.5.1 機能が使えるかは 4 つの条件の積

ある機能が使えるかどうかは、次の 4 つが**すべて**揃って初めて決まります。

```
使える = OS のバージョン
       × Agility SDK のバージョン(ランタイム)
       × GPU ドライバの実装
       × ハードウェアの世代
```

Agility SDK が解決するのは 2 番目だけです。**残りの 3 つは別問題として残ります。**

具体例で見ます。第37章で扱う Shader Execution Reordering(SER)の場合。

| 条件 | 必要なもの | 満たさないと |
|---|---|---|
| OS | Windows 10 1909 以降 | Agility SDK 自体が動かない |
| ランタイム | Agility SDK 1.619 以降 | API が存在しない(コンパイルエラー) |
| シェーダー | Shader Model 6.9 対応 DXC | HLSL がコンパイルできない |
| ドライバ | SER 対応ドライバ | 対応クエリが偽を返す |
| ハードウェア | NVIDIA は Ada 以降 | API は呼べるが並べ替えが起きない |

最後の行に注目してください。**API が成功しても、実際には何も起きない**というケースがあります。DXR 1.2 では「デバイスが実際にリオーダーを行うか」を問い合わせられるようになりましたが、こうした「呼べるが効かない」状態が存在することは知っておく必要があります。

### 4.5.2 NVIDIA ドライバの実装状況

第2章で対象を NVIDIA に固定したのは、まさにこの部分で有利だからです。

Agility SDK の新機能に対するドライバ実装は、**NVIDIA が最も早く、最も広い**という傾向が続いています。1.619 で SM 6.9 と DXR 1.2 が正式版になった際も、対応の幅は NVIDIA が最も広い状態でした。

これは「NVIDIA が優れている」という話ではなく、**本書のような学習用途では、新機能を試すまでの待ち時間が短いほうが良い**という実務的な理由です。他ベンダの環境では、同じコードが「API はあるがドライバが未対応」で止まることがあります。

ドライバのバージョン確認は第2章 2.2.3 節の手順で行えます。**新しい Agility SDK を入れたら、ドライバも更新してください。** 片方だけ新しくしても、その差分の機能は使えません。

### 4.5.3 切り分けの順序

新機能が動かないとき、疑う順番を決めておくと早く解決します。

```
① エクスポートシンボルは出ているか       → dumpbin /exports(4.3.5)
② D3D12Core.dll は差し替わっているか     → 4.7 節の確認コード
③ ドライバは新しいか                     → nvidia-smi(第2章)
④ GPU は対応世代か                       → 第2章 2.1.2 の表
⑤ CheckFeatureSupport は何と答えるか     → 第7章
```

**経験上、①と②で全体の 8 割が解決します。** ③以降を疑う前に、まず自分の設定を確認してください。

---

## 4.6 開発者モードと配布時の注意点

### 4.6.1 開発者モードが必要になる場面

| 使うもの | 開発者モード |
|---|---|
| retail 版 Agility SDK(1.6xx) | 不要 |
| **preview 版 Agility SDK(1.7xx-preview)** | **必須** |
| PIX の GPU キャプチャ | 必要 |

本書は retail 版を使うので、Agility SDK のためだけなら開発者モードは不要です。ただし第2章 2.4.4 節で PIX のために有効化を勧めているので、多くの読者はすでに有効になっているはずです。

有効化は `設定 → システム → 開発者向け → 開発者モード` から行います。

### 4.6.2 配布時に何を同梱するか

本書は学習が目的なので配布の話は本題ではありませんが、方針だけ示しておきます。

| ファイル | 開発時 | 配布時 |
|---|---|---|
| `D3D12Core.dll` | 必要 | **必要** |
| `d3d12SDKLayers.dll` | 必要 | **不要(除外する)** |
| `*.pdb` | あると便利 | 不要 |

`d3d12SDKLayers.dll` はデバッグレイヤーです。エンドユーザーの環境では使われないので、インストーラから除外します。DirectX チームも明確にそう推奨しています。

なお、フォルダ構成は `D3D12SDKPath` と一致させる必要があります。開発時に `D3D12\` を使っているなら、配布パッケージでも `D3D12\` です。

### 4.6.3 副次的な利点:デバッグレイヤーが OS 機能に依存しなくなる

Agility SDK を導入すると、思わぬところで楽になります。

従来、Direct3D 12 のデバッグレイヤーを使うには、Windows の**オプション機能「グラフィックス ツール」**をインストールする必要がありました。これがないと `D3D12GetDebugInterface` が `DXGI_ERROR_SDK_COMPONENT_MISSING` で失敗します。

Agility SDK では `d3d12SDKLayers.dll` をアプリが同梱するので、**OS のオプション機能に依存しません。** 新しい PC でも、環境構築の手数が一つ減ります。

第7章でデバッグレイヤーを有効にする際、この恩恵を受けることになります。

---

## 4.7 導入されたか確認する方法

4.1.4 節で述べた通り、Agility SDK は失敗しても黙っています。**実行時に確認する手段を、今のうちに作っておきます。**

### 4.7.1 確認の原理

2 つの Win32 API を使います。

**`GetModuleHandleW(L"D3D12Core.dll")`**
プロセスに読み込まれているモジュールのハンドルを返します。まだ読み込まれていなければ `nullptr` です。ここから `GetModuleFileNameW` でフルパスを取得すれば、**OS 標準版とアプリ同梱版のどちらが使われているかがわかります**。

```
C:\Windows\System32\D3D12Core.dll                    ← OS 標準(失敗)
C:\dev\...\build\x64\Debug\D3D12\D3D12Core.dll       ← Agility SDK(成功)
```

**`GetProcAddress(core, "D3D12SDKVersion")`**
`D3D12Core.dll` 自身も `D3D12SDKVersion` というシンボルをエクスポートしています。アプリ側と同じ名前ですが、こちらは「この DLL のバージョンはいくつか」を表します。読み出せば、**実際に読み込まれた実装のバージョン番号**が得られます。

### 4.7.2 ランタイムを読み込ませる

`D3D12Core.dll` は遅延読み込みされます。何らかの D3D12 API を呼ぶまで、プロセスには存在しません。

デバイスを作れば確実ですが、それは第7章の内容です。ここでは**デバイスを作らずにランタイムだけ初期化させる**方法を使います。

```cpp
::D3D12CreateDevice(nullptr, D3D_FEATURE_LEVEL_11_0,
                    __uuidof(ID3D12Device), nullptr);
```

第 4 引数に `nullptr` を渡すと、`D3D12CreateDevice` は**デバイスを作らず、作成可能かどうかだけを返します**。成功時の戻り値は `S_OK` ではなく `S_FALSE` です。

これは仕様として定められた使い方で、対応確認によく用いられます。第7章では同じ関数を正式に使います。

### 4.7.3 確認コードを書く

`src/main.cpp` を次のように書き換えます。

```cpp
// src/main.cpp
#include "pch.h"
#include "std_import.h"
#if USE_STD_MODULE
import std;
#endif

#include <GFSDK_Aftermath.h>

namespace
{
    //---------------------------------------------------------
    // D3D12 ランタイムの読み込み状態
    //---------------------------------------------------------
    struct RuntimeInfo
    {
        bool          loaded     = false;
        std::wstring  path;
        std::uint32_t sdkVersion = 0;
    };

    // 実行ファイルのあるディレクトリ
    std::filesystem::path GetExecutableDirectory()
    {
        wchar_t buffer[MAX_PATH]{};
        const DWORD length = ::GetModuleFileNameW(nullptr, buffer, MAX_PATH);
        if (length == 0 || length == MAX_PATH)
        {
            return {};
        }
        return std::filesystem::path(buffer).parent_path();
    }

    // D3D12 ランタイムを初期化させる。デバイスは作らない。
    // 第4引数に nullptr を渡すと「作成可能か」だけを返す(成功時 S_FALSE)。
    bool TouchD3D12Runtime()
    {
        const HRESULT hr = ::D3D12CreateDevice(
            nullptr,                    // 既定のアダプタ
            D3D_FEATURE_LEVEL_11_0,
            __uuidof(ID3D12Device),
            nullptr);                   // デバイスは作らない
        return SUCCEEDED(hr);
    }

    // 読み込まれている D3D12Core.dll を調べる
    RuntimeInfo QueryRuntimeInfo()
    {
        RuntimeInfo info{};

        const HMODULE core = ::GetModuleHandleW(L"D3D12Core.dll");
        if (core == nullptr)
        {
            return info;
        }
        info.loaded = true;

        wchar_t buffer[MAX_PATH]{};
        if (::GetModuleFileNameW(core, buffer, MAX_PATH) != 0)
        {
            info.path = buffer;
        }

        // D3D12Core.dll 自身がエクスポートしているバージョン番号を読む
        if (const auto* version = reinterpret_cast<const std::uint32_t*>(
                ::GetProcAddress(core, "D3D12SDKVersion")))
        {
            info.sdkVersion = *version;
        }

        return info;
    }
}

int main()
{
    std::println("=== D3D12Book : Environment Check ===");
    std::println("");

    //---------------------------------------------------------
    // ビルド環境
    //---------------------------------------------------------
    std::println("[Build]");
    std::println("  MSVC          : {}", _MSC_VER);
#if USE_STD_MODULE
    std::println("  std library   : import std (module)");
#else
    std::println("  std library   : #include (header)");
#endif
#ifdef D3D12_SDK_VERSION
    std::println("  header SDK    : {}", D3D12_SDK_VERSION);
#endif
    std::println("");

    //---------------------------------------------------------
    // Agility SDK
    //---------------------------------------------------------
    std::println("[Agility SDK]");

    if (!TouchD3D12Runtime())
    {
        std::println("  D3D12 runtime : FAILED to initialize");
        return 1;
    }

    const RuntimeInfo runtime = QueryRuntimeInfo();

    if (!runtime.loaded)
    {
        std::println("  D3D12Core.dll : not loaded");
        return 1;
    }

    const auto exeDir = GetExecutableDirectory();
    const bool isLocal =
        std::filesystem::path(runtime.path).native().starts_with(exeDir.native());

    std::println("  D3D12Core.dll : {}", runtime.path);
    std::println("  SDK version   : {}", runtime.sdkVersion);
    std::println("  source        : {}",
                 isLocal ? "Agility SDK (application-local)  [OK]"
                         : "OS inbox runtime  [Agility SDK NOT active]");
    std::println("");

    //---------------------------------------------------------
    // Nsight Aftermath
    //---------------------------------------------------------
    std::println("[Nsight Aftermath]");

    constexpr std::uint32_t aftermathVersion = GFSDK_Aftermath_Version_API;
    std::println("  SDK version   : {}.{}",
                 (aftermathVersion >> 16) & 0xFFFF,
                 aftermathVersion & 0xFFFF);

    const auto dllPath = exeDir / L"GFSDK_Aftermath_Lib.x64.dll";
    std::println("  DLL           : {}",
                 std::filesystem::exists(dllPath) ? "found" : "NOT FOUND");
    std::println("");

    return isLocal ? 0 : 1;
}
```

> **`D3D12_SDK_VERSION` について**
>
> ヘッダ側が定義しているバージョン番号です。`#ifdef` で囲んであるのは、SDK のバージョンによって定義の有無が変わりうるためです。定義されていれば、**ヘッダのバージョンとランタイムのバージョンが一致しているか**を目で比べられます。
>
> 両者がずれている場合、ヘッダは新しいのにランタイムが古い(またはその逆)という状態です。`D3D12SDKVersion` の値と NuGet パッケージのバージョンを確認してください。

---

## ✅ 本章のゴール:Agility SDK が有効であることを確認する

### Step 1:ビルドする

`Ctrl + Shift + B`

### Step 2:出力ディレクトリを確認する

```
build/x64/Debug/
├─ D3D12Book.exe
├─ GFSDK_Aftermath_Lib.x64.dll
└─ D3D12/
    ├─ D3D12Core.dll
    └─ d3d12SDKLayers.dll
```

`D3D12/` フォルダが存在し、中に DLL が入っていることを確認してください。

### Step 3:エクスポートを確認する

x64 Native Tools Command Prompt で:

```
dumpbin /exports build\x64\Debug\D3D12Book.exe
```

`D3D12SDKPath` と `D3D12SDKVersion` の 2 つが並んでいること。

### Step 4:実行する

`Ctrl + F5`

**成功時の出力**

```
=== D3D12Book : Environment Check ===

[Build]
  MSVC          : 1951
  std library   : import std (module)
  header SDK    : 619

[Agility SDK]
  D3D12Core.dll : C:\dev\D3D12Book\build\x64\Debug\D3D12\D3D12Core.dll
  SDK version   : 619
  source        : Agility SDK (application-local)  [OK]

[Nsight Aftermath]
  SDK version   : 2025.5
  DLL           : found
```

**失敗時の出力**

```
[Agility SDK]
  D3D12Core.dll : C:\Windows\System32\D3D12Core.dll
  SDK version   : 614
  source        : OS inbox runtime  [Agility SDK NOT active]
```

パスが `System32` になっていたら、エクスポートシンボルが効いていません。次のトラブルシューティングを参照してください。

---

### 本章の達成状態

- [ ] NuGet で `Microsoft.Direct3D.D3D12`(retail 版)を追加した
- [ ] `pch.h` に `d3d12.h` / `dxgi1_6.h` / `dxgidebug.h` を追加した
- [ ] `d3d12.lib` / `dxgi.lib` / `dxguid.lib` をリンクした
- [ ] `src/AgilitySDK.cpp` を作り、2 つのシンボルをエクスポートした
- [ ] `D3D12SDKVersion` の値が NuGet パッケージのバージョンと一致している
- [ ] `dumpbin /exports` で 2 つのシンボルが確認できた
- [ ] `build/x64/Debug/D3D12/` に DLL がコピーされている
- [ ] 実行して `source : Agility SDK (application-local)` と表示された
- [ ] `d3dx12.h` を include していない

---

## トラブルシューティング

| 症状 | 原因 | 対処 |
|---|---|---|
| `source : OS inbox runtime` | エクスポートが効いていない | まず `dumpbin /exports` で確認 |
| `dumpbin` に 2 つのシンボルが出ない | `extern` の書き忘れ | 4.3.3 を確認 |
| 同上 | `AgilitySDK.cpp` がプロジェクトに未追加 | ソリューションエクスプローラーで確認 |
| `D3D12Core.dll : not loaded` | `D3D12CreateDevice` が呼ばれていない | `TouchD3D12Runtime()` の戻り値を確認 |
| SDK version がヘッダと違う | 番号の不一致 | `D3D12SDKVersion` の値と NuGet のバージョンを一致させる |
| `const char8_t` からの変換エラー | `u8` プレフィックス | 外す(4.3.2) |
| `D3D12_BARRIER_TYPE_*` が未定義 | 古い `d3d12.h` を掴んでいる | インクルードパスの順序(4.2.4) |
| `LNK2001: IID_ID3D12Device` | `dxguid.lib` 未リンク | 追加の依存ファイルに追加 |
| `D3D12/` フォルダができない | `.targets` が動いていない | NuGet を再インストール、または手動コピー(4.4.2) |
| 新機能だけ動かない | ドライバまたは GPU 世代 | 4.5.3 の順序で切り分け |

---

## まとめ

**1. Direct3D 12 のランタイムは 2 つの DLL に分かれている。**
`d3d12.dll` がローダー、`D3D12Core.dll` が実装本体です。後者を差し替えられるようにしたのが Agility SDK です。

**2. 失敗しても何も起きない。**
エクスポートシンボルがなければ、静かに OS 標準版が使われます。コンパイルもリンクも実行も成功します。だからこそ、実行時に確認する手段を持つ必要があります。

**3. C++23 では `u8` プレフィックスを外す。**
公式ドキュメントの例は C++17 時代のものです。`const char8_t*` から `const char*` への暗黙変換は C++20 でなくなりました。

**4. `extern` を省略すると内部リンケージになる。**
`const` は名前空間スコープで既定が内部リンケージです。`__declspec(dllexport)` だけでは足りません。

**5. Agility SDK はランタイムしか解決しない。**
機能が使えるかどうかは、OS × ランタイム × ドライバ × ハードウェアの積で決まります。

次章では Win32 のウィンドウを作ります。ここまで配管工事が続きましたが、次章の終わりには画面に何かが表示されます。

---

## 参考リンク

| 内容 | URL |
|---|---|
| Agility SDK ダウンロード | https://devblogs.microsoft.com/directx/directx12agility/ |
| D3D12 再頒布可能ランタイム仕様 | https://microsoft.github.io/DirectX-Specs/d3d/D3D12Redistributable.html |
| Agility SDK 入門(公式ブログ) | https://devblogs.microsoft.com/directx/gettingstarted-dx12agility/ |
| Shader Model 6.9 / Agility SDK 1.619 | https://devblogs.microsoft.com/directx/shader-model-6-9-retail-and-more/ |
| DirectX Developer Blog | https://devblogs.microsoft.com/directx/ |

> **本章の記述の基準日:2026 年 7 月 31 日**
>
> Agility SDK 1.619.0(retail)を前提としています。`D3D12SDKVersion` の値は、読者がインストールしたパッケージのバージョンに合わせて読み替えてください。**この数字の一致だけは、絶対に妥協できません。**
