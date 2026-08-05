# 第33章 バインドレス(Shader Model 6.6)

第25章 25.3 節で、ドローコールの削減に取り組みました。

```
ソートなし:  material 398 回の切り替え
ソートあり:  material   8 回
```

**大幅に改善しましたが、切り替えそのものは残っています。** マテリアルが増えれば、それに比例して切り替えも増えます。

**本章では、この切り替えを消します。**

```
バインドレス:  material 0 回
```

**発想は単純です。すべてのテクスチャをヒープに置いたまま、シェーダーに「何番を使え」と番号だけを渡します。**

そして、**第21章 21.1.3 節で予告した内容が、ここで回収されます。**

> **`index` を持たせている点に注目してください。** CPU / GPU ハンドルだけでも描画はできますが、**第33章のバインドレスでは、この番号そのものをシェーダーに渡します。** そのときのために用意しておきます。

**本章のゴール**
Shader Model 6.6 の Dynamic Resources を使い、ディスクリプタテーブルを廃止する。マテリアルの切り替えをゼロにする。

---

## 33.1 これまでの方式の限界

### 33.1.1 何が問題か

**第20章 20.5 節で、テクスチャを SRV としてバインドしました。**

```cpp
m_commandList->SetGraphicsRootDescriptorTable(
    3, m_materials[object->materialIndex].srvHandle);
```

**この 1 行が、マテリアルごとに必要です。**

**問題は 3 つあります。**

| 問題 | 説明 |
|---|---|
| **切り替えのコスト** | 描画のたびに GPU の状態が変わる |
| **テーブルの構造が固定** | 「SRV が 2 つ」と決めたら変えられない |
| **数が可変にできない** | マテリアルによってテクスチャ数が違う場合に困る |

**3 番目が本質的です。**

```
マテリアル A:  ディフューズのみ
マテリアル B:  ディフューズ + 法線マップ
マテリアル C:  ディフューズ + 法線 + 粗さ + AO
```

**従来の方式では、最大数に合わせたテーブルを作り、使わない枠にダミーを入れる**ことになります。

### 33.1.2 Resource Binding Tier

**第7章 7.5.3 節で問い合わせた `ResourceBindingTier` が、ここで意味を持ちます。**

| Tier | ヒープあたりの CBV_SRV_UAV | テーブルあたりのディスクリプタ |
|---|---|---|
| Tier 1 | 1,000,000 | **制限あり** |
| Tier 2 | 1,000,000 | 制限緩和 |
| **Tier 3** | **1,000,000** | **無制限** |

**Tier 3 では、ディスクリプタテーブルのサイズが実行時に決まります。**

```cpp
range.NumDescriptors = UINT_MAX;   // 無制限
```

**これが「バインドレス」の第一世代です。** Shader Model 5.1 の頃から可能でした。

```hlsl
Texture2D gTextures[] : register(t0);   // サイズ未指定の配列

float4 PSMain(...) : SV_Target
{
    return gTextures[materialIndex].Sample(gSampler, uv);
}
```

**これでも動きます。** ただし、テーブルのバインドは依然として必要です。

### 33.1.3 Shader Model 6.6 の Dynamic Resources

**Shader Model 6.6 で、テーブルすら不要になりました。**

```hlsl
Texture2D texture = ResourceDescriptorHeap[textureIndex];
float4 color = texture.Sample(sampler, uv);
```

**`ResourceDescriptorHeap` は、バインドされているヒープそのものを表します。**

**宣言も、レジスタの割り当ても、テーブルも要りません。** インデックスを渡すだけです。

| 世代 | 必要なもの |
|---|---|
| 従来 | ルートシグネチャの宣言 + テーブルのバインド |
| Tier 3 バインドレス | ルートシグネチャの宣言 + テーブルのバインド(1 回) |
| **Dynamic Resources** | **フラグの指定のみ** |

**本章では Dynamic Resources を使います。**

---

## 33.2 対応を確認する

### 33.2.1 必要な条件

**第7章 7.5.5 節で作った `DeviceCaps::SupportsBindless()` を思い出してください。**

```cpp
bool DeviceCaps::SupportsBindless() const noexcept
{
    return resourceBindingTier >= D3D12_RESOURCE_BINDING_TIER_3
        && maxShaderModel      >= D3D_SHADER_MODEL_6_6;
}
```

**2 つの条件が必要です。**

| 条件 | 確認方法 |
|---|---|
| Resource Binding Tier 3 | `D3D12_FEATURE_D3D12_OPTIONS` |
| **Shader Model 6.6 以上** | `D3D12_FEATURE_SHADER_MODEL` |

**第2章 2.1.2 節の表では、Turing 以降で「○」としていました。** ドライバが新しければ対応しています。

**シェーダーのコンパイルにも要件があります。**

```
dxc -T ps_6_6 ...    ← 6.6 以上
```

**第13章 13.1.2 節で、本書のベースラインを `6_6` にした理由がこれです。**

> 第33章のバインドレス(Dynamic Resources)が 6.6 を要求する

### 33.2.2 起動時に確認する

```cpp
if (!m_caps.SupportsBindless())
{
    LOG_WARN(L"bindless is not supported. falling back to descriptor tables.");
    LOG_WARN(L"  binding tier : {}", static_cast<int>(m_caps.resourceBindingTier));
    LOG_WARN(L"  shader model : 6.{}",
             static_cast<int>(m_caps.maxShaderModel) & 0xF);
    m_useBindless = false;
}
```

**フォールバックを用意するかは、判断が分かれます。**

**本書は用意します。** 理由は 2 つです。

- **対応の有無で絵が変わらないことを確認できる**(検証手段になる)
- 第2章 2.5 節の方針(NVIDIA は「対象」であって「依存先」ではない)

**ただし、シェーダーは 2 種類必要になります。** 保守のコストは無視できません。

**実務では「6.6 必須」と割り切る選択も十分にありえます。**

---

## 33.3 ルートシグネチャを変える

### 33.3.1 フラグを追加する

**Dynamic Resources を使うには、2 つのフラグが必要です。**

```cpp
versioned.Desc_1_1.Flags =
      D3D12_ROOT_SIGNATURE_FLAG_ALLOW_INPUT_ASSEMBLER_INPUT_LAYOUT
    | D3D12_ROOT_SIGNATURE_FLAG_CBV_SRV_UAV_HEAP_DIRECTLY_INDEXED   // ← 追加
    | D3D12_ROOT_SIGNATURE_FLAG_SAMPLER_HEAP_DIRECTLY_INDEXED       // ← 追加
    | D3D12_ROOT_SIGNATURE_FLAG_DENY_HULL_SHADER_ROOT_ACCESS
    | D3D12_ROOT_SIGNATURE_FLAG_DENY_DOMAIN_SHADER_ROOT_ACCESS
    | D3D12_ROOT_SIGNATURE_FLAG_DENY_GEOMETRY_SHADER_ROOT_ACCESS;
```

| フラグ | 有効にするもの |
|---|---|
| `CBV_SRV_UAV_HEAP_DIRECTLY_INDEXED` | **`ResourceDescriptorHeap`** |
| `SAMPLER_HEAP_DIRECTLY_INDEXED` | `SamplerDescriptorHeap` |

**サンプラー側は、本書では使いません。** 第20章 20.5.2 節で静的サンプラーを選んだためです。**それでもフラグは指定しておきます。** 後で使いたくなったとき、ルートシグネチャを作り直さずに済みます。

### 33.3.2 パラメータが激減する

**第24章 24.6.2 節のルートシグネチャは、こうでした。**

```cpp
params[0]: b0 (CBV)  シーン定数
params[1]: b1 (CBV)  オブジェクト定数
params[2]: b2 (CBV)  マテリアル定数
params[3]: t0 (テーブル)  テクスチャ
```

**バインドレスでは、テーブルが消えます。**

```cpp
D3D12_ROOT_PARAMETER1 params[3]{};

//--- b0: シーン定数 ---
params[0].ParameterType = D3D12_ROOT_PARAMETER_TYPE_CBV;
params[0].Descriptor.ShaderRegister = 0;
params[0].ShaderVisibility = D3D12_SHADER_VISIBILITY_ALL;

//--- b1: オブジェクト定数 ---
params[1].ParameterType = D3D12_ROOT_PARAMETER_TYPE_CBV;
params[1].Descriptor.ShaderRegister = 1;
params[1].ShaderVisibility = D3D12_SHADER_VISIBILITY_ALL;

//--- b2: マテリアル情報(インデックスを含む)---
params[2].ParameterType = D3D12_ROOT_PARAMETER_TYPE_CBV;
params[2].Descriptor.ShaderRegister = 2;
params[2].ShaderVisibility = D3D12_SHADER_VISIBILITY_PIXEL;
```

**予算の確認をします**(第14章 14.1.3 節)。

| | 従来 | **バインドレス** |
|---|---|---|
| ルートディスクリプタ × 3 | 6 DWORD | 6 DWORD |
| ディスクリプタテーブル × 1 | 1 DWORD | **0** |
| **合計** | **7 DWORD** | **6 DWORD** |

**わずか 1 DWORD の節約に見えます。**

**しかし、本質はそこではありません。** テクスチャの種類がいくら増えても、**予算が増えないこと**が重要です。

```
法線マップを追加     → 従来:テーブルを拡張  / バインドレス:変更なし
粗さマップを追加     → 従来:テーブルを拡張  / バインドレス:変更なし
マテリアルごとに別々 → 従来:困難            / バインドレス:自然
```

### 33.3.3 ルート定数という選択肢

**マテリアル定数バッファすら不要にできます。**

```cpp
//--- b2 をルート定数にする ---
params[2].ParameterType             = D3D12_ROOT_PARAMETER_TYPE_32BIT_CONSTANTS;
params[2].Constants.ShaderRegister  = 2;
params[2].Constants.Num32BitValues  = 4;   // インデックス 4 個
params[2].ShaderVisibility          = D3D12_SHADER_VISIBILITY_PIXEL;
```

```cpp
const std::uint32_t indices[4] = {
    material.diffuseTextureIndex,
    material.normalTextureIndex,
    material.roughnessTextureIndex,
    material.materialDataIndex,
};

m_commandList->SetGraphicsRoot32BitConstants(2, 4, indices, 0);
```

**第18章 18.4.1 節で「ルート定数は最速」と書きました。** 4 DWORD なら、予算にも余裕があります。

**ただし、マテリアルのパラメータ(色、粗さの値など)は別途必要です。** 本書は定数バッファ方式を採り、その中にインデックスを含めます。

---

## 33.4 シェーダーを書く

### 33.4.1 `ResourceDescriptorHeap`

**最も単純な形です。**

```hlsl
cbuffer MaterialConstants : register(b2)
{
    float4 diffuseColor;
    float4 specularColor;
    uint   diffuseTextureIndex;
    uint   normalTextureIndex;
    uint   roughnessTextureIndex;
    uint   padding;
};

SamplerState gSampler : register(s0);   // 静的サンプラー(第20章)

float4 PSMain(VSOutput input) : SV_Target
{
    //--- ヒープから直接取り出す ---
    Texture2D diffuseTexture = ResourceDescriptorHeap[diffuseTextureIndex];

    const float4 albedo = diffuseTexture.Sample(gSampler, input.uv);

    // ...
}
```

**宣言が要りません。** `register(t0)` も、ルートシグネチャでの対応づけも不要です。

### 33.4.2 型は代入時に決まる

**`ResourceDescriptorHeap[i]` は、型を持ちません。**

```hlsl
Texture2D          tex    = ResourceDescriptorHeap[index];
StructuredBuffer<X> buf   = ResourceDescriptorHeap[index];
RWTexture2D<float4> rwTex = ResourceDescriptorHeap[index];
```

**代入先の型で解釈が決まります。**

**型を間違えても、コンパイルは通ります。** 実行時に不正な結果になるか、クラッシュします。

> **GPU-Based Validation が検出する**
>
> 第30章 30.6.1 節で挙げた検証内容に、これが含まれます。
>
> | 検証内容 | 例 |
> |---|---|
> | **デスクリプタの型が不一致** | SRV を期待している場所に CBV |
>
> **バインドレスでは、この誤りが起こりやすくなります。** 型の対応がコード上で見えないためです。
>
> **GBV を定期的に走らせることが、より重要になります。**

### 33.4.3 非一様インデックス

**警告が出ることがあります。**

```
warning: Resource index is not uniform across the wave
```

**同じ warp 内のスレッドが、異なるインデックスを使う場合です。**

**ピクセルシェーダーでは、これが普通に起こります。** 隣り合うピクセルが別のマテリアルを持つことがあるからです。

**`NonUniformResourceIndex` で明示します。**

```hlsl
Texture2D texture = ResourceDescriptorHeap[
    NonUniformResourceIndex(materialIndex)];
```

**これがないと、未定義動作になります。**

**GPU は「warp 内で同じインデックス」を仮定して最適化します。** 実際には違う場合、**warp の代表の値が全員に使われる**ことがあります。

```
スレッド 0: インデックス 5 を要求
スレッド 1: インデックス 8 を要求
   → 両方ともインデックス 5 のテクスチャを読む(誤り)
```

**症状は「一部のピクセルだけ違うテクスチャが貼られる」です。** 原因の特定が非常に困難です。

> **いつ必要か**
>
> | 場面 | 必要性 |
> |---|---|
> | 定数バッファから読んだインデックス | **不要**(全スレッドで同じ) |
> | 頂点属性から補間されたインデックス | **必要** |
> | 計算で求めたインデックス | **必要** |
> | インスタンス ID から引いたインデックス | **必要**(第34章) |
>
> **迷ったら付けてください。** 不要な場合に付けても、多くの環境で最適化により消えます。

**第2章 2.3.1 節で説明したダイバージェンスと、同じ根を持つ問題です。**

### 33.4.4 定数バッファも同様に扱える

```hlsl
struct MaterialData
{
    float4 diffuseColor;
    float4 specularColor;
    uint   diffuseTextureIndex;
    uint   normalTextureIndex;
    uint   roughnessTextureIndex;
    uint   flags;
};

//--- 構造化バッファとして全マテリアルを保持 ---
StructuredBuffer<MaterialData> gMaterials =
    ResourceDescriptorHeap[materialBufferIndex];

float4 PSMain(VSOutput input) : SV_Target
{
    const MaterialData material = gMaterials[input.materialIndex];

    Texture2D diffuseTexture = ResourceDescriptorHeap[
        NonUniformResourceIndex(material.diffuseTextureIndex)];

    // ...
}
```

**マテリアル情報そのものも、GPU 上のバッファに置けます。**

**これが「究極形」です。** ルートシグネチャに渡すのは、**シーン定数とマテリアルバッファのインデックスだけ**になります。

**第34章の GPU カリングでは、この形が前提になります。**

---

## 33.5 実装する

### 33.5.1 インデックスを管理する

**第21章 21.1.3 節の `DescriptorHandle` に、既に `index` があります。**

```cpp
struct DescriptorHandle
{
    D3D12_CPU_DESCRIPTOR_HANDLE cpu{};
    D3D12_GPU_DESCRIPTOR_HANDLE gpu{};
    UINT index = kInvalidIndex;    // ← これを使う
};
```

**インデックスは、ヒープの先頭からの通し番号です。**

```cpp
const UINT global = m_baseIndex + local;
handle.index = global;
```

**この番号を、そのままシェーダーに渡します。**

```cpp
struct Material
{
    // ...
    std::uint32_t diffuseTextureIndex   = kInvalidDescriptorIndex;
    std::uint32_t normalTextureIndex    = kInvalidDescriptorIndex;
    std::uint32_t roughnessTextureIndex = kInvalidDescriptorIndex;
};
```

```cpp
//--- テクスチャを読み込んだとき ---
auto texture = uploader.UploadTexture(image, name);
const auto srv = m_descriptorHeap.Static().Allocate();

device->CreateShaderResourceView(texture->Get(), &srvDesc, srv.cpu);

material.diffuseTextureIndex = srv.index;   // ← 番号を保存
```

**`gpu` ハンドルを使う必要がなくなりました。**

### 33.5.2 無効なインデックスへの対処

**テクスチャがないマテリアルもあります。**

```cpp
inline constexpr std::uint32_t kInvalidDescriptorIndex = 0xFFFFFFFF;
```

**シェーダーで分岐する方法もありますが、非推奨です。**

```hlsl
//--- △ 分岐は避けたい ---
float4 albedo = float4(1, 1, 1, 1);
if (material.diffuseTextureIndex != 0xFFFFFFFF)
{
    Texture2D tex = ResourceDescriptorHeap[
        NonUniformResourceIndex(material.diffuseTextureIndex)];
    albedo = tex.Sample(gSampler, input.uv);
}
```

**warp 内で分岐すると、両方の経路が実行されます**(第2章 2.3.1 節)。

**ダミーテクスチャを用意するほうが優れています。**

```cpp
//--- 起動時に、1×1 の白テクスチャを作る ---
Core::Status CreateDefaultTextures(ResourceUploader& uploader)
{
    //--- 白(ディフューズ用)---
    constexpr std::uint32_t whitePixel = 0xFFFFFFFF;
    m_defaultWhite = CreateSolidColorTexture(uploader, whitePixel, L"DefaultWhite");

    //--- 法線マップ用(0.5, 0.5, 1.0 = 真上を向く)---
    constexpr std::uint32_t flatNormal = 0xFFFF8080;
    m_defaultNormal = CreateSolidColorTexture(uploader, flatNormal, L"DefaultNormal");

    //--- 黒(発光用など)---
    constexpr std::uint32_t blackPixel = 0xFF000000;
    m_defaultBlack = CreateSolidColorTexture(uploader, blackPixel, L"DefaultBlack");

    return {};
}
```

```cpp
//--- マテリアル読み込み時に、なければ既定値を入れる ---
material.diffuseTextureIndex = diffuseSrv.IsValid()
    ? diffuseSrv.index
    : m_defaultWhiteIndex;
```

**分岐が消え、シェーダーが単純になります。**

**メモリのコストは、1×1 テクスチャ 3 枚ぶんです。** 無視できます。

### 33.5.3 描画コード

**第25章 25.3.2 節のループが、こうなります。**

```cpp
//--- ループの前に 1 回だけ ---
ID3D12DescriptorHeap* heaps[] = { m_descriptorHeap.Get() };
m_commandList->SetDescriptorHeaps(1, heaps);
m_commandList->SetGraphicsRootSignature(m_rootSignature.Get());
m_commandList->SetGraphicsRootConstantBufferView(0, sceneConstantsAddress);

std::uint32_t lastPso  = 0xFFFFFFFF;
std::uint32_t lastMesh = 0xFFFFFFFF;

for (const RenderObject* object : visible)
{
    if (object->psoIndex != lastPso)
    {
        m_commandList->SetPipelineState(m_psos[object->psoIndex].Get());
        lastPso = object->psoIndex;
    }

    if (object->meshIndex != lastMesh)
    {
        const auto& mesh = m_meshes[object->meshIndex];
        m_commandList->IASetVertexBuffers(0, 1, &mesh.vbv);
        m_commandList->IASetIndexBuffer(&mesh.ibv);
        lastMesh = object->meshIndex;
    }

    //--- ★ マテリアルの切り替えが消えた ★ ---

    //--- オブジェクト定数 ---
    const auto objectAlloc =
        m_uploadBuffer.AllocateConstants(sizeof(ObjectConstants));
    objectAlloc.Write(BuildObjectConstants(*object));
    m_commandList->SetGraphicsRootConstantBufferView(1, objectAlloc.gpuAddress);

    //--- マテリアル定数(インデックスを含む)---
    const auto materialAlloc =
        m_uploadBuffer.AllocateConstants(sizeof(MaterialConstants));
    materialAlloc.Write(m_materials[object->materialIndex].ToConstants());
    m_commandList->SetGraphicsRootConstantBufferView(2, materialAlloc.gpuAddress);

    m_commandList->DrawIndexedInstanced(mesh.indexCount, 1, 0, 0, 0);
}
```

**`SetGraphicsRootDescriptorTable` が完全に消えました。**

**ソートの優先順位も変わります。**

```cpp
std::ranges::sort(opaque, [](const auto* a, const auto* b)
{
    if (a->psoIndex  != b->psoIndex)  return a->psoIndex  < b->psoIndex;
    if (a->meshIndex != b->meshIndex) return a->meshIndex < b->meshIndex;

    //--- マテリアルでソートする理由がなくなった ---
    return a->distanceToCamera < b->distanceToCamera;   // 手前から
});
```

**第25章 25.5 節で「状態と距離のどちらを優先するか」という問題がありました。**

**マテリアルが状態から外れたので、距離ソートの優先度を上げられます。** Early-Z がより効くようになります(第19章 19.1.3 節)。

### 33.5.4 ヒープを 1 つに統一する

**第21章 21.1.2 節の設計が、ここで完成します。**

> **起動時に十分大きなヒープを 1 つ作り、その中を自分で切り分けて使う。**

**バインドレスでは、これが必然になります。**

```
CBV_SRV_UAV ヒープ(8192 個)
┌─────────────────────┬──────────────────────────────┐
│ 永続領域(2048)     │ 一時領域(2048 × 3 フレーム)  │
│                      │                               │
│ [0]   DefaultWhite   │ 毎フレームの CBV など          │
│ [1]   DefaultNormal  │                               │
│ [2]   DefaultBlack   │                               │
│ [3]   Texture_A      │                               │
│ [4]   Texture_B      │                               │
│ ...                  │                               │
└─────────────────────┴──────────────────────────────┘
        ↑
   この番号がシェーダーへ渡る
```

**永続領域のインデックスが、そのままマテリアルに保存されます。**

**一時領域は、フレームごとに巻き戻されます**(第21章 21.1.4 節)。**バインドレスで参照するのは永続領域だけです。**

---

## 33.6 デバッグが難しくなる

### 33.6.1 何が見えなくなるか

**バインドレスには代償があります。**

| | 従来 | **バインドレス** |
|---|---|---|
| ルートシグネチャ | **どのリソースを使うか宣言されている** | 宣言がない |
| Nsight Graphics | **バインドされたリソースが一覧表示** | ヒープ全体しか見えない |
| コンパイル時の検証 | 型の対応がチェックされる | **されない** |

**第29章 29.2.5 節で、Pipeline ビューを使いました。**

> `Pipeline` ビューで、そのドローコール時点の全設定が見られます。
> **バインドされたリソース**

**バインドレスでは、この情報が減ります。** 「ヒープ全体がバインドされている」としか分かりません。

### 33.6.2 対策

**3 つの手段があります。**

**対策 1:デバッグ名を徹底する**

**第6章 6.5 節の習慣が、ここでさらに重要になります。**

**Nsight Graphics でヒープの中身を見るとき、名前がなければ何が入っているか分かりません。**

```
Descriptor Heap: MainDescriptorHeap
  [0]  SRV → 'DefaultWhite'
  [1]  SRV → 'DefaultNormal'
  [2]  SRV → 'DefaultBlack'
  [3]  SRV → 'Brick_Diffuse'
  [4]  SRV → 'Brick_Normal'
```

**名前があれば、インデックスと実体の対応が追えます。**

**対策 2:インデックスをログに出す**

```cpp
void LogDescriptorAssignment(std::wstring_view name, UINT index)
{
    LOG_TRACE(L"descriptor [{}] = {}", index, name);
}
```

```
[Trace] Renderer.cpp(288): descriptor [3] = Brick_Diffuse
[Trace] Renderer.cpp(288): descriptor [4] = Brick_Normal
```

**クラッシュ時に、どのインデックスが何を指していたかを追跡できます。**

**対策 3:GPU-Based Validation を定期的に走らせる**

**第30章 30.6 節で導入したものです。**

**型の不一致や、無効なインデックスを検出できます。**

```
D3D12 ERROR: GPU-BASED VALIDATION: Draw, Descriptor heap index out of bounds:
  Heap Index: [8500], Heap Size: [8192],
  Shader Stage: PIXEL, Shader Code: Lit.hlsl(64)
```

**バインドレスでは、GBV の重要度が上がります。**

### 33.6.3 Aftermath での見え方

**第31章 31.4.2 節で、ページフォルト時にリソース名が表示されることを確認しました。**

**バインドレスでも、これは変わりません。**

```
Page Fault:
  GPU Virtual Address: 0x0000000204A00000
  Resource: 'Brick_Diffuse'
```

**リソース自体は名前を持っているので、特定できます。**

**ただし、「なぜそのリソースにアクセスしたのか」は分かりにくくなります。** インデックスの誤りが原因の場合、**どのマテリアルが誤ったインデックスを持っていたかを、別途調べる必要があります。**

**対策 2 のログが、ここで役立ちます。**

---

## 33.7 フォールバックを用意する

### 33.7.1 シェーダーを 2 種類作る

**33.2.2 節でフォールバックを用意すると決めました。**

**プリプロセッサで切り替えます。**

```hlsl
//=====================================================
// shaders/Lit.hlsl
//=====================================================

#ifndef USE_BINDLESS
    #define USE_BINDLESS 1
#endif

cbuffer MaterialConstants : register(b2)
{
    float4 diffuseColor;
    float4 specularColor;
    uint   diffuseTextureIndex;
    uint   normalTextureIndex;
    uint   roughnessTextureIndex;
    uint   materialFlags;
};

SamplerState gSampler : register(s0);

#if !USE_BINDLESS
    //--- 従来方式:テーブルでバインド ---
    Texture2D gDiffuseTexture   : register(t0);
    Texture2D gNormalTexture    : register(t1);
    Texture2D gRoughnessTexture : register(t2);
#endif

//-----------------------------------------------------
// テクスチャの取得を抽象化する
//-----------------------------------------------------
float4 SampleDiffuse(float2 uv)
{
#if USE_BINDLESS
    Texture2D texture = ResourceDescriptorHeap[
        NonUniformResourceIndex(diffuseTextureIndex)];
    return texture.Sample(gSampler, uv);
#else
    return gDiffuseTexture.Sample(gSampler, uv);
#endif
}

float3 SampleNormal(float2 uv)
{
#if USE_BINDLESS
    Texture2D texture = ResourceDescriptorHeap[
        NonUniformResourceIndex(normalTextureIndex)];
    return texture.Sample(gSampler, uv).xyz;
#else
    return gNormalTexture.Sample(gSampler, uv).xyz;
#endif
}

//-----------------------------------------------------
// 本体は共通
//-----------------------------------------------------
float4 PSMain(VSOutput input) : SV_Target
{
    const float4 albedo = SampleDiffuse(input.uv);
    // ...
}
```

**関数で抽象化することで、本体のコードが 1 つで済みます。**

### 33.7.2 ビルド設定

**第13章 13.7.2 節のカスタムビルドツールに、2 つの出力を追加します。**

```
"$(DxcExe)" -T ps_6_6 -E PSMain -D USE_BINDLESS=1 ^
    -Fo "$(ShaderOutDir)Lit.PS.bindless.cso" ...

"$(DxcExe)" -T ps_6_0 -E PSMain -D USE_BINDLESS=0 ^
    -Fo "$(ShaderOutDir)Lit.PS.fallback.cso" ...
```

**フォールバック版はシェーダーモデル 6.0 でコンパイルできます。** より広い環境で動きます。

**PSO も 2 つ必要です。**

```cpp
const auto& psName = m_useBindless
    ? L"Lit.PS.bindless.cso"
    : L"Lit.PS.fallback.cso";
```

### 33.7.3 保守のコストを認識する

**正直に書いておきます。フォールバックの保守は面倒です。**

- シェーダーを変更するたび、両方で動作確認が必要
- ビルド時間が倍になる
- ルートシグネチャも 2 種類必要

**「6.6 必須」と割り切る選択も、十分に合理的です。**

**判断基準を示します。**

| 状況 | 推奨 |
|---|---|
| 学習・実験 | **バインドレスのみ** |
| 対象環境を限定できる | **バインドレスのみ** |
| 幅広い環境に配布する | フォールバックを用意 |
| 既存プロジェクトへの導入 | 段階的に移行 |

**本書はフォールバックを示しましたが、以降の章では簡潔さのためバインドレス版のみを扱います。**

---

## ✅ 本章のゴール:状態切り替え回数が激減する

### Step 1:対応を確認する

```
[Info ] GraphicsDevice.cpp(293): binding tier  : 3
[Info ] GraphicsDevice.cpp(289): shader model  : 6.9
[Info ] GraphicsDevice.cpp(310): ch.33 bindless      : OK
```

**第7章 7.5.5 節の判定関数が、ここで使われます。**

### Step 2:統計を比較する

**第25章 25.3.3 節で作った統計を使います。**

```
従来方式:
  objects 336/336, draws 336, pso 2, mesh 3, material 8

バインドレス:
  objects 336/336, draws 336, pso 2, mesh 3, material 0
```

**マテリアル切り替えがゼロになりました。**

**マテリアルの種類を増やして、比較してみてください。**

| マテリアル数 | 従来 | バインドレス |
|---|---|---|
| 3 | 8 回 | **0 回** |
| 20 | 45 回 | **0 回** |
| 100 | 200 回以上 | **0 回** |

**バインドレスでは、マテリアル数に依存しません。**

### Step 3:`NonUniformResourceIndex` を外してみる

```hlsl
Texture2D texture = ResourceDescriptorHeap[diffuseTextureIndex];   // ❌
```

**コンパイル時に警告が出ます。**

```
warning: Resource index is not uniform across the wave
```

**実行すると、複数のマテリアルが混在する境界で、テクスチャが誤って表示されることがあります。**

**症状が出ない場合もあります。** ドライバの実装や、warp 内のマテリアル分布によります。

**「たまに壊れる」の典型です**(第30章 30.7.1 節)。

**確認したら元に戻してください。**

### Step 4:無効なインデックスを渡してみる

```cpp
material.diffuseTextureIndex = 99999;   // ❌ ヒープサイズを超える
```

**GPU-Based Validation を有効にします**(第30章 30.6 節)。

```
D3D12 ERROR: GPU-BASED VALIDATION: Draw, Descriptor heap index out of bounds:
  Heap Index: [99999], Heap Size: [8192],
  Shader Stage: PIXEL, Shader Code: Lit.hlsl(64)
```

**シェーダーの行番号まで出ています**(第13章の PDB)。

**GBV を無効にすると、多くの場合はページフォルトでクラッシュします。**

**第31章の Aftermath で解析してください。**

**確認したら元に戻してください。**

### Step 5:型を間違えてみる

```hlsl
//--- SRV のインデックスを、構造化バッファとして解釈 ---
StructuredBuffer<float4> buf = ResourceDescriptorHeap[diffuseTextureIndex];
const float4 value = buf[0];   // ❌
```

**コンパイルは通ります。**

**GBV が検出します。**

```
D3D12 ERROR: GPU-BASED VALIDATION: Descriptor type mismatch:
  Expected: SRV (Buffer), Actual: SRV (Texture2D)
```

**33.4.2 節で説明した「型は代入時に決まる」の危険性です。**

**確認したら元に戻してください。**

### Step 6:ダミーテクスチャの効果を確認する

**テクスチャのないマテリアルを作ります。**

```cpp
material.diffuseTexture = "";   // テクスチャなし
```

**33.5.2 節のダミーテクスチャが使われ、白として描画されます。**

**分岐版と比べて、シェーダーが単純になっていることを確認してください。**

### Step 7:ヒープの中身を見る

**Nsight Graphics でキャプチャし、デスクリプタヒープを開きます。**

```
Descriptor Heap: MainDescriptorHeap (8192 descriptors)
  [0]    SRV  'DefaultWhite'       (1x1, R8G8B8A8_UNORM)
  [1]    SRV  'DefaultNormal'      (1x1, R8G8B8A8_UNORM)
  [2]    SRV  'DefaultBlack'       (1x1, R8G8B8A8_UNORM)
  [3]    SRV  'Brick_Diffuse'      (1024x1024, BC7_UNORM_SRGB)
  ...
```

**33.6.2 節の対策 1 が効いていることを確認してください。**

**デバッグ名を消すと、`<unnamed>` の羅列になります。**

### Step 8:ソートの優先順位を変える

**33.5.3 節で述べた通り、マテリアルソートが不要になりました。**

**距離優先に変えて、Early-Z の効果を測ってください。**

```cpp
//--- 手前から描く ---
return a->distanceToCamera < b->distanceToCamera;
```

**GPU Trace で、ピクセルシェーダーの実行回数が減っているかを確認します**(第29章 29.3 節)。

**オーバードローが多いシーンほど、効果が大きくなります。**

### Step 9:フォールバックと比較する

```cpp
m_useBindless = false;
```

**絵がまったく同じであることを確認してください。**

**統計を比べると、マテリアル切り替えの差が見えます。**

**フレームレートの差も測ってください。** マテリアル数が少ないと差は小さいですが、**数十種類あれば明確になります。**

---

### 本章の達成状態

- [ ] `SupportsBindless()` で対応を確認した
- [ ] ルートシグネチャに `HEAP_DIRECTLY_INDEXED` フラグを付けた
- [ ] ディスクリプタテーブルを削除した
- [ ] `ResourceDescriptorHeap` でテクスチャを取得している
- [ ] **`NonUniformResourceIndex` を使っている**
- [ ] `DescriptorHandle::index` をマテリアルに保存している
- [ ] ダミーテクスチャで分岐を回避した
- [ ] マテリアル切り替えがゼロになった
- [ ] ソートを距離優先に変えた
- [ ] デバッグ名でヒープの中身が追える
- [ ] GPU-Based Validation で検証した
- [ ] フォールバック版を用意した(任意)

---

## トラブルシューティング

| 症状 | 原因 | 対処 |
|---|---|---|
| コンパイルエラー | シェーダーモデルが 6.6 未満 | `-T ps_6_6`(33.2.1) |
| ルートシグネチャ生成に失敗 | フラグ不足 | `HEAP_DIRECTLY_INDEXED`(33.3.1) |
| 一部のピクセルが違うテクスチャ | `NonUniformResourceIndex` 忘れ | 33.4.3 |
| ページフォルト | インデックスが範囲外 | GBV で検証(Step 4) |
| 型の不一致 | 代入先の型が誤り | GBV で検証(33.4.2) |
| テクスチャが真っ黒 | 無効なインデックス | ダミーテクスチャ(33.5.2) |
| ヒープの中身が読めない | デバッグ名がない | 第6章 6.5 節 |
| 対応していない環境で動かない | フォールバック未実装 | 33.7 |
| `SetDescriptorHeaps` を忘れた | ヒープがバインドされていない | 第18章 18.5.3 節 |

---

## まとめ

**1. インデックスだけを渡す。**
ディスクリプタテーブルも、レジスタの宣言も不要です。**シェーダーが「何番を使え」と言われるだけになります。**

**2. 第21章の `index` が、ここで回収された。**
`DescriptorHandle` に番号を持たせた理由が、12 章越しに明らかになりました。

**3. マテリアル数に依存しなくなる。**
予算の節約は 1 DWORD ですが、本質は「テクスチャの種類がいくら増えても構造が変わらない」ことです。

**4. `NonUniformResourceIndex` を忘れない。**
ピクセルシェーダーでは、warp 内でインデックスが異なるのが普通です。**忘れると「たまに壊れる」状態になります。**

**5. ダミーテクスチャで分岐を消す。**
1×1 の白・法線・黒を用意すれば、無効なインデックスの分岐が不要になります。

**6. デバッグが難しくなる。**
ルートシグネチャに宣言がないため、ツールから「何を使っているか」が見えにくくなります。**デバッグ名と GBV の重要度が上がります。**

**7. ソートの優先順位が変わる。**
マテリアルが状態から外れたので、距離ソートを優先できます。**Early-Z がより効くようになります。**

次章ではインスタンシングと `ExecuteIndirect` を扱います。**本章で「マテリアルをインデックスで参照する」形にしたので、インスタンスごとに異なるテクスチャを使えるようになりました。** 第25章 25.3.5 節で「バインドレスで解決します」と書いた制約が、そこで解消されます。

---

## 参考リンク

| 内容 | URL |
|---|---|
| Dynamic Resources | https://microsoft.github.io/DirectX-Specs/d3d/HLSL_SM_6_6_DynamicResources.html |
| `ResourceDescriptorHeap` | https://learn.microsoft.com/ja-jp/windows/win32/direct3dhlsl/resourcedescriptorheap |
| `NonUniformResourceIndex` | https://learn.microsoft.com/ja-jp/windows/win32/direct3dhlsl/nonuniformresourceindex |
| Resource Binding Tier | https://learn.microsoft.com/ja-jp/windows/win32/direct3d12/hardware-support |
| ルートシグネチャのフラグ | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/ne-d3d12-d3d12_root_signature_flags |
