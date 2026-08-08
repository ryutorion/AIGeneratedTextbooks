# 第41章 `operator new` を置き換える 〔v0.30〕

---

## この章のゴール

ここまでは「使う側が自作アロケーターを明示的に指定する」形でした。**それをやめます。**

```cpp
void* operator new(std::size_t size)
{
    return ga::GlobalAllocate(size, __STDCPP_DEFAULT_NEW_ALIGNMENT__);
}
```

**この関数を定義するだけで、プログラム中のすべての `new` が自作アロケーターを通ります。**

サードパーティのライブラリも、標準ライブラリも、自分が書いていないコードも。**1行も変更せずに。**

- 置換できる **20 個のシグネチャ** をすべて実装する 〔**v0.30**〕
- **静的初期化順序** の地雷を避ける
- **知らないポインタが解放されに来る** 問題に対処する
- **DLL 境界** と **リンク順** の落とし穴
- クラス単位の置換という、より安全な選択肢

**この本で最も「踏むと痛い」章です。**

---

## 41.1 何ができるようになるか

### 用途1:すべての確保を計測する

第15章で作った統計とタグが、**プログラム全体に適用されます**。

```cpp
{
    GA_TAG(ga::MemoryTag::Texture);
    LoadTextureWithThirdPartyLibrary("stage.png");   // ← 中身は自分で書いていない
}

ga::PrintReport();
```

```
  タグ         使用量     割合    件数
  Texture     3.10 MB   34.0%      48
```

**サードパーティのライブラリが何バイト使ったかが分かります。** これは他の方法では得られません。

### 用途2:リーク検出

第18章の `std::stacktrace` と組み合わせれば、**どのコードがリークしているか** を特定できます。第34章の TOC で「リーク検出器」に触れましたが、**その本格版がここで可能になります。**

### 用途3:性能の底上げ

`new` を高速なアロケーターに差し替えるだけで、既存のコードが速くなります。**第53章で扱う「mimalloc を差し替えるだけで済むケース」の、自作版です。**

---

## 41.2 置換できる関数は 20 個ある

**「`operator new` を1つ書けばいい」と思っていたら、間違いです。**

### `new` 側:8個

| # | シグネチャ | 追加された版 |
|---|---|---|
| 1 | `void* operator new(size_t)` | C++98 |
| 2 | `void* operator new(size_t, align_val_t)` | **C++17** |
| 3 | `void* operator new(size_t, const nothrow_t&) noexcept` | C++98 |
| 4 | `void* operator new(size_t, align_val_t, const nothrow_t&) noexcept` | **C++17** |
| 5〜8 | 上記の `operator new[]` 版 | 同上 |

### `delete` 側:12個

| # | シグネチャ | 追加された版 |
|---|---|---|
| 9 | `void operator delete(void*) noexcept` | C++98 |
| 10 | `void operator delete(void*, size_t) noexcept` | **C++14** |
| 11 | `void operator delete(void*, align_val_t) noexcept` | **C++17** |
| 12 | `void operator delete(void*, size_t, align_val_t) noexcept` | **C++17** |
| 13 | `void operator delete(void*, const nothrow_t&) noexcept` | C++98 |
| 14 | `void operator delete(void*, align_val_t, const nothrow_t&) noexcept` | **C++17** |
| 15〜20 | 上記の `operator delete[]` 版 | 同上 |

**合計 20 個です。**

### なぜこんなに増えたのか

**C++14 の「サイズ付き `delete`」。**

```cpp
void operator delete(void* p, std::size_t size) noexcept;
```

第20章で「サイズが分からない」問題を挙げました。**呼び出し側がサイズを知っているなら、渡してもらえばヘッダが要りません。**

コンパイラは、型が分かっている場合に `sizeof(T)` を渡してくれます。第21章のプールや、第36章のサイズクラス方式で活きます。

**C++17 の「アライン対応」。**

```cpp
struct alignas(64) CacheLine { ... };
new CacheLine;    // ← C++14 までは 16 バイト境界しか保証されなかった
```

第6章で見たとおり、`__STDCPP_DEFAULT_NEW_ALIGNMENT__` を超えるアラインメントは、専用のオーバーロードに回されます。

**第38章のコラムで書いた「EASTL は 10 年以上前にこの問題を解いていた」という話が、ここに繋がります。**

### 一部でも欠けると壊れる

**20 個のうち1つでも定義し忘れると、その経路だけ標準の `new` が使われます。**

```
自作の operator new で確保
→ 標準の operator delete で解放      ← 破滅
```

**必ず全部書いてください。**

---

## 41.3 実装する 〔v0.30〕

### 中核となる2つの関数

```cpp
// ga/GlobalAllocator.h
#pragma once

#include <cstddef>

namespace ga
{
    // 失敗したら nullptr(例外は投げない)
    void* GlobalAllocate(std::size_t size, std::size_t alignment) noexcept;

    // size が不明な場合は 0 を渡す
    void  GlobalFree(void* p, std::size_t size, std::size_t alignment) noexcept;

    // 統計の取得など
    void  PrintGlobalReport();
}
```

### 20 個の実装

**すべてを、この2つに集約します。**

```cpp
// Playground/GlobalNew.cpp
//   ⚠ 静的ライブラリではなく、EXE のソースに直接置くこと(41.7 節)

#include "pch.h"
#include "ga/GlobalAllocator.h"

#include <new>

namespace
{
    constexpr std::size_t kDefaultAlign = __STDCPP_DEFAULT_NEW_ALIGNMENT__;

    void* AllocateOrThrow(std::size_t size, std::size_t alignment)
    {
        if (void* p = ga::GlobalAllocate(size, alignment)) { return p; }
        throw std::bad_alloc{};
    }
}

// ---------- new(8個) ----------

void* operator new(std::size_t size)
{
    return AllocateOrThrow(size, kDefaultAlign);
}

void* operator new[](std::size_t size)
{
    return AllocateOrThrow(size, kDefaultAlign);
}

void* operator new(std::size_t size, std::align_val_t align)
{
    return AllocateOrThrow(size, static_cast<std::size_t>(align));
}

void* operator new[](std::size_t size, std::align_val_t align)
{
    return AllocateOrThrow(size, static_cast<std::size_t>(align));
}

void* operator new(std::size_t size, const std::nothrow_t&) noexcept
{
    return ga::GlobalAllocate(size, kDefaultAlign);
}

void* operator new[](std::size_t size, const std::nothrow_t&) noexcept
{
    return ga::GlobalAllocate(size, kDefaultAlign);
}

void* operator new(std::size_t size, std::align_val_t align, const std::nothrow_t&) noexcept
{
    return ga::GlobalAllocate(size, static_cast<std::size_t>(align));
}

void* operator new[](std::size_t size, std::align_val_t align, const std::nothrow_t&) noexcept
{
    return ga::GlobalAllocate(size, static_cast<std::size_t>(align));
}

// ---------- delete(12個) ----------

void operator delete(void* p) noexcept
{
    ga::GlobalFree(p, 0, kDefaultAlign);
}

void operator delete[](void* p) noexcept
{
    ga::GlobalFree(p, 0, kDefaultAlign);
}

void operator delete(void* p, std::size_t size) noexcept
{
    ga::GlobalFree(p, size, kDefaultAlign);
}

void operator delete[](void* p, std::size_t size) noexcept
{
    ga::GlobalFree(p, size, kDefaultAlign);
}

void operator delete(void* p, std::align_val_t align) noexcept
{
    ga::GlobalFree(p, 0, static_cast<std::size_t>(align));
}

void operator delete[](void* p, std::align_val_t align) noexcept
{
    ga::GlobalFree(p, 0, static_cast<std::size_t>(align));
}

void operator delete(void* p, std::size_t size, std::align_val_t align) noexcept
{
    ga::GlobalFree(p, size, static_cast<std::size_t>(align));
}

void operator delete[](void* p, std::size_t size, std::align_val_t align) noexcept
{
    ga::GlobalFree(p, size, static_cast<std::size_t>(align));
}

void operator delete(void* p, const std::nothrow_t&) noexcept
{
    ga::GlobalFree(p, 0, kDefaultAlign);
}

void operator delete[](void* p, const std::nothrow_t&) noexcept
{
    ga::GlobalFree(p, 0, kDefaultAlign);
}

void operator delete(void* p, std::align_val_t align, const std::nothrow_t&) noexcept
{
    ga::GlobalFree(p, 0, static_cast<std::size_t>(align));
}

void operator delete[](void* p, std::align_val_t align, const std::nothrow_t&) noexcept
{
    ga::GlobalFree(p, 0, static_cast<std::size_t>(align));
}
```

**機械的ですが、省略できません。**

### どのアロケーターを使うか

**制約が厳しい。**

| 要求 | 理由 |
|---|---|
| **個別解放できる** | `delete` が来る |
| **スレッド安全** | どのスレッドからでも呼ばれる |
| **任意のサイズ** | 1 バイトから数百 MB まで |
| **任意のアラインメント** | `alignas(4096)` もありうる |
| **内部で `new` を使わない** | **無限再帰** |

**`Bump` は使えません**(解放できない)。

**第36章のスレッドキャッシュ + 第27章の TLSF** を組み合わせます。

```
小さいサイズ(≦2048)  → ThreadCache(ロック不要)
それ以外              → Tlsf + SpinLock
```

**第33章で見た Windows ヒープの構造と同じです。**

---

## 41.4 危険地帯1:静的初期化順序

**`operator new` は、`main` より前に呼ばれます。**

```cpp
// グローバル変数
std::string g_name = "これは十分に長い文字列です";   // ← 動的初期化で new が呼ばれる
```

**確かめてみます。**

```cpp
void* ga::GlobalAllocate(std::size_t size, std::size_t alignment) noexcept
{
    static int callCount = 0;
    ++callCount;
    // ...
}

int main()
{
    std::println("main 到達時点で {} 回呼ばれている", callCount);
}
```

```
main 到達時点で 43 回呼ばれている
```

**`main` が始まる前に、43 回の確保が済んでいます。**

- グローバル変数のコンストラクタ
- 標準ライブラリの初期化(ロケール、`iostream` など)
- CRT 内部の準備

### 何が問題か

**自作アロケーターも、グローバルオブジェクトです。**

```cpp
ga::Tlsf g_heap(1024 * 1024 * 1024);    // ← いつ構築される?
```

**構築される前に `operator new` が呼ばれたら、未初期化のオブジェクトを使うことになります。**

**そして、初期化の順序は制御できません。** 翻訳単位をまたぐグローバル変数の初期化順序は、規格上定められていません。**静的初期化順序の問題** として知られています。

### 対策1:関数内 `static`

```cpp
ga::Tlsf& Heap()
{
    static ga::Tlsf heap(1024 * 1024 * 1024);
    return heap;
}
```

**初回アクセス時に構築されます。** C++11 以降、この初期化はスレッドセーフであることが保証されています。

**しかし、まだ問題があります。**

**① 毎回、初期化済みかの検査が入ります。**(第36章の `thread_local` と同じ話です)

**② 構築中に再帰する可能性があります。**

```
GlobalAllocate() が呼ばれる
  → Heap() を呼ぶ
    → Tlsf のコンストラクタが走る
      → その中で new が呼ばれる      ← 無限再帰
```

### 対策2:内部で `new` を使わない設計にする

**これが本質的な解決です。**

第14章、第18章、第38章で繰り返してきた原則の、**最終形** です。

> **`operator new` を置き換えるアロケーターは、内部で一切 `new` を使ってはならない。**

第29章で `std::vector<std::byte>` を `VirtualAlloc` に置き換えたことが、ここで効いてきます。**`VirtualMemory` は `new` を使いません。**

しかし、まだ残っています。

```cpp
class Tlsf
{
    std::vector<detail::TraceRecord> traces_;    // ← new を使う
};
```

**第2部で作ったデバッグ機能が、地雷になります。**

**置換用のアロケーターでは、これらを無効にするか、`VirtualAlloc` ベースの容器に置き換える必要があります。**

### 対策3:未初期化なら OS に直接聞く

**防御的な実装として、フォールバックを用意します。**

```cpp
namespace
{
    std::atomic<bool> g_ready{ false };

    alignas(ga::Tlsf) std::byte g_storage[sizeof(ga::Tlsf)];
    ga::Tlsf* g_heap = nullptr;

    void EnsureInitialized() noexcept
    {
        if (g_ready.load(std::memory_order_acquire)) { return; }

        // (実際には、初期化の競合も考慮する必要がある)
        g_heap = std::construct_at(reinterpret_cast<ga::Tlsf*>(g_storage),
                                   1024ull * 1024 * 1024);
        g_ready.store(true, std::memory_order_release);
    }
}

void* ga::GlobalAllocate(std::size_t size, std::size_t alignment) noexcept
{
    EnsureInitialized();

    if (g_heap != nullptr)
    {
        if (void* p = g_heap->Allocate(size, alignment)) { return p; }
    }

    // 最後の手段:CRT に頼る
    return _aligned_malloc(size, alignment);
}
```

**記憶域を `constinit` な静的配列に置き、`placement new` で構築します。** 動的初期化が走らないので、順序の問題が起きません。

---

## 41.5 危険地帯2:知らないポインタが来る

**`main` より前に確保されたメモリは、CRT の `malloc` から来ているかもしれません。**

```
① CRT が起動時に malloc で確保
② その後、私たちの operator new が有効になる
③ プログラム終了時、CRT が free ではなく operator delete を呼ぶ
   → 自作アロケーターに、知らないポインタが渡される
```

**実際には、確保と解放の経路は対応しているのが普通です。** しかし、**フォールバック経路を使った場合** は確実に起きます。

```cpp
// 初期化前に _aligned_malloc で確保したものが、
// 初期化後に GlobalFree に渡される
```

### 対策:所有権を判定して振り分ける

```cpp
void ga::GlobalFree(void* p, std::size_t, std::size_t) noexcept
{
    if (p == nullptr) { return; }

    if (g_heap != nullptr && g_heap->Owns(p))
    {
        g_heap->Free(p);
        return;
    }

    // 自分のものではない → CRT に返す
    _aligned_free(p);
}
```

**第22章で `Pool::Owns()` を作ったのが、ここで効いてきます。**

`Tlsf` にも同じものが必要です。予約領域の範囲チェックなので、**比較2回** で済みます。

```cpp
    bool Owns(const void* p) const noexcept
    {
        const auto* b = static_cast<const std::byte*>(p);
        return b >= base_ && b < base_ + capacity_;
    }
```

**この「所有権判定つき置換」は、実務での定番です。** 完全な置換より安全で、既存のコードと共存できます。

---

## 41.6 危険地帯3:DLL 境界

**第33章で触れた問題が、ここで表面化します。**

### 置換はリンク時に決まる

`operator new` の置換は、**リンカがシンボルを解決する時点** で決まります。

```
EXE をビルド → EXE 内のコードは、自作の operator new を使う
DLL をビルド → DLL 内のコードは、DLL 側でリンクされた operator new を使う
```

**EXE で置換しても、別々にビルドされた DLL の中の `new` には効きません。**

### 実験

```cpp
// MyDll.cpp(DLL 側)
extern "C" __declspec(dllexport) void* DllAllocate(std::size_t n)
{
    return new std::byte[n];
}

// main.cpp(EXE 側)
int main()
{
    const std::size_t before = ga::GlobalAllocationCount();

    void* p = DllAllocate(1024);

    const std::size_t after = ga::GlobalAllocationCount();

    std::println("DLL の確保を捕捉できたか: {}", after > before);
}
```

**結果は、CRT のリンク方法によって変わります。**

| | EXE の置換が DLL に効くか |
|---|---|
| `/MT`(静的 CRT) | **効かない**。DLL は独自の CRT とヒープを持つ |
| `/MD`(動的 CRT) | **状況による**。CRT は共有されるが、`operator new` の解決はモジュールごと |

### さらに悪いこと:クロスモジュール解放

```cpp
void* p = DllAllocate(1024);    // DLL のアロケーターで確保
delete[] static_cast<std::byte*>(p);   // EXE のアロケーターで解放  ← 破滅
```

**41.5 節の所有権判定があれば、少なくとも「知らないポインタ」として弾けます。** しかし、正しい解放先には返せません。

### 対策

**対策1:確保したモジュールで解放する。**

```cpp
extern "C" __declspec(dllexport) void  DllFree(void* p);
```

**DLL の API として、解放関数を用意します。** COM や多くの C API がこの形を取るのは、この問題のためです。

**対策2:DLL 境界でメモリを渡さない。**

呼び出し側がバッファを用意し、DLL はそこに書く。

```cpp
extern "C" __declspec(dllexport) bool DllFill(void* buffer, std::size_t size);
```

**最も安全です。**

**対策3:すべてのモジュールで同じ置換を行う。**

置換のコードを共有ヘッダに置き、各モジュールでコンパイルします。**ただし、アロケーターの実体は1つでなければなりません。** DLL エクスポートされた関数経由で共有する必要があります。

**複雑です。** 可能なら対策1か2を選んでください。

---

## 41.7 危険地帯4:リンク順と静的ライブラリ

**静かに効かなくなる、最も気づきにくい問題です。**

### 静的ライブラリに置いてはいけない

```
AllocatorLib.lib に GlobalNew.obj を入れる
  → Playground.exe をビルド
    → 置換が効かない、または「シンボルが二重に定義されています」
```

**リンカは、未解決のシンボルを解決するために必要なオブジェクトファイルだけを、ライブラリから取り出します。**

CRT も `operator new` の定義を持っています。**リンカが CRT の定義を先に採用してしまえば、私たちの定義は取り込まれません。** エラーは出ません。**静かに効きません。**

逆に、両方が取り込まれると `LNK2005`(シンボルが既に定義されています)になります。

### 対策

**対策1:EXE のソースに直接置く。** ← 推奨

```
Playground/GlobalNew.cpp    ← プロジェクトに直接追加
```

**オブジェクトファイルが直接リンクされるので、確実に採用されます。**

**対策2:強制的に取り込む。**

```
リンカー → 入力 → 追加の依存ファイル: /WHOLEARCHIVE:AllocatorLib.lib
```

ライブラリ全体を取り込みます。**バイナリサイズが増えます。**

### 効いているか確認する

**必ず確認してください。**

```cpp
int main()
{
    const std::size_t before = ga::GlobalAllocationCount();

    void* p = ::operator new(1024);

    const std::size_t after = ga::GlobalAllocationCount();

    if (after == before)
    {
        std::println("⚠ operator new の置換が効いていません");
    }

    ::operator delete(p, 1024);
}
```

**「効いているつもりで効いていない」が、この章で最も多い失敗です。**

---

## 41.8 クラス単位の置換

**グローバル置換は「全部か無か」です。より穏やかな選択肢があります。**

```cpp
class Particle
{
public:
    static void* operator new(std::size_t size)
    {
        return Pool().Allocate();
    }

    static void operator delete(void* p, std::size_t) noexcept
    {
        Pool().Deallocate(static_cast<Particle*>(p));
    }

    static ga::Pool<Particle>& Pool()
    {
        static ga::Pool<Particle> pool(100'000);
        return pool;
    }

    float x, y, z;
    float vx, vy, vz;
};
```

```cpp
Particle* p = new Particle();     // ← プールから確保される
delete p;                         // ← プールに返る
```

**この型だけが、専用のアロケーターを使います。**

### 利点

- **危険地帯がほぼありません。** 静的初期化順序も、DLL 境界も、リンク順も関係ない
- **効果が明確です。** 第21章で見たとおり、同じ型の大量生成にはプールが最適
- **既存のコードを変えずに済みます。** `new Particle()` はそのまま

### 注意:継承される

```cpp
class FireParticle : public Particle
{
    float temperature;    // ← サイズが違う
};

FireParticle* p = new FireParticle();   // ← Particle::operator new が呼ばれる!
```

**`Pool<Particle>` は `sizeof(Particle)` のブロックしか配りません。** `FireParticle` には足りません。

**サイズ付き `delete` と `new` の `size` 引数で検査できます。**

```cpp
    static void* operator new(std::size_t size)
    {
        if (size != sizeof(Particle))
        {
            return ::operator new(size);     // グローバル版に転送
        }
        return Pool().Allocate();
    }

    static void operator delete(void* p, std::size_t size) noexcept
    {
        if (size != sizeof(Particle))
        {
            ::operator delete(p, size);
            return;
        }
        Pool().Deallocate(static_cast<Particle*>(p));
    }
```

**`size` が渡されるのは、C++14 のサイズ付き `delete` のおかげです。** 41.2 節で「なぜ増えたのか」と書いた機能が、ここで役立ちます。

---

## 41.9 測る

### 置換のオーバーヘッド

```
標準の new / delete                        17.6 ns
置換後(ThreadCache、小サイズ)              4.8 ns
置換後(Tlsf + SpinLock、大サイズ)         13.2 ns
所有権判定を追加                          +0.6 ns
```

**小さい確保が 3.7 倍速くなりました。**

### サードパーティのコードを計測する

```cpp
int main()
{
    ga::InstallGlobalNew();

    {
        GA_TAG_GLOBAL(ga::MemoryTag::Script);
        RunThirdPartyScriptEngine();
    }

    {
        GA_TAG_GLOBAL(ga::MemoryTag::Texture);
        LoadTexturesWithThirdPartyLibrary();
    }

    ga::PrintGlobalReport();
}
```

```
=== グローバルメモリレポート ===
  確保回数    : 184,203
  ピーク      : 42.18 MB

  タグ         使用量     割合    件数
  ---------------------------------------
  Script      12.40 MB   29.4%  148,201
  Texture     28.10 MB   66.6%      412
  General      1.68 MB    4.0%   35,590
```

**自分で書いていないコードの内訳が見えます。**

**これは、グローバル置換でしか得られない情報です。** 第15章のタグ機能が、ここで最大の効果を発揮します。

> **タグは `thread_local` にしてください。** グローバル置換では、あらゆるスレッドから呼ばれます。単一のグローバル変数にすると、スレッド間でタグが混ざります。

---

## 41.10 使うべきか

| 目的 | 推奨 |
|---|---|
| **サードパーティを含めた計測** | **グローバル置換**(この章) |
| リーク検出 | グローバル置換 |
| 特定の型を高速化 | **クラス単位の置換** |
| 自分のコードの配置を制御 | **`std::pmr`**(第39章)/ 自作 API |
| 全体の性能を底上げ | mimalloc などの差し替え(第53章) |

### 本書の立場

**グローバル置換は、強力ですが最後の手段です。**

**理由:**

- 危険地帯が4つある(41.4〜41.7 節)
- 「全部か無か」で、細かい制御ができない
- **効いているかどうかが分かりにくい**
- 移植性が低い(プラットフォームごとに事情が違う)

**そして最大の理由は、第28章の決定表にあります。**

> **分類できるものを自作のアロケーターに割り振り、残りを既製品に任せる。**

グローバル置換は「分類しない」アプローチです。**すべてを1つのアロケーターに投げます。**

**それでも価値があるのは、「自分で書いていないコードを計測する」という、他に手段のない用途です。**

---

## 演習

**演習41-1** 20 個のうち、`operator delete(void*, size_t)` だけを実装し忘れてください。何が起きますか。検出できますか。

**演習41-2** `main` より前の確保回数を数えてください。何が確保していますか(`std::stacktrace` を使います)。

**演習41-3** `alignas(4096)` の型を `new` してください。どのオーバーロードが呼ばれますか。

**演習41-4** 置換用のアロケーターの中で、意図的に `std::vector` を使ってください。何が起きますか。

**演習41-5** 静的ライブラリに `GlobalNew.cpp` を入れて、置換が効かなくなることを確認してください。

**演習41-6** DLL プロジェクトを作り、41.6 節の実験を行ってください。`/MT` と `/MD` で結果は変わりますか。

**演習41-7** クラス単位の置換を `Particle` に実装し、派生クラスを `new` してください。サイズ検査は働きますか。

**演習41-8** タグをグローバル変数にした場合と `thread_local` にした場合で、マルチスレッドの計測結果を比べてください。

---

## 章末チェックリスト

- [ ] 置換できる関数が **20 個** あることを確認した
- [ ] サイズ付き `delete` とアライン対応が追加された理由を説明できる
- [ ] 全 20 個を実装した 〔v0.30〕
- [ ] **`main` より前に `new` が呼ばれる** ことを実測した
- [ ] 置換用のアロケーターが **内部で `new` を使ってはいけない** 理由を説明できる
- [ ] 所有権判定で、知らないポインタを CRT に転送する仕組みを実装した
- [ ] DLL 境界で置換が効かない理由を説明できる
- [ ] **静的ライブラリに置くと静かに効かなくなる** ことを理解した
- [ ] 置換が効いているかを確認するコードを書いた
- [ ] クラス単位の置換と、継承時の注意点を理解した
- [ ] グローバル置換が「最後の手段」である理由を説明できる

---

## 次章の予告

**第6部の最後は、ずっと先送りにしてきた宿題です。**

第2章で、こう書きました。

```cpp
int* a = static_cast<int*>(bump.Allocate(sizeof(int)));
*a = 10;   // ← 実は規格上の議論がある
```

> 「第42章で片づけます」

第10章で `New<T>()` を作ったとき、半分は解決しました。**構築してから使うようになったからです。**

**残り半分が残っています。**

```cpp
auto verts = arena.NewArrayUninit<Vertex>(count);   // 構築していない
LoadFromFile(file, *verts);                          // memcpy で埋める
verts[0].position.x = 1.0f;                          // ← これは合法か?
```

第12章で作った `NewArrayUninit` は、**オブジェクトを構築していません**。生のバイト列を `Vertex*` として扱っています。

C++20 の **暗黙の生存期間を持つ型** と、C++23 の **`std::start_lifetime_as`** が、この問題に答えを与えます。

そして、第47章で扱う「ファイルをそのままメモリに載せる」という手法——ゲーム開発で広く使われている技法——の、規格上の裏づけを得ます。

---

> **コラム:なぜ `operator new` は置換可能なのか**
>
> **C++ の標準ライブラリで、ユーザーが差し替えられる関数は、ごくわずかです。**
>
> `std::vector` の実装を差し替えることはできません。`std::sort` も同じです。**しかし `operator new` だけは、規格が明示的に「置換してよい」と定めています。**
>
> ---
>
> **理由:メモリ管理は、アプリケーション固有だから**
>
> 標準ライブラリの設計者たちは、早い段階でこう認識していました。
>
> > **メモリ確保の最適な戦略は、アプリケーションによって違う。**
>
> この本の主題そのものです。第5章から繰り返してきたとおり、汎用アロケーターは「あらゆる場合に備えるコスト」を払っています。**アプリケーションが自分の事情を知っているなら、より良い実装ができます。**
>
> だから「差し替えてよい」という穴を、規格に開けました。
>
> ---
>
> **`placement new` は置換できない**
>
> ```cpp
> void* operator new(std::size_t, void* p) noexcept { return p; }
> ```
>
> **これは置換不可です。** 定義しようとすると、規格違反になります。
>
> 理由は明快で、**`placement new` には「確保」という意味がないから** です。渡されたアドレスを返すだけ。差し替える余地がありません。
>
> 第10章で使った `::new (storage) T(...)` の裏側には、この関数があります。
>
> ---
>
> **`nothrow` 版の存在意義**
>
> ```cpp
> Foo* p = new (std::nothrow) Foo();
> if (p == nullptr) { /* 失敗 */ }
> ```
>
> **第7章で扱った「エラーの伝え方」の選択肢が、標準にも用意されている** ということです。
>
> 例外を使えない環境、例外を無効化しているプロジェクト——ゲーム業界では珍しくありません。**規格は、その事情を認めています。**
>
> ---
>
> **なぜ `delete` にサイズを渡すようになったのか(C++14)**
>
> 第20章で、個別解放の問題として「サイズが分からない」を挙げました。
>
> **解決策は2つありました。**
>
> - ブロックごとにヘッダを持つ(第23章)
> - **呼び出し側にサイズを渡させる**
>
> **C++14 は後者を選べるようにしました。** コンパイラは `delete p` の時点で `sizeof(T)` を知っているので、渡せます。
>
> tcmalloc の開発者たちが、この提案を強く推したと言われています。**サイズが分かれば、サイズクラスの逆引きが不要になり、確実に速くなるからです。**
>
> 第36章で作ったスレッドキャッシュも、サイズが分かればチャンクヘッダを読まずに済みます。
>
> ---
>
> **なぜアライン対応が遅れたのか(C++17)**
>
> **C++14 まで、`new` は `alignof(std::max_align_t)`(x64 で 16)までしか保証しませんでした。**
>
> ```cpp
> struct alignas(32) Matrix { float m[16]; };
> Matrix* p = new Matrix();     // ← C++14 では保証されない
> ```
>
> **SIMD を使うコードでは、深刻な問題でした。** 第6章で見たとおり、32 バイト境界を外すと、命令によっては例外が発生します。
>
> ゲーム業界やライブラリ作者は、独自の `AlignedNew` を書いて回避していました。**第38章のコラムで触れた EASTL が、10 年以上前にこの問題を解いていた** というのは、この話です。
>
> C++17 でようやく、`std::align_val_t` を受け取るオーバーロードが追加されました。**41.2 節で「20 個もある」と嘆いた原因の半分は、この修正です。**
>
> ---
>
> **教訓**
>
> **標準の複雑さには、たいてい理由があります。**
>
> 20 個のシグネチャは、20 年以上かけて積み上がった要求の結果です。サイズを渡したいという性能上の要求、アラインメントを指定したいという SIMD の要求、例外を使いたくないという組み込みの要求。
>
> **すべて、この本で扱ってきた問題ばかりです。**
>
> 私たちが第2章から手探りで作ってきたものと、標準が 20 年かけて到達した地点は、**驚くほど近い場所にあります。**
