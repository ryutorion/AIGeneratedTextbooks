# 第29章 Nsight Graphics でフレームを解剖する

第4部に入ります。

第3部を終えた時点で、コードは数千行になりました。レンダリングパスは 6 つ、リソースの状態遷移は複雑に絡み合っています。**そして、ここから先の章はさらに難しくなります。**

第32章のコンピュートシェーダー、第36章のメッシュシェーダー、第37章のレイトレーシング。**これらは「書いたつもりが動かない」ことが日常的に起こる領域です。** そのとき、デバッグレイヤーのメッセージだけでは足りません。

**「絵が出ない」の原因は、GPU の中にあります。**

- 渡したはずのテクスチャが、実は空だった
- シェーダーが期待と違う値を読んでいた
- ドローコールが実は 0 回だった
- 深度テストで全部落ちていた

**これらは、フレームを解剖しなければ分かりません。**

本章では **Nsight Graphics** の使い方を扱います。第2章 2.4.1 節でインストールしたまま、ここまで一度も使いませんでした。**ようやく出番です。**

**本章のゴール**
自分のレンダラのフレームをキャプチャし、ドローコールを追い、リソースの中身を目で見る。GPU Trace でボトルネックを特定できるようになる。

---

## 29.1 キャプチャの前に

### 29.1.1 準備が効いてくる

**本章で最も価値のある準備は、既に済んでいます。**

| 準備 | 出典 | 何が変わるか |
|---|---|---|
| **すべてのオブジェクトに名前を付けた** | 第6章 6.5 節 | リソース一覧が読める |
| **シェーダーの PDB を出力した** | 第13章 13.6 節 | シェーダーのソースが見える |
| **Agility SDK を導入した** | 第4章 | 最新機能が解析できる |

**名前がなければ、リソース一覧はこうなります。**

```
ID3D12Resource (0x000001F2A4B3C120)
ID3D12Resource (0x000001F2A4B3D340)
ID3D12Resource (0x000001F2A4B3E560)
```

**名前があれば、こうです。**

```
SceneColorMS
ShadowMap
BloomTexture[0]
```

**第6章 6.5.2 節で「第31章の Aftermath で効いてくる」と書きました。** Nsight Graphics でも、まったく同じ理由で効きます。

### 29.1.2 デバッグマーカーを入れる

**もう一段、読みやすくできます。**

**PIX イベントマーカーは、Nsight Graphics でも認識されます。** ドローコールの一覧が、パスごとにグループ化されて表示されるようになります。

```cpp
// src/Graphics/DebugMarker.h
#pragma once
#include "std_import.h"

//---------------------------------------------------------------
// GPU イベントマーカー。
// Nsight Graphics と PIX の両方で認識される。
//
// D3D12_ENABLE_DEBUG_MARKERS を 0 にすると消える。
//---------------------------------------------------------------
#if !defined(D3D12_ENABLE_DEBUG_MARKERS)
    #define D3D12_ENABLE_DEBUG_MARKERS 1
#endif

namespace Graphics
{
#if D3D12_ENABLE_DEBUG_MARKERS

    //--- PIX の書式。第3引数は UTF-16 文字列 ---
    inline constexpr UINT kPixEventUnicode = 0;

    inline void BeginEvent(ID3D12GraphicsCommandList* commandList,
                           std::wstring_view name)
    {
        const std::wstring text(name);
        commandList->BeginEvent(
            kPixEventUnicode,
            text.c_str(),
            static_cast<UINT>((text.size() + 1) * sizeof(wchar_t)));
    }

    inline void EndEvent(ID3D12GraphicsCommandList* commandList)
    {
        commandList->EndEvent();
    }

    //-----------------------------------------------------------
    // RAII でスコープを表現する。
    //-----------------------------------------------------------
    class ScopedEvent
    {
    public:
        ScopedEvent(ID3D12GraphicsCommandList* commandList,
                    std::wstring_view name)
            : m_commandList(commandList)
        {
            BeginEvent(m_commandList, name);
        }

        ~ScopedEvent()
        {
            EndEvent(m_commandList);
        }

        ScopedEvent(const ScopedEvent&)            = delete;
        ScopedEvent& operator=(const ScopedEvent&) = delete;

    private:
        ID3D12GraphicsCommandList* m_commandList;
    };

#else

    inline void BeginEvent(ID3D12GraphicsCommandList*, std::wstring_view) {}
    inline void EndEvent(ID3D12GraphicsCommandList*) {}

    class ScopedEvent
    {
    public:
        ScopedEvent(ID3D12GraphicsCommandList*, std::wstring_view) {}
    };

#endif
}

//---------------------------------------------------------------
// 使い方:
//   GPU_EVENT(commandList, L"Shadow Pass");
//---------------------------------------------------------------
#define GPU_EVENT_CONCAT_IMPL(a, b) a##b
#define GPU_EVENT_CONCAT(a, b)      GPU_EVENT_CONCAT_IMPL(a, b)

#define GPU_EVENT(cmdList, name)                                    \
    ::Graphics::ScopedEvent GPU_EVENT_CONCAT(scopedEvent_, __LINE__) \
        { (cmdList), (name) }
```

**`__LINE__` で変数名を作っている**のは、同じスコープ内で複数のマーカーを使えるようにするためです。第6章 6.3.4 節の `HR_WIDEN` と同じく、2 段構えのマクロが必要になります。

**レンダリングパスに適用します。**

```cpp
Core::Status Renderer::RenderFrame(const Camera& camera)
{
    // ... フレームリソースの準備 ...

    {
        GPU_EVENT(m_commandList.Get(), L"Shadow Pass");
        RenderShadowPass(m_shadowCasters);
    }

    {
        GPU_EVENT(m_commandList.Get(), L"Scene");

        {
            GPU_EVENT(m_commandList.Get(), L"Opaque");
            DrawOpaque(camera);
        }
        {
            GPU_EVENT(m_commandList.Get(), L"Transparent");
            DrawTransparent(camera);
        }
    }

    {
        GPU_EVENT(m_commandList.Get(), L"Resolve MSAA");
        ResolveMsaa();
    }

    {
        GPU_EVENT(m_commandList.Get(), L"Bloom");
        RenderBloom();
    }

    {
        GPU_EVENT(m_commandList.Get(), L"Composite");
        Composite();
    }

    // ...
}
```

**入れ子にできます。** Nsight Graphics では、この階層がそのままツリーとして表示されます。

**これがない場合、数百のドローコールが平坦に並びます。** どこからどこまでがシャドウパスなのか、判別する手段がありません。

> **リリースビルドでも残すか**
>
> 第6章 6.5.4 節で、デバッグ名について同じ議論をしました。
>
> **マーカーのコストは、名前より小さくありません。** 文字列を毎フレーム渡すので、数百個あれば無視できなくなります。
>
> **本書は既定で有効にしますが、切り替えられるようにしておきます。** 第38章で性能を測る際、マーカーを外した状態でも計測してください。

### 29.1.3 Aftermath との排他に注意

**第2章 2.4.4 節で警告した内容を、ここで再確認します。**

> **Aftermath は、グラフィックスデバッガがアタッチされている間は動作しません。**

**Nsight Graphics でキャプチャしている間、Aftermath のクラッシュダンプは生成されません。**

```
デバッグの流れ:

① 絵がおかしい      → Nsight Graphics で解剖する(本章)
② GPU がクラッシュ  → 素の状態で実行し、Aftermath のダンプを取る(第31章)
                       → 生成されたダンプを Nsight Graphics で開く
```

**「Nsight でキャプチャしながらクラッシュ原因を調べる」ことはできません。** 用途を分けてください。

---

## 29.2 Frame Debugger

### 29.2.1 キャプチャする

**手順です。**

1. Nsight Graphics を起動する
2. `Connect to process` を選択
3. **Activity で `Frame Debugger` を選ぶ**
4. `Application Executable` に自分の exe を指定
5. `Working Directory` を出力ディレクトリに設定
6. `Launch` を押す

**アプリが起動したら、`F11` でキャプチャします**(既定のホットキー)。

**うまくいかない場合の確認点です。**

| 症状 | 原因 |
|---|---|
| 起動しない | 作業ディレクトリが違う(DLL が見つからない) |
| キャプチャできない | 別のデバッガがアタッチされている |
| 何も表示されない | `Present` が呼ばれていない |
| **Agility SDK が効かない** | **作業ディレクトリの設定**(第4章 4.4 節) |

**とくに最後が要注意です。** `D3D12SDKPath` は exe からの相対パスなので(第4章 4.3.1 節)、**作業ディレクトリが違うと OS 標準のランタイムが使われます。**

### 29.2.2 イベントリストを読む

**キャプチャすると、そのフレームのすべての API 呼び出しが一覧表示されます。**

```
▼ Frame 1247
  ▼ CommandList: MainCommandList
    ▼ Shadow Pass                          ← 29.1.2 のマーカー
        ClearDepthStencilView
        DrawIndexedInstanced (×48)
    ▼ Scene
      ▼ Opaque
          ClearRenderTargetView
          DrawIndexedInstanced (×312)
      ▼ Transparent
          DrawIndexedInstanced (×24)
    ▼ Resolve MSAA
        ResolveSubresource
    ▼ Bloom
        DrawInstanced (×7)
    ▼ Composite
        DrawInstanced
```

**マーカーのおかげで構造が見えています。**

**確認すべきことがあります。**

| 項目 | 見るべき点 |
|---|---|
| **ドローコールの数** | 想定と合っているか |
| **パスの順序** | 意図した順で実行されているか |
| **バリアの位置** | 過剰でないか、不足していないか |

**「ドローコールが 0 だった」というバグは、実際によくあります。** カリングが効きすぎている、キューが空、条件分岐が間違っている —— **一覧を見れば一目で分かります。**

### 29.2.3 スクラブして途中経過を見る

**Frame Debugger の中核機能です。**

**イベントリストの任意の位置をクリックすると、その時点でのレンダーターゲットの状態が表示されます。**

```
ドローコール 1  →  三角形が 1 つ描かれた状態
ドローコール 2  →  2 つ描かれた状態
...
ドローコール 50 →  50 個描かれた状態
```

**絵が完成していく過程を、1 ステップずつ追えます。**

**これが強力なのは、「どこで壊れたか」が分かるからです。**

```
シャドウパスの後  → シャドウマップは正しい
不透明パスの後    → 影が落ちている
半透明パスの後    → ❌ 半透明が真っ黒になった  ← ここが原因
```

**第28章のトラブルシューティング表にある症状は、ほぼすべてこの方法で切り分けられます。**

### 29.2.4 リソースを目で見る

**`Resources` ビューで、すべてのリソースの中身を確認できます。**

**とくに有用な使い方を挙げます。**

#### 深度バッファを見る

**第19章で作った深度バッファが、正しく書かれているかを確認できます。**

**Reversed-Z を使っている場合(第19章 19.5 節)、普通と逆に見えます。**

| | 手前 | 奥 |
|---|---|---|
| 標準 | 黒 | 白 |
| **Reversed-Z** | **白** | **黒** |

**第19章 19.5.5 節のコラムで予告した通りです。** 見え方が逆でも正常です。

#### シャドウマップを見る

**第27章の Step 1 で、可視化パスを自作しました。** Nsight Graphics なら、その必要がありません。

**深度値の範囲が狭くて見づらい場合、表示範囲を調整できます。** ツール側で `Range` を設定してください。

#### テクスチャのミップレベルを見る

**第20章 20.6.5 節でミップマップを扱いました。** 各レベルが正しく生成されているかを、目で確認できます。

**「遠くでちらつく」という症状のとき、ミップが正しく作られているかをまず疑います。**

#### HDR の値を確認する

**第26章 26.1.2 節で、シーンカラーを HDR フォーマットにしました。**

**Nsight Graphics では、1.0 を超える値を確認できます。** ピクセルにマウスを合わせると、実際の数値が表示されます。

**「ブルームが出ない」とき、本当に 1.0 を超えているかを確認できます。**

### 29.2.5 パイプラインステートを確認する

**`Pipeline` ビューで、そのドローコール時点の全設定が見られます。**

| 確認できるもの | 対応する章 |
|---|---|
| 入力レイアウト | 第14章 14.4.5 節 |
| ルートシグネチャの内容 | 第14章 14.2 節 |
| ラスタライザステート | 第14章 14.4.4 節 |
| ブレンドステート | 第28章 28.1 節 |
| 深度ステンシルステート | 第19章 19.4 節 |
| バインドされたリソース | 第18章、第20章 |

**第15章のトラブルシューティング表を思い出してください。**

> **三角形が出ないときのチェック順**
> 1. 巻き順
> 2. シザー矩形
> 3. ビューポート
> 4. `SampleMask`
> 5. 書き込みマスク

**この 5 つを、すべてここで確認できます。**

**第14章 14.4.3 節の「罠 1:`SampleMask = 0`」は、この画面で `0x00000000` と表示されます。** 一目で分かります。

### 29.2.6 シェーダーのソースを見る

**第13章 13.6 節で PDB を出力した成果が、ここで現れます。**

**シェーダーを選択すると、HLSL のソースコードが表示されます。**

**PDB がなければ、DXIL の逆アセンブルしか見られません。**

```
; 逆アセンブル(PDB なし)
%3 = call float @dx.op.loadInput.f32(i32 4, i32 0, i32 0, i8 0, i32 undef)
%4 = call float @dx.op.loadInput.f32(i32 4, i32 0, i32 0, i8 1, i32 undef)
...
```

```hlsl
// HLSL ソース(PDB あり)
float4 PSMain(VSOutput input) : SV_Target
{
    const float3 N = normalize(input.normalWS);
    ...
}
```

**第13章 13.6.3 節で「後から設定しても手遅れ」と書いた理由が、ここでも当てはまります。**

**PDB が見つからない場合、Nsight Graphics に検索パスを教えます。**

```
Tools → Options → Shader Debug Info → Search Paths
  → build/x64/Debug/shaders/pdb/ を追加
```

**第13章 13.6.2 節で `-Fd` にディレクトリを渡した**ので、ハッシュ名の PDB がここに並んでいます。**ツールは自動で対応するファイルを見つけます。**

---

## 29.3 GPU Trace

### 29.3.1 Frame Debugger との違い

| | Frame Debugger | **GPU Trace** |
|---|---|---|
| 見るもの | **何が起きたか** | **どれくらい時間がかかったか** |
| 用途 | 絵が正しいかの検証 | **性能の解析** |
| 粒度 | ドローコール単位 | **ハードウェアユニット単位** |

**「絵は正しいが遅い」ときに使うのが GPU Trace です。**

### 29.3.2 何が見えるか

**GPU の各ユニットの稼働率が、時系列で表示されます。**

| 指標 | 意味 |
|---|---|
| **SM Throughput** | 演算ユニットの稼働率 |
| **SM Occupancy** | 同時に抱えている warp の割合(第2章 2.3.1 節) |
| **VRAM Bandwidth** | メモリ帯域の使用率 |
| **L2 Hit Rate** | キャッシュの効き具合 |
| **Texture Throughput** | テクスチャユニットの稼働率 |
| **ROP Throughput** | 出力ユニット(ブレンド、深度)の稼働率 |

**第2章 2.3.1 節で説明した SM、warp、occupancy が、ここで実測値として現れます。**

### 29.3.3 ボトルネックを判断する

**どのユニットが 100% に近いかで、何が律速しているかが分かります。**

| 高い指標 | ボトルネック | 対策の方向 |
|---|---|---|
| SM Throughput | **演算** | シェーダーを軽くする |
| VRAM Bandwidth | **メモリ帯域** | テクスチャ圧縮、解像度を下げる |
| Texture Throughput | **テクスチャ読み取り** | サンプル数を減らす、ミップを使う |
| ROP Throughput | **出力** | オーバードローを減らす |
| **すべて低い** | **依存関係で待っている** | バリアの見直し、並列化 |

**最後の行が重要です。**

**すべてのユニットが遊んでいるのに遅い**場合、GPU は「待って」います。原因は次のいずれかです。

- **バリアが過剰**(第30章)
- CPU がコマンドを供給できていない(第35章)
- 前のパスの完了を待っている

**「何もしていない時間」が見えるのが、GPU Trace の価値です。**

### 29.3.4 本書のパイプラインを測る

**第28章までに作ったパイプラインを測ると、おおよそこうなります。**

```
Shadow Pass    ████░░░░░░  ROP 中心(深度書き込みのみ)
Opaque         ████████░░  SM とテクスチャのバランス
Transparent    ███░░░░░░░  ROP 中心(ブレンド)
Resolve MSAA   ██████░░░░  帯域中心
Bloom          █████░░░░░  テクスチャ中心(サンプリングが多い)
Composite      ██░░░░░░░░  軽い
```

**シャドウパスが ROP 中心なのは、第27章 27.6.1 節でピクセルシェーダーを省略したからです。** 深度を書くだけの処理になっています。

**ブルームがテクスチャ中心なのは、ガウスぼかしが大量のサンプリングを行うためです**(第26章 26.4 節)。**バイリニア補間による半減が効いているかどうかも、ここで測れます。**

### 29.3.5 Occupancy を見る

**第2章 2.3.1 節で、occupancy について書きました。**

> occupancy とは、SM が同時に抱えられる warp の最大数に対して、実際に何 warp を抱えられているかの比率です。

**GPU Trace では、これが実測値として表示されます。**

**低い場合の原因は 3 つです**(第2章 2.3.1 節)。

| 要因 | 確認方法 |
|---|---|
| レジスタ使用量が多い | Shader Profiler で確認 |
| 共有メモリの使いすぎ | `groupshared` のサイズ(第32章) |
| スレッドグループサイズ | `[numthreads]` の値(第32章) |

**ただし、第2章でも書いた通り「高ければ良い」という単純な指標ではありません。**

**レジスタを潤沢に使って 1 スレッドあたりの効率を上げたほうが速い場合もあります。** occupancy が 50% でも、SM Throughput が 90% なら問題ありません。

**「occupancy が低く、かつ SM Throughput も低い」ときが、改善の余地がある状態です。**

---

## 29.4 Shader Profiler

### 29.4.1 命令レベルで見る

**シェーダーのどの部分で時間を使っているかを特定できます。**

```hlsl
float4 PSMain(VSOutput input) : SV_Target
{
    const float3 N = normalize(input.normalWS);        //  2%
    const float3 L = -normalize(lightDirectionWS.xyz); //  1%
    const float3 V = normalize(...);                   //  2%

    const float shadow = ComputeShadow(...);           // 45%  ← ここ
    const float4 tex = gDiffuseTexture.Sample(...);    // 30%
    // ...
}
```

**第27章の PCF が重いことが、数値で分かります。**

**3×3 のサンプリングは 9 回の `SampleCmpLevelZero` です**(第27章 27.5.3 節)。**カーネルを小さくするか、解像度を下げるかの判断材料になります。**

### 29.4.2 メモリアクセスの分析

**「どの命令でストールしているか」も分かります。**

| ストールの理由 | 意味 |
|---|---|
| **Texture** | テクスチャの読み取り待ち |
| **Memory** | バッファの読み取り待ち |
| **Instruction** | 命令フェッチ待ち |
| **Barrier** | `GroupMemoryBarrierWithGroupSync` 待ち(第32章) |

**テクスチャ待ちが多い場合、次の対策があります。**

- ミップマップを使う(キャッシュ効率が上がる)
- テクスチャを圧縮する(第20章 20.7 節)
- サンプリング回数を減らす

---

## 29.5 PIX との使い分け

**第2章 2.4.4 節で示した表を、実際に使った経験を踏まえて更新します。**

| 目的 | 適したツール | 理由 |
|---|---|---|
| ドローコールを追う | **どちらでも** | 機能はほぼ同等 |
| リソースを目で見る | **どちらでも** | 同上 |
| **SM 稼働率、帯域の解析** | **Nsight Graphics** | ハードウェア寄りの情報 |
| **Aftermath ダンプの解析** | **Nsight Graphics** | 唯一の選択肢 |
| **API の使用状況の検証** | **PIX** | D3D12 の仕様に忠実 |
| **DRED の確認** | **PIX** | 第38章 |
| 他ベンダでも同じ手順 | **PIX** | ベンダ非依存 |

**本書は NVIDIA を対象としているので、Nsight Graphics を主に使います。**

**ただし、PIX にしかない利点もあります。**

**PIX の `Timing Capture` は、CPU と GPU のタイムラインを同時に見られます。** 第35章でマルチスレッド化したとき、これが有用になります。

**また、PIX はドローコールごとに「このドローが最終画像にどう寄与したか」を可視化する機能を持ちます。** オーバードローの分析に便利です。

---

## 29.6 デバッグを効率化する

### 29.6.1 特定のフレームでキャプチャする

**「たまに起こるバグ」を捕まえたいことがあります。**

**プログラム側からキャプチャを要求できます。**

```cpp
#include <NsightGraphicsAPI.h>   // Nsight Graphics SDK

if (input.WasKeyPressed(VK_F11))
{
    NGFX_Injection_ExecuteActivityCommand();   // キャプチャを開始
}
```

**あるいは、条件を満たしたときに自動でキャプチャします。**

```cpp
if (m_frameCount == m_captureTargetFrame)
{
    TriggerCapture();
}
```

**「500 フレーム目で必ず壊れる」というバグに有効です。**

### 29.6.2 決定論的に再現する

**キャプチャしたフレームは、何度でも再実行できます。**

**しかし、それは「そのフレームの API 呼び出し」を再現するだけです。** アプリ側の状態(カメラ位置、時間)は含まれません。

**再現性を高めるには、アプリ側で工夫します。**

```cpp
//--- デバッグ用:時間を固定する ---
#if defined(_DEBUG)
if (m_freezeTime)
{
    deltaTime = 1.0f / 60.0f;   // 常に一定
}
#endif
```

```cpp
//--- カメラ位置を保存・復元する ---
void SaveCameraState(const Camera& camera, const std::filesystem::path& path);
void LoadCameraState(Camera& camera, const std::filesystem::path& path);
```

**「あの角度から見たときだけ壊れる」というバグを、何度でも再現できるようになります。**

**これは Nsight Graphics の機能ではなく、アプリ側の設計です。** しかし、デバッグの効率を大きく左右します。

### 29.6.3 デバッグ描画を用意する

**ツールで見るより、画面に出したほうが速い場合もあります。**

**第23章 23.4.3 節で法線の可視化を作りました。** 同じ発想で、いくつか用意しておくと便利です。

```cpp
enum class DebugView
{
    None,
    Normal,          // 第23章
    Depth,
    ShadowMap,       // 第27章
    Overdraw,
    Wireframe,       // 第16章
};
```

**オーバードローの可視化は、加算合成で実装できます**(第28章 28.1.2 節)。

```hlsl
float4 PSOverdraw(VSOutput input) : SV_Target
{
    return float4(0.1f, 0.0f, 0.0f, 1.0f);   // 加算合成で描く
}
```

**重なった回数だけ赤くなります。** 半透明が多すぎる場所が一目で分かります。

---

## ✅ 本章のゴール:自分のフレームを解析できる

### Step 1:マーカーを入れる

**29.1.2 節の `GPU_EVENT` を、すべてのパスに入れてください。**

**キャプチャして、階層が表示されることを確認します。**

```
▼ Shadow Pass
▼ Scene
  ▼ Opaque
  ▼ Transparent
▼ Resolve MSAA
▼ Bloom
▼ Composite
```

**入れる前と後で、イベントリストの読みやすさを比べてみてください。**

### Step 2:ドローコールの数を確認する

**第25章 25.3.3 節で統計を取るようにしました。** その値と、Nsight Graphics が示す数が一致するはずです。

```
アプリのログ:  draws 336
Nsight:        DrawIndexedInstanced × 336
```

**食い違っていたら、どちらかが間違っています。**

### Step 3:スクラブして絵の完成過程を見る

**シャドウパスの直後、不透明の直後、半透明の直後 —— それぞれの時点でレンダーターゲットを確認してください。**

**各パスが期待通りの結果を出しているかが分かります。**

### Step 4:リソースを確認する

**次の 4 つを見てください。**

| リソース | 確認点 |
|---|---|
| **ShadowMap** | 物体のシルエットが見えるか |
| **SceneColorMS** | MSAA が有効か(サンプル数の表示) |
| **BloomTexture[0]** | 明るい部分だけが残っているか |
| **深度バッファ** | Reversed-Z なら白黒が逆か |

### Step 5:シェーダーのソースを確認する

**PDB が正しく読み込まれ、HLSL のソースが表示されることを確認してください。**

**表示されない場合、29.2.6 節の検索パス設定を確認します。**

### Step 6:わざと壊してみる

**第15章のトラブルシューティング表から、1 つ選んで再現します。**

```cpp
desc.SampleMask = 0;   // ❌ 第14章 14.4.3 節の罠 1
```

**Pipeline ビューで `SampleMask: 0x00000000` と表示されることを確認してください。**

**第15章 Step 4 では「三角形が消える」としか分かりませんでした。** Nsight Graphics なら、**原因が直接見えます。**

**確認したら元に戻してください。**

### Step 7:GPU Trace を実行する

**Activity を `GPU Trace` に変更してキャプチャします。**

**各パスの所要時間と、ユニットの稼働率を確認してください。**

```
Shadow Pass     0.42 ms
Opaque          1.85 ms
Transparent     0.31 ms
Resolve MSAA    0.28 ms
Bloom           0.94 ms
Composite       0.12 ms
─────────────────────
Total           3.92 ms
```

**どのパスが重いかが分かります。**

### Step 8:ボトルネックを特定する

**最も重いパスについて、どのユニットが高いかを確認してください。**

**29.3.3 節の表と照らし合わせて、対策の方向を考えます。**

**実際に対策を打つのは第38章ですが、ここで「何が律速しているか」を把握しておきます。**

### Step 9:MSAA の影響を測る

**サンプル数を変えて、GPU Trace を比較してください。**

| 設定 | 予想される変化 |
|---|---|
| 1× → 4× | ROP と帯域が増える |
| 解像度を上げる | すべてが増える |

**第28章 Step 6 では「フレームレートを比べる」としか言えませんでした。** GPU Trace なら、**どこにコストが乗ったかが分かります。**

---

### 本章の達成状態

- [ ] `GPU_EVENT` マクロを実装した
- [ ] すべてのレンダリングパスにマーカーを入れた
- [ ] Nsight Graphics でキャプチャできた
- [ ] イベントリストが階層表示されている
- [ ] ドローコール数がアプリの統計と一致した
- [ ] スクラブして絵の完成過程を確認した
- [ ] シャドウマップの内容を確認した
- [ ] シェーダーの HLSL ソースが表示された
- [ ] Pipeline ビューで PSO の設定を確認した
- [ ] GPU Trace で各パスの時間を測った
- [ ] ボトルネックとなるユニットを特定した
- [ ] Aftermath との排他関係を理解した

---

## トラブルシューティング

| 症状 | 原因 | 対処 |
|---|---|---|
| アプリが起動しない | 作業ディレクトリが違う | DLL の場所を確認 |
| Agility SDK が効かない | 同上 | 第4章 4.4 節 |
| キャプチャできない | 別のデバッガがアタッチ中 | VS のデバッガを外す |
| リソース名が出ない | `SetName` を呼んでいない | 第6章 6.5 節 |
| イベントが平坦に並ぶ | マーカーがない | 29.1.2 |
| シェーダーのソースが出ない | PDB が見つからない | 29.2.6 |
| 同上 | `-Zi` を付けていない | 第13章 13.6.4 節 |
| 深度が逆に見える | Reversed-Z | 仕様(第19章 19.5.5) |
| Aftermath のダンプが出ない | デバッガがアタッチ中 | 29.1.3 |
| GPU Trace の値が不安定 | 他のアプリが GPU を使用中 | 閉じてから測る |
| 数値が毎回大きく変わる | クロックが変動している | ロックする(第38章) |

---

## まとめ

**1. 準備は既に終わっていた。**
デバッグ名(第6章)、シェーダー PDB(第13章)、Agility SDK(第4章)。**これらがなければ、ツールは半分の力しか出せません。**

**2. マーカーを入れると、フレームの構造が見える。**
数百のドローコールが平坦に並ぶのと、パスごとに階層化されているのとでは、読みやすさが桁違いです。

**3. スクラブが Frame Debugger の中核。**
絵が完成していく過程を追えば、「どこで壊れたか」が特定できます。

**4. Pipeline ビューは、第15章のチェックリストそのもの。**
巻き順、シザー矩形、`SampleMask`、書き込みマスク。**すべてがここで確認できます。**

**5. GPU Trace は「時間」を見る。**
Frame Debugger が「何が起きたか」なら、GPU Trace は「どれくらいかかったか」です。

**6. すべてのユニットが低いときは、待っている。**
バリアの過剰、CPU の供給不足、依存関係。**これが第30章と第35章の主題です。**

**7. Aftermath とは排他。**
デバッガがアタッチしている間、クラッシュダンプは生成されません。**用途を分けてください。**

次章では、リソースバリアを整理します。**第11章から手で書き続けてきた状態遷移を、Enhanced Barriers へ移行します。** 第26章 26.1.4 節で「状態追跡が複雑になってきた」と書いた問題に、正面から取り組みます。

---

## 参考リンク

| 内容 | URL |
|---|---|
| Nsight Graphics ドキュメント | https://docs.nvidia.com/nsight-graphics/ |
| Frame Debugger の使い方 | https://docs.nvidia.com/nsight-graphics/UserGuide/#frame_debugger |
| GPU Trace | https://docs.nvidia.com/nsight-graphics/UserGuide/#gpu_trace |
| PIX イベントマーカー | https://devblogs.microsoft.com/pix/winpixeventruntime/ |
| `ID3D12GraphicsCommandList::BeginEvent` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/nf-d3d12-id3d12graphicscommandlist-beginevent |
| Nsight Graphics SDK(自動キャプチャ) | https://developer.nvidia.com/nsight-graphics |
