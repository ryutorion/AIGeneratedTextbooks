# 第20章 ビルドとテスト

## この章について

`GameMath` は、Visual Studio のプロジェクトとして動いています。しかし実務では、それだけでは足りません。

- CI サーバーでビルドしたい
- 別のプラットフォームでもビルドしたい
- テストを自動で走らせたい
- 他のチームに配布したい

この章では、そのための土台を整えます。

1. MSBuild がモジュールをどうビルドしているのか(20.1)
2. **CMake + Ninja** でビルドする(20.2)
3. テストプロジェクトから `import` する(20.3)
4. **ライブラリとして配布する**(20.4)
5. パッケージマネージャの現状(20.5)

4番目に、モジュールのいちばん実務的な難所があります。第1章 1.4.3 で書いたとおり、**`.ifc` は配布物ではありません。** では何を配布するのか ── その答えは、思ったより素直です。

そして 20.3 では、第14章 14.4.5 で保留にした「モジュールでは公開 API しかテストできない」という問題を、実際のテストコードとして形にします。

---

## 20.1 MSBuild でのモジュールのビルド順序とスキャン

### 20.1.1 何が起きているのか

第3章 3.4.5 で、出力ウィンドウのビルド順序を確認しました。

```
1>Vector.ixx
1>Matrix.ixx
1>main.cpp
```

これがどう決まっているのかを、改めて整理します。

ヘッダの時代、`.cpp` ファイルは互いに独立していました。どんな順序でも、何個同時でも構いませんでした。

モジュールでは、順序に制約があります。`main.cpp` をコンパイルするには `gamemath.ifc` が必要で、`gamemath.ifc` を作るには `gamemath.core.ifc` が必要で ── という連鎖があります(第1章 1.3.3)。

**この連鎖を、ビルドシステムが把握していなければなりません。**

### 20.1.2 スキャンという前段階

そのために、MSBuild はコンパイルの前に**スキャン**を行います。

```
[1] スキャン        ソースを読み、どのモジュールを提供し、どのモジュールを必要とするかを調べる
        ↓
[2] 依存グラフの構築  誰が誰より先か、を決める
        ↓
[3] コンパイル       順序に従って、可能な範囲で並列に
        ↓
[4] リンク
```

第2章 2.3.2 で確認した設定が、ここで効いています。

**モジュール依存関係のソースをスキャンする**(`/scanDependencies`)

- 「いいえ」 ── `.ixx` とヘッダユニット指定のファイルだけをスキャン
- 「はい」 ── **すべての C++ ソース**をスキャン

第18章 18.2.3 で「はい」に変更しました。実装単位(`module gamemath.core;` で始まる `.cpp`)が5つになった `GameMath` では、この設定が必要です。

### 20.1.3 本書で触れた設定の総まとめ

ここまでに登場したプロジェクト設定を、一覧にしておきます。

| 場所 | 項目 | `GameMath` の値 | 章 |
|---|---|---|---|
| C/C++ → 言語 | C++ 言語標準 | `/std:c++latest` | 2.2.3 |
| C/C++ → 言語 | ISO C++23 標準ライブラリ モジュールのビルド | はい | 2.3.3 |
| C/C++ → 言語 | 準拠モード | はい (`/permissive-`) | 13.4.7 |
| C/C++ → 全般 | モジュール依存関係のソースをスキャンする | **はい** | 2.3.2 / 18.2.3 |
| C/C++ → 全般 | インクルードをインポートに変換する | いいえ | 18.2.5 |
| C/C++ → 出力ファイル | モジュール出力ファイル名 | 既定 | 2.3.4 |
| C/C++ → 全般 | 追加のモジュール依存関係 (`/reference`) | 既定 | 20.3.1 |
| C/C++ → 全般 | 追加のヘッダー ユニット依存関係 (`/headerUnit`) | 既定 | 18.2 |

**ファイル単位の設定**

| ファイル | コンパイル言語の選択 | 章 |
|---|---|---|
| `*.ixx`(インターフェイス) | 自動(`/interface`) | 2.3.1 |
| `Core/Detail.ixx`(内部パーティション) | **`/internalPartition`** | 10.2.2 |
| `*.cpp`(実装単位) | 既定 | 5.2.5 |

`Detail.ixx` だけが手動設定を必要とします。**移行やプロジェクト再作成のときに、いちばん忘れられる項目です。**

### 20.1.4 並列ビルドとの関係

`/MP`(複数プロセッサによるコンパイル)を有効にしている場合、依存グラフの形が並列度を決めます。

```
        :vector          ← 最初に1つだけ。並列度 1
        ↙   ↓   ↘
  :detail :matrix :quaternion   ← 3つ同時。並列度 3
        ↘   ↓   ↙
        :transform       ← 並列度 1
```

**依存が深いほど、並列度は下がります。** 第1章 1.3.3 で「モジュールが持ち込む新しいコスト」として挙げたのが、これです。

逆に言えば、**土台を薄く、その上を横に広げる**構造にすると、並列度が上がります。第12章 12.1.5 の指針にはありませんでしたが、実は設計の判断材料になります。

とはいえ、これは最適化の話です。まず正しい依存関係を作り、遅ければ形を見直す ── という順序で構いません。第21章で計測します。

### 20.1.5 MSBuild の限界

Visual Studio の中で開発するぶんには、MSBuild で不自由はありません。

しかし、次のような場面では限界が来ます。

- **Linux や macOS でもビルドしたい**
- **CI で GCC や Clang も走らせたい**
- **ビルドの設定をバージョン管理で読みやすく保ちたい**(`.vcxproj` は XML で差分が読みにくい)
- **プロジェクトファイルを手で管理したくない**

そこで CMake です。

---

## 20.2 CMake + Ninja でモジュールをビルドする

### 20.2.1 必要なバージョン

モジュールの CMake 対応は、**バージョン 3.25 で実験的に導入され、3.28 で正式化されました。**

| 必要なもの | バージョン |
|---|---|
| CMake | **3.28 以上** |
| Ninja(Ninja ジェネレータを使う場合) | **1.11 以上** |
| MSVC | VS 2022 17.4 以上 |

第2章 2.1.3 で「Windows 用 C++ CMake ツール」を入れておくよう書きました。ここで使います。入っていなければ、Visual Studio Installer から追加してください。

CMake のバージョンは、コマンドプロンプトで確認できます。

```
cmake --version
```

### 20.2.2 `GameMath` の `CMakeLists.txt`

プロジェクトのルートに `CMakeLists.txt` を作ってください。

```cmake
cmake_minimum_required(VERSION 3.28)

project(GameMath LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# ───────── ライブラリ ─────────

add_library(gamemath)

target_sources(gamemath
    PUBLIC
        FILE_SET CXX_MODULES FILES
            GameMath.ixx
            Core/Core.ixx
            Core/Vector.ixx
            Core/Basis.ixx
            Core/Detail.ixx
            Core/Matrix.ixx
            Core/Quaternion.ixx
            Core/Transform.ixx
            Geometry/Geometry.ixx
            Geometry/Shapes.ixx
            Geometry/Intersect.ixx
            Random/Random.ixx
            Debug/Debug.ixx
    PRIVATE
            Core/Basis.cpp
            Core/Matrix.cpp
            Core/Quaternion.cpp
            Core/Transform.cpp
            Geometry/Intersect.cpp
            Debug/Debug.cpp
)

# ───────── サンプル ─────────

add_executable(gamemath_sample main.cpp)
target_link_libraries(gamemath_sample PRIVATE gamemath)
```

### 20.2.3 `FILE_SET CXX_MODULES` の意味

新しいのは、この部分だけです。

```cmake
target_sources(gamemath
    PUBLIC
        FILE_SET CXX_MODULES FILES
            GameMath.ixx
            ...
)
```

**「これらのファイルはモジュールインターフェイスを提供する」**と CMake に伝えています。

CMake は、この指定を受けて次のことを行います。

1. **これらのファイルをスキャンする。** どのモジュールを提供し、何を必要とするかを調べます
2. **依存グラフを構築し、ビルド順序を決める**
3. **コンパイラごとの適切なオプションを付ける**(MSVC なら `/interface`)
4. **`import` する側に `.ifc` の場所を教える**(MSVC なら `/reference`)

MSBuild が自動でやっていたことを、CMake も自動でやります。**手順が明示的になっただけです。**

`PUBLIC` にしている理由は 20.4 で説明します。配布に関わります。

**実装単位は `PRIVATE` に置く**

`Basis.cpp` などの実装単位は、`FILE_SET CXX_MODULES` には**含めません**。通常のソースファイルとして `PRIVATE` に置きます。

実装単位は `.ifc` を生成しないからです(第5章 5.2.5)。CMake が「モジュールを提供するファイル」として扱うのは、インターフェイス単位と内部パーティションだけです。

ただし、実装単位も**スキャンの対象にはなります**。`module gamemath.core;` と書いてあるので、依存関係の解決が必要だからです。CMake は C++20 以上のターゲットでは、これを自動的に行います。

> **内部パーティションについて**
>
> `Core/Detail.ixx` は内部パーティションです(第10章)。MSVC では `/internalPartition` が必要でした。
>
> CMake がこのオプションを自動的に付けるかどうかは、バージョンによって挙動が変わる可能性があります。**ビルドが失敗する場合は、ソースファイルのプロパティで明示してください。**
>
> ```cmake
> set_source_files_properties(Core/Detail.ixx
>     PROPERTIES COMPILE_OPTIONS "/internalPartition")
> ```
>
> ただしこれは MSVC 専用のオプションなので、他のコンパイラでは条件分岐が必要になります。**内部パーティションは、クロスプラットフォームのビルドで最も扱いにくい要素です。**

### 20.2.4 `import std;` はまだ実験的

ここに大きな注意があります。

**CMake における `import std;` の対応は、まだ実験的な段階です。**

CMake のドキュメントによれば、`import std;` を使うには次が必要です。

- **`CMAKE_EXPERIMENTAL_CXX_IMPORT_STD` というゲート変数**に、特定の UUID を設定する
- ターゲットの **`CXX_MODULE_STD` プロパティ**を `ON` にする
- ターゲットが **C++23 以上**であること

そして、決定的な制約があります。

> 現時点では **Ninja ジェネレータだけが `import std;` に対応している。** Visual Studio ジェネレータは、インポートされたターゲットの BMI をビルドできないため。

つまり、`cmake -G "Visual Studio 17 2022"` では `import std;` が使えません。**Ninja を使う必要があります。**

さらに、**ゲート変数の UUID は CMake のバージョンごとに変わります。** 「まだ安定していないので、意識的に有効化した人だけが使ってください」という仕組みです。古い記事の UUID をコピーしても動きません。

**現在の値は、使用する CMake バージョンの `cmake-cxxmodules(7)` ドキュメントで確認してください。**

```cmake
# 値はバージョンごとに異なる。ドキュメントで確認すること
set(CMAKE_EXPERIMENTAL_CXX_IMPORT_STD "<ドキュメントに記載された UUID>")

set(CMAKE_CXX_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_library(gamemath)
set_target_properties(gamemath PROPERTIES CXX_MODULE_STD ON)
```

> **第6章の方針が試される場面**
>
> 第6章 6.6.2 で、`/std:c++20` を使いたい読者のために「GMF + `#include <cmath>`」への置き換え表を用意しました。
>
> **CMake でクロスプラットフォームにビルドしたいなら、その選択肢を真剣に検討する価値があります。** `import std;` を諦めれば、CMake の実験的機能に依存せず、Visual Studio ジェネレータも使えます。
>
> 「新しい機能を使う」ことと「どこでもビルドできる」ことは、しばしば両立しません。プロジェクトの事情で決めてください。

### 20.2.5 ビルドしてみる

Developer Command Prompt for VS 2026 から実行します。

```
cd <プロジェクトのフォルダー>

cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Release
cmake --build build
```

`build/gamemath_sample.exe` ができれば成功です。

`-G Ninja` を `-G "Visual Studio 17 2022"` に変えれば、`.sln` が生成されます。ただし 20.2.4 の制約により、`import std;` を使っている場合は Ninja が必要です。

**CMake は Visual Studio の設定を置き換えるものではありません。** 両方を並行して維持することも可能です。日常の開発は Visual Studio、CI は CMake + Ninja ── という構成は現実的です。

ただし、**設定が二重になる**ので、片方だけ変更して食い違う事故が起きます。どちらかを正とすると決めておいてください。

---

## 20.3 テストプロジェクトから `GameMath` を import する

### 20.3.1 テストを追加する

`CMakeLists.txt` に、テスト用の実行ファイルを追加します。

```cmake
# ───────── テスト ─────────

enable_testing()

add_executable(gamemath_tests Tests/TestMain.cpp)
target_link_libraries(gamemath_tests PRIVATE gamemath)

add_test(NAME gamemath_tests COMMAND gamemath_tests)
```

`target_link_libraries` を書くだけで、テスト側から `import gamemath;` が使えるようになります。CMake が `.ifc` の場所をコンパイラに伝えてくれます。

> **Visual Studio でテストプロジェクトを作る場合**
>
> ソリューションに実行可能プロジェクトを追加し、**[プロジェクト] → [参照の追加]** で `GameMath` プロジェクトを参照します。
>
> プロジェクト参照があれば、`.ifc` の場所は自動的に解決されます。うまくいかない場合は、**[C/C++] → [全般] → 追加のモジュール依存関係**(`/reference`)で `.ifc` の場所を明示してください。

### 20.3.2 公開 API しかテストできない

第14章 14.4.5 で保留にした問題です。

テストコードは `import gamemath;` と書くだけの、普通の利用者です。だから **`export` されていないものはテストできません。**

`SafeAcos`(第10章)を直接テストすることはできません。

```cpp
// テストコード
import gamemath;

// gamemath::SafeAcos(1.5f);   // ← エラー。内部パーティションにある
```

第14章 14.4.5 で3つの選択肢を挙げ、**選択肢1(公開 API 経由でテストする)**を採ると決めました。実際にやってみます。

`SafeAcos` は「定義域外の入力で NaN を返さない」ためのものでした。これを使っているのは `AngleBetween` と `Slerp` です。だから、**その2つを境界条件でテストします。**

```cpp
// 同一のベクトルどうしの角度は 0。内積が 1 を超えても NaN にならないこと
const gamemath::Vector3 v{ 1.0f, 2.0f, 3.0f };
Check(!std::isnan(gamemath::AngleBetween(v, v)), "AngleBetween(v, v) is not NaN");
```

内部関数の境界条件を直接突けないぶん、テストの精度は落ちます。しかし、**テストが実装に依存しなくなる**という利点があります。`SafeAcos` を別の手法に置き換えても、テストは無傷です。

### 20.3.3 不変条件をテストにする

第14章 14.4.3 で列挙した不変条件を、実際のコードにします。テストフレームワークは使わず、最小限の仕組みで書きます。

`Tests/TestMain.cpp` を作ってください。

```cpp
// Tests/TestMain.cpp
import std;
import gamemath;

namespace {

int g_failures = 0;

void Check(bool condition, std::string_view what)
{
    if (!condition) {
        std::println("FAIL: {}", what);
        ++g_failures;
    }
}

constexpr float kTol = 1.0e-5f;

// ───────── ベクトル ─────────

void TestVector()
{
    using namespace gamemath;

    const Vector3 a{ 1.0f, 2.0f, 3.0f };
    const Vector3 b{ 4.0f, 5.0f, 6.0f };

    Check(NearlyEqual(a.Normalized().Length(), 1.0f, kTol),
          "Normalized().Length() == 1");
    Check(NearlyEqual(Dot(a, b), Dot(b, a), kTol),
          "Dot is commutative");
    Check(NearlyEqual(Dot(Cross(a, b), a), 0.0f, kTol),
          "Cross is perpendicular to a");
    Check(NearlyEqual(Cross(a, b), Vector3{ 0.0f, 0.0f, 0.0f } - Cross(b, a), kTol),
          "Cross is anti-commutative");

    // ゼロベクトルの正規化 ── 仕様として決めた振る舞い(第14章 14.4.4)
    const Vector3 zero{ 0.0f, 0.0f, 0.0f };
    Check(NearlyEqual(zero.Normalized(), zero, kTol),
          "Normalized() of zero vector returns zero");
}

// ───────── 行列 ─────────

void TestMatrix()
{
    using namespace gamemath;

    const float half = std::numbers::pi_v<float> * 0.5f;
    const Matrix4x4 id = Matrix4x4::Identity();
    const Matrix4x4 r  = RotationZ(half);

    Check(NearlyEqual(r * id, r, kTol),               "M * I == M");
    Check(NearlyEqual(Transpose(Transpose(r)), r, kTol), "Transpose twice");
    Check(NearlyEqual(Transpose(r) * r, id, kTol),    "Rotation is orthogonal");

    const Vector3 v{ 1.0f, 2.0f, 3.0f };
    Check(NearlyEqual(TransformDirection(r, v).Length(), v.Length(), kTol),
          "Rotation preserves length");

    // 平行移動は方向に影響しない
    const Matrix4x4 t = Translation(Vector3{ 1.0f, 2.0f, 3.0f });
    Check(NearlyEqual(TransformDirection(t, v), v, kTol),
          "Translation does not affect direction");

    // SIMD 版とスカラー版が一致する(第16章 16.2.4)
    Check(NearlyEqual(Matrix4x4::MultiplyScalar(r, t),
                      Matrix4x4::MultiplySimd(r, t), kTol),
          "Scalar and SIMD agree");
}

// ───────── クォータニオン ─────────

void TestQuaternion()
{
    using namespace gamemath;

    const float half = std::numbers::pi_v<float> * 0.5f;
    const Vector3 zAxis{ 0.0f, 0.0f, 1.0f };
    const Vector3 xAxis{ 1.0f, 0.0f, 0.0f };
    const Vector3 v{ 1.0f, 2.0f, 3.0f };

    const Quaternion q  = FromAxisAngle(zAxis, half);
    const Quaternion q2 = FromAxisAngle(xAxis, half * 0.5f);

    Check(NearlyEqual(q * q.Conjugate(), Quaternion::Identity(), kTol),
          "q * conj(q) == identity");
    Check(NearlyEqual(Rotate(q, v).Length(), v.Length(), kTol),
          "Rotate preserves length");
    Check(NearlyEqual(Rotate(q * q2, v), Rotate(q, Rotate(q2, v)), kTol),
          "Rotate(a*b, v) == Rotate(a, Rotate(b, v))");
    Check(NearlyEqual(Slerp(q, q2, 0.0f), q, kTol), "Slerp(a, b, 0) == a");
    Check(NearlyEqual(Slerp(q, q2, 1.0f), q2, kTol), "Slerp(a, b, 1) == b");
    Check(NearlyEqual(Slerp(q, q2, 0.5f).Length(), 1.0f, kTol),
          "Slerp result is normalized");

    // 符号の曖昧さを考慮した往復(第14章 14.4.4)
    const Quaternion back = ToQuaternion(ToMatrix(q));
    Check(NearlyEqual(back, q, kTol) || NearlyEqual(back, -q, kTol),
          "ToQuaternion(ToMatrix(q)) == +/- q");

    Check(NearlyEqual(ToMatrix(q), RotationZ(half), kTol),
          "ToMatrix(FromAxisAngle) == RotationZ");

    // 内部関数 SafeAcos を、公開 API 経由で検証する(20.3.2)
    Check(!std::isnan(AngleBetween(v, v)), "AngleBetween(v, v) is not NaN");
    Check(!std::isnan(Slerp(q, q, 0.5f).w), "Slerp of identical quaternions");
}

// ───────── 幾何 ─────────

void TestGeometry()
{
    using namespace gamemath;

    const Sphere s{ Vector3{ 0.0f, 0.0f, 0.0f }, 2.0f };
    const Plane ground{ Vector3{ 0.0f, 1.0f, 0.0f }, 0.0f };
    const Ray down{ Vector3{ 0.0f, 10.0f, 0.0f }, Vector3{ 0.0f, -1.0f, 0.0f } };

    const RayHit hitPlane = Intersect(down, ground);
    Check(hitPlane.hit, "Ray hits plane");
    Check(NearlyEqual(hitPlane.distance, 10.0f, kTol), "Plane hit distance");

    const RayHit hitSphere = Intersect(down, s);
    Check(hitSphere.hit, "Ray hits sphere");
    Check(NearlyEqual(hitSphere.distance, 8.0f, kTol), "Sphere hit distance");
    Check(NearlyEqual(hitSphere.normal.Length(), 1.0f, kTol), "Hit normal is unit");

    // 外れるレイ
    const Ray miss{ Vector3{ 10.0f, 10.0f, 0.0f }, Vector3{ 0.0f, -1.0f, 0.0f } };
    Check(!Intersect(miss, s).hit, "Ray misses sphere");

    // 境界ボックス
    Check(Overlaps(s.Bounds(), s), "Sphere overlaps its own bounds");
    Check(Overlaps(s, s.Bounds()), "Argument order does not matter");
}

// ───────── 乱数 ─────────

void TestRandom()
{
    Random r1{ 12345 };
    Random r2{ 12345 };

    for (int i = 0; i < 100; ++i) {
        if (r1.NextFloat() != r2.NextFloat()) {
            Check(false, "Same seed produces same sequence");
            return;
        }
    }

    Random r3{ 999 };
    float minV = 1.0f;
    float maxV = 0.0f;
    for (int i = 0; i < 100000; ++i) {
        const float v = r3.NextFloat();
        minV = std::min(minV, v);
        maxV = std::max(maxV, v);
    }
    Check(minV >= 0.0f && maxV < 1.0f, "NextFloat is in [0, 1)");
    Check(minV < 0.01f && maxV > 0.99f, "NextFloat covers the range");
}

} // namespace

int main()
{
    TestVector();
    TestMatrix();
    TestQuaternion();
    TestGeometry();
    TestRandom();

    if (g_failures == 0) {
        std::println("All tests passed.");
        return 0;
    }

    std::println("{} test(s) failed.", g_failures);
    return 1;
}
```

`Random` を使うために `using namespace gamemath;` を `TestRandom` にも足すか、修飾してください。

ビルドして実行します。

```
cmake --build build
ctest --test-dir build --output-on-failure
```

```
All tests passed.
```

### 20.3.4 コンパイル時テストも書ける

第8章 8.5.2 以来使ってきた `static_assert` は、**最も強力なテスト**です。失敗したらビルドが止まるので、見逃しようがありません。

`constexpr` にできる関数(第14章 14.3.5)については、実行時テストより `static_assert` を優先してください。

```cpp
// Tests/TestMain.cpp の名前空間スコープ
namespace {

using namespace gamemath;

constexpr Vector3 ca{ 1.0f, 2.0f, 3.0f };
constexpr Vector3 cb{ 4.0f, 5.0f, 6.0f };

static_assert(Dot(ca, cb) == 32.0f);
static_assert(Dot(Cross(ca, cb), ca) == 0.0f);
static_assert(Matrix4x4::Identity() * Matrix4x4::Identity() == Matrix4x4::Identity());
static_assert(ToMatrix(Quaternion::Identity()) == Matrix4x4::Identity());
static_assert(Overlaps(AABB{ Vector3{0,0,0}, Vector3{2,2,2} },
                       AABB{ Vector3{1,1,1}, Vector3{3,3,3} }));

} // namespace
```

第16章 16.3.3 で `if consteval` を使ったおかげで、行列の積も `static_assert` で検証できています。

### 20.3.5 テストフレームワークを使う場合

実務では Catch2、GoogleTest、doctest といったフレームワークを使うことが多いでしょう。

**注意点が1つあります。** これらはヘッダオンリー、またはヘッダ + ライブラリの形で提供されます。つまり第18章の問題が発生します。

テストコードの中で、こういう組み合わせになります。

```cpp
#include <catch2/catch_test_macros.hpp>   // ヘッダ
import gamemath;                          // モジュール
```

**これ自体は問題ありません。** `#include` と名前付きモジュールの `import` は共存できます。

危険なのは、第6章 6.5.2 の混在です。テストフレームワークが `<vector>` や `<string>` を `#include` していて、同じファイルで `import std;` も書くと、問題が起きる可能性があります。

**対策**

- テストコードでは `import std;` を使わず、`#include` で揃える
- または、フレームワークを使わずに 20.3.3 のような素朴な仕組みで書く

`GameMath` のテストは後者にしました。依存を増やさずに済み、本書の範囲で完結するからです。

---

## 20.4 ライブラリとして配布する

### 20.4.1 `.ifc` は配布物ではない

第1章 1.4.3 と第2章 2.4.4 で繰り返してきたことを、改めて確認します。

**`.ifc` は、コンパイラのバージョン・オプション・ターゲットに強く結びついたファイルです。** 少しでも条件が違えば読めません。Visual Studio を更新しただけで使えなくなります。

だから、**`.ifc` を配布することはできません。**

「モジュールにすればバイナリ配布が楽になる」という期待があるとしたら、それは誤解です。第1章 1.4.3 のとおりです。

### 20.4.2 では何を配布するのか

答えは素直です。

**モジュールインターフェイスの `.ixx` を配布します。**

そして利用者が、自分の環境でコンパイルして `.ifc` を作ります。

**これは、ヘッダを配布するのと本質的に同じことです。**

```
ヘッダ時代:   .h を配布 → 利用者が #include して解析する
モジュール:   .ixx を配布 → 利用者が import してコンパイルする
```

`.h` も「利用者の環境でコンパイルされるソース」でした。`.ixx` も同じです。**新しい問題ではありません。**

実装の扱いも同じです。

| | 配布するもの |
|---|---|
| ヘッダオンリーライブラリ | `.h` だけ |
| ヘッダ + バイナリ | `.h` + `.lib` |
| **モジュールオンリー** | **`.ixx` だけ**(実装もインターフェイスに書く) |
| **モジュール + バイナリ** | **`.ixx` + `.lib`**(実装単位はコンパイル済み) |

`GameMath` は最後の形になります。テンプレートと `constexpr` 関数はインターフェイスにあり(第8章 8.5.5)、実装単位の中身は `.lib` に入ります。

### 20.4.3 CMake で install する

CMake は、この形の配布に対応しています。

```cmake
include(GNUInstallDirs)

install(TARGETS gamemath
    EXPORT gamemath-targets
    ARCHIVE  DESTINATION ${CMAKE_INSTALL_LIBDIR}
    LIBRARY  DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME  DESTINATION ${CMAKE_INSTALL_BINDIR}
    FILE_SET CXX_MODULES DESTINATION ${CMAKE_INSTALL_DATADIR}/gamemath/modules
)

install(EXPORT gamemath-targets
    FILE gamemath-targets.cmake
    NAMESPACE gamemath::
    DESTINATION ${CMAKE_INSTALL_DATADIR}/gamemath/cmake
    CXX_MODULES_DIRECTORY cxx-modules
)
```

注目してほしいのは2か所です。

**`FILE_SET CXX_MODULES DESTINATION ...`**

**モジュールインターフェイスの「ソース」をインストールします。** `.ifc` ではありません。20.2.3 で `PUBLIC` にしていたのは、このためでした。`PRIVATE` にしたファイル(実装単位)はインストールされません。

配置場所に**確立した慣習はまだありません。** CMake の設定ファイルの近くに置くのが穏当な選択とされています。

**`CXX_MODULES_DIRECTORY`**

CMake が、モジュールごとに生成する追加の情報ファイルの置き場所です。利用者側の CMake が、モジュールをどうビルドすればよいかを知るために使います。

### 20.4.4 利用者側が引き受けること

利用者は、`find_package(gamemath)` して `target_link_libraries` するだけです ── 理屈のうえでは。

しかし、いくつかの制約を引き受けることになります。

**制約1: ビルドシステムがモジュールに対応している必要がある**

`.ixx` はコンパイルされなければなりません。それも、正しい順序で。**利用者が古いビルドシステムを使っていたら、そもそも使えません。**

ヘッダなら、どんなビルドシステムでも `#include` できました。ここは明確な後退です。

**制約2: C++ の言語標準が制約される**

`GameMath` が C++23 を要求すると、利用者も C++23 以上でビルドすることになります。

そして実務では、**利用者がもっと新しい標準を使いたい**場合に問題が起きることがあります。ライブラリ側が `cxx_std_23` を「インターフェイス要件」として宣言すると、利用者側がそれより新しい標準に上げられない、という報告があります。

**制約3: コンパイルオプションの整合**

`.ifc` はオプションに敏感です。例外の設定、RTTI、イテレータデバッグレベル ── これらが食い違うと、モジュールを読めないか、読めても実行時に壊れます。

これはヘッダ時代の ODR 問題と同種のものです。**モジュールでも解決していません**(第1章 1.4.3)。

### 20.4.5 現実的な配布戦略

以上を踏まえると、現時点での現実的な選択肢は3つです。

**戦略1: ソース配布(推奨)**

`.ixx` と `.cpp` をすべて配布し、利用者のビルドに組み込んでもらいます。CMake のサブディレクトリとして取り込む形です。

**長所** ── オプションの整合問題が起きません。デバッグしやすい
**短所** ── 利用者のビルド時間が増えます

数学ライブラリのように小さく、テンプレート中心のライブラリなら、これがいちばん素直です。**そもそもテンプレートは `.ifc` に載るので、隠す意味がありません**(第8章 8.2.6)。

**戦略2: `.ixx` + `.lib`**

20.4.3 の形です。実装単位の中身をバイナリにできます。

**長所** ── 実装を隠せます(実装単位に置いたぶんだけ)
**短所** ── オプションの整合が必要です

**戦略3: ヘッダも併せて提供する**

第19章 19.2 の構成A です。移行期のライブラリなら、これが最も広く使ってもらえます。

**「まだモジュールに移行していない利用者を切り捨てない」**という判断です。

### 20.4.6 ヘッダ時代との比較

正直に整理しておきます。

| | ヘッダ | モジュール |
|---|---|---|
| 配布するもの | `.h`(+ `.lib`) | `.ixx`(+ `.lib`) |
| 利用者のビルドシステム要件 | **なし** | **モジュール対応が必要** |
| 言語標準の要件 | ゆるい | **厳しい** |
| コンパイルオプションの整合 | 必要 | 必要(変わらず) |
| 実装を隠せるか | 一部 | 一部(実装単位のぶん) |
| 利用者のビルド時間 | 悪い | **良い** |

**配布の観点では、モジュールは制約が増えています。** 得られるのは利用者のビルド時間だけです。

だからこそ、第19章 19.1 で「移行しないという選択肢」を強調しました。**再配布するライブラリのモジュール化は、利用者の環境を選ぶという副作用を伴います。**

---

## 20.5 vcpkg / パッケージ化の現状

### 20.5.1 状況

パッケージマネージャ(vcpkg、Conan など)でモジュールを配布する仕組みは、**まだ確立の途上にあります。**

理由は 20.4 で見たとおりです。

- `.ifc` を配れないので、ソース配布かソースビルドになる
- 利用者のビルドシステムがモジュールに対応している必要がある
- コンパイルオプションの整合が必要

vcpkg は元々ソースからビルドする方式なので、この点では相性が悪くありません。しかし、**パッケージ側がモジュールを提供しているかどうか**は、また別の問題です。

**2026 年時点では、主要なライブラリの大半はヘッダで提供されています。**

### 20.5.2 いま取れる方法

パッケージマネージャから取得したライブラリを、モジュール化された自分のコードから使いたい場合 ── **第18章 18.4 の3案がそのまま当てはまります。**

| 状況 | 方法 |
|---|---|
| そのライブラリがモジュールを提供している | そのまま `import` |
| ヘッダのみ。1〜2か所で使う | GMF で `#include`(18.4.2) |
| ヘッダのみ。多数の場所で使う | ヘッダユニット(18.4.3) |
| ヘッダのみ。依存を隠したい | **ラッパーモジュール**(18.4.4) |

### 20.5.3 情報は必ず確認する

この分野は動きが速く、本書の記述はすぐに古くなります。

- CMake の `cmake-cxxmodules(7)` ドキュメント
- vcpkg / Conan の公式ドキュメント
- 使いたいライブラリのリポジトリ

**実際に手を動かす前に、必ず現在の情報を確認してください。** とくに `import std;` に関する CMake の対応は、まだ実験段階なので変更が入ります(20.2.4)。

---

## 20.6 この章のまとめ

- MSBuild は「スキャン → 依存グラフ構築 → コンパイル → リンク」の順で処理する
- **依存が深いほど並列度が下がる。** 土台を薄く、その上を横に広げる構造が有利
- `Core/Detail.ixx`(内部パーティション)だけがファイル単位の手動設定を必要とする。**移行時にいちばん忘れられる**
- CMake は **3.28 以上**でモジュールを正式サポート。Ninja は 1.11 以上
- **`target_sources(... PUBLIC FILE_SET CXX_MODULES FILES ...)`** でインターフェイス単位を指定する
- 実装単位は `FILE_SET` に含めず、通常のソースとして `PRIVATE` に置く
- **CMake の `import std;` 対応はまだ実験的。** ゲート変数の UUID はバージョンごとに変わり、Ninja ジェネレータでしか使えない
- クロスプラットフォームを重視するなら、**第6章 6.6.2 の GMF 版を選ぶ判断もある**
- テストコードは普通の利用者。**`export` されていないものはテストできない**
- 内部関数は、それを使う公開関数を**境界条件で**テストする
- `constexpr` にできるものは `static_assert` で検証する。**失敗したらビルドが止まるので見逃せない**
- テストフレームワークを使うなら、`import std;` と `#include` の混在に注意する
- **`.ifc` は配布できない。配布するのは `.ixx`(ソース)**
- **これはヘッダを配布するのと本質的に同じ。** 新しい問題ではない
- CMake の `install(TARGETS ... FILE_SET CXX_MODULES DESTINATION ...)` がモジュールソースをインストールする
- 利用者は、**モジュール対応のビルドシステム**と**言語標準の制約**を引き受けることになる
- 配布の観点では、モジュールは**制約が増えている**。得られるのは利用者のビルド時間だけ
- パッケージマネージャの対応はまだ途上。大半のライブラリはヘッダのまま

## 次章に向けて

ここまで何度も「測ってから」と書いてきました。第21章で、実際に測ります。

- ヘッダ版とモジュール版の**同一のライブラリ**を用意する
- フルビルドと増分ビルドを計測する
- 結果をどう読むか

第19章 19.1.2 で「移行の動機になるのは、測定に基づくビルド時間」と書きました。その測定を、いよいよ行います。

**結果は、期待どおりとは限りません。** そこで何を見るべきか、どこで効いてくるのかを、正直に扱います。
