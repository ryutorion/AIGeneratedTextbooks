# 第19章 深度バッファ

前章の立方体は、回転させると**面の前後関係が破綻**していました。奥の面が手前を隠し、回転の途中で唐突に描画順が入れ替わります。

原因は単純です。**インデックスの並び順が、そのまま描画順になっている**からです。後から描いたものが、前に描いたものを上書きします。

これを解決するのが深度バッファです。**ピクセルごとに「今そこに何が描かれているか、それはどれくらい奥か」を記録し、より手前のものだけを残します。**

本章で追加するコードは 100 行程度です。しかし、**第17章 17.6.4 節で予告した「深度の精度」という厄介な問題**が、ここで実体を持って現れます。`nearZ` の値ひとつで、遠くの面がちらつくかどうかが決まります。

**本章のゴール**
深度バッファを作り、深度テストを有効にする。立方体の前後関係が正しく描画される。あわせて、Reversed-Z による精度改善を理解する。

---

## 19.1 深度テストとは

### 19.1.1 描画順に依存しない可視判定

**深度バッファは、レンダーターゲットと同じ大きさの、もう 1 枚のテクスチャです。** 色の代わりに「奥行き」を保持します。

ピクセルが描かれるたびに、次のことが起こります。

```
① ピクセルシェーダーが、そのピクセルの色と深度を出す
        ↓
② 深度バッファの、その位置の値と比較する
        ↓
③-a 比較に通った → 色を書き、深度も更新する
③-b 通らなかった → 何もしない(捨てる)
```

**比較の方法は指定できます。** 既定は「より小さい(手前)なら通す」です。

```cpp
desc.DepthFunc = D3D12_COMPARISON_FUNC_LESS;
```

第17章 17.6.1 節で確認した通り、**D3D の深度は近クリップ面が 0、遠クリップ面が 1** です。したがって「小さいほど手前」になります。

**この仕組みのおかげで、描画順を気にしなくてよくなります。** 奥の面を先に描いても後に描いても、結果は同じです。

### 19.1.2 ソートでは解決しない

「奥から順に描けばいいのでは」と考えたくなります。**うまくいきません。**

```
    A          B
   ／＼      ／
  ／  ＼  ／
 ／     ×
        ＼
```

**互いに貫通している 2 つの面**は、どちらを先に描いても正しくなりません。三角形を分割しない限り、順序では表現できないのです。

さらに、**三角形の数だけソートするコスト**もかかります。数万ポリゴンのモデルを毎フレーム並べ替えるのは現実的ではありません。

**深度バッファは、ピクセル単位で判定するのでこの問題がありません。**

> **それでもソートが必要になる場合**
>
> **半透明**の描画では、深度テストだけでは足りません。奥のものが透けて見える必要があるので、「手前が描かれたら奥は捨てる」という判定が使えないからです。
>
> 第28章で半透明を扱うとき、**不透明は深度テスト、半透明は奥からのソート**という組み合わせにします。

### 19.1.3 Early-Z

**深度テストは、本来ピクセルシェーダーの後に行われます。** 「シェーダーが色を出す → 深度を比較する」という順序です。

しかし、**隠れると分かっているピクセルのシェーダーを実行するのは無駄**です。そこで GPU は、可能な場合に**深度テストをシェーダーの前に前倒し**します。これを Early-Z と呼びます。

```
通常:      ラスタライズ → ピクセルシェーダー → 深度テスト
Early-Z:   ラスタライズ → 深度テスト → ピクセルシェーダー
```

**重い背景の前に大きな物体があるようなシーンでは、劇的に効きます。**

ただし、**次の場合は前倒しできません。**

| 条件 | 理由 |
|---|---|
| ピクセルシェーダーが `SV_Depth` に書く | 深度が確定しないと比較できない |
| `discard` / `clip` を使う | ピクセルが残るか分からない |
| アルファトゥカバレッジを使う | 同上 |

**本書のシェーダーはどれにも当てはまらないので、Early-Z が効きます。**

> **`SV_Depth` を使いたい場合の逃げ道**
>
> どうしても深度を書き換えたい場合、**変化の向きが分かっていれば** Early-Z を保てます。
>
> ```hlsl
> float4 PSMain(VSOutput input, out float depth : SV_DepthLessEqual) : SV_Target
> ```
>
> 「深度は必ず元の値以下になる」と宣言することで、GPU は「元の値で落ちるなら、書き換えても落ちる」と判断できます。**第29章で Nsight を使うようになったら、Early-Z が効いているかを確認できます。**

---

## 19.2 深度バッファを作る

### 19.2.1 テクスチャの `D3D12_RESOURCE_DESC`

**第15章 15.2.3 節ではバッファ用の記述子を作りました。今度はテクスチャです。**

```cpp
D3D12_RESOURCE_DESC desc{};
desc.Dimension          = D3D12_RESOURCE_DIMENSION_TEXTURE2D;
desc.Alignment          = 0;
desc.Width              = width;
desc.Height             = height;
desc.DepthOrArraySize   = 1;
desc.MipLevels          = 1;
desc.Format             = DXGI_FORMAT_D32_FLOAT;
desc.SampleDesc.Count   = 1;
desc.SampleDesc.Quality = 0;
desc.Layout             = D3D12_TEXTURE_LAYOUT_UNKNOWN;     // ← バッファと違う
desc.Flags              = D3D12_RESOURCE_FLAG_ALLOW_DEPTH_STENCIL;
```

**バッファとの違いが 4 つあります。**

| フィールド | バッファ | テクスチャ |
|---|---|---|
| `Dimension` | `BUFFER` | `TEXTURE2D` |
| `Height` | 常に `1` | 実際の高さ |
| `Format` | 常に `UNKNOWN` | 実際のフォーマット |
| **`Layout`** | **`ROW_MAJOR`** | **`UNKNOWN`** |

**`Layout` の違いが重要です。**

バッファは「バイトが順に並んだもの」なので `ROW_MAJOR` でした。**テクスチャは違います。** GPU は 2 次元のアクセス効率を上げるため、ピクセルを独自の順序(スウィズル)で並べます。その並べ方は**ハードウェアごとに異なり、非公開**です。

`D3D12_TEXTURE_LAYOUT_UNKNOWN` は、**「並べ方はドライバに任せる」**という指定です。だからテクスチャは `Map` して直接書き込めません。**第20章でテクスチャを転送するとき、この事実が効いてきます。**

`D3D12_RESOURCE_FLAG_ALLOW_DEPTH_STENCIL` は**必須**です。付け忘れると DSV を作るときに失敗します。

**ヘルパーを追加します。**

```cpp
// src/Graphics/D3D12Helpers.h に追加

//---------------------------------------------------------------
// 2D テクスチャ用のリソース記述子。
// CD3DX12_RESOURCE_DESC::Tex2D() の代替。
//---------------------------------------------------------------
[[nodiscard]] inline D3D12_RESOURCE_DESC MakeTexture2DDesc(
    DXGI_FORMAT format,
    UINT64      width,
    UINT        height,
    UINT16      mipLevels   = 1,
    UINT16      arraySize   = 1,
    D3D12_RESOURCE_FLAGS flags = D3D12_RESOURCE_FLAG_NONE) noexcept
{
    D3D12_RESOURCE_DESC desc{};
    desc.Dimension          = D3D12_RESOURCE_DIMENSION_TEXTURE2D;
    desc.Alignment          = 0;
    desc.Width              = width;
    desc.Height             = height;
    desc.DepthOrArraySize   = arraySize;
    desc.MipLevels          = mipLevels;
    desc.Format             = format;
    desc.SampleDesc.Count   = 1;
    desc.SampleDesc.Quality = 0;
    desc.Layout             = D3D12_TEXTURE_LAYOUT_UNKNOWN;
    desc.Flags              = flags;
    return desc;
}
```

**自作ヘルパーが 11 個になりました。**

### 19.2.2 深度フォーマットの選択

| フォーマット | 深度 | ステンシル | 1 ピクセルあたり |
|---|---|---|---|
| `D16_UNORM` | 16bit 固定小数 | なし | 2 バイト |
| `D24_UNORM_S8_UINT` | 24bit 固定小数 | 8bit | 4 バイト |
| **`D32_FLOAT`** | **32bit 浮動小数** | なし | **4 バイト** |
| `D32_FLOAT_S8X24_UINT` | 32bit 浮動小数 | 8bit | 8 バイト |

**本書は `D32_FLOAT` を使います。**

理由は明快です。**`D24_UNORM_S8_UINT` と同じ 4 バイトでありながら、精度が高いから**です。24bit の固定小数より、32bit の浮動小数のほうが表現力があります。

ステンシルは、本書では使いません。必要になるのは、鏡面反射やアウトライン描画といった応用です。

> **`D32_FLOAT` は Reversed-Z の前提でもある**
>
> 19.5 節で扱う Reversed-Z は、**深度が浮動小数であることを前提にした手法**です。`D24_UNORM` では効果がありません。
>
> このためにも `D32_FLOAT` を選んでおく価値があります。

### 19.2.3 最適化クリア値 —— 第7章の伏線

```cpp
D3D12_CLEAR_VALUE clearValue{};
clearValue.Format               = DXGI_FORMAT_D32_FLOAT;
clearValue.DepthStencil.Depth   = 1.0f;
clearValue.DepthStencil.Stencil = 0;

HR_TRY(device->CreateCommittedResource(
    &heapProps,
    D3D12_HEAP_FLAG_NONE,
    &desc,
    D3D12_RESOURCE_STATE_DEPTH_WRITE,   // ← 初期状態
    &clearValue,                        // ← 今回は nullptr ではない
    IID_PPV_ARGS(&m_depthBuffer)));
```

**第15章と第16章では、この引数に `nullptr` を渡していました。** バッファにはクリアの概念がないからです。

**レンダーターゲットと深度バッファでは、指定すべきです。**

GPU は、「このバッファは 1.0 でクリアされる」と分かっていれば、**実際にメモリを 1.0 で埋める代わりに「クリア済み」という印を立てるだけ**で済ませられます(高速クリア)。

**そして、指定した値と違う値でクリアすると、警告が出ます。**

```
D3D12 WARNING: ID3D12GraphicsCommandList::ClearDepthStencilView:
  The application did not pass any clear value to resource creation.
  The clear operation is typically slower as a result...
  [ EXECUTION WARNING #820: CLEARDEPTHSTENCILVIEW_MISMATCHINGCLEARVALUE]
```

**この警告 ID に見覚えがあるはずです。** 第7章 7.6.3 節で、「メッセージを抑制する例」として挙げたものです。

そこでこう書きました。

> **本書のルール:抑制する場合は、必ずコメントで理由を書く。**
> 理由を書けないなら、それは抑制すべきではないメッセージです。

**まさにこれが該当します。** この警告は「性能が落ちている」という有益な指摘なので、**抑制するのではなく、値を一致させるのが正解です。**

19.4 節でクリアするときも、必ず `1.0f` を使います。

### 19.2.4 深度バッファは 1 枚でよい

**素朴な疑問があります。**

第12章でバックバッファを 3 枚にしました。**深度バッファも 3 枚要るのでしょうか。**

**要りません。1 枚で十分です。**

理由は、**バックバッファと深度バッファでは、解放されるタイミングが違う**からです。

| | いつ解放されるか |
|---|---|
| **バックバッファ** | **表示が終わるまで OS が握っている** |
| **深度バッファ** | そのフレームのコマンドが終われば自由 |

バックバッファは `Present` した後も、ディスプレイに表示されている間は使用中です。**その間、我々のコマンドキューの外で押さえられています。** だから複数枚必要でした。

深度バッファは画面に出ません。**そして、同じコマンドキューに投入されたコマンドは、投入順に実行されます**(第9章 9.5.2 節)。フレーム N のコマンドが完了してから、フレーム N+1 のコマンドが始まります。**2 つのフレームが同時に深度バッファへ書くことはありえません。**

**だから 1 枚で足ります。** 第12章 12.2.1 節の判断基準「GPU がまだ読んでいる可能性があるものはフレームごとに持つ」を、正しく適用した結果です。

---

## 19.3 DSV ヒープとビュー

### 19.3.1 DSV ヒープ

```cpp
D3D12_DESCRIPTOR_HEAP_DESC heapDesc{};
heapDesc.Type           = D3D12_DESCRIPTOR_HEAP_TYPE_DSV;
heapDesc.NumDescriptors = 1;                              // 19.2.4 の通り
heapDesc.Flags          = D3D12_DESCRIPTOR_HEAP_FLAG_NONE; // 可視にできない
heapDesc.NodeMask       = 0;

HR_TRY(device->CreateDescriptorHeap(&heapDesc, IID_PPV_ARGS(&m_dsvHeap)));
Core::SetDebugName(m_dsvHeap.Get(), L"DsvHeap");
```

**第11章 11.2.3 節で述べた通り、DSV ヒープはシェーダー可視にできません。** `SHADER_VISIBLE` を指定すると生成に失敗します。

### 19.3.2 DSV を作る

```cpp
D3D12_DEPTH_STENCIL_VIEW_DESC dsvDesc{};
dsvDesc.Format             = DXGI_FORMAT_D32_FLOAT;
dsvDesc.ViewDimension      = D3D12_DSV_DIMENSION_TEXTURE2D;
dsvDesc.Flags              = D3D12_DSV_FLAG_NONE;
dsvDesc.Texture2D.MipSlice = 0;

device->CreateDepthStencilView(
    m_depthBuffer.Get(),
    &dsvDesc,
    m_dsvHeap->GetCPUDescriptorHandleForHeapStart());
```

**第 2 引数に `nullptr` を渡してもかまいません。** その場合、リソース自身のフォーマットが使われます(第11章 11.4 節の RTV と同じです)。

**`D3D12_DSV_FLAG_NONE` 以外の値**には、次があります。

| フラグ | 意味 |
|---|---|
| `READ_ONLY_DEPTH` | 深度を読むだけで書かない |
| `READ_ONLY_STENCIL` | ステンシルを読むだけで書かない |

読み取り専用にすると、**同じ深度バッファを DSV と SRV で同時に使えます。** 深度を参照しながら描画する場面(第26章のポストエフェクトなど)で使います。

> **深度バッファをシェーダーから読みたい場合**
>
> 第27章のシャドウマップでは、深度バッファを**テクスチャとして読み取ります。** そのとき、少し工夫が必要になります。
>
> `D32_FLOAT` は「深度専用」のフォーマットで、SRV としては使えません。**型を持たない `R32_TYPELESS` でリソースを作り、DSV は `D32_FLOAT`、SRV は `R32_FLOAT` として別々のビューを作ります。**
>
> ```
> リソース:  DXGI_FORMAT_R32_TYPELESS
>    ├─ DSV: DXGI_FORMAT_D32_FLOAT   (深度として書く)
>    └─ SRV: DXGI_FORMAT_R32_FLOAT   (テクスチャとして読む)
> ```
>
> **第11章 11.2.1 節で「デスクリプタはリソースではなく解釈の指示書」と書いた**ことが、ここで具体的な意味を持ちます。同じメモリを、2 通りに解釈しています。
>
> 本章では読まないので、素直に `D32_FLOAT` で作ります。**第27章で 2 行変更します。**

### 19.3.3 リサイズで作り直す

**第12章 12.4.5 節に残したコメントを、ここで回収します。**

```cpp
Core::Status Renderer::Resize(UINT width, UINT height)
{
    // ... スワップチェーンのリサイズ ...

    m_width  = width;
    m_height = height;

    //--- ★ 深度バッファを作り直す ★ ---
    if (auto r = CreateDepthBuffer(width, height); !r)
    {
        return r;
    }

    //--- ビューポートとシザー矩形(第15章 15.4.3 節)---
    // ...

    // 第26章でオフスクリーンターゲットを追加したら、ここに加える

    return {};
}
```

**深度バッファは、バックバッファと同じ大きさでなければなりません。** 食い違うと、`OMSetRenderTargets` の時点でデバッグレイヤーがエラーを出します。

`CreateDepthBuffer` の中では、**古いリソースを `ComPtr` に代入し直すだけ**で解放されます。第12章 12.4.1 節で `WaitForGpuIdle` を済ませているので、安全です。

---

## 19.4 深度ステートの設定とクリア

### 19.4.1 PSO を直す

**第14章 14.4.6 節で書いた「暫定」の行を削除します。**

```cpp
//--- 深度バッファはまだない(第19章で有効にする) ---
desc.DepthStencilState.DepthEnable = FALSE;      // ❌ この行を削除
desc.DSVFormat                     = DXGI_FORMAT_UNKNOWN;
```

```cpp
//--- 深度テストを有効にする ---
desc.DSVFormat = DXGI_FORMAT_D32_FLOAT;          // ✅
```

**`DepthStencilState` は、第14章で作った `DefaultDepthStencilDesc()` のままで正しく動きます。**

```cpp
desc.DepthEnable    = TRUE;
desc.DepthWriteMask = D3D12_DEPTH_WRITE_MASK_ALL;
desc.DepthFunc      = D3D12_COMPARISON_FUNC_LESS;
desc.StencilEnable  = FALSE;
```

**第14章で `d3dx12.h` の既定値に合わせておいた判断が、ここで報われます。** 変更するのは `DSVFormat` の 1 行だけです。

### 19.4.2 主な設定項目

#### `DepthFunc`

| 値 | 通す条件 |
|---|---|
| **`LESS`** | **新しい深度 < 既存(既定)** |
| `LESS_EQUAL` | 新しい深度 ≤ 既存 |
| `GREATER` | 新しい深度 > 既存(**Reversed-Z 用**) |
| `GREATER_EQUAL` | 新しい深度 ≥ 既存 |
| `EQUAL` | 一致 |
| `ALWAYS` | 常に通す(深度テスト無効と同じ) |
| `NEVER` | 常に落とす |

**`LESS_EQUAL` が有用な場面があります。** 同じジオメトリを 2 回描くとき(1 回目で深度だけ書き、2 回目で色を描く「Z プリパス」など)、`LESS` だと 2 回目が必ず落ちてしまいます。

#### `DepthWriteMask`

| 値 | 意味 |
|---|---|
| **`ALL`** | **テストに通ったら深度を書く(既定)** |
| `ZERO` | **テストはするが、深度は書かない** |

**`ZERO` は半透明描画で使います**(第28章)。半透明のものは「奥のものに隠されるか」は判定したいが、「後ろのものを隠す」ことはしたくないからです。

### 19.4.3 毎フレームクリアする

```cpp
const D3D12_CPU_DESCRIPTOR_HANDLE dsv =
    m_dsvHeap->GetCPUDescriptorHandleForHeapStart();

//--- レンダーターゲットと深度バッファを設定 ---
m_commandList->OMSetRenderTargets(1, &rtv, FALSE, &dsv);   // ← 第4引数

//--- 両方クリアする ---
m_commandList->ClearRenderTargetView(rtv, clearColor, 0, nullptr);
m_commandList->ClearDepthStencilView(
    dsv,
    D3D12_CLEAR_FLAG_DEPTH,   // ステンシルは使わない
    1.0f,                     // ← 19.2.3 の最適化クリア値と一致させる
    0,                        // ステンシル値
    0, nullptr);              // クリアする矩形(全体)
```

**`OMSetRenderTargets` の第 4 引数**が、これまで `nullptr` だったところです。

**クリアを忘れると、前フレームの深度が残ります。** 最初のフレームは正しく描かれ、2 フレーム目以降は**ほとんどのピクセルが深度テストに落ちて、絵が更新されなくなります。** 19.6 節の実験で確認します。

### 19.4.4 バリアは不要

**深度バッファには、毎フレームのバリアが要りません。**

生成時に `D3D12_RESOURCE_STATE_DEPTH_WRITE` にしてあり、**ずっとその状態のまま**だからです。書くだけで、読まないので、遷移する理由がありません。

第11章のバックバッファが `PRESENT ↔ RENDER_TARGET` を往復していたのと対照的です。**バックバッファは DXGI に渡すために状態を戻す必要がありましたが、深度バッファは誰にも渡しません。**

**第26章でポストエフェクトから深度を読むようになると、事情が変わります。**

```
DEPTH_WRITE → PIXEL_SHADER_RESOURCE → DEPTH_WRITE
```

という往復が必要になります。**そのとき、第11章で書いた `MakeTransitionBarrier` が再び登場します。**

---

## 19.5 深度精度と Reversed-Z

### 19.5.1 深度値は線形ではない

**第17章 17.6.4 節で予告した問題を、数値で見ます。**

射影行列を通した後の深度は、次の式になります(第17章 17.6.3 節の導出より)。

```
        f        n
d = ───────── (1 - ─)
     f - n         z
```

**`1/z` に比例します。** 線形ではありません。

`n = 0.1`、`f = 1000` として、いくつかの `z` で計算します。

| ビュー空間の z | 深度値 d |
|---|---|
| 0.1(近クリップ面) | 0.000000 |
| 0.2 | 0.500050 |
| 1 | 0.900090 |
| 10 | 0.990099 |
| 100 | 0.999001 |
| 1000(遠クリップ面) | 1.000000 |

**`z` が 0.1 から 0.2 へ動いただけで、深度の半分を使い切っています。**

そして `z = 100` から `z = 1000` までの 900 メートルは、`0.999001` から `1.000000` までの **0.001 の範囲**に押し込められています。

### 19.5.2 浮動小数の精度は 1.0 付近で最も粗い

**問題は、ここで浮動小数の性質と噛み合わないことです。**

`float` は、値が小さいほど細かく表現できます。

| 値の範囲 | 隣り合う値の間隔 |
|---|---|
| 1.0 付近 | 約 6.0 × 10⁻⁸ |
| 0.001 付近 | 約 6.0 × 10⁻¹¹ |
| 0.0000001 付近 | 約 6.0 × 10⁻¹⁵ |

**深度値は 1.0 付近に集中し、そこは浮動小数がもっとも粗い領域です。** 最悪の組み合わせです。

`z = 1000` 付近では、1 メートル動いても深度は約 10⁻⁷ しか変わりません。**浮動小数の刻み(6 × 10⁻⁸)と同程度です。** 区別できなくなり、面がちらつきます。これが **Z ファイティング**です。

### 19.5.3 Reversed-Z —— 2 つの非線形を打ち消し合わせる

**発想は単純です。深度の向きを逆にします。**

```
標準:      近 = 0.0    遠 = 1.0
Reversed:  近 = 1.0    遠 = 0.0
```

こうすると、**遠くのものが 0.0 付近に来ます。** そこは浮動小数がもっとも細かい領域です。

| ビュー空間の z | 標準の d | **Reversed の d** |
|---|---|---|
| 0.1(近) | 0.000000 | **1.000000** |
| 1 | 0.900090 | **0.099910** |
| 10 | 0.990099 | **0.009901** |
| 100 | 0.999001 | **0.000999** |
| 1000(遠) | 1.000000 | **0.000000** |

**`1/z` による集中と、浮動小数の精度分布が、ほぼ正確に打ち消し合います。** 結果として、**近くから遠くまでほぼ均一な精度**が得られます。

`z = 1000` 付近での 1 メートルは、Reversed-Z では約 10⁻⁷ の差になりますが、**その領域の浮動小数の刻みは 10⁻¹⁴ 程度**です。**1000 万倍の余裕があります。**

### 19.5.4 実装に必要な 4 つの変更

**Reversed-Z の導入は、5 行程度で済みます。ただし 4 箇所すべてを変える必要があります。**

**① 射影行列で `nearZ` と `farZ` を入れ替える**

```cpp
const auto proj = Math::PerspectiveFovLH(
    Math::ToRadians(60.0f), aspect,
    farZ,      // ← 入れ替え
    nearZ);    // ← 入れ替え
```

**② クリア値を 0.0 にする**

```cpp
clearValue.DepthStencil.Depth = 0.0f;
```

```cpp
m_commandList->ClearDepthStencilView(dsv, D3D12_CLEAR_FLAG_DEPTH,
                                     0.0f, 0, 0, nullptr);
```

**③ 比較関数を `GREATER` にする**

```cpp
desc.DepthStencilState.DepthFunc = D3D12_COMPARISON_FUNC_GREATER;
```

**④ 深度フォーマットが浮動小数であることを確認する**

```cpp
DXGI_FORMAT_D32_FLOAT      // ✅
DXGI_FORMAT_D24_UNORM_S8_UINT   // ❌ 効果なし
```

**④ を見落とすと、まったく意味がありません。**

`D24_UNORM` は**固定小数**です。0 から 1 の範囲を等間隔に 24bit で刻むだけなので、**どこを 0 にしても精度は変わりません。** 19.5.2 節の「浮動小数の精度分布」という前提が成り立たないからです。

**Reversed-Z は、浮動小数の深度バッファでのみ意味を持ちます。**

### 19.5.5 本書での扱い

**本書は、切り替えられる形で用意します。**

```cpp
// src/Graphics/DepthConfig.h
#pragma once

//---------------------------------------------------------------
// Reversed-Z を使うかどうか。
//
// 1 にすると、近 = 1.0、遠 = 0.0 になり、遠方の深度精度が
// 大幅に改善する(第19章 19.5 節)。
//
// D32_FLOAT のような浮動小数フォーマットでのみ意味がある。
//---------------------------------------------------------------
#define USE_REVERSED_Z 1

namespace Graphics
{
#if USE_REVERSED_Z
    inline constexpr float kDepthClearValue = 0.0f;
    inline constexpr D3D12_COMPARISON_FUNC kDepthFunc =
        D3D12_COMPARISON_FUNC_GREATER;
#else
    inline constexpr float kDepthClearValue = 1.0f;
    inline constexpr D3D12_COMPARISON_FUNC kDepthFunc =
        D3D12_COMPARISON_FUNC_LESS;
#endif
}
```

**定数を 1 箇所にまとめておくのが要点です。** 4 箇所のうち 1 つでも直し忘れると、**画面が真っ暗になるか、何も描かれなくなります。**

射影行列側も揃えます。

```cpp
[[nodiscard]] inline Math::Matrix4x4 MakeProjection(
    float fovY, float aspect, float nearZ, float farZ) noexcept
{
#if USE_REVERSED_Z
    return Math::PerspectiveFovLH(fovY, aspect, farZ, nearZ);
#else
    return Math::PerspectiveFovLH(fovY, aspect, nearZ, farZ);
#endif
}
```

> **Reversed-Z にすると混乱する場面もある**
>
> Nsight Graphics や PIX で深度バッファを覗くと、**普通と逆に見えます。** 手前が白く、奥が黒くなります。
>
> また、ネット上の資料やシェーダーのサンプルは、ほぼすべて標準の向きを前提にしています。**深度を使うポストエフェクト(第26章)を移植するとき、読み替えが必要になります。**
>
> **利点が明確なので本書は採用しますが、「常に正解」ではありません。** シーンが狭く、`nearZ` を十分大きく取れるなら、標準のままでも困りません。

---

## ✅ 本章のゴール:前後関係が正しく描画される

### Step 1:実行する

**立方体が回転し、面の前後関係が正しくなります。**

第18章 Step 2 で目に焼き付けた破綻が消えているはずです。

- 奥の面が手前を隠さない
- 回転の途中で描画順が入れ替わらない
- どの角度から見ても自然

**カリングを `NONE` にしてみると、深度テストの効果がより明確に分かります。** 裏面も描かれますが、手前の面に隠されて見えません。

### Step 2:クリアを忘れてみる

```cpp
// m_commandList->ClearDepthStencilView(...);   ← コメントアウト
```

**1 フレーム目は正しく描かれ、2 フレーム目以降は絵が止まります。**

前フレームの深度が残っているため、**ほとんどのピクセルが深度テストに落ちます。** 回転しているのに、最初の姿のまま固まったように見えるはずです。

**確認したら元に戻してください。**

### Step 3:クリア値を食い違わせる

```cpp
// リソース生成時
clearValue.DepthStencil.Depth = 1.0f;

// クリア時
m_commandList->ClearDepthStencilView(dsv, D3D12_CLEAR_FLAG_DEPTH,
                                     0.5f, 0, 0, nullptr);   // ❌ 食い違い
```

```
[Warn ] Log.cpp(60): [D3D12] (id 820) D3D12 WARNING:
  ClearDepthStencilView: The clear values do not match those passed
  to resource creation. ...
```

**第7章 7.6.3 節で「抑制する例」として挙げたメッセージの実物です。**

そして、**これは抑制すべきではない**警告です。性能が落ちていることを教えてくれています。**正しい対処は、値を一致させることです。**

**確認したら元に戻してください。**

### Step 4:比較関数を間違える

Reversed-Z を有効にした状態で、比較関数だけ `LESS` に戻します。

```cpp
desc.DepthStencilState.DepthFunc = D3D12_COMPARISON_FUNC_LESS;   // ❌
```

**何も描かれません。**

クリア値が `0.0` で、すべての深度値が `0.0` 以上なので、`LESS` の判定に一度も通らないからです。

**19.5.4 節で「4 箇所すべてを変える必要がある」と書いた意味が、これで分かります。** 1 つ忘れるだけで全滅します。

**確認したら元に戻してください。**

### Step 5:精度を体感する(任意)

**Z ファイティングを実際に起こしてみます。**

`nearZ` を極端に小さく、`farZ` を極端に大きくします。

```cpp
constexpr float kNearZ = 0.001f;    // ← 極端に小さく
constexpr float kFarZ  = 10000.0f;  // ← 極端に大きく
```

そして、**立方体をもう 1 つ、ほんのわずかに大きくして重ねて描きます。**

```cpp
const auto world2 = Math::Scaling(1.001f, 1.001f, 1.001f) * world;
```

**標準の深度(`USE_REVERSED_Z 0`)では、面がちらつきます。** 2 つの立方体の表面が近すぎて、深度が区別できなくなるからです。

**`USE_REVERSED_Z 1` に切り替えると、ちらつきが消えます。**

**この違いが、19.5 節のすべてです。**

より現実的なシーンでは、第23章でモデルを読み込むと、地面と物体が接する部分などで自然に発生します。

---

### 本章の達成状態

- [ ] `MakeTexture2DDesc` を自作した
- [ ] `Layout` が `UNKNOWN` になっている(バッファとの違い)
- [ ] `ALLOW_DEPTH_STENCIL` フラグを付けた
- [ ] 最適化クリア値を指定した
- [ ] クリア時の値が最適化クリア値と一致している
- [ ] 深度バッファを 1 枚だけ作った(理由を理解した)
- [ ] DSV ヒープを `SHADER_VISIBLE` なしで作った
- [ ] `Resize` で深度バッファを作り直している
- [ ] PSO の `DepthEnable = FALSE` を削除した
- [ ] `DSVFormat` を設定した
- [ ] `OMSetRenderTargets` の第 4 引数に DSV を渡している
- [ ] 毎フレーム深度をクリアしている
- [ ] **立方体の前後関係が正しくなった**
- [ ] Reversed-Z の 4 つの変更点を理解した

---

## トラブルシューティング

| 症状 | 原因 | 対処 |
|---|---|---|
| DSV 生成に失敗 | `ALLOW_DEPTH_STENCIL` を付け忘れた | 19.2.1 |
| 同上 | DSV ヒープに `SHADER_VISIBLE` を指定 | 19.3.1 |
| `OMSetRenderTargets` でエラー | 深度バッファのサイズ不一致 | `Resize` を確認(19.3.3) |
| 同上 | PSO の `DSVFormat` が `UNKNOWN` | 19.4.1 |
| 2 フレーム目から絵が止まる | 深度をクリアしていない | 19.4.3 |
| クリア値の警告 | 最適化クリア値と不一致 | **抑制せず一致させる**(19.2.3) |
| 何も描かれない | Reversed-Z の設定が中途半端 | 4 箇所すべて確認(19.5.4) |
| Reversed-Z の効果がない | フォーマットが `D24_UNORM` | `D32_FLOAT` に(19.5.4 ④) |
| 遠くの面がちらつく | `nearZ` が小さすぎる | 大きくする、または Reversed-Z |
| 半透明が正しく描けない | 深度テストだけでは不足 | **第28章で扱う** |
| Nsight で深度が逆に見える | Reversed-Z を使っている | 仕様(19.5.5) |

---

## まとめ

**1. 深度バッファは、描画順への依存をなくす。**
ピクセル単位で判定するので、貫通する面も正しく扱えます。ソートでは解決できません。

**2. テクスチャの `Layout` は `UNKNOWN`。**
バッファの `ROW_MAJOR` とは違います。GPU が独自の並べ方をするため、`Map` して直接書けません。**第20章でこの事実が効いてきます。**

**3. 最適化クリア値を指定し、その値でクリアする。**
食い違うと性能が落ち、警告が出ます。**第7章で「抑制の例」に挙げたメッセージは、抑制すべきではないものでした。**

**4. 深度バッファは 1 枚でよい。**
同じキューのコマンドは順に実行されるため、2 つのフレームが同時に書くことはありません。バックバッファが複数必要なのは、表示中は OS が握っているからです。

**5. 深度値は `1/z` に比例し、1.0 付近に集中する。**
そして浮動小数は 1.0 付近でもっとも粗い。**最悪の組み合わせです。**

**6. Reversed-Z は 2 つの非線形を打ち消し合わせる。**
遠方を 0.0 付近に写すことで、ほぼ均一な精度が得られます。**ただし浮動小数フォーマットでのみ有効で、4 箇所すべてを変える必要があります。**

**7. 第14章の既定値が報われた。**
`DefaultDepthStencilDesc()` を `d3dx12.h` と同じ値にしておいたので、変更は `DSVFormat` の 1 行だけで済みました。

次章はテクスチャです。DDS ファイルの構造を読み解き、ローダを自作し、そして **`UpdateSubresources()` なしでテクスチャを転送します。** 本章で触れた「テクスチャの `Layout` は `UNKNOWN`」という事実が、`GetCopyableFootprints` と行ピッチのアラインメントという形で立ちはだかります。**本書で最も手強い転送処理です。**

---

## 参考リンク

| 内容 | URL |
|---|---|
| 深度ステンシル ビュー | https://learn.microsoft.com/ja-jp/windows/win32/direct3d12/depth-stencil-view--dsv- |
| `D3D12_DEPTH_STENCIL_DESC` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/ns-d3d12-d3d12_depth_stencil_desc |
| `ClearDepthStencilView` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/nf-d3d12-id3d12graphicscommandlist-cleardepthstencilview |
| `D3D12_CLEAR_VALUE` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/ns-d3d12-d3d12_clear_value |
| 深度バッファの精度 | https://developer.nvidia.com/content/depth-precision-visualized |
| 保守的な深度出力(`SV_DepthLessEqual`) | https://learn.microsoft.com/ja-jp/windows/win32/direct3dhlsl/dx-graphics-hlsl-semantics |
