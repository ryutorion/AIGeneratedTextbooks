# 第30章 バリアを正しく理解する

第11章 11.5 節で、初めてリソースバリアを書きました。

```cpp
const auto barrier = MakeTransitionBarrier(
    backBuffer,
    D3D12_RESOURCE_STATE_PRESENT,
    D3D12_RESOURCE_STATE_RENDER_TARGET);
```

**あのときは、状態が 2 つしかありませんでした。** `PRESENT` と `RENDER_TARGET` を往復するだけです。

**今はどうでしょうか。**

```
ShadowMap:      DEPTH_WRITE ↔ PIXEL_SHADER_RESOURCE
SceneColorMS:   RENDER_TARGET ↔ RESOLVE_SOURCE
SceneColor:     RESOLVE_DEST ↔ PIXEL_SHADER_RESOURCE ↔ RENDER_TARGET
BloomTexture:   RENDER_TARGET ↔ PIXEL_SHADER_RESOURCE  (×3 段)
BackBuffer:     PRESENT ↔ RENDER_TARGET
```

**6 つのリソースが、それぞれ複数の状態を行き来しています。** そして第26章 26.1.4 節で書いた通り、**現在の状態を追跡しているのは我々自身です。**

本章では、この設計自体を見直します。**Enhanced Barriers** は、レガシーバリアが抱えていた根本的な問題に対する答えです。

**本章のゴール**
レガシーバリアの限界を理解し、Enhanced Barriers へ移行する。GPU-Based Validation でバリアの誤りを検出できるようになる。

---

## 30.1 レガシーバリアの限界

### 30.1.1 状態は「用途の集合」だった

**`D3D12_RESOURCE_STATES` を、改めて見てみます。**

```cpp
typedef enum D3D12_RESOURCE_STATES {
    D3D12_RESOURCE_STATE_COMMON                     = 0,
    D3D12_RESOURCE_STATE_VERTEX_AND_CONSTANT_BUFFER = 0x1,
    D3D12_RESOURCE_STATE_INDEX_BUFFER               = 0x2,
    D3D12_RESOURCE_STATE_RENDER_TARGET              = 0x4,
    D3D12_RESOURCE_STATE_UNORDERED_ACCESS           = 0x8,
    D3D12_RESOURCE_STATE_DEPTH_WRITE                = 0x10,
    D3D12_RESOURCE_STATE_DEPTH_READ                 = 0x20,
    D3D12_RESOURCE_STATE_NON_PIXEL_SHADER_RESOURCE  = 0x40,
    D3D12_RESOURCE_STATE_PIXEL_SHADER_RESOURCE      = 0x80,
    // ...
} D3D12_RESOURCE_STATES;
```

**ビットフラグです。** 組み合わせられます。

```cpp
D3D12_RESOURCE_STATE_PIXEL_SHADER_RESOURCE |
D3D12_RESOURCE_STATE_NON_PIXEL_SHADER_RESOURCE
```

**問題は、この「状態」が 2 つの異なる概念を混ぜていることです。**

| 概念 | 例 |
|---|---|
| **どう使うか(アクセス)** | 読む / 書く / レンダーターゲット |
| **メモリの配置(レイアウト)** | 圧縮されているか、スウィズルの形式 |

**そして 3 つ目の概念が、暗黙のうちに含まれています。**

| 概念 | 説明 |
|---|---|
| **いつ待つか(同期)** | どのパイプラインステージの完了を待つか |

**レガシーバリアでは、この 3 つが分離できません。**

### 30.1.2 過剰な同期が起きる

**具体例で見ます。**

```cpp
//--- テクスチャを、ピクセルシェーダーから読める状態にする ---
const auto barrier = MakeTransitionBarrier(
    texture,
    D3D12_RESOURCE_STATE_COPY_DEST,
    D3D12_RESOURCE_STATE_PIXEL_SHADER_RESOURCE);
commandList->ResourceBarrier(1, &barrier);
```

**このバリアが、実際に何を要求しているか。**

```
「これ以前のすべてのコマンドの、すべてのステージが完了するまで待て」
「メモリのレイアウトを変換せよ」
「キャッシュをフラッシュせよ」
```

**本当に必要なのは、コピー処理の完了だけです。** それ以前に実行された頂点シェーダーやラスタライズは、このテクスチャと無関係です。

**それでも待たされます。** レガシーバリアには「何を待つか」を指定する手段がないからです。

### 30.1.3 状態の追跡が破綻する

**より深刻な問題があります。**

**リソースの状態は、コマンドリストをまたいで追跡する必要があります。**

```cpp
// コマンドリスト A
TransitionTo(listA, texture, PIXEL_SHADER_RESOURCE);

// コマンドリスト B(別スレッドで記録)
TransitionTo(listB, texture, RENDER_TARGET);   // 現在の状態は?
```

**第35章でマルチスレッド化すると、これが本格的に問題になります。** 記録の順序と実行の順序が一致しない場合、**状態の追跡は原理的に不可能です。**

**回避策として `D3D12_RESOURCE_STATE_COMMON` を経由する**という手法がありますが、**それは最も重い同期を挟むことを意味します。**

### 30.1.4 サブリソース単位の制約

**第20章 20.3 節でサブリソースを扱いました。**

```cpp
barrier.Transition.Subresource = D3D12_RESOURCE_BARRIER_ALL_SUBRESOURCES;
```

**個別に遷移させることもできますが、追跡がさらに複雑になります。**

```
ミップ 9 段 × 配列 6 枚 = 54 サブリソース
```

**それぞれの状態を管理するのは、現実的ではありません。** 結果として、多くの実装が「全部まとめて遷移」で済ませます。**必要のない同期が発生します。**

---

## 30.2 Enhanced Barriers の設計

### 30.2.1 3 つに分離する

**Enhanced Barriers は、レガシーが混ぜていた概念を分離しました。**

| 概念 | 型 | 何を指定するか |
|---|---|---|
| **同期** | `D3D12_BARRIER_SYNC` | **どのステージの完了を待つか** |
| **アクセス** | `D3D12_BARRIER_ACCESS` | **どう使うか** |
| **レイアウト** | `D3D12_BARRIER_LAYOUT` | **メモリの配置**(テクスチャのみ) |

**これにより、必要な同期だけを指定できます。**

```
レガシー:  COPY_DEST → PIXEL_SHADER_RESOURCE
             ↓ 実際には「全部待つ」

Enhanced:  Sync:   COPY → PIXEL_SHADING
           Access: COPY_DEST → SHADER_RESOURCE
           Layout: COPY_DEST → SHADER_RESOURCE
             ↓ コピーの完了だけを待つ
```

### 30.2.2 同期スコープ

```cpp
typedef enum D3D12_BARRIER_SYNC {
    D3D12_BARRIER_SYNC_NONE                   = 0,
    D3D12_BARRIER_SYNC_ALL                    = 0x1,
    D3D12_BARRIER_SYNC_DRAW                   = 0x2,
    D3D12_BARRIER_SYNC_INDEX_INPUT            = 0x4,
    D3D12_BARRIER_SYNC_VERTEX_SHADING         = 0x8,
    D3D12_BARRIER_SYNC_PIXEL_SHADING          = 0x10,
    D3D12_BARRIER_SYNC_DEPTH_STENCIL          = 0x20,
    D3D12_BARRIER_SYNC_RENDER_TARGET          = 0x40,
    D3D12_BARRIER_SYNC_COMPUTE_SHADING        = 0x80,
    D3D12_BARRIER_SYNC_RAYTRACING             = 0x100,
    D3D12_BARRIER_SYNC_COPY                   = 0x200,
    D3D12_BARRIER_SYNC_RESOLVE                = 0x400,
    // ...
} D3D12_BARRIER_SYNC;
```

**`SyncBefore` と `SyncAfter` の 2 つを指定します。**

```
SyncBefore: 「この処理の完了を待つ」
SyncAfter:  「この処理が始まる前に完了させる」
```

**具体例です。**

| 場面 | SyncBefore | SyncAfter |
|---|---|---|
| コピー → シェーダー読み取り | `COPY` | `PIXEL_SHADING` |
| レンダーターゲット → シェーダー読み取り | `RENDER_TARGET` | `PIXEL_SHADING` |
| 深度書き込み → シェーダー読み取り | `DEPTH_STENCIL` | `PIXEL_SHADING` |
| コンピュート → 描画 | `COMPUTE_SHADING` | `DRAW` |

**`ALL` を指定すると、レガシーと同じ「全部待つ」になります。** 移行の第一歩としては安全ですが、**最適化の余地を捨てることになります。**

### 30.2.3 アクセス

```cpp
typedef enum D3D12_BARRIER_ACCESS {
    D3D12_BARRIER_ACCESS_COMMON                = 0,
    D3D12_BARRIER_ACCESS_VERTEX_BUFFER         = 0x1,
    D3D12_BARRIER_ACCESS_CONSTANT_BUFFER       = 0x2,
    D3D12_BARRIER_ACCESS_INDEX_BUFFER          = 0x4,
    D3D12_BARRIER_ACCESS_RENDER_TARGET         = 0x8,
    D3D12_BARRIER_ACCESS_UNORDERED_ACCESS      = 0x10,
    D3D12_BARRIER_ACCESS_DEPTH_STENCIL_WRITE   = 0x20,
    D3D12_BARRIER_ACCESS_DEPTH_STENCIL_READ    = 0x40,
    D3D12_BARRIER_ACCESS_SHADER_RESOURCE       = 0x80,
    D3D12_BARRIER_ACCESS_COPY_DEST             = 0x100,
    D3D12_BARRIER_ACCESS_COPY_SOURCE           = 0x200,
    D3D12_BARRIER_ACCESS_RESOLVE_DEST          = 0x400,
    D3D12_BARRIER_ACCESS_RESOLVE_SOURCE        = 0x800,
    // ...
    D3D12_BARRIER_ACCESS_NO_ACCESS             = 0x80000000,
} D3D12_BARRIER_ACCESS;
```

**レガシーの状態とよく似ています。** 移行時の対応は素直です。

**`NO_ACCESS` という特別な値があります。**

```
「このリソースには、しばらくアクセスしない」
```

**エイリアシング(同じメモリを複数のリソースで共有する)や、リソースの破棄前に使います。**

### 30.2.4 レイアウト

**これがレガシーになかった概念です。**

```cpp
typedef enum D3D12_BARRIER_LAYOUT {
    D3D12_BARRIER_LAYOUT_UNDEFINED              = 0xffffffff,
    D3D12_BARRIER_LAYOUT_COMMON                 = 0,
    D3D12_BARRIER_LAYOUT_PRESENT                = 0,
    D3D12_BARRIER_LAYOUT_GENERIC_READ           = 1,
    D3D12_BARRIER_LAYOUT_RENDER_TARGET          = 2,
    D3D12_BARRIER_LAYOUT_UNORDERED_ACCESS       = 3,
    D3D12_BARRIER_LAYOUT_DEPTH_STENCIL_WRITE    = 4,
    D3D12_BARRIER_LAYOUT_DEPTH_STENCIL_READ     = 5,
    D3D12_BARRIER_LAYOUT_SHADER_RESOURCE        = 6,
    D3D12_BARRIER_LAYOUT_COPY_SOURCE            = 7,
    D3D12_BARRIER_LAYOUT_COPY_DEST              = 8,
    D3D12_BARRIER_LAYOUT_RESOLVE_SOURCE         = 9,
    D3D12_BARRIER_LAYOUT_RESOLVE_DEST           = 10,
    // ...
} D3D12_BARRIER_LAYOUT;
```

**第19章 19.2.1 節で書いた内容を思い出してください。**

> GPU は 2 次元のアクセス効率を上げるため、ピクセルを独自の順序(スウィズル)で並べます。その並べ方はハードウェアごとに異なり、非公開です。

**レイアウトとは、この並べ方のことです。**

用途によって最適な並べ方が違うので、**切り替えるときに実際のメモリ変換が発生します。** レガシーバリアでは、これが状態遷移に暗黙的に含まれていました。

**分離したことで、2 つの利点が生まれます。**

**利点 1:レイアウトが同じなら、変換を省略できる**

```
SHADER_RESOURCE → SHADER_RESOURCE(アクセスだけ変更)
  → メモリ変換は不要
```

**利点 2:`UNDEFINED` で内容を捨てられる**

```cpp
barrier.LayoutBefore = D3D12_BARRIER_LAYOUT_UNDEFINED;
```

**「現在の内容は不要なので、変換せず破棄してよい」という指示です。**

**毎フレームクリアするレンダーターゲットでは、これが有効です。** 前フレームの内容を保持するための変換が省略されます。

**バッファにはレイアウトがありません。** バッファは常に線形なので、`D3D12_BARRIER_LAYOUT_UNDEFINED` を指定します。

---

## 30.3 対応を確認する

**第7章 7.5.3 節で、`OPTIONS12` を問い合わせるようにしました。**

```cpp
D3D12_FEATURE_DATA_D3D12_OPTIONS12 options12{};
if (QueryFeature(device, D3D12_FEATURE_D3D12_OPTIONS12, options12))
{
    m_caps.enhancedBarriers = options12.EnhancedBarriersSupported;
}
```

**そして第7章 7.5.5 節の `DeviceCaps` に保持しました。** ここで使います。

```cpp
LOG_INFO(L"enhanced barriers : {}",
         caps.enhancedBarriers ? L"yes" : L"no");
```

**Agility SDK 1.610 以降と、対応ドライバが必要です。** 第2章 2.1.2 節の表では、Turing 以降のすべての世代で「○」としていました。**ドライバが新しければ、対応しています。**

**インターフェースの取得も必要です。**

```cpp
Microsoft::WRL::ComPtr<ID3D12GraphicsCommandList7> commandList7;

if (SUCCEEDED(m_commandList.As(&commandList7)))
{
    // Enhanced Barriers が使える
}
```

**第6章 6.1.4 節で説明した、インターフェースのバージョンです。** `Barrier()` メソッドは `ID3D12GraphicsCommandList7` で追加されました。

---

## 30.4 レガシーとの対応

### 30.4.1 対応表

**移行するときの読み替え表です。**

| レガシーの状態 | Sync | Access | Layout |
|---|---|---|---|
| `COMMON` / `PRESENT` | `ALL` | `COMMON` | `COMMON` / `PRESENT` |
| `RENDER_TARGET` | `RENDER_TARGET` | `RENDER_TARGET` | `RENDER_TARGET` |
| `DEPTH_WRITE` | `DEPTH_STENCIL` | `DEPTH_STENCIL_WRITE` | `DEPTH_STENCIL_WRITE` |
| `DEPTH_READ` | `DEPTH_STENCIL` | `DEPTH_STENCIL_READ` | `DEPTH_STENCIL_READ` |
| `PIXEL_SHADER_RESOURCE` | `PIXEL_SHADING` | `SHADER_RESOURCE` | `SHADER_RESOURCE` |
| `NON_PIXEL_SHADER_RESOURCE` | `VERTEX_SHADING` / `COMPUTE_SHADING` | `SHADER_RESOURCE` | `SHADER_RESOURCE` |
| `UNORDERED_ACCESS` | `COMPUTE_SHADING` | `UNORDERED_ACCESS` | `UNORDERED_ACCESS` |
| `COPY_DEST` | `COPY` | `COPY_DEST` | `COPY_DEST` |
| `COPY_SOURCE` | `COPY` | `COPY_SOURCE` | `COPY_SOURCE` |
| `RESOLVE_DEST` | `RESOLVE` | `RESOLVE_DEST` | `RESOLVE_DEST` |
| `RESOLVE_SOURCE` | `RESOLVE` | `RESOLVE_SOURCE` | `RESOLVE_SOURCE` |
| `VERTEX_AND_CONSTANT_BUFFER` | `VERTEX_SHADING` | `VERTEX_BUFFER` / `CONSTANT_BUFFER` | (なし) |
| `INDEX_BUFFER` | `INDEX_INPUT` | `INDEX_BUFFER` | (なし) |

**`Sync` の列は「最も狭い範囲」を示しています。** 分からなければ `ALL` にしても動きますが、最適化の効果は減ります。

### 30.4.2 混在できる

**重要な点です。**

**Enhanced Barriers とレガシーバリアは、同じアプリケーション内で混在できます。**

**ただし、同じリソースに対しては、どちらか一方に統一する必要があります。**

```
✅ TextureA はレガシー、TextureB は Enhanced
❌ TextureA をレガシーで遷移し、次に Enhanced で遷移
```

**段階的に移行できます。** 一度に全部書き換える必要はありません。

**本書は、まず新しいコードで Enhanced を使い、既存部分を順に置き換える形にします。**

---

## 30.5 実装する

### 30.5.1 構造体

```cpp
typedef struct D3D12_TEXTURE_BARRIER {
    D3D12_BARRIER_SYNC        SyncBefore;
    D3D12_BARRIER_SYNC        SyncAfter;
    D3D12_BARRIER_ACCESS      AccessBefore;
    D3D12_BARRIER_ACCESS      AccessAfter;
    D3D12_BARRIER_LAYOUT      LayoutBefore;
    D3D12_BARRIER_LAYOUT      LayoutAfter;
    ID3D12Resource*           pResource;
    D3D12_BARRIER_SUBRESOURCE_RANGE Subresources;
    D3D12_TEXTURE_BARRIER_FLAGS     Flags;
} D3D12_TEXTURE_BARRIER;

typedef struct D3D12_BUFFER_BARRIER {
    D3D12_BARRIER_SYNC   SyncBefore;
    D3D12_BARRIER_SYNC   SyncAfter;
    D3D12_BARRIER_ACCESS AccessBefore;
    D3D12_BARRIER_ACCESS AccessAfter;
    ID3D12Resource*      pResource;
    UINT64               Offset;
    UINT64               Size;
} D3D12_BUFFER_BARRIER;
```

**バッファにレイアウトがない**ことが、構造体からも分かります。

**フィールドが増えました。** レガシーは 4 つでしたが、テクスチャバリアは 9 つです。**そのぶん、正確に指定できます。**

### 30.5.2 ヘルパーを書く

**第11章 11.5.4 節で `MakeTransitionBarrier` を作ったのと同じ発想です。**

```cpp
// src/Graphics/D3D12Helpers.h に追加

//---------------------------------------------------------------
// テクスチャバリア。
// CD3DX12_TEXTURE_BARRIER の代替。
//---------------------------------------------------------------
[[nodiscard]] inline D3D12_TEXTURE_BARRIER MakeTextureBarrier(
    ID3D12Resource*      resource,
    D3D12_BARRIER_SYNC   syncBefore,
    D3D12_BARRIER_SYNC   syncAfter,
    D3D12_BARRIER_ACCESS accessBefore,
    D3D12_BARRIER_ACCESS accessAfter,
    D3D12_BARRIER_LAYOUT layoutBefore,
    D3D12_BARRIER_LAYOUT layoutAfter,
    D3D12_TEXTURE_BARRIER_FLAGS flags =
        D3D12_TEXTURE_BARRIER_FLAG_NONE) noexcept
{
    D3D12_TEXTURE_BARRIER barrier{};
    barrier.SyncBefore   = syncBefore;
    barrier.SyncAfter    = syncAfter;
    barrier.AccessBefore = accessBefore;
    barrier.AccessAfter  = accessAfter;
    barrier.LayoutBefore = layoutBefore;
    barrier.LayoutAfter  = layoutAfter;
    barrier.pResource    = resource;
    barrier.Flags        = flags;

    //--- 全サブリソース ---
    barrier.Subresources.IndexOrFirstMipLevel = 0xFFFFFFFF;
    barrier.Subresources.NumMipLevels         = 0;
    barrier.Subresources.FirstArraySlice      = 0;
    barrier.Subresources.NumArraySlices       = 0;
    barrier.Subresources.FirstPlane           = 0;
    barrier.Subresources.NumPlanes            = 0;

    return barrier;
}

//---------------------------------------------------------------
// バッファバリア。
//---------------------------------------------------------------
[[nodiscard]] inline D3D12_BUFFER_BARRIER MakeBufferBarrier(
    ID3D12Resource*      resource,
    D3D12_BARRIER_SYNC   syncBefore,
    D3D12_BARRIER_SYNC   syncAfter,
    D3D12_BARRIER_ACCESS accessBefore,
    D3D12_BARRIER_ACCESS accessAfter) noexcept
{
    D3D12_BUFFER_BARRIER barrier{};
    barrier.SyncBefore   = syncBefore;
    barrier.SyncAfter    = syncAfter;
    barrier.AccessBefore = accessBefore;
    barrier.AccessAfter  = accessAfter;
    barrier.pResource    = resource;
    barrier.Offset       = 0;
    barrier.Size         = UINT64_MAX;      // 全体
    return barrier;
}
```

**`IndexOrFirstMipLevel = 0xFFFFFFFF` が「全サブリソース」を意味します。** レガシーの `D3D12_RESOURCE_BARRIER_ALL_SUBRESOURCES` に相当します。

**自作ヘルパーが 13 個になりました。**

### 30.5.3 バリアグループ

**Enhanced Barriers は、種類ごとにグループ化して発行します。**

```cpp
typedef struct D3D12_BARRIER_GROUP {
    D3D12_BARRIER_TYPE Type;
    UINT32             NumBarriers;
    union {
        const D3D12_GLOBAL_BARRIER*  pGlobalBarriers;
        const D3D12_TEXTURE_BARRIER* pTextureBarriers;
        const D3D12_BUFFER_BARRIER*  pBufferBarriers;
    };
} D3D12_BARRIER_GROUP;
```

**共用体です。** 第11章 11.5.2 節のレガシーバリアと同じ形ですが、**こちらは配列を指します。**

```cpp
//--- 複数のテクスチャバリアをまとめる ---
const D3D12_TEXTURE_BARRIER textureBarriers[] = {
    MakeTextureBarrier(sceneColor, ...),
    MakeTextureBarrier(shadowMap, ...),
};

D3D12_BARRIER_GROUP group{};
group.Type             = D3D12_BARRIER_TYPE_TEXTURE;
group.NumBarriers      = static_cast<UINT32>(std::size(textureBarriers));
group.pTextureBarriers = textureBarriers;

commandList7->Barrier(1, &group);
```

**第11章 11.5.4 節のコラムで「バリアはまとめて発行する」と書きました。** Enhanced Barriers では、**グループという形で明示的に構造化されています。**

**種類が違うものは、別グループにします。**

```cpp
const D3D12_BARRIER_GROUP groups[] = {
    { .Type = D3D12_BARRIER_TYPE_TEXTURE, .NumBarriers = 2,
      .pTextureBarriers = textureBarriers },
    { .Type = D3D12_BARRIER_TYPE_BUFFER,  .NumBarriers = 1,
      .pBufferBarriers = bufferBarriers },
};

commandList7->Barrier(2, groups);
```

**C++20 の指定イニシャライザを使うと読みやすくなります。**

### 30.5.4 状態追跡を書き換える

**第26章 26.1.4 節で作った `TransitionTo` を、Enhanced 版にします。**

```cpp
// src/Graphics/RenderTexture.h

struct BarrierState
{
    D3D12_BARRIER_SYNC   sync   = D3D12_BARRIER_SYNC_NONE;
    D3D12_BARRIER_ACCESS access = D3D12_BARRIER_ACCESS_NO_ACCESS;
    D3D12_BARRIER_LAYOUT layout = D3D12_BARRIER_LAYOUT_UNDEFINED;
};

class RenderTexture
{
public:
    // ...
    const BarrierState& State() const noexcept { return m_state; }
    void SetState(const BarrierState& state) noexcept { m_state = state; }

private:
    BarrierState m_state{};
};
```

```cpp
//---------------------------------------------------------------
// Enhanced Barriers による状態遷移。
//---------------------------------------------------------------
void TransitionTo(ID3D12GraphicsCommandList7* commandList,
                  RenderTexture& texture,
                  const BarrierState& newState,
                  bool discardContents = false)
{
    const auto& current = texture.State();

    //--- 何も変わらないなら省略 ---
    if (current.access == newState.access &&
        current.layout == newState.layout)
    {
        return;
    }

    const auto barrier = MakeTextureBarrier(
        texture.Get(),
        current.sync,
        newState.sync,
        current.access,
        newState.access,
        //--- 内容が不要なら UNDEFINED(30.2.4 節)---
        discardContents ? D3D12_BARRIER_LAYOUT_UNDEFINED : current.layout,
        newState.layout);

    D3D12_BARRIER_GROUP group{};
    group.Type             = D3D12_BARRIER_TYPE_TEXTURE;
    group.NumBarriers      = 1;
    group.pTextureBarriers = &barrier;

    commandList->Barrier(1, &group);

    texture.SetState(newState);
}
```

**よく使う状態を定数にしておきます。**

```cpp
namespace BarrierStates
{
    inline constexpr BarrierState kRenderTarget{
        D3D12_BARRIER_SYNC_RENDER_TARGET,
        D3D12_BARRIER_ACCESS_RENDER_TARGET,
        D3D12_BARRIER_LAYOUT_RENDER_TARGET,
    };

    inline constexpr BarrierState kPixelShaderResource{
        D3D12_BARRIER_SYNC_PIXEL_SHADING,
        D3D12_BARRIER_ACCESS_SHADER_RESOURCE,
        D3D12_BARRIER_LAYOUT_SHADER_RESOURCE,
    };

    inline constexpr BarrierState kDepthWrite{
        D3D12_BARRIER_SYNC_DEPTH_STENCIL,
        D3D12_BARRIER_ACCESS_DEPTH_STENCIL_WRITE,
        D3D12_BARRIER_LAYOUT_DEPTH_STENCIL_WRITE,
    };

    inline constexpr BarrierState kResolveSource{
        D3D12_BARRIER_SYNC_RESOLVE,
        D3D12_BARRIER_ACCESS_RESOLVE_SOURCE,
        D3D12_BARRIER_LAYOUT_RESOLVE_SOURCE,
    };

    inline constexpr BarrierState kResolveDest{
        D3D12_BARRIER_SYNC_RESOLVE,
        D3D12_BARRIER_ACCESS_RESOLVE_DEST,
        D3D12_BARRIER_LAYOUT_RESOLVE_DEST,
    };

    inline constexpr BarrierState kPresent{
        D3D12_BARRIER_SYNC_ALL,
        D3D12_BARRIER_ACCESS_COMMON,
        D3D12_BARRIER_LAYOUT_PRESENT,
    };
}
```

**使う側は、レガシー版とほぼ同じ形になります。**

```cpp
TransitionTo(m_commandList7.Get(), m_sceneColor,
             BarrierStates::kRenderTarget,
             true);   // ← 内容を破棄してよい
```

### 30.5.5 バックバッファの扱い

**第11章から書いてきたバックバッファのバリアも置き換えます。**

```cpp
//--- PRESENT → RENDER_TARGET ---
const auto toRenderTarget = MakeTextureBarrier(
    backBuffer,
    D3D12_BARRIER_SYNC_NONE,              // 最初のバリアなので待たない
    D3D12_BARRIER_SYNC_RENDER_TARGET,
    D3D12_BARRIER_ACCESS_NO_ACCESS,
    D3D12_BARRIER_ACCESS_RENDER_TARGET,
    D3D12_BARRIER_LAYOUT_UNDEFINED,       // ← 前フレームの内容は不要
    D3D12_BARRIER_LAYOUT_RENDER_TARGET);
```

**`SYNC_NONE` と `LAYOUT_UNDEFINED` の組み合わせが要点です。**

**第11章 11.1.2 節で書いた通り、FLIP_DISCARD ではバッファの内容が未定義になります。** だから前フレームの内容を保持する必要がありません。**`UNDEFINED` を指定すれば、レイアウト変換が省略されます。**

**戻すときは、`SYNC_NONE` を `SyncAfter` に指定します。**

```cpp
//--- RENDER_TARGET → PRESENT ---
const auto toPresent = MakeTextureBarrier(
    backBuffer,
    D3D12_BARRIER_SYNC_RENDER_TARGET,
    D3D12_BARRIER_SYNC_NONE,              // この後は何もしない
    D3D12_BARRIER_ACCESS_RENDER_TARGET,
    D3D12_BARRIER_ACCESS_NO_ACCESS,
    D3D12_BARRIER_LAYOUT_RENDER_TARGET,
    D3D12_BARRIER_LAYOUT_PRESENT);
```

**`SYNC_NONE` は「同期不要」を意味します。** `Present` は DXGI が別途同期するので、こちらで待つ必要がありません。

---

## 30.6 GPU-Based Validation

### 30.6.1 何を検証するか

**第7章 7.1.3 節で導入を保留したものを、ここで有効にします。**

> GPU-Based Validation(GBV)は、シェーダーの実行中にディスクリプタの妥当性やリソースの状態を検証します。**強力ですが、10〜100 倍遅くなります。**
> **本書の方針:設定で切り替えられるようにし、既定は無効。** 具体的なバグを追うときにだけ有効にします(第30章)。

**GBV が検出するもの:**

| 検証内容 | 例 |
|---|---|
| **リソースの状態が不正** | 読もうとした時点で `RENDER_TARGET` のまま |
| **デスクリプタが未初期化** | ヒープの位置に何も書かれていない |
| **デスクリプタの型が不一致** | SRV を期待している場所に CBV |
| **ルートシグネチャとの不一致** | 宣言していないレジスタを読んだ |
| **`DATA_STATIC` 違反** | 変わらないと宣言したデータを変更した |

**最後の項目は、第18章 18.4.3 節で警告した内容です。**

> **宣言と実際の挙動が食い違うと、静かに壊れます。** 「変わらない」と言っておいて変えると、古い値が使われることがあります。**GPU-Based Validation を有効にすると検出できます。**

**ここで実際に検出できるようになります。**

### 30.6.2 有効にする

**第7章 7.1.3 節で用意した設定を使います。**

```cpp
Graphics::GraphicsDevice::Config config{};
config.enableDebugLayer         = true;
config.enableGpuBasedValidation = true;   // ← 有効にする
```

```cpp
ComPtr<ID3D12Debug1> debug1;
if (SUCCEEDED(debug.As(&debug1)))
{
    debug1->SetEnableGPUBasedValidation(TRUE);
    debug1->SetEnableSynchronizedCommandQueueValidation(TRUE);
}
```

**`SetEnableSynchronizedCommandQueueValidation` も有効にします。** 複数キューを使う場合(第35章)の検証が加わります。

### 30.6.3 レガシーバリアの検証を強制する

**移行の途中で有用な設定があります。**

```cpp
ComPtr<ID3D12Debug6> debug6;
if (SUCCEEDED(debug.As(&debug6)))
{
    debug6->SetForceLegacyBarrierValidation(TRUE);
}
```

**Enhanced Barriers を使っていても、レガシーの規則で検証します。**

**なぜ有用か。** Enhanced Barriers はレガシーより緩い規則を持つため、**「Enhanced では通るが、レガシーでは不正」というコードが書けてしまいます。**

**古いドライバや他ベンダの環境で動かす可能性があるなら、この検証を通しておくと安全です。**

**本書は NVIDIA を前提としていますが、第2章 2.5 節で述べた通り「依存先」にはしていません。** この設定を有効にしておくと、移植性が保てます。

### 30.6.4 実行してみる

**GBV を有効にすると、劇的に遅くなります。**

```
通常:      3.92 ms
GBV 有効:  187.4 ms      ← 約 48 倍
```

**フレームレートは 5fps 程度になります。** デバッグ用途以外では使えません。

**エラーが出た場合の例です。**

```
D3D12 ERROR: GPU-BASED VALIDATION: Draw, Descriptor heap index out of bounds:
  Heap Index To DescriptorTableStart: [4], Descriptor Table Size: [2],
  Shader Stage: PIXEL, Root Parameter Index: [3], Draw Index: [127],
  Shader Code: Lit.hlsl(87), ...
```

**シェーダーのファイル名と行番号まで出ています。** 第13章 13.6 節で PDB を出力した成果です。

---

## 30.7 バリア不足を追う

### 30.7.1 「たまに壊れる」の正体

**バリアが不足していると、次のような症状が出ます。**

| 症状 | 説明 |
|---|---|
| **たまに絵が壊れる** | 前フレームの内容が混ざる |
| **環境によって結果が違う** | GPU のタイミングに依存 |
| **デバッガを通すと直る** | 実行が遅くなり、偶然間に合う |

**最後の項目が最悪です。** 「デバッグビルドでは再現しない」という状況になります。

**原因は、GPU が並列に処理しているからです。**

```
パス A: レンダーターゲットに書く
パス B: そのテクスチャを読む

バリアがない → B が A の完了前に実行されうる
```

### 30.7.2 検出する手段

**3 つの方法があります。**

**方法 1:GPU-Based Validation**

30.6 節の通りです。**状態が不正な時点で検出されます。**

**方法 2:Nsight Graphics で確認する**

第29章 29.2.2 節で、イベントリストにバリアが表示されることを見ました。**必要な位置にバリアがあるかを目視できます。**

**方法 3:意図的に遅延させる**

**デバッグ用に、パスの間に強制的な同期を挟みます。**

```cpp
#if defined(_DEBUG)
if (m_forceFullBarriers)
{
    //--- 全部待つ ---
    D3D12_GLOBAL_BARRIER globalBarrier{};
    globalBarrier.SyncBefore   = D3D12_BARRIER_SYNC_ALL;
    globalBarrier.SyncAfter    = D3D12_BARRIER_SYNC_ALL;
    globalBarrier.AccessBefore = D3D12_BARRIER_ACCESS_COMMON;
    globalBarrier.AccessAfter  = D3D12_BARRIER_ACCESS_COMMON;

    D3D12_BARRIER_GROUP group{};
    group.Type            = D3D12_BARRIER_TYPE_GLOBAL;
    group.NumBarriers     = 1;
    group.pGlobalBarriers = &globalBarrier;

    commandList7->Barrier(1, &group);
}
#endif
```

**これを有効にして絵が直るなら、バリア不足が原因です。**

**`D3D12_GLOBAL_BARRIER` は、すべてのリソースに対する同期です。** 非常に重いので、**デバッグ以外では使いません。**

### 30.7.3 過剰なバリアを見つける

**逆に、不要なバリアを見つけることもできます。**

**第29章 29.3.3 節で書いた通りです。**

> **すべてのユニットが低い**場合、GPU は「待って」います。原因は次のいずれかです。
> - **バリアが過剰**(第30章)

**GPU Trace で、バリアの前後に空白があれば過剰です。**

**Enhanced Barriers なら、`Sync` を狭めることで改善できます。**

```cpp
//--- 改善前:全部待つ ---
D3D12_BARRIER_SYNC_ALL

//--- 改善後:コピーの完了だけ待つ ---
D3D12_BARRIER_SYNC_COPY
```

---

## 30.8 分割バリア

### 30.8.1 待ち時間を隠す

**バリアには「開始」と「終了」を分離する機能があります。**

**レガシーでは `BEGIN_ONLY` / `END_ONLY` フラグでした**(第11章 11.5.2 節)。

**Enhanced Barriers では、`SYNC_SPLIT` を使います。**

```cpp
//--- 開始:「これから遷移する」と予告 ---
auto beginBarrier = MakeTextureBarrier(
    texture,
    D3D12_BARRIER_SYNC_RENDER_TARGET,
    D3D12_BARRIER_SYNC_SPLIT,             // ← 分割の開始
    D3D12_BARRIER_ACCESS_RENDER_TARGET,
    D3D12_BARRIER_ACCESS_SHADER_RESOURCE,
    D3D12_BARRIER_LAYOUT_RENDER_TARGET,
    D3D12_BARRIER_LAYOUT_SHADER_RESOURCE);

commandList7->Barrier(1, &beginGroup);

//--- 間に、関係のない処理を挟む ---
DrawSomethingElse();

//--- 終了:「ここで完了させる」 ---
auto endBarrier = MakeTextureBarrier(
    texture,
    D3D12_BARRIER_SYNC_SPLIT,             // ← 分割の終了
    D3D12_BARRIER_SYNC_PIXEL_SHADING,
    D3D12_BARRIER_ACCESS_RENDER_TARGET,
    D3D12_BARRIER_ACCESS_SHADER_RESOURCE,
    D3D12_BARRIER_LAYOUT_RENDER_TARGET,
    D3D12_BARRIER_LAYOUT_SHADER_RESOURCE);

commandList7->Barrier(1, &endGroup);
```

**間に挟んだ処理と、レイアウト変換が並行して進みます。**

### 30.8.2 いつ使うか

**本書のパイプラインでは、効果が限定的です。**

```
Resolve MSAA → Bloom → Composite
```

**各パスが直前の結果に依存しているので、隠せる時間がありません。**

**効果があるのは、独立した処理が並んでいる場合です。**

```
シャドウマップ生成 → [遷移開始]
不透明パス(シャドウマップを使わない部分)
[遷移終了] → シャドウマップを使う描画
```

**第35章でマルチスレッド化すると、より活用の場面が増えます。**

**本書では紹介に留めます。** 使わなくても正しく動きます。

---

## 30.9 UAV バリア

### 30.9.1 レガシーでの扱い

**第11章 11.5.2 節で触れた `D3D12_RESOURCE_BARRIER_TYPE_UAV` です。**

**「同じリソースへの書き込みが完了してから、次の読み書きを始める」ことを保証します。** 状態は変わりません。

```cpp
D3D12_RESOURCE_BARRIER barrier{};
barrier.Type          = D3D12_RESOURCE_BARRIER_TYPE_UAV;
barrier.UAV.pResource = resource;
```

### 30.9.2 Enhanced での書き方

**アクセスとレイアウトを変えず、同期だけを指定します。**

```cpp
auto barrier = MakeTextureBarrier(
    resource,
    D3D12_BARRIER_SYNC_COMPUTE_SHADING,
    D3D12_BARRIER_SYNC_COMPUTE_SHADING,
    D3D12_BARRIER_ACCESS_UNORDERED_ACCESS,
    D3D12_BARRIER_ACCESS_UNORDERED_ACCESS,
    D3D12_BARRIER_LAYOUT_UNORDERED_ACCESS,
    D3D12_BARRIER_LAYOUT_UNORDERED_ACCESS);
```

**前後が同じ値です。** これが UAV バリアに相当します。

**第32章でコンピュートシェーダーを扱うとき、これが必要になります。**

```
ディスパッチ 1: バッファに書く
[UAV バリア]
ディスパッチ 2: そのバッファを読む
```

**バリアがないと、ディスパッチ 2 が古い値を読む可能性があります。**

---

## ✅ 本章のゴール:バリアを整理する

### Step 1:対応を確認する

```
[Info ] GraphicsDevice.cpp(302): enhanced barriers : yes
[Info ] Renderer.cpp(88): using ID3D12GraphicsCommandList7
```

**`no` の場合、Agility SDK かドライバが古い可能性があります**(第4章 4.5 節)。

**本書のコードは、レガシー版も残しておくと安全です。**

```cpp
if (m_caps.enhancedBarriers && m_commandList7)
{
    TransitionEnhanced(...);
}
else
{
    TransitionLegacy(...);
}
```

### Step 2:1 つのリソースから移行する

**まず `m_sceneColor` だけを Enhanced に切り替えてください。**

**絵が変わらないことを確認します。**

**30.4.2 節の通り、同じリソースに対してレガシーと Enhanced を混ぜてはいけません。** 1 つずつ完全に移行します。

### Step 3:すべて移行する

**6 つのリソースすべてを Enhanced にします。**

| リソース | 状態遷移 |
|---|---|
| ShadowMap | `kDepthWrite ↔ kPixelShaderResource` |
| SceneColorMS | `kRenderTarget ↔ kResolveSource` |
| SceneColor | `kResolveDest ↔ kPixelShaderResource` |
| BloomTexture[0..2] | `kRenderTarget ↔ kPixelShaderResource` |
| BackBuffer | `kPresent ↔ kRenderTarget` |

### Step 4:`UNDEFINED` で内容を破棄する

**毎フレームクリアするリソースに、`discardContents = true` を指定してください。**

| リソース | 破棄してよいか |
|---|---|
| SceneColorMS | **よい**(毎フレームクリア) |
| BackBuffer | **よい**(FLIP_DISCARD) |
| ShadowMap | **よい**(毎フレームクリア) |
| BloomTexture | **よい**(毎フレーム上書き) |

**GPU Trace で、レイアウト変換の時間が減っているかを確認してください。**

**効果は環境によります。** 変わらない場合もありますが、**害はありません。**

### Step 5:同期スコープを狭める

**最初は `SYNC_ALL` で書いておき、段階的に狭めてください。**

```cpp
//--- 段階 1:動くことを確認 ---
D3D12_BARRIER_SYNC_ALL → D3D12_BARRIER_SYNC_ALL

//--- 段階 2:狭める ---
D3D12_BARRIER_SYNC_RENDER_TARGET → D3D12_BARRIER_SYNC_PIXEL_SHADING
```

**狭めすぎると壊れます。** そのときは GBV が検出してくれます。

### Step 6:GPU-Based Validation を有効にする

```cpp
config.enableGpuBasedValidation = true;
```

**極端に遅くなります。** 5fps 程度になるのが普通です。

**エラーが出なければ、バリアもデスクリプタも正しく設定されています。**

**確認したら無効に戻してください。**

### Step 7:わざとバリアを削る

**ブルームパスの入力バリアを削除してみてください。**

```cpp
// TransitionTo(cmdList, m_sceneColor, kPixelShaderResource);
```

**GBV を有効にすると、確実に検出されます。**

```
D3D12 ERROR: GPU-BASED VALIDATION: Draw, Incompatible resource state:
  Resource: 'SceneColor', Subresource Index: [0],
  Resource State: D3D12_RESOURCE_STATE_RESOLVE_DEST,
  Required State: D3D12_RESOURCE_STATE_PIXEL_SHADER_RESOURCE, ...
```

**リソース名が出ている**ことに注目してください(第6章 6.5 節)。

**GBV を無効にすると、多くの場合は「たまに壊れる」状態になります。** 30.7.1 節で説明した症状です。

**確認したら元に戻してください。**

### Step 8:強制的な全同期を試す

**30.7.2 節の方法 3 を実装し、切り替えてみてください。**

```cpp
if (input.WasKeyPressed('B'))
{
    m_forceFullBarriers = !m_forceFullBarriers;
}
```

**Step 7 の状態で有効にすると、絵が直るはずです。**

**これが「バリア不足を切り分ける」手段です。**

### Step 9:レガシー検証を有効にする

```cpp
debug6->SetForceLegacyBarrierValidation(TRUE);
```

**Enhanced Barriers でも、レガシーの規則で検証されます。**

**エラーが出た場合、それは「Enhanced では許されるが、レガシーでは不正」なコードです。** 移植性を考えるなら修正しておきます。

---

### 本章の達成状態

- [ ] `EnhancedBarriersSupported` を確認した
- [ ] `ID3D12GraphicsCommandList7` を取得した
- [ ] `MakeTextureBarrier` / `MakeBufferBarrier` を自作した
- [ ] `BarrierState` で 3 つの概念を保持している
- [ ] よく使う状態を定数化した
- [ ] すべてのリソースを Enhanced に移行した
- [ ] レガシーと Enhanced を混在させていない
- [ ] 毎フレームクリアするリソースに `UNDEFINED` を使った
- [ ] 同期スコープを必要最小限に狭めた
- [ ] GPU-Based Validation で検証した
- [ ] レガシー検証も通した
- [ ] 強制全同期の切り替えを実装した

---

## トラブルシューティング

| 症状 | 原因 | 対処 |
|---|---|---|
| `Barrier` メソッドがない | `ID3D12GraphicsCommandList7` 未取得 | 30.3 |
| 対応が `no` | Agility SDK かドライバが古い | 第4章 4.5 節 |
| バリアでエラー | レガシーと混在 | 統一する(30.4.2) |
| 絵がたまに壊れる | バリア不足 | GBV で検証(30.6) |
| 同上 | 同期スコープが狭すぎ | 広げる(30.5.2) |
| 極端に遅い | GBV が有効 | デバッグ時のみ使う(30.6.4) |
| すべてのユニットが遊んでいる | バリアが過剰 | `Sync` を狭める(30.7.3) |
| `DATA_STATIC` 違反 | 第18章 18.4.3 節 | GBV で検出される |
| レガシー検証でエラー | Enhanced 固有の書き方 | 移植性のため修正 |
| UAV の結果がおかしい | UAV バリアがない | 30.9 |

---

## まとめ

**1. レガシーバリアは 3 つの概念を混ぜていた。**
同期、アクセス、レイアウト。**分離できないため、必要以上の同期が発生していました。**

**2. Enhanced Barriers は「何を待つか」を指定できる。**
コピーの完了だけを待つ、といった細かい制御が可能です。

**3. レイアウトという概念が明示された。**
第19章 19.2.1 節で触れた「GPU 独自の並べ方」が、型として現れました。**同じレイアウトなら変換を省略できます。**

**4. `UNDEFINED` で内容を破棄できる。**
毎フレームクリアするリソースでは、前フレームの内容を保持する変換が不要です。

**5. 混在できるが、リソース単位で統一する。**
段階的な移行が可能です。1 つずつ完全に置き換えてください。

**6. GPU-Based Validation が最後の砦。**
状態の不正、デスクリプタの誤り、`DATA_STATIC` 違反。**第18章 18.4.3 節で予告した検証が、ここで実際に働きます。**

**7. 「たまに壊れる」はバリア不足を疑う。**
デバッガを通すと直る、環境によって違う —— **典型的な症状です。** 強制全同期で切り分けられます。

次章では、いよいよ Aftermath を本格的に使います。**第8章で組み込んだ最小構成に、イベントマーカーとシェーダーデバッグ情報を追加し、意図的にクラッシュさせて原因を HLSL の行番号まで特定します。** 第13章 13.6 節で PDB を出力した投資が、そこで最大の見返りを生みます。

---

## 参考リンク

| 内容 | URL |
|---|---|
| Enhanced Barriers 仕様 | https://microsoft.github.io/DirectX-Specs/d3d/D3D12EnhancedBarriers.html |
| `D3D12_BARRIER_SYNC` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/ne-d3d12-d3d12_barrier_sync |
| `D3D12_BARRIER_ACCESS` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/ne-d3d12-d3d12_barrier_access |
| `D3D12_BARRIER_LAYOUT` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/ne-d3d12-d3d12_barrier_layout |
| `ID3D12GraphicsCommandList7::Barrier` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/nf-d3d12-id3d12graphicscommandlist7-barrier |
| GPU-Based Validation | https://learn.microsoft.com/ja-jp/windows/win32/direct3d12/using-d3d12-debug-layer-gpu-based-validation |
