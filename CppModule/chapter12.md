# 第12章 モジュール構成を設計する

## この章について

第9章から第11章で、モジュールを分割する道具が5つ揃いました。

- サブモジュール(ドットで名付けた、独立したモジュール)
- インターフェイスパーティション
- 内部パーティション
- 実装単位
- プライベートモジュールフラグメント

道具の説明は終わりです。この章は**設計**の話です。新しい構文はほとんど出てきません。

やることは、いまの `GameMath` を作り直すことです。第9章 9.5.4 で予告した構成 ── `gamemath.core` / `gamemath.geometry` / 傘としての `gamemath` ── を実際に組み立てます。

そして途中で、1つの規則を見つけます。

> **公開インターフェイスに他モジュールの型が現れるなら、そのモジュールを `export import` する。現れないなら `import` にとどめる。**

第6章で「`export import std;` はやってはいけない」と書きました。この章では逆に「`export import gamemath.core;` は**しなければならない**」ことになります。矛盾しているようですが、上の1つの規則で両方が説明できます。

---

## 12.1 現時点の `GameMath` の全体像を図にする

### 12.1.1 図にする

まず現状を正確に把握します。第11章を終えた時点の構成です。

```
┌── モジュール: gamemath ──────────────────────────┐
│                                                 │
│  GameMath.ixx    export module gamemath;         │
│                  export import :vector;          │
│                  export import :basis;           │
│                                                 │
│  Vector.ixx      export module gamemath:vector;  │
│  Basis.ixx       export module gamemath:basis;   │
│  Detail.ixx      module gamemath:detail;    [内部] │
│  Basis.cpp       module gamemath;           [実装] │
└─────────────────────────────────────────────────┘

┌── モジュール: gamemath.random ───────────────────┐
│  Random.ixx      export module gamemath.random;  │
│                  module :private;                │
└─────────────────────────────────────────────────┘

main.cpp          import gamemath;
                  import gamemath.random;
```

問題が3つあります。

### 12.1.2 問題1 ── 利用者から見て一貫性がない

利用者は2行書きます。

```cpp
import gamemath;
import gamemath.random;
```

なぜ乱数だけ別なのか、利用者には分かりません。実は理由がありました ── 第11章 11.1.5 の制約(プライベートモジュールフラグメントを使うモジュールは1ファイルで完結しなければならない)です。

つまり、**ライブラリの内部事情が、利用者の書き方に漏れています。**

これはよくない兆候です。「なぜこう書くのか」を説明するのに実装の話をしなければならないなら、設計が漏れています。

### 12.1.3 問題2 ── これから増えるものの置き場所がない

第13章以降で追加するものを並べます。

| 章 | 追加するもの | 性質 |
|---|---|---|
| 13 | `Matrix4x4`、変換行列 | 線形代数 |
| 14 | `Quaternion`、`Slerp` | 線形代数 |
| 15 | `AABB` / `Sphere` / `Ray` / `Plane`、交差判定 | 幾何 |

前の2つと最後で、性質が違います。

`Matrix4x4` と `Quaternion` は、`Vector` と同じ層です。互いを必要とし、常に一緒に使われます。

一方、幾何プリミティブと交差判定は、その上に乗ります。`Vector` を使いますが、`Vector` は幾何を必要としません。そして**幾何を使わない人がいます。** UI のレイアウト計算をしているコードは、`Vector2` は使っても `AABB` の交差判定は使いません。

全部を `gamemath` に詰め込むと、`Vector2` だけ使いたい人にも交差判定の `.ifc` を読ませることになります。第9章 9.5 で確認したとおり、**パーティションでは利用者に選ばせられません。**

### 12.1.4 問題3 ── 名前が実態と合っていない

いまのモジュール `gamemath` には、ベクトルと座標系だけが入っています。名前は「ゲーム数学の全部」なのに、中身は一部です。

これから幾何を足すとして、`gamemath` に入れるのか、別モジュールにするのか。別モジュールにするなら、`gamemath` という名前は「ベクトルと行列」を指すことになり、名前と実態がずれます。

### 12.1.5 設計の指針を決める

作り直す前に、何を基準に分けるのかを決めます。ここが曖昧だと、あとで揺れます。

**指針1: 一緒に変更されるものは、同じモジュールに置く**

`Vector` と `Matrix4x4` は、片方を変えればもう片方も変えることが多い型です。分けても再コンパイルが連鎖するだけで、意味がありません。

**指針2: 一緒に使われるものは、同じモジュールに置く**

利用者が常に両方を `import` するなら、分ける意味は薄いです。

**指針3: 利用者が「これだけ欲しい」と言う単位で、サブモジュールを切る**

これが分割の主要な基準です。「幾何は要らない」「乱数だけ欲しい」── そういう要求が現実にあるなら、そこが境界です。

**指針4: 依存は一方向にする**

第10章 10.4 の原則です。モジュール間でも同じです。

**指針5: ファイルの分割にはパーティションを使う**

利用者には見せません(第9章 9.5)。

この5つで、構成が決まります。

---

## 12.2 `gamemath` / `gamemath.core` / `gamemath.geometry` の階層化

### 12.2.1 目標の構成

指針にしたがうと、こうなります。

```
┌── gamemath  (傘) ──────────────────────────────────┐
│  export import gamemath.core;                      │
│  export import gamemath.geometry;                  │
│  export import gamemath.random;                    │
└────────────────────────────────────────────────────┘
        ↑                ↑                  ↑
┌───────────────┐ ┌──────────────────┐ ┌──────────────┐
│ gamemath.core │ │gamemath.geometry │ │gamemath.random│
│   :vector     │ │   :shapes        │ │  (1ファイル)   │
│   :basis      │ │   :intersect(15章)│ │              │
│   :matrix(13) │ │                  │ │              │
│   :quaternion │ │ import           │ │              │
│   :detail     │ │   gamemath.core  │ │              │
└───────────────┘ └──────────────────┘ └──────────────┘
        ↑                  │
        └──────────────────┘
```

**3つのサブモジュールと、1つの傘。** それぞれの中は、パーティションで分割します。

利用者は、必要に応じて選べます。

```cpp
import gamemath;             // 全部
import gamemath.core;        // 線形代数だけ
import gamemath.geometry;    // 幾何(core も付いてくる)
import gamemath.random;      // 乱数だけ
```

これを4手順で作ります。

### 12.2.2 手順1 ── `gamemath` を `gamemath.core` に改名する

いまのモジュール `gamemath` は、線形代数の担当になります。改名します。

**変更するのは4か所です。**

```cpp
// GameMath.ixx → Core.ixx にファイル名も変更
export module gamemath.core;      // ← 変更

export import :vector;
export import :basis;
```

```cpp
// Vector.ixx
export module gamemath.core:vector;   // ← 変更
```

```cpp
// Basis.ixx
export module gamemath.core:basis;    // ← 変更
```

```cpp
// Detail.ixx
module gamemath.core:detail;          // ← 変更
```

```cpp
// Basis.cpp
module gamemath.core;                 // ← 変更
```

**パーティションを `import` する行は変えません。**

```cpp
import :vector;      // モジュール名を書かないので、変更不要
import :detail;
```

第9章 9.3.2 で「パーティションを `import` するときはモジュール名を書かない」と説明しました。その設計のおかげで、モジュール名を変えてもパーティション間の `import` は無傷です。地味ですが、ありがたい性質です。

**名前空間は変えません。** 相変わらず `namespace gamemath` です。第4章 4.5 で確認したとおり、モジュール名と名前空間は独立しています。`gamemath.core` というモジュールが `gamemath` 名前空間の中身を提供する ── 何も問題ありません。

この時点で `main.cpp` は壊れます。次の手順まで待ってください。

### 12.2.3 手順2 ── フォルダーを整える

ファイルが増えてきたので、フォルダーで分けます。

```
GameMath/
├── GameMath.ixx           (次の手順で作る)
├── Core/
│   ├── Core.ixx
│   ├── Vector.ixx
│   ├── Basis.ixx
│   ├── Detail.ixx
│   └── Basis.cpp
├── Geometry/
│   └── (次の手順で作る)
├── Random/
│   └── Random.ixx
└── main.cpp
```

**Visual Studio の「フィルター」は、実際のフォルダーではありません。** ソリューションエクスプローラー上の見た目だけを整えるものです。

本書では、**ディスク上に実際のフォルダーを作ることを勧めます。** 理由は第20章です。CMake でビルドする話をするとき、実際のフォルダー構造があるほうが素直に書けます。

手順は、エクスプローラーでフォルダーを作ってファイルを移動し、Visual Studio 側では一度プロジェクトから除外して再度追加する、という流れになります。面倒ですが一度だけです。

**モジュール名とフォルダー名の対応も、言語仕様では決まっていません。** 第9章 9.4.6 で決めた規約に、1つ足します。

> **規約4: サブモジュールごとにフォルダーを作り、フォルダー名はモジュール名の末尾要素とする**

`gamemath.core` → `Core/`。探しやすさのためです。

### 12.2.4 手順3 ── `gamemath.geometry` を作る

新しいサブモジュールを作ります。中身は `AABB`(軸平行境界ボックス)です。ゲームで最も使われる幾何プリミティブで、`Vector3` だけあれば書けます。

`Geometry/Shapes.ixx` を作ってください。

```cpp
// Geometry/Shapes.ixx
export module gamemath.geometry:shapes;

import gamemath.core;
import std;

namespace gamemath {

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

export constexpr bool Overlaps(const AABB& a, const AABB& b)
{
    return a.min.x() <= b.max.x() && a.max.x() >= b.min.x()
        && a.min.y() <= b.max.y() && a.max.y() >= b.min.y()
        && a.min.z() <= b.max.z() && a.max.z() >= b.min.z();
}

} // namespace gamemath
```

**2行目に注目してください。**

```cpp
import gamemath.core;
```

パーティションではなく、**別のモジュール**を `import` しています。本書で初めてのモジュール間の依存です。

書き方はいつもと同じです。パーティションのように名前を省略することはできません。`gamemath.geometry` から見れば `gamemath.core` は完全に外部のモジュールで、名前が似ているのは人間の都合にすぎません(第3章 3.2.4)。

次に、`gamemath.geometry` のプライマリを作ります。`Geometry/Geometry.ixx` です。

```cpp
// Geometry/Geometry.ixx
export module gamemath.geometry;

export import gamemath.core;      // ← export が付いている
export import :shapes;
```

`export import gamemath.core;` の意味は 12.3 で詳しく扱います。いまは「必要なもの」として受け入れてください。

### 12.2.5 手順4 ── 傘としての `gamemath` を作る

最後に、全部をまとめる「傘」を作ります。プロジェクトのルートに `GameMath.ixx` を作ってください。

```cpp
// GameMath.ixx
export module gamemath;

export import gamemath.core;
export import gamemath.geometry;
export import gamemath.random;
```

これだけです。**中身を持たないモジュール**です。3つのサブモジュールを取り込んで、そのまま再公開しているだけです。

> **`gamemath` と `gamemath.core` の関係**
>
> 名前が親子のように見えますが、C++ にとっては無関係な2つの名前です(第3章 3.2.4)。
>
> `gamemath` が `gamemath.core` を `export import` しているという**その事実だけ**が、両者の関係です。名前を `umbrella` と `linalg` にしても、動作はまったく同じです。
>
> 名前を揃えるのは、人間が構造を推測しやすいようにするためです。

### 12.2.6 動作確認

`main.cpp` を書き換えます。

```cpp
// main.cpp
import std;
import gamemath;      // ← 1行になった

int main()
{
    // ...(既存のコードはそのまま動く)

    // --- 幾何の確認 ---
    constexpr gamemath::AABB boxA{
        gamemath::Vector3{ 0.0f, 0.0f, 0.0f },
        gamemath::Vector3{ 2.0f, 2.0f, 2.0f }
    };
    constexpr gamemath::AABB boxB{
        gamemath::Vector3{ 1.0f, 1.0f, 1.0f },
        gamemath::Vector3{ 3.0f, 3.0f, 3.0f }
    };

    static_assert(gamemath::Overlaps(boxA, boxB));
    static_assert(boxA.Contains(gamemath::Vector3{ 1.0f, 1.0f, 1.0f }));

    constexpr gamemath::AABB merged = gamemath::Merge(boxA, boxB);

    const gamemath::Vector3 c = merged.Center();
    std::println("Merged center  = ({:.2f}, {:.2f}, {:.2f})", c.x(), c.y(), c.z());
    std::println("Overlaps       = {}", gamemath::Overlaps(boxA, boxB));

    return 0;
}
```

`import gamemath.random;` の行は不要になりました。傘が取り込んでいます。

ビルドして実行してください。

```
Merged center  = (1.50, 1.50, 1.50)
Overlaps       = true
```

`static_assert` が通っていることにも注目してください。`Merge` も `Overlaps` も `constexpr` なので、コンパイル時に判定されています(第8章 8.5)。

### 12.2.7 モジュール名の付け方について

`gamemath.core` という名前を選びましたが、他の候補もありました。

| 候補 | 長所 | 短所 |
|---|---|---|
| `gamemath.core` | 「土台」だと分かる | 中身が何かは分からない |
| `gamemath.linalg` | 線形代数だと明示 | 略語が分かりにくい |
| `gamemath.vector` | 具体的 | 行列も入るので嘘になる |
| `gamemath.math` | ── | `gamemath` と重複していて意味がない |

`core` を選んだのは、**「他が依存する土台である」という関係を名前で表現できる**からです。中身が何かは、`import gamemath.core;` した人がすぐに分かります。一方、依存関係は名前を見ないと分かりません。

モジュール名は**利用者が最初に目にするドキュメント**です。中身の列挙よりも、構造を伝えるほうが役に立ちます。

---

## 12.3 `export import` による再エクスポートの功罪

### 12.3.1 【実験】`export` を外すとどうなるか

`Geometry.ixx` で `export import gamemath.core;` と書きました。この `export` が何をしているのか、外して確かめます。

```cpp
// Geometry/Geometry.ixx
export module gamemath.geometry;

import gamemath.core;        // ← export を削除
export import :shapes;
```

そして `main.cpp` で、**傘を使わず**に `gamemath.geometry` だけを `import` してみてください。

```cpp
// main.cpp(実験用に一時的に)
import std;
import gamemath.geometry;    // ← 傘ではなく、幾何だけ

int main()
{
    constexpr gamemath::AABB boxA{
        gamemath::Vector3{ 0.0f, 0.0f, 0.0f },      // ← ここ
        gamemath::Vector3{ 2.0f, 2.0f, 2.0f }
    };
    // ...
}
```

ビルドしてください。**エラーになります。**

```
error C2065: 'Vector3': 定義されていない識別子です。
```

`AABB` は見えています。しかし `Vector3` が見えません。

これは第7章 7.5 で `BasisPair` に起きたことと、まったく同じ現象です。

- `AABB` は `export` されているので**見える**
- `AABB` のメンバの型である `Vector3` は、`.ifc` を通じて**到達可能**だが**見えない**

だから、こう書けば動きます。

```cpp
constexpr auto boxA = gamemath::Merge(...);   // auto なら OK
```

しかし `AABB` を構築するには `Vector3` という名前が必要です。書けません。

**利用者から見れば「`AABB` は使えるのに作れない」という理解不能な状態です。**

### 12.3.2 規則 ── 公開インターフェイスに出る型は再エクスポートする

`export` を戻してください。

```cpp
export import gamemath.core;
```

これでビルドが通ります。`gamemath.geometry` を `import` した側に、`gamemath.core` の公開内容もそのまま届くようになりました。

ここから規則が導けます。第7章 7.5.4 で立てた原則の、モジュール版です。

第7章の原則:

> 公開する宣言のシグネチャに現れる型は、必ず公開する。

第12章の原則:

> **公開する宣言のシグネチャに、他のモジュールの型が現れるなら、そのモジュールを `export import` する。**

`AABB` のメンバは `Vector3` です。`Merge` の引数も戻り値も `AABB` を経由して `Vector3` を含みます。`Vector3` は `gamemath.core` のものです。だから `gamemath.core` を再エクスポートしなければなりません。

判断の手順はこうです。

1. このモジュールが公開している型・関数のシグネチャを見る
2. そこに他のモジュール由来の型が現れているか確認する
3. 現れていれば、そのモジュールを `export import`
4. 現れていなければ、`import` にとどめる

### 12.3.3 なぜ `import std;` は再エクスポートしないのか

第6章 6.5.4 で、こう書きました。

> `export import std;` はやってはいけない。利用者の環境を選ぶライブラリになってしまう。

いま「他モジュールの型が公開インターフェイスに出るなら再エクスポートせよ」と言っています。矛盾していないでしょうか。

矛盾していません。**`GameMath` の公開インターフェイスに、標準ライブラリの型は1つも現れていないからです。**

確認してみてください。`Vector`、`AABB`、`BasisPair`、`Random` ── どれも `float` と自前の型だけでできています。`std::vector` も `std::string` も `std::optional` も出てきません。

だから 12.3.2 の手順の4番、「現れていなければ `import` にとどめる」に該当します。

**1つの規則で、両方が説明できました。**

そして、これは第6章 6.5.4 で書いた設計方針の見返りです。

> 公開インターフェイスに標準ライブラリの型を出さない。

あの方針を守ってきたおかげで、`export import std;` が不要になり、利用者の環境を選ばないライブラリになっています。**第15章で「交差判定の結果をどう返すか」を決めるとき、この方針が試されます。** `std::optional<Hit>` を返したくなるからです。

### 12.3.4 傘モジュールの代償

`export import` の便利さの裏側を見ます。

`import gamemath;` と書くと、`gamemath.core` と `gamemath.geometry` と `gamemath.random` の `.ifc` を**すべて**読み込みます。

つまり、**傘を使うかぎり、分割した意味はありません。**

`Vector2` だけ使いたい人が `import gamemath;` と書けば、幾何も乱数も読み込まれます。第9章 9.1.4 でパーティションについて指摘した代償が、傘モジュールでも同じように発生します。

では傘は無駄なのか ── そうでもありません。

- **試作段階では便利です。** 何が必要か分からないうちは、全部入りで書き始めたい
- **小さなプロジェクトでは、分割の効果より書きやすさが勝ちます**
- **「まず動かす」ための入口が1つあると、学習コストが下がります**

要するに、傘は**便利さと引き換えにビルド時間を払う選択肢**です。提供はしますが、それを使うかどうかは利用者に決めてもらいます。

### 12.3.5 `export import` を使わないという選択

もう1つの立場を紹介しておきます。

**傘モジュールを提供しない、という設計です。**

利用者は必ず具体的なサブモジュールを指定します。

```cpp
import gamemath.core;
import gamemath.geometry;
```

長所は、**利用者が依存を意識せざるを得ないこと**です。`import` の行を見れば、そのファイルが何に依存しているかが一目で分かります。ビルド時間も自然に最小になります。

短所は、**書くのが面倒なこと**と、**ライブラリの内部構成を知る必要があること**です。

大規模なプロジェクトほど前者の価値が上がり、小規模なプロジェクトほど後者の負担が重くなります。

`GameMath` は入門書の題材なので、傘を提供します。しかし業務のライブラリで「傘を作らない」という判断をするなら、それは十分に合理的です。

---

## 12.4 利用者から見た `import` の粒度を決める

### 12.4.1 選べるようになった

いま、利用者には4つの選択肢があります。

| 書き方 | 得られるもの | `.ifc` の読み込み量 |
|---|---|---|
| `import gamemath;` | 全部 | 最大 |
| `import gamemath.core;` | `Vector` / `Matrix4x4` / `Quaternion` / 座標系 | 中 |
| `import gamemath.geometry;` | 幾何 **+ `gamemath.core`** | 大 |
| `import gamemath.random;` | 乱数のみ | 小 |

3行目に注意してください。`gamemath.geometry` は `gamemath.core` を再エクスポートしているので、幾何だけを取り込むことはできません。**それが正しい設計です**(12.3.2)。幾何は `Vector3` なしには成立しないからです。

4行目の `gamemath.random` は、`gamemath.core` を必要としません。だから独立して軽いままです。

### 12.4.2 粒度はサブモジュールの単位で決まる

利用者が選べる粒度は、**サブモジュールの区切りそのもの**です。それより細かくは選べません。パーティションは外から見えないからです(第9章 9.5)。

これが、設計における最も重要な意思決定です。

**サブモジュールの境界 = 利用者に提示する選択肢**

だから、指針3 ──「利用者が『これだけ欲しい』と言う単位でサブモジュールを切る」── が効いてきます。逆に言えば、**利用者が選ばない境界でサブモジュールを切っても、複雑さが増えるだけです。**

たとえば `gamemath.core` を `gamemath.vector` と `gamemath.matrix` に分けるべきか ── 指針1と2に照らすと、分けるべきではありません。両者は一緒に変更され、一緒に使われます。分けても、利用者は常に両方 `import` することになります。

### 12.4.3 どの粒度を推奨するか

ライブラリの作者は、推奨を示す責任があります。「好きなように使ってください」では利用者が迷います。

`GameMath` の推奨は、こうします。

**推奨1: プロダクションコードでは、具体的なサブモジュールを `import` する**

```cpp
import gamemath.core;
```

**推奨2: 試作・学習・小さなツールでは、傘を使ってよい**

```cpp
import gamemath;
```

**推奨3: 1つの翻訳単位に、傘と個別モジュールを混ぜない**

```cpp
import gamemath;         // 悪い例
import gamemath.core;    // 重複していて、意図が読めない
```

害はありませんが、読み手が混乱します。

### 12.4.4 ドキュメントに書くべきこと

モジュール構成は、利用者にとって API の一部です。次の3つは必ず文書化してください。

**1. 提供するモジュールの一覧と、それぞれの中身**

**2. モジュール間の依存関係**

「`gamemath.geometry` を `import` すると `gamemath.core` も付いてくる」を明記します。これは利用者が観測できるふるまいです。

**3. パーティションは非公開であること**

書かないと、`.ixx` を見た利用者が `import gamemath.core:vector;` と書こうとして、エラーの理由が分からずに困ります。

逆に、**書いてはいけないこともあります。** パーティションの一覧や、内部パーティションの存在です。それらは変更する自由を残しておくべき部分です。ドキュメントに書いた時点で、利用者はそれに依存し始めます。

### 12.4.5 構成を変えるときの互換性

将来、構成を変えたくなったときにどうなるかを整理しておきます。

| 変更 | 利用者への影響 |
|---|---|
| パーティションを分割・統合・改名する | **なし** |
| 内部パーティションを追加・削除する | **なし** |
| 実装単位を分割する | **なし** |
| サブモジュールを追加する | なし(傘に追加すれば、傘の利用者は自動的に使える) |
| サブモジュールを改名・分割する | **`import` の書き換えが必要** |
| 傘の構成を変える | 場合による |

**パーティションの変更が利用者に影響しない** ── これがパーティションを使う最大の見返りです。ファイル構成をいくら変えても、利用者は無傷です。

逆に、**サブモジュールの境界は簡単には変えられません。** だから、最初に決めるときに慎重になるべき部分です。

「迷ったらパーティションにしておく」が安全な方針です。あとでサブモジュールに昇格させることはできますが、その逆(公開したサブモジュールを内部のパーティションに引っ込める)は利用者のコードを壊します。

---

## 12.5 設計の指針まとめ

この章で使った判断基準を、使える形にまとめます。

**モジュールの境界を決める**

1. 一緒に変更されるものは、同じモジュールに置く
2. 一緒に使われるものは、同じモジュールに置く
3. 利用者が「これだけ欲しい」と言う単位で、サブモジュールを切る
4. 依存は一方向にする(第10章 10.4)
5. 迷ったらパーティションにする(あとで昇格できる)

**ファイルの分割**

6. ファイルを分けたいだけなら、パーティションを使う
7. 実装だけを共有したいなら、内部パーティション(第10章)
8. 1ファイルで完結する小さなモジュールなら、プライベートモジュールフラグメント(第11章)

**`export import` の判断**

9. 公開インターフェイスに他モジュールの型が現れるなら `export import`、現れないなら `import`
10. 標準ライブラリは `export import` しない。そのために、公開インターフェイスに標準ライブラリの型を出さない

**利用者への提示**

11. モジュール一覧と依存関係を文書化する
12. パーティションは文書化しない(変更の自由を残す)
13. 推奨する `import` の粒度を示す

---

## 12.6 この章のまとめ

- サブモジュールの境界は、**利用者に提示する選択肢**そのもの。それより細かい粒度は提供できない
- `GameMath` は `gamemath.core` / `gamemath.geometry` / `gamemath.random` の3つと、傘としての `gamemath` に整理した
- モジュール名を変えても、パーティションを `import` する行(`import :vector;`)は変更不要
- 名前空間はモジュール構成とは無関係。すべて `gamemath` のまま
- モジュール名は、中身の列挙よりも**構造**を伝えるほうが役に立つ
- **公開インターフェイスに他モジュールの型が現れるなら、そのモジュールを `export import` する**
- `export` を忘れると、「型は使えるのに名前が書けない」状態になる(第7章 7.5 と同じ現象)
- `import std;` を再エクスポートしないのは、公開インターフェイスに標準ライブラリの型が出ていないから。同じ規則で説明できる
- 傘モジュールは便利だが、使えば分割の効果は消える。**提供するが、使うかは利用者が決める**
- 傘を提供しないという設計も合理的
- パーティションの変更は利用者に影響しない。サブモジュールの境界は簡単に変えられない
- 迷ったらパーティションにしておく。昇格はできるが、降格はできない

## 次章に向けて

第3部が終わりました。構成が決まったので、あとは中身を埋めていく作業です。

第4部では、`GameMath` を実際に数学ライブラリとして充実させます。

- 第13章 `Matrix4x4` ── `gamemath.core:matrix` パーティションを追加します。演算子オーバーロードと **ADL** の落とし穴、そして隠しフレンドという技法を扱います
- 第14章 `Quaternion` ── `Matrix4x4` との相互変換で、第10章 10.4.5 の「循環しそうになったときの対処」が実戦になります
- 第15章 幾何と交差判定 ── `gamemath.geometry:intersect` を追加します。そして「交差判定の結果を `std::optional` で返すか」という、第6章と第12章の設計方針を試す判断が待っています

構成を先に決めておいたおかげで、これから追加するものの置き場所は、もう決まっています。**設計を先に済ませておく**というのは、そういうことです。
