# 第39章 `std::pmr` に乗る 〔v0.28〕

---

## この章のゴール

第38章の最後に残った問題は、これでした。

```cpp
std::vector<int>                          // 標準
std::vector<int, ga::BumpAllocator<int>>  // 自作。別の型
```

**型が違うので、関数の引数として渡し合えません。**

C++17 の `std::pmr` は、**アロケーターを「型」ではなく「値」にする** ことで、これを解決します。

```cpp
std::pmr::vector<int> a(&arenaResource);
std::pmr::vector<int> b(&anotherResource);

void Process(const std::pmr::vector<int>& v);   // どちらも渡せる
```

- `memory_resource` のインターフェイスと、NVI イディオム
- `Bump` と `Tlsf` を `memory_resource` として実装する 〔**v0.28**〕
- 標準が用意しているリソース——とくに **`monotonic_buffer_resource`**(私たちの `Bump` の親戚)
- **仮想呼び出しのコスト** を実測する(第7章の主張が、ここで崩れます)
- 入れ子のコンテナに、リソースが自動で伝播する仕組み

---

## 39.1 型ではなく値にする

### 仕組み

```cpp
namespace std::pmr
{
    template <class T>
    class polymorphic_allocator
    {
        memory_resource* resource_;    // ← 実体はこれだけ
    public:
        T* allocate(std::size_t n)
        {
            return static_cast<T*>(
                resource_->allocate(n * sizeof(T), alignof(T)));
        }
        // ...
    };

    template <class T>
    using vector = std::vector<T, polymorphic_allocator<T>>;
}
```

**`polymorphic_allocator<T>` は、`memory_resource*` を1本持つだけです。**

型パラメータは `T` だけなので、`std::pmr::vector<int>` は **常に同じ型** になります。どのリソースを使っていても。

**振る舞いの違いは、仮想関数で切り替えます。**

```
第38章:型で切り替える(コンパイル時)
第39章:値で切り替える(実行時)
```

**古典的な、テンプレート対仮想関数のトレードオフです。**

---

## 39.2 `memory_resource` のインターフェイス

```cpp
class memory_resource
{
public:
    virtual ~memory_resource();

    // 公開されるのは、この3つ(仮想ではない)
    void* allocate(std::size_t bytes, std::size_t alignment = alignof(std::max_align_t));
    void  deallocate(void* p, std::size_t bytes, std::size_t alignment = alignof(std::max_align_t));
    bool  is_equal(const memory_resource& other) const noexcept;

private:
    // 派生クラスが実装するのは、こちら
    virtual void* do_allocate(std::size_t bytes, std::size_t alignment) = 0;
    virtual void  do_deallocate(void* p, std::size_t bytes, std::size_t alignment) = 0;
    virtual bool  do_is_equal(const memory_resource& other) const noexcept = 0;
};
```

### なぜ `do_` を付けて分けているのか

**NVI(Non-Virtual Interface)イディオム** と呼ばれる設計です。

- **公開する関数は非仮想。** 引数の検査、事前・事後条件、ログなどを、基底クラスが一元的に行える
- **仮想関数は非公開。** 派生クラスは「中身の実装」だけを担当する

「呼び出し方は基底が決め、やり方は派生が決める」という分担です。

**私たちのアロケーターでは、公開関数を差し替えられません。** その代わり、引数の扱いが統一されます。

### `is_equal` の重要性

第38章の `operator==` に相当します。

```cpp
bool do_is_equal(const memory_resource& other) const noexcept override
{
    return this == &other;
}
```

**2つのリソースが「同じ」なら、一方が確保したメモリを他方が解放してよい** という意味です。

`this == &other` は、最も保守的で安全な実装です。「同じインスタンスのときだけ等しい」。

**より緩い実装も可能です。** たとえば「同じアリーナを指しているなら等しい」。ただし、`dynamic_cast` が必要になります。

```cpp
bool do_is_equal(const memory_resource& other) const noexcept override
{
    const auto* p = dynamic_cast<const BumpResource*>(&other);
    return p != nullptr && p->arena_ == arena_;
}
```

**本書では、単純な `this` 比較を採用します。**

---

## 39.3 実装する 〔v0.28〕

```cpp
// ga/MemoryResource.h
#pragma once

#include "ga/Bump.h"
#include "ga/Tlsf.h"

#include <memory_resource>
#include <new>

namespace ga
{
    // --- Bump を memory_resource として公開する ---
    class BumpResource final : public std::pmr::memory_resource
    {
    public:
        explicit BumpResource(Bump& arena) noexcept
            : arena_(&arena)
        {
        }

        Bump& Arena() const noexcept { return *arena_; }

    private:
        void* do_allocate(std::size_t bytes, std::size_t alignment) override
        {
            auto r = arena_->Allocate(bytes, alignment);

            if (!r) { throw std::bad_alloc{}; }

            return *r;
        }

        void do_deallocate(void*, std::size_t, std::size_t) override
        {
            // Bump は個別解放しない
        }

        bool do_is_equal(const std::pmr::memory_resource& other) const noexcept override
        {
            return this == &other;
        }

        Bump* arena_;
    };

    // --- Tlsf を memory_resource として公開する ---
    class TlsfResource final : public std::pmr::memory_resource
    {
    public:
        explicit TlsfResource(Tlsf& heap) noexcept
            : heap_(&heap)
        {
        }

    private:
        void* do_allocate(std::size_t bytes, std::size_t alignment) override
        {
            if (alignment > kDefaultAlignment)
            {
                // 第23章の実装は 16 バイト超のアラインメントに未対応
                throw std::bad_alloc{};
            }

            void* p = heap_->Allocate(bytes);
            if (p == nullptr) { throw std::bad_alloc{}; }

            return p;
        }

        void do_deallocate(void* p, std::size_t, std::size_t) override
        {
            heap_->Free(p);
        }

        bool do_is_equal(const std::pmr::memory_resource& other) const noexcept override
        {
            return this == &other;
        }

        Tlsf* heap_;
    };
}
```

**20 行ほどです。** 第38章のアダプタ(リバインド、伝播の型、桁溢れ検査)に比べて、はるかに単純です。

### 単純になった理由

| | 第38章のテンプレート版 | この章の pmr 版 |
|---|---|---|
| リバインド | 必要 | **不要**(バイト列を扱うだけ) |
| `propagate_on_*` | 4種類を検討 | **不要**(既定で足りる) |
| 桁溢れの検査 | 自分で書く | **`polymorphic_allocator` がやる** |
| 型ごとのインスタンス化 | `T` ごとに1つ | **1つだけ** |

**`memory_resource` は「バイト列を配る」ことだけに責任を持ちます。** 型のことは `polymorphic_allocator` が引き受けます。

**第10章のコラムで見た「確保と構築の分離」が、ここでも効いています。**

---

## 39.4 使ってみる

```cpp
int main()
{
    ga::Bump arena(16 * 1024 * 1024);
    ga::BumpResource resource(arena);

    // --- vector ---
    std::pmr::vector<int> v(&resource);
    for (int i = 0; i < 10; ++i) { v.push_back(i); }

    // --- string ---
    std::pmr::string s("これはアリーナ上の文字列です", &resource);

    // --- map ---
    std::pmr::map<int, int> m(&resource);
    for (int i = 0; i < 100; ++i) { m[i] = i * 2; }

    std::println("アリーナ使用量: {}", ga::FormatBytes(arena.Used()));
}
```

**型名が短くなりました。** 第38章の `std::map<int, int, std::less<int>, ga::BumpAllocator<std::pair<const int, int>>>` と比べてください。

### 型が同じであることを確認する

```cpp
void Process(const std::pmr::vector<int>& v)
{
    std::println("要素数: {}", v.size());
}

int main()
{
    ga::Bump arenaA(1024 * 1024);
    ga::Bump arenaB(1024 * 1024);

    ga::BumpResource resA(arenaA);
    ga::BumpResource resB(arenaB);

    std::pmr::vector<int> a(&resA);
    std::pmr::vector<int> b(&resB);
    std::pmr::vector<int> c;            // 既定のリソース(new/delete)

    Process(a);
    Process(b);
    Process(c);        // ← 3つとも同じ関数に渡せる
}
```

**これが `std::pmr` の存在理由です。**

### 既定のリソース

```cpp
std::pmr::get_default_resource();          // 既定は new_delete_resource()
std::pmr::set_default_resource(&resource); // 差し替える
std::pmr::new_delete_resource();           // new/delete を使う
std::pmr::null_memory_resource();          // 常に bad_alloc を投げる
```

**`set_default_resource` は、プロセス全体に影響します。** すべてのスレッドの、すべての `std::pmr` コンテナに効きます。

**気軽に呼ぶべきではありません。** 「ライブラリの中で勝手に差し替える」といったことをすると、利用者を混乱させます。

---

## 39.5 標準が用意しているリソース

**標準ライブラリには、すでに3つのリソースが入っています。**

### `monotonic_buffer_resource` ← 私たちの `Bump` の親戚

第8章のコラムで予告したものです。

```cpp
std::pmr::monotonic_buffer_resource res;

std::pmr::vector<int> v(&res);
for (int i = 0; i < 1000; ++i) { v.push_back(i); }

res.release();      // ← まとめて解放(私たちの Reset() に相当)
```

**動作は `Bump` とほぼ同じです。**

- 前へ進むだけ
- `deallocate` は何もしない
- `release()` で一括解放

**"monotonic"(単調)という名前が、ポインタが一方向にしか進まないことを表しています。**

**スタック上のバッファを渡せます。**

```cpp
std::byte buffer[4096];
std::pmr::monotonic_buffer_resource res(buffer, sizeof(buffer));

std::pmr::vector<int> v(&res);      // ヒープを一切触らずに動く
```

**足りなくなったら、上流のリソースからもらいます。** 既定では `new/delete` です。

### ヒープを絶対に触らせない

```cpp
std::byte buffer[4096];
std::pmr::monotonic_buffer_resource res(buffer, sizeof(buffer),
                                        std::pmr::null_memory_resource());

std::pmr::vector<int> v(&res);
// 4096 バイトを超えると std::bad_alloc
```

**`null_memory_resource()` を上流に指定すると、溢れた瞬間に例外になります。**

**これは強力な定番パターンです。** 「この関数は絶対にヒープ確保をしない」ことを、**コードで保証できます。**

リアルタイム処理や、割り込みハンドラに近い場所で使えます。

### `unsynchronized_pool_resource` ← 私たちの `Pool` の親戚

**サイズクラス別のプールです。** 第21章と第25章を組み合わせたものと考えてください。

```cpp
std::pmr::pool_options opt;
opt.max_blocks_per_chunk        = 1024;
opt.largest_required_pool_block = 2048;

std::pmr::unsynchronized_pool_resource pool(opt);

std::pmr::list<int> list(&pool);     // ノードがプールから確保される
```

**個別解放ができます。** `monotonic` と違い、`deallocate` が実際に返却します。

`synchronized_pool_resource` は、これのスレッド安全版です(内部でロックを使います)。

### 上流を指定して組み合わせる

```cpp
ga::Bump arena(64 * 1024 * 1024);
ga::BumpResource arenaRes(arena);

// プールが足りなくなったら、私たちのアリーナからもらう
std::pmr::unsynchronized_pool_resource pool(&arenaRes);

std::pmr::list<int> list(&pool);
```

**リソースを積み重ねられます。** `std::pmr` の設計で、最も気の利いた部分です。

---

## 39.6 コストを測る

**第7章で、こう書きました。**

> 測定結果には現れませんでした。`Allocate` がクラス定義の中に書かれていて **インライン展開される** ためです。
> **ただし、これが崩れる場面があります。** 第45章で、アロケーターを仮想関数のインターフェイス越しに使えるようにします。

**その場面が、予定より早く来ました。**

### 確保1回のコスト

```
ga::Bump::Allocate()(直接)                    2.1 ns
ga::BumpAllocator<T>(第38章、テンプレート)      2.4 ns
ga::BumpResource(この章、pmr)                  5.1 ns
std::pmr::monotonic_buffer_resource            4.6 ns
::operator new                                17.6 ns
```

**pmr は、テンプレート版の 2.1 倍です。**

### 2.7 ns の内訳

**① 仮想呼び出し。** 仮想関数テーブルを引き、間接ジャンプします。分岐予測が効けば速いのですが、インライン展開は不可能です。

**② インライン展開されないこと。** これが大きい。`Bump::Allocate` の中身(比較、加算、アラインメント計算)が、すべて実際の関数呼び出しになります。引数の受け渡しも発生します。

**③ 引数が増える。** `do_allocate(bytes, alignment)` は、常に両方を渡します。テンプレート版では `alignof(T)` がコンパイル時定数でした。

### 実際のワークロードでは

```
                                          構築(100万要素)
std::list<int>(new)                            42.1 ms
std::list<int, ga::BumpAllocator<int>>         11.4 ms
std::pmr::list<int>(BumpResource)              14.2 ms
std::pmr::list<int>(monotonic_buffer_resource) 13.1 ms
```

**pmr はテンプレート版より 25% 遅い。** しかし **`new` より 3.0 倍速い。**

```
                                          push_back(reserve 済み、100万)
std::vector<int>(new)                           3.1 ms
std::vector<int, ga::BumpAllocator<int>>        2.9 ms
std::pmr::vector<int>(BumpResource)             2.9 ms
```

**`vector` では差が出ません。** 確保が1回だけなので、当然です。

### 判断

> **確保が頻繁でないなら、pmr のコストは無視できます。**
> **確保がホットパスにあるなら、テンプレート版を検討してください。**

第12章から繰り返している教訓と同じです。**確保がボトルネックかどうかを、まず測ってください。**

### 標準のリソースと自作の比較

```
ga::BumpResource                       5.1 ns
std::pmr::monotonic_buffer_resource    4.6 ns
```

**標準のほうがやや速い。** 私たちの `Bump` には、統計、タグ、ガード、トレースが付いているためです(Debug では無効化しても、分岐が残ります)。

**では標準のものを使えばよいのか。** そうとも言えません。

| | `monotonic_buffer_resource` | `ga::BumpResource` |
|---|---|---|
| 速度 | やや速い | やや遅い |
| タグ別集計(第15章) | **なし** | あり |
| 確保元の記録(第14章) | **なし** | あり |
| ガードバイト(第17章) | **なし** | あり |
| 可視化(第19章) | **なし** | あり |
| `Rewind`(第9章) | **なし** | あり |

**第2部で作った観測手段が、まるごと使えません。**

第17章のコラムで書いたことが、逆向きに現れています。

> 自作アロケーターを使うということは、既存のデバッグ支援を1つ失うということでもある。

**標準のものを使うと、自作のデバッグ支援を失います。**

---

## 39.7 伝播の話(第38章の回収)

`polymorphic_allocator` の伝播の型は、**すべて `false_type`** です。

```cpp
propagate_on_container_copy_assignment = false_type
propagate_on_container_move_assignment = false_type
propagate_on_container_swap            = false_type
is_always_equal                        = false_type
```

**第38章で「危険だ」と書いた設定そのものです。**

### では swap は未定義動作なのか

```cpp
std::pmr::vector<int> a(&resA);
std::pmr::vector<int> b(&resB);

a.swap(b);      // ← リソースが違う
```

**はい、未定義動作です。**

第38章と同じ問題が、標準の型でも起きます。

### なぜ標準はこの設定を選んだのか

**`polymorphic_allocator` は、意図的に「伝播しない」設計です。**

理由は、**リソースの寿命** にあります。

```cpp
{
    ga::Bump localArena(1024);
    ga::BumpResource localRes(localArena);

    std::pmr::vector<int> tmp(&localRes);
    // ...

    globalVector = std::move(tmp);   // ← 伝播したら、globalVector がローカルのリソースを指す
}
// localArena は破棄された。globalVector は宙ぶらりん
```

**伝播しないほうが、寿命の事故が起きにくい。** その代わり、ムーブ代入が O(n) になります。

### 実務上の扱い

> **同じリソースを使うコンテナ間でだけ、代入と swap を行う。**
> **異なるリソース間では、要素をコピー/ムーブする(O(n) を受け入れる)。**

型が同じなので **渡し合いは自由** です。制約があるのは、代入と swap だけです。

---

## 39.8 入れ子への自動伝播

**`std::pmr` の、最も気の利いた機能です。**

```cpp
ga::Bump arena(1024 * 1024);
ga::BumpResource res(arena);

std::pmr::vector<std::pmr::string> names(&res);

names.emplace_back("これは十分に長い文字列なのでヒープを確保します");
```

**`std::pmr::string` の中身も、`res` から確保されます。**

明示的にリソースを渡していないのに、です。

### 仕組み:uses-allocator 構築

`polymorphic_allocator` は、要素を構築するとき、**その型がアロケーターを受け取れるかを調べます**。

```cpp
if constexpr (std::uses_allocator_v<T, polymorphic_allocator<>>)
{
    // T のコンストラクタに、自分のアロケーターを渡す
    construct(p, args..., resource_);
}
else
{
    construct(p, args...);
}
```

`std::pmr::string` は `std::uses_allocator` に対応しているので、**自動的にリソースが渡ります。**

### 第38章ではこうならない

```cpp
std::vector<std::string, ga::BumpAllocator<std::string>> v(arena);
v.emplace_back("長い文字列...");
```

**`std::string` の中身は `new` から確保されます。**

`std::string` は `ga::BumpAllocator` を知らないからです。伝播させるには `std::scoped_allocator_adaptor` という別の道具が必要でした。**`std::pmr` には、それが組み込まれています。**

### 自作の型を対応させる

```cpp
class Level
{
public:
    using allocator_type = std::pmr::polymorphic_allocator<>;

    explicit Level(allocator_type alloc = {})
        : name_(alloc)
        , objects_(alloc)
    {
    }

    // アロケーターを最後の引数として受け取る版も用意する
    Level(const Level& other, allocator_type alloc)
        : name_(other.name_, alloc)
        , objects_(other.objects_, alloc)
    {
    }

private:
    std::pmr::string         name_;
    std::pmr::vector<Object> objects_;
};
```

**`allocator_type` という型別名を定義し、コンストラクタの最後でアロケーターを受け取る。** これが規約です。

```cpp
std::pmr::vector<Level> levels(&res);
levels.emplace_back();       // ← Level の中の string も vector も res を使う
```

**アプリケーション全体を、1つのアリーナに載せられます。**

> **これは非常に強力です。** 「このシーンに関するメモリは、全部このアリーナ」という設計が、入れ子の深いデータ構造に対しても成立します。
>
> **第44章のシーンアロケーターで、この形を使います。**

---

## 39.9 判断

| 状況 | 選ぶもの |
|---|---|
| 自分のコードだけで完結する | **自作 API**(`New<T>` / `NewArray<T>`) |
| 標準コンテナが必要 + 確保がホット | **テンプレート版**(第38章) |
| 標準コンテナが必要 + 型を揃えたい | **`std::pmr`**(この章) |
| 入れ子のコンテナ | **`std::pmr`**(自動伝播) |
| 既存の API と繋ぐ | **`std::pmr`** |
| 単純なアリーナで十分 | `std::pmr::monotonic_buffer_resource` |
| 観測手段が必要 | **自作の `memory_resource`** |
| ヒープを絶対に触らせない | `monotonic` + `null_memory_resource` |

### 本書の推奨

**アプリケーションのコードでは `std::pmr` を、性能が critical な内側では自作 API を。**

```cpp
// 外側:pmr で書く(読みやすい、繋げやすい)
std::pmr::vector<DrawCommand> commands(&frameResource);

// 内側:自作 API で書く(速い)
auto verts = frameArena.NewArrayUninit<Vertex>(count);
```

**両方を使い分けられることが、この章の成果です。**

---

## 演習

**演習39-1** `do_is_equal` を `dynamic_cast` 版に変えて、同じアリーナを指す2つの `BumpResource` を作ってください。swap は動きますか。

**演習39-2** `Pool` を `memory_resource` として実装してください。サイズが合わない要求はどう扱いますか。

**演習39-3** `std::pmr::monotonic_buffer_resource` に、上流として `ga::BumpResource` を指定してください。どちらがどれだけ確保しますか。

**演習39-4** スタックバッファ + `null_memory_resource` の組み合わせで、`std::pmr::vector` を溢れさせてください。何が起きますか。

**演習39-5** 39.6 節の測定を、`std::pmr::unordered_map` で行ってください。差はどれくらいですか。

**演習39-6** `set_default_resource` で既定を `BumpResource` に差し替え、`std::pmr::vector<int> v;` と書いてください。どこから確保されますか。

**演習39-7** 39.8 節の `Level` クラスを実装し、`std::pmr::vector<Level>` に入れてください。すべてのメモリが1つのアリーナに載ることを、`arena.Used()` で確認してください。

**演習39-8** `memory_resource` を継承したデバッグ用のリソース(確保をログに出す、上流に転送するだけ)を書いてください。既存のリソースに被せられますか。

---

## 章末チェックリスト

- [ ] `polymorphic_allocator` が `memory_resource*` を持つだけであることを理解した
- [ ] NVI イディオムと、`do_` を付ける理由を説明できる
- [ ] `BumpResource` / `TlsfResource` を実装した 〔v0.28〕
- [ ] 第38章のテンプレート版より実装が単純になる理由を説明できる
- [ ] 異なるリソースの `std::pmr::vector` を、同じ関数に渡した
- [ ] `monotonic_buffer_resource` が `Bump` とほぼ同じ動作であることを確認した
- [ ] `null_memory_resource` を上流にする定番パターンを使った
- [ ] **仮想呼び出しのコスト(2.1 倍)** を実測した
- [ ] 標準のリソースを使うと、自作の観測手段を失うことを理解した
- [ ] `polymorphic_allocator` の伝播がすべて `false_type` である理由を説明できる
- [ ] 入れ子のコンテナにリソースが自動伝播することを確認した

---

## 次章の予告

コンテナは繋がりました。**次はスマートポインタです。**

```cpp
auto p = std::make_unique<Enemy>(100);      // ← new を使う
auto q = std::make_shared<Enemy>(100);      // ← new を使う
```

第40章では、これらを自作アロケーターの上に載せます。

```cpp
auto p = arena.MakeUnique<Enemy>(100);
```

**単純に見えて、設計上の論点がいくつもあります。**

- **カスタムデリータのサイズコスト。** `std::unique_ptr<T>` は 8 バイトですが、デリータを持たせると膨らみます。`[[no_unique_address]]` で回避できるか
- **`std::allocate_shared` が1回の確保で済む理由。** 制御ブロックとオブジェクトを、なぜ一緒に確保できるのか
- **アリーナとスマートポインタは、そもそも相性が良いのか。** `Reset()` で一括解放するアリーナに、参照カウントは必要でしょうか

最後の問いが重要です。**便利だからといって、全部を組み合わせるべきとは限りません。**

---

> **コラム:`std::pmr` は、どこから来たのか**
>
> `std::pmr` は、C++17 で突然現れたわけではありません。**20 年近い実績を持つ設計の標準化** でした。
>
> ---
>
> **Bloomberg の BDE**
>
> 金融情報サービスの Bloomberg 社は、大規模な C++ コードベースを持っています。同社が公開している **BDE**(Basic Development Environment)というライブラリ群には、`bslma::Allocator` という抽象基底クラスがありました。
>
> ```cpp
> class Allocator
> {
> public:
>     virtual void* allocate(size_type size) = 0;
>     virtual void  deallocate(void* address) = 0;
> };
> ```
>
> **`std::pmr::memory_resource` と、ほぼ同じ形です。**
>
> ---
>
> **なぜ仮想関数を選んだのか**
>
> Bloomberg の技術者たち(John Lakos らが中心となって標準化を提案しました)が挙げた理由は、実務的なものでした。
>
> **1. 型が変わらない。** 大規模なコードベースでは、「メモリの出どころを変えたい」という要求と、「関数のシグネチャを変えたくない」という要求が同時に存在します。テンプレートでは両立できません。
>
> **2. コンパイル時間。** アロケーターごとにコンテナがインスタンス化されると、コンパイル時間とバイナリサイズが膨れ上がります。
>
> **3. 実行時に決めたい。** 設定ファイルやコマンドライン引数で、メモリの出どころを切り替える。テンプレートでは不可能です。
>
> **4. テストしやすい。** テスト用のリソース(確保を記録する、意図的に失敗させる)を、実行時に差し込めます。
>
> ---
>
> **仮想呼び出しのコストについて**
>
> 39.6 節で 2.1 倍という数字を測りました。**Bloomberg の主張は「その程度は許容できる」というものでした。**
>
> 彼らの用途では、確保がホットパスの中心にあることは稀でした。それより、**大規模なコードベースの保守性** のほうが重要だった。
>
> **私たちの用途は違います。** 第5章から見てきたとおり、ゲームでは 1 ナノ秒を削る場面があります。
>
> **だから両方を使い分けます。** 39.9 節の判断表は、その結論です。
>
> ---
>
> **標準化の過程で足されたもの**
>
> `std::pmr` には、Bloomberg の設計になかった要素も加わりました。
>
> **`monotonic_buffer_resource`。** アリーナの標準実装です。**私たちが第2章から作ってきたものが、標準ライブラリに入っている** わけです。
>
> **`pool_resource`。** サイズクラス別のプールです。第21章と第25章の組み合わせに相当します。
>
> **uses-allocator 構築。** 39.8 節で見た、入れ子への自動伝播です。C++11 の `std::scoped_allocator_adaptor` を、`polymorphic_allocator` に組み込んだ形です。
>
> ---
>
> **教訓**
>
> **標準ライブラリに入るものの多くは、どこかで長く使われた実績を持っています。**
>
> 私たちが第2章から手探りで作ってきたものが、`monotonic_buffer_resource` として既にあった。第21章のプールが、`pool_resource` としてあった。
>
> **がっかりする必要はありません。**
>
> 39.6 節で見たとおり、標準のものには **観測手段がありません**。タグ別集計も、ガードバイトも、可視化も。そして第9章の `Rewind` に相当する機能もありません。
>
> **「なぜそう作られているか」を理解した上で使うのと、ブラックボックスとして使うのとでは、まったく違います。** この本の価値は、そこにあります。
