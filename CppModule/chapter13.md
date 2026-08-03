# 第13章 行列 Matrix4x4

## この章について

第4部に入ります。構成は第12章で決まったので、ここからは中身を埋めていく作業です。

この章の主題は `Matrix4x4` ですが、**行列そのものより重要な話**が途中にあります。**演算子オーバーロードと ADL(実引数依存の名前探索)**です。

第4章 4.3.2 で、こういう落とし穴を扱いました。

> クラスを `export` しても、そのクラスを引数に取る自由関数は `export` されない。

行列を追加すると、演算子の数が一気に増えます。そして `export` を1つ書き忘れるだけで、利用者は意味不明なエラーに遭遇します。

その対策として、**隠しフレンド**(hidden friend)という技法を導入します。演算子をクラスの中に書くことで、`export` が不要になります。ただし**使ってはいけない場面**もあり、その線引きがこの章のいちばん実用的な部分です。

内容は5つです。

1. `gamemath.core:matrix` パーティションを追加する(13.1)
2. `:vector` への依存を書く(13.2)
3. 演算子と ADL の落とし穴(13.3)
4. 隠しフレンドで解決する(13.4)
5. 変換行列を実装する(13.5)

---

## 13.1 `gamemath.core:matrix` パーティションを追加する

### 13.1.1 置き場所は決まっている

第12章 12.1.3 で、こう整理しました。

> `Matrix4x4` と `Quaternion` は、`Vector` と同じ層です。互いを必要とし、常に一緒に使われます。

指針1(一緒に変更されるものは同じモジュールに)と指針2(一緒に使われるものは同じモジュールに)により、**`gamemath.core` のパーティション**にします。サブモジュールにはしません。

設計を先に済ませておくと、こういう判断で迷いません。

### 13.1.2 設計方針を決める

行列には、実装以前に決めておくべきことが2つあります。ここを曖昧にしたまま書き始めると、あとで全部書き直すことになります。

**決定1: 格納順序 ── 行優先(row-major)**

`m[i][j]` の `i` を行、`j` を列とします。メモリ上では1行ずつ並びます。

**決定2: ベクトルの規約 ── 列ベクトル(`M * v`)**

数学の教科書と同じ規約です。ベクトルを縦に置き、行列を左から掛けます。

```
┌ m00 m01 m02 m03 ┐   ┌ x ┐
│ m10 m11 m12 m13 │ × │ y │
│ m20 m21 m22 m23 │   │ z │
└ m30 m31 m32 m33 ┘   └ 1 ┘
```

この規約では、**平行移動成分は最右列**(`m[0][3]`、`m[1][3]`、`m[2][3]`)に入ります。

そして変換の合成は、**右から左に適用**されます。`A * B * v` は「まず `B` を適用し、次に `A` を適用する」という意味です。

> **もう1つの規約について**
>
> DirectX の伝統的な API は、行ベクトル規約(`v * M`)を採っています。この場合、平行移動成分は最下行に入り、合成は左から右に適用されます。
>
> どちらが正しいというものではありません。**大事なのは、1つのライブラリの中で混ぜないこと**です。混ざったコードのデバッグは、経験上いちばん報われない作業になります。
>
> 本書は数学の慣習に合わせて列ベクトル規約を採ります。ドキュメントとコメントに明記してください。第12章 12.4.4 で「文書化すべきこと」を挙げましたが、**規約は API の一部**です。

### 13.1.3 パーティションを作る

`Core/Matrix.ixx` を作ってください。

```cpp
// Core/Matrix.ixx
export module gamemath.core:matrix;

import :vector;
import std;

namespace gamemath {

// 行優先で格納する。列ベクトル規約(M * v)。
// 平行移動成分は最右列 m[0][3], m[1][3], m[2][3]。
export struct Matrix4x4
{
    float m[4][4];

    static constexpr Matrix4x4 Identity()
    {
        Matrix4x4 r{};
        for (std::size_t i = 0; i < 4; ++i) {
            r.m[i][i] = 1.0f;
        }
        return r;
    }

    constexpr float& operator()(std::size_t row, std::size_t col)
    {
        return m[row][col];
    }

    constexpr float operator()(std::size_t row, std::size_t col) const
    {
        return m[row][col];
    }
};

export constexpr Matrix4x4 operator*(const Matrix4x4& a, const Matrix4x4& b)
{
    Matrix4x4 r{};
    for (std::size_t i = 0; i < 4; ++i) {
        for (std::size_t j = 0; j < 4; ++j) {
            float sum = 0.0f;
            for (std::size_t k = 0; k < 4; ++k) {
                sum += a.m[i][k] * b.m[k][j];
            }
            r.m[i][j] = sum;
        }
    }
    return r;
}

export constexpr Matrix4x4 operator*(const Matrix4x4& a, float s)
{
    Matrix4x4 r{};
    for (std::size_t i = 0; i < 4; ++i) {
        for (std::size_t j = 0; j < 4; ++j) {
            r.m[i][j] = a.m[i][j] * s;
        }
    }
    return r;
}

export constexpr bool operator==(const Matrix4x4& a, const Matrix4x4& b)
{
    for (std::size_t i = 0; i < 4; ++i) {
        for (std::size_t j = 0; j < 4; ++j) {
            if (a.m[i][j] != b.m[i][j]) {
                return false;
            }
        }
    }
    return true;
}

export constexpr Matrix4x4 Transpose(const Matrix4x4& a)
{
    Matrix4x4 r{};
    for (std::size_t i = 0; i < 4; ++i) {
        for (std::size_t j = 0; j < 4; ++j) {
            r.m[i][j] = a.m[j][i];
        }
    }
    return r;
}

} // namespace gamemath
```

いまは演算子を**自由関数**として書き、それぞれに `export` を付けています。13.4 でこれを書き換えます。

`Identity()` は `static` メンバ関数にしました。第4章 4.3.4 で確認したとおり、**クラスを `export` すれば `public` メンバは丸ごと使えるようになります**。個別の `export` は不要です。

`operator()` を添字アクセスに使っています。`operator[]` は C++23 から多次元引数を取れますが、`operator()` のほうが行列の記法に近く、広く使われています。

### 13.1.4 プライマリにつなぐ

`Core/Core.ixx` に1行足します。

```cpp
// Core/Core.ixx
export module gamemath.core;

export import :vector;
export import :basis;
export import :matrix;      // ← 追加
```

第9章 9.2.4 の規則です。インターフェイスパーティションは、プライマリから `import` されていなければなりません。そして利用者に届けるために `export` を付けます。

### 13.1.5 動作確認

`main.cpp` に追加します。

```cpp
// main.cpp
constexpr gamemath::Matrix4x4 id = gamemath::Matrix4x4::Identity();

static_assert(id(0, 0) == 1.0f);
static_assert(id(0, 1) == 0.0f);
static_assert(id * id == id);
static_assert(gamemath::Transpose(id) == id);

std::println("Identity * Identity == Identity : {}", id * id == id);
```

ビルドして実行してください。

```
Identity * Identity == Identity : true
```

`static_assert` が通っていることに注目してください。行列の積が**コンパイル時に**計算されています。第8章 8.5 で確認した性質です。単位行列の積のような、コンパイル時に確定するものは確定させておきます。

---

## 13.2 `:vector` パーティションへの依存を書く

### 13.2.1 依存を確認する

`Matrix.ixx` の2行目です。

```cpp
import :vector;
```

いまの `Matrix4x4` はまだ `Vector3` を使っていません。しかし 13.5 で変換行列を書くときに必要になるので、先に書いておきます。

パーティション名だけで `import` できるのは、同じモジュールの中だからです(第9章 9.3.2)。第12章 12.2.2 でモジュール名を `gamemath` から `gamemath.core` に変えたときも、この行は変更不要でした。

### 13.2.2 依存の向きを確認する

第10章 10.4.4 の依存グラフを更新します。

```
gamemath.core (プライマリ)
   export import :vector;
   export import :basis;
   export import :matrix;
        ↓                ↓
   ┌────────┐      ┌──────────┐
   │ :basis │      │ :matrix  │
   └────────┘      └──────────┘
        ↓                ↓
   ┌────────┐            │
   │ :detail│            │
   └────────┘            │
        ↓                │
   ┌──────────────────────┴──┐
   │        :vector          │  ← 土台。何も import しない
   └─────────────────────────┘
```

`:matrix` と `:basis` は、どちらも `:vector` に依存し、互いには依存しません。**同じ層に並ぶ、独立した2つのパーティション**です。

これが健全な形です。第10章 10.4.3 で述べた「依存は一方向」が守られています。

### 13.2.3 なぜ `:vector` を `:matrix` に依存させないのか

逆向きにしたくなる場面があります。たとえば `Vector3` に、こういうメンバを足したくなります。

```cpp
// やってはいけない例
struct Vector3
{
    Vector3 Transformed(const Matrix4x4& m) const;   // Matrix4x4 が必要
};
```

書けません。`:vector` が `:matrix` を `import` することになり、`:matrix` は `:vector` を `import` しているので**循環します**(第10章 10.4.1)。

対処は第10章 10.4.5 のとおりで、この場合は**対処2(依存の向きを1つに決める)**を選びます。変換関数を `:matrix` 側に置きます。

```cpp
// :matrix に置く
export constexpr Vector3 TransformPoint(const Matrix4x4& m, const Vector3& p);
```

「`v.Transformed(m)` と書きたい」という気持ちは分かりますが、`TransformPoint(m, v)` で我慢します。**ベクトルは行列を知らない**という関係を保つほうが、ライブラリとして健全です。

そして「より基本的なほうを下に置く」という目安(第10章 10.4.5)にも合っています。行列がなくてもベクトルは成立しますが、逆は成立しません。

---

## 13.3 演算子オーバーロードと ADL

### 13.3.1 いまの書き方の問題

13.1.3 では、演算子を自由関数として書き、それぞれに `export` を付けました。

```cpp
export constexpr Matrix4x4 operator*(const Matrix4x4& a, const Matrix4x4& b) { /* ... */ }
export constexpr Matrix4x4 operator*(const Matrix4x4& a, float s)            { /* ... */ }
export constexpr bool      operator==(const Matrix4x4& a, const Matrix4x4& b){ /* ... */ }
```

動きます。しかし、これから増えます。`operator+`、`operator-`、`operator*=`、`operator!=`、`Vector` との組み合わせ ── 数学ライブラリの演算子は10個や20個になります。

そのすべてに `export` を付け忘れないでいられるでしょうか。

### 13.3.2 【実験】1つだけ `export` を忘れてみる

`operator*(const Matrix4x4&, float)` から `export` を外してください。

```cpp
constexpr Matrix4x4 operator*(const Matrix4x4& a, float s)   // ← export を削除
{
    // ...
}
```

`main.cpp` で使ってみます。

```cpp
const gamemath::Matrix4x4 doubled = id * 2.0f;
```

ビルドしてください。エラーになります。

```
error C2676: 二項演算子 '*': 'const gamemath::Matrix4x4' は、この演算子または
             定義済の演算子に適切な型への変換の定義を行いません。
```

第4章 4.3.1 で見たのと同じエラーです。

問題は、**このエラーが原因を教えてくれないこと**です。「`export` が付いていません」とは言いません。「そんな演算子はない」と言うだけです。

しかも、エラーが出るのは**利用者のコード**です。ライブラリの作者は気づきません。テストコードで `Matrix4x4 * float` を使っていなければ、リリースまで発覚しない可能性があります。

**`export` を戻してください。**

### 13.3.3 規則 ── 取り込む側の宣言は、モジュールの中の探索に参加しない

ここで、モジュールと名前探索についての重要な規則を確認します。Microsoft のドキュメントは、こう述べています。

> 取り込む側の翻訳単位にある宣言は、取り込まれたモジュールの中でのオーバーロード解決や名前探索に参加しない。

ヘッダの世界と決定的に違う点です。

ヘッダなら、こういうことができました。

```cpp
#include "library.h"       // ライブラリのテンプレートが入っている

// あとから自分の型用のオーバーロードを足す
void Print(MyType x) { /* ... */ }

int main()
{
    LibraryFunction(MyType{});   // ライブラリのテンプレートが Print(MyType) を見つける
}
```

テンプレートの実体化はテキストが展開された**後**に行われるので、あとから足した宣言も探索の対象になりました。ADL による**カスタマイズ**という設計手法が、これで成立していました。

モジュールでは成立しません。`library` モジュールは、それがコンパイルされた時点で見えていたものしか知りません。利用者があとから足した宣言は届きません。

第7章 7.2.3 で、`main.cpp` に `kEpsilon` を定義しても `gamemath` の中の `kEpsilon` と衝突しなかったことを確認しました。あれと同じ壁が、探索についても立っているわけです。

### 13.3.4 「利用者が拡張する」設計は成立しない

この規則から、設計上の結論が出ます。

**`GameMath` は、「利用者が自分の型用のオーバーロードを足せば動く」という形の拡張性を提供できません。**

たとえば、こういう設計は成立しません。

```cpp
// :vector に、こう書いたとして
export template <typename T>
T LengthOf(const T& v)
{
    return Length(v);      // ADL で利用者の Length を見つけてほしい
}
```

利用者が自分の `Length(MyVector)` を書いても、`LengthOf` からは見えません。

代わりに使うのが**コンセプト**です。第8章 8.3 で `FloatingPoint` を定義したときの手法です。

```cpp
export template <typename T>
concept HasLength = requires(const T& v) { { v.Length() } -> std::convertible_to<float>; };

export template <HasLength T>
float LengthOf(const T& v)
{
    return v.Length();     // メンバ関数なら ADL に頼らない
}
```

メンバ関数の呼び出しは、名前空間の探索を経由しません。オブジェクトの型から直接引かれるので、モジュール境界の影響を受けません。

**モジュール時代のカスタマイズは、ADL ではなくコンセプトとメンバ関数で行う。** これは第16章で SIMD の切り替えを設計するときに、もう一度出てきます。第7章 7.1.3 で「マクロによる設定の切り替えは成立しない」と書いたのと同じ構図です。**モジュールは境界を固くする代わりに、境界をまたぐ仕掛けを封じます。**

> **コラム: この領域はまだ荒れています**
>
> ADL とモジュールの組み合わせは、コンパイラの実装が最も追いついていない領域の1つです。「規格上は動くはずだがコンパイラが受け付けない」「あるコンパイラでは動くが別のコンパイラでは動かない」という報告が、いまも各コンパイラのバグトラッカーに上がっています。
>
> とくに、テンプレートの中で依存名の演算子を使う場合と、`export` していない型の隠しフレンドが関わる場合に問題が出やすいようです。
>
> 対策は保守的に書くことです。**演算子は素直に、公開する型に対して、隠しフレンドまたは `export` 付きの自由関数として書く。** 凝った仕掛けは避ける。付録E に、コンパイラごとの差異をまとめています。

### 13.3.5 数学ライブラリにとっての意味

`GameMath` に必要なのは、凝った拡張性ではありません。**`Matrix4x4` や `Vector3` という具体的な型に対する演算子が、確実に見つかること**です。

そして問題は「`export` を書き忘れる」という、人間の不注意でした。

これを構造的に防ぐ方法があります。

---

## 13.4 隠しフレンド(hidden friend)を使う

### 13.4.1 隠しフレンドとは

**クラス定義の中だけで宣言された `friend` 関数**を、隠しフレンドと呼びます。

```cpp
struct Matrix4x4
{
    float m[4][4];

    friend constexpr Matrix4x4 operator*(const Matrix4x4& a, const Matrix4x4& b)
    {
        // ...
    }
};
```

`friend` なのでメンバ関数ではありません。自由関数です。しかし、クラスの外に宣言がないので、**通常の名前探索では見つかりません**。

見つかるのは **ADL 経由だけ**です。つまり、引数の型が `Matrix4x4` である呼び出しでしか見つかりません。

これが「隠れている」という表現の意味です。

### 13.4.2 書き換える

`Matrix.ixx` の3つの演算子を、クラスの中に移します。

```cpp
// Core/Matrix.ixx
export module gamemath.core:matrix;

import :vector;
import std;

namespace gamemath {

export struct Matrix4x4
{
    float m[4][4];

    static constexpr Matrix4x4 Identity()
    {
        Matrix4x4 r{};
        for (std::size_t i = 0; i < 4; ++i) {
            r.m[i][i] = 1.0f;
        }
        return r;
    }

    constexpr float& operator()(std::size_t row, std::size_t col)
    {
        return m[row][col];
    }

    constexpr float operator()(std::size_t row, std::size_t col) const
    {
        return m[row][col];
    }

    // ───── 以下、隠しフレンド。export は不要 ─────

    friend constexpr Matrix4x4 operator*(const Matrix4x4& a, const Matrix4x4& b)
    {
        Matrix4x4 r{};
        for (std::size_t i = 0; i < 4; ++i) {
            for (std::size_t j = 0; j < 4; ++j) {
                float sum = 0.0f;
                for (std::size_t k = 0; k < 4; ++k) {
                    sum += a.m[i][k] * b.m[k][j];
                }
                r.m[i][j] = sum;
            }
        }
        return r;
    }

    friend constexpr Matrix4x4 operator*(const Matrix4x4& a, float s)
    {
        Matrix4x4 r{};
        for (std::size_t i = 0; i < 4; ++i) {
            for (std::size_t j = 0; j < 4; ++j) {
                r.m[i][j] = a.m[i][j] * s;
            }
        }
        return r;
    }

    friend constexpr bool operator==(const Matrix4x4& a, const Matrix4x4& b)
    {
        for (std::size_t i = 0; i < 4; ++i) {
            for (std::size_t j = 0; j < 4; ++j) {
                if (a.m[i][j] != b.m[i][j]) {
                    return false;
                }
            }
        }
        return true;
    }
};

export constexpr Matrix4x4 Transpose(const Matrix4x4& a)
{
    Matrix4x4 r{};
    for (std::size_t i = 0; i < 4; ++i) {
        for (std::size_t j = 0; j < 4; ++j) {
            r.m[i][j] = a.m[j][i];
        }
    }
    return r;
}

} // namespace gamemath
```

ビルドして実行してください。13.1.5 と同じ結果になります。`static_assert` も通ります。

### 13.4.3 `export` が不要になった

**3つの演算子に `export` が付いていません。**

必要ないのです。`Matrix4x4` を `export` した時点で、その隠しフレンドも一緒に使えるようになります。

理由は2つ組み合わさっています。

第一に、隠しフレンドは**クラスの一部として宣言されている**ので、クラスが公開されればそれも届きます。

第二に、隠しフレンドは ADL でしか見つかりません。そして ADL は、**引数の型に関連するクラスの `friend` 宣言を、通常の探索で見えなくても見つけます**。これは C++ の古くからある規則で、モジュールとは無関係に成り立っています。

結果として、**「`export` を書き忘れる」という事故が構造的に起こらなくなりました。** 演算子を1つ追加するとき、`export` について考える必要がありません。

第4章 4.3.2 で扱った落とし穴 ── 「クラスを `export` しても自由関数は `export` されない」── に対する、いちばんきれいな答えがこれです。

### 13.4.4 隠しフレンドの3つの利点

`export` 不要は、モジュール特有の利点です。それ以外にも従来から知られた利点があります。

**利点1: オーバーロード解決の候補が減る**

名前空間スコープに `operator==` が100個あると、`==` を使うたびにコンパイラは100個を候補として検討します。隠しフレンドなら、引数の型が一致しないかぎり候補に入りません。

大規模なコードベースでは、これがコンパイル時間に効きます。

**利点2: 暗黙変換で呼ばれなくなる**

これが実は重要です。自由関数として書いた場合、暗黙変換によって意図しない呼び出しが成立してしまうことがあります。

```cpp
// 自由関数だった場合
struct Meters { float v; operator float() const { return v; } };

Matrix4x4 m = id * Meters{ 2.0f };   // float に暗黙変換されて通る
```

隠しフレンドは、**引数のどちらかが `Matrix4x4` そのものである場合にしか候補になりません**。関係のない型からの暗黙変換で呼ばれることが減ります。

**利点3: `const` の付け忘れが減る**

両方の引数を明示的に書くので、片方だけ `const` を忘れるといった非対称が起きにくくなります。

### 13.4.5 【重要】限界 ── 修飾して呼べない

ここが、この章でいちばん実務的に重要な点です。

**隠しフレンドは、修飾名では呼べません。**

```cpp
gamemath::operator*(a, b);      // ← 見つからない
```

修飾名による呼び出しは、指定した名前空間の中だけを見ます。ADL を行いません。そして隠しフレンドは ADL でしか見つからないので、修飾すると消えます。

演算子については、これは問題になりません。`a * b` と書くのが普通で、`gamemath::operator*(a, b)` と書く人はいません。

**しかし、名前付き関数では致命的です。**

いま `main.cpp` には、こういうコードがあります。

```cpp
gamemath::Dot(a, b);
gamemath::Cross(a, b);
gamemath::Transpose(id);
```

これらを隠しフレンドにしてしまうと、**すべて書けなくなります。**

```cpp
gamemath::Dot(a, b);     // ← 隠しフレンドにしたら、エラーになる
Dot(a, b);               // ← こう書くしかない
```

利用者に「修飾せずに書いてください」と強制することになります。`using namespace gamemath;` を書くか、修飾なしで書くか ── どちらも押しつけです。

第4章 4.4.2 のコラムで、ADL のおかげで `Cross(a, b)` と修飾なしでも書けることに触れました。あれは「書ける」という話でした。隠しフレンドにすると「そうしか書けない」に変わります。

### 13.4.6 使い分けの基準

以上から、`GameMath` の方針が決まります。

| 種類 | 書き方 | 理由 |
|---|---|---|
| **演算子**(`*` `+` `==` `<=>` など) | **隠しフレンド** | 修飾して呼ばれない。`export` 忘れを防げる |
| **名前付き関数**(`Dot` `Cross` `Transpose` `Translation` など) | **`export` 付きの自由関数** | 利用者が修飾して呼べる必要がある |
| **メンバ関数**(`Length` `Normalized` `Identity` など) | クラスの中に普通に書く | クラスの `export` で届く |

覚え方はこうです。

> **記号で呼ぶものは隠しフレンド。名前で呼ぶものは `export` 付きの自由関数。**

`Vector.ixx` の演算子も、同じ方針で書き換えられます。ただし `Vector<T, N>` はテンプレートなので、隠しフレンドの書き方が少し変わります。

```cpp
export template <FloatingPoint T, std::size_t N>
struct Vector
{
    T e[N];

    // ...

    friend constexpr Vector operator+(const Vector& a, const Vector& b)
    {
        Vector r{};
        for (std::size_t i = 0; i < N; ++i) {
            r.e[i] = a.e[i] + b.e[i];
        }
        return r;
    }

    friend constexpr Vector operator-(const Vector& a, const Vector& b)
    {
        Vector r{};
        for (std::size_t i = 0; i < N; ++i) {
            r.e[i] = a.e[i] - b.e[i];
        }
        return r;
    }

    friend constexpr Vector operator*(const Vector& v, T s)
    {
        Vector r{};
        for (std::size_t i = 0; i < N; ++i) {
            r.e[i] = v.e[i] * s;
        }
        return r;
    }
};
```

クラステンプレートの中では、`Vector` と書くだけで `Vector<T, N>` を意味します(注入クラス名)。テンプレート引数を書き直す必要がなく、かえって読みやすくなります。

`Dot` と `Cross` は自由関数のまま、`export` を付けて残します。`gamemath::Dot(a, b)` と書けるようにするためです。

`Vector.ixx` を書き換えて、ビルドが通ることを確認してください。`main.cpp` は変更不要です。

### 13.4.7 MSVC での注意

Microsoft のドキュメントに、こういう記述があります。

> C++20 モジュールを使うには、標準の隠しフレンドのふるまいが必要である。

MSVC には歴史的な事情があり、`/permissive-`(標準準拠モード)でないと隠しフレンドの扱いが標準と違っていました。旧来のふるまいでは、隠しフレンドを本来より広い文脈で候補に含めてしまい、コンパイルが遅くなります。

`/permissive-` は Visual Studio の新規 C++ プロジェクトでは既定で有効なので、通常は何もしなくて構いません。念のため確認するなら、**[C/C++] → [言語] → [準拠モード]** が「はい (/permissive-)」になっているかを見てください。

`/Zc:hiddenFriend-` で旧来のふるまいに戻すオプションもありますが、モジュールを使うなら指定してはいけません。

---

## 13.5 変換行列(平行移動・回転・スケール)を実装する

道具が揃ったので、実用的な行列を作ります。

### 13.5.1 平行移動とスケール

`Matrix.ixx` の `Transpose` の後ろに追加します。

```cpp
// Core/Matrix.ixx

export constexpr Matrix4x4 Translation(const Vector3& t)
{
    Matrix4x4 r = Matrix4x4::Identity();
    r.m[0][3] = t.x();
    r.m[1][3] = t.y();
    r.m[2][3] = t.z();
    return r;
}

export constexpr Matrix4x4 Scale(const Vector3& s)
{
    Matrix4x4 r{};
    r.m[0][0] = s.x();
    r.m[1][1] = s.y();
    r.m[2][2] = s.z();
    r.m[3][3] = 1.0f;
    return r;
}
```

平行移動成分が最右列に入っています。13.1.2 で決めた列ベクトル規約のとおりです。

どちらも `constexpr` です。単位行列や固定のスケールは、コンパイル時に確定できます。

### 13.5.2 軸ごとの回転

```cpp
export Matrix4x4 RotationX(float radians)
{
    const float c = std::cos(radians);
    const float s = std::sin(radians);

    Matrix4x4 r = Matrix4x4::Identity();
    r.m[1][1] =  c;  r.m[1][2] = -s;
    r.m[2][1] =  s;  r.m[2][2] =  c;
    return r;
}

export Matrix4x4 RotationY(float radians)
{
    const float c = std::cos(radians);
    const float s = std::sin(radians);

    Matrix4x4 r = Matrix4x4::Identity();
    r.m[0][0] =  c;  r.m[0][2] =  s;
    r.m[2][0] = -s;  r.m[2][2] =  c;
    return r;
}

export Matrix4x4 RotationZ(float radians)
{
    const float c = std::cos(radians);
    const float s = std::sin(radians);

    Matrix4x4 r = Matrix4x4::Identity();
    r.m[0][0] =  c;  r.m[0][1] = -s;
    r.m[1][0] =  s;  r.m[1][1] =  c;
    return r;
}
```

**`constexpr` が付いていません。** `std::cos` と `std::sin` が `constexpr` ではないからです。第8章 8.5.3 で `Length()` について説明したのと同じ事情です。

定義はインターフェイスに残します。短く、毎フレーム呼ばれる可能性があるので、インライン展開してほしいからです(第5章 5.6.5 の基準)。

### 13.5.3 任意軸回転 ── 実装単位へ

任意の軸まわりの回転は、ロドリゲスの回転公式を使います。行数が多く、呼び出し頻度は低い ── 第5章 5.6.5 の基準で**実装単位**に置く関数です。

宣言をインターフェイスに書きます。

```cpp
// Core/Matrix.ixx
export Matrix4x4 RotationAxis(const Vector3& axis, float radians);
```

`Core/Matrix.cpp` を新しく作ります。

```cpp
// Core/Matrix.cpp
module gamemath.core;

import std;

namespace gamemath {

Matrix4x4 RotationAxis(const Vector3& axis, float radians)
{
    const Vector3 n = axis.Normalized();
    const float x = n.x();
    const float y = n.y();
    const float z = n.z();

    const float c  = std::cos(radians);
    const float s  = std::sin(radians);
    const float t  = 1.0f - c;

    Matrix4x4 r{};
    r.m[0][0] = t * x * x + c;
    r.m[0][1] = t * x * y - s * z;
    r.m[0][2] = t * x * z + s * y;

    r.m[1][0] = t * x * y + s * z;
    r.m[1][1] = t * y * y + c;
    r.m[1][2] = t * y * z - s * x;

    r.m[2][0] = t * x * z - s * y;
    r.m[2][1] = t * y * z + s * x;
    r.m[2][2] = t * z * z + c;

    r.m[3][3] = 1.0f;
    return r;
}

} // namespace gamemath
```

`module gamemath.core;` と書いています。これで **`gamemath.core` の2つ目の実装単位**になりました(1つ目は `Basis.cpp`)。第10章 10.3.1 の表で確認したとおり、実装単位はいくつあっても構いません。

`import :matrix;` を書いていないことに注意してください。実装単位はプライマリを暗黙に `import` し、プライマリが `:matrix` を `export import` しているので、届いています(第9章 9.4.4)。

### 13.5.4 点と方向の変換 ── なぜ `operator*` にしないのか

ベクトルを行列で変換する関数を追加します。ここには設計上の判断が必要です。

```cpp
// Core/Matrix.ixx

// 位置を変換する(同次座標の w = 1 として扱う)
export constexpr Vector3 TransformPoint(const Matrix4x4& a, const Vector3& p)
{
    return Vector3{
        a.m[0][0] * p.x() + a.m[0][1] * p.y() + a.m[0][2] * p.z() + a.m[0][3],
        a.m[1][0] * p.x() + a.m[1][1] * p.y() + a.m[1][2] * p.z() + a.m[1][3],
        a.m[2][0] * p.x() + a.m[2][1] * p.y() + a.m[2][2] * p.z() + a.m[2][3]
    };
}

// 方向を変換する(w = 0。平行移動の影響を受けない)
export constexpr Vector3 TransformDirection(const Matrix4x4& a, const Vector3& d)
{
    return Vector3{
        a.m[0][0] * d.x() + a.m[0][1] * d.y() + a.m[0][2] * d.z(),
        a.m[1][0] * d.x() + a.m[1][1] * d.y() + a.m[1][2] * d.z(),
        a.m[2][0] * d.x() + a.m[2][1] * d.y() + a.m[2][2] * d.z()
    };
}
```

**`operator*(Matrix4x4, Vector3)` を作りませんでした。** 意図的です。

`Vector3` は、位置を表すこともあれば方向を表すこともあります。型が同じなので、コンパイラには区別できません。そして変換の規則が違います。

- **位置**は平行移動の影響を受ける(`w = 1`)
- **方向**は平行移動の影響を受けない(`w = 0`)

`m * v` という1つの演算子を作ると、どちらか一方を選ばなければなりません。そして**選んだほうと違う意図で使われたときに、静かに間違った結果を返します。**

これはゲーム開発で最も見つけにくいバグの一種です。法線ベクトルが平行移動されて、光の向きが物体の位置によって変わる ── 症状を見ても原因にたどり着けません。

だから、**名前で書かせます。** `TransformPoint` と `TransformDirection` は、どちらを意図しているかがコードに残ります。

13.4.6 の方針では「記号で呼ぶものは隠しフレンド」でした。ここでは**そもそも記号にしない**という判断をしています。演算子は「意味が1つに決まるとき」だけ作るものです。

### 13.5.5 動作確認

`main.cpp` に追加します。

```cpp
// main.cpp

// --- 変換行列の確認 ---
const gamemath::Vector3 t{ 1.0f, 2.0f, 3.0f };
const gamemath::Matrix4x4 T = gamemath::Translation(t);

const gamemath::Vector3 origin{ 0.0f, 0.0f, 0.0f };

const gamemath::Vector3 movedPoint = gamemath::TransformPoint(T, origin);
const gamemath::Vector3 movedDir   = gamemath::TransformDirection(T, origin);

std::println("TransformPoint(T, 0)     = ({:.2f}, {:.2f}, {:.2f})",
             movedPoint.x(), movedPoint.y(), movedPoint.z());
std::println("TransformDirection(T, 0) = ({:.2f}, {:.2f}, {:.2f})",
             movedDir.x(), movedDir.y(), movedDir.z());

// 90 度の Z 回転で X 軸が Y 軸に移る
const float half = std::numbers::pi_v<float> * 0.5f;
const gamemath::Matrix4x4 Rz = gamemath::RotationZ(half);
const gamemath::Vector3 xAxis{ 1.0f, 0.0f, 0.0f };
const gamemath::Vector3 rotated = gamemath::TransformDirection(Rz, xAxis);

std::println("RotationZ(90) * X        = ({:.4f}, {:.4f}, {:.4f})",
             rotated.x(), rotated.y(), rotated.z());

// 合成の順序
const gamemath::Matrix4x4 S = gamemath::Scale(gamemath::Vector3{ 2.0f, 2.0f, 2.0f });
const gamemath::Vector3 st = gamemath::TransformPoint(S * T, origin);
const gamemath::Vector3 ts = gamemath::TransformPoint(T * S, origin);

std::println("S * T applied to 0       = ({:.2f}, {:.2f}, {:.2f})", st.x(), st.y(), st.z());
std::println("T * S applied to 0       = ({:.2f}, {:.2f}, {:.2f})", ts.x(), ts.y(), ts.z());

// 任意軸回転(実装単位にある)
const gamemath::Matrix4x4 Ra =
    gamemath::RotationAxis(gamemath::Vector3{ 0.0f, 0.0f, 1.0f }, half);
const gamemath::Vector3 ra = gamemath::TransformDirection(Ra, xAxis);

std::println("RotationAxis(Z, 90) * X  = ({:.4f}, {:.4f}, {:.4f})",
             ra.x(), ra.y(), ra.z());
```

ビルドして実行してください。

```
TransformPoint(T, 0)     = (1.00, 2.00, 3.00)
TransformDirection(T, 0) = (0.00, 0.00, 0.00)
RotationZ(90) * X        = (-0.0000, 1.0000, 0.0000)
S * T applied to 0       = (2.00, 4.00, 6.00)
T * S applied to 0       = (1.00, 2.00, 3.00)
RotationAxis(Z, 90) * X  = (-0.0000, 1.0000, 0.0000)
```

確認すべき点が4つあります。

**1. 位置と方向の違い。** 同じ `(0,0,0)` を同じ平行移動行列で変換して、結果が違います。`TransformDirection` は平行移動を無視しています。13.5.4 の設計の意図どおりです。

**2. 回転。** X 軸が Y 軸に移りました。`-0.0000` は浮動小数点の誤差です(`cos(π/2)` が正確に 0 にならないため)。

**3. 合成の順序。** `S * T` は「平行移動してから2倍」で `(2,4,6)`、`T * S` は「2倍してから平行移動」で `(1,2,3)` になりました。列ベクトル規約では**右から適用される**という 13.1.2 の説明が確認できます。

**4. 任意軸回転。** `RotationZ` と同じ結果になっています。実装単位に置いた関数も、正しくリンクされています。

---

## 13.6 この章のまとめ

- `Matrix4x4` は `gamemath.core:matrix` パーティションに置いた。第12章の指針1・2による判断
- 行列は、**格納順序**(行優先)と**ベクトルの規約**(列ベクトル)を最初に決めて文書化する。混ぜてはいけない
- `:matrix` と `:basis` は同じ層に並び、どちらも `:vector` に依存する。逆向きの依存は循環するので作らない
- 「ベクトルを行列で変換する関数」は `:matrix` 側に置く。**ベクトルは行列を知らない**
- **取り込む側の宣言は、取り込まれたモジュールの中での探索に参加しない**
- そのため、**ADL による拡張(利用者がオーバーロードを足す)という設計は成立しない**。代わりにコンセプトとメンバ関数を使う
- **隠しフレンド** ── クラス定義の中だけで宣言された `friend` 関数。ADL でしか見つからない
- 隠しフレンドは、**クラスを `export` すれば一緒に届く。個別の `export` が不要**
- 他の利点 ── オーバーロード候補が減る、暗黙変換で呼ばれにくい、`const` の非対称が起きにくい
- **限界 ── 修飾名で呼べない。** `gamemath::Dot(a, b)` と書けなくなる
- 方針: **記号で呼ぶもの(演算子)は隠しフレンド。名前で呼ぶものは `export` 付きの自由関数**
- MSVC では `/permissive-` が必要(既定で有効)。`/Zc:hiddenFriend-` は指定してはいけない
- `operator*(Matrix4x4, Vector3)` は**作らない**。位置と方向で変換規則が違い、静かに間違うため
- 演算子は「意味が1つに決まるとき」だけ作る

## 次章に向けて

第14章では `Quaternion` を追加します。回転を表現する4成分の数で、行列より軽く、補間に向いています。

そして、この章で避けた問題が正面から出てきます。**`Matrix4x4` と `Quaternion` の相互変換**です。

- クォータニオンから行列を作りたい
- 行列からクォータニオンを取り出したい

素直に書くと、`:matrix` と `:quaternion` が互いを `import` して循環します。第10章 10.4.5 で挙げた3つの対処のうち、どれを選ぶべきか ── 実際に手を動かして決めます。

さらに `Slerp`(球面線形補間)を実装しながら、`constexpr` の限界にも触れます。第10章で用意した `SafeAcos` が、ようやく本来の出番を迎えます。
