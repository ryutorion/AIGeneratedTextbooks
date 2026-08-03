# 第14章 クォータニオン

## この章について

`Quaternion` を追加します。回転を4つの数で表す仕組みで、行列より軽く、補間に向いています。

しかしこの章の主題は、クォータニオンの数学ではありません。**第10章 10.4.5 で挙げた「循環しそうになったときの3つの対処」を、実際に使うこと**です。

`Quaternion` と `Matrix4x4` は、相互に変換したくなります。

- クォータニオンから回転行列を作る(シェーダに送るため)
- 回転行列からクォータニオンを取り出す(補間するため)

素直に書くと、`:matrix` と `:quaternion` が互いを `import` して循環します。第10章 10.4.1 で見たとおり、モジュールでは循環できません。ヘッダ時代の前方宣言のような回避策もありません。

3つの案を並べて、選びます。

そして 14.3 では `Slerp`(球面線形補間)を実装します。第10章で用意した `SafeAcos` が、ようやく本来の出番を迎えます。そこで `constexpr` の限界に2つの角度からぶつかることになります。

最後の 14.4 は、少し性質の違う節です。数学ライブラリのテストをどう書くかという話で、**モジュールにすると「公開 API しかテストできなくなる」**という問題に触れます。

---

## 14.1 `gamemath.core:quaternion` を追加する

### 14.1.1 なぜクォータニオンを使うのか

第13章で回転行列を作りました。それで足りない理由を、簡単に押さえておきます。

**理由1: 補間が破綻する**

2つの回転行列の成分を単純に線形補間すると、回転行列でなくなります。長さが変わり、直交性が崩れ、物体が潰れます。

**理由2: サイズが大きい**

回転だけなら 4 個の数で足ります。行列は 16 個です。数千体のキャラクターのアニメーションデータでは、この差が効きます。

**理由3: 誤差が蓄積すると回転でなくなる**

行列を掛け続けると直交性が崩れていきます。クォータニオンなら、正規化するだけで回転に戻せます。

### 14.1.2 表現を決める

第13章 13.1.2 と同じく、実装より先に規約を決めます。

**成分の順序: `x, y, z, w`**

`x, y, z` がベクトル部、`w` がスカラー部です。メモリ上でこの順に並べます。

`w` を先に置く流儀(`w, x, y, z`)もあり、数学の教科書ではそちらが多いのですが、グラフィックス API との受け渡しでは `x, y, z, w` が一般的です。**ドキュメントに明記してください。**

**単位クォータニオンを前提とする**

回転を表すのは長さ 1 のクォータニオンだけです。ライブラリの関数は、引数が正規化済みであることを期待します。そうしないと、正規化のチェックがあらゆる関数に散らばって重くなります。

### 14.1.3 パーティションを作る

`Core/Quaternion.ixx` を作ってください。第13章 13.4.6 の方針にしたがい、演算子は**隠しフレンド**、名前付き関数は **`export` 付きの自由関数**にします。

```cpp
// Core/Quaternion.ixx
export module gamemath.core:quaternion;

import :vector;
import std;

namespace gamemath {

// x, y, z: ベクトル部 / w: スカラー部
// 回転を表すのは長さ 1 のものに限る
export struct Quaternion
{
    float x;
    float y;
    float z;
    float w;

    static constexpr Quaternion Identity()
    {
        return Quaternion{ 0.0f, 0.0f, 0.0f, 1.0f };
    }

    constexpr float LengthSquared() const
    {
        return x * x + y * y + z * z + w * w;
    }

    float Length() const
    {
        return std::sqrt(LengthSquared());
    }

    Quaternion Normalized() const
    {
        const float len = Length();
        if (len <= kEpsilon<float>) {
            return Identity();
        }
        const float inv = 1.0f / len;
        return Quaternion{ x * inv, y * inv, z * inv, w * inv };
    }

    // 共役。単位クォータニオンなら逆回転にあたる
    constexpr Quaternion Conjugate() const
    {
        return Quaternion{ -x, -y, -z, w };
    }

    // ───── 隠しフレンド ─────

    // ハミルトン積。回転の合成にあたる
    friend constexpr Quaternion operator*(const Quaternion& a, const Quaternion& b)
    {
        return Quaternion{
            a.w * b.x + a.x * b.w + a.y * b.z - a.z * b.y,
            a.w * b.y - a.x * b.z + a.y * b.w + a.z * b.x,
            a.w * b.z + a.x * b.y - a.y * b.x + a.z * b.w,
            a.w * b.w - a.x * b.x - a.y * b.y - a.z * b.z
        };
    }

    friend constexpr Quaternion operator*(const Quaternion& q, float s)
    {
        return Quaternion{ q.x * s, q.y * s, q.z * s, q.w * s };
    }

    friend constexpr Quaternion operator+(const Quaternion& a, const Quaternion& b)
    {
        return Quaternion{ a.x + b.x, a.y + b.y, a.z + b.z, a.w + b.w };
    }

    friend constexpr Quaternion operator-(const Quaternion& q)
    {
        return Quaternion{ -q.x, -q.y, -q.z, -q.w };
    }

    friend constexpr bool operator==(const Quaternion& a, const Quaternion& b)
    {
        return a.x == b.x && a.y == b.y && a.z == b.z && a.w == b.w;
    }
};

export constexpr float Dot(const Quaternion& a, const Quaternion& b)
{
    return a.x * b.x + a.y * b.y + a.z * b.z + a.w * b.w;
}

export Quaternion FromAxisAngle(const Vector3& axis, float radians)
{
    const Vector3 n = axis.Normalized();
    const float half = radians * 0.5f;
    const float s = std::sin(half);
    return Quaternion{ n.x() * s, n.y() * s, n.z() * s, std::cos(half) };
}

// v を q で回転させる
export constexpr Vector3 Rotate(const Quaternion& q, const Vector3& v)
{
    const Vector3 u{ q.x, q.y, q.z };
    const Vector3 t = Cross(u, v) * 2.0f;
    return v + t * q.w + Cross(u, t);
}

// 球面線形補間(定義は実装単位にある)
export Quaternion Slerp(const Quaternion& a, const Quaternion& b, float t);

} // namespace gamemath
```

`kEpsilon<float>` を使っていることに注目してください。`:vector` にある非公開の変数テンプレートです。第9章 9.4.2 で確認したとおり、**同じモジュールのパーティションどうしでは、`export` されていない宣言も見えます。**

そして第10章 10.3.3 で決めた原則も守れています。`Normalized()` はインターフェイスに書いたインライン関数ですが、使っている `kEpsilon` は**内部パーティションではなくインターフェイスパーティション**にあるので、到達可能性が保証されます。

### 14.1.4 `Dot` のオーバーロードがパーティションをまたぐ

`Dot` に注目してください。

```cpp
// :vector にあるもの
export template <FloatingPoint T, std::size_t N>
constexpr T Dot(const Vector<T, N>& a, const Vector<T, N>& b);

// :quaternion にあるもの
export constexpr float Dot(const Quaternion& a, const Quaternion& b);
```

**同じ名前の関数が、別々のパーティションにあります。**

これは問題になりません。どちらも `export` されていて、プライマリが両方を `export import` しているので、利用者からは両方が見えます。あとは通常のオーバーロード解決で、引数の型に応じて選ばれます。

```cpp
gamemath::Dot(v1, v2);   // :vector のテンプレート版
gamemath::Dot(q1, q2);   // :quaternion の Quaternion 版
```

ただし、**片方の `export` を忘れると、静かに困ったことになります。** `Quaternion` 版を `export` し忘れると、`Dot(q1, q2)` はテンプレート版に当たろうとして、意味不明なエラーを出します。第13章 13.3.2 で見たのと同じ問題です。

**オーバーロードの集合がパーティションをまたぐときは、全部が `export` されていることを確認してください。** これは隠しフレンドでは防げないケースです(`Dot` は名前で呼ぶので自由関数にしなければならない)。

### 14.1.5 プライマリにつなぐ

```cpp
// Core/Core.ixx
export module gamemath.core;

export import :vector;
export import :basis;
export import :matrix;
export import :quaternion;      // ← 追加
```

### 14.1.6 実装単位を作る

`Slerp` の定義は 14.3 で書きます。いまは空の実装単位を用意しておきましょう。`Core/Quaternion.cpp` です。

```cpp
// Core/Quaternion.cpp
module gamemath.core;

import :detail;
import std;

namespace gamemath {

// Slerp はここに書く(14.3)

} // namespace gamemath
```

**この状態ではビルドできません。** `Slerp` を宣言したのに定義していないので、`main.cpp` から呼べばリンクエラーになります。呼ばなければ通ります。14.3 まで、`Slerp` は使わないでください。

### 14.1.7 動作確認

`main.cpp` に追加します。

```cpp
// main.cpp

// --- クォータニオンの確認 ---
const float half = std::numbers::pi_v<float> * 0.5f;
const gamemath::Vector3 zAxis{ 0.0f, 0.0f, 1.0f };
const gamemath::Vector3 xAxis{ 1.0f, 0.0f, 0.0f };

const gamemath::Quaternion qz = gamemath::FromAxisAngle(zAxis, half);

std::println("q(Z,90)         = ({:.4f}, {:.4f}, {:.4f}, {:.4f})",
             qz.x, qz.y, qz.z, qz.w);

const gamemath::Vector3 rotatedByQ = gamemath::Rotate(qz, xAxis);
std::println("Rotate(q, X)    = ({:.4f}, {:.4f}, {:.4f})",
             rotatedByQ.x(), rotatedByQ.y(), rotatedByQ.z());

// 共役を掛けると単位クォータニオンに戻る
const gamemath::Quaternion roundTrip = qz * qz.Conjugate();
std::println("q * conj(q)     = ({:.4f}, {:.4f}, {:.4f}, {:.4f})",
             roundTrip.x, roundTrip.y, roundTrip.z, roundTrip.w);

static_assert(gamemath::Quaternion::Identity().w == 1.0f);
static_assert(gamemath::Dot(gamemath::Quaternion::Identity(),
                            gamemath::Quaternion::Identity()) == 1.0f);
```

ビルドして実行してください。

```
q(Z,90)         = (0.0000, 0.0000, 0.7071, 0.7071)
Rotate(q, X)    = (0.0000, 1.0000, 0.0000)
q * conj(q)     = (0.0000, 0.0000, 0.0000, 1.0000)
```

X 軸が Y 軸に移り、共役を掛けると単位クォータニオンに戻りました。第13章 13.5.5 の `RotationZ(90)` と同じ結果です。

---

## 14.2 Matrix との相互変換で生じる相互依存を解く

### 14.2.1 やりたいこと

必要なのは2つの関数です。

```cpp
Matrix4x4  ToMatrix(const Quaternion& q);      // クォータニオン → 行列
Quaternion ToQuaternion(const Matrix4x4& m);   // 行列 → クォータニオン
```

前者はシェーダに送る行列を作るため、後者はアニメーションデータを補間可能な形に変えるために使います。どちらも実務で必要です。

### 14.2.2 【実験】素直に書くと循環する

素直に考えると、こうなります。

- `ToMatrix` は `Quaternion` を受け取るので、`:quaternion` に置く
- `ToQuaternion` は `Matrix4x4` を受け取るので、`:matrix` に置く

やってみましょう。

```cpp
// Core/Quaternion.ixx
export module gamemath.core:quaternion;

import :vector;
import :matrix;                 // ← 追加
import std;

namespace gamemath {
// ...
export Matrix4x4 ToMatrix(const Quaternion& q);
}
```

```cpp
// Core/Matrix.ixx
export module gamemath.core:matrix;

import :vector;
import :quaternion;             // ← 追加
import std;

namespace gamemath {
// ...
export Quaternion ToQuaternion(const Matrix4x4& m);
}
```

ビルドしてください。**循環依存のエラーになります。**

```
:matrix  →  :quaternion  →  :matrix  →  ...
```

第10章 10.4.1 で作ったのと同じ状況です。ヘッダなら前方宣言で切り抜けられましたが、モジュールでは切り抜けられません。`.ifc` の生成順序が決まらないからです。

**追加した2行を削除してください。**

### 14.2.3 3つの案を検討する

第10章 10.4.5 で挙げた3つの対処を、この場合に当てはめて考えます。

**案A ── 依存の向きを1つに決める(対処2)**

両方の関数を片方のパーティションに置きます。たとえば `:quaternion` に置くと、`:quaternion` が `:matrix` を `import` する一方通行になります。

```
:quaternion  →  :matrix  →  :vector
```

**長所** ── いちばん簡単です。ファイルが増えません。

**短所** ── `Quaternion` を使いたいだけの人が、`Matrix4x4` も引きずり込むことになります。パーティションなので利用者から見た `import` は変わりませんが、`:quaternion` の `.ifc` が `:matrix` に依存するので、行列を変更するとクォータニオンも再コンパイルされます。

そして概念的に妙です。**クォータニオンは行列を知らなくても成立する**のに、依存させてしまいます。第13章 13.2.3 で「ベクトルは行列を知らない」と決めたのと同じ判断が、ここでも問われます。

**案B ── ブリッジとなるパーティションを作る(対処1の変形)**

変換関数だけを、新しいパーティションに切り出します。

```
        :transform
        ↓        ↓
   :matrix   :quaternion
        ↓        ↓
         :vector
```

`:transform` が両方を `import` します。`:matrix` と `:quaternion` は互いを知りません。

**長所** ── 2つの型が独立を保てます。そして変換関数は今後増えます(オイラー角との変換、TRS の分解、`LookAt` など)。置き場所が最初からあるほうが健全です。

**短所** ── ファイルとパーティションが増えます。

**案C ── 宣言と定義を分ける(対処3)**

インターフェイスには宣言だけを置き、定義を実装単位に持っていく方法です。実装単位は依存の循環に関与しないので(第10章 10.4.4)、そこで両方を `import` できます。

**この場合、使えません。**

理由は、**宣言そのものが両方の型を必要とするから**です。

```cpp
export Matrix4x4 ToMatrix(const Quaternion& q);   // 両方の型名が必要
```

この行を `:quaternion` に書くには `Matrix4x4` が見えていなければならず、そのためには `import :matrix;` が必要です。定義を実装単位に移しても、宣言の問題は残ります。

対処3が有効なのは、**宣言に必要な型がすでに見えていて、定義の中身だけが別のモジュールを必要とする場合**です。第10章 10.4.5 で挙げた例は、実はその条件を満たしていませんでした。ここで訂正しておきます。

### 14.2.4 案B を選ぶ

**`GameMath` では案B(ブリッジパーティション)を採ります。**

決め手は、第12章 12.1.5 の指針1です。

> 一緒に変更されるものは、同じモジュールに置く

`Matrix4x4` と `Quaternion` は、一緒に**使われます**が、一緒に**変更されるとは限りません**。行列の実装を SIMD 化しても、クォータニオンの表現は変わりません。

そして「相互変換」という関心事は、どちらの型にも属しません。**両方を知っている第三者の仕事**です。それを表す場所を作るのが素直です。

案A のほうが手軽なのは事実です。パーティションが2つ3つのライブラリなら、案A で十分でしょう。しかし `GameMath` はこれから幾何プリミティブも増えます。「型どうしをつなぐもの」の置き場所を決めておく価値があります。

### 14.2.5 `:transform` パーティションを作る

`Core/Transform.ixx` を作ってください。

```cpp
// Core/Transform.ixx
export module gamemath.core:transform;

import :vector;
import :matrix;
import :quaternion;

namespace gamemath {

// クォータニオン(正規化済みであること)から回転行列を作る
export constexpr Matrix4x4 ToMatrix(const Quaternion& q)
{
    const float xx = q.x * q.x;
    const float yy = q.y * q.y;
    const float zz = q.z * q.z;
    const float xy = q.x * q.y;
    const float xz = q.x * q.z;
    const float yz = q.y * q.z;
    const float wx = q.w * q.x;
    const float wy = q.w * q.y;
    const float wz = q.w * q.z;

    Matrix4x4 r{};

    r.m[0][0] = 1.0f - 2.0f * (yy + zz);
    r.m[0][1] =        2.0f * (xy - wz);
    r.m[0][2] =        2.0f * (xz + wy);

    r.m[1][0] =        2.0f * (xy + wz);
    r.m[1][1] = 1.0f - 2.0f * (xx + zz);
    r.m[1][2] =        2.0f * (yz - wx);

    r.m[2][0] =        2.0f * (xz - wy);
    r.m[2][1] =        2.0f * (yz + wx);
    r.m[2][2] = 1.0f - 2.0f * (xx + yy);

    r.m[3][3] = 1.0f;
    return r;
}

// 回転行列からクォータニオンを取り出す(定義は実装単位)
export Quaternion ToQuaternion(const Matrix4x4& m);

} // namespace gamemath
```

`ToMatrix` は `constexpr` にできます。四則演算しか使っていないからです。定義もインターフェイスに置きます。行列を毎フレーム作る用途があるので、インライン展開してほしいところです(第5章 5.6.5)。

### 14.2.6 `ToQuaternion` を実装単位に置く

`ToQuaternion` は分岐が多く、行数もあります。呼び出し頻度は `ToMatrix` より低いので、**実装単位**に置きます。

`Core/Transform.cpp` を作ってください。

```cpp
// Core/Transform.cpp
module gamemath.core;

import std;

namespace gamemath {

Quaternion ToQuaternion(const Matrix4x4& m)
{
    const float trace = m.m[0][0] + m.m[1][1] + m.m[2][2];

    if (trace > 0.0f) {
        const float s = std::sqrt(trace + 1.0f) * 2.0f;
        const float inv = 1.0f / s;
        return Quaternion{
            (m.m[2][1] - m.m[1][2]) * inv,
            (m.m[0][2] - m.m[2][0]) * inv,
            (m.m[1][0] - m.m[0][1]) * inv,
            0.25f * s
        };
    }

    if (m.m[0][0] > m.m[1][1] && m.m[0][0] > m.m[2][2]) {
        const float s = std::sqrt(1.0f + m.m[0][0] - m.m[1][1] - m.m[2][2]) * 2.0f;
        const float inv = 1.0f / s;
        return Quaternion{
            0.25f * s,
            (m.m[0][1] + m.m[1][0]) * inv,
            (m.m[0][2] + m.m[2][0]) * inv,
            (m.m[2][1] - m.m[1][2]) * inv
        };
    }

    if (m.m[1][1] > m.m[2][2]) {
        const float s = std::sqrt(1.0f + m.m[1][1] - m.m[0][0] - m.m[2][2]) * 2.0f;
        const float inv = 1.0f / s;
        return Quaternion{
            (m.m[0][1] + m.m[1][0]) * inv,
            0.25f * s,
            (m.m[1][2] + m.m[2][1]) * inv,
            (m.m[0][2] - m.m[2][0]) * inv
        };
    }

    const float s = std::sqrt(1.0f + m.m[2][2] - m.m[0][0] - m.m[1][1]) * 2.0f;
    const float inv = 1.0f / s;
    return Quaternion{
        (m.m[0][2] + m.m[2][0]) * inv,
        (m.m[1][2] + m.m[2][1]) * inv,
        0.25f * s,
        (m.m[1][0] - m.m[0][1]) * inv
    };
}

} // namespace gamemath
```

`import :transform;` を書いていないことに注意してください。実装単位はプライマリを暗黙に `import` し、次の手順でプライマリが `:transform` を `export import` するので、それで届きます。

### 14.2.7 プライマリにつなぐ

```cpp
// Core/Core.ixx
export module gamemath.core;

export import :vector;
export import :basis;
export import :matrix;
export import :quaternion;
export import :transform;       // ← 追加
```

**`:transform` は最後に書きます。** 順序に意味はありませんが(第3章 3.4.4)、依存の層と同じ順に並べておくと、あとで読むときに構造が伝わります。

### 14.2.8 依存グラフを更新する

```
              gamemath.core (プライマリ)
              export import :vector;
              export import :basis;
              export import :matrix;
              export import :quaternion;
              export import :transform;
                       │
        ┌──────────────┴────────────┐
        │       :transform          │  ToMatrix / ToQuaternion
        └──────┬─────────────┬──────┘
               ↓             ↓
        ┌──────────┐  ┌──────────────┐
        │ :matrix  │  │ :quaternion  │   ← 互いを知らない
        └──────────┘  └──────────────┘
               ↓             ↓
        ┌──────────┐         │
        │ :basis   │         │
        └──────────┘         │
               ↓             │
        ┌──────────┐         │
        │ :detail  │         │
        └──────────┘         │
               ↓             │
        ┌────────────────────┴──┐
        │       :vector         │   ← 土台
        └───────────────────────┘
```

`:matrix` と `:quaternion` が同じ層に並び、互いを知らないままになっています。狙いどおりです。

### 14.2.9 動作確認

`main.cpp` に追加します。

```cpp
// main.cpp

// --- 相互変換の確認 ---
const gamemath::Matrix4x4 fromQ = gamemath::ToMatrix(qz);
const gamemath::Matrix4x4 fromR = gamemath::RotationZ(half);

std::println("ToMatrix(q) row0 = ({:.4f}, {:.4f}, {:.4f})",
             fromQ(0, 0), fromQ(0, 1), fromQ(0, 2));
std::println("RotationZ   row0 = ({:.4f}, {:.4f}, {:.4f})",
             fromR(0, 0), fromR(0, 1), fromR(0, 2));

const gamemath::Quaternion backQ = gamemath::ToQuaternion(fromQ);
std::println("ToQuaternion     = ({:.4f}, {:.4f}, {:.4f}, {:.4f})",
             backQ.x, backQ.y, backQ.z, backQ.w);

// ToMatrix は constexpr
static_assert(gamemath::ToMatrix(gamemath::Quaternion::Identity())
              == gamemath::Matrix4x4::Identity());
```

ビルドして実行してください。

```
ToMatrix(q) row0 = (0.0000, -1.0000, 0.0000)
RotationZ   row0 = (-0.0000, -1.0000, 0.0000)
ToQuaternion     = (0.0000, 0.0000, 0.7071, 0.7071)
```

`ToMatrix(qz)` が `RotationZ(90°)` と一致し、そこから戻したクォータニオンが元の `qz` と一致しました。往復できています。

`static_assert` も通っています。単位クォータニオンから単位行列が、コンパイル時に作られています。

---

## 14.3 `Slerp` を実装しながら `constexpr` の限界を知る

### 14.3.1 なぜ `Slerp` が必要か

2つの回転の間を補間する方法を、3つ並べます。

**成分の線形補間(Lerp)** ── 長さが 1 でなくなり、回転でなくなります。

**線形補間して正規化(Nlerp)** ── 回転にはなりますが、角速度が一定になりません。等間隔の `t` に対して、回転の進み方が中央で速くなります。

**球面線形補間(Slerp)** ── 4次元球面上を最短の弧に沿って進みます。角速度が一定になります。

カメラの向きの補間や、アニメーションのブレンドでは Slerp が求められます。

### 14.3.2 `SafeAcos` の出番

Slerp の計算には、2つのクォータニオンのなす角が必要です。

```cpp
const float theta = std::acos(Dot(a, b));
```

ここが第10章 10.1.2 で説明した問題そのものです。`a` と `b` が正規化されていても、浮動小数点の誤差で内積が `1.0000001` になることがあります。すると `std::acos` は **NaN** を返し、キャラクターの姿勢が吹き飛びます。

第10章で `:detail` に用意した `SafeAcos` を使います。

```cpp
const float theta = SafeAcos(Dot(a, b));
```

そして、ここに制約が1つ付いてきます。

**`SafeAcos` は内部パーティションにあるので、実装単位からしか使えません。**

第10章 10.3.3 で決めた原則です。

> 内部パーティションに置いてよいのは、実装単位から使うものだけ。
> インターフェイスに書いたインライン関数・テンプレート・`constexpr` 関数から呼んではいけない。

だから `Slerp` の定義は、必然的に実装単位に置くことになります。ちょうど 14.1.6 で `Core/Quaternion.cpp` を用意しておいたのは、このためでした。

### 14.3.3 実装する

`Core/Quaternion.cpp` に書きます。

```cpp
// Core/Quaternion.cpp
module gamemath.core;

import :detail;
import std;

namespace gamemath {

Quaternion Slerp(const Quaternion& a, const Quaternion& b, float t)
{
    float d = Dot(a, b);

    // q と -q は同じ回転を表す。内積が負なら、遠回りしている
    Quaternion end = b;
    if (d < 0.0f) {
        end = -b;
        d = -d;
    }

    // ほとんど同じ向きなら、線形補間で十分(sin による除算を避ける)
    if (d > 0.9995f) {
        const Quaternion mixed = a * (1.0f - t) + end * t;
        return mixed.Normalized();
    }

    const float theta = SafeAcos(d);          // ← 内部パーティションの関数
    const float sinTheta = std::sin(theta);
    const float invSin = 1.0f / sinTheta;

    const float wa = std::sin((1.0f - t) * theta) * invSin;
    const float wb = std::sin(t * theta) * invSin;

    return a * wa + end * wb;
}

} // namespace gamemath
```

3つの工夫が入っています。

**工夫1: 符号の反転。** `q` と `-q` は同じ回転を表します。内積が負なら、球面上で遠回りする側を選んでしまっているので、片方の符号を反転させて最短経路にします。この性質は 14.4.4 でもう一度出てきます。

**工夫2: ほぼ同一の場合の分岐。** `theta` が 0 に近いと `sin(theta)` も 0 に近づき、除算が破綻します。`d > 0.9995` のときは線形補間して正規化します(Nlerp)。この範囲では両者の差は目に見えません。

**工夫3: `SafeAcos`。** 定義域外の入力で NaN にならないようにします。

ビルドしてください。`Slerp` の定義ができたので、通るようになりました。

### 14.3.4 `constexpr` にできない2つの理由

`Slerp` に `constexpr` は付いていません。理由が**2つ独立に**あります。区別して理解しておく価値があります。

**理由1: 標準ライブラリの関数が `constexpr` でない**

`std::sin` と `std::acos` は、C++23 の時点で `constexpr` ではありません。第8章 8.5.3 で `Length()` について説明したのと同じ事情です。

これは**関数が `constexpr` として書けるかどうか**の問題です。

**理由2: 定義が実装単位にある**

仮に `std::sin` が `constexpr` になったとしても、`Slerp` の定義は `Quaternion.cpp` にあります。第8章 8.5.4 で確認したとおり、`constexpr` 関数をコンパイル時に評価するには、**定義が到達可能でなければなりません**。実装単位の中身は到達可能ではありません(第7章 7.4.3)。

これは**書いた `constexpr` が使えるかどうか**の問題です。

つまり `Slerp` は、2つの門のどちらも通れません。

```
[門1] constexpr として書けるか?     → std::sin が constexpr でない → ✕
[門2] 定義が到達可能か?             → 実装単位にある              → ✕
```

そして理由2は、`SafeAcos` が内部パーティションにあることの帰結でもあります。**設計上の判断が、`constexpr` の可否に連鎖している**わけです。

### 14.3.5 `constexpr` の境界線はどこにあるか

`GameMath` 全体を見渡すと、`constexpr` の境界がはっきり見えてきます。

| `constexpr` にできる | `constexpr` にできない |
|---|---|
| `Dot` / `Cross` | `Length` / `Normalized`(`sqrt`) |
| `LengthSquared` | `RotationX/Y/Z`(`sin` / `cos`) |
| `operator+` / `-` / `*` | `FromAxisAngle`(`sin` / `cos`) |
| `Matrix4x4::Identity` | `Slerp`(`sin` / `acos`) |
| `Transpose` | `AngleBetween`(`acos`) |
| `Translation` / `Scale` | `Orthogonal`(実装単位にある) |
| `ToMatrix` | `ToQuaternion`(`sqrt` + 実装単位) |

境界線は明確です。

> **四則演算だけで書けるものは `constexpr` にできる。平方根と三角関数を使うものはできない。**

そして、この線引きは実用的にも意味を持っています。

`constexpr` にできる側は、コンパイル時に確定できる ── つまり**定数として書ける値**です。単位行列、固定のスケール、既知の平行移動。これらは実行時に計算する必要がありません。

`constexpr` にできない側は、実行時のパラメータに依存する ── つまり**毎フレーム変わる値**です。

ライブラリの利用者に伝えるべきことも、この線に沿います。「回転行列はコンパイル時に作れません」と書いておけば、利用者は無駄に `constexpr` を試さずに済みます。

> **C++26 で変わる見込みです**
>
> `<cmath>` の数学関数を `constexpr` にする提案が標準化の議論を通っています。実装されれば、上の表の右側の多くが左側に移ります。
>
> そのときライブラリ側でやることは、`constexpr` を付け足すことだけです。**ただし、実装単位に置いた関数は移れません。** 到達可能性の問題は規格の変更では解決しないからです。
>
> つまり「いま実装単位に置くかどうか」は、将来 `constexpr` にできるかどうかの判断でもあります。第5章 5.6.5 の基準に、もう1つ観点が加わったことになります。

### 14.3.6 動作確認

`main.cpp` に追加します。

```cpp
// main.cpp

// --- Slerp の確認 ---
const gamemath::Quaternion qStart = gamemath::Quaternion::Identity();
const gamemath::Quaternion qEnd   = gamemath::FromAxisAngle(zAxis, half);

for (int i = 0; i <= 4; ++i) {
    const float t = static_cast<float>(i) * 0.25f;
    const gamemath::Quaternion s = gamemath::Slerp(qStart, qEnd, t);
    const gamemath::Vector3 dir = gamemath::Rotate(s, xAxis);
    const float angle = gamemath::AngleBetween(xAxis, dir);

    std::println("Slerp t={:.2f} -> angle = {:.2f} deg  |q| = {:.4f}",
                 t, angle * 180.0f / std::numbers::pi_v<float>, s.Length());
}
```

ビルドして実行してください。

```
Slerp t=0.00 -> angle = 0.00 deg  |q| = 1.0000
Slerp t=0.25 -> angle = 22.50 deg  |q| = 1.0000
Slerp t=0.50 -> angle = 45.00 deg  |q| = 1.0000
Slerp t=0.75 -> angle = 67.50 deg  |q| = 1.0000
Slerp t=1.00 -> angle = 90.00 deg  |q| = 1.0000
```

角度が **22.5 度ずつ等間隔**に進んでいます。これが Slerp の性質です。Nlerp なら、中央付近が速くなって等間隔にはなりません。

そして長さが常に 1.0000 を保っています。回転として正しい状態が維持されています。

第10章で用意した `SafeAcos`(`Slerp` の内部)と `AngleBetween`(検証に使用)が、どちらも役に立ちました。

---

## 14.4 数学的な単体テストを書く準備

`Slerp` が正しく動いているかどうか、上の出力を目で見て確認しました。しかし、これは持続可能な方法ではありません。

第20章でテストプロジェクトを作りますが、その前に**数学ライブラリのテストに特有の問題**を整理しておきます。

### 14.4.1 浮動小数点は `==` で比較できない

先ほどの出力を見返してください。

```
ToMatrix(q) row0 = (0.0000, -1.0000, 0.0000)
RotationZ   row0 = (-0.0000, -1.0000, 0.0000)
```

`0.0000` と `-0.0000` があります。同じ値のはずですが、計算経路が違うので厳密には一致しません。

だから、こういうテストは書けません。

```cpp
assert(ToMatrix(qz) == RotationZ(half));   // 通らない
```

`Matrix4x4` の `operator==` は成分を `==` で比較しています。1ビットでも違えば `false` です。

**許容誤差つきの比較が必要です。**

### 14.4.2 `NearlyEqual` を用意する

ここで、設計上の気づきがあります。

`:detail` に `IsNearlyZero` のような関数を置きたくなりますが、**それでは足りません**。テストコードはモジュールの外にあるので、内部パーティションの関数は使えません(第10章 10.3.3)。

**テストのために、許容誤差つきの比較関数を公開する必要があります。**

これは妥協ではありません。ライブラリの利用者も同じものを必要とします。「2つの位置がほぼ同じか」を判定したい場面は、ゲームコードのあちこちにあります。

`Core/Vector.ixx` に追加してください。

```cpp
// Core/Vector.ixx(namespace gamemath の中)

export template <FloatingPoint T>
constexpr bool NearlyEqual(T a, T b, T tolerance = kEpsilon<T>)
{
    const T d = a - b;
    return (d < T(0) ? -d : d) <= tolerance;
}

export template <FloatingPoint T, std::size_t N>
constexpr bool NearlyEqual(const Vector<T, N>& a, const Vector<T, N>& b,
                           T tolerance = kEpsilon<T>)
{
    for (std::size_t i = 0; i < N; ++i) {
        if (!NearlyEqual(a.e[i], b.e[i], tolerance)) {
            return false;
        }
    }
    return true;
}
```

`Core/Matrix.ixx` に追加します。

```cpp
export constexpr bool NearlyEqual(const Matrix4x4& a, const Matrix4x4& b,
                                  float tolerance = kEpsilon<float>)
{
    for (std::size_t i = 0; i < 4; ++i) {
        for (std::size_t j = 0; j < 4; ++j) {
            if (!NearlyEqual(a.m[i][j], b.m[i][j], tolerance)) {
                return false;
            }
        }
    }
    return true;
}
```

`Core/Quaternion.ixx` に追加します。

```cpp
export constexpr bool NearlyEqual(const Quaternion& a, const Quaternion& b,
                                  float tolerance = kEpsilon<float>)
{
    return NearlyEqual(a.x, b.x, tolerance)
        && NearlyEqual(a.y, b.y, tolerance)
        && NearlyEqual(a.z, b.z, tolerance)
        && NearlyEqual(a.w, b.w, tolerance);
}
```

`NearlyEqual` のオーバーロード集合が、**3つのパーティションにまたがりました。** 14.1.4 で `Dot` について述べたとおり、これは問題ありません。ただし**全部に `export` を付けること**を忘れないでください。

なお、既定引数に `kEpsilon<T>` を使っています。`kEpsilon` は `export` していない変数テンプレートですが、`NearlyEqual` は `constexpr` テンプレートで到達可能な領域(インターフェイスパーティション)にあるので問題ありません。第10章 10.3.3 の原則を守っています。

### 14.4.3 検証すべき不変条件

数学ライブラリのテストは、「入力 X に対して出力 Y」という形よりも、**成り立つべき性質(不変条件)**を確認する形が有効です。期待値をいちいち手計算する必要がなく、広い入力範囲を検査できます。

`GameMath` について確認すべき性質を挙げます。

**ベクトル**

- `v.Normalized().Length()` が 1 になる(ゼロベクトルを除く)
- `Dot(a, b) == Dot(b, a)`
- `Dot(Cross(a, b), a) == 0`(外積は両方に垂直)
- `Cross(a, b) == -Cross(b, a)`

**行列**

- `M * Identity == M`
- `Transpose(Transpose(M)) == M`
- `TransformDirection(R, v).Length() == v.Length()`(回転は長さを保つ)
- `Transpose(R) * R == Identity`(回転行列は直交)

**クォータニオン**

- `q * q.Conjugate() == Identity`(正規化済みなら)
- `Rotate(q, v).Length() == v.Length()`
- `Rotate(a * b, v) == Rotate(a, Rotate(b, v))`(合成の順序)
- `Slerp(a, b, 0) == a` と `Slerp(a, b, 1) == b`
- `Slerp(a, b, t).Length() == 1`

**相互変換**

- `ToMatrix(FromAxisAngle(axis, θ)) == RotationAxis(axis, θ)`
- `ToQuaternion(ToMatrix(q)) == q`(往復)

最後の1つには、注意が必要です。

### 14.4.4 クォータニオンの符号の曖昧さ

`q` と `-q` は、**同じ回転を表します。**

14.3.3 の工夫1 で使った性質です。回転角 θ のクォータニオンは `(axis * sin(θ/2), cos(θ/2))` ですが、`θ + 360°` の回転は同じ姿勢を表し、そのクォータニオンは符号が反転します。

つまり、往復のテストはこう書かなければなりません。

```cpp
const Quaternion back = ToQuaternion(ToMatrix(q));
const bool ok = NearlyEqual(back, q) || NearlyEqual(back, -q);
```

これを知らずにテストを書くと、「たまに失敗するテスト」ができあがります。入力によって `ToQuaternion` がどちらの符号を返すかが変わるからです。

**数学的に等価だが表現が一意でない** ── この種の性質は、数学ライブラリのテストで頻繁に問題になります。他の例も挙げておきます。

- ゼロベクトルの正規化 ── 何を返すのが正しいのか、仕様で決めておく必要があります(`GameMath` はゼロベクトルを返します)
- 角度 ── `0` と `2π` は同じ向き
- 平行なベクトルの外積 ── ゼロベクトルになり、垂直方向が定まらない

**テストを書く前に、仕様を決めてください。** そして決めたことをドキュメントに書いてください。第12章 12.4.4 で「規約は API の一部」と書いたのと同じことです。

### 14.4.5 モジュールでは公開 API しかテストできない

最後に、モジュール特有の問題です。

ヘッダの時代、内部関数をテストしたければ、テストコードから内部ヘッダを `#include` すればよかったはずです。

```cpp
// テストコード(ヘッダ時代)
#include "GameMath/detail/SafeAcos.h"

TEST(SafeAcos, ClampsDomain)
{
    assert(SafeAcos(1.5f) == 0.0f);
}
```

モジュールではできません。`SafeAcos` は内部パーティションにあり、外からは見えません。

選択肢は3つあります。

**選択肢1: 公開 API 経由でテストする**

`SafeAcos` を直接呼ぶのではなく、それを使っている `Slerp` や `AngleBetween` を、定義域の境界付近でテストします。

```cpp
// 内積が 1 を超えかねない状況を作る
const Quaternion q = /* ほぼ同一の2つ */;
assert(!std::isnan(Slerp(q, q, 0.5f).w));
```

**長所** ── テストが実装に依存しません。内部構造を変えてもテストは無傷です。

**短所** ── 内部関数の境界条件を直接突けません。カバレッジが下がります。

**選択肢2: テストコードをモジュールの一部にする**

テストファイルの先頭に `module gamemath.core;` と書けば、それは `gamemath.core` の実装単位になります。実装単位はモジュールの内部を全部見られるので、`import :detail;` して `SafeAcos` を直接テストできます。

**長所** ── 何でもテストできます。

**短所** ── テストコードがライブラリの一部になります。ビルド構成で切り分ける必要があり、うっかりリリースビルドに混ぜる危険があります。

**選択肢3: 公開してしまう**

`SafeAcos` を `export` します。「テストしたいものは公開 API であるべきだ」という立場です。

**長所** ── 単純です。

**短所** ── 変更の自由を失います。第7章 7.2 で得たカプセル化を、テストのために手放すことになります。

**`GameMath` では選択肢1を採ります。** 内部関数は、それを使う公開関数を通してテストします。第20章でテストプロジェクトを作るときに、この方針で書きます。

そのうえで、1つ設計上の助言があります。

**「テストしにくい」は、「公開すべきだ」のサインかもしれません。**

`SafeAcos` や `NearlyEqual` のように、単体でテストしたくなるほど独立した機能は、利用者にとっても有用なことが多いのです。実際 `NearlyEqual` は、14.4.2 でテストのために公開しましたが、それは利用者にとっても必要なものでした。

内部に隠すか公開するかの判断に、**「テストしたいか」という観点を加えてください。**

### 14.4.6 実際に確かめる

不変条件のうちいくつかを、いま確認しておきましょう。

```cpp
// main.cpp

// --- 不変条件の確認 ---
const gamemath::Quaternion qa = gamemath::FromAxisAngle(zAxis, half);
const gamemath::Quaternion qb = gamemath::FromAxisAngle(xAxis, half * 0.5f);

const gamemath::Quaternion composed = qa * qb;
const gamemath::Vector3 v{ 1.0f, 2.0f, 3.0f };

const bool rotationOrder = gamemath::NearlyEqual(
    gamemath::Rotate(composed, v),
    gamemath::Rotate(qa, gamemath::Rotate(qb, v)),
    1.0e-5f);

const gamemath::Quaternion back = gamemath::ToQuaternion(gamemath::ToMatrix(qa));
const bool roundTrip = gamemath::NearlyEqual(back, qa, 1.0e-5f)
                    || gamemath::NearlyEqual(back, -qa, 1.0e-5f);

const bool matchesMatrix = gamemath::NearlyEqual(
    gamemath::ToMatrix(qa),
    gamemath::RotationZ(half),
    1.0e-5f);

const bool slerpEnds = gamemath::NearlyEqual(gamemath::Slerp(qa, qb, 0.0f), qa)
                    && gamemath::NearlyEqual(gamemath::Slerp(qa, qb, 1.0f), qb);

std::println("Rotate(a*b, v) == Rotate(a, Rotate(b, v)) : {}", rotationOrder);
std::println("ToQuaternion(ToMatrix(q)) == +/-q         : {}", roundTrip);
std::println("ToMatrix(q) == RotationZ                  : {}", matchesMatrix);
std::println("Slerp endpoints                           : {}", slerpEnds);
```

ビルドして実行してください。

```
Rotate(a*b, v) == Rotate(a, Rotate(b, v)) : true
ToQuaternion(ToMatrix(q)) == +/-q         : true
ToMatrix(q) == RotationZ                  : true
Slerp endpoints                           : true
```

許容誤差を `1.0e-5f` に緩めていることに注意してください。既定の `kEpsilon`(`1e-6`)では、計算の段数が多い箇所で失敗します。**許容誤差は、計算経路の長さに応じて決めるものです。** 一律の値を使い回すと、厳しすぎて誤検出するか、緩すぎて見逃すかのどちらかになります。

---

## 14.5 この章のまとめ

- クォータニオンは成分の順序(`x, y, z, w`)と単位クォータニオン前提を決めて文書化する
- 演算子は隠しフレンド、名前付き関数は `export` 付きの自由関数(第13章 13.4.6 の方針)
- **オーバーロード集合がパーティションをまたいでよい。** ただし全部に `export` が必要
- `Matrix4x4` と `Quaternion` の相互変換は循環依存を生む。ヘッダ時代の前方宣言のような回避策はない
- 3つの対処を検討した結果、**ブリッジパーティション `:transform`** を採った。2つの型が独立を保てる
- **第10章の対処3(宣言と定義を分ける)は、この場合使えない。** 宣言そのものが両方の型を必要とするため
- `SafeAcos` は内部パーティションにあるので、それを使う `Slerp` は必然的に実装単位に置くことになる
- **`constexpr` にできない理由は2つある** ── 標準ライブラリの関数が `constexpr` でない(書けない)/ 定義が実装単位にある(到達可能でない)
- `GameMath` における `constexpr` の境界は「四則演算だけで書けるか」。平方根と三角関数を使うものは不可
- C++26 で数学関数が `constexpr` になっても、**実装単位に置いた関数は移れない**
- 浮動小数点は `==` で比較できない。**許容誤差つきの `NearlyEqual` を公開する必要がある**
- 許容誤差は、計算経路の長さに応じて決める
- テストは期待値よりも**不変条件**で書く
- **`q` と `-q` は同じ回転**。往復テストは符号の両方を許す必要がある
- **モジュールでは公開 API しかテストできない。** `GameMath` は公開関数を通して内部をテストする方針
- **「テストしにくい」は「公開すべきだ」のサインかもしれない**

## 次章に向けて

第15章では `gamemath.geometry` を充実させます。第12章で `AABB` を置いた場所です。

- `Sphere` / `Ray` / `Plane` を追加する
- 交差判定を `:intersect` パーティションに集める

そして、ここで第6章 6.5.4 と第12章 12.3.3 の設計方針が試されます。

**交差判定の結果を、どう返すか。**

「レイと球が交差したかどうか」と「交差した場合の距離」を、1つの関数で返したい。素直に考えると `std::optional<float>` です。しかしそれをやると、`GameMath` の公開インターフェイスに標準ライブラリの型が現れます。

すると `export import std;` が必要になり、第6章で決めた「利用者の環境を選ばないライブラリ」という方針が崩れます。

どうするか ── 実際に手を動かして決めます。
