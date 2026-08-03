# 第17章 数学ライブラリを自作する

前章の立方体は、CPU 側で座標を焼き込んだ「にせもの」でした。回転させたければ頂点バッファを作り直すしかなく、DEFAULT ヒープに置いた時点でそれすらできません。

正しくやるには、**頂点シェーダーで行列変換をする**必要があります。そのためには行列が要ります。

**本章は Direct3D の API をほとんど呼びません。** ひたすら数式を C++ に落とし込む章です。地味ですが、ここを曖昧にしたまま進むと、**第18章以降ずっと「なぜか映らない」「なぜか裏返る」と戦い続ける**ことになります。

そして本章には、3D グラフィックスで最も多くの人を悩ませてきた話題が含まれます。**行優先か列優先か。** この混乱は、実は**2 つの別々の問題が混同されている**ことから生じます。17.3 節でこれを分離します。

**本章のゴール**
`Vector2/3/4` と `Matrix4x4` を自作し、平行移動・回転・スケール・ビュー・射影の各行列を実装する。手計算した値と照合する単体テストを書き、すべて通す。

---

## 17.1 DirectXMath を使わない前提の確認

### 17.1.1 DirectXMath は良いライブラリです

最初に公平に書いておきます。**DirectXMath は優れています。**

- Windows SDK に含まれており、追加の入手作業が不要
- SSE / AVX による SIMD 最適化が施されている
- 十数年にわたって使われ、検証されている
- Microsoft のサンプルコードがすべてこれを使っている

**実務では、素直に使うべき場面が多いはずです。** 本書の自作版より、確実に速く動きます。

### 17.1.2 それでも自作する理由

第1章 1.3 節の線引きを思い出してください。本書が自作するのは、**レンダラの設計判断を伴うもの**でした。

**数学ライブラリは、まさにその典型です。**

| 決めなければならないこと | 選択肢 |
|---|---|
| ベクトルと行列の掛け算の向き | 行ベクトル(`v * M`)/ 列ベクトル(`M * v`) |
| メモリ上の並び | 行優先 / 列優先 |
| 座標系の手 | 左手系 / 右手系 |
| 深度の範囲 | [0, 1] / [-1, 1] |
| 回転の正方向 | 時計回り / 反時計回り |

**これらの選択は、シェーダーの書き方、モデルの読み込み、カリングの向き、深度テストの比較関数まで波及します。** ライブラリを使うと、選択が「既に決まっているもの」として見えなくなります。

そして、**自作すると全部を明示的に決めることになります。** 本章の価値の大半は、実装そのものより、この「決める」作業にあります。

### 17.1.3 実務上の理由もある

もう一つ、地味ですが無視できない理由があります。**`XMVECTOR` と `XMMATRIX` は 16 バイト境界にアラインされている必要があります。**

そのため、DirectXMath では 2 種類の型を使い分けます。

| 型 | 用途 |
|---|---|
| `XMVECTOR` / `XMMATRIX` | 計算用。**アライン必須** |
| `XMFLOAT3` / `XMFLOAT4X4` | 保存用。アライン不要 |

構造体のメンバや `std::vector` の要素にするには、後者を使い、計算のたびに `XMLoadFloat3` / `XMStoreFloat3` で往復させます。**慣れれば自然ですが、初学者には確実につまずきポイントです。**

**本書の自作版は SIMD を使わないので、この区別が不要です。** どこにでも置け、`std::vector` にも入り、`constexpr` にもできます。

### 17.1.4 本書が採用する規約

**先に結論を書きます。** 以後、この規約で一貫します。

| 項目 | 本書の選択 | 理由 |
|---|---|---|
| ベクトルの向き | **行ベクトル** `v' = v * M` | D3D 系の文献と一致 |
| メモリ配置 | **行優先** `m[row][col]` | 上記と組み合わせて自然 |
| 座標系 | **左手系**(+Z が奥) | D3D の慣習 |
| 深度範囲 | **[0, 1]** | D3D の仕様。選択の余地なし |
| 変換の合成 | `World * View * Projection` | 適用順に左から右へ |
| HLSL 側 | `row_major float4x4` + `mul(v, M)` | **転置を一切しない** |

**最後の行が重要です。** 多くのサンプルコードは、行列を GPU に送る前に転置しています。**本書は転置しません。** 理由は 17.3.4 節で説明します。

---

## 17.2 `Vector2/3/4`

### 17.2.1 設計方針

```cpp
// src/Math/Vector.h
#pragma once
#include "std_import.h"

namespace Math
{
    inline constexpr float Pi      = std::numbers::pi_v<float>;
    inline constexpr float TwoPi   = Pi * 2.0f;
    inline constexpr float HalfPi  = Pi * 0.5f;

    [[nodiscard]] constexpr float ToRadians(float degrees) noexcept
    {
        return degrees * (Pi / 180.0f);
    }

    [[nodiscard]] constexpr float ToDegrees(float radians) noexcept
    {
        return radians * (180.0f / Pi);
    }
}
```

`std::numbers::pi_v<float>` は C++20 の機能です。**`3.14159265f` を手で書く必要はありません。**

```cpp
namespace Math
{
    struct Vector3
    {
        float x = 0.0f;
        float y = 0.0f;
        float z = 0.0f;

        constexpr Vector3() noexcept = default;
        constexpr Vector3(float x_, float y_, float z_) noexcept
            : x(x_), y(y_), z(z_) {}

        [[nodiscard]] constexpr bool operator==(const Vector3&) const noexcept
            = default;

        constexpr Vector3& operator+=(const Vector3& v) noexcept
        {
            x += v.x; y += v.y; z += v.z;
            return *this;
        }
        constexpr Vector3& operator-=(const Vector3& v) noexcept
        {
            x -= v.x; y -= v.y; z -= v.z;
            return *this;
        }
        constexpr Vector3& operator*=(float s) noexcept
        {
            x *= s; y *= s; z *= s;
            return *this;
        }
    };

    //--- 二項演算 ---
    [[nodiscard]] constexpr Vector3 operator+(Vector3 a, const Vector3& b) noexcept
    {
        return a += b;
    }
    [[nodiscard]] constexpr Vector3 operator-(Vector3 a, const Vector3& b) noexcept
    {
        return a -= b;
    }
    [[nodiscard]] constexpr Vector3 operator-(const Vector3& v) noexcept
    {
        return { -v.x, -v.y, -v.z };
    }
    [[nodiscard]] constexpr Vector3 operator*(Vector3 v, float s) noexcept
    {
        return v *= s;
    }
    [[nodiscard]] constexpr Vector3 operator*(float s, Vector3 v) noexcept
    {
        return v *= s;
    }
}
```

**`operator==` を `= default` にしている**点に注目してください。C++20 の機能で、メンバごとの比較が自動生成されます。

> **浮動小数点の `==` について**
>
> 誤差を考えれば、`==` は本来危険です。それでも定義しているのは、**`static_assert` によるコンパイル時テストで使いたいから**です(17.7 節)。
>
> 実行時の比較には、許容誤差つきの関数を別途用意します。

### 17.2.2 基本演算

```cpp
[[nodiscard]] constexpr float Dot(const Vector3& a, const Vector3& b) noexcept
{
    return a.x * b.x + a.y * b.y + a.z * b.z;
}

[[nodiscard]] constexpr Vector3 Cross(const Vector3& a, const Vector3& b) noexcept
{
    return {
        a.y * b.z - a.z * b.y,
        a.z * b.x - a.x * b.z,
        a.x * b.y - a.y * b.x,
    };
}

[[nodiscard]] constexpr float LengthSquared(const Vector3& v) noexcept
{
    return Dot(v, v);
}

[[nodiscard]] inline float Length(const Vector3& v) noexcept
{
    return std::sqrt(LengthSquared(v));
}

[[nodiscard]] inline Vector3 Normalize(const Vector3& v) noexcept
{
    const float lengthSq = LengthSquared(v);
    if (lengthSq <= 0.0f)
    {
        return {};       // ゼロベクトルはゼロのまま返す
    }
    return v * (1.0f / std::sqrt(lengthSq));
}
```

**`LengthSquared` を分けている**のは、単なる最適化ではありません。**「どちらが長いか」を比べるだけなら平方根は不要**です。第25章で描画順のソートをするとき、これを使います。

`Length` と `Normalize` が `constexpr` でないのは、**`std::sqrt` が C++23 では `constexpr` でない**からです(C++26 で `constexpr` になります)。

> **`Cross` の向きを確認する**
>
> `Cross(x軸, y軸)` が `z軸` になることを確かめてください。
>
> ```
> Cross((1,0,0), (0,1,0)) = (0*0-0*1, 0*0-1*0, 1*1-0*0) = (0, 0, 1)
> ```
>
> **これが左手系での正しい向きです。** 左手の親指を +X、人差し指を +Y に向けると、中指が +Z(奥)を指します。
>
> 第16章 16.2.3 節で使った「外積が外向きになるように並べる」規則は、この定義に基づいています。

### 17.2.3 `Vector4` と `Vector2`

`Vector4` は同型に定義します。`Vector3` からの変換を用意しておくと便利です。

```cpp
struct Vector4
{
    float x = 0.0f, y = 0.0f, z = 0.0f, w = 0.0f;

    constexpr Vector4() noexcept = default;
    constexpr Vector4(float x_, float y_, float z_, float w_) noexcept
        : x(x_), y(y_), z(z_), w(w_) {}

    // 点として拡張(w = 1)/ 方向として拡張(w = 0)
    constexpr explicit Vector4(const Vector3& v, float w_ = 1.0f) noexcept
        : x(v.x), y(v.y), z(v.z), w(w_) {}

    [[nodiscard]] constexpr Vector3 XYZ() const noexcept { return { x, y, z }; }

    [[nodiscard]] constexpr bool operator==(const Vector4&) const noexcept
        = default;
};
```

**`w` の値には意味があります。**

| `w` | 意味 |
|---|---|
| `1` | **点**。平行移動の影響を受ける |
| `0` | **方向**。平行移動の影響を受けない |

法線や光線の向きは `w = 0` です。**ここを間違えると、モデルを動かしたときに法線まで一緒に移動して、ライティングが壊れます**(第22章)。

### 17.2.4 ログに出せるようにする

デバッグのために、`std::format` で出力できるようにします。

```cpp
template <>
struct std::formatter<Math::Vector3, wchar_t>
{
    constexpr auto parse(std::wformat_parse_context& ctx) noexcept
    {
        return ctx.begin();
    }

    auto format(const Math::Vector3& v, std::wformat_context& ctx) const
    {
        return std::format_to(ctx.out(), L"({:7.3f}, {:7.3f}, {:7.3f})",
                              v.x, v.y, v.z);
    }
};
```

これで第6章のログマクロがそのまま使えます。

```cpp
LOG_INFO(L"camera position = {}", cameraPos);
```

```
[Info ] Camera.cpp(42): camera position = (  0.000,   2.000,  -5.000)
```

**桁を揃えて出力している**のは、複数行に並んだときに読みやすくするためです。行列を出力するときに効いてきます。

---

## 17.3 行優先 / 列優先の混乱を断つ

**本章でもっとも重要な節です。**

### 17.3.1 混同されている 2 つの問題

「D3D は行優先、OpenGL は列優先」という説明をよく見かけます。**この説明は、2 つの別々の話を 1 つにまとめてしまっています。**

**問題 A:数学の記法 —— ベクトルは行か列か**

```
行ベクトル:  v' = v * M        v は 1×4、M は 4×4
列ベクトル:  v' = M * v        M は 4×4、v は 4×1
```

**問題 B:メモリの並び —— 2 次元配列をどう詰めるか**

```
行優先(row-major):    m[0][0], m[0][1], m[0][2], m[0][3], m[1][0], ...
列優先(column-major): m[0][0], m[1][0], m[2][0], m[3][0], m[0][1], ...
```

**この 2 つは完全に独立です。** 4 通りの組み合わせがありえます。

### 17.3.2 なぜ混同されるのか

**「行ベクトル × 行優先」と「列ベクトル × 列優先」は、同じ変換に対して同じメモリ内容になるから**です。

平行移動 (10, 20, 30) を例にとります。

**行ベクトル × 行優先**

```
      | 1   0   0   0 |
M  =  | 0   1   0   0 |        平行移動成分は「4 行目」
      | 0   0   1   0 |
      | 10  20  30  1 |

メモリ: 1,0,0,0, 0,1,0,0, 0,0,1,0, 10,20,30,1
```

**列ベクトル × 列優先**

```
      | 1   0   0   10 |
M  =  | 0   1   0   20 |       平行移動成分は「4 列目」
      | 0   0   1   30 |
      | 0   0   0   1  |

メモリ: 1,0,0,0, 0,1,0,0, 0,0,1,0, 10,20,30,1     ← 同じ!
```

**バイト列は完全に一致します。** だから、片方の流儀で書かれたコードを、もう片方の流儀に移植するとき、**掛け算の順序を逆にするだけで動いてしまう**ことがよくあります。

そして、それが「行優先と列優先は掛ける順が逆」という**不正確な理解**を広めました。

### 17.3.3 本書の選択を明示する

**問題 A:行ベクトル。`v' = v * M`**

```cpp
Vector4 result = Transform(v, matrix);   // v * M
```

**問題 B:行優先。`m[row][col]`**

```cpp
struct Matrix4x4
{
    float m[4][4];   // m[行][列]
};
```

**平行移動成分は `m[3][0]`, `m[3][1]`, `m[3][2]` に入ります。**

この組み合わせを選ぶ理由は、**D3D 系の文献・サンプル・DirectXMath がすべてこの流儀だから**です。他の資料を参照するとき、読み替えが不要になります。

**合成の順序**は、適用したい順に左から右です。

```cpp
Matrix4x4 mvp = world * view * projection;
```

「まずワールドへ、次にビューへ、最後に射影へ」という日本語の順序と、コードの並びが一致します。**これは行ベクトル流儀の大きな利点です。** 列ベクトル流儀では `P * V * W` と逆順に書くことになります。

### 17.3.4 HLSL 側をどう合わせるか

**ここが実際に事故が起きる場所です。**

HLSL は、定数バッファ内の `float4x4` を **既定で列優先に詰めます。** つまり、行優先で並べたメモリをそのまま送ると、**シェーダー側では転置された行列として解釈されます。**

**対処法は 3 つあります。**

| 方法 | C++ 側 | HLSL 側 |
|---|---|---|
| **A.(本書)** | そのまま送る | `row_major float4x4` と宣言 |
| B. | **転置してから送る** | 既定のまま、`mul(v, M)` |
| C. | そのまま送る | 既定のまま、`mul(M, v)` |

**B は最も広く使われています。** Microsoft のサンプルの多くがこの形です。しかし、**転置という操作が 1 つ挟まる**ぶん、間違いの余地が生まれます。「転置したっけ?」を毎回考えることになります。

**C は動きますが、C++ と HLSL で掛け算の順序が逆になります。** 読むたびに頭を切り替える必要があり、混乱の元です。

**本書は A を採ります。**

```hlsl
cbuffer SceneConstants : register(b0)
{
    row_major float4x4 worldViewProj;
};

VSOutput VSMain(VSInput input)
{
    VSOutput output;
    output.position = mul(float4(input.position, 1.0f), worldViewProj);
    //                    ^^^^^^^^^^^^^^^^^^^^^^^^^^  ^^^^^^^^^^^^^^
    //                    v                        *  M
    return output;
}
```

**C++ 側と HLSL 側が、まったく同じ形になります。**

```cpp
// C++
Vector4 clip = Transform(Vector4(position, 1.0f), worldViewProj);
```

```hlsl
// HLSL
float4 clip = mul(float4(position, 1.0f), worldViewProj);
```

**転置は一切登場しません。** 第18章で定数バッファを実装するとき、この選択の恩恵が分かります。

> **`-Zpr` オプションでも同じことができる**
>
> DXC には `-Zpr`(pack matrices in row-major order)というオプションがあり、すべての行列を行優先で詰めさせられます。`row_major` を書く手間が省けます。
>
> **本書は `row_major` を明示します。** コンパイルオプションに依存すると、そのシェーダーを別のプロジェクトにコピーしたときに静かに壊れるからです。**ソースを見れば分かる形にしておきます。**

---

## 17.4 `Matrix4x4` と基本の変換

### 17.4.1 型の定義

```cpp
// src/Math/Matrix.h
#pragma once
#include "std_import.h"
#include "Math/Vector.h"

namespace Math
{
    struct Matrix4x4
    {
        // m[行][列]。平行移動成分は m[3][0..2]。
        float m[4][4]{};

        //-------------------------------------------------------
        // 要素アクセス。
        // C++23 の "deducing this" により、const 版と非 const 版を
        // 1 つの定義で兼ねる。
        //-------------------------------------------------------
        template <typename Self>
        [[nodiscard]] constexpr auto&& operator()(
            this Self&& self, int row, int col) noexcept
        {
            return std::forward<Self>(self).m[row][col];
        }

        [[nodiscard]] constexpr bool operator==(const Matrix4x4&) const noexcept
            = default;

        [[nodiscard]] static constexpr Matrix4x4 Identity() noexcept
        {
            Matrix4x4 result{};
            result.m[0][0] = 1.0f;
            result.m[1][1] = 1.0f;
            result.m[2][2] = 1.0f;
            result.m[3][3] = 1.0f;
            return result;
        }
    };
}
```

**`operator()` に C++23 の「deducing this」を使いました。**

従来なら、こう書く必要がありました。

```cpp
// C++20 まで:同じ中身を 2 回書く
constexpr float&       operator()(int r, int c)       { return m[r][c]; }
constexpr const float& operator()(int r, int c) const { return m[r][c]; }
```

**deducing this を使えば 1 つで済みます。** 引数の `this Self&& self` が呼び出し元の値カテゴリと const 性を受け取るので、戻り値がそれに追従します。

> **コンパイルが通らない場合**
>
> deducing this は比較的新しい機能です。環境によっては、上の 2 行版に書き換えてください。**本書の他の部分には影響しません。**

### 17.4.2 掛け算

**行ベクトル・行優先での定義です。**

```cpp
[[nodiscard]] constexpr Matrix4x4 operator*(
    const Matrix4x4& a, const Matrix4x4& b) noexcept
{
    Matrix4x4 result{};
    for (int row = 0; row < 4; ++row)
    {
        for (int col = 0; col < 4; ++col)
        {
            float sum = 0.0f;
            for (int k = 0; k < 4; ++k)
            {
                sum += a.m[row][k] * b.m[k][col];
            }
            result.m[row][col] = sum;
        }
    }
    return result;
}
```

ベクトルとの掛け算です。

```cpp
//--- 4 成分ベクトル × 行列 ---
[[nodiscard]] constexpr Vector4 Transform(
    const Vector4& v, const Matrix4x4& mat) noexcept
{
    return {
        v.x*mat.m[0][0] + v.y*mat.m[1][0] + v.z*mat.m[2][0] + v.w*mat.m[3][0],
        v.x*mat.m[0][1] + v.y*mat.m[1][1] + v.z*mat.m[2][1] + v.w*mat.m[3][1],
        v.x*mat.m[0][2] + v.y*mat.m[1][2] + v.z*mat.m[2][2] + v.w*mat.m[3][2],
        v.x*mat.m[0][3] + v.y*mat.m[1][3] + v.z*mat.m[2][3] + v.w*mat.m[3][3],
    };
}

//--- 点として変換(w = 1) ---
[[nodiscard]] constexpr Vector3 TransformPoint(
    const Vector3& v, const Matrix4x4& mat) noexcept
{
    return Transform(Vector4(v, 1.0f), mat).XYZ();
}

//--- 方向として変換(w = 0。平行移動を受けない) ---
[[nodiscard]] constexpr Vector3 TransformDirection(
    const Vector3& v, const Matrix4x4& mat) noexcept
{
    return Transform(Vector4(v, 0.0f), mat).XYZ();
}
```

**`TransformPoint` と `TransformDirection` を分けている**ことが重要です。名前で区別しておけば、法線を点として変換してしまう事故を防げます。

### 17.4.3 平行移動・スケール・回転

```cpp
[[nodiscard]] constexpr Matrix4x4 Translation(float x, float y, float z) noexcept
{
    Matrix4x4 result = Matrix4x4::Identity();
    result.m[3][0] = x;      // ← 行ベクトル流儀では 4 行目
    result.m[3][1] = y;
    result.m[3][2] = z;
    return result;
}

[[nodiscard]] constexpr Matrix4x4 Scaling(float x, float y, float z) noexcept
{
    Matrix4x4 result{};
    result.m[0][0] = x;
    result.m[1][1] = y;
    result.m[2][2] = z;
    result.m[3][3] = 1.0f;
    return result;
}
```

回転行列です。**左手系での正の回転**を定義します。

```cpp
[[nodiscard]] inline Matrix4x4 RotationX(float radians) noexcept
{
    const float c = std::cos(radians);
    const float s = std::sin(radians);

    Matrix4x4 r = Matrix4x4::Identity();
    r.m[1][1] =  c;  r.m[1][2] = s;
    r.m[2][1] = -s;  r.m[2][2] = c;
    return r;
}

[[nodiscard]] inline Matrix4x4 RotationY(float radians) noexcept
{
    const float c = std::cos(radians);
    const float s = std::sin(radians);

    Matrix4x4 r = Matrix4x4::Identity();
    r.m[0][0] = c;  r.m[0][2] = -s;
    r.m[2][0] = s;  r.m[2][2] =  c;
    return r;
}

[[nodiscard]] inline Matrix4x4 RotationZ(float radians) noexcept
{
    const float c = std::cos(radians);
    const float s = std::sin(radians);

    Matrix4x4 r = Matrix4x4::Identity();
    r.m[0][0] =  c;  r.m[0][1] = s;
    r.m[1][0] = -s;  r.m[1][1] = c;
    return r;
}
```

**符号の位置は間違えやすい箇所です。** 覚えるのではなく、**テストで確認します**(17.7 節)。

たとえば `RotationY(90°)` は、+Z 方向の点を +X 方向へ移すはずです。

```
v = (0, 0, 1)
v * RotationY(90°) = (0*0 + 0*0 + 1*1, 0, 0*(-1) + 0*0 + 1*0) = (1, 0, 0)
```

**これを `static_assert` ではなく実行時テストで確かめます**(`std::sin` が `constexpr` でないため)。

### 17.4.4 転置とアフィン逆行列

```cpp
[[nodiscard]] constexpr Matrix4x4 Transpose(const Matrix4x4& mat) noexcept
{
    Matrix4x4 result{};
    for (int row = 0; row < 4; ++row)
    {
        for (int col = 0; col < 4; ++col)
        {
            result.m[row][col] = mat.m[col][row];
        }
    }
    return result;
}
```

**逆行列は、用途を限れば簡単に書けます。**

平行移動・回転・スケールだけで構成された行列(アフィン変換)なら、一般的な余因子展開は不要です。

```cpp
//---------------------------------------------------------------
// 平行移動・回転・スケールのみで構成された行列の逆行列。
// せん断や射影を含む行列には使えない。
//---------------------------------------------------------------
[[nodiscard]] inline Matrix4x4 InverseAffine(const Matrix4x4& mat) noexcept
{
    //--- 上 3×3 の逆行列(スケールつき回転)---
    Matrix4x4 result = Matrix4x4::Identity();

    // 各軸のスケールの二乗
    const float sx = mat.m[0][0]*mat.m[0][0] + mat.m[0][1]*mat.m[0][1]
                   + mat.m[0][2]*mat.m[0][2];
    const float sy = mat.m[1][0]*mat.m[1][0] + mat.m[1][1]*mat.m[1][1]
                   + mat.m[1][2]*mat.m[1][2];
    const float sz = mat.m[2][0]*mat.m[2][0] + mat.m[2][1]*mat.m[2][1]
                   + mat.m[2][2]*mat.m[2][2];

    const float ix = (sx > 0.0f) ? 1.0f / sx : 0.0f;
    const float iy = (sy > 0.0f) ? 1.0f / sy : 0.0f;
    const float iz = (sz > 0.0f) ? 1.0f / sz : 0.0f;

    // 転置してスケールの二乗で割る
    result.m[0][0] = mat.m[0][0] * ix;
    result.m[0][1] = mat.m[1][0] * iy;
    result.m[0][2] = mat.m[2][0] * iz;
    result.m[1][0] = mat.m[0][1] * ix;
    result.m[1][1] = mat.m[1][1] * iy;
    result.m[1][2] = mat.m[2][1] * iz;
    result.m[2][0] = mat.m[0][2] * ix;
    result.m[2][1] = mat.m[1][2] * iy;
    result.m[2][2] = mat.m[2][2] * iz;

    //--- 平行移動成分 ---
    const Vector3 t{ mat.m[3][0], mat.m[3][1], mat.m[3][2] };
    result.m[3][0] = -(t.x*result.m[0][0] + t.y*result.m[1][0] + t.z*result.m[2][0]);
    result.m[3][1] = -(t.x*result.m[0][1] + t.y*result.m[1][1] + t.z*result.m[2][1]);
    result.m[3][2] = -(t.x*result.m[0][2] + t.y*result.m[1][2] + t.z*result.m[2][2]);

    return result;
}
```

**第22章で法線を変換するとき、`Transpose(InverseAffine(world))` が必要になります。** ここで用意しておきます。

---

## 17.5 ビュー行列 `LookAt`

### 17.5.1 何をする行列か

**ワールド空間の座標を、カメラを原点とする空間(ビュー空間)へ移す行列**です。

```
ワールド空間              ビュー空間
                          カメラが原点
   ● カメラ         →     カメラの向きが +Z
   ▲ 物体                 物体は前方に
```

「カメラを動かす」のではなく、**世界のほうを動かして、カメラが原点で +Z を向いている状態にする**と考えると分かりやすくなります。

### 17.5.2 実装

```cpp
//---------------------------------------------------------------
// 左手系のビュー行列。
//   eye    : カメラの位置
//   target : 注視点
//   up     : 上方向(通常は (0,1,0))
//---------------------------------------------------------------
[[nodiscard]] inline Matrix4x4 LookAtLH(
    const Vector3& eye,
    const Vector3& target,
    const Vector3& up) noexcept
{
    //--- カメラの 3 軸を求める ---
    const Vector3 zaxis = Normalize(target - eye);       // 前方
    const Vector3 xaxis = Normalize(Cross(up, zaxis));   // 右
    const Vector3 yaxis = Cross(zaxis, xaxis);           // 上(直交化済み)

    Matrix4x4 result = Matrix4x4::Identity();

    //--- 回転成分(3 軸を列として並べる = 転置した回転)---
    result.m[0][0] = xaxis.x;  result.m[0][1] = yaxis.x;  result.m[0][2] = zaxis.x;
    result.m[1][0] = xaxis.y;  result.m[1][1] = yaxis.y;  result.m[1][2] = zaxis.y;
    result.m[2][0] = xaxis.z;  result.m[2][1] = yaxis.z;  result.m[2][2] = zaxis.z;

    //--- 平行移動成分 ---
    result.m[3][0] = -Dot(xaxis, eye);
    result.m[3][1] = -Dot(yaxis, eye);
    result.m[3][2] = -Dot(zaxis, eye);

    return result;
}
```

**軸の求め方に注目してください。**

```cpp
const Vector3 zaxis = Normalize(target - eye);
```

**左手系では、カメラは +Z 方向を向きます。** だから `target - eye` がそのまま Z 軸です。右手系なら `eye - target` になります。

```cpp
const Vector3 yaxis = Cross(zaxis, xaxis);
```

**`up` をそのまま Y 軸にしていません。** `up` は「だいたい上」を示すヒントにすぎず、`zaxis` と直交しているとは限りません。外積を 2 回とることで、**必ず直交する 3 軸**が得られます。

> **`up` と視線が平行だと壊れる**
>
> 真上を見上げる場合、`up = (0,1,0)` と `zaxis` が平行になり、`Cross(up, zaxis)` がゼロベクトルになります。**正規化で 0 除算が起きます。**
>
> 第22章でカメラを実装するとき、仰角に制限をかけて回避します。**行列側で対処するのではなく、入力側で防ぐのが定石です。**

---

## 17.6 射影行列と D3D の深度範囲

### 17.6.1 深度は [0, 1]

**Direct3D の深度範囲は [0, 1] です。** OpenGL の [-1, 1] とは違います。

| | 近クリップ面 | 遠クリップ面 |
|---|---|---|
| **Direct3D** | **0.0** | **1.0** |
| OpenGL(既定) | -1.0 | 1.0 |

**これは選択の余地がない仕様です。** 第11章 11.6 節でビューポートの `MinDepth` / `MaxDepth` を `0.0` / `1.0` にしたのも、これに合わせたものです。

OpenGL 向けの射影行列を持ってくると、**手前半分が消えます。** ネット上の資料を参照するときは、どちらの流儀か必ず確認してください。

### 17.6.2 透視投影

```cpp
//---------------------------------------------------------------
// 左手系・深度範囲 [0,1] の透視投影行列。
//   fovY        : 垂直方向の視野角(ラジアン)
//   aspectRatio : 幅 / 高さ
//   nearZ, farZ : クリップ面までの距離(どちらも正)
//---------------------------------------------------------------
[[nodiscard]] inline Matrix4x4 PerspectiveFovLH(
    float fovY, float aspectRatio, float nearZ, float farZ) noexcept
{
    const float h = 1.0f / std::tan(fovY * 0.5f);
    const float w = h / aspectRatio;
    const float range = farZ / (farZ - nearZ);

    Matrix4x4 result{};
    result.m[0][0] = w;
    result.m[1][1] = h;
    result.m[2][2] = range;
    result.m[2][3] = 1.0f;               // ← w に z を入れる
    result.m[3][2] = -range * nearZ;
    return result;
}
```

**`m[2][3] = 1.0f` が透視投影の本体です。**

この要素があるおかげで、変換後の `w` 成分にビュー空間の `z` が入ります。

```
出力の w = 入力の z
```

そして GPU は、ラスタライズの直前に **`xyz` を `w` で割ります**(透視除算)。遠くのものほど大きな値で割られるので、小さく見えます。**これが遠近感の正体です。**

### 17.6.3 深度が [0, 1] になることを確かめる

ビュー空間の点 `(0, 0, z, 1)` を変換します。

```
出力 z = z * range - range * nearZ = range * (z - nearZ)
出力 w = z
```

**近クリップ面(z = nearZ)のとき:**

```
出力 z = range * (nearZ - nearZ) = 0
除算後 = 0 / nearZ = 0        ✓
```

**遠クリップ面(z = farZ)のとき:**

```
出力 z = range * (farZ - nearZ) = farZ/(farZ-nearZ) * (farZ-nearZ) = farZ
除算後 = farZ / farZ = 1       ✓
```

**手計算で確認できました。** 17.7 節でこれをテストコードにします。

### 17.6.4 精度の話 —— `nearZ` を小さくしすぎない

**深度値の精度は、`nearZ` に強く依存します。**

除算後の深度は `range * (z - nearZ) / z` です。この値の変化率は、**`z` が小さいところで極端に大きく、遠くではほとんど変わりません。**

```
nearZ = 0.01, farZ = 1000 のとき

  z =   0.01 〜  0.02  → 深度 0.000 〜 0.500   ← 半分を使い切る
  z = 500    〜 1000   → 深度 0.999 〜 1.000   ← ほぼ差がない
```

**遠くの物体同士で深度が区別できなくなり、ちらつきます**(Z ファイティング)。第19章 19.1 節で実物を見ます。

**対策は単純です。`nearZ` をできるだけ大きくしてください。** `farZ` を小さくするより、はるかに効果があります。

> **Reversed-Z について**
>
> `nearZ` と `farZ` を入れ替えて、深度を 1 → 0 の向きに使う手法があります。浮動小数点は 0 付近で精度が高いため、**遠方の精度が劇的に改善します。**
>
> 深度バッファのクリア値と比較関数も反転させる必要があります。**第19章 19.5 節で扱います。**

### 17.6.5 平行投影

影の生成(第27章)などで使います。

```cpp
[[nodiscard]] constexpr Matrix4x4 OrthographicLH(
    float width, float height, float nearZ, float farZ) noexcept
{
    const float range = 1.0f / (farZ - nearZ);

    Matrix4x4 result{};
    result.m[0][0] = 2.0f / width;
    result.m[1][1] = 2.0f / height;
    result.m[2][2] = range;
    result.m[3][2] = -range * nearZ;
    result.m[3][3] = 1.0f;
    return result;
}
```

**`m[2][3]` がありません。** 透視除算が起きないので、遠くのものも同じ大きさで描かれます。

---

## 17.7 単体テストを書いて手計算と照合する

### 17.7.1 なぜテストを書くのか

**行列の符号を 1 つ間違えても、絵は出ます。** ただし、裏返っていたり、上下逆だったり、微妙にずれていたりします。

そして、**それがレンダリングのバグなのか、行列のバグなのかを切り分ける手段がありません。** 第22章でライティングが変になったとき、法線変換を疑うべきか行列を疑うべきか分からなくなります。

**行列は、絵を見る前に正しさを確定させておくべき部分です。**

### 17.7.2 コンパイル時にできることは `static_assert` で

**`constexpr` にしておいた恩恵が、ここで返ってきます。**

```cpp
// src/Math/MathTests.cpp

namespace
{
    using namespace Math;

    //=========================================================
    // コンパイル時テスト
    //   通らなければビルドが止まる。実行すら不要。
    //=========================================================

    //--- ベクトルの基本演算 ---
    static_assert(Vector3{1,2,3} + Vector3{4,5,6} == Vector3{5,7,9});
    static_assert(Vector3{4,5,6} - Vector3{1,2,3} == Vector3{3,3,3});
    static_assert(Vector3{1,2,3} * 2.0f == Vector3{2,4,6});
    static_assert(-Vector3{1,2,3} == Vector3{-1,-2,-3});

    //--- 内積 ---
    static_assert(Dot(Vector3{1,0,0}, Vector3{0,1,0}) == 0.0f);
    static_assert(Dot(Vector3{1,2,3}, Vector3{4,5,6}) == 32.0f);
    static_assert(LengthSquared(Vector3{3,4,0}) == 25.0f);

    //--- 外積(左手系の確認)---
    static_assert(Cross(Vector3{1,0,0}, Vector3{0,1,0}) == Vector3{0,0,1});
    static_assert(Cross(Vector3{0,1,0}, Vector3{0,0,1}) == Vector3{1,0,0});
    static_assert(Cross(Vector3{0,0,1}, Vector3{1,0,0}) == Vector3{0,1,0});

    //--- 単位行列 ---
    static_assert(Matrix4x4::Identity() * Matrix4x4::Identity()
                  == Matrix4x4::Identity());

    //--- 平行移動 ---
    static_assert(TransformPoint(Vector3{1,2,3}, Translation(10,20,30))
                  == Vector3{11,22,33});

    //--- 方向は平行移動の影響を受けない ---
    static_assert(TransformDirection(Vector3{1,0,0}, Translation(10,20,30))
                  == Vector3{1,0,0});

    //--- スケール ---
    static_assert(TransformPoint(Vector3{1,2,3}, Scaling(2,3,4))
                  == Vector3{2,6,12});

    //--- 合成順序:v * (A*B) == (v*A) * B ---
    constexpr Matrix4x4 kA = Translation(1, 2, 3);
    constexpr Matrix4x4 kB = Scaling(2, 2, 2);
    static_assert(TransformPoint(Vector3{1,1,1}, kA * kB)
                  == TransformPoint(TransformPoint(Vector3{1,1,1}, kA), kB));

    //--- 平行移動 → スケール の順で (1,1,1) は (4,6,8) になる ---
    static_assert(TransformPoint(Vector3{1,1,1}, kA * kB) == Vector3{4,6,8});
}
```

**ここまでのテストは、実行しなくても検証されます。** ビルドが通れば正しい、ということです。

最後のテストが特に意味を持ちます。`(1,1,1)` を `(1,2,3)` 動かすと `(2,3,4)`、それを 2 倍して `(4,6,8)`。**手計算と一致しました。** これで合成の順序が「左から順に適用」であることが確定します。

### 17.7.3 三角関数を含むものは実行時に

`std::sin` / `std::cos` / `std::tan` は C++23 では `constexpr` ではないため(C++26 で解禁されます)、これらを含む行列は実行時にテストします。

**小さなテスト用の道具を作ります。** 第6章のログと `std::source_location` を再利用します。

```cpp
namespace
{
    int g_failures = 0;

    void Check(bool condition,
               std::wstring_view expression,
               const std::source_location& where =
                   std::source_location::current())
    {
        if (!condition)
        {
            ++g_failures;
            Core::Log::WriteRaw(Core::LogLevel::Error, where,
                std::format(L"FAILED: {}", expression));
        }
    }

    bool NearEqual(float a, float b, float tolerance = 1e-5f) noexcept
    {
        return std::abs(a - b) <= tolerance;
    }

    bool NearEqual(const Vector3& a, const Vector3& b,
                   float tolerance = 1e-5f) noexcept
    {
        return NearEqual(a.x, b.x, tolerance)
            && NearEqual(a.y, b.y, tolerance)
            && NearEqual(a.z, b.z, tolerance);
    }
}

#define CHECK(expr) Check((expr), L#expr)
```

`L#expr` を直接書けない件は、第6章 6.3.4 節で用意した 2 段構えのマクロと同じ手法です。

### 17.7.4 回転のテスト

```cpp
void TestRotation()
{
    //--- Y 軸まわりに 90 度:+Z が +X になる ---
    {
        const auto r = RotationY(HalfPi);
        const auto v = TransformPoint(Vector3{0, 0, 1}, r);
        CHECK(NearEqual(v, Vector3{1, 0, 0}));
    }

    //--- X 軸まわりに 90 度:+Y が +Z になる ---
    {
        const auto r = RotationX(HalfPi);
        const auto v = TransformPoint(Vector3{0, 1, 0}, r);
        CHECK(NearEqual(v, Vector3{0, 0, 1}));
    }

    //--- Z 軸まわりに 90 度:+X が +Y になる ---
    {
        const auto r = RotationZ(HalfPi);
        const auto v = TransformPoint(Vector3{1, 0, 0}, r);
        CHECK(NearEqual(v, Vector3{0, 1, 0}));
    }

    //--- 360 度回すと元に戻る ---
    {
        const auto r = RotationY(TwoPi);
        const auto v = TransformPoint(Vector3{1, 2, 3}, r);
        CHECK(NearEqual(v, Vector3{1, 2, 3}, 1e-4f));
    }

    //--- 逆回転を掛けると単位行列になる ---
    {
        const auto a = RotationY(0.7f);
        const auto b = RotationY(-0.7f);
        const auto v = TransformPoint(Vector3{1, 2, 3}, a * b);
        CHECK(NearEqual(v, Vector3{1, 2, 3}, 1e-4f));
    }
}
```

**「+Z が +X になる」の 3 つは、覚えるべき値ではなく、決めた規約の確認です。** 符号を間違えていれば、ここで落ちます。

### 17.7.5 ビュー行列のテスト

```cpp
void TestLookAt()
{
    //--- 原点から 5 だけ手前にカメラを置く ---
    const auto view = LookAtLH(
        Vector3{0, 0, -5},   // eye
        Vector3{0, 0,  0},   // target
        Vector3{0, 1,  0});  // up

    //--- 原点は、カメラの 5 だけ前方に見える ---
    CHECK(NearEqual(TransformPoint(Vector3{0,0,0}, view), Vector3{0, 0, 5}));

    //--- カメラ自身は原点に移る ---
    CHECK(NearEqual(TransformPoint(Vector3{0,0,-5}, view), Vector3{0, 0, 0}));

    //--- 右にある点は、ビュー空間でも右 ---
    CHECK(NearEqual(TransformPoint(Vector3{1,0,0}, view), Vector3{1, 0, 5}));

    //--- 上にある点は、ビュー空間でも上 ---
    CHECK(NearEqual(TransformPoint(Vector3{0,1,0}, view), Vector3{0, 1, 5}));
}
```

**「カメラ自身が原点に移る」が、ビュー行列の定義そのものです。**

### 17.7.6 射影行列のテスト

17.6.3 節の手計算を、そのままコードにします。

```cpp
void TestProjection()
{
    constexpr float kNear = 1.0f;
    constexpr float kFar  = 100.0f;

    const auto proj = PerspectiveFovLH(
        ToRadians(90.0f), 1.0f, kNear, kFar);

    //--- 近クリップ面 → 深度 0 ---
    {
        const auto clip = Transform(Vector4{0, 0, kNear, 1}, proj);
        CHECK(NearEqual(clip.w, kNear));
        CHECK(NearEqual(clip.z / clip.w, 0.0f));
    }

    //--- 遠クリップ面 → 深度 1 ---
    {
        const auto clip = Transform(Vector4{0, 0, kFar, 1}, proj);
        CHECK(NearEqual(clip.w, kFar));
        CHECK(NearEqual(clip.z / clip.w, 1.0f, 1e-4f));
    }

    //--- 視野角 90 度なら、z と同じだけ横にずれた点が画面端 ---
    {
        const auto clip = Transform(Vector4{10, 0, 10, 1}, proj);
        CHECK(NearEqual(clip.x / clip.w, 1.0f, 1e-4f));
    }

    //--- 縦横比が効いている ---
    {
        const auto wide = PerspectiveFovLH(ToRadians(90.0f), 2.0f, kNear, kFar);
        const auto clip = Transform(Vector4{10, 0, 10, 1}, wide);
        CHECK(NearEqual(clip.x / clip.w, 0.5f, 1e-4f));
    }
}
```

**3 番目のテストが直感的です。** 視野角 90 度なら、視錐台の断面は正方形の一辺が 45 度ずつ開いた形になります。`z = 10` の位置で `x = 10` の点は、ちょうど右端に来ます。

### 17.7.7 テストを実行する

```cpp
bool Math::RunTests()
{
    g_failures = 0;

    LOG_INFO(L"--- math tests ---");

    TestRotation();
    TestLookAt();
    TestProjection();
    TestTransformChain();

    if (g_failures == 0)
    {
        LOG_INFO(L"all math tests passed");
        return true;
    }

    LOG_FATAL(L"{} math test(s) FAILED", g_failures);
    return false;
}
```

Debug ビルドの起動時に呼びます。

```cpp
int WINAPI wWinMain(...)
{
#if defined(_DEBUG)
    if (!Math::RunTests())
    {
        ::MessageBoxW(nullptr, L"数学ライブラリのテストに失敗しました。",
                      L"D3D12Book", MB_OK | MB_ICONERROR);
        return 1;
    }
#endif
    // ... 以下、通常の初期化 ...
}
```

**起動時に落ちるようにしてあります。** 「テストは失敗しているが、とりあえず動かす」を許すと、テストを書いた意味がなくなります。

---

## ✅ 本章のゴール:すべてのテストが通る

### Step 1:ビルドが通る

**`static_assert` のテスト(17.7.2 節)は、ビルドが通った時点で全部合格しています。**

わざと壊してみてください。

```cpp
static_assert(Cross(Vector3{1,0,0}, Vector3{0,1,0}) == Vector3{0,0,-1});   // ❌
```

```
error C2338: static_assert failed
```

**元に戻してください。**

### Step 2:実行時テストが通る

```
[Info ] MathTests.cpp(198): --- math tests ---
[Info ] MathTests.cpp(210): all math tests passed
```

### Step 3:わざと符号を間違える

`RotationY` の符号を入れ替えてみます。

```cpp
r.m[0][2] = s;    // ❌ 本来は -s
r.m[2][0] = -s;   // ❌ 本来は  s
```

```
[Error] MathTests.cpp(118): FAILED: NearEqual(v, Vector3{1, 0, 0})
[Fatal] MathTests.cpp(215): 1 math test(s) FAILED
```

**行番号と、失敗した式そのものが表示されます。** 第6章 6.4 節で `std::source_location` を使ったログを作った成果です。

**元に戻してください。**

### Step 4:行列をログに出してみる

`Matrix4x4` 用のフォーマッタも用意しておくと、デバッグが楽になります。

```cpp
LOG_INFO(L"view =\n{}", view);
```

```
[Info ] main.cpp(88): view =
  1.000   0.000   0.000   0.000
  0.000   1.000   0.000   0.000
  0.000   0.000   1.000   0.000
  0.000   0.000   5.000   1.000
```

**4 行目に平行移動成分が出ている**ことを確認してください。17.3.3 節で決めた規約の通りです。

**列に出ていたら、どこかで転置しています。**

---

### 本章の達成状態

- [ ] 行ベクトル・行優先の規約を採用した
- [ ] `Vector2/3/4` を `constexpr` で実装した
- [ ] `operator==` を `= default` にした
- [ ] `Matrix4x4` の要素アクセスに deducing this を使った
- [ ] 平行移動成分が `m[3][0..2]` にある
- [ ] `TransformPoint` と `TransformDirection` を分けた
- [ ] `LookAtLH` で 3 軸を直交化している
- [ ] 射影行列の深度範囲が [0, 1] になっている
- [ ] `std::formatter` でログに出せるようにした
- [ ] `static_assert` によるコンパイル時テストを書いた
- [ ] 三角関数を含むテストを実行時に書いた
- [ ] **すべてのテストが通った**
- [ ] Step 3 で、失敗が正しく検出されることを確認した

---

## トラブルシューティング

| 症状 | 原因 | 対処 |
|---|---|---|
| deducing this でコンパイルエラー | 環境が対応していない | const / 非 const の 2 つに分ける(17.4.1) |
| 平行移動が効かない | `w = 0` で変換している | `TransformPoint` を使う(17.4.2) |
| 法線が一緒に移動する | `w = 1` で変換している | `TransformDirection` を使う |
| 合成の順序が逆に見える | 列ベクトル流儀の資料を参照した | 17.3 節 |
| ログで平行移動が列に出る | どこかで転置している | 17.3.3 |
| 手前半分が映らない | OpenGL 用の射影行列 | 深度範囲を [0,1] に(17.6.1) |
| 真上を見るとカメラが壊れる | `up` と視線が平行 | 仰角に制限をかける(17.5.2) |
| 遠くの面がちらつく | `nearZ` が小さすぎる | 大きくする(17.6.4) |
| `Length` が `constexpr` にできない | `std::sqrt` は C++26 まで待ち | `LengthSquared` を使う |

---

## まとめ

**1. 数学ライブラリの選択は、レンダラ全体に波及する。**
行ベクトルか列ベクトルか、左手系か右手系か、深度範囲はどうか。**自作すると、これらを明示的に決めることになります。** それが本章の主な価値です。

**2. 「行優先 / 列優先」は 2 つの問題の混同。**
数学の記法(行ベクトル / 列ベクトル)とメモリ配置(行優先 / 列優先)は独立です。両者が入れ替わると同じバイト列になるため、混同されてきました。

**3. 本書は転置しない。**
C++ 側は行ベクトル・行優先、HLSL 側は `row_major` と `mul(v, M)`。**両者が同じ形になり、転置という操作が一切登場しません。**

**4. 深度範囲 [0, 1] は D3D の仕様。**
OpenGL 向けの射影行列を持ってくると、手前半分が消えます。

**5. `nearZ` を小さくしすぎない。**
深度の精度は `nearZ` に強く依存します。`farZ` を縮めるより効果があります。

**6. 行列は、絵を見る前に正しさを確定させる。**
符号を 1 つ間違えても絵は出ます。そして「レンダリングのバグか行列のバグか」が切り分けられなくなります。**`constexpr` にしておけば、大半のテストはコンパイル時に済みます。**

次章では、この行列を GPU へ送ります。定数バッファ、256 バイトアラインメントの罠、そしてルートパラメータの 3 形態。**第16章で「間違っている」と書いた暫定コードを、正しい形に置き換えます。立方体が回り出します。**

---

## 参考リンク

| 内容 | URL |
|---|---|
| DirectXMath の座標系と行列規約 | https://learn.microsoft.com/ja-jp/windows/win32/dxmath/pg-xnamath-migration-d3dx |
| 座標系とジオメトリ | https://learn.microsoft.com/ja-jp/windows/win32/direct3d12/coordinate-systems-and-geometry |
| HLSL の行列パッキング(`row_major`) | https://learn.microsoft.com/ja-jp/windows/win32/direct3dhlsl/dx-graphics-hlsl-per-component-math |
| `std::numbers`(cppreference) | https://ja.cppreference.com/w/cpp/numeric/constants |
| Explicit object parameter(deducing this) | https://ja.cppreference.com/w/cpp/language/member_functions |
