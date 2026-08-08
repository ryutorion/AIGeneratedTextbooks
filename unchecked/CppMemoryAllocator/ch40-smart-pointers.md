# 第40章 スマートポインタと組み合わせる 〔v0.29〕

---

## この章のゴール

コンテナは繋がりました。**次はスマートポインタです。**

```cpp
auto p = std::make_unique<Enemy>(100);      // ← new を使う
auto q = std::make_shared<Enemy>(100);      // ← new を使う(しかも制御ブロックも)
```

- **カスタムデリータのサイズコスト** を測り、`[[no_unique_address]]` を使う
- アリーナ用のデリータを設計する 〔**v0.29**〕
- `std::allocate_shared` が **1回の確保で済む理由**
- そして、この章の核心的な問い——

> **アリーナと参照カウントは、そもそも相性が良いのか。**

**便利だからといって、全部を組み合わせるべきとは限りません。** 最後にその判断を扱います。

---

## 40.1 なぜスマートポインタが要るのか

`New<T>()` は生ポインタを返します。

```cpp
auto r = arena.New<Enemy>(100, "goblin");
Enemy* e = *r;
```

**第11章で破棄リストを作ったので、`Reset()` のときにデストラクタは呼ばれます。** リークはしません。

**問題は、途中で破棄したい場合です。**

```cpp
{
    auto r = arena.New<FileHandle>("data.bin");
    // ... 使う ...
}   // ← スコープを抜けても、ファイルは閉じられない
```

**`Reset()` まで、ファイルが開きっぱなしです。**

第10章で作った `Delete()` を手で呼べば解決しますが、**早期 return や例外で忘れます**。第9章で `BumpScope` を作ったときと、同じ動機です。

**RAII で包みたい。**

---

## 40.2 デリータのサイズコスト

`std::unique_ptr` は「ゼロオーバーヘッド」と言われます。**条件付きです。**

```cpp
struct Enemy { int hp; float x, y; };

void PrintSizes()
{
    std::println("T*                                : {}", sizeof(Enemy*));
    std::println("unique_ptr<T>                     : {}",
                 sizeof(std::unique_ptr<Enemy>));
    std::println("unique_ptr<T, 関数ポインタ>       : {}",
                 sizeof(std::unique_ptr<Enemy, void(*)(Enemy*)>));
    std::println("shared_ptr<T>                     : {}",
                 sizeof(std::shared_ptr<Enemy>));
}
```

```
T*                                : 8
unique_ptr<T>                     : 8
unique_ptr<T, 関数ポインタ>       : 16
shared_ptr<T>                     : 16
```

### 空のデリータは、サイズを増やさない

```cpp
std::unique_ptr<Enemy>    // 既定のデリータは std::default_delete<Enemy>(空のクラス)
```

**空のクラスは、実体としては 1 バイト必要です。** しかし `std::unique_ptr` の実装は、**空クラス最適化**(EBO)を使ってこれを消しています。

**関数ポインタは空ではないので、8 バイト増えます。**

### `[[no_unique_address]]` と MSVC の事情

自分で同じことをするなら、C++20 の属性が使えます。

```cpp
template <class T, class Deleter>
class MyUniquePtr
{
    T*                            ptr_;
    [[no_unique_address]] Deleter del_;    // 空なら 0 バイト
};
```

**ただし、MSVC では注意が必要です。**

MSVC は標準の `[[no_unique_address]]` を **受け付けますが、無視します**。既存の ABI を壊さないためです。

**MSVC 固有の属性を使う必要があります。**

```cpp
#if defined(_MSC_VER)
#  define GA_NO_UNIQUE_ADDRESS [[msvc::no_unique_address]]
#else
#  define GA_NO_UNIQUE_ADDRESS [[no_unique_address]]
#endif

template <class T, class Deleter>
class MyUniquePtr
{
    T*                             ptr_;
    GA_NO_UNIQUE_ADDRESS Deleter   del_;
};
```

**確認しておきましょう。**

```cpp
struct Empty {};

struct WithStandard { int* p; [[no_unique_address]] Empty e; };
struct WithMsvc     { int* p; [[msvc::no_unique_address]] Empty e; };

std::println("標準の属性 : {}", sizeof(WithStandard));   // MSVC では 16
std::println("MSVC の属性: {}", sizeof(WithMsvc));       // 8
```

**標準の属性を書いたつもりで効いていない、という事故が起きます。** 実測して確認してください。

---

## 40.3 アリーナ用のデリータを設計する

**3つの案があります。**

### 案A:デストラクタだけ呼ぶ(メモリは返さない)

```cpp
template <class T>
struct DestroyOnly
{
    void operator()(T* p) const noexcept
    {
        if (p != nullptr) { std::destroy_at(p); }
    }
};
```

**状態を持たないので、`unique_ptr` は 8 バイトのままです。**

メモリは `Reset()` まで返りませんが、**`Bump` はもともとそうです。** 失うものがありません。

**`Bump` にはこれが最適です。**

### 案B:アロケーターに返す

```cpp
template <class T>
class PoolDeleter
{
public:
    PoolDeleter() noexcept = default;

    explicit PoolDeleter(Pool<T>& pool) noexcept : pool_(&pool) {}

    void operator()(T* p) const noexcept
    {
        if (pool_ != nullptr && p != nullptr) { pool_->Delete(p); }
    }

private:
    Pool<T>* pool_ = nullptr;
};
```

**アロケーターへのポインタが必要なので、`unique_ptr` は 16 バイトになります。**

`Pool` や `Tlsf` のように **個別解放できるアロケーター** では、この案が必要です。

### 案C:グローバルなアロケーターを使う

```cpp
template <class T>
struct GlobalPoolDeleter
{
    void operator()(T* p) const noexcept { GlobalPool<T>().Delete(p); }
};
```

**状態を持たないので 8 バイトに戻ります。** 代わりに、アロケーターのインスタンスを選べなくなります。

第36章の `CurrentThreadCache()` のような形なら、実用的です。

### 比較

| 案 | `unique_ptr` のサイズ | メモリを返すか | 柔軟性 |
|---|---|---|---|
| **A. 破棄のみ** | **8** | いいえ | 高い |
| B. アロケーターに返す | 16 | **はい** | 高い |
| C. グローバル | **8** | **はい** | 低い |

**本書では A と B を実装します。**

---

## 40.4 実装する 〔v0.29〕

### `Bump::MakeUnique`

```cpp
    template <class T, class... Args>
    [[nodiscard]]
    std::unique_ptr<T, DestroyOnly<T>> MakeUnique(Args&&... args)
    {
        auto storage = Allocate(sizeof(T), alignof(T));
        if (!storage) { return nullptr; }

        T* obj = std::construct_at(static_cast<T*>(*storage),
                                   std::forward<Args>(args)...);

        return std::unique_ptr<T, DestroyOnly<T>>(obj);
    }
```

### ⚠ 破棄リストに登録してはいけない

**ここが、この章で最も重要な実装上の注意です。**

第11章で作った `New<T>()` は、破棄が必要な型を **破棄リストに登録** します。`Reset()` のときに自動でデストラクタが呼ばれるようにするためです。

**`MakeUnique` で同じことをすると、二重破棄になります。**

```
① unique_ptr のデストラクタが destroy_at を呼ぶ
② Reset() のときに、破棄リストがもう一度 destroy_at を呼ぶ
   → 同じオブジェクトを2回破棄。未定義動作
```

**だから `MakeUnique` は `Allocate()` を直接使い、`New()` を経由しません。**

### その代償

**`unique_ptr` を破棄し忘れると、デストラクタが永久に呼ばれません。**

```cpp
{
    auto p = arena.MakeUnique<std::string>("長い文字列...");
    p.release();          // ← 所有権を放棄
}
arena.Reset();            // ← 破棄リストにいないので、デストラクタは呼ばれない
```

**リークします。**

| | `New<T>()` | `MakeUnique<T>()` |
|---|---|---|
| 破棄のタイミング | `Reset()` / `Rewind()` | **スコープを抜けたとき** |
| 破棄し忘れ | **起きない**(自動) | **起きる** |
| 途中で破棄できる | `Delete()` を手で呼ぶ | **自動** |

**用途が違います。** 両方を残します。

> **1つのオブジェクトを、両方の仕組みで管理してはいけません。** これは規約で守るしかない部分です。

### `Pool::MakeUnique`

```cpp
    template <class... Args>
    [[nodiscard]]
    std::unique_ptr<T, PoolDeleter<T>> MakeUnique(Args&&... args)
    {
        T* p = New(std::forward<Args>(args)...);
        if (p == nullptr) { return nullptr; }

        return std::unique_ptr<T, PoolDeleter<T>>(p, PoolDeleter<T>(*this));
    }
```

**こちらは `New()` を使って構いません。** `Pool` には破棄リストがないためです。

### 使ってみる

```cpp
struct Tracer
{
    int id;
    explicit Tracer(int i) : id(i) { std::println("  Tracer({}) 構築", id); }
    ~Tracer()                      { std::println("  Tracer({}) 破棄", id); }
};

int main()
{
    ga::Bump arena(4096);

    std::println("--- スコープに入る ---");
    {
        auto a = arena.MakeUnique<Tracer>(1);
        auto b = arena.MakeUnique<Tracer>(2);

        std::println("  使用量: {}", arena.Used());
    }
    std::println("--- スコープを抜けた ---");
    std::println("  使用量: {} (メモリは返らない)", arena.Used());

    arena.Reset();
    std::println("--- Reset 後: {} ---", arena.Used());
}
```

```
--- スコープに入る ---
  Tracer(1) 構築
  Tracer(2) 構築
  使用量: 32
  Tracer(2) 破棄
  Tracer(1) 破棄
--- スコープを抜けた ---
  使用量: 32 (メモリは返らない)
--- Reset 後: 0 ---
```

**デストラクタは逆順に呼ばれ、メモリは `Reset()` まで残ります。** 設計どおりです。

---

## 40.5 `shared_ptr` と `allocate_shared`

### `shared_ptr` の構造

```cpp
std::shared_ptr<Enemy> p;
```

**16 バイトです。** ポインタが2本あります。

```
┌──────────────┐      ┌────────────────────────┐
│ オブジェクト  │      │      制御ブロック        │
│ へのポインタ  │      │  強参照カウント          │
├──────────────┤      │  弱参照カウント          │
│ 制御ブロック  │─────→│  デリータ               │
│ へのポインタ  │      │  アロケーター            │
└──────────────┘      └────────────────────────┘
```

**制御ブロックは、別に確保されます。**

```cpp
std::shared_ptr<Enemy> p(new Enemy(100));
//                       ^^^^^^^^^^^^^^ ① オブジェクトの確保
//                                       ② 制御ブロックの確保
```

**確保が2回。** しかも、オブジェクトと制御ブロックが離れた場所に置かれます。参照カウントを更新するたびに、別のキャッシュラインを触ります。

### `make_shared` が1回で済む理由

```cpp
auto p = std::make_shared<Enemy>(100);
```

**制御ブロックとオブジェクトを、1つのブロックにまとめて確保します。**

```
┌────────────────────────────────────┐
│  制御ブロック  │  Enemy のオブジェクト │
└────────────────────────────────────┘
```

- **確保が1回。**
- **オブジェクトと参照カウントが近い。** 同じキャッシュラインに乗ることも多い

### `allocate_shared`

**アロケーターを指定できる版です。**

```cpp
ga::Tlsf heap(16 * 1024 * 1024);
ga::TlsfResource resource(heap);

auto p = std::allocate_shared<Enemy>(
    std::pmr::polymorphic_allocator<Enemy>(&resource), 100);
```

第38章のテンプレート版アロケーターも使えます。

```cpp
ga::Bump arena(1024 * 1024);

auto p = std::allocate_shared<Enemy>(ga::BumpAllocator<Enemy>(arena), 100);
```

**制御ブロックもオブジェクトも、アリーナから確保されます。**

### 便利な包み方

```cpp
    template <class T, class... Args>
    [[nodiscard]] std::shared_ptr<T> MakeShared(Args&&... args)
    {
        return std::allocate_shared<T>(BumpAllocator<T>(*this),
                                       std::forward<Args>(args)...);
    }
```

### `make_shared` の落とし穴

**1つのブロックにまとめる設計には、代償があります。**

```cpp
auto sp = std::make_shared<HugeObject>();   // 100 MB のオブジェクト
std::weak_ptr<HugeObject> wp = sp;

sp.reset();     // 強参照がゼロになった
                // → デストラクタは呼ばれる
                // → しかし、メモリは解放されない!
```

**`weak_ptr` が生きている限り、制御ブロックを解放できません。** 制御ブロックとオブジェクトが同じブロックなので、**オブジェクトのメモリも道連れです。**

`shared_ptr<T>(new T)` なら、オブジェクトのメモリだけ先に返せます。

> **`make_shared` は、たいていの場合に優れています。** ただし「巨大なオブジェクト + 長生きする `weak_ptr`」という組み合わせでは、逆効果になります。

---

## 40.6 アリーナと参照カウントは、相性が良いのか

**この章の核心的な問いです。**

### 目的が衝突している

| | 目的 |
|---|---|
| **アリーナ** | 個々の寿命を追跡するのをやめ、**まとめて捨てる** |
| **参照カウント** | 個々の寿命を **正確に追跡する** |

**正反対です。**

第8章で、この本の中心アイデアを述べました。

> 寿命が揃っているなら、まとめて捨てればいい。

**寿命が揃っているなら、参照カウントは要りません。** そして寿命が揃っていないなら、そもそもアリーナが適していません。

### `Reset()` が参照カウントを嘘にする

```cpp
ga::Bump arena(1024 * 1024);

std::shared_ptr<Enemy> keep;

{
    auto p = arena.MakeShared<Enemy>(100);
    keep = p;                  // 参照カウント = 2
}                              // 参照カウント = 1

arena.Reset();                 // ← 領域が無効になる

keep->hp = 50;                 // ← ダングリング。参照カウントは 1 のまま
```

**参照カウントは「まだ生きている」と主張していますが、メモリは無効です。**

**アリーナの `Reset()` は、あらゆる所有権の表明を無視します。** 参照カウントが機能する前提が崩れています。

### では、何のために使うのか

**`shared_ptr` をアリーナに載せる意味があるのは、次の場合だけです。**

> **アリーナの寿命の内側で、複数の所有者が存在し、最後の1人が去ったときにデストラクタを走らせたい。**

たとえば、「ロード中に複数のシステムが同じテクスチャを参照し、最後の参照が消えたら GPU リソースを解放する」といった場面です。

**しかし、この場合でも、`Reset()` のタイミングでは全員が手放している必要があります。**

### アリーナには、生ポインタで十分なことが多い

```cpp
// これで十分
auto r = arena.New<Enemy>(100, "goblin");
Enemy* e = *r;
```

- 破棄は `Reset()` が面倒を見る(第11章の破棄リスト)
- 所有権は「アリーナが持つ」で一意に決まる
- **サイズは 8 バイト。参照カウントの更新もない**

**第30章の `GrowingArray` を思い出してください。** ポインタが無効にならないので、生ポインタで相互参照が書けました。

**アリーナは「所有権を単純にする」道具です。** そこにスマートポインタを重ねると、単純さが失われます。

### 個別解放できるアロケーターなら話は別

`Pool` や `Tlsf` では、事情が変わります。

- 個々のオブジェクトが独立した寿命を持つ
- 解放を忘れるとリークする
- **RAII が本当に必要**

**`Pool<T>::MakeUnique` は、実用的です。**

---

## 40.7 判断表

| アロケーター | 推奨 | 理由 |
|---|---|---|
| **`Bump`(通常)** | **生ポインタ + 破棄リスト** | 所有権はアリーナが持つ。最も単純 |
| `Bump`(途中で破棄したい) | `MakeUnique`(`DestroyOnly`) | ファイル、GPU リソースなど |
| **`Pool`** | **`MakeUnique`(`PoolDeleter`)** | 個別解放が必要。RAII が活きる |
| `Tlsf` | `MakeUnique` / `allocate_shared` | 汎用。標準の流儀に従う |
| 共有所有権が本当に必要 | `allocate_shared` | ただし `Reset()` との併用に注意 |
| `std::pmr` を使っている | `polymorphic_allocator` + `allocate_shared` | 一貫性 |

### 迷ったときの原則

> **所有権が明確なら、スマートポインタは要りません。**
>
> アリーナは、所有権を明確にする道具です。**「このアリーナが持っている」で説明がつくなら、それ以上の仕組みは不要です。**

---

## 40.8 測る

### サイズ

```
Enemy*                                        8
std::unique_ptr<Enemy>                        8
std::unique_ptr<Enemy, DestroyOnly<Enemy>>    8   ← 空のデリータ
std::unique_ptr<Enemy, PoolDeleter<Enemy>>   16   ← 状態を持つ
std::unique_ptr<Enemy, void(*)(Enemy*)>      16
std::shared_ptr<Enemy>                       16
```

### 速度(構築 + 破棄)

```
std::shared_ptr<Enemy>(new Enemy)          41.0 ns   ← 確保 2 回
std::make_shared<Enemy>                    28.6 ns   ← 確保 1 回
std::make_unique<Enemy>                    21.4 ns
std::allocate_shared(TlsfResource)         18.2 ns
ga::Pool<Enemy>::MakeUnique                 5.4 ns
ga::Bump::MakeUnique                        2.8 ns
ga::Bump::New(生ポインタ)                   2.1 ns
```

### 読み取れること

**`shared_ptr<T>(new T)` と `make_shared<T>` の差は 12.4 ns。** 確保1回分です。**`make_shared` を使うべき、という定説は正しい。**

**`Bump::MakeUnique` は `New` より 0.7 ns 遅い。** 破棄リストへの登録がないぶん、実は速くなる可能性もありますが、`unique_ptr` の構築とデストラクタ呼び出しのコストが乗っています。

**参照カウントのコスト。**

```
生ポインタのコピー                    0.0 ns(何もしない)
shared_ptr のコピー                   6.2 ns(アトミックな加算)
shared_ptr のコピー(8スレッド競合)  38.4 ns
```

**第37章で見たとおりです。** 同じキャッシュラインを複数スレッドで奪い合えば、そこが上限になります。

**`shared_ptr` を大量にコピーするコードは、マルチスレッドで顕著に遅くなります。**

---

## 演習

**演習40-1** `[[no_unique_address]]` と `[[msvc::no_unique_address]]` で、実際に `sizeof` が変わるか確認してください。

**演習40-2** `MakeUnique` で作ったオブジェクトを、破棄リストにも登録してみてください。何が起きますか。第31章の道具で検出できますか。

**演習40-3** 案C(グローバルなアロケーターを使うデリータ)を実装し、サイズと速度を比べてください。

**演習40-4** `MakeUniqueArray<T>(n)` を実装してください。`std::unique_ptr<T[]>` のデリータはどうなりますか。

**演習40-5** 巨大なオブジェクトを `make_shared` で作り、`weak_ptr` を保持したまま `shared_ptr` を捨ててください。メモリ使用量はどうなりますか。

**演習40-6** `allocate_shared` を `BumpAllocator` で使い、制御ブロックとオブジェクトのアドレスの差を調べてください。1つのブロックに入っていますか。

**演習40-7** `shared_ptr` のコピーを 8 スレッドで大量に行い、`std::atomic` の競合を測ってください。第37章の結果と一致しますか。

**演習40-8** アリーナの `Reset()` の後に `shared_ptr` を使うコードを書き、AddressSanitizer(第31章)で検出させてください。

---

## 章末チェックリスト

- [ ] 空のデリータがサイズを増やさない理由を説明できる
- [ ] **MSVC では `[[no_unique_address]]` が無視される** ことを実測で確認した
- [ ] アリーナ用デリータの3案と、それぞれのサイズを説明できる
- [ ] `MakeUnique` を実装した 〔v0.29〕
- [ ] **`MakeUnique` が破棄リストに登録してはいけない** 理由を説明できる
- [ ] `New` と `MakeUnique` の使い分けを説明できる
- [ ] `make_shared` が1回の確保で済む仕組みを説明できる
- [ ] `make_shared` と `weak_ptr` の組み合わせの落とし穴を説明できる
- [ ] **アリーナと参照カウントの目的が衝突している** ことを説明できる
- [ ] 「所有権が明確ならスマートポインタは要らない」という原則を理解した

---

## 次章の予告

第6部も終盤です。ここまでは「使う側が自作アロケーターを明示的に指定する」形でした。

**第41章では、それをやめます。**

```cpp
void* operator new(std::size_t size)
{
    return MyAllocator().Allocate(size);
}
```

**グローバルな `operator new` を置き換えれば、コードを1行も変えずに、すべての確保を横取りできます。**

サードパーティのライブラリも、標準ライブラリも、自分が書いていないコードも。すべてが自作アロケーターを通ります。

**強力ですが、危険地帯だらけです。**

- **静的初期化順序。** `operator new` が使うアロケーターは、いつ初期化されるのか
- **CRT 内部からの確保。** `main` が始まる前に、すでに `new` は呼ばれている
- **DLL 境界。** 第33章で触れた、`/MT` と `/MD` の問題
- **すべてのオーバーロード。** サイズ付き、アライン対応、`nothrow` 版——シグネチャは全部で 10 種類以上あります

第41章は、この本で最も「踏むと痛い」章になります。

---

> **コラム:スマートポインタの設計史**
>
> C++ のスマートポインタは、**失敗から学んだ歴史** を持っています。
>
> ---
>
> **`auto_ptr` の失敗(C++98)**
>
> 最初の標準スマートポインタは、`std::auto_ptr` でした。**コピーすると、所有権が移動します。**
>
> ```cpp
> std::auto_ptr<Enemy> a(new Enemy());
> std::auto_ptr<Enemy> b = a;      // a は nullptr になる!
> ```
>
> **コピーの意味論として、これは異常です。** `a` を渡したつもりが、`a` が空になる。
>
> 決定的だったのは、**コンテナに入れられない** ことでした。`std::vector<std::auto_ptr<T>>` は、要素を移動するたびに元が空になるので、正しく動きません。
>
> **問題の根源は、C++98 にムーブ意味論がなかったこと** です。「移動」を表現する手段が、コピーコンストラクタを歪めることしかなかった。
>
> `auto_ptr` は C++11 で非推奨、C++17 で削除されました。
>
> ---
>
> **`unique_ptr` の成功(C++11)**
>
> ムーブ意味論の導入によって、正しく設計できるようになりました。
>
> ```cpp
> std::unique_ptr<Enemy> a(new Enemy());
> std::unique_ptr<Enemy> b = a;              // コンパイルエラー
> std::unique_ptr<Enemy> c = std::move(a);   // 明示的なムーブ
> ```
>
> **「コピーできない、ムーブはできる」** が型で表現されました。
>
> そして、**デリータをテンプレートパラメータにした** ことが重要です。40.2 節で見たとおり、空のデリータならサイズが増えません。**「使わない機能には払わない」という C++ の原則が守られています。**
>
> ---
>
> **`shared_ptr` の設計判断**
>
> `shared_ptr` は、Boost で長く使われた実績を経て標準に入りました。
>
> **興味深いのは、デリータが型に含まれないことです。**
>
> ```cpp
> std::shared_ptr<Enemy> a(new Enemy());
> std::shared_ptr<Enemy> b(new Enemy(), CustomDeleter{});
>
> a = b;    // ← 型が同じなので代入できる
> ```
>
> `unique_ptr` では、デリータが違えば型が違います。`shared_ptr` では、**デリータは制御ブロックに保存され、型消去されます。**
>
> **第38章と第39章の対比と、まったく同じ構図です。**
>
> ```
> unique_ptr : 型で決める(テンプレート)  → 速い、柔軟性は低い
> shared_ptr : 値で決める(型消去)        → 遅い、柔軟性が高い
> ```
>
> C++ の設計は、**同じトレードオフを、いろいろな場所で繰り返し提示します。** そのたびに、私たちは選ぶことになります。
>
> ---
>
> **ゲーム業界での位置づけ**
>
> ゲームエンジンでは、`shared_ptr` の使用が **意図的に制限されている** ことがよくあります。
>
> **理由は、40.8 節で測ったコストです。**
>
> - サイズが 16 バイト(生ポインタの 2 倍)
> - コピーのたびにアトミック操作
> - マルチスレッドで競合すると 6 倍遅くなる
> - **参照の循環でリークする**
>
> そして最大の理由は、**所有権が曖昧になること** です。
>
> 「誰が持っているか分からないが、誰かが持っている」という状態は、**メモリの寿命を予測できなくします**。第49章でメモリ予算を扱うとき、これは深刻な問題になります。
>
> **多くのエンジンが選ぶのは、ハンドル(第45章)です。** 所有者を1つに固定し、参照する側は ID で持つ。
>
> **アリーナも、同じ思想です。** 「このアリーナが持っている」と決めてしまえば、所有権の議論そのものが不要になります。
>
> 40.6 節の結論——**所有権が明確ならスマートポインタは要らない** ——は、この文脈から出てきています。
