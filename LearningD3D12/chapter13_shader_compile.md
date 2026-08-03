# 第13章 シェーダーを書いてコンパイルする

ここから「絵を描く」段階に入ります。まずはシェーダーです。

本章で書く HLSL は、**わずか 20 行**です。頂点を素通しして、色をそのまま返すだけの、これ以上ないほど単純なシェーダーです。それなのに章がこの長さになるのは、**コンパイルの設定に手を抜けない理由がある**からです。

第2章 2.4.3 節、第8章、そして第31章 —— 本書は繰り返し「Aftermath が GPU クラッシュの原因を HLSL の行番号で教えてくれる」と書いてきました。**それが実現するかどうかは、本章のビルド設定で決まります。**

デバッグ情報を出力せずにビルドしたシェーダーは、クラッシュしても機械語のアドレスしか教えてくれません。しかも、**後から気づいても手遅れです。** そのビルドで作られたダンプは、二度と解読できません。

だから、まだ三角形も出ていないこの段階で、**PDB の分離出力**を設定します。

**本章のゴール**
最小の頂点シェーダーとピクセルシェーダーを書き、DXC でコンパイルする。`.cso` と、シェーダーハッシュを名前とする `.pdb` が出力され、C++ 側から読み込めることを確認する。

---

## 13.1 HLSL とシェーダーステージの概観

### 13.1.1 グラフィックスパイプライン

三角形が画面のピクセルになるまでの流れです。

```
    頂点バッファ
        ↓
┌───────────────────┐
│ Input Assembler   │ 固定機能。頂点データを組み立てる
└─────────┬─────────┘
          ↓
┌───────────────────┐
│ Vertex Shader     │ ★ 頂点ごとに 1 回実行
└─────────┬─────────┘
          ↓
┌───────────────────┐
│ Hull / Tess / Dom │ ☆ テッセレーション(省略可)
└─────────┬─────────┘
          ↓
┌───────────────────┐
│ Geometry Shader   │ ☆ 省略可。NVIDIA では遅いので避ける
└─────────┬─────────┘
          ↓
┌───────────────────┐
│ Rasterizer        │ 固定機能。三角形をピクセルに分解
└─────────┬─────────┘
          ↓
┌───────────────────┐
│ Pixel Shader      │ ★ ピクセルごとに 1 回実行
└─────────┬─────────┘
          ↓
┌───────────────────┐
│ Output Merger     │ 固定機能。深度テスト、ブレンド
└─────────┬─────────┘
          ↓
    レンダーターゲット
```

★ が**本章で書く 2 つ**、☆ が本書では使わないものです。

**ジオメトリシェーダーについて。** 使えますが、本書では最後まで使いません。NVIDIA のハードウェアでは実装が非効率で、性能が大きく落ちるためです。同じことをしたければ、第36章のメッシュシェーダーが正しい後継です。

なお、コンピュートシェーダー(第32章)とメッシュシェーダー(第36章)は、この流れとは別の経路を持ちます。

### 13.1.2 ターゲット文字列

コンパイル時には、**どのステージ用か**と**どのシェーダーモデルか**を指定します。

```
vs_6_6
│  │
│  └── シェーダーモデル 6.6
└───── Vertex Shader
```

| 接頭辞 | ステージ | 登場する章 |
|---|---|---|
| `vs_` | 頂点シェーダー | **本章** |
| `ps_` | ピクセルシェーダー | **本章** |
| `cs_` | コンピュートシェーダー | 第32章 |
| `ms_` | メッシュシェーダー | 第36章 |
| `as_` | Amplification シェーダー | 第36章 |
| `lib_` | ライブラリ(レイトレーシング) | 第37章 |

**本書のベースラインは `6_6` です。**

第7章で `CheckFeatureSupport` を実装したとき、Shader Model 6.6 以上が使えることを確認しました(Turing 世代以降)。6.6 を基準にする理由は 3 つあります。

- 第33章のバインドレス(Dynamic Resources)が 6.6 を要求する
- 全部同じターゲットにしておけば、どのシェーダーがどのモデルか覚える必要がない
- 対象ハードウェアで確実に動くことを確認済み

第37章で DXR 1.2 を扱うときだけ、`6_9` に上げます。

### 13.1.3 セマンティクス

HLSL 特有の概念です。**変数に「これは何を表すか」というラベルを付けます。**

```hlsl
float3 position : POSITION;
//                ^^^^^^^^ セマンティクス
```

セマンティクスには 2 種類あります。

| 種類 | 例 | 意味 |
|---|---|---|
| **ユーザー定義** | `POSITION` `COLOR` `TEXCOORD0` | 名前は自由。ステージ間で一致していればよい |
| **システム値**(`SV_` 接頭辞) | `SV_Position` `SV_Target` | パイプラインの固定機能部に対する特別な意味を持つ |

**`SV_` が付くものは特別です。**

| セマンティクス | 意味 |
|---|---|
| `SV_Position` | 頂点シェーダーの出力では**クリップ空間の座標**。ピクセルシェーダーの入力では**画面上のピクセル座標** |
| `SV_Target` | ピクセルシェーダーの出力先レンダーターゲット |
| `SV_VertexID` | 頂点の通し番号(入力なしで描画するときに使う) |
| `SV_InstanceID` | インスタンス番号(第34章) |

**`SV_Position` は必ず `float4` でなければなりません。** `float3` にするとコンパイルエラーになります。

ユーザー定義セマンティクスは、**頂点シェーダーの出力とピクセルシェーダーの入力で名前が一致していれば**、そのまま値が渡ります。名前が一致しないと、ピクセルシェーダー側の値が未定義になります(デバッグレイヤーが警告します)。

---

## 13.2 DXC と FXC、そして DXIL

### 13.2.1 2 つのコンパイラ

| | FXC | **DXC** |
|---|---|---|
| 実体 | `fxc.exe` | `dxc.exe` / `dxcompiler.dll` |
| 基盤 | 独自実装 | **LLVM / Clang** |
| 出力形式 | DXBC | **DXIL** |
| 対応シェーダーモデル | 〜 5.1 | **6.0 〜** |
| 状態 | メンテナンスのみ | **現行** |

**本書は DXC のみを使います。** Shader Model 6.0 以降は FXC ではコンパイルできません。

DXC が Clang ベースであることには、実用的な恩恵があります。**エラーメッセージが格段に読みやすくなりました**(13.8 節)。

### 13.2.2 DXIL とは

DXC が出力する中間表現です。**LLVM のビットコードを基にしています。**

```
HLSL ソース → [DXC] → DXIL → [ドライバ] → GPU の機械語
```

DXIL は最終的な機械語ではありません。**ドライバが実行時に、そのハードウェア向けの命令に変換します。** PSO を生成する瞬間(第14章)がその変換のタイミングです。

このため、シェーダーのコンパイルエラーは 2 段階で起こりえます。

| 段階 | いつ | 誰が検出するか |
|---|---|---|
| HLSL → DXIL | ビルド時 | DXC |
| DXIL → 機械語 | PSO 生成時 | ドライバ |

**後者は第14章の主題です。** DXC が通したのに `CreateGraphicsPipelineState` が失敗する、ということが起こります。

### 13.2.3 DXIL には署名が必要

**見落とすと必ず引っかかる仕様です。**

DXIL は、`dxil.dll` によって**検証され、署名されます。** 署名のない DXIL は、ランタイムが受け付けません(Windows の開発者モードが有効な場合を除く)。

そして `dxil.dll` は、**`dxc.exe` と同じディレクトリに存在しなければなりません。** なければ DXC は署名を行わず、警告だけ出して未署名のシェーダーを吐きます。

```
warning: DXIL signing library (dxil.dll) not found.
Resulting DXIL will not be signed for use in release environments.
```

**この警告を無視しないでください。** 開発者モードが有効な自分の PC では動き、無効な環境では動かない、という状態になります。

### 13.2.4 DXC はどこから入手するか

| 入手元 | 特徴 |
|---|---|
| Windows SDK | `bin/x64/dxc.exe`。すぐ使えるが、**バージョンが古いことが多い** |
| **NuGet `Microsoft.Direct3D.DXC`** | **最新。本書はこれ** |
| GitHub リリース | NuGet と同じ内容 |

**新しいシェーダーモデルは、新しい DXC がなければコンパイルできません。** Shader Model 6.9 を使う第37章では、Windows SDK 同梱版では足りない可能性が高いです。

第2章で Aftermath SDK を `external/` に置いたのと同じ形で配置します。

```
D3D12Book/
├─ external/
│   ├─ NsightAftermath/
│   └─ dxc/                    ← 追加
│       ├─ bin/x64/
│       │   ├─ dxc.exe
│       │   ├─ dxcompiler.dll
│       │   └─ dxil.dll        ← 署名に必要(13.2.3)
│       ├─ inc/
│       │   └─ dxcapi.h
│       └─ lib/x64/
│           └─ dxcompiler.lib
```

**入手方法**

NuGet パッケージ `Microsoft.Direct3D.DXC` をダウンロードし、`.nupkg`(実体は ZIP)を展開して `build/native/` 以下を `external/dxc/` にコピーします。

VS の NuGet マネージャからプロジェクトに追加しても構いませんが、**本書は明示的に配置する方式を採ります。** どのバージョンの DXC を使っているかがディレクトリを見れば分かり、後から差し替えるのも簡単だからです。

---

## 13.3 バージョンの整合

### 13.3.1 三点セット

第4章 4.5.1 節で「機能が使えるかは 4 つの条件の積」と書きました。シェーダーの場合は次の 3 つが揃う必要があります。

```
シェーダーが動く = DXC のバージョン      (コンパイルできるか)
                 × Agility SDK のバージョン (ランタイムが受け付けるか)
                 × ドライバ               (ハードウェアが実行できるか)
```

具体例です。

| 使いたいもの | DXC | Agility SDK | ドライバ |
|---|---|---|---|
| Shader Model 6.6(第33章) | 1.6 以降 | 1.4 以降 | 対応版 |
| Shader Model 6.9 / DXR 1.2(第37章) | **1.9 以降** | **1.619 以降** | 対応版 |

**症状の出方が段階ごとに違います。**

| どこが古いか | 症状 |
|---|---|
| DXC | `error: invalid profile vs_6_9` などのコンパイルエラー |
| Agility SDK | PSO 生成が `E_INVALIDARG` で失敗 |
| ドライバ | `CheckFeatureSupport` が非対応を返す |

**エラーの出る場所で、どれが古いかが分かります。** 第4章 4.5.3 節の切り分け表に、この列が加わったと考えてください。

### 13.3.2 本書のベースライン

執筆時点で使用しているバージョンです。

| 項目 | バージョン |
|---|---|
| DXC | 1.9(2026 年 7 月版) |
| Agility SDK | 1.619(retail) |
| シェーダーモデル(既定) | **6.6** |
| シェーダーモデル(第37章) | 6.9 |
| HLSL 言語バージョン | 2021 |

`-HV 2021` を明示的に指定します。DXC の既定値はバージョンによって変わるので、**明示しておいたほうが再現性が高くなります。**

---

## 13.4 ビルド時コンパイルと実行時コンパイル

### 13.4.1 2 つの方式

| | ビルド時 | 実行時 |
|---|---|---|
| 手段 | `dxc.exe` | `IDxcCompiler3`(`dxcompiler.dll`) |
| エラーの発見 | **ビルド時** | 実行時 |
| 起動速度 | **速い** | 遅い |
| 配布物 | `.cso` のみ | `.hlsl` + `dxcompiler.dll` + `dxil.dll` |
| シェーダーのホットリロード | できない | **できる** |

### 13.4.2 本書はビルド時を選ぶ

理由は 3 つです。

**1. エラーがビルド時に出る**
シェーダーのタイプミスでアプリが起動しない、という状況を避けられます。Visual Studio のエラー一覧に表示され、ダブルクリックで該当行へ飛べます(13.8 節)。

**2. 配布物が減る**
`dxcompiler.dll` と `dxil.dll`(合計 30MB 以上)を同梱せずに済みます。

**3. フラグを完全に制御できる**
13.6 節で扱う PDB の分離出力は、フラグの指定が肝心です。**ビルドスクリプトに書いておけば、確実に、毎回、同じ設定でコンパイルされます。**

### 13.4.3 実行時コンパイルについて

**シェーダーのホットリロードは、開発効率を大きく変えます。** アプリを起動したまま `.hlsl` を保存すると、数百ミリ秒で結果が画面に反映される —— これに慣れると戻れません。

実装の骨格だけ示しておきます。

```cpp
#include <dxcapi.h>

ComPtr<IDxcUtils>     utils;
ComPtr<IDxcCompiler3> compiler;
::DxcCreateInstance(CLSID_DxcUtils,    IID_PPV_ARGS(&utils));
::DxcCreateInstance(CLSID_DxcCompiler, IID_PPV_ARGS(&compiler));

const wchar_t* args[] = {
    L"Triangle.hlsl",
    L"-E", L"VSMain",
    L"-T", L"vs_6_6",
    L"-Zi",
    L"-Qstrip_reflect",
};

DxcBuffer source{};
source.Ptr      = sourceText.data();
source.Size     = sourceText.size();
source.Encoding = DXC_CP_UTF8;

ComPtr<IDxcResult> result;
compiler->Compile(&source, args, std::size(args),
                  includeHandler.Get(), IID_PPV_ARGS(&result));

// エラーメッセージの取り出し
ComPtr<IDxcBlobUtf8> errors;
result->GetOutput(DXC_OUT_ERRORS, IID_PPV_ARGS(&errors), nullptr);
if (errors && errors->GetStringLength() > 0)
{
    LOG_ERROR(L"{}", Core::ToWide(errors->GetStringPointer()));
}

// 本体の取り出し
ComPtr<IDxcBlob> object;
result->GetOutput(DXC_OUT_OBJECT, IID_PPV_ARGS(&object), nullptr);
```

**エラーメッセージが `IDxcBlobUtf8`(ナロー文字列)で返る**点に注目してください。第6章 6.3.4 節で `ToWide` を用意した理由の一つです。

本書ではこの経路を使いませんが、**第29章で Nsight Graphics を導入した後、開発効率を上げたくなったら実装してみてください。** 必要な部品はすべて揃っています。

---

## 13.5 最小のシェーダーを書く

### 13.5.1 `Triangle.hlsl`

プロジェクトに `shaders/` ディレクトリを作り、次のファイルを置きます。

```hlsl
//=====================================================
// shaders/Triangle.hlsl
//
// 最小の頂点シェーダーとピクセルシェーダー。
// 第15章で、このシェーダーを使って三角形を描く。
//=====================================================

//-----------------------------------------------------
// 入力:頂点バッファから来るデータ
//   セマンティクス名は、第14章の InputLayout と
//   一致させる必要がある。
//-----------------------------------------------------
struct VSInput
{
    float3 position : POSITION;
    float4 color    : COLOR;
};

//-----------------------------------------------------
// 頂点シェーダーの出力 = ピクセルシェーダーの入力
//   SV_Position は必須。float4 でなければならない。
//-----------------------------------------------------
struct VSOutput
{
    float4 position : SV_Position;
    float4 color    : COLOR;
};

//-----------------------------------------------------
// 頂点シェーダー
//   今回は座標変換をせず、そのまま出力する。
//   つまり、頂点座標はすでにクリップ空間にある前提。
//   行列による変換は第18章で導入する。
//-----------------------------------------------------
VSOutput VSMain(VSInput input)
{
    VSOutput output;
    output.position = float4(input.position, 1.0f);
    output.color    = input.color;
    return output;
}

//-----------------------------------------------------
// ピクセルシェーダー
//   補間された色をそのまま返す。
//-----------------------------------------------------
float4 PSMain(VSOutput input) : SV_Target
{
    return input.color;
}
```

**20 行です。** これ以上削れません。

### 13.5.2 何が起きているのか

**頂点シェーダー**は、入力された座標に `w = 1.0` を足して 4 成分にし、そのまま出力しています。

「変換をしていないなら、頂点はどこに描かれるのか」という疑問が出ます。答えは、**クリップ空間の座標としてそのまま解釈される**です。

クリップ空間では、画面に映る範囲が次のように決まっています。

```
x: -1.0(左端)  〜  +1.0(右端)
y: -1.0(下端)  〜  +1.0(上端)
z:  0.0(手前)  〜   1.0(奥)      ← D3D は [0,1]。OpenGL の [-1,1] とは違う
```

だから第15章では、`(0, 0.5, 0)` のような、はじめからこの範囲に収まる座標を与えます。**行列による変換は第18章で導入します。**

**ピクセルシェーダー**は、色をそのまま返しています。3 つの頂点に別々の色を与えると、**ラスタライザが自動的に補間**するので、グラデーションのかかった三角形になります。第15章 15.6 節でこれを確認します。

---

## 13.6 デバッグ情報を出力する

**本章でもっとも重要な節です。**

### 13.6.1 3 つの方式

DXC がシェーダーのデバッグ情報を扱う方式は 3 つあります。

| 方式 | フラグ | 出力 |
|---|---|---|
| **埋め込み** | `-Zi -Qembed_debug` | `.cso` の中にデバッグ情報が入る |
| **分離** | `-Zi -Fd <パス>` | 別ファイル(PDB)に出る |
| **スリム** | `-Zs -Fd <パス>` | 最小限の PDB(ソースとコンパイル引数のみ) |

**埋め込みは避けてください。** シェーダーのバイナリが数倍に膨らみます。配布するものに開発用の情報を含めることにもなります。

**スリム(`-Zs`)は魅力的です。** PDB が小さく、コンパイルも速くなります。ソースコードとコンパイル引数だけを保存し、詳細なデバッグ情報は必要になった時点でツール側が再生成します。

**本書は分離方式(`-Zi -Fd`)を使います。** スリムでも多くのツールは動作しますが、**Aftermath のシェーダーソース対応は `-Zi` を前提として文書化されている**ため、確実な側を選びます。

### 13.6.2 `-Fd` にディレクトリを渡す

**ここが決め手です。**

```
-Fd out\shaders\pdb\Triangle.VS.pdb    ← ファイル名を指定
-Fd out\shaders\pdb\                   ← ディレクトリを指定(末尾の \ が重要)
```

**後者を使います。**

ディレクトリを指定すると、DXC は**シェーダーのハッシュから自動生成した名前**で PDB を書き出します。そして、**そのハッシュはシェーダーバイナリの中にも記録されます。**

```
Triangle.VS.cso  ─── 中に記録されたハッシュ ──→  <hash>.pdb
```

デバッグツールは、シェーダーバイナリからハッシュを読み取り、その名前の PDB を探します。**ファイル名の対応表を自分で管理する必要がありません。**

**末尾のバックスラッシュを忘れないでください。** ないとファイル名として解釈されます。

### 13.6.3 なぜ今これをやるのか

**第31章への布石です。**

第31章で、GPU クラッシュの原因を HLSL の行番号まで特定する実習を行います。それが成立するには、次の 2 つが必要です。

| 必要なもの | いつ用意するか |
|---|---|
| **ドライバが生成するシェーダーデバッグ情報** | 実行時。Aftermath の機能フラグで有効化(第31章 31.1.4 節) |
| **DXC が出力する PDB** | **ビルド時。本章** |

NVIDIA の Aftermath ドキュメントが示す推奨手順も、まさにこの形です。

```
dxc -Zi [..] -Fo shader.bin -Fd debugInfo\ shader.hlsl
```

コンパイラが生成する PDB の名前は、**シェーダーの DebugName と一致します。** 第31章では `GFSDK_Aftermath_GetShaderDebugName` でシェーダーバイナリから DebugName を取り出し、その名前の PDB を渡す、という流れになります。

**そしてここが肝心なのですが —— 後から設定しても手遅れです。**

デバッグ情報なしでビルドしたシェーダーがクラッシュしても、そのときのダンプには機械語のアドレスしか残りません。「よし、デバッグ情報を有効にして再ビルドしよう」と思っても、**再ビルドしたバイナリは別物**なので、既に取得したダンプとは対応しません。再現させ直すところからやり直しです。

**だから、まだ三角形も出ていない今、設定します。**

> **PDB は捨てないこと**
>
> `build/` はバージョン管理の対象外にしました(第3章 3.2.5 節)。開発中はそれで構いません。
>
> しかし、**エンドユーザーからクラッシュダンプを回収する設計にするなら、リリースビルドの PDB は保管しなければなりません。** ダンプだけあっても、対応する PDB がなければ解読できないからです。第31章 31.6.2 節で改めて扱います。

### 13.6.4 本書のフラグ

```
-T vs_6_6          ターゲット(ステージとシェーダーモデル)
-E VSMain          エントリポイント
-HV 2021           HLSL 言語バージョンを明示
-Zi                デバッグ情報を生成
-Qstrip_reflect    リフレクション情報を .cso から除く
-Fo <出力>.cso     コンパイル結果
-Fd <PDBディレクトリ>\   PDB の出力先(末尾の \ が必須)
-WX                警告をエラーとして扱う
```

`-Qstrip_reflect` は、リフレクション情報(シェーダーの入出力や定数バッファの構造を記述したデータ)を `.cso` から取り除きます。**本書はリフレクションを使わない**ので、ファイルを小さくできます。

`-WX` については 13.8.3 節で説明します。

**Debug ビルドと Release ビルドで最適化オプションを分けます。**

| 構成 | 追加フラグ |
|---|---|
| Debug | `-Od`(最適化なし。デバッグしやすい) |
| Release | `-O3`(既定。省略可) |

---

## 13.7 ビルドに組み込む

### 13.7.1 DXC のパスを定義する

`.vcxproj` にプロパティを追加します(ソリューションエクスプローラーでプロジェクトを右クリック → 「プロジェクトのアンロード」→ 編集)。

```xml
<PropertyGroup Label="UserMacros">
  <!-- external/dxc/ に配置した DXC を使う(13.2.4)。
       Windows SDK 同梱版を使う場合は次に差し替える:
       $(WindowsSdkVerBinPath)x64\dxc.exe -->
  <DxcExe>$(SolutionDir)external\dxc\bin\x64\dxc.exe</DxcExe>
  <ShaderOutDir>$(OutDir)shaders\</ShaderOutDir>
  <ShaderPdbDir>$(OutDir)shaders\pdb\</ShaderPdbDir>
</PropertyGroup>
```

### 13.7.2 カスタムビルドツールを設定する

1. `shaders/Triangle.hlsl` をプロジェクトに追加する
2. ファイルを右クリック → プロパティ
3. **「項目の種類」を「カスタム ビルド ツール」に変更**して適用

> **「HLSL コンパイラ」ではなくカスタムビルドツールを使う理由**
>
> Visual Studio には HLSL 用の項目の種類が用意されています。しかし、`-Fd` によるディレクトリ指定など、本書に必要なフラグを細かく制御できません。
>
> **本書は「何が実行されているかを全部知っている」ことを重視します**(第1章 1.3 節)。コマンドラインを自分で書くほうが、この方針に合っています。

「カスタム ビルド ツール」のプロパティに、次を設定します。

**コマンド ライン**(すべての構成)

```
if not exist "$(ShaderPdbDir)" mkdir "$(ShaderPdbDir)"
"$(DxcExe)" -T vs_6_6 -E VSMain -HV 2021 -Zi -WX -Qstrip_reflect -Fo "$(ShaderOutDir)Triangle.VS.cso" -Fd "$(ShaderPdbDir)" "%(FullPath)"
"$(DxcExe)" -T ps_6_6 -E PSMain -HV 2021 -Zi -WX -Qstrip_reflect -Fo "$(ShaderOutDir)Triangle.PS.cso" -Fd "$(ShaderPdbDir)" "%(FullPath)"
```

**説明**

```
Compiling Triangle.hlsl
```

**出力ファイル**

```
$(ShaderOutDir)Triangle.VS.cso;$(ShaderOutDir)Triangle.PS.cso
```

**追加の依存ファイル**(インクルードする `.hlsli` があれば列挙する。今は空でよい)

> **「出力ファイル」を正しく書く意味**
>
> ここに書いたファイルのタイムスタンプと `.hlsl` を比較して、**MSBuild が再ビルドの要否を判断します。** 空にすると毎回コンパイルされ、間違ったパスを書くと変更しても再コンパイルされません。
>
> 「シェーダーを直したのに反映されない」の原因は、たいていここです。

**Debug 構成だけ `-Od` を追加**したい場合は、構成を「Debug」に切り替えてコマンドラインを編集します。

### 13.7.3 出力の構成

ビルドすると、次のようになります。

```
build/x64/Debug/
├─ D3D12Book.exe
├─ GFSDK_Aftermath_Lib.x64.dll
├─ D3D12/                          ← Agility SDK(第4章)
│   ├─ D3D12Core.dll
│   └─ d3d12SDKLayers.dll
└─ shaders/                        ← 本章で追加
    ├─ Triangle.VS.cso
    ├─ Triangle.PS.cso
    └─ pdb/
        ├─ 1a2b3c4d....pdb         ← ハッシュが名前になる
        └─ 5e6f7a8b....pdb
```

**PDB のファイル名がハッシュになっていることを確認してください。** `Triangle.VS.pdb` のような名前になっていたら、`-Fd` の末尾のバックスラッシュが抜けています(13.6.2 節)。

---

## 13.8 コンパイルエラーの読み方

### 13.8.1 DXC のエラー形式

DXC は Clang ベースなので、**エラーメッセージも Clang 形式**です。

```
C:\dev\D3D12Book\shaders\Triangle.hlsl(28,12): error: use of undeclared identifier 'positon'; did you mean 'position'?
    output.positon = float4(input.position, 1.0f);
           ^~~~~~~
           position
```

- **ファイル名と行番号、列番号**が出る
- 該当行が引用され、`^` で位置が示される
- **候補まで提示される**

そして、Visual Studio の**エラー一覧に表示され、ダブルクリックでその行にジャンプできます。** `ファイル名(行,列): error:` という形式が、VS の認識するパターンだからです。

第6章 6.4.1 節でログの書式を `ファイル名(行番号):` にしたのと同じ理屈です。

### 13.8.2 よくあるエラー

| エラー | 原因 |
|---|---|
| `use of undeclared identifier` | タイプミス |
| `entry point not found` | `-E` の名前が関数名と違う |
| `invalid profile vs_6_9` | **DXC が古い**(13.3.1) |
| `Semantic must be float4` | `SV_Position` を `float3` にした |
| `cannot convert from 'float3' to 'float4'` | 型の不一致。`float4(v, 1.0f)` で拡張する |
| `no matching function for call to ...` | 引数の型か数が違う |
| `SV_Target semantic must be used on ...` | ピクセルシェーダー以外で `SV_Target` を使った |

### 13.8.3 `-WX` を付ける

**警告をエラーとして扱います。**

シェーダーの警告は、放置すると溜まります。そして溜まると読まなくなります。

```
warning: potential division by zero
warning: implicit truncation of vector type
```

2 つ目は特に危険で、`float4` を `float3` に代入したときなどに出ます。**動きますが、意図した動作かどうかは分かりません。**

**最初から `-WX` を付けておけば、警告が溜まる余地がありません。** 意図的に無視したい警告があれば、`-Wno-<name>` で個別に抑制します。第7章 7.6.3 節でデバッグレイヤーのメッセージ抑制について書いたのと同じ方針です —— **「全部黙らせる」ではなく「これは調べたうえで無視する」** を残します。

---

## 13.9 `.cso` を読み込む

コンパイル結果を C++ 側から読みます。**ただのバイナリファイルなので、特別な API は不要です。**

```cpp
// src/Graphics/ShaderBlob.h
#pragma once
#include "std_import.h"
#include "Core/Error.h"

namespace Graphics
{
    class ShaderBlob
    {
    public:
        Core::Status LoadFromFile(const std::filesystem::path& path);

        D3D12_SHADER_BYTECODE Bytecode() const noexcept
        {
            return D3D12_SHADER_BYTECODE{
                m_data.data(),
                m_data.size()
            };
        }

        bool  Empty() const noexcept { return m_data.empty(); }
        std::size_t Size() const noexcept { return m_data.size(); }

    private:
        std::vector<std::byte> m_data;
    };

    // 実行ファイルのあるディレクトリを基準にした shaders/ のパス
    std::filesystem::path ShaderDirectory();
}
```

```cpp
// src/Graphics/ShaderBlob.cpp(抜粋)

Core::Status ShaderBlob::LoadFromFile(const std::filesystem::path& path)
{
    std::error_code ec;
    const auto size = std::filesystem::file_size(path, ec);
    if (ec)
    {
        LOG_ERROR(L"shader not found: {}", path.wstring());
        return std::unexpected(Core::MakeError(
            HRESULT_FROM_WIN32(ERROR_FILE_NOT_FOUND), L"shader not found"));
    }

    std::ifstream file(path, std::ios::binary);
    if (!file)
    {
        return std::unexpected(Core::MakeError(
            E_FAIL, L"failed to open shader file"));
    }

    m_data.resize(static_cast<std::size_t>(size));
    file.read(reinterpret_cast<char*>(m_data.data()),
              static_cast<std::streamsize>(size));

    if (!file)
    {
        return std::unexpected(Core::MakeError(
            E_FAIL, L"failed to read shader file"));
    }

    //--- DXIL コンテナの検査 ---
    // DXIL は DXBC コンテナ形式を流用しているため、
    // 先頭 4 バイトは 'D','X','B','C' になる。
    if (m_data.size() < 4 ||
        std::memcmp(m_data.data(), "DXBC", 4) != 0)
    {
        LOG_ERROR(L"not a valid DXIL container: {}", path.wstring());
        return std::unexpected(Core::MakeError(
            E_FAIL, L"invalid shader container"));
    }

    LOG_INFO(L"shader loaded: {} ({} bytes)",
             path.filename().wstring(), m_data.size());
    return {};
}
```

**先頭 4 バイトの検査**を入れました。DXIL は、旧来の DXBC コンテナ形式をそのまま流用しているため、マジックナンバーは `DXBC` です。

これがあると、**ファイルの取り違えやコンパイル失敗を、その場で検出できます。** 空ファイルや HLSL ソースを誤って読み込んだ場合、第14章の PSO 生成まで気づかないところが、ここで止まります。

`D3D12_SHADER_BYTECODE` はポインタと長さの組にすぎません。**`ShaderBlob` が生きている間だけ有効**なので、第14章で PSO を作るまで保持しておく必要があります。

---

## ✅ 本章のゴール:`.cso` と `.pdb` が出力され、読み込める

### Step 1:ビルドする

`Ctrl + Shift + B`

**出力ウィンドウに DXC の実行が見えること。**

```
1>Compiling Triangle.hlsl
```

### Step 2:出力ファイルを確認する

`build/x64/Debug/shaders/` を開きます。

- [ ] `Triangle.VS.cso` がある
- [ ] `Triangle.PS.cso` がある
- [ ] `pdb/` の中に、**ハッシュらしい長い 16 進数の名前**の `.pdb` が 2 つある

**PDB の名前が `Triangle.VS.pdb` になっていたら失敗です。** `-Fd` の末尾のバックスラッシュを確認してください(13.6.2 節)。

### Step 3:シェーダーのハッシュを確認する(任意)

DXC でシェーダーの中身を覗けます。

```
external\dxc\bin\x64\dxc.exe -dumpbin build\x64\Debug\shaders\Triangle.VS.cso
```

コンテナに含まれるパートの一覧と、記録されているデバッグ情報のファイル名が確認できます。**PDB のファイル名がシェーダー内部に記録されている**という 13.6.2 節の説明を、目で確かめられます。

### Step 4:C++ から読み込む

```cpp
Graphics::ShaderBlob vs;
Graphics::ShaderBlob ps;

const auto dir = Graphics::ShaderDirectory();

if (auto r = vs.LoadFromFile(dir / L"Triangle.VS.cso"); !r)
{
    Core::ReportError(r.error());
    return 1;
}
if (auto r = ps.LoadFromFile(dir / L"Triangle.PS.cso"); !r)
{
    Core::ReportError(r.error());
    return 1;
}
```

**期待される出力**

```
[Info ] ShaderBlob.cpp(48): shader loaded: Triangle.VS.cso (3216 bytes)
[Info ] ShaderBlob.cpp(48): shader loaded: Triangle.PS.cso (2984 bytes)
```

サイズは環境によって変わります。**数キロバイト程度なら正常です。** 数十キロバイトあるようなら、`-Qembed_debug` が付いているか `-Qstrip_reflect` が抜けています。

### Step 5:わざとエラーを出す

`Triangle.hlsl` の 1 箇所をわざと壊します。

```hlsl
output.positon = float4(input.position, 1.0f);   // ❌ position のタイプミス
```

ビルドすると失敗します。

```
1>C:\dev\D3D12Book\shaders\Triangle.hlsl(28,12): error: use of undeclared identifier 'positon'; did you mean 'position'?
```

**エラー一覧の行をダブルクリックして、該当行にジャンプできることを確認してください。**

**確認したら元に戻してください。**

### Step 6:署名の警告が出ていないことを確認する

出力ウィンドウを検索して、次の警告が**出ていない**ことを確認します。

```
warning: DXIL signing library (dxil.dll) not found.
```

出ている場合、`external/dxc/bin/x64/` に `dxil.dll` を配置してください(13.2.3 節)。

---

### 本章の達成状態

- [ ] `external/dxc/` に DXC を配置した(`dxil.dll` を含む)
- [ ] `shaders/Triangle.hlsl` を作成した
- [ ] カスタムビルドツールで DXC を呼んでいる
- [ ] 「出力ファイル」を正しく設定し、増分ビルドが効いている
- [ ] `-Zi` と `-Fd <ディレクトリ>\` を指定している
- [ ] **PDB がハッシュ名で出力されている**
- [ ] `-WX` を付けている
- [ ] `-HV 2021` を明示している
- [ ] `.cso` を読み込み、`DXBC` マジックを検査している
- [ ] 署名の警告が出ていない
- [ ] Step 5 でエラー一覧からジャンプできることを確認した

---

## トラブルシューティング

| 症状 | 原因 | 対処 |
|---|---|---|
| `dxc.exe` が見つからない | `$(DxcExe)` のパス誤り | 13.7.1 |
| PDB がハッシュ名にならない | `-Fd` の末尾に `\` がない | 13.6.2 |
| `dxil.dll not found` の警告 | 配置漏れ | 13.2.4 |
| 開発者モードでのみ動く | 未署名の DXIL | 同上(13.2.3) |
| `.cso` が巨大 | `-Qembed_debug` が付いている | 13.6.1 |
| シェーダーを直しても反映されない | 「出力ファイル」の設定誤り | 13.7.2 |
| 毎回フルコンパイルされる | 「出力ファイル」が空 | 同上 |
| `invalid profile vs_6_9` | DXC が古い | 新しい DXC を入れる(13.3.1) |
| `entry point not found` | `-E` の名前が違う | 関数名と一致させる |
| `Semantic must be float4` | `SV_Position` が `float3` | 13.1.3 |
| 読み込みで `invalid shader container` | ファイルの取り違え、または 0 バイト | 13.9 |
| 日本語コメントで文字化け | ソースの文字コード | UTF-8(BOM なし)で保存 |

---

## まとめ

**1. 本書のシェーダーは DXC で `vs_6_6` / `ps_6_6` にコンパイルする。**
FXC は Shader Model 6.0 以降に対応していません。第37章だけ `6_9` に上げます。

**2. DXIL には署名が必要。**
`dxil.dll` が `dxc.exe` の隣になければ未署名になり、開発者モード以外では動きません。警告を見逃さないでください。

**3. シェーダーが動くには 3 つのバージョンが揃う必要がある。**
DXC、Agility SDK、ドライバ。**どこでエラーが出たかで、どれが古いかが分かります。**

**4. `-Zi -Fd <ディレクトリ>\` で PDB を分離出力する。**
末尾のバックスラッシュを付けると、シェーダーのハッシュがファイル名になります。ツールはシェーダーバイナリからハッシュを読んで PDB を探すので、**対応表を自分で管理する必要がありません。**

**5. この設定は、後から追加しても手遅れになる。**
デバッグ情報なしのビルドで取得したクラッシュダンプは、二度と解読できません。再ビルドしたバイナリは別物だからです。**まだ三角形も出ていないうちに設定する理由は、これに尽きます。**

**6. `-WX` で警告を溜めない。**
「暗黙の切り捨て」のような警告は、動くだけに放置されがちです。最初から禁止しておけば溜まりようがありません。

次章では、ルートシグネチャとパイプラインステートオブジェクト(PSO)を作ります。**本書で最大の構造体、`D3D12_GRAPHICS_PIPELINE_STATE_DESC` と対面する章です。** フィールドは 20 個以上あり、`d3dx12.h` のヘルパーは使えません。第9章 9.3.3 節で決めた `{}` の習慣が、いよいよ本領を発揮します。

---

## 参考リンク

| 内容 | URL |
|---|---|
| DirectX Shader Compiler | https://github.com/microsoft/DirectXShaderCompiler |
| `dxc.exe` と `dxcompiler.dll` の使い方 | https://github.com/microsoft/DirectXShaderCompiler/wiki/Using-dxc.exe-and-dxcompiler.dll |
| NuGet `Microsoft.Direct3D.DXC` | https://www.nuget.org/packages/Microsoft.Direct3D.DXC |
| PIX の自動 PDB 解決 | https://devblogs.microsoft.com/pix/using-automatic-shader-pdb-resolution-in-pix/ |
| Aftermath:シェーダーのコンパイル | https://docs.nvidia.com/nsight-aftermath/SDK/topics/shader_compilation.html |
| HLSL のセマンティクス | https://learn.microsoft.com/ja-jp/windows/win32/direct3dhlsl/dx-graphics-hlsl-semantics |
| Shader Model 6.x | https://learn.microsoft.com/ja-jp/windows/win32/direct3dhlsl/hlsl-shader-model-6-0-features-for-direct3d-12 |
