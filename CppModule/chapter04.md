# 第4章 Vector3 をモジュールにする

## この章について

第3章までで、モジュールの骨格 ── `export module` と `import` ── を確認しました。ここから `GameMath` の実装に入ります。

最初のお題は `Vector3` です。ゲーム開発で最も頻繁に使う型であり、これから作る行列やクォータニオンの土台にもなります。

この章の進め方は、少し変わっています。

1. **まずヘッダファイルで書きます**(4.1)
2. それをそのまま `.ixx` に移します(4.2)
3. `export` を付けていきます(4.3)
4. 名前空間を導入します(4.4)

「わざわざヘッダで書いてから移す」のは遠回りに見えますが、これには理由があります。実務でモジュールを導入するとき、多くの場合ゼロから書くのではなく**既存のヘッダを移す**ことになるからです。そのとき何を直す必要があるのかを、ここで体験しておきます。

そして 4.5 で、本書で最も間違えやすい論点を扱います。**モジュール名と名前空間は別物である**という話です。手を動かして確かめてもらいます。

---

## 4.1 まずはヘッダで書いた Vector3 を用意する

### 4.1.1 プロジェクトを作る

新しいプロジェクトを作ります。手順は第3章 3.1.1 と同じです。

1. 「空のプロジェクト」、名前は `GameMath`
2. [構成]「すべての構成」、[プラットフォーム]「すべてのプラットフォーム」
3. **C++ 言語標準** を `/std:c++latest`
4. **ISO C++23 標準ライブラリ モジュールのビルド** を「はい」

このプロジェクトは、第22章まで育て続けます。当面は動作を確認したいので実行可能なプロジェクト(`.exe`)にしておきます。ライブラリとして切り出す話は第20章で扱います。

### 4.1.2 `Vector3.h` を書く

「ヘッダー ファイル」を右クリック → [追加] → [新しい項目] → 「ヘッダー ファイル (.h)」で `Vector3.h` を追加します。

```cpp
// Vector3.h
#pragma once

struct Vector3
{
    float x;
    float y;
    float z;

    Vector3& operator+=(const Vector3& rhs)
    {
        x += rhs.x;
        y += rhs.y;
        z += rhs.z;
        return *this;
    }

    Vector3& operator-=(const Vector3& rhs)
    {
        x -= rhs.x;
        y -= rhs.y;
        z -= rhs.z;
        return *this;
    }

    Vector3& operator*=(float s)
    {
        x *= s;
        y *= s;
        z *= s;
        return *this;
    }

    float LengthSquared() const
    {
        return x * x + y * y + z * z;
    }
};

inline Vector3 operator+(const Vector3& a, const Vector3& b)
{
    return Vector3{ a.x + b.x, a.y + b.y, a.z + b.z };
}

inline Vector3 operator-(const Vector3& a, const Vector3& b)
{
    return Vector3{ a.x - b.x, a.y - b.y, a.z - b.z };
}

inline Vector3 operator*(const Vector3& v, float s)
{
    return Vector3{ v.x * s, v.y * s, v.z * s };
}

inline float Dot(const Vector3& a, const Vector3& b)
{
    return a.x * b.x + a.y * b.y + a.z * b.z;
}

inline Vector3 Cross(const Vector3& a, const Vector3& b)
{
    return Vector3{
        a.y * b.z - a.z * b.y,
        a.z * b.x - a.x * b.z,
        a.x * b.y - a.y * b.x
    };
}
```

ごく普通のヘッダオンリーな実装です。3点だけ補足します。

**`inline` が付いている理由。** 自由関数(クラスのメンバではない関数)をヘッダに定義するとき、`inline` を付けないと、このヘッダを `#include` した `.cpp` の数だけ定義が生まれ、リンク時に多重定義エラーになります。第3章 3.6.2 で触れた話です。クラスのメンバ関数をクラス定義の中に書いた場合は暗黙に `inline` になるので、`operator+=` などには付けていません。

**名前空間がない理由。** 本来なら `namespace gamemath` に入れるべきですが、意図的に省いています。4.4 でモジュールと合わせて導入します。

**`Length()` がない理由。** 平方根が必要になるからです。これには理由があるので、4.6 で説明します。

### 4.1.3 動かして確認する

`main.cpp` を追加します。

```cpp
// main.cpp
#include <cstdio>

#include "Vector3.h"

int main()
{
    const Vector3 a{ 1.0f, 2.0f, 3.0f };
    const Vector3 b{ 4.0f, 5.0f, 6.0f };

    const Vector3 sum = a + b;
    const Vector3 crs = Cross(a, b);

    std::printf("a + b   = (%.1f, %.1f, %.1f)\n", sum.x, sum.y, sum.z);
    std::printf("Cross   = (%.1f, %.1f, %.1f)\n", crs.x, crs.y, crs.z);
    std::printf("Dot     = %.1f\n", Dot(a, b));
    std::printf("|a|^2   = %.1f\n", a.LengthSquared());

    return 0;
}
```

ビルドして実行してください。

```
a + b   = (5.0, 7.0, 9.0)
Cross   = (-3.0, 6.0, -3.0)
Dot     = 32.0
|a|^2   = 14.0
```

これが**出発点**です。この出力を覚えておいてください。以降、コードをモジュールに作り替えていきますが、この出力が変わらないことが「正しく移せた」証拠になります。

---

## 4.2 そのまま `.ixx` に移す

### 4.2.1 `Vector3.ixx` を追加する

「ソース ファイル」を右クリック → [追加] → [新しい項目] → 「C++ モジュール インターフェイス ユニット (.ixx)」で `Vector3.ixx` を追加します。テンプレートの中身は削除して空にしてください。

### 4.2.2 中身を移して、3か所を直す

`Vector3.h` の中身をまるごとコピーして `Vector3.ixx` に貼り付け、次の3か所を直します。

**直す1: `#pragma once` を削除する**

第3章 3.4.3 で確認したとおり、モジュールにインクルードガードは要りません。書く意味がありません。

**直す2: 先頭にモジュール宣言を置く**

```cpp
export module gamemath.vector;
```

モジュール名は `gamemath.vector` にします。ドットに階層の意味がないことは 3.2.4 で確認しました。これは「`gamemath` に関係する `vector` のモジュール」という**人間向けの命名**です。

**直す3: `inline` を削除する**

自由関数から `inline` を外します。モジュールではテキストが複数の翻訳単位に貼り付けられないので、多重定義の心配がありません。

結果、`Vector3.ixx` はこうなります。

```cpp
// Vector3.ixx
export module gamemath.vector;

struct Vector3
{
    float x;
    float y;
    float z;

    Vector3& operator+=(const Vector3& rhs)
    {
        x += rhs.x;
        y += rhs.y;
        z += rhs.z;
        return *this;
    }

    Vector3& operator-=(const Vector3& rhs)
    {
        x -= rhs.x;
        y -= rhs.y;
        z -= rhs.z;
        return *this;
    }

    Vector3& operator*=(float s)
    {
        x *= s;
        y *= s;
        z *= s;
        return *this;
    }

    float LengthSquared() const
    {
        return x * x + y * y + z * z;
    }
};

Vector3 operator+(const Vector3& a, const Vector3& b)
{
    return Vector3{ a.x + b.x, a.y + b.y, a.z + b.z };
}

Vector3 operator-(const Vector3& a, const Vector3& b)
{
    return Vector3{ a.x - b.x, a.y - b.y, a.z - b.z };
}

Vector3 operator*(const Vector3& v, float s)
{
    return Vector3{ v.x * s, v.y * s, v.z * s };
}

float Dot(const Vector3& a, const Vector3& b)
{
    return a.x * b.x + a.y * b.y + a.z * b.z;
}

Vector3 Cross(const Vector3& a, const Vector3& b)
{
    return Vector3{
        a.y * b.z - a.z * b.y,
        a.z * b.x - a.x * b.z,
        a.x * b.y - a.y * b.x
    };
}
```

> **`inline` を消すべきなのか、消してもよいだけなのか**
>
> モジュールインターフェイス単位に `inline` を残しておいても、コンパイルは通ります。害はありません。
>
> ただし、`inline` の意味は変わります。ヘッダでの `inline` は「多重定義を許す」という目的でしたが、モジュールではその問題が存在しないので、残った意味は「インライン展開のヒント」だけになります。目的が違う以上、惰性で残すと読み手を混乱させます。
>
> 本書では、モジュールに移すときに `inline` を外す方針にします。インライン展開を明示的に促したい箇所には、意図をもって付け直します。第16章で SIMD を扱うときに、この判断が必要になります。

### 4.2.3 利用側を書き換える

`main.cpp` の `#include` を `import` に変えます。

```cpp
// main.cpp
#include <cstdio>

import gamemath.vector;   // ← 変更

int main()
{
    const Vector3 a{ 1.0f, 2.0f, 3.0f };
    // ...(以下同じ)
}
```

そして、**`Vector3.h` をプロジェクトから削除してください。** 役目を終えました(ファイルごと消して構いません)。

### 4.2.4 ビルドすると失敗する

ビルドしてください。**エラーになります。**

```
error C2065: 'Vector3': 定義されていない識別子です。
```

`main.cpp` から `Vector3` が見えません。当然です。何も `export` していないからです。

第3章 3.5.2 で見たエラーと同じ形をしています。「アクセスできません」ではなく「**識別子がありません**」。`Vector3` は `gamemath.vector` モジュールの中にだけ存在し、外からは存在しないのと同じ状態です。

移す作業はここまでで完了しています。次の節で公開していきます。

---

## 4.3 クラス全体を `export` する

### 4.3.1 まず `struct` に `export` を付ける

`Vector3.ixx` の `struct Vector3` に `export` を付けます。

```cpp
export struct Vector3
{
    // ...
};
```

ビルドしてください。**まだエラーになります。** ただし、エラーの内容が変わりました。

```
error C2676: 二項演算子 '+': 'const Vector3' は、この演算子または定義済の演算子に
             適切な型への変換の定義を行いません。
```

`Vector3` は見えるようになりました。しかし `operator+` が見つかりません。

### 4.3.2 なぜ演算子が見えないのか

**クラスを `export` しても、そのクラスを引数に取る自由関数は `export` されません。**

当たり前といえば当たり前です。`operator+` はクラスのメンバではなく、独立した関数だからです。`export` は宣言ごとに付けるものなので、クラスに付けた `export` が別の宣言に及ぶことはありません。

しかしこれは、ヘッダから移行するときに**最も踏みやすい落とし穴**です。ヘッダの世界では「ファイル単位」で公開・非公開が決まっていたので、「クラスを公開したら、周辺の関数も一緒に公開される」という感覚が身についています。モジュールは宣言単位です。

数学ライブラリは特に影響を受けます。演算子オーバーロード、`Dot`、`Cross`、変換関数 ── 型に付随する自由関数がたくさんあるからです。**型を公開したら、その型を使う自由関数も一つずつ公開する**と覚えてください。

### 4.3.3 自由関数にも `export` を付ける

5つの自由関数すべてに `export` を付けます。

```cpp
export Vector3 operator+(const Vector3& a, const Vector3& b) { /* ... */ }
export Vector3 operator-(const Vector3& a, const Vector3& b) { /* ... */ }
export Vector3 operator*(const Vector3& v, float s)          { /* ... */ }
export float   Dot(const Vector3& a, const Vector3& b)       { /* ... */ }
export Vector3 Cross(const Vector3& a, const Vector3& b)     { /* ... */ }
```

ビルドして実行してください。

```
a + b   = (5.0, 7.0, 9.0)
Cross   = (-3.0, 6.0, -3.0)
Dot     = 32.0
|a|^2   = 14.0
```

4.1.3 と同じ出力です。モジュール化が完了しました。

### 4.3.4 クラスを `export` すると何が公開されるのか

整理しておきます。`export struct Vector3 { ... };` と書いたとき、公開されるのは次のものです。

- **型 `Vector3` そのもの**(`Vector3 v;` と書けるようになる)
- **`public` メンバ変数**(`v.x` と書けるようになる)
- **`public` メンバ関数**(`v.LengthSquared()` と呼べるようになる)
- **クラス定義の中で定義された演算子**(`v += w` が使えるようになる)

つまり、**クラスに `export` を1つ付ければ、その `public` インターフェイスは丸ごと使えるようになります。**

`operator+=` は `Vector3` のメンバとして定義したので、これで公開されました。一方 `operator+` は自由関数なので、別途 `export` が必要でした。同じ「演算子」でも扱いが違うのは、この差です。

### 4.3.5 メンバ単位の `export` はできない

「`LengthSquared` だけ隠したい」と思ったら、どう書けばよいでしょうか。

```cpp
export struct Vector3
{
    float x, y, z;

    export float LengthSquared() const;   // ← エラー
};
```

これは書けません。第3章 3.3.4 で述べたとおり、`export` は名前空間スコープの宣言にしか付けられません。クラスのメンバには付けられないのです。

クラスの中で見せる・見せないを制御するのは、これまでどおり `public` / `private` の仕事です。

```cpp
export struct Vector3
{
    float x, y, z;

    float LengthSquared() const;

private:
    void SomethingInternal();   // ← これは呼べない
};
```

**`export` と `private` は、別のレイヤーの仕組みです。**

| | 制御するもの | 単位 |
|---|---|---|
| `export` | モジュールの外から見えるか | 名前空間スコープの宣言 |
| `public` / `private` | クラスの外から使えるか | クラスのメンバ |

そして第1章 1.1.6 で指摘した `private` の限界 ── 「アクセスできないだけで、見えている」── はモジュールでも変わりません。`private` メンバはクラス定義の一部として `.ifc` に記録されます。実装を本当に隠したい場合の手段は、第11章で扱います。

### 4.3.6 現時点の全体像

いま、`GameMath` プロジェクトは次の構成です。

```
GameMath/
├── Vector3.ixx    ... module gamemath.vector
└── main.cpp       ... import gamemath.vector
```

ファイルは2つだけです。ヘッダ版なら `Vector3.h` と `main.cpp` の2つでしたが、`Vector3.cpp` が必要になる場面(定義を分けたいとき)を考えると、モジュール版のほうが少なくて済みます。

---

## 4.4 名前空間 `gamemath` を導入する

いまのところ、`Vector3` も `Dot` も `Cross` もグローバル名前空間にいます。ライブラリとしては望ましくありません。`Cross` のような一般的な名前が、利用者のコードと衝突するおそれがあるからです。

名前空間を導入します。

### 4.4.1 名前空間で囲む

`Vector3.ixx` の**モジュール宣言より後ろ**を、名前空間で囲みます。

```cpp
// Vector3.ixx
export module gamemath.vector;

namespace gamemath {

export struct Vector3
{
    // ...
};

export Vector3 operator+(const Vector3& a, const Vector3& b) { /* ... */ }
// ...(残りも同様)

} // namespace gamemath
```

モジュール宣言は名前空間の**外**、つまりファイルの先頭に置いたままです。モジュール宣言はファイルの最初の宣言でなければならないので(3.2.2)、名前空間で囲むことはできません。

### 4.4.2 利用側を直す

`main.cpp` で修飾が必要になります。

```cpp
// main.cpp
#include <cstdio>

import gamemath.vector;

int main()
{
    const gamemath::Vector3 a{ 1.0f, 2.0f, 3.0f };
    const gamemath::Vector3 b{ 4.0f, 5.0f, 6.0f };

    const gamemath::Vector3 sum = a + b;
    const gamemath::Vector3 crs = gamemath::Cross(a, b);

    std::printf("a + b   = (%.1f, %.1f, %.1f)\n", sum.x, sum.y, sum.z);
    std::printf("Cross   = (%.1f, %.1f, %.1f)\n", crs.x, crs.y, crs.z);
    std::printf("Dot     = %.1f\n", gamemath::Dot(a, b));
    std::printf("|a|^2   = %.1f\n", a.LengthSquared());

    return 0;
}
```

ビルドして、同じ出力になることを確認してください。

> **`a + b` に修飾が要らないのはなぜか**
>
> `gamemath::Cross(a, b)` は修飾が必要なのに、`a + b` はそのまま書けています。
>
> これは **ADL(実引数依存の名前探索)** が働いているためです。`a` と `b` の型が `gamemath::Vector3` なので、コンパイラは `gamemath` 名前空間の中も探しに行きます。
>
> 実は `Cross(a, b)` も、修飾なしで書けます(試してみてください)。本書では読みやすさのために明示的に修飾していますが、どちらでも動きます。
>
> ADL はモジュールと組み合わせると独特の落とし穴を生みます。第13章で正面から扱います。

### 4.4.3 `export namespace` という書き方

名前空間全体を公開する書き方もあります。

```cpp
export module gamemath.vector;

export namespace gamemath {

struct Vector3 { /* ... */ };

Vector3 operator+(const Vector3& a, const Vector3& b) { /* ... */ }
float Dot(const Vector3& a, const Vector3& b) { /* ... */ }
// ...

}
```

`namespace` に `export` を付けると、その中の宣言がすべて公開されます。個別に `export` を書く必要がなくなり、確かに短くなります。

第3章 3.3.3 で紹介した `export { }` ブロックと同じ発想です。

**本書ではこの書き方を使いません。** 理由も同じです。

- 1行を見ただけで公開・非公開が判断できない
- 非公開にしたいものが1つ出てきたとき、名前空間を分けるか、`export` の書き方を変えるかの判断を迫られる
- 「うっかり全部公開してしまう」事故が起きやすい

`GameMath` は数百行になり、内部ヘルパーも増えていきます。**公開は明示的な行為であるべき**というのが本書の立場です。

ただし、これは方針であって規則ではありません。小さなモジュールや、全部公開すると決まっているモジュールでは `export namespace` が読みやすいこともあります。

### 4.4.4 名前空間そのものは公開されるのか

細かい点ですが、疑問に思う人がいるので触れておきます。

「`namespace gamemath` に `export` を付けていないのに、利用側で `gamemath::Vector3` と書けるのはなぜか」

名前空間は、変数や関数のような**実体ではありません**。名前をまとめるための入れ物です。だから、名前空間そのものが「公開される・されない」という話にはなりません。

中に公開された宣言が1つでもあれば、利用側からその名前空間の名前が見えるようになります。逆に、中身が全部非公開なら、その名前空間は利用側から見て空っぽ ── 実質的に存在しないのと同じです。

---

## 4.5 モジュール名と名前空間は別物である

ここがこの章の核心です。

いま、私たちは `gamemath.vector` というモジュールの中に `gamemath` という名前空間を作りました。名前が似ているので、同じもののように見えます。

**まったく別のものです。** 実験で確かめます。

### 4.5.1 【実験1】モジュール名だけ変えてみる

`Vector3.ixx` のモジュール宣言を、意味のない名前に変えます。

```cpp
// Vector3.ixx
export module banana;   // ← 変更

namespace gamemath {
// ...(以下すべて変更なし)
}
```

`main.cpp` の `import` も合わせます。

```cpp
import banana;   // ← 変更
```

**それ以外は一切変更しません。** `gamemath::Vector3` も `gamemath::Cross` もそのままです。

ビルドして実行してください。**動きます。** 出力も同じです。

モジュール名を変えても、名前空間には何の影響もありませんでした。変更が必要だったのは、`export module` の行と `import` の行だけです。

**元に戻してください**(`gamemath.vector`)。

### 4.5.2 【実験2】名前空間だけ変えてみる

今度は逆です。モジュール名は `gamemath.vector` のまま、名前空間だけ変えます。

```cpp
// Vector3.ixx
export module gamemath.vector;   // 変更なし

namespace physics {              // ← 変更
// ...
} // namespace physics
```

`main.cpp` では、`import` 行は変えず、修飾だけ変えます。

```cpp
import gamemath.vector;   // 変更なし

int main()
{
    const physics::Vector3 a{ 1.0f, 2.0f, 3.0f };   // ← 変更
    // ...
}
```

ビルドして実行してください。**動きます。**

`gamemath.vector` というモジュールが、`physics` という名前空間の中身を提供しています。奇妙に見えますが、C++ としては何も間違っていません。

**元に戻してください**(`gamemath`)。

### 4.5.3 【実験3】`import` は `using namespace` ではない

もう1つ、勘違いしやすい点があります。

```cpp
import gamemath.vector;

int main()
{
    const Vector3 a{ 1.0f, 2.0f, 3.0f };   // 修飾なし
    // ...
}
```

これはエラーになります。

```
error C2065: 'Vector3': 定義されていない識別子です。
```

`import` は、モジュールが公開している名前を**そのままの形で**持ち込みます。`gamemath` 名前空間の中にある型は、`gamemath::Vector3` として持ち込まれます。名前空間が剥がれることはありません。

`#include` と同じです。ヘッダを `#include` しても `using namespace` にはならないのと同じことです。

つまり、`import` した側で修飾を省きたければ、これまでどおり自分で書きます。

```cpp
import gamemath.vector;

using namespace gamemath;   // 自分で書く

int main()
{
    const Vector3 a{ 1.0f, 2.0f, 3.0f };   // 通る
}
```

**確認できたら、`using namespace` は消して、修飾する形に戻してください。** ライブラリの利用例として、修飾ありのほうが適切です。

### 4.5.4 2つの違いを整理する

| | モジュール名 | 名前空間 |
|---|---|---|
| 何を決めるか | **どのファイルから読み込むか** | **名前をどう書くか** |
| 書く場所 | ファイルの先頭に1回 | コードを囲む |
| 利用側の書き方 | `import gamemath.vector;` | `gamemath::Vector3` |
| ドットの意味 | 名前の一部(階層ではない) | ── |
| 区切り文字 | `.` | `::` |
| 入れ子 | できない(平坦な名前) | できる |
| 実行時の存在 | なし(ビルド時だけの概念) | なし(名前の話だけ) |
| 決めるのは | **ビルドの単位** | **名前の衝突回避** |

ひとことで言えば、こうです。

- **モジュール名**は、コンパイラに「どの `.ifc` を読めばよいか」を教えるための名前
- **名前空間**は、人間とコンパイラに「この名前はどのグループのものか」を教えるための名前

解決している問題がまったく違います。

### 4.5.5 1つの名前空間に、複数のモジュールが入れる

この独立性は、実は非常に重要です。

`GameMath` はこれから、`Vector3` だけでなく `Matrix4x4` や `Quaternion` も持つようになります。すべて `gamemath` 名前空間に置きたい。しかしモジュールは分けたい ── `Vector3` しか使わない人に `Quaternion` の `.ifc` まで読ませたくないからです。

モジュールと名前空間が独立だからこそ、これができます。

```cpp
// Vector3.ixx
export module gamemath.vector;
namespace gamemath { export struct Vector3 { /* ... */ }; }
```

```cpp
// Matrix4x4.ixx
export module gamemath.matrix;
namespace gamemath { export struct Matrix4x4 { /* ... */ }; }
```

利用側はこう書きます。

```cpp
import gamemath.vector;
import gamemath.matrix;

gamemath::Vector3 v;
gamemath::Matrix4x4 m;
```

**名前空間は共通、モジュールは別。** ライブラリの利用者から見れば `gamemath` という1つのまとまりですが、ビルドの単位としては分かれています。

この構造をどう設計するかが、第9章から第12章のテーマです。第13章で `Matrix4x4` を実際に追加するときには、この形になります。

### 4.5.6 命名の慣習

「別物なら、なぜ `gamemath.vector` と `gamemath` という似た名前にするのか」

**人間が混乱しないためです。**

慣習として、モジュール名の先頭要素とトップレベルの名前空間名を揃えることが多く行われています。`gamemath.vector` を `import` すれば `gamemath::` 何かが手に入る、と予想できるからです。

これは規約であって、言語の要求ではありません。しかし守る価値のある規約です。実験1と実験2で見たとおり、揃えないこともできますが、揃えないコードは読み手に余計な負荷をかけます。

本書では次の対応で統一します。

| モジュール名 | 名前空間 | 内容 |
|---|---|---|
| `gamemath.vector` | `gamemath` | ベクトル |
| `gamemath.matrix` | `gamemath` | 行列 |
| `gamemath.quaternion` | `gamemath` | クォータニオン |
| `gamemath.geometry` | `gamemath` | 幾何プリミティブ |

名前空間はすべて `gamemath` で共通です。モジュールだけが分かれます。

---

## 4.6 なぜ `Length()` を書かなかったのか

この章で作った `Vector3` には、`LengthSquared()` はあっても `Length()` がありません。ベクトルライブラリとしては明らかに不自然です。

理由は、`Length()` に平方根が必要だからです。

```cpp
float Length() const
{
    return std::sqrt(x * x + y * y + z * z);   // <cmath> が要る
}
```

`std::sqrt` を使うには `<cmath>` が必要です。そして ── **`.ixx` の中に `#include <cmath>` と書くと、うまくいきません。**

第3章 3.2.2 で「モジュール宣言はファイルの最初の宣言でなければならない」と述べました。では `#include` はモジュール宣言の前に書けるのか、後に書けるのか。後に書いた場合、取り込まれた `<cmath>` の中身はどう扱われるのか。

これは1つの節では収まらない話題で、**第6章がまるごとこのテーマ**です。グローバルモジュールフラグメントという新しい構文と、`import std;` という選択肢の両方を扱います。

そこで本書では、第6章まで**標準ライブラリを一切使わない**という制約を自分に課しています。`LengthSquared()` だけで我慢しているのはそのためです。

不便ですが、悪いことばかりでもありません。ゲームの内側のループでは、そもそも平方根を避けて `LengthSquared()` で比較するのが定石です。「どちらのベクトルが長いか」を知りたいだけなら、平方根は要りません。

`Length()` と `Normalized()` は、第6章で追加します。

---

## 4.7 この章のまとめ

- ヘッダを `.ixx` に移すときに直すのは3か所 ── インクルードガードの削除、モジュール宣言の追加、`inline` の削除
- **クラスを `export` しても、そのクラスを引数に取る自由関数は `export` されない。** 演算子や `Dot` / `Cross` には個別に `export` が必要
- クラスに `export` を付けると、その `public` メンバは丸ごと使えるようになる
- `export` はメンバに個別に付けられない。クラス内の制御は `public` / `private` の仕事
- `export` と `private` は別レイヤーの仕組み
- モジュール宣言は名前空間の外、ファイルの先頭に置く
- `export namespace` で全体を公開できるが、本書では個別 `export` を方針とする
- **モジュール名と名前空間は完全に独立している。** 片方を変えてももう片方には影響しない
- `import` は `using namespace` ではない。名前空間の修飾は必要
- 1つの名前空間に、複数のモジュールが宣言を提供できる。`GameMath` はこの構造を採る
- 名前を揃えるのは人間のための慣習であって、言語の要求ではない

## 次章に向けて

`Vector3.ixx` は、まだ100行に届かない程度です。しかしこれから `Matrix4x4` との相互変換、比較演算子、補間関数と増えていけば、あっという間に膨れます。

インターフェイスの定義と実装が同じファイルに同居していると、次の問題が起きます。

- インターフェイスを読みたいだけの人が、実装まで読まされる
- 実装を1行変えただけで、`.ifc` が作り直され、`import` している側が全部再コンパイルされる(第2章 2.4.3 で体験しました)

第5章では、**モジュール実装単位**という仕組みを導入して、宣言と定義を別のファイルに分けます。ヘッダと `.cpp` に分けるのと似ていますが、いくつか重要な違いがあります。特に「実装単位からはインターフェイスの中身が全部見える」という性質は、ヘッダの世界にはなかったものです。
