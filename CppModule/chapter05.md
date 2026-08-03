# 第5章 インターフェイスと実装を分ける

## この章について

第4章で `Vector3` をモジュール化しました。インターフェイスも実装も、`Vector3.ixx` という1つのファイルに入っています。

この章では、それを2つに分けます。ヘッダファイルと `.cpp` に分けるのと似ていますが、モジュールの分け方には独特の性質がいくつかあります。

- 実装側のファイルからは、**インターフェイスの中身が全部見えます**。`import` も `#include` も要りません
- 実装側のファイルには、**`export` を書けません**
- そして最も重要なことに ── **分けるべきかどうかは、それ自体が設計判断です**

最後の点について、この章は少し変わった構成を取ります。5.3 で `Cross()` を分割し、5.6 でそれを**元に戻します**。道具の使い方を覚えることと、使うべき場面を見極めることは別だからです。数学ライブラリでは、この見極めを誤ると実行速度に直接跳ね返ります。

代わりに、分割するのが適切な関数を1つ新しく追加します。それがこの章の成果物になります。

---

## 5.1 `.ixx` が肥大化してきた

### 5.1.1 いまの `Vector3.ixx`

第4章が終わった時点で、`Vector3.ixx` はこうなっています。

```cpp
// Vector3.ixx
export module gamemath.vector;

namespace gamemath {

export struct Vector3
{
    float x;
    float y;
    float z;

    Vector3& operator+=(const Vector3& rhs) { /* 5行 */ }
    Vector3& operator-=(const Vector3& rhs) { /* 5行 */ }
    Vector3& operator*=(float s)            { /* 5行 */ }
    float LengthSquared() const             { /* 1行 */ }
};

export Vector3 operator+(const Vector3& a, const Vector3& b) { /* ... */ }
export Vector3 operator-(const Vector3& a, const Vector3& b) { /* ... */ }
export Vector3 operator*(const Vector3& v, float s)          { /* ... */ }
export float   Dot(const Vector3& a, const Vector3& b)       { /* ... */ }
export Vector3 Cross(const Vector3& a, const Vector3& b)     { /* ... */ }

} // namespace gamemath
```

80行ほどです。まだ読める量ですが、これから増えます。第6章で `Length()` と `Normalized()`、第13章で行列との変換、第14章でクォータニオンとの相互運用 ── 実務のベクトルクラスは、200行から500行になるのが普通です。

### 5.1.2 問題1 ── インターフェイスが見えない

「`Vector3` に何ができるのか」を知りたい人は、このファイルを開きます。そして、実装コードの間から宣言を拾い読みすることになります。

ヘッダファイルの時代には、これが自然に解決していました。`Vector3.h` を見れば宣言だけが並んでいて、実装は `Vector3.cpp` にある ── 意図してそうしたというより、`inline` を付けない限りそう書くしかなかったからですが、結果として「宣言の一覧」が手に入っていました。

モジュールでは定義をインターフェイスに書けてしまうので、意識的に分けないと一覧が失われます。

### 5.1.3 問題2 ── 実装を変えると、全員が再コンパイルされる

こちらのほうが実害があります。第2章 2.4.3 で確認した現象を、`GameMath` でもう一度見ておきましょう。

`Vector3.ixx` の `Cross` の中身を、意味を変えずに書き換えてみてください。たとえば一時変数を使う形に。

```cpp
export Vector3 Cross(const Vector3& a, const Vector3& b)
{
    const float cx = a.y * b.z - a.z * b.y;   // ← 書き方を変えただけ
    const float cy = a.z * b.x - a.x * b.z;
    const float cz = a.x * b.y - a.y * b.x;
    return Vector3{ cx, cy, cz };
}
```

ビルドしてください。出力ウィンドウに `main.cpp` が現れます。

`main.cpp` は1文字も変えていません。`Vector3` のインターフェイスも変わっていません。それでも再コンパイルされました。`.ifc` が作り直されたからです。

いまはファイルが2つなので一瞬ですが、`gamemath.vector` を `import` しているファイルが 200 個あったら、実装の一時変数を1つ増やすたびに 200 ファイルが再コンパイルされます。

**インターフェイス単位に書いたものは、すべて `.ifc` の一部になります。** 公開・非公開は関係ありません。ファイルが変われば `.ifc` が変わり、依存する側は作り直しになります。

### 5.1.4 欲しいもの

まとめると、こういうファイル構成が欲しくなります。

- **インターフェイスのファイル** ── 宣言だけが並んでいる。滅多に変わらない
- **実装のファイル** ── 定義が入っている。ここを変えても `.ifc` は変わらない

ヘッダと `.cpp` の関係そのものです。モジュールでこれを実現するのが、**モジュール実装単位**です。

---

## 5.2 実装単位を作る

### 5.2.1 `Vector3.cpp` を追加する

「ソース ファイル」を右クリック → [追加] → [新しい項目] → **「C++ ファイル (.cpp)」** を選び、`Vector3.cpp` として追加します。

**`.ixx` ではなく `.cpp` です。** ここは重要なので、後で理由を説明します。

内容は1行だけにします。

```cpp
// Vector3.cpp
module gamemath.vector;
```

### 5.2.2 この1行の意味

第3章 3.2.2 で、モジュール宣言を分解しました。

```cpp
export module gamemath.vector;   // インターフェイス単位
module gamemath.vector;          // 実装単位
```

違いは `export` の有無だけです。

- `export` **あり** ── このファイルは `gamemath.vector` の**外向きの顔**を定義する
- `export` **なし** ── このファイルは `gamemath.vector` の**一部**だが、外向きの顔ではない

`export` のない `module gamemath.vector;` で始まるファイルは、**モジュール実装単位**(module implementation unit)になります。

このファイルは `.ifc` を生成しません。普通の `.cpp` と同じように `.obj` になり、リンクされます。ただの `.cpp` と違うのは、`gamemath.vector` というモジュールに所属している点だけです。

### 5.2.3 ビルドして確認する

ビルドしてください。通ります。

出力ウィンドウを見てください。

```
1>Vector3.ixx
1>Vector3.cpp
1>main.cpp
```

`Vector3.ixx` が最初です。`Vector3.cpp` も `main.cpp` も、`gamemath.vector` の `.ifc` を必要とするからです。

中身が1行しかない実装単位ですが、これで文法的には完成しています。次の節で中身を移します。

### 5.2.4 用語を整理する

新しい言葉が増えてきたので、ここで整理します。

**モジュール単位**(module unit)とは、モジュール宣言を含む翻訳単位のことです。つまり `export module ...;` または `module ...;` で始まるファイルです。モジュール単位には種類があります。

| 種類 | 書き出し | `.ifc` | 役割 |
|---|---|---|---|
| **プライマリモジュールインターフェイス単位** | `export module M;` | 生成する | モジュールの外向きの顔。**1モジュールに1つだけ** |
| **モジュール実装単位** | `module M;` | 生成しない | 実装を書く。**いくつあってもよい** |
| モジュールインターフェイスパーティション | `export module M:P;` | 生成する | 第9章 |
| 内部パーティション | `module M:P;` | 生成する | 第10章 |

この章で扱うのは、上の2つです。下の2つは第9章と第10章で登場します。

いま `gamemath.vector` は、次の2つのモジュール単位からできています。

- `Vector3.ixx` ── プライマリモジュールインターフェイス単位
- `Vector3.cpp` ── モジュール実装単位

そして `main.cpp` は、`gamemath.vector` を `import` しているだけで、モジュール単位ではありません。モジュール宣言がないからです。こういう普通の翻訳単位は、**グローバルモジュール**に属している、と言います。

### 5.2.5 なぜ拡張子は `.cpp` なのか

第2章 2.3.1 で確認したとおり、MSVC は `.ixx` を自動的にモジュールインターフェイス単位(`/interface`)として扱います。

実装単位は**インターフェイス単位ではありません**。`.ifc` を生成しないので、`/interface` を付けてはいけません。だから `.ixx` は使わず、普通の `.cpp` にします。

拡張子の使い分けをまとめると、こうなります。

| 拡張子 | 中身 | コンパイル |
|---|---|---|
| `.ixx` | `export module ...;` で始まる | `/interface` が自動で付く |
| `.cpp` | `module ...;` で始まる、または普通のコード | 通常のコンパイル |

`.cpp` の中身がモジュール実装単位かどうかは、ファイルの先頭を見ればコンパイラが判断します。プロジェクトの設定で何かを指定する必要はありません。

> **コラム: 実装単位とビルド設定**
>
> 第2章 2.3.2 で「モジュール依存関係のソースをスキャンする」という設定を確認しました。既定は「いいえ」で、その理由は「`.ixx` は設定に関係なく常にスキャンされるから」でした。
>
> ではいま、`.cpp` である `Vector3.cpp` がモジュールに参加しました。この依存関係は追跡されるのでしょうか。
>
> Visual Studio のプロジェクトは、インターフェイス単位を先にコンパイルするようビルド順序を組むので、本書の構成では既定のままで問題なく動きます。ビルドしてエラーが出ないなら、そのままで構いません。
>
> ただし、ファイルが増えて並列ビルドが効き始めると、順序に起因するエラー(「モジュールが見つからない」がビルドのたびに出たり出なかったりする)が起きる可能性があります。そのときは **[C/C++] → [全般] → モジュール依存関係のソースをスキャンする** を「はい」にしてください。すべての `.cpp` がスキャン対象になり、依存グラフが正確になります。
>
> この設定を正式に扱うのは第18章です。

---

## 5.3 宣言と定義を分割する

### 5.3.1 `Cross()` を宣言だけにする

`Vector3.ixx` の `Cross` から、本体を取り除きます。

```cpp
// Vector3.ixx(該当部分)
export Vector3 Cross(const Vector3& a, const Vector3& b);   // ← 宣言だけ
```

ビルドしてください。**リンクエラーになります。**

```
error LNK2019: 未解決の外部シンボル
  "struct gamemath::Vector3 __cdecl gamemath::Cross(struct gamemath::Vector3 const &,
   struct gamemath::Vector3 const &)" が関数 main で参照されました
```

第3章 3.5.3 で見たのと同じ形のエラーです。ただし意味は違います。あのときは「別の実体を宣言してしまった」せいでしたが、今回は素直に「定義がない」だけです。

宣言はあるので `main.cpp` のコンパイルは通りました。しかしリンカが `Cross` の本体を探して、見つからなかったのです。

**この状態は正常です。** 定義をまだ書いていないのだから、当然です。

### 5.3.2 定義を実装単位に移す

`Vector3.cpp` に定義を書きます。

```cpp
// Vector3.cpp
module gamemath.vector;

namespace gamemath {

Vector3 Cross(const Vector3& a, const Vector3& b)
{
    return Vector3{
        a.y * b.z - a.z * b.y,
        a.z * b.x - a.x * b.z,
        a.x * b.y - a.y * b.x
    };
}

} // namespace gamemath
```

ビルドして実行してください。第4章と同じ出力になります。

```
a + b   = (5.0, 7.0, 9.0)
Cross   = (-3.0, 6.0, -3.0)
Dot     = 32.0
|a|^2   = 14.0
```

分割できました。

ここで4つ、気づいてほしいことがあります。

**`import` を書いていません。** `Vector3.cpp` は `Vector3` 型も `Cross` の宣言も使っていますが、`import gamemath.vector;` とは書いていません。これは 5.4 のテーマです。

**`#include` も書いていません。** ヘッダの世界なら `#include "Vector3.h"` が必要でした。

**`export` を書いていません。** インターフェイス側では `export Vector3 Cross(...)` でしたが、こちらは `export` なしです。これは 5.5 のテーマです。

**名前空間は書いています。** `namespace gamemath { }` は必要です。実装単位はモジュールに所属していますが、名前空間は別の話だからです。第4章 4.5 で確認したとおりです。

### 5.3.3 メンバ関数も移せる

クラスのメンバ関数も同じように分けられます。`LengthSquared` を移してみましょう。

```cpp
// Vector3.ixx(該当部分)
export struct Vector3
{
    float x;
    float y;
    float z;

    Vector3& operator+=(const Vector3& rhs) { /* そのまま */ }
    Vector3& operator-=(const Vector3& rhs) { /* そのまま */ }
    Vector3& operator*=(float s)            { /* そのまま */ }

    float LengthSquared() const;   // ← 宣言だけ
};
```

```cpp
// Vector3.cpp
module gamemath.vector;

namespace gamemath {

float Vector3::LengthSquared() const
{
    return x * x + y * y + z * z;
}

Vector3 Cross(const Vector3& a, const Vector3& b)
{
    // ...
}

} // namespace gamemath
```

`Vector3::LengthSquared` という書き方は、ヘッダと `.cpp` に分けるときとまったく同じです。

ビルドして、同じ出力になることを確認してください。

### 5.3.4 現時点の全体像

```
GameMath/
├── Vector3.ixx    ... export module gamemath.vector;   宣言 + 一部の定義
├── Vector3.cpp    ... module gamemath.vector;          定義
└── main.cpp       ... import gamemath.vector;
```

ヘッダ版の `Vector3.h` / `Vector3.cpp` / `main.cpp` と、見た目はそっくりです。しかし中身の性質は違います。次の2つの節で、その違いを見ていきます。

---

## 5.4 実装単位からはインターフェイスの中身が全部見える

### 5.4.1 `import` を書いていないのに、なぜ使えるのか

`Vector3.cpp` には `import` も `#include` もありません。それなのに `Vector3` 型が使えています。

理由はこうです。

**モジュール実装単位は、そのモジュールのプライマリインターフェイス単位を暗黙に `import` します。**

`module gamemath.vector;` と書いた時点で、`gamemath.vector` のインターフェイスは自動的に取り込まれています。書く必要がないどころか、書く場所もありません。

ヘッダの世界では、`Vector3.cpp` の先頭に `#include "Vector3.h"` が必要でした。これを書き忘れると、宣言と定義が食い違ってもコンパイルが通ってしまい、リンクエラーになる ── という定番の事故がありました。

モジュールではこの事故が起こりません。実装単位は必ずインターフェイスを見ているからです。

### 5.4.2 【実験】非公開の実体も見える

もう一歩踏み込みます。暗黙の `import` は、`export` されたものだけを持ち込むのでしょうか。それとも全部でしょうか。

`Vector3.ixx` に、**`export` を付けない**ヘルパー関数を追加してください。

```cpp
// Vector3.ixx(namespace gamemath の中)

// export なし = モジュールの外からは見えない
float AbsFloat(float v)
{
    return v < 0.0f ? -v : v;
}
```

これを `Vector3.cpp` から使ってみます。

```cpp
// Vector3.cpp
module gamemath.vector;

namespace gamemath {

float Vector3::LengthSquared() const
{
    return x * x + y * y + z * z;
}

Vector3 Cross(const Vector3& a, const Vector3& b)
{
    return Vector3{
        a.y * b.z - a.z * b.y,
        a.z * b.x - a.x * b.z,
        a.x * b.y - a.y * b.x
    };
}

// 非公開のヘルパーを、何も書かずに呼べる
float AbsSum(const Vector3& v)
{
    return AbsFloat(v.x) + AbsFloat(v.y) + AbsFloat(v.z);
}

} // namespace gamemath
```

ビルドしてください。**通ります。**

`AbsFloat` は `export` されていません。`main.cpp` からは見えません(試してみてください。「識別子が見つかりません」になります)。

しかし `Vector3.cpp` からは見えます。**同じモジュールの中だからです。**

第3章 3.5.4 で導入した**モジュールリンケージ**を思い出してください。`export` されていない実体は「同じモジュールの中だけで通用する名前」でした。`Vector3.cpp` は `gamemath.vector` モジュールの一部なので、その「中」に含まれます。

これは実務上とても便利です。ヘッダの世界で内部ヘルパーを共有したければ、`detail` 名前空間に入れて公開ヘッダに書くか、`.cpp` ごとに重複して書くしかありませんでした。モジュールなら、インターフェイス単位に `export` なしで書けば、実装単位から自由に使えて、外からは見えません。

**実験が終わったら、`AbsSum` を削除してください。** `AbsFloat` は 5.6 で使うので残しておきます。

### 5.4.3 見えるのはプライマリインターフェイスだけ

念のための補足です。暗黙に `import` されるのは、**そのモジュールのプライマリインターフェイス単位**だけです。

`Vector3.cpp` が `gamemath.matrix` を使いたければ、明示的に `import gamemath.matrix;` と書く必要があります。第13章で実際にそうします。

また、実装単位が複数ある場合、実装単位どうしはお互いを見ません。`Vector3Impl1.cpp` に書いた関数を `Vector3Impl2.cpp` から呼びたければ、宣言をインターフェイス単位(または後述する内部パーティション)に置く必要があります。共有の置き場所は、あくまでインターフェイス側です。

---

## 5.5 実装単位に `export` は書けない

### 5.5.1 やってみる

`Vector3.cpp` の `Cross` に `export` を付けてみてください。

```cpp
// Vector3.cpp
module gamemath.vector;

namespace gamemath {

export Vector3 Cross(const Vector3& a, const Vector3& b)   // ← export を追加
{
    // ...
}

}
```

ビルドすると、エラーになります。「実装単位で `export` は使えない」という趣旨のメッセージが出ます。

**`export` を消して、元に戻してください。**

### 5.5.2 なぜ禁止されているのか

一見すると不便に思えるかもしれません。しかし、これは非常に重要な制約です。

**モジュールのインターフェイスは、インターフェイス単位を読めば完全に分かる。**

この保証が成り立つからです。

実装単位で `export` が書けてしまったら、「このモジュールが何を公開しているか」を知るために、すべての実装単位を開いて確認しなければなりません。実装ファイルが10個あれば、10個全部です。インターフェイス単位という概念が意味をなくします。

ヘッダの世界を思い出してください。`Vector3.h` には宣言が並んでいましたが、`Vector3.cpp` に外部リンケージの関数を勝手に足すことは可能でした。それを他の `.cpp` から自分で宣言して呼ぶこともできました(第3章 3.6.5 で実際にやりました)。ヘッダは「公開しているものの一覧」ではなく、「公開していると作者が思っているものの一覧」でしかなかったのです。

モジュールでは、この曖昧さが言語仕様として排除されました。

### 5.5.3 では実装単位で定義したものは何になるのか

`Vector3.cpp` で `Cross` を定義しました。この `Cross` は、`export` されているのでしょうか。

されています。ただし、**`export` したのはインターフェイス単位のほう**です。

```cpp
// Vector3.ixx
export Vector3 Cross(const Vector3& a, const Vector3& b);   // ここで公開が決まる
```

```cpp
// Vector3.cpp
Vector3 Cross(const Vector3& a, const Vector3& b) { /* ... */ }   // 実装を提供するだけ
```

宣言の性質(公開かどうか)はインターフェイス単位で決まり、実装単位はそれに定義を与えるだけです。役割がはっきり分かれています。

逆に、インターフェイス単位に宣言がない関数を実装単位で定義したら、それは非公開(モジュールリンケージ)の関数になります。5.4.2 の `AbsSum` がそうでした。

---

## 5.6 この分割は `Vector3` に向いているか

道具の使い方は分かりました。ここからは、**使うべきかどうか**の話です。

### 5.6.1 インライン展開が失われる

`Cross` を実装単位に移したことで、失ったものがあります。

`main.cpp` をコンパイルするとき、コンパイラは `.ifc` を読みます。そこには `Cross` の**宣言**しかありません。本体は `Vector3.cpp` の `.obj` の中です。

つまり、コンパイラは `Cross(a, b)` という呼び出しを、**関数呼び出しとして生成するしかありません**。中身を展開して最適化することができないのです。

`Cross` をインターフェイス単位に書いていたときは違いました。本体が `.ifc` に入っているので、コンパイラは中身を見て、そのまま埋め込み、周辺のコードと一緒に最適化できました。

ゲームの内側のループで毎フレーム何万回も呼ばれる関数にとって、これは深刻です。`Cross` の中身は掛け算6回と引き算3回です。関数呼び出しのオーバーヘッド(引数のコピー、スタックフレーム、戻り値の受け渡し)のほうが、計算そのものより大きくなりかねません。

第1章 1.2.4 で、Pimpl イディオムについて「数学ライブラリにとって、インライン展開されることは機能要件の一部です」と書きました。同じことがここでも起きています。

> **リンク時最適化という逃げ道**
>
> [C/C++] → [最適化] → **プログラム全体の最適化**(`/GL`)と、[リンカー] → [最適化] → **リンク時のコード生成**(`/LTCG`)を有効にすると、リンカが `.obj` をまたいでインライン展開を試みます。実装単位に移した関数も展開される可能性があります。
>
> ただし、これは「可能性がある」であって保証ではありません。そしてビルド時間が大幅に伸びます。設計の失敗をビルドオプションで埋め合わせるのは、順序が逆です。

### 5.6.2 `Cross` を戻す

というわけで、`Cross` と `LengthSquared` をインターフェイス単位に戻します。

`Vector3.ixx` を第4章の状態に戻してください(`Cross` と `LengthSquared` に本体を書き戻す)。ただし、5.4.2 で追加した `AbsFloat` は残します。

```cpp
// Vector3.ixx
export module gamemath.vector;

namespace gamemath {

export struct Vector3
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

export Vector3 operator+(const Vector3& a, const Vector3& b)
{
    return Vector3{ a.x + b.x, a.y + b.y, a.z + b.z };
}

export Vector3 operator-(const Vector3& a, const Vector3& b)
{
    return Vector3{ a.x - b.x, a.y - b.y, a.z - b.z };
}

export Vector3 operator*(const Vector3& v, float s)
{
    return Vector3{ v.x * s, v.y * s, v.z * s };
}

export float Dot(const Vector3& a, const Vector3& b)
{
    return a.x * b.x + a.y * b.y + a.z * b.z;
}

export Vector3 Cross(const Vector3& a, const Vector3& b)
{
    return Vector3{
        a.y * b.z - a.z * b.y,
        a.z * b.x - a.x * b.z,
        a.x * b.y - a.y * b.x
    };
}

// 非公開のヘルパー
float AbsFloat(float v)
{
    return v < 0.0f ? -v : v;
}

} // namespace gamemath
```

`Vector3.cpp` は、いったんモジュール宣言だけに戻します。

```cpp
// Vector3.cpp
module gamemath.vector;
```

ビルドして、出力が変わらないことを確認してください。

### 5.6.3 では、実装単位に何を置くのか

`Vector3.cpp` を空のまま残すのは無駄です。ここに置くのがふさわしい関数を、1つ追加しましょう。

`Orthogonal()` ── 与えられたベクトルに垂直なベクトルを1つ返す関数です。カメラの姿勢を組み立てたり、法線から接空間の基底を作ったりするときに使います。

この関数には、`Cross` とは違う性質があります。

- **行数が多い。** 分岐が入るので10行以上になります
- **呼び出し頻度が低い。** 座標系を1回組み立てるときに呼ばれるだけで、内側のループには現れません
- **実装が変わりうる。** より数値的に安定な手法に差し替える余地があります

インライン展開されなくても困らず、実装が変わっても利用側を再コンパイルしたくない ── まさに実装単位に置くべき関数です。

`Vector3.ixx` に宣言を追加します。

```cpp
// Vector3.ixx(Cross の後ろに追加)

export Vector3 Orthogonal(const Vector3& v);
```

`Vector3.cpp` に定義を書きます。

```cpp
// Vector3.cpp
module gamemath.vector;

namespace gamemath {

Vector3 Orthogonal(const Vector3& v)
{
    // 最も成分の小さい軸との外積を取ると、退化しにくい
    const float ax = AbsFloat(v.x);
    const float ay = AbsFloat(v.y);
    const float az = AbsFloat(v.z);

    if (ax <= ay && ax <= az) {
        return Cross(v, Vector3{ 1.0f, 0.0f, 0.0f });
    }
    if (ay <= az) {
        return Cross(v, Vector3{ 0.0f, 1.0f, 0.0f });
    }
    return Cross(v, Vector3{ 0.0f, 0.0f, 1.0f });
}

} // namespace gamemath
```

5.4.2 で確認したとおり、非公開の `AbsFloat` も、公開されている `Cross` も、`import` を書かずに使えています。

`main.cpp` で確認します。

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
    const gamemath::Vector3 ort = gamemath::Orthogonal(a);

    std::printf("a + b      = (%.1f, %.1f, %.1f)\n", sum.x, sum.y, sum.z);
    std::printf("Cross      = (%.1f, %.1f, %.1f)\n", crs.x, crs.y, crs.z);
    std::printf("Dot        = %.1f\n", gamemath::Dot(a, b));
    std::printf("|a|^2      = %.1f\n", a.LengthSquared());
    std::printf("Orthogonal = (%.1f, %.1f, %.1f)  Dot(a,o) = %.1f\n",
                ort.x, ort.y, ort.z, gamemath::Dot(a, ort));

    return 0;
}
```

ビルドして実行してください。

```
a + b      = (5.0, 7.0, 9.0)
Cross      = (-3.0, 6.0, -3.0)
Dot        = 32.0
|a|^2      = 14.0
Orthogonal = (0.0, 3.0, -2.0)  Dot(a,o) = 0.0
```

`Dot(a, o)` が 0 になっていれば、確かに垂直です。

### 5.6.4 【実験】再コンパイルの範囲を確かめる

分割の効果を確認します。

`Vector3.cpp` の `Orthogonal` の中身を書き換えてください。コメントを1行足すだけでも構いません。

ビルドします。出力ウィンドウを見てください。

```
1>Vector3.cpp
```

**`main.cpp` は再コンパイルされていません。**

5.1.3 では、インターフェイス単位を触ると `main.cpp` まで巻き込まれました。実装単位を触っても `.ifc` は変わらないので、依存する側は無傷です。

これが分割の見返りです。

### 5.6.5 本書の方針

`GameMath` では、次の基準で置き場所を決めます。

**インターフェイス単位に定義を書く**

- 数行で終わる演算(演算子、`Dot`、`Cross`、成分アクセス)
- 内側のループで呼ばれるもの
- `constexpr` にしたいもの(第8章)
- テンプレート(第8章。そもそも選択の余地がありません)

**実装単位に定義を書く**

- 行数が多いもの(目安として10行以上)
- 呼び出し頻度が低いもの(初期化、変換、セットアップ)
- 実装が今後変わりそうなもの
- 重いヘッダを必要とするもの(次項)

数学ライブラリの場合、実際には**大半がインターフェイス単位に残ります**。これは他の分野のライブラリとかなり違う判断です。一般的なアプリケーションコードなら「原則として実装は隠す」でよいのですが、`GameMath` はインライン展開が命なので逆になります。

第14章の `Slerp`、第15章の衝突判定関数群は、実装単位の出番です。

### 5.6.6 実装単位のもう1つの用途

分割の動機として、ここまで「読みやすさ」と「再コンパイル範囲」を挙げました。実はもう1つ、より切実な用途があります。

**重いヘッダを、インターフェイスに持ち込まずに使える。**

`Orthogonal` を数値的により安定な実装にしたいとします。平方根が必要になり、`<cmath>` が要ります。このとき、`<cmath>` をインターフェイス単位に持ち込むのか、実装単位だけに閉じ込めるのか ── ここに選択肢が生まれます。

そしてこの選択には、第4章 4.6 で先送りにした問題が絡んできます。**そもそもモジュールのファイルで `#include` をどう書くのか**、という問題です。

次章のテーマです。

---

## 5.7 この章のまとめ

- `export` のない `module M;` で始まるファイルは**モジュール実装単位**になる
- 実装単位は `.ifc` を生成しない。拡張子は `.ixx` ではなく `.cpp` を使う
- プライマリモジュールインターフェイス単位は1モジュールに1つ。実装単位はいくつあってもよい
- **実装単位は、プライマリインターフェイス単位を暗黙に `import` する。** `import` も `#include` も書かない
- 暗黙の `import` では、`export` されていない実体も見える。同じモジュールの中だからである
- **実装単位に `export` は書けない。** モジュールが何を公開しているかは、インターフェイス単位だけを読めば分かる
- インターフェイス単位を変更すると、`import` している側は再コンパイルされる。実装単位を変更しても再コンパイルされない
- **しかし、実装単位に移した関数はインライン展開できなくなる。** 数学ライブラリでは致命的になりうる
- `GameMath` では、短くて頻繁に呼ばれるものはインターフェイスに残し、長くて呼び出し頻度の低いものを実装単位に置く

## 次章に向けて

ここまで、標準ライブラリを一度も使わずに来ました。`Length()` も `Normalized()` も書けないままです。

第6章では、この制約を外します。`.ixx` の中で `<cmath>` をどう扱うのか ── これには**グローバルモジュールフラグメント**という専用の構文があります。そして C++23 の `import std;` という、まったく別の選択肢もあります。

どちらを選ぶべきか、混ぜるとどうなるか、`GameMath` はどうするか。第6章を終えたとき、`Vector3` はようやくベクトルライブラリらしくなります。
