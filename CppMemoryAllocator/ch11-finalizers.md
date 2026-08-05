# 第11章 デストラクタの後始末 〔v0.7〕

---

## この章のゴール

前章で、`New<T>()` を手に入れると同時に問題を招き入れました。

**デストラクタが呼ばれない。**

`std::string` を1つ置いて `Reset()` を呼ぶと、それだけでリークします。RAII を使うすべての型が同じ問題を抱えます。

この章では、第10章で挙げた **方針C** を実装します。

- 破棄リスト(finalizer chain)という仕組みを作る
- 関数ポインタで **型を消す**
- リストのノードをどこに置くかを設計する
- `Reset()` / `Rewind()` で **逆順に** デストラクタを呼ぶ 〔**v0.7**〕
- そして、**この機能のコストを測り、使わないという選択肢と比べる**

最後の項目が重要です。この章で作る仕組みは、第8章で誇った「`Reset()` は O(1)」という性質を壊します。それでもなお作る価値があるのか、正面から検討します。

---

## 11.1 何を作るのか

考え方は単純です。

> 破棄が必要なオブジェクトを構築したら、**「あとでこれを破棄せよ」という記録を残す**。
> `Reset()` のときに、記録を逆順にたどってデストラクタを呼ぶ。

記録は連結リストにします。新しく構築したものを先頭に繋いでいけば、先頭からたどるだけで自然に **構築の逆順** になります。

```
   板の中身

   ┌──────┬──────┬──────┬──────┬──────┬──────┐
   │ str1 │ node1│ str2 │ node2│ str3 │ node3│
   └──────┴───┬──┴──────┴───┬──┴──────┴───┬──┘
              │              │             │
   finalizers_┘              │             │
        │                    │             │
        └── node3 ──► node2 ─┘             │
                       │                   │
                       └── node1 ──► null ─┘
```

`finalizers_` が指すのは **最後に構築したもの** のノードです。そこから辿れば、str3 → str2 → str1 の順に破棄されます。

### なぜ逆順でなければならないか

C++ の他のあらゆる場所と同じ理由です。後から作られたものが、先に作られたものを参照している可能性があるからです。

```cpp
auto* logger = arena.New<Logger>();
auto* system = arena.New<System>(logger);   // logger を参照している
```

`logger` を先に壊すと、`System` のデストラクタが死んだ `logger` を触ります。ローカル変数もメンバ変数も、C++ は必ず逆順に破棄します。私たちも合わせます。

---

## 11.2 型を消す

問題があります。リストのノードには、**どの型のデストラクタを呼ぶか** を記録しなければなりません。しかしリストは1本で、そこには `std::string` も `std::vector` も `Enemy` も混ざります。

型の違う要素を、1本のリストにどう並べるか。

### 関数ポインタを持たせる

型ごとに、その型を破棄する小さな関数を用意します。

```cpp
template <class T>
void DestroyThunk(void* p) noexcept
{
    std::destroy_at(static_cast<T*>(p));
}
```

`DestroyThunk<std::string>` と `DestroyThunk<Enemy>` は別々の関数として実体化されますが、**シグネチャはどちらも `void(void*)` です。**

だから、関数ポインタとして1本のリストに並べられます。

```cpp
struct Finalizer
{
    void      (*destroy)(void*) noexcept;   // 型ごとの破棄関数
    void*       object;                     // 破棄する対象
    Finalizer*  next;                       // 次のノード
};
```

x64 では 24 バイトです。

> **これが「型消去」(type erasure)** と呼ばれる技法です。`std::function` も `std::any` も、仮想関数も、根っこは同じ発想です。「型ごとに違う処理」を「共通のシグネチャ」に押し込めることで、異なる型を一様に扱えるようにします。
>
> ここでは関数ポインタ1本で済んでいます。仮想関数を使えばもっと大げさになりますし、`std::function` を使えば動的確保が発生するかもしれません。**必要最小限の型消去** を選ぶことが、性能を保つコツです。

### `noexcept` について

`DestroyThunk` を `noexcept` にしています。デストラクタが例外を投げた場合、`std::terminate` が呼ばれます。

これは意図的です。破棄処理の途中で例外が飛ぶと、残りのオブジェクトが破棄されるかどうかが不明確になります。C++ 自体、C++11 以降はデストラクタを既定で `noexcept` にしています。同じ方針に従います。

---

## 11.3 ノードをどこに置くか

`Finalizer` ノードを格納する場所を決めます。3つ考えられます。

### 案A:別の `std::vector` に持つ

```cpp
std::vector<Finalizer> finalizers_;
```

素直ですが、問題があります。

- **メモリを配るクラスが、別の場所でメモリを確保する** ことになる
- `push_back` のたびに再確保が起きうる(第5章で見た µs 級のスパイク)
- `Rewind()` のとき、どこまで戻すかを別途管理する必要がある

### 案B:板の中に、後ろから詰める

板の前からオブジェクトを、後ろからノードを詰めていく方式です。

```
   ┌──────────────────────────┬─────────┐
   │ オブジェクト →           │ ← ノード │
   └──────────────────────────┴─────────┘
```

メモリ効率は良いのですが、`offset_` を2本管理することになり、溢れ判定も複雑になります。

### 案C:板の中に、オブジェクトと一緒に詰める ← 採用

ノードもただのオブジェクトなので、`Allocate()` で普通に確保します。

```
   ┌──────┬──────┬──────┬──────┐
   │ str1 │ node1│ str2 │ node2│
   └──────┴──────┴──────┴──────┘
```

**利点が多い方式です。**

- 新しい仕組みが要らない(既存の `Allocate()` を使うだけ)
- ノードは必ずオブジェクトより **後ろ** にあるので、`Rewind()` で一緒に消える
- 溢れ判定も既存のものがそのまま働く
- 追加のメモリ確保が発生しない

欠点は、オブジェクト同士が連続して並ばなくなることです。間にノードが挟まるので、キャッシュ効率がわずかに落ちます。これは後で測ります。

**このように、管理用のデータを対象データの中に埋め込む方式を、侵入的(intrusive)と呼びます。** 第21章のプールアロケーターでも、同じ発想が出てきます。

---

## 11.4 `Marker` にチェーンの頭を持たせる

`Rewind()` が正しく動くために、もう1つ仕掛けが要ります。

第9章の `Marker` は `offset_` を記録していました。同じように、**その時点のチェーンの先頭** も記録します。

```cpp
struct Marker
{
    std::size_t   offset      = 0;
    std::size_t   padding     = 0;
    std::uint32_t depth       = 0;
    std::uint32_t epoch       = kInvalidEpoch;
    Finalizer*    finalizers  = nullptr;   // v0.7 で追加
};
```

`Rewind(m)` は、こう動きます。

1. 現在の `finalizers_` から、`m.finalizers` に到達するまで破棄関数を呼ぶ
2. `finalizers_` を `m.finalizers` に戻す
3. `offset_` を `m.offset` に戻す

**破棄を先に、巻き戻しを後に。** 順序が逆だと、まだデストラクタが必要なメモリを解放済みとして扱ってしまいます。

---

## 11.5 実装する 〔v0.7〕

```cpp
// ---------------------------------------------------------
// 破棄リストのノード
// ---------------------------------------------------------
struct Finalizer
{
    void      (*destroy)(void*) noexcept;
    void*       object;
    Finalizer*  next;
};

template <class T>
void DestroyThunk(void* p) noexcept
{
    std::destroy_at(static_cast<T*>(p));
}

// ---------------------------------------------------------
class Bump
{
public:
    // ... Marker に finalizers を追加 ...

    ~Bump()
    {
        RunFinalizersUntil(nullptr);   // 破棄されるときも忘れずに
    }

    // コピーもムーブも禁止(チェーンが板の中を指しているため)
    Bump(const Bump&)            = delete;
    Bump& operator=(const Bump&) = delete;
    Bump(Bump&&)                 = delete;
    Bump& operator=(Bump&&)      = delete;

    // --- v0.7:破棄登録つきの構築 ---
    template <class T, class... Args>
    [[nodiscard]]
    std::expected<T*, AllocError> New(Args&&... args)
    {
        auto storage = Allocate(sizeof(T), alignof(T));
        if (!storage)
        {
            return std::unexpected(storage.error());
        }

        if constexpr (std::is_trivially_destructible_v<T>)
        {
            // 破棄が不要な型は、記録しない(コストゼロ)
            return std::construct_at(static_cast<T*>(*storage),
                                     std::forward<Args>(args)...);
        }
        else
        {
            // ノードの領域も先に確保しておく
            auto node = Allocate(sizeof(Finalizer), alignof(Finalizer));
            if (!node)
            {
                return std::unexpected(node.error());
            }

            // 構築が成功してから、はじめてリストに繋ぐ
            T* obj = std::construct_at(static_cast<T*>(*storage),
                                       std::forward<Args>(args)...);

            auto* f = std::construct_at(static_cast<Finalizer*>(*node),
                                        Finalizer{ &DestroyThunk<T>, obj, finalizers_ });
            finalizers_ = f;

            return obj;
        }
    }

    void Rewind(const Marker& m) noexcept
    {
        assert(m.epoch == epoch_     && "Reset をまたいだマーカーです");
        assert(m.depth + 1 == depth_ && "巻き戻しの順序が LIFO ではありません");
        assert(m.offset <= offset_   && "前方に巻き戻そうとしています");

        if (m.epoch != epoch_ || m.offset > offset_) { return; }

        RunFinalizersUntil(m.finalizers);   // ← 先に破棄

        if (offset_ > peak_) { peak_ = offset_; }

        offset_     = m.offset;
        padding_    = m.padding;
        depth_      = m.depth;
        finalizers_ = m.finalizers;
    }

    void Reset() noexcept
    {
        RunFinalizersUntil(nullptr);        // ← 先に破棄

        if (offset_ > peak_) { peak_ = offset_; }

        offset_     = 0;
        padding_    = 0;
        depth_      = 0;
        finalizers_ = nullptr;
        ++epoch_;
    }

    [[nodiscard]]
    Marker Mark() noexcept
    {
        return Marker{ offset_, padding_, depth_++, epoch_, finalizers_ };
    }

    // 登録されている破棄処理の数(デバッグ用)
    std::size_t PendingFinalizerCount() const noexcept
    {
        std::size_t n = 0;
        for (const Finalizer* f = finalizers_; f != nullptr; f = f->next) { ++n; }
        return n;
    }

private:
    // stop に到達するまで、先頭から順に破棄する
    void RunFinalizersUntil(Finalizer* stop) noexcept
    {
        while (finalizers_ != stop)
        {
            Finalizer* node = finalizers_;
            finalizers_ = node->next;

            node->destroy(node->object);
        }
    }

    // ... 既存のメンバ ...
    Finalizer* finalizers_ = nullptr;   // v0.7
};
```

### 設計上のポイント

**`if constexpr` による分岐。** `std::is_trivially_destructible_v<T>` が真なら、ノードの確保もリストへの登録も **コンパイル時に消えます**。`Vec3` や `int` を確保するコストは、第10章とまったく同じです。

**確保を先に、構築を後に。** ノードの領域を確保してから `T` を構築しています。逆にすると、構築後にノードの確保が失敗した場合、破棄されないオブジェクトが生まれてしまいます。

**リストに繋ぐのは構築が成功してから。** `T` のコンストラクタが例外を投げた場合、ノードはリストに入りません。存在しないオブジェクトに対してデストラクタが呼ばれる事故を防げます。

**デストラクタを追加した。** `Bump` 自身が破棄されるときも、残っているオブジェクトを片づけます。これがないと、`Reset()` を呼ばずにスコープを抜けたときにリークします。

**コピーとムーブを禁止した。** `finalizers_` は板の内部を指しています。コピーすれば、2つの `Bump` が同じオブジェクトを破棄しようとします。ムーブも、`std::vector` の中身が移動するとポインタの意味が変わりうるため、いまは禁止しておきます。

---

## 11.6 動かす

### 破棄の順序を確認する

```cpp
struct Tracer
{
    int id;
    explicit Tracer(int i) : id(i) { std::println("  Tracer({}) 構築", id); }
    ~Tracer()                      { std::println("  Tracer({}) 破棄", id); }
};

int main()
{
    Bump bump(1024);

    std::println("--- 構築 ---");
    auto a = bump.New<Tracer>(1);
    auto b = bump.New<Tracer>(2);
    auto c = bump.New<Tracer>(3);

    std::println("登録数: {}", bump.PendingFinalizerCount());

    std::println("--- Reset ---");
    bump.Reset();
    std::println("--- 完了 ---");
}
```

```
--- 構築 ---
  Tracer(1) 構築
  Tracer(2) 構築
  Tracer(3) 構築
登録数: 3
--- Reset ---
  Tracer(3) 破棄
  Tracer(2) 破棄
  Tracer(1) 破棄
--- 完了 ---
```

**3 → 2 → 1 の逆順で破棄されました。**

### `Rewind()` で途中まで

```cpp
int main()
{
    Bump bump(1024);

    auto a = bump.New<Tracer>(1);

    {
        BumpScope scope(bump);

        auto b = bump.New<Tracer>(2);
        auto c = bump.New<Tracer>(3);

        std::println("スコープ内: 登録数 {}", bump.PendingFinalizerCount());
    }   // ← ここで 3, 2 だけ破棄される

    std::println("スコープ外: 登録数 {}", bump.PendingFinalizerCount());

    bump.Reset();
}
```

```
  Tracer(1) 構築
  Tracer(2) 構築
  Tracer(3) 構築
スコープ内: 登録数 3
  Tracer(3) 破棄
  Tracer(2) 破棄
スコープ外: 登録数 1
  Tracer(1) 破棄
```

`Tracer(1)` はスコープの外で作られたので、スコープを抜けても生き残っています。`BumpScope` と破棄リストが正しく噛み合っています。

### リークが消えたことを確認する

第10章でリークしたコードを、そのまま走らせます。

```cpp
int main()
{
    _CrtSetDbgFlag(_CRTDBG_ALLOC_MEM_DF | _CRTDBG_LEAK_CHECK_DF);

    {
        Bump bump(4096);
        auto s = bump.New<std::string>("これは十分に長い文字列なのでヒープを確保します");
        std::println("文字列: {}", **s);
        bump.Reset();
    }
}
```

出力ウィンドウを見てください。

```
(Detected memory leaks! は出ない)
```

**リークが消えました。** アリーナの上で `std::string` を安全に使えるようになりました。

### テスト

```cpp
struct DtorCounter
{
    static inline int count = 0;
    ~DtorCounter() { ++count; }
};

void Test_DestructorsAreCalledOnReset()
{
    DtorCounter::count = 0;

    {
        Bump bump(4096);
        for (int i = 0; i < 10; ++i)
        {
            const bool ok = bump.New<DtorCounter>().has_value();
            assert(ok);
        }

        assert(DtorCounter::count == 0);
        assert(bump.PendingFinalizerCount() == 10);

        bump.Reset();
        assert(DtorCounter::count == 10);
        assert(bump.PendingFinalizerCount() == 0);
    }

    std::println("[ OK ] Test_DestructorsAreCalledOnReset");
}

void Test_TrivialTypesAreNotRegistered()
{
    Bump bump(4096);

    for (int i = 0; i < 100; ++i)
    {
        const bool ok = bump.New<int>(i).has_value();
        assert(ok);
    }

    // int は自明に破棄可能なので、1つも登録されない
    assert(bump.PendingFinalizerCount() == 0);

    std::println("[ OK ] Test_TrivialTypesAreNotRegistered");
}

void Test_DestructorsRunWhenArenaDies()
{
    DtorCounter::count = 0;

    {
        Bump bump(4096);
        const bool ok = bump.New<DtorCounter>().has_value();
        assert(ok);
        // Reset を呼ばずにスコープを抜ける
    }

    assert(DtorCounter::count == 1);   // デストラクタが片づけた

    std::println("[ OK ] Test_DestructorsRunWhenArenaDies");
}
```

---

## 11.7 コスト:`O(1)` が `O(n)` に戻る

ここからが、この章のもう1つの主題です。

第8章で、私たちはこう誇りました。

> `Bump::Reset()` は **O(1)**。100万個入っていようと、整数を1つ書き換えるだけ。

**その性質は、破棄が必要な型を載せた瞬間に失われます。**

### 測る

```cpp
struct Small { int a, b, c, d; };                    // 自明に破棄可能

struct WithDtor { int a, b, c, d; ~WithDtor() {} };  // 自明でない

template <class T>
double MeasureResetCost(std::size_t count)
{
    Bump bump(count * 64 + 4096);

    for (std::size_t i = 0; i < count; ++i)
    {
        bench::Escape(bump.New<T>().value_or(nullptr));
    }

    const auto a = std::chrono::steady_clock::now();
    bump.Reset();
    const auto b = std::chrono::steady_clock::now();

    return std::chrono::duration<double, std::nano>(b - a).count();
}

int main()
{
    constexpr std::size_t kCount = 1'000'000;

    std::println("100万オブジェクトの Reset()");
    std::println("  自明に破棄可能  : {:>12.0f} ns", MeasureResetCost<Small>(kCount));
    std::println("  デストラクタあり: {:>12.0f} ns", MeasureResetCost<WithDtor>(kCount));
}
```

```
100万オブジェクトの Reset()
  自明に破棄可能  :            0 ns
  デストラクタあり:      8420000 ns
```

**8.4 ミリ秒。** 16.6 ms 予算の半分です。

デストラクタの中身は空なのに、これだけかかります。かかっているのは、

- 連結リストを100万回たどる(ポインタを追うので、キャッシュに優しくない)
- 関数ポインタ経由で100万回呼び出す(間接呼び出しは分岐予測が効きにくい)

という処理です。**デストラクタが空でも O(n) は O(n) です。**

### 確保コストも上がる

```
New<Small>()    median=      2.1  p95=      2.2  max=        3.0
New<WithDtor>() median=      4.6  p95=      4.8  max=        6.2
```

**2倍以上** になりました。ノードの確保(24 バイト)と、リストへの繋ぎ込みのぶんです。

メモリ消費も増えます。16 バイトのオブジェクトに 24 バイトのノードが付くので、**実質 2.5 倍** です(アラインメントの詰め物も含めて)。

---

## 11.8 だから「載せない」という選択がある

数字が出揃いました。整理します。

| | 自明に破棄可能な型 | デストラクタが必要な型 |
|---|---|---|
| 確保コスト | 2.1 ns | 4.6 ns |
| `Reset()` | **O(1)** / 約 0 ns | **O(n)** / 8.4 ms(100万件) |
| メモリ | 16 バイト | 40 バイト超 |
| 使える型 | POD、単純な構造体 | 何でも |

第10章で挙げた **方針A**——「自明に破棄可能な型だけ載せる」——の意味が、はっきりしたはずです。

> **破棄リストを作らないことが、性能上の設計判断になる。**

### 実際にどう使い分けるか

本書の立場は、**両方持つ** です。`New<T>()` は自動的に判断してくれるので、使う側は何も意識しなくても正しく動きます。しかし、性能が問題になる場所では意図を明示すべきです。

```cpp
// フレームアロケーターに載せるのは POD だけ、と決める
auto* cmd = frameArena.NewTrivial<DrawCommand>(...);   // ← コンパイル時に強制

// シーンアロケーターは何でも受け入れる(切り替えは数秒に1回なので O(n) でよい)
auto* name = sceneArena.New<std::string>("stage_01");
```

`NewTrivial<T>` は第10章で作った `static_assert` 付きの版です。**「ここには破棄が必要な型を置かない」という設計判断を、型で表明する** ために使います。

### 判断の目安

| 状況 | 選択 |
|---|---|
| 毎フレーム `Reset()` する | **`NewTrivial` に限定する** |
| シーン切り替え時に `Reset()` する | `New` でよい(数秒に1回なら 8 ms も許容範囲) |
| オブジェクト数が数千以下 | `New` でよい |
| オブジェクト数が数十万以上 | 破棄が要らない設計にできないか検討する |
| 破棄が要るが数も多い | 型ごとに専用のプールを持つ(第21章) |

### 設計を変えるという手もある

「デストラクタが必要な型を大量に置きたい」という要求が出てきたら、**そもそもの設計を疑う** 価値があります。

たとえば `std::string` を10万個アリーナに置きたいなら、次のような代替があります。

- 文字列本体をアリーナに直接書き、`std::string_view` で参照する
- 全文字列を1本のバッファに連結し、オフセットで参照する

どちらも破棄が不要になります。**破棄リストが要らない形にデータを設計し直す** ほうが、破棄リストを速くするより、たいてい効果が大きい。

これは第9部(ゲーム特有のパターン)を通して繰り返し出てくる考え方です。

---

## 11.9 この章の完成コード

```cpp
struct Finalizer
{
    void      (*destroy)(void*) noexcept;
    void*       object;
    Finalizer*  next;
};

template <class T>
void DestroyThunk(void* p) noexcept
{
    std::destroy_at(static_cast<T*>(p));
}

class Bump
{
public:
    static constexpr std::uint32_t kInvalidEpoch = 0xFFFF'FFFFu;

    struct Marker
    {
        std::size_t   offset     = 0;
        std::size_t   padding    = 0;
        std::uint32_t depth      = 0;
        std::uint32_t epoch      = kInvalidEpoch;
        Finalizer*    finalizers = nullptr;
    };

    explicit Bump(std::size_t capacity) : buffer_(capacity) {}

    ~Bump() { RunFinalizersUntil(nullptr); }

    Bump(const Bump&)            = delete;
    Bump& operator=(const Bump&) = delete;
    Bump(Bump&&)                 = delete;
    Bump& operator=(Bump&&)      = delete;

    [[nodiscard]]
    AllocResult Allocate(std::size_t size,
                         std::size_t alignment = kDefaultAlignment) noexcept
    {
        // (第7章のまま)
        ...
    }

    template <class T, class... Args>
    [[nodiscard]]
    std::expected<T*, AllocError> New(Args&&... args)
    {
        auto storage = Allocate(sizeof(T), alignof(T));
        if (!storage) { return std::unexpected(storage.error()); }

        if constexpr (std::is_trivially_destructible_v<T>)
        {
            return std::construct_at(static_cast<T*>(*storage),
                                     std::forward<Args>(args)...);
        }
        else
        {
            auto node = Allocate(sizeof(Finalizer), alignof(Finalizer));
            if (!node) { return std::unexpected(node.error()); }

            T* obj = std::construct_at(static_cast<T*>(*storage),
                                       std::forward<Args>(args)...);

            auto* f = std::construct_at(static_cast<Finalizer*>(*node),
                                        Finalizer{ &DestroyThunk<T>, obj, finalizers_ });
            finalizers_ = f;

            return obj;
        }
    }

    template <class T, class... Args>
    [[nodiscard]]
    std::expected<T*, AllocError> NewTrivial(Args&&... args)
    {
        static_assert(std::is_trivially_destructible_v<T>,
                      "この型は破棄処理が必要です。New<T> を使ってください。");
        return New<T>(std::forward<Args>(args)...);
    }

    [[nodiscard]]
    Marker Mark() noexcept
    {
        return Marker{ offset_, padding_, depth_++, epoch_, finalizers_ };
    }

    void Rewind(const Marker& m) noexcept
    {
        assert(m.epoch == epoch_     && "Reset をまたいだマーカーです");
        assert(m.depth + 1 == depth_ && "巻き戻しの順序が LIFO ではありません");
        assert(m.offset <= offset_   && "前方に巻き戻そうとしています");

        if (m.epoch != epoch_ || m.offset > offset_) { return; }

        RunFinalizersUntil(m.finalizers);

        if (offset_ > peak_) { peak_ = offset_; }

        offset_     = m.offset;
        padding_    = m.padding;
        depth_      = m.depth;
        finalizers_ = m.finalizers;
    }

    void Reset() noexcept
    {
        RunFinalizersUntil(nullptr);

        if (offset_ > peak_) { peak_ = offset_; }

        offset_     = 0;
        padding_    = 0;
        depth_      = 0;
        finalizers_ = nullptr;
        ++epoch_;
    }

    std::size_t Used()      const noexcept { return offset_; }
    std::size_t Capacity()  const noexcept { return buffer_.size(); }
    std::size_t Remaining() const noexcept { return Capacity() - Used(); }
    std::size_t Padding()   const noexcept { return padding_; }
    std::size_t Peak()      const noexcept { return (offset_ > peak_) ? offset_ : peak_; }

    std::size_t PendingFinalizerCount() const noexcept
    {
        std::size_t n = 0;
        for (const Finalizer* f = finalizers_; f; f = f->next) { ++n; }
        return n;
    }

    const std::byte* Base() const noexcept { return buffer_.data(); }

private:
    void RunFinalizersUntil(Finalizer* stop) noexcept
    {
        while (finalizers_ != stop)
        {
            Finalizer* node = finalizers_;
            finalizers_ = node->next;
            node->destroy(node->object);
        }
    }

    std::vector<std::byte> buffer_;
    std::size_t            offset_     = 0;
    std::size_t            padding_    = 0;
    std::size_t            peak_       = 0;
    std::uint32_t          depth_      = 0;
    std::uint32_t          epoch_      = 0;
    Finalizer*             finalizers_ = nullptr;
};
```

---

## 演習

**演習11-1** `Finalizer` は 24 バイトです。`next` を持たず、ノードを配列として連続配置する設計にすれば 16 バイトに減らせます。どう実装しますか。`Rewind()` は書けますか。

**演習11-2** `DestroyThunk<T>` は型ごとに1つ実体化されます。100種類の型を使うと、実行ファイルはどれくらい大きくなりますか。実測してください。

**演習11-3** デストラクタの中で例外を投げる型を作り、`Reset()` を呼んでください。何が起きますか。

**演習11-4** デストラクタの中でさらに `New<T>()` を呼ぶ型を作ってください。何が起きますか。これは防ぐべきでしょうか。

**演習11-5** `Bump` のムーブを許すには、何を直す必要がありますか。`std::vector` のムーブ後もポインタは有効ですが、それで十分でしょうか。

**演習11-6** 11.7 の測定を、リストをたどる代わりに配列を順に走査する実装で行い、比較してください。どれくらい速くなりますか。

**演習11-7** 11.8 で触れた「文字列本体をアリーナに直接書き、`std::string_view` で参照する」方式を実装してください。破棄リストは何件になりますか。

---

## 章末チェックリスト

- [ ] 破棄リストを実装した 〔v0.7〕
- [ ] 関数ポインタによる型消去の仕組みを説明できる
- [ ] ノードをアリーナ内に置く(侵入的)理由を説明できる
- [ ] `Marker` にチェーンの頭を持たせる理由を説明できる
- [ ] 逆順で破棄されることを実演で確認した
- [ ] `std::string` のリークが消えたことを確認した
- [ ] **`Reset()` が O(n) になった** ことを測定で確認した
- [ ] `NewTrivial` を使う判断基準を説明できる

---

## 次章の予告

オブジェクトを1つ作れるようになりました。次は **配列** です。

```cpp
auto particles = bump.NewArray<Particle>(10'000);
```

配列には、単体とは違う論点があります。要素数をどう返すか。境界外アクセスをどう防ぐか。破棄が必要な型なら、10,000 個ぶんのノードを作るのか(答えは「作らない」です)。

第12章では `std::span<T>` を返す形にします。ポインタと要素数が一体になった型で、範囲 for も `std::ranges` のアルゴリズムもそのまま使えます。アリーナ上のデータが、いよいよ「普通の C++」として扱えるようになります。

---

> **コラム:後始末を予約する、という発想**
>
> 「今は何もせず、後でまとめて片づける」という仕組みは、いろいろな言語や環境で再発明されてきました。
>
> **Apache のプールには cleanup 登録があります。** `apr_pool_cleanup_register` でプールに後始末関数を登録しておくと、プールが破棄されるときに呼ばれます。ファイルハンドルやソケットの解放に使われます。この章で作ったものと、ほぼ同じ構造です。
>
> **Objective-C のオートリリースプール** は、`autorelease` されたオブジェクトをプールに溜め、プールが排出されるときにまとめて `release` します。iOS アプリのイベントループが1周するたびにプールが排出される、という設計でした。
>
> **Go の `defer`** は、関数の終わりに実行する処理を登録します。実行順は登録の逆順です。実装は関数フレームに繋がる連結リストで、この章の `finalizers_` とよく似ています。
>
> **C++ のスコープガード**(第9章の `BumpScope`)も同じ発想です。Boost.ScopeExit や、標準化が検討されている `scope_exit` は、これを汎用化したものです。
>
> ---
>
> 対照的なのが、**ガベージコレクタのファイナライザ** です。
>
> Java の `finalize()` や C# のファイナライザは、「オブジェクトが回収されるときに呼ばれる後始末」ですが、**いつ呼ばれるか分かりません**。GC が動かなければ永久に呼ばれないこともあります。この不確定性のため、ファイナライザでファイルを閉じる、といった使い方は誤りとされ、Java では `finalize()` そのものが非推奨になりました。
>
> C# が `IDisposable` と `using` を用意し、Java が try-with-resources を導入したのは、「後始末のタイミングを明示的に制御したい」という要求への答えです。**C++ が最初から持っていたものに、他の言語が追いついてきた** とも言えます。
>
> ---
>
> この章で私たちが作ったのは、C++ の RAII を **アリーナの単位に拡張したもの** です。
>
> 通常の C++ では、破棄のタイミングはスコープが決めます。アリーナでは、`Reset()` や `Rewind()` が決めます。決めるのが誰であれ、**「決定的なタイミングで、逆順に、確実に呼ばれる」** という性質は保たれています。
>
> そして 11.7 で見たとおり、その保証にはコストがあります。C++ が「ゼロオーバーヘッド原則」を掲げながら、実際には「使わないものには払わない」と言っているのは、まさにこういう場面のためです。`if constexpr` で自明な型を除外したのは、その原則の実践でした。
