# 第5章 happens-beforeを道具として使う

第1部で私たちは、x64とARMという2つの現場を這い回ってきました。ストアバッファ、FIFO、フォワーディング、MESI、RFO――得た知識は本物ですが、ひとつ困ったことがあります。**この知識はアーキテクチャの数だけ増えていく**のです。POWERに移植するたび、RISC-Vが来るたび、新しい機械モデルを解剖し直すのでしょうか。

C++11のメモリモデルは、この問いに対する言語側の回答です。発想はこうです――プログラマとコンパイラの間で、**アーキテクチャに依存しない1つの契約**を結ぶ。プログラマは契約の語彙(この章で学ぶhappens-before)で順序の要求を書く。コンパイラは、その要求をターゲットごとの機械語(x64なら素のmov、ARMならstlr/ldar)に翻訳する責任を負う。契約さえ守れば、プログラマは第1部の知識を**すべて忘れても**正しいコードが書けます。

とはいえ本書は第1部を忘れるために書いたのではありません。この章のもうひとつの狙いは、契約の各条文を読むたびに「ああ、これはあの実験のあれを抽象化した条文だ」と**両側から照らす**ことです。抽象を実験で、実験を抽象で裏打ちする――第1部を通過した読者だけができる読み方です。

---

## 5.1 データ競合の正確な定義 ― 第1章のコードのどこが「競合」だったのか逐条確認

第1章から「データ競合」という言葉を、雰囲気で使い続けてきました。契約書を読むには、まず用語の定義からです。C++規格の定義を、条文の構造を保ったまま平易に書き下します。

> **定義(衝突する操作)**: 2つのメモリ操作が**衝突する(conflict)**とは、同じメモリ位置に対する操作であって、少なくとも一方が書き込みであることをいう。
>
> **定義(データ競合)**: 異なるスレッドに属する2つの衝突する操作があり、**少なくとも一方がアトミック操作でなく**、かつ**どちらも他方にhappens-beforeしない**とき、その実行は**データ競合**を含む。データ競合を含むプログラムの動作は**未定義**である。

6.9.2.1 (C++20)
Two expression evaluations conflict if one of them modifies a memory location (6.7.1) and the other one reads
or modifies the same memory location.

The execution of a program contains a data race if it contains two potentially concurrent conflicting actions,
at least one of which is not atomic, and neither happens before the other, except for the special case for
signal handlers described below.

Two actions are potentially concurrent if
(21.1) — they are performed by different threads, or
(21.2) — they are unsequenced, at least one is performed by a signal handler, and they are not both performed
by the same signal handler invocation.

短い条文ですが、一語一語に重みがあります。逐条で読みましょう。

**「同じメモリ位置」。** 競合は変数単位ではなく**メモリ位置**単位で定義されます。別々の変数、構造体の別々のメンバ、配列の別々の要素は、それぞれ別のメモリ位置です(例外はただひとつ、隣接するビットフィールドの並びで、これは1つのメモリ位置を共有します――章末演習1)。ここで第4章の記憶が疼くはずです。フォールスシェアリング(リスト4-2)では、4スレッドが**同じキャッシュライン**を奪い合っていました。しかし触っていたのは別々の配列要素、つまり**別々のメモリ位置**。したがってあれはデータ競合では**ありません**。遅いだけの、完全に合法なプログラムです(演習2でTSanが沈黙することを確認します)。**競合は論理の概念、フォールスシェアリングは物理の現象**――この線引きが、規格の「メモリ位置」という語の仕事です。

**「少なくとも一方がアトミック操作でなく」。** 裏を返せば、**両方がアトミック操作なら、定義上データ競合になりえません**。メモリオーダーがrelaxedであってもです。第2章のリトマステスト群でrelaxedアトミックを使った本当の理由がこれでした。あのテスト群は、盛大に壊れて見えますが、規格上は**競合ゼロの well-defined なプログラム**であり、観測された(0,0)もr≠42も「未定義動作の産物」ではなく「規格が許容する実行結果のひとつ」です。壊れているのは私たちの期待であって、プログラムの定義性ではありません。

**「どちらも他方にhappens-beforeしない」。** 定義の心臓部に、まだ説明していない用語が埋まっています。直感的な先取りをすれば、happens-beforeとは「一方が他方より前に起きたと、**規格の規則だけから証明できる**」という関係です(5.3節で構築します)。ここで大事なのは、定義が**時刻を一切参照していない**ことです。データ競合とは「同時にアクセスしてしまうこと」では**ありません**。実行のタイミングがたまたま何秒ずれていようと、happens-before関係が**証明できなければ**競合です。競合は実行の性質である以前に、**プログラムの構造の欠陥**なのです(この読み方の威力は5.2節のTSanで体感します)。

**「動作は未定義」。** 第1部の不可解だった観測が、この4文字で一斉に説明されます。第1.2節でコンパイラがループを`counter += 1000000`に畳み込めたのも、第2.4節でロードをループ外に吊り上げて無限ループにできたのも、「データ競合のないプログラムの観測可能な動作を変えない限り何をしてもよい(as-ifルール)。競合があるならそもそも守るべき動作が存在しない」という論理の帰結です。俗に「ベニグンな(無害な)データ競合」――値が多少ずれても実害がない競合――という言い訳を聞くことがありますが、C++にその概念はありません。未定義は未定義です。値がずれるどころか、ループが消え、分岐が消え、プログラムが別物になります。

## 5.2 【コード】ThreadSanitizerに第1〜2章の全コードを食わせて警告を読み解く
それでは第1章の被告人を、この定義で正式に裁いておきます。リスト1-1の`counter++`:スレッド1の書き込みとスレッド2の書き込み(および読み取り)は、同じメモリ位置`counter`への衝突する操作であり、どちらも非アトミックで、両スレッド間にhappens-beforeを作る仕掛け(mutexもatomicもjoin前の同期も)は何ひとつない。**有罪、データ競合、動作未定義。** リスト1-3(volatile版)も、volatileはこの定義のどの語にも影響しないため同罪。リスト1-4(atomic版)は「両方アトミック」の免責により競合なし――これが第1.5節の「直った」の正確な意味です。

---


定義が手に入ったので、定義を機械的に検査してくれる道具を本格導入します。**ThreadSanitizer(TSan)**は、コンパイル時に全メモリアクセスへ計装を織り込み、実行時にhappens-before関係を追跡して、5.1節の定義に該当するアクセス対を報告するツールです。使い方はフラグ1つです。

```console
$ g++ -std=c++20 -O2 -g -pthread -fsanitize=thread race1.cpp -o race1_tsan
$ ./race1_tsan
```

**警告の解剖から始めます。** リスト1-1に対する出力(抜粋)です。

```text
==================
WARNING: ThreadSanitizer: data race (pid=42137)
  Read of size 4 at 0x0000004041a8 by thread T2:            ← ①
    #0 worker() race1.cpp:9 (race1_tsan+0x1295)
  Previous write of size 4 at 0x0000004041a8 by thread T1:   ← ②
    #0 worker() race1.cpp:9 (race1_tsan+0x12a8)
  Location is global 'counter' of size 4 at 0x0000004041a8   ← ③
  Thread T2 (tid=42140, running) created by main thread at:  ← ④
    #0 pthread_create ...
    #1 main race1.cpp:15
SUMMARY: ThreadSanitizer: data race race1.cpp:9 in worker()
==================
```

読み方は定義と一対一です。①と②が**衝突する操作の対**(同じアドレス、一方が書き込み、別スレッド)。③が**メモリ位置**の身元(グローバル変数counter)。④はスレッドの出自で、happens-beforeの探索範囲を示します。TSanは「T1の書き込みとT2の読み取りの間に、追跡してきたhappens-before辺が1本も無い」ことを確認してこの警告を出しています。まさに5.1節の定義の執行官です。

ここで、5.1節の「競合は構造の欠陥である」を実験で裏付けましょう。Nを1000に減らすと、素のrace1は高確率で正しい値2000を出します。

```console
$ ./race1_small          # 計装なし、N=1000
counter = 2000 (期待値 2000)     # 「正しく」動いてしまった
$ ./race1_small_tsan     # TSan計装あり
counter = 2000 (期待値 2000)
==================
WARNING: ThreadSanitizer: data race ...   # それでも検出!
```

> **観測事実13**: TSanは「値が壊れた瞬間」ではなく「happens-beforeの不在」を検出する。**結果がたまたま正しい実行からでも競合を報告できる**。運任せのストレステスト(壊れるまで回す)とは検出原理が根本的に異なる。

第1〜2章の全ソースを一括で法廷に召喚した結果が表5-1です。

**表5-1: 第1部の全コードのTSan判定**

| コード | 内容 | TSan判定 | 5.1節の定義での理由 |
|---|---|---|---|
| リスト1-1 | 素の`counter++` | **競合検出** | 非アトミック同士の衝突、hbなし |
| リスト1-3 | volatile版 | **競合検出** | volatileは定義に無関係 |
| リスト1-4 | atomic版 | 沈黙 | 両方アトミック |
| リスト2-1 | SB(relaxed) | 沈黙 | 両方アトミック |
| リスト2-2 | MP(relaxed) | 沈黙 | 両方アトミック |
| リスト2-3 | シグナルとstopフラグ | **競合検出**(※) | 非アトミックへの非同期アクセス |
| リスト4-2 | フォールスシェアリング | 沈黙 | 別々のメモリ位置=衝突なし |

(※シグナルハンドラ絡みの検出はTSanのオプションと環境に依存します)

この表には教訓と、ひとつの**不吉な空欄**があります。リスト2-2のMPは、ARM実機で100万回に200回壊れたのに、TSanは**沈黙**です。当然です――両方アトミックだから競合ではなく、TSanは競合の検出器だからです。**競合フリーは正しさを意味しません。** relaxedのMPは「未定義動作ではないが、仕様どおりに壊れる」プログラムなのです。ではTSanは順序の誤りには完全に無力なのか? 次の実験が、この章でいちばん面白い答えを出します。

**TSanはメモリオーダーを理解しています。** MPテストの`data_`だけを素の`int`に戻した変種を2つ用意します。

**リスト5-1: mp_plain.cpp ― data_は素のint、readyのメモリオーダーだけを変える**

```cpp
int data_ = 0;                          // わざと非アトミックに戻す
std::atomic<int> ready{0};

// --- 変種R: フラグはrelaxed ---
// 送信側                                // 受信側
data_ = 42;                             while (ready.load(memory_order_relaxed) == 0) {}
ready.store(1, memory_order_relaxed);   r = data_;

// --- 変種A: フラグはrelease/acquire ---
data_ = 42;                             while (ready.load(memory_order_acquire) == 0) {}
ready.store(1, memory_order_release);   r = data_;
```

```console
$ ./mp_plain_R_tsan
WARNING: ThreadSanitizer: data race       # data_ への競合を検出!
  Read of size 4 ... mp_plain.cpp (r = data_)
  Previous write of size 4 ... mp_plain.cpp (data_ = 42)
$ ./mp_plain_A_tsan
(沈黙)
```

**まったく同じ`data_`アクセスが、フラグのメモリオーダー1語で有罪にも無罪にもなりました。** TSanはアトミック操作を見つけると、そのメモリオーダーに応じてhappens-before辺を張るかどうかを決めています。release→acquireの対は辺を張る(だから`data_`の書きと読みの間にhbが通り、無罪)。relaxedの対は張らない(hbが通らず、非アトミックな`data_`同士が衝突して有罪)。

> **観測事実14**: relaxedは、値は運ぶがhappens-beforeを運ばない。release/acquireの対はhappens-beforeを運ぶ。TSanはこの差を正確に追跡しており、**競合検出器であると同時に、happens-before設計の検算器**として使える。

つまりこういう実務技法が成立します――**保護したいデータをあえて非アトミックのまま書き、TSanが黙るまで同期を設計する**。データを片端からatomicにして黙らせるのは、警報機の電池を抜く行為です(リトマステストという特殊目的ではそれが正解でしたが、実務では逆です)。TSanにも限界はあり(実行されなかった経路は見えない、計装で5〜15倍遅くなる等――第11章で検出限界を攻めます)、それでもこの検算器なしで並行コードを書くのは、コンパイラの警告を全部切って書くのに似ています。

---

## 5.3 sequenced-before / synchronizes-with / happens-before ― 図とコードの対応付け

さて、いよいよ契約の中心概念を組み立てます。happens-beforeは一枚岩の定義ではなく、**2種類の細い糸を撚り合わせたロープ**です。

**1本目の糸: sequenced-before(先行実行順)。** 同一スレッド内の順序です。おおざっぱには「ソースコード上の順序」――`a = 1;`の後に`b = 2;`と書けば、前者は後者にsequenced-beforeです(厳密には完全式の内部に未規定順序の領域が残りますが、本書の範囲では文単位の順序と思って差し支えありません)。ここで第2章の記憶と照合してください。sequenced-beforeは**契約上の順序**であって、機械語やCPUがその順に**実行する**約束ではありません。リスト2-4でコンパイラは2つのストアを平然と入れ替えました。あれは契約違反ではないのです――入れ替えがプログラムの定義された観測結果を変えない限り(as-if)、契約は「結果の見え方」しか縛らないからです。

6.9.1 (C++20)
Sequenced before is an asymmetric, transitive, pair-wise relation between evaluations executed by a single
thread (6.9.2), which induces a partial order among those evaluations. Given any two evaluations A and B,
if A is sequenced before B (or, equivalently, B is sequenced after A), then the execution of A shall precede
the execution of B. If A is not sequenced before B and B is not sequenced before A, then A and B are
unsequenced. [Note: The execution of unsequenced evaluations can overlap. —end note] Evaluations A and
B are indeterminately sequenced when either A is sequenced before B or B is sequenced before A, but it is
unspecified which. [Note: Indeterminately sequenced evaluations cannot overlap, but either could be executed
first. —end note] An expression X is said to be sequenced before an expression Y if every value computation
and every side effect associated with the expression X is sequenced before every value computation and every
side effect associated with the expression Y.

**2本目の糸: synchronizes-with(同期辺)。** スレッドを**またぐ**唯一の糸で、これは書けば自動的に生まれるものではなく、**特定の道具を使った箇所にだけ**発生します。主な発生源を挙げます。

Certain library calls synchronize with other library calls performed by another thread. For example, an
atomic store-release synchronizes with a load-acquire that takes its value from the store (31.4). [Note: Except
in the specified cases, reading a later value does not necessarily ensure visibility as described below. Such a
requirement would sometimes interfere with efficient implementation. —end note] [Note: The specifications
of the synchronization operations define when one reads the value written by another. For atomic objects,
the definition is clear. All operations on a given mutex occur in a single total order. Each mutex acquisition
“reads the value written” by the last mutex release. —end note]

**表5-2: synchronizes-with辺のおもな発生源**

| 送り側(辺の根元) | 受け側(辺の先端) | 成立条件 |
|---|---|---|
| `store(v, release)` | `load(acquire)` | ロードが**そのストアの値を読んだ**とき |
| `mutex::unlock()` | 同じmutexの後続の`lock()` | 常に |
| `std::thread`の構築(親側) | スレッド関数の開始(子側) | 常に |
| スレッド関数の終了(子側) | `join()`の完了(親側) | 常に |
| `promise::set_value` | `future::get` | 常に |
| seq_cst操作 | seq_cst操作 | release/acquireとしての条件+全順序S(第7章) |

release→acquireの行の成立条件に注意してください。「releaseストアとacquireロードを書けば辺が張られる」のではなく、「acquireロードが**まさにそのreleaseストアが書いた値を読み取った**とき」に初めて辺が張られます。辺は静的な文の対ではなく、**実行ごとの値の受け渡し**に付随するのです。

**ロープ: happens-before。** 2種類の糸を交互につないだ経路が存在するとき、始点は終点にhappens-beforeする、と定義します(推移的に閉じます)。MPパターン(変種A)で全体像を図解します。

```
   送信スレッド                         受信スレッド

   data_ = 42                ①
      │ sequenced-before
      ▼
   ready.store(1, release)   ②
       ╲
        ╲ synchronizes-with
         ╲ (③が②の書いた1を読んだから)
          ╲
           ▼
   ready.load(acquire) == 1  ③
      │ sequenced-before
      ▼
   r = data_                 ④

結論: ① →sb→ ② →sw→ ③ →sb→ ④ ゆえに ① happens-before ④
```

このロープが張れると、契約は何をくれるのでしょうか。2つの条文が発動します。第一に**可視性**:①が④にhappens-beforeし、①と④の間に`data_`への他の書き込みが挟まらないなら、④は①の書いた42を**読まなければならない**。第二に**競合の免責**:①と④は衝突する非アトミック操作の対ですが、hbが通っているので5.1節の定義に該当せず、競合になりません(TSanが変種Aで沈黙した理由の、これが正式な条文です)。

逆に、ロープのどこか1本が切れると――変種Rのようにrelease/acquireをrelaxedに落とすと――②→③の辺が消え、①と④は「どちらも他方にhbしない衝突対」に転落して競合=未定義動作です。**relaxedは値を運ぶがhbを運ばない**(観測事実14)を、いまや条文の言葉で言えます:relaxedの対はsynchronizes-with辺の発生源リスト(表5-2)に載っていない、と。

強調しておきたい読み筋が2つあります。

**happens-beforeは時刻ではなく証明です。** ①が④より壁時計で1時間早く実行されても、ロープが張れなければhbはありません。逆に、ロープさえ張れれば、CPUが裏でどれだけリオーダリングしようと(第2章)、ストアバッファに何が滞留しようと(第3章)、④は42を見ます。**翻訳の義務はコンパイラ側にあり**、ARMでは②をstlrに、③をldarにして(第3.6.2節の変種Dそのものです)、x64ではTSOの保証で足りるので素のmovのままにして、契約を履行するのです。この翻訳の答え合わせが第8章の主題です。

**契約にはもう1階、土台があります。** 第4.5節で予告した**変更順序(modification order)**です。契約は、各アトミックオブジェクトごとに「そのオブジェクトへの全書き込みの、全スレッドが合意する1本の順序」の存在を無条件に保証し、これを4本の一貫性規則(write-write、read-read、read-write、write-read coherence)で守ります。read-read coherenceは「同じ変数を2回読んで新→旧の順に見えることはない」――第4.5節のCoRRテスト(観測事実12)がARM実機で1億回確認した、まさにあれです。ハードウェアのコヒーレンスが物理で保証していたものを、契約は言語仕様として全アーキテクチャに義務付けている。**変更順序=1つの場所の合意(無償)、happens-before=場所をまたぐ順序(自己申告制)**という第4章の境界線が、契約書の章立てにそのまま写し取られています。

最後に、契約の最上階だけ予告します。プログラムが競合フリーで、かつアトミック操作をすべてデフォルト(seq_cst)で使う限り、**プログラム全体が逐次一貫(SC)に、つまり「全操作を1本に並べたインターリーブ」の直感どおりに振る舞う**ことが保証されます。DRF-SC(Data-Race-Free implies Sequential Consistency)と呼ばれるこの定理こそ、契約がプログラマに支払う最大の対価です。第2章で粉砕されたあの素朴な直感は、契約の条件を満たした者にだけ、返還されるのです。relaxedやacquire/releaseを使うことは、この返還を部分的に辞退して性能を買う行為であり、その値付けの技術が第7章の主題になります。

---

## 5.4 【コード】mutexで直す版とatomicで直す版を並べて、同じhappens-before関係を作っていることを確認する

契約の語彙を手にした仕上げとして、多くの入門者が別物だと思っている2つの道具――mutexとatomic――が、契約上は**同じ形のロープを張る道具**であることを確認します。題材はもちろんMPパターンです。

**リスト5-2: mp_mutex.cpp / mp_atomic.cpp ― 2つの修正版(データはどちらも素のint)**

```cpp
// ===== 版M: mutexで直す =====
int data_ = 0;
bool ready = false;
std::mutex m;

void sender() {
    std::lock_guard<std::mutex> g(m);   // lock
    data_ = 42;
    ready = true;
}                                        // ← unlock(デストラクタ)

void receiver() {
    for (;;) {
        std::lock_guard<std::mutex> g(m);   // lock
        if (ready) { r = data_; return; }
    }                                        // ← unlock
}

// ===== 版A: atomicで直す(5.2節の変種Aと同じ)=====
int data_ = 0;
std::atomic<bool> ready{false};

void sender() {
    data_ = 42;
    ready.store(true, std::memory_order_release);
}
void receiver() {
    while (!ready.load(std::memory_order_acquire)) {}
    r = data_;
}
```

両者とも`data_`は保護対象の素のintのままです。まずTSanの判定:**両版とも沈黙**。ARM実機での破れ:**両版とも0回**。契約上の理由を、2枚の図を重ねて確認します。

```
 版M                                    版A
 data_ = 42                             data_ = 42
    │ sb                                   │ sb
    ▼                                      ▼
 m.unlock()                             ready.store(release)
     ╲ sw(表5-2: unlock→後続lock)           ╲ sw(表5-2: release→acquire、値1の受け渡し)
      ▼                                      ▼
 m.lock() [readyが真だった回]            ready.load(acquire) == true
    │ sb                                   │ sb
    ▼                                      ▼
 r = data_                              r = data_
```

**同じ形です。** 根元がsb、スレッド間を渡る1本のsw、先端がsb。使った道具(表5-2の行)が違うだけで、張られたロープの構造――したがって契約が保証する可視性と競合免責――は同一です。mutexは「相互排他の道具」、atomicは「順序の道具」と分類されがちですが、契約の目には、どちらも**synchronizes-with辺の発生源**という同じ品詞に見えているのです。

品詞が同じなら、翻訳結果も似ているはずです。版MのARM向けバイナリから、ロック解放の中身を覗きます(実装はlibc/OSにより異なります。以下はApple Silicon上のシステムライブラリで観察した典型例の抜粋です)。

```asm
; pthread_mutex_unlock 内部(抜粋・簡略化)
    ...
    stlr   w9, [x8]        ; ← store-release! ロック変数の解放書き込み
    ...
; pthread_mutex_lock 内部(抜粋・簡略化)
    ...
    ldaxr  w9, [x8]        ; ← load-acquire(排他モニタ付き)
    ...
```

見覚えのある命令が出てきました。`stlr`と`ldaxr`(acquire付きの排他ロード)――第3.6.2節で私たちがMPの変種Dを組むのに使ったのと同じ系統の、release/acquire命令です。**mutexとは、release/acquireの対を「待機の仕組み(第9.2節のfutex)と一緒に箱詰めした既製品」**にほかなりません。x64版も覗けば、ロック獲得側に`lock cmpxchg`が見つかります。第3.5.2節で学んだとおり、lock付きRMWは暗黙の全バリア――x64ではそれだけでTSOの唯一の隙間まで塞がってお釣りが来ます。

> **観測事実15**: mutex版とatomic版のMP修正は、(1) TSan判定、(2) ARM実機での挙動、(3) 契約上のロープの形、(4) ARMで発行される順序付き命令の系統、の4点すべてで一致する。mutexは魔法ではなく、release/acquireの既製品パッケージである。

では2つの版は完全に等価なのでしょうか。いいえ、契約外の性質は違います。版Mは受信側がロックを**取り合う**ため、送信側を待たせうる(ブロッキング)。版Aの受信側はスピンで待ち、送信側を一切妨げない。スループット、公平性、優先度逆転への耐性――第3部で延々と比較することになる工学的性質は大きく異なります。しかし**正しさの構造は同じ**。この「正しさと工学の分離」こそ、契約の語彙を学んだ最大の配当です。以後、本書のあらゆる同期コードのレビューは「まず図を描いてロープが繋がっているか(正しさ)、次にその張り方は高いか安いか(工学)」の2段階で進みます。

なお、第1章のカウンタをmutexで直す版とfetch_addで直す版の比較は演習4に回しますが、一点だけ注意を。カウンタの正しさの核心は、可視性のロープに加えて**RMWの不可分性**(第1.3節)と、契約側では「RMWは変更順序上の直前の値を読む」という保証にあります。happens-beforeは万能の一枚岩ではなく、変更順序・不可分性と組み合わせて使う道具立ての一部です。その道具箱の中身を1つずつ手に取るのが、次の第6章です。

---

### 第5章のまとめ

- データ競合の定義は「同じメモリ位置/衝突/非アトミックを含む/**hbなし**」の4条件。時刻や同時性は無関係で、競合は構造の欠陥である。該当すれば未定義動作であり、第1部で見たコンパイラの「暴挙」はすべてこの条文で合法化されていた(5.1節)。
- フォールスシェアリングは別メモリ位置ゆえ競合ではない(論理と物理の分離)。両方アトミックなら定義上競合ではない――relaxedで壊れるMPは「仕様どおりに壊れる合法プログラム」(5.1節)。
- **観測事実13**: TSanはhappens-beforeの不在を検出する。正しい結果を出した実行からでも競合を報告できる。
- **観測事実14**: relaxedは値を運ぶがhbを運ばない。TSanはメモリオーダーを理解しており、happens-before設計の検算器として使える(データを素のまま書き、TSanが黙るまで同期を設計せよ)。
- happens-before = sequenced-before(スレッド内)とsynchronizes-with(表5-2の発生源、値の受け渡しに付随)を撚ったロープ。ロープが通れば可視性と競合免責、切れれば未定義動作(5.3節)。
- 契約の土台には各アトミック変数の変更順序(=第4章のコヒーレンスの言語版)、最上階にはDRF-SC定理(競合フリー+seq_cstなら逐次一貫の直感を返還)がある(5.3節)。
- **観測事実15**: mutexとrelease/acquireは同じ形のロープを張る同じ品詞の道具であり、ARMでは同系統の命令(stlr/ldar系)に翻訳される。正しさの構造と工学的性質を分離して評価せよ(5.4節)。

### 章末演習

1. 隣接ビットフィールド`struct { int a:4; int b:4; };`の`a`と`b`を別スレッドから書き、TSanの判定と`-O2`での実挙動を観察せよ。`int a:4; int c; int b:4;`と間に通常メンバを挟むと判定がどう変わるか。「メモリ位置」の定義を規格の条文(intro.memory)と突き合わせること。
2. リスト4-2(フォールスシェアリング)をTSanでビルド・実行し、沈黙することを確認せよ。「TSanが黙る=速い」ではないことを、第4章の計測結果と並べて一文で述べよ。
3. TSanの検出が**実行された経路**に限られることを示す実験を設計せよ。ヒント: 競合するコードを`if (argc > 5)`の中に置き、通常の引数で実行する。カバレッジとサニタイザの関係について考察を書け。
4. 第1章のカウンタを版M(mutexでlock/++/unlock)と版A(fetch_add(relaxed))で書き、(a) TSan判定、(b) x64とARMでの実行時間、(c) それぞれの正しさの根拠(hbか、変更順序+不可分性か)を比較する表を作れ。relaxedのfetch_addが正しい理由を5.3節の語彙で3行以内で述べよ(第7.4節の予習になる)。
5. `std::promise`/`std::future`で版Aと同じMPを書き、表5-2のどの行が使われたかを図示せよ。さらに`std::condition_variable`版も書き、「cvはmutexのロープに相乗りしている」ことを、spurious wakeup対策のwhileループと絡めて説明せよ。
6. **発展**: 第3.6.2節の変種B(送信release/受信relaxed)を、データを素のintにした形で書き、TSanが競合を報告することを確認せよ。「片側だけの誠意はロープにならない」ことを、synchronizes-withの成立条件(表5-2の1行目)から説明せよ。あわせて、この変種がARM実機で実際に壊れた(観測事実9)こととTSan報告の関係を論ぜよ。
