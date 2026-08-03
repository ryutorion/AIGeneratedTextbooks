# 第16章 インデックスバッファと基本図形

三角形が出ました。次は立方体です。

三角形から立方体へ進むと、**2 つの新しい問題**が現れます。

**1 つ目は頂点の重複です。** 立方体は 12 枚の三角形でできています。素直に書けば 36 個の頂点が必要ですが、実際の角は 8 個しかありません。同じ座標を 4 回も 5 回も書くのは無駄です。**インデックスバッファ**がこれを解決します。

**2 つ目は置き場所です。** 第15章では UPLOAD ヒープを使いました。CPU から `Map` して書けるので楽でしたが、**GPU から見れば PCIe バスの向こう側にある遅いメモリ**です。動かない形状データを毎フレームそこから読むのは無駄です。VRAM 上の DEFAULT ヒープに置くべきですが、**CPU から直接書けません。**

そこで転送が必要になります。`d3dx12.h` には `UpdateSubresources()` という便利な関数がありますが、**本書は使いません。** 自分で書きます。

**本章のゴール**
インデックスバッファで立方体を定義し、DEFAULT ヒープへ転送して描画する。ワイヤーフレームで立方体が表示される。

---

## 16.1 インデックス描画

### 16.1.1 なぜ必要か

四角形を三角形 2 枚で作る場合を考えます。

```
  3---2        3---2      3
  |   |   →   | /        | \
  0---1        0          0---1
```

**素直に書くと 6 頂点**です。しかし実際の角は 4 つで、**2 つが重複**しています。

立方体ならもっと極端です。

| | 頂点数 |
|---|---|
| インデックスなし | 12 三角形 × 3 = **36** |
| インデックスあり | **8**(角の数)+ インデックス 36 個 |

インデックスは 16bit なら 2 バイト、頂点は本書の構造体で 28 バイトです。

```
インデックスなし: 36 × 28 =        1008 バイト
インデックスあり:  8 × 28 + 36 × 2 = 296 バイト
```

**3 分の 1 以下になります。** そして、この差はモデルが大きくなるほど広がります。

**さらに重要な効果があります。** GPU は変換済みの頂点をキャッシュしており、**同じインデックスが再び現れたら頂点シェーダーを再実行しません。** 頂点シェーダーの実行回数そのものが減ります。

> **すべての頂点を共有できるとは限らない**
>
> 立方体を 8 頂点で表せるのは、**頂点が「位置」と「色」しか持たないから**です。
>
> 第22章でライティングを実装すると、頂点に**法線**が必要になります。立方体の角では、隣り合う 3 つの面で法線が違います。位置は同じでも法線が違うので、**共有できません。**
>
> そのため、法線つきの立方体は **8 頂点ではなく 24 頂点**(6 面 × 4 隅)になります。第21章 21.2 節で、この判定を自動化する処理を書きます。

### 16.1.2 インデックスバッファビュー

頂点バッファビューとほぼ同じ形です。

```cpp
D3D12_INDEX_BUFFER_VIEW ibv{};
ibv.BufferLocation = indexBuffer->GetGPUVirtualAddress();
ibv.SizeInBytes    = sizeof(kCubeIndices);
ibv.Format         = DXGI_FORMAT_R16_UINT;
```

**`Format` に指定できるのは 2 つだけです。**

| フォーマット | 型 | 上限 |
|---|---|---|
| `DXGI_FORMAT_R16_UINT` | `uint16_t` | 65,535 頂点 |
| `DXGI_FORMAT_R32_UINT` | `uint32_t` | 約 43 億頂点 |

他の値を入れると `E_INVALIDARG` になります。

**可能な限り 16bit を使ってください。** 帯域が半分で済み、インデックスバッファは頻繁に読まれるため効果があります。第21章でモデルを読み込むとき、頂点数を見て自動的に選ぶようにします。

頂点バッファビューと同じく、**これはデスクリプタではありません。** デスクリプタヒープに置かず、直接コマンドリストに渡します(第15章 15.3.3 節)。

### 16.1.3 `DrawIndexedInstanced`

```cpp
commandList->IASetVertexBuffers(0, 1, &m_vertexBufferView);
commandList->IASetIndexBuffer(&m_indexBufferView);

commandList->DrawIndexedInstanced(
    36,   // IndexCountPerInstance
    1,    // InstanceCount
    0,    // StartIndexLocation
    0,    // BaseVertexLocation
    0);   // StartInstanceLocation
```

第15章の `DrawInstanced` との違いは、**頂点数ではなくインデックス数**を指定する点と、`BaseVertexLocation` が増えている点です。

```cpp
INT BaseVertexLocation;   // ← 符号つき整数
```

**インデックスの値に加算されるオフセット**です。「頂点バッファの先頭から N 番目を 0 番として扱う」という指定になります。

これは、**複数のメッシュを 1 本のバッファにまとめる**ときに効きます。

```
頂点バッファ:  [メッシュA の頂点][メッシュB の頂点]
                ↑ Base=0          ↑ Base=100

インデックスバッファ: [A のインデックス][B のインデックス]
                       ↑ Start=0        ↑ Start=150
```

各メッシュのインデックスを 0 起点のまま書いておき、描画時にオフセットを指定できます。**インデックスを書き換えずに統合できる**わけです。第25章で複数オブジェクトを扱うときに使います。

---

## 16.2 四角形、そして立方体

### 16.2.1 四角形

まず四角形で確認します。

```cpp
constexpr Vertex kQuadVertices[] = {
    { { -0.5f, -0.5f, 0.5f }, { 1, 0, 0, 1 } },   // 0 左下
    { {  0.5f, -0.5f, 0.5f }, { 0, 1, 0, 1 } },   // 1 右下
    { {  0.5f,  0.5f, 0.5f }, { 0, 0, 1, 1 } },   // 2 右上
    { { -0.5f,  0.5f, 0.5f }, { 1, 1, 0, 1 } },   // 3 左上
};

constexpr std::uint16_t kQuadIndices[] = {
    0, 3, 2,   // 左下 → 左上 → 右上
    0, 2, 1,   // 左下 → 右上 → 右下
};
```

**巻き順の確認方法は第15章 15.1.2 節と同じです。** 画面を時計の文字盤だと思ってください。

```
0 = 7時(左下)   1 = 5時(右下)
3 = 11時(左上)  2 = 1時(右上)
```

`0 → 3 → 2` は 7時 → 11時 → 1時。**時計回りです。**
`0 → 2 → 1` は 7時 → 1時 → 5時。**同じく時計回りです。**

**z を 0.5 にしている点にも注意してください。** クリップ空間の深度範囲は [0, 1] なので(第13章 13.5.2 節)、`z = 0` でもよいのですが、範囲外にすると**クリップされて消えます。**

### 16.2.2 立方体の 8 頂点

角に番号を振ります。

```
        7--------6
       /|       /|          y
      / |      / |          |
     4--------5  |          +---x
     |  3-----|--2         /
     | /      | /         z
     |/       |/
     0--------1
```

| 番号 | 座標 |
|---|---|
| 0 | (-0.5, -0.5, -0.5) |
| 1 | (+0.5, -0.5, -0.5) |
| 2 | (+0.5, +0.5, -0.5) |
| 3 | (-0.5, +0.5, -0.5) |
| 4 | (-0.5, -0.5, +0.5) |
| 5 | (+0.5, -0.5, +0.5) |
| 6 | (+0.5, +0.5, +0.5) |
| 7 | (-0.5, +0.5, +0.5) |

**0〜3 が手前の面(z = -0.5)、4〜7 が奥の面(z = +0.5)** です。Direct3D の慣習では、**+z が画面の奥**に向かいます。

色は座標から機械的に決めます。

```cpp
// 座標 [-0.5, 0.5] を色 [0, 1] に写す
// → RGB 立方体の 8 頂点になり、見た目に分かりやすい
color = { x + 0.5f, y + 0.5f, z + 0.5f, 1.0f };
```

### 16.2.3 巻き順を揃える

**6 面それぞれについて、「外から見て時計回り」に並べる必要があります。**

面ごとに時計の文字盤で考えるのは、視点が変わるので大変です。**3 次元で判定できる規則を使います。**

> **規則:三角形 (v0, v1, v2) について、`(v1 - v0) × (v2 - v0)` が外向きになるように並べる。**

第15章で確認した手前の面で検算します。三角形 `(0, 3, 2)`:

```
v0 = (-0.5, -0.5, -0.5)
v1 = (-0.5, +0.5, -0.5)      e1 = v1 - v0 = (0, 1, 0)
v2 = (+0.5, +0.5, -0.5)      e2 = v2 - v0 = (1, 0, 0)

e1 × e2 = (0, 0, -1)
```

**手前の面の外向きは -z 方向**なので、正しく外を向いています。第15章で「時計の文字盤」で確かめた結果と一致しました。

この規則で 6 面を並べたものが次です。

```cpp
constexpr std::uint16_t kCubeIndices[] = {
    // 手前 (-Z)
    0, 3, 2,   0, 2, 1,
    // 奥 (+Z)
    4, 5, 6,   4, 6, 7,
    // 左 (-X)
    0, 4, 7,   0, 7, 3,
    // 右 (+X)
    1, 2, 6,   1, 6, 5,
    // 上 (+Y)
    3, 7, 6,   3, 6, 2,
    // 下 (-Y)
    0, 1, 5,   0, 5, 4,
};

static_assert(std::size(kCubeIndices) == 36);
```

**1 面でも間違えると、その面だけが消えます。** ワイヤーフレーム(16.4 節)で確認すると分かりやすくなります。

### 16.2.4 とりあえず CPU 側で変換する(暫定)

**このままでは立方体に見えません。**

第13章のシェーダーは座標変換をしないので、頂点はクリップ空間の値として扱われます。手前の面と奥の面が完全に重なり、**ただの正方形**になります。

正しい解決は、**頂点シェーダーで行列変換をすること**です。しかし、そのためには行列(第17章)と定数バッファ(第18章)が要ります。

**本章では、CPU 側で座標を変換してからバッファに詰めます。**

```cpp
namespace
{
    // 【暫定】第17章で Matrix4x4 として整理し、
    //         第18章で頂点シェーダーへ移す。
    constexpr float kRotY    = 0.6f;   // 約 34 度
    constexpr float kRotX    = 0.5f;   // 約 29 度
    constexpr float kCameraZ = 3.0f;
    constexpr float kScale   = 2.0f;

    void ProjectPoint(const float in[3], float out[3])
    {
        //--- Y 軸まわりに回転 ---
        const float cy = std::cos(kRotY);
        const float sy = std::sin(kRotY);
        const float x1 =  in[0] * cy + in[2] * sy;
        const float y1 =  in[1];
        const float z1 = -in[0] * sy + in[2] * cy;

        //--- X 軸まわりに回転 ---
        const float cx = std::cos(kRotX);
        const float sx = std::sin(kRotX);
        const float x2 = x1;
        const float y2 = y1 * cx - z1 * sx;
        const float z2 = y1 * sx + z1 * cx;

        //--- カメラを後ろへ下げ、単純な透視除算 ---
        const float z = z2 + kCameraZ;
        out[0] = x2 * kScale / z;
        out[1] = y2 * kScale / z;

        // 深度バッファはまだない(第19章)。
        // ただしクリップ範囲 [0,1] を外れると消えるので、
        // 固定値を入れておく。
        out[2] = 0.5f;
    }

    std::vector<Vertex> BuildCubeVertices()
    {
        std::vector<Vertex> vertices;
        vertices.reserve(8);

        for (const auto& corner : kCubeCorners)
        {
            Vertex v{};
            ProjectPoint(corner, v.position);
            v.color[0] = corner[0] + 0.5f;
            v.color[1] = corner[1] + 0.5f;
            v.color[2] = corner[2] + 0.5f;
            v.color[3] = 1.0f;
            vertices.push_back(v);
        }
        return vertices;
    }
}
```

**このコードには重大な制約があります。**

> **立方体を回転させたければ、頂点バッファを作り直すことになります。**

毎フレーム 8 頂点を再計算して転送する —— **これは完全に間違ったやり方です。** 頂点データは変わっていません。変わったのは視点だけです。

**そして本章では、そもそもそれができません。** 16.3 節で頂点データを DEFAULT ヒープに置くと、CPU から書き換えられなくなるからです。

**この行き詰まりが、第18章の定数バッファの動機です。** 「形は固定、変換だけ毎フレーム変える」を実現する仕組みが必要になります。

> **縦横比について**
>
> ウィンドウが正方形でないため、立方体は横に伸びて見えます。第15章 15.5 節の三角形と同じ現象です。
>
> 第18章で射影行列を導入すると、縦横比を考慮した変換ができるようになります。

---

## 16.3 DEFAULT ヒープへの転送を自分で書く

### 16.3.1 なぜ DEFAULT ヒープなのか

第15章 15.2.1 節で見た通りです。

| ヒープ | 置かれる場所 | GPU からの読み取り |
|---|---|---|
| UPLOAD | システムメモリ | PCIe バス越し。**遅い** |
| **DEFAULT** | **VRAM** | **直結。速い** |

頂点バッファとインデックスバッファは、**毎フレーム、頂点の数だけ読まれます。** 三角形 3 つなら気になりませんが、第21章でモデルを読み込めば数万頂点になります。

**動かないデータは DEFAULT ヒープに置く。** これが原則です。

### 16.3.2 `UpdateSubresources()` が使えない

`d3dx12.h` には、この転送を 1 行で済ませる関数があります。

```cpp
// 本書では使わない
UpdateSubresources<1>(commandList, destBuffer, stagingBuffer,
                      0, 0, 1, &subresourceData);
```

**本書は使いません**(第1章 1.3.1 節)。自分で書きます。

**幸い、バッファの転送は単純です。** テクスチャ(第20章)と違い、行ピッチのアラインメントもフットプリントの計算も不要です。**`CopyBufferRegion` を 1 回呼ぶだけ**で済みます。

第20章でテクスチャを転送するとき、この差を実感することになります。

### 16.3.3 転送の手順

```
① DEFAULT ヒープにリソースを作る(状態 = COPY_DEST)
        ↓
② UPLOAD ヒープに中間バッファを作る
        ↓
③ 中間バッファに Map して memcpy
        ↓
④ CopyBufferRegion で中間 → DEFAULT へコピーを記録
        ↓
⑤ バリアで COPY_DEST → 本来の用途へ遷移
        ↓
⑥ コマンドリストを閉じて実行
        ↓
⑦ GPU の完了を待つ          ← 忘れてはいけない
        ↓
⑧ 中間バッファを解放
```

**⑦ が最重要です。** 16.3.4 節で詳しく述べます。

コードにするとこうなります。

```cpp
//--- ① 転送先 ---
const auto defaultHeap = MakeHeapProperties(D3D12_HEAP_TYPE_DEFAULT);
const auto bufferDesc  = MakeBufferDesc(sizeInBytes);

ComPtr<ID3D12Resource> buffer;
HR_TRY(device->CreateCommittedResource(
    &defaultHeap,
    D3D12_HEAP_FLAG_NONE,
    &bufferDesc,
    D3D12_RESOURCE_STATE_COPY_DEST,   // ← コピー先として作る
    nullptr,
    IID_PPV_ARGS(&buffer)));

//--- ② 中間バッファ ---
const auto uploadHeap = MakeHeapProperties(D3D12_HEAP_TYPE_UPLOAD);

ComPtr<ID3D12Resource> staging;
HR_TRY(device->CreateCommittedResource(
    &uploadHeap,
    D3D12_HEAP_FLAG_NONE,
    &bufferDesc,
    D3D12_RESOURCE_STATE_GENERIC_READ,   // UPLOAD は必ずこれ(第15章)
    nullptr,
    IID_PPV_ARGS(&staging)));

//--- ③ 書き込む ---
void* mapped = nullptr;
const D3D12_RANGE readRange{ 0, 0 };
HR_TRY(staging->Map(0, &readRange, &mapped));
std::memcpy(mapped, data, sizeInBytes);
staging->Unmap(0, nullptr);

//--- ④ コピーを記録 ---
commandList->CopyBufferRegion(
    buffer.Get(), 0,      // コピー先とオフセット
    staging.Get(), 0,     // コピー元とオフセット
    sizeInBytes);

//--- ⑤ 用途に応じた状態へ ---
const auto barrier = MakeTransitionBarrier(
    buffer.Get(),
    D3D12_RESOURCE_STATE_COPY_DEST,
    finalState);
commandList->ResourceBarrier(1, &barrier);
```

**⑤ の遷移先は用途によって変わります。**

| 用途 | 状態 |
|---|---|
| 頂点バッファ | `D3D12_RESOURCE_STATE_VERTEX_AND_CONSTANT_BUFFER` |
| インデックスバッファ | `D3D12_RESOURCE_STATE_INDEX_BUFFER` |
| 定数バッファ(第18章) | `D3D12_RESOURCE_STATE_VERTEX_AND_CONSTANT_BUFFER` |
| シェーダーから読む(第20章) | `D3D12_RESOURCE_STATE_PIXEL_SHADER_RESOURCE` など |

### 16.3.4 中間バッファの寿命

**ここが最も間違えやすい箇所です。**

```cpp
{
    ComPtr<ID3D12Resource> staging;
    // ... 中間バッファを作って、コピーコマンドを記録 ...
}   // ❌ ここで staging が解放される
commandList->Close();
queue->ExecuteCommandLists(1, lists);
```

**`CopyBufferRegion` を呼んだ時点では、コピーは実行されていません。** コマンドリストに記録されただけです(第9章 9.1.1 節)。

GPU が実際にコピーするのは、`ExecuteCommandLists` の後、しかも**いつになるかは分かりません。** そのとき中間バッファが解放されていれば、**GPU は消えたメモリを読みます。**

**第10章 10.4.1 節で終了処理について述べたのと同じ話です。** D3D12 は GPU の使用状況を参照カウントで管理しません。

**対策は 2 つあります。**

1. **GPU の完了を待ってから解放する**(本章の方法)
2. 中間バッファを保持し続け、後で安全なタイミングで解放する(第21章)

初期化時の一度きりの転送なら、待って構いません。**第10章で作った `WaitForGpuIdle` を使います。**

### 16.3.5 状態の昇格について

**参考書やサンプルコードを読むと、⑤ のバリアがないものを見かけます。** 誤りではありません。

D3D12 には**状態の暗黙的な昇格(state promotion)**という仕組みがあります。

> **バッファと一部のテクスチャは、`COMMON` 状態から任意の状態へ、バリアなしで自動的に遷移する。**

つまり、DEFAULT ヒープのバッファを `D3D12_RESOURCE_STATE_COMMON` で作れば、コピー時に自動的に `COPY_DEST` として扱われ、描画時には自動的に `VERTEX_AND_CONSTANT_BUFFER` として扱われます。**バリアは要りません。**

さらに**状態の減衰(state decay)**もあり、コマンドリストの実行が終わると `COMMON` に戻ります。

**それでも本書は明示的にバリアを書きます。** 理由は 3 つです。

- **規則が複雑です。** 「バッファは昇格する」「テクスチャは条件つき」「減衰はキューの種類による」—— 覚え違いをすると、静かに壊れます
- **明示的なほうが読んで分かります。** リソースが今どの状態にあるかが、コードに書いてあります
- **第30章の Enhanced Barriers へ移行しやすくなります。** 遷移を書いてあれば、対応させるだけで済みます

**「知っていて省略する」のと「知らずに書かない」のは別物です。** 本書は前者を目指しますが、コードは後者と見分けがつく形にしておきます。

### 16.3.6 `ResourceUploader` を実装する

転送処理は、この先何度も使います。第20章のテクスチャ、第21章のモデル —— **初期化時に GPU へデータを送る場面は繰り返し現れます。**

**まとめておきます。**

```cpp
// src/Graphics/ResourceUploader.h
#pragma once
#include "std_import.h"
#include "Core/Error.h"
#include "Graphics/Fence.h"

namespace Graphics
{
    //-----------------------------------------------------------
    // 初期化時のリソース転送をまとめて行う。
    //
    //   uploader.Begin();
    //   auto vb = uploader.UploadBuffer(...);
    //   auto ib = uploader.UploadBuffer(...);
    //   uploader.End(queue);      ← ここで実行し、完了を待つ
    //
    // 中間バッファは End() が返るまで保持される。
    //-----------------------------------------------------------
    class ResourceUploader
    {
    public:
        Core::Status Initialize(ID3D12Device* device);
        void         Shutdown();

        Core::Status Begin();

        Core::Result<Microsoft::WRL::ComPtr<ID3D12Resource>> UploadBuffer(
            const void*           data,
            UINT64                sizeInBytes,
            D3D12_RESOURCE_STATES finalState,
            std::wstring_view     name);

        Core::Status End(ID3D12CommandQueue* queue);

    private:
        ID3D12Device* m_device = nullptr;

        Microsoft::WRL::ComPtr<ID3D12CommandAllocator>    m_allocator;
        Microsoft::WRL::ComPtr<ID3D12GraphicsCommandList> m_commandList;
        Fence m_fence;

        // End() まで保持しなければならない(16.3.4)
        std::vector<Microsoft::WRL::ComPtr<ID3D12Resource>> m_stagingBuffers;

        bool m_recording = false;
    };
}
```

**専用のコマンドアロケータとコマンドリストを持つ**点に注目してください。第12章 12.2 節で作ったフレームリソースを流用してはいけません。**初期化時の転送と、毎フレームの描画は、別の寿命を持つからです。**

実装の要所です。

```cpp
Core::Result<ComPtr<ID3D12Resource>> ResourceUploader::UploadBuffer(
    const void* data, UINT64 sizeInBytes,
    D3D12_RESOURCE_STATES finalState, std::wstring_view name)
{
    D3D_ASSERT_MSG(m_recording, L"Begin() を先に呼ぶこと");

    //--- ① 転送先(DEFAULT ヒープ) ---
    const auto defaultHeap = MakeHeapProperties(D3D12_HEAP_TYPE_DEFAULT);
    const auto desc        = MakeBufferDesc(sizeInBytes);

    ComPtr<ID3D12Resource> buffer;
    HR_TRY(m_device->CreateCommittedResource(
        &defaultHeap, D3D12_HEAP_FLAG_NONE, &desc,
        D3D12_RESOURCE_STATE_COPY_DEST, nullptr,
        IID_PPV_ARGS(&buffer)));

    Core::SetDebugName(buffer.Get(), name);

    //--- ② 中間バッファ(UPLOAD ヒープ) ---
    const auto uploadHeap = MakeHeapProperties(D3D12_HEAP_TYPE_UPLOAD);

    ComPtr<ID3D12Resource> staging;
    HR_TRY(m_device->CreateCommittedResource(
        &uploadHeap, D3D12_HEAP_FLAG_NONE, &desc,
        D3D12_RESOURCE_STATE_GENERIC_READ, nullptr,
        IID_PPV_ARGS(&staging)));

    Core::SetDebugNameF(staging.Get(), L"Staging({})", name);

    //--- ③ 書き込む ---
    void* mapped = nullptr;
    const D3D12_RANGE readRange{ 0, 0 };
    HR_TRY(staging->Map(0, &readRange, &mapped));
    std::memcpy(mapped, data, static_cast<std::size_t>(sizeInBytes));
    staging->Unmap(0, nullptr);

    //--- ④ コピーを記録 ---
    m_commandList->CopyBufferRegion(
        buffer.Get(), 0, staging.Get(), 0, sizeInBytes);

    //--- ⑤ 状態遷移 ---
    const auto barrier = MakeTransitionBarrier(
        buffer.Get(), D3D12_RESOURCE_STATE_COPY_DEST, finalState);
    m_commandList->ResourceBarrier(1, &barrier);

    //--- 中間バッファを End() まで保持する(16.3.4) ---
    m_stagingBuffers.push_back(std::move(staging));

    return buffer;
}

Core::Status ResourceUploader::End(ID3D12CommandQueue* queue)
{
    D3D_ASSERT(m_recording);

    HR_TRY(m_commandList->Close());

    ID3D12CommandList* lists[] = { m_commandList.Get() };
    queue->ExecuteCommandLists(1, lists);

    //--- 完了を待つ。これがあるから中間バッファを解放できる ---
    if (auto r = WaitForGpuIdle(queue, m_fence); !r)
    {
        return r;
    }

    LOG_INFO(L"uploaded {} buffer(s)", m_stagingBuffers.size());

    m_stagingBuffers.clear();   // ← 待った後だから安全
    m_recording = false;
    return {};
}
```

**`m_stagingBuffers.clear()` が `WaitForGpuIdle` の後にある**ことが、この実装の要です。順序を入れ替えると、動くこともあれば落ちることもある、という最悪の状態になります。

使い方はこうなります。

```cpp
Graphics::ResourceUploader uploader;
if (auto r = uploader.Initialize(device); !r) { /* ... */ }

if (auto r = uploader.Begin(); !r) { /* ... */ }

const auto vertices = BuildCubeVertices();

auto vb = uploader.UploadBuffer(
    vertices.data(),
    vertices.size() * sizeof(Vertex),
    D3D12_RESOURCE_STATE_VERTEX_AND_CONSTANT_BUFFER,
    L"CubeVertexBuffer");

auto ib = uploader.UploadBuffer(
    kCubeIndices, sizeof(kCubeIndices),
    D3D12_RESOURCE_STATE_INDEX_BUFFER,
    L"CubeIndexBuffer");

if (auto r = uploader.End(queue); !r) { /* ... */ }
```

**複数のバッファを 1 回の実行にまとめられます。** 転送ごとに GPU を待つのは無駄なので、`Begin` / `End` で挟む形にしました。

---

## 16.4 トポロジ・カリング・ワイヤーフレーム

### 16.4.1 ワイヤーフレーム用の PSO

**塗りつぶしを線描画に変えます。**

```cpp
auto desc = DefaultGraphicsPipelineStateDesc();
// ... 共通の設定 ...
desc.RasterizerState.FillMode = D3D12_FILL_MODE_WIREFRAME;
desc.RasterizerState.CullMode = D3D12_CULL_MODE_NONE;
```

**カリングも切ります。** ワイヤーフレームで奥の辺まで見たいからです。

**PSO が 2 つになりました。** 第14章 14.5 節で「当面 1 つ」と書きましたが、ここで増えます。とはいえまだ 2 つなので、素直に 2 つ持っておきます。

```cpp
ComPtr<ID3D12PipelineState> m_psoSolid;
ComPtr<ID3D12PipelineState> m_psoWireframe;
bool m_wireframe = true;
```

**同じルートシグネチャを使い回している**点に注目してください。第14章 14.3 節で「C++ 側にルートシグネチャを書く理由」として挙げた「共有しやすい」が、さっそく効いています。

描画時に切り替えます。

```cpp
m_commandList->SetPipelineState(
    m_wireframe ? m_psoWireframe.Get() : m_psoSolid.Get());
```

キー入力で切り替えられるようにしておくと、確認が楽になります。第22章で本格的な入力処理を作るまでの間に合わせとして、次で十分です。

```cpp
// Window の WndProc に追加
case WM_KEYDOWN:
    if (wParam == 'W' && OnToggleWireframe)
    {
        OnToggleWireframe();
    }
    return 0;
```

### 16.4.2 ワイヤーフレームで何が見えるか

**三角形分割が見えます。**

```
    +--------+
   /|\      /|
  / | \    / |
 +--------+  |
 |  |   \ |  |
 |  +----\|--+
 | /      \ /
 |/        X
 +--------+
```

各面に**対角線**が入っています。四角形が三角形 2 枚でできていることが、目で確認できます。

**これは巻き順の検証にも使えます。** カリングを `BACK` に戻すと、外を向いている面だけが残ります。**1 面でも巻き順を間違えていれば、その面の 2 本の対角線が消えるか、逆に裏面だけが残ります。**

### 16.4.3 トポロジを変えてみる

```cpp
m_commandList->IASetPrimitiveTopology(D3D_PRIMITIVE_TOPOLOGY_LINELIST);
```

**インデックスを 2 個ずつ組にして線分として描きます。** 三角形用のインデックス列をそのまま線で描くので、意味のある図形にはなりませんが、**トポロジがインデックスの解釈を変えるだけのもの**だと分かります。

**注意点が一つあります。** PSO 側の `PrimitiveTopologyType` は `TRIANGLE` のままなので、厳密には食い違っています。デバッグレイヤーが警告する場合があります。

**本来は、線を描くなら PSO 側も `LINE` にすべきです。**

```cpp
desc.PrimitiveTopologyType = D3D12_PRIMITIVE_TOPOLOGY_TYPE_LINE;
```

第15章 15.5.3 節で「PSO は大分類、コマンドリストは具体的な並べ方」と説明しました。**大分類が違えば、PSO も別に必要です。**

---

## ✅ 本章のゴール:立方体のワイヤーフレームが出る

### Step 1:四角形を確認する

まず立方体の前に、四角形で動作を確認します。

- [ ] 4 頂点・6 インデックスで四角形が表示される
- [ ] 頂点の色が四隅から補間されている

**インデックス描画が機能していることを、単純な形で確かめてから進みます。**

### Step 2:立方体を表示する

```
[Info ] ResourceUploader.cpp(112): uploaded 2 buffer(s)
```

**斜めから見た立方体が、ワイヤーフレームで表示されます。**

- 12 本の辺
- 各面に 1 本ずつの対角線(三角形分割)
- 頂点の色が RGB の立方体状に分布

### Step 3:カリングを有効にする

```cpp
desc.RasterizerState.CullMode = D3D12_CULL_MODE_BACK;
```

**手前の 3 面だけが残ります。** 奥の 3 面は裏を向いているので捨てられます。

**このとき、6 面すべてが正しく処理されているかを確認してください。**

| 症状 | 意味 |
|---|---|
| 手前 3 面がきれいに残る | **巻き順は正しい** |
| ある面だけ消える | その面の巻き順が逆 |
| 奥の面が残り手前が消える | 全体の巻き順が逆 |

**16.2.3 節の規則で並べたなら、正しく出るはずです。**

### Step 4:塗りつぶしに戻す

```cpp
desc.RasterizerState.FillMode = D3D12_FILL_MODE_SOLID;
desc.RasterizerState.CullMode = D3D12_CULL_MODE_BACK;
```

**色のついた立方体が表示されます。**

ここで**重要な観察**があります。**面の前後関係が正しくありません。**

奥の面が手前の面を上書きすることがあります。**深度テストがないから**です。描画順序がそのまま重なり順になっています。

**これが第19章の動機です。** 深度バッファを導入すると解決します。

### Step 5:中間バッファの寿命を壊してみる

**16.3.4 節の警告を確かめます。**

`ResourceUploader::End` の順序を入れ替えてください。

```cpp
m_stagingBuffers.clear();              // ❌ 待つ前に解放
if (auto r = WaitForGpuIdle(queue, m_fence); !r) { return r; }
```

**結果は環境によって変わります。**

- 何も起きない(コピーが先に終わっていた)
- デバッグレイヤーが警告を出す
- 描画結果が壊れる
- クラッシュする

**「何も起きない」が一番危険です。** 第10章 10.4.1 節で述べた通り、たまたま間に合っただけの状態です。データが大きくなれば必ず壊れます。

**確認したら元に戻してください。**

### Step 6:巻き順を 1 面だけ壊す

```cpp
// 上面だけ逆順にする
3, 6, 7,   3, 2, 6,   // ❌
```

カリングを `BACK` にすると、**上面だけが消えます。** ワイヤーフレームなら対角線の様子で判別できます。

**確認したら元に戻してください。**

---

### 本章の達成状態

- [ ] インデックスバッファビューの `Format` を `R16_UINT` にした
- [ ] `DrawIndexedInstanced` で描画している
- [ ] 立方体の 6 面すべての巻き順が正しい
- [ ] `ResourceUploader` を実装した
- [ ] DEFAULT ヒープを `COPY_DEST` で作成している
- [ ] `CopyBufferRegion` でコピーしている
- [ ] 用途に応じた状態へ遷移させている
- [ ] **中間バッファを `WaitForGpuIdle` の後に解放している**
- [ ] 頂点バッファとインデックスバッファに名前を付けた
- [ ] ワイヤーフレーム用の PSO を作った
- [ ] **立方体のワイヤーフレームが表示された**
- [ ] Step 3 でカリングによる面の消え方を確認した
- [ ] Step 4 で深度テストがないことによる破綻を確認した

---

## トラブルシューティング

| 症状 | 原因 | 対処 |
|---|---|---|
| `IASetIndexBuffer` でエラー | `Format` が `R16_UINT`/`R32_UINT` 以外 | 16.1.2 |
| 一部の面だけ消える | その面の巻き順が逆 | 16.2.3 の規則で確認 |
| 全部消える | 全体の巻き順が逆、またはカリング | `CULL_NONE` で切り分け |
| ただの正方形に見える | 座標変換をしていない | 16.2.4(暫定コード) |
| 何も出ない | クリップ範囲 [0,1] の外 | `z` を確認(16.2.1) |
| `CreateCommittedResource` が失敗 | DEFAULT なのに `GENERIC_READ` | `COPY_DEST` にする |
| コピー後に描画するとエラー | 状態遷移を忘れた | 16.3.3 の ⑤ |
| たまに壊れる / たまに落ちる | **中間バッファの寿命** | 16.3.4 |
| 面の前後関係がおかしい | 深度テストがない | **第19章で解決** |
| 立方体が横に伸びる | 縦横比を考慮していない | **第18章で解決** |
| 回転させたい | 頂点バッファを作り直すしかない | **第18章で解決** |

---

## まとめ

**1. インデックスバッファは容量と実行回数の両方を減らす。**
頂点の重複が消えるだけでなく、頂点キャッシュが効いて頂点シェーダーの実行回数そのものが減ります。

**2. `Format` は `R16_UINT` か `R32_UINT` の 2 択。**
可能な限り 16bit を使ってください。

**3. 巻き順は 3 次元で判定できる。**
`(v1-v0) × (v2-v0)` が外向きになるように並べます。面ごとに視点を変えて考えるより確実です。

**4. 動かないデータは DEFAULT ヒープへ。**
UPLOAD ヒープはシステムメモリ上にあり、GPU からは PCIe 越しの遅いアクセスになります。

**5. バッファの転送は `CopyBufferRegion` 1 回。**
`UpdateSubresources()` を使わなくても、バッファなら難しくありません。**テクスチャは事情が違います**(第20章)。

**6. 中間バッファは GPU の完了を待ってから解放する。**
`CopyBufferRegion` を呼んだ時点では、まだ何も起きていません。**「たまたま動く」が一番危険です。**

**7. 状態の暗黙的昇格を知っていて、あえて書く。**
バリアを省略できる規則は存在しますが、複雑です。明示的に書くほうが読みやすく、第30章への移行も楽になります。

**8. 本章の暫定コードは間違っている。**
CPU 側で座標を変換する方式では、視点を変えるたびに頂点バッファを作り直すことになります。**第18章で正します。**

次章では数学ライブラリを自作します。`Vector3`、`Matrix4x4`、そして `LookAt` と射影行列。**DirectXMath を使わない、本書で最も「フルスクラッチ」な章の一つです。** 単体テストを書いて手計算と照合するところまでやります。

---

## 参考リンク

| 内容 | URL |
|---|---|
| `D3D12_INDEX_BUFFER_VIEW` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/ns-d3d12-d3d12_index_buffer_view |
| `DrawIndexedInstanced` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/nf-d3d12-id3d12graphicscommandlist-drawindexedinstanced |
| `CopyBufferRegion` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/nf-d3d12-id3d12graphicscommandlist-copybufferregion |
| リソースの状態遷移と昇格 | https://learn.microsoft.com/ja-jp/windows/win32/direct3d12/using-resource-barriers-to-synchronize-resource-states-in-direct3d-12 |
| プリミティブ トポロジ | https://learn.microsoft.com/ja-jp/windows/win32/direct3d12/primitive-topologies |
