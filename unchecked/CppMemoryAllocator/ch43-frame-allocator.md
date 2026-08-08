# 第43章 フレームアロケーター 〔v0.31〕

---

## この章のゴール

**第7部が始まります。ここまでに作った道具を、ゲームの形に組み上げます。**

最初は、最も基本的で、最もよく使われるパターンです。

```cpp
void UpdateFrame()
{
    auto* commands = frame.Current().NewArrayUninit<DrawCommand>(count);

    UpdatePhysics(frame.Current());
    UpdateAnimation(frame.Current());
    Render(commands);

    frame.EndFrame();      // ← ここで全部捨てる
}
```

**第8章で作った `Reset()` が、ついに本来の用途で使われます。**

- フレーム単位の寿命を持つデータを整理する
- 最も単純な形(アリーナ1本)を実装し、測る
- **前フレームのデータを参照したい** という要求に、ダブルバッファで答える 〔**v0.31**〕
- GPU との同期のために、面数を増やす
- ワーカースレッドごとに持たせる(第34章の回収)
- **フレームをまたいだ参照を、型で検出する**

---

## 43.1 フレームという単位

第3章で、ゲームのメモリを4つの寿命に分類しました。

**この章は、3番目——「フレーム単位」を扱います。**

### 何が該当するか

```
入力処理     → 入力イベントのリスト
ゲームロジック → 更新対象のリスト、イベントキュー
物理演算     → 衝突候補のペア、接触点の配列
アニメーション → ボーン行列、ブレンドの中間結果
カリング     → 可視オブジェクトのリスト
描画準備     → 描画コマンド、ソート用のキー配列
UI          → レイアウト計算の結果
```

**どれも、そのフレームが終われば不要になります。**

そして、**毎フレーム同じように生まれ、同じように死にます。**

### `new` を使うとどうなるか

1フレームで 10,000 回の確保があるとします。第5章と第8章の数字を使います。

```
確保 : 10,000 × 17.6 ns = 176 µs
解放 : 10,000 × 15.2 ns = 152 µs
────────────────────────────────
合計 : 328 µs = 16.6 ms 予算の 2.0%
```

**しかも、これは平均値の話です。** 第5章で見たとおり、`new` には µs 級のスパイクが混ざります。**33 フレームに1回、100 µs 超のスパイクが発生する計算でした。**

### アリーナを使うと

```
確保 : 10,000 × 2.1 ns = 21 µs
解放 : Reset() 1回     ≈ 0 µs
────────────────────────────────
合計 : 21 µs = 16.6 ms 予算の 0.13%
```

**15.6 倍。307 µs の節約です。**

**この本の第2部までの内容が、そのまま答えになっています。**

---

## 43.2 最も単純な形

```cpp
ga::Bump frameArena(64 * 1024 * 1024);

void GameLoop()
{
    while (running)
    {
        UpdateInput(frameArena);
        UpdateGameplay(frameArena);
        UpdatePhysics(frameArena);
        UpdateAnimation(frameArena);
        BuildDrawList(frameArena);
        Render(frameArena);

        frameArena.Reset();      // ← フレームの終わり
    }
}
```

**これだけです。**

### 3つの注意点

**① `NewTrivial` を使う。**

第11章で、破棄リストのコストを測りました。

```
Reset() のコスト(100万オブジェクト):
  自明に破棄可能な型   : 約 0 ns(O(1))
  デストラクタが必要な型: 8.4 ms(O(n))
```

**毎フレーム `Reset()` するアリーナで 8.4 ms は致命的です。**

**フレームアリーナに載せるのは、自明に破棄可能な型だけ** と決めてください。第10章で作った `NewTrivial<T>` が、これをコンパイル時に強制します。

```cpp
auto* cmd = frameArena.NewTrivial<DrawCommand>();   // ← 型で保証される
```

**② 予約を大きく取る。**

第29章の `VirtualMemory` のおかげで、**使わなければ物理メモリを消費しません。**

```cpp
ga::Bump frameArena(256 * 1024 * 1024);   // 256 MB 予約。実際は数 MB
```

**「足りるだろうか」と悩む必要がなくなりました。**

**③ ピークを監視する。**

第8章で作った `Peak()` が、ここで役立ちます。

```cpp
        frameArena.Reset();

        if (frameArena.Peak() > warningThreshold)
        {
            std::println("⚠ フレームメモリのピークが {} に達しました",
                         ga::FormatBytes(frameArena.Peak()));
        }
```

**第49章のメモリ予算に直結する情報です。**

---

## 43.3 問題:前フレームのデータが要る

**単純な形には、致命的な弱点があります。**

```cpp
frameArena.Reset();     // ← 前フレームのデータが全部消える
```

**しかし、前フレームのデータが必要な場面があります。**

### ケース1:補間と速度計算

```cpp
velocity = (currentPos - previousPos) / deltaTime;
```

**前フレームの座標が要ります。**

### ケース2:当たり判定の継続

```
前フレームで接触していたか?
  接触していない → OnCollisionEnter
  接触していた   → OnCollisionStay
今フレームで接触していない + 前フレームで接触していた → OnCollisionExit
```

**前フレームの接触リストと突き合わせる必要があります。**

### ケース3:GPU がまだ読んでいる ← 最も重要

**これが決定的です。**

```cpp
BuildDrawList(frameArena);       // CPU が描画コマンドを書く
SubmitToGPU(commands);           // GPU に投げる
frameArena.Reset();              // ← GPU はまだ読んでいる!
```

**CPU と GPU は並行して動きます。** CPU がフレーム N の処理を終えても、GPU はまだフレーム N-1 や N-2 を描いているかもしれません。

**GPU が読んでいるメモリを上書きすれば、描画がめちゃくちゃになります。**

**しかも、症状が不定です。** 稀に画面が乱れる、特定のマシンでだけ壊れる——**最も追いにくいバグの一種です。**

---

## 43.4 ダブルバッファ

**答えは、アリーナを2面持つことです。**

```
フレーム 0 : 面A に書く。面B は空
フレーム 1 : 面B に書く。面A は「前フレーム」として読める
フレーム 2 : 面A に書く(まず Reset)。面B は「前フレーム」
フレーム 3 : 面B に書く(まず Reset)。面A は「前フレーム」
```

```
        ┌─────────────┐   ┌─────────────┐
        │    面 A      │   │    面 B      │
        └─────────────┘   └─────────────┘
フレーム0    書く              (空)
フレーム1   前フレーム          書く
フレーム2   Reset→書く         前フレーム
フレーム3   前フレーム         Reset→書く
```

**「これから使う面」だけを `Reset` します。** 現在の面は、次のフレームで「前フレーム」として読まれるからです。

### 実装 〔v0.31〕

```cpp
// ga/FrameAllocator.h
#pragma once

#include "ga/Bump.h"

#include <array>
#include <cstdint>
#include <memory>

namespace ga
{
    // N 面を巡回して使うフレームアロケーター
    template <std::size_t N = 2>
    class FrameAllocator
    {
        static_assert(N >= 1, "面数は 1 以上でなければなりません");

    public:
        explicit FrameAllocator(std::size_t bytesPerFrame)
        {
            for (auto& a : arenas_)
            {
                a = std::make_unique<Bump>(bytesPerFrame);
            }
        }

        // 今フレームの書き込み先
        Bump& Current() noexcept { return *arenas_[index_]; }

        // agoFrames フレーム前の面(0 = 今、1 = 前、…)
        const Bump& Frame(std::size_t agoFrames) const noexcept
        {
            assert(agoFrames < N && "その面はすでに再利用されています");
            return *arenas_[(index_ + N - agoFrames) % N];
        }

        // フレームの終わり:面を切り替え、次に使う面をクリアする
        void EndFrame() noexcept
        {
            ++frameNumber_;
            index_ = (index_ + 1) % N;

            arenas_[index_]->Reset();
        }

        std::uint64_t FrameNumber() const noexcept { return frameNumber_; }

        // frame 番号のデータが、まだ生きているか
        bool IsAlive(std::uint64_t frame) const noexcept
        {
            return frameNumber_ >= frame && (frameNumber_ - frame) < N;
        }

        // 全面の合計使用量
        std::size_t TotalUsed() const noexcept
        {
            std::size_t total = 0;
            for (const auto& a : arenas_) { total += a->Used(); }
            return total;
        }

    private:
        std::array<std::unique_ptr<Bump>, N> arenas_;
        std::size_t   index_       = 0;
        std::uint64_t frameNumber_ = 0;
    };
}
```

### 使う

```cpp
ga::FrameAllocator<2> frame(64 * 1024 * 1024);

struct FrameData
{
    std::span<Transform> transforms;
    std::span<Contact>   contacts;
};

FrameData* g_current  = nullptr;
FrameData* g_previous = nullptr;

void UpdateFrame()
{
    g_previous = g_current;

    g_current = frame.Current().NewTrivial<FrameData>().value_or(nullptr);

    g_current->transforms = *frame.Current().NewArrayUninit<Transform>(objectCount);
    g_current->contacts   = *frame.Current().NewArrayUninit<Contact>(maxContacts);

    UpdatePhysics(*g_current, g_previous);   // ← 前フレームを読める

    Render(*g_current);

    frame.EndFrame();
}
```

**前フレームのデータを、そのまま読めます。**

### 代償:メモリが2倍

```
1面 16 MB × 2 = 32 MB
```

**第29章の予約を使えば、実際に触った分しか物理メモリを消費しません。** 予約を 2 倍取ること自体のコストは、ほぼゼロです。

**ただし、両方の面が実際に使われるので、物理メモリは 2 倍必要です。**

---

## 43.5 GPU に合わせて面数を増やす

**43.3 節のケース3が、面数を決めます。**

### CPU と GPU のずれ

```
時刻:      →→→→→→→→→→→→→→→→→→→→→→→→→→→→

CPU:  [フレーム0][フレーム1][フレーム2][フレーム3]
GPU:            [フレーム0][フレーム1][フレーム2]
                 ↑ 1〜2 フレーム遅れる
```

**GPU が実際に描くのは、CPU が指示を出してから 1〜3 フレーム後です。**

パイプラインを深くするほどスループットは上がりますが、**入力からの遅延(レイテンシ)も増えます。** 多くのゲームは 2〜3 フレームのバッファを使います。

### 必要な面数

```
面数 ≥ GPU の遅れフレーム数 + 1
```

**トリプルバッファリング(3面)が一般的です。**

```cpp
ga::FrameAllocator<3> frame(64 * 1024 * 1024);   // 3面 = 192 MB
```

### フェンスで確認する

**「GPU が本当に読み終わったか」は、推測ではなく確認すべきです。**

Direct3D 12 や Vulkan では **フェンス** という仕組みがあります。

```cpp
void EndFrame()
{
    // このフレームの描画完了を待つための番号を記録
    fenceValues_[index_] = SignalGpuFence();

    ++frameNumber_;
    index_ = (index_ + 1) % N;

    // これから使う面の、GPU 処理完了を待つ
    WaitForGpuFence(fenceValues_[index_]);

    arenas_[index_]->Reset();
}
```

**待つ必要があるということは、面数が足りていないということです。** 通常は待たずに済むよう、面数を決めます。

**第48章で、GPU メモリの管理と合わせて扱います。**

---

## 43.6 ワーカースレッドごとに持つ

**第34章の結論を思い出してください。**

> **共有しないことが、最も効果的です。** 8スレッドで 210 倍の差がありました。

### 設計

```cpp
struct WorkerContext
{
    ga::FrameAllocator<2> frame{ 16 * 1024 * 1024 };
};

class JobSystem
{
public:
    explicit JobSystem(std::size_t workerCount)
        : workers_(workerCount)
    {
    }

    ga::Bump& CurrentArena(std::size_t workerIndex) noexcept
    {
        return workers_[workerIndex].frame.Current();
    }

    void EndFrame() noexcept
    {
        for (auto& w : workers_) { w.frame.EndFrame(); }
    }

private:
    std::vector<WorkerContext> workers_;
};
```

**ワーカーごとに独立したフレームアロケーターを持ちます。**

### ジョブの中では `BumpScope`

```cpp
void RunJob(JobSystem& jobs, std::size_t workerIndex, Job& job)
{
    ga::Bump& arena = jobs.CurrentArena(workerIndex);

    ga::BumpScope scope(arena);       // ← 第9章

    job.Execute(arena);

}   // ← ジョブの一時領域だけが返る
```

**フレーム末まで生かしたいデータは、`BumpScope` の外で確保します。**

```cpp
    // フレーム末まで生きる
    auto* result = arena.NewTrivial<JobResult>();

    {
        ga::BumpScope scope(arena);   // 一時的な作業
        job.Execute(arena, *result);
    }
```

### 偽共有への配慮(第35章の回収)

```cpp
struct alignas(std::hardware_destructive_interference_size) WorkerContext
{
    ga::FrameAllocator<2> frame{ 16 * 1024 * 1024 };
};
```

**`std::vector<WorkerContext>` は連続配置されます。** 隣り合うワーカーの状態が同じキャッシュラインに乗ると、第35章で見た **偽共有** が起きます。

**`alignas` で分離してください。**

### スケーリング

```
ワーカー数   1フレームあたりの確保処理
    1              21.0 µs
    2              10.6 µs
    4               5.4 µs
    8               2.8 µs
```

**ほぼ線形です。** 共有していないので、当然の結果です。

### メモリ消費

```
8 ワーカー × 2 面 × 16 MB = 256 MB(予約)
```

**予約なので、実際に触った分だけが物理メモリです。** 第29章の恩恵が、ここで大きく効きます。

---

## 43.7 フレームをまたいだ参照を検出する

**ダブルバッファには、新しい危険があります。**

```cpp
Transform* t = g_current->transforms.data();

// ... 3 フレーム後 ...

t->position.x = 1.0f;      // ← すでに Reset された領域
```

**2面なら、2フレーム後には上書きされます。** 第8章で見た「リセット後のポインタ」問題が、周期的に起きます。

### 第9章の `epoch` を再利用する

**フレーム番号を持たせたポインタを作ります。**

```cpp
// ga/FrameRef.h
#pragma once

#include <cassert>
#include <cstdint>

namespace ga
{
    // フレームアロケーター上のデータを指す、検証つきの参照
    template <class T>
    class FrameRef
    {
    public:
        FrameRef() noexcept = default;

        FrameRef(T* p, std::uint64_t frame) noexcept
            : ptr_(p), frame_(frame)
        {
        }

        // 生きていれば取り出す。死んでいれば nullptr
        template <std::size_t N>
        [[nodiscard]] T* Get(const FrameAllocator<N>& fa) const noexcept
        {
            if (!fa.IsAlive(frame_))
            {
                assert(false && "このフレームのデータはすでに破棄されています");
                return nullptr;
            }
            return ptr_;
        }

        std::uint64_t Frame() const noexcept { return frame_; }

    private:
        T*            ptr_   = nullptr;
        std::uint64_t frame_ = 0;
    };
}
```

**`FrameAllocator` に、`FrameRef` を作る関数を足します。**

```cpp
        template <class T, class... Args>
        [[nodiscard]] FrameRef<T> NewRef(Args&&... args)
        {
            auto r = Current().template NewTrivial<T>(std::forward<Args>(args)...);
            if (!r) { return {}; }

            return FrameRef<T>(*r, frameNumber_);
        }
```

### 使う

```cpp
ga::FrameRef<FrameData> current = frame.NewRef<FrameData>();

// ... 別の場所で ...

if (FrameData* p = current.Get(frame))
{
    p->transforms[0].position.x = 1.0f;
}
```

**古い参照を使えば、`assert` が鳴ります。**

```
Assertion failed: このフレームのデータはすでに破棄されています
```

### コスト

```
生ポインタ           8 バイト
FrameRef<T>         16 バイト
Get() のコスト      0.4 ns(比較2回)
```

**Release で消したい場合は、第14章の方式でマクロ切り替えにします。**

> **第9章の `Marker` に `epoch_` を持たせたのと、まったく同じ発想です。**
>
> 第37章の ABA 問題でも、同じ構造が現れました。**「値は同じだが、意味が変わっている」ことを検出するには、世代番号を持つしかありません。**

---

## 43.8 溢れたときどうするか

**フレームの途中で確保に失敗したら、どうするか。**

### 原則:溢れないように設計する

第7章のコラムで書いたとおりです。

> ゲームのメモリ量は事前に決まっている。足りなくなったのは設計ミスであって、実行時に回復する話ではない。

**そして第29章の予約のおかげで、大きく取ることのコストがほぼゼロになりました。**

```cpp
ga::FrameAllocator<2> frame(512 * 1024 * 1024);   // 512 MB 予約
```

**実質的に溢れません。**

### それでも:緊急脱出路

**製品では、フレーム中にクラッシュするわけにはいきません。**

```cpp
    [[nodiscard]] void* Allocate(std::size_t size, std::size_t alignment) noexcept
    {
        if (auto r = Current().Allocate(size, alignment)) { return *r; }

        // --- 緊急脱出路 ---
        ++overflowCount_;
        overflowBytes_ += size;

        void* p = GlobalHeap().Allocate(size, alignment);
        if (p != nullptr) { overflowBlocks_.push_back(p); }

        return p;
    }

    void EndFrame() noexcept
    {
        // 脱出路で確保したものを、まとめて返す
        for (void* p : overflowBlocks_) { GlobalHeap().Free(p); }
        overflowBlocks_.clear();

        if (overflowCount_ > 0)
        {
            std::println("⚠ フレームメモリが {} 回溢れました({})",
                         overflowCount_, ga::FormatBytes(overflowBytes_));
            overflowCount_ = 0;
            overflowBytes_ = 0;
        }

        // ... 従来の切り替え処理 ...
    }
```

**設計:**

- **落ちない。** グローバルヒープにフォールバックする
- **黙らない。** 必ず警告を出す
- **溜め込まない。** フレーム末に必ず解放する

> **「動くが遅い」状態にして、開発者に気づかせる。** これが実務的な落としどころです。
>
> **警告を無視できないようにしてください。** ログに1行流すだけでは、誰も見ません。画面に赤い文字で出す、ビルドを失敗させる、といった強い手段が必要な場合もあります。

---

## 43.9 測る

### 実際のフレームループを模した測定

```cpp
struct DrawCommand { std::uint64_t sortKey; void* mesh; void* material; std::uint32_t flags; };

void SimulateFrame(ga::Bump& arena, std::size_t objectCount)
{
    auto transforms = arena.NewArrayUninit<Transform>(objectCount);
    auto visible    = arena.NewArrayUninit<std::uint32_t>(objectCount);
    auto commands   = arena.NewArrayUninit<DrawCommand>(objectCount);

    // 小さい確保も多数
    for (std::size_t i = 0; i < objectCount / 10; ++i)
    {
        (void)arena.NewTrivial<Contact>();
    }

    bench::Escape(transforms->data());
    bench::Escape(visible->data());
    bench::Escape(commands->data());
}
```

### 結果(オブジェクト 10,000 個)

```
                          1フレームあたり   16.6 ms 予算に対して
new / delete                   328 µs             1.98%
FrameAllocator(1面)             21 µs             0.13%
FrameAllocator(2面)             21 µs             0.13%
FrameAllocator(8ワーカー)      2.8 µs             0.02%
```

**面数を増やしても、確保のコストは変わりません。** 使う面が変わるだけです。

**307 µs の節約は、16.6 ms 予算の 1.85%。**

### スパイクの比較

第5章のヒストグラムを、フレーム単位で取ります。

```
                       中央値      最大値
new / delete           328 µs     892 µs
FrameAllocator          21 µs      34 µs
```

**最大値で 26 倍の差。**

`new` の 892 µs は、16.6 ms 予算の **5.4%** を1フレームで消費します。他の処理が予算いっぱいなら、**そのフレームは落ちます。**

---

## 演習

**演習43-1** `FrameAllocator<1>` で 43.4 節のコードを動かし、前フレームのデータがどうなるか確認してください。

**演習43-2** `NewTrivial` ではなく `New` を使い、デストラクタを持つ型をフレームアリーナに大量に載せてください。`EndFrame()` の時間はどうなりますか。

**演習43-3** `FrameRef` を使い、3フレーム前のデータにアクセスしてください。`assert` は鳴りますか。

**演習43-4** ワーカーごとの `WorkerContext` から `alignas` を外し、8ワーカーで測ってください。差はありますか。

**演習43-5** 緊急脱出路を実装し、意図的に溢れさせてください。フレーム時間はどれくらい悪化しますか。

**演習43-6** 各フレームのピーク使用量を記録し、1000 フレーム分のグラフを出力してください。どんな形になりますか。

**演習43-7** フレームアリーナの内容を、第19章の可視化で毎フレーム画像に出力してください。動画にすると何が見えますか。

**演習43-8** `FrameAllocator` に `std::pmr::memory_resource` の顔を持たせ、`std::pmr::vector` をフレームアリーナに載せてください。

---

## 章末チェックリスト

- [ ] フレーム単位の寿命を持つデータを列挙できる
- [ ] `new` との差(307 µs / フレーム)を計算した
- [ ] **`NewTrivial` を使うべき理由** を、第11章の測定と結びつけて説明できる
- [ ] 前フレームのデータが必要な3つのケースを挙げられる
- [ ] **GPU がまだ読んでいる** という問題を説明できる
- [ ] ダブルバッファを実装した 〔v0.31〕
- [ ] 「これから使う面」を `Reset` する理由を説明できる
- [ ] 必要な面数の決め方を説明できる
- [ ] ワーカーごとにアロケーターを持ち、偽共有を避けた
- [ ] `FrameRef` でフレームをまたいだ参照を検出した
- [ ] 溢れたときの緊急脱出路を設計した

---

## 次章の予告

**フレームの次は、シーンです。**

```
フレーム : 16.6 ms ごとに全部捨てる
シーン   : 数分〜数十分ごとに全部捨てる
```

**寿命の長さが違うだけで、構造は同じです。** 第44章では、シーン(レベル)単位のアリーナを作ります。

**しかし、いくつか違いがあります。**

- **デストラクタが必要な型を載せられる。** 数分に1回なら、第11章の O(n) も許容範囲です
- **ロード処理と結びつく。** ファイルを読み、パースし、GPU に転送する一連の流れ
- **切り替えの瞬間に、2つのシーンが同時に存在する。** 次のシーンをロードしながら、現在のシーンを動かし続けたい
- **断片化が原理的に起きない。** これがアリーナ方式の最大の利点です

そして、第39章で見た **`std::pmr` の入れ子への自動伝播** が、ここで本領を発揮します。シーンに属するすべてのオブジェクトが、1つのアリーナに載ります。

---

> **コラム:フレームアロケーターは、どこにでもある**
>
> **この章で作ったものは、ほぼすべての商用ゲームエンジンに存在します。** 名前が違うだけです。
>
> ---
>
> **Unreal Engine:`FMemStack`**
>
> Unreal には `FMemStack` というスタック型のアロケーターがあり、`FMemMark` でマーカーを取り、スコープを抜けると巻き戻ります。
>
> **第9章で作った `Marker` と `BumpScope` そのものです。**
>
> レンダリング関連では、フレームごとに使い捨てるメモリのための仕組みが別に用意されています。GPU との同期を考慮した、複数面のバッファリングも行われます。
>
> ---
>
> **Unity:`Allocator.Temp` / `Allocator.TempJob`**
>
> Unity の C# Job System では、`NativeArray` を確保するときにアロケーターの種類を指定します。
>
> ```
> Allocator.Temp     : 1フレーム(実際には数フレーム)で自動的に解放される
> Allocator.TempJob  : 4フレーム以内に解放されることを期待。超えると警告
> Allocator.Persistent : 明示的に解放するまで生きる
> ```
>
> **43.8 節で「警告を無視できないようにしてください」と書きました。** Unity は実際にそうしています。`TempJob` を4フレーム以上保持すると、コンソールに警告が出ます。
>
> **寿命の分類を、API の型として表現している** 好例です。第3章の4分類が、そのまま列挙型になっています。
>
> ---
>
> **id Tech:フレームアロケーター**
>
> id Software のエンジンには、古くからフレーム単位のメモリ管理がありました。**Doom や Quake の時代から、メモリを「ゾーン」に分けて管理する設計が使われていました。**
>
> 第8章のコラムで触れた「ゾーン」という呼び方は、ここから来ています。
>
> ---
>
> **共通する構造**
>
> どのエンジンも、次の要素を持っています。
>
> 1. **バンプアロケーター**(ポインタを進めるだけ)
> 2. **フレーム末の一括リセット**
> 3. **スコープ単位の巻き戻し**(マーカー)
> 4. **複数面のバッファリング**(GPU との同期)
> 5. **ワーカースレッドごとの分離**
>
> **本書の第2章、第8章、第9章、この章で作ってきたものと、完全に一致します。**
>
> ---
>
> **なぜ同じものが何度も作られるのか**
>
> **問題の構造が同じだからです。**
>
> - リアルタイムで動く
> - 明確な周期(フレーム)がある
> - 周期内で生まれ、周期末に死ぬデータが大量にある
> - **最悪実行時間が重要**
>
> この条件が揃えば、**行き着く先は同じです。** ゲームエンジンに限らず、リアルタイム映像処理、音声処理、シミュレーションでも、同じ構造が現れます。
>
> ---
>
> **そして、これは最も単純な部類の技法です**
>
> 第27章の TLSF は 300 行でした。第36章のスレッドキャッシュも複雑でした。
>
> **フレームアロケーターの核心は、`offset_ = 0` の1行です。**
>
> それでいて、実際のゲームで最も大きな効果を出します。**307 µs、予算の 1.85%。** TLSF を投入しても、これほどの改善は得られません。
>
> **問題の構造を理解して、適切な道具を選ぶこと。** 複雑な道具を作ることではありません。
>
> **この本が第2章から繰り返してきた主張が、ここで最も分かりやすい形で現れています。**
