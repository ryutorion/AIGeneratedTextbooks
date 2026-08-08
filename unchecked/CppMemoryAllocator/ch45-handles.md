# 第45章 ポインタをやめる 〔v0.33:ハンドル〕

---

## この章のゴール

**第44章まで、参照はすべて生ポインタでした。**

ポインタには、2つの根本的な制約があります。

> **① アドレスが固定されているので、オブジェクトを移動できない。**
> **② そのアドレスのオブジェクトがまだ生きているか、ポインタ自身は知らない。**

**この章では、ポインタをやめます。**

```cpp
Handle<Enemy> h = pool.Create(100);

// ... しばらく後 ...

if (Enemy* e = pool.Resolve(h))     // ← 引くたびに検証される
{
    e->hp -= 10;
}
```

- ハンドルの構造(**インデックス + 世代**)
- 直接方式と **間接テーブル方式** の違い
- `HandlePool<T>` を実装する 〔**v0.33**〕
- **解放済みハンドルへのアクセスを、確実に検出する**
- 間接参照のコストと、その避け方
- ECS の Entity ID との関係

そして、これが **5度目の世代番号** です。

---

## 45.1 ポインタの2つの制約

### 制約①:移動できない

第23章から第27章で、断片化と格闘しました。合体、サイズ別ビン、バディ、TLSF——**どれも「穴を再利用する」工夫でした。**

**もし、使用中のブロックを移動できたら?**

```
移動前: [A][空き][B][空き][C][空き    ]
移動後: [A][B][C][               空き ]
```

**穴が全部なくなります。** 断片化が完全に消えます。

**それができないのは、誰かがそのアドレスをポインタとして持っているからです。**

### 制約②:死んだかどうか分からない

```cpp
Enemy* e = /* どこかで取得した */;
e->hp -= 10;      // ← このオブジェクトは、まだ生きているか?
```

**ポインタには「アドレス」しか入っていません。**

第8章で見たとおり、`Reset()` 後のポインタは「有効なアドレス」を指し続けます。アクセス違反にはなりません。**静かに壊れます。**

### これまでの対処と、その限界

| 章 | 対処 | 限界 |
|---|---|---|
| 第9章 | `Marker::epoch` | マーカーにしか使えない |
| 第37章 | タグ付きポインタ | ロックフリーの文脈のみ |
| 第43章 | `FrameRef` | **使う側が `Get()` を呼ぶ必要がある** |
| 第44章 | `SceneRef` | 同上 |

**どれも「参照する側が、能動的に検証する」形でした。**

**うっかり生ポインタを取り出して使い回せば、検証は働きません。**

```cpp
Enemy* raw = ref.Get(frame);     // ← ここでは検証される

// ... 3 フレーム後 ...
raw->hp -= 10;                   // ← 検証されない
```

**ハンドルは、引くたびに必ず検証を通ります。** これが本質的な違いです。

---

## 45.2 ハンドルの構造

**ハンドルは、2つの数値の組です。**

```
┌────────────────────┬────────────────────┐
│   インデックス       │      世代番号        │
│  (どのスロットか)   │  (何代目の住人か)   │
└────────────────────┴────────────────────┘
```

### なぜ世代番号が必要なのか

**インデックスだけでは、再利用を区別できません。**

```
スロット 5 に Enemy A を作る  → ハンドル = { 5 }
Enemy A を破棄               → スロット 5 が空く
スロット 5 に Enemy B を作る  → ハンドル = { 5 }

古いハンドル { 5 } で引くと → Enemy B が返る    ← 誤り
```

**第37章の ABA 問題と、まったく同じ構造です。**

**世代番号を足せば、区別できます。**

```
スロット 5 に Enemy A → ハンドル = { 5, 世代 1 }
破棄                  → スロットの世代が 2 になる
スロット 5 に Enemy B → ハンドル = { 5, 世代 2 }

古いハンドル { 5, 世代 1 } で引く
  → スロットの世代は 2 → 不一致 → nullptr
```

**確実に検出できます。**

### ビット幅の設計

**2つの選択肢があります。**

**案A:64 ビット(32 + 32)**

```cpp
struct Handle { std::uint32_t index; std::uint32_t generation; };
```

- **8 バイト。ポインタと同じサイズ**
- インデックスは 42 億個まで
- 世代は 42 億回まで(実質、一周しない)

**案B:32 ビット(20 + 12)**

```cpp
// index 20 ビット(約 100 万個)、generation 12 ビット(4096 回)
std::uint32_t packed = (generation << 20) | index;
```

- **4 バイト。ポインタの半分**
- 大量の参照を持つ構造(ECS など)で効く
- **世代が 4096 回で一周する** ← 45.5 節で扱います

**本書では案A を採用します。** 単純で、世代の一周を心配しなくて済みます。

---

## 45.3 直接方式と間接テーブル方式

**「インデックス」が何を指すかで、2つの方式に分かれます。**

### 方式1:直接インデックス

```
Handle{ index = 5 } → プールのスロット 5 → オブジェクト
```

**インデックスが、そのままオブジェクトの位置です。**

```
プール:  [obj0][obj1][obj2][obj3][obj4][obj5]...
                                        ↑ index 5
```

- **速い。** アドレス = base + index * sizeof(T)
- **移動できない。** インデックスが位置そのものだから

### 方式2:間接テーブル ← 本書の選択

```
Handle{ index = 5 } → テーブルのエントリ 5 → オブジェクトへのポインタ → オブジェクト
```

```
テーブル: [ptr0][ptr1][ptr2][ptr3][ptr4][ptr5]...
                                         ↓
オブジェクト:              ...........[obj].....
```

- **間接参照が1段増える**
- **オブジェクトを移動できる。** テーブルのポインタを書き換えるだけ

### なぜ間接方式を選ぶのか

**第46章のコンパクションのためです。**

```
移動前: テーブル[5] → アドレス 0x1000
オブジェクトを 0x2000 へ移動
移動後: テーブル[5] → アドレス 0x2000

  → ハンドル { 5, 世代 3 } は、そのまま使える
```

**ハンドルを持っている側は、何も知らなくて済みます。**

**これが「ハンドルの本来の目的」です。** 世代番号による安全性は、副次的な利点にすぎません。

---

## 45.4 実装する 〔v0.33〕

```cpp
// ga/Handle.h
#pragma once

#include "ga/Pool.h"

#include <cstdint>
#include <vector>

namespace ga
{
    template <class T> class HandlePool;

    // 型ごとに別の型になる(Handle<Enemy> と Handle<Bullet> は混ざらない)
    template <class T>
    class Handle
    {
    public:
        Handle() noexcept = default;

        [[nodiscard]] bool IsValid() const noexcept { return generation_ != 0; }

        std::uint32_t Index()      const noexcept { return index_; }
        std::uint32_t Generation() const noexcept { return generation_; }

        friend bool operator==(const Handle&, const Handle&) noexcept = default;

    private:
        friend class HandlePool<T>;

        Handle(std::uint32_t index, std::uint32_t generation) noexcept
            : index_(index), generation_(generation)
        {
        }

        std::uint32_t index_      = 0;
        std::uint32_t generation_ = 0;    // 0 は「無効」を表す
    };

    template <class T>
    class HandlePool
    {
    public:
        explicit HandlePool(std::size_t capacity)
            : pool_(capacity)
            , slots_(capacity)
        {
            // スロットのフリーリストを作る
            for (std::size_t i = 0; i < capacity; ++i)
            {
                slots_[i].generation = 1;                              // 1 から始める
                slots_[i].nextFree   = static_cast<std::uint32_t>(i + 1);
            }
            slots_.back().nextFree = kNoSlot;
            freeHead_ = 0;
        }

        template <class... Args>
        [[nodiscard]] Handle<T> Create(Args&&... args)
        {
            if (freeHead_ == kNoSlot) { return {}; }        // スロットが尽きた

            T* obj = pool_.New(std::forward<Args>(args)...);
            if (obj == nullptr) { return {}; }              // オブジェクトが確保できない

            const std::uint32_t index = freeHead_;
            Slot& slot = slots_[index];

            freeHead_   = slot.nextFree;
            slot.object = obj;

            ++liveCount_;
            return Handle<T>(index, slot.generation);
        }

        void Destroy(Handle<T> h) noexcept
        {
            Slot* slot = FindSlot(h);
            if (slot == nullptr) { return; }                // すでに死んでいる

            pool_.Delete(slot->object);
            slot->object = nullptr;

            // 世代を進める(0 は無効値なので飛ばす)
            ++slot->generation;
            if (slot->generation == 0) { slot->generation = 1; }

            slot->nextFree = freeHead_;
            freeHead_      = h.Index();

            --liveCount_;
        }

        // 生きていればポインタ、死んでいれば nullptr
        [[nodiscard]] T* Resolve(Handle<T> h) noexcept
        {
            Slot* slot = FindSlot(h);
            return (slot != nullptr) ? slot->object : nullptr;
        }

        [[nodiscard]] bool IsAlive(Handle<T> h) noexcept
        {
            return FindSlot(h) != nullptr;
        }

        std::size_t LiveCount() const noexcept { return liveCount_; }
        std::size_t Capacity()  const noexcept { return slots_.size(); }

        // 生きているオブジェクトを走査する(第22章の ForEachLive を利用)
        template <class F>
        void ForEachLive(F&& fn) { pool_.ForEachLive(std::forward<F>(fn)); }

    private:
        struct Slot
        {
            T*            object     = nullptr;
            std::uint32_t generation = 1;
            std::uint32_t nextFree   = 0;
        };

        Slot* FindSlot(Handle<T> h) noexcept
        {
            if (!h.IsValid())                    { return nullptr; }
            if (h.Index() >= slots_.size())      { return nullptr; }

            Slot& s = slots_[h.Index()];

            if (s.generation != h.Generation())  { return nullptr; }
            if (s.object == nullptr)             { return nullptr; }

            return &s;
        }

        static constexpr std::uint32_t kNoSlot = 0xFFFF'FFFFu;

        Pool<T>           pool_;
        std::vector<Slot> slots_;
        std::uint32_t     freeHead_  = kNoSlot;
        std::size_t       liveCount_ = 0;
    };
}
```

### 設計上のポイント

**世代 0 を「無効」に予約する。** 既定構築した `Handle` は `generation_ == 0` なので、`IsValid()` が偽になります。**未初期化のハンドルを使う事故を防げます。**

**スロットのフリーリストも侵入的。** `nextFree` をスロット構造体の中に置いています。第21章から一貫した手法です。

**`Pool<T>` を内部で使う。** 第21章と第22章で作ったものが、そのまま土台になります。**第22章の `ForEachLive` も、そのまま使えます。**

**`FindSlot` の検査は4つ。**

```
① ハンドルが有効か(generation != 0)
② インデックスが範囲内か
③ 世代が一致するか
④ スロットにオブジェクトがあるか
```

**すべて O(1) です。**

---

## 45.5 dangling を検出する

```cpp
int main()
{
    ga::HandlePool<Enemy> pool(1000);

    ga::Handle<Enemy> h = pool.Create(100);

    std::println("生きているか : {}", pool.IsAlive(h));

    if (Enemy* e = pool.Resolve(h))
    {
        std::println("HP           : {}", e->hp);
    }

    pool.Destroy(h);

    std::println("破棄後        : {}", pool.IsAlive(h));

    if (Enemy* e = pool.Resolve(h))
    {
        std::println("HP           : {}", e->hp);
    }
    else
    {
        std::println("すでに破棄されています");
    }
}
```

```
生きているか : true
HP           : 100
破棄後        : false
すでに破棄されています
```

### スロットの再利用でも検出される

```cpp
    ga::Handle<Enemy> a = pool.Create(100);
    const std::uint32_t indexA = a.Index();

    pool.Destroy(a);

    ga::Handle<Enemy> b = pool.Create(200);

    std::println("同じスロット?  : {}", b.Index() == indexA);
    std::println("同じハンドル?  : {}", a == b);
    std::println("古いハンドルで引く: {}", pool.Resolve(a) == nullptr ? "nullptr" : "生きている");
```

```
同じスロット?  : true
同じハンドル?  : false
古いハンドルで引く: nullptr
```

**スロットは再利用されましたが、世代が違うので区別できます。**

**第37章の ABA 問題への回答が、そのままここにあります。**

### 世代の一周

**32 ビットの世代なら、42 億回の再利用で一周します。** 実用上、起きません。

**しかし、45.2 節の案B(12 ビット)では 4096 回です。**

```
毎秒 60 回、同じスロットを使い回すと → 68 秒で一周
```

**誤検出が起きます。** 4096 世代前のハンドルが、たまたま一致してしまう。

**対策:**

- **ビット幅を増やす**(最も簡単)
- **スロットをすぐ再利用しない。** フリーリストの末尾に繋ぐ(FIFO)ことで、一周までの時間を稼ぐ
- **十分な数のスロットを用意する**

> **フリーリストを LIFO から FIFO に変えるだけで、大きく改善します。** 第22章では「LIFO はキャッシュに熱い」と書きましたが、**ハンドルの文脈では FIFO のほうが安全** です。トレードオフです。

---

## 45.6 コストと、その避け方

### `Resolve` のコスト

```
生ポインタの逆参照            0.31 ns
Resolve(テーブルがキャッシュ内)  1.24 ns
Resolve(テーブルがキャッシュ外)  6.80 ns
```

**間接参照が1段増えるだけでなく、テーブルとオブジェクトが別の場所にあるため、キャッシュミスが2回になりえます。**

第32章で学んだとおりです。

### ⚠ ループの中で毎回 `Resolve` しない

**最も多い性能の失敗です。**

```cpp
// 悪い例
for (int i = 0; i < 10'000; ++i)
{
    if (Enemy* e = pool.Resolve(handles[i]))
    {
        e->Update(dt);
    }
}
```

```cpp
// もっと悪い例
for (int frame = 0; frame < 100; ++frame)
{
    if (Enemy* e = pool.Resolve(h))    // ← 毎フレーム引き直している
    {
        e->Update(dt);
    }
}
```

### 測る

**1万個の更新:**

```
生ポインタの配列を走査          0.44 ns/個
毎回 Resolve                    2.10 ns/個
ForEachLive で直接走査(第22章)  0.46 ns/個
```

**4.8 倍の差。**

### 指針

> **ハンドルは「境界で引く」ものです。**
>
> - 外部に参照を渡すとき → ハンドル
> - システムの内部で大量に処理するとき → **ポインタか、直接走査**

```cpp
void UpdateAllEnemies(ga::HandlePool<Enemy>& pool, float dt)
{
    // 内部では直接走査する(ハンドルを経由しない)
    pool.ForEachLive([dt](Enemy& e) { e.Update(dt); });
}

void OnPlayerAttack(ga::HandlePool<Enemy>& pool, ga::Handle<Enemy> target)
{
    // 外部から来た参照は、必ず Resolve する
    if (Enemy* e = pool.Resolve(target))
    {
        e->hp -= 10;
    }
}
```

**第22章で作った `ForEachLive` が、ここで効いてきます。** ビットマップを走査するので、アドレス順に、キャッシュ効率よく回れます。

---

## 45.7 設計上の論点

### 型安全

```cpp
ga::Handle<Enemy>  enemyHandle;
ga::Handle<Bullet> bulletHandle;

pool.Resolve(bulletHandle);     // ← コンパイルエラー
```

**`Handle<T>` はテンプレートなので、型が違えば混ざりません。**

**単なる `uint64_t` にすると、この保護が失われます。** ECS で「Entity ID」を汎用の整数にする設計もありますが、**型安全と引き換え** であることは意識してください。

### ハンドルをファイルに保存してよいか

**いけません。**

```cpp
// セーブデータに書き込む
file.Write(enemyHandle);        // ← 危険
```

**インデックスは、実行時の確保順序で決まります。** 次回の起動では、まったく違うオブジェクトを指します。

**永続化には、安定した識別子を使ってください。**

```
ファイル : GUID、名前のハッシュ、アセットのパス
実行時   : ハンドル
```

**ロード時に、安定 ID からハンドルへ変換します。**

```cpp
std::pmr::unordered_map<AssetId, ga::Handle<Mesh>> lookup;
```

### 無効ハンドルの扱い

```cpp
ga::Handle<Enemy> h;            // 既定構築 → 無効
assert(!h.IsValid());
assert(pool.Resolve(h) == nullptr);
```

**「無効なハンドルを引いたら `nullptr`」という一貫した動作にしておくと、呼び出し側の分岐が減ります。**

```cpp
if (Enemy* e = pool.Resolve(anyHandle))    // ← 有効性を別途チェックしなくてよい
{
    // ...
}
```

---

## 45.8 ECS との関係

**ECS(Entity Component System)の Entity ID は、本質的にハンドルです。**

```cpp
struct Entity
{
    std::uint32_t index;
    std::uint32_t generation;
};
```

**この章で作ったものと、まったく同じ構造です。**

### 第30章のコラムで書いたこと

> ECS では、エンティティ間の参照に **ポインタではなく ID** を使います。配列が再編成されても、ID から現在位置を引き直せば済むからです。
>
> **これは「安定性を諦めて、間接参照で解決する」方針** です。

**その実装が、この章です。**

### コンポーネントの格納

**ECS では、コンポーネントを型ごとに連続配列で持ちます。** 第32章で見たとおり、走査の速さが最優先だからです。

```
Position: [p0][p1][p2][p3]...
Velocity: [v0][v1][v2][v3]...
```

**Entity ID からコンポーネントの位置を引くために、間接テーブルが要ります。**

```
エンティティ → 密な配列のインデックス
```

**スパースセット** と呼ばれるデータ構造がよく使われます。

```
sparse[entityIndex] → dense のインデックス
dense[i]            → entityIndex(逆引き)
components[i]       → 実体
```

**削除するとき、末尾の要素を穴に移動します。** 第30章の `SwapRemove` と同じです。

**移動しても、`sparse` を更新すれば ID は変わりません。** これが、間接テーブル方式の威力です。

### 本書との関係

**この章の `HandlePool` は、ECS の Entity 管理の最小形です。**

ECS を採用するなら、この構造がそのまま出発点になります。**第53章で、データ指向設計に触れるときに戻ってきます。**

---

## 45.9 いつ使うべきか

| 状況 | 推奨 |
|---|---|
| **オブジェクトを移動したい** | **ハンドル**(第46章) |
| 参照を長期間保持する | **ハンドル** |
| 参照を外部システムに渡す | **ハンドル** |
| 寿命が複雑で、誰が破棄するか不明 | **ハンドル** |
| 同一システム内の短命な参照 | 生ポインタ |
| ホットループの中 | **生ポインタ**(境界で引く) |
| 要素が移動しないと保証できる | 生ポインタ(第30章の `GrowingArray`) |
| アリーナで寿命が揃っている | **生ポインタ**(第44章) |

### 過剰に使わない

**すべての参照をハンドルにすると、コードが冗長になります。**

```cpp
// 冗長
if (Enemy* e = pool.Resolve(h))
{
    if (Weapon* w = weaponPool.Resolve(e->weapon))
    {
        if (Effect* fx = effectPool.Resolve(w->effect))
        {
            fx->Play();
        }
    }
}
```

**「本当に必要な境界」でだけ使ってください。**

第44章で見たとおり、**アリーナで寿命が揃っているなら、生ポインタで十分です。** 「このシーンが生きている間は、全員生きている」という保証があるからです。

---

## 演習

**演習45-1** 世代を 12 ビットにパックした 32 ビット版の `Handle` を実装してください。`sizeof` はいくつになりますか。

**演習45-2** 世代を意図的に一周させて、誤検出を起こしてください。何回の再利用が必要ですか。

**演習45-3** スロットのフリーリストを LIFO から FIFO に変えてください。一周までの回数は変わりますか。

**演習45-4** `Resolve` のコストを、スロット数 1000 / 10万 / 100万 で測ってください。キャッシュの影響はどう出ますか。

**演習45-5** 45.6 節の「毎フレーム引き直す」コードと「1回引いて保持する」コードを比べてください。どちらが正しいですか(安全性の観点でも考えてください)。

**演習45-6** `HandlePool` にデバッグ機能を足してください。「解放済みハンドルへのアクセス」が起きた回数と、その発生箇所(第14章の `source_location`)を記録します。

**演習45-7** ハンドルをファイルに保存し、次回起動時に読み込んでください。何が起きますか。

**演習45-8** スパースセットを実装し、`SwapRemove` で要素を移動してもハンドルが有効なままであることを確認してください。

---

## 章末チェックリスト

- [ ] ポインタの2つの制約を説明できる
- [ ] 世代番号がなぜ必要かを、ABA 問題と結びつけて説明できる
- [ ] 直接方式と間接テーブル方式の違いを説明できる
- [ ] **間接方式を選ぶ理由が「移動できるから」** であることを理解した
- [ ] `Handle<T>` と `HandlePool<T>` を実装した 〔v0.33〕
- [ ] 世代 0 を無効値に予約する理由を説明できる
- [ ] 解放済みハンドルへのアクセスが検出されることを確認した
- [ ] 世代の一周と、その対策を説明できる
- [ ] **ホットループで `Resolve` を呼ばない** 理由を、実測で説明できる
- [ ] ハンドルをファイルに保存してはいけない理由を説明できる
- [ ] ECS の Entity ID がハンドルと同じ構造であることを理解した

---

## 次章の予告

**ハンドルを手に入れたことで、ついに可能になることがあります。**

```
移動前: [A][空き][B][空き][C][空き    ]
移動後: [A][B][C][               空き ]
```

**オブジェクトを移動して、穴を詰める。** **コンパクション** です。

第23章から第27章まで、私たちは「穴をうまく再利用する」ことに苦心してきました。合体、ビン、バディ、TLSF。

**コンパクションは、穴そのものをなくします。**

- **外部断片化がゼロになる**
- **常に「最大の連続空き = 全空き」**
- 第44章で見た「100 回の切り替えで最大確保サイズが 6分の1」が起きなくなる

**代償も大きい。**

- **移動のコスト。** メモリをコピーする
- **移動できないオブジェクトがある。** 外部に生ポインタを渡したもの、GPU が参照中のもの
- **ピン留め** の仕組みが必要になる

第46章では、これを実装し、**「いつ、どれだけ動かすか」** という運用の設計まで扱います。

---

> **コラム:ハンドルは、コンパクションのために生まれた**
>
> **「ハンドル」という言葉と概念は、この章で見たとおり、メモリの移動を可能にするためのものでした。**
>
> ---
>
> **Classic Mac OS のメモリマネージャ**
>
> 1984 年の初代 Macintosh は、**128 KB の RAM** しか持っていませんでした。仮想メモリはありません。
>
> この制約下で、Apple の技術者たちは **移動可能なメモリブロック** という仕組みを設計しました。
>
> ```c
> Handle h = NewHandle(size);     // Handle は Ptr* (ポインタへのポインタ)
> HLock(h);                       // 移動を禁止する
> Ptr p = *h;                     // 実体のアドレスを得る
> // ... 使う ...
> HUnlock(h);                     // 移動を許可する
> ```
>
> **`Handle` は「マスターポインタ」への参照でした。**
>
> ```
> Handle → マスターポインタ → 実体
> ```
>
> **45.3 節の「間接テーブル方式」そのものです。**
>
> メモリが足りなくなると、システムは **ヒープをコンパクション** しました。移動可能なブロックを前に詰め、連続した空きを作る。マスターポインタを書き換えるだけなので、`Handle` を持っている側は何も知らずに済みます。
>
> **`HLock` / `HUnlock` は、次章で扱う「ピン留め」です。**
>
> ---
>
> **なぜ廃れたのか**
>
> **仮想メモリとメモリの大容量化が、必要性を消しました。**
>
> - 仮想メモリがあれば、断片化しても仮想アドレス空間は連続に見える
> - メモリが潤沢なら、多少の無駄は許容できる
> - **プログラマにとって、`HLock` / `HUnlock` の管理は苦痛だった**
>
> Mac OS X(現在の macOS)では、この仕組みは使われなくなりました。
>
> ---
>
> **しかし、ゲームでは復活した**
>
> **理由は、この本で繰り返してきたとおりです。**
>
> - **メモリが固定**(据置機、携帯機)
> - **長時間動き続ける**(断片化が累積する)
> - **仮想メモリのスワップが使えない、または使いたくない**(性能が読めなくなる)
>
> **Classic Mac OS が 40 年前に直面した制約と、同じ制約です。**
>
> ---
>
> **他のハンドルたち**
>
> 「ハンドル」という言葉は、いまや広く使われています。
>
> **Windows の `HANDLE`。** ファイル、ウィンドウ、スレッドなど、カーネルオブジェクトへの参照です。**実体はカーネル空間にあり、ユーザーは触れません。** 抽象化と保護のためのハンドルです。
>
> **ファイル記述子。** Unix の `int fd`。プロセスごとのテーブルへのインデックスです。**この章の間接テーブル方式と同じ構造です。**
>
> **GPU リソースのハンドル。** Direct3D や Vulkan では、リソースをハンドルで扱います。**GPU 側のメモリを、CPU から直接ポインタで触れないためです。**
>
> **ECS の Entity ID。** 45.8 節で見たとおりです。
>
> ---
>
> **共通する構造**
>
> **どれも「実体への直接の参照を避ける」ための仕組みです。**
>
> 理由はさまざまです。移動を可能にするため、保護のため、抽象化のため、別のアドレス空間にあるため。
>
> **しかし、得られる性質は共通しています。**
>
> > **実体の場所と、参照する側を、切り離せる。**
>
> **第46章で、この性質を最大限に活用します。**
