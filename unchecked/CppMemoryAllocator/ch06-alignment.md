# 第6章 アラインメントを直す 〔v0.2〕

---

## この章のゴール

前章で `Bump` は `new` に10倍以上の差をつけました。しかし、そのアロケーターは **壊れています**。

第3章で見たこの出力を思い出してください。

```
p0 : +0     (size 4)
p1 : +4     (size 7)
p2 : +11    (size 100)
```

`p2` は板の先頭から11バイト目です。ここに `int` を置いてよいのか。`double` は。SIMD 用の型は。

この章で扱うのは **正しさの問題** です。速度の最適化ではありません。今のままでは、環境によってはクラッシュします。

- アラインメントとは何か、`alignof` / `alignas` の使い方
- 守らないと何が起きるか(3つの帰結)を、実測とテストで確認する
- `AlignUp()` を書き、`Allocate(size, alignment)` に対応させる 〔**v0.2**〕
- 16バイト境界を要求する型で検証する
- 代償として生まれる「パディングという無駄」を測る

---

## 6.1 アラインメントとは何か

CPU は、メモリを1バイトずつ気ままに読んでいるわけではありません。決まった大きさの単位で、決まった区切りから読み書きするようにできています。

**アラインメント**(alignment、整列)とは、「この型は、アドレスがこの数の倍数の位置に置かれていなければならない」という要求のことです。

C++ では `alignof` で調べられます。書いて確かめましょう。

```cpp
#include <cstddef>
#include <print>

struct Small  { char  a; };
struct Mixed  { char  a; int b; };
struct Vec4   { float x, y, z, w; };

struct alignas(16) AlignedVec4 { float x, y, z, w; };

int main()
{
    std::println("{:<16} size={:>3}  align={:>3}", "char",        sizeof(char),        alignof(char));
    std::println("{:<16} size={:>3}  align={:>3}", "short",       sizeof(short),       alignof(short));
    std::println("{:<16} size={:>3}  align={:>3}", "int",         sizeof(int),         alignof(int));
    std::println("{:<16} size={:>3}  align={:>3}", "double",      sizeof(double),      alignof(double));
    std::println("{:<16} size={:>3}  align={:>3}", "void*",       sizeof(void*),       alignof(void*));
    std::println("");
    std::println("{:<16} size={:>3}  align={:>3}", "Small",       sizeof(Small),       alignof(Small));
    std::println("{:<16} size={:>3}  align={:>3}", "Mixed",       sizeof(Mixed),       alignof(Mixed));
    std::println("{:<16} size={:>3}  align={:>3}", "Vec4",        sizeof(Vec4),        alignof(Vec4));
    std::println("{:<16} size={:>3}  align={:>3}", "AlignedVec4", sizeof(AlignedVec4), alignof(AlignedVec4));
}
```

x64 環境での結果です。

```
char             size=  1  align=  1
short            size=  2  align=  2
int              size=  4  align=  4
double           size=  8  align=  8
void*            size=  8  align=  8
                 
Small            size=  1  align=  1
Mixed            size=  8  align=  4
Vec4             size= 16  align=  4
AlignedVec4      size= 16  align= 16
```

読み取れることが3つあります。

**基本型のアラインメントは、たいていサイズと同じです。** `int` は4の倍数のアドレス、`double` は8の倍数のアドレスに置かれる必要があります。

**構造体のアラインメントは、メンバの中で最も大きいものに従います。** `Mixed` は `char` と `int` を持つので、`int` に合わせて4になります。サイズが8なのは、`char` の後ろに3バイトの詰め物が入るためです(この詰め物の話は演習に回します)。

**`alignas` で明示的に強めることができます。** `AlignedVec4` はサイズこそ `Vec4` と同じ16ですが、アラインメント要求が16になっています。SIMD 命令で扱う型で頻出する指定です。

### 「そのアドレスに置ける」とは

たとえば `int`(align=4)なら、置いてよいアドレスは次のとおりです。

```
   0    4    8   12   16   20   24 ...    ← ○ 置ける
   1  2  3  5  6  7  9 10 11 13 ...       ← × 置けない
```

第3章で見た `p2 : +11` は、**11 は4の倍数ではない** ので、`int` を置いてはいけない位置です。

---

## 6.2 守らないと何が起きるのか

「x64 では非アラインアクセスができるから大丈夫」という話を聞いたことがあるかもしれません。半分正しく、半分危険です。3つに分けて見ていきます。

### 帰結1:規格上は未定義動作

まず、C++ の規格の話です。

型 `T` のオブジェクトは、`alignof(T)` の倍数のアドレスにしか存在できません。それ以外の場所にあるものは `T` のオブジェクトではないので、`T*` として読み書きした時点で未定義動作です。

未定義動作は「たまたま動く」ことがあります。しかしそれは、コンパイラが最適化の方針を変えた瞬間に崩れます。「今は動いている」は保証になりません。

### 帰結2:速度が落ちる(実測できる)

x64 の CPU は、非アラインな読み書きを命令レベルでは受け付けます。ただし、**キャッシュラインをまたぐと** 明確に遅くなります。

キャッシュラインは 64 バイトです。CPU はメモリを 64 バイト単位でキャッシュに読み込みます。8バイトの値が 60 バイト目から始まっていると、2本のキャッシュラインにまたがることになり、CPU は2回ぶんの処理をしなければなりません。

測ってみましょう。第4章の道具を使います。

```cpp
#include <cstring>
#include <cstdint>

// 64バイトごとに 1 つの std::uint64_t を読み書きする。
// misalign を変えると、キャッシュラインをまたぐかどうかが変わる。
bench::Result MeasureU64Access(std::size_t misalign)
{
    constexpr std::size_t kBlocks = 1 << 16;   // 65536 ブロック

    std::vector<std::byte> buf(kBlocks * 64 + 256);

    // バッファ先頭を 64 バイト境界に揃える
    auto raw     = reinterpret_cast<std::uintptr_t>(buf.data());
    auto aligned = (raw + 63) & ~static_cast<std::uintptr_t>(63);

    return bench::MeasureBatch(200, kBlocks, [&, i = std::size_t{0}]() mutable {
        std::byte* p = reinterpret_cast<std::byte*>(aligned + (i % kBlocks) * 64 + misalign);

        std::uint64_t v;
        std::memcpy(&v, p, sizeof(v));      // 非アラインでも合法な読み方
        v += 1;
        std::memcpy(p, &v, sizeof(v));

        bench::Escape(static_cast<std::uintptr_t>(v));
        ++i;
    });
}

int main()
{
    bench::Print("揃っている (+0)   ", MeasureU64Access(0));
    bench::Print("ずれている (+4)   ", MeasureU64Access(4));
    bench::Print("ライン跨ぎ (+60)  ", MeasureU64Access(60));
}
```

結果の例です。

```
揃っている (+0)     median=      1.1  p95=      1.2  max=        4.8   (mean=      1.1)
ずれている (+4)     median=      1.2  p95=      1.3  max=        5.1   (mean=      1.2)
ライン跨ぎ (+60)    median=      2.3  p95=      2.5  max=        7.9   (mean=      2.4)
```

- **+4 のずれ**:ほとんど変わりません。64 バイトの内側に収まっているためです
- **+60(ライン跨ぎ)**:**約2倍** 遅くなりました

「x64 では非アラインでも動く」は本当です。しかしタダではありません。そして、アロケーターが半端なアドレスを返し続ければ、ライン跨ぎは確率的に起き続けます。

> **`std::memcpy` を使った理由**
> 非アラインなアドレスを `std::uint64_t*` にキャストして読むのは未定義動作です。`std::memcpy` はバイト単位のコピーとして定義されているので合法で、しかもコンパイラは十分に最適化してくれます。非アラインなデータを扱う正しい方法として覚えておいてください。

### 帰結3:落ちる

ここからは「遅い」では済まない話です。

**SIMD 型。** `__m128`(16バイト)を扱う命令のうち、`movaps` などのアラインド版は、アドレスが16の倍数でないと **例外を発生させます**。`alignas(16)` を付けた型を半端な位置に置き、SIMD で読もうとした瞬間にアクセス違反です。

**`std::atomic`。** アトミック操作がロックフリーであることの前提は、対象が適切に整列していることです。整列していなければ、ロックフリー性が失われるか、正しく動きません。

**他のアーキテクチャ。** ARM の一部の命令、特に複数レジスタを一度に扱う命令は、非アラインアクセスでフォルトします。x64 では動いていたコードが、移植した先で落ちる——ゲーム開発では現実的なシナリオです。

**まとめると:**

| 帰結 | x64 での症状 |
|---|---|
| 規格上の未定義動作 | 今は動くかもしれない |
| キャッシュライン跨ぎ | 2倍程度遅くなる |
| SIMD / atomic / 他アーキテクチャ | **落ちる** |

直しましょう。

---

## 6.3 `AlignUp` を書く

やることは単純です。ある数を、指定した倍数まで **切り上げる**。

```
現在位置 11、要求アラインメント 4
  → 次の4の倍数は 12
  → 1バイト分の詰め物(パディング)を入れて 12 から使う
```

素直に書くとこうなります。

```cpp
std::uintptr_t AlignUpSlow(std::uintptr_t value, std::size_t alignment)
{
    const std::uintptr_t remainder = value % alignment;
    return (remainder == 0) ? value : value + (alignment - remainder);
}
```

正しく動きますが、剰余算(`%`)は除算命令を使うので重い処理です。アロケーターの最も内側で毎回呼ばれることを考えると、避けたいところです。

### 2の冪であることを利用する

アラインメントは、必ず **2の冪** です(1, 2, 4, 8, 16, 32, ...)。これは C++ の規格が保証しています。

2の冪なら、ビット演算で書けます。

```cpp
#include <bit>
#include <cassert>
#include <cstdint>

constexpr std::uintptr_t AlignUp(std::uintptr_t value, std::size_t alignment) noexcept
{
    // アラインメントは 2 の冪でなければならない
    assert(std::has_single_bit(alignment));

    const std::uintptr_t mask = static_cast<std::uintptr_t>(alignment) - 1;
    return (value + mask) & ~mask;
}
```

`std::has_single_bit`(C++20、`<bit>`)は「立っているビットが1本だけか」を調べる関数です。2の冪であることの検査そのものです。

### なぜこれで動くのか

`alignment = 16` の場合を追いかけます。

```
mask  = 16 - 1 = 15 = 0b0000'1111
~mask =              0b1111'0000
```

`~mask` との `&` は、**下位4ビットを切り落とす**、つまり16の倍数に切り下げる操作です。

切り下げる前に `mask` を足しておけば、切り上げになります。

```
value = 11 = 0b0000'1011
value + 15 = 26 = 0b0001'1010
       & ~15    = 0b0001'0000 = 16   ✓

value = 16 = 0b0001'0000
value + 15 = 31 = 0b0001'1111
       & ~15    = 0b0001'0000 = 16   ✓(すでに揃っていれば変わらない)
```

加算1回、`&` 1回。除算命令は登場しません。

### テストを書く

```cpp
void Test_AlignUp()
{
    assert(AlignUp(0,  4) == 0);
    assert(AlignUp(1,  4) == 4);
    assert(AlignUp(3,  4) == 4);
    assert(AlignUp(4,  4) == 4);
    assert(AlignUp(5,  4) == 8);

    assert(AlignUp(11, 4)  == 12);
    assert(AlignUp(11, 8)  == 16);
    assert(AlignUp(11, 16) == 16);
    assert(AlignUp(17, 16) == 32);

    assert(AlignUp(100, 1) == 100);   // align=1 なら何もしない

    std::println("[ OK ] Test_AlignUp");
}
```

`alignment == 1` のとき `mask == 0` なので `(value + 0) & ~0` となり、値がそのまま返ります。境界値としてきちんと動きます。

---

## 6.4 `Allocate` に組み込む 〔v0.2〕

### 重要:揃えるのは「オフセット」ではなく「アドレス」

ここで間違えやすい点があります。次のように書きたくなるかもしれません。

```cpp
offset_ = AlignUp(offset_, alignment);   // ← 危ない
```

これは **オフセットを揃えている** だけです。板の先頭アドレスが 16 の倍数でなければ、オフセットを 16 の倍数にしても、実際のアドレスは 16 の倍数になりません。

```
buffer_.data() = 0x...08   ← 先頭が 8 の倍数でしかない
offset_ = 16
実際のアドレス = 0x...18   ← 16 の倍数ではない!
```

`std::vector<std::byte>` の先頭アドレスがどこまで揃っているかは、保証が限られています。標準のアロケーターは `operator new` を使うので、通常は `__STDCPP_DEFAULT_NEW_ALIGNMENT__`(MSVC の x64 では 16)までは期待できますが、32 や 64 バイト境界は保証されません。

**必ず実アドレスを揃えてください。**

### 実装

```cpp
class Bump
{
public:
    explicit Bump(std::size_t capacity)
        : buffer_(capacity)
    {
    }

    void* Allocate(std::size_t size,
                   std::size_t alignment = kDefaultAlignment)
    {
        assert(std::has_single_bit(alignment));

        const auto base    = reinterpret_cast<std::uintptr_t>(buffer_.data());
        const auto current = base + offset_;
        const auto aligned = AlignUp(current, alignment);

        const std::size_t padding = static_cast<std::size_t>(aligned - current);

        offset_  += padding + size;
        padding_ += padding;          // 無駄になったバイト数を記録

        return reinterpret_cast<void*>(aligned);
    }

    std::size_t Used()      const noexcept { return offset_; }
    std::size_t Capacity()  const noexcept { return buffer_.size(); }
    std::size_t Remaining() const noexcept { return Capacity() - Used(); }

    // パディングで捨てられたバイト数
    std::size_t Padding()   const noexcept { return padding_; }

    const std::byte* Base() const noexcept { return buffer_.data(); }

private:
    std::vector<std::byte> buffer_;
    std::size_t            offset_  = 0;
    std::size_t            padding_ = 0;
};
```

`Allocate` は3行から6行になりました。増えたのは、

- 現在のアドレスを求める
- 切り上げる
- パディング量を計算して足す

だけです。除算も分岐もありません。第5章で測った 1.8 ns からの悪化は、ごくわずかなはずです(演習6-4で確かめてください)。

### 既定のアラインメントを決める

```cpp
inline constexpr std::size_t kDefaultAlignment = __STDCPP_DEFAULT_NEW_ALIGNMENT__;
```

`__STDCPP_DEFAULT_NEW_ALIGNMENT__` は、「`operator new` が何も言わなくても保証してくれるアラインメント」を表す規格のマクロです。MSVC の x64 では **16** です。

`Bump` の既定値をこれに合わせておけば、「`new` で確保していたものを `Bump` に置き換える」だけで済みます。控えめな値(たとえば 8)にすると、`new` なら通っていたコードが `Bump` では壊れる、という嫌な差が生まれます。

**アラインメントを指定するには:**

```cpp
void* p = bump.Allocate(sizeof(T), alignof(T));
```

毎回この2つを書くのは面倒です。第10章で `New<T>()` を作るとき、`sizeof` と `alignof` を自動で埋めるようにします。

---

## 6.5 既存のテストが落ちる

ここで、第3章で書いたテストを実行してください。

```
Assertion failed: bump.Used() == 11, file ...\Playground.cpp, line 58
```

`Test_UsedIncreases` が落ちます。

```cpp
bump.Allocate(4);
assert(bump.Used() == 4);      // ← まだ通る

bump.Allocate(7);
assert(bump.Used() == 11);     // ← 落ちる
```

既定のアラインメントが16になったので、2回目の確保は 4 バイト目ではなく **16 バイト目** から始まります。`Used()` は 11 ではなく 23 です。

**これはバグではなく、仕様変更です。** テストを直します。

```cpp
void Test_UsedIncreases()
{
    Bump bump(1024);

    bump.Allocate(4, 1);          // アラインメント1を明示 → 詰め物なし
    assert(bump.Used() == 4);

    bump.Allocate(7, 1);
    assert(bump.Used() == 11);

    bump.Allocate(100, 1);
    assert(bump.Used() == 111);
    assert(bump.Padding() == 0);

    std::println("[ OK ] Test_UsedIncreases");
}
```

`alignment = 1` を明示することで、以前と同じ「詰めるだけ」の動作になります。

同様に `Test_AddressesAreContiguous` も、アラインメント1を指定するか、パディングを織り込んだ期待値に直してください。

> **仕様を変えたらテストが落ちるのは、テストが正しく働いている証拠です。**
> 落ちなかったら、そのテストは何も検査していません。第3章で「必ず一度は失敗させろ」と書いたのは、このためでもあります。

---

## 6.6 アラインメントを検査するテストを足す

本題のテストです。

```cpp
// アドレスが指定のアラインメントに揃っているか
bool IsAlignedTo(const void* p, std::size_t alignment) noexcept
{
    return (reinterpret_cast<std::uintptr_t>(p) % alignment) == 0;
}

struct alignas(16) Vec4  { float  x, y, z, w; };
struct alignas(32) Mat2x4{ double m[8]; };

void Test_AlignmentIsRespected()
{
    Bump bump(4096);

    // 1バイト確保して、わざと位置をずらす
    bump.Allocate(1, 1);

    // その状態から各種の型を確保する
    void* pInt   = bump.Allocate(sizeof(int),    alignof(int));
    bump.Allocate(1, 1);
    void* pDbl   = bump.Allocate(sizeof(double), alignof(double));
    bump.Allocate(1, 1);
    void* pVec4  = bump.Allocate(sizeof(Vec4),   alignof(Vec4));
    bump.Allocate(1, 1);
    void* pMat   = bump.Allocate(sizeof(Mat2x4), alignof(Mat2x4));

    assert(IsAlignedTo(pInt,  alignof(int)));
    assert(IsAlignedTo(pDbl,  alignof(double)));
    assert(IsAlignedTo(pVec4, 16));
    assert(IsAlignedTo(pMat,  32));

    std::println("[ OK ] Test_AlignmentIsRespected");
}
```

各確保の前に `Allocate(1, 1)` を挟んでいるのがポイントです。こうすると位置が必ず半端になるので、切り上げ処理が確実に働く状況を作れます。

### v0.1 でこのテストを走らせると

`Bump` を第5章の版に戻して、このテストを実行してみてください。

```
Assertion failed: IsAlignedTo(pInt, alignof(int))
```

1つ目で落ちます。**v0.1 は、このテストを1つも通りません。**

第5章で「10倍速い」と喜んだアロケーターは、`int` すら正しく置けなかったわけです。速度の前に正しさ、という話の実例になりました。

### 実際に SIMD で読んでみる(任意)

もう少し実感したい人向けの実験です。

```cpp
#include <immintrin.h>

void ExperimentSimd()
{
    Bump bump(4096);

    bump.Allocate(1, 1);                      // 位置をずらす
    void* p = bump.Allocate(sizeof(Vec4), 1); // ← アラインメントを無視して確保

    std::println("アドレス: {} (16の倍数? {})", p, IsAlignedTo(p, 16));

    // アラインド版のロード命令。16の倍数でないとアクセス違反になりうる
    __m128 v = _mm_load_ps(static_cast<const float*>(p));
    _mm_store_ps(static_cast<float*>(p), v);

    std::println("SIMD アクセス完了");
}
```

環境やコンパイラの命令選択によっては、ここでプログラムが落ちます。落ちなかった場合でも、それは「たまたま `movups`(非アラインド版)が選ばれた」だけかもしれません。**動いたことは、正しいことの証明にはなりません。**

---

## 6.7 代償:パディングという無駄

アラインメントを守ると、必ず捨てるバイトが出ます。`Padding()` を足したのはこれを見るためです。

```cpp
void ExperimentPaddingWaste()
{
    // 1バイトの確保を、16バイト境界で 10000 回
    Bump bump(1024 * 1024);

    for (int i = 0; i < 10'000; ++i)
    {
        bump.Allocate(1, 16);
    }

    std::println("要求された合計 : {:>8} バイト", 10'000);
    std::println("実際に使った量 : {:>8} バイト", bump.Used());
    std::println("パディングの無駄: {:>8} バイト ({:.1f}%)",
                 bump.Padding(),
                 100.0 * bump.Padding() / bump.Used());
}
```

```
要求された合計 :    10000 バイト
実際に使った量 :   160000 バイト
パディングの無駄:   150000 バイト (93.8%)
```

**93.8% が無駄になりました。**

1バイトのデータを置くたびに15バイト捨てているので当然です。極端な例ですが、示唆はあります。

> **アラインメントは正しさのために必要だが、メモリを浪費する。**

この「要求サイズより多く使ってしまう無駄」を **内部断片化**(internal fragmentation)と呼びます。第20章以降で登場する「外部断片化」(空きが細切れになって使えなくなる現象)とは別物なので、区別して覚えてください。

### 減らす方法

無駄を減らす一般的な手は2つあります。

**1. 必要以上に強いアラインメントを要求しない**

上の例で `alignof(char) == 1` を渡していれば、無駄はゼロでした。既定値の16は安全側に倒した値です。型が分かっているなら `alignof(T)` を渡すべきです(第10章の `New<T>()` はこれを自動化します)。

**2. 大きいアラインメントを要求するものから先に確保する**

確保の順序を変えるだけで、パディングが減ります。

```cpp
// 悪い例:1バイト → 16バイト境界 → 1バイト → 16バイト境界 …
// 良い例:16バイト境界のものをまとめて → 1バイトのものをまとめて
```

構造体のメンバ順を大きい型から並べると `sizeof` が縮むのと同じ理屈です(演習6-1)。

---

## 6.8 この章の完成コード

```cpp
#include <bit>
#include <cassert>
#include <cstddef>
#include <cstdint>
#include <print>
#include <vector>

// ---------------------------------------------------------
// アラインメント計算
// ---------------------------------------------------------
inline constexpr std::size_t kDefaultAlignment = __STDCPP_DEFAULT_NEW_ALIGNMENT__;

constexpr std::uintptr_t AlignUp(std::uintptr_t value, std::size_t alignment) noexcept
{
    assert(std::has_single_bit(alignment));

    const std::uintptr_t mask = static_cast<std::uintptr_t>(alignment) - 1;
    return (value + mask) & ~mask;
}

inline bool IsAlignedTo(const void* p, std::size_t alignment) noexcept
{
    return (reinterpret_cast<std::uintptr_t>(p) % alignment) == 0;
}

// ---------------------------------------------------------
// Bump アロケーター v0.2 — アラインメント対応
// ---------------------------------------------------------
class Bump
{
public:
    explicit Bump(std::size_t capacity)
        : buffer_(capacity)
    {
    }

    void* Allocate(std::size_t size,
                   std::size_t alignment = kDefaultAlignment)
    {
        assert(std::has_single_bit(alignment));

        const auto base    = reinterpret_cast<std::uintptr_t>(buffer_.data());
        const auto current = base + offset_;
        const auto aligned = AlignUp(current, alignment);

        const std::size_t padding = static_cast<std::size_t>(aligned - current);

        offset_  += padding + size;
        padding_ += padding;

        return reinterpret_cast<void*>(aligned);
    }

    std::size_t Used()      const noexcept { return offset_; }
    std::size_t Capacity()  const noexcept { return buffer_.size(); }
    std::size_t Remaining() const noexcept { return Capacity() - Used(); }
    std::size_t Padding()   const noexcept { return padding_; }

    const std::byte* Base() const noexcept { return buffer_.data(); }

private:
    std::vector<std::byte> buffer_;
    std::size_t            offset_  = 0;
    std::size_t            padding_ = 0;
};
```

---

## 演習

**演習6-1** 次の構造体の `sizeof` を予想してから、実際に確かめてください。メンバの順序を入れ替えて最小にできますか。

```cpp
struct A { char a; double b; char c; int d; };
```

**演習6-2** `AlignUp` に、2の冪でない値(たとえば 3)を渡すとどうなりますか。Debug 構成と Release 構成で挙動を比べてください。Release で守るにはどうすればよいでしょうか。

**演習6-3** `AlignDown`(切り下げ)を書いてください。どんな場面で必要になりそうですか。

**演習6-4** 第5章のベンチマークを v0.2 で走らせ、v0.1 と比べてください。アラインメント処理を足したコストはどれくらいですか。

**演習6-5** `MeasureU64Access` の `misalign` を 0 から 63 まで全部試し、結果を並べてください。どこで急に遅くなりますか。予想と一致しますか。

**演習6-6** `Allocate` の直後に `assert(IsAlignedTo(result, alignment))` を入れてください。これは有用なテストでしょうか、それとも実装をそのまま書き写しただけの無意味なテストでしょうか。理由も考えてください。

---

## 章末チェックリスト

- [ ] `alignof` で各種の型のアラインメント要求を確認した
- [ ] キャッシュライン跨ぎのコストを実測した
- [ ] `AlignUp` をビット演算で実装し、テストを書いた
- [ ] **オフセットではなくアドレスを揃える** 理由を説明できる
- [ ] `Allocate(size, alignment)` に対応させた 〔v0.2〕
- [ ] `alignas(16)` / `alignas(32)` の型でテストが通った
- [ ] v0.1 では同じテストが落ちることを確認した
- [ ] パディングによる無駄(内部断片化)を測った

---

## 次章の予告

`Bump` は正しいアドレスを返すようになりました。しかし、まだ致命的な欠陥が残っています。

**容量を超えても、何も言わずに壊れる。**

第3章の実験で見たとおり、16バイトの板から400バイトを切り出しても、エラーは出ませんでした。返ってくるのは板の外のアドレスで、そこに書き込めば無関係なメモリが壊れます。

第7章では、これを型で表現します。C++23 の `std::expected` を使い、「確保は失敗しうる」という事実をシグネチャに書きます。`nullptr` を返すのか、例外を投げるのか、`expected` を返すのか——アロケーターの設計者が必ず一度は悩む選択について、腰を据えて考えます。

---

> **コラム:なぜハードウェアは整列を要求するのか**
>
> CPU がメモリを読むとき、実際にはバス幅の単位(64ビットや128ビット)でまとめて読み込みます。整列していれば、必要なデータはその1回に収まります。整列していなければ、2回読んで、繋ぎ合わせる必要があります。
>
> 初期の CPU は、この繋ぎ合わせをハードウェアで面倒みてくれませんでした。非アラインアクセスは即座に例外です。x86 系は互換性を重んじる文化から、早い段階でハードウェアによる救済を実装しました。おかげで「x86 では非アラインでも動く」という状況が生まれ、それが今日まで続いています。
>
> ---
>
> しかし SIMD の登場で、話が戻ります。
>
> 1999年の SSE で導入された `movaps` は、16 バイト境界を要求し、外れれば例外を投げました。非アラインド版の `movups` もありましたが、当時は目に見えて遅く、「SIMD を使うなら 16 バイト境界に揃えるのは開発者の責任」という文化が定着しました。ゲームエンジンで `alignas(16)` のベクトル型を見かけるのは、この時代の名残です。
>
> その後 Nehalem(2008年)以降、`movups` の性能は改善され、揃っている場合はアラインド版とほぼ同等になりました。それでもライン跨ぎのコストは残りますし、`movaps` を使うコードは今も大量にあります。**ハードウェアが優しくなっても、正しく揃える理由はなくなっていません。**
>
> ---
>
> ゲーム開発でアラインメントが特に重要なのは、扱うデータの性質もあります。
>
> 頂点データ、行列、ベクトル、テクスチャ。どれも SIMD で一括処理される候補であり、そして GPU に渡されるデータでもあります。GPU 側の要求はさらに厳しく、たとえば Direct3D 12 の定数バッファは 256 バイト境界を要求します。第48章で GPU メモリを扱うとき、この `AlignUp` がそのまま活躍します。
>
> 20行のアロケーターに足した6行は、ここまでずっと効き続けます。
