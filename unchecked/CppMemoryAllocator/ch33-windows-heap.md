# 第33章 Windows ヒープの正体を覗く

---

## この章のゴール

第5章から一貫して、私たちは `new` を相手に戦ってきました。**しかし、`new` が実際に何をしているのかは、まだ調べていません。**

「サイズクラスに丸めて、フリーリストを探して、ロックして……」と説明はしましたが、**中を見たわけではありません。**

この章で、その正体を追います。

```
new → operator new → malloc → HeapAlloc → NtAllocateVirtualMemory
```

- 各層が何をしているかを、実測で分解する
- **Low Fragmentation Heap(LFH)** が何を解いたのか
- **LFH が起動する瞬間** を観測する
- `HEAP_NO_SERIALIZE` で、**スレッド安全性のコスト** を測る
- `HeapWalk` で、プロセスヒープの中身を自分で歩く

そして最後に、第28章のコラムで認めた負債に向き合います。

> **私たちの実装が速いのは、一部はスレッド安全性を捨てた対価である。**

---

## 33.1 層を分解する

`new Foo` と書いたとき、何段の関数を通るか。

```
  new Foo
    │
    ├─ operator new(size)              ← C++ ランタイム
    │     │
    │     ├─ malloc(size)              ← CRT
    │     │     │
    │     │     ├─ HeapAlloc(heap, 0, size)   ← Windows API
    │     │     │     │
    │     │     │     ├─ LFH(前段)     ← 小さいサイズはここで完結
    │     │     │     └─ バックエンド    ← 大きいサイズ、LFH が効かない場合
    │     │     │           │
    │     │     │           └─ NtAllocateVirtualMemory   ← カーネル
    │     │
    │     └─ (失敗時)new_handler → std::bad_alloc
    │
    └─ Foo のコンストラクタ
```

**5層あります。** それぞれの厚みを測ります。

```cpp
constexpr std::size_t kSize = 64;

auto rNew   = bench::MeasureBatch(200, 100'000, [] {
    void* p = ::operator new(kSize);
    bench::Escape(p);
    ::operator delete(p, kSize);
});

auto rMalloc = bench::MeasureBatch(200, 100'000, [] {
    void* p = std::malloc(kSize);
    bench::Escape(p);
    std::free(p);
});

HANDLE processHeap = GetProcessHeap();

auto rHeap = bench::MeasureBatch(200, 100'000, [&] {
    void* p = HeapAlloc(processHeap, 0, kSize);
    bench::Escape(p);
    HeapFree(processHeap, 0, p);
});
```

```
::operator new / delete   median=     17.6 ns
std::malloc / free        median=     16.9 ns
HeapAlloc / HeapFree      median=     15.2 ns
```

### 読み取れること

**`operator new` と `malloc` の差は 0.7 ns。** ほぼ何もしていません。

MSVC の `operator new` は、おおよそこういう実装です。

```cpp
void* operator new(std::size_t size)
{
    for (;;)
    {
        if (void* p = std::malloc(size)) { return p; }

        // 失敗したら new_handler を呼んで、もう一度試す
        std::new_handler handler = std::get_new_handler();
        if (handler == nullptr) { throw std::bad_alloc{}; }
        handler();
    }
}
```

**成功する限り、`malloc` を呼ぶだけです。**

**`malloc` と `HeapAlloc` の差は 1.7 ns。** こちらも薄い層です。

現代の MSVC の CRT では、`malloc` は **プロセスヒープに対する `HeapAlloc` の薄いラッパ** です。

> **昔は違いました。** Visual C++ 6.0 の時代までは、CRT が独自の「スモールブロックヒープ」を持っていて、小さい確保を自前で処理していました。Windows ヒープの性能が改善されたことで、この層は廃止されました。**車輪の再発明をやめた、という判断です。**

**つまり、`new` の重さの大半は `HeapAlloc` にあります。**

---

## 33.2 Windows ヒープの構造

`HeapAlloc` の中身は、大きく2段構えになっています。

```
       HeapAlloc(heap, flags, size)
              │
    ┌─────────┴──────────┐
    │                    │
  前段(LFH)          バックエンド
  小さいサイズ         それ以外
  サイズクラス別       フリーリスト + サイズ別の索引
    │                    │
    └──────┬─────────────┘
           │
    セグメント(VirtualAlloc で確保した大きな領域)
           │
    (非常に大きい確保は、直接 VirtualAlloc へ)
```

### ブロックの構造

Windows ヒープも、私たちが第23章で作ったものと同じ形をしています。

```
┌──────────────┬─────────────────────────┐
│  ヘッダ       │   ユーザーに返す領域      │
│ (16 バイト)   │                         │
└──────────────┴─────────────────────────┘
```

**x64 でのヘッダは 16 バイト、確保の粒度も 16 バイト** です。

第23章で私たちが選んだ「ヘッダ 16 バイト、16 バイト刻み」と、**まったく同じ数字** です。偶然ではありません。アラインメント要求から、自然にこの値に落ち着きます。

### 大きな確保は VirtualAlloc へ

一定サイズを超える要求は、ヒープを経由せず **直接 `VirtualAlloc`** で確保されます(既定のしきい値は 512 KB 前後)。

これを実測してみます。

```cpp
for (std::size_t size : { 1024u, 16384u, 262144u, 524288u, 1048576u, 4194304u })
{
    auto r = bench::MeasureBatch(100, 100, [&] {
        void* p = HeapAlloc(processHeap, 0, size);
        bench::Escape(p);
        HeapFree(processHeap, 0, p);
    });

    std::println("{:>10} : {:>10.1f} ns", ga::FormatBytes(size), r.median);
}
```

```
   1.00 KB :       15.8 ns
  16.00 KB :       21.4 ns
 256.00 KB :       48.2 ns
 512.00 KB :    18400.0 ns   ← ここで段差
   1.00 MB :    32100.0 ns
   4.00 MB :   118000.0 ns
```

**512 KB を境に、380 倍の段差があります。**

`VirtualAlloc` のシステムコールと、解放時の `VirtualFree` が発生するためです。**しかも、解放すると OS に返るので、次に同じサイズを確保するとページフォルトが再発します。**

第5章で観測した「`new` の最大値 412 µs」の正体の一部が、これです。

> **私たちの第29章の設計——予約したまま返さない——が、なぜ有効かが分かります。** OS との往復を避けることで、このスパイクを消しています。

---

## 33.3 LFH が起動する瞬間を観測する

**Low Fragmentation Heap(LFH)** は、Windows XP で導入され、Vista 以降は既定で有効な仕組みです。

### 何を解いたのか

初期の Windows NT のヒープは、**単一のフリーリストと単一のロック** で動いていました。

**問題1:小さい確保が大量にあると、フリーリストが長くなる。** 第23章で私たちが経験したのと同じです。

**問題2:断片化。** サイズがばらばらのブロックが混在し、穴が細切れになります。

**問題3:ロックの競合。** マルチスレッドのアプリケーションでは、全スレッドが1つのロックを取り合います。

### LFH の解決策

**サイズクラスごとに、専用の領域(サブセグメント)を用意します。**

```
サイズクラス 64 バイト → 64 バイトのブロックだけが並ぶ領域
サイズクラス 80 バイト → 80 バイトのブロックだけが並ぶ領域
...
```

**これは、私たちが第21章で作った `Pool` そのものです。**

サイズが揃っているので、断片化が起きません。フリーリストの探索も要りません。さらに、CPU ごと・スレッドごとにキャッシュを持つことで、ロックの競合も減らしています。

**第25章のビン分割と、第21章のプールを組み合わせたもの** と考えると、構造がよく分かります。

### 起動を観測する

**LFH は最初から有効なわけではありません。** 同じサイズの確保がある程度の回数を超えると、「このサイズはよく使われる」と判断して起動します。

**その瞬間を測ってみます。**

```cpp
int main()
{
    // 私有ヒープを作る(プロセスヒープは既に温まっているため)
    HANDLE heap = HeapCreate(0, 0, 0);

    constexpr std::size_t kSize  = 64;
    constexpr std::size_t kCount = 2000;

    std::vector<void*>  ptrs(kCount);
    std::vector<double> ns(kCount);

    for (std::size_t i = 0; i < kCount; ++i)
    {
        const auto t0 = std::chrono::steady_clock::now();
        ptrs[i] = HeapAlloc(heap, 0, kSize);
        const auto t1 = std::chrono::steady_clock::now();

        ns[i] = std::chrono::duration<double, std::nano>(t1 - t0).count();
        bench::Escape(ptrs[i]);
    }

    // 区間ごとの中央値を出す
    auto MedianOf = [&](std::size_t from, std::size_t to) {
        std::vector<double> v(ns.begin() + from, ns.begin() + to);
        std::ranges::sort(v);
        return v[v.size() / 2];
    };

    std::println("  1〜  16 回目 : {:>7.1f} ns", MedianOf(0, 16));
    std::println(" 17〜  32 回目 : {:>7.1f} ns", MedianOf(16, 32));
    std::println(" 33〜 100 回目 : {:>7.1f} ns", MedianOf(32, 100));
    std::println("101〜1000 回目 : {:>7.1f} ns", MedianOf(100, 1000));

    for (void* p : ptrs) { HeapFree(heap, 0, p); }
    HeapDestroy(heap);
}
```

```
  1〜  16 回目 :    42.8 ns
 17〜  32 回目 :    39.1 ns
 33〜 100 回目 :    14.6 ns   ← ここで LFH が起動した
101〜1000 回目 :    13.2 ns
```

**ある時点から、3倍速くなります。**

Windows が「このサイズはよく使われる」と判断し、専用の領域を用意した瞬間です。

### 明示的に有効化する

```cpp
ULONG info = 2;    // 2 = LFH
HeapSetInformation(heap, HeapCompatibilityInformation, &info, sizeof(info));
```

最初から有効にしておけば、起動待ちの遅さを避けられます。

> **実装の詳細は Windows のバージョンによって変わります。** 起動の条件、しきい値、無効化の可否などは、公式にはあまり文書化されていません。**「ある程度使われると速くなる」という挙動があることを知っておくのが実用的です。**

### この挙動が意味すること

**ベンチマークの取り方に影響します。**

```cpp
// 最初の100回だけ測る → LFH が起動していない状態を測ってしまう
```

第4章で「ウォームアップに数回空回しする」と書きました。**Windows ヒープを測るなら、数百回のウォームアップが必要です。**

そして実際のアプリケーションでは、**起動直後は遅い** ということでもあります。ロード中に十分な確保が起きていれば温まりますが、そうでなければ最初の数フレームで LFH の起動コストを払います。

---

## 33.4 スレッド安全性のコストを測る

第28章のコラムで、こう認めました。

> `new` の実装はマルチスレッドで安全に動くよう設計されている。私たちの実装には、その配慮が一切ない。「`new` より速い」と言ってきたが、その一部はスレッド安全性を捨てた対価である。

**測ってみます。**

`HeapCreate` には、**排他制御を行わないヒープ** を作るフラグがあります。

```cpp
HANDLE serialized   = HeapCreate(0, 0, 0);
HANDLE unserialized = HeapCreate(HEAP_NO_SERIALIZE, 0, 0);
```

`HEAP_NO_SERIALIZE` を指定したヒープは、**単一スレッドからしか使ってはいけません**。その代わり、ロックのコストがなくなります。

```cpp
    // 十分にウォームアップしてから測る
    auto rSer = bench::MeasureBatch(200, 100'000, [&] {
        void* p = HeapAlloc(serialized, 0, 64);
        bench::Escape(p);
        HeapFree(serialized, 0, p);
    });

    auto rUnser = bench::MeasureBatch(200, 100'000, [&] {
        void* p = HeapAlloc(unserialized, HEAP_NO_SERIALIZE, 64);
        bench::Escape(p);
        HeapFree(unserialized, HEAP_NO_SERIALIZE, p);
    });
```

```
HeapAlloc(排他制御あり)     median=     15.2 ns
HeapAlloc(排他制御なし)     median=     11.4 ns
```

**3.8 ns がスレッド安全性のコストです。**

### 改めて並べる

```
Bump              1.8 ns   単一スレッド、解放できない
Pool              2.9 ns   単一スレッド、サイズ固定
TLSF             10.2 ns   単一スレッド、汎用
HeapAlloc(NO_SER) 11.4 ns  単一スレッド、汎用
HeapAlloc         15.2 ns  マルチスレッド安全、汎用
malloc            16.9 ns  + CRT の層
new               17.6 ns  + C++ の層
```

### 結論

**同じ条件(単一スレッド、汎用)で比べると、TLSF は 10.2 ns、Windows ヒープは 11.4 ns。差は 12% しかありません。**

第5章から「`new` より 10 倍速い」と言ってきましたが、**汎用アロケーターどうしの比較では、ほぼ互角です。**

**私たちが大きく勝てるのは、`Bump` と `Pool` の場合だけ** です。そしてそれは実装の巧拙ではなく、**制約を選んだ結果** です。

> **Windows ヒープは、よくできています。** 20 年以上かけて磨かれた、真剣な実装です。私たちが 300 行で書いた TLSF が互角なのは、TLSF が優れているからでもありますが、**同じ土俵に立っていないから** でもあります。
>
> 第5部で、この土俵の差を埋めにいきます。

---

## 33.5 ヒープの中身を歩く

Windows は、ヒープの内部を調べる API を提供しています。**第19章で作った可視化を、Windows ヒープに対して行えます。**

```cpp
void WalkProcessHeap()
{
    HANDLE heap = GetProcessHeap();

    if (!HeapLock(heap)) { return; }

    PROCESS_HEAP_ENTRY entry{};
    entry.lpData = nullptr;

    std::size_t busyCount = 0, busyBytes = 0;
    std::size_t freeCount = 0, freeBytes = 0;
    std::size_t largestFree = 0;

    while (HeapWalk(heap, &entry))
    {
        if (entry.wFlags & PROCESS_HEAP_ENTRY_BUSY)
        {
            ++busyCount;
            busyBytes += entry.cbData;
        }
        else if (entry.wFlags & PROCESS_HEAP_UNCOMMITTED_RANGE)
        {
            // 未コミット領域
        }
        else
        {
            ++freeCount;
            freeBytes += entry.cbData;
            if (entry.cbData > largestFree) { largestFree = entry.cbData; }
        }
    }

    HeapUnlock(heap);

    std::println("=== プロセスヒープ ===");
    std::println("  使用中 : {:>6} 個  {}", busyCount, ga::FormatBytes(busyBytes));
    std::println("  空き   : {:>6} 個  {}", freeCount, ga::FormatBytes(freeBytes));
    std::println("  最大空き: {}", ga::FormatBytes(largestFree));

    if (freeBytes > 0)
    {
        const double frag = 1.0 - static_cast<double>(largestFree)
                                / static_cast<double>(freeBytes);
        std::println("  外部断片化: {:.3f}", frag);
    }
}
```

**第19章で定義した外部断片化の指標を、Windows ヒープに適用できます。**

```
=== プロセスヒープ ===
  使用中 :   1842 個  412.30 KB
  空き   :     97 個  148.20 KB
  最大空き: 64.00 KB
  外部断片化: 0.568
```

### 使いどころ

**`HeapLock` している間、他のスレッドはヒープを使えません。** 実行中に呼ぶと止まります。デバッグ時や、終了直前のレポートに限るべきです。

`HeapValidate` を使えば、ヒープの整合性を検査することもできます。**第22章で `VerifyGuards` を作ったのと同じ発想です。**

```cpp
if (!HeapValidate(heap, 0, nullptr))
{
    std::println("ヒープが壊れています");
}
```

---

## 33.6 注意:ヒープは1つではない

実務で問題になる話を書いておきます。

**プロセスには、複数のヒープが存在します。**

```cpp
DWORD count = GetProcessHeaps(0, nullptr);
std::vector<HANDLE> heaps(count);
GetProcessHeaps(count, heaps.data());

std::println("このプロセスのヒープ数: {}", count);
```

```
このプロセスのヒープ数: 7
```

- プロセスヒープ(`GetProcessHeap`)
- CRT が使うヒープ
- 各種 DLL が独自に作ったヒープ
- COM のヒープ

### 静的 CRT と動的 CRT

ここが罠です。

```
/MD : CRT を DLL として使う  → プロセス内で1つの CRT ヒープ
/MT : CRT を静的リンク       → モジュールごとに別の CRT ヒープ
```

**`/MT` でビルドした DLL と EXE がある場合、それぞれが独自のヒープを持ちます。**

```cpp
// DLL 側で確保
void* p = MyDllAllocate();

// EXE 側で解放  ← 別のヒープに返そうとする。クラッシュまたは静かな破壊
free(p);
```

**これは、C++ のライブラリ設計における古典的な落とし穴です。**

**対策:**

- **確保したモジュールで解放する**(DLL に `MyDllFree` を用意する)
- `/MD` に統一する
- そもそも DLL 境界を越えてメモリを渡さない設計にする

**第41章で `operator new` を置き換えるとき、この問題に正面から向き合います。**

---

## 33.7 私たちの実装との対応表

第3部で作ったものと、Windows ヒープの構造を並べます。

| Windows ヒープ | 私たちの実装 | 章 |
|---|---|---|
| ヘッダ 16 バイト、粒度 16 バイト | `FreeListHeader` | 23 |
| フリーリストとサイズ別の索引 | サイズ別ビン | 25 |
| ブロックの分割と合体 | 合体(境界タグ) | 24 |
| **LFH のサイズクラス別領域** | **`Pool`** | 21 |
| 大きな確保を `VirtualAlloc` へ委譲 | `VirtualMemory` | 29 |
| ロックによる排他制御 | **なし** | 第5部で |
| CPU / スレッドごとのキャッシュ | **なし** | 第36章で |
| `HeapValidate` | `VerifyGuards` | 17 |
| `HeapWalk` | `ForEachBlock` | 23 |

**構造はよく似ています。** 30 年の歴史を持つ実装と、私たちが 13 章で書いたものが、同じ部品でできている。

**違うのは、上の2行がまだ空欄であること。**

---

## 演習

**演習33-1** `operator new` / `malloc` / `HeapAlloc` の差を、確保サイズを変えて測ってください。差は一定ですか。

**演習33-2** 33.3 節の LFH 起動実験を、サイズを変えて実行してください。起動のタイミングは変わりますか。

**演習33-3** `HeapSetInformation` で最初から LFH を有効にすると、起動待ちの遅さは消えますか。

**演習33-4** 512 KB の境界を、より細かく探ってください。段差はぴったり 512 KB ですか。

**演習33-5** `HeapWalk` で、確保と解放を大量に行った後のヒープを調べてください。外部断片化はどれくらいですか。第23章の実験と比べてください。

**演習33-6** `HeapValidate` を呼ぶコストを測ってください。実行中に定期的に呼べますか。

**演習33-7** `HEAP_NO_SERIALIZE` のヒープを複数スレッドから使ってみてください。何が起きますか(壊れるので、実験用のプロジェクトで行ってください)。

**演習33-8** `GetProcessHeaps` でヒープを列挙し、それぞれのサイズを調べてください。どのヒープが最も大きいですか。

---

## 章末チェックリスト

- [ ] `new` から `HeapAlloc` までの層を、実測で分解した
- [ ] `operator new` と `malloc` が薄いラッパであることを確認した
- [ ] Windows ヒープのヘッダと粒度が、私たちの実装と同じ数字であることを知った
- [ ] 大きな確保が `VirtualAlloc` に委譲され、段差ができることを確認した
- [ ] **LFH が起動する瞬間** を観測した
- [ ] LFH が `Pool` と同じ発想であることを説明できる
- [ ] `HEAP_NO_SERIALIZE` で、スレッド安全性のコストを測った
- [ ] **同条件では TLSF と Windows ヒープが互角** であることを確認した
- [ ] `/MT` と `/MD` によるヒープの分離という落とし穴を理解した

---

## 次章の予告

第4部が終わり、**第5部(マルチスレッド)** が始まります。

33.4 節で、私たちの実装が「スレッド安全性を捨てた対価」で速いことを認めました。**その負債を返します。**

第34章では、まず **壊してみます**。

```cpp
// 8スレッドから同時に確保する
std::vector<std::thread> threads;
for (int i = 0; i < 8; ++i)
{
    threads.emplace_back([&] {
        for (int k = 0; k < 100'000; ++k) { arena.Allocate(64); }
    });
}
```

**何が起きるか。** `offset_` の更新が競合し、2つのスレッドが同じアドレスを受け取ります。第22章で見た二重解放と同じ、**2つのオブジェクトが同じメモリを共有する** 状態です。

そして、この壊れ方は **再現しません**。実行するたびに違う場所が壊れ、Debug ビルドでは起きず、負荷をかけたときだけ発生する。**最も厄介な種類のバグです。**

第34章から第37章で、ロック、スレッドローカル、そしてロックフリーへと進みます。

---

> **コラム:高性能アロケーターの系譜と、ゲーム業界の選択**
>
> 2000 年代以降、汎用アロケーターは激しく進化しました。**マルチコア化が引き金です。**
>
> ---
>
> **tcmalloc(Google、2005 年頃)**
>
> 名前の由来は "thread-caching malloc"。**スレッドごとにキャッシュを持つ** という発想を広めました。
>
> 各スレッドが自分専用のフリーリストを持ち、そこで完結する限りロックが不要です。足りなくなったら中央のヒープから、まとめて取ってきます。
>
> **第36章で、私たちも同じことをします。**
>
> ---
>
> **jemalloc(2005 年頃〜)**
>
> FreeBSD の標準 malloc として開発され、後に Facebook が大規模に採用・改良しました。
>
> **アリーナ** という概念が中心にあります。ヒープを複数のアリーナに分割し、スレッドをアリーナに割り当てます。競合が減り、断片化も局所化されます。
>
> **この「アリーナ」は、本書で使ってきた意味とは違います。** 第8章のコラムで触れたとおり、同じ語が別のものを指す典型例です。
>
> jemalloc は **断片化への対策** でも知られています。長時間動き続けるサーバーで、メモリ使用量が際限なく増えていく問題に、正面から取り組みました。
>
> ---
>
> **mimalloc(Microsoft、2019 年)**
>
> 比較的新しい実装です。**単純さと性能を両立させる** ことを目指しています。
>
> 特徴的なのは「フリーリストのシャーディング」という工夫です。ページごとに複数のフリーリストを持ち、ローカルな解放とリモートな解放(他スレッドからの解放)を分けます。これにより、アトミック操作の頻度を減らしています。
>
> **コードが読みやすいことでも評価されています。** 実装を勉強する対象としては、最も取り組みやすいでしょう。
>
> ---
>
> **snmalloc(Microsoft Research、2019 年)**
>
> 「メッセージパッシング」の発想を持ち込みました。他スレッドが確保したメモリを解放するとき、直接触るのではなく、**所有者スレッドにメッセージを送ります**。
>
> 生産者・消費者パターン(あるスレッドが確保し、別のスレッドが解放する)に強い設計です。
>
> ---
>
> **では、ゲーム業界はなぜ内製を続けたのか**
>
> これらの実装は、どれも優秀です。実際、`new` を mimalloc に差し替えるだけで速くなるプロジェクトはたくさんあります(第53章で扱います)。
>
> **それでも、ゲームエンジンは自前のアロケーターを持ち続けています。** 理由は複数あります。
>
> **1. 最悪値の保証。** 汎用アロケーターは平均性能を最適化します。第27章で見たとおり、ゲームが欲しいのは最悪値の保証です。
>
> **2. メモリ予算の管理。** 第15章で作ったタグ別集計は、汎用アロケーターでは得られません。「テクスチャが何バイト使っているか」を知る手段が要ります。据置機では、これが必須です。
>
> **3. 寿命の知識。** 第8章から繰り返しているとおりです。「このフレームで全部捨てる」と分かっているなら、汎用アロケーターの仕事の9割は無駄です。
>
> **4. プラットフォームの制約。** ゲーム機の SDK が提供するメモリ API は、PC とは違います。移植性のあるアロケーターを自前で持つほうが、結果的に楽になります。
>
> **5. GPU メモリ。** 第26章と第48章で扱うとおり、GPU 側のメモリは CPU 側の `malloc` では管理できません。**どのみち自前のアロケーターが必要です。**
>
> ---
>
> **とはいえ、全部を自作する必要はありません。**
>
> 実際のエンジンでよくある構成は、こうです。
>
> ```
> フレーム/シーン/一時   → 自作のアリーナ(本書の第2部)
> 同じ型の大量生成       → 自作のプール(本書の第21章)
> GPU メモリ            → 自作(本書の第26章、第48章)
> それ以外の汎用         → mimalloc などを差し替え
> ```
>
> **「分類できるものを自作のアロケーターに割り振り、残りを既製品に任せる。」**
>
> 第28章の決定表が示していたのは、まさにこの構成です。そして第53章で、この判断をもう一度扱います。
