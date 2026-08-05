# 第3章 動いているのか確かめる

---

## この章のゴール

前章で `Bump` は動きました。`a = 10` と表示され、期待どおりでした。

ですが「表示が期待どおりだった」ことと「正しく動いている」ことは、同じではありません。とくにメモリを扱うコードでは、この差が命取りになります。

この章では、`Bump` の内部を **見る手段** と、正しさを **自動で確かめる手段** を作ります。

- 使ったバイト数・残りバイト数を報告させる
- アドレスを揃った形式で表示する
- メモリの中身をバイト単位でダンプする
- `assert` によるテスト関数を書く
- わざとバグを入れて、テストが実際に捕まえることを確認する

なお、この章で **アロケーターの振る舞いは1ミリも変えません**。足すのは観測手段だけです。バージョンは `v0.1` のままです。

地味な章です。しかし第23章でフリーリストの合体処理を書くとき、この章で身につけた習慣があなたを救います。

---

## 3.1 まず、演習2-2の答え合わせ

前章の演習2-2はこうでした。

> `Bump` の容量を `16` にして、100回確保するとどうなるか。

やってみます。

```cpp
#include <cstddef>
#include <vector>
#include <print>

// Bump クラス(v0.1)は前章のまま

int main()
{
    Bump bump(16);   // たった16バイト!

    for (int i = 0; i < 100; ++i)
    {
        int* p = static_cast<int*>(bump.Allocate(sizeof(int)));
        *p = i;
    }

    std::println("100回の確保が完了しました");
}
```

実行してみてください。

**おそらく、何事もなく最後まで実行されます。**

```
100回の確保が完了しました
```

16バイトしかない板から、400バイトを切り出しました。エラーメッセージはありません。例外も飛びません。クラッシュもしません。

### これが最悪の挙動である

5個目以降の `int` は、`std::vector` が確保した領域の **外側** に書き込まれています。そこには何があるでしょうか。

- `std::vector` が内部で使っている管理情報
- `Bump` オブジェクト自身のメンバ変数
- 他の変数
- ヒープの管理構造
- あるいは、何も割り当てられていないページ

運が良ければ、すぐクラッシュします。運が悪ければ、**何事もなく動き続けて、数分後に全然関係ない場所で落ちます**。

メモリ破壊のバグが恐れられるのは、この「原因と結果が時間的にも空間的にも離れる」性質のためです。落ちた場所を100時間眺めても、原因は見つかりません。

### だから、まず見えるようにする

この問題そのものは第7章で解決します。ですが解決する前に、まず **状態が見えていない** ことを何とかしなければなりません。

今の `Bump` は、外から見て何も分かりません。どれだけ使ったのか、あとどれだけ残っているのか、そもそも溢れているのか。すべてが `private` の奥に隠れています。

---

## 3.2 内部状態を報告させる

`Bump` に3つのメンバ関数を足します。

```cpp
class Bump
{
public:
    explicit Bump(std::size_t capacity)
        : buffer_(capacity)
    {
    }

    void* Allocate(std::size_t size)
    {
        std::byte* result = buffer_.data() + offset_;
        offset_ += size;
        return result;
    }

    // --- ここから追加 ---

    // これまでに切り出したバイト数
    std::size_t Used() const noexcept
    {
        return offset_;
    }

    // 板全体のバイト数
    std::size_t Capacity() const noexcept
    {
        return buffer_.size();
    }

    // 残りバイト数
    std::size_t Remaining() const noexcept
    {
        return Capacity() - Used();
    }

private:
    std::vector<std::byte> buffer_;
    std::size_t            offset_ = 0;
};
```

3つとも `const noexcept` を付けています。状態を変えないし、失敗しようがないからです。この2語があると、呼び出し側は安心して使えますし、コンパイラも最適化しやすくなります。

### 使ってみる

```cpp
int main()
{
    Bump bump(1024);

    std::println("初期状態  : used={} / cap={} (残り {})",
                 bump.Used(), bump.Capacity(), bump.Remaining());

    bump.Allocate(4);
    std::println("4バイト後 : used={} / cap={} (残り {})",
                 bump.Used(), bump.Capacity(), bump.Remaining());

    bump.Allocate(100);
    std::println("100バイト後: used={} / cap={} (残り {})",
                 bump.Used(), bump.Capacity(), bump.Remaining());
}
```

```
初期状態  : used=0 / cap=1024 (残り 1024)
4バイト後 : used=4 / cap=1024 (残り 1020)
100バイト後: used=104 / cap=1024 (残り 920)
```

内部で何が起きているかが、外から見えるようになりました。

### `Remaining()` の落とし穴

ここで一度、容量16の実験に戻ってみてください。

```cpp
Bump bump(16);
for (int i = 0; i < 100; ++i)
{
    bump.Allocate(4);
}
std::println("used={} / cap={} / remaining={}",
             bump.Used(), bump.Capacity(), bump.Remaining());
```

```
used=400 / cap=16 / remaining=18446744073709551600
```

`Remaining()` が天文学的な数字になりました。

`Capacity() - Used()` は `16 - 400` です。`std::size_t` は符号なし整数なので、負の値になれません。結果は「一周回って」巨大な正の数になります。18446744073709551600 は `2^64 - 400` です。

これは **符号なし整数のラップアラウンド** と呼ばれる挙動で、未定義動作ではなく規格どおりの正しい動作です。しかし、意図とは違います。

この現象を覚えておいてください。第7章で容量チェックを書くとき、次のような条件式は **バグ** になります。

```cpp
if (Remaining() >= size)   // ← すでに溢れていると常に真になる!
```

正しくはこう書きます。

```cpp
if (offset_ + size <= Capacity())   // ← こちらも offset_ + size の桁溢れに注意
```

今は「符号なし引き算は危ない」とだけ心に留めておきましょう。

---

## 3.3 アドレスを読みやすく表示する

前章では `std::println("{}", p)` でアドレスを出しました。改めて見ると、少し読みにくいものでした。

```
p0 = 0x1d4f2c8a3b0
p1 = 0x1d4f2c8a3b4
```

桁数が揃っていないので、複数並べたときに差が目で追えません。書式を整えます。

```cpp
#include <cstdint>

void PrintAddress(const char* label, const void* p)
{
    auto n = reinterpret_cast<std::uintptr_t>(p);
    std::println("{:<4} : {:#018x}", label, n);
}
```

書式指定子の意味は次のとおりです。

| 指定 | 意味 |
|---|---|
| `{:<4}` | 左寄せ、幅4 |
| `{:#018x}` | 16進(`x`)、`0x` を付ける(`#`)、幅18、ゼロ埋め(`0`) |

幅を18にしているのは、64ビットのアドレスが16桁、`0x` の2文字を足して18文字になるためです。

```cpp
int main()
{
    Bump bump(1024);

    void* p0 = bump.Allocate(4);
    void* p1 = bump.Allocate(4);
    void* p2 = bump.Allocate(4);

    PrintAddress("p0", p0);
    PrintAddress("p1", p1);
    PrintAddress("p2", p2);
}
```

```
p0   : 0x0000023f5ca8a3b0
p1   : 0x0000023f5ca8a3b4
p2   : 0x0000023f5ca8a3b8
```

桁が揃い、末尾の `b0` → `b4` → `b8` の変化が一目で読めるようになりました。

### 絶対アドレスではなく、相対オフセットで見る

とはいえ、この数字は実行するたびに変わります。毎回変わる16桁の数字を目で追うのは疲れますし、「前回の実行結果と比べる」ことができません。

本当に知りたいのは「板の先頭から何バイト目か」です。それを表示させましょう。

`Bump` に先頭アドレスを返す関数を足します。

```cpp
    // デバッグ用:板の先頭アドレス
    const std::byte* Base() const noexcept
    {
        return buffer_.data();
    }
```

これを使って、相対位置を表示します。

```cpp
int main()
{
    Bump bump(1024);

    void* p0 = bump.Allocate(4);
    void* p1 = bump.Allocate(7);
    void* p2 = bump.Allocate(100);

    auto base = reinterpret_cast<std::uintptr_t>(bump.Base());

    auto offset = [base](const void* p) {
        return reinterpret_cast<std::uintptr_t>(p) - base;
    };

    std::println("p0 : +{:<5} (size 4)",   offset(p0));
    std::println("p1 : +{:<5} (size 7)",   offset(p1));
    std::println("p2 : +{:<5} (size 100)", offset(p2));
}
```

```
p0 : +0     (size 4)
p1 : +4     (size 7)
p2 : +11    (size 100)
```

**これは実行するたびに同じ結果になります。** 0 → 4 → 11 と、要求サイズぶんきっちり進んでいることが確認できました。再現する数字が出るようになったので、次節のテストにも使えます。

---

## 3.4 メモリの中身を覗く

もう一歩踏み込んで、実際のバイト列を見てみましょう。デバッガのメモリウィンドウでもできますが、自分で書いておくと後々便利です。

```cpp
void DumpBytes(const void* p, std::size_t size)
{
    const std::byte* bytes = static_cast<const std::byte*>(p);

    for (std::size_t i = 0; i < size; ++i)
    {
        std::print("{:02x} ", std::to_integer<unsigned>(bytes[i]));

        if ((i + 1) % 16 == 0)
        {
            std::println("");
        }
    }
    std::println("");
}
```

`std::to_integer<unsigned>` は `std::byte` を数値に変換する関数です。`std::byte` は算術型ではないので、直接 `std::print` には渡せません(第2章で触れた「意味のない生バイト」という設計がここに効いています)。

### `int` を3つ置いて、バイト列を見る

```cpp
int main()
{
    Bump bump(32);

    std::println("--- 確保前 ---");
    DumpBytes(bump.Base(), 32);

    int* a = static_cast<int*>(bump.Allocate(sizeof(int)));
    int* b = static_cast<int*>(bump.Allocate(sizeof(int)));
    int* c = static_cast<int*>(bump.Allocate(sizeof(int)));

    *a = 10;
    *b = 20;
    *c = 30;

    std::println("--- 書き込み後 ---");
    DumpBytes(bump.Base(), 32);
}
```

```
--- 確保前 ---
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00

--- 書き込み後 ---
0a 00 00 00 14 00 00 00 1e 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
```

**これが「メモリの正体」です。**

- `0a` は10進の10、`14` は20、`1e` は30
- 4バイトずつ、隙間なく並んでいる
- 12バイト目以降は手つかず

### なぜ `0a 00 00 00` なのか

`10` という `int` の値が `0a 00 00 00` と並んでいます。`00 00 00 0a` ではありません。

これは x86/x64 が **リトルエンディアン** だからです。多バイトの数値を、下位バイトから順にメモリへ置きます。

普段は意識せずに済みますが、生メモリを直接ダンプするようになると必ず出会います。「10 を書いたのに 0a が先頭に来ている」と混乱しないよう、ここで一度確認しておきました。

### 「確保前がゼロ」に依存してはいけない

確保前のダンプが全部 `00` でした。これは `std::vector<std::byte>` が要素を値初期化(ゼロ初期化)するためです。

**しかしこれは、たまたまです。**

第29章で `VirtualAlloc` に土台を差し替えたとき、新しく確保したページはやはりゼロで返ってきます(OS がセキュリティのためにゼロクリアします)。しかし、一度使って `Reset()` した領域は前のデータが残ったままです。

「確保したメモリはゼロになっている」という前提でコードを書くと、後で必ず刺されます。第16章では、この危険を炙り出すために、確保時に `0xCD`、解放時に `0xDD` で塗りつぶす仕組みを入れます。そのとき、この `DumpBytes` がそのまま活躍します。

---

## 3.5 目で見るのをやめる

ここまでは「表示して、人間が目で確認する」やり方でした。これは最初のうちは有効ですが、限界があります。

- 毎回、出力を読まなければならない
- 読み飛ばす
- 出力が増えると追えなくなる
- 何より、**壊れたときに気づかない**

コードに検査させましょう。使うのは `assert` だけで十分です。

### `assert` の基本

```cpp
#include <cassert>

assert(条件);
```

条件が偽なら、その場でプログラムを止め、ファイル名・行番号・条件式を表示します。真なら何もしません。

Visual Studio では、Debug 構成で `assert` が失敗するとダイアログが出て、「再試行」を押すとその行でデバッガが止まります。原因の調査に直行できます。

### ⚠ 重要:`assert` は Release では消える

`assert` は `NDEBUG` マクロが定義されていると、**何もしないものに置き換わります**。Visual Studio の Release 構成では `NDEBUG` が定義されているので、`assert` は完全に消滅します。

これは意図された仕様です。製品版で検査コストを払わないための仕組みです。

ただし、2つ注意点があります。

**注意1:テストは Debug 構成で走らせる**

Release でテストを実行しても、`assert` が消えているので何も検査されません。「テストが全部通った!」と喜んでいたら、実は1つも実行されていなかった、という事故が起きます。

本書では次のように使い分けます。

| 目的 | 構成 |
|---|---|
| テストの実行 | **Debug** |
| 速度の計測 | **Release**(第1章参照) |

**注意2:`assert` の中に副作用を書かない**

```cpp
assert(bump.Allocate(4) != nullptr);   // ← 絶対にダメ
```

Release ではこの行ごと消えるので、確保そのものが行われなくなります。Debug と Release で挙動が変わるという、最も見つけにくいバグの типаです。

`assert` の中には **調べるだけ** の式を書いてください。

> Release でも必ず検査したい場面は必ず出てきます。そのための独自マクロは第17章(ガードバイト)と第51章(デバッグ機能の切り離し)で用意します。今は標準の `assert` で進めます。

---

## 3.6 テスト関数を書く

`Bump` の性質を、関数の形で書き下します。

```cpp
#include <cassert>

// -----------------------------------------------------------
// テスト1:初期状態
// -----------------------------------------------------------
void Test_InitialState()
{
    Bump bump(1024);

    assert(bump.Used() == 0);
    assert(bump.Capacity() == 1024);
    assert(bump.Remaining() == 1024);

    std::println("[ OK ] Test_InitialState");
}

// -----------------------------------------------------------
// テスト2:確保するとその分だけ使用量が増える
// -----------------------------------------------------------
void Test_UsedIncreases()
{
    Bump bump(1024);

    bump.Allocate(4);
    assert(bump.Used() == 4);

    bump.Allocate(7);
    assert(bump.Used() == 11);

    bump.Allocate(100);
    assert(bump.Used() == 111);

    assert(bump.Remaining() == 1024 - 111);

    std::println("[ OK ] Test_UsedIncreases");
}

// -----------------------------------------------------------
// テスト3:アドレスが要求サイズぶん連続して並ぶ
// -----------------------------------------------------------
void Test_AddressesAreContiguous()
{
    Bump bump(1024);

    auto* p0 = static_cast<std::byte*>(bump.Allocate(4));
    auto* p1 = static_cast<std::byte*>(bump.Allocate(8));
    auto* p2 = static_cast<std::byte*>(bump.Allocate(16));

    assert(p1 - p0 == 4);
    assert(p2 - p1 == 8);

    std::println("[ OK ] Test_AddressesAreContiguous");
}

// -----------------------------------------------------------
// テスト4:確保した領域は板の中に収まっている
// -----------------------------------------------------------
void Test_AllocationsAreInsideBuffer()
{
    Bump bump(64);

    const std::byte* base = bump.Base();

    auto* p = static_cast<std::byte*>(bump.Allocate(32));

    assert(p >= base);
    assert(p + 32 <= base + bump.Capacity());

    std::println("[ OK ] Test_AllocationsAreInsideBuffer");
}

// -----------------------------------------------------------
// テスト5:書き込んだ値が読み戻せる(領域が重なっていない)
// -----------------------------------------------------------
void Test_ValuesDoNotOverlap()
{
    Bump bump(1024);

    int* a = static_cast<int*>(bump.Allocate(sizeof(int)));
    int* b = static_cast<int*>(bump.Allocate(sizeof(int)));
    int* c = static_cast<int*>(bump.Allocate(sizeof(int)));

    *a = 10;
    *b = 20;
    *c = 30;

    // 後から書いた値が前の値を壊していないこと
    assert(*a == 10);
    assert(*b == 20);
    assert(*c == 30);

    std::println("[ OK ] Test_ValuesDoNotOverlap");
}

// -----------------------------------------------------------
void RunAllTests()
{
    std::println("=== Bump v0.1 テスト ===");

    Test_InitialState();
    Test_UsedIncreases();
    Test_AddressesAreContiguous();
    Test_AllocationsAreInsideBuffer();
    Test_ValuesDoNotOverlap();

    std::println("=== 全テスト成功 ===");
}

int main()
{
    RunAllTests();
}
```

Debug 構成で **Ctrl + F5**。

```
=== Bump v0.1 テスト ===
[ OK ] Test_InitialState
[ OK ] Test_UsedIncreases
[ OK ] Test_AddressesAreContiguous
[ OK ] Test_AllocationsAreInsideBuffer
[ OK ] Test_ValuesDoNotOverlap
=== 全テスト成功 ===
```

### テストの粒度について

テストフレームワークは使いません。関数と `assert` だけです。

理由は、この段階では**それで十分だから**です。外部ライブラリを導入すると、ビルド設定の話が増え、本題から離れます。第50章で本格的なテストとファジングを扱うときに、必要なら道具を足せばいい。

大事なのはフレームワークではなく、「性質を1つずつ言葉にして、コードに書く」という習慣のほうです。

**テスト5に注目してください。** これは「3つの領域が重なっていないこと」を検査しています。当たり前に見えますが、アロケーターにとってはこれこそが最も基本的な契約です。

> **アロケーターの最も基本的な契約:**
> 返した領域どうしは、絶対に重ならない。

第21章以降でフリーリストを実装すると、この契約を破るバグを必ず一度は書きます。そのとき、この形のテストが最初に鳴ります。

---

## 3.7 テストが本当に働くか確かめる

テストを書いたら、**必ず一度は失敗させてください。** 一度も失敗を見ていないテストは、実は何も検査していないかもしれません。

`Allocate` にわざとバグを入れます。ポインタを進める行を消してみましょう。

```cpp
    void* Allocate(std::size_t size)
    {
        std::byte* result = buffer_.data() + offset_;
        // offset_ += size;      ← コメントアウト!
        return result;
    }
```

Debug 構成で実行します。

```
=== Bump v0.1 テスト ===
Assertion failed: bump.Used() == 4, file ...\Playground.cpp, line 42
```

止まりました。ダイアログの「再試行」を押すと、`Test_UsedIncreases` の該当行でデバッガが止まります。

もし `Test_UsedIncreases` を書いていなかったら、このバグはどう現れたでしょうか。すべての確保が同じアドレスを返し、`*a`、`*b`、`*c` が全部同じ場所を指し、最後に書いた `30` だけが読めて、`a = 30, b = 30, c = 30` と表示されたはずです。

そして「なぜ全部30になるのか」を、しばらく悩むことになります。

### 別のバグも試す

もう1つ、よくある書き間違いです。

```cpp
    void* Allocate(std::size_t size)
    {
        offset_ += size;                                // ← 先に進めてしまった
        std::byte* result = buffer_.data() + offset_;
        return result;
    }
```

順番を逆にしただけです。実行すると `Test_AddressesAreContiguous` が落ちるはずです。

このバグは `Used()` の値は正しいままなので、テスト2は通ります。テスト3があって初めて捕まります。**複数の角度からテストを書く意味は、ここにあります。**

確認できたら、`Allocate` を正しい形に戻してください。

---

## 3.8 まだ検出できないもの

テストが5つ揃いました。では、3.1節の「容量16に400バイト書き込む」問題は検出できるでしょうか。

書いてみます。

```cpp
void Test_OverflowIsDetected()
{
    Bump bump(16);

    bump.Allocate(4);
    bump.Allocate(4);
    bump.Allocate(4);
    bump.Allocate(4);   // ここで満杯

    void* p = bump.Allocate(4);   // 溢れる!

    assert(p == nullptr);         // ← 落ちる
}
```

このテストは **失敗します**。`Allocate` は `nullptr` を返さないからです。板の外のアドレスを、平然と返します。

つまり現時点では、**テストを書いても防げません**。検査する仕組みではなく、`Allocate` そのものを直す必要があります。

これが第7章の仕事です。そこで `std::expected` を使って「確保に失敗しうる」という事実を型で表現し、このテストが通るようにします。

ただし、その前に第6章でもう1つ、より根本的な問題を片づけます。今の `Bump` は、`p1 : +4`、`p2 : +11` のような **半端な位置** にアドレスを返しています。11バイト目に `int` を置いてよいのか——という話です。

---

## 3.9 この章の完成コード

`Bump` クラスの最終形です。

```cpp
#include <cassert>
#include <cstddef>
#include <cstdint>
#include <print>
#include <vector>

// ---------------------------------------------------------
// Bump アロケーター v0.1(観測手段つき)
// ---------------------------------------------------------
class Bump
{
public:
    explicit Bump(std::size_t capacity)
        : buffer_(capacity)
    {
    }

    void* Allocate(std::size_t size)
    {
        std::byte* result = buffer_.data() + offset_;
        offset_ += size;
        return result;
    }

    std::size_t Used() const noexcept
    {
        return offset_;
    }

    std::size_t Capacity() const noexcept
    {
        return buffer_.size();
    }

    std::size_t Remaining() const noexcept
    {
        return Capacity() - Used();
    }

    // デバッグ用:板の先頭アドレス
    const std::byte* Base() const noexcept
    {
        return buffer_.data();
    }

private:
    std::vector<std::byte> buffer_;
    std::size_t            offset_ = 0;
};
```

これに `DumpBytes`、`PrintAddress`、5つのテスト関数、`RunAllTests` が付属します。

**この章で `Allocate` は1文字も変えていません。** 足したのは観測と検証の手段だけです。それでも、`Bump` の信頼性はまったく別物になりました。

---

## 演習

**演習3-1** テスト関数をもう1つ書いてください。「`Allocate(0)` を呼んでも `Used()` が変化しない」ことを検査します。実行すると通りますか。

**演習3-2** `Remaining()` を、溢れているときに 0 を返すように直してください。どう書けば安全でしょうか。(ヒント:引き算の前に大小を比べる)

**演習3-3** `DumpBytes` を改造し、16バイトごとに先頭からのオフセットを `+0000:` のような形式で表示するようにしてください。

**演習3-4** `double` を1つ確保して値を書き、`DumpBytes` でバイト列を見てください。`int` のときとどう違いますか。

**演習3-5** 3.7節のように、`Allocate` に別のバグを自分で仕込んでみてください。今の5つのテストで捕まえられないバグを作れますか。作れたとしたら、どんなテストを足せばいいでしょうか。

---

## 章末チェックリスト

- [ ] `Used()` / `Capacity()` / `Remaining()` を実装した
- [ ] 溢れたときに `Remaining()` が巨大な数になることを確認した
- [ ] `DumpBytes` でメモリの中身を見た
- [ ] リトルエンディアンで `0a 00 00 00` と並ぶことを確認した
- [ ] `assert` が Release では消えることを理解した
- [ ] テスト関数を5つ書き、Debug 構成で全部通した
- [ ] **わざとバグを入れて、テストが実際に落ちることを確認した**

---

## 次章の予告

これで `Bump` は「正しく動いている」と言えるようになりました。次に知りたいのは、**速いのかどうか** です。

第4章では、時間を測る道具を作ります。30行ほどのベンチマークヘルパーです。

ここでも1つ、はっきりした方針を立てます。**平均値を信じない**、という方針です。ゲームのようなリアルタイムソフトでは、平均が良くても最悪値が悪ければ、それは画面のカクつきとして目に見えてしまいます。だから中央値と最大値を見ます。

道具ができたら、第5章でいよいよ `new` と勝負します。

---

> **コラム:なぜアドレスは毎回変わるのか**
>
> 3.3節で「アドレスは実行するたびに変わる」と書きました。同じプログラムを同じ環境で動かしているのに、なぜでしょうか。
>
> **ASLR**(Address Space Layout Randomization、アドレス空間配置のランダム化)というセキュリティ機構のためです。プロセスを起動するたびに、実行ファイル・DLL・スタック・ヒープの配置アドレスをランダムにずらします。
>
> 目的は攻撃の防止です。バッファオーバーフローを突く攻撃の多くは、「この関数はこのアドレスにあるはずだ」という前提で成り立っています。毎回配置が変わると、攻撃者は狙いを定められません。Windows では Vista から導入され、現在は既定で有効です。
>
> 開発者にとっては、少し不便でもあります。「前回の実行では 0x7ff6... だったのに」という比較ができません。だから本書では、絶対アドレスではなく **板の先頭からの相対オフセット** で議論します。これなら毎回同じ数字になり、テストにも書けます。
>
> ちなみに、アロケーターを自作する動機の1つにセキュリティがあります。第17章で入れるガードバイト、第31章のガードページは、どちらも元をたどればメモリ破壊攻撃への対抗策として発展してきた技術です。ゲームのアロケーターは速度のために作りますが、そこで使う道具の多くは、まったく別の戦場から来ています。
