# 第22章 プールを実戦仕様にする 〔v0.15〕

---

## この章のゴール

`Pool<T>` は速く、断片化もしません。しかし、**壊し方があまりに簡単** です。

```cpp
pool.Delete(p);
pool.Delete(p);      // 二重解放
```

```cpp
poolB.Delete(poolA.New());   // 別のプールに返す
```

どちらも、いまは何のエラーも出ません。そして数フレーム後、まったく無関係な場所でゲームが壊れます。

この章では、それを止めます。

- 3つの事故を実演し、何が起きるか目で見る
- **所属チェック** を O(1) で行う
- **二重解放の検出** を、使用中ビットマップで行う 〔**v0.15**〕
- そのビットマップが、**第21章で見た走査の劣化まで解決してしまう** ことを発見する
- フリーリスト方式とビットマップ方式を正面から比較する

---

## 22.1 3つの事故を実演する

### 事故1:二重解放

```cpp
int main()
{
    ga::Pool<Particle> pool(10);

    Particle* p = pool.New();
    std::println("p = {}", static_cast<void*>(p));

    pool.Delete(p);
    pool.Delete(p);          // ← 2回目

    Particle* a = pool.New();
    Particle* b = pool.New();

    std::println("a = {}", static_cast<void*>(a));
    std::println("b = {}", static_cast<void*>(b));
    std::println("a == b ? {}", a == b);
}
```

```
p = 0x1f3a8c2b3c0
a = 0x1f3a8c2b3c0
b = 0x1f3a8c2b3c0
a == b ? true
```

**同じアドレスが2回返りました。**

理由はフリーリストの構造にあります。

```
1回目の Delete:  free_ → [p] → [空きブロック群...]
2回目の Delete:  free_ → [p] → [p] → ???
```

`p` の中身に「次は `p` の次だったもの」が書かれ、それがもう一度 `p` を指すノードで上書きされます。**リストが自分自身を含む形になります。**

結果、`a` と `b` は同じ 32 バイトを共有します。パーティクル A の座標を書くと、パーティクル B の座標が変わる。**再現性の低い、原因の見えないバグ** の典型です。

さらに `live_` が 2 回減るので、カウンタも狂います。

### 事故2:別のプールに返す

```cpp
    ga::Pool<Particle> poolA(10);
    ga::Pool<Particle> poolB(10);

    Particle* p = poolA.New();
    poolB.Delete(p);              // ← 間違い

    Particle* q = poolB.New();
    std::println("q は poolA の領域? {}", q == p);
```

```
q は poolA の領域? true
```

**`poolB` が、`poolA` の領域を配り始めました。**

`poolA` はそのブロックを「使用中」だと思っています。`poolB` もそのブロックを配ります。2つのプールが同じメモリを別々に管理する状態になり、以降は何が起きても不思議ではありません。

さらに、`poolB` の `live_` は減算されているので、**存在しないブロックを1つ持っていることになります**。最終的には `poolB` の容量を超えて配ろうとします。

### 事故3:プールと無関係なポインタ

```cpp
    Particle stackParticle;
    pool.Delete(&stackParticle);   // スタック上のオブジェクト
```

フリーリストの先頭が、**スタック領域を指します**。関数から戻った瞬間、その領域は別の用途に使われます。次の `New()` は、生きているスタックフレームの真ん中を返すでしょう。

---

## 22.2 所属チェック

3つの事故のうち2つは、「そのポインタは本当にこのプールのものか」を確かめれば防げます。

プールは連続した領域を持っているので、**2つの比較と1つの剰余** で判定できます。

```cpp
    bool Owns(const void* p) const noexcept
    {
        const auto* b = static_cast<const std::byte*>(p);

        // ① 範囲の中か
        if (b < base_ || b >= base_ + capacity_ * kBlockSize) { return false; }

        // ② ブロックの先頭に揃っているか
        return (static_cast<std::size_t>(b - base_) % kBlockSize) == 0;
    }
```

**O(1) です。** フリーリストを走査する必要はありません。

### ②が必要な理由

範囲チェックだけでは不十分です。

```cpp
Particle* p = pool.New();
pool.Delete(p + 1);         // ← 範囲内だが、ブロックの途中かもしれない
```

`kBlockSize` が 32 で `sizeof(Particle)` が 32 なら `p + 1` も境界に乗りますが、一般には乗りません。ずれた位置をフリーリストに繋げば、以降のブロック配置がすべてずれます。

### 剰余算は重くないのか

`%` は除算命令を使うので重い——第6章でそう述べました。しかしここでは問題になりません。

**`kBlockSize` は `constexpr` です。** コンパイラは定数除算を、乗算とシフトの組み合わせに置き換えます。除算命令は生成されません。

さらに `kBlockSize` が2の冪なら、単なるビット論理積になります。

> **定数で割る場合、心配は不要です。** 重いのは「実行時に決まる値で割る」場合です。

---

## 22.3 二重解放をどう検出するか

事故1は、所属チェックでは防げません。`p` は正しくこのプールのものだからです。

**「そのブロックはいま使用中か」を知る必要があります。**

### 案A:フリーリストを走査する

```cpp
for (auto* n = free_; n; n = n->next)
{
    if (n == node) { /* すでに空き = 二重解放 */ }
}
```

正確ですが **O(n)** です。空きブロックが1万個あれば1万回比較します。`Delete` が O(1) でなくなるのは受け入れがたい。

### 案B:ブロックにマジックナンバーを書く

解放時に、ブロックの先頭に特別な値を書いておきます。

```cpp
struct FreeNode
{
    FreeNode*     next;
    std::uint64_t magic;   // 0xF2EE'F2EE'F2EE'F2EE
};
```

`Delete` のとき、すでにマジックナンバーが入っていれば二重解放と判定します。**O(1)、追加メモリは 8 バイト × ブロック数** です。

弱点は **偽陽性** です。ユーザーがたまたま同じ値を書いていたら、誤検出します。可能性は低いですが、ゼロではありません。

### 案C:使用中ビットマップ ← 採用

ブロックごとに1ビット持ちます。

```cpp
std::vector<std::uint64_t> usedBits_;   // 64 ブロックで 1 ワード
```

| | |
|---|---|
| 判定 | **O(1)、確実**(偽陽性なし) |
| メモリ | ブロック数 ÷ 8 バイト |
| 1万ブロック | **1.25 KB** |

320 KB のプールに対して 1.25 KB。**0.4% の追加コスト** です。

```cpp
    bool IsUsed(std::size_t i) const noexcept
    {
        return (usedBits_[i >> 6] >> (i & 63)) & 1u;
    }

    void SetUsed(std::size_t i) noexcept
    {
        usedBits_[i >> 6] |= (std::uint64_t{1} << (i & 63));
    }

    void ClearUsed(std::size_t i) noexcept
    {
        usedBits_[i >> 6] &= ~(std::uint64_t{1} << (i & 63));
    }
```

`i >> 6` は `i / 64`、`i & 63` は `i % 64` です。2の冪なので、シフトとマスクで書けます。

---

## 22.4 実装する 〔v0.15〕

### エラーの伝え方

第9章で立てた原則を思い出してください。

> **回復できるものはエラー、回復できないものはバグ。**

二重解放も所属違反も、**プログラムのバグ** です。呼び出し側が実行時に対処できることはありません。したがって `assert` 系の扱いにします。

ただし、ライブラリとして使われることを考えて、コールバックを差し込めるようにします。

```cpp
namespace ga
{
    enum class PoolError
    {
        NotOwned,     // このプールのポインタではない
        DoubleFree,   // すでに解放されている
    };

    constexpr const char* ToString(PoolError e) noexcept
    {
        switch (e)
        {
        case PoolError::NotOwned:   return "NotOwned";
        case PoolError::DoubleFree: return "DoubleFree";
        }
        return "?";
    }

    using PoolErrorCallback = void (*)(PoolError, const void*, void* user) noexcept;
}
```

### Release でも検査を残す

第17章のガードバイトは、Debug 専用にしました。コストが4倍だったからです。

**今回は違います。**

| 検査 | コスト |
|---|---|
| 所属チェック | 比較2回 + 定数除算1回 |
| 二重解放チェック | メモリ読み1回 + ビット演算 |

**どちらも 1 ナノ秒未満** です。そして防げる事故の深刻さは、第17章のはみ出し検出と同等かそれ以上。

> **この2つは Release でも常時有効にします。**
>
> デバッグ機能を一律に「Debug のみ」と決めるのではなく、**コストと効果を個別に判断する**。第51章で整理する考え方の先取りです。

### 本体

```cpp
    void Deallocate(T* p) noexcept
    {
        if (p == nullptr) { return; }

        if (!Owns(p))
        {
            ReportError(PoolError::NotOwned, p);
            return;                       // 壊すより、何もしないほうがまし
        }

        const std::size_t index = IndexOf(p);

        if (!IsUsed(index))
        {
            ReportError(PoolError::DoubleFree, p);
            return;
        }

        ClearUsed(index);

#if GA_ENABLE_MEMORY_PATTERN
        FillPattern(p, kBlockSize, kPatternFreed);
#endif

        free_ = std::construct_at(reinterpret_cast<detail::FreeNode*>(p),
                                  detail::FreeNode{ free_ });
        --live_;
    }
```

**エラー時に「何もしない」** ことを選んでいます。第9章の `Rewind()` と同じ判断です。壊れた状態で処理を続けるより、無視するほうが被害が小さい。

`Allocate` 側も対応させます。

```cpp
    [[nodiscard]] T* Allocate() noexcept
    {
        if (free_ == nullptr) { return nullptr; }

        detail::FreeNode* node = free_;
        free_ = node->next;

        const std::size_t index = IndexOf(node);
        SetUsed(index);

        ++live_;
        if (live_ > peak_) { peak_ = live_; }

#if GA_ENABLE_MEMORY_PATTERN
        FillPattern(node, kBlockSize, kPatternAllocated);
#endif

        return reinterpret_cast<T*>(node);
    }
```

第16章の塗りつぶしも入れました。解放済みブロックを読むバグが、`0xDDDDDDDD` で見えるようになります。

### 検出させてみる

```cpp
void OnPoolError(ga::PoolError e, const void* p, void*) noexcept
{
    std::println("[プールエラー] {} : {}", ToString(e), p);
}

int main()
{
    ga::Pool<Particle> poolA(10);
    ga::Pool<Particle> poolB(10);

    poolA.SetErrorCallback(&OnPoolError);
    poolB.SetErrorCallback(&OnPoolError);

    Particle* p = poolA.New();

    poolB.Delete(p);        // 事故2
    poolA.Delete(p);
    poolA.Delete(p);        // 事故1

    Particle stackParticle;
    poolA.Delete(&stackParticle);   // 事故3

    std::println("live = {}", poolA.Live());
}
```

```
[プールエラー] NotOwned : 0x1f3a8c2b3c0
[プールエラー] DoubleFree : 0x1f3a8c2b3c0
[プールエラー] NotOwned : 0x9a3fcff5a8
live = 0
```

**3つの事故がすべて検出され、しかもプールの状態は壊れていません。** `live` は正しく 0 です。

---

## 22.5 思わぬ副産物:走査の順序が戻る

ビットマップを導入したことで、**第21章の最後で見た問題が解決できます**。

21.8 節では、ランダム順に解放して再確保すると、ポインタ配列の並びが乱れ、走査が7倍遅くなりました。

**ビットマップがあれば、ポインタ配列に頼る必要がありません。**

```cpp
    // 使用中のブロックを、アドレス順に走査する
    template <class F>
    void ForEachLive(F&& fn)
    {
        for (std::size_t w = 0; w < usedBits_.size(); ++w)
        {
            std::uint64_t bits = usedBits_[w];

            while (bits != 0)
            {
                const int b = std::countr_zero(bits);   // 最下位の 1 の位置
                bits &= bits - 1;                       // その 1 を落とす

                const std::size_t index = (w << 6) + static_cast<std::size_t>(b);
                fn(*reinterpret_cast<T*>(base_ + index * kBlockSize));
            }
        }
    }
```

### 2つのビット演算

**`std::countr_zero(bits)`**(C++20、`<bit>`)は、最下位から連続する 0 の個数を返します。つまり **最下位の 1 が立っている位置** です。

**`bits &= bits - 1`** は、最下位の 1 を落とす古典的なイディオムです。

```
bits     = 0b0101'1000
bits - 1 = 0b0101'0111
AND      = 0b0101'0000    ← 最下位の 1 が消えた
```

この2つを組み合わせると、**立っているビットだけを順に、しかも空きを飛ばして** 巡回できます。1ワードで 64 ブロックを一度に扱えるので、生存率が低いときは特に効率的です。

### 効果を測る

第21章と同じ実験をします。ランダム順に全解放してから再確保し、走査速度を比べます。

```
ポインタ配列で走査(初回、昇順)       : 0.41 ns/個
ポインタ配列で走査(再確保後、乱れた) : 2.87 ns/個
ForEachLive で走査(再確保後)         : 0.46 ns/個
```

**順序の乱れの影響を、完全に受けません。**

ビットマップは常にアドレス順に走査するので、ポインタ配列がどれだけ乱れていても関係ありません。

### 生存率が低いとき

```
生存率 100% : 0.46 ns/個
生存率  50% : 0.52 ns/個
生存率  10% : 0.94 ns/個
生存率   1% : 3.10 ns/個
```

生存率が下がると、1個あたりのコストは上がります。空のワードを読み飛ばすコストが、生きている要素数で割られるためです。

ただし **全体の時間は減っています**(1% なら 100 個しか処理しないので)。1個あたりの数字に惑わされないでください。

> **検査のために入れたビットマップが、性能改善の道具にもなりました。**
>
> こういうことは、よく起きます。デバッグのための構造が、たまたま別の用途に使えることがある。逆に、性能のための構造がデバッグを助けることもあります。**設計は、目的を1つに絞りすぎないほうが得なことがあります。**

---

## 22.6 ビットマップだけで作ってみる

ここまで来ると、当然の疑問が湧きます。

> **ビットマップがあるなら、フリーリストは要らないのでは?**

試してみましょう。フリーリストを持たず、空きビットを探して確保する版です。

```cpp
// ga/BitmapPool.h
template <class T>
class BitmapPool
{
public:
    explicit BitmapPool(std::size_t capacity)
        : buffer_(capacity * kBlockSize + kBlockAlign)
        , usedBits_((capacity + 63) / 64, 0)
        , capacity_(capacity)
    {
        const auto raw = reinterpret_cast<std::uintptr_t>(buffer_.data());
        base_ = reinterpret_cast<std::byte*>(AlignUp(raw, kBlockAlign));

        // 容量を超える部分のビットは、はじめから 1 にしておく
        for (std::size_t i = capacity_; i < usedBits_.size() * 64; ++i)
        {
            usedBits_[i >> 6] |= (std::uint64_t{1} << (i & 63));
        }
    }

    [[nodiscard]] T* Allocate() noexcept
    {
        const std::size_t n = usedBits_.size();

        for (std::size_t k = 0; k < n; ++k)
        {
            const std::size_t w = (cursor_ + k) % n;

            const std::uint64_t bits = usedBits_[w];
            if (bits == ~std::uint64_t{0}) { continue; }   // このワードは満杯

            const int b = std::countr_one(bits);           // 最初の 0 の位置
            usedBits_[w] |= (std::uint64_t{1} << b);

            cursor_ = w;
            ++live_;

            const std::size_t index = (w << 6) + static_cast<std::size_t>(b);
            return reinterpret_cast<T*>(base_ + index * kBlockSize);
        }

        return nullptr;
    }

    void Deallocate(T* p) noexcept
    {
        if (p == nullptr) { return; }
        if (!Owns(p))     { ReportError(PoolError::NotOwned, p);   return; }

        const std::size_t index = IndexOf(p);
        if (!IsUsed(index)) { ReportError(PoolError::DoubleFree, p); return; }

        usedBits_[index >> 6] &= ~(std::uint64_t{1} << (index & 63));
        --live_;
    }
};
```

### `std::countr_one`

`bits` の最下位から連続する 1 の個数を返します。これは **最初に 0 が現れる位置** です。

```
bits = 0b0000'0111  → countr_one = 3 → ビット3が空き
bits = 0b1111'1110  → countr_one = 0 → ビット0が空き
```

空きを1命令で見つけられます(MSVC は `tzcnt` 系の命令に落とします)。

### `cursor_` の役割

前回見つけたワードから探索を再開します。第20章で見た **next fit** と同じ発想です。

先頭から探し直すと、前半が埋まっているときに毎回同じワードを読み飛ばすことになります。

---

## 22.7 比較する

同じ条件で両方を測ります。

### 確保・解放の速度

```
                        ほぼ空  半分埋まる  95% 埋まる
Pool (フリーリスト)      2.9      2.9        2.9      ns
BitmapPool              3.6      4.1       14.8      ns
```

**フリーリストは、埋まり具合に影響されません。** 先頭を取るだけだからです。

**ビットマップは、満杯に近づくと遅くなります。** 空きビットを探して、満杯のワードを読み飛ばす必要があるためです。95% で 5 倍遅くなっています。

### 走査

```
                        生存率100%  生存率10%
Pool::ForEachLive          0.46      0.94    ns/個
BitmapPool::ForEachLive    0.46      0.94    ns/個
```

**同じです。** どちらも同じビットマップを持っているので当然です。

### メモリ

```
容量 10000 の Pool<Particle>(32 バイト):

  ブロック領域    : 320.0 KB
  ビットマップ    :   1.25 KB
  フリーリスト    :   0 バイト(空きブロック内に埋め込み)
  ────────────────────────────
  Pool 合計       : 321.25 KB
  BitmapPool 合計 : 321.25 KB
```

**同じです。** フリーリストは追加メモリを使わないので、差が出ません。

### 順序の性質

| | 確保されるブロックの順序 |
|---|---|
| Pool(フリーリスト) | **解放の逆順**(LIFO) |
| BitmapPool | **常に若い番号から**(アドレス順) |

ここに実質的な違いがあります。

**フリーリストは、直前に解放したブロックを返します。** キャッシュに残っている可能性が高く、確保直後のアクセスは速い。

**ビットマップは、常に前詰めします。** 生きているブロックがメモリの前方に集まるので、`ForEachLive` の走査範囲が狭くなります。長時間動かしたときの挙動が安定します。

### まとめ

| | Pool(フリーリスト) | BitmapPool |
|---|---|---|
| 確保 | **O(1)、常に一定** | O(容量/64)、満杯付近で遅い |
| 解放 | O(1) | O(1) |
| 走査 | O(容量/64) | O(容量/64) |
| 追加メモリ | 検査用ビットマップのみ | ビットマップのみ |
| 確保順 | LIFO(キャッシュに熱い) | アドレス順(前詰め) |
| 実装の複雑さ | やや高い(2つの構造) | **低い(1つだけ)** |

### どちらを選ぶか

**確保・解放が頻繁なら、フリーリスト。** 満杯付近でも速度が落ちないのは大きな利点です。弾やパーティクルのように、毎フレーム大量に生成・破棄されるものに向きます。

**単純さを取るなら、ビットマップ。** 構造が1つしかないので、バグが入る余地が少ない。確保頻度が低く、走査が主体なら、こちらで十分です。

**本書は `Pool`(フリーリスト + 検査用ビットマップ)を採用します。** 確保の最悪時間が一定であることを重視しました。第2章から一貫して、平均より最悪値を見る方針です。

---

## 22.8 完成コードの要点

```cpp
template <class T>
class Pool
{
public:
    static constexpr std::size_t kBlockSize  = /* 第21章のまま */;
    static constexpr std::size_t kBlockAlign = /* 第21章のまま */;

    explicit Pool(std::size_t capacity);

    [[nodiscard]] T* Allocate() noexcept;
    void             Deallocate(T* p) noexcept;

    template <class... Args> [[nodiscard]] T* New(Args&&... args);
    void                                      Delete(T* p) noexcept;

    // --- v0.15 ---
    bool Owns(const void* p) const noexcept;
    void SetErrorCallback(PoolErrorCallback cb, void* user = nullptr) noexcept;

    template <class F> void ForEachLive(F&& fn);

    std::size_t Capacity() const noexcept;
    std::size_t Live()     const noexcept;
    std::size_t Peak()     const noexcept;

private:
    std::size_t IndexOf(const void* p) const noexcept
    {
        return static_cast<std::size_t>(static_cast<const std::byte*>(p) - base_) / kBlockSize;
    }

    bool IsUsed(std::size_t i)    const noexcept;
    void SetUsed(std::size_t i)         noexcept;
    void ClearUsed(std::size_t i)       noexcept;

    void ReportError(PoolError e, const void* p) const noexcept
    {
        if (errorCallback_) { errorCallback_(e, p, errorUser_); }
        else                { assert(false && "プールの使い方が誤っています"); }
    }

    std::vector<std::byte>      buffer_;
    std::vector<std::uint64_t>  usedBits_;
    std::byte*                  base_ = nullptr;
    detail::FreeNode*           free_ = nullptr;
    std::size_t                 capacity_ = 0;
    std::size_t                 live_     = 0;
    std::size_t                 peak_     = 0;
    PoolErrorCallback           errorCallback_ = nullptr;
    void*                       errorUser_     = nullptr;
};
```

---

## 演習

**演習22-1** `Pool` のデストラクタで、まだ生きているオブジェクトのデストラクタを呼ぶようにしてください。`ForEachLive` が使えますか。

**演習22-2** 22.3 の案B(マジックナンバー)を実装し、ビットマップ版と比べてください。偽陽性が起きる状況を作れますか。

**演習22-3** `ForEachLive` に、途中で走査を打ち切れる機能(戻り値が `false` なら中断)を追加してください。

**演習22-4** `BitmapPool::Allocate` の `cursor_` を使わない(常に先頭から探す)版を作り、速度を比べてください。95% 埋まったときの差はどれくらいですか。

**演習22-5** 生存率 1% で `ForEachLive` を呼ぶとき、走査するワードのほとんどが 0 です。これを速くする方法を考えてください(ヒント:階層化)。

**演習22-6** 二重解放を検出したとき、「何もしない」以外の選択肢を3つ挙げ、それぞれの長短を述べてください。

**演習22-7** `Owns` の剰余算が、コンパイラによってどう変換されるか確認してください(逆アセンブルを見ます)。`kBlockSize` が 32 のときと 33 のときで違いますか。

---

## 章末チェックリスト

- [ ] 二重解放でフリーリストが輪になることを実演した
- [ ] 別のプールに返すと何が起きるかを実演した
- [ ] 所属チェックを O(1) で実装した
- [ ] 二重解放の検出方法3案を比較し、選んだ理由を説明できる
- [ ] **この2つの検査を Release でも残す** 判断の根拠を説明できる
- [ ] `ForEachLive` で走査順序の劣化が解決することを確認した
- [ ] `std::countr_zero` と `bits &= bits - 1` の働きを説明できる
- [ ] フリーリスト方式とビットマップ方式の長短を比較できる

---

## 次章の予告

プールは強力ですが、**サイズが1種類** という制約は重い。第21章で見たとおり、型ごとにプールを作ると、メモリを融通できなくなります。

第23章から、**可変サイズの個別解放** に踏み込みます。第20章で紙の上でやったことを、コードにします。

- ブロックごとにヘッダを持ち、サイズを記録する
- 空きブロックをフリーリストで繋ぐ
- first fit で探す
- 見つかった穴が大きければ、分割する

そして、**第19章で作った可視化が、ついに面白くなります**。

```
第19章(バンプ):
████████████████████████░░░░░░░░

第23章(フリーリスト、合体なし):
███░█░███░██░░█░██░█░░███░█░░░░░
```

穴だらけの絵と、跳ね上がる断片化指標。第20章で紙の上で見た現象が、実物のアロケーターの上で再現されます。

そして、それを第24章で直します。

---

> **コラム:二重解放は、なぜ「攻撃」になるのか**
>
> 22.1 節の事故1を、もう一度見てください。二重解放によって、**同じアドレスが2回配られました**。
>
> これは単なるバグではありません。**攻撃者にとっては、贈り物です。**
>
> ---
>
> 攻撃者が「同じメモリを指す2つのオブジェクト」を作れたとします。片方を型 A として、もう片方を型 B として扱えます。
>
> 型 A に整数を書き込み、型 B のポインタとして読み出す——これができれば、**任意のアドレスを指すポインタを作れます**。そこから任意コード実行まで、あと数歩です。
>
> このような攻撃は **type confusion** と呼ばれ、ブラウザやカーネルの脆弱性報告に頻繁に登場します。二重解放や解放後使用(use-after-free)は、その入り口になります。
>
> ---
>
> さらに古典的なのが、**フリーリストそのものを狙う** 攻撃です。
>
> 私たちの `Pool` を見てください。フリーリストのポインタは、**空きブロックの中身** に書かれています。
>
> もし攻撃者が、解放済みブロックの中身を書き換えられたら?
>
> ```
> free_ → [ next = 攻撃者が選んだアドレス ]
> ```
>
> 次の `Allocate()` は、**攻撃者が指定したアドレス** を返します。そこに好きな値を書き込めます。関数ポインタのテーブルでも、戻りアドレスでも。
>
> glibc の malloc に対する「unlink 攻撃」や「tcache poisoning」と呼ばれる手法は、まさにこの構造を突くものです。**侵入的フリーリストは、速さと引き換えに、この弱点を抱えています。**
>
> ---
>
> だから、現代のアロケーターは検査を強化してきました。
>
> **glibc** は、tcache に二重解放の検出を追加しました。ブロックに「どの tcache に属するか」の印を書き、解放時に確認します。
>
> **mimalloc** や **hardened_malloc** は、フリーリストのポインタを **暗号化** します。アドレスに秘密の値を XOR しておくので、書き換えられても正しいポインタになりません。
>
> **Chrome の PartitionAlloc** は、型ごとに領域(パーティション)を分けます。型 A の領域と型 B の領域が重ならないので、type confusion が成立しにくくなります。
>
> ---
>
> **私たちの目的はセキュリティではありません。** ゲームのアロケーターに、攻撃者を想定した防御は通常不要です。
>
> しかし、この章で追加した2つの検査は、**結果的に同じ攻撃経路を塞いでいます**。
>
> - 所属チェック → 他所のポインタでフリーリストを乗っ取れない
> - 二重解放チェック → 同じブロックを2回配らせられない
>
> **バグを見つけるための検査と、攻撃を防ぐための検査は、しばしば一致します。** 「正しくないことが起きたら止める」という原則が、両方に効いているからです。
>
> 第31章でガードページを、第36章で AddressSanitizer を扱うときにも、同じ構図が現れます。これらの技術は、もともとセキュリティの文脈で発展したものが、デバッグに転用されたケースです。
