# 第23章 モデルを読み込む

立方体は 24 頂点でした(第20章 20.6.2 節)。**次は数万頂点のモデルです。**

本章で書くのは、テキストファイルを読んで頂点配列に変換するだけのコードです。Direct3D の API はほとんど登場しません。しかし、**ここで扱う 3 つの問題は、レンダラの品質を直接左右します。**

| 問題 | 影響する章 |
|---|---|
| **頂点の重複排除** | 描画性能 |
| **法線の計算** | 第24章(ライティング) |
| **接空間の計算** | 法線マップを使う場合 |

そして、第16章 16.1.1 節で予告した問題に、ここで正面から取り組みます。

> 立方体を 8 頂点で表せるのは、頂点が「位置」と「色」しか持たないからです。法線つきの立方体は 8 頂点ではなく 24 頂点になります。**第21章 21.2 節で、この判定を自動化する処理を書きます。**

**章番号がずれましたが、その処理を本章で書きます。**

**本章のゴール**
OBJ / MTL パーサを自作し、頂点を重複排除し、法線と接空間を計算する。3D モデルが表示される。

---

## 23.1 なぜ OBJ なのか

### 23.1.1 形式の選択

| 形式 | 特徴 | パーサの難易度 |
|---|---|---|
| **OBJ** | **テキスト。単純。広く普及** | **易しい** |
| FBX | バイナリ。業界標準 | SDK が必須 |
| glTF 2.0 | JSON + バイナリ。現代的 | 中程度 |
| USD | 大規模シーン向け | 難しい |

**本書は OBJ から始めます。**

理由は、**パーサが 200 行程度で書けるから**です。第1章 1.3 節の線引きにより Assimp のようなライブラリは使わないので、自分で書ける形式である必要があります。

**そして OBJ には、本章の主題に都合の良い性質があります。**

> **OBJ は、位置・法線・UV を別々のインデックスで参照します。**

これが 23.3 節で扱う「頂点の重複排除」を、必然的な作業にします。**GPU は 1 種類のインデックスしか扱えない**からです。

### 23.1.2 OBJ の限界

**公平に書いておきます。OBJ は古い形式です。**

| できないこと | 影響 |
|---|---|
| アニメーション | 静的なモデルのみ |
| シーングラフ | 階層構造を持てない |
| PBR マテリアル | 拡張はあるが標準ではない |
| バイナリ形式 | 読み込みが遅い |

**実用的なエンジンを作るなら、glTF 2.0 が現代的な選択です。** 23.6 節で構造だけ示します。

**それでも OBJ から始めるのは、本章の主題が「ファイル形式」ではなく「頂点データの構築」だからです。** 重複排除も法線計算も、どの形式でも同じことをやります。

---

## 23.2 OBJ / MTL パーサ

### 23.2.1 OBJ の構造

**テキストファイルで、1 行に 1 つの要素が書かれています。**

```
# コメント
mtllib cube.mtl          マテリアルファイルの指定

v  1.0 1.0 -1.0          頂点位置        (vertex)
vt 0.5 0.5               テクスチャ座標   (vertex texture)
vn 0.0 1.0  0.0          法線            (vertex normal)

usemtl Material_1        以降の面が使うマテリアル
f 1/1/1 2/2/1 3/3/1      面(三角形)
```

**`f` の書式が要点です。**

```
f 位置/UV/法線  位置/UV/法線  位置/UV/法線
```

**インデックスは 1 から始まります。** 0 ではありません。C++ の配列に使うときは 1 を引きます。

**省略形もあります。**

| 書式 | 意味 |
|---|---|
| `f 1 2 3` | 位置のみ |
| `f 1/1 2/2 3/3` | 位置と UV |
| `f 1//1 2//1 3//1` | 位置と法線(UV なし) |
| `f 1/1/1 2/2/1 3/3/1` | 全部 |

**負のインデックス**も許されています。`-1` は「最後に定義されたもの」を指します。**対応しておかないと、一部のファイルが読めません。**

### 23.2.2 パーサを書く

```cpp
// src/Assets/ObjLoader.h
#pragma once
#include "std_import.h"
#include "Core/Error.h"
#include "Math/Vector.h"

namespace Assets
{
    //-----------------------------------------------------------
    // 最終的な頂点。GPU へ送る形。
    //-----------------------------------------------------------
    struct MeshVertex
    {
        Math::Vector3 position;
        Math::Vector3 normal;
        Math::Vector4 tangent;    // w に従法線の向き(23.5 節)
        Math::Vector2 uv;
    };
    static_assert(sizeof(MeshVertex) == 48);

    //-----------------------------------------------------------
    // 1 つのマテリアルで描かれる範囲
    //-----------------------------------------------------------
    struct SubMesh
    {
        std::uint32_t indexOffset   = 0;
        std::uint32_t indexCount    = 0;
        std::uint32_t materialIndex = 0;
    };

    struct Material
    {
        std::string   name;
        Math::Vector3 diffuse{ 0.8f, 0.8f, 0.8f };
        Math::Vector3 specular{ 0.0f, 0.0f, 0.0f };
        float         shininess = 32.0f;
        std::string   diffuseTexture;
        std::string   normalTexture;
    };

    struct Mesh
    {
        std::vector<MeshVertex>    vertices;
        std::vector<std::uint32_t> indices;
        std::vector<SubMesh>       subMeshes;
        std::vector<Material>      materials;

        //--- 境界(第25章のカリングで使う)---
        Math::Vector3 boundsMin{};
        Math::Vector3 boundsMax{};
    };

    struct LoadOptions
    {
        bool  generateNormals  = true;   // vn がない場合に生成
        bool  generateTangents = true;
        bool  flipV            = false;  // 23.4.4 節
        float smoothingAngle   = 60.0f;  // 度。これ以上開いたら分割
    };

    Core::Result<Mesh> LoadObj(const std::filesystem::path& path,
                               const LoadOptions& options = {});
}
```

読み込みの本体です。

```cpp
namespace
{
    //--- 中間表現:OBJ の生データ ---
    struct ObjData
    {
        std::vector<Math::Vector3> positions;
        std::vector<Math::Vector3> normals;
        std::vector<Math::Vector2> uvs;
    };

    //--- f の 1 要素("1/2/3" の部分)---
    struct FaceVertex
    {
        int position = -1;
        int uv       = -1;
        int normal   = -1;

        [[nodiscard]] bool operator==(const FaceVertex&) const noexcept = default;
    };

    //-------------------------------------------------------
    // "1/2/3" 形式を解析する。
    // 省略形と負のインデックスに対応する。
    //-------------------------------------------------------
    FaceVertex ParseFaceVertex(std::string_view token, const ObjData& data)
    {
        FaceVertex result{};

        // '/' で分割する
        std::array<std::string_view, 3> parts{};
        std::size_t partCount = 0;
        std::size_t start = 0;

        for (std::size_t i = 0; i <= token.size() && partCount < 3; ++i)
        {
            if (i == token.size() || token[i] == '/')
            {
                parts[partCount++] = token.substr(start, i - start);
                start = i + 1;
            }
        }

        //--- 1 始まり / 負のインデックスを 0 始まりへ ---
        const auto resolve = [](std::string_view text, std::size_t count) -> int
        {
            if (text.empty()) return -1;

            int value = 0;
            const auto [ptr, ec] = std::from_chars(
                text.data(), text.data() + text.size(), value);
            if (ec != std::errc{}) return -1;

            if (value > 0)
            {
                return value - 1;                          // 1 始まり
            }
            if (value < 0)
            {
                return static_cast<int>(count) + value;    // 末尾から
            }
            return -1;                                     // 0 は不正
        };

        result.position = resolve(parts[0], data.positions.size());
        result.uv       = resolve(parts[1], data.uvs.size());
        result.normal   = resolve(parts[2], data.normals.size());

        return result;
    }
}
```

**`std::from_chars` を使っている**点に注目してください。`std::stoi` や `atoi` より速く、例外を投げず、ロケールの影響も受けません。

**ロケールの問題は実際に起こります。** ドイツ語圏の Windows では小数点がカンマなので、`std::stof` が `1.5` を `1` と解釈します。**`std::from_chars` はロケール非依存です。**

### 23.2.3 行を処理する

```cpp
Core::Result<Mesh> LoadObj(const std::filesystem::path& path,
                           const LoadOptions& options)
{
    std::ifstream file(path);
    if (!file)
    {
        LOG_ERROR(L"obj not found: {}", path.wstring());
        return std::unexpected(Core::MakeError(
            HRESULT_FROM_WIN32(ERROR_FILE_NOT_FOUND), L"LoadObj"));
    }

    ObjData data{};
    std::vector<std::array<FaceVertex, 3>> faces;

    Mesh mesh{};
    std::uint32_t currentMaterial = 0;
    std::vector<std::pair<std::uint32_t, std::uint32_t>> materialRanges;

    std::string line;
    std::size_t lineNumber = 0;

    while (std::getline(file, line))
    {
        ++lineNumber;

        //--- コメントと空行を飛ばす ---
        const auto commentPos = line.find('#');
        if (commentPos != std::string::npos)
        {
            line.resize(commentPos);
        }

        auto tokens = Tokenize(line);
        if (tokens.empty()) continue;

        const std::string_view keyword = tokens[0];

        //--- 頂点位置 ---
        if (keyword == "v" && tokens.size() >= 4)
        {
            data.positions.push_back({
                ParseFloat(tokens[1]),
                ParseFloat(tokens[2]),
                ParseFloat(tokens[3]) });
        }
        //--- テクスチャ座標 ---
        else if (keyword == "vt" && tokens.size() >= 3)
        {
            float u = ParseFloat(tokens[1]);
            float v = ParseFloat(tokens[2]);

            if (options.flipV) v = 1.0f - v;    // 23.4.4 節

            data.uvs.push_back({ u, v });
        }
        //--- 法線 ---
        else if (keyword == "vn" && tokens.size() >= 4)
        {
            data.normals.push_back({
                ParseFloat(tokens[1]),
                ParseFloat(tokens[2]),
                ParseFloat(tokens[3]) });
        }
        //--- 面 ---
        else if (keyword == "f" && tokens.size() >= 4)
        {
            //--- 多角形を三角形に分割する(ファン分割)---
            const std::size_t cornerCount = tokens.size() - 1;

            std::vector<FaceVertex> corners;
            corners.reserve(cornerCount);
            for (std::size_t i = 1; i < tokens.size(); ++i)
            {
                corners.push_back(ParseFaceVertex(tokens[i], data));
            }

            for (std::size_t i = 1; i + 1 < cornerCount; ++i)
            {
                faces.push_back({ corners[0], corners[i], corners[i + 1] });
            }
        }
        //--- マテリアル指定 ---
        else if (keyword == "usemtl" && tokens.size() >= 2)
        {
            // 範囲の切れ目を記録する
            materialRanges.push_back({
                static_cast<std::uint32_t>(faces.size() * 3),
                currentMaterial });

            currentMaterial = FindOrAddMaterial(mesh.materials, tokens[1]);
        }
        //--- マテリアルファイル ---
        else if (keyword == "mtllib" && tokens.size() >= 2)
        {
            const auto mtlPath = path.parent_path() /
                std::filesystem::path(std::string(tokens[1]));

            if (auto r = LoadMtl(mtlPath, mesh.materials); !r)
            {
                LOG_WARN(L"failed to load mtl: {}", mtlPath.wstring());
                // 続行する。マテリアルは既定値になる
            }
        }
    }

    LOG_INFO(L"obj parsed: {} positions, {} normals, {} uvs, {} triangles",
             data.positions.size(), data.normals.size(),
             data.uvs.size(), faces.size());

    //--- 以降、23.3 節の重複排除へ ---
    return BuildMesh(data, faces, materialRanges, mesh, options);
}
```

**多角形のファン分割**に注目してください。OBJ の `f` は 4 頂点以上を許します。

```
f 1 2 3 4      ← 四角形

→  (1,2,3) と (1,3,4) の 2 三角形に分割
```

**凸多角形なら、これで正しく分割できます。** 凹多角形は破綻しますが、実用上のモデルではまず問題になりません。

**`mtllib` の読み込みに失敗しても続行する**のも意図的です。マテリアルがなくてもジオメトリは表示できます。**「テクスチャが見つからないから起動しない」は、開発中に困ります。**

### 23.2.4 MTL パーサ

```cpp
Core::Status LoadMtl(const std::filesystem::path& path,
                     std::vector<Material>& materials)
{
    std::ifstream file(path);
    if (!file) return std::unexpected(Core::MakeError(E_FAIL, L"mtl not found"));

    Material* current = nullptr;
    std::string line;

    while (std::getline(file, line))
    {
        const auto commentPos = line.find('#');
        if (commentPos != std::string::npos) line.resize(commentPos);

        auto tokens = Tokenize(line);
        if (tokens.empty()) continue;

        const std::string_view keyword = tokens[0];

        if (keyword == "newmtl" && tokens.size() >= 2)
        {
            materials.push_back(Material{ .name = std::string(tokens[1]) });
            current = &materials.back();
        }
        else if (current == nullptr)
        {
            continue;      // newmtl の前の記述は無視
        }
        else if (keyword == "Kd" && tokens.size() >= 4)      // 拡散色
        {
            current->diffuse = { ParseFloat(tokens[1]),
                                 ParseFloat(tokens[2]),
                                 ParseFloat(tokens[3]) };
        }
        else if (keyword == "Ks" && tokens.size() >= 4)      // 鏡面色
        {
            current->specular = { ParseFloat(tokens[1]),
                                  ParseFloat(tokens[2]),
                                  ParseFloat(tokens[3]) };
        }
        else if (keyword == "Ns" && tokens.size() >= 2)      // 鏡面の鋭さ
        {
            current->shininess = ParseFloat(tokens[1]);
        }
        else if (keyword == "map_Kd" && tokens.size() >= 2)  // 拡散テクスチャ
        {
            current->diffuseTexture = ParseTexturePath(tokens);
        }
        else if ((keyword == "map_Bump" || keyword == "bump" ||
                  keyword == "norm") && tokens.size() >= 2)
        {
            current->normalTexture = ParseTexturePath(tokens);
        }
    }

    LOG_INFO(L"mtl loaded: {} material(s)", materials.size());
    return {};
}
```

> **テクスチャのパスに注意**
>
> `map_Kd` の行には、オプションが付くことがあります。
>
> ```
> map_Kd -bm 0.5 -o 1 1 1 brick_diffuse.png
> ```
>
> **単純に `tokens[1]` を取ると、`-bm` をファイル名だと思ってしまいます。** `-` で始まるトークンを飛ばす処理が必要です。
>
> また、**パスに空白が含まれることもあります。** 厳密にやるなら、最後の `-` オプションの後ろを全部つなげます。`ParseTexturePath` でこれを行います。

**テクスチャの拡張子は `.png` や `.jpg` であることがほとんどです。** 第20章で DDS しか読めないようにしたので、**拡張子を `.dds` に差し替えて探す**という処理を入れておくと実用的です。

```cpp
std::filesystem::path ResolveTexturePath(
    const std::filesystem::path& baseDir, const std::string& name)
{
    auto path = baseDir / name;

    //--- そのまま存在すれば使う ---
    if (std::filesystem::exists(path)) return path;

    //--- .dds に差し替えて探す(第20章)---
    path.replace_extension(".dds");
    if (std::filesystem::exists(path)) return path;

    LOG_WARN(L"texture not found: {}", (baseDir / name).wstring());
    return {};
}
```

---

## 23.3 頂点の重複排除

### 23.3.1 なぜ必要か

**OBJ と GPU の、インデックスの扱いが違います。**

```
OBJ:  f 1/5/3    位置は 1 番、UV は 5 番、法線は 3 番
GPU:  インデックス 42    →  頂点配列の 42 番目(全属性を含む)
```

**GPU は 1 種類のインデックスしか扱えません。** したがって、**「位置・UV・法線の組み合わせ」ごとに 1 頂点を作る**必要があります。

**第16章 16.1.1 節で予告した内容が、ここで具体的になります。**

> 立方体の角では、隣り合う 3 つの面で法線が違います。位置は同じでも法線が違うので、共有できません。

**OBJ ファイルの中では、立方体の角は `v` として 8 個しか定義されていません。** しかし `f` の中で `1/1/1`、`1/2/2`、`1/3/3` のように、異なる組み合わせで参照されます。**それぞれが別の頂点になります。**

### 23.3.2 ハッシュマップで重複を排除する

```cpp
namespace
{
    //--- FaceVertex をハッシュマップのキーにする ---
    struct FaceVertexHash
    {
        std::size_t operator()(const FaceVertex& v) const noexcept
        {
            // 3 つの整数を混ぜる。boost::hash_combine と同じ手法
            std::size_t seed = 0;
            const auto combine = [&seed](int value)
            {
                seed ^= std::hash<int>{}(value)
                      + 0x9e3779b9 + (seed << 6) + (seed >> 2);
            };
            combine(v.position);
            combine(v.uv);
            combine(v.normal);
            return seed;
        }
    };
}

void DeduplicateVertices(
    const ObjData& data,
    const std::vector<std::array<FaceVertex, 3>>& faces,
    Mesh& mesh)
{
    std::unordered_map<FaceVertex, std::uint32_t, FaceVertexHash> lookup;

    mesh.vertices.reserve(faces.size() * 3 / 2);   // 経験的な見積もり
    mesh.indices.reserve(faces.size() * 3);
    lookup.reserve(faces.size() * 2);

    for (const auto& face : faces)
    {
        for (const FaceVertex& fv : face)
        {
            //--- 既に作った組み合わせか ---
            if (const auto it = lookup.find(fv); it != lookup.end())
            {
                mesh.indices.push_back(it->second);
                continue;
            }

            //--- 新しい頂点を作る ---
            MeshVertex vertex{};

            if (fv.position >= 0 &&
                fv.position < static_cast<int>(data.positions.size()))
            {
                vertex.position = data.positions[fv.position];
            }
            if (fv.uv >= 0 && fv.uv < static_cast<int>(data.uvs.size()))
            {
                vertex.uv = data.uvs[fv.uv];
            }
            if (fv.normal >= 0 &&
                fv.normal < static_cast<int>(data.normals.size()))
            {
                vertex.normal = data.normals[fv.normal];
            }

            const auto index = static_cast<std::uint32_t>(mesh.vertices.size());
            mesh.vertices.push_back(vertex);
            lookup.emplace(fv, index);
            mesh.indices.push_back(index);
        }
    }

    LOG_INFO(L"deduplicated: {} triangles -> {} vertices ({} unique)",
             faces.size(), mesh.vertices.size(), lookup.size());
}
```

**範囲チェックを入れている**のが実用上の要点です。壊れた OBJ ファイルは実在します。**チェックがないと、その場でクラッシュします。**

**`reserve` の見積もり**は、経験則です。三角形の数 × 3 が上限で、共有が進めばその半分程度になります。**外れても動きますが、再確保が減ります。**

### 23.3.3 インデックスの型を選ぶ

**第16章 16.1.2 節で「可能な限り 16bit を使う」と書きました。** ここで自動化します。

```cpp
struct MeshBuffers
{
    ComPtr<ID3D12Resource>   vertexBuffer;
    ComPtr<ID3D12Resource>   indexBuffer;
    D3D12_VERTEX_BUFFER_VIEW vbv{};
    D3D12_INDEX_BUFFER_VIEW  ibv{};
};

Core::Result<MeshBuffers> UploadMesh(
    ResourceUploader& uploader, const Mesh& mesh, std::wstring_view name)
{
    MeshBuffers buffers{};

    //--- 頂点バッファ ---
    auto vb = uploader.UploadBuffer(
        mesh.vertices.data(),
        mesh.vertices.size() * sizeof(MeshVertex),
        D3D12_RESOURCE_STATE_VERTEX_AND_CONSTANT_BUFFER,
        std::format(L"{}_VB", name));
    if (!vb) return std::unexpected(vb.error());

    buffers.vertexBuffer = *vb;
    buffers.vbv.BufferLocation = (*vb)->GetGPUVirtualAddress();
    buffers.vbv.SizeInBytes    = static_cast<UINT>(
        mesh.vertices.size() * sizeof(MeshVertex));
    buffers.vbv.StrideInBytes  = sizeof(MeshVertex);

    //--- インデックスは頂点数で型を決める ---
    const bool use16bit = mesh.vertices.size() <= 65535;

    if (use16bit)
    {
        std::vector<std::uint16_t> indices16;
        indices16.reserve(mesh.indices.size());
        for (const auto index : mesh.indices)
        {
            indices16.push_back(static_cast<std::uint16_t>(index));
        }

        auto ib = uploader.UploadBuffer(
            indices16.data(), indices16.size() * sizeof(std::uint16_t),
            D3D12_RESOURCE_STATE_INDEX_BUFFER,
            std::format(L"{}_IB", name));
        if (!ib) return std::unexpected(ib.error());

        buffers.indexBuffer = *ib;
        buffers.ibv.Format  = DXGI_FORMAT_R16_UINT;
        buffers.ibv.SizeInBytes = static_cast<UINT>(
            indices16.size() * sizeof(std::uint16_t));
    }
    else
    {
        // 32bit のまま
        // ...
        buffers.ibv.Format = DXGI_FORMAT_R32_UINT;
    }

    buffers.ibv.BufferLocation = buffers.indexBuffer->GetGPUVirtualAddress();

    LOG_INFO(L"mesh uploaded: {} vertices, {} indices ({}bit)",
             mesh.vertices.size(), mesh.indices.size(), use16bit ? 16 : 32);

    return buffers;
}
```

**第21章の `ResourceUploader` が、そのまま使えます。** テクスチャもメッシュも、同じ仕組みで転送できます。

---

## 23.4 法線を計算する

### 23.4.1 `vn` がない場合

**OBJ には法線が含まれていないことがあります。** その場合、自分で計算します。

**面法線は、外積で求まります**(第16章 16.2.3 節と同じ考え方)。

```cpp
Math::Vector3 ComputeFaceNormal(const Math::Vector3& v0,
                                const Math::Vector3& v1,
                                const Math::Vector3& v2) noexcept
{
    const auto e1 = v1 - v0;
    const auto e2 = v2 - v0;
    return Math::Normalize(Math::Cross(e1, e2));
}
```

**頂点法線は、その頂点を共有する面の法線を平均します。**

```cpp
void ComputeSmoothNormals(Mesh& mesh)
{
    //--- 一度クリアする ---
    for (auto& vertex : mesh.vertices)
    {
        vertex.normal = {};
    }

    //--- 面法線を加算していく ---
    for (std::size_t i = 0; i + 2 < mesh.indices.size(); i += 3)
    {
        const auto i0 = mesh.indices[i];
        const auto i1 = mesh.indices[i + 1];
        const auto i2 = mesh.indices[i + 2];

        const auto& v0 = mesh.vertices[i0].position;
        const auto& v1 = mesh.vertices[i1].position;
        const auto& v2 = mesh.vertices[i2].position;

        //--- 正規化しない外積を足す ---
        // 長さが面積に比例するので、大きい面ほど重みが増す
        const auto weighted = Math::Cross(v1 - v0, v2 - v0);

        mesh.vertices[i0].normal += weighted;
        mesh.vertices[i1].normal += weighted;
        mesh.vertices[i2].normal += weighted;
    }

    //--- 正規化する ---
    for (auto& vertex : mesh.vertices)
    {
        vertex.normal = Math::Normalize(vertex.normal);
    }
}
```

**正規化しない外積を足している**のが要点です。

外積の長さは**三角形の面積の 2 倍**になります。これをそのまま足すと、**大きい面ほど強く影響します。** これは望ましい性質です。細かい三角形が密集している部分に法線が引っ張られるのを防げます。

### 23.4.2 スムージング角

**問題があります。この方法では、立方体の角も平均されてしまいます。**

```
理想:  各面が別々の法線を持つ → 角がはっきり見える
実際:  平均される → 角が丸く見える
```

**解決策は、「隣接面の角度が大きければ、頂点を分割する」ことです。**

```cpp
void ComputeNormalsWithSmoothingAngle(Mesh& mesh, float smoothingAngleDegrees)
{
    const float threshold = std::cos(Math::ToRadians(smoothingAngleDegrees));

    //--- ① 位置が同じ頂点をグループ化する ---
    // (重複排除で分かれた頂点も、位置は同じ)
    std::unordered_map<PositionKey, std::vector<std::uint32_t>,
                       PositionKeyHash> groups;

    for (std::uint32_t i = 0; i < mesh.vertices.size(); ++i)
    {
        groups[MakePositionKey(mesh.vertices[i].position)].push_back(i);
    }

    //--- ② 各三角形の面法線を求める ---
    const std::size_t triangleCount = mesh.indices.size() / 3;
    std::vector<Math::Vector3> faceNormals(triangleCount);
    std::vector<std::vector<std::uint32_t>> vertexToFaces(mesh.vertices.size());

    for (std::size_t t = 0; t < triangleCount; ++t)
    {
        const auto i0 = mesh.indices[t * 3];
        const auto i1 = mesh.indices[t * 3 + 1];
        const auto i2 = mesh.indices[t * 3 + 2];

        faceNormals[t] = ComputeFaceNormal(
            mesh.vertices[i0].position,
            mesh.vertices[i1].position,
            mesh.vertices[i2].position);

        vertexToFaces[i0].push_back(static_cast<std::uint32_t>(t));
        vertexToFaces[i1].push_back(static_cast<std::uint32_t>(t));
        vertexToFaces[i2].push_back(static_cast<std::uint32_t>(t));
    }

    //--- ③ 角度がしきい値以内の面だけを平均する ---
    for (std::uint32_t v = 0; v < mesh.vertices.size(); ++v)
    {
        if (vertexToFaces[v].empty()) continue;

        // この頂点が属する最初の面を基準にする
        const auto& base = faceNormals[vertexToFaces[v][0]];

        Math::Vector3 sum{};
        for (const auto faceIndex : vertexToFaces[v])
        {
            const auto& normal = faceNormals[faceIndex];
            if (Math::Dot(base, normal) >= threshold)
            {
                sum += normal;
            }
        }

        mesh.vertices[v].normal = Math::Normalize(sum);
    }
}
```

**`smoothingAngle = 60` 度が実用的な既定値です。**

| 角度 | 結果 |
|---|---|
| 0 度 | すべて分割。完全にフラット |
| **60 度** | **立方体は角が立ち、球は滑らか** |
| 180 度 | すべて平均。すべて滑らか |

**位置でグループ化する処理(①)**が必要なのは、23.3 節の重複排除によって、**同じ位置の頂点が複数に分かれているから**です。UV が違うだけで分割された頂点も、法線の計算では同じ点として扱う必要があります。

> **浮動小数の比較に注意**
>
> `PositionKey` は、座標をそのままキーにできません。**わずかな誤差で別の頂点として扱われます。**
>
> 実用的には、**座標を量子化します。**
>
> ```cpp
> struct PositionKey
> {
>     std::int32_t x, y, z;
>     bool operator==(const PositionKey&) const noexcept = default;
> };
>
> PositionKey MakePositionKey(const Math::Vector3& p) noexcept
> {
>     constexpr float kScale = 10000.0f;   // 0.0001 単位
>     return {
>         static_cast<std::int32_t>(std::round(p.x * kScale)),
>         static_cast<std::int32_t>(std::round(p.y * kScale)),
>         static_cast<std::int32_t>(std::round(p.z * kScale)),
>     };
> }
> ```
>
> **境界付近では取りこぼしがありますが、実用上はこれで十分です。**

### 23.4.3 法線の向きを確認する

**法線が裏返っていると、第24章でライティングが真っ黒になります。**

**第22章でカメラを作ったので、確認は簡単です。** 法線を色として表示するデバッグ描画を用意します。

```hlsl
float4 PSDebugNormal(VSOutput input) : SV_Target
{
    // [-1,1] を [0,1] へ写す
    const float3 color = normalize(input.normalWS) * 0.5f + 0.5f;
    return float4(color, 1.0f);
}
```

| 見え方 | 法線の向き |
|---|---|
| 赤 | +X |
| 緑 | +Y |
| 青 | +Z |
| 暗い色 | 負の方向 |

**モデルを回して、外側を向いた面が明るい色になっていれば正しい**です。裏返っていれば、暗い色(補色)になります。

**第16章 16.2.3 節で「巻き順を間違えると面が消える」と書きました。** 法線の向きも、巻き順から導かれます。**両者は連動しています。**

### 23.4.4 UV の V 軸

**第20章 20.6.1 節で書いた通り、D3D の UV は左上が原点です。**

**OBJ は、慣習的に左下原点です。** Blender、Maya、3ds Max のいずれも、OBJ エクスポート時は左下原点で書き出します。

**そのため、`flipV` オプションが必要になります。**

```cpp
if (options.flipV) v = 1.0f - v;
```

**「テクスチャが上下逆に貼られる」なら、これが原因です。**

**既定値をどちらにするか**は、判断が分かれます。本書は `false` にしています。**モデルによって異なるので、確認して切り替えるのが確実**だからです。

---

## 23.5 接空間

### 23.5.1 何のために必要か

**法線マップを使うために必要です。**

法線マップは、テクスチャに法線の向きを記録したものです。しかし、**そこに書かれているのは「面に対する相対的な向き」**です。

```
法線マップの値: (0.5, 0.5, 1.0)  →  「面の真上」

面が上を向いていれば → ワールドの (0, 1, 0)
面が右を向いていれば → ワールドの (1, 0, 0)
```

**同じ値でも、面の向きによって意味が変わります。** この変換をするための座標系が接空間です。

```
接空間の 3 軸:
  T (tangent)   = UV の U 方向
  B (bitangent) = UV の V 方向
  N (normal)    = 面の法線
```

**本書では法線マップを扱いませんが、接空間の計算は本章で済ませます。** データ構造に含めておけば、後から拡張しやすくなります。

### 23.5.2 計算する

**三角形の 2 辺と、対応する UV の差分から求まります。**

```cpp
void ComputeTangents(Mesh& mesh)
{
    std::vector<Math::Vector3> tangents(mesh.vertices.size());
    std::vector<Math::Vector3> bitangents(mesh.vertices.size());

    for (std::size_t i = 0; i + 2 < mesh.indices.size(); i += 3)
    {
        const auto i0 = mesh.indices[i];
        const auto i1 = mesh.indices[i + 1];
        const auto i2 = mesh.indices[i + 2];

        const auto& p0 = mesh.vertices[i0].position;
        const auto& p1 = mesh.vertices[i1].position;
        const auto& p2 = mesh.vertices[i2].position;

        const auto& uv0 = mesh.vertices[i0].uv;
        const auto& uv1 = mesh.vertices[i1].uv;
        const auto& uv2 = mesh.vertices[i2].uv;

        const auto e1 = p1 - p0;
        const auto e2 = p2 - p0;

        const float du1 = uv1.x - uv0.x;
        const float dv1 = uv1.y - uv0.y;
        const float du2 = uv2.x - uv0.x;
        const float dv2 = uv2.y - uv0.y;

        const float determinant = du1 * dv2 - du2 * dv1;

        //--- 縮退した UV を避ける ---
        if (std::abs(determinant) < 1e-8f)
        {
            continue;
        }

        const float r = 1.0f / determinant;

        const Math::Vector3 tangent   = (e1 * dv2 - e2 * dv1) * r;
        const Math::Vector3 bitangent = (e2 * du1 - e1 * du2) * r;

        tangents[i0]   += tangent;
        tangents[i1]   += tangent;
        tangents[i2]   += tangent;
        bitangents[i0] += bitangent;
        bitangents[i1] += bitangent;
        bitangents[i2] += bitangent;
    }

    //--- グラム・シュミット直交化 ---
    for (std::size_t i = 0; i < mesh.vertices.size(); ++i)
    {
        const auto& n = mesh.vertices[i].normal;
        const auto& t = tangents[i];

        //--- 法線成分を取り除く ---
        auto orthogonal = t - n * Math::Dot(n, t);

        if (Math::LengthSquared(orthogonal) < 1e-12f)
        {
            //--- 縮退した場合は、適当な直交ベクトルを作る ---
            orthogonal = std::abs(n.x) < 0.9f
                ? Math::Cross(n, Math::Vector3{ 1, 0, 0 })
                : Math::Cross(n, Math::Vector3{ 0, 1, 0 });
        }

        orthogonal = Math::Normalize(orthogonal);

        //--- w に従法線の向きを記録する ---
        const float handedness =
            (Math::Dot(Math::Cross(n, orthogonal), bitangents[i]) < 0.0f)
            ? -1.0f : 1.0f;

        mesh.vertices[i].tangent = Math::Vector4(orthogonal, handedness);
    }
}
```

**`w` に符号を入れている**のが、業界で標準的な手法です。

従法線(bitangent)を頂点データに含めると 12 バイト増えますが、**シェーダー側で復元できます。**

```hlsl
float3 bitangent = cross(normal, tangent.xyz) * tangent.w;
```

**`w` の 4 バイトだけで済みます。** 第20章の `MeshVertex` を 48 バイトに収められたのは、この工夫によるものです。

**縮退の処理を 2 箇所に入れている**点にも注目してください。UV が潰れている三角形(すべて同じ UV を持つなど)は実在します。**チェックがないと `1/0` でゼロ除算が発生し、NaN が伝播します。**

---

## 23.6 glTF 2.0 について

**本書では実装しませんが、次に進むなら glTF 2.0 です。**

### 23.6.1 構造

```
model.gltf   ← JSON。シーン構造、マテリアル、参照情報
model.bin    ← バイナリ。頂点・インデックスの生データ
textures/    ← 画像ファイル

または

model.glb    ← 上記を 1 ファイルにまとめたバイナリ形式
```

**OBJ との決定的な違いは、頂点データがバイナリで、GPU にそのまま送れる形で入っていることです。**

```json
{
  "accessors": [{
    "bufferView": 0,
    "componentType": 5126,      // FLOAT
    "count": 24,
    "type": "VEC3"              // float3
  }],
  "bufferViews": [{
    "buffer": 0,
    "byteOffset": 0,
    "byteLength": 288
  }]
}
```

**パースは JSON の解析が中心で、頂点データは `memcpy` するだけです。** テキストから数値へ変換する処理が要りません。**数万頂点のモデルでは、読み込み時間が桁違いになります。**

### 23.6.2 何が得られるか

| 機能 | OBJ | glTF 2.0 |
|---|---|---|
| シーングラフ | なし | **あり** |
| PBR マテリアル | なし | **標準** |
| アニメーション | なし | **あり** |
| スキニング | なし | **あり** |
| 読み込み速度 | 遅い | **速い** |

**PBR(物理ベースレンダリング)マテリアルが標準なのは大きな利点です。** 金属度と粗さという、現代的なパラメータで記述されています。

### 23.6.3 実装の勘所

**JSON パーサが必要です。** 本書の方針なら自作することになりますが、**JSON のパーサは 500 行程度**で書けます。OBJ パーサより少し大きい程度です。

**注意点を 3 つ挙げます。**

**1. glTF は右手系、Y-up**
D3D の左手系に変換する必要があります。**Z 軸を反転し、三角形の巻き順も反転します。**

**2. インデックスの型が可変**
`componentType` が `5121`(UNSIGNED_BYTE)、`5123`(UNSIGNED_SHORT)、`5125`(UNSIGNED_INT)のいずれかです。

**3. `byteStride` に対応する**
頂点属性がインターリーブされている場合、飛び飛びに読む必要があります。

**本書を終えた読者にとって、良い次の題材になるはずです。**

---

## ✅ 本章のゴール:3D モデルが表示される

### Step 1:モデルを用意する

**テスト用のモデルは、次のような条件を満たすものが適しています。**

- 頂点数が数千〜数万
- テクスチャが 1 枚以上
- 曲面と平面の両方を含む(法線の確認用)

**Stanford Bunny や Utah Teapot のような定番モデルも、法線の確認には十分です。**

テクスチャは第20章の `texconv` で DDS に変換しておきます。

### Step 2:読み込む

```
[Info ] ObjLoader.cpp(178): obj parsed: 8146 positions, 8146 normals, 8501 uvs, 16292 triangles
[Info ] ObjLoader.cpp(245): deduplicated: 16292 triangles -> 9573 vertices (9573 unique)
[Info ] ObjLoader.cpp(312): tangents computed
[Info ] MtlLoader.cpp(88): mtl loaded: 3 material(s)
[Info ] MeshUpload.cpp(64): mesh uploaded: 9573 vertices, 48876 indices (32bit)
```

**重複排除の効果に注目してください。** 三角形 16292 個 × 3 = 48876 頂点になるはずが、**9573 頂点に減っています。** 5 分の 1 です。

### Step 3:法線を確認する

**デバッグ描画に切り替えます。**

```cpp
if (input.WasKeyPressed('N'))
{
    m_debugMode = (m_debugMode == DebugMode::Normal)
                ? DebugMode::None : DebugMode::Normal;
}
```

**第22章のカメラで、モデルの周りを回ってください。**

- 上を向いた面が緑
- 右を向いた面が赤
- 手前を向いた面が青

**裏返っている面があれば、暗い色になります。**

### Step 4:スムージング角を変える

```cpp
LoadOptions options{};
options.smoothingAngle = 0.0f;      // すべてフラット
```

**面ごとにはっきり分かれます。** ローポリゴンの見た目になります。

```cpp
options.smoothingAngle = 180.0f;    // すべて平均
```

**角が丸くなります。** 立方体のような形状では、明らかにおかしく見えます。

**60 度に戻すと、曲面は滑らか、角は鋭くなります。**

### Step 5:UV の向きを確認する

```cpp
options.flipV = true;
```

**テクスチャが上下反転します。** 元が正しかったか間違っていたかが、これで分かります。

**文字が含まれるテクスチャを使うと、判定が容易です。**

### Step 6:重複排除を無効にしてみる

```cpp
// 常に新しい頂点を作る
// if (const auto it = lookup.find(fv); it != lookup.end()) { ... }
```

```
[Info ] ObjLoader.cpp(245): deduplicated: 16292 triangles -> 48876 vertices
```

**頂点数が 5 倍になります。**

- 頂点バッファのサイズが 5 倍
- 頂点シェーダーの実行回数が増える(頂点キャッシュが効かない)
- インデックスが 65535 を超え、32bit になる

**フレームレートを比べてみてください。** モデルが大きいほど差が出ます。

**確認したら元に戻してください。**

### Step 7:壊れた OBJ を読ませる

**範囲チェックが効いているかを確かめます。**

テキストエディタで、`f` の行のインデックスを巨大な値に書き換えてください。

```
f 99999/1/1 2/2/1 3/3/1
```

**クラッシュせず、その頂点が原点に置かれるだけになります。**

**チェックを外すと、その場でクラッシュします。** 実在するモデルファイルには、壊れたものが混ざっています。

---

### 本章の達成状態

- [ ] OBJ パーサを実装した(省略形・負のインデックス対応)
- [ ] 多角形をファン分割している
- [ ] `std::from_chars` を使っている(ロケール非依存)
- [ ] MTL パーサを実装した
- [ ] テクスチャパスのオプションを飛ばしている
- [ ] `.dds` への差し替え検索を入れた
- [ ] ハッシュマップで頂点を重複排除した
- [ ] インデックスの範囲チェックを入れた
- [ ] 頂点数に応じて 16bit / 32bit を選んでいる
- [ ] 面積で重み付けした法線を計算した
- [ ] スムージング角による分割を実装した
- [ ] 位置の量子化でグループ化している
- [ ] 接空間を計算し、`w` に符号を入れた
- [ ] 縮退した UV を処理している
- [ ] **3D モデルが表示された**
- [ ] Step 3 で法線の向きを確認した

---

## トラブルシューティング

| 症状 | 原因 | 対処 |
|---|---|---|
| 読み込みで落ちる | インデックスの範囲外 | 範囲チェック(23.3.2) |
| 小数がおかしい | ロケールの影響 | `std::from_chars`(23.2.2) |
| 面が欠ける | 多角形を分割していない | ファン分割(23.2.3) |
| 一部のモデルが読めない | 負のインデックス | 23.2.2 |
| テクスチャが見つからない | パスにオプションが混入 | 23.2.4 |
| テクスチャが上下逆 | UV の V 軸 | `flipV`(23.4.4) |
| モデルが真っ黒 | 法線が裏返り | 法線デバッグ表示(23.4.3) |
| 角が丸い | スムージング角が大きすぎ | 60 度前後に(23.4.2) |
| すべてフラット | スムージング角が 0 | 同上 |
| 頂点数が異常に多い | 重複排除が効いていない | 23.3.2 |
| 法線に NaN が出る | ゼロ除算 | 縮退のチェック(23.5.2) |
| 面が裏返って見える | 巻き順 | 第16章 16.2.3 節 |

---

## まとめ

**1. GPU は 1 種類のインデックスしか扱えない。**
OBJ は位置・UV・法線を別々に参照するため、**組み合わせごとに頂点を作る**必要があります。これが重複排除の本質です。

**2. 立方体が 24 頂点になる理由が、ここで確定した。**
第16章 16.1.1 節で予告した通りです。**位置が同じでも、法線や UV が違えば別の頂点です。**

**3. 法線は面積で重み付けして平均する。**
正規化しない外積を足すだけで実現できます。細かい三角形に引っ張られるのを防げます。

**4. スムージング角で「角を立てるか丸めるか」を制御する。**
60 度前後が実用的な既定値です。**立方体は角が立ち、球は滑らかになります。**

**5. 接空間は `w` に符号を入れて 4 成分で持つ。**
従法線はシェーダー側で復元できます。頂点あたり 8 バイトの節約になります。

**6. 壊れたファイルは実在する。**
範囲チェックと縮退の処理は、飾りではありません。**チェックがなければ、その場でクラッシュします。**

**7. 動かせるから確認できる。**
第22章でカメラを作っていなければ、法線が裏返っていることに気づけません。**「動かせるようにしてから進む」という判断が、ここで効いています。**

次章ではライティングを実装します。Lambert と Blinn-Phong、光源の種類、そして**法線変換の落とし穴**。第17章 17.4.4 節で `InverseAffine` を用意しておいた理由が、そこで明らかになります。

---

## 参考リンク

| 内容 | URL |
|---|---|
| Wavefront OBJ 形式の仕様 | https://www.fileformat.info/format/wavefrontobj/egff.htm |
| MTL 形式の仕様 | https://www.fileformat.info/format/material/ |
| glTF 2.0 仕様 | https://registry.khronos.org/glTF/specs/2.0/glTF-2.0.html |
| glTF サンプルモデル | https://github.com/KhronosGroup/glTF-Sample-Models |
| 接空間の計算(Lengyel の手法) | https://terathon.com/blog/tangent-space.html |
| `std::from_chars` | https://ja.cppreference.com/w/cpp/utility/from_chars |
