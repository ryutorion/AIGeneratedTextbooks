# 第8章 まとめて捨てる 〔v0.4〕

---

## この章のゴール

`Bump` は正しく、速く、安全になりました。しかし、まだ使い物になりません。

**一度使い切ったら、それで終わり。**

1024 バイトの板を使い切ったら、その `Bump` はもう何も返せません。第5章のベンチマークでも、サンプルごとに `Bump` を作り直していました。ゲームのメインループで毎フレーム `Bump` を作り直すわけにはいきません。

この章で足すのは、たった1行です。

```cpp
void Reset() noexcept { offset_ = 0; }
```

しかしこの1行が、`Bump` を「おもちゃ」から「実戦で使える道具」に変えます。

- `Reset()` を実装する 〔**v0.4**〕
- 「これは本当に解放と呼べるのか」を考える
- **解放のコスト** を `new` と比べる(この章の目玉です)
- リセット後に残るポインタの危険性を、実際に踏んでみる
- なぜゲームでこれが強力なのかを、寿命の観点から整理する

---

## 8.1 `Reset()` を書く

実装から見せます。

```cpp
    void Reset() noexcept
    {
        // ピーク使用量だけは覚えておく
        if (offset_ > peak_) { peak_ = offset_; }

        offset_  = 0;
        padding_ = 0;
    }
```

本質は `offset_ = 0;` の1行です。板の先頭に刃を戻すだけ。

### ピーク使用量を記録する

`Reset()` すると `Used()` はゼロに戻ります。すると「このアロケーターは実際どれくらい使われていたのか」が分からなくなります。

そこで、リセット前の値を記録しておきます。

```cpp
    std::size_t Peak() const noexcept
    {
        return (offset_ > peak_) ? offset_ : peak_;
    }
```

これは第49章のメモリ予算管理に直結する情報です。「毎フレーム最大どれだけ使ったか」が分かれば、板のサイズを適切に決められます。実際のゲーム開発では、ピーク値だけを見ていると言ってもいいくらい重要な数字です。

### 動かしてみる

```cpp
int main()
{
    Bump bump(1024);

    for (int frame = 0; frame < 3; ++frame)
    {
        // 「フレーム中の処理」のつもり
        for (int i = 0; i < 10; ++i)
        {
            auto r = bump.Allocate(16, 16);
            assert(r.has_value());
        }

        std::println("frame {}: used={} peak={}", frame, bump.Used(), bump.Peak());

        bump.Reset();   // フレームの終わり
    }

    std::println("最終: used={} peak={}", bump.Used(), bump.Peak());
}
```

```
frame 0: used=160 peak=160
frame 1: used=160 peak=160
frame 2: used=160 peak=160
最終: used=0 peak=160
```

1024 バイトの板で、いくらでもフレームを回せるようになりました。

---

## 8.2 これは「解放」と呼べるのか

`Reset()` がやったことを、正確に把握しておきましょう。**やっていないこと** のほうが重要です。

| やること | やらないこと |
|---|---|
| `offset_` を 0 に戻す | メモリを OS に返す |
| | デストラクタを呼ぶ |
| | メモリの内容を消す |
| | 古いポインタを無効化する |

### メモリは OS に返らない

`buffer_` はそのまま残ります。1024 バイトの板は、`Bump` が破棄されるまで確保されっぱなしです。

これは欠点ではなく、**利点** です。返してしまえば、次に使うときにまた OS から取り直さなければなりません。第5章で見たとおり、その処理には µs 級のスパイクが伴います。

板を握りっぱなしにすることで、そのコストを **最初の1回だけ** に押し込めています。

### デストラクタは呼ばれない

`Reset()` は `offset_` を書き換えるだけです。その領域に置かれていたオブジェクトのデストラクタは、呼ばれません。

```cpp
auto r = bump.Allocate(sizeof(std::string), alignof(std::string));
// ... ここに std::string を構築したとして ...
bump.Reset();   // ← デストラクタは呼ばれない。std::string が内部で確保したメモリはリーク
```

現時点ではまだオブジェクトを構築する手段(`New<T>()`)がないので実害はありませんが、第10章でそれを作った瞬間、この問題が現実になります。第11章で正面から扱います。

### 中身は消えない

`Reset()` の後、板の中身は前のデータがそのまま残っています。次に確保した領域には、前フレームのゴミが入っています。

「確保したメモリはゼロで初期化されている」と思い込んでいると、ここで刺されます。第3章で警告したとおりです。第16章で `0xCD` 塗りつぶしを入れると、この種のバグが一瞬で見つかるようになります。

---

## 8.3 危険:リセット後もポインタは「使えてしまう」

`Reset()` の最大の落とし穴です。実際に踏んでみましょう。

```cpp
int main()
{
    Bump bump(1024);

    // --- 1回目の確保 ---
    auto r1 = bump.Allocate(sizeof(int), alignof(int));
    int* p = static_cast<int*>(*r1);
    *p = 42;

    std::println("リセット前 : *p = {}  (アドレス {})", *p, static_cast<void*>(p));

    // --- リセット ---
    bump.Reset();

    // --- 2回目の確保 ---
    auto r2 = bump.Allocate(sizeof(int), alignof(int));
    int* q = static_cast<int*>(*r2);
    *q = 99;

    std::println("リセット後 : *q = {}  (アドレス {})", *q, static_cast<void*>(q));
    std::println("古いポインタ: *p = {}  (アドレス {})", *p, static_cast<void*>(p));
    std::println("p == q ? {}", p == q);
}
```

```
リセット前 : *p = 42  (アドレス 0x1f8c2a4b3c0)
リセット後 : *q = 99  (アドレス 0x1f8c2a4b3c0)
古いポインタ: *p = 99  (アドレス 0x1f8c2a4b3c0)
p == q ? true
```

**`p` と `q` は同じアドレスです。**

`p` を通して書けば `q` の内容が変わり、その逆も起きます。リセット前に取ったポインタを持ち越すと、まったく無関係な2つのデータが同じ場所を共有することになります。

### 質が悪いのは「落ちない」こと

このバグの厄介さは、**アクセス違反にならない** 点にあります。`p` が指しているのは板の内側であり、有効なアドレスです。OS もハードウェアも、何も文句を言いません。

`new` / `delete` なら、解放済みのメモリにアクセスすればクラッシュする可能性がありました(Debug 構成なら塗りつぶしパターンで気づけます)。バンプアロケーターでは、その手がかりすらありません。

### 対処法

3段階あります。

**1. 規律。** リセットするアロケーターから取ったポインタは、リセットをまたいで持ち越さない。これが基本です。第43章のフレームアロケーターでは、これがそのまま運用ルールになります。

**2. 検出。** 第16章で `Reset()` 時に領域を `0xDD` で塗りつぶすようにすると、古いポインタから読んだ値が明らかにおかしくなり、気づけるようになります。

**3. 型で防ぐ。** 第45章でハンドルを導入すると、世代カウンタによって「古い参照」を検出できるようになります。ポインタをやめるという、最も根本的な解決策です。

今の段階では、**1** の規律で進みます。

---

## 8.4 解放のコストを比べる

ここからがこの章の本題です。

第5章では **確保** のコストを比べました。`Bump` が 10〜17 倍速いという結果でした。今度は **解放** を比べます。

### 何を比べるのか

100万個のオブジェクトを使い終わったとき、後片付けにどれだけかかるか。

| | 後片付けの方法 |
|---|---|
| `Bump` | `Reset()` を1回呼ぶ |
| `new` | `delete` を100万回呼ぶ |

書いてみましょう。

```cpp
// --- Bump:100万回確保してから Reset ---
double MeasureBumpTeardown(std::size_t count, std::size_t size)
{
    Bump bump(count * size + 4096);

    for (std::size_t i = 0; i < count; ++i)
    {
        bench::Escape(bump.Allocate(size).value_or(nullptr));
    }

    const auto a = std::chrono::steady_clock::now();
    bump.Reset();                               // ← これだけ
    const auto b = std::chrono::steady_clock::now();

    return std::chrono::duration<double, std::nano>(b - a).count();
}

// --- new:100万回確保してから100万回 delete ---
double MeasureNewTeardown(std::size_t count, std::size_t size)
{
    std::vector<void*> ptrs(count);

    for (std::size_t i = 0; i < count; ++i)
    {
        ptrs[i] = ::operator new(size);
        bench::Escape(ptrs[i]);
    }

    const auto a = std::chrono::steady_clock::now();
    for (std::size_t i = 0; i < count; ++i)     // ← 100万回まわる
    {
        ::operator delete(ptrs[i], size);
    }
    const auto b = std::chrono::steady_clock::now();

    return std::chrono::duration<double, std::nano>(b - a).count();
}

int main()
{
    constexpr std::size_t kCount = 1'000'000;
    constexpr std::size_t kSize  = 16;

    const double bumpNs = MeasureBumpTeardown(kCount, kSize);
    const double newNs  = MeasureNewTeardown(kCount, kSize);

    std::println("100万オブジェクトの後片付け");
    std::println("  Bump::Reset()     : {:>12.1f} ns  ({:.4f} ms)", bumpNs, bumpNs / 1e6);
    std::println("  operator delete×n : {:>12.1f} ns  ({:.4f} ms)", newNs,  newNs  / 1e6);
    std::println("  比                : {:.0f} 倍", newNs / (bumpNs > 0 ? bumpNs : 1.0));
}
```

### 結果

```
100万オブジェクトの後片付け
  Bump::Reset()     :          0.0 ns  (0.0000 ms)
  operator delete×n :   15840000.0 ns  (15.8400 ms)
```

`Reset()` は **0.0 ns**。第4章で確認したとおり、時計の分解能は 100 ns なので、計測不能なほど速いということです。実際には数ナノ秒でしょう。

一方、`delete` を100万回呼ぶのに **15.8 ミリ秒**。

### 16.6 ms 予算で考える

第5章と同じ物差しを当てます。

| | 100万オブジェクトの後片付け | 16.6 ms 予算に対して |
|---|---|---|
| `Bump::Reset()` | 約 0 ms | **0%** |
| `delete` ×100万 | 15.8 ms | **95%** |

**フレーム予算をほぼ丸ごと使い切ります。**

これは「遅い」という話ではありません。1フレームでは終わらないということです。後片付けだけで、そのフレームは確実に落ちます。

### 計算量が違う

差の本質は、実装の巧拙ではありません。

| | 計算量 |
|---|---|
| `Bump::Reset()` | **O(1)** |
| `delete` ×n | O(n) |

`Reset()` は、板に 10 個入っていようと 1000 万個入っていようと、同じ時間で終わります。整数を1つ書き換えるだけだからです。

**これが「まとめて捨てる」ことの威力です。** 個別に解放するアロケーターは、原理的にこの性質を持てません。

---

## 8.5 確保 → リセット → 再確保のループ

実際の使われ方に近い形で測ってみます。

```cpp
bench::Result BenchBumpWithReset(std::size_t countPerFrame, std::size_t size, std::size_t frames)
{
    Bump bump(countPerFrame * size + 4096);

    return bench::MeasureBatch(frames, countPerFrame, [&, i = std::size_t{0}]() mutable {
        bench::Escape(bump.Allocate(size).value_or(nullptr));

        if (++i == countPerFrame)
        {
            i = 0;
            bump.Reset();     // フレームの終わり
        }
    });
}
```

第5章では、サンプルごとに `Bump` を作り直していました(`std::vector` の確保とゼロクリアが毎回発生していました)。`Reset()` があれば、その必要はありません。**ベンチマークのコードまで簡潔になります。**

```
Bump (Reset ループ)   median=      1.8  p95=      1.9  max=        2.2
new  (確保+解放)      median=     17.6  p95=     18.3  max=       19.0
```

確保のコストは第5章と変わりません。しかし今回は、**このループを何時間でも回し続けられます**。メモリ使用量は一定で、断片化も起きません。

### おまけ:キャッシュに乗り続ける

`Reset()` を繰り返すと、毎フレーム **同じアドレス範囲** を使い回すことになります。その領域は CPU のキャッシュに乗ったままです。

`new` の場合、ヒープの状態によって確保されるアドレスは変わります。キャッシュに乗っている保証はありません。

この差がどれくらい効くかは第32章で測りますが、「同じ場所を使い続ける」ことには、確保の速さとは別の価値があるということを覚えておいてください。

---

## 8.6 なぜゲームでこれが強力なのか

ここで第3章の分類に戻ります。ゲームのメモリには、寿命が4種類ありました。

| 寿命 | 例 | 解放のタイミング |
|---|---|---|
| **永続** | 設定、コアシステム | 終了時 |
| **レベル単位** | ステージのモデル、テクスチャ | シーン切り替え時 |
| **フレーム単位** | 描画コマンド、当たり判定の候補リスト | 毎フレーム末 |
| **一時** | 関数内の作業領域 | スコープを抜けるとき |

上3つは、いずれも **「ある時点で、まとめて全部いらなくなる」** という構造をしています。

```
   フレーム開始                              フレーム終了
        │                                         │
        ├── 当たり判定の候補リスト ────────────────┤
        ├──── 描画コマンド ───────────────────────┤
        ├─ パーティクルの中間計算 ────────────────┤
        ├────── UI のレイアウト結果 ──────────────┤
        │                                         │
        └────────────────────────────────────► Reset()
```

これらを個別に解放する意味はありません。全部同時に死ぬのですから。

### 汎用アロケーターは、この情報を使えない

`new` / `delete` には、「これらは同時に死ぬ」という情報が伝わりません。1つ1つが独立した寿命を持つ可能性があるという前提で扱うしかありません。だから、

- 個別に解放を受け付ける
- 解放された穴を記録する
- 穴を再利用できるよう探索する
- 隣接する穴を結合する

という仕事をせざるを得ません。第5章で見た 17.6 ns の中身です。

**私たちは、その情報を持っています。** 「このアロケーターに入るものは全部同時に死ぬ」と知っているからこそ、仕事を全部捨てられます。

> **アロケーターの設計とは、「自分が何を知っているか」を性能に変換する作業です。**

### 使い分けの見取り図

| 寿命 | 使うもの | 本書で扱う章 |
|---|---|---|
| 永続 | 起動時に一括確保、解放しない | 第39章 |
| レベル単位 | シーン用の `Bump`、切り替え時に `Reset` | 第44章 |
| フレーム単位 | フレーム用の `Bump`、毎フレーム `Reset` | 第43章 |
| 一時 | スタックアロケーター(次章) | 第9章 |
| **上記に当てはまらないもの** | 個別解放できるアロケーター | 第20〜27章 |

最後の行が、この本の第3部です。すべてが綺麗に分類できるわけではありません。しかし、**分類できるものを分類しておけば、残りは驚くほど少なくなります。**

---

## 8.7 テストを書く

```cpp
void Test_ResetRewindsOffset()
{
    Bump bump(1024);

    const bool ok1 = bump.Allocate(100, 1).has_value();
    assert(ok1);
    assert(bump.Used() == 100);

    bump.Reset();
    assert(bump.Used() == 0);
    assert(bump.Remaining() == 1024);

    std::println("[ OK ] Test_ResetRewindsOffset");
}

void Test_ResetReusesSameAddress()
{
    Bump bump(1024);

    auto r1 = bump.Allocate(16, 16);
    assert(r1.has_value());
    void* first = *r1;

    bump.Reset();

    auto r2 = bump.Allocate(16, 16);
    assert(r2.has_value());
    void* second = *r2;

    assert(first == second);   // 同じアドレスが返る

    std::println("[ OK ] Test_ResetReusesSameAddress");
}

void Test_ResetRecoversFromOutOfMemory()
{
    Bump bump(64);

    // 使い切る
    for (int i = 0; i < 4; ++i)
    {
        const bool ok = bump.Allocate(16, 16).has_value();
        assert(ok);
    }

    // 溢れる
    assert(!bump.Allocate(16, 16).has_value());

    // リセットすれば復活する
    bump.Reset();
    assert(bump.Allocate(16, 16).has_value());

    std::println("[ OK ] Test_ResetRecoversFromOutOfMemory");
}

void Test_PeakSurvivesReset()
{
    Bump bump(1024);

    const bool ok = bump.Allocate(300, 1).has_value();
    assert(ok);
    assert(bump.Peak() == 300);

    bump.Reset();

    assert(bump.Used() == 0);
    assert(bump.Peak() == 300);   // ピークは残る

    // 次はもっと少なく使う
    const bool ok2 = bump.Allocate(100, 1).has_value();
    assert(ok2);
    assert(bump.Peak() == 300);   // 更新されない

    std::println("[ OK ] Test_PeakSurvivesReset");
}
```

`Test_ResetReusesSameAddress` は、8.3 節で見た危険性を **テストとして固定** したものです。「同じアドレスが返る」のは仕様であって、バグではありません。仕様なら、テストに書いておくべきです。

---

## 8.8 この章の完成コード

`Bump` クラスの差分だけ示します(他は第7章のまま)。

```cpp
class Bump
{
public:
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

    // --- ここから v0.4 の追加 ---

    // すべての確保をまとめて捨てる。O(1)。
    // 注意:デストラクタは呼ばれず、以前のポインタは無効になる。
    void Reset() noexcept
    {
        if (offset_ > peak_) { peak_ = offset_; }

        offset_  = 0;
        padding_ = 0;
    }

    // これまでの最大使用量(Reset をまたいで保持される)
    std::size_t Peak() const noexcept
    {
        return (offset_ > peak_) ? offset_ : peak_;
    }

    std::size_t Used()      const noexcept { return offset_; }
    std::size_t Capacity()  const noexcept { return buffer_.size(); }
    std::size_t Remaining() const noexcept { return Capacity() - Used(); }
    std::size_t Padding()   const noexcept { return padding_; }

    const std::byte* Base() const noexcept { return buffer_.data(); }

private:
    std::vector<std::byte> buffer_;
    std::size_t            offset_  = 0;
    std::size_t            padding_ = 0;
    std::size_t            peak_    = 0;   // v0.4
};
```

コメントに **「以前のポインタは無効になる」** と明記しました。この危険性は実装から読み取れないので、必ずドキュメントに残してください。

---

## 演習

**演習8-1** `Reset()` を呼ぶ前に `Peak()` を更新するのではなく、`Allocate()` の中で毎回更新する実装に変えてください。どちらが良いでしょうか。性能と正確さの両面から考えてください。

**演習8-2** `Reset()` の後、板の中身が前のデータのまま残っていることを `DumpBytes`(第3章)で確認してください。

**演習8-3** リセット回数を数える `ResetCount()` を足してください。デバッグでどんなときに役立ちそうですか。

**演習8-4** 8.4 節の測定を、確保サイズ 16 → 256 バイトに変えて実行してください。`delete` 側の時間はどう変わりますか。`Reset()` はどうですか。

**演習8-5** 8.3 節の危険なコードを書いて、実際に `*p` が変化することを確認してください。その後、`Reset()` の中で板全体を `0xDD` で埋めるようにすると、何が起きますか。(第16章の先取りです)

**演習8-6** `Bump` を2つ用意し、片方はフレーム用、もう片方はシーン用として使うプログラムを書いてください。フレーム用だけを毎回 `Reset()` します。どんな不都合が起きうるか考えてください。

---

## 章末チェックリスト

- [ ] `Reset()` を実装した 〔v0.4〕
- [ ] 確保 → リセット → 再確保のループを動かした
- [ ] `Reset()` が **やらないこと** を4つ挙げられる
- [ ] リセット後も古いポインタが「使えてしまう」ことを実際に確認した
- [ ] 100万オブジェクトの後片付けコストを `delete` と比べた
- [ ] `Reset()` が O(1)、`delete` が O(n) であることを理解した
- [ ] 寿命が揃ったデータという考え方を説明できる

---

## 次章の予告

`Reset()` は強力ですが、粗い道具です。**全部か、何も捨てないか** の二択しかありません。

こんな場面を考えてください。

```
フレーム開始
  ├─ 描画コマンドを確保(フレーム末まで生きる)
  │
  │  ├─ 当たり判定の作業領域を確保(この関数の中だけ)
  │  ├─ ソート用の一時バッファ(この関数の中だけ)
  │  └─ 関数を抜ける ← ここで作業領域だけ捨てたい
  │
  └─ フレーム末に Reset()
```

関数の中だけで使う一時領域を、関数を抜けるときに返したい。しかし `Reset()` を呼べば、描画コマンドまで消えてしまいます。

第9章では **マーカー** を導入します。「今の位置」を覚えておいて、そこまで巻き戻す。スタックのように後入れ先出しで使える形にします。RAII と組み合わせれば、`{}` を抜けるだけで自動的に巻き戻るようになります。

---

> **コラム:アリーナ、リージョン、プール、ゾーン**
>
> 「まとめて確保して、まとめて捨てる」という発想には、分野ごとに違う名前が付いています。
>
> **リージョン(region)。** 学術的にはこの呼び方が主流です。1994年に Tofte と Talpin が発表した **リージョン推論** は、プログラムを解析して「この値はどのリージョンに属し、いつまとめて解放できるか」をコンパイラが自動で決める技術でした。ガベージコレクションに頼らずメモリを管理する道を示した仕事で、ML Kit という処理系に実装されています。Rust のライフタイムも、遠い親戚と言えます。
>
> **プール(pool)。** Apache HTTP Server と、そこから独立した Apache Portable Runtime が有名です。1つの HTTP リクエストの処理中に確保したものは、リクエストが終われば全部いらない——という構造をそのままメモリ管理に持ち込みました。リクエスト処理は「寿命が揃ったデータ」の典型例です。ゲームのフレームと、まったく同じ構造をしています。
>
> ただし「プール」という語は、第21章で扱う **固定サイズのプールアロケーター** を指すこともあります。文脈で判断してください。
>
> **メモリコンテキスト(memory context)。** PostgreSQL の呼び方です。クエリの実行中に使うメモリをコンテキストに紐づけ、クエリが終わったらコンテキストごと破棄します。親子関係を持てるのが特徴で、親を捨てれば子も全部消えます。
>
> **ゾーン(zone)。** id Software の Quake で使われた呼び方です。ゲーム業界でこの手法が古くから使われてきたことの証拠でもあります。
>
> **アリーナ(arena)。** 本書で主に使う語です。ただし注意が必要で、jemalloc の文脈では「スレッドごとに分割されたヒープ」という別の意味になります(第15章で扱います)。同じ語が違うものを指す典型例です。
>
> ---
>
> 名前はばらばらですが、根っこにあるアイデアは1つです。
>
> > **個々のオブジェクトの寿命を追いかけるのをやめて、寿命の「まとまり」を管理する。**
>
> この発想が、独立に何度も再発明されてきたこと自体が、その有効性を物語っています。Web サーバーでも、データベースでも、コンパイラでも、ゲームでも、同じ構造が見つかります。
>
> そして C++ の標準ライブラリにも、C++17 で `std::pmr::monotonic_buffer_resource` という形で入りました。第39章で、いま書いている `Bump` をその枠組みに乗せます。20行から始めたクラスが、標準ライブラリと同じ土俵に立つことになります。
