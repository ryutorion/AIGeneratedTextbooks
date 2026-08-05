# 第7章 溢れたときにどうするか 〔v0.3〕

---

## この章のゴール

`Bump` は速く、アドレスも正しく揃うようになりました。しかし、まだ最大の欠陥が残っています。

**容量を超えても、何も言わずに壊れる。**

第3章で確認したとおりです。16バイトの板から400バイトを切り出しても、エラーは出ませんでした。返ってくるのは板の外のアドレスで、そこに書き込めば無関係なメモリが破壊されます。

この章では、これを型で表現します。

- 溢れたときに実際に何が壊れるかを、目で見る
- エラーの伝え方の選択肢を4つ比較する(`nullptr` / 例外 / `std::expected` / 即死)
- C++23 の `std::expected<void*, AllocError>` に書き換える 〔**v0.3**〕
- 符号なし整数の罠を避けた、正しい溢れ判定を書く
- 呼び出し側をすべて書き換え、そのコストを測る

「失敗しうる関数のシグネチャをどう書くか」は、アロケーターに限らず C++ の設計で必ず一度は悩む問題です。腰を据えて考えます。

---

## 7.1 溢れると何が壊れるのか

抽象的な話の前に、実際に壊してみましょう。**Debug 構成** で実行してください。

```cpp
int main()
{
    {
        Bump bump(64);          // 64バイトしかない

        // 容量を大きく超えて確保し、書き込む
        for (int i = 0; i < 100; ++i)
        {
            auto* p = static_cast<int*>(bump.Allocate(sizeof(int), alignof(int)));
            *p = i;
        }

        std::println("書き込み完了。ここまでは何も起きない。");
    }   // ← ここで bump が破棄される

    std::println("プログラム終了");
}
```

実行すると、こうなります。

```
書き込み完了。ここまでは何も起きない。
```

そして、その直後にダイアログが出ます。

```
HEAP CORRUPTION DETECTED: after Normal block (#xxx) at 0x...
CRT detected that the application wrote to memory after end of heap buffer.
```

### 何が起きたか

`Bump` は 64 バイトの板に 400 バイト書き込みました。あふれた 336 バイトは、`std::vector` が確保したヒープブロックの **外側** に書き込まれています。

そこには、CRT のデバッグヒープが置いている検査用のバイト列がありました。`std::vector` のデストラクタがメモリを返却するとき、CRT はその検査用バイト列を調べ、破壊されていることに気づいてダイアログを出しました。

### 注目すべき点

**エラーが出たのは、書き込んだ瞬間ではありません。** スコープを抜けて `std::vector` が解放されるときです。

この例では数行しか離れていませんが、実際のプログラムでは何秒も、何分も後になります。しかも、報告される場所は破壊した場所ではなく、**破壊された側** です。デバッガが指すのは、まったく無関係なコードです。

さらに悪いことに、Release 構成ではこの検査用バイト列がありません。**エラーは出ず、静かに何かが壊れます。**

> **これが「メモリ破壊バグが恐れられる」理由です。**
> 原因と結果が、時間的にも空間的にも離れます。

第36章で AddressSanitizer を有効にすると、書き込んだ瞬間に止められるようになります。しかしそれは検出の話であって、根本の解決ではありません。**アロケーターが、そもそも板の外を返さないようにする** のが正しい直し方です。

---

## 7.2 失敗をどう伝えるか:4つの選択肢

`Allocate` が「できません」と言う方法は、大きく4つあります。それぞれ実際に使われている流儀です。

### 選択肢1:`nullptr` を返す

C の伝統です。`malloc` がこれです。

```cpp
void* Allocate(std::size_t size, std::size_t alignment);   // 失敗したら nullptr
```

| 長所 | 短所 |
|---|---|
| コストがゼロ | **チェックを忘れても警告が出ない** |
| 実装が単純 | 失敗の理由が分からない |
| どこでも使える | `nullptr` を返す他の意味と区別できない |

最大の問題は、無視できてしまうことです。

```cpp
void* p = bump.Allocate(1000);
std::memset(p, 0, 1000);          // p が nullptr でもコンパイルは通る
```

### 選択肢2:例外を投げる

C++ の伝統です。`operator new` は失敗すると `std::bad_alloc` を投げます。

```cpp
void* Allocate(std::size_t size, std::size_t alignment);   // 失敗したら throw
```

| 長所 | 短所 |
|---|---|
| **無視できない** | 巻き戻し時間が予測できない |
| 成功時のコストがゼロ | `noexcept` の関数から呼べない |
| エラー情報を運べる | 例外を禁止している現場が多い |

ゲーム開発で例外が敬遠される理由は、主に「巻き戻し(スタックアンワインド)にかかる時間が読めない」点です。第2章で見たとおり、リアルタイムソフトでは最悪実行時間が予測できることに価値があります。

また、多くのゲームエンジンやコンソール向けのプロジェクトでは、コードサイズと性能のために例外を無効化しています。そうした環境で使えるアロケーターにしたいなら、例外は選べません。

### 選択肢3:`std::expected` を返す

C++23 で追加された型です。「成功したら値、失敗したらエラー」を1つの戻り値で表します。

```cpp
std::expected<void*, AllocError> Allocate(std::size_t size, std::size_t alignment);
```

| 長所 | 短所 |
|---|---|
| 失敗の可能性が **シグネチャに書いてある** | 呼び出し側が冗長になる |
| `[[nodiscard]]` で無視を防げる | 戻り値が大きくなる |
| エラーの理由を運べる | C++23 が必要 |
| 例外も巻き戻しも使わない | |

### 選択肢4:失敗を許さない(即死)

意外に思うかもしれませんが、実際のゲーム開発で **最もよく採られる** 選択肢です。

```cpp
void* Allocate(std::size_t size, std::size_t alignment)
{
    if (足りない) { FatalError("out of memory"); }   // ログを吐いて即終了
    ...
}
```

考え方はこうです。

> ゲームのメモリ量は事前に決まっている。足りなくなったのは設計ミスであって、実行時に回復する話ではない。ならば、その場で落として、開発中に直すべきだ。

第49章でメモリ予算の話をするとき、この考え方に戻ってきます。「予算超過は実行時エラーではなく、開発時に潰すバグである」という文化は、据置機の固定メモリという制約の中で育ちました。

---

## 7.3 本書が `std::expected` を選ぶ理由

4つのうち、本書は **選択肢3** を採ります。理由は3つです。

**1. 失敗の可能性が型に現れる。** シグネチャを見れば、この関数が失敗しうることが分かります。ドキュメントを読む必要も、コメントを信じる必要もありません。

**2. 学習用途に向いている。** この本を読みながら書くコードでは、溢れは頻繁に起きます。そのたびに静かに壊れるより、明示的にエラーが返るほうが、何が起きたか理解しやすい。

**3. 他の選択肢に変換しやすい。** `expected` を返す実装があれば、その上に「失敗したら即死する薄いラッパー」を被せるのは簡単です。逆は難しい。

```cpp
// 即死版が欲しければ、こう被せるだけ
void* AllocateOrDie(std::size_t size, std::size_t alignment)
{
    auto r = bump.Allocate(size, alignment);
    if (!r) { FatalError(...); }
    return *r;
}
```

つまり `expected` は、**最も情報量の多い基底** です。そこから他の流儀へは下りられます。

> **実際の製品で何を選ぶかは、プロジェクトの方針次第です。** 本書の選択が唯一の正解ではありません。ただ、選択肢の性質を理解したうえで選んでほしい、というのがこの節の趣旨です。

---

## 7.4 エラー型を定義する

まずエラーの種類を決めます。

```cpp
#include <expected>

enum class AllocError
{
    OutOfMemory,        // 容量が足りない
    InvalidAlignment,   // アラインメントが2の冪でない
    SizeTooLarge,       // サイズの計算が桁溢れする
};

constexpr const char* ToString(AllocError e) noexcept
{
    switch (e)
    {
    case AllocError::OutOfMemory:      return "OutOfMemory";
    case AllocError::InvalidAlignment: return "InvalidAlignment";
    case AllocError::SizeTooLarge:     return "SizeTooLarge";
    }
    return "Unknown";
}
```

`enum class` を使っているので、`int` に暗黙変換されません。エラーコードの取り違えを防げます。

`ToString` はデバッグ表示用です。第14章でログ機能を作るとき、そのまま使えます。

### 戻り値の型に別名を付ける

毎回 `std::expected<void*, AllocError>` と書くのは長いので、別名を用意します。

```cpp
using AllocResult = std::expected<void*, AllocError>;
```

---

## 7.5 溢れ判定を正しく書く

ここが技術的に一番注意の要る部分です。**素直に書くと、間違えます。**

### 間違い1:第3章の罠

```cpp
if (Remaining() >= size) { /* 確保できる */ }   // ← バグ
```

第3章で見たとおり、`Remaining()` は `Capacity() - Used()` です。すでに溢れていると符号なしの引き算がラップアラウンドし、天文学的な値になります。結果、**溢れているときほど「確保できる」と判定されます。**

### 間違い2:足してから比べる

```cpp
if (offset_ + padding + size <= Capacity()) { /* 確保できる */ }   // ← まだ危ない
```

`size` に巨大な値(たとえば `SIZE_MAX`)が渡されると、`offset_ + padding + size` が桁溢れして小さな値になります。結果、やはり「確保できる」と判定されます。

呼び出し側が意図的に巨大な値を渡すことは稀ですが、**計算の結果として大きな値が入り込む** ことは十分あります。

```cpp
std::size_t count = 0;
// ... 何かの計算で count が 0 になった ...
bump.Allocate((count - 1) * sizeof(T));   // (0-1) = SIZE_MAX
```

### 正しい書き方:引き算を安全な向きにする

**足して比べるのではなく、引いて比べます。** ただし、引く前に引ける状態か確かめます。

```cpp
// 1) アラインメント後の位置が、そもそも板を超えていないか
if (padding > capacity - offset_) → OutOfMemory

// 2) 残りに size が収まるか
const std::size_t alignedOffset = offset_ + padding;
if (size > capacity - alignedOffset) → OutOfMemory
```

どちらの引き算も、**引かれる数のほうが大きいことを事前に保証してから** 行っています。

- 1つ目:`offset_ <= capacity` は不変条件として保たれている(確保に成功したときしか `offset_` を進めないため)
- 2つ目:1つ目を通過した時点で `alignedOffset <= capacity` が確定している

ラップアラウンドは起きません。

> **符号なし整数の引き算を書くときは、毎回こう問いかけてください。**
> 「引かれる数のほうが大きいことを、私は保証できているか?」

---

## 7.6 実装する 〔v0.3〕

```cpp
class Bump
{
public:
    explicit Bump(std::size_t capacity)
        : buffer_(capacity)
    {
    }

    [[nodiscard]]
    AllocResult Allocate(std::size_t size,
                         std::size_t alignment = kDefaultAlignment) noexcept
    {
        // --- 引数の検査 ---
        if (!std::has_single_bit(alignment))
        {
            return std::unexpected(AllocError::InvalidAlignment);
        }

        const std::size_t capacity = buffer_.size();

        // --- アラインメントによる詰め物を計算 ---
        const auto base    = reinterpret_cast<std::uintptr_t>(buffer_.data());
        const auto current = base + offset_;
        const auto aligned = AlignUp(current, alignment);

        const std::size_t padding = static_cast<std::size_t>(aligned - current);

        // --- 溢れ判定(引き算の向きに注意)---
        if (padding > capacity - offset_)
        {
            return std::unexpected(AllocError::OutOfMemory);
        }

        const std::size_t alignedOffset = offset_ + padding;

        if (size > capacity - alignedOffset)
        {
            return std::unexpected(AllocError::OutOfMemory);
        }

        // --- ここまで来たら成功が確定 ---
        offset_   = alignedOffset + size;
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

### 設計上のポイント

**`[[nodiscard]]` を付けた。** 戻り値を捨てると警告が出ます。`expected` を無視することは、エラーを無視することと同義なので、これは重要です。

```cpp
bump.Allocate(100);   // warning C4834: '[[nodiscard]]' 属性を持つ関数の戻り値を破棄しています
```

**`noexcept` を付けた。** この関数はもう例外を投げません。エラーは戻り値で運びます。呼び出し側は、例外安全性を考える必要がなくなります。

**状態を変えるのは最後だけ。** 検査をすべて通過してから `offset_` と `padding_` を更新しています。途中で失敗しても、`Bump` の状態は一切変わりません。

> これを **強い例外安全性**(あるいは commit-or-rollback)と呼びます。「失敗したら、何も起きなかったことにする」という保証です。エラーからの回復を書く側にとって、この保証があるかないかは大違いです。

**`Remaining()` は直していない。** 溢れが起きなくなったので、`offset_ <= capacity` が常に成り立ちます。ラップアラウンドはもう起きません。第3章で見た問題は、根元から消えました。

### `assert` を落とした理由

第6章では `assert(std::has_single_bit(alignment))` を書いていました。v0.3 ではこれを `InvalidAlignment` エラーに変えています。

`assert` は Release で消えます。「Debug では検出できるが Release では黙って壊れる」という状態を、エラー処理を導入した今も残しておく理由はありません。

---

## 7.7 呼び出し側を書き換える

戻り値の型が変わったので、**これまでのコードは全部コンパイルが通らなくなります。** 正直に向き合いましょう。

### 基本の書き方

```cpp
auto r = bump.Allocate(sizeof(int), alignof(int));

if (!r)
{
    std::println("確保失敗: {}", ToString(r.error()));
    return;
}

int* p = static_cast<int*>(*r);   // * で中身を取り出す
*p = 42;
```

`expected` の主な操作です。

| 書き方 | 意味 |
|---|---|
| `r.has_value()` / `if (r)` | 成功したか |
| `*r` / `r.value()` | 値を取り出す |
| `r.error()` | エラーを取り出す |
| `r.value_or(nullptr)` | 失敗なら既定値 |

`*r` は検査なしで取り出します。失敗している状態で呼ぶと未定義動作です。`r.value()` は失敗していると `std::bad_expected_access` を投げます。**確認済みなら `*r`、そうでなければ `value_or`** と使い分けてください。

### モナディック操作

C++23 の `expected` には、成功時だけ処理を繋ぐ関数があります。

```cpp
auto r = bump.Allocate(sizeof(int), alignof(int))
             .transform([](void* p) { return static_cast<int*>(p); });

// r の型は std::expected<int*, AllocError>
```

`transform` は成功時だけ関数を適用し、失敗ならエラーをそのまま素通しします。`and_then` は、関数自体が `expected` を返す場合に使います。

エラー処理の分岐を書かずに済むので、確保を連鎖させるときに読みやすくなります。ただし、慣れないうちは素直に `if (!r)` で書いても構いません。

### テストを直す

第3章と第6章のテストを、新しいシグネチャに合わせます。

```cpp
void Test_UsedIncreases()
{
    Bump bump(1024);

    assert(bump.Allocate(4, 1).has_value());
    assert(bump.Used() == 4);

    assert(bump.Allocate(7, 1).has_value());
    assert(bump.Used() == 11);

    assert(bump.Allocate(100, 1).has_value());
    assert(bump.Used() == 111);

    std::println("[ OK ] Test_UsedIncreases");
}
```

> **注意:`assert` の中に確保を書いてしまった!**
> 第3章で「`assert` の中に副作用を書くな」と警告したばかりです。この書き方は Release で確保ごと消えます。
>
> テスト関数は Debug でしか走らせない前提なので実害はありませんが、危うい書き方であることは意識してください。正しくはこうです。
>
> ```cpp
> const bool ok = bump.Allocate(4, 1).has_value();
> assert(ok);
> ```

### ベンチマークを直す

第5章の `BenchBump` も直します。

```cpp
for (std::size_t i = 0; i < count; ++i)
{
    auto r = bump.Allocate(size);
    bench::Escape(r.value_or(nullptr));
}
```

---

## 7.8 わざと溢れさせる

いよいよ本題の検証です。

```cpp
void Test_OverflowIsDetected()
{
    Bump bump(64);

    // 16バイト × 4 = 64バイト。ぴったり使い切る
    for (int i = 0; i < 4; ++i)
    {
        auto r = bump.Allocate(16, 16);
        assert(r.has_value());
    }

    assert(bump.Used() == 64);
    assert(bump.Remaining() == 0);

    // 5個目は必ず失敗する
    auto r = bump.Allocate(16, 16);

    assert(!r.has_value());
    assert(r.error() == AllocError::OutOfMemory);

    // 失敗しても状態は変わっていない(強い保証)
    assert(bump.Used() == 64);

    std::println("[ OK ] Test_OverflowIsDetected");
}

void Test_InvalidAlignmentIsDetected()
{
    Bump bump(1024);

    auto r = bump.Allocate(16, 3);   // 3 は2の冪ではない

    assert(!r.has_value());
    assert(r.error() == AllocError::InvalidAlignment);

    std::println("[ OK ] Test_InvalidAlignmentIsDetected");
}

void Test_HugeSizeIsDetected()
{
    Bump bump(1024);

    // 桁溢れを誘う巨大な値
    auto r = bump.Allocate(std::numeric_limits<std::size_t>::max(), 1);

    assert(!r.has_value());
    assert(r.error() == AllocError::OutOfMemory);

    std::println("[ OK ] Test_HugeSizeIsDetected");
}
```

第3章の 3.8 節で「このテストは今は失敗します」と書いたものが、ついに通るようになりました。

### 7.1 節の実験をやり直す

冒頭の「64バイトに400バイト書き込む」プログラムを、v0.3 で書き直します。

```cpp
int main()
{
    Bump bump(64);

    int successCount = 0;
    int failCount    = 0;

    for (int i = 0; i < 100; ++i)
    {
        auto r = bump.Allocate(sizeof(int), alignof(int));

        if (r)
        {
            *static_cast<int*>(*r) = i;
            ++successCount;
        }
        else
        {
            ++failCount;
        }
    }

    std::println("成功: {} 回 / 失敗: {} 回", successCount, failCount);
    std::println("使用量: {} / {}", bump.Used(), bump.Capacity());
}
```

```
成功: 16 回 / 失敗: 84 回
使用量: 64 / 64
```

**ヒープ破壊のダイアログは出ません。** 板に収まるぶんだけ成功し、あとはすべて拒否されました。

`int` は4バイト、アラインメントも4なので、64バイトにちょうど16個入ります。計算が合っています。

---

## 7.9 コストを測る

エラー処理を足したぶん、遅くなったはずです。第5章のベンチマークを走らせて確かめます。

```
Bump v0.2 (検査なし)   median=      1.9  p95=      2.0  max=        2.3
Bump v0.3 (検査あり)   median=      2.0  p95=      2.1  max=        2.4
new (確保のみ)         median=     31.4  p95=     34.8  max=       39.2
```

**ほとんど変わりません。** 0.1 ns 程度です。

### なぜこんなに安いのか

足したのは、比較2回と分岐2回だけです。しかも分岐の結果は毎回同じ(成功側)なので、CPU の分岐予測がほぼ100%当たります。予測が当たった分岐のコストは、実質ゼロに近くなります。

### `expected` の戻り値は重くないのか

`std::expected<void*, AllocError>` は 16 バイト程度の構造体です。MSVC の x64 呼び出し規約では、8バイトを超える構造体は **メモリ経由で返される** のが原則です。レジスタ1本で返せる `void*` に比べれば、明らかに不利に見えます。

しかし、測定結果には現れませんでした。`Allocate` がクラス定義の中に書かれていて **インライン展開される** ためです。インライン展開されれば呼び出し規約は関係なくなり、コンパイラは中間の構造体を丸ごと消せます。

> **ただし、これが崩れる場面があります。**
> 第45章で、アロケーターを仮想関数のインターフェイス越しに使えるようにします。仮想関数はインライン展開できないので、そこでは呼び出し規約のコストが実際に効いてきます。そのときに、あらためて測り直します。

「抽象化のコストはゼロ」という主張は、条件付きで正しい——ということです。条件を意識しておいてください。

---

## 7.10 この章の完成コード

```cpp
#include <bit>
#include <cassert>
#include <cstddef>
#include <cstdint>
#include <expected>
#include <limits>
#include <print>
#include <vector>

inline constexpr std::size_t kDefaultAlignment = __STDCPP_DEFAULT_NEW_ALIGNMENT__;

constexpr std::uintptr_t AlignUp(std::uintptr_t value, std::size_t alignment) noexcept
{
    const std::uintptr_t mask = static_cast<std::uintptr_t>(alignment) - 1;
    return (value + mask) & ~mask;
}

inline bool IsAlignedTo(const void* p, std::size_t alignment) noexcept
{
    return (reinterpret_cast<std::uintptr_t>(p) % alignment) == 0;
}

// ---------------------------------------------------------
// エラー
// ---------------------------------------------------------
enum class AllocError
{
    OutOfMemory,
    InvalidAlignment,
    SizeTooLarge,
};

constexpr const char* ToString(AllocError e) noexcept
{
    switch (e)
    {
    case AllocError::OutOfMemory:      return "OutOfMemory";
    case AllocError::InvalidAlignment: return "InvalidAlignment";
    case AllocError::SizeTooLarge:     return "SizeTooLarge";
    }
    return "Unknown";
}

using AllocResult = std::expected<void*, AllocError>;

// ---------------------------------------------------------
// Bump アロケーター v0.3 — エラー処理つき
// ---------------------------------------------------------
class Bump
{
public:
    explicit Bump(std::size_t capacity)
        : buffer_(capacity)
    {
    }

    [[nodiscard]]
    AllocResult Allocate(std::size_t size,
                         std::size_t alignment = kDefaultAlignment) noexcept
    {
        if (!std::has_single_bit(alignment))
        {
            return std::unexpected(AllocError::InvalidAlignment);
        }

        const std::size_t capacity = buffer_.size();

        const auto base    = reinterpret_cast<std::uintptr_t>(buffer_.data());
        const auto current = base + offset_;
        const auto aligned = AlignUp(current, alignment);

        const std::size_t padding = static_cast<std::size_t>(aligned - current);

        if (padding > capacity - offset_)
        {
            return std::unexpected(AllocError::OutOfMemory);
        }

        const std::size_t alignedOffset = offset_ + padding;

        if (size > capacity - alignedOffset)
        {
            return std::unexpected(AllocError::OutOfMemory);
        }

        offset_   = alignedOffset + size;
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

**演習7-1** `AllocateOrDie(size, alignment)` を書いてください。失敗したらメッセージを出して `std::abort()` します。この関数があると、テストコードはどれくらい書きやすくなりますか。

**演習7-2** `SizeTooLarge` を定義しましたが、実装ではまだ使っていません。どんな条件でこのエラーを返すべきでしょうか。`OutOfMemory` と区別する価値はありますか。

**演習7-3** `Allocate(0, 16)` を呼ぶと何が返りますか。それは妥当な設計でしょうか。`malloc(0)` の挙動を調べて比べてください。

**演習7-4** 7.5 節の「間違い2」の実装を書いて、`Allocate(SIZE_MAX, 1)` を呼んでみてください。何が起きますか。正しい実装との差を確認してください。

**演習7-5** `[[nodiscard]]` を外して、戻り値を無視するコードを書いてください。警告は出ますか。この属性の価値をどう評価しますか。

**演習7-6** モナディック操作(`transform` / `and_then`)を使って、「`int` を確保して 42 を書き込み、成功なら値を、失敗ならエラー名を表示する」処理を、`if` を使わずに書いてください。読みやすくなりましたか。

---

## 章末チェックリスト

- [ ] Debug 構成で、溢れがヒープ破壊として検出されるのを見た
- [ ] エラー伝達の4つの選択肢と、それぞれの長短を説明できる
- [ ] `std::expected<void*, AllocError>` に書き換えた 〔v0.3〕
- [ ] 符号なし整数の引き算を安全に書く方法を理解した
- [ ] 失敗しても状態が変わらないこと(強い保証)を確認した
- [ ] 溢れ・不正アラインメント・巨大サイズの3つのテストが通った
- [ ] エラー処理のコストを測り、ほぼゼロであることを確認した

---

## 次章の予告

`Bump` は、正しく、速く、安全になりました。しかし致命的な制約が残っています。

**一度使い切ったら、それで終わり。**

これを解決するのが、次章です。しかも、たった1行で。

```cpp
void Reset() noexcept { offset_ = 0; }
```

この1行が、なぜゲーム開発でこれほど強力なのか。第3章で立てた「寿命が揃っているなら、まとめて捨てればいい」という仮説を、いよいよ実装で確かめます。

---

> **コラム:メモリ不足に、人はどう向き合ってきたか**
>
> 「メモリが足りなかったら、どうするか」という問いへの答えは、時代と環境で大きく変わってきました。
>
> **C の時代**、答えは明快でした。`malloc` が `NULL` を返したらチェックする。教科書にはそう書いてあり、実際そう書くべきでした。メモリが数百キロバイトしかない環境では、確保の失敗は日常的な出来事だったからです。
>
> **仮想メモリが普及すると**、話がややこしくなりました。Linux には **オーバーコミット** という仕組みがあります。`malloc` は成功を返すが、実際の物理メモリは書き込むまで割り当てられません。その結果、`malloc` の戻り値をいくらチェックしても、実際にメモリが枯渇するのは **書き込んだ瞬間** になります。そして OOM Killer が、まったく無関係なプロセスを選んで殺します。
>
> こうなると「`malloc` の戻り値をチェックする」という作法の意味は薄れます。実際、デスクトップアプリケーションの多くは、メモリ不足からの回復を真面目に実装していません。回復コードを書いても、テストできず、動作を確認できないためです。**動かないエラー処理は、書かないほうがまだましだ** という考え方さえあります。
>
> ---
>
> **ゲーム機の世界は、まったく違います。**
>
> 搭載メモリは固定です。オーバーコミットはなく、他のプロセスとの奪い合いもありません。使える量は最初から分かっています。
>
> だからこそ、メモリ不足は「実行時に起きうる例外的事象」ではなく、**「設計を間違えた証拠」** として扱われます。回復するのではなく、開発中に検出して直す。第49章で作るメモリ予算の仕組みは、この考え方の実装です。
>
> ---
>
> では、この章で `expected` を導入したのは無駄だったのでしょうか。
>
> そうではありません。7.3 節で書いたとおり、`expected` は最も情報量の多い形です。「失敗したら即死」にしたければ、上から被せるだけで済みます。逆に、即死する実装から「失敗を返す実装」を作ることはできません。
>
> **どちらの方針も選べる状態にしておく。** それが、ライブラリとしてのアロケーターに求められる態度です。
