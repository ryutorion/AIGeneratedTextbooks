# 第10章 オブジェクトを作る 〔v0.6〕

---

## この章のゴール

ここまで `Bump` が返してきたのは、常に **生のバイト列** でした。使う側は毎回こう書いています。

```cpp
auto r = bump.Allocate(sizeof(Enemy), alignof(Enemy));
if (!r) { return; }
Enemy* e = static_cast<Enemy*>(*r);
```

3行かかります。`sizeof` と `alignof` を手で書いており、片方だけ書き間違えてもコンパイラは何も言いません。そして何より、**コンストラクタが呼ばれていません**。

この章では、こう書けるようにします。

```cpp
auto e = bump.New<Enemy>(100, "goblin");
```

- 記憶域(storage)とオブジェクト(object)の違いを理解する
- placement new と `std::construct_at` を使う
- `New<T>(args...)` を実装する 〔**v0.6**〕
- **デストラクタが呼ばれない** ことを実演し、実際にリークさせる

最後の項目が、この章のもう1つの主題です。便利な道具を手に入れると同時に、新しい問題を招き入れることになります。

---

## 10.1 記憶域とオブジェクトは違う

C++ には、明確に区別すべき2つの概念があります。

| | 何か |
|---|---|
| **記憶域**(storage) | ただのバイト列。値も意味も持たない |
| **オブジェクト**(object) | 型を持ち、生きている実体 |

`Bump::Allocate()` が返しているのは記憶域です。オブジェクトではありません。

オブジェクトは、**構築されたときに生まれ、破棄されたときに死にます**。生まれていないものを `T*` として読み書きするのは、規格の上では未定義動作です。

### 第2章で保留にした話

第2章で、こんな注記を置いたのを覚えているでしょうか。

```cpp
int* a = static_cast<int*>(bump.Allocate(sizeof(int)));
*a = 10;   // ← 実は規格上の議論がある
```

`int` のオブジェクトはまだ生まれていないのに、`int*` として書き込んでいました。「第42章で片づけます」と書いた問題です。

この章で導入する構築の仕組みを使えば、**その問題は半分解決します**。ちゃんとオブジェクトを生まれさせてから使うようになるからです。

残り半分——「`int` のような型なら、構築せずに書いてもよいのでは?」という議論は、C++20 で導入された **暗黙の生存期間を持つ型**(implicit-lifetime types)と、C++23 の `std::start_lifetime_as` に関わります。第42章で扱います。

### なぜ厳密さが必要なのか

「動いてるからいいじゃないか」と思うかもしれません。しかし、`std::string` を置いてみれば話は変わります。

```cpp
auto r = bump.Allocate(sizeof(std::string), alignof(std::string));
std::string* s = static_cast<std::string*>(*r);

*s = "hello";   // ← クラッシュする可能性が高い
```

`std::string` の代入演算子は、「自分は正しく構築済みである」という前提で動きます。内部のポインタを見て、必要なら解放しようとします。しかしそこにあるのは前フレームのゴミです。**でたらめなアドレスに `delete` が飛びます。**

`int` では見逃されていた区別が、クラスになると即座に牙を剥きます。

---

## 10.2 placement new

すでに確保済みの記憶域に、オブジェクトを構築する構文があります。

```cpp
#include <new>

void* storage = /* 確保済みの領域 */;

Enemy* e = ::new (storage) Enemy(100, "goblin");
```

`new` の後ろに括弧でアドレスを渡すのが **placement new** です。

**この `new` はメモリを確保しません。** 渡されたアドレスの上に、コンストラクタを呼ぶだけです。

### 3つの注意点

**1. 先頭に `::` を付ける。**

```cpp
::new (storage) Enemy(...)      // ○
new (storage) Enemy(...)        // △
```

`::` がないと、そのクラスが定義しているクラス固有の `operator new` が呼ばれる可能性があります。グローバルの placement new を確実に使うため、`::` を付けます。

**2. `delete` してはいけない。**

```cpp
delete e;   // ← 絶対にダメ
```

`delete` は「デストラクタを呼び、`operator delete` でメモリを解放する」処理です。しかしこのメモリは `operator new` から来ていません。`Bump` の板の内側です。解放しようとした瞬間に壊れます。

デストラクタを呼びたいなら、直接呼びます。

```cpp
e->~Enemy();          // 明示的なデストラクタ呼び出し
std::destroy_at(e);   // C++17 以降。こちらが読みやすい
```

**3. `<new>` のインクルードが必要。**

忘れると、環境によってはコンパイルが通りません。

---

## 10.3 `std::construct_at`

C++20 で、placement new をラップした関数が入りました。

```cpp
#include <memory>

Enemy* e = std::construct_at(static_cast<Enemy*>(storage), 100, "goblin");
```

やっていることは placement new と同じですが、いくつか利点があります。

- **戻り値が `T*`** なので、キャストが1回で済む
- **`constexpr` で使える**(コンパイル時計算に対応した実装が書ける)
- 対になる `std::destroy_at` があり、名前が揃う
- `::new` の書き忘れといったミスが起きない

本書では `std::construct_at` を標準の手段として使います。

### 注意:集成体の初期化

`std::construct_at` は、内部で **丸括弧** の初期化を行います。

```cpp
::new (p) T(args...)     // 丸括弧
```

コンストラクタを持たない集成体(aggregate)については、C++20 で丸括弧による初期化が認められました。したがって次は通ります。

```cpp
struct Vec3 { float x, y, z; };

auto p = std::construct_at(ptr, 1.0f, 2.0f, 3.0f);   // C++20 以降
```

もしコンパイラのバージョンによってこれが通らない場合は、placement new で波括弧を使ってください。

```cpp
Vec3* p = ::new (storage) Vec3{1.0f, 2.0f, 3.0f};
```

---

## 10.4 `New<T>()` を実装する 〔v0.6〕

`Bump` にメンバ関数テンプレートを足します。

```cpp
#include <memory>
#include <type_traits>
#include <utility>

class Bump
{
public:
    // ... 既存のメンバ ...

    // --- v0.6:オブジェクトを構築して返す ---
    template <class T, class... Args>
    [[nodiscard]]
    std::expected<T*, AllocError> New(Args&&... args)
    {
        auto r = Allocate(sizeof(T), alignof(T));

        if (!r)
        {
            return std::unexpected(r.error());
        }

        return std::construct_at(static_cast<T*>(*r),
                                 std::forward<Args>(args)...);
    }
};
```

たった10行です。しかし、いくつも仕事をしています。

### やっていること

**サイズとアラインメントを自動で決める。** `sizeof(T)` と `alignof(T)` を書き間違える余地がなくなりました。`alignas(32)` の型を渡せば、32 バイト境界が自動的に要求されます。

**引数を完全転送する。** `Args&&... args` と `std::forward` の組み合わせで、`T` のどんなコンストラクタにも対応します。ムーブしか許さない型も、参照を取る型も、そのまま渡せます。

**エラーを型ごと乗せ換える。** `std::expected<void*, AllocError>` から `std::expected<T*, AllocError>` へ変換しています。失敗したときは `std::unexpected` でエラーを引き継ぎます。

**キャストが1回に減る。** 呼び出し側に `static_cast` は残りません。

### 使ってみる

```cpp
struct Enemy
{
    int  hp;
    char name[16];

    Enemy(int h, const char* n) : hp(h)
    {
        std::snprintf(name, sizeof(name), "%s", n);
        std::println("  Enemy(\"{}\", hp={}) を構築", name, hp);
    }
};

struct alignas(32) Matrix
{
    float m[16];
};

int main()
{
    Bump bump(4096);

    auto e = bump.New<Enemy>(100, "goblin");
    if (e)
    {
        std::println("hp = {}", (*e)->hp);
    }

    auto m = bump.New<Matrix>();
    if (m)
    {
        std::println("Matrix のアドレスは32の倍数? {}", IsAlignedTo(*m, 32));
    }
}
```

```
  Enemy("goblin", hp=100) を構築
hp = 100
Matrix のアドレスは32の倍数? true
```

**コンストラクタが呼ばれ、アラインメントも守られています。**

### `(*e)->hp` という書き方

`e` の型は `std::expected<Enemy*, AllocError>` です。`*e` で `Enemy*` を取り出し、`->` でメンバにアクセスするので `(*e)->hp` になります。

やや読みにくいので、確認後に生ポインタへ移しておくと楽です。

```cpp
auto r = bump.New<Enemy>(100, "goblin");
if (!r) { return; }

Enemy* e = *r;      // 以降は普通のポインタとして扱う
e->hp -= 10;
```

### メンバ関数にした理由と、その代償

`New` を `Bump` のメンバにしました。`bump.New<Enemy>(...)` と書けて自然です。

代償として、**将来アロケーターが増えるたびに同じコードを書くことになります**。第20章以降でプールやフリーリストを作ると、それぞれに `New` が必要です。

本来なら、アロケーターの共通インターフェイスに対する自由関数テンプレートとして1つ書くべきです。第45章でライブラリとして整理するとき、そう作り直します。今はメンバのままで進みます。

### 例外について

`T` のコンストラクタが例外を投げた場合、**確保した領域は返りません**。`offset_` はすでに進んでいるからです。

厳密に対処するなら、`Mark()` を取って例外時に `Rewind()` する形になります。ただし成功時にも `depth_` の後始末が必要になり、実装が煩雑になります。

本書では簡素な実装を採ります。無駄になるのは、次の `Reset()` または `Rewind()` までの間だけです。例外を投げるコンストラクタをアリーナに置くこと自体が稀である、という判断です。

---

## 10.5 デストラクタは呼ばれない

さて、便利になりました。ここで招き入れた問題を直視します。

### 実演1:デストラクタが呼ばれないことを見る

```cpp
struct Tracer
{
    int id;

    explicit Tracer(int i) : id(i)
    {
        std::println("  Tracer({}) 構築", id);
    }

    ~Tracer()
    {
        std::println("  Tracer({}) 破棄", id);
    }
};

int main()
{
    Bump bump(1024);

    std::println("--- 構築 ---");
    auto a = bump.New<Tracer>(1);
    auto b = bump.New<Tracer>(2);
    auto c = bump.New<Tracer>(3);

    std::println("--- Reset を呼ぶ ---");
    bump.Reset();
    std::println("--- Reset から戻った ---");
}
```

```
--- 構築 ---
  Tracer(1) 構築
  Tracer(2) 構築
  Tracer(3) 構築
--- Reset を呼ぶ ---
--- Reset から戻った ---
```

**「破棄」が1行も出ていません。**

`Reset()` は `offset_` を 0 に戻しただけです。そこに何が置かれていたかを、`Bump` は一切知りません。

`Rewind()` も `BumpScope` も同じです。`BumpScope` はスコープを抜けるときに巻き戻しますが、その中で構築されたオブジェクトのデストラクタは呼びません。

### 実演2:実際にリークさせる

`Tracer` のようにデストラクタが何もしない型なら、呼ばれなくても実害はありません。問題は、デストラクタが **資源を解放する** 型です。

```cpp
#define _CRTDBG_MAP_ALLOC
#include <crtdbg.h>
#include <string>

int main()
{
    // 終了時にリークを報告させる(Debug 構成のみ有効)
    _CrtSetDbgFlag(_CRTDBG_ALLOC_MEM_DF | _CRTDBG_LEAK_CHECK_DF);

    {
        Bump bump(4096);

        // 短い文字列は内部に収まる(SSO)ので、あえて長い文字列にする
        auto s = bump.New<std::string>("これは十分に長い文字列なのでヒープを確保します");

        std::println("文字列: {}", **s);

        bump.Reset();   // デストラクタは呼ばれない
    }
}
```

Debug 構成で実行し、Visual Studio の **出力ウィンドウ** を見てください。

```
Detected memory leaks!
Dumping objects ->
{146} normal block at 0x000001F3A8C2B3C0, 72 bytes long.
 Data: <これは十分に長い文> ...
Object dump complete.
```

**リークしています。**

`std::string` は、長い文字列を保持するとき内部でヒープからメモリを確保します。その解放はデストラクタの仕事です。デストラクタが呼ばれなければ、そのメモリは永久に返りません。

`Bump` の板は解放されました。しかし、`std::string` が **板の外に** 確保したメモリは残ったままです。

### 同じ問題を起こす型

- `std::string`(長い文字列)
- `std::vector`、`std::map` など、ほぼすべてのコンテナ
- `std::unique_ptr`、`std::shared_ptr`
- ファイルハンドル、ソケット、GPU リソースを持つクラス
- ミューテックスをロックしている RAII オブジェクト

**要するに、RAII を使っているすべての型です。** C++ で普通に書かれたクラスの大半が該当します。

---

## 10.6 どう向き合うか:3つの方針

この問題への対処は、大きく3つに分かれます。

### 方針A:自明に破棄可能な型だけを載せる

デストラクタが何もしない型——**自明に破棄可能**(trivially destructible)な型——だけをアリーナに置く、というルールを設けます。

```cpp
std::is_trivially_destructible_v<int>          // true
std::is_trivially_destructible_v<Vec3>         // true(POD なら)
std::is_trivially_destructible_v<std::string>  // false
```

コンパイル時に強制できます。

```cpp
    template <class T, class... Args>
    [[nodiscard]]
    std::expected<T*, AllocError> NewTrivial(Args&&... args)
    {
        static_assert(std::is_trivially_destructible_v<T>,
                      "この型は破棄処理が必要です。New<T> を使い、"
                      "破棄を自分で管理してください。");

        return New<T>(std::forward<Args>(args)...);
    }
```

`NewTrivial<std::string>` と書いた瞬間に、コンパイルエラーになります。

```
error C2338: static_assert failed: 'この型は破棄処理が必要です。...'
```

**この方針は、実際のゲーム開発でよく採られます。** フレームアロケーターに載せるのは頂点データ、描画コマンド、行列といった単純な構造体だけ、と決めておけば、破棄の問題は原理的に起きません。

弱点は、便利な型が使えないことです。文字列を組み立てたいときに `std::string` が使えないのは、それなりに不便です。

### 方針B:手で破棄する

構築したら、対応する破棄を手で書きます。

```cpp
    template <class T>
    void Delete(T* p) noexcept
    {
        if (p) { std::destroy_at(p); }
        // メモリは返さない(Reset/Rewind を待つ)
    }
```

```cpp
auto r = bump.New<std::string>("長い文字列...");
if (r)
{
    std::string* s = *r;
    // ... 使う ...
    bump.Delete(s);      // 忘れると即リーク
}
bump.Reset();
```

`new` / `delete` と同じ手間が戻ってきます。しかも「忘れても何も起きない」ので、`delete` より発見が遅れます。

### 方針C:アロケーターに覚えさせる

構築したオブジェクトを記録しておき、`Reset()` / `Rewind()` のときに自動でデストラクタを呼ぶ。

これが理想ですが、実装コストがかかります。**第11章のテーマです。**

### 本書の方針

第11章で **方針C** を実装します。ただし、方針Aの `NewTrivial` も残します。

理由は性能です。破棄リストの管理はタダではありません。「デストラクタが不要だと分かっている型」については、その仕組みを丸ごとスキップできます。第11章では、この使い分けを型で自動化します。

---

## 10.7 コストを測る

`New<T>()` は `new T` に対してどうでしょうか。

```cpp
struct Vec3 { float x, y, z; };

int main()
{
    constexpr std::size_t kSamples = 1000;
    constexpr std::size_t kOps     = 10'000;

    Bump bump(kOps * sizeof(Vec3) + 4096);

    auto rBump = bench::MeasureBatch(kSamples, kOps, [&, i = std::size_t{0}]() mutable {
        bench::Escape(bump.New<Vec3>().value_or(nullptr));
        if (++i == kOps) { i = 0; bump.Reset(); }
    });

    auto rNew = bench::MeasureBatch(kSamples, kOps, [] {
        Vec3* p = new Vec3();
        bench::Escape(p);
        delete p;
    });

    bench::Print("bump.New<Vec3>()", rBump);
    bench::Print("new Vec3 / delete", rNew);
}
```

```
bump.New<Vec3>()      median=      2.1  p95=      2.2  max=        3.0
new Vec3 / delete     median=     19.4  p95=     20.1  max=     2340.0
```

**9倍の差。** 第5章と同じ傾向です。

`Allocate()` 単体の 1.8 ns から 2.1 ns に増えていますが、これは `Vec3` の構築(値初期化)のぶんです。アロケーター側のオーバーヘッドはほぼありません。テンプレートは全部インライン展開されます。

---

## 10.8 この章の完成コード

```cpp
#include <memory>
#include <new>
#include <type_traits>
#include <utility>

class Bump
{
public:
    // ... 第9章までのメンバ ...

    // --- v0.6:オブジェクトの構築 ---

    // 記憶域を確保し、その上に T を構築する。
    // 注意:デストラクタは自動では呼ばれない。
    template <class T, class... Args>
    [[nodiscard]]
    std::expected<T*, AllocError> New(Args&&... args)
    {
        auto r = Allocate(sizeof(T), alignof(T));

        if (!r)
        {
            return std::unexpected(r.error());
        }

        return std::construct_at(static_cast<T*>(*r),
                                 std::forward<Args>(args)...);
    }

    // 破棄処理が不要な型に限定した版。
    // Reset() だけで安全に片づけられることが型で保証される。
    template <class T, class... Args>
    [[nodiscard]]
    std::expected<T*, AllocError> NewTrivial(Args&&... args)
    {
        static_assert(std::is_trivially_destructible_v<T>,
                      "この型は破棄処理が必要です。New<T> を使い、"
                      "破棄を自分で管理してください。");

        return New<T>(std::forward<Args>(args)...);
    }

    // 明示的に破棄する(メモリは返さない)
    template <class T>
    void Delete(T* p) noexcept
    {
        if (p) { std::destroy_at(p); }
    }
};
```

---

## 演習

**演習10-1** `Enemy` にデストラクタを足し、`New<Enemy>` で作ったあと `Reset()` を呼んでください。デストラクタは呼ばれますか。`Delete()` を呼んだ場合はどうですか。

**演習10-2** `New<T>()` の中で、`T` のコンストラクタが例外を投げるようにしてください。`Used()` はどうなりますか。それは許容できる挙動でしょうか。

**演習10-3** `std::is_trivially_destructible_v` を、身の回りの型で調べてみてください。`std::array<int, 10>` は? `std::pair<int, std::string>` は? 予想と一致しますか。

**演習10-4** `New<T>()` を、`Bump` のメンバではなく自由関数として書き直してください。どんな引数を取るべきでしょうか。将来アロケーターが増えたとき、どちらが有利ですか。

**演習10-5** 値初期化(`New<Vec3>()`)と、初期化なし(生の記憶域をそのまま返す)で、速度を比較してください。1万個の `Vec3` でどれくらい違いますか。

**演習10-6** `Delete()` はメモリを返しません。もし「最後に確保したオブジェクトなら、メモリも返す」という実装にするとしたら、どう書きますか。それは良い設計でしょうか。

**演習10-7** `bump.New<std::string>` を `NewTrivial` に変えてコンパイルし、エラーメッセージを確認してください。読んで意味が分かるメッセージになっていますか。

---

## 章末チェックリスト

- [ ] 記憶域とオブジェクトの違いを説明できる
- [ ] placement new の3つの注意点を挙げられる
- [ ] `std::construct_at` / `std::destroy_at` を使った
- [ ] `New<T>(args...)` を実装した 〔v0.6〕
- [ ] `alignas(32)` の型でアラインメントが守られることを確認した
- [ ] **デストラクタが呼ばれない** ことを実演で確認した
- [ ] `std::string` を置いて、実際にリークを検出した
- [ ] 対処の3方針と、それぞれの長短を説明できる

---

## 次章の予告

第11章では、方針C——**アロケーターにデストラクタを覚えさせる**——を実装します。

考え方は単純です。破棄が必要なオブジェクトを構築したら、「このアドレスの、この型を、あとで破棄せよ」という記録を残しておく。`Reset()` や `Rewind()` のときに、その記録を **逆順に** たどってデストラクタを呼ぶ。

問題は、記録をどこに置くかです。別の配列を用意すれば、そのメモリはどこから来るのか。板の中に埋め込むなら、どういう形で並べるのか。そして、`Rewind()` で途中まで巻き戻すとき、どこまで破棄すればよいのか。

小さな仕組みですが、設計判断がいくつも詰まっています。そして完成すれば、`std::string` も `std::vector` も、アリーナの上で安全に使えるようになります。

---

> **コラム:確保と構築を分ける、という発想**
>
> `new T` は、2つの仕事を1つの式にまとめています。メモリを確保し、そこにオブジェクトを構築する。便利ですが、**分けられないと困る場面** があります。
>
> 最も分かりやすいのが `std::vector` です。
>
> ```cpp
> std::vector<Enemy> v;
> v.reserve(1000);   // メモリだけ確保する。Enemy は1体も作らない
> ```
>
> `reserve` は 1000 体分の記憶域を用意しますが、コンストラクタは1回も呼びません。`push_back` されて初めて、その場所にオブジェクトが構築されます。もし `new Enemy[1000]` のように確保と構築が一体だったら、この動作は実現できません。
>
> だから STL のアロケーターは、最初から2つを分けていました。
>
> ```cpp
> pointer allocate(size_type n);        // 記憶域だけ
> void construct(pointer p, ...);       // 構築だけ
> ```
>
> ---
>
> 面白いのは、その後の変遷です。
>
> C++11 で `allocator_traits` が導入され、`construct` / `destroy` は「アロケーターが提供しなければ既定の実装が使われる」ものになりました。つまり、アロケーターの必須の仕事ではなくなったのです。
>
> そして C++20 では、`std::allocator` から `construct` と `destroy` が **削除されました**。代わりに `std::construct_at` / `std::destroy_at` という、アロケーターとは独立した自由関数が用意されました。
>
> この変化が示しているのは、「メモリを配ること」と「オブジェクトを作ること」は本来別の関心事だ、という認識です。アロケーターはバイト列を配ることに専念し、構築は構築で独立した道具に任せる。
>
> ---
>
> ゲーム向けのライブラリは、もっと早くからこの分離を徹底していました。
>
> EA が公開した EASTL には、未初期化領域を扱う関数群が充実しています。Unreal Engine の `TArray` も、要素の構築と記憶域の確保を明確に分けています。理由は性能です。1万個の頂点データを確保するとき、コンストラクタを1万回呼ぶのと、記憶域だけ取って `memcpy` で埋めるのとでは、話がまるで違います。
>
> 本書の `Bump` も、この分離を保っています。`Allocate()` は記憶域だけ。`New<T>()` はその上に構築を重ねた、薄い層にすぎません。第12章で配列を扱うとき、そして第38章で `std::vector` に自作アロケーターを差し込むとき、この分離が効いてきます。
