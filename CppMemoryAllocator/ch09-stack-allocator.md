# 第9章 途中まで巻き戻す 〔v0.5:スタックアロケーター〕

---

## この章のゴール

`Reset()` は強力ですが、粗い道具です。**全部捨てるか、何も捨てないか** の二択しかありません。

この章では、その中間を手に入れます。

- **マーカー** という考え方を導入する
- `Mark()` と `Rewind()` を実装する 〔**v0.5**〕
- 「エラー」と「バグ」を区別し、それぞれに合った伝え方を選ぶ
- RAII のスコープガードを作り、`{}` を抜けたら自動で巻き戻るようにする
- ネストの順序違反を検出する仕組みを入れる

ここまで来ると、`Bump` は **スタックアロケーター** と呼ばれる形になります。第3章で挙げた4つの寿命のうち、「一時」を担当する道具です。

---

## 9.1 `Reset()` では粗すぎる

こんな状況を考えてください。

```
フレーム開始
  │
  ├─ 描画コマンドを確保 ────────────── フレーム末まで生きる
  │
  │   UpdatePhysics() を呼ぶ
  │     ├─ 衝突候補リストを確保 ─┐
  │     ├─ ソート用バッファを確保 ├─ この関数の中だけ
  │     └─ 関数を抜ける ─────────┘  ← ここで返したい
  │
  ├─ さらに描画コマンドを確保
  │
  └─ フレーム末に Reset()
```

`UpdatePhysics()` の作業領域は、関数を抜けたら不要です。しかし `Reset()` を呼べば、描画コマンドまで消えてしまいます。

かといって作業領域を放置すれば、毎フレーム板が食い潰されていきます。関数が1000回呼ばれれば、1000回分のゴミが積み上がります。

**必要なのは、「ここまで戻る」という操作です。**

---

## 9.2 マーカーという考え方

板の比喩に戻ります。

```
    ┌────┬────┬────────────────────────────────────┐
    │描画│描画│                                    │
    └────┴────┴────────────────────────────────────┘
                ↑
              offset_ = 32
```

ここで **印を付けます**。「今は32バイト目」と紙に書いて、ポケットに入れておく。

```
    ┌────┬────┬──────┬──────┬────────────────────┐
    │描画│描画│ 候補 │ ソート│                    │
    └────┴────┴──────┴──────┴────────────────────┘
                ↑                ↑
              印 = 32       offset_ = 96
```

作業が終わったら、ポケットの紙を見て `offset_` をそこに戻します。

```
    ┌────┬────┬────────────────────────────────────┐
    │描画│描画│  ← 96 まで使ったが、32 に戻した     │
    └────┴────┴────────────────────────────────────┘
                ↑
              offset_ = 32
```

**これだけです。** `Reset()` が「常に 0 に戻る」のに対し、`Rewind()` は「好きな位置に戻る」。

### 制約:後入れ先出し

この仕組みには制約があります。**印を付けた順とは逆の順にしか戻せません。**

```
印A(32) → 印B(64) → ... → 印B に戻る → 印A に戻る    ○
印A(32) → 印B(64) → ... → 印A に戻る → 印B に戻る    ×
```

2番目がなぜダメか。印A に戻ると `offset_` は 32 になります。その後に印B(64)へ「戻ろう」とすると、`offset_` は 32 から 64 へ **進んで** しまいます。すでに他の誰かが使っているかもしれない領域を、再び配り始めることになります。

この後入れ先出し(LIFO)の性質から、この方式は **スタックアロケーター** と呼ばれます。関数の呼び出しと戻りが自然に LIFO になるので、相性が良いのです。

---

## 9.3 `Marker` 型を設計する

素朴に考えると、`Mark()` は `std::size_t` を返せば済みそうです。

```cpp
std::size_t Mark() const noexcept { return offset_; }   // ← やめておく
```

しかし、これは避けます。理由が3つあります。

**1. 他の数値と混ざる。** `std::size_t` はサイズにも個数にもインデックスにも使われます。`Rewind(count)` と書き間違えてもコンパイルが通ってしまいます。

**2. 別のアロケーターのマーカーを渡せてしまう。** `Bump` を2つ使っていて、片方のマーカーをもう片方に渡す——これもコンパイルは通ります。

**3. 復元すべき情報が `offset_` だけとは限らない。** 実際、`padding_` も戻す必要があります。将来さらに増えるかもしれません。

専用の型を作ります。

```cpp
class Bump
{
public:
    // 巻き戻し位置を表す不透明な値
    struct Marker
    {
        std::size_t   offset  = 0;
        std::size_t   padding = 0;
        std::uint32_t depth   = 0;              // ネストの深さ
        std::uint32_t epoch   = kInvalidEpoch;  // Reset 世代
    };

    static constexpr std::uint32_t kInvalidEpoch = 0xFFFF'FFFFu;
    ...
};
```

`depth` と `epoch` は、検査のためのフィールドです。次節以降で使います。

### 構成によってレイアウトを変えない

「`depth` と `epoch` は Debug でしか使わないのだから、`#ifndef NDEBUG` で囲めばいい」と考えるかもしれません。

**やめてください。** 構成によって構造体のサイズが変わると、Debug でビルドしたコードと Release でビルドしたコードを混ぜてリンクしたときに、静かに壊れます。ODR 違反と呼ばれる問題で、原因の特定が非常に困難です。

**フィールドは常に持ち、検査だけを構成で切り替える。** これが安全な作り方です。16 バイト増えることを気にする場面ではありません。

---

## 9.4 `Mark()` と `Rewind()` を実装する 〔v0.5〕

```cpp
class Bump
{
public:
    // --- 現在位置に印を付ける ---
    [[nodiscard]]
    Marker Mark() noexcept
    {
        return Marker{ offset_, padding_, depth_++, epoch_ };
    }

    // --- 印の位置まで巻き戻す ---
    void Rewind(const Marker& m) noexcept
    {
        // 【契約違反の検出:Debug のみ】
        assert(m.epoch == epoch_        && "Reset をまたいだマーカーです");
        assert(m.depth + 1 == depth_    && "巻き戻しの順序が LIFO ではありません");
        assert(m.offset <= offset_      && "前方に巻き戻そうとしています");

        // 【最低限の防衛:Release でも有効】
        if (m.epoch != epoch_ || m.offset > offset_)
        {
            return;   // 何もしないほうが、壊すよりまし
        }

        if (offset_ > peak_) { peak_ = offset_; }

        offset_  = m.offset;
        padding_ = m.padding;
        depth_   = m.depth;
    }

    // --- Reset は「先頭まで巻き戻す」こと ---
    void Reset() noexcept
    {
        if (offset_ > peak_) { peak_ = offset_; }

        offset_  = 0;
        padding_ = 0;
        depth_   = 0;
        ++epoch_;      // これ以前のマーカーをすべて無効化する
    }

private:
    std::vector<std::byte> buffer_;
    std::size_t            offset_  = 0;
    std::size_t            padding_ = 0;
    std::size_t            peak_    = 0;
    std::uint32_t          depth_   = 0;   // v0.5
    std::uint32_t          epoch_   = 0;   // v0.5
};
```

### `epoch_` の役割

`Reset()` のたびに `epoch_` を増やします。マーカーは作られた時点の `epoch` を覚えているので、`Reset()` をまたいだ古いマーカーは一致しなくなり、検出できます。

なぜこれが必要か。`Reset()` の後に再び確保が進むと、`offset_` が古いマーカーの位置を超えることがあります。そうなると `m.offset <= offset_` の検査だけでは通ってしまい、**別のフレームのデータを巻き戻し先にしてしまいます**。

```
frame 0: Mark() → offset=32 のマーカーを取得
frame 0: Reset()
frame 1: 100 バイト確保 → offset_ = 100
frame 1: 古いマーカー(offset=32)で Rewind → 32 に戻ってしまう!
```

`epoch_` があれば、この事故を止められます。

### `depth_` の役割

`Mark()` のたびに深さを1つ増やし、マーカーに記録します。`Rewind()` は「自分がいちばん内側であること」を確認します。

9.2 節で見た順序違反を、この1行が捕まえます。

---

## 9.5 エラーとバグを区別する

ここで、第7章と設計方針が違うことに気づいたでしょうか。

| | 伝え方 |
|---|---|
| `Allocate()` の失敗 | `std::expected` を返す |
| `Rewind()` の順序違反 | `assert` で落とす |

なぜ揃えないのか。**種類が違うからです。**

**メモリ不足は「実行時の条件」です。** プログラムは正しく書かれているが、たまたま板が足りなかった。入力データが大きかったのかもしれません。呼び出し側が対処できる可能性があり、対処法もあります(諦める、別のアロケーターを使う、品質を落とす)。

**巻き戻しの順序違反は「プログラムのバグ」です。** どんな入力でも起きてはいけません。実行時に「順序が違いました」と返されたところで、呼び出し側にできることはありません。バグを直す以外に道はない。

> **回復できるものはエラー、回復できないものはバグ。**
> エラーは値で返す。バグは、できるだけ早く、大きな音を立てて知らせる。

この区別は、アロケーターに限らず C++ の API 設計全般で使えます。すべてを `expected` にするのが良い設計ではありません。

### Release でも最低限は守る

とはいえ、`assert` は Release で消えます。順序違反が Release で起きたとき、`offset_` が前方に飛べば **すでに配った領域を再び配る** ことになり、被害は甚大です。

そこで、比較2回ぶんの防衛だけは Release にも残しました。

```cpp
if (m.epoch != epoch_ || m.offset > offset_) { return; }
```

**「何もしない」ことを選んでいる** 点に注意してください。バグは残りますが、メモリ破壊よりはるかにましです。

> 第51章で、この種の「Release でも残す検査」と「Debug だけの検査」を体系的に整理します。

---

## 9.6 手で使ってみる

```cpp
int main()
{
    Bump bump(1024);

    // 長生きするデータ
    auto rA = bump.Allocate(32, 16);
    assert(rA.has_value());
    std::println("長生きデータ確保後 : used={}", bump.Used());

    // ここに印を付ける
    const auto marker = bump.Mark();
    std::println("マーカー取得       : offset={}", marker.offset);

    // 一時的な作業
    for (int i = 0; i < 5; ++i)
    {
        auto r = bump.Allocate(16, 16);
        assert(r.has_value());
    }
    std::println("一時データ確保後   : used={}", bump.Used());

    // 巻き戻す
    bump.Rewind(marker);
    std::println("巻き戻し後         : used={}", bump.Used());

    // 長生きデータはまだ生きている
    std::println("長生きデータのアドレス: {}", *rA);
}
```

```
長生きデータ確保後 : used=32
マーカー取得       : offset=32
一時データ確保後   : used=112
巻き戻し後         : used=32
長生きデータのアドレス: 0x1f3a8c2b3c0
```

80 バイトの一時領域だけが返り、長生きデータはそのまま残りました。

---

## 9.7 RAII でスコープガードを作る

手で `Mark()` と `Rewind()` を書く方式には、明白な弱点があります。

```cpp
void Process(Bump& bump)
{
    const auto m = bump.Mark();

    auto r = bump.Allocate(1000);
    if (!r) { return; }          // ← Rewind し忘れ!

    // ...

    bump.Rewind(m);
}
```

早期 return、例外、`break`、複数の出口。どれか1つで忘れれば、そのぶん板が漏れます。

C++ には、これを解決する定番の道具があります。**RAII** です。

```cpp
// スコープを抜けたら自動的に巻き戻す
class BumpScope
{
public:
    explicit BumpScope(Bump& bump) noexcept
        : bump_(&bump)
        , marker_(bump.Mark())
    {
    }

    ~BumpScope()
    {
        bump_->Rewind(marker_);
    }

    // コピーもムーブも禁止
    BumpScope(const BumpScope&)            = delete;
    BumpScope& operator=(const BumpScope&) = delete;
    BumpScope(BumpScope&&)                 = delete;
    BumpScope& operator=(BumpScope&&)      = delete;

private:
    Bump*        bump_;
    Bump::Marker marker_;
};
```

### なぜムーブも禁止するのか

コピーを禁止するのは自明です(2回巻き戻ってしまう)。ではムーブは?

ムーブを許すと、スコープガードを関数の外に持ち出せるようになります。すると **破棄される順序が、作られた順序の逆であるという保証が崩れます**。LIFO を強制するために存在する型が、LIFO を破る道具になってしまう。

禁止しておけば、`BumpScope` は必ずスコープの中で生まれ、スコープの中で死にます。**言語の仕組みが LIFO を保証してくれます。**

### 使う

```cpp
void Process(Bump& bump)
{
    BumpScope scope(bump);        // ← これだけ

    auto r = bump.Allocate(1000);
    if (!r) { return; }           // ← ここで抜けても巻き戻る

    // ...
}                                 // ← ここでも巻き戻る
```

出口がいくつあっても、例外が飛んでも、必ず巻き戻ります。

### 効果を確認する

```cpp
std::size_t DoWork(Bump& bump, int n)
{
    BumpScope scope(bump);

    auto r = bump.Allocate(n * sizeof(int), alignof(int));
    if (!r) { return 0; }

    int* work = static_cast<int*>(*r);
    for (int i = 0; i < n; ++i) { work[i] = i; }

    bench::Escape(work);
    return bump.Used();
}

int main()
{
    Bump bump(4096);

    for (int i = 0; i < 5; ++i)
    {
        const std::size_t inside = DoWork(bump, 100);
        std::println("呼び出し {}: 関数内 used={} / 戻った後 used={}",
                     i, inside, bump.Used());
    }
}
```

```
呼び出し 0: 関数内 used=400 / 戻った後 used=0
呼び出し 1: 関数内 used=400 / 戻った後 used=0
呼び出し 2: 関数内 used=400 / 戻った後 used=0
呼び出し 3: 関数内 used=400 / 戻った後 used=0
呼び出し 4: 関数内 used=400 / 戻った後 used=0
```

**何度呼んでも使用量が増えません。** 4096 バイトの板で、この関数を何億回でも呼べます。

`BumpScope` を消して同じプログラムを走らせると、5回目には 2000 バイトまで積み上がります。100万回呼べば溢れます。

---

## 9.8 ネストと順序違反

### 正しいネスト

```cpp
void Test_NestedScopes()
{
    Bump bump(1024);

    const bool ok = bump.Allocate(16, 16).has_value();
    assert(ok);
    assert(bump.Used() == 16);

    {
        BumpScope outer(bump);
        assert(bump.Allocate(16, 16).has_value());
        assert(bump.Used() == 32);

        {
            BumpScope inner(bump);
            assert(bump.Allocate(16, 16).has_value());
            assert(bump.Used() == 48);
        }   // inner が巻き戻る

        assert(bump.Used() == 32);
    }   // outer が巻き戻る

    assert(bump.Used() == 16);

    std::println("[ OK ] Test_NestedScopes");
}
```

デストラクタは構築の逆順に呼ばれるので、`inner` → `outer` の順で巻き戻ります。LIFO が自動的に守られます。

### 順序違反を起こしてみる

手動の `Mark` / `Rewind` を使えば、違反を作れます。

```cpp
void ExperimentOrderViolation()
{
    Bump bump(1024);

    const auto m1 = bump.Mark();
    bump.Allocate(16, 16).value_or(nullptr);

    const auto m2 = bump.Mark();
    bump.Allocate(16, 16).value_or(nullptr);

    bump.Rewind(m1);   // 先に外側を巻き戻す
    bump.Rewind(m2);   // ← 順序違反!
}
```

Debug 構成で実行すると、2つ目の `Rewind` で止まります。

```
Assertion failed: m.depth + 1 == depth_ && "巻き戻しの順序が LIFO ではありません"
```

しかも `m2.offset` は現在の `offset_` より大きいので、前方への巻き戻しの検査にも引っかかります。二重に守られています。

### `Reset()` をまたぐ違反

```cpp
void ExperimentStaleMarker()
{
    Bump bump(1024);

    bump.Allocate(32, 16).value_or(nullptr);
    const auto m = bump.Mark();     // epoch = 0 のマーカー

    bump.Reset();                   // epoch が 1 になる

    bump.Allocate(100, 16).value_or(nullptr);   // offset_ = 100

    bump.Rewind(m);                 // ← 古いマーカー
}
```

```
Assertion failed: m.epoch == epoch_ && "Reset をまたいだマーカーです"
```

`m.offset` は 32、現在の `offset_` は 100 なので、位置の検査だけでは通ってしまいます。`epoch_` があるからこそ捕まえられました。

---

## 9.9 コストを測る

`Mark()` と `Rewind()` を足したことで、確保が遅くなっていないか確認します。

```
Bump v0.4 (Reset のみ)     median=      1.8  p95=      1.9  max=        2.2
Bump v0.5 (Marker つき)    median=      1.8  p95=      1.9  max=        2.2
```

**変わりません。** `Allocate()` には手を入れていないので当然です。

`Mark()` / `Rewind()` 自体のコストも測っておきます。

```cpp
auto r = bench::MeasureBatch(1000, 10000, [&] {
    BumpScope scope(bump);
    bench::Escape(bump.Allocate(16, 16).value_or(nullptr));
});
bench::Print("Scope + Allocate", r);
```

```
Scope + Allocate       median=      2.4  p95=      2.5  max=        4.1
```

スコープガードのぶんが 0.6 ns ほど。整数を4つコピーして4つ書き戻しているだけなので、妥当な数字です。

比較のため、同じことを `new` / `delete` でやると:

```
new + delete           median=     17.8  p95=     18.5  max=      2100.0
```

**7倍以上の差** があり、最大値では 500 倍以上です。

---

## 9.10 これで何が手に入ったか

第3章で挙げた4つの寿命のうち、3つに対応できるようになりました。

| 寿命 | 対応する操作 | この本での担当 |
|---|---|---|
| 永続 | 確保したまま放置 | 済 |
| レベル単位 | `Reset()` | 済(第8章) |
| フレーム単位 | `Reset()` | 済(第8章) |
| **一時** | **`BumpScope`** | **済(この章)** |
| その他 | 個別解放 | 第20章〜 |

しかも、これらは **同じ1つのアロケーターの上で共存できます**。

```cpp
Bump frameArena(16 * 1024 * 1024);

void UpdateFrame()
{
    // フレーム末まで生きるデータ
    auto* drawCommands = AllocateDrawCommands(frameArena);

    {
        BumpScope scope(frameArena);       // 一時的な作業
        UpdatePhysics(frameArena);
    }                                       // ← 作業領域だけ返る

    {
        BumpScope scope(frameArena);
        UpdateAnimation(frameArena);
    }

    Render(drawCommands);

    frameArena.Reset();                     // ← フレーム末に全部返す
}
```

これが、実際のゲームエンジンで使われている形にかなり近いものです。第43章で、これを本格的なフレームアロケーターに仕上げます。

---

## 9.11 この章の完成コード

```cpp
class Bump
{
public:
    static constexpr std::uint32_t kInvalidEpoch = 0xFFFF'FFFFu;

    struct Marker
    {
        std::size_t   offset  = 0;
        std::size_t   padding = 0;
        std::uint32_t depth   = 0;
        std::uint32_t epoch   = kInvalidEpoch;
    };

    explicit Bump(std::size_t capacity)
        : buffer_(capacity)
    {
    }

    [[nodiscard]]
    AllocResult Allocate(std::size_t size,
                         std::size_t alignment = kDefaultAlignment) noexcept
    {
        // (第7章のまま)
        ...
    }

    // --- v0.5:マーカー ---

    [[nodiscard]]
    Marker Mark() noexcept
    {
        return Marker{ offset_, padding_, depth_++, epoch_ };
    }

    void Rewind(const Marker& m) noexcept
    {
        assert(m.epoch == epoch_     && "Reset をまたいだマーカーです");
        assert(m.depth + 1 == depth_ && "巻き戻しの順序が LIFO ではありません");
        assert(m.offset <= offset_   && "前方に巻き戻そうとしています");

        if (m.epoch != epoch_ || m.offset > offset_) { return; }

        if (offset_ > peak_) { peak_ = offset_; }

        offset_  = m.offset;
        padding_ = m.padding;
        depth_   = m.depth;
    }

    void Reset() noexcept
    {
        if (offset_ > peak_) { peak_ = offset_; }

        offset_  = 0;
        padding_ = 0;
        depth_   = 0;
        ++epoch_;
    }

    std::size_t Used()      const noexcept { return offset_; }
    std::size_t Capacity()  const noexcept { return buffer_.size(); }
    std::size_t Remaining() const noexcept { return Capacity() - Used(); }
    std::size_t Padding()   const noexcept { return padding_; }
    std::size_t Peak()      const noexcept { return (offset_ > peak_) ? offset_ : peak_; }

    const std::byte* Base() const noexcept { return buffer_.data(); }

private:
    std::vector<std::byte> buffer_;
    std::size_t            offset_  = 0;
    std::size_t            padding_ = 0;
    std::size_t            peak_    = 0;
    std::uint32_t          depth_   = 0;
    std::uint32_t          epoch_   = 0;
};

// ---------------------------------------------------------
// スコープを抜けたら自動的に巻き戻す
// ---------------------------------------------------------
class BumpScope
{
public:
    explicit BumpScope(Bump& bump) noexcept
        : bump_(&bump)
        , marker_(bump.Mark())
    {
    }

    ~BumpScope()
    {
        bump_->Rewind(marker_);
    }

    BumpScope(const BumpScope&)            = delete;
    BumpScope& operator=(const BumpScope&) = delete;
    BumpScope(BumpScope&&)                 = delete;
    BumpScope& operator=(BumpScope&&)      = delete;

private:
    Bump*        bump_;
    Bump::Marker marker_;
};
```

---

## 演習

**演習9-1** `Reset()` を `Rewind(Marker{0, 0, 0, epoch_})` として実装できるでしょうか。できるなら、そのほうが良い設計ですか。

**演習9-2** `BumpScope` に「巻き戻しを取り消す」機能(`Commit()` や `Dismiss()`)を足すと、どんな場面で便利でしょうか。危険はありますか。

**演習9-3** `Mark()` に `[[nodiscard]]` を付けています。これがないと、どんな書き間違いが見逃されますか。

**演習9-4** `BumpScope` のムーブを許した場合に壊れるコードを、実際に書いてみてください。

**演習9-5** 再帰関数の中で `BumpScope` を使い、深さ1000まで再帰させてください。`Used()` はどう変化しますか。`depth_` はどうですか。

**演習9-6** `depth_` を `std::uint32_t` にしています。オーバーフローする状況はありえますか。ありうるとしたら、どう守りますか。

**演習9-7** 2つの `Bump` を用意し、片方のマーカーをもう片方の `Rewind()` に渡してください。今の実装は検出できますか。できないなら、どう直しますか。

---

## 章末チェックリスト

- [ ] `Mark()` / `Rewind()` を実装した 〔v0.5〕
- [ ] マーカーに専用の型を使う理由を3つ挙げられる
- [ ] `epoch_` が防いでいる事故を説明できる
- [ ] **エラーとバグの区別** と、それぞれの伝え方を説明できる
- [ ] `BumpScope` を作り、ムーブを禁止した理由を説明できる
- [ ] 関数を何度呼んでも `Used()` が増えないことを確認した
- [ ] 順序違反と古いマーカーの両方で `assert` が落ちることを見た

---

## 次章の予告

ここまで、`Bump` が返すのは常に **生のバイト列** でした。使う側は毎回こう書いています。

```cpp
auto r = bump.Allocate(sizeof(Enemy), alignof(Enemy));
if (!r) { return; }
Enemy* e = static_cast<Enemy*>(*r);
```

3行かかるうえ、`sizeof` と `alignof` を手で書いています。片方だけ書き間違えても、コンパイラは何も言いません。

第10章では `New<T>(args...)` を作ります。型を渡せば、サイズもアラインメントも自動で決まり、**コンストラクタまで呼ばれる** ようにします。

そして、そこで避けられない問題に突き当たります。**デストラクタは誰が呼ぶのか。** `Reset()` も `Rewind()` も、デストラクタを呼びません。`std::string` を1つ置いた瞬間に、それはリークになります。

その始末が第11章です。

---

> **コラム:本物のスタックと、`alloca` という誘惑**
>
> この章で作ったものは、C++ のコールスタックによく似ています。関数に入るとスタックポインタが下がり、抜けると戻る。私たちの `offset_` と `Rewind()` は、まさにその真似です。
>
> では、本物のスタックを直接使えばよいのでは——という発想が当然出てきます。実際、その手段はあります。
>
> **`alloca`** (MSVC では `_alloca`、より安全な `_malloca`)は、実行時に決まるサイズの領域をスタック上に確保します。関数を抜ければ自動的に解放されます。C99 の **可変長配列(VLA)** も同じ発想で、こちらは一部のコンパイラが C++ の拡張として提供しています(標準 C++ にはありません)。
>
> 速度は圧倒的です。スタックポインタを引き算するだけなので、私たちの `Allocate()` よりさらに軽い。
>
> **しかし、ゲーム開発ではまず使われません。** 理由がいくつもあります。
>
> **スタックは狭い。** Windows の既定のスタックサイズはスレッドあたり 1 MB です。数メガバイトの作業領域を取ることはできません。
>
> **溢れると即死する。** ヒープの確保失敗は検出して対処できますが、スタックオーバーフローは多くの場合そのまま落ちます。しかも `alloca` は失敗を返しません。サイズが実行時に決まるということは、**入力データ次第でスタックが吹き飛ぶ** ということです。攻撃経路にもなります。
>
> **ループの中で使うと積み上がる。** `alloca` はスコープではなく関数の終わりまで解放されません。ループの中で呼べば、回数分だけ積み上がります。
>
> **寿命がスコープに固定される。** 関数を抜けたら必ず消えます。「この関数で作って、呼び出し元まで持ち帰る」ができません。
>
> ---
>
> 私たちの `Bump` は、これらすべてを回避しています。
>
> - サイズは自分で決められる(16 MB でも 1 GB でも)
> - 溢れは `std::expected` で返る
> - `BumpScope` はスコープ単位で正しく巻き戻る
> - スコープを超えて生かしたければ、`BumpScope` を使わなければいい
>
> **スタックの「速さ」と「自動性」を、ヒープの「柔軟さ」の上で再現する。** それが、スタックアロケーターという道具の位置づけです。
>
> ちなみに `_malloca` は、小さければスタック、大きければヒープという使い分けを自動でやってくれます。発想としては悪くありませんが、対応する `_freea` を呼ぶ必要があり、結局 RAII で包むことになります。それなら最初から `Bump` と `BumpScope` でよい、というのが本書の立場です。
