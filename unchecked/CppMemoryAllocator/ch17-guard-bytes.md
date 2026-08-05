# 第17章 はみ出しを検出する 〔v0.12〕

---

## この章のゴール

前章の塗りつぶしは、**間違ったメモリを読んでいる** ことを教えてくれます。しかし、**書き込みのはみ出し** は検出できません。

```cpp
auto r = arena.NewArray<int>(10);
(*r)[10] = 1;      // ← 1つ外に書いた
```

書き込んだ先は、次の確保が使う領域です。`0xCD` のパターンが上書きされるだけで、誰も気づきません。

この章では **ガードバイト**(カナリア)を導入します。

```
┌─────────┬──────────┬────────────────┬──────────┐
│ヘッダ   │ 0xFD×16  │  ユーザー領域   │ 0xFD×16  │
└─────────┴──────────┴────────────────┴──────────┘
            前ガード                     後ガード
```

- 確保領域の前後に見張り番のバイト列を置く 〔**v0.12**〕
- `Reset()` / `Rewind()` のときに、壊れていないか検査する
- **どこで確保された領域が壊れたか** を報告する(第14章の `source_location` が効きます)
- 検出できないケースを正直に整理し、より強力な手段へ橋を架ける

---

## 17.1 塗りつぶしでは見つからないもの

まず、いま何が起きているか確認します。

```cpp
int main()
{
    ga::Bump arena(1024);

    auto a = arena.NewArray<int>(4);      // 16 バイト
    auto b = arena.NewArray<int>(4);      // 16 バイト

    std::span<int> first  = *a;
    std::span<int> second = *b;

    // わざと1つ外に書く
    int* raw = first.data();
    raw[4] = 0x11111111;                  // ← first の範囲外

    std::println("second[0] = {:#x}", static_cast<unsigned>(second[0]));
}
```

Debug 構成での出力です。

```
second[0] = 0x11111111
```

**`first` への書き込みが、`second` の中身を書き換えました。**

エラーは出ません。第16章の `0xCD` パターンも、上書きされてしまえば手がかりになりません。

### なぜこれが厄介か

`second` を使うコードは、まったく正しく書かれています。バグは `first` を使うコードにあります。しかし症状が現れるのは `second` 側です。

**原因と症状が、別のコードに分かれます。** `second` をいくら見直しても、原因は見つかりません。

---

## 17.2 カナリアという発想

炭鉱では、かつてカナリアを籠に入れて坑内に持ち込みました。有毒ガスに人間より敏感なので、カナリアが弱れば避難する——という早期警戒の仕組みです。

同じ発想をメモリに持ち込みます。**壊れたら分かるものを、わざと危険な位置に置く。**

```
┌──────────┬────────────────┬──────────┐
│ 0xFD×16  │  ユーザー領域   │ 0xFD×16  │
└──────────┴────────────────┴──────────┘
   カナリア                    カナリア
```

ユーザーが範囲を1バイトでも超えて書けば、`0xFD` が別の値に変わります。後で検査すれば、はみ出しがあったと分かります。

### `0xFD` を選ぶ理由

第16章の表を再掲します。

| 値 | 意味 |
|---|---|
| `0xCD` | 確保したが未初期化 |
| `0xDD` | 破棄済み |
| **`0xFD`** | **No Man's Land:確保領域の前後の番兵** |

MSVC の CRT デバッグヒープが、まさに同じ目的で使っている値です。慣習に合わせます。

### スタックにも同じ仕組みがある

C++ の世界では、この技法はすでに広く使われています。

Visual Studio の `/GS` オプション(既定で有効)は、**スタックのローカル変数と戻りアドレスの間にカナリアを置きます**。関数から戻る直前に検査し、壊れていればプログラムを即座に終了させます。

バッファオーバーフローによる攻撃——ローカル配列を溢れさせて戻りアドレスを書き換える——を防ぐための仕組みです。

**私たちがこれからやるのは、その手法をヒープ側に持ち込むこと** です。目的が「攻撃の防止」から「バグの発見」に変わるだけで、原理は同じです。

---

## 17.3 レイアウトを設計する

### 素朴な案とその問題

```
[前ガード][ユーザー領域][後ガード]
```

一見これでよさそうですが、**アラインメントが崩れます**。

ガードを 16 バイトにして、ユーザーが `alignas(32)` の型を要求したとします。板の先頭が 32 の倍数でも、そこから 16 進めた位置は 32 の倍数ではありません。

### 解決:前ガードのサイズをアラインメントの倍数にする

```cpp
frontSize = AlignUpSize(必要なサイズ, alignment);
```

前ガードの領域をアラインメントの倍数に切り上げれば、`raw + frontSize` は `raw` と同じアラインメントを保ちます。

### 検査に必要な情報をどこに置くか

後で検査するには、「どこにガードがあるか」を覚えておく必要があります。第11章のファイナライザと同じ問題です。

**同じ解決策を採ります。** 情報をブロックの先頭に埋め込み、連結リストで繋ぎます。

```
raw
 │
 ▼
┌─────────────┬────────┬──────────┬────────────────┬──────────┐
│ GuardBlock  │ 隙間    │ 0xFD×16  │  ユーザー領域   │ 0xFD×16  │
│ (ヘッダ)     │        │          │                │          │
└─────────────┴────────┴──────────┴────────────────┴──────────┘
                                   ▲
                                   user = raw + frontSize
```

「隙間」は、`frontSize` をアラインメントの倍数に切り上げたときに生じる余りです。

### ヘッダの中身

```cpp
namespace ga::detail
{
    struct GuardBlock
    {
        std::byte*           userBegin      = nullptr;
        std::size_t          userSize       = 0;
        std::size_t          frontGuardSize = 0;
        std::source_location location{};      // ← 第14章の成果
        GuardBlock*          next           = nullptr;
    };
}
```

**`source_location` を持たせるのが要点です。**

これがあると、エラーメッセージが劇的に実用的になります。

```
[ガード違反] main.cpp:42 で確保した 16 バイトの領域の 後ろ が壊れています(オフセット +0)
```

「どこかで壊れました」と「main.cpp の 42 行目で確保した領域が壊れました」では、調査の手間がまるで違います。第14章で作った仕組みが、ここで効いてきます。

---

## 17.4 実装する 〔v0.12〕

### 準備:サイズ用の `AlignUpSize`

第6章の `AlignUp` は `std::uintptr_t` 用でした。サイズにも使えるものを足します。

```cpp
// ga/Core.h
constexpr std::size_t AlignUpSize(std::size_t value, std::size_t alignment) noexcept
{
    assert(std::has_single_bit(alignment));

    const std::size_t mask = alignment - 1;
    return (value + mask) & ~mask;
}
```

### 確保処理を2層に分ける

これまでの `Allocate()` を `AllocateRaw()` に改名し、その上にガード付きの層を被せます。

```cpp
    [[nodiscard]]
    AllocResult Allocate(std::size_t size,
                         std::size_t alignment = kDefaultAlignment,
                         const std::source_location& loc = std::source_location::current()) noexcept
    {
#if GA_ENABLE_GUARD_BYTES
        auto r = AllocateGuarded(size, alignment, loc);
#else
        auto r = AllocateRaw(size, alignment, loc);
#endif
        if (r) { RecordAllocation(*r, size, alignment, lastPadding_, loc); }
        return r;
    }
```

### ⚠ デバッグ機能が統計を汚さないようにする

ここで見落としやすい点があります。

ガード付きで確保すると、実際に消費するバイト数は `frontSize + size + kGuardSize` です。もし `RecordAllocation` にこの値を渡してしまうと、**第15章で作った統計が、ガードのぶんだけ水増しされます**。

「Mesh が 4.19 MB」というレポートが、Debug 構成でだけ「Mesh が 12.5 MB」になってしまう。数字を信じられなくなります。

**記録するのは、ユーザーが要求したサイズです。** だから `RecordAllocation` の呼び出しを、ガード処理より外側に出しました。

> **デバッグ機能は、他のデバッグ機能の観測結果を歪めてはいけません。** 当たり前のようですが、機能が増えると意外に破りやすい原則です。

### ガード付きの確保

```cpp
private:
    [[nodiscard]]
    AllocResult AllocateGuarded(std::size_t size, std::size_t alignment,
                                const std::source_location& loc) noexcept
    {
        using detail::GuardBlock;

        const std::size_t blockAlign =
            (alignment > alignof(GuardBlock)) ? alignment : alignof(GuardBlock);

        const std::size_t frontSize =
            AlignUpSize(sizeof(GuardBlock) + kGuardSize, blockAlign);

        // 合計サイズの桁溢れを検査
        const std::size_t overhead = frontSize + kGuardSize;
        if (size > (std::numeric_limits<std::size_t>::max)() - overhead)
        {
            return std::unexpected(AllocError::SizeTooLarge);
        }

        auto raw = AllocateRaw(size + overhead, blockAlign, loc);
        if (!raw) { return std::unexpected(raw.error()); }

        std::byte* base = static_cast<std::byte*>(*raw);
        std::byte* user = base + frontSize;

        // ヘッダを構築してリストに繋ぐ
        auto* header = std::construct_at(
            reinterpret_cast<GuardBlock*>(base),
            GuardBlock{ user, size, kGuardSize, loc, guardBlocks_ });
        guardBlocks_ = header;

        // ガードとユーザー領域を塗る
        FillPattern(user - kGuardSize, kGuardSize, kPatternGuard);
        FillPattern(user + size,       kGuardSize, kPatternGuard);
        FillPattern(user, size, kPatternAllocated);

        return static_cast<void*>(user);
    }
```

### 検査する

```cpp
public:
    struct GuardViolation
    {
        const void*          userBegin = nullptr;
        std::size_t          userSize  = 0;
        bool                 isFront   = false;   // 前ガードか後ガードか
        std::size_t          offset    = 0;       // ガード内で最初に壊れた位置
        std::source_location location{};
    };

    using GuardCallback = void (*)(const GuardViolation&, void* user) noexcept;

    void SetGuardCallback(GuardCallback cb, void* user = nullptr) noexcept
    {
        guardCallback_ = cb;
        guardUser_     = user;
    }

    // いま生きているすべての領域を検査する。壊れていた件数を返す。
    std::size_t VerifyGuards() const noexcept
    {
        return VerifyGuardsUntil(nullptr);
    }

private:
    static constexpr std::size_t kNoViolation = static_cast<std::size_t>(-1);

    // ガード領域を調べ、最初に壊れた位置を返す
    static std::size_t FindCorruption(const std::byte* p, std::size_t n) noexcept
    {
        for (std::size_t i = 0; i < n; ++i)
        {
            if (p[i] != kPatternGuard) { return i; }
        }
        return kNoViolation;
    }

    std::size_t VerifyGuardsUntil(detail::GuardBlock* stop) const noexcept
    {
        std::size_t violations = 0;

        for (auto* b = guardBlocks_; b != stop; b = b->next)
        {
            if (const auto i = FindCorruption(b->userBegin - b->frontGuardSize,
                                              b->frontGuardSize);
                i != kNoViolation)
            {
                ReportViolation({ b->userBegin, b->userSize, true, i, b->location });
                ++violations;
            }

            if (const auto i = FindCorruption(b->userBegin + b->userSize, kGuardSize);
                i != kNoViolation)
            {
                ReportViolation({ b->userBegin, b->userSize, false, i, b->location });
                ++violations;
            }
        }

        return violations;
    }

    void ReportViolation(const GuardViolation& v) const noexcept
    {
        if (guardCallback_)
        {
            guardCallback_(v, guardUser_);
        }
        else
        {
            assert(false && "ガードバイトが破壊されました");
        }
    }
```

### `Reset()` / `Rewind()` に組み込む

**順序が重要です。** 第16章と同じ話です。

```cpp
    void Reset() noexcept
    {
#if GA_ENABLE_GUARD_BYTES
        VerifyGuardsUntil(nullptr);      // ① 検査(まだ壊す前に)
        guardBlocks_ = nullptr;
#endif
        RunFinalizersUntil(nullptr);     // ② デストラクタ

#if GA_ENABLE_MEMORY_PATTERN
        FillPattern(buffer_.data(), offset_, kPatternFreed);   // ③ 塗りつぶし
#endif
        // ④ オフセットと統計のクリア
    }
```

検査を最初にやらないと、塗りつぶしがガードを `0xDD` に書き換えてしまい、**すべてのブロックが「壊れている」と報告されます**。

`Marker` にもガードリストの先頭を持たせます。第11章でファイナライザに対して行ったのと同じです。

```cpp
    struct Marker
    {
        std::size_t          offset     = 0;
        std::size_t          padding    = 0;
        std::uint32_t        depth      = 0;
        std::uint32_t        epoch      = kInvalidEpoch;
        detail::Finalizer*   finalizers = nullptr;
        detail::GuardBlock*  guards     = nullptr;   // v0.12
    };
```

### 切り替えマクロ

```cpp
#ifndef GA_ENABLE_GUARD_BYTES
#  ifdef NDEBUG
#    define GA_ENABLE_GUARD_BYTES 0
#  else
#    define GA_ENABLE_GUARD_BYTES 1
#  endif
#endif
```

> **メンバ変数(`guardBlocks_`、`guardCallback_`、`guardUser_`)は `#if` で囲みません。** 第9章・第14章と同じ理由です。構成によってクラスのサイズが変わると、ODR 違反の温床になります。

---

## 17.5 検出させてみる

```cpp
void OnGuardViolation(const ga::Bump::GuardViolation& v, void*) noexcept
{
    std::string_view file = v.location.file_name();
    if (auto pos = file.find_last_of("\\/"); pos != std::string_view::npos)
    {
        file = file.substr(pos + 1);
    }

    std::println("[ガード違反] {}:{} で確保した {} バイトの領域の {} が壊れています(+{})",
                 file, v.location.line(), v.userSize,
                 v.isFront ? "手前" : "後ろ", v.offset);
}

int main()
{
    ga::Bump arena(4096);
    arena.SetGuardCallback(&OnGuardViolation);

    auto r = arena.NewArray<int>(4);          // ← 42 行目
    std::span<int> values = *r;

    values.data()[4] = 0x11111111;            // 1つ外に書く

    arena.Reset();                            // ここで検査される
}
```

```
[ガード違反] main.cpp:42 で確保した 16 バイトの領域の 後ろ が壊れています(+0)
```

**確保した場所が特定できました。**

### 前へのはみ出しも検出できる

```cpp
    values.data()[-1] = 0x22222222;   // 手前に書く
```

```
[ガード違反] main.cpp:42 で確保した 16 バイトの領域の 手前 が壊れています(+12)
```

`+12` は、16 バイトのガードのうち 12 バイト目から壊れている、つまり最後の 4 バイトが書き換えられたことを示します。

### `memcpy` の off-by-one

現実によくあるパターンです。

```cpp
auto r = arena.NewArray<char>(16);
char src[17] = "0123456789ABCDEF";        // 終端を含めて 17 バイト

std::memcpy(r->data(), src, sizeof(src)); // ← 17 バイトコピーしてしまう
```

```
[ガード違反] main.cpp:58 で確保した 16 バイトの領域の 後ろ が壊れています(+0)
```

**1バイトのはみ出しを捕まえました。** `sizeof(src)` が 17 であることに気づかない、という典型的なミスです。

### テスト

```cpp
void Test_GuardDetectsOverrun()
{
#if GA_ENABLE_GUARD_BYTES
    ga::Bump arena(1024);

    int violations = 0;
    arena.SetGuardCallback(
        [](const ga::Bump::GuardViolation&, void* user) noexcept {
            ++*static_cast<int*>(user);
        },
        &violations);

    auto r = arena.Allocate(16, 1);
    std::byte* p = static_cast<std::byte*>(*r);

    assert(arena.VerifyGuards() == 0);   // まだ無傷

    p[16] = std::byte{ 0x99 };           // はみ出す

    assert(arena.VerifyGuards() == 1);   // 検出された
    assert(violations == 1);

    std::println("[ OK ] Test_GuardDetectsOverrun");
#endif
}
```

---

## 17.6 いつ検査するか

`Reset()` のときだけでは、検出が遅すぎることがあります。

| タイミング | 長所 | 短所 |
|---|---|---|
| `Reset()` / `Rewind()` 時 | 自動。コストは1周期に1回 | フレームの最後まで分からない |
| **毎フレーム明示的に呼ぶ** | 1フレーム以内に絞れる | 生きているブロック数に比例したコスト |
| **怪しい処理の前後で呼ぶ** | 原因を絞り込める | 手で書く必要がある |

3番目が実用的です。

```cpp
void UpdateFrame(ga::Bump& arena)
{
    UpdatePhysics(arena);
    assert(arena.VerifyGuards() == 0);      // ここまでは無事か?

    UpdateAnimation(arena);
    assert(arena.VerifyGuards() == 0);

    BuildDrawList(arena);
    assert(arena.VerifyGuards() == 0);
}
```

検査を挟む間隔を狭めていけば、**二分探索の要領で原因の関数を特定できます**。

---

## 17.7 検出できないこと

正直に限界を書きます。

### 1. 遠くへの書き込みは素通りする

```cpp
auto r = arena.NewArray<int>(4);
r->data()[1000] = 1;      // ← ガードを飛び越えた
```

ガードは 16 バイトです。それを超えた先に書けば、ガードは無傷のまま、別のブロックのユーザー領域が壊れます。

**添字の計算を間違えた場合(`i * stride + j` の取り違えなど)は、この形になりがちです。**

### 2. 検出が遅れる

書き込んだ瞬間ではなく、次に `VerifyGuards()` が呼ばれるまで分かりません。その間にプログラムは動き続け、別の場所でクラッシュするかもしれません。

### 3. 読み取りのはみ出しは検出できない

```cpp
int x = r->data()[4];     // 範囲外を「読む」
```

読むだけならガードは壊れません。**ただし、読んだ値は `0xFD` パターンになります。** `int` なら `-33686019`。第16章の考え方で気づける可能性はあります。

### 4. メモリを大量に消費する

後述します。

### より強力な手段へ

これらの限界を超えるには、**ハードウェアの助け** が必要です。

| 章 | 手段 | 検出できること |
|---|---|---|
| 第31章 | ガードページ(`VirtualAlloc`) | はみ出した **瞬間** にアクセス違反。距離に関係なく検出 |
| 第36章 | AddressSanitizer | 読み書き両方、範囲外、解放後使用、リーク |

ガードバイトは、**手軽さと引き換えに精度を落とした手段** です。それでも、ライブラリ側だけで完結し、追加のツールも設定も要らないという利点があります。

---

## 17.8 コストを測る

### メモリ

これが最大の代償です。

```cpp
sizeof(GuardBlock)                    = 64 バイト(source_location が 32 バイト)
frontSize = AlignUpSize(64 + 16, 16)  = 80 バイト
後ガード                               = 16 バイト
────────────────────────────────────────────────
1確保あたりのオーバーヘッド             = 96 バイト
```

16 バイトの確保に 96 バイトの追加コスト。**実質7倍** です。

```
1万回の 16 バイト確保:
  ガードなし :   160 KB
  ガードあり : 1,120 KB
```

**Debug 構成では、板を大きめに取る必要があります。** これは実務上の注意点です。「Debug でだけメモリ不足になる」という現象が起きたら、まずこれを疑ってください。

### 速度

```
v0.11 (塗りつぶしのみ)   median=      6.9  p95=      7.2
v0.12 (ガードあり)       median=     26.4  p95=     28.1
```

**約4倍。** 増えた処理は、

- ガードの塗りつぶし(32 バイト × 2回の `memset`)
- ヘッダの構築とリストへの繋ぎ込み
- 確保サイズの増加によるキャッシュ効率の悪化

`Reset()` も、生きているブロック数に比例した検査時間がかかるようになります。

### まとめ:Debug 専用機能である

| 機能 | Debug | Release |
|---|---|---|
| 統計・タグ(第15章) | ○ | 任意 |
| 塗りつぶし(第16章) | ○ | × |
| **ガードバイト(第17章)** | **○** | **×** |

第51章で、これらをまとめて制御する仕組みを整理します。

---

## 17.9 この章の完成コード

```cpp
// ga/Pattern.h に追加
namespace ga
{
    inline constexpr std::byte  kPatternGuard{ 0xFD };
    inline constexpr std::size_t kGuardSize = 16;
}

// ga/detail/GuardBlock.h(新規)
#pragma once

#include <cstddef>
#include <source_location>

namespace ga::detail
{
    struct GuardBlock
    {
        std::byte*           userBegin      = nullptr;
        std::size_t          userSize       = 0;
        std::size_t          frontGuardSize = 0;
        std::source_location location{};
        GuardBlock*          next           = nullptr;
    };
}
```

`ga/Bump.h` への追加は 17.4 節のとおりです。新しいメンバは3つ。

```cpp
    detail::GuardBlock* guardBlocks_   = nullptr;
    GuardCallback       guardCallback_ = nullptr;
    void*               guardUser_     = nullptr;
```

---

## 演習

**演習17-1** `kGuardSize` を 4 バイトに減らすと、検出できなくなるバグの例を作ってください。逆に 256 バイトに増やすと、何が改善しますか。

**演習17-2** `GuardBlock` から `source_location` を外すと、ヘッダは何バイトになりますか。エラーメッセージはどれだけ使いにくくなりますか。

**演習17-3** `Reset()` の中で、検査を塗りつぶしの **後** に移してください。何が報告されますか。理由を説明してください。

**演習17-4** ガードのパターンを、ブロックごとに違う値(アドレスから計算したハッシュなど)にすると、どんな利点がありますか。

**演習17-5** 17.7 節の「遠くへの書き込み」を実際に作り、検出されないことを確認してください。そのうえで、どのブロックが壊れたかを `VerifyGuards()` で特定できますか。

**演習17-6** `VerifyGuards()` を毎フレーム呼ぶ場合、生きているブロックが10万個あるとどれくらいの時間がかかりますか。実測してください。

**演習17-7** ガード違反を検出したとき、`assert` ではなく `std::abort()` で即座に落とす設計と、ログだけ出して続行する設計を比べてください。どちらが良いでしょうか。

---

## 章末チェックリスト

- [ ] 塗りつぶしだけでは書き込みのはみ出しが検出できないことを確認した
- [ ] ガードバイトの配置とアラインメントの問題を理解した
- [ ] `GuardBlock` に `source_location` を持たせる価値を説明できる
- [ ] **デバッグ機能が統計を汚さない** ようにする理由を説明できる
- [ ] 検査 → 破棄 → 塗りつぶし の順序が必要な理由を説明できる
- [ ] 範囲外書き込みを実際に検出させた
- [ ] 検出できない4つのケースを挙げられる
- [ ] メモリのオーバーヘッド(1確保あたり約96バイト)を確認した

---

## 次章の予告

ガード違反が報告されるようになりました。しかし報告に含まれるのは、**確保した場所** です。

```
main.cpp:42 で確保した 16 バイトの領域が壊れています
```

知りたいのは、**壊した場所** かもしれません。そして「42行目で確保された」だけでは、その関数がどこから呼ばれたのか分かりません。同じ関数が10か所から呼ばれていたら、どの経路か特定できません。

第18章では **`std::stacktrace`**(C++23)を使います。確保時のコールスタック全体を記録し、リーク時やガード違反時に表示します。

```
[ガード違反] 16 バイトの領域が壊れています
  確保元:
    ga::Bump::NewArray<int>          Bump.h:243
    LoadMesh                          MeshLoader.cpp:88
    LoadStage                         StageLoader.cpp:34
    main                              main.cpp:12
```

ただし、これは高価な機能です。1回の記録にマイクロ秒単位の時間がかかり、メモリも消費します。**どこまで記録するか** というトレードオフを、実測しながら決めていきます。

---

> **コラム:CRT デバッグヒープ — 車輪はすでに発明されていた**
>
> この章で作ったものは、実は Visual Studio に最初から入っています。**CRT デバッグヒープ** です。
>
> Debug 構成でビルドすると、`malloc` や `new` は自動的にデバッグ版に置き換わります。確保されるブロックの構造は、私たちが作ったものとほぼ同じです。
>
> ```
> ┌──────────────────┬──────────┬────────────┬──────────┐
> │ _CrtMemBlockHeader│ 0xFD × 4 │ ユーザー領域 │ 0xFD × 4 │
> └──────────────────┴──────────┴────────────┴──────────┘
> ```
>
> ヘッダには、ファイル名、行番号、確保サイズ、確保の通し番号、そして前後のブロックへのポインタが入っています。**私たちの `GuardBlock` と、発想も構造もそっくりです。**
>
> ---
>
> 使える機能も豊富です。
>
> **`_CrtSetDbgFlag(_CRTDBG_LEAK_CHECK_DF)`** — 第10章で使いました。終了時に未解放ブロックを一覧表示します。
>
> **`_CrtCheckMemory()`** — すべてのブロックのガードを検査します。私たちの `VerifyGuards()` に相当します。
>
> **`_CrtSetBreakAlloc(146)`** — 146 番目の確保でブレークします。リークレポートに出た番号を指定すれば、**その確保が起きた瞬間にデバッガで止まれます**。これは非常に強力です。
>
> **`_CrtMemCheckpoint` / `_CrtMemDifference`** — 2時点のスナップショットを取り、差分を表示します。「この処理の前後でメモリが増えていないか」を調べられます。
>
> ---
>
> **では、なぜ自分で作ったのか。**
>
> 第一に、**CRT デバッグヒープは `malloc` / `new` にしか効きません**。私たちの `Bump` が配るメモリは、CRT から見れば「1回の大きな確保」でしかありません。その内側で何が起きているかは、CRT には見えないのです。
>
> **自作アロケーターを使うということは、既存のデバッグ支援を1つ失うということ** でもあります。だから自分で作り直す必要がありました。これは自作の隠れたコストであり、第53章で「いつ自作すべきでないか」を考えるときの材料の1つになります。
>
> 第二に、仕組みを理解しておく価値があります。CRT デバッグヒープが何をしているか知っていれば、`0xFD` が見えたときに何が起きたか即座に分かります。
>
> ---
>
> ちなみに、**CRT デバッグヒープと自作アロケーターは共存できます**。板そのものは `std::vector` 経由で `new` から取っているので、板の外側へのはみ出しは CRT が検出してくれます(第7章で見たとおりです)。
>
> 板の内側は自分で見張り、板の外側は CRT が見張る。**二重の網** になっています。
