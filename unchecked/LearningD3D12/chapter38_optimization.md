# 第38章 パフォーマンス最適化

**最終章です。**

本書では、繰り返し同じことを書いてきました。

> **最適化は、測定してから行うべきです**(第25章 25.4.4 節)
> **効果は測らなければ分からない**(第35章 35.6.3 節)
> **どちらを優先すべきかは、測ってから決めるべき判断です**(第25章 25.5 節)

**その「測る手段」を、ここで整えます。**

本章では、新しいレンダリング機能を追加しません。**代わりに、これまで作ったものを計測し、分析し、判断する方法を扱います。**

そして最後に、**第8章で保留した DRED** を導入します。第31章の Aftermath と組み合わせることで、デバイスロストの解析が完成します。

**本章のゴール**
GPU タイムスタンプで各パスの時間を測る。ボトルネックを特定する手順を確立する。DRED を導入し、Aftermath と併用する。

---

## 38.1 測る前に

### 38.1.1 何が測定を狂わせるか

**素朴に測ると、値が毎回大きく変わります。**

**原因は 4 つあります。**

| 原因 | 影響 |
|---|---|
| **GPU クロックの変動** | **最大 30% 以上** |
| 他のアプリケーション | 不定 |
| 電源設定 | 大きい |
| 最初の数フレーム | シェーダーのコンパイル、キャッシュの初期化 |

**とくにクロックの変動が厄介です。**

**GPU は、負荷や温度に応じて動作周波数を変えます。** 軽い処理では下がり、重い処理では上がり、熱くなると下がります。

**この状態で測っても、比較になりません。**

### 38.1.2 クロックを固定する

**NVIDIA では、`nvidia-smi` で固定できます**(第2章 2.2.3 節)。

```
nvidia-smi --lock-gpu-clocks=1500,1500
```

**管理者権限が必要です。**

**測定が終わったら解除してください。**

```
nvidia-smi --reset-gpu-clocks
```

**利用可能なクロックを調べるには:**

```
nvidia-smi --query-supported-clocks=gr --format=csv
```

> **Nsight Graphics には自動化機能がある**
>
> GPU Trace を実行すると、**自動的にクロックを固定してから測定します。**
>
> 手動での操作が不要になるので、第29章 29.3 節の手順を使うのが簡単です。

### 38.1.3 測定の手順

**再現性のある測定には、手順が必要です。**

```
① 他のアプリケーションを閉じる
② GPU クロックを固定する
③ アプリを起動し、数秒待つ(ウォームアップ)
④ カメラを固定する
⑤ 100 フレーム以上を測定
⑥ 平均と中央値を取る
```

**④ が重要です。**

**第29章 29.6.2 節で、カメラ位置の保存・復元を勧めました。**

> **「あの角度から見たときだけ壊れる」というバグを、何度でも再現できるようになります。**

**測定でも、同じ仕組みが役立ちます。**

```cpp
//--- ベンチマークモード ---
if (m_benchmarkMode)
{
    LoadCameraState(camera, "benchmark_camera.json");
    m_freezeTime = true;              // 第29章 29.6.2 節
}
```

**⑥ で中央値も取るのは、外れ値の影響を見るためです。**

```
平均 3.92 ms、中央値 3.88 ms  → 安定している
平均 4.51 ms、中央値 3.90 ms  → 時々スパイクがある
```

---

## 38.2 GPU タイムスタンプ

### 38.2.1 クエリヒープ

**GPU の時刻を記録する仕組みです。**

```cpp
D3D12_QUERY_HEAP_DESC desc{};
desc.Type     = D3D12_QUERY_HEAP_TYPE_TIMESTAMP;
desc.Count    = kMaxTimestamps;
desc.NodeMask = 0;

ComPtr<ID3D12QueryHeap> queryHeap;
HR_TRY(device->CreateQueryHeap(&desc, IID_PPV_ARGS(&queryHeap)));

Core::SetDebugName(queryHeap.Get(), L"TimestampQueryHeap");
```

**結果を読み戻すバッファも必要です。**

**READBACK ヒープを使います**(第15章 15.2.1 節、第34章 34.5.2 節)。

```cpp
const auto heapProps = MakeHeapProperties(D3D12_HEAP_TYPE_READBACK);
const auto bufferDesc = MakeBufferDesc(
    kMaxTimestamps * sizeof(std::uint64_t) * kBackBufferCount);

HR_TRY(device->CreateCommittedResource(
    &heapProps, D3D12_HEAP_FLAG_NONE, &bufferDesc,
    D3D12_RESOURCE_STATE_COPY_DEST,      // READBACK は必ずこれ
    nullptr,
    IID_PPV_ARGS(&m_readbackBuffer)));
```

**第15章 15.2.4 節の表を思い出してください。**

| ヒープ | 必要な初期状態 |
|---|---|
| UPLOAD | `GENERIC_READ` |
| **READBACK** | **`COPY_DEST`** |
| DEFAULT | 自由 |

**フレーム数ぶん確保しているのは、第12章 12.2.1 節の判断基準によります。**

> **「GPU がまだ読んでいる可能性があるものは、フレームごとに持つ。」**

### 38.2.2 記録する

```cpp
void GpuProfiler::BeginScope(ID3D12GraphicsCommandList* commandList,
                             std::string_view name)
{
    const UINT index = m_currentIndex++;

    commandList->EndQuery(m_queryHeap.Get(),
                          D3D12_QUERY_TYPE_TIMESTAMP,
                          index);

    m_scopes.push_back({ std::string(name), index, 0 });
    m_scopeStack.push_back(m_scopes.size() - 1);
}

void GpuProfiler::EndScope(ID3D12GraphicsCommandList* commandList)
{
    const UINT index = m_currentIndex++;

    commandList->EndQuery(m_queryHeap.Get(),
                          D3D12_QUERY_TYPE_TIMESTAMP,
                          index);

    m_scopes[m_scopeStack.back()].endIndex = index;
    m_scopeStack.pop_back();
}
```

**`BeginQuery` ではなく `EndQuery` を使う**点に注意してください。

**タイムスタンプクエリは、瞬間の値を記録するだけです。** 開始と終了という概念がないので、`EndQuery` を 2 回呼びます。

**紛らわしい API 設計ですが、仕様です。**

### 38.2.3 RAII でまとめる

**第29章 29.1.2 節の `ScopedEvent`、第31章 31.2.5 節の `ScopedMarker` と同じ形にします。**

```cpp
class ScopedGpuTimer
{
public:
    ScopedGpuTimer(GpuProfiler& profiler,
                   ID3D12GraphicsCommandList* commandList,
                   std::string_view name)
        : m_profiler(profiler)
        , m_commandList(commandList)
    {
        m_profiler.BeginScope(m_commandList, name);
    }

    ~ScopedGpuTimer()
    {
        m_profiler.EndScope(m_commandList);
    }

private:
    GpuProfiler& m_profiler;
    ID3D12GraphicsCommandList* m_commandList;
};
```

**3 種類のマーカーを、1 つのマクロにまとめます。**

```cpp
//---------------------------------------------------------------
// ツール用マーカー(第29章)
// + Aftermath マーカー(第31章)
// + GPU タイマー(第38章)
//---------------------------------------------------------------
class ScopedGpuScope
{
public:
    ScopedGpuScope(ID3D12GraphicsCommandList* commandList,
                   GFSDK_Aftermath_ContextHandle aftermathContext,
                   GpuProfiler& profiler,
                   std::string_view name)
        : m_toolMarker(commandList, Core::ToWide(name))
        , m_aftermathMarker(aftermathContext, name)
        , m_timer(profiler, commandList, name)
    {
    }

private:
    Graphics::ScopedEvent   m_toolMarker;
    Aftermath::ScopedMarker m_aftermathMarker;
    ScopedGpuTimer          m_timer;
};

#define GPU_SCOPE(cmdList, ctx, profiler, name)                     \
    ScopedGpuScope GPU_EVENT_CONCAT(gpuScope_, __LINE__)            \
        { (cmdList), (ctx), (profiler), (name) }
```

**使い方は変わりません。**

```cpp
{
    GPU_SCOPE(m_commandList.Get(), m_aftermathContext, m_profiler,
              "Shadow Pass");
    RenderShadowPass(m_shadowCasters);
}
```

**1 行で、可視化・クラッシュ解析・計測がすべて有効になります。**

### 38.2.4 結果を読み取る

**フレームの終わりに、クエリの結果をバッファへコピーします。**

```cpp
void GpuProfiler::ResolveQueries(ID3D12GraphicsCommandList* commandList,
                                 UINT frameIndex)
{
    if (m_currentIndex == 0) return;

    const UINT64 offset = frameIndex * kMaxTimestamps * sizeof(std::uint64_t);

    commandList->ResolveQueryData(
        m_queryHeap.Get(),
        D3D12_QUERY_TYPE_TIMESTAMP,
        0,                    // 開始インデックス
        m_currentIndex,       // 個数
        m_readbackBuffer.Get(),
        offset);
}
```

**読み取りは、GPU の完了後です。**

**第34章 34.5.2 節と同じく、フェンスで待ちます。**

```cpp
void GpuProfiler::CollectResults(UINT frameIndex, std::uint64_t frequency)
{
    const UINT64 offset = frameIndex * kMaxTimestamps * sizeof(std::uint64_t);

    //--- 読み取る範囲を指定する ---
    const D3D12_RANGE readRange{
        static_cast<SIZE_T>(offset),
        static_cast<SIZE_T>(offset + kMaxTimestamps * sizeof(std::uint64_t))
    };

    void* mapped = nullptr;
    if (FAILED(m_readbackBuffer->Map(0, &readRange, &mapped)))
    {
        return;
    }

    const auto* timestamps =
        reinterpret_cast<const std::uint64_t*>(
            static_cast<const std::byte*>(mapped) + offset);

    for (auto& scope : m_scopes)
    {
        const std::uint64_t begin = timestamps[scope.beginIndex];
        const std::uint64_t end   = timestamps[scope.endIndex];

        //--- ティックを秒へ変換 ---
        scope.milliseconds =
            static_cast<double>(end - begin) / frequency * 1000.0;
    }

    //--- 書き込んでいないので、範囲を空にする ---
    const D3D12_RANGE writeRange{ 0, 0 };
    m_readbackBuffer->Unmap(0, &writeRange);
}
```

**`Map` の第 2 引数に読み取り範囲を指定しています。**

**第15章 15.3.1 節では `{0, 0}` を渡していました。** UPLOAD ヒープでは読まないからです。

**READBACK ヒープでは逆で、読む範囲を正確に指定します。** 指定しないと、ドライバが不要な同期を行う可能性があります。

### 38.2.5 周波数を取得する

**タイムスタンプの単位は「ティック」です。** 秒に変換するには、周波数が必要です。

```cpp
std::uint64_t frequency = 0;
HR_TRY(m_queue->GetTimestampFrequency(&frequency));

LOG_INFO(L"timestamp frequency: {} Hz", frequency);
```

**キューごとに異なる場合があります。**

**第35章でコピーキューやコンピュートキューを導入したなら、それぞれで取得してください。**

> **キューをまたいだ比較はできない**
>
> **異なるキューのタイムスタンプは、同じ時間軸にありません。**
>
> 描画キューとコンピュートキューの時刻を直接比較しても、意味のある結果は得られません。
>
> **キューをまたいだ分析には、Nsight Systems を使ってください**(38.4 節)。

### 38.2.6 統計を取る

**1 フレームの値では、判断できません。**

```cpp
struct ScopeStatistics
{
    std::string name;

    double current = 0.0;
    double average = 0.0;
    double minimum = 1e30;
    double maximum = 0.0;
    double median  = 0.0;

    std::deque<double> history;   // 直近 N フレーム
};
```

```cpp
void GpuProfiler::UpdateStatistics()
{
    for (auto& stats : m_statistics)
    {
        stats.history.push_back(stats.current);
        if (stats.history.size() > kHistorySize)
        {
            stats.history.pop_front();
        }

        //--- 平均 ---
        stats.average = std::accumulate(
            stats.history.begin(), stats.history.end(), 0.0)
            / stats.history.size();

        //--- 最小・最大 ---
        const auto [minIt, maxIt] =
            std::ranges::minmax_element(stats.history);
        stats.minimum = *minIt;
        stats.maximum = *maxIt;

        //--- 中央値 ---
        std::vector<double> sorted(stats.history.begin(), stats.history.end());
        std::ranges::nth_element(sorted, sorted.begin() + sorted.size() / 2);
        stats.median = sorted[sorted.size() / 2];
    }
}
```

**出力例です。**

```
=== GPU Timings (avg over 120 frames) ===
  Shadow Pass        0.42 ms  (min 0.38, max 0.51, med 0.41)
  Opaque             1.85 ms  (min 1.72, max 2.34, med 1.83)
  Transparent        0.31 ms  (min 0.28, max 0.39, med 0.30)
  Resolve MSAA       0.28 ms  (min 0.27, max 0.31, med 0.28)
  Bloom              0.94 ms  (min 0.91, max 1.02, med 0.93)
  Composite          0.12 ms  (min 0.11, max 0.14, med 0.12)
  ─────────────────────────
  Total              3.92 ms
```

**`max` が `median` から大きく離れている場合、スパイクが起きています。**

**原因は、PSO の生成(第14章 14.5 節)、リソースの初回アクセス、メモリの再確保などです。**

---

## 38.3 CPU 側の計測

### 38.3.1 何を測るか

**GPU だけを測っても、全体像は見えません。**

```
CPU: [更新][カリング][記録][投入]
GPU:                       [実行]
```

**CPU が遅ければ、GPU は待つことになります。**

**第29章 29.3.3 節で書いた通りです。**

> **すべてのユニットが低い**場合、GPU は「待って」います。原因は次のいずれかです。
> - **CPU がコマンドを供給できていない**

### 38.3.2 簡易プロファイラ

**第22章 22.2.2 節のタイマを使います。**

```cpp
class CpuProfiler
{
public:
    class Scope
    {
    public:
        Scope(CpuProfiler& profiler, std::string_view name)
            : m_profiler(profiler)
            , m_index(profiler.Begin(name))
        {
        }

        ~Scope() { m_profiler.End(m_index); }

    private:
        CpuProfiler& m_profiler;
        std::size_t  m_index;
    };

    std::size_t Begin(std::string_view name);
    void        End(std::size_t index);

    void BeginFrame();
    void LogResults() const;

private:
    struct Entry
    {
        std::string name;
        std::chrono::steady_clock::time_point start;
        double milliseconds = 0.0;
        int    depth = 0;
    };

    std::vector<Entry> m_entries;
    int m_currentDepth = 0;
};

#define CPU_SCOPE(profiler, name)                                   \
    CpuProfiler::Scope GPU_EVENT_CONCAT(cpuScope_, __LINE__)        \
        { (profiler), (name) }
```

**使い方です。**

```cpp
Core::Status Renderer::RenderFrame(const Camera& camera)
{
    CPU_SCOPE(m_cpuProfiler, "RenderFrame");

    {
        CPU_SCOPE(m_cpuProfiler, "Update Objects");
        UpdateObjects(m_objects, camera, deltaTime);
    }
    {
        CPU_SCOPE(m_cpuProfiler, "Culling");
        CullObjects(camera);
    }
    {
        CPU_SCOPE(m_cpuProfiler, "Record");
        RecordFrameParallel(camera);
    }
    // ...
}
```

**出力例です。**

```
=== CPU Timings ===
  RenderFrame           2.14 ms
    Update Objects      0.31 ms
    Culling             0.08 ms
    Record              1.62 ms
    Submit              0.13 ms
```

**第34章で GPU カリングを導入したなら、`Culling` が極端に小さくなっているはずです。**

### 38.3.3 CPU と GPU のどちらが律速か

**両方の合計を比べます。**

```
CPU: 2.14 ms
GPU: 3.92 ms
     ↓
GPU が律速
```

**ただし、これは単純化しすぎです。**

**第12章でパイプライン化したので、CPU と GPU は並行して動きます。**

```
CPU: [───── 2.14 ms ─────]
GPU:      [────────── 3.92 ms ──────────]
```

**フレーム時間は、遅いほうに支配されます。**

**判断の目安です。**

| 状況 | ボトルネック | 対策の方向 |
|---|---|---|
| CPU ≫ GPU | **CPU** | 記録の並列化(第35章)、GPU 駆動(第34章) |
| CPU ≪ GPU | **GPU** | シェーダー、解像度、描画量 |
| CPU ≈ GPU | 両方 | 全体を見直す |

---

## 38.4 Nsight Systems

### 38.4.1 タイムスタンプでは見えないもの

**38.2 節の計測では、次が見えません。**

| 見えないもの | 理由 |
|---|---|
| **キューをまたいだ関係** | 時間軸が異なる(38.2.5 節のコラム) |
| **CPU と GPU の同期** | フェンスの待機時間 |
| **ドライバ内部の処理** | アプリからは見えない |
| **OS のスケジューリング** | 同上 |

**Nsight Systems が、これを埋めます。**

**第35章 35.7.5 節で使ったツールです。**

### 38.4.2 何が見えるか

```
CPU Main:    [Update][Record][Submit][─── Wait ───]
CPU Worker0:         [Record────────]
CPU Worker1:         [Record────────]

GPU Direct:                  [Shadow][Opaque][Post]
GPU Compute:                 [Particles]
GPU Copy:            [Texture Upload────]
```

**このタイムラインから、多くのことが分かります。**

| 観察 | 意味 |
|---|---|
| **Main の Wait が長い** | GPU が律速 |
| **GPU に隙間がある** | CPU が律速、またはバリア過剰 |
| **Worker の長さが違う** | 負荷が偏っている(第35章) |
| **Compute が重なっていない** | 非同期化が効いていない(第35章 35.6.3 節) |

### 38.4.3 フェンスの待機を可視化する

**Nsight Systems は、フェンスの待機も表示します。**

**第10章で作った `Fence::Wait` が、どれだけ時間を使っているかが見えます。**

```
CPU: [Record][Submit][─── Fence::Wait 8.2 ms ───][Record]
```

**8.2 ms 待っているなら、GPU が明確に律速です。**

**逆に、待機がほぼゼロなら、CPU が追いついていません。**

### 38.4.4 NVTX でアプリ側の情報を渡す

**Nsight Systems には、アプリケーションから注釈を付ける仕組みがあります。**

```cpp
#include <nvtx3/nvToolsExt.h>

void Renderer::RenderFrame(const Camera& camera)
{
    nvtxRangePushA("RenderFrame");

    nvtxRangePushA("Update");
    UpdateObjects(...);
    nvtxRangePop();

    // ...

    nvtxRangePop();
}
```

**38.3.2 節の `CPU_SCOPE` に統合できます。**

```cpp
CpuProfiler::Scope::Scope(CpuProfiler& profiler, std::string_view name)
    : m_profiler(profiler)
    , m_index(profiler.Begin(name))
{
#if defined(USE_NVTX)
    nvtxRangePushA(std::string(name).c_str());
#endif
}
```

**タイムライン上に、自分の付けた名前が表示されるようになります。**

---

## 38.5 ボトルネックの特定

### 38.5.1 手順

**闇雲に最適化しても、効果はありません。**

```
① フレーム全体の時間を測る(38.2 節)
        ↓
② CPU と GPU のどちらが律速かを判断(38.3.3 節)
        ↓
③ GPU なら、パスごとの時間を見る(38.2.6 節)
        ↓
④ 最も重いパスについて、ユニットの稼働率を見る(第29章 29.3 節)
        ↓
⑤ 対策を打つ
        ↓
⑥ 再測定して効果を確認
```

**⑥ を省略しないでください。**

**「速くなったはず」は、しばしば間違っています。**

### 38.5.2 GPU のボトルネック

**第29章 29.3.3 節の表を、再掲します。**

| 高い指標 | ボトルネック | 対策 |
|---|---|---|
| SM Throughput | 演算 | シェーダーを軽く |
| VRAM Bandwidth | メモリ帯域 | 圧縮、解像度 |
| Texture Throughput | テクスチャ読み取り | サンプル数、ミップ |
| ROP Throughput | 出力 | オーバードロー削減 |
| すべて低い | 待っている | バリア、並列化 |

**本書で作ったパスに、具体的な対策を当てはめます。**

#### シャドウパス(第27章)

| ボトルネック | 対策 |
|---|---|
| ROP | 解像度を下げる |
| 頂点処理 | カリング(第34章)、LOD |

**ピクセルシェーダーを省略済み**なので(第27章 27.6.1 節)、既に最適化されています。

#### 不透明パス(第24章〜第28章)

| ボトルネック | 対策 |
|---|---|
| SM | シェーダーの命令数を減らす |
| Texture | ミップマップ、圧縮(第20章 20.7 節) |
| **オーバードロー** | **手前からソート**(第25章 25.5 節) |

**Early-Z が効いているかを確認してください**(第19章 19.1.3 節)。

#### ブルーム(第26章)

| ボトルネック | 対策 |
|---|---|
| Texture | **サンプル数を減らす**(第26章 26.4.2 節) |
| 帯域 | **より小さい解像度で**(第26章 26.4.3 節) |

**バイリニア補間による半減が効いているかを、第26章 Step 5 で確認できます。**

#### レイトレーシング(第37章)

| ボトルネック | 対策 |
|---|---|
| **ダイバージェンス** | **SER**(第37章 37.6 節) |
| 交差判定 | 加速構造の品質、`PREFER_FAST_TRACE` |
| シェーディング | ペイロードを小さく(第37章 37.3.4 節) |

### 38.5.3 Shader Profiler

**第29章 29.4 節で扱いました。**

**シェーダーのどの命令が重いかを特定できます。**

```hlsl
float4 PSMain(VSOutput input) : SV_Target
{
    const float3 N = normalize(input.normalWS);        //  2%
    const float shadow = ComputeShadow(...);           // 45%  ← ここ
    const float4 tex = gDiffuseTexture.Sample(...);    // 30%
    // ...
}
```

**第27章の PCF が重いことが、数値で分かります。**

**対策の候補です。**

| 対策 | 効果 | 副作用 |
|---|---|---|
| カーネルを 3×3 → 1×1 | 大きい | 境界がギザギザ |
| シャドウマップの解像度を下げる | 中程度 | 品質低下 |
| 距離に応じて PCF を切り替える | 中程度 | 実装が複雑 |

**「どこまで品質を落とせるか」は、作るものによって変わります。**

### 38.5.4 解像度スケーリング

**最も確実で、最も効果の大きい対策です。**

```cpp
//--- 内部解像度を下げ、最後に拡大する ---
const UINT renderWidth  = static_cast<UINT>(m_width  * m_renderScale);
const UINT renderHeight = static_cast<UINT>(m_height * m_renderScale);
```

**第26章のオフスクリーン構成が、そのまま使えます。**

```
シーン描画  → 内部解像度(70%)
        ↓
ポストエフェクト → 内部解像度
        ↓
合成         → 画面解像度(拡大)
```

**ピクセル数は解像度の 2 乗に比例します。**

| スケール | ピクセル数 | 予想される改善 |
|---|---|---|
| 100% | 100% | 基準 |
| **80%** | **64%** | **大きい** |
| 70% | 49% | 非常に大きい |
| 50% | 25% | 劇的(品質は明確に低下) |

**ピクセル処理が律速なら、これが最も効きます。**

**頂点処理や CPU が律速なら、効果はありません。**

> **アップスケーリング技術**
>
> 単純な拡大では、ぼやけます。
>
> **DLSS(NVIDIA)、FSR(AMD)、XeSS(Intel)** は、機械学習や時間的な情報を使って高品質に拡大します。
>
> **DLSS は NVAPI が必要なので、本書では扱いません**(第2章 2.5 節)。付録 H に入り口をまとめます。

---

## 38.6 NVIDIA ハードウェアで効きやすい最適化

### 38.6.1 これまでに扱ったもの

**第2章で NVIDIA を対象と定めたことで、具体的な判断ができました。**

| 最適化 | 根拠 | 出典 |
|---|---|---|
| **`numthreads` を 32 の倍数に** | warp = 32 | 第32章 32.3.4 節 |
| **メッシュレットを 64 頂点に** | 同上 | 第36章 36.4.1 節 |
| **Wave Intrinsics で縮約** | warp 内の直接通信 | 第32章 32.4.4 節 |
| **GPU Upload Heaps** | Resizable BAR | 第21章 21.3 節 |
| **SER** | Ada 以降のハードウェア支援 | 第37章 37.6 節 |

**第2章 2.3.2 節の表の通りになりました。**

### 38.6.2 追加で有効なもの

#### 定数バッファよりルート定数

**第18章 18.4.1 節で扱った 3 形態です。**

**少数の値なら、ルート定数が最速です。**

```cpp
//--- 4 個以下の値なら ---
m_commandList->SetGraphicsRoot32BitConstants(2, 4, indices, 0);
```

**間接参照がなく、レジスタに直接載ります。**

#### テクスチャの圧縮

**第20章 20.7 節のブロック圧縮です。**

**帯域が律速の場合、効果が大きくなります。**

| フォーマット | 圧縮率 | 品質 |
|---|---|---|
| `R8G8B8A8_UNORM` | 1:1 | 基準 |
| `BC1` | **6:1** | 低(アルファなし) |
| `BC7` | **4:1** | **高** |
| `BC5` | 4:1 | 法線マップ用 |

**BC7 は品質が高く、汎用的です。**

#### 深度プリパス

**オーバードローが多いシーンで有効です。**

```
① 深度だけを描く(ピクセルシェーダーなし)
② 深度テストを EQUAL にして、色を描く
```

**第27章 27.6.1 節で、シャドウパスに同じ手法を使いました。**

**第19章 19.4.2 節で `LESS_EQUAL` に触れたのも、この用途です。**

> **`LESS_EQUAL` が有用な場面があります。** 同じジオメトリを 2 回描くとき(1 回目で深度だけ書き、2 回目で色を描く「Z プリパス」など)、`LESS` だと 2 回目が必ず落ちてしまいます。

**ただし、頂点処理が 2 回になります。** 効果があるかは測定次第です。

### 38.6.3 効かないことが多いもの

**逆に、期待したほど効かないものもあります。**

| 手法 | 理由 |
|---|---|
| **ドローコールの削減**(数だけ) | D3D12 では既に軽い(第25章 25.3.1 節) |
| **頂点数の削減**(わずか) | 通常はピクセル処理が律速 |
| **分岐の削除**(一様な分岐) | 一様なら分岐は安い |
| **`half` の使用** | 精度が落ちる割に効果が小さい場合が多い |

**測定なしに「速くなるはず」と思い込むのが、最も危険です。**

---

## 38.7 DRED

### 38.7.1 第8章で保留したもの

**第7章 7.1.4 節で、こう書きました。**

> DRED(Device Removed Extended Data)は、デバイスロスト時に「GPU が最後にどこまで実行したか」を記録する仕組みです。**第38章で扱います**が、**有効化の位置は本節と同じ、デバイス生成の前**です。
>
> 本書は次章で Aftermath を導入し、そちらを主たる解析手段とします。役割が重なるため DRED は第38章まで保留しますが、**「有効化するならここ」ということだけ覚えておいてください。**

**ここで導入します。**

### 38.7.2 Aftermath との違い

| | DRED | Aftermath |
|---|---|---|
| 提供元 | **Microsoft** | NVIDIA |
| 動作環境 | **すべての GPU** | NVIDIA のみ |
| 取得できる情報 | ブレッドクラム、ページフォルト | **+ シェーダーの行番号** |
| ダンプファイル | なし(その場で読む) | **`.nv-gpudmp`** |
| 解析ツール | PIX、自前 | Nsight Graphics |

**Aftermath のほうが情報量は多いです。**

**しかし、DRED には利点があります。**

| 利点 | 説明 |
|---|---|
| **追加の SDK が不要** | D3D12 に組み込まれている |
| **ベンダ非依存** | AMD、Intel でも動く |
| **その場で読める** | ファイルの往復が不要 |

**両方を有効にして、併用するのが最善です。**

### 38.7.3 有効にする

**デバイス生成の前です。**

```cpp
Core::Status EnableDred()
{
    ComPtr<ID3D12DeviceRemovedExtendedDataSettings1> dredSettings;

    if (FAILED(::D3D12GetDebugInterface(IID_PPV_ARGS(&dredSettings))))
    {
        LOG_WARN(L"DRED is not available");
        return {};
    }

    //--- 自動ブレッドクラム ---
    dredSettings->SetAutoBreadcrumbsEnablement(
        D3D12_DRED_ENABLEMENT_FORCED_ON);

    //--- ページフォルト情報 ---
    dredSettings->SetPageFaultEnablement(
        D3D12_DRED_ENABLEMENT_FORCED_ON);

    //--- ブレッドクラムのコンテキスト(1.1 以降)---
    dredSettings->SetBreadcrumbContextEnablement(
        D3D12_DRED_ENABLEMENT_FORCED_ON);

    LOG_INFO(L"DRED enabled");
    return {};
}
```

**`ID3D12DeviceRemovedExtendedDataSettings1` は、コンテキスト機能を持つ版です。**

**第6章 6.1.4 節のインターフェースバージョンです。**

### 38.7.4 ブレッドクラム

**「GPU がどこまで実行したか」を記録します。**

**第31章 31.2 節の Aftermath マーカーと、同じ発想です。**

**ただし DRED は、自動的に記録します。**

```cpp
D3D12_DRED_ENABLEMENT_FORCED_ON
```

**このフラグにより、すべての描画コマンドが記録されます。**

**明示的なマーカーも追加できます。**

```cpp
commandList2->WriteBufferImmediate(...);   // 手動のブレッドクラム
```

**本書では、第31章のマーカーで代用します。** 二重に管理する意味がありません。

### 38.7.5 デバイスロスト時に読む

**第10章 10.5.4 節で作った `OnDeviceLost` を拡張します。**

```cpp
void OnDeviceLost(ID3D12Device* device, HRESULT reason)
{
    LOG_FATAL(L"=== DEVICE LOST ===");
    LOG_FATAL(L"reason: {}", Core::FormatHResult(reason));

    //--- ① DRED を読む(その場で読める)---
    LogDredInfo(device);

    //--- ② Aftermath のダンプ生成を待つ(第31章)---
    Aftermath::WaitForCrashDump();
}
```

```cpp
void LogDredInfo(ID3D12Device* device)
{
    ComPtr<ID3D12DeviceRemovedExtendedData1> dred;
    if (FAILED(device->QueryInterface(IID_PPV_ARGS(&dred))))
    {
        LOG_WARN(L"DRED data is not available");
        return;
    }

    //--- ブレッドクラム ---
    D3D12_DRED_AUTO_BREADCRUMBS_OUTPUT1 breadcrumbs{};
    if (SUCCEEDED(dred->GetAutoBreadcrumbsOutput1(&breadcrumbs)))
    {
        LOG_FATAL(L"--- DRED Breadcrumbs ---");

        const auto* node = breadcrumbs.pHeadAutoBreadcrumbNode;
        while (node != nullptr)
        {
            LogBreadcrumbNode(*node);
            node = node->pNext;
        }
    }

    //--- ページフォルト ---
    D3D12_DRED_PAGE_FAULT_OUTPUT2 pageFault{};
    if (SUCCEEDED(dred->GetPageFaultAllocationOutput2(&pageFault)))
    {
        LOG_FATAL(L"--- DRED Page Fault ---");
        LOG_FATAL(L"  VA: {:#018x}", pageFault.PageFaultVA);

        LogAllocationNodes(L"Existing",
                           pageFault.pHeadExistingAllocationNode);
        LogAllocationNodes(L"Recently freed",
                           pageFault.pHeadRecentFreedAllocationNode);
    }
}
```

**ブレッドクラムの読み方です。**

```cpp
void LogBreadcrumbNode(const D3D12_AUTO_BREADCRUMB_NODE1& node)
{
    const auto* listName = node.pCommandListDebugNameW;
    const auto* queueName = node.pCommandQueueDebugNameW;

    LOG_FATAL(L"  Queue: {}, List: {}",
              queueName ? queueName : L"<unnamed>",
              listName  ? listName  : L"<unnamed>");

    const UINT lastCompleted = *node.pLastBreadcrumbValue;
    const UINT total = node.BreadcrumbCount;

    LOG_FATAL(L"    completed {} / {} operations", lastCompleted, total);

    //--- 完了した最後の数個と、実行中のものを表示 ---
    const UINT begin = (lastCompleted > 5) ? lastCompleted - 5 : 0;
    const UINT end   = std::min(lastCompleted + 3, total);

    for (UINT i = begin; i < end; ++i)
    {
        const auto op = node.pCommandHistory[i];
        const wchar_t* status = (i < lastCompleted) ? L"done" : L"PENDING";

        LOG_FATAL(L"    [{}] {} : {}",
                  i, ToString(op), status);
    }
}
```

**`pCommandListDebugNameW` と `pCommandQueueDebugNameW` に注目してください。**

**第6章 6.5 節で名前を付けた成果が、ここでも現れます。**

**第31章 31.4.2 節、第21章 21.5.2 節に続いて、3 度目です。**

**出力例です。**

```
[Fatal] DeviceLost.cpp(88): --- DRED Breadcrumbs ---
[Fatal] DeviceLost.cpp(94):   Queue: DirectQueue, List: WorkerList[2]
[Fatal] DeviceLost.cpp(101):     completed 187 / 312 operations
[Fatal] DeviceLost.cpp(112):     [182] DrawIndexedInstanced : done
[Fatal] DeviceLost.cpp(112):     [183] SetGraphicsRootConstantBufferView : done
[Fatal] DeviceLost.cpp(112):     [184] DrawIndexedInstanced : done
[Fatal] DeviceLost.cpp(112):     [185] SetPipelineState : done
[Fatal] DeviceLost.cpp(112):     [186] DrawIndexedInstanced : done
[Fatal] DeviceLost.cpp(112):     [187] Dispatch : PENDING          ← ここで停止
```

**どのコマンドで止まったかが、正確に分かります。**

**第35章で並列記録を導入したので、`WorkerList[2]` という名前も役立ちます。**

### 38.7.6 両方を使う

**それぞれの強みを活かします。**

```
デバイスロストが発生
        ↓
① DRED:どのコマンドで止まったか
        ↓
② Aftermath:シェーダーのどの行か
```

**DRED で「Dispatch で止まった」と分かり、Aftermath で「`ParticleUpdate.hlsl` の 78 行目」と分かります。**

**第31章 31.5.4 節で作ったクラッシュテストで、両方の出力を確認できます。**

---

## 38.8 よくあるクラッシュ原因

### 38.8.1 一覧

**本書で扱った内容を、症状から逆引きできる形にまとめます。**

| 症状 | 疑うべき原因 | 参照 |
|---|---|---|
| **起動直後にクラッシュ** | Agility SDK の設定 | 第4章 4.3 節 |
| | デバイス生成の失敗 | 第7章 |
| **描画開始時にクラッシュ** | PSO の設定誤り | 第14章 14.4.3 節 |
| | ルートシグネチャとの不一致 | 第14章 14.2.3 節 |
| **数フレーム後にクラッシュ** | 中間バッファの寿命 | 第16章 16.3.4 節 |
| | フレームリソースの共有 | 第12章 12.2 節 |
| **たまにクラッシュ** | バリア不足 | 第30章 30.7 節 |
| | データ競合(マルチスレッド) | 第35章 35.7 節 |
| **終了時にクラッシュ** | GPU の完了を待っていない | 第10章 10.4 節 |
| **リサイズでクラッシュ** | バックバッファの参照が残存 | 第12章 12.4.2 節 |
| | 0×0 サイズ | 第12章 12.4.5 節 |
| **ページフォルト** | 範囲外アクセス | 第32章 32.3.3 節 |
| | 無効なデスクリプタ | 第33章 33.5.2 節 |
| **TDR(タイムアウト)** | 無限ループ | 第31章 31.5.1 節 |
| | 再帰の深さ超過 | 第37章 37.3.5 節 |
| | 描画量が多すぎる | 38.5 節 |

### 38.8.2 切り分けの手順

```
① デバッグレイヤーは何と言っているか        第7章 7.6 節
        ↓
② GPU-Based Validation で再現するか          第30章 30.6 節
        ↓
③ DRED はどのコマンドを指しているか          38.7 節
        ↓
④ Aftermath はどの行を指しているか            第31章 31.4 節
        ↓
⑤ 該当箇所を Nsight Graphics で確認           第29章
```

**①で解決することが、実は最も多いです。**

**第7章 7.6.1 節で `SetBreakOnSeverity` を設定したのは、このためでした。**

### 38.8.3 再現しないバグへの対処

**最も厄介なのが、「たまにしか起きない」バグです。**

**対策を 4 つ挙げます。**

**対策 1:強制的に条件を悪化させる**

```cpp
//--- 全同期を強制(第30章 30.7.2 節)---
m_forceFullBarriers = true;

//--- スレッド数を最大に(第35章)---
config.recordingThreadCount = 16;

//--- GPU-Based Validation(第30章 30.6 節)---
config.enableGpuBasedValidation = true;
```

**「直る」なら、その領域に原因があります。**

**対策 2:長時間実行する**

```cpp
//--- 自動でカメラを動かし続ける ---
if (m_stressTest)
{
    camera.SetPosition(GenerateRandomPosition(m_frameCount));
}
```

**数時間放置すれば、再現率が上がります。**

**対策 3:ログを常時記録する**

**第6章 6.4 節のログを、ファイルへ出力します。**

```cpp
void Log::WriteRaw(LogLevel level, ...)
{
    ::OutputDebugStringW(line.c_str());

#if defined(ENABLE_FILE_LOG)
    WriteToLogFile(line);
#endif
}
```

**クラッシュ直前の状況が残ります。**

**対策 4:ダンプを回収する**

**第31章 31.6.3 節の仕組みです。**

**自分の環境で再現しなくても、他の環境のダンプから分かることがあります。**

---

## ✅ 本章のゴール:測定して判断できる

### Step 1:クロックを固定する

```
nvidia-smi --lock-gpu-clocks=1500,1500
```

**固定前後で、測定値のばらつきを比べてください。**

```
固定なし:  3.21 〜 4.87 ms(変動 52%)
固定あり:  3.88 〜 3.96 ms(変動 2%)
```

**この差が、38.1.1 節で述べた問題です。**

**測定後は解除してください。**

### Step 2:GPU タイムスタンプを実装する

```
=== GPU Timings (avg over 120 frames) ===
  Shadow Pass        0.42 ms
  Opaque             1.85 ms
  Transparent        0.31 ms
  Resolve MSAA       0.28 ms
  Bloom              0.94 ms
  Composite          0.12 ms
  ─────────────────────────
  Total              3.92 ms
```

**Nsight Graphics の GPU Trace(第29章 29.3 節)と比較してください。**

**近い値になるはずです。**

**大きくずれる場合、クエリの位置を確認してください。**

### Step 3:CPU 側も測る

```
=== CPU Timings ===
  RenderFrame           2.14 ms
    Update Objects      0.31 ms
    Culling             0.08 ms
    Record              1.62 ms
```

**38.3.3 節の判断基準で、どちらが律速かを determine してください。**

### Step 4:解像度スケーリングを試す

```cpp
m_renderScale = 0.7f;
```

**ピクセル処理が律速なら、大幅に改善します。**

| スケール | GPU 時間 |
|---|---|
| 100% | 3.92 ms |
| 80% | ? |
| 70% | ? |
| 50% | ? |

**表を埋めてみてください。**

**2 乗に比例する変化になっていれば、ピクセル処理が律速です。**

**比例していなければ、他の要因があります。**

### Step 5:最も重いパスを最適化する

**Step 2 の結果から、最も重いパスを選びます。**

**第29章 29.3 節の GPU Trace で、どのユニットが高いかを確認します。**

**38.5.2 節の表から対策を選び、実施します。**

**再測定して、効果を確認してください。**

### Step 6:効かない最適化を試す

**38.6.3 節の「効かないことが多いもの」を、実際に試してください。**

```cpp
//--- ドローコールを半分にする ---
// (インスタンシングをより積極的に使う)
```

**測定値がほとんど変わらないことを確認してください。**

**「速くなるはず」という思い込みが、いかに当てにならないかが分かります。**

### Step 7:DRED を有効にする

```
[Info ] GraphicsDevice.cpp(48): DRED enabled
```

**第31章 31.5 節のクラッシュテストを実行してください。**

```
[Fatal] DeviceLost.cpp(88): --- DRED Breadcrumbs ---
[Fatal] DeviceLost.cpp(94):   Queue: DirectQueue, List: MainCommandList
[Fatal] DeviceLost.cpp(101):     completed 12 / 15 operations
[Fatal] DeviceLost.cpp(112):     [12] Dispatch : PENDING
```

**Aftermath の出力と併せて確認してください。**

| ツール | 分かること |
|---|---|
| **DRED** | **Dispatch で止まった** |
| **Aftermath** | **`CrashTest.hlsl` の 14 行目** |

**両方あって、初めて完全な情報になります。**

### Step 8:名前を消してみる

```cpp
#define D3D12_ENABLE_DEBUG_NAMES 0
```

**DRED の出力を確認してください。**

```
Queue: <unnamed>, List: <unnamed>
```

**第31章 Step 8、第21章 Step 4 に続いて、3 度目の確認です。**

**元に戻してください。**

### Step 9:ストレステストを実行する

**38.8.3 節の対策 2 です。**

```cpp
config.enableGpuBasedValidation = true;
config.recordingThreadCount     = 16;
m_forceFullBarriers             = false;
m_stressTest                    = true;
```

**30 分以上放置してください。**

**エラーが出なければ、実装は安定しています。**

---

### 本章の達成状態

- [ ] GPU クロックを固定して測定した
- [ ] タイムスタンプクエリを実装した
- [ ] `EndQuery` を 2 回呼んでいる(`BeginQuery` ではない)
- [ ] READBACK ヒープで結果を読み取った
- [ ] `Map` の読み取り範囲を指定した
- [ ] 統計(平均・中央値・最大)を取った
- [ ] CPU 側も計測した
- [ ] 3 種類のマーカーを 1 つのマクロに統合した
- [ ] Nsight Systems でタイムラインを確認した
- [ ] ボトルネックを特定した
- [ ] 対策を打ち、再測定した
- [ ] DRED を有効にした
- [ ] DRED と Aftermath を併用した
- [ ] **測定に基づいて判断できるようになった**

---

## トラブルシューティング

| 症状 | 原因 | 対処 |
|---|---|---|
| 測定値が毎回大きく違う | クロックの変動 | 38.1.2 |
| タイムスタンプが 0 | `ResolveQueryData` を呼んでいない | 38.2.4 |
| 同上 | GPU の完了を待っていない | 第34章 34.5.2 節 |
| 値が異常に大きい | 周波数の取得誤り | 38.2.5 |
| キュー間で辻褄が合わない | 時間軸が異なる | 38.2.5 のコラム |
| CPU 時間が短すぎる | 非同期実行の誤解 | 38.3.3 |
| 最適化しても速くならない | ボトルネックが別の場所 | 38.5.1 |
| DRED が使えない | 開発者モード | 第4章 4.6.1 節 |
| ブレッドクラムが空 | 有効化の位置が誤り | 38.7.3 |
| 名前が `<unnamed>` | `SetName` を呼んでいない | 第6章 6.5 節 |

---

## まとめ

**1. 測る前に、環境を整える。**
GPU クロックの変動は、最大 30% 以上の誤差を生みます。**固定しなければ、比較になりません。**

**2. タイムスタンプは `EndQuery` を 2 回。**
`BeginQuery` は使いません。紛らわしい API ですが、瞬間の値を記録する仕組みだからです。

**3. 統計を取る。**
1 フレームの値では判断できません。**平均と中央値の差が、スパイクの有無を示します。**

**4. CPU と GPU の両方を見る。**
片方だけでは、どちらが律速か分かりません。**Nsight Systems なら、待機時間まで見えます。**

**5. 手順に従って絞り込む。**
全体 → CPU/GPU → パス → ユニット → 対策 → **再測定**。最後を省略しないでください。

**6. 解像度スケーリングが最も確実。**
ピクセル数は解像度の 2 乗に比例します。**ピクセル処理が律速なら、これが最も効きます。**

**7. 「速くなるはず」は当てにならない。**
第25章から繰り返してきた「測ってから判断する」を、実践する手段が揃いました。

**8. DRED と Aftermath は併用する。**
DRED が「どのコマンドか」、Aftermath が「どの行か」。**両方あって、初めて完全な情報になります。**

**9. 名前が、3 度目の価値を発揮した。**
第21章のリーク検出、第31章のクラッシュ解析、そして本章の DRED。**第6章 6.5 節で決めた習慣が、最後まで効き続けました。**

---

## 本書を終えて

**38 章、お疲れさまでした。**

**第1章 1.3 節で、こう書きました。**

> **使わない:** d3dx12.h、DirectXMath、DirectXTK12、WIC、Assimp

**その代償として自作したものを、数えてみます。**

### 自作したヘルパー

```
OffsetHandle (CPU / GPU)             第11章
MakeTransitionBarrier                第11章
DefaultRasterizerDesc                第14章
DefaultBlendDesc                     第14章
DefaultDepthStencilDesc              第14章
DefaultGraphicsPipelineStateDesc     第14章
MakeHeapProperties                   第15章
MakeBufferDesc                       第15章
AlignUp                              第18章
MakeTexture2DDesc                    第19章
AlphaBlendDesc                       第28章
AdditiveBlendDesc                    第28章
MakeTextureBarrier                   第30章
MakeBufferBarrier                    第30章
MakeUavBufferBarrier                 第32章
PipelineStateSubobject               第36章
DefaultMeshShaderPipelineStateStream 第36章
StateObjectBuilder                   第37章
```

**18 個。ファイル数枚に収まりました。**

### 自作したライブラリ

| 機能 | 章 | 代替されたもの |
|---|---|---|
| COM ヘルパー | 第6章 | (ComPtr は使用) |
| ログとアサート | 第6章 | — |
| 数学ライブラリ | 第17章 | **DirectXMath** |
| DDS ローダ | 第20章 | **WIC / DirectXTex** |
| サブリソース転送 | 第20章 | **UpdateSubresources** |
| デスクリプタ管理 | 第21章 | — |
| OBJ ローダ | 第23章 | **Assimp** |
| メッシュレット生成 | 第36章 | **DirectXMesh** |

**「ライブラリを使わない」ことで、何が得られたか。**

**構造体の中身を知っています。** `D3D12_RESOURCE_BARRIER` が何を並べているのか、`SampleMask` が何をしているのか、`GetCopyableFootprints` が何を計算しているのか。

**設計の選択肢を知っています。** 行ベクトルか列ベクトルか、ルート定数かテーブルか、Reversed-Z を使うかどうか。**すべて自分で決めました。**

**そして、壊れたときに追えます。** 第29章から第31章、そして本章で揃えた道具は、**自分で書いたコードだからこそ意味を持ちます。**

### これから

**本書は入門書です。** 扱わなかったものが、まだ多くあります。

| 分野 | 次の一歩 |
|---|---|
| **glTF 2.0** | 第23章 23.6 節 |
| **PBR マテリアル** | 第24章の Blinn-Phong から |
| **カスケードシャドウ** | 第27章 27.3.3 節 |
| **順序独立透明性** | 第28章 28.2.5 節 |
| **パストレーシング** | 第37章から |
| **DLSS / Reflex** | 付録 H |
| **クォータニオン** | 第25章 25.2.1 節 |

**土台はできています。**

**どこから手をつけても、これまで作ったコードの上に積めます。**

---

## 参考リンク

| 内容 | URL |
|---|---|
| タイムスタンプクエリ | https://learn.microsoft.com/ja-jp/windows/win32/direct3d12/timing |
| `ResolveQueryData` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/nf-d3d12-id3d12graphicscommandlist-resolvequerydata |
| DRED | https://learn.microsoft.com/ja-jp/windows/win32/direct3d12/use-dred |
| `ID3D12DeviceRemovedExtendedData` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/nn-d3d12-id3d12deviceremovedextendeddata |
| Nsight Systems | https://docs.nvidia.com/nsight-systems/ |
| NVTX | https://github.com/NVIDIA/NVTX |
| GPU クロックの固定 | https://developer.nvidia.com/blog/advanced-api-performance-setstablepowerstate/ |
