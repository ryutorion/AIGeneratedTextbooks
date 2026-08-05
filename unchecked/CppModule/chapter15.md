# 第15章 幾何プリミティブと衝突判定

## この章について

第12章で `gamemath.geometry` を作り、`AABB` を1つだけ置きました。この章で中身を揃えます。

- `Sphere` / `Ray` / `Plane` を追加する(15.2)
- 交差判定を `:intersect` パーティションに集める(15.3)

ただし、この章の山場は幾何の数学ではありません。**交差判定の結果をどう返すか**という設計判断です。

「レイと球が交差したか」と「交差した位置」を1つの関数で返したい。素直に考えれば `std::optional<float>` です。しかしそれをやると、`GameMath` の公開インターフェイスに標準ライブラリの型が現れます。

第6章 6.5.4 と第12章 12.3.3 で決めた方針が、ここで試されます。

そして 15.4 では、**モジュールをまたぐテンプレート**の注意点を扱います。第9章 9.4.2 で「同じモジュールのパーティションどうしでは非公開の宣言も見える」と学びました。`gamemath.geometry` は `gamemath.core` とは**別のモジュール**です。この違いが、思わぬところで顔を出します。

---

## 15.1 `gamemath.geometry` モジュールを充実させる

### 15.1.1 いまの状態

第12章 12.2.4 で作った構成です。

```
gamemath.geometry
├── Geometry.ixx    export module gamemath.geometry;
│                   export import gamemath.core;
│                   export import :shapes;
└── Shapes.ixx      export module gamemath.geometry:shapes;
                    import gamemath.core;
                    → AABB / Merge / Overlaps
```

`AABB` と、その `Merge` / `Overlaps` があるだけです。

### 15.1.2 パーティションの分け方を決める

追加するものを、2つのパーティションに分けます。

| パーティション | 内容 | 性質 |
|---|---|---|
| `:shapes` | `AABB` / `Sphere` / `Ray` / `Plane` | **データの定義**。滅多に変わらない |
| `:intersect` | `Overlaps` / `Intersect` / `RayHit` | **アルゴリズム**。増え続け、改良され続ける |

分ける理由は、第12章 12.1.5 の指針1です。

> 一緒に変更されるものは、同じモジュールに置く

形状の定義は安定しています。`Sphere` が「中心と半径」であることは、この先も変わりません。

一方、交差判定は変わります。判定の組み合わせは増え(形状が N 種類なら組み合わせは N² 通り)、アルゴリズムは改良されます(分離軸定理、GJK、SIMD 化)。

そして第5章 5.1.3 で確認したとおり、**インターフェイスパーティションを変更すると `import` している側は再コンパイルされます。** アルゴリズムを触るたびに形状の定義も巻き込まれるのは無駄です。

分けておけば、`:intersect` を変更しても `:shapes` の `.ifc` は無傷です。

なお、この分割は**利用者には見えません**(第9章 9.5)。利用者は相変わらず `import gamemath.geometry;` または `import gamemath;` と書くだけです。

### 15.1.3 依存の向き

```
      :intersect
       ↓      ↓
   :shapes  gamemath.core
       ↓      ↓
   gamemath.core
```

`:intersect` は `:shapes` と `gamemath.core` の両方に依存します。`:shapes` は `gamemath.core` だけに依存します。一方向です。

`:shapes` が `:intersect` を必要とすることはありません ── 形状は自分がどう衝突判定されるかを知らなくてよい、という設計です。第13章 13.2.3 で「ベクトルは行列を知らない」と決めたのと同じ考え方です。

---

## 15.2 AABB / Sphere / Ray / Plane

### 15.2.1 4つの型の性質

実装の前に、性質を整理します。あとで効いてきます。

| 型 | 表現 | 有界か | 前提 |
|---|---|---|---|
| `AABB` | 最小点・最大点 | **有界** | `min <= max` |
| `Sphere` | 中心・半径 | **有界** | `radius >= 0` |
| `Ray` | 始点・方向 | 無限 | 方向は正規化済み |
| `Plane` | 法線・原点からの距離 | 無限 | 法線は正規化済み |

「有界かどうか」の列に注目してください。`AABB` と `Sphere` は境界ボックスを持てますが、`Ray` と `Plane` は持てません。無限に伸びているからです。

この違いは 15.4 で、コンセプトとして表現することになります。

「前提」の列も重要です。第14章 14.1.2 で単位クォータニオンについて決めたのと同じで、**関数の中でいちいち検査しません**。正規化済みであることを利用者の責任にします。そのかわり、ドキュメントに明記します。

### 15.2.2 `:shapes` を書き換える

`Geometry/Shapes.ixx` を書き換えます。第13章 13.4.6 の方針にしたがい、演算子は隠しフレンド、名前付き関数は `export` 付きの自由関数です。

```cpp
// Geometry/Shapes.ixx
export module gamemath.geometry:shapes;

import gamemath.core;
import std;

namespace gamemath {

// 軸平行境界ボックス。min <= max であること
export struct AABB
{
    Vector3 min;
    Vector3 max;

    constexpr Vector3 Center() const
    {
        return (min + max) * 0.5f;
    }

    constexpr Vector3 Extents() const
    {
        return (max - min) * 0.5f;
    }

    constexpr bool Contains(const Vector3& p) const
    {
        return p.x() >= min.x() && p.x() <= max.x()
            && p.y() >= min.y() && p.y() <= max.y()
            && p.z() >= min.z() && p.z() <= max.z();
    }

    // 自分自身が境界ボックス
    constexpr AABB Bounds() const
    {
        return *this;
    }
};

// 球。radius >= 0 であること
export struct Sphere
{
    Vector3 center;
    float   radius;

    constexpr bool Contains(const Vector3& p) const
    {
        return (p - center).LengthSquared() <= radius * radius;
    }

    constexpr AABB Bounds() const
    {
        const Vector3 r{ radius, radius, radius };
        return AABB{ center - r, center + r };
    }
};

// 半直線。direction は正規化済みであること
export struct Ray
{
    Vector3 origin;
    Vector3 direction;

    constexpr Vector3 At(float t) const
    {
        return origin + direction * t;
    }
};

// 平面。normal は正規化済み。distance は原点から法線方向への符号付き距離
export struct Plane
{
    Vector3 normal;
    float   distance;

    // 正なら法線側、負なら裏側
    constexpr float SignedDistance(const Vector3& p) const
    {
        return Dot(normal, p) - distance;
    }
};

export constexpr AABB Merge(const AABB& a, const AABB& b)
{
    AABB r{};
    for (std::size_t i = 0; i < 3; ++i) {
        r.min[i] = a.min[i] < b.min[i] ? a.min[i] : b.min[i];
        r.max[i] = a.max[i] > b.max[i] ? a.max[i] : b.max[i];
    }
    return r;
}

} // namespace gamemath
```

第12章にあった `Overlaps` は削除してください。15.3 で `:intersect` に移します。

**すべて `constexpr` にできました。** 平方根も三角関数も使っていないからです。第14章 14.3.5 の境界線のとおりです。

`Sphere::Contains` の中で `p - center` と書いています。この `operator-` は `Vector` の**隠しフレンド**です(第13章 13.4.6)。`gamemath.geometry` は別のモジュールですが、`Vector` は `gamemath.core` から `export` されているので、その隠しフレンドも ADL で見つかります。**隠しフレンドはモジュール境界を越えます。**

### 15.2.3 `Bounds()` というメンバ

`AABB` と `Sphere` に `Bounds()` を持たせました。`Ray` と `Plane` にはありません。

これは 15.4 で「境界ボックスを持つ形状」というコンセプトを定義するための布石です。**メンバ関数**にしている理由も、そこで説明します。

### 15.2.4 動作確認

`main.cpp` に追加します。

```cpp
// main.cpp

// --- 形状の確認 ---
constexpr gamemath::Sphere sphere{
    gamemath::Vector3{ 0.0f, 0.0f, 0.0f },
    2.0f
};

constexpr gamemath::AABB sphereBox = sphere.Bounds();

static_assert(sphere.Contains(gamemath::Vector3{ 1.0f, 1.0f, 1.0f }));
static_assert(!sphere.Contains(gamemath::Vector3{ 2.0f, 2.0f, 2.0f }));
static_assert(sphereBox.min.x() == -2.0f);

constexpr gamemath::Plane ground{
    gamemath::Vector3{ 0.0f, 1.0f, 0.0f },
    0.0f
};

static_assert(ground.SignedDistance(gamemath::Vector3{ 0.0f, 5.0f, 0.0f }) == 5.0f);
static_assert(ground.SignedDistance(gamemath::Vector3{ 0.0f, -3.0f, 0.0f }) == -3.0f);

std::println("Sphere bounds  = ({:.1f}, {:.1f}, {:.1f}) - ({:.1f}, {:.1f}, {:.1f})",
             sphereBox.min.x(), sphereBox.min.y(), sphereBox.min.z(),
             sphereBox.max.x(), sphereBox.max.y(), sphereBox.max.z());
```

ビルドして実行してください。

```
Sphere bounds  = (-2.0, -2.0, -2.0) - (2.0, 2.0, 2.0)
```

`static_assert` がすべて通っています。形状の判定が**コンパイル時に**行われています。

---

## 15.3 交差判定関数群をエクスポートする

### 15.3.1 やりたいこと

レイと球の交差判定を書きたい。返すべき情報は、こうです。

- 交差したか
- 交差した場合、レイに沿った距離
- 交差点の座標
- その点での法線

1つの関数で、これらをどう返すか。設計の選択肢を並べます。

### 15.3.2 案A ── `std::optional`

現代的な C++ の第一候補です。

```cpp
export std::optional<float> Intersect(const Ray& r, const Sphere& s);
```

**長所** ── 「値がない」ことを型で表現できます。値を取り出す前に必ず存在を確認させられるので、使い方を間違えにくいのです。

**短所が2つあります。**

**短所1: 情報が1つしか返せない。** 交差点や法線も欲しければ、`std::optional<RayHit>` のように結果型を別に作ることになります。それなら `std::optional` の恩恵は薄れます。

**短所2: 公開インターフェイスに標準ライブラリの型が現れる。** これが決定的です。

### 15.3.3 なぜ標準ライブラリの型を公開できないのか

第12章 12.3.2 で立てた規則を思い出してください。

> 公開する宣言のシグネチャに、他のモジュールの型が現れるなら、そのモジュールを `export import` する。

`std::optional` は、モジュール `std` の型です。だから `export import std;` が必要になります。

しかし第6章 6.5.4 で、こう決めました。

> `export import std;` はやってはいけない。利用者の環境を選ぶライブラリになってしまう。

`export import std;` をすると、`import gamemath;` した利用者に標準ライブラリ全体が見えるようになります。すると、その利用者が `#include <vector>` と書いていた場合、第6章 6.5.2 の混在問題が発生します。

**では `export import std;` を書かずに `std::optional` を返したらどうなるか。**

第12章 12.3.1 で実験したのと同じことが起きます。

```cpp
auto hit = gamemath::Intersect(ray, sphere);   // ← 通る(auto)
if (hit) { use(*hit); }                        // ← 通る(メンバは到達可能)

std::optional<float> h2 = gamemath::Intersect(ray, sphere);   // 利用者が自分で
                                                             // import std; していれば通る
```

利用者が自分で `import std;` していれば動きます。していなければ、`auto` で受けてその場で使い切る分には動き、型名を書こうとすると失敗します。

つまり、**利用者に「`import std;` も書いてください」という暗黙の要求を課すことになります。** ライブラリとしては不親切です。

### 15.3.4 残る3つの案

**案B ── 自前の結果型**

```cpp
export struct RayHit
{
    bool    hit;
    float   distance;
    Vector3 point;
    Vector3 normal;
};

export RayHit Intersect(const Ray& r, const Sphere& s);
```

**長所** ── 標準ライブラリに依存しません。必要な情報を全部返せます。`constexpr` にもできます。連続したメモリに収まるので、大量に処理するときにキャッシュに乗りやすい。

**短所** ── `hit` を確認せずに `distance` を読めてしまいます。`std::optional` のような強制力がありません。

**案C ── 出力引数**

```cpp
export bool Intersect(const Ray& r, const Sphere& s, float& outDistance);
```

**長所** ── 最小限です。DirectX や物理エンジンの API で広く使われている形です。

**短所** ── 引数の順序を間違えやすく、初期化されていない変数を渡す事故が起きます。関数の呼び出しを式の中に埋め込めません。

**案D ── 番兵値**

```cpp
export float Intersect(const Ray& r, const Sphere& s);   // 交差しなければ -1
```

**短所** ── 戻り値を検査せずに使うと静かに壊れます。第13章 13.5.4 で `operator*` を作らなかったのと同じ理由で、避けるべき設計です。

### 15.3.5 案B を選ぶ

**`GameMath` では案B(自前の結果型)を採ります。**

決め手は、実は「標準ライブラリを避けたいから」ではありません。**そもそも返したい情報が複数あるから**です。

15.3.1 で挙げた4つの情報 ── 交差の有無、距離、座標、法線 ── を返すなら、結果型は必要です。そして結果型を作るなら、`std::optional` で包む意味は薄くなります。`hit` フラグが同じ役割を果たすからです。

**設計上の制約(標準ライブラリの型を公開しない)と、領域の要求(複数の情報を返したい)が、同じ方向を指しました。**

こういうときは迷いません。逆に、もし返したい情報が `bool` と `float` だけだったら、これは本物のトレードオフになっていたはずです。「`std::optional` の安全性」と「利用者の環境を選ばない」を天秤にかけることになり、答えは一義的ではありませんでした。

**案B の短所への対策**

`hit` を確認せずに `distance` を読めてしまう問題は、**既定値**で緩和します。

```cpp
export struct RayHit
{
    bool    hit      = false;
    float   distance = 0.0f;
    Vector3 point{};
    Vector3 normal{};
};
```

交差しなかった場合は `RayHit{}` を返します。すべてゼロなので、確認を忘れても「原点で交差した」ように振る舞います。正しくはありませんが、未初期化のゴミ値よりは追跡しやすい失敗です。

### 15.3.6 `:intersect` パーティションを作る

`Geometry/Intersect.ixx` を作ってください。

```cpp
// Geometry/Intersect.ixx
export module gamemath.geometry:intersect;

import gamemath.core;
import :shapes;
import std;

namespace gamemath {

// レイの交差結果
export struct RayHit
{
    bool    hit      = false;
    float   distance = 0.0f;
    Vector3 point{};
    Vector3 normal{};
};

// ───── 重なり判定(真偽だけ) ─────

export constexpr bool Overlaps(const AABB& a, const AABB& b)
{
    return a.min.x() <= b.max.x() && a.max.x() >= b.min.x()
        && a.min.y() <= b.max.y() && a.max.y() >= b.min.y()
        && a.min.z() <= b.max.z() && a.max.z() >= b.min.z();
}

export constexpr bool Overlaps(const Sphere& a, const Sphere& b)
{
    const float r = a.radius + b.radius;
    return (a.center - b.center).LengthSquared() <= r * r;
}

export constexpr bool Overlaps(const AABB& box, const Sphere& s)
{
    // 球の中心を AABB に押し込めた点との距離を見る
    Vector3 closest = s.center;
    for (std::size_t i = 0; i < 3; ++i) {
        if (closest[i] < box.min[i]) { closest[i] = box.min[i]; }
        if (closest[i] > box.max[i]) { closest[i] = box.max[i]; }
    }
    return (closest - s.center).LengthSquared() <= s.radius * s.radius;
}

// 引数の順序を気にせず呼べるように
export constexpr bool Overlaps(const Sphere& s, const AABB& box)
{
    return Overlaps(box, s);
}

// ───── レイとの交差 ─────

// 平面との交差。平方根が要らないので constexpr にできる
export constexpr RayHit Intersect(const Ray& r, const Plane& p)
{
    const float denom = Dot(p.normal, r.direction);

    if (NearlyEqual(denom, 0.0f)) {
        return RayHit{};                       // レイが平面と平行
    }

    const float t = -p.SignedDistance(r.origin) / denom;

    if (t < 0.0f) {
        return RayHit{};                       // 後方で交差している
    }

    // 常にレイに向いた側の法線を返す
    const float sign = denom < 0.0f ? 1.0f : -1.0f;
    return RayHit{ true, t, r.At(t), p.normal * sign };
}

// 球との交差(定義は実装単位。平方根が必要)
export RayHit Intersect(const Ray& r, const Sphere& s);

} // namespace gamemath
```

`Overlaps` を4つ用意しました。`Sphere` と `AABB` の組み合わせは、引数の順序を入れ替えたものも定義しています。**利用者に引数の順序を覚えさせないためです。**

`Intersect(Ray, Plane)` は `constexpr` です。四則演算だけで書けるからです(第14章 14.3.5)。

### 15.3.7 実装単位を作る

`Geometry/Intersect.cpp` を作ってください。

```cpp
// Geometry/Intersect.cpp
module gamemath.geometry;

import std;

namespace gamemath {

RayHit Intersect(const Ray& r, const Sphere& s)
{
    const Vector3 oc = r.origin - s.center;

    // direction は正規化済みなので、二次方程式の a は 1
    const float b = Dot(oc, r.direction);
    const float c = oc.LengthSquared() - s.radius * s.radius;
    const float disc = b * b - c;

    if (disc < 0.0f) {
        return RayHit{};
    }

    const float sq = std::sqrt(disc);

    float t = -b - sq;              // 手前の交点
    if (t < 0.0f) {
        t = -b + sq;                // 始点が球の内側なら奥の交点
    }
    if (t < 0.0f) {
        return RayHit{};            // 球はレイの後方
    }

    const Vector3 p = r.At(t);
    return RayHit{ true, t, p, (p - s.center).Normalized() };
}

} // namespace gamemath
```

**`gamemath.geometry` の最初の実装単位です。** `module gamemath.geometry;` と書きます(パーティション名は付けません)。

`import :intersect;` を書いていないことに注意してください。実装単位はプライマリを暗黙に `import` し、次の手順でプライマリが `:intersect` を `export import` するので、届きます(第9章 9.4.4)。

### 15.3.8 プライマリにつなぐ

```cpp
// Geometry/Geometry.ixx
export module gamemath.geometry;

export import gamemath.core;
export import :shapes;
export import :intersect;      // ← 追加
```

### 15.3.9 `Overlaps` を移したが、利用者は無傷

ここで1つ確認しておきたいことがあります。

`Overlaps(AABB, AABB)` は、第12章では `:shapes` にありました。この章で `:intersect` に移しました。

**利用者のコードは1行も変わっていません。**

第12章 12.4.5 の表で示したとおりです。

| 変更 | 利用者への影響 |
|---|---|
| パーティションを分割・統合・改名する | **なし** |

`main.cpp` は `import gamemath;` と書いているだけで、`Overlaps` がどのパーティションにあるかを知りません。知る手段もありません(第9章 9.5)。

**これがパーティションを使う最大の見返りです。** 内部構成を後から変えられる自由が、実際に役に立った場面です。

もし第12章で `gamemath.geometry.shapes` のようなサブモジュールにしていたら、利用者の `import` を書き換えてもらう必要がありました。

### 15.3.10 動作確認

```cpp
// main.cpp

// --- 交差判定の確認 ---
constexpr gamemath::Ray ray{
    gamemath::Vector3{ 0.0f, 10.0f, 0.0f },      // 上から
    gamemath::Vector3{ 0.0f, -1.0f, 0.0f }       // 真下へ
};

// 平面との交差(コンパイル時に計算できる)
constexpr gamemath::RayHit planeHit = gamemath::Intersect(ray, ground);
static_assert(planeHit.hit);
static_assert(planeHit.distance == 10.0f);

std::println("Ray x Plane  : hit={} t={:.2f} point=({:.2f}, {:.2f}, {:.2f})",
             planeHit.hit, planeHit.distance,
             planeHit.point.x(), planeHit.point.y(), planeHit.point.z());

// 球との交差(実行時)
const gamemath::RayHit sphereHit = gamemath::Intersect(ray, sphere);
std::println("Ray x Sphere : hit={} t={:.2f} point=({:.2f}, {:.2f}, {:.2f})",
             sphereHit.hit, sphereHit.distance,
             sphereHit.point.x(), sphereHit.point.y(), sphereHit.point.z());
std::println("               normal=({:.2f}, {:.2f}, {:.2f})",
             sphereHit.normal.x(), sphereHit.normal.y(), sphereHit.normal.z());

// 交差しないレイ
constexpr gamemath::Ray missRay{
    gamemath::Vector3{ 10.0f, 10.0f, 0.0f },
    gamemath::Vector3{ 0.0f, -1.0f, 0.0f }
};
const gamemath::RayHit missed = gamemath::Intersect(missRay, sphere);
std::println("Miss         : hit={} t={:.2f}", missed.hit, missed.distance);

// 重なり判定
static_assert(gamemath::Overlaps(sphereBox, sphere));
static_assert(gamemath::Overlaps(sphere, sphereBox));   // 順序を入れ替えても呼べる
```

ビルドして実行してください。

```
Ray x Plane  : hit=true t=10.00 point=(0.00, 0.00, 0.00)
Ray x Sphere : hit=true t=8.00 point=(0.00, 2.00, 0.00)
               normal=(0.00, 1.00, 0.00)
Miss         : hit=false t=0.00
```

平面との交差がコンパイル時に確定していることに注目してください。`static_assert(planeHit.distance == 10.0f)` が通っています。

そして交差しなかった場合、`distance` が `0.00` になっています。15.3.5 で既定値を入れておいた効果です。未初期化のゴミ値なら、デバッガで見たときに混乱するところでした。

> **練習: レイと AABB の交差**
>
> スラブ法(3軸それぞれについて、レイが板状の領域に入っている区間を求め、共通部分を取る)で実装できます。平方根が要らないので `constexpr` にできます。
>
> 交差判定のなかでも呼び出し頻度が飛び抜けて高い関数です。BVH をたどるときに毎フレーム数万回呼ばれるので、第5章 5.6.5 の基準では**インターフェイスパーティションに定義を置く**べき関数になります。行数が多くても、です。

### 15.3.11 内部では標準ライブラリを使ってよい

誤解を避けるために、はっきりさせておきます。

**「標準ライブラリの型を公開しない」は、「使わない」ではありません。**

`Intersect.cpp` を見てください。`std::sqrt` を使っています。`Overlaps` の実装では `std::size_t` を使っています。

制約がかかるのは**公開する宣言のシグネチャ**だけです。実装の中では自由に使えます。

将来 `:intersect` に「レイが貫通したすべての形状を返す」関数を追加するとしても、こう書けます。

```cpp
// 内部では std::vector を使い、結果は自前の型で返す
export int IntersectAll(const Ray& r, const Sphere* spheres, int count,
                        RayHit* outHits, int maxHits);
```

公開シグネチャには生ポインタと `int` しか現れません。内部で `std::vector` を使って集計し、最後に書き出せばよいのです。

書き味は悪くなります。それが「利用者の環境を選ばない」ことの代償です。第6章 6.6.1 で決めた方針を守り続けるなら、この不便さは受け入れることになります。

**そして、これは永続的な制約ではありません。** 標準ライブラリのモジュール化が普及し、`#include` と `import std;` の混在を気にしなくてよくなれば、この制約は外せます。そのとき公開シグネチャを `std::span<const Sphere>` に変えるのは、破壊的変更になります。

**移行期のライブラリ設計は、いつ制約を外すかという判断も含む** ── そう考えておいてください。

---

## 15.4 モジュールをまたぐテンプレートの注意点

ここまで、`gamemath.geometry` から `gamemath.core` のものを使ってきました。パーティション間の `import` とは違う点があるので、整理します。

### 15.4.1 【実験】別モジュールから `kEpsilon` は見えない

`Intersect.ixx` の `Intersect(Ray, Plane)` では、平行判定に `NearlyEqual(denom, 0.0f)` を使いました。

許容誤差を明示的に書きたくなったとします。

```cpp
// Geometry/Intersect.ixx(実験)
if (NearlyEqual(denom, 0.0f, kEpsilon<float>)) {    // ← 書いてみる
    return RayHit{};
}
```

ビルドしてください。**エラーになります。**

```
error C2065: 'kEpsilon': 定義されていない識別子です。
```

`kEpsilon` は `gamemath.core:vector` にあります。しかし `export` されていません。

第9章 9.4.2 では、こう学びました。

> 同じモジュールのパーティションを `import` すると、`export` されていない宣言も見えます。

だから第14章 14.1.3 で、`:quaternion` から `kEpsilon` を使えました。`:quaternion` は `gamemath.core` のパーティションだからです。

**`gamemath.geometry` は別のモジュールです。** 通常の `import` と同じ規則が適用されます ── `export` されたものだけが見えます(第7章 7.4.3)。

**この行を元に戻してください。**

### 15.4.2 同じモジュール内と、モジュールをまたぐ場合

規則を並べておきます。

| 取り込み方 | `export` したもの | `export` していないもの |
|---|---|---|
| 同じモジュールのパーティション(`import :p;`) | 見える | **見える** |
| 別のモジュール(`import m;`) | 見える | **見えない**(到達可能ではある) |

`gamemath.core` と `gamemath.geometry` を別のモジュールにしたことで、境界が1つできました。第12章 12.4.2 で「サブモジュールの境界 = 利用者に提示する選択肢」と書きましたが、**それは同時に、ライブラリ内部でも壁になる**わけです。

これは短所ではなく、設計の一貫性です。`gamemath.geometry` は `gamemath.core` の内部事情に依存すべきではありません。依存してしまえば、`gamemath.core` の内部を変えるときに `gamemath.geometry` が壊れます。

### 15.4.3 対処 ── 公開するか、自前で持つか

許容誤差が本当に必要なら、2つの道があります。

**道1: `gamemath.core` から `export` する**

```cpp
// Core/Vector.ixx
export template <FloatingPoint T>
constexpr T kEpsilon = T(1e-6);      // ← export を追加
```

**道2: `gamemath.geometry` が自分の許容誤差を持つ**

```cpp
// Geometry/Intersect.ixx
constexpr float kGeometryEpsilon = 1e-5f;    // 幾何用。core より緩い
```

**`GameMath` では道1を採ります。** 理由は2つあります。

**理由1: すでに実質的に公開されている。**

`NearlyEqual` の既定引数が `kEpsilon<T>` です。利用者が `NearlyEqual(a, b)` と書けば、この値が使われます。つまり `kEpsilon` の値は、すでに `GameMath` の振る舞いの一部として観測可能です。

「隠しているつもりだが、実質公開されている」── この状態はよくありません。**明示的に `export` するほうが誠実です。** そして値を変えるときは、破壊的変更として扱います。

**理由2: 利用者も必要とする。**

第14章 14.4.2 で `NearlyEqual` を公開したときと同じ話です。「ライブラリの内部で必要なもの」は、たいてい利用者も必要とします。

第14章 14.4.5 で書いた助言を、もう一度書きます。

> 「テストしにくい」は、「公開すべきだ」のサインかもしれません。

「別モジュールから使えない」も、同じサインです。

`Core/Vector.ixx` の `kEpsilon` に `export` を付けてください。ビルドが通り、15.4.1 の実験も通るようになります。

### 15.4.4 既定引数は例外的に通る

細かいけれど混乱しやすい点を1つ。

`export` を付ける前の状態でも、こう書けば通っていました。

```cpp
NearlyEqual(denom, 0.0f);      // 既定引数で kEpsilon<float> が使われる
```

`kEpsilon` が見えないのに、なぜ通るのか。

**既定引数の式は、関数を宣言した場所で名前解決されます。** 呼び出した場所ではありません。`NearlyEqual` は `Core/Vector.ixx` で宣言されていて、そこでは `kEpsilon` が見えています。

呼び出し側は、`kEpsilon` という名前を知る必要がありません。値が使われるだけです。

「使えるが名前が書けない」── 第7章 7.4 で扱った**到達可能だが見えない**という状態の、別の現れ方です。

### 15.4.5 コンセプトも `export` が必要

第8章 8.3.4 で確認したことが、モジュール境界でも同じように効きます。

`gamemath.geometry` で、こう書きたくなったとします。

```cpp
export template <FloatingPoint T>
constexpr T Volume(const AABB& box);
```

これは通ります。`FloatingPoint` は `gamemath.core:vector` で `export` されているからです(第8章 8.3.2)。

もし `export` を忘れていたら、`gamemath.geometry` からは書けませんでした。第8章 8.3.4 では「利用者が自分のテンプレートに書けなくなる」と説明しましたが、**別モジュールの自分自身も、その「利用者」に含まれる**わけです。

`export` の判断は、外部の利用者だけでなく、自分の別モジュールのためでもあります。

### 15.4.6 拡張点はメンバ関数で作る

15.2.3 で「`Bounds()` をメンバ関数にした理由は 15.4 で説明する」と書きました。ここです。

「境界ボックスを持つ形状」というコンセプトを定義し、それを使う汎用関数を書きます。

```cpp
// Geometry/Intersect.ixx(末尾に追加)

// 境界ボックスを持つ形状
export template <typename S>
concept HasBounds = requires(const S& s) {
    { s.Bounds() } -> std::same_as<AABB>;
};

// 広域判定。境界ボックスだけで大雑把に振り分ける
export template <HasBounds A, HasBounds B>
constexpr bool BroadPhaseOverlaps(const A& a, const B& b)
{
    return Overlaps(a.Bounds(), b.Bounds());
}
```

`AABB` と `Sphere` は `Bounds()` を持つので、この条件を満たします。`Ray` と `Plane` は持たないので満たしません。15.2.1 の表の「有界か」の列が、コンセプトとして表現されました。

**ここで重要なのは、`s.Bounds()` が**メンバ関数の呼び出し**であることです。**

自由関数で書くこともできました。

```cpp
// こうはしない
export template <typename S>
concept HasBounds = requires(const S& s) {
    { Bounds(s) } -> std::same_as<AABB>;      // ADL に頼る
};
```

第13章 13.3.3 で確認した規則を思い出してください。

> 取り込む側の翻訳単位にある宣言は、取り込まれたモジュールの中でのオーバーロード解決や名前探索に参加しない。

自由関数版にすると、`BroadPhaseOverlaps` の中の `Bounds(a)` は ADL で探されます。そして利用者が自分の形状用に `Bounds` を書いても、モジュールの中からは見えません。

**メンバ関数の呼び出しは、名前空間の探索を経由しません。** オブジェクトの型から直接引かれるので、モジュール境界の影響を受けません。

第13章 13.3.4 で書いたことの、実際の適用例です。

> モジュール時代のカスタマイズは、ADL ではなくコンセプトとメンバ関数で行う。

### 15.4.7 【実験】利用者が自分の形状を足す

本当に拡張できるのか、確かめます。`main.cpp` に、自分の形状を定義してください。

```cpp
// main.cpp(main の外)

// 利用者が定義する独自の形状
struct Capsule
{
    gamemath::Vector3 a;
    gamemath::Vector3 b;
    float radius;

    constexpr gamemath::AABB Bounds() const
    {
        const gamemath::Vector3 r{ radius, radius, radius };
        gamemath::AABB box{};
        for (std::size_t i = 0; i < 3; ++i) {
            box.min[i] = (a[i] < b[i] ? a[i] : b[i]) - radius;
            box.max[i] = (a[i] > b[i] ? a[i] : b[i]) + radius;
        }
        return box;
    }
};
```

`main` の中で使ってみます。

```cpp
constexpr Capsule capsule{
    gamemath::Vector3{ 0.0f, 0.0f, 0.0f },
    gamemath::Vector3{ 0.0f, 4.0f, 0.0f },
    1.0f
};

static_assert(gamemath::HasBounds<Capsule>);

std::println("Capsule vs Sphere (broad) : {}",
             gamemath::BroadPhaseOverlaps(capsule, sphere));
std::println("Capsule vs Capsule (broad): {}",
             gamemath::BroadPhaseOverlaps(capsule, capsule));

// Ray は境界ボックスを持たないので、条件を満たさない
static_assert(!gamemath::HasBounds<gamemath::Ray>);
```

ビルドして実行してください。

```
Capsule vs Sphere (broad) : true
Capsule vs Capsule (broad): true
```

**利用者が定義した型が、ライブラリのテンプレートで使えました。** しかも `static_assert(gamemath::HasBounds<Capsule>)` がコンパイル時に検証されています。

これが「モジュール時代の拡張性」の形です。ADL によるカスタマイズは境界を越えられませんが、**コンセプト + メンバ関数**なら越えられます。

`Ray` が条件を満たさないことも `static_assert` で確認できています。無限に伸びる形状に境界ボックスを求めようとするコードは、コンパイル時に弾かれます。

### 15.4.8 明示的インスタンス化はモジュールをまたがない

最後に、第8章 8.4 で扱った明示的インスタンス化について注意を1つ。

`gamemath.core` では `Orthogonal<float>` と `Orthogonal<double>` を明示的にインスタンス化しました(第8章 8.4.2)。

**`gamemath.geometry` から、`gamemath.core` のテンプレートを明示的にインスタンス化してはいけません。**

```cpp
// Geometry/Intersect.cpp
// やってはいけない
template Vector<long double, 3> Orthogonal(const Vector<long double, 3>&);
```

そもそも `FloatingPoint` コンセプトが `long double` を許さないので、この例は通りませんが、仮に通ったとしても避けるべきです。実体がどのモジュールに属するのかが曖昧になり、同じ実体が複数の `.obj` に生まれる危険があります。

**明示的インスタンス化は、テンプレートを定義したモジュールの中で行ってください。** 別モジュールで追加の型が必要になったら、テンプレートの定義をインターフェイスに移す(明示的インスタンス化をやめる)ほうが素直です。

第8章 8.4.5 で「明示的インスタンス化が向いていない場面」として「利用者が任意の型で使うことを想定している」を挙げました。**別モジュールも、その意味では「利用者」です。**

---

## 15.5 この章のまとめ

- `:shapes`(データ)と `:intersect`(アルゴリズム)を分けた。変更の頻度が違うため
- この分割は利用者に見えない。だから `Overlaps` をパーティション間で移動しても利用者は無傷
- 形状は「有界かどうか」で性質が分かれる。`AABB` / `Sphere` は境界ボックスを持ち、`Ray` / `Plane` は持たない
- 正規化などの前提は関数内で検査せず、ドキュメントに明記する
- **交差判定の結果は自前の型(`RayHit`)で返す。** `std::optional` は使わない
- 理由は「標準ライブラリの型を公開したくない」だけでなく、**返したい情報が複数あるから**。制約と要求が同じ方向を指した
- 未確認アクセスの危険は、メンバの**既定値**で緩和する
- **公開シグネチャに標準ライブラリの型を出さないのは、内部で使ってはいけないという意味ではない**
- この制約は移行期のもの。いつ外すかも設計判断
- **別モジュールからは、`export` されていない宣言は見えない。** 同じモジュールのパーティションどうしとは規則が違う
- 「別モジュールから使えない」は「公開すべきだ」のサイン。`kEpsilon` は `export` した
- **既定引数の式は、宣言した場所で名前解決される。** 呼び出し側から名前が見えなくてもよい
- コンセプトの `export` は、外部の利用者のためだけでなく、自分の別モジュールのためでもある
- **拡張点はメンバ関数で作る。** ADL はモジュール境界を越えないが、コンセプト + メンバ関数なら越える
- 利用者が定義した型が、ライブラリのテンプレートで使えることを確認した
- **明示的インスタンス化は、テンプレートを定義したモジュールの中で行う**

## 次章に向けて

数学ライブラリとしての機能は、これでひととおり揃いました。第16章では性能に踏み込みます。**SIMD 対応**です。

そして、そこで本書最大の「壁」にぶつかります。

```cpp
#ifdef GAMEMATH_USE_SIMD
// SIMD 版
#else
// スカラー版
#endif
```

**この設計は、モジュールでは成立しません。** 第7章 7.1.3 で予告し、第13章 13.3.4 でも触れた問題です。マクロはモジュール境界を越えません。

- `<immintrin.h>` はどこに書くのか(グローバルモジュールフラグメントの実戦)
- マクロで切り替えられないなら、どうやって切り替えるのか
- 答えは、この章で使った**コンセプト**です

第16章は難度が上がります。読み飛ばして第17章に進んでも支障はありませんが、「モジュールが何を封じたのか」がいちばん鮮明に見える章でもあります。
