# 第8章 テンプレートとモジュール

## この章について

第7章で「見える」と「到達可能」の区別を学びました。そのとき、到達可能という概念が必要な理由として3つ挙げ、**最大のものはテンプレートである**と書きました。

この章で、その理由が具体的な形で現れます。

やることは4つです。

1. `Vector3` を `Vector<T, N>` に一般化する(8.1)
2. テンプレートの定義を実装単位に置けない理由を、実際に試して確かめる(8.2)
3. コンセプトを定義し、`export` する(8.3)
4. 明示的インスタンス化で、その制約を部分的に回避する(8.4)

そして最後に、`constexpr` を扱います(8.5)。テンプレートとよく似た性質を持っているからです。

この章を終えると、`GameMath` は 2次元・3次元・4次元、`float` と `double` を扱えるようになります。そして第5章以来の懸案 ── 実装をどこに置くか ── に、テンプレートという新しい制約が加わることになります。

---

## 8.1 `Vector3` を `Vector<T, N>` に一般化する

### 8.1.1 なぜ一般化するのか

いまの `Vector3` は、`float` の3成分に固定されています。ゲーム開発では、これで足りないことがあります。

- **2次元** ── UI 座標、テクスチャ座標、2D ゲーム
- **4次元** ── 同次座標、RGBA カラー、シェーダとのやり取り
- **`double`** ── 広大なワールドの座標、物理シミュレーションの積分

これらのために `Vector2`、`Vector4`、`Vector3d` を別々に書くと、同じコードが4回現れます。バグを直すときに4か所直すことになります。

テンプレートで一度書きます。

### 8.1.2 ファイル名について

これから `Vector3.ixx` は3次元専用ではなくなるので、ファイル名を `Vector.ixx` / `Vector.cpp` に変えます。

**モジュール名 `gamemath.vector` は変わりません。** 第4章 4.5 で確認したとおり、モジュール名とファイル名は無関係です。`import gamemath.vector;` と書いている `main.cpp` は、何も変更しなくて構いません。

改名が面倒なら、`Vector3.ixx` のままでも動きます。以降の説明では `Vector.ixx` と呼びます。

### 8.1.3 テンプレートに書き換える

`Vector.ixx` を書き換えます。この段階では**コンセプトをまだ使いません**。`typename T` のままです(8.3 で直します)。

```cpp
// Vector.ixx
export module gamemath.vector;

import std;

namespace gamemath {

// 非公開の定数(変数テンプレートになった)
template <typename T>
constexpr T kEpsilon = T(1e-6);

export template <typename T, std::size_t N>
struct Vector
{
    T e[N];

    constexpr T& operator[](std::size_t i)
    {
        return e[i];
    }

    constexpr const T& operator[](std::size_t i) const
    {
        return e[i];
    }

    constexpr Vector& operator+=(const Vector& rhs)
    {
        for (std::size_t i = 0; i < N; ++i) {
            e[i] += rhs.e[i];
        }
        return *this;
    }

    constexpr Vector& operator-=(const Vector& rhs)
    {
        for (std::size_t i = 0; i < N; ++i) {
            e[i] -= rhs.e[i];
        }
        return *this;
    }

    constexpr Vector& operator*=(T s)
    {
        for (std::size_t i = 0; i < N; ++i) {
            e[i] *= s;
        }
        return *this;
    }

    constexpr T LengthSquared() const
    {
        T sum = T(0);
        for (std::size_t i = 0; i < N; ++i) {
            sum += e[i] * e[i];
        }
        return sum;
    }

    T Length() const
    {
        return std::sqrt(LengthSquared());
    }

    Vector Normalized() const
    {
        const T len = Length();
        if (len <= kEpsilon<T>) {
            return Vector{};
        }
        const T inv = T(1) / len;
        Vector r{};
        for (std::size_t i = 0; i < N; ++i) {
            r.e[i] = e[i] * inv;
        }
        return r;
    }
};

export template <typename T, std::size_t N>
constexpr Vector<T, N> operator+(const Vector<T, N>& a, const Vector<T, N>& b)
{
    Vector<T, N> r{};
    for (std::size_t i = 0; i < N; ++i) {
        r.e[i] = a.e[i] + b.e[i];
    }
    return r;
}

export template <typename T, std::size_t N>
constexpr Vector<T, N> operator-(const Vector<T, N>& a, const Vector<T, N>& b)
{
    Vector<T, N> r{};
    for (std::size_t i = 0; i < N; ++i) {
        r.e[i] = a.e[i] - b.e[i];
    }
    return r;
}

export template <typename T, std::size_t N>
constexpr Vector<T, N> operator*(const Vector<T, N>& v, T s)
{
    Vector<T, N> r{};
    for (std::size_t i = 0; i < N; ++i) {
        r.e[i] = v.e[i] * s;
    }
    return r;
}

export template <typename T, std::size_t N>
constexpr T Dot(const Vector<T, N>& a, const Vector<T, N>& b)
{
    T sum = T(0);
    for (std::size_t i = 0; i < N; ++i) {
        sum += a.e[i] * b.e[i];
    }
    return sum;
}

// 外積は3次元でのみ意味を持つ
export template <typename T>
constexpr Vector<T, 3> Cross(const Vector<T, 3>& a, const Vector<T, 3>& b)
{
    return Vector<T, 3>{
        a.e[1] * b.e[2] - a.e[2] * b.e[1],
        a.e[2] * b.e[0] - a.e[0] * b.e[2],
        a.e[0] * b.e[1] - a.e[1] * b.e[0]
    };
}

} // namespace gamemath
```

`Cross` に注目してください。引数を `Vector<T, 3>` と書くことで、**3次元でしか呼べない**関数になりました。`Vector2` に対して `Cross` を呼ぶと、テンプレートの実引数推定に失敗してエラーになります。次元の制約が、型で表現できています。

### 8.1.4 エイリアスを用意する

毎回 `Vector<float, 3>` と書くのは苦痛です。別名を用意します。

```cpp
// Vector.ixx(Vector の定義の後ろ、自由関数の前あたり)

export using Vector2  = Vector<float, 2>;
export using Vector3  = Vector<float, 3>;
export using Vector4  = Vector<float, 4>;
export using Vector3d = Vector<double, 3>;
```

**エイリアスにも `export` が必要です。** 第7章 7.5.4 で確認した原則 ── `export` は宣言単位 ── が、ここでも効きます。`Vector` テンプレートを `export` しても、その別名は自動的には公開されません。

これで、既存のコードは `gamemath::Vector3` のまま動きます。ただし1か所だけ変更が必要です。

### 8.1.5 `.x` が使えなくなった

メンバ変数が `float x, y, z;` から `T e[N];` に変わりました。`v.x` とは書けません。

当面は `operator[]` を使います。

```cpp
v[0]   // 旧 v.x
v[1]   // 旧 v.y
v[2]   // 旧 v.z
```

名前付きのアクセサは 8.3.6 で追加します。そこで新しい構文が1つ必要になるので、順番を待ってください。

### 8.1.6 実装単位の関数を移す

`Vector.cpp` には `Orthogonal` と `MakeBasis` がありました。

`MakeBasis` は3次元の `float` 専用なので、テンプレートにする必要がありません。`BasisPair` ともども、そのままで構いません(`.x` → `[0]` の書き換えだけ)。

問題は `Orthogonal` です。テンプレートにしたい ── `double` でも使いたいからです。

そして**テンプレートにすると、実装単位に置いておけません。** これがこの章の中心的な論点です。

いったん、定義をインターフェイス単位に移してください。

```cpp
// Vector.ixx(Cross の後ろに追加)

export template <typename T>
Vector<T, 3> Orthogonal(const Vector<T, 3>& v)
{
    if (v.LengthSquared() <= kEpsilon<T>) {
        return Vector<T, 3>{ T(1), T(0), T(0) };
    }

    const T ax = std::abs(v.e[0]);
    const T ay = std::abs(v.e[1]);
    const T az = std::abs(v.e[2]);

    if (ax <= ay && ax <= az) {
        return Cross(v, Vector<T, 3>{ T(1), T(0), T(0) });
    }
    if (ay <= az) {
        return Cross(v, Vector<T, 3>{ T(0), T(1), T(0) });
    }
    return Cross(v, Vector<T, 3>{ T(0), T(0), T(1) });
}

export struct BasisPair
{
    Vector3 tangent;
    Vector3 bitangent;
};

export BasisPair MakeBasis(const Vector3& normal);
```

`Vector.cpp` は `MakeBasis` だけになります。

```cpp
// Vector.cpp
module gamemath.vector;

import std;

namespace gamemath {

BasisPair MakeBasis(const Vector3& normal)
{
    const Vector3 n = normal.Normalized();
    const Vector3 t = Orthogonal(n).Normalized();
    const Vector3 b = Cross(n, t);
    return BasisPair{ t, b };
}

} // namespace gamemath
```

### 8.1.7 動かして確認する

`main.cpp` を書き換えます。

```cpp
// main.cpp
import std;
import gamemath.vector;

int main()
{
    const gamemath::Vector3 a{ 1.0f, 2.0f, 3.0f };
    const gamemath::Vector3 b{ 4.0f, 5.0f, 6.0f };

    const gamemath::Vector3 sum = a + b;
    const gamemath::Vector3 crs = gamemath::Cross(a, b);
    const gamemath::Vector3 nrm = a.Normalized();
    const gamemath::BasisPair basis = gamemath::MakeBasis(nrm);

    std::println("a + b      = ({:.2f}, {:.2f}, {:.2f})", sum[0], sum[1], sum[2]);
    std::println("Cross      = ({:.2f}, {:.2f}, {:.2f})", crs[0], crs[1], crs[2]);
    std::println("Dot        = {:.2f}", gamemath::Dot(a, b));
    std::println("|a|        = {:.4f}", a.Length());
    std::println("Normalized = ({:.4f}, {:.4f}, {:.4f})", nrm[0], nrm[1], nrm[2]);
    std::println("Basis: Dot(n,t) = {:.4f}  Dot(n,b) = {:.4f}",
                 gamemath::Dot(nrm, basis.tangent),
                 gamemath::Dot(nrm, basis.bitangent));

    // 新しくできるようになったこと
    const gamemath::Vector2 uv{ 0.25f, 0.75f };
    const gamemath::Vector4 rgba{ 1.0f, 0.5f, 0.0f, 1.0f };
    const gamemath::Vector3d world{ 1.0e7, 2.0e7, 3.0e7 };

    std::println("uv         = ({:.2f}, {:.2f})  |uv| = {:.4f}",
                 uv[0], uv[1], uv.Length());
    std::println("rgba       = ({:.2f}, {:.2f}, {:.2f}, {:.2f})",
                 rgba[0], rgba[1], rgba[2], rgba[3]);
    std::println("world(d)   = |w| = {:.4f}", world.Length());

    return 0;
}
```

ビルドして実行します。

```
a + b      = (5.00, 7.00, 9.00)
Cross      = (-3.00, 6.00, -3.00)
Dot        = 32.00
|a|        = 3.7417
Normalized = (0.2673, 0.5345, 0.8018)
Basis: Dot(n,t) = 0.0000  Dot(n,b) = 0.0000
uv         = (0.25, 0.75)  |uv| = 0.7906
rgba       = (1.00, 0.50, 0.00, 1.00)
world(d)   = |w| = 37416573.8674
```

1本のテンプレートから、2次元・3次元・4次元、`float` と `double` が生まれました。

---

## 8.2 テンプレートの定義はなぜ到達可能でなければならないか

8.1.6 で、`Orthogonal` の定義をインターフェイス単位に移しました。「そうしないと動かない」と書きましたが、本当でしょうか。確かめます。

### 8.2.1 【実験】実装単位に戻してみる

`Vector.ixx` からは `Orthogonal` の**宣言だけ**を残します。

```cpp
// Vector.ixx(Cross の後ろ)

export template <typename T>
Vector<T, 3> Orthogonal(const Vector<T, 3>& v);   // ← 宣言だけ
```

`Vector.cpp` に定義を移します。

```cpp
// Vector.cpp
module gamemath.vector;

import std;

namespace gamemath {

template <typename T>
Vector<T, 3> Orthogonal(const Vector<T, 3>& v)
{
    if (v.LengthSquared() <= kEpsilon<T>) {
        return Vector<T, 3>{ T(1), T(0), T(0) };
    }

    const T ax = std::abs(v.e[0]);
    const T ay = std::abs(v.e[1]);
    const T az = std::abs(v.e[2]);

    if (ax <= ay && ax <= az) {
        return Cross(v, Vector<T, 3>{ T(1), T(0), T(0) });
    }
    if (ay <= az) {
        return Cross(v, Vector<T, 3>{ T(0), T(1), T(0) });
    }
    return Cross(v, Vector<T, 3>{ T(0), T(0), T(1) });
}

BasisPair MakeBasis(const Vector3& normal)
{
    // ...
}

} // namespace gamemath
```

第5章でやったことと、形は同じです。宣言をインターフェイスに、定義を実装単位に。

ビルドしてください。

### 8.2.2 リンクエラーを読む

`main.cpp` から `Orthogonal` を呼んでいなければ、実は通ってしまいます。呼んでみましょう。`main.cpp` に1行足してください。

```cpp
const gamemath::Vector3 ort = gamemath::Orthogonal(a);
std::println("Orthogonal = ({:.2f}, {:.2f}, {:.2f})", ort[0], ort[1], ort[2]);
```

ビルドすると、**リンクエラーになります。**

```
error LNK2019: 未解決の外部シンボル
  "struct gamemath::Vector<float,3> __cdecl gamemath::Orthogonal<float>(
   struct gamemath::Vector<float,3> const &)" が関数 main で参照されました
```

コンパイルは通っています。宣言はあるからです。しかしリンクで失敗しました。

そして注目してほしいのは、`Vector.cpp` に定義があるにもかかわらず失敗している点です。第5章で `Cross` を分割したときは、これで動きました。何が違うのでしょうか。

### 8.2.3 テンプレートは「関数」ではなく「設計図」

答えは、**`Vector.cpp` の `Orthogonal` は関数ではない**からです。

テンプレートは、関数そのものではありません。「型が決まったら、こういう関数を作れ」という**設計図**です。設計図からは機械語が生成されません。

`Vector.cpp` をコンパイルしたとき、コンパイラは `Orthogonal` の設計図を読みました。しかし `T` が何であるか誰も指定していないので、何も生成しませんでした。`Vector.cpp` の `.obj` に `Orthogonal<float>` は入っていません。

一方 `main.cpp` では、`Orthogonal(a)` という呼び出しから `T = float` が決まります。ここで `Orthogonal<float>` という実体が必要になります。

しかし `main.cpp` が読んだ `.ifc` には、宣言しか入っていません。設計図が届いていないので、`main.cpp` は実体を作れません。「どこかに `Orthogonal<float>` があるはずだ」とリンカに丸投げし、リンカが見つけられなかった ── これがエラーの正体です。

**テンプレートを使う場所には、定義そのものが届いていなければなりません。**

### 8.2.4 ヘッダ時代の常識との接続

この現象自体は、モジュール特有ではありません。ヘッダの世界でも同じでした。

だからこそ、テンプレートは `.h` に書くのが常識だったのです。`.cpp` に書いたテンプレートは、他の翻訳単位から使えませんでした。

STL がすべてヘッダで提供されているのも、`Boost` の多くがヘッダオンリーなのも、根はここにあります。そして第1章 1.1.4 で見たビルド時間の問題 ── テンプレートを多用するライブラリほど、ヘッダが巨大になり、解析コストが跳ね上がる ── も、ここから来ていました。

モジュールでは、この構図が少し変わります。定義がインターフェイス単位にあることは変わりませんが、**解析は1回だけ**です。テキストとして毎回展開されるのとは違います。

とはいえ、「定義を隠せない」という性質そのものは残っています。

### 8.2.5 到達可能性がすべてを説明する

第7章の言葉を使うと、この現象は1行で説明できます。

**実装単位に書いたものは、到達可能ではない。テンプレートは到達可能でなければならない。**

第7章 7.4.3 の表を再掲します。

| | 見える | 到達可能 |
|---|---|---|
| `export` した宣言(インターフェイス単位) | ○ | ○ |
| `export` しない宣言(インターフェイス単位) | ✕ | **○** |
| 実装単位に書いた宣言 | ✕ | ✕ |

3行目です。実装単位の中身は `.ifc` に入らないので、`import` した側には何も届きません。

だから、テンプレートの置き場所は**インターフェイス単位一択**になります。

面白いのは、2行目です。**`export` していなくても、到達可能なら使えます。** 実際、`Orthogonal` の中で使っている `kEpsilon<T>` は `export` していません。それでも `main.cpp` での実体化が成功するのは、`kEpsilon` が到達可能だからです。

第7章 7.4.2 で「到達可能という概念が必要な最大の理由はテンプレート」と書いた意味が、これで具体的になったと思います。

### 8.2.6 その副作用

テンプレートがインターフェイス単位にしか置けないということは、いくつかの帰結を生みます。

**実装を隠せません。** 利用者が `.ifc` から定義を取り出すことは(通常は)ありませんが、少なくとも「実装を変えても利用者に影響しない」という保証はなくなります。

**変更すれば再コンパイルされます。** 第7章 7.4.5 で扱った問題です。テンプレートの中身を1行変えるだけで、`import` している全ファイルが再コンパイルされます。

**`.ifc` が大きくなります。** テンプレートを増やすほど、`.ifc` に載る量が増えます。

これらは第1章 1.4.5 で「テンプレートのコストは消えない」と書いた内容と重なります。モジュールはテンプレートの解析回数を減らしますが、テンプレートを隠せるようにはしません。

**とはいえ、逃げ道が1つだけあります。** それが 8.4 の明示的インスタンス化です。

いまは実験を元に戻して、`Orthogonal` の定義をインターフェイス単位(`Vector.ixx`)に戻してください。ビルドが通ることを確認します。8.4 でまた動かします。

---

## 8.3 コンセプトを定義してエクスポートする

### 8.3.1 いまの `Vector<T, N>` の問題

こう書けてしまいます。

```cpp
gamemath::Vector<int, 3> v{ 1, 2, 3 };
std::println("{}", v.Length());
```

`T = int` です。`Length()` は `std::sqrt(int)` を呼び、`double` が返り、`int` に切り詰められます。`Normalized()` は `1 / len` が整数除算になって、ほぼ確実にゼロベクトルを返します。

コンパイルは通ります。動きます。**間違った答えを返します。**

もっと悪いのは `Vector<std::string, 3>` のような場合で、こちらはコンパイルエラーになりますが、エラーメッセージは `Vector` の内部を指します。利用者は自分が何を間違えたのか分かりません。

**`Vector` は浮動小数点数でしか使えない** ── この制約を、型で表現します。

### 8.3.2 `FloatingPoint` を定義する

`Vector.ixx` の先頭近くに、コンセプトを追加します。

```cpp
// Vector.ixx
export module gamemath.vector;

import std;

namespace gamemath {

// GameMath がサポートするスカラー型
export template <typename T>
concept FloatingPoint = std::same_as<T, float> || std::same_as<T, double>;

// ...
```

標準ライブラリにも `std::floating_point` がありますが、そちらは `long double` も含みます。`GameMath` は `float` と `double` だけを対象にするので、自分で定義します。ゲーム開発で `long double` を使うことはまずありませんし、プラットフォームによって精度が違うので避けたい型です。

**コンセプトにも `export` が必要です。** 理由は 8.3.4 で確認します。

### 8.3.3 適用する

`typename T` を `FloatingPoint T` に置き換えていきます。

```cpp
template <FloatingPoint T>
constexpr T kEpsilon = T(1e-6);

export template <FloatingPoint T, std::size_t N>
struct Vector
{
    // ...
};

export template <FloatingPoint T, std::size_t N>
constexpr Vector<T, N> operator+(const Vector<T, N>& a, const Vector<T, N>& b)
{
    // ...
}

// 以下、すべての自由関数について同様
```

ビルドしてください。通ります。これまでのコードは何も変わりません。

そして、`Vector<int, 3>` を書いてみてください。

```
error C7602: 'gamemath::Vector': 関連付けられた制約が満たされていません
```

エラーが**使った側**で出るようになりました。しかも「制約が満たされていない」という、原因を指したメッセージです。

### 8.3.4 【実験】コンセプトを `export` し忘れると

`FloatingPoint` から `export` を外してみてください。

```cpp
template <typename T>          // ← export を削除
concept FloatingPoint = std::same_as<T, float> || std::same_as<T, double>;
```

ビルドしてください。**通ります。**

`main.cpp` は `FloatingPoint` という名前を書いていません。`gamemath::Vector3` を使っているだけです。テンプレートの実体化に必要な情報は、第7章の「到達可能」で届いています。だから動きます。

では、`export` する意味はどこにあるのか。

**利用者が自分でテンプレートを書くときです。**

```cpp
// main.cpp
template <gamemath::FloatingPoint T>          // ← ここで名前が必要
gamemath::Vector<T, 3> Doubled(const gamemath::Vector<T, 3>& v)
{
    return v + v;
}
```

これを書くと、`export` を外した状態ではエラーになります。

```
error C2065: 'FloatingPoint': 定義されていない識別子です。
```

第7章 7.5 で `BasisPair` に起きたことと、まったく同じ構図です。**動くけれど、名前が書けない。** そして、そのことに気づくのはライブラリの利用者であって、作者ではありません。

**`export` を戻してください。** そして 7.5.4 の原則を思い出してください。

> 公開する宣言のシグネチャに現れる型は、必ず公開する。

コンセプトも、この「シグネチャに現れるもの」に含まれます。`template <FloatingPoint T>` と書いた時点で、`FloatingPoint` はテンプレートのインターフェイスの一部です。

上の `Doubled` は実験用なので、確認できたら削除してください。

### 8.3.5 `requires` 節で次元を制約する

コンセプトのもう1つの使い道が、**メンバごとの制約**です。8.1.5 で先送りにした、名前付きアクセサを追加します。

```cpp
export template <FloatingPoint T, std::size_t N>
struct Vector
{
    T e[N];

    constexpr T& operator[](std::size_t i)             { return e[i]; }
    constexpr const T& operator[](std::size_t i) const { return e[i]; }

    constexpr T x() const requires (N >= 1) { return e[0]; }
    constexpr T y() const requires (N >= 2) { return e[1]; }
    constexpr T z() const requires (N >= 3) { return e[2]; }
    constexpr T w() const requires (N >= 4) { return e[3]; }

    // ...(以下同じ)
};
```

`requires (N >= 1)` は、「この条件を満たすときだけ、このメンバが存在する」という指定です。

- `Vector2` には `x()` と `y()` だけがあります
- `Vector3` には `z()` まで
- `Vector4` には `w()` まで

`Vector2` に対して `.z()` を呼ぶと、エラーになります。**存在しないメンバを呼ぼうとしている**という、正しいエラーです。ヘッダ時代の `Vector2` に `z` を足してしまう事故が、構造的に防がれます。

`main.cpp` を書き換えて、読みやすくしましょう。

```cpp
std::println("a + b      = ({:.2f}, {:.2f}, {:.2f})", sum.x(), sum.y(), sum.z());
std::println("Cross      = ({:.2f}, {:.2f}, {:.2f})", crs.x(), crs.y(), crs.z());
std::println("uv         = ({:.2f}, {:.2f})  |uv| = {:.4f}",
             uv.x(), uv.y(), uv.Length());
std::println("rgba       = ({:.2f}, {:.2f}, {:.2f}, {:.2f})",
             rgba.x(), rgba.y(), rgba.z(), rgba.w());
```

試しに `uv.z()` と書いてみて、エラーになることを確認してください。

> **メンバ変数の `.x` に戻せないのか**
>
> 戻せます。共用体を使う、次元ごとに部分特殊化する、基底クラスを分ける ── どの方法にも欠点があり、実際のライブラリでも選択が分かれています。
>
> 本書ではアクセサ関数を採用します。`constexpr` なので最適化後のコストはゼロで、`requires` によって次元の制約を型で表現できるためです。ただし、既存コードからの移行では `.x` を保ちたい要求が強く、そのために共用体を使う判断も十分に合理的です。

---

## 8.4 明示的インスタンス化

### 8.4.1 動機

8.2 で「テンプレートの定義はインターフェイス単位に置くしかない」と結論しました。8.2.6 で挙げた副作用 ── 隠せない、再コンパイルを誘発する、`.ifc` が膨らむ ── は、そのまま受け入れるしかないのでしょうか。

ひとつだけ抜け道があります。

**対応する型をあらかじめ決めてしまえば、実体を先に作っておける。**

`GameMath` の `Vector<T, N>` の `T` は、`FloatingPoint` によって `float` と `double` の2つに限定されています。ならば、その2つの実体を**ライブラリ側で作っておいて**、利用者にはそれを使ってもらえばよいはずです。

これが**明示的インスタンス化**(explicit instantiation)です。

### 8.4.2 3点セット

やることは3つです。

1. **インターフェイス単位に、宣言だけを置く**(`export` 付き)
2. **実装単位に、定義を書く**
3. **実装単位に、明示的インスタンス化を書く**

`Orthogonal` でやってみましょう。8.2.1 の実験と 1・2 は同じです。3 が加わります。

**インターフェイス単位:**

```cpp
// Vector.ixx(Cross の後ろ)

export template <FloatingPoint T>
Vector<T, 3> Orthogonal(const Vector<T, 3>& v);   // 宣言だけ
```

**実装単位:**

```cpp
// Vector.cpp
module gamemath.vector;

import std;

namespace gamemath {

template <FloatingPoint T>
Vector<T, 3> Orthogonal(const Vector<T, 3>& v)
{
    if (v.LengthSquared() <= kEpsilon<T>) {
        return Vector<T, 3>{ T(1), T(0), T(0) };
    }

    const T ax = std::abs(v.x());
    const T ay = std::abs(v.y());
    const T az = std::abs(v.z());

    if (ax <= ay && ax <= az) {
        return Cross(v, Vector<T, 3>{ T(1), T(0), T(0) });
    }
    if (ay <= az) {
        return Cross(v, Vector<T, 3>{ T(0), T(1), T(0) });
    }
    return Cross(v, Vector<T, 3>{ T(0), T(0), T(1) });
}

// ★ 明示的インスタンス化
template Vector<float, 3>  Orthogonal(const Vector<float, 3>&);
template Vector<double, 3> Orthogonal(const Vector<double, 3>&);

BasisPair MakeBasis(const Vector3& normal)
{
    const Vector3 n = normal.Normalized();
    const Vector3 t = Orthogonal(n).Normalized();
    const Vector3 b = Cross(n, t);
    return BasisPair{ t, b };
}

} // namespace gamemath
```

`template` で始まり `<...>` を伴わない宣言 ── これが明示的インスタンス化の構文です。「この型で実体を作れ」という指示です。

これによって、`Vector.cpp` の `.obj` の中に `Orthogonal<float>` と `Orthogonal<double>` の機械語が生成されます。

ビルドして実行してください。8.2.2 でリンクエラーになったコードが、今度は通ります。

```
Orthogonal = (0.00, 3.00, -2.00)
```

`double` 版も試せます。

```cpp
const gamemath::Vector3d wd{ 1.0, 2.0, 3.0 };
const gamemath::Vector3d od = gamemath::Orthogonal(wd);
std::println("Orthogonal(d) = ({:.2f}, {:.2f}, {:.2f})", od.x(), od.y(), od.z());
```

### 8.4.3 対応していない型を使うと

`FloatingPoint` は `float` と `double` だけなので、いまのところ他の型は使えません。しかし仮に、コンセプトを緩めて `long double` を許したとしましょう。

```cpp
gamemath::Vector<long double, 3> v{ 1.0L, 2.0L, 3.0L };
auto o = gamemath::Orthogonal(v);   // ← リンクエラー
```

コンパイルは通り、**リンクで失敗します。** 8.2.2 とまったく同じエラーです。

明示的インスタンス化していない型については、実体が存在しないからです。

これが明示的インスタンス化の代償です。**ライブラリ側があらかじめ列挙した型しか使えなくなります。**

`GameMath` の場合、`FloatingPoint` コンセプトが `float` と `double` に限定しているので、コンセプトと明示的インスタンス化の対象が一致します。**制約を型で表現しておいたことが、ここで効いています。** 対応していない型は、リンクエラーではなくコンセプト違反として、分かりやすいエラーになります。

### 8.4.4 `extern template` は要らない

ヘッダの世界で明示的インスタンス化を使うときは、`extern template` という宣言をヘッダに書くのが定番でした。

```cpp
// ヘッダ時代
extern template Vector<float, 3> Orthogonal(const Vector<float, 3>&);
```

これは「ここでは実体化するな、どこかに実体があるから」という指示です。ヘッダには定義が書いてあるので、放っておくと各翻訳単位が勝手に実体を作ってしまうからです。

モジュールでは不要です。インターフェイス単位に定義がないので、実体化しようがありません。**放っておいても勝手に作られません。**

構造がすっきりしています。

### 8.4.5 いつ使うべきか

明示的インスタンス化は強力ですが、代償が大きい手法です。

**向いている場面**

- 対応する型が少数に確定している(`float` / `double` だけ、など)
- その関数が長く、`.ifc` に載せたくない
- 実装を頻繁に変更する予定がある
- コンパイル時間が実測で問題になっている

**向いていない場面**

- 利用者が任意の型で使うことを想定している
- 短い関数(インライン展開の効果のほうが大きい。第5章 5.6.1)
- `constexpr` にしたい(8.5 で扱います)

### 8.4.6 `GameMath` の方針

第5章 5.6.5 で決めた基準に、テンプレートの条件を加えます。

| 種類 | 置き場所 |
|---|---|
| 短い関数・演算子(テンプレートかどうかを問わず) | インターフェイス単位に定義 |
| `constexpr` にしたいもの | インターフェイス単位に定義 |
| 長く、呼び出し頻度が低い**非**テンプレート | 実装単位 |
| 長く、呼び出し頻度が低いテンプレートで、型が確定しているもの | **実装単位 + 明示的インスタンス化** |
| 利用者が任意の型で使うテンプレート | インターフェイス単位に定義 |

`Orthogonal` は4行目に該当します。`GameMath` の中では、これが唯一の明示的インスタンス化の例になります。第15章の衝突判定でも同じ判断が必要になりますが、そちらは非テンプレートで済ませる予定です。

---

## 8.5 `constexpr` 関数とコンパイル時計算

### 8.5.1 `constexpr` はすでに書いてある

8.1.3 のコードを見返してください。`operator+`、`Dot`、`Cross`、`LengthSquared`、`operator[]` に `constexpr` が付いています。

モジュールだからといって、`constexpr` の書き方は変わりません。特別な構文はありません。

しかし、モジュールとの関係で押さえるべき性質が1つあります。それは 8.5.4 で扱います。まず動かしてみましょう。

### 8.5.2 `static_assert` で確かめる

`main.cpp` に書いてみてください。

```cpp
// main.cpp(main の外でも中でもよい)

constexpr gamemath::Vector3 ca{ 1.0f, 2.0f, 3.0f };
constexpr gamemath::Vector3 cb{ 4.0f, 5.0f, 6.0f };

static_assert(gamemath::Dot(ca, cb) == 32.0f);

constexpr gamemath::Vector3 ccross = gamemath::Cross(ca, cb);
static_assert(ccross.x() == -3.0f);
static_assert(ccross.y() ==  6.0f);
static_assert(ccross.z() == -3.0f);

static_assert(ca.LengthSquared() == 14.0f);
```

ビルドしてください。通ります。

**この計算は、実行時には1回も行われていません。** コンパイル時にすべて済んでいます。生成されたバイナリには、計算結果の定数だけが埋め込まれます。

ゲーム開発では、これが効く場面がたくさんあります。座標系の基底ベクトル、単位行列、よく使う回転量 ── コンパイル時に確定できるものは、確定させておくべきです。

### 8.5.3 `Length()` を `constexpr` にできない理由

`LengthSquared()` には `constexpr` が付いていますが、`Length()` には付いていません。

理由は `std::sqrt` です。C++23 の時点で、`<cmath>` の数学関数はまだ `constexpr` ではありません。だから `Length()` も `constexpr` にできません。

(C++26 でこれらが `constexpr` になる方向で議論が進んでいます。標準が更新されれば、`Length()` にも `constexpr` を付けられるようになります。)

これはモジュールとは無関係な、標準ライブラリ側の事情です。しかし実務上は、**比較には `LengthSquared()` を使う**という設計方針を後押しします。第4章 4.6 で書いたとおりです。

### 8.5.4 `constexpr` も到達可能性を要求する

ここがモジュールとの接点です。

**`constexpr` 関数をコンパイル時に評価するには、その定義が到達可能でなければなりません。**

理由はテンプレートと同じです。コンパイラは、呼び出し側で関数の中身を実行してみる必要があります。中身が届いていなければ、実行できません。

つまり、**`constexpr` 関数を実装単位に置くと、`constexpr` としての価値がなくなります。**

確かめてみましょう。`Dot` の定義を `Vector.cpp` に移す ── ことはできません。テンプレートだからです(8.2)。

では、テンプレートではない `constexpr` 関数で試します。`Vector.ixx` に追加してください。

```cpp
// Vector.ixx
export constexpr float DegreesToRadians(float degrees);   // 宣言だけ
```

```cpp
// Vector.cpp
constexpr float DegreesToRadians(float degrees)
{
    return degrees * 3.14159265f / 180.0f;
}
```

```cpp
// main.cpp
static_assert(gamemath::DegreesToRadians(180.0f) > 3.14f);   // ← エラー
```

`static_assert` が通りません。「定数式ではない」という趣旨のエラーになります。実行時に呼ぶだけなら動きますが、コンパイル時評価はできません。

**確認できたら、この3か所を削除してください。**

### 8.5.5 3つが同じ規則で説明できる

第7章から続いてきた話が、ここで1つにまとまります。

| これをやりたい | 定義がどこにあれば良いか |
|---|---|
| 関数を呼ぶだけ | どこでもよい(リンクできればよい) |
| **インライン展開させたい** | **到達可能** |
| **テンプレートを実体化したい** | **到達可能** |
| **`constexpr` でコンパイル時評価したい** | **到達可能** |

つまり、**コンパイラに「仕事をさせたい」ものはすべて、インターフェイス単位に置く必要があります。**

実装単位に置けるのは、「呼ぶだけでよい関数」だけです。

数学ライブラリの関数の大半は、インライン展開させたいか、テンプレートか、`constexpr` にしたいかのどれかです。だから `GameMath` は、コードの大部分がインターフェイス単位に集まる構造になります。第5章 5.6.5 で「数学ライブラリは他の分野と判断が逆になる」と書いた理由が、これで完全に説明できました。

---

## 8.6 この章のまとめ

- `Vector<T, N>` に一般化すると、1本のテンプレートから 2/3/4 次元、`float`/`double` が得られる
- エイリアス(`using Vector3 = ...`)にも `export` が必要
- **テンプレートの定義は、実装単位に置けない。** テンプレートは設計図であって、機械語を生成しないため
- 実装単位の中身は**到達可能ではない**ので、`import` した側は実体化できない。結果はリンクエラー
- テンプレートを使う場所には、定義そのものが届いていなければならない。ヘッダ時代と同じ制約
- ただしモジュールでは、その定義の**解析が1回で済む**点が違う
- **コンセプトにも `export` が必要。** 忘れても動くが、利用者が自分のテンプレートに書けなくなる
- `requires` 節で、次元に応じてメンバの有無を制御できる
- **明示的インスタンス化**(実装単位に定義 + `template ...;`)を使えば、テンプレートを実装単位に置ける
- ただし、あらかじめ列挙した型しか使えなくなる。コンセプトで型を絞ってあると相性が良い
- モジュールでは `extern template` は不要
- **インライン展開・テンプレートの実体化・`constexpr` 評価は、すべて「到達可能」を要求する**
- 数学ライブラリはこの3つのどれかに該当する関数が大半なので、コードはインターフェイス単位に集まる

## 次章に向けて

`Vector.ixx` は、そろそろ 200 行に届こうとしています。そして、これから追加するものが控えています。

- `Matrix4x4`(第13章)
- `Quaternion`(第14章)
- `AABB` / `Sphere` / `Ray` / `Plane` と衝突判定(第15章)

これを全部 `Vector.ixx` に書き足していくわけにはいきません。かといって、第4章 4.5.5 で見たように、モジュールを完全に分けると `import` が増え、依存関係の管理が必要になります。

第9章では、**モジュールパーティション**という仕組みを導入します。1つのモジュールを、複数のファイルに分割する方法です。ヘッダを分割するのとは違い、外から見た姿は1つのモジュールのまま保たれます。

第3部では、`GameMath` を「育てる」段階から「組み立てる」段階に進めます。
