# 第6章 標準ライブラリをどう使うか

## この章について

第4章から、本書は自分に1つ制約を課してきました。**標準ライブラリを一切使わない**という制約です。

そのせいで `Vector3` には `Length()` がありません。`std::sqrt` が必要で、`std::sqrt` を使うには `<cmath>` が必要で、そして ── `.ixx` の中で `#include` をどう扱うかは、それ自体が1章分の話題だからです。

この章でその制約を外します。扱うのは次の4つです。

1. `.ixx` に素直に `#include` を書くと何が起きるか(6.1)
2. **グローバルモジュールフラグメント**という専用の領域(6.2、6.3)
3. C++23 の **`import std;`** という別の道(6.4)
4. 両者を混ぜると何が起きるか、そして `GameMath` はどちらを選ぶか(6.5、6.6)

3番目と4番目には、はっきりした「正解」がありません。移行期の C++ が抱えている現実的な問題で、プロジェクトの事情によって答えが変わります。本書は1つの立場を選びますが、なぜそう選ぶのかを説明したうえで、別の選択をしたい人のための道も用意します。

この章を終えると、`Vector3` はようやくベクトルライブラリらしくなります。

---

## 6.1 `#include <cmath>` を `.ixx` に書いてみる

### 6.1.1 `Length()` を書きたい

やりたいことは単純です。

```cpp
float Length() const
{
    return std::sqrt(LengthSquared());
}
```

`std::sqrt` を使うには `<cmath>` が必要です。素直に書いてみましょう。

### 6.1.2 素直に書く

`Vector3.ixx` の、モジュール宣言のすぐ後ろに `#include` を置きます。

```cpp
// Vector3.ixx
export module gamemath.vector;

#include <cmath>          // ← 追加

namespace gamemath {

export struct Vector3
{
    float x;
    float y;
    float z;

    // ...

    float LengthSquared() const
    {
        return x * x + y * y + z * z;
    }

    float Length() const          // ← 追加
    {
        return std::sqrt(LengthSquared());
    }
};

// ...
```

ビルドしてください。**警告が出ます。**

```
warning C5244: '#include <cmath>' in the purview of module 'gamemath.vector'
               appears erroneous. Consider moving that directive before the
               module declaration, or replace the textual inclusion with
               'import <cmath>;'.
```

日本語にすると、「モジュール `gamemath.vector` の**購買域**(purview)の中にある `#include <cmath>` は、誤りのように見えます。その指令をモジュール宣言の前に移すか、テキストによる取り込みを `import <cmath>;` に置き換えることを検討してください」となります。

**購買域**(purview)という耳慣れない語が出てきました。モジュール宣言より後ろの領域、つまり「そのモジュールに属する範囲」のことです。この章の鍵になる概念なので、6.1.5 で説明します。

### 6.1.3 警告の言うとおりに移してみる

警告は「モジュール宣言の前に移せ」と言っています。従ってみましょう。

```cpp
// Vector3.ixx
#include <cmath>                   // ← 前に移した

export module gamemath.vector;

namespace gamemath {
// ...
```

ビルドしてください。**今度は別の警告が出ます。**

```
warning C5201: a module declaration can appear only at the start of a
               translation unit unless a global module fragment is used.
```

「モジュール宣言は、**グローバルモジュールフラグメント**を使わない限り、翻訳単位の先頭にしか置けません」

第3章 3.2.2 で「モジュール宣言はファイルの最初の宣言でなければならない」と説明したとおりです。そして、そこで「`#include` を前に置きたい場合には専用の仕組みが必要で、第6章で扱う」と予告していました。それがこの**グローバルモジュールフラグメント**です。

コンパイラが答えを教えてくれました。C++ のエラーメッセージとしては珍しく親切です。

### 6.1.4 「警告」であることの危うさ

ここで一度立ち止まります。

C5244 も C5201 も、**エラーではなく警告**です。つまり、**ビルドは通ってしまいます。**

実際、この状態で実行してみると、おそらく正しく動きます。`std::sqrt` は計算され、`Length()` は正しい値を返すでしょう。

だからこそ危ないのです。「警告は出るけど動いているから」と放置すると、後で説明のつかない不具合を踏みます。しかもその不具合は、コードが増えて、モジュールが増えてから現れます。

なぜ危険なのか、理由を理解しておきましょう。

### 6.1.5 「所属」という考え方

C++20 のモジュールには、**宣言がどのモジュールに属するか**(attachment)という概念があります。

原則はこうです。

**モジュールの購買域(モジュール宣言より後ろ)に書かれた宣言は、そのモジュールに属する。**

そして第3章 3.5.4 で見たとおり、モジュールに属していて `export` されていない宣言は、**モジュールリンケージ**を持ちます。同じモジュールの中だけで通用する名前になります。

さて。`#include <cmath>` を購買域に書くと、何が起きるでしょうか。

`<cmath>` の中身は、プリプロセッサによってその場にテキスト展開されます。展開された何百もの宣言 ── `std::sqrt`、`std::sin`、`std::cos`、その他すべて ── が、**`gamemath.vector` モジュールに属することになります。**

つまり、`gamemath.vector` の中の `std::sqrt` は、他の翻訳単位が `#include <cmath>` して得る `std::sqrt` とは、**別の実体**になります。名前も引数も戻り値も同じですが、コンパイラにとっては別物です。

これは、第3章 3.5.3 で体験したことと同じ構図です。あのとき、`main.cpp` で自分で宣言した `Sub` は、モジュールの中の `Sub` とは別物になり、リンクエラーになりました。

具体的に何が起きるか、代表的なものを挙げます。

- **リンクエラー。** 定義が別の場所(標準ライブラリのバイナリ)にあるのに、モジュール内の宣言と結びつかない
- **型の不一致。** ヘッダで定義されたクラスを購買域で取り込むと、そのクラスは他所の同名クラスと別の型になる。関数に渡そうとすると型エラーになる
- **多重定義。** 2つのモジュールが同じヘッダを購買域で取り込むと、同じ実体が2つできる

`<cmath>` の場合、中身の多くが C 言語リンケージを持つ関数なので、たまたま問題が表面化しにくいのです。これが「動いてしまう」理由です。しかし `<vector>` や `<string>` のように C++ の型が中心のヘッダでは、遠慮なく壊れます。

**購買域に `#include` を書いてはいけません。** 警告が出たら、必ず直してください。

---

## 6.2 グローバルモジュールフラグメントを書く

### 6.2.1 `module;` を置く

やることは1行です。ファイルの先頭に、`module;` とだけ書きます。

```cpp
// Vector3.ixx
module;                          // ← これを追加

#include <cmath>

export module gamemath.vector;

namespace gamemath {
// ...
```

ビルドしてください。**警告なしで通ります。**

実行して、`Length()` を確認しましょう。`main.cpp` に1行足します。

```cpp
std::printf("|a|        = %.4f\n", a.Length());
```

```
|a|        = 3.7417
```

`sqrt(14) = 3.7417...` です。正しく動いています。

### 6.2.2 ファイルが3つの領域に分かれた

`module;` を書いたことで、`Vector3.ixx` は3つの領域を持つようになりました。

```cpp
module;                              // ┐
                                     // │ ① グローバルモジュールフラグメント
#include <cmath>                     // │
                                     // ┘
export module gamemath.vector;       // ② モジュール宣言

namespace gamemath {                 // ┐
                                     // │ ③ モジュールの購買域
    // ... コード ...                 // │
                                     // ┘
}
```

| 領域 | 呼び方 | 書けるもの | 所属 |
|---|---|---|---|
| ① | グローバルモジュールフラグメント | **プリプロセッサ指令だけ** | グローバルモジュール |
| ② | モジュール宣言 | ── | ── |
| ③ | 購買域(purview) | 通常の C++ コード | このモジュール |

**グローバルモジュールフラグメント**(global module fragment、GMF)は、`module;` から `export module ...;` までの領域です。

ここに書かれたものは、**グローバルモジュール**に属します。つまり、モジュールが存在しない普通の C++ の世界と同じ扱いになります。`std::sqrt` は `gamemath.vector` に属さず、他の翻訳単位が知っている `std::sqrt` とちゃんと同じ実体になります。

6.1.5 で説明した問題が、これで解決しました。

### 6.2.3 【実験】GMF の中身は公開されない

1つ確かめておきましょう。`Vector3.ixx` が `<cmath>` を取り込んだことで、`main.cpp` でも `std::sqrt` が使えるようになるのでしょうか。

`main.cpp` に書いてみてください。

```cpp
// main.cpp
#include <cstdio>

import gamemath.vector;

int main()
{
    std::printf("%f\n", std::sqrt(2.0f));   // ← 試してみる
    // ...
}
```

**エラーになります。** `std::sqrt` は見つかりません。

当然です。GMF の中身は `export` されていません。というより、`export` する方法がありません。GMF に書けるのはプリプロセッサ指令だけで、`export` は C++ の宣言に付けるものだからです。

第1章 1.3.1 で「依存が伝染しない」と書いた性質が、ここに現れています。ヘッダの世界では、`#include "Vector3.h"` した瞬間に `<cmath>` の中身がすべて降ってきました。モジュールでは降ってきません。`main.cpp` が `std::sqrt` を使いたければ、`main.cpp` が自分で `<cmath>` を取り込みます。

**この行は削除してください。**

---

## 6.3 グローバルモジュールフラグメントの厳密なルール

便利な仕組みですが、制約が厳しいので、正確に押さえておきます。

### 6.3.1 ルール1 ── `module;` はファイルの最初のトークン

`module;` の前に置けるのは、**コメントと空白だけ**です。

```cpp
// Copyright (c) 2026  ← OK。コメントはトークンではない

module;
#include <cmath>
export module gamemath.vector;
```

```cpp
#pragma once   // ← NG。これはトークン

module;
```

トークンが1つでも前にあると、コンパイラは「このファイルには GMF がない」と判断します。すると `module;` は宣言として解釈され、意味不明なエラーになります。

「あるはずの GMF が認識されない」というトラブルの原因は、たいていこれです。ファイルの先頭に何か紛れ込んでいないか確認してください。

### 6.3.2 ルール2 ── 書けるのはプリプロセッサ指令だけ

GMF の中に C++ のコードは書けません。

```cpp
module;

#include <cmath>          // OK
#define GAMEMATH_FAST 1   // OK
#ifdef _WIN32             // OK
#include <intrin.h>       // OK
#endif                    // OK

int x = 0;                // ← NG。宣言は書けない

export module gamemath.vector;
```

`#include`、`#define`、`#if` / `#ifdef` / `#endif`、`#pragma` ── プリプロセッサの世界のものだけです。

これは制約であると同時に、GMF の役割をはっきりさせています。**GMF は「モジュールの外の世界から、必要なものを持ち込むための入口」**であって、コードを書く場所ではありません。

### 6.3.3 ルール3 ── 公開もされず、伝播もしない

6.2.3 で確認したとおりです。念のため整理します。

- GMF で取り込んだ宣言は、`import` した側から**見えません**
- GMF で定義したマクロも、`import` した側に**漏れません**

2番目は重要です。第3章 3.6.4 で「名前付きモジュールはマクロをエクスポートしない、例外はない」と書きました。GMF に書いたマクロも例外ではありません。

つまり、`Windows.h` を GMF で取り込んでも、`min` / `max` マクロが利用者に漏れることはありません。第1章 1.1.5 で見た問題が、この仕組みで封じ込められます。

### 6.3.4 ルール4 ── 必要なものだけが `.ifc` に入る

心配になった人がいるかもしれません。「GMF で `<cmath>` を取り込んだら、`.ifc` に `<cmath>` 全部が入って巨大になるのでは?」

なりません。コンパイラは、**モジュールが公開している宣言から到達できるものだけ**を `.ifc` に記録します。使っていない宣言は捨てられます。

`Vector3` は `std::sqrt` しか使っていないので、`.ifc` に持ち込まれるのはその周辺だけです。`std::sin` も `std::cos` も入りません。

実際に確かめられます。`Vector3.ixx` に `module;` と `#include <cmath>` を足す前後で、`gamemath.vector.ifc` のサイズを比べてみてください。第2章 2.4.1 の手順です。増えてはいますが、`<cmath>` 全体の量ではないはずです。

### 6.3.5 実装単位にも書ける

GMF はインターフェイス単位専用ではありません。実装単位にも書けます。

```cpp
// Vector3.cpp
module;

#include <cmath>

module gamemath.vector;

namespace gamemath {
// ...
}
```

第5章 5.6.6 で予告したことが、これで可能になりました。**重いヘッダを、インターフェイスに持ち込まずに使う**という選択ができます。

- インターフェイス単位の GMF に書く → `.ifc` に反映される。`import` する側のコンパイル時間に影響しうる
- 実装単位の GMF に書く → `.ifc` に影響しない。完全に閉じ込められる

インターフェイス側で必要なもの(`Length()` のための `std::sqrt` など)だけを GMF に置き、それ以外は実装単位に追いやる ── これが基本的な設計判断になります。

### 6.3.6 GMF のまとめ

| 項目 | 内容 |
|---|---|
| 書き方 | ファイル先頭に `module;`、モジュール宣言まで |
| 前に置けるもの | コメントと空白だけ |
| 中に書けるもの | プリプロセッサ指令だけ |
| 所属 | グローバルモジュール(モジュールに属さない) |
| 公開 | されない。マクロも漏れない |
| `.ifc` への影響 | 使われている宣言だけが持ち込まれる |
| 書ける場所 | インターフェイス単位、実装単位のどちらでも |

---

## 6.4 `import std;` を使う

GMF は、C++20 が用意した「ヘッダと共存するための仕組み」です。既存のヘッダをモジュールから使うには、これしかありません。

しかし標準ライブラリに限っては、C++23 でもっと素直な方法が入りました。

### 6.4.1 書き換える

`Vector3.ixx` を、こう書き換えてください。

```cpp
// Vector3.ixx
export module gamemath.vector;

import std;                      // ← GMF の代わりに

namespace gamemath {
// ...(以下そのまま)
```

`module;` と `#include <cmath>` の2行が消え、`import std;` の1行になりました。

ビルドしてください。通ります。実行結果も同じです。

**動かない場合** ── 第2章 2.3.3 の設定を確認してください。`import std;` には次の2つが**両方**必要です。

- C++ 言語標準が `/std:c++latest`
- **ISO C++23 標準ライブラリ モジュールのビルド** が「はい」

### 6.4.2 `import` を書く場所

`import std;` は、**モジュール宣言の直後**に書きます。この領域を**モジュールプリアンブル**(module preamble)と呼びます。

```cpp
export module gamemath.vector;   // モジュール宣言

import std;                      // ┐
import gamemath.matrix;          // ┤ モジュールプリアンブル
                                 // ┘
namespace gamemath {             // ここからコード
```

モジュールのファイルでは、`import` はプリアンブルにまとめて書く必要があります。コードの途中に書くことはできません。

第3章 3.4.4 で「`.ixx` の中では `import` を書ける場所に制約がある」と予告したのが、これです。普通の `.cpp` ではもう少し自由ですが、揃えておくほうが読みやすいので、本書では常にプリアンブルに書きます。

### 6.4.3 実装単位にも書く

`Vector3.cpp` も書き換えます。第5章の `Orthogonal` で、自前の `AbsFloat` の代わりに `std::abs` を使います。

```cpp
// Vector3.cpp
module gamemath.vector;

import std;                      // ← 必要

namespace gamemath {

Vector3 Orthogonal(const Vector3& v)
{
    const float ax = std::abs(v.x);
    const float ay = std::abs(v.y);
    const float az = std::abs(v.z);

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

**実装単位にも `import std;` が要ります。**

第5章 5.4 で「実装単位はインターフェイス単位を暗黙に `import` する」と説明しました。しかし暗黙に持ち込まれるのは**インターフェイス単位に書かれた宣言**であって、インターフェイス単位が `import` したものまでは引き継がれません。

`import` は伝播しません。**各モジュール単位は、自分が必要とするものを自分で `import` します。**

これに伴い、第5章で自前で書いた `AbsFloat` は不要になりました。`Vector3.ixx` から削除してください。

> 非公開のヘルパーをインターフェイス単位に置き、実装単位から使う ── という技法自体は有効です。第10章で内部パーティションを扱うとき、より大きな規模で再登場します。

### 6.4.4 `std` と `std.compat`

標準ライブラリのモジュールは2つあります。

| モジュール | 提供するもの |
|---|---|
| `std` | `std::` 名前空間の中身。C 由来の関数も `std::printf` のように `std::` 付きで提供 |
| `std.compat` | `std` の全部に加えて、グローバル名前空間の `::printf`、`::size_t`、`::strlen` など |

違いは、C 由来の関数をグローバル名前空間にも置くかどうかです。

`import std;` では `std::printf` は使えますが、`::printf` は使えません。ヘッダの世界では `#include <cstdio>` してもグローバル名前空間に `printf` が漏れてくるのが普通でしたが、その曖昧さが解消されています。

既存コードが `::printf` や `::strlen` を大量に使っているなら `std.compat` が移行を楽にします。新しく書くなら `std` で十分です。

**`GameMath` では `import std;` を使います。**

### 6.4.5 利用側も揃える

`main.cpp` も `import std;` にしましょう。C++23 の `std::println` が使えるので、書式もすっきりします。

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
    const gamemath::Vector3 ort = gamemath::Orthogonal(a);
    const gamemath::Vector3 nrm = a.Normalized();

    std::println("a + b      = ({:.2f}, {:.2f}, {:.2f})", sum.x, sum.y, sum.z);
    std::println("Cross      = ({:.2f}, {:.2f}, {:.2f})", crs.x, crs.y, crs.z);
    std::println("Dot        = {:.2f}", gamemath::Dot(a, b));
    std::println("|a|        = {:.4f}", a.Length());
    std::println("|a|^2      = {:.2f}", a.LengthSquared());
    std::println("Normalized = ({:.4f}, {:.4f}, {:.4f})  |n| = {:.4f}",
                 nrm.x, nrm.y, nrm.z, nrm.Length());
    std::println("Orthogonal = ({:.2f}, {:.2f}, {:.2f})  Dot = {:.2f}",
                 ort.x, ort.y, ort.z, gamemath::Dot(a, ort));

    return 0;
}
```

`Normalized()` はまだ書いていないので、次の 6.6.3 で追加します。いったんその2行を外してビルドし、動くことを確認してもおきましょう。

`#include <cstdio>` が消えました。この `main.cpp` には、プリプロセッサ指令が1行もありません。

### 6.4.6 GMF 版と何が違うのか

同じ `Length()` を実現する2つの方法を並べます。

```cpp
// GMF 版
module;
#include <cmath>
export module gamemath.vector;
```

```cpp
// import std; 版
export module gamemath.vector;
import std;
```

| | GMF + `#include` | `import std;` |
|---|---|---|
| 必要な規格 | C++20 | C++23 |
| Visual Studio の設定 | 不要 | `/std:c++latest` + 標準ライブラリモジュールのビルド |
| 取り込む量 | 指定したヘッダだけ | 標準ライブラリ全体 |
| 初回ビルド | 速い | 遅い(`std.ifc` を作る) |
| 2回目以降 | ヘッダを毎回解析 | `std.ifc` を読むだけ |
| マクロ汚染 | GMF 内には持ち込まれる | 起きない |
| 標準以外のヘッダ | これしかない | 使えない |

「取り込む量」の行は、直感に反するかもしれません。`import std;` は標準ライブラリを丸ごと取り込むので、`<cmath>` だけが欲しい場合には過剰に見えます。

しかし実測すると、`import std;` のほうが速いことがほとんどです。理由は第1章 1.3.3 の掛け算の話です。`std.ifc` は一度作れば全翻訳単位で使い回されますが、`#include <cmath>` は取り込んだ翻訳単位の数だけ解析されます。

そして `<cmath>` だけで済むことは、実際にはあまりありません。`<vector>`、`<algorithm>`、`<span>`、`<format>` ── 使うヘッダが増えるほど、`import std;` の一括方式が有利になります。

---

## 6.5 `import std;` と `#include` の混在

### 6.5.1 Microsoft の公式な注意

標準ライブラリのモジュールについて、Microsoft のドキュメントには明確な注意書きがあります。

**ヘッダユニットと名前付きモジュールを混ぜないこと。** たとえば、同じファイルの中で `import <vector>;` と `import std;` を併用してはいけません。

(`import <vector>;` という書き方 ── ヘッダユニット ── については第18章で扱います。ここでは「そういう書き方がある」とだけ知っておいてください。)

同じ危うさは、`#include <vector>` と `import std;` の併用にもあります。

### 6.5.2 なぜ危ないのか

理屈は 6.1.5 と同じ「所属」の話です。

1つの翻訳単位に、同じ実体が**2つの経路**から入ってきます。

- `#include <vector>` ── テキスト展開されて、グローバルモジュールに属する `std::vector`
- `import std;` ── `std.ifc` から読み込まれた、モジュール `std` に属する `std::vector`

コンパイラはこの2つを同一視できるはずですが、実装の細部で食い違いが生じます。テンプレートの実体化、`constexpr` の評価、デフォルト引数、属性 ── どこか1か所でも扱いが違えば、コンパイルエラーになります。

しかも、エラーメッセージが標準ライブラリの内部を指すので、原因にたどり着くのが困難です。

**動いてしまうこともあります。** しかしそれは運です。ヘッダを1つ足しただけで、あるいはコンパイラを更新しただけで壊れます。

**1つの翻訳単位の中では、どちらか一方に統一してください。**

### 6.5.3 これは翻訳単位ごとの話である

一方で、過度に恐れる必要もありません。**混在の問題は、1つの翻訳単位の中で起きる話です。**

- `Vector3.ixx` が `import std;` を使う
- `main.cpp` が `#include <cstdio>` を使う

これは問題になりません。別の翻訳単位だからです。

`import gamemath.vector;` は、`gamemath.vector` が公開している宣言だけを持ち込みます。`Vector3.ixx` の中の `import std;` は再エクスポートされていないので、`main.cpp` には流れ込みません。

`GameMath` を配布したとき、利用者が `#include` 派であっても構わない ── ということです。

### 6.5.4 ライブラリ作者としての設計指針

ただし、条件が1つあります。

**公開インターフェイスに標準ライブラリの型を出さないこと。**

たとえば `GameMath` が、こういう関数を公開したとします。

```cpp
export std::vector<Vector3> GenerateSphere(int segments);
```

このとき `std::vector` は、利用者の翻訳単位から**到達可能**になります。利用者が `#include <vector>` していれば、6.5.2 の危うさが再現します。

`GameMath` の公開インターフェイスは、いまのところ `float` と `Vector3` だけでできています。標準ライブラリの型は1つも出てきません。だから利用者は、`#include` 派でも `import std;` 派でも自由です。

これは偶然ではなく、数学ライブラリとして自然な設計です。そして移行期の C++ では、この「標準ライブラリの型を公開インターフェイスに出さない」という設計が、思わぬ形で利用者を助けます。

第15章で衝突判定を扱うとき、「交差した点の一覧」をどう返すかで、この判断を迫られます。

---

## 6.6 本書での方針を決める

### 6.6.1 方針

`GameMath` では、次のように決めます。

**1. 標準ライブラリは `import std;` で取り込む**

インターフェイス単位でも実装単位でも、モジュール宣言の直後に `import std;` と書きます。

**2. 標準ヘッダを `#include` しない**

同じ翻訳単位に `import std;` と `#include <vector>` を混在させません。

**3. グローバルモジュールフラグメントは、`import` できないヘッダのために取っておく**

`<immintrin.h>`(SIMD 組み込み関数)や `<Windows.h>` のような、標準ライブラリではないヘッダは `import std;` では取り込めません。これらは GMF を使います。第16章と第22章で登場します。

**4. 公開インターフェイスに標準ライブラリの型を出さない**

利用者の環境を選ばないライブラリにするためです。

### 6.6.2 `/std:c++20` で進めたい場合

業務プロジェクトの都合で `/std:c++latest` を使えない読者もいるでしょう。第2章 2.2.3 のコラムで書いたとおり、`c++latest` は「まだ確定していない機能も含む」設定なので、これを避けるのは正当な判断です。

その場合は、`import std;` の代わりに GMF を使ってください。本書のコードは、次の置き換えでそのまま動きます。

```cpp
// import std; 版(本書の標準)
export module gamemath.vector;
import std;
```

```cpp
// GMF 版(C++20 でも動く)
module;
#include <cmath>
export module gamemath.vector;
```

必要なヘッダは、章ごとに次のとおりです。

| 章 | 必要なヘッダ |
|---|---|
| 第6章〜第13章 | `<cmath>` |
| 第14章 | `<cmath>` |
| 第15章 | `<cmath>`、`<optional>` |
| 第16章 | `<immintrin.h>`(こちらは GMF 必須) |

第16章以降は、どちらの方針でも GMF が必要になります。

### 6.6.3 完成した `Vector3`

方針が決まったので、`Normalized()` を追加して、この章の成果物を完成させます。

```cpp
// Vector3.ixx
export module gamemath.vector;

import std;

namespace gamemath {

// 非公開の定数
constexpr float kEpsilon = 1e-6f;

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

    float Length() const
    {
        return std::sqrt(LengthSquared());
    }

    Vector3 Normalized() const
    {
        const float len = Length();
        if (len <= kEpsilon) {
            return Vector3{ 0.0f, 0.0f, 0.0f };
        }
        const float inv = 1.0f / len;
        return Vector3{ x * inv, y * inv, z * inv };
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

export Vector3 Orthogonal(const Vector3& v);

} // namespace gamemath
```

`kEpsilon` に `export` を付けていないことに注意してください。これは `Normalized()` の内部事情であって、利用者が知る必要のない値です。第3章 3.5.4 のモジュールリンケージが効いており、`main.cpp` から `gamemath::kEpsilon` と書くことはできません。試してみてください。

ビルドして実行します。

```
a + b      = (5.00, 7.00, 9.00)
Cross      = (-3.00, 6.00, -3.00)
Dot        = 32.00
|a|        = 3.7417
|a|^2      = 14.00
Normalized = (0.2673, 0.5345, 0.8018)  |n| = 1.0000
Orthogonal = (0.00, 3.00, -2.00)  Dot = 0.00
```

正規化したベクトルの長さが 1.0000 になっていれば成功です。

第4章から続いた「標準ライブラリなし」の制約は、これで終わりです。

---

## 6.7 この章のまとめ

- モジュール宣言より後ろの領域を**購買域**(purview)と呼ぶ。ここに書かれた宣言は、そのモジュールに**属する**
- 購買域に `#include` を書くと、取り込んだ宣言がすべてそのモジュールに属してしまい、他所の同名の実体と別物になる。MSVC は **C5244** で警告する
- **警告であってエラーではない。** ビルドは通ってしまうので、放置すると後で壊れる
- **グローバルモジュールフラグメント**(`module;` から モジュール宣言まで)に書けば、取り込んだものはグローバルモジュールに属し、正しく扱われる
- GMF の前に置けるのはコメントと空白だけ。中に書けるのはプリプロセッサ指令だけ
- GMF の中身は公開されず、マクロも漏れない。`.ifc` には必要な宣言だけが持ち込まれる
- GMF はインターフェイス単位にも実装単位にも書ける。実装単位に置けば `.ifc` に影響しない
- **`import std;`** は C++23 の機能。`/std:c++latest` と「ISO C++23 標準ライブラリ モジュールのビルド」の両方が必要
- `import` はモジュールプリアンブル(モジュール宣言の直後)に書く
- **`import` は伝播しない。** 実装単位にも自分で `import std;` を書く
- 1つの翻訳単位の中で `import std;` と `#include <標準ヘッダ>` を混ぜない
- ただしこれは翻訳単位ごとの話。ライブラリが `import std;` を使っていても、利用者は `#include` 派で構わない
- そのためには、**公開インターフェイスに標準ライブラリの型を出さない**設計が有効

## 次章に向けて

ここまでで、モジュールの基本的な道具はひととおり揃いました。`export module`、`import`、実装単位、グローバルモジュールフラグメント ── これだけで、実用的なモジュールが書けます。

第7章は、少し性質の違う章です。新しい構文は出てきません。代わりに、**「見える(visible)」と「到達可能(reachable)」の違い**を扱います。

第3章 3.5.5 で予告し、第5章でも触れた区別です。地味なテーマですが、ここを飛ばすと、第8章以降で出会うコンパイルエラーがまったく読めなくなります。

- `export` していない型が、公開関数の戻り値に現れたらどうなるか
- なぜテンプレートは特別扱いが必要なのか
- 「エラーにならないのに、なぜか使えない」のはどういうときか

本書で最も丁寧に書く章です。急がずに読んでください。
