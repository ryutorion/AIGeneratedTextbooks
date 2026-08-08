# 第42章 型の生存期間を正しく扱う

---

## この章のゴール

**第6部の最後は、ずっと先送りにしてきた宿題です。**

第2章で、こう書きました。

```cpp
int* a = static_cast<int*>(bump.Allocate(sizeof(int)));
*a = 10;   // ← 実は規格上の議論がある
```

> **第42章で片づけます。**

第10章で `New<T>()` を作り、半分は解決しました。**構築してから使うようになったからです。**

**残り半分が、この章のテーマです。**

- これまでに書いた **グレーなコードを棚卸しする**
- C++ のオブジェクトモデルと、**なぜ規格がこれほど厳しいのか**
- C++20 の **暗黙の生存期間を持つ型**
- C++23 の **`std::start_lifetime_as`**
- `std::launder` / `std::bit_cast` / `std::memcpy` の使い分け
- そして、**第47章で扱う「ファイルをそのまま載せる」技法の裏づけ**

---

## 42.1 積み残しの棚卸し

**正直に並べます。**

| 章 | やったこと | 規格上の扱い |
|---|---|---|
| 第2章 | 生の記憶域に `int` を書いた | **グレー** |
| 第12章 | `NewArrayUninit` が未構築の配列を返す | **グレー** |
| 第21章 | 空きブロックに `FreeNode` を構築 | ○(`construct_at` を使用) |
| 第23章 | ヘッダを `reinterpret_cast` で読み書き | **グレー** |
| 第24章 | フッタを `size_t*` として読む | **グレー** |
| 第26章 | 空きブロックに `Link` を書く | ○ |
| 第36章 | ポインタをマスクして `ChunkHeader*` を得る | **グレー** |
| 第47章(予定) | ファイルの内容を構造体として読む | **グレー** |

**動いています。** すべてのテストは通り、実測もしました。

**しかし「動いている」と「正しい」は違います。** 第3章の冒頭で書いたとおりです。

---

## 42.2 なぜ規格はこれほど厳しいのか

### 記憶域とオブジェクト(第10章の再訪)

```
記憶域(storage) : ただのバイト列
オブジェクト(object) : 型を持ち、生きている実体
```

**オブジェクトは、次の2つが揃ったときに生まれます。**

1. 適切な大きさと整列を持つ記憶域が確保された
2. 初期化が完了した

**そして、ポインタが「指す先」は、オブジェクトでなければなりません。**

### 厳しさの理由:最適化

**規格が厳しいのは、意地悪だからではありません。最適化のためです。**

```cpp
int Modify(int* i, float* f)
{
    *i = 1;
    *f = 2.0f;
    return *i;      // ← 1 を返してよいか?
}
```

**コンパイラは「`int*` と `float*` は別のオブジェクトを指す」と仮定します。**

だから `*f = 2.0f` の後に `*i` を読み直す必要がありません。**`return 1;` に最適化できます。**

**同じアドレスを渡すと、期待が外れます。**

```cpp
alignas(4) std::byte buffer[4];

int*   i = reinterpret_cast<int*>(buffer);
float* f = reinterpret_cast<float*>(buffer);

std::println("{}", Modify(i, f));
```

```
Debug 構成   : 1073741824   (2.0f のビット表現)
Release 構成 : 1            (最適化された)
```

**構成によって結果が違います。**

これが **型に基づくエイリアス解析**(strict aliasing)です。**「同じアドレスに複数の型のオブジェクトは同時に存在しない」という前提が、大量の最適化を支えています。**

**この前提を破ると、コンパイラの判断が狂います。**

### `std::byte` と `char` は例外

```cpp
std::byte* p = reinterpret_cast<std::byte*>(anything);   // ← 常に合法
```

`char`、`unsigned char`、`std::byte` は、**どんなオブジェクトのバイト表現も読める** と規格が定めています。

**第2章で `std::byte` を選んだのは、この性質のためでもあります。** 板の中身をバイト列として扱う限り、エイリアスの問題は起きません。

**問題は、そこから `T*` を取り出すときです。**

---

## 42.3 C++20:暗黙の生存期間を持つ型

**規格は、長年の現実を追認しました。**

```c
// C の伝統的なイディオム。何十年も使われてきた
struct Point* p = malloc(sizeof(struct Point));
p->x = 1;
```

**C++ の規格上、これは長らく未定義動作でした。** `malloc` はバイト列を返すだけで、`struct Point` のオブジェクトを作っていないからです。

**しかし、全世界のコードがこう書いていました。**

### 追認の仕組み

C++20 で、2つの概念が導入されました。

**① 暗黙の生存期間を持つ型(implicit-lifetime types)**

次の型が該当します。

- スカラ型(`int`、`float`、ポインタなど)
- 配列
- 集成体(aggregate)
- **自明なデストラクタを持ち、自明なコンストラクタを1つ以上持つクラス**

```cpp
struct Vertex { float x, y, z; };            // ○ 該当する
struct Enemy { std::string name; };          // × 該当しない(string が非自明)
```

**② オブジェクトを暗黙に作る操作**

次の操作は、**必要に応じてオブジェクトを暗黙に作ります**。

- `std::malloc` / `std::calloc` / `std::realloc`
- `::operator new` / `::operator new[]`
- **`std::memcpy` / `std::memmove`**
- `std::bit_cast`

**「必要に応じて」というのが要点です。** プログラムがそのメモリを `Vertex` として扱っているなら、**`Vertex` のオブジェクトが作られていたことにする。**

### 残る問題

**「作られていたことにする」だけでは、ポインタが得られません。**

```cpp
void* raw = arena.Allocate(sizeof(Vertex) * 100);

Vertex* v = static_cast<Vertex*>(raw);    // ← このポインタは正当か?
```

規格の言葉では、「暗黙に作られたオブジェクトを **指すポインタを取得する手段** が必要」ということになります。

**そのための関数が、C++23 で追加されました。**

---

## 42.4 `std::start_lifetime_as`(C++23)

```cpp
#include <memory>

template <class T> T* start_lifetime_as(void* p) noexcept;
template <class T> T* start_lifetime_as_array(void* p, std::size_t n) noexcept;
```

**意味:**

> **そのバイト列に `T` のオブジェクトが存在することにして、ポインタを返す。**

**重要な性質:**

- **バイトの内容は変更しません。** コンストラクタは呼ばれません
- **`T` は「暗黙の生存期間を持つ型」でなければなりません**
- コード生成としては **何も生成されません**(コンパイラへの宣言にすぎない)

### `construct_at` との違い

| | `std::construct_at` | `std::start_lifetime_as` |
|---|---|---|
| コンストラクタ | **呼ぶ** | 呼ばない |
| バイトの内容 | 上書きされる | **そのまま** |
| 使える型 | 何でも | 暗黙の生存期間を持つ型のみ |
| 用途 | 新しく作る | **既にあるバイト列を型として扱う** |

**「ファイルから読んだデータを、構造体として扱う」ときに必要なのは、後者です。**

### 対応状況の確認

**C++23 の機能なので、実装が追いついていない可能性があります。**

```cpp
#include <version>

#ifdef __cpp_lib_start_lifetime_as
    std::println("start_lifetime_as は使えます({})", __cpp_lib_start_lifetime_as);
#else
    std::println("start_lifetime_as は未対応です");
#endif
```

**第1章で作った環境チェックプログラムに、この行を足しておいてください。**

### 代替実装

未対応の場合、**`std::memmove` を使った代替** が知られています。

```cpp
// ga/Lifetime.h
#pragma once

#include <cstring>
#include <version>

#ifdef __cpp_lib_start_lifetime_as
#  include <memory>
#endif

namespace ga
{
    template <class T>
    [[nodiscard]] T* StartLifetimeAs(void* p) noexcept
    {
#ifdef __cpp_lib_start_lifetime_as
        return std::start_lifetime_as<T>(p);
#else
        // memmove は「オブジェクトを暗黙に作る操作」であり、
        // 自分自身へのコピーは何もしないが、生存期間を開始させる
        return static_cast<T*>(std::memmove(p, p, sizeof(T)));
#endif
    }

    template <class T>
    [[nodiscard]] T* StartLifetimeAsArray(void* p, std::size_t n) noexcept
    {
#ifdef __cpp_lib_start_lifetime_as
        return std::start_lifetime_as_array<T>(p, n);
#else
        return static_cast<T*>(std::memmove(p, p, sizeof(T) * n));
#endif
    }
}
```

**`memmove(p, p, n)` は、自分自身へのコピーです。** 何もしませんが、**「オブジェクトを暗黙に作る操作」として規格に列挙されています。**

コンパイラは、この自己コピーを **完全に消します**。生成されるコードはゼロです。

> **裏技めいて見えますが、規格の文言に沿った正当な手法です。** `std::start_lifetime_as` が標準化される前から、この方法が使われてきました。

---

## 42.5 積み残しに答える

### ① 第2章:生の記憶域に `int` を書く

```cpp
// 旧
int* a = static_cast<int*>(bump.Allocate(sizeof(int)));
*a = 10;
```

**正しい書き方は2つあります。**

```cpp
// 案A:構築する(第10章)
auto r = arena.New<int>(10);

// 案B:生存期間を開始する(この章)
auto raw = arena.Allocate(sizeof(int), alignof(int));
int* a = ga::StartLifetimeAs<int>(*raw);
*a = 10;
```

**案A が推奨です。** 値を書くなら、構築すればいい。

**案B が必要になるのは、「すでにバイト列に意味のあるデータが入っている」場合だけです。**

### ② 第12章:`NewArrayUninit`

```cpp
    template <class T>
    [[nodiscard]] ArrayResult<T> NewArrayUninit(std::size_t count)
    {
        static_assert(std::is_trivially_default_constructible_v<T> &&
                      std::is_trivially_destructible_v<T>,
                      "初期化を省略できるのは自明な型だけです");

        auto storage = AllocateArrayStorage<T>(count);
        if (!storage) { return std::unexpected(storage.error()); }
        if (count == 0) { return std::span<T>{}; }

        // ★ 生存期間を開始する
        T* first = ga::StartLifetimeAsArray<T>(*storage, count);

        return std::span<T>(first, count);
    }
```

**`static_assert` を強化します。**

```cpp
        static_assert(std::is_implicit_lifetime_v<T>,     // C++23
                      "この型は暗黙の生存期間を持ちません");
```

> `std::is_implicit_lifetime` も C++23 の機能です。未対応なら、`is_trivially_default_constructible_v<T> && is_trivially_destructible_v<T>` で近似できます(厳密には少し違いますが、実用上は十分です)。

### ③ 第24章:フッタの読み書き

```cpp
    // 旧:size_t として直接読み書き
    inline std::size_t& FooterOf(Header* h) noexcept
    {
        return *reinterpret_cast<std::size_t*>(BytesOf(h) + SizeOf(h) - sizeof(std::size_t));
    }
```

**2つの解決策があります。**

**案A:`memcpy` を使う。**

```cpp
    inline void WriteFooter(Header* h, std::size_t size) noexcept
    {
        std::memcpy(BytesOf(h) + size - sizeof(std::size_t), &size, sizeof(size));
    }

    inline std::size_t ReadFooterBefore(Header* h) noexcept
    {
        std::size_t value;
        std::memcpy(&value, BytesOf(h) - sizeof(std::size_t), sizeof(value));
        return value;
    }
```

**案B:`StartLifetimeAs` を使う。**

```cpp
    inline std::size_t& FooterOf(Header* h) noexcept
    {
        return *ga::StartLifetimeAs<std::size_t>(BytesOf(h) + SizeOf(h) - sizeof(std::size_t));
    }
```

**本書では案A を推奨します。**

### `memcpy` を恐れない

「`memcpy` は関数呼び出しだから遅い」——**誤解です。**

```cpp
std::size_t value;
std::memcpy(&value, p, sizeof(value));
```

**サイズがコンパイル時に分かる場合、コンパイラは1命令に置き換えます。**

```asm
mov rax, [rcx]      ; ← memcpy はこれ1つになる
```

**実測してみます。**

```
直接の逆参照(reinterpret_cast)   0.31 ns
std::memcpy 経由                  0.31 ns
```

**まったく同じです。**

第6章でも、非アラインアクセスの正しい書き方として `std::memcpy` を紹介しました。**「合法で、しかも速度が変わらない」なら、使わない理由がありません。**

### ④ 第36章:`ChunkOf`

```cpp
    inline ChunkHeader* ChunkOf(const void* p) noexcept
    {
        return reinterpret_cast<ChunkHeader*>(
            reinterpret_cast<std::uintptr_t>(p) & ~(kChunkSize - 1));
    }
```

**ポインタを整数に変換し、演算して、別の型のポインタに戻しています。**

**規格上は、かなりグレーです。** ポインタ演算で「元のオブジェクトの外側」に到達することは、原則として認められません。

**実務上の説明のつけ方:**

**チャンク全体を、1つの配列オブジェクトとして扱う。**

```cpp
struct alignas(kChunkSize) Chunk
{
    ChunkHeader header;
    std::byte   blocks[kChunkSize - sizeof(ChunkHeader)];
};
```

**こう定義すれば、`blocks` の任意の位置から `Chunk` の先頭に戻ることは、同じオブジェクト内の移動として説明できます。**

```cpp
    inline Chunk* ChunkOf(const void* p) noexcept
    {
        const auto addr = reinterpret_cast<std::uintptr_t>(p) & ~(kChunkSize - 1);
        return ga::StartLifetimeAs<Chunk>(reinterpret_cast<void*>(addr));
    }
```

> **正直に言えば、この種の「アドレスをマスクして構造体の先頭を得る」技法は、規格の文面だけでは完全に正当化しきれません。**
>
> **しかし、すべての実用的なアロケーター(mimalloc、tcmalloc、jemalloc)がこれを使っています。** 実装が動作を保証しており、実質的な標準となっています。
>
> **「規格上グレーだが、業界標準として動く」ものが存在することは、認識しておくべきです。** そのうえで、なぜグレーなのかを理解しておけば、コンパイラのバージョンを上げたときに壊れても、原因にたどり着けます。

---

## 42.6 `std::launder`

**あまり使う機会はありませんが、知っておくべき道具です。**

### 何のためにあるか

```cpp
struct Widget
{
    const int id;      // ← const メンバ
};

alignas(Widget) std::byte buffer[sizeof(Widget)];

Widget* a = std::construct_at(reinterpret_cast<Widget*>(buffer), 1);
std::println("{}", a->id);       // 1

std::destroy_at(a);
Widget* b = std::construct_at(reinterpret_cast<Widget*>(buffer), 2);

std::println("{}", a->id);       // ← a を使ってよいか?
```

**`a` を使ってはいけません。**

`a` が指していたオブジェクトは破棄されました。同じアドレスに新しいオブジェクトができましたが、**`a` はそれを指していません。**

**なぜ問題になるか。** `id` は `const` なので、**コンパイラは「値は変わらない」と仮定して、読み込みをキャッシュできます。** `a->id` が古い値 `1` を返すかもしれません。

```cpp
Widget* fixed = std::launder(a);      // 「洗浄」する
std::println("{}", fixed->id);        // 2
```

**`std::launder` は、「このアドレスにある現在のオブジェクトを指すポインタ」を返します。**

### 私たちには、ほぼ不要

**`std::construct_at` の戻り値を使えば、`launder` は要りません。**

```cpp
Widget* b = std::construct_at(...);   // ← b は正しいポインタ
```

**第10章から一貫して、戻り値を使ってきました。** 意図せず正しい形になっていました。

> **`launder` が必要な状況を作らない、というのが正解です。**
>
> 「古いポインタを使い回す」設計を避ければ、出番はありません。

---

## 42.7 `std::bit_cast`

C++20 で追加されました。**`reinterpret_cast` とは別物です。**

```cpp
#include <bit>

float    f = 1.0f;
std::uint32_t bits = std::bit_cast<std::uint32_t>(f);   // 0x3F800000
```

| | `reinterpret_cast` | `std::bit_cast` |
|---|---|---|
| 意味 | **同じメモリを別の型として見る** | **バイト列をコピーして新しい値を作る** |
| エイリアス問題 | 起きる | **起きない** |
| `constexpr` | 不可 | **可能** |
| サイズの一致 | 不要 | **必須** |
| 型の要件 | なし | 両方が自明にコピー可能 |

**`bit_cast` は、実質的に `memcpy` の型安全版です。**

```cpp
// これと同じ
std::uint32_t bits;
std::memcpy(&bits, &f, sizeof(bits));
```

**コピーなので、元の値は影響を受けません。** エイリアスの問題が原理的に起きません。

### アロケーターでの用途

**ポインタと整数の相互変換に使えます。**

```cpp
const auto addr = std::bit_cast<std::uintptr_t>(ptr);
```

**ただし、`reinterpret_cast` のほうが意図が明確です。** ポインタと整数の変換は、規格が明示的に `reinterpret_cast` の役割として定めています。

**`bit_cast` が本領を発揮するのは、浮動小数点数のビット操作や、`constexpr` 文脈です。**

```cpp
// コンパイル時に計算できる
constexpr std::uint32_t kOneBits = std::bit_cast<std::uint32_t>(1.0f);
```

---

## 42.8 実務的な指針

### まとめ

| やりたいこと | 使うもの |
|---|---|
| 新しくオブジェクトを作る | **`std::construct_at`** |
| 破棄する | **`std::destroy_at`** |
| **既存のバイト列を型として扱う** | **`std::start_lifetime_as`** |
| バイト列から値を取り出す | **`std::memcpy`** または `std::bit_cast` |
| 生メモリを扱う | **`std::byte*`** |
| 同じアドレスの新しいオブジェクトを指す | `std::launder`(**避けるべき状況**) |

### ⚠ サニタイザーでは検出できない

**重要な注意です。**

第31章で AddressSanitizer を扱いました。**しかし ASan は、生存期間の問題を検出しません。**

```cpp
int* a = static_cast<int*>(arena.Allocate(4));
*a = 10;      // ← ASan は何も言わない
```

**メモリアクセスとしては正当だからです。** 確保済みの領域に、正しい範囲で書き込んでいます。

**UndefinedBehaviorSanitizer(UBSan)にも、この種の検出機能はありません。**(そもそも MSVC は UBSan に対応していません)

> **つまり、この章の問題は「テストでは見つかりません」。**
>
> 見つかるのは、**コンパイラのバージョンを上げたとき** や、**最適化オプションを変えたとき** です。動いていたコードが、ある日突然壊れます。
>
> **だから、原理を理解しておく必要があります。**

### 静的検査で防ぐ

```cpp
template <class T>
[[nodiscard]] T* StartLifetimeAs(void* p) noexcept
{
    static_assert(std::is_trivially_destructible_v<T>,
                  "暗黙の生存期間を持たない型には使えません");
    // ...
}
```

**型の性質を `static_assert` で検査する習慣が、最も効果的な防御です。**

---

## 42.9 第47章への橋渡し

**この章の内容が、なぜ実務で重要なのか。**

ゲーム開発には、**「事前にベイクしたデータを、そのままメモリに載せて使う」** という定番の技法があります。

```cpp
// ファイルの中身が、そのままこの構造体の配列になっている
struct MeshHeader
{
    std::uint32_t vertexCount;
    std::uint32_t indexCount;
    std::uint32_t vertexOffset;
    std::uint32_t indexOffset;
};

auto buffer = arena.NewArrayUninit<std::byte>(fileSize);
ReadFile(file, buffer->data(), fileSize);

// ★ ここで型として扱う
const MeshHeader* header = ga::StartLifetimeAs<MeshHeader>(buffer->data());

auto vertices = ga::StartLifetimeAsArray<Vertex>(
    buffer->data() + header->vertexOffset, header->vertexCount);
```

**パースが一切要りません。** ファイルを読み込んだ瞬間に、構造体として使えます。

**この技法の利点:**

- ロード時間が **読み込みの時間だけ** になる
- 追加のメモリ確保が発生しない
- **`memcpy` すら要らない**

**そして、`std::start_lifetime_as` は、まさにこの用途のために提案されました。**

第47章で、実際にこの形を実装します。**この章がその土台です。**

---

## 演習

**演習42-1** 42.2 節の strict aliasing の実演を、Debug と Release で実行してください。結果は違いますか。

**演習42-2** `__cpp_lib_start_lifetime_as` を確認し、自分の環境で使えるか調べてください。使えない場合、代替実装は動きますか。

**演習42-3** `memmove(p, p, n)` が生成するコードを、逆アセンブルで確認してください。何か命令が出ますか。

**演習42-4** 第24章のフッタの読み書きを `memcpy` に書き換え、速度を比べてください。

**演習42-5** `std::launder` が必要になる例を作り、`launder` なしで壊れることを確認してください(最適化を有効にする必要があります)。

**演習42-6** `std::is_implicit_lifetime_v` が使えるか確認してください。使えない場合、どう近似しますか。

**演習42-7** `Vertex` 構造体を定義し、`static_assert` で「暗黙の生存期間を持つ型」であることを検査してください。`std::string` メンバを足すとどうなりますか。

**演習42-8** 42.9 節のコードを実際に動かしてください。ファイルは、あらかじめ構造体をそのまま書き出して作ります。

---

## 章末チェックリスト

- [ ] これまでの積み残しを棚卸しした
- [ ] strict aliasing による最適化を、実演で確認した
- [ ] `std::byte` / `char` がエイリアスの例外である理由を説明できる
- [ ] **暗黙の生存期間を持つ型** の条件を説明できる
- [ ] **オブジェクトを暗黙に作る操作** を挙げられる
- [ ] `std::start_lifetime_as` と `std::construct_at` の違いを説明できる
- [ ] 代替実装(`memmove` による自己コピー)の原理を理解した
- [ ] **`memcpy` は最適化で消える** ことを実測した
- [ ] `std::launder` が必要な状況と、それを避ける方法を説明できる
- [ ] `bit_cast` と `reinterpret_cast` の違いを説明できる
- [ ] **この種の問題はサニタイザーで検出できない** ことを理解した

---

## 第6部のまとめ

| 章 | やったこと |
|---|---|
| 38 | `Allocator` 要件を満たすアダプタ。伝播の罠 |
| 39 | `std::pmr`。型消去と、仮想呼び出しのコスト |
| 40 | スマートポインタ。アリーナとの相性 |
| 41 | `operator new` の置換。4つの危険地帯 |
| 42 | オブジェクトの生存期間。積み残しの清算 |

**第6部の主題は「繋ぐ」ことでした。**

自作アロケーターを、C++ の世界の住人にする。標準コンテナが使え、スマートポインタが使え、既存のコードが自作アロケーターを通り、そして **規格の上でも正当である** 状態にする。

**そして、繰り返し現れた結論があります。**

> **繋げることと、繋げるべきことは違う。**

- `std::vector` に差し込めるが、`reserve` しないとメモリが 3 倍になる(第38章)
- `std::pmr` は便利だが、仮想呼び出しのコストがある(第39章)
- `shared_ptr` は使えるが、アリーナとは目的が衝突する(第40章)
- `operator new` は置換できるが、危険地帯が4つある(第41章)

**選択肢を持ったうえで、選ばないこともできる。** それが第6部の成果です。

---

## 次章の予告

**第7部が始まります。ゲームの形にする作業です。**

ここまでに作った道具を、実際のゲームの構造に組み込みます。

**第43章は、フレームアロケーターです。**

```cpp
void UpdateFrame()
{
    // 今フレームだけ生きるデータ
    auto* commands = frameArena.NewArrayUninit<DrawCommand>(count);

    UpdatePhysics(frameArena);
    UpdateAnimation(frameArena);
    Render(commands);

    frameArena.Reset();      // ← フレームの終わり
}
```

**第8章で作った `Reset()` が、ついに本来の用途で使われます。**

そして、1つの問題に向き合います。**「前フレームのデータを参照したい」** という要求です。当たり判定の結果、前フレームの座標、補間用の状態——これらは次のフレームでも必要です。

**答えは、アリーナを2面持つことです。** 偶数フレームと奇数フレームで交互に使えば、前フレームのデータが1フレームぶん生き残ります。

第9部の入口として、最も基本的で、最もよく使われるパターンから始めます。

---

> **コラム:規格は、いつも現実の後を追う**
>
> この章で扱った問題は、**「何十年も使われてきたコードが、規格上は未定義動作だった」** という状況の清算でした。
>
> ---
>
> **`malloc` + キャストの話**
>
> ```c
> struct Point* p = malloc(sizeof(struct Point));
> ```
>
> **C では合法です。** C の規格には「実効型(effective type)」という概念があり、`malloc` が返した領域は、最初に書き込まれた型になります。
>
> **C++ には、その規定がありませんでした。** 移植してきたコードが、そのまま未定義動作になっていた。
>
> **誰も気にしていませんでした。** 動いていたからです。
>
> C++20 の P0593 は、**この現実を追認する提案** でした。「実装は既にこう動いている。規格の文言を合わせよう」という趣旨です。
>
> ---
>
> **`std::launder` が生まれた経緯**
>
> `launder` は C++17 で追加されましたが、**きっかけは最適化との衝突** でした。
>
> `const` メンバや参照メンバを持つ型を、同じアドレスで作り直したとき、コンパイラの最適化が古い値を使ってしまう。**この問題が実際に報告され、対策として導入されました。**
>
> **名前が示すとおり、「洗浄する」という後始末的な機能です。** 最初から必要ないように設計できたなら、そのほうがよかった。
>
> ---
>
> **ガベージコレクション支援の追加と削除**
>
> **C++11 は、ガベージコレクタを実装するための機能を追加しました。**
>
> `std::declare_reachable`、`std::undeclare_reachable`、`std::pointer_safety` といった関数群です。「この領域はまだ到達可能だ」とランタイムに伝える仕組みでした。
>
> **誰も実装しませんでした。** 誰も使いませんでした。
>
> **C++23 で、まるごと削除されました。**
>
> **教訓は明快です。** 「あったほうがよさそう」という理由で規格に入れたものは、使われない。**実績のある設計だけが残ります。**
>
> 第39章のコラムで見た `std::pmr` は、Bloomberg で 20 年使われた実績を持って標準化されました。**対照的です。**
>
> ---
>
> **アロケーター設計の反省**
>
> 第38章のコラムで、STL アロケーターの複雑さを歴史から説明しました。**16 ビット時代のセグメントモデル、無状態の前提、C++11 の伝播規則。**
>
> **どれも、その時代には理由がありました。** そして、前提が変わった後も、互換性のために残り続けています。
>
> **C++ の規格は、次のジレンマを抱えています。**
>
> - 既存のコードを壊せない(互換性)
> - しかし、設計の誤りは修正したい
>
> **結果として、「新しい方法を足すが、古い方法も残す」** という形になります。`std::pmr` が追加されても、テンプレート版のアロケーターは残っています。
>
> **選択肢が増えることは、複雑さが増すことでもあります。**
>
> ---
>
> **私たちにとっての教訓**
>
> **① 動いていることは、正しさの証明ではない。**
>
> 42.8 節で書いたとおり、この種の問題はテストでは見つかりません。**原理を理解しておくことだけが防御です。**
>
> **② しかし、規格を厳密に守ることが常に最善とも限らない。**
>
> 42.5 節の `ChunkOf` のように、**すべての実用的な実装が使っている技法** が、規格の文面では正当化しきれないことがあります。
>
> **重要なのは、「なぜグレーなのか」を知った上で使うことです。** 知らずに使うのと、知って使うのとでは、問題が起きたときの対処が違います。
>
> **③ 規格は、いずれ現実に追いつく。**
>
> P0593 がそうでした。`std::start_lifetime_as` がそうでした。
>
> **私たちが第2章から書いてきたコードは、20 年前なら「グレー」でした。いまは、根拠を持って書けます。**
