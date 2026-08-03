# 第25章 複数オブジェクトの描画

ここまで、画面には常に 1 つのモデルしかありませんでした。**本章で、それを数百個に増やします。**

やること自体は単純です。**オブジェクトごとにワールド行列を変えて、ループで描く。** それだけです。

しかし、そこから 3 つの問題が立ち上がります。

| 問題 | 本章での扱い |
|---|---|
| **オブジェクトごとの定数をどう渡すか** | 25.1 節 |
| **ドローコールが増えると重くなる** | 25.3 節 |
| **親子関係のある物体をどう扱うか** | 25.4 節 |

そして、**第21章で作ったリングバッファが、ようやく本領を発揮します。** 定数バッファ 1 つのために作った仕組みではありませんでした。

**本章のゴール**
数百個のオブジェクトを個別のワールド行列で描画する。ドローコールの削減手法を理解し、最小限のシーングラフを実装する。

---

## 25.1 描画パラメータの持たせ方

### 25.1.1 3 つの方法

**オブジェクトごとに異なるワールド行列を渡す方法は、いくつかあります。**

| 方法 | 仕組み | 予算(第14章 14.1.3 節) |
|---|---|---|
| **A. ルート定数** | 行列を直接埋め込む | **16 DWORD** |
| **B. ルートディスクリプタ** | リングバッファのアドレスを渡す | 2 DWORD |
| **C. 構造化バッファ + インデックス** | 全オブジェクト分をまとめ、番号だけ渡す | **1 DWORD** |

**A は予算を食いすぎます。** `ObjectConstants` は行列 3 つで 192 バイト、48 DWORD です。**予算 64 DWORD の大半を占めます。**

**本書は B から始め、25.3 節で C を検討します。**

### 25.1.2 ルートディスクリプタ方式

**第24章 24.6.2 節で、既にこの形になっています。**

```cpp
for (const auto& object : m_objects)
{
    //--- ① リングバッファから借りる ---
    const auto allocation =
        m_uploadBuffer.AllocateConstants(sizeof(ObjectConstants));

    //--- ② 行列を計算して書き込む ---
    ObjectConstants constants{};
    constants.world         = object.worldMatrix;
    constants.worldViewProj = object.worldMatrix * viewProj;
    constants.normalMatrix  =
        Math::Transpose(Math::InverseAffine(object.worldMatrix));

    allocation.Write(constants);

    //--- ③ アドレスを渡して描画 ---
    m_commandList->SetGraphicsRootConstantBufferView(1, allocation.gpuAddress);
    m_commandList->DrawIndexedInstanced(indexCount, 1, 0, 0, 0);
}
```

**これだけで、オブジェクトごとに異なる変換が適用されます。**

**第21章 21.2 節で「第25章で複数のオブジェクトを描くようになると、固定サイズでは足りません」と書きました。** リングバッファなら、100 個でも 1000 個でも同じコードで動きます。

### 25.1.3 使用量を確認する

**リングバッファのピーク使用量が、ここで意味を持ちます**(第21章 21.2.3 節)。

```
[Info ] UploadRingBuffer.cpp(88): upload buffer peak usage: 51200 / 1048576 bytes (4.9%)
```

**200 個のオブジェクトで 51200 バイトです。** 1 個あたり 256 バイト(アラインメント込み)なので、計算が合っています。

**この数字を見れば、バッファサイズが適切かどうか判断できます。** 使用率が 90% を超えるようなら増やすべきですし、常に 1% 未満なら減らせます。

---

## 25.2 描画の準備

### 25.2.1 オブジェクトの表現

```cpp
// src/Scene/RenderObject.h
#pragma once
#include "std_import.h"
#include "Math/Matrix.h"

namespace Scene
{
    struct Transform
    {
        Math::Vector3 position{ 0.0f, 0.0f, 0.0f };
        Math::Vector3 rotation{ 0.0f, 0.0f, 0.0f };   // ラジアン
        Math::Vector3 scale{ 1.0f, 1.0f, 1.0f };

        [[nodiscard]] Math::Matrix4x4 ToMatrix() const noexcept
        {
            //--- スケール → 回転 → 平行移動 の順で適用 ---
            return Math::Scaling(scale.x, scale.y, scale.z)
                 * Math::RotationX(rotation.x)
                 * Math::RotationY(rotation.y)
                 * Math::RotationZ(rotation.z)
                 * Math::Translation(position.x, position.y, position.z);
        }
    };

    struct RenderObject
    {
        Transform transform;

        std::uint32_t meshIndex     = 0;
        std::uint32_t materialIndex = 0;

        //--- 描画時に計算される ---
        Math::Matrix4x4 worldMatrix{};
        float           distanceToCamera = 0.0f;
        bool            visible = true;
    };
}
```

**変換の順序に注意してください。**

第17章 17.3.3 節で決めた行ベクトル流儀では、**適用したい順に左から右へ並べます。**

```
Scale * RotX * RotY * RotZ * Translation
  ①      ②      ③      ④        ⑤
```

**この順序でなければ、直感に反する結果になります。** たとえば平行移動を最初に置くと、**回転が原点ではなく移動後の位置を中心に掛かります。**

> **回転の表現について**
>
> オイラー角(X, Y, Z の 3 回転)は直感的ですが、**ジンバルロック**という問題があります。特定の姿勢で自由度が 1 つ失われます。
>
> 実用的なエンジンでは**クォータニオン**を使います。本書では扱いませんが、第17章の数学ライブラリに追加するのは良い練習になります。

### 25.2.2 シーンを組み立てる

**確認しやすいシーンを作ります。**

```cpp
void BuildTestScene(std::vector<RenderObject>& objects,
                    int gridSize = 10)
{
    objects.clear();
    objects.reserve(static_cast<std::size_t>(gridSize) * gridSize);

    const float spacing = 3.0f;
    const float offset  = (gridSize - 1) * spacing * 0.5f;

    for (int z = 0; z < gridSize; ++z)
    {
        for (int x = 0; x < gridSize; ++x)
        {
            RenderObject object{};

            object.transform.position = {
                x * spacing - offset,
                0.0f,
                z * spacing - offset,
            };

            //--- 位置に応じて少しずつ変化をつける ---
            object.transform.rotation.y =
                (x + z) * 0.3f;
            object.transform.scale =
                Math::Vector3{ 1.0f, 1.0f, 1.0f } * (0.8f + (x % 3) * 0.2f);

            object.materialIndex = (x + z) % materialCount;

            objects.push_back(object);
        }
    }

    LOG_INFO(L"scene built: {} objects", objects.size());
}
```

**格子状に並べる**のは、確認のためです。

- 抜けや重なりが一目で分かる
- カメラを動かしたときの見え方が予測できる
- 数を変えて性能を測りやすい

### 25.2.3 更新と描画を分ける

```cpp
void Renderer::UpdateObjects(std::vector<RenderObject>& objects,
                             const Camera& camera,
                             float deltaTime)
{
    const auto cameraPos = camera.Position();

    for (auto& object : objects)
    {
        //--- ① ワールド行列を計算 ---
        object.worldMatrix = object.transform.ToMatrix();

        //--- ② カメラからの距離(ソートとカリングで使う)---
        const Math::Vector3 objectPos{
            object.worldMatrix.m[3][0],
            object.worldMatrix.m[3][1],
            object.worldMatrix.m[3][2],
        };
        object.distanceToCamera =
            Math::Length(objectPos - cameraPos);

        object.visible = true;   // 25.3.4 節でカリングを入れる
    }
}
```

**行列の計算を描画ループの外に出しています。**

理由は 2 つあります。

- **CPU の作業を、コマンド記録から分離できる**(第35章の並列化で効く)
- **ソートやカリングの前に、位置が確定している必要がある**

**`worldMatrix.m[3][0..2]` から位置を取り出している**のは、第17章 17.3.3 節の規約によります。行ベクトル流儀では、平行移動成分が 4 行目に入ります。

---

## 25.3 ドローコールを減らす

### 25.3.1 なぜ重いのか

**ドローコールそのものは、GPU にとって軽い処理です。** 重いのは、**その周辺で起こる状態変更**です。

```cpp
for (const auto& object : objects)
{
    commandList->SetPipelineState(pso);              // ← 重い
    commandList->SetGraphicsRootSignature(rootSig);  // ← 重い
    commandList->SetDescriptorHeaps(1, heaps);       // ← 非常に重い
    commandList->IASetVertexBuffers(0, 1, &vbv);
    commandList->IASetIndexBuffer(&ibv);
    commandList->SetGraphicsRootConstantBufferView(1, address);
    commandList->DrawIndexedInstanced(...);
}
```

**コストの順序は、おおよそ次の通りです。**

| 操作 | コスト |
|---|---|
| `SetDescriptorHeaps` | **最も重い**(パイプラインのフラッシュを伴うことがある) |
| `SetPipelineState` | 重い |
| `SetGraphicsRootSignature` | 重い(**設定済みのパラメータが全部消える**) |
| `IASetVertexBuffers` / `IASetIndexBuffer` | 中程度 |
| `SetGraphicsRoot*` | **軽い** |
| `DrawIndexedInstanced` | **軽い** |

**下 2 つ以外を、できるだけ減らすのが目標です。**

### 25.3.2 状態でソートする

**同じ状態のものをまとめて描けば、切り替え回数が減ります。**

```cpp
void SortForRendering(std::vector<RenderObject*>& visible)
{
    std::ranges::sort(visible,
        [](const RenderObject* a, const RenderObject* b)
        {
            //--- ① PSO(パイプライン)---
            if (a->psoIndex != b->psoIndex)
                return a->psoIndex < b->psoIndex;

            //--- ② メッシュ(頂点バッファ)---
            if (a->meshIndex != b->meshIndex)
                return a->meshIndex < b->meshIndex;

            //--- ③ マテリアル(テクスチャ)---
            return a->materialIndex < b->materialIndex;
        });
}
```

**優先順位は、コストの高い順です。** PSO の切り替えが最も高いので、それを最外側にします。

描画側では、**前回と同じなら設定を飛ばします。**

```cpp
std::uint32_t lastPso      = 0xFFFFFFFF;
std::uint32_t lastMesh     = 0xFFFFFFFF;
std::uint32_t lastMaterial = 0xFFFFFFFF;

for (const RenderObject* object : visible)
{
    //--- 変わったときだけ設定する ---
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

    if (object->materialIndex != lastMaterial)
    {
        m_commandList->SetGraphicsRootDescriptorTable(
            3, m_materials[object->materialIndex].srvHandle);
        lastMaterial = object->materialIndex;
    }

    //--- オブジェクト定数は毎回必要 ---
    const auto allocation =
        m_uploadBuffer.AllocateConstants(sizeof(ObjectConstants));
    allocation.Write(BuildObjectConstants(*object));

    m_commandList->SetGraphicsRootConstantBufferView(1, allocation.gpuAddress);
    m_commandList->DrawIndexedInstanced(mesh.indexCount, 1, 0, 0, 0);
}
```

**`SetDescriptorHeaps` と `SetGraphicsRootSignature` は、ループの外に出します。**

```cpp
//--- ループの前に 1 回だけ ---
ID3D12DescriptorHeap* heaps[] = { m_descriptorHeap.Get() };
m_commandList->SetDescriptorHeaps(1, heaps);
m_commandList->SetGraphicsRootSignature(m_rootSignature.Get());
m_commandList->SetGraphicsRootConstantBufferView(0, sceneConstantsAddress);
```

**シーン定数も 1 回で済みます。** 第24章 24.6.1 節で定数を更新頻度で 3 つに分けたのは、このためでした。

### 25.3.3 統計を取る

**効果を測れなければ、改善したかどうか分かりません。**

```cpp
struct RenderStats
{
    std::uint32_t objectsTotal    = 0;
    std::uint32_t objectsVisible  = 0;
    std::uint32_t drawCalls       = 0;
    std::uint32_t psoChanges      = 0;
    std::uint32_t meshChanges     = 0;
    std::uint32_t materialChanges = 0;
    std::uint32_t triangles       = 0;
};
```

**ソートの有無で比較してみます。**

```
ソートなし:
  draw calls: 400, pso: 400, mesh: 400, material: 398

ソートあり:
  draw calls: 400, pso: 2, mesh: 3, material: 8
```

**ドローコールは変わりませんが、状態変更が劇的に減りました。**

**これが「ドローコールを減らす」の実態です。** 呼び出し回数そのものより、**周辺の状態変更を減らすほうが効きます。**

### 25.3.4 視錐台カリング

**画面に映らないオブジェクトは、描かないのが最も速い方法です。**

```cpp
struct BoundingSphere
{
    Math::Vector3 center;
    float         radius = 0.0f;
};

//--- 視錐台の 6 平面 ---
struct Frustum
{
    // 平面は ax + by + cz + d = 0 の形。法線は内向き
    std::array<Math::Vector4, 6> planes;

    [[nodiscard]] bool Intersects(const BoundingSphere& sphere) const noexcept
    {
        for (const auto& plane : planes)
        {
            const float distance =
                  plane.x * sphere.center.x
                + plane.y * sphere.center.y
                + plane.z * sphere.center.z
                + plane.w;

            //--- 完全に外側なら除外 ---
            if (distance < -sphere.radius)
            {
                return false;
            }
        }
        return true;
    }
};
```

**視錐台の平面は、ビュー射影行列から抽出できます。**

```cpp
Frustum ExtractFrustum(const Math::Matrix4x4& viewProj)
{
    Frustum frustum{};
    const auto& m = viewProj.m;

    //--- 行ベクトル流儀での抽出 ---
    // 左:  w + x,  右:  w - x
    // 下:  w + y,  上:  w - y
    // 近:  z,      遠:  w - z   (D3D の深度範囲 [0,1])

    frustum.planes[0] = { m[0][3]+m[0][0], m[1][3]+m[1][0],
                          m[2][3]+m[2][0], m[3][3]+m[3][0] };  // 左
    frustum.planes[1] = { m[0][3]-m[0][0], m[1][3]-m[1][0],
                          m[2][3]-m[2][0], m[3][3]-m[3][0] };  // 右
    frustum.planes[2] = { m[0][3]+m[0][1], m[1][3]+m[1][1],
                          m[2][3]+m[2][1], m[3][3]+m[3][1] };  // 下
    frustum.planes[3] = { m[0][3]-m[0][1], m[1][3]-m[1][1],
                          m[2][3]-m[2][1], m[3][3]-m[3][1] };  // 上
    frustum.planes[4] = { m[0][2], m[1][2], m[2][2], m[3][2] }; // 近
    frustum.planes[5] = { m[0][3]-m[0][2], m[1][3]-m[1][2],
                          m[2][3]-m[2][2], m[3][3]-m[3][2] };  // 遠

    //--- 正規化する(距離を正しく測るため)---
    for (auto& plane : frustum.planes)
    {
        const float length = std::sqrt(
            plane.x * plane.x + plane.y * plane.y + plane.z * plane.z);
        if (length > 0.0f)
        {
            plane.x /= length;
            plane.y /= length;
            plane.z /= length;
            plane.w /= length;
        }
    }

    return frustum;
}
```

> **Reversed-Z のときは近平面の式が変わる**
>
> 第19章 19.5 節で Reversed-Z を採用した場合、**近平面と遠平面が入れ替わります。**
>
> ```cpp
> #if USE_REVERSED_Z
>     frustum.planes[4] = { m[0][3]-m[0][2], ... };  // 近
>     frustum.planes[5] = { m[0][2], ... };          // 遠
> #endif
> ```
>
> **「カリングで手前のものが消える」なら、これが原因です。**

**境界球は、第23章で計算した境界ボックスから作れます。**

```cpp
BoundingSphere ComputeBoundingSphere(const Mesh& mesh)
{
    const auto center = (mesh.boundsMin + mesh.boundsMax) * 0.5f;
    const auto extent = (mesh.boundsMax - mesh.boundsMin) * 0.5f;

    return { center, Math::Length(extent) };
}
```

**ワールド空間へ変換するときは、スケールを考慮します。**

```cpp
BoundingSphere TransformSphere(const BoundingSphere& sphere,
                               const Math::Matrix4x4& world)
{
    BoundingSphere result{};
    result.center = Math::TransformPoint(sphere.center, world);

    //--- 最大スケールで半径を拡大する ---
    const float sx = Math::Length(Math::Vector3{
        world.m[0][0], world.m[0][1], world.m[0][2] });
    const float sy = Math::Length(Math::Vector3{
        world.m[1][0], world.m[1][1], world.m[1][2] });
    const float sz = Math::Length(Math::Vector3{
        world.m[2][0], world.m[2][1], world.m[2][2] });

    result.radius = sphere.radius * std::max({ sx, sy, sz });
    return result;
}
```

**非一様スケールでは、最大値を使って安全側に倒します。** 厳密には楕円体になりますが、**カリングは「除外しすぎない」ことが重要**なので、これで構いません。

### 25.3.5 インスタンシング

**同じメッシュを大量に描くなら、これが最も効果的です。**

```cpp
m_commandList->DrawIndexedInstanced(
    indexCount,
    instanceCount,    // ← 1 ではなく N
    0, 0, 0);
```

**1 回のドローコールで N 個描けます。**

各インスタンスの行列は、構造化バッファから読みます。

```hlsl
struct InstanceData
{
    row_major float4x4 world;
    row_major float4x4 normalMatrix;
};

StructuredBuffer<InstanceData> gInstances : register(t1);

VSOutput VSMain(VSInput input, uint instanceId : SV_InstanceID)
{
    const InstanceData instance = gInstances[instanceId];

    VSOutput output;
    output.positionWS = mul(float4(input.position, 1.0f), instance.world).xyz;
    output.position   = mul(float4(output.positionWS, 1.0f),
                            mul(view, projection));
    output.normalWS   = mul(float4(input.normal, 0.0f),
                            instance.normalMatrix).xyz;
    output.uv = input.uv;
    return output;
}
```

**`SV_InstanceID` は自動で渡されます**(第13章 13.1.3 節)。

**本書ではインスタンシングを第34章で本格的に扱います。** ここでは「そういう手段がある」ことだけ示します。

**制約もあります。**

- 同じメッシュ、同じマテリアルでなければならない
- インスタンスごとに異なるテクスチャは使えない(**第33章のバインドレスで解決します**)

---

## 25.4 最小限のシーングラフ

### 25.4.1 なぜ必要か

**親子関係のある物体を扱うためです。**

```
車体
 ├─ 前輪 左
 ├─ 前輪 右
 ├─ 後輪 左
 └─ 後輪 右
```

**車体を動かせば、車輪も一緒に動いてほしい。** 車輪は車体からの相対位置で指定したい。**それがシーングラフです。**

### 25.4.2 実装

```cpp
// src/Scene/SceneNode.h
namespace Scene
{
    struct SceneNode
    {
        std::string name;
        Transform   localTransform;

        //--- 階層 ---
        std::int32_t parentIndex = -1;              // -1 = ルート
        std::vector<std::int32_t> childIndices;

        //--- 描画データ(なければ空)---
        std::int32_t meshIndex     = -1;
        std::int32_t materialIndex = -1;

        //--- 計算結果 ---
        Math::Matrix4x4 worldMatrix{};
        bool            dirty = true;
    };

    class Scene
    {
    public:
        std::int32_t AddNode(std::string name,
                             std::int32_t parentIndex = -1);

        SceneNode& GetNode(std::int32_t index) { return m_nodes[index]; }

        //--- 全ノードのワールド行列を更新する ---
        void UpdateTransforms();

        //--- 描画対象のノードを集める ---
        void CollectDrawables(std::vector<const SceneNode*>& out) const;

    private:
        void UpdateNodeRecursive(std::int32_t index,
                                 const Math::Matrix4x4& parentWorld);

        std::vector<SceneNode> m_nodes;
        std::vector<std::int32_t> m_rootIndices;
    };
}
```

**ポインタではなくインデックスで階層を表現しています。**

**理由は 3 つあります。**

| 理由 | 説明 |
|---|---|
| **メモリの連続性** | `std::vector` に詰まるのでキャッシュに載る |
| **再配置に強い** | `push_back` でポインタが無効化されない |
| **シリアライズが簡単** | そのままファイルに書ける |

**大規模なシーンでは、この差が効いてきます。**

### 25.4.3 変換の伝播

```cpp
void Scene::UpdateTransforms()
{
    for (const auto rootIndex : m_rootIndices)
    {
        UpdateNodeRecursive(rootIndex, Math::Matrix4x4::Identity());
    }
}

void Scene::UpdateNodeRecursive(std::int32_t index,
                                const Math::Matrix4x4& parentWorld)
{
    auto& node = m_nodes[index];

    //--- ローカル変換に親のワールド変換を掛ける ---
    node.worldMatrix = node.localTransform.ToMatrix() * parentWorld;
    node.dirty = false;

    //--- 子へ伝播 ---
    for (const auto childIndex : node.childIndices)
    {
        UpdateNodeRecursive(childIndex, node.worldMatrix);
    }
}
```

**掛ける順序に注意してください。**

```cpp
node.worldMatrix = local * parentWorld;   // ✅ 行ベクトル流儀
```

**「まずローカル変換、次に親の変換」という順序です。** 第17章 17.3.3 節の規約通り、適用順に左から並べます。

**列ベクトル流儀の資料では `parentWorld * local` と書かれています。** 見比べるときは注意してください。

> **再帰の深さについて**
>
> 深い階層では、スタックオーバーフローの危険があります。実用的なエンジンでは、**ノードを親が先に来る順序で並べておき、ループで処理します。**
>
> ```cpp
> for (auto& node : m_nodes)   // 親が必ず先に来るよう整列済み
> {
>     const auto& parentWorld = (node.parentIndex >= 0)
>         ? m_nodes[node.parentIndex].worldMatrix
>         : Math::Matrix4x4::Identity();
>
>     node.worldMatrix = node.localTransform.ToMatrix() * parentWorld;
> }
> ```
>
> **再帰がなくなり、並列化もしやすくなります。** 本書は分かりやすさを優先して再帰版を示しました。

### 25.4.4 変換行列は毎フレーム計算する

**「変わっていないノードは計算を飛ばしたい」と考えたくなります。**

`dirty` フラグを用意しましたが、**本書では毎フレーム全部計算します。**

理由は、**行列の乗算が非常に軽いから**です。4×4 の乗算は 64 回の積和で、数千ノードでも問題になりません。

**フラグの管理を導入すると、次の問題が生じます。**

- 親が動いたら子も dirty にする必要がある
- どこかで立て忘れると、静かに壊れる
- **バグの原因が「更新されていない」なので、非常に見つけにくい**

**最適化は、測定してから行うべきです**(第38章)。数千ノードで実際に問題になったときに考えれば十分です。

---

## 25.5 半透明への準備

**第28章で扱う内容ですが、ソートの話が出たので触れておきます。**

**不透明と半透明では、必要なソート順が逆です。**

| | ソート順 | 理由 |
|---|---|---|
| **不透明** | **手前から奥へ** | Early-Z が効く(第19章 19.1.3 節) |
| **半透明** | **奥から手前へ** | 奥のものが透けて見える必要がある |

**手前から描くと、奥のピクセルは深度テストで捨てられます。** ピクセルシェーダーが実行されないので速くなります。

**ただし、状態でのソートと衝突します。**

```
状態でソート  →  PSO の切り替えが減る
距離でソート  →  Early-Z が効く
```

**両立させるには、状態を第一キー、距離を第二キーにします。**

```cpp
std::ranges::sort(opaque, [](const auto* a, const auto* b)
{
    if (a->psoIndex != b->psoIndex) return a->psoIndex < b->psoIndex;
    if (a->meshIndex != b->meshIndex) return a->meshIndex < b->meshIndex;
    return a->distanceToCamera < b->distanceToCamera;   // 手前から
});
```

**実用的には、状態変更のコストのほうが大きいことが多いです。** どちらを優先すべきかは、**測ってから決めるべき**判断です。

---

## ✅ 本章のゴール:多数の物体が個別に動く

### Step 1:格子状に並べる

```cpp
BuildTestScene(m_objects, 10);   // 100 個
```

**100 個のモデルが格子状に並びます。**

- それぞれ異なる回転
- 位置に応じたスケールの変化
- マテリアルが循環している

**第22章のカメラで飛び回ってみてください。**

### Step 2:統計を確認する

タイトルバーか、ログに出力します。

```
[Info ] Renderer.cpp(340): objects 100/100, draws 100,
        pso 1, mesh 1, material 3, tris 1,914,600
```

**マテリアル切り替えが 3 回になっています。** 3 種類のマテリアルを循環させているので、ソートが効いている証拠です。

### Step 3:ソートを無効にしてみる

```cpp
// SortForRendering(visible);   ← コメントアウト
```

```
pso 1, mesh 1, material 98
```

**マテリアル切り替えが 98 回に増えます。**

**フレームレートを比べてください。** マテリアルが多いシーンほど差が出ます。

**確認したら元に戻してください。**

### Step 4:数を増やす

```cpp
BuildTestScene(m_objects, 50);   // 2500 個
```

**フレームレートが落ちるはずです。**

```
[Info ] UploadRingBuffer.cpp(88): upload buffer peak usage: 640000 / 1048576 bytes (61.0%)
```

**リングバッファの使用率が上がっています。** 第21章 21.2.3 節で記録するようにしたピーク値が、ここで役立ちます。

**さらに増やすと枯渇します。**

```
[Error] UploadRingBuffer.cpp(72): upload buffer exhausted: need 256 bytes, 128 available
[Fatal] UploadRingBuffer.cpp(73): assertion failed: false (アップロードバッファが不足しました)
```

**黙って壊れず、アサートで止まります**(第21章 21.2.3 節)。設定値を増やしてください。

### Step 5:カリングを有効にする

```cpp
object.visible = frustum.Intersects(worldSphere);
```

**カメラを回して、統計を見てください。**

```
objects 45/2500, draws 45
```

**視界に入っているものだけが描かれます。**

**カメラを回転させると、数が動的に変化します。** これがカリングの効果です。

### Step 6:カリングの境界を確認する

**画面端でオブジェクトが消えるようなら、境界球が小さすぎます。**

```cpp
result.radius = sphere.radius * 1.0f;    // ❌ 余裕がない
result.radius = sphere.radius * 1.1f;    // 安全側
```

**カリングは「除外しすぎない」ことが重要です。** 見えているものを消してしまうより、少し多めに描くほうが安全です。

**Reversed-Z を使っている場合、手前のものが消えるなら 25.3.4 節のコラムを確認してください。**

### Step 7:シーングラフを試す

```cpp
const auto bodyIndex  = scene.AddNode("Body");
const auto wheelIndex = scene.AddNode("Wheel_FL", bodyIndex);

scene.GetNode(wheelIndex).localTransform.position = { -1.0f, -0.5f, 1.5f };
```

**車体を動かすと、車輪も一緒に動きます。**

```cpp
scene.GetNode(bodyIndex).localTransform.position.x += deltaTime;
```

**車輪だけを回転させることもできます。**

```cpp
scene.GetNode(wheelIndex).localTransform.rotation.x += deltaTime * 5.0f;
```

**親の変換と子の変換が、正しく合成されていることを確認してください。**

### Step 8:掛ける順序を逆にしてみる

```cpp
node.worldMatrix = parentWorld * node.localTransform.ToMatrix();   // ❌
```

**子の位置が、まったく違う場所になります。**

**第17章 17.3.3 節で決めた規約が守られていない**ためです。行ベクトル流儀では `local * parent` です。

**確認したら元に戻してください。**

---

### 本章の達成状態

- [ ] リングバッファからオブジェクト定数を確保している
- [ ] 変換の順序が Scale → Rotation → Translation になっている
- [ ] 更新と描画を分離した
- [ ] 状態でソートしている
- [ ] 前回と同じ状態なら設定を飛ばしている
- [ ] `SetDescriptorHeaps` をループの外に出した
- [ ] 統計を取得している
- [ ] 視錐台カリングを実装した
- [ ] 境界球にスケールを反映している
- [ ] シーングラフをインデックスで表現した
- [ ] 変換を `local * parent` の順で合成している
- [ ] **数百個のオブジェクトが個別に動く**
- [ ] Step 3 でソートの効果を確認した
- [ ] Step 5 でカリングの効果を確認した

---

## トラブルシューティング

| 症状 | 原因 | 対処 |
|---|---|---|
| 全部同じ場所に描かれる | 定数を上書きしている | リングバッファから毎回確保(25.1.2) |
| 回転が変な中心で起こる | 変換の順序 | Scale→Rot→Trans(25.2.1) |
| 子が親についてこない | 合成の順序 | `local * parent`(25.4.3) |
| バッファ枯渇でアサート | オブジェクトが多すぎる | サイズを増やす(Step 4) |
| 画面端で消える | 境界球が小さい | 余裕を持たせる(Step 6) |
| 手前のものが消える | Reversed-Z の平面 | 25.3.4 のコラム |
| ソートしても速くならない | 状態が元から少ない | 統計で確認(25.3.3) |
| フレームレートが安定しない | カリングで描画数が変動 | 正常 |
| 半透明が正しく描けない | ソート順 | **第28章で扱う** |

---

## まとめ

**1. リングバッファがあれば、オブジェクト数は自由。**
第21章で作った仕組みが、ここで本領を発揮しました。**定数バッファ 1 つのために作ったものではありませんでした。**

**2. 変換の順序は Scale → Rotation → Translation。**
行ベクトル流儀では、適用順に左から並べます。**平行移動を先に置くと、回転の中心が変わります。**

**3. ドローコールより状態変更のほうが重い。**
`SetDescriptorHeaps` が最も重く、`Draw` は軽い。**ソートで状態変更を減らすのが本質です。**

**4. 統計を取らなければ、改善したか分からない。**
ソートの有無で状態変更が 98 回から 3 回に減る、といった数字が見えて初めて判断できます。

**5. カリングは「除外しすぎない」ことが重要。**
見えているものを消すより、少し多めに描くほうが安全です。**境界球には余裕を持たせます。**

**6. シーングラフはインデックスで表現する。**
ポインタより、メモリの連続性・再配置への強さ・シリアライズのしやすさで優れます。

**7. 最適化は測ってから。**
`dirty` フラグによる更新の省略は、バグの温床になります。**行列の乗算は十分軽いので、まず全部計算します。**

次章では、オフスクリーンレンダリングとポストエフェクトを扱います。**画面全体に効果をかけるための仕組み**を作り、グレースケールやブルームを実装します。第11章から使ってきた「バックバッファに直接描く」形から、**中間バッファを経由する形**へ移行します。

---

## 参考リンク

| 内容 | URL |
|---|---|
| `DrawIndexedInstanced` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/nf-d3d12-id3d12graphicscommandlist-drawindexedinstanced |
| 構造化バッファ | https://learn.microsoft.com/ja-jp/windows/win32/direct3dhlsl/sm5-object-structuredbuffer |
| `SV_InstanceID` | https://learn.microsoft.com/ja-jp/windows/win32/direct3dhlsl/dx-graphics-hlsl-semantics |
| 視錐台平面の抽出 | https://www.gamedevs.org/uploads/fast-extraction-viewing-frustum-planes-from-world-view-projection-matrix.pdf |
| バッチングとステートソート | https://learn.microsoft.com/ja-jp/windows/win32/direct3d12/optimizing-command-lists |
