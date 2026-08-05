# 第29章 `std::vector<std::byte>` をやめる 〔v0.21〕

---

## この章のゴール

第4部が始まります。ここまで、すべてのアロケーターの土台は `std::vector<std::byte>` でした。

```cpp
class Bump
{
    std::vector<std::byte> buffer_;   // ← これを捨てる
};
```

第2章で「ブートストラップだから」と正当化し、「第29章で置き換える」と予告しました。**その章です。**

- `std::vector` を土台にしていることの限界を、実測で確認する
- **予約(reserve)とコミット(commit)の分離** という考え方
- `<Windows.h>` との付き合い方(`WIN32_LEAN_AND_MEAN`、`NOMINMAX`)
- `VirtualMemory` クラスを書き、`Bump` の土台を差し替える 〔**v0.21**〕
- 1 GB 予約しても物理メモリを消費しないことを、数字で確認する

そして、これが手に入ることで **第30章と第31章が可能になります**。

---

## 29.1 `std::vector<std::byte>` の限界

### 限界1:確保した瞬間に、全部物理メモリを消費する

```cpp
ga::Bump arena(1024 * 1024 * 1024);   // 1 GB
```

このコード1行で、何が起きるか測ってみます。

```cpp
#include <psapi.h>

std::size_t GetPrivateBytes()
{
    PROCESS_MEMORY_COUNTERS_EX pmc{};
    pmc.cb = sizeof(pmc);
    GetProcessMemoryInfo(GetCurrentProcess(),
                         reinterpret_cast<PROCESS_MEMORY_COUNTERS*>(&pmc), sizeof(pmc));
    return pmc.PrivateUsage;
}

int main()
{
    std::println("開始時       : {}", ga::FormatBytes(GetPrivateBytes()));

    const auto t0 = std::chrono::steady_clock::now();
    std::vector<std::byte> buffer(1024ull * 1024 * 1024);
    const auto t1 = std::chrono::steady_clock::now();

    std::println("1 GB 確保後  : {}", ga::FormatBytes(GetPrivateBytes()));
    std::println("かかった時間 : {:.1f} ms",
                 std::chrono::duration<double, std::milli>(t1 - t0).count());

    bench::Escape(buffer.data());
}
```

```
開始時       : 3.42 MB
1 GB 確保後  : 1.00 GB
かかった時間 : 214.3 ms
```

**1 バイトも使っていないのに、1 GB を消費しました。** しかも 214 ミリ秒——**13 フレーム分** かかっています。

原因は `std::vector` の値初期化です。全要素をゼロで埋めるため、**1 GB 全部に触ります**。触られたページには、物理メモリが割り当てられます。

第12章で `NewArrayUninit` を作ったときと同じ問題が、土台そのもので起きています。

### 限界2:あとから大きくできない

板が足りなくなったとき、`std::vector` を `resize` すると **中身が移動します**。すでに配ったポインタが全部無効になります。

**アロケーターにとって、これは致命的です。** 「板は最初に決めたサイズで固定」という制約を、受け入れるしかありませんでした。

### 限界3:アラインメントが 16 までしか保証されない

第6章で触れたとおりです。`std::vector<std::byte>` の先頭アドレスは、`operator new` の既定アラインメント(x64 で 16)までしか保証されません。

第26章のバディで「板の先頭が容量と同じ大きさに整列していれば、すべてのブロックが自分のサイズに整列する」と書きましたが、**それを保証する手段がありませんでした**。

### 限界4:ページ単位の制御ができない

第17章のガードバイトには限界がありました。

> 遠くへの書き込みは素通りする。検出が遅れる。

**根本的な解決は、はみ出した瞬間にハードウェアに止めてもらうこと** です。しかしそれには、特定のページを「アクセス禁止」に設定する必要があります。`std::vector` にはその機能がありません。

---

## 29.2 予約とコミットの分離

OS は、もっと柔軟な仕組みを提供しています。

### 仮想アドレスと物理メモリ

プログラムが扱うアドレスは、**仮想アドレス** です。物理メモリのアドレスではありません。

```
   仮想アドレス空間              物理メモリ
   ┌──────────────┐            ┌──────────┐
   │  0x1000000   │ ─────────→ │  ページA  │
   │  0x1001000   │ ─────────→ │  ページB  │
   │  0x1002000   │ ────┐      │  ページC  │
   │     ...      │     ×      └──────────┘
   │  0x1FFF000   │  (未割り当て)
   └──────────────┘
```

**対応付けは、ページ単位(4 KB)で行われます。** 対応が存在しないアドレスに触ると、CPU が **ページフォルト** を起こし、OS に処理が移ります。

### Windows の2段階

Windows では、メモリの確保が2段階に分かれています。

| 段階 | 何をするか | 消費するもの |
|---|---|---|
| **予約**(MEM_RESERVE) | アドレス空間を押さえる | **仮想アドレス空間だけ** |
| **コミット**(MEM_COMMIT) | 使う権利を確定する | コミットチャージ(物理メモリ + ページファイル) |

**予約は、ほとんどタダです。**

64 ビットのプロセスでは、ユーザーモードのアドレス空間が 128 TB あります。**1 GB や 10 GB を予約しても、痛くも痒くもありません。**

```cpp
// 1 GB のアドレス空間を押さえる。物理メモリは 0 バイト
void* base = VirtualAlloc(nullptr, 1GB, MEM_RESERVE, PAGE_NOACCESS);

// そのうち 64 KB だけ使えるようにする
VirtualAlloc(base, 64KB, MEM_COMMIT, PAGE_READWRITE);
```

### コミットしても、まだ物理メモリは使われない

正確に言うと、もう一段あります。

**コミットした時点では、物理メモリはまだ割り当てられません。** 「必要になったら渡す」という約束(コミットチャージ)が記録されるだけです。

実際に物理メモリが割り当てられるのは、**そのページに初めて触ったとき** です。このとき起きるページフォルトを **デマンドゼロフォルト** と呼びます。OS はゼロで埋めたページを用意して、対応付けます。

```
予約       → アドレス空間だけ
コミット   → 「使える」という約束(コミットチャージを消費)
初回アクセス → 物理メモリが割り当てられる(ページフォルト)
```

**私たちのアロケーターにとって重要なのは、この3段階を自分で制御できることです。**

### 単位に注意

```cpp
SYSTEM_INFO info;
GetSystemInfo(&info);

info.dwPageSize;               // 4096(ページサイズ)
info.dwAllocationGranularity;  // 65536(予約の粒度)
```

- **コミットはページ単位(4 KB)** に切り上げられます
- **予約のアドレスは 64 KB 単位** に切り下げられます

「予約の粒度が 64 KB」というのは、`VirtualAlloc` で返るアドレスが必ず 64 KB の倍数になるということです。

**これは嬉しい副作用をもたらします。** 第6章から悩んできたアラインメントの問題が、土台のレベルで解決します。64 KB 境界に整列した領域なら、その中のどんなアラインメント要求も(64 KB までなら)満たせます。

---

## 29.3 `<Windows.h>` の作法

Windows API を使うには `<Windows.h>` が必要です。**しかし、素直に書いてはいけません。**

```cpp
#include <Windows.h>    // ← これだけだと、いろいろ壊れる
```

### 作法1:`WIN32_LEAN_AND_MEAN`

`<Windows.h>` は、既定で膨大な数のヘッダを取り込みます。暗号化、DDE、RPC、シェル、Winsock——**そのほとんどは不要** です。

```cpp
#define WIN32_LEAN_AND_MEAN
#include <Windows.h>
```

これで、めったに使わないヘッダが除外されます。**コンパイル時間が目に見えて短くなります。**

### 作法2:`NOMINMAX` ← 第12章の伏線

`<Windows.h>` は、次のマクロを定義します。

```cpp
#define min(a,b)  (((a) < (b)) ? (a) : (b))
#define max(a,b)  (((a) > (b)) ? (a) : (b))
```

**これが、標準ライブラリを壊します。**

```cpp
std::numeric_limits<std::size_t>::max()
//                                ^^^ マクロとして展開されようとする
//                                → 引数が足りないというエラー

std::min(a, b)   // → std::(((a)<(b))?(a):(b)) という意味不明なコード
```

第12章で、私はこう書きました。

```cpp
if (count > (std::numeric_limits<std::size_t>::max)() / sizeof(T))
//          ^                                       ^
//          括弧で囲んでいる
```

**この括弧の理由が、いま明らかになりました。** 関数名を括弧で囲むと、関数形式マクロとして展開されなくなります。ライブラリ側の防御策です。

しかし、根本的には次のように定義すべきです。

```cpp
#define NOMINMAX
#include <Windows.h>
```

### 作法3:他のマクロにも注意

`<Windows.h>` が定義するマクロは、`min` / `max` だけではありません。

| マクロ | 問題 |
|---|---|
| `small` | `char` に置換される。`small` という識別子が使えなくなる |
| `near` / `far` | 16 ビット時代の名残 |
| `CreateWindow` など | `CreateWindowW` に置換される(A/W の切り替え) |

**これらは、C++ の名前空間では防げません。** マクロは名前空間を無視するからです。第13章のコラムで「モジュールが解決しようとした問題」として触れた話です。

### 作法4:公開ヘッダに入れない ← 第13章の伏線

最も重要な作法です。

```cpp
// ga/VirtualMemory.h(公開ヘッダ)
#include <Windows.h>    // ← 絶対にダメ
```

このヘッダを使う人全員に、`min` / `max` / `small` のマクロを押し付けることになります。

**実装を `.cpp` に隠します。**

```
ga/VirtualMemory.h   ← インターフェイスだけ。Windows.h を含まない
ga/VirtualMemory.cpp ← ここで Windows.h を使う
```

第13章で、こう書きました。

> **静的ライブラリを作る理由:** 第29章以降で `VirtualAlloc` を扱う章から、Windows API を呼ぶ実装コードが出てきます。それを `.cpp` に置きます。`<Windows.h>` を公開ヘッダに含めたくないので、実装を隠す場所が要ります。

**その予告どおりです。** 16 章前に用意した器が、ようやく中身を持ちます。

---

## 29.4 `VirtualMemory` クラスを書く 〔v0.21〕

### 公開ヘッダ

```cpp
// AllocatorLib/include/ga/VirtualMemory.h
#pragma once

#include <cstddef>

namespace ga
{
    // 仮想アドレス空間を予約し、必要な分だけコミットする
    class VirtualMemory
    {
    public:
        VirtualMemory() noexcept = default;

        // reserveBytes 分のアドレス空間を予約する(物理メモリは消費しない)
        explicit VirtualMemory(std::size_t reserveBytes);

        ~VirtualMemory();

        VirtualMemory(VirtualMemory&& other) noexcept;
        VirtualMemory& operator=(VirtualMemory&& other) noexcept;

        VirtualMemory(const VirtualMemory&)            = delete;
        VirtualMemory& operator=(const VirtualMemory&) = delete;

        // 先頭から bytes バイト目までを使えるようにする
        bool CommitTo(std::size_t bytes) noexcept;

        // bytes より後ろのコミットを解除し、物理メモリを OS に返す
        bool DecommitFrom(std::size_t bytes) noexcept;

        std::byte*  Base()      const noexcept { return base_; }
        std::size_t Reserved()  const noexcept { return reserved_; }
        std::size_t Committed() const noexcept { return committed_; }

        static std::size_t PageSize()              noexcept;
        static std::size_t AllocationGranularity() noexcept;

    private:
        std::byte*  base_      = nullptr;
        std::size_t reserved_  = 0;
        std::size_t committed_ = 0;
    };
}
```

**`<Windows.h>` はどこにもありません。** `std::byte*` と `std::size_t` だけです。

### 実装

```cpp
// AllocatorLib/src/VirtualMemory.cpp
#include "pch.h"

#define WIN32_LEAN_AND_MEAN
#define NOMINMAX
#include <Windows.h>

#include "ga/VirtualMemory.h"
#include "ga/Core.h"

namespace ga
{
    namespace
    {
        // 何度も VirtualAlloc を呼ばないよう、まとめてコミットする単位
        constexpr std::size_t kCommitChunk = 64 * 1024;

        SYSTEM_INFO GetInfo() noexcept
        {
            SYSTEM_INFO info{};
            GetSystemInfo(&info);
            return info;
        }
    }

    std::size_t VirtualMemory::PageSize() noexcept
    {
        static const std::size_t value = GetInfo().dwPageSize;
        return value;
    }

    std::size_t VirtualMemory::AllocationGranularity() noexcept
    {
        static const std::size_t value = GetInfo().dwAllocationGranularity;
        return value;
    }

    VirtualMemory::VirtualMemory(std::size_t reserveBytes)
    {
        if (reserveBytes == 0) { return; }

        const std::size_t rounded = AlignUpSize(reserveBytes, AllocationGranularity());

        void* p = VirtualAlloc(nullptr, rounded, MEM_RESERVE, PAGE_NOACCESS);
        if (p == nullptr) { return; }        // 予約に失敗

        base_     = static_cast<std::byte*>(p);
        reserved_ = rounded;
    }

    VirtualMemory::~VirtualMemory()
    {
        if (base_ != nullptr)
        {
            // MEM_RELEASE のときは、サイズに 0 を渡す規則
            VirtualFree(base_, 0, MEM_RELEASE);
        }
    }

    VirtualMemory::VirtualMemory(VirtualMemory&& other) noexcept
        : base_(other.base_), reserved_(other.reserved_), committed_(other.committed_)
    {
        other.base_      = nullptr;
        other.reserved_  = 0;
        other.committed_ = 0;
    }

    VirtualMemory& VirtualMemory::operator=(VirtualMemory&& other) noexcept
    {
        if (this != &other)
        {
            if (base_ != nullptr) { VirtualFree(base_, 0, MEM_RELEASE); }

            base_      = std::exchange(other.base_, nullptr);
            reserved_  = std::exchange(other.reserved_, 0);
            committed_ = std::exchange(other.committed_, 0);
        }
        return *this;
    }

    bool VirtualMemory::CommitTo(std::size_t bytes) noexcept
    {
        if (bytes <= committed_) { return true; }     // すでにコミット済み
        if (bytes > reserved_)   { return false; }    // 予約を超えている

        std::size_t target = AlignUpSize(bytes, kCommitChunk);
        if (target > reserved_) { target = reserved_; }

        void* p = VirtualAlloc(base_ + committed_, target - committed_,
                               MEM_COMMIT, PAGE_READWRITE);
        if (p == nullptr) { return false; }

        committed_ = target;
        return true;
    }

    bool VirtualMemory::DecommitFrom(std::size_t bytes) noexcept
    {
        const std::size_t target = AlignUpSize(bytes, PageSize());
        if (target >= committed_) { return true; }

        if (!VirtualFree(base_ + target, committed_ - target, MEM_DECOMMIT))
        {
            return false;
        }

        committed_ = target;
        return true;
    }
}
```

### 実装のポイント

**`VirtualFree(base, 0, MEM_RELEASE)`。** 解放のとき、サイズには **0 を渡さなければなりません**。予約したときのサイズを渡すとエラーになります。API の癖です。

**コミットの粒度。** 1 バイト増えるたびに `VirtualAlloc` を呼ぶのは無駄です。システムコールは数マイクロ秒かかります。**64 KB 単位でまとめてコミット** します。

**予約の切り上げ。** 予約サイズを 64 KB(割り当て粒度)に切り上げています。どのみち切り上げられるので、こちら側でも数字を合わせておきます。

**ムーブのみ。** アドレス空間の所有権を持つので、コピーはできません。第11章で `Bump` をコピー禁止にしたのと同じ理由です。

---

## 29.5 `Bump` の土台を差し替える

```cpp
class Bump
{
public:
    explicit Bump(std::size_t capacity)
        : memory_(capacity)
    {
        base_ = memory_.Base();
    }

    [[nodiscard]]
    AllocResult Allocate(std::size_t size,
                         std::size_t alignment = kDefaultAlignment,
                         const std::source_location& loc = std::source_location::current()) noexcept
    {
        // ... 検査は従来どおり ...

        const std::size_t newOffset = alignedOffset + size;

        // --- ここが v0.21 の追加 ---
        if (!memory_.CommitTo(newOffset))
        {
            return std::unexpected(AllocError::OutOfMemory);
        }

        offset_   = newOffset;
        padding_ += padding;

        // ...
    }

    std::size_t Capacity()  const noexcept { return memory_.Reserved(); }
    std::size_t Committed() const noexcept { return memory_.Committed(); }

private:
    VirtualMemory memory_;
    std::byte*    base_ = nullptr;
    // std::vector<std::byte> buffer_;   ← 削除
};
```

**変更はこれだけです。** `Allocate` に1つ条件が増え、`buffer_` が `memory_` になりました。

### `Reset()` で物理メモリを返すか

```cpp
    void Reset() noexcept
    {
        // ... 従来の処理 ...

        // 使わなくなった領域を OS に返す?
        // memory_.DecommitFrom(0);      ← やらない
    }
```

**やりません。**

返してしまうと、次のフレームで再びコミットとページフォルトが発生します。第8章で「板を握りっぱなしにすることで、コストを最初の1回に押し込める」と述べたとおりです。

代わりに、明示的に呼べる関数を用意します。

```cpp
    // ピーク時のコミット量を、現在の使用量まで縮める
    void Trim() noexcept
    {
        memory_.DecommitFrom(offset_);
    }
```

**シーンの切り替え時など、しばらく使わないと分かっているときだけ呼びます。**

---

## 29.6 測る

### 1 GB の予約は、ほぼタダ

```cpp
int main()
{
    std::println("開始時       : {}", ga::FormatBytes(GetPrivateBytes()));

    const auto t0 = std::chrono::steady_clock::now();
    ga::Bump arena(1024ull * 1024 * 1024);       // 1 GB 予約
    const auto t1 = std::chrono::steady_clock::now();

    std::println("1 GB 予約後  : {}", ga::FormatBytes(GetPrivateBytes()));
    std::println("かかった時間 : {:.1f} µs",
                 std::chrono::duration<double, std::micro>(t1 - t0).count());
    std::println("容量 {} / コミット済み {}",
                 ga::FormatBytes(arena.Capacity()),
                 ga::FormatBytes(arena.Committed()));
}
```

```
開始時       : 3.42 MB
1 GB 予約後  : 3.43 MB
かかった時間 : 28.4 µs
容量 1.00 GB / コミット済み 0 B
```

**物理メモリの増加は 10 KB 程度。時間は 28 マイクロ秒。**

`std::vector` 版と比べます。

| | 物理メモリ | 時間 |
|---|---|---|
| `std::vector<std::byte>(1GB)` | **1.00 GB** | **214 ms** |
| `VirtualMemory(1GB)` | 約 0 | **0.028 ms** |

**7,600 倍速く、メモリ消費は実質ゼロ。**

### 使った分だけ増える

```cpp
    ga::Bump arena(1024ull * 1024 * 1024);

    for (int i = 0; i < 5; ++i)
    {
        (void)arena.Allocate(1024 * 1024);    // 1 MB ずつ確保

        std::println("{} MB 確保 → コミット {} / 物理 {}",
                     i + 1,
                     ga::FormatBytes(arena.Committed()),
                     ga::FormatBytes(GetPrivateBytes()));
    }
```

```
1 MB 確保 → コミット 1.00 MB / 物理 4.46 MB
2 MB 確保 → コミット 2.00 MB / 物理 5.47 MB
3 MB 確保 → コミット 3.00 MB / 物理 6.47 MB
4 MB 確保 → コミット 4.00 MB / 物理 7.47 MB
5 MB 確保 → コミット 5.00 MB / 物理 8.47 MB
```

**必要な分だけ、きれいに増えています。**

### コミットのコスト

```
コミットが発生しない Allocate  : 2.1 ns
コミットが発生する Allocate    : 6800.0 ns
```

**64 KB のコミットに、約 6.8 マイクロ秒。**

内訳は、`VirtualAlloc` のシステムコールと、その後のページフォルト(64 KB = 16 ページ分)です。

**ただし、これは 64 KB あたり1回です。** 32 バイトの確保が 2000 回続く間に、1回だけ発生します。ならして考えれば 1 確保あたり 3.4 ns の上乗せです。

そして **2周目からはゼロです。** `Reset()` してもコミットは解除しないので、同じ領域を使い回す限り、コストは最初の1回だけです。

### 平常時の速度への影響

```
v0.20 (std::vector 土台)   median=      2.1  p95=      2.2
v0.21 (VirtualMemory)      median=      2.1  p95=      2.3
```

**変わりません。** 増えたのは `bytes <= committed_` という比較1回だけで、しかも分岐予測がほぼ 100% 当たります。

### 最悪値には現れる

```
v0.20  max=     2800 ns
v0.21  max=     7100 ns
```

**コミットのスパイクが、最大値に出ます。**

第27章で TLSF の最悪値を 300 ns まで削ったのに、土台を変えたせいで 7100 ns のスパイクが復活しました。

**これは正当な代償です。** メモリを OS から取ってくる以上、避けられません。気になる場合の対策は2つあります。

**対策1:起動時に必要量をコミットしておく。**

```cpp
    arena.Prewarm(16 * 1024 * 1024);   // 16 MB を先にコミット
```

ロード画面のうちに済ませておけば、フレーム中には発生しません。

**対策2:コミット粒度を大きくする。** 64 KB を 1 MB にすれば、発生回数が 16 分の 1 になります。ただし1回のコストは増えます。

---

## 29.7 何が可能になったか

土台を差し替えたことで、次の3つが可能になりました。

### 1. 板を大きめに取っても損しない

これまでは「1 MB か 16 MB か」を慎重に決める必要がありました。大きすぎれば無駄、小さすぎれば溢れる。

**いまは、迷ったら大きく取れます。** 使わなければ物理メモリを消費しません。

```cpp
ga::Bump frameArena(256 * 1024 * 1024);   // 256 MB 予約。実際に使うのは数 MB
```

第7章で作ったエラー処理の出番が減ります。**溢れにくくなる** という、地味ですが大きな改善です。

### 2. 要素が移動しない、成長する配列(第30章)

`std::vector` は、容量を超えると再確保して中身を移動します。ポインタが全部無効になります。

**予約済みのアドレス空間の上でなら、移動なしで成長できます。** 追加のコミットをするだけです。

第30章で、これを実装します。

### 3. ガードページ(第31章)

特定のページを `PAGE_NOACCESS` にすれば、**触った瞬間にアクセス違反** が起きます。

第17章のガードバイトの限界——遠くへの書き込みを見逃す、検出が遅れる——が、根本的に解決します。

---

## 演習

**演習29-1** 10 GB を予約してみてください。成功しますか。32 ビットビルドではどうですか。

**演習29-2** `kCommitChunk` を 4 KB / 1 MB に変えて、確保の最大値がどう変わるか測ってください。

**演習29-3** `Prewarm(bytes)` を実装してください。起動時に呼ぶと、確保の最大値はどうなりますか。

**演習29-4** `Trim()` を呼んだ直後に、同じ量を確保し直してください。時間はどうなりますか。

**演習29-5** `VirtualQuery` を使って、予約領域の状態(コミット済み/未コミット)を表示する関数を書いてください。

**演習29-6** `NOMINMAX` を定義せずに `<Windows.h>` をインクルードし、`std::numeric_limits<int>::max()` を呼んでみてください。エラーメッセージは読めますか。

**演習29-7** `VirtualAlloc` が返すアドレスが、常に 64 KB の倍数であることを確認してください。この性質は、第26章のバディにどう役立ちますか。

**演習29-8** `Pool` と `FreeList` と `Tlsf` の土台も `VirtualMemory` に差し替えてください。どの実装が最も恩恵を受けますか。

---

## 章末チェックリスト

- [ ] `std::vector<std::byte>(1GB)` が 1 GB の物理メモリを消費することを実測した
- [ ] 予約・コミット・初回アクセスの3段階を説明できる
- [ ] `WIN32_LEAN_AND_MEAN` と `NOMINMAX` の役割を説明できる
- [ ] **公開ヘッダに `<Windows.h>` を入れない** 理由を説明できる
- [ ] 第12章で括弧を付けた理由が分かった
- [ ] `VirtualMemory` クラスを実装し、`.cpp` に Windows API を隠した 〔v0.21〕
- [ ] 1 GB の予約が 28 µs、物理メモリ消費ほぼ 0 であることを確認した
- [ ] コミットのスパイクが最大値に現れることを確認し、対策を2つ挙げられる

---

## 次章の予告

予約したアドレス空間の上では、**再確保なしにデータを増やせます**。

```cpp
ga::GrowingArray<Particle> particles(1'000'000);   // 100 万個ぶん予約

particles.PushBack(p);   // 増える
particles.PushBack(p);   // 増える
// ...

Particle* first = &particles[0];
particles.PushBack(p);   // ← std::vector なら first が無効になるかもしれない
                         //    GrowingArray では絶対に無効にならない
```

**`std::vector` の最大の落とし穴——イテレータとポインタの無効化——が、構造的に起きなくなります。**

第30章では、これを実装し、`std::vector` と比較します。再確保のコストがゼロになるぶん、大量のデータを積み上げる場面で大きな差が出ます。

そして、この性質は性能以上に **設計上の自由** をもたらします。「ポインタを保持していいのか」を毎回悩まなくて済むのは、想像以上に大きな違いです。

---

> **コラム:予約とコミットの分離は、どこから来たのか**
>
> 「予約」と「コミット」を明示的に分ける API は、実は **Windows の特徴** です。
>
> ---
>
> **Unix 系の流儀**
>
> Linux や macOS の `mmap` には、この2段階がありません。
>
> ```c
> void* p = mmap(NULL, 1GB, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);
> ```
>
> この1行で 1 GB が「使える状態」になります。**しかし、物理メモリは消費しません。** 触ったページだけが、後から割り当てられます。
>
> これを **オーバーコミット** と呼びます。OS は、実際には用意できない量のメモリを「使える」と答えます。
>
> **多くの場合、これで問題ありません。** プログラムは確保した領域を全部使うとは限らないからです。
>
> **問題が起きるのは、本当に足りなくなったときです。** すでに「使える」と答えてしまっているので、断ることができません。そこで Linux は **OOM Killer** を動かし、プロセスを選んで強制終了させます。
>
> **メモリを使いすぎたプロセスとは限りません。** まったく無関係なプロセスが殺されることもあります。
>
> ---
>
> **Windows の流儀**
>
> Windows は、この曖昧さを嫌いました。
>
> **コミットした時点で、OS は「必ず用意する」と約束します。** 物理メモリかページファイルのどちらかで、確実に裏付けを取ります。裏付けが取れなければ、`VirtualAlloc` はその場で失敗します。
>
> ```
> コミットチャージ = 全プロセスがコミットした量の合計
> コミット上限     = 物理メモリ + ページファイルのサイズ
> ```
>
> **上限を超えるコミットは、できません。** 代わりに、Windows には OOM Killer がありません。約束を守れる範囲でしか約束しないからです。
>
> 「予約」は、この約束を **後回しにする** ための仕組みです。アドレス空間だけ押さえておいて、約束は必要になってから取る。
>
> ---
>
> **どちらが良いのか**
>
> 一長一短です。
>
> Unix 流は柔軟で、多くの場合うまく動きます。しかし、メモリ不足が起きたときの挙動が予測できません。
>
> Windows 流は厳格で、失敗が早い段階で分かります。しかし、実際には使わないメモリのためにページファイルを用意する必要があります。
>
> ---
>
> **ゲーム開発にとって**
>
> **Windows 流の厳格さは、むしろ好都合です。**
>
> 第7章のコラムで書いたとおり、ゲーム機の世界では「メモリ不足は設計ミス」です。実行時に回復するものではなく、開発中に検出して直すもの。
>
> **コミットが失敗するなら、その場で分かります。** OOM Killer に無関係なプロセスを殺されて、原因が分からないまま調査する——という事態は起きません。
>
> そして、この章で手に入れた「予約は大きく、コミットは小さく」という道具は、**ゲームのメモリ設計と非常に相性が良い**。
>
> ```
> 予約  : 起動時に、想定される最大量を大きめに確保
> コミット: 実際のレベルに応じて、必要な分だけ
> ```
>
> 第49章でメモリ予算を設計するとき、この2段階がそのまま「予算の枠」と「実使用量」に対応します。
