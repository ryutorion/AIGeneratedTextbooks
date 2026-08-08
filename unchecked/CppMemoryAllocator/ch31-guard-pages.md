# 第31章 ガードページを自分で置く 〔v0.23〕

---

## この章のゴール

第17章で作ったガードバイトには、正直に書いた限界がありました。

> **遠くへの書き込みは素通りする。検出が遅れる。読み取りは検出できない。**

この章で、根本的に解決します。**ソフトウェアで見張るのをやめて、ハードウェアに見張らせます。**

```cpp
// このページに触れたら、その瞬間に例外
VirtualAlloc(guardPage, 4096, MEM_RESERVE, PAGE_NOACCESS);
```

- ページ保護によって、はみ出した **瞬間** に止める 〔**v0.23**〕
- 構造化例外を捕まえて、**どの確保のどこにはみ出したか** を報告する
- 代償(メモリ 128 倍、速度 4000 倍)を測り、使いどころを定める
- Windows 標準の **PageHeap / Application Verifier** との関係
- **AddressSanitizer** を有効にし、自作アロケーターの意味を教える方法

第2部で作った検出の道具立てが、ここで完成します。

---

## 31.1 ハードウェアに見張らせる

CPU は、メモリアクセスのたびに **ページテーブル** を参照します。そこには「このページは読めるか、書けるか」という情報が入っています。

**アクセス権のないページに触れば、CPU が例外を発生させます。** ソフトウェアの検査は一切要りません。**コストはゼロです。**

### Windows のページ保護

```cpp
VirtualProtect(address, size, PAGE_NOACCESS, &oldProtect);
```

| 保護属性 | 意味 |
|---|---|
| `PAGE_READWRITE` | 読み書きできる |
| `PAGE_READONLY` | 読めるが書けない |
| **`PAGE_NOACCESS`** | **触ると例外** |
| `PAGE_GUARD` | 触ると **1回だけ** 例外。その後は普通のページになる |

`PAGE_GUARD` は、Windows がスタックの自動拡張に使っている仕組みです。スタックの末端に置いておき、触られたら「スタックが伸びた」と判断してページを追加します。**1回限りなので、私たちの用途には向きません。**

**`PAGE_NOACCESS` を使います。**

### 予約したままのページも、触れば例外

もう1つ、うまい手があります。

**コミットしていないページに触っても、アクセス違反になります。**

```cpp
// ガードページはコミットしない。予約したまま
VirtualAlloc(base, totalSize, MEM_RESERVE, PAGE_NOACCESS);

// データ部分だけコミットする
VirtualAlloc(base, dataSize, MEM_COMMIT, PAGE_READWRITE);
```

**ガードページのコミットチャージがゼロになります。** 第29章で学んだ「予約とコミットの分離」が、ここで効いてきます。

`VirtualProtect` を呼ぶ必要すらありません。**何もしないことが、保護になります。**

---

## 31.2 レイアウトを設計する

保護の単位は **ページ(4 KB)** です。したがって、**ユーザー領域の境界を、ページ境界に合わせる** 必要があります。

### 末尾ガード(オーバーラン検出)

```
                        ページ境界
                            ↓
┌───────────────────────────┬─────────────┐
│  (未使用)  │ ユーザー領域  │  ガードページ │
│           │   32 バイト   │  (予約のみ)   │
└───────────────────────────┴─────────────┘
             ▲               ▲
           user          user + 32 = ページ境界
```

**ユーザー領域の末尾を、ページ境界にぴったり合わせます。**

`user[32]` に触れば、それはガードページです。**1バイトはみ出した瞬間に落ちます。**

前側の「未使用」部分は無駄になりますが、これは仕方ありません。

### 先頭ガード(アンダーラン検出)

```
  ページ境界
      ↓
┌─────────────┬───────────────────────────┐
│ ガードページ │ ユーザー領域  │  (未使用)   │
│  (予約のみ)  │   32 バイト   │            │
└─────────────┴───────────────────────────┘
               ▲
             user = ページ境界
```

`user[-1]` に触れば落ちます。

### 両方は同時にできない

**1つの確保に対して、末尾と先頭の両方をページ境界に合わせることはできません。** サイズがページの倍数でない限り、どちらか一方です。

これは私たちの実装の都合ではなく、**ページ保護という仕組みの本質的な制約** です。後述する Windows の PageHeap も、同じ選択を迫られます。

**既定は末尾ガードにします。** 実際のバグは、オーバーラン(後ろへのはみ出し)が圧倒的に多いためです。

---

## 31.3 実装する 〔v0.23〕

```cpp
// ga/GuardedAllocator.h(公開ヘッダ:Windows.h を含まない)
#pragma once

#include "ga/Core.h"

#include <cstddef>
#include <source_location>

namespace ga
{
    enum class GuardSide
    {
        Back,    // オーバーラン検出(既定)
        Front,   // アンダーラン検出
    };

    // 1確保につき最低 2 ページを使う、極めて重いデバッグ用アロケーター。
    // 「常時使う」ものではなく「バグを追い詰めるとき」の道具。
    class GuardedAllocator
    {
    public:
        explicit GuardedAllocator(GuardSide side = GuardSide::Back,
                                  bool holdAddressOnFree = true);
        ~GuardedAllocator();

        GuardedAllocator(const GuardedAllocator&)            = delete;
        GuardedAllocator& operator=(const GuardedAllocator&) = delete;

        [[nodiscard]]
        void* Allocate(std::size_t size,
                       std::size_t alignment = kDefaultAlignment,
                       const std::source_location& loc = std::source_location::current());

        void Free(void* p);

        // アクセス違反が起きたアドレスについて説明する
        // (どの確保の、どれだけ外側か)
        bool Describe(const void* faultAddress, std::string& out) const;

        std::size_t LiveCount()      const noexcept;
        std::size_t ReservedBytes()  const noexcept;
        std::size_t CommittedBytes() const noexcept;

    private:
        struct Impl;
        std::unique_ptr<Impl> impl_;
    };
}
```

**`Impl` を前方宣言して、実装を隠しています。** いわゆる pimpl です。第13章の方針(公開ヘッダに `<Windows.h>` を入れない)を守るための定石です。

### 実装

```cpp
// AllocatorLib/src/GuardedAllocator.cpp
#include "pch.h"

#define WIN32_LEAN_AND_MEAN
#define NOMINMAX
#include <Windows.h>

#include "ga/GuardedAllocator.h"

#include <format>
#include <unordered_map>

namespace ga
{
    struct GuardedAllocator::Impl
    {
        struct Record
        {
            std::byte*           base       = nullptr;   // 予約の先頭
            std::size_t          totalBytes = 0;         // 予約の総量
            std::byte*           user       = nullptr;   // 返したアドレス
            std::size_t          userSize   = 0;
            bool                 freed      = false;
            std::source_location loc{};
        };

        GuardSide side;
        bool      holdAddressOnFree;
        std::size_t pageSize = 0;

        std::unordered_map<const void*, Record> records;   // user → Record
        std::size_t reservedBytes  = 0;
        std::size_t committedBytes = 0;
    };

    namespace
    {
        std::size_t QueryPageSize() noexcept
        {
            SYSTEM_INFO info{};
            GetSystemInfo(&info);
            return info.dwPageSize;
        }
    }

    GuardedAllocator::GuardedAllocator(GuardSide side, bool holdAddressOnFree)
        : impl_(std::make_unique<Impl>())
    {
        impl_->side              = side;
        impl_->holdAddressOnFree = holdAddressOnFree;
        impl_->pageSize          = QueryPageSize();
    }

    GuardedAllocator::~GuardedAllocator()
    {
        for (auto& [user, rec] : impl_->records)
        {
            VirtualFree(rec.base, 0, MEM_RELEASE);
        }
    }

    void* GuardedAllocator::Allocate(std::size_t size, std::size_t alignment,
                                     const std::source_location& loc)
    {
        if (size == 0) { return nullptr; }

        const std::size_t page      = impl_->pageSize;
        const std::size_t dataPages = AlignUpSize(size, page);
        const std::size_t total     = dataPages + page;      // データ + ガード1ページ

        // ① 全体を予約する(ガードページはコミットしないので、常に例外になる)
        void* raw = VirtualAlloc(nullptr, total, MEM_RESERVE, PAGE_NOACCESS);
        if (raw == nullptr) { return nullptr; }

        auto* base = static_cast<std::byte*>(raw);

        std::byte* dataBegin = nullptr;
        std::byte* user      = nullptr;

        if (impl_->side == GuardSide::Back)
        {
            // [データ領域][ガード]
            dataBegin = base;
            user      = base + dataPages - size;
            user      = reinterpret_cast<std::byte*>(
                            AlignDown(reinterpret_cast<std::uintptr_t>(user), alignment));
        }
        else
        {
            // [ガード][データ領域]
            dataBegin = base + page;
            user      = dataBegin;
        }

        // ② データ部分だけコミットする
        if (VirtualAlloc(dataBegin, dataPages, MEM_COMMIT, PAGE_READWRITE) == nullptr)
        {
            VirtualFree(base, 0, MEM_RELEASE);
            return nullptr;
        }

#if GA_ENABLE_MEMORY_PATTERN
        std::memset(dataBegin, 0xCD, dataPages);
#endif

        impl_->records.emplace(user, Impl::Record{ base, total, user, size, false, loc });
        impl_->reservedBytes  += total;
        impl_->committedBytes += dataPages;

        return user;
    }

    void GuardedAllocator::Free(void* p)
    {
        if (p == nullptr) { return; }

        auto it = impl_->records.find(p);
        if (it == impl_->records.end())
        {
            assert(false && "このアロケーターのポインタではありません");
            return;
        }

        Impl::Record& rec = it->second;

        if (rec.freed)
        {
            assert(false && "二重解放です");
            return;
        }

        if (impl_->holdAddressOnFree)
        {
            // コミットだけ解除する。予約は保持したままなので、
            // 解放後にアクセスすると必ずアクセス違反になる
            VirtualFree(rec.base, 0, MEM_DECOMMIT);
            rec.freed = true;
            impl_->committedBytes -= (rec.totalBytes - impl_->pageSize);
        }
        else
        {
            VirtualFree(rec.base, 0, MEM_RELEASE);
            impl_->reservedBytes  -= rec.totalBytes;
            impl_->committedBytes -= (rec.totalBytes - impl_->pageSize);
            impl_->records.erase(it);
        }
    }
}
```

### 設計上のポイント

**ガードページはコミットしない。** 31.1 節で述べたとおりです。`VirtualProtect` の呼び出しが不要になり、コミットチャージも消費しません。

**`holdAddressOnFree` が強力。** 解放時にコミットだけ解除し、予約は保持します。すると、**そのアドレスは二度と他の確保に使われません**。解放後にアクセスすれば、必ずアクセス違反になります。

第17章で「Use-after-free を検出するための遅延解放(quarantine)」に触れましたが、**この方式では遅延ではなく永久です。** アドレス空間を消費し続けますが、64 ビットなら実質無限です。

**記録をシステムのヒープに持つ。** `std::unordered_map` を使っています。第18章で述べた原則(デバッグ機能は自分自身を使わない)に従っています。

---

## 31.4 はみ出した瞬間に止まる

```cpp
int main()
{
    ga::GuardedAllocator guarded;

    auto* p = static_cast<std::byte*>(guarded.Allocate(32));

    std::println("確保: {}", static_cast<void*>(p));

    p[31] = std::byte{ 1 };     // 最後の1バイト。OK
    std::println("p[31] への書き込み成功");

    p[32] = std::byte{ 1 };     // ← 1バイトはみ出した

    std::println("ここには到達しない");
}
```

```
確保: 0x1f3a8c2bfe0
p[31] への書き込み成功
```

**そこで止まります。** Visual Studio では、`p[32] = ...` の行でアクセス違反が報告されます。

```
例外がスローされました: 書き込みアクセス違反。
場所は 0x000001F3A8C2C000 です。
```

**書いた瞬間、その命令で止まっています。** 第17章のように「`Reset()` のときに気づく」のではありません。

### 読み取りも検出できる

```cpp
    const std::byte x = p[32];   // 読むだけ
```

```
例外がスローされました: 読み取りアクセス違反。
```

**ガードバイトでは不可能だった検出です。**

### 遠くへのアクセスも検出できる

```cpp
    p[100] = std::byte{ 1 };     // 大きくはみ出す
```

ガードページは 4 KB あるので、**+32 から +4095 までのどこに触っても捕まります**。

第17章の 16 バイトのガードバイトは、`p[100]` を素通りさせていました。

### 解放後のアクセスも検出できる

```cpp
    guarded.Free(p);
    p[0] = std::byte{ 1 };       // 解放後に触る
```

```
例外がスローされました: 書き込みアクセス違反。
```

`holdAddressOnFree` が有効なので、そのアドレスは永久にアクセス不能です。

---

## 31.5 例外を捕まえて、詳しく報告する

「アクセス違反が起きた」だけでは、まだ情報が足りません。**どの確保の、どれだけ外側か** を知りたい。

Windows の **ベクタ化例外ハンドラ** を使います。

```cpp
// Playground/CrashReport.cpp
#define WIN32_LEAN_AND_MEAN
#define NOMINMAX
#include <Windows.h>

#include "ga/GuardedAllocator.h"

namespace
{
    ga::GuardedAllocator* g_guarded = nullptr;

    LONG CALLBACK OnException(EXCEPTION_POINTERS* info)
    {
        const auto* rec = info->ExceptionRecord;

        if (rec->ExceptionCode != EXCEPTION_ACCESS_VIOLATION)
        {
            return EXCEPTION_CONTINUE_SEARCH;
        }

        const ULONG_PTR kind    = rec->ExceptionInformation[0];   // 0=読み, 1=書き
        const auto*     address = reinterpret_cast<const void*>(rec->ExceptionInformation[1]);

        std::println("");
        std::println("=== アクセス違反 ===");
        std::println("  アドレス : {}", address);
        std::println("  種別     : {}", (kind == 1) ? "書き込み" : "読み取り");

        std::string detail;
        if (g_guarded != nullptr && g_guarded->Describe(address, detail))
        {
            std::println("  詳細     : {}", detail);
        }

        std::println("  現在のスタック:");
        ga::PrintTrace(std::stacktrace::current(2, 10), "    ");

        return EXCEPTION_CONTINUE_SEARCH;   // 通常の処理(デバッガ/終了)に任せる
    }
}

void InstallCrashReport(ga::GuardedAllocator& guarded)
{
    g_guarded = &guarded;
    AddVectoredExceptionHandler(1, &OnException);
}
```

`ExceptionInformation` の意味は決まっています。

| 添字 | 内容 |
|---|---|
| `[0]` | 0 = 読み取り、1 = 書き込み、8 = 実行(DEP) |
| `[1]` | **アクセスしようとしたアドレス** |

### `Describe` の実装

```cpp
    bool GuardedAllocator::Describe(const void* faultAddress, std::string& out) const
    {
        const auto* fault = static_cast<const std::byte*>(faultAddress);

        for (const auto& [user, rec] : impl_->records)
        {
            if (fault < rec.base || fault >= rec.base + rec.totalBytes) { continue; }

            const std::ptrdiff_t offset = fault - rec.user;

            std::string_view file = rec.loc.file_name();
            if (auto pos = file.find_last_of("\\/"); pos != std::string_view::npos)
            {
                file = file.substr(pos + 1);
            }

            if (rec.freed)
            {
                out = std::format("{}:{} で確保し、すでに解放済みの {} バイトの領域(+{})",
                                  file, rec.loc.line(), rec.userSize, offset);
            }
            else if (offset >= static_cast<std::ptrdiff_t>(rec.userSize))
            {
                out = std::format("{}:{} で確保した {} バイトの領域の {} バイト後ろ",
                                  file, rec.loc.line(), rec.userSize,
                                  offset - static_cast<std::ptrdiff_t>(rec.userSize));
            }
            else
            {
                out = std::format("{}:{} で確保した {} バイトの領域の {} バイト手前",
                                  file, rec.loc.line(), rec.userSize, -offset);
            }
            return true;
        }
        return false;
    }
```

### 出力

```
=== アクセス違反 ===
  アドレス : 0x000001F3A8C2C000
  種別     : 書き込み
  詳細     : main.cpp:42 で確保した 32 バイトの領域の 0 バイト後ろ
  現在のスタック:
    UpdateParticles   ParticleSystem.cpp(128)
    UpdateFrame       Game.cpp(64)
    main              main.cpp(20)
```

**「どこで確保された、何バイトの領域の、どれだけ外側に、読み書きどちらでアクセスしたか」** が、1画面で分かります。

第14章の `source_location`、第18章の `std::stacktrace`、そしてこの章のページ保護。**3つが揃って、この報告が成立しています。**

---

## 31.6 代償

### メモリ

```
32 バイトの確保:
  コミット      : 4,096 バイト(データページ)
  アドレス空間  : 8,192 バイト(データ + ガード)

  → コミットで 128 倍、アドレス空間で 256 倍
```

**1万個の確保で 40 MB。** 通常なら 320 KB です。

`holdAddressOnFree` を有効にすると、**解放してもアドレス空間は返りません**。長時間動かすと、予約が積み上がります。

### 速度

```
Bump::Allocate           2.1 ns
GuardedAllocator::Allocate  8,400 ns
```

**4000 倍。** システムコールが2回(予約とコミット)発生するためです。

解放も、`VirtualFree` のシステムコールで 5 マイクロ秒程度かかります。

### だから「常時」ではない

第17章のガードバイトは、Debug 構成で常時有効にできました。**ガードページは違います。**

> **これは「バグを追い詰めるときだけ使う道具」です。**

典型的な使い方:

1. 「メモリが壊れているらしい」という症状が出る
2. 第17章のガードバイトで、おおよその位置を絞る
3. **疑わしい確保だけを** `GuardedAllocator` に切り替える
4. 一発で犯人が分かる

「全部の確保をガードページにする」のは、小さなテストケースでのみ現実的です。

---

## 31.7 Windows 標準の道具:PageHeap

実は、**Windows は同じ機能を標準で提供しています。**

**PageHeap**(ページヒープ)は、`malloc` / `new` が確保する全ブロックに、ガードページを付ける仕組みです。**アプリケーションのコードを1行も変えずに** 有効化できます。

### 有効にする

**Application Verifier** を使う方法が一般的です。

1. Application Verifier を起動する(Windows SDK に含まれます)
2. 対象の実行ファイルを追加する
3. **Basics → Heaps** にチェックを入れる
4. 実行する

または、`gflags.exe` をコマンドラインで使います。

```
gflags /p /enable MyGame.exe /full
```

`/full` が **完全ページヒープ** で、この章で作ったものと同じ動作をします。`/full` なしだと軽量版(ガードバイト方式)になります。

### 何が起きるか

```
                通常              PageHeap 有効時
new(32)     → 48 バイト消費   →  8 KB 消費
はみ出し     → 気づかない      →  即座にアクセス違反
```

**私たちが作ったものと、まったく同じです。**

### では、なぜ自分で作ったのか

**PageHeap は `malloc` / `new` にしか効きません。**

第17章のコラムで、CRT デバッグヒープについて同じことを書きました。

> 自作アロケーターを使うということは、既存のデバッグ支援を1つ失うということでもある。

`Bump` や `Pool` が配るメモリは、OS から見れば「1回の大きな確保」の内側です。PageHeap には見えません。**自分で作るしかありませんでした。**

**とはいえ、PageHeap も併用すべきです。** サードパーティのライブラリや、CRT 自身が確保するメモリは、PageHeap でしか見張れません。

---

## 31.8 AddressSanitizer

より強力な選択肢が、Visual Studio 2019 以降に組み込まれています。

### 有効にする

**プロジェクトのプロパティ(すべての構成)で:**

| 項目 | 設定 |
|---|---|
| C/C++ → 全般 → **アドレス サニタイザー** | **はい (/fsanitize=address)** |
| C/C++ → コード生成 → **基本ランタイム チェック** | **既定** |
| リンカー → 全般 → **インクリメンタル リンクを有効にする** | **いいえ** |

**基本ランタイムチェック(`/RTC1`)との併用はできません。** Debug 構成では既定で有効になっているので、切る必要があります。

### 何ができるか

```cpp
int main()
{
    int* p = new int[10];
    p[10] = 1;              // 範囲外
    delete[] p;
}
```

```
=================================================================
==12345==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x...
WRITE of size 4 at 0x... thread T0
    #0 main main.cpp:4
0x... is located 0 bytes after 40-byte region [0x...,0x...)
allocated by thread T0 here:
    #0 operator new[]
    #1 main main.cpp:3
```

**私たちが 31.5 節で苦労して作った報告と、ほぼ同じ内容です。** しかも、確保時のスタックトレースまで自動で出ます。

さらに、ASan は次のものも検出します。

- ヒープの範囲外(読み書き)
- **スタックの範囲外**
- **グローバル変数の範囲外**
- 解放後の使用
- 二重解放
- メモリリーク
- スタックのスコープ外使用

**ページ保護では届かない領域(スタック、グローバル)まで見張れます。**

### 仕組み

ASan は **シャドウメモリ** を使います。プログラムの 8 バイトごとに、シャドウメモリの 1 バイトが対応します。

```
通常のメモリ 8 バイト → シャドウ 1 バイト
  「この 8 バイトのうち、何バイトが有効か」を記録
```

コンパイラが、**すべてのメモリアクセスの前に検査コードを挿入** します。

```cpp
// 元のコード
p[i] = 1;

// ASan が挿入するコード(概念)
if (IsPoisoned(&p[i])) { ReportError(); }
p[i] = 1;
```

**メモリのオーバーヘッドは約 1/8 + α、速度は 2〜3 倍遅くなります。** ガードページの 128 倍・4000 倍と比べれば、はるかに軽量です。

### 自作アロケーターに ASan の目を効かせる

**ここが重要です。**

ASan は `malloc` / `new` を置き換えて動作します。**私たちの `Bump` が板の内側で何をしているかは、ASan にも見えません。**

しかし、**教えることができます。**

```cpp
#include <sanitizer/asan_interface.h>

// この領域は「使用不可」だと ASan に伝える
__asan_poison_memory_region(ptr, size);

// この領域は「使用可能」だと ASan に伝える
__asan_unpoison_memory_region(ptr, size);
```

`Bump` に組み込みます。

```cpp
    explicit Bump(std::size_t capacity) : memory_(capacity)
    {
        base_ = memory_.Base();

        // 板全体を「使用不可」にしておく
        GA_ASAN_POISON(base_, memory_.Reserved());
    }

    AllocResult Allocate(...) noexcept
    {
        // ... 従来の処理 ...

        // 配った領域だけ「使用可能」にする
        GA_ASAN_UNPOISON(result, size);

        return result;
    }

    void Reset() noexcept
    {
        // ... 従来の処理 ...

        // 全部「使用不可」に戻す
        GA_ASAN_POISON(base_, memory_.Committed());
    }
```

マクロで包んでおきます。

```cpp
// ga/Core.h
#if defined(__SANITIZE_ADDRESS__) || defined(GA_USE_ASAN)
#  include <sanitizer/asan_interface.h>
#  define GA_ASAN_POISON(p, n)   __asan_poison_memory_region((p), (n))
#  define GA_ASAN_UNPOISON(p, n) __asan_unpoison_memory_region((p), (n))
#else
#  define GA_ASAN_POISON(p, n)   ((void)0)
#  define GA_ASAN_UNPOISON(p, n) ((void)0)
#endif
```

**これで、`Bump` から確保した領域のはみ出しも、`Reset()` 後の使用も、ASan が検出してくれます。**

第17章のコラムで「自作アロケーターを使うと既存のデバッグ支援を失う」と書きました。**ASan については、その損失を取り戻せます。**

> **環境によってヘッダのパスやマクロ名が異なることがあります。** コンパイルが通らない場合は、Visual Studio のドキュメントで現在の指定方法を確認してください。

---

## 31.9 使い分け

| 手段 | 検出できるもの | メモリ | 速度 | 自作アロケーターに効くか |
|---|---|---|---|---|
| **塗りつぶし**(第16章) | 未初期化読み、解放後読み(間接的) | +0% | 1.3倍遅 | ○ |
| **ガードバイト**(第17章) | 隣接のはみ出し(書き込みのみ)、遅延検出 | +96 B/確保 | 4倍遅 | ○ |
| **ガードページ**(この章) | 任意距離のはみ出し、読み書き両方、**即時**、解放後使用 | **128倍** | **4000倍遅** | ○ |
| **PageHeap** | 同上 | 同上 | 同上 | **×**(`new`/`malloc` のみ) |
| **AddressSanitizer** | 上記ほぼ全部 + スタック + グローバル | **+約1/8** | **2〜3倍遅** | △(poison を教えれば ○) |

### 推奨する順序

1. **日常の開発**:塗りつぶし + ガードバイト(Debug 構成で常時)
2. **定期的に**:AddressSanitizer でテストを流す(CI で回すのが理想)
3. **バグを追い詰めるとき**:ガードページを、疑わしい箇所だけに
4. **サードパーティを疑うとき**:PageHeap

**ASan が最もコストパフォーマンスに優れています。** 2〜3 倍の速度低下は、テスト実行なら許容できる範囲です。

**ガードページは最終手段です。** しかし、ASan が使えない状況(特定のプラットフォーム、他のツールとの競合)では、これが唯一の手段になります。**自分で作れることに意味があります。**

---

## 演習

**演習31-1** 先頭ガード(`GuardSide::Front`)で `p[-1]` に触れてください。検出されますか。末尾ガードでは検出できますか。

**演習31-2** `holdAddressOnFree` を `false` にして、解放後のアクセスを試してください。検出されますか。何度か繰り返すとどうなりますか。

**演習31-3** 4096 バイトぴったりの確保をすると、末尾ガードと先頭ガードの両方が成立します。実装を確認してください。

**演習31-4** `GuardedAllocator` で 1 万個確保し、タスクマネージャーでコミットサイズを確認してください。予想と一致しますか。

**演習31-5** AddressSanitizer を有効にして、第23章の `FreeList` に範囲外書き込みをさせてください。`GA_ASAN_POISON` を入れる前と後で、検出できるかが変わりますか。

**演習31-6** ASan を有効にして、第28章のベンチマークを実行してください。何倍遅くなりますか。

**演習31-7** ガードページと ASan を同時に有効にすると何が起きますか。併用すべきでしょうか。

**演習31-8** `Describe` は線形探索です。確保が 10 万個あるとき、例外ハンドラの中でどれくらいかかりますか。改善する必要はありますか。

---

## 章末チェックリスト

- [ ] `PAGE_NOACCESS` と `PAGE_GUARD` の違いを説明できる
- [ ] **ガードページをコミットしない** ことの利点を説明できる
- [ ] 末尾ガードと先頭ガードを両立できない理由を説明できる
- [ ] `GuardedAllocator` を実装した 〔v0.23〕
- [ ] 1バイトのはみ出しで、その場で止まることを確認した
- [ ] 読み取りのはみ出しも検出できることを確認した
- [ ] 例外ハンドラで、確保元と外れた距離を報告させた
- [ ] メモリ 128 倍・速度 4000 倍という代償を測った
- [ ] AddressSanitizer を有効にし、`__asan_poison_memory_region` で自作アロケーターを教えた

---

## 次章の予告

第4部の最後は、**性能の話に戻ります**。

第21章で、こんな結果を見ました。

```
走査 (new 版、散在)   3.8 ns/個
走査 (Pool 版、連続)  0.4 ns/個
```

**9.5 倍の差。** 確保速度ではなく、**確保されたメモリの配置** が生んだ差です。

第32章では、この現象を正面から扱います。

- キャッシュライン、プリフェッチ、TLB
- 連番でたどる配列と、ポインタでたどる配列
- リンクリスト vs 連続配列(同じ要素数)

そして、この本を通して繰り返してきた言葉を、最後に数字で裏づけます。

> **アロケーターの仕事は、速く返すことではなく、良い場所を返すこと。**

---

> **コラム:ページフォルトを味方につける**
>
> この章では、アクセス違反を **バグの通知** として使いました。しかし、ページ保護の応用範囲はもっと広い。**「例外」を「通知」として使う技法** は、OS やランタイムのあちこちで見つかります。
>
> ---
>
> **デマンドページング**
>
> 第29章で見たとおりです。コミットしたページに初めて触ると、ページフォルトが起きて OS が物理メモリを割り当てます。
>
> **「使われるまで割り当てない」を、ページフォルトで実現しています。** アプリケーションは何も意識しません。
>
> ---
>
> **コピーオンライト**
>
> プロセスを複製するとき(Unix の `fork`)、メモリ全体をコピーするのは無駄です。**読むだけなら共有できます。**
>
> そこで、両方のプロセスのページを **読み取り専用** にしておきます。どちらかが書き込もうとするとページフォルトが起き、OS がその時点で初めてコピーを作ります。
>
> **「書かれるまでコピーしない」を、ページ保護で実現しています。**
>
> ---
>
> **スタックの自動拡張**
>
> 31.1 節で触れた `PAGE_GUARD` の用途です。Windows は、スタックの末端に1ページのガードを置いています。
>
> スタックが伸びてそのページに触れると、例外が発生します。OS はそれを捕まえ、**「スタックが伸びた」と解釈して** 新しいページをコミットし、さらに1つ下にガードを置き直します。
>
> **スタックオーバーフローの検出も、同じ仕組みです。** 予約の限界まで来たら、今度は本物の例外として通知されます。
>
> ---
>
> **メモリマップトファイル**
>
> ファイルをメモリにマップすると、ファイルの内容がアドレス空間に現れます。読み書きは普通のメモリアクセスです。
>
> 実際のファイル読み込みは、**そのページに触ったときのページフォルトで** 行われます。第47章で「ファイルをそのまま載せる」を扱うとき、この仕組みを使います。
>
> ---
>
> **ガベージコレクタの書き込み検出**
>
> 世代別 GC では、「古い世代のオブジェクトが、新しい世代を指すようになった」ことを知る必要があります。
>
> 一部の実装は、古い世代のページを **読み取り専用** にしておきます。書き込みがあればページフォルトが起きるので、「このページに変更があった」と記録できます。
>
> **すべての書き込みにチェックコードを挿入する代わりに、ハードウェアに検出させています。**
>
> ---
>
> **共通する構図**
>
> どれも同じ形をしています。
>
> > **「めったに起きないこと」を、ハードウェアの例外機構に任せる。**
> > **普段の実行にはコストをかけず、起きたときだけ処理する。**
>
> 私たちのガードページも、まさにこれです。**普段のメモリアクセスは1命令も増えません。** はみ出したときだけ、CPU が教えてくれます。
>
> AddressSanitizer が「すべてのアクセスに検査コードを挿入する」方式なのと、対照的です。ASan は柔軟で検出力が高いが、常時コストがかかる。ガードページはコストゼロだが、粒度が粗い。
>
> **「頻繁な処理はハードウェアに任せ、まれな処理をソフトウェアで書く」** ——これは性能設計の基本形の1つです。第32章で扱うキャッシュも、同じ思想で作られています。
