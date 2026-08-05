# 第14章 誰が確保したのか記録する 〔v0.9〕

---

## この章のゴール

第2部が始まります。ここからしばらく、`Bump` の **中で何が起きているかを見えるようにする** 作業が続きます。

まずは、いちばん基本的な情報から。**その確保は、どこから呼ばれたのか。**

```
[alloc] main.cpp:42  UpdateParticles()  size=320000 align=4  -> 0x1f3a8c2b3c0
[alloc] main.cpp:58  BuildDrawList()    size=64     align=16 -> 0x1f3a8c7a180
```

この章で扱うこと。

- C++20 の `std::source_location` を使う
- **既定引数が呼び出し側で評価される** という仕組みを利用する 〔**v0.9**〕
- パラメータパックとの相性問題に突き当たり、解決する
- ログのコールバックと、直近の確保を覚えるリングバッファを作る
- コストを測る(速度と、バイナリサイズの両方)
- Release でコンパイル時に消す方法を用意する

---

## 14.1 いま何が見えていないか

第7章で、溢れを検出できるようにしました。しかし、返ってくるのはこれだけです。

```
確保失敗: OutOfMemory
```

**どこで失敗したのか分かりません。**

デバッガでブレークすれば呼び出し元は分かります。しかし、次のような状況では役に立ちません。

- 「板が思ったより早く埋まる。誰が大量に確保しているのか」
- 「使用量のピークが 3 MB もある。内訳は?」
- 「バグ報告のログだけが手元にある。再現しない」

必要なのは、**確保のたびに記録を残す** 仕組みです。

---

## 14.2 `std::source_location`

C++20 で追加された型です。ソースコード上の位置を表します。

```cpp
#include <source_location>

void Report(const std::source_location& loc = std::source_location::current())
{
    std::println("{}:{}  関数 {}",
                 loc.file_name(),
                 loc.line(),
                 loc.function_name());
}

int main()
{
    Report();
}
```

```
C:\dev\GrowingAllocator\Playground\main.cpp:12  関数 int __cdecl main(void)
```

| メンバ | 内容 |
|---|---|
| `file_name()` | ファイルのパス |
| `line()` | 行番号 |
| `column()` | 桁位置 |
| `function_name()` | 関数のシグネチャ |

### 既定引数が呼び出し側で評価される

この仕組みの肝は、`std::source_location::current()` を **既定引数** に置くことです。

C++ の既定引数は、**関数の定義された場所ではなく、呼び出された場所で評価されます**。`current()` は「自分が書かれた位置」を返すので、結果的に **呼び出し元の位置** が入ります。

呼び出し側は何も書きません。

```cpp
Report();     // ← 引数なし。それでも位置が分かる
```

これは従来 `__FILE__` と `__LINE__` をマクロで渡すしかなかった処理を、**マクロなしで** 実現するものです。

### 落とし穴:転送すると位置が変わる

```cpp
void Inner(const std::source_location& loc = std::source_location::current())
{
    std::println("{}:{}", loc.file_name(), loc.line());
}

void Outer()
{
    Inner();      // ← ここが記録される
}

int main()
{
    Outer();      // ← ここではない
}
```

`Outer()` が `Inner()` を呼んだ位置が記録されます。`main()` の位置は失われます。

**中継する関数は、位置を引数として引き継がなければなりません。**

```cpp
void Outer(const std::source_location& loc = std::source_location::current())
{
    Inner(loc);   // ← 受け取ったものを渡す
}
```

この性質が、14.4 節で問題になります。

---

## 14.3 `Allocate()` に組み込む

`Allocate()` には既に既定引数(`alignment`)があります。その後ろに足します。

```cpp
    [[nodiscard]]
    AllocResult Allocate(std::size_t size,
                         std::size_t alignment = kDefaultAlignment,
                         const std::source_location& loc = std::source_location::current()) noexcept
    {
        // ... 既存の処理 ...

        offset_   = alignedOffset + size;
        padding_ += padding;

        void* result = reinterpret_cast<void*>(aligned);

        RecordAllocation(result, size, alignment, padding, loc);   // ← 追加

        return result;
    }
```

**呼び出し側のコードは1文字も変わりません。**

```cpp
auto r = bump.Allocate(100);              // 従来どおり
auto r = bump.Allocate(100, 16);          // 従来どおり
```

これが既定引数を使う最大の利点です。既存のコードを壊さずに、情報を追加できます。

### 記録する内容

```cpp
namespace ga
{
    struct AllocationInfo
    {
        const void*          address   = nullptr;
        std::size_t          size      = 0;
        std::size_t          alignment = 0;
        std::size_t          padding   = 0;
        std::source_location location{};
    };
}
```

アドレス、要求サイズ、アラインメント、捨てたパディング、そして位置。デバッグに必要な最小限です。

---

## 14.4 パラメータパックの壁

`New<T>()` にも同じことをしたい——ところが、ここで壁に当たります。

```cpp
    template <class T, class... Args>
    auto New(Args&&... args,
             const std::source_location& loc = std::source_location::current());
    //       ^^^^^^^^^^^^^^^^^^^^^^^^^ ← パラメータパックの後ろには置けない
```

**可変長引数の後ろに、通常の引数は書けません。** 書いても、常にパックのほうが引数を食べてしまいます。

そして、位置を渡さずに `New()` から `Allocate()` を呼ぶと、記録されるのは **`Bump.h` の中の行番号** になります。

```
[alloc] ga/Bump.h:187  size=32 align=8
```

**まったく役に立ちません。** 全部の確保が同じ行を指します。

### 解決策を3つ検討する

**案1:位置を第1引数にする。**

```cpp
    template <class T, class... Args>
    auto NewAt(const std::source_location& loc, Args&&... args);
```

パックの前なら置けます。しかし、呼び出し側が毎回書くことになります。

```cpp
bump.NewAt<Enemy>(std::source_location::current(), 100, "goblin");   // 長い
```

**案2:マクロで包む。**

案1に、短く書くための皮を被せます。

```cpp
#define GA_NEW(arena, Type, ...) \
    (arena).NewAt<Type>(std::source_location::current() __VA_OPT__(,) __VA_ARGS__)
```

```cpp
auto e = GA_NEW(bump, Enemy, 100, "goblin");
auto v = GA_NEW(bump, Vec3);                  // 引数ゼロでも書ける
```

`__VA_OPT__`(C++20)は、可変長引数が空のときにカンマを消してくれます。これがないと、引数なしの呼び出しで余分なカンマが残ります。

**案3:アロケーター側に「現在位置」を持たせる。**

スコープガードで、いまどの処理をしているかを登録しておく方式です。

```cpp
{
    GA_SCOPE(bump, "パーティクル更新");
    // このスコープ内の確保はすべて「パーティクル更新」として記録される
}
```

正確な行番号は失われますが、**意味のある単位でまとめられます**。実際のゲームエンジンでは、こちらのほうがよく使われます。

### 本書の選択

**案1と案2を採用します。** 案3は次章(第15章)でタグ機能として扱います。

マクロを使うのは、第13章で「マクロをほとんど使わない」と書いた方針に反するように見えるかもしれません。しかし、**`__FILE__` / `__LINE__` の時代からずっと、この用途だけはマクロが必要でした**。`std::source_location` は「マクロを引数に埋め込む必要」をなくしましたが、「呼び出し位置を可変長引数と一緒に運ぶ」問題までは解決していません。

マクロには接頭辞 `GA_` を付け、ヘッダで定義することの影響を最小限にします。

---

## 14.5 ログを出す

記録した情報を、どこへ流すか。`Bump` の中で `std::println` を呼ぶのは避けます。**ライブラリが勝手に標準出力を汚すべきではありません。**

コールバックを登録できるようにします。

```cpp
    using LogCallback = void (*)(const AllocationInfo&, void* user) noexcept;

    void SetLogCallback(LogCallback cb, void* user = nullptr) noexcept
    {
        logCallback_ = cb;
        logUser_     = user;
    }
```

`std::function` ではなく、**関数ポインタ + ユーザーデータ** という C 風の形にしています。理由は、`std::function` が内部でメモリを確保する可能性があるからです。**アロケーターの中でメモリを確保するのは避けたい。**

### 使う

```cpp
void PrintAllocation(const ga::AllocationInfo& info, void*) noexcept
{
    // フルパスは長いので、ファイル名だけ取り出す
    std::string_view file = info.location.file_name();
    if (auto pos = file.find_last_of("\\/"); pos != std::string_view::npos)
    {
        file = file.substr(pos + 1);
    }

    std::println("[alloc] {}:{}  size={} align={} padding={} -> {}",
                 file, info.location.line(),
                 info.size, info.alignment, info.padding,
                 info.address);
}

int main()
{
    ga::Bump bump(4096);
    bump.SetLogCallback(&PrintAllocation);

    auto a = bump.Allocate(100);
    auto b = bump.Allocate(64, 32);
    auto c = GA_NEW(bump, Vec3);
    auto d = bump.NewArrayAt<int>(std::source_location::current(), 10);
}
```

```
[alloc] main.cpp:22  size=100 align=16 padding=0 -> 0x1f3a8c2b3c0
[alloc] main.cpp:23  size=64 align=32 padding=28 -> 0x1f3a8c2b440
[alloc] main.cpp:24  size=12 align=4 padding=0 -> 0x1f3a8c2b480
[alloc] main.cpp:25  size=40 align=4 padding=0 -> 0x1f3a8c2b48c
```

**どの行が何バイト確保したかが、そのまま見えます。**

2行目の `padding=28` にも注目してください。32 バイト境界を要求したせいで、28 バイト捨てています。第6章で作った `Padding()` は合計しか分かりませんでしたが、これで **どの確保が無駄を出しているか** が特定できます。

---

## 14.6 直近の確保を覚えておく

ログはリアルタイムで流れていきます。問題が起きた **後から** 調べたいこともあります。

固定長のリングバッファに、直近の確保を残しておきます。

```cpp
    static constexpr std::size_t kRecentCapacity = 16;

    // 直近の確保(新しい順)
    void ForEachRecent(auto&& fn) const
    {
        const std::size_t n = (allocCount_ < kRecentCapacity) ? allocCount_ : kRecentCapacity;

        for (std::size_t i = 0; i < n; ++i)
        {
            // recentHead_ の1つ手前が最新
            const std::size_t idx = (recentHead_ + kRecentCapacity - 1 - i) % kRecentCapacity;
            fn(recent_[idx]);
        }
    }

    std::size_t AllocationCount() const noexcept { return allocCount_; }
```

記録側です。

```cpp
    void RecordAllocation(const void* address, std::size_t size, std::size_t alignment,
                          std::size_t padding, const std::source_location& loc) noexcept
    {
        ++allocCount_;

        const AllocationInfo info{ address, size, alignment, padding, loc };

        recent_[recentHead_] = info;
        recentHead_ = (recentHead_ + 1) % kRecentCapacity;

        if (logCallback_)
        {
            logCallback_(info, logUser_);
        }
    }
```

固定長の配列なので、**メモリを確保しません**。アロケーターのデバッグ機能が別のメモリ確保に依存する、という気持ちの悪い構造を避けられます。

### 溢れたときに吐き出す

```cpp
int main()
{
    ga::Bump bump(256);

    for (int i = 0; i < 100; ++i)
    {
        auto r = bump.Allocate(16);

        if (!r)
        {
            std::println("=== 確保失敗({})。直近の確保 ===", ToString(r.error()));

            bump.ForEachRecent([](const ga::AllocationInfo& info) {
                std::println("  {}:{}  size={}",
                             info.location.file_name(), info.location.line(), info.size);
            });

            std::println("総確保回数: {}  使用量: {}/{}",
                         bump.AllocationCount(), bump.Used(), bump.Capacity());
            break;
        }
    }
}
```

```
=== 確保失敗(OutOfMemory)。直近の確保 ===
  ...\main.cpp:8  size=16
  ...\main.cpp:8  size=16
  ...(16件)
総確保回数: 16  使用量: 256/256
```

**失敗した瞬間の状況が、そのまま残ります。**

実際のゲームでは、これをクラッシュレポートに含めます。「再現しないバグ」に対する数少ない手がかりになります。

---

## 14.7 コストを測る

デバッグ機能は、必ずコストを伴います。正直に測ります。

### 速度

```
v0.8 (記録なし)              median=      2.1  p95=      2.2  max=        3.0
v0.9 (記録あり・ログなし)    median=      3.4  p95=      3.6  max=        4.5
v0.9 (記録あり・ログあり)    median=  18500.0  p95=  19200.0  max=    41000.0
```

**記録だけなら 1.3 ns の増加** です。`AllocationInfo`(40 バイト程度)をリングバッファにコピーするコストです。

**ログのコールバックを付けると、5000 倍以上** になります。`std::println` が遅いためで、アロケーターのせいではありません。**ログは常時有効にするものではない** ということです。

### バイナリサイズ

`std::source_location` は、ファイル名と関数名の文字列をバイナリに埋め込みます。

| | 実行ファイルのサイズ |
|---|---|
| v0.8 | 41 KB |
| v0.9 | 48 KB |

7 KB 増えました。呼び出し箇所ごとに、パスと関数シグネチャの文字列が入るためです。

関数名には注意が必要です。テンプレート関数の場合、`function_name()` は展開後の完全なシグネチャを返します。

```
class std::expected<struct Vec3 *,enum ga::AllocError> __cdecl ga::Bump::NewAt<struct Vec3>(...)
```

**1つで 100 バイトを超えます。** 確保箇所が数千あるプロジェクトでは、無視できない量になります。

---

## 14.8 コンパイル時に消す

Release では、この機能を丸ごと消したい。マクロで切り替えます。

```cpp
// ga/Core.h
#ifndef GA_ENABLE_ALLOC_TRACKING
#  ifdef NDEBUG
#    define GA_ENABLE_ALLOC_TRACKING 0
#  else
#    define GA_ENABLE_ALLOC_TRACKING 1
#  endif
#endif
```

```cpp
    void RecordAllocation(const void* address, std::size_t size, std::size_t alignment,
                          std::size_t padding, const std::source_location& loc) noexcept
    {
#if GA_ENABLE_ALLOC_TRACKING
        ++allocCount_;
        // ... 記録処理 ...
#else
        (void)address; (void)size; (void)alignment; (void)padding; (void)loc;
#endif
    }
```

### ⚠ メンバは消さない

第9章で `Marker` について書いた注意が、ここでも当てはまります。

```cpp
#if GA_ENABLE_ALLOC_TRACKING
    std::array<AllocationInfo, kRecentCapacity> recent_;   // ← これはやらない
#endif
```

**構成によってクラスのサイズが変わると、ODR 違反の温床になります。** Debug でビルドしたライブラリと Release でビルドしたコードを混ぜた瞬間、静かに壊れます。

メンバは常に持ち、**コードだけを消します**。640 バイトほど無駄になりますが、安全と引き換えなら安いものです。

> **本当に消したい場合は、テンプレートパラメータにするのが正解です。**
> ```cpp
> template <bool EnableTracking> class BasicBump { ... };
> using Bump      = BasicBump<GA_ENABLE_ALLOC_TRACKING>;
> ```
> 型が別物になるので、混ぜてもリンクエラーで気づけます。第51章でこの方式に移行します。

### 位置情報そのものは消えない

注意点があります。`GA_ENABLE_ALLOC_TRACKING` を 0 にしても、`std::source_location::current()` は **呼び出し側で評価されます**。文字列はバイナリに残る可能性があります。

完全に消すには、マクロ側で切り替える必要があります。

```cpp
#if GA_ENABLE_ALLOC_TRACKING
#  define GA_NEW(arena, Type, ...) \
       (arena).NewAt<Type>(std::source_location::current() __VA_OPT__(,) __VA_ARGS__)
#else
#  define GA_NEW(arena, Type, ...) \
       (arena).New<Type>(__VA_ARGS__)
#endif
```

---

## 14.9 この章の完成コード

`ga/AllocationInfo.h`(新規):

```cpp
#pragma once

#include <cstddef>
#include <source_location>

namespace ga
{
    struct AllocationInfo
    {
        const void*          address   = nullptr;
        std::size_t          size      = 0;
        std::size_t          alignment = 0;
        std::size_t          padding   = 0;
        std::source_location location{};
    };
}
```

`ga/Bump.h` の差分:

```cpp
namespace ga
{
    class Bump
    {
    public:
        using LogCallback = void (*)(const AllocationInfo&, void* user) noexcept;

        static constexpr std::size_t kRecentCapacity = 16;

        // --- v0.9:確保元の記録 ---

        [[nodiscard]]
        AllocResult Allocate(std::size_t size,
                             std::size_t alignment = kDefaultAlignment,
                             const std::source_location& loc = std::source_location::current()) noexcept
        {
            // ... 検査は第7章のまま ...

            offset_   = alignedOffset + size;
            padding_ += padding;

            void* result = reinterpret_cast<void*>(aligned);
            RecordAllocation(result, size, alignment, padding, loc);
            return result;
        }

        template <class T, class... Args>
        [[nodiscard]]
        std::expected<T*, AllocError> NewAt(const std::source_location& loc, Args&&... args)
        {
            auto storage = Allocate(sizeof(T), alignof(T), loc);
            // ... 第11章のまま ...
        }

        template <class T, class... Args>
        [[nodiscard]]
        std::expected<T*, AllocError> New(Args&&... args)
        {
            return NewAt<T>(std::source_location::current(), std::forward<Args>(args)...);
        }

        void SetLogCallback(LogCallback cb, void* user = nullptr) noexcept
        {
            logCallback_ = cb;
            logUser_     = user;
        }

        std::size_t AllocationCount() const noexcept { return allocCount_; }

        void ForEachRecent(auto&& fn) const
        {
            const std::size_t n = (allocCount_ < kRecentCapacity) ? allocCount_ : kRecentCapacity;

            for (std::size_t i = 0; i < n; ++i)
            {
                const std::size_t idx = (recentHead_ + kRecentCapacity - 1 - i) % kRecentCapacity;
                fn(recent_[idx]);
            }
        }

    private:
        void RecordAllocation(const void* address, std::size_t size, std::size_t alignment,
                              std::size_t padding, const std::source_location& loc) noexcept
        {
#if GA_ENABLE_ALLOC_TRACKING
            ++allocCount_;

            const AllocationInfo info{ address, size, alignment, padding, loc };

            recent_[recentHead_] = info;
            recentHead_ = (recentHead_ + 1) % kRecentCapacity;

            if (logCallback_) { logCallback_(info, logUser_); }
#else
            (void)address; (void)size; (void)alignment; (void)padding; (void)loc;
#endif
        }

        // ... 既存のメンバ ...
        std::array<AllocationInfo, kRecentCapacity> recent_{};
        std::size_t  recentHead_  = 0;
        std::size_t  allocCount_  = 0;
        LogCallback  logCallback_ = nullptr;
        void*        logUser_     = nullptr;
    };
}
```

マクロ:

```cpp
// ga/Core.h
#define GA_NEW(arena, Type, ...) \
    (arena).NewAt<Type>(std::source_location::current() __VA_OPT__(,) __VA_ARGS__)

#define GA_NEW_ARRAY(arena, Type, count) \
    (arena).NewArrayAt<Type>(std::source_location::current(), (count))
```

---

## 演習

**演習14-1** `New<T>()` を `NewAt<T>()` に転送するとき、`std::source_location::current()` は `New` の中の行を指します。それでも `New` を残す意味はありますか。

**演習14-2** `GA_NEW` を使わずに `bump.New<Vec3>()` を呼び、ログを見てください。どのファイルの何行目が表示されますか。

**演習14-3** `__VA_OPT__` を使わずに `GA_NEW` を定義しようとすると、引数ゼロの場合に何が起きますか。

**演習14-4** リングバッファの容量を 16 から 256 に増やしてください。`sizeof(ga::Bump)` はどれくらい増えますか。

**演習14-5** ログのコールバックを、`std::println` ではなくファイルへの書き出しに変えてください。速度はどう変わりますか。

**演習14-6** 確保サイズの大きい順に上位10件を保持する機能を追加してください。リングバッファとどちらが有用でしょうか。

**演習14-7** `function_name()` を出力に含めて、実行ファイルのサイズがどれだけ増えるか測ってください。テンプレートを多用した場合はどうですか。

---

## 章末チェックリスト

- [ ] `std::source_location` の4つのメンバを使った
- [ ] 既定引数が **呼び出し側で** 評価される仕組みを説明できる
- [ ] 中継する関数では位置を明示的に引き継ぐ必要があることを理解した
- [ ] パラメータパックの後ろに既定引数を置けない理由を説明できる
- [ ] `NewAt` + `GA_NEW` マクロを実装した 〔v0.9〕
- [ ] ログのコールバックを登録し、確保元を表示した
- [ ] リングバッファで直近の確保を保持した
- [ ] 記録のコスト(速度・バイナリサイズ)を測った
- [ ] コンパイル時に消す際、**メンバは消さない** 理由を説明できる

---

## 次章の予告

「どこから確保されたか」が分かるようになりました。次に知りたいのは **「全体でどうなっているか」** です。

第15章では統計を取ります。確保回数、平均サイズ、サイズの分布、そしてピーク使用量。さらに、**タグ(カテゴリ)別の集計** を導入します。

```
=== メモリ内訳 ===
  Mesh      :  4.2 MB (38%)
  Texture   :  3.1 MB (28%)
  Audio     :  1.8 MB (16%)
  Script    :  1.2 MB (11%)
  その他    :  0.7 MB ( 7%)
```

この形の情報こそ、実際のゲーム開発で最も見られているものです。そして第14章で見送った「案3:スコープでまとめる」方式が、ここで登場します。

---

> **コラム:`__FILE__` と `__LINE__` からの脱出**
>
> 呼び出し位置を記録したいという要求は、C の時代からありました。標準的な手段は、プリプロセッサが定義するマクロです。
>
> ```c
> #define TRACE(msg) printf("%s:%d: %s\n", __FILE__, __LINE__, msg)
> ```
>
> `__FILE__` と `__LINE__` は、**プリプロセッサが展開する場所** で置き換わります。だからマクロの中に書けば、呼び出し側の位置になります。関数の中に書いてもその関数の位置にしかならないので、**マクロにするしかありませんでした**。
>
> これが長年、C++ プログラマを悩ませてきました。
>
> **マクロは名前空間を無視します。** `TRACE` という名前を定義したら、プロジェクト全体で `TRACE` が使えなくなります。
>
> **マクロは型を見ません。** 引数を2回評価する、演算子の優先順位で壊れる、`if` の中で波括弧なしに使うと崩れる。有名な落とし穴が山ほどあります。
>
> **マクロはデバッガから見えません。** ステップ実行しても、展開後のコードが見えるだけです。
>
> それでも、代替手段がありませんでした。ログ、アサート、メモリ確保の追跡——**呼び出し位置が必要なものは、すべてマクロで書くしかなかった** のです。
>
> ---
>
> `std::source_location`(C++20)は、この状況を大きく変えました。
>
> 既定引数が呼び出し側で評価されるという、**言語に元からあった性質** を利用しています。新しい魔法ではありません。`current()` がコンパイラの組み込み機能として位置を返せるようにしただけで、あとは既存の仕組みが働きます。
>
> おかげで、ログ関数もアサート関数も、**普通の関数として書けるようになりました**。名前空間に入れられます。オーバーロードできます。テンプレートにできます。デバッガでステップインできます。
>
> ---
>
> ただし、14.4 節で見たとおり、**マクロが完全に不要になったわけではありません**。
>
> 可変長引数と組み合わせると、既定引数を置く場所がありません。この制約は言語の文法から来ているので、ライブラリ側の工夫では回避できません。
>
> C++ の進化は、こうした「9割は解決したが1割が残る」という形をよく取ります。`std::span` が配列の長さ問題をほぼ解決しつつ、C の API との境界では依然ポインタと個数を扱う必要があるのと同じです。
>
> 残った1割にマクロを使うことは、敗北ではありません。**マクロの使用箇所を、1割に閉じ込められたこと** が勝利です。

