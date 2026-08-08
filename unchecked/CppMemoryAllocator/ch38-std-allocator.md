# 第38章 `std::vector` に自作アロケーターを渡す 〔v0.27〕

---

## この章のゴール

第6部が始まります。ここまで作ってきたアロケーターは、**私たち専用の API** を持っています。

```cpp
auto r = arena.Allocate(size, alignment);
auto p = arena.New<Enemy>(100, "goblin");
auto s = arena.NewArray<Particle>(1000);
```

**標準ライブラリとは繋がっていません。**

```cpp
std::vector<Particle> v;      // ← new を使ってしまう
std::map<int, Enemy>  m;      // ← ノードごとに new
```

この章で繋ぎます。

- C++ の **`Allocator` 要件** を最小限で満たすアダプタを書く 〔**v0.27**〕
- `std::vector` / `std::list` / `std::map` を、自作アロケーターの上で動かす
- **4つの罠** に、順番に落ちる
- とくに `propagate_on_container_*` という長い名前の型を理解する
- そして **「差し込めるからといって、差し込むべきとは限らない」** という結論に至る

---

## 38.1 `Allocator` 要件

C++11 以降、標準コンテナが要求するものは、驚くほど少なくなりました。

**必須は4つだけです。**

```cpp
template <class T>
class MyAllocator
{
public:
    using value_type = T;                       // ①

    T*   allocate(std::size_t n);               // ②
    void deallocate(T* p, std::size_t n);       // ③

    // ④ 等値比較(C++20 では == だけ書けば != は自動生成)
    friend bool operator==(const MyAllocator&, const MyAllocator&) noexcept;
};
```

さらに、**別の型へのコピー構築** が必要です。

```cpp
    template <class U>
    MyAllocator(const MyAllocator<U>& other) noexcept;
```

`std::list<int>` は `int` を確保するのではなく、**`int` を含むノード** を確保します。だから `MyAllocator<int>` から `MyAllocator<ListNode>` を作れる必要があります。**リバインド** と呼ばれる仕組みです。

### 昔は、もっと大変だった

C++98 の時代は、次のすべてを自分で書く必要がありました。

```cpp
typedef T*              pointer;
typedef const T*        const_pointer;
typedef T&              reference;
typedef const T&        const_reference;
typedef std::size_t     size_type;
typedef std::ptrdiff_t  difference_type;

template <class U> struct rebind { typedef MyAllocator<U> other; };

pointer   address(reference x) const;
size_type max_size() const;
void      construct(pointer p, const T& val);
void      destroy(pointer p);
```

**C++11 の `std::allocator_traits` が、これらすべてに既定値を用意しました。** 書かなければ、標準的な実装が自動的に使われます。

第10章のコラムで、`construct` / `destroy` が C++20 で `std::allocator` から削除され、`std::construct_at` / `std::destroy_at` に移った話をしました。**アロケーターの責務が「メモリを配ること」に絞られていく流れの一部です。**

---

## 38.2 アダプタを書く 〔v0.27〕

```cpp
// ga/StdAllocator.h
#pragma once

#include "ga/Bump.h"

#include <limits>
#include <memory>
#include <new>
#include <type_traits>

namespace ga
{
    template <class T>
    class BumpAllocator
    {
    public:
        using value_type = T;

        // --- 伝播の方針(38.5 節で詳しく)---
        using propagate_on_container_copy_assignment = std::false_type;
        using propagate_on_container_move_assignment = std::true_type;
        using propagate_on_container_swap            = std::true_type;
        using is_always_equal                        = std::false_type;

        BumpAllocator(Bump& arena) noexcept
            : arena_(&arena)
        {
        }

        // リバインド用のコピー構築
        template <class U>
        BumpAllocator(const BumpAllocator<U>& other) noexcept
            : arena_(other.arena_)
        {
        }

        [[nodiscard]] T* allocate(std::size_t n)
        {
            if (n > (std::numeric_limits<std::size_t>::max)() / sizeof(T))
            {
                throw std::bad_alloc{};
            }

            auto r = arena_->Allocate(n * sizeof(T), alignof(T));

            if (!r) { throw std::bad_alloc{}; }

            return static_cast<T*>(*r);
        }

        void deallocate(T*, std::size_t) noexcept
        {
            // Bump は個別解放しない。Reset() を待つ。
        }

        friend bool operator==(const BumpAllocator& a, const BumpAllocator& b) noexcept
        {
            return a.arena_ == b.arena_;
        }

    private:
        template <class U> friend class BumpAllocator;

        Bump* arena_;
    };
}
```

### 設計上の判断

**① 例外を投げる。**

第7章で `std::expected` を選び、「例外は使わない」と決めました。**しかし標準コンテナは、確保の失敗を例外で受け取ります。** 選択の余地がありません。

```cpp
if (!r) { throw std::bad_alloc{}; }
```

**例外を無効化しているプロジェクトでは、`std::abort()` するしかありません。** 第7章の「選択肢4:失敗を許さない(即死)」です。

```cpp
if (!r)
{
#if GA_USE_EXCEPTIONS
    throw std::bad_alloc{};
#else
    std::abort();
#endif
}
```

**② `deallocate` が何もしない。**

`Bump` は個別解放できません。**返された領域は、`Reset()` まで放置されます。**

これが 38.4 節の罠に直結します。

**③ 暗黙のコンストラクタにした。**

```cpp
BumpAllocator(Bump& arena) noexcept    // explicit なし
```

`explicit` を付けると、次のように書けなくなります。

```cpp
std::vector<int, ga::BumpAllocator<int>> v(arena);   // Bump& から変換
```

**利便性を優先しました。**

**④ サイズの桁溢れ検査。**

第12章で `NewArray` に入れたのと同じ検査です。**`n * sizeof(T)` は溢れます。** `allocate` は標準の入口なので、防御は必須です。

---

## 38.3 動かす

### `std::vector`

```cpp
int main()
{
    ga::Bump arena(16 * 1024 * 1024);

    using IntVec = std::vector<int, ga::BumpAllocator<int>>;

    IntVec v(arena);          // アロケーターを渡す

    for (int i = 0; i < 10; ++i) { v.push_back(i); }

    std::println("要素数: {}", v.size());
    std::println("アリーナ使用量: {}", ga::FormatBytes(arena.Used()));

    for (int x : v) { std::print("{} ", x); }
    std::println("");
}
```

```
要素数: 10
アリーナ使用量: 240 B
10 個の要素: 0 1 2 3 4 5 6 7 8 9
```

**動きました。**

`std::vector` の全機能——`push_back`、範囲 for、`std::ranges` のアルゴリズム——が、アリーナの上で使えます。

### `std::list` と `std::map`

**ノードごとに確保するコンテナのほうが、恩恵が大きい。**

```cpp
    using IntList = std::list<int, ga::BumpAllocator<int>>;
    using IntMap  = std::map<int, int, std::less<int>,
                             ga::BumpAllocator<std::pair<const int, int>>>;

    IntList list(arena);
    for (int i = 0; i < 1000; ++i) { list.push_back(i); }

    IntMap map(std::less<int>{}, arena);
    for (int i = 0; i < 1000; ++i) { map[i] = i * 2; }
```

**リバインドが働いています。**

`std::list<int>` に渡したのは `BumpAllocator<int>` ですが、内部では `BumpAllocator<_List_node<int>>` に変換されて使われます。テンプレートのコピーコンストラクタが、それを可能にしています。

### `std::string`

```cpp
    using ArenaString = std::basic_string<char, std::char_traits<char>,
                                          ga::BumpAllocator<char>>;

    ArenaString s(arena);
    s = "これはアリーナ上の文字列です";
```

**第11章で破棄リストを作った動機を思い出してください。** `std::string` を `New<std::string>()` で確保すると、**中身のヒープ確保が漏れる** 問題がありました。

**このアダプタなら、中身もアリーナに載ります。** ただし、デストラクタの問題は残ります(38.6 節)。

---

## 38.4 罠1:再確保がアリーナを食い潰す

**`deallocate` が何もしない** ことの帰結です。

```cpp
int main()
{
    ga::Bump arena(256 * 1024 * 1024);

    std::vector<int, ga::BumpAllocator<int>> v(arena);

    for (int i = 0; i < 1'000'000; ++i) { v.push_back(i); }

    std::println("要素の総バイト数 : {}", ga::FormatBytes(v.size() * sizeof(int)));
    std::println("アリーナ使用量   : {}", ga::FormatBytes(arena.Used()));
    std::println("倍率             : {:.2f}", 
                 static_cast<double>(arena.Used()) / (v.size() * sizeof(int)));
}
```

```
要素の総バイト数 : 3.81 MB
アリーナ使用量   : 11.07 MB
倍率             : 2.90
```

**2.9 倍のメモリを消費しています。**

### なぜか

第30章で見たとおり、`std::vector` は容量を超えると **約 1.5 倍に拡張** します。

```
確保: 1 → 2 → 3 → 4 → 6 → 9 → ... → 1,000,000
```

**古い領域は `deallocate` されますが、`Bump` は返しません。** 積み上がります。

```
合計 = 1 + 2 + 3 + 4 + 6 + ... ≈ 最終サイズの 3 倍
```

### 対策

**対策1:`reserve` する。**

```cpp
    v.reserve(1'000'000);
```

```
アリーナ使用量 : 3.81 MB
倍率           : 1.00
```

**完璧です。** 再確保が起きないので、無駄がゼロになります。

**対策2:第30章の `GrowingArray` を使う。**

そもそも `std::vector` を使わない、という選択です。**上限を当てる必要すらありません。**

**対策3:個別解放できるアロケーターを使う。**

`Tlsf` のアダプタを書けば、`deallocate` が実際に返却します。

```cpp
        void deallocate(T* p, std::size_t) noexcept
        {
            allocator_->Free(p);
        }
```

**ただし、その場合はアリーナの利点(速さ、一括解放)が薄れます。**

> **第30章で見たとおり、`std::vector` の再確保はもともと厄介です。** アリーナに載せると、その厄介さが増幅されます。**`reserve` を書く習慣が、ここでは必須になります。**

---

## 38.5 罠2:ステートフルアロケーターと伝播

**ここが、この章で最も理解しにくい部分です。**

私たちのアロケーターは **状態を持っています**(`arena_` ポインタ)。C++98 のアロケーターは状態を持てない前提でしたが、C++11 で正式に許されました。

**その代わり、複雑な規則が導入されました。**

### 問題の所在

```cpp
ga::Bump arenaA(1024 * 1024);
ga::Bump arenaB(1024 * 1024);

std::vector<int, ga::BumpAllocator<int>> a(arenaA);
std::vector<int, ga::BumpAllocator<int>> b(arenaB);

a = b;              // コピー代入:a のアロケーターはどうなる?
a = std::move(b);   // ムーブ代入:b のメモリを a が持つ?
a.swap(b);          // swap:アロケーターも入れ替わる?
```

**「アロケーターを一緒に運ぶかどうか」を、型で指定します。**

| 型 | 略称 | 制御するもの |
|---|---|---|
| `propagate_on_container_copy_assignment` | POCCA | コピー代入 |
| `propagate_on_container_move_assignment` | POCMA | ムーブ代入 |
| `propagate_on_container_swap` | POCS | swap |
| `select_on_container_copy_construction` | SOCCC | コピー構築 |

**既定値は、最初の3つが `false_type` です。**

### 最も危険な罠:`swap`

```cpp
    a.swap(b);      // POCS = false_type で、アロケーターが等しくない場合
```

**未定義動作です。**

規格には、こう書かれています。

> POCS が `false` で、2つのアロケーターが等しくない場合、swap の動作は未定義。

**なぜか。**

`swap` は、内部のポインタを交換するだけの O(1) 操作です。アロケーターを交換しないと、**`a` が `arenaB` のメモリを持ち、`arenaA` のアロケーターで解放しようとします。**

**別のアリーナに返すことになります。** 第22章で見た「別のプールに返す」事故と、まったく同じです。

### 私たちの選択

```cpp
using propagate_on_container_swap = std::true_type;
```

**`true` にすれば、swap のときにアロケーターも交換されます。** 未定義動作は起きません。

### ムーブ代入の罠

```cpp
    a = std::move(b);      // POCMA = false_type で、アロケーターが等しくない場合
```

**未定義動作ではありませんが、O(n) になります。**

アロケーターが違うと、`b` のメモリを `a` がそのまま持つことができません。**要素を1つずつムーブします。**

```
POCMA = false かつ 不等 : 要素ごとのムーブ(O(n))
POCMA = true            : ポインタとアロケーターを奪う(O(1))
```

**「ムーブは速い」という期待が、静かに裏切られます。**

```cpp
using propagate_on_container_move_assignment = std::true_type;
```

### コピー代入は `false` のままにする

```cpp
using propagate_on_container_copy_assignment = std::false_type;
```

**こちらは `false` が適切です。**

`a = b` としたとき、**`a` は自分のアリーナ(`arenaA`)を使い続けるべき** です。`b` のアリーナに乗り移ったら、驚きます。

```
POCCA = false : a は arenaA に、b の要素をコピーする  ← 自然
POCCA = true  : a が arenaB を使い始める              ← 驚く
```

### `is_always_equal`

```cpp
using is_always_equal = std::false_type;
```

「このアロケーターのインスタンスは、常に等しいか」を表します。

`std::allocator` のような **無状態** のアロケーターは `true` です。すべてのインスタンスが等価なので、コンテナは比較を省略できます。

**私たちは状態を持つので `false` です。** アリーナが違えば、アロケーターも違います。

### まとめ

```cpp
// ステートフルなアロケーターの、実用的な既定値
using propagate_on_container_copy_assignment = std::false_type;   // 自分のアリーナを保つ
using propagate_on_container_move_assignment = std::true_type;    // O(1) ムーブのため
using propagate_on_container_swap            = std::true_type;    // 未定義動作を避けるため
using is_always_equal                        = std::false_type;   // 状態を持つ
```

> **`POCMA = true` にすると、ムーブ代入の後で `a` が `b` のアリーナを使うことになります。** アリーナの寿命が絡む場合、これも驚きの元になります。
>
> **最も安全なのは、「異なるアリーナのコンテナ間で代入・swap をしない」という運用ルール** です。型で強制できないので、規約とレビューで守ることになります。

---

## 38.6 罠3:寿命

```cpp
std::vector<int, ga::BumpAllocator<int>> MakeVector()
{
    ga::Bump arena(1024);                          // ← ローカル変数
    std::vector<int, ga::BumpAllocator<int>> v(arena);
    v.push_back(42);
    return v;                                       // ← アリーナは消える
}
```

**返された `vector` は、破棄されたアリーナのメモリを指しています。**

コンパイラは何も言いません。**`vector` はアロケーターをコピーして持ちますが、それはポインタです。** アリーナ本体の寿命は追跡されません。

### `Reset()` との組み合わせ

もっと見つけにくい形もあります。

```cpp
ga::Bump arena(1024 * 1024);

{
    std::vector<int, ga::BumpAllocator<int>> v(arena);
    v.push_back(42);

    arena.Reset();          // ← ここでリセット

}   // ← v のデストラクタが、リセット済みの領域に対して deallocate を呼ぶ
```

`Bump` の `deallocate` は何もしないので、**この例では実害がありません**。

**しかし、`Tlsf` のアダプタなら破滅します。** 解放済みの領域を、もう一度解放しようとします。

> **アリーナの寿命 > コンテナの寿命** を、必ず守ってください。
>
> 第11章の破棄リストと組み合わせて、`arena.New<ArenaVector>()` のようにコンテナ自体をアリーナに載せる手もあります。そうすれば `Reset()` のときにデストラクタが先に呼ばれます。**ただし、順序に依存する危うい設計です。**

---

## 38.7 罠4:アロケーターが自分自身を使う

第18章で立てた原則を思い出してください。

> **アロケーターのデバッグ機能は、そのアロケーター自身を使ってはならない。**

**この章のアダプタは、その原則を破る道具になります。**

```cpp
class Bump
{
    std::vector<detail::TraceRecord, BumpAllocator<detail::TraceRecord>> traces_;
    //                               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ 絶対にダメ
};
```

`Allocate()` の中でトレースを記録し、その記録が `Allocate()` を呼びます。**無限再帰です。**

**「差し込めるようになった」ことで、この罠を踏みやすくなりました。**

同じことは、第15章の統計や第19章のタグ境界にも当てはまります。**アロケーターの内部データは、必ずシステムのヒープに置いてください。**

---

## 38.8 測る

### `std::vector`

```
                                   push_back 1M    走査(1要素)
std::vector<int>(new)                14.2 ms        0.31 ns
std::vector<int>(BumpAllocator)      11.8 ms        0.31 ns
std::vector<int>(new, reserve)        3.1 ms        0.31 ns
std::vector<int>(Bump, reserve)       2.9 ms        0.31 ns
```

**差は小さい。**

`reserve` した場合、確保は1回だけです。**アロケーターの速度は、ほとんど関係ありません。**

第12章で見た教訓と同じです。

> 確保が 10 倍速くても、初期化に時間がかかるなら、全体では変わらない。

### `std::list`

**ノードごとに確保するコンテナでは、話が変わります。**

```
                                    構築(100万要素)   走査(1要素)
std::list<int>(new)                     42.1 ms         4.20 ns
std::list<int>(BumpAllocator)           11.4 ms         0.95 ns
```

**構築が 3.7 倍、走査が 4.4 倍。**

第32章の 32.7 節で測った数字と一致しています。**ノードが連続配置されるからです。**

```
「リンクリストが遅い」のではない。「ノードが散らばっていると遅い」のだ。
```

### `std::map`

```
                                    構築(10万要素)   検索(1回)
std::map<int,int>(new)                  18.4 ms        142 ns
std::map<int,int>(BumpAllocator)         7.2 ms         88 ns
```

**検索が 1.6 倍速い。** 木のノードが連続配置され、探索中のキャッシュミスが減っています。

---

## 38.9 いつ使うべきか

| 用途 | 評価 |
|---|---|
| **`std::list` / `std::map` / `std::set`** | **◎** ノードが連続配置され、大きく改善 |
| フレーム内の短命な `std::vector` | ○ `reserve` すること |
| フレーム内の短命な `std::string` | ○ |
| **既存コードの配置だけ変えたい** | **○** 型を変えるだけで済む |
| 長寿命の `std::vector` | △ 第30章の `GrowingArray` を検討 |
| `reserve` できない `std::vector` | × メモリを 3 倍食う |
| 異なるアリーナ間で swap する | **×** 規約で禁止すること |
| アロケーター自身の内部データ | **×** 無限再帰 |

### 型名が長くなる問題

```cpp
std::vector<int, ga::BumpAllocator<int>> v(arena);
std::map<int, Enemy, std::less<int>,
         ga::BumpAllocator<std::pair<const int, Enemy>>> m(std::less<int>{}, arena);
```

**書いていられません。** エイリアスを用意します。

```cpp
namespace ga
{
    template <class T>
    using ArenaVector = std::vector<T, BumpAllocator<T>>;

    template <class K, class V, class Cmp = std::less<K>>
    using ArenaMap = std::map<K, V, Cmp, BumpAllocator<std::pair<const K, V>>>;

    using ArenaString = std::basic_string<char, std::char_traits<char>, BumpAllocator<char>>;
}
```

**それでも、`std::vector<T>` と `ArenaVector<T>` は別の型です。**

```cpp
void Process(const std::vector<int>& v);       // ← ArenaVector を渡せない
```

**関数の引数として、相互に渡せません。** 既存のコードベースに導入するとき、これが最大の障害になります。

**次章の `std::pmr` が、この問題を解決します。**

---

## 演習

**演習38-1** `Tlsf` 用のアダプタを書いてください。`deallocate` が実際に解放する場合、38.4 節の倍率はどうなりますか。

**演習38-2** `propagate_on_container_swap` を `false_type` にして、異なるアリーナの `vector` を swap してください。何が起きますか。

**演習38-3** `propagate_on_container_move_assignment` を `false_type` にして、ムーブ代入の時間を測ってください。O(n) になりますか。

**演習38-4** `operator==` を常に `true` を返すように変えてください。何が壊れますか。

**演習38-5** 38.6 節のダングリングの例を実行してください。AddressSanitizer(第31章)は検出しますか。

**演習38-6** `std::unordered_map` に自作アロケーターを渡してください。バケット配列とノードの両方がアリーナに載りますか。

**演習38-7** `select_on_container_copy_construction` を定義し、コピー構築時に別のアリーナを使うようにしてください。どんな用途がありますか。

**演習38-8** `std::vector<std::string, ArenaAllocator>` を作ると、文字列の中身はどこに確保されますか。

---

## 章末チェックリスト

- [ ] `Allocator` 要件の必須4項目を挙げられる
- [ ] リバインドの仕組みと、なぜ必要かを説明できる
- [ ] アダプタを実装し、`vector` / `list` / `map` を動かした 〔v0.27〕
- [ ] 標準コンテナには例外で失敗を伝える必要があることを理解した
- [ ] **再確保でメモリが 2.9 倍になる** ことを確認した
- [ ] `propagate_on_container_swap` が `false` だと **未定義動作** になる理由を説明できる
- [ ] POCMA が `false` だとムーブが O(n) になる理由を説明できる
- [ ] POCCA を `false` にすべき理由を説明できる
- [ ] アリーナの寿命がコンテナより長くなければならない理由を説明できる
- [ ] **アロケーターの内部データに自作アロケーターを使ってはいけない** ことを再確認した
- [ ] `std::list` で 4.4 倍の改善が出ることを測った

---

## 次章の予告

38.9 節で挙げた最後の問題——**型が変わってしまう** ——を解決します。

```cpp
std::vector<int>                          // 標準
std::vector<int, ga::BumpAllocator<int>>  // 自作。別の型
```

**C++17 の `std::pmr` は、アロケーターを「型」ではなく「値」にします。**

```cpp
std::pmr::vector<int> a(&arenaResource);
std::pmr::vector<int> b(&anotherResource);

// a と b は同じ型!
void Process(const std::pmr::vector<int>& v);   // どちらも渡せる
```

仕組みは仮想関数です。`std::pmr::memory_resource` という抽象基底クラスを継承し、`do_allocate` / `do_deallocate` を実装します。

**代償は、仮想呼び出しのコストです。** 第7章で「インライン展開されるからコストゼロ」と書いた `std::expected` の話が、ここで崩れます。実測して、いくら払うのかを確かめます。

そして、標準ライブラリにすでに用意されている `std::pmr::monotonic_buffer_resource` ——**第8章のコラムで予告した、私たちの `Bump` の親戚** ——を使ってみます。

---

> **コラム:STL アロケーターは、なぜ難しいのか**
>
> C++ のアロケーターは、**最も評判の悪い機能の1つ** です。理由を歴史から見てみます。
>
> ---
>
> **出発点の誤り:セグメントモデル**
>
> STL が設計された 1990 年代前半、16 ビットの MS-DOS が現役でした。**メモリは「セグメント」に分かれており、`near` ポインタと `far` ポインタがありました。**
>
> アロケーターの `pointer` 型が `T*` ではなく **`typedef` で差し替え可能** だったのは、このためです。「アロケーターごとに、違うポインタ型を使えるようにする」という設計でした。
>
> **その必要は、フラットなアドレス空間の普及とともに消えました。** しかし、複雑さだけが残りました。
>
> ---
>
> **無状態という前提**
>
> C++98 のアロケーターは、**状態を持たない前提** で設計されました。
>
> 規格には「同じ型のアロケーターは、すべて交換可能でなければならない」という趣旨の要件がありました。つまり、`MyAllocator<int>` のインスタンスはすべて等価。**アリーナへのポインタを持つ、といったことはできません。**
>
> 実装によっては動きましたが、**移植性のあるコードは書けませんでした。**
>
> ---
>
> **C++11 の修正**
>
> C++11 で、大きく改善されました。
>
> - `std::allocator_traits` により、書くべきものが激減した
> - **ステートフルなアロケーターが正式に許された**
> - その代わり、`propagate_on_container_*` という規則が導入された
>
> 38.5 節で見たとおり、**この規則は複雑です。** 4種類あり、既定値が直感に反し、間違えると未定義動作になります。
>
> **「ステートフルを許した代償」** と言えます。
>
> ---
>
> **C++17 の別解:`std::pmr`**
>
> 型パラメータをやめ、**実行時に切り替える** 方式が追加されました。次章の主題です。
>
> **型が同じになるので、`propagate_on_*` の問題も大幅に軽減されます。** その代わり、仮想呼び出しのコストを払います。
>
> ---
>
> **ゲーム業界の別解:EASTL**
>
> EA が公開した EASTL は、**まったく違う設計** を採りました。
>
> - アロケーターはテンプレートパラメータだが、**コンテナが値として保持する**
> - リバインドが要らない(アロケーターは型に依存せず、バイト列を配る)
> - `allocate(size, alignment, offset)` のように、**アラインメントを直接指定できる**
> - デバッグ用の名前を持てる(第15章のタグに相当)
>
> **「標準のアロケーターは、ゲームには使いにくい」という判断から生まれた設計です。**
>
> とくに「アラインメントを指定できない」ことは、SIMD を多用するゲームでは致命的でした。C++17 でようやく `std::allocator` がアライン対応の `operator new` を使うようになりましたが、EASTL は 10 年以上前にこの問題を解いていました。
>
> ---
>
> **教訓**
>
> **標準ライブラリの設計は、その時代の制約を反映しています。** そして、いったん規格に入ったものは簡単には変えられません。
>
> 私たちが第13章で「モジュールを使わない」と決めたとき、「時期の問題であって技術的優劣ではない」と書きました。**アロケーターの複雑さも、同じ種類の問題です。** 設計が悪かったのではなく、前提が変わったのです。
>
> だからこそ、**自分の用途に合うなら、標準に従わない選択もある** ということを覚えておいてください。第30章の `GrowingArray` は、`Allocator` 要件とは無関係に作りました。**そのほうが単純で、速く、安全でした。**
