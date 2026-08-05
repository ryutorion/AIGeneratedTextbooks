# 第9章 コマンドキューとコマンドリスト

デバイスができました。しかし、デバイスは**何も実行しません**。

`ID3D12Device` の役割は、オブジェクトを生成することだけです。`CreateCommandQueue`、`CreateCommittedResource`、`CreateGraphicsPipelineState` —— メソッド名がほぼすべて `Create` で始まっているのは偶然ではありません。**デバイスは工場であって、実行装置ではありません。**

では誰が実行するのか。本章で作る 3 つのオブジェクトです。

Direct3D 11 から来た読者にとって、ここが最初の大きな断絶になります。D3D11 の `ID3D11DeviceContext` は「呼んだら実行される」ように見えました(実際にはドライバが裏でまとめていましたが、それは見えませんでした)。D3D12 にそんなものはありません。**命令を紙に書き、束ねて、窓口に出す。** それだけです。実行されたかどうかは、こちらから確認しに行かない限りわかりません。

本章の終わりに動くのは、**何もしないコマンドリスト**です。画面には何も出ません。それでも、ここで作る配管がこの先すべての描画を運びます。

**本章のゴール**
コマンドキュー・コマンドアロケータ・コマンドリストを生成し、空のコマンドリストをエラーなく実行する。あわせて、状態遷移の規則を意図的に破り、デバッグレイヤーが検出することを確認する。

---

## 9.1 GPU は非同期に動く

### 9.1.1 「記録して投げる」

D3D12 のコマンド実行は、次の 3 段階で進みます。

```
① 記録   CPU がコマンドリストに命令を書き込む
              ↓
② 投入   ExecuteCommandLists でキューに渡す
              ↓
③ 実行   GPU が、いつか、実行する
```

**重要なのは ③ の「いつか」です。**

`ExecuteCommandLists` は**即座に戻ります**。戻った時点で、GPU はまだ何も始めていないかもしれませんし、すでに全部終わっているかもしれません。**知る手段は、この章の範囲にはありません。** 第10章でフェンスを導入して初めて確認できるようになります。

この非対称性が、D3D12 のあらゆる難しさの根源です。

- CPU は先へ進む
- GPU は遅れてついてくる
- **その差がどれだけあるかは、こちらが管理しない限り不明**

### 9.1.2 Direct3D 11 との違い

| | Direct3D 11 | Direct3D 12 |
|---|---|---|
| 命令の発行 | `ID3D11DeviceContext` に直接呼ぶ | コマンドリストに記録する |
| 実行の開始 | ドライバが適当なタイミングで | `ExecuteCommandLists` で明示的に |
| 同期 | ドライバが自動でやる | **すべて自分でやる** |
| メモリの解放 | 参照カウントで安全 | **GPU が使い終わるまで自分で待つ** |

最後の行がとくに効いてきます。D3D11 では、使用中のリソースを解放しようとしてもドライバが面倒を見てくれました。D3D12 では、**GPU が読んでいる最中のメモリを解放すると、そのままクラッシュします。**

本章で扱うコマンドアロケータが、その最初の実例になります。

---

## 9.2 3 つのオブジェクトの役割分担

### 9.2.1 全体像

```
┌──────────────────────────┐
│ ID3D12CommandAllocator   │  記録されたコマンドの「置き場所」
│                          │  = メモリそのもの
└────────────▲─────────────┘
             │ 書き込む
┌────────────┴─────────────┐
│ ID3D12GraphicsCommandList│  記録するための「道具」
│                          │  = メモリを持たない記録インターフェース
└────────────┬─────────────┘
             │ ExecuteCommandLists
┌────────────▼─────────────┐
│ ID3D12CommandQueue       │  GPU への「投入口」
│                          │  = 投げ込まれた順に実行される
└──────────────────────────┘
```

比喩で言えば、こうなります。

| オブジェクト | 比喩 |
|---|---|
| **CommandAllocator** | ノート(紙そのもの) |
| **CommandList** | ペン(書くための道具) |
| **CommandQueue** | 提出窓口 |

**この比喩から、重要な帰結が 2 つ導かれます。**

**帰結 1:ペンは、書き終わったらすぐ次に使える**
コマンドリストは記録用のインターフェースにすぎず、記録内容を保持していません。提出した直後に `Reset` して次の記録を始められます。**GPU の実行を待つ必要はありません。**

**帰結 2:ノートは、読まれ終わるまで捨てられない**
コマンドアロケータには実際のコマンドデータが入っています。GPU がまだそれを読んでいる最中に `Reset` すると、**読んでいる紙を破り捨てることになります。**

**この非対称性が、本章でもっとも重要な事実です。**

| 操作 | いつ安全か |
|---|---|
| `CommandList::Reset()` | **`ExecuteCommandLists` の直後でよい** |
| `CommandAllocator::Reset()` | **GPU が実行を終えた後** |

そして、GPU が実行を終えたかどうかを知る手段は、第10章まで手に入りません。

### 9.2.2 コマンドリストの種類

```cpp
typedef enum D3D12_COMMAND_LIST_TYPE {
    D3D12_COMMAND_LIST_TYPE_DIRECT  = 0,
    D3D12_COMMAND_LIST_TYPE_BUNDLE  = 1,
    D3D12_COMMAND_LIST_TYPE_COMPUTE = 2,
    D3D12_COMMAND_LIST_TYPE_COPY    = 3,
    // 以下、ビデオ関連
} D3D12_COMMAND_LIST_TYPE;
```

| 種類 | できること | 使う章 |
|---|---|---|
| **DIRECT** | 描画・コンピュート・コピー、すべて | **本章から** |
| COMPUTE | コンピュートとコピー | 第32章、第35章 |
| COPY | コピーのみ | 第35章 |
| BUNDLE | 記録した内容を再生する。キューには直接投入しない | 第35章 |

**DIRECT はすべてができる代わりに、専用キューより効率が落ちる場面があります。** たとえば大量のテクスチャ転送を COPY キューに逃がせば、描画と並行して進められます(第35章)。

本章では DIRECT のみを使います。**種類を増やすのは、増やす理由ができてからです。**

> **アロケータとリストの種類は一致させる**
>
> `D3D12_COMMAND_LIST_TYPE_DIRECT` のアロケータからは、DIRECT のコマンドリストしか作れません。食い違うとデバッグレイヤーが指摘します。

### 9.2.3 対応関係の規則

覚えるべき規則は 3 つです。

**規則 1:1 つのアロケータに、同時に記録できるリストは 1 つだけ**

```
Allocator A ← List 1(記録中)
            ← List 2(記録中)   ❌ 不可
```

記録が終わって `Close()` すれば、同じアロケータを別のリストに使えます。ただし実務では、**アロケータを増やすほうが素直**です。

**規則 2:1 つのリストは、複数のアロケータを渡り歩ける**

`Reset(allocator, pso)` のたびに、どのアロケータへ記録するかを指定します。

**規則 3:マルチスレッドで記録するなら、スレッドごとにアロケータとリストを分ける**

コマンドリストもアロケータも、**スレッドセーフではありません。** 第35章で並列記録を扱いますが、そこでの原則は「共有しない」です。

---

## 9.3 `D3D12_COMMAND_QUEUE_DESC` を手で埋める

### 9.3.1 構造体の全フィールド

```cpp
typedef struct D3D12_COMMAND_QUEUE_DESC {
    D3D12_COMMAND_LIST_TYPE   Type;
    INT                       Priority;
    D3D12_COMMAND_QUEUE_FLAGS Flags;
    UINT                      NodeMask;
} D3D12_COMMAND_QUEUE_DESC;
```

**4 つしかありません。** 本書で最初に手で埋める D3D12 構造体としては、ちょうどよい大きさです。

`d3dx12.h` を使えば `CD3DX12_COMMAND_QUEUE_DESC` の 1 行で済みますが、本書は使いません(第1章 1.3.1 節)。**この 4 つを自分で埋めることで、「キューには種類と優先度がある」という事実が頭に入ります。**

### 9.3.2 各フィールドの意味

#### `Type`

作りたいキューの種類です。ここに指定した種類のコマンドリストだけを投入できます。

```cpp
desc.Type = D3D12_COMMAND_LIST_TYPE_DIRECT;
```

#### `Priority`

```cpp
typedef enum D3D12_COMMAND_QUEUE_PRIORITY {
    D3D12_COMMAND_QUEUE_PRIORITY_NORMAL          = 0,
    D3D12_COMMAND_QUEUE_PRIORITY_HIGH            = 100,
    D3D12_COMMAND_QUEUE_PRIORITY_GLOBAL_REALTIME = 10000
} D3D12_COMMAND_QUEUE_PRIORITY;
```

複数のキューがあるとき、GPU の時間をどう配分するかのヒントです。

**`GLOBAL_REALTIME` には注意が必要です。** システム全体に対して優先権を主張するもので、**特権が必要**です。権限がなければ `CreateCommandQueue` が失敗します。VR のような用途のためのもので、通常のアプリケーションが使うものではありません。

本書は常に `NORMAL` を使います。

```cpp
desc.Priority = D3D12_COMMAND_QUEUE_PRIORITY_NORMAL;
```

> **`Priority` の型は `INT`**
>
> 列挙型ではなく `INT` です。上記以外の値も入れられますが、実際に意味を持つのは定義済みの 3 値だけです。

#### `Flags`

```cpp
typedef enum D3D12_COMMAND_QUEUE_FLAGS {
    D3D12_COMMAND_QUEUE_FLAG_NONE                = 0,
    D3D12_COMMAND_QUEUE_FLAG_DISABLE_GPU_TIMEOUT = 0x1
} D3D12_COMMAND_QUEUE_FLAGS;
```

**`DISABLE_GPU_TIMEOUT` は、開発中に使ってはいけません。**

TDR(Timeout Detection and Recovery)は、GPU が一定時間(既定で 2 秒)応答しないと OS がドライバをリセットする仕組みです。これがあるおかげで、無限ループのシェーダーを書いてしまっても PC は復帰します。

このフラグを立てると、**そのキューでは TDR が働きません。** 無限ループを踏んだ瞬間に画面が固まり、電源ボタン以外に手段がなくなります。しかも TDR が発火しないため、**Aftermath のクラッシュダンプも生成されません。**

本書は常に `NONE` です。

```cpp
desc.Flags = D3D12_COMMAND_QUEUE_FLAG_NONE;
```

#### `NodeMask`

複数の GPU をリンクして使う場合に、どの GPU かを指定するビットマスクです。

**単一 GPU では `0` を指定します。** `0` は「既定のノード」と解釈されます。

```cpp
desc.NodeMask = 0;
```

本書はマルチ GPU を扱いません。以後、`NodeMask` が出てきたら常に `0` です。

### 9.3.3 `{}` を忘れない

**本書のスタイルでもっとも重要な習慣**を、ここで確立します。

```cpp
D3D12_COMMAND_QUEUE_DESC desc{};   // ← この {} が命綱
```

`{}` を付けると、**すべてのフィールドがゼロで初期化されます。** 付けないと、スタック上のゴミがそのまま残ります。

今回の構造体はフィールドが 4 つしかないので、全部埋めれば `{}` がなくても動きます。しかし第14章の `D3D12_GRAPHICS_PIPELINE_STATE_DESC` は**フィールドが 20 個以上**あり、その多くは既定値のまま使います。`{}` を忘れると、埋めなかったフィールドにゴミが入ります。

**症状は最悪です。** 動くこともあれば `E_INVALIDARG` になることもあり、実行するたびに変わることもあります。

> **本書のルール:D3D12 の構造体は、必ず `{}` を付けて宣言する。**
>
> 例外はありません。`d3dx12.h` のヘルパーを使わない以上、これが唯一の防御です。

### 9.3.4 生成する

```cpp
D3D12_COMMAND_QUEUE_DESC desc{};
desc.Type     = D3D12_COMMAND_LIST_TYPE_DIRECT;
desc.Priority = D3D12_COMMAND_QUEUE_PRIORITY_NORMAL;
desc.Flags    = D3D12_COMMAND_QUEUE_FLAG_NONE;
desc.NodeMask = 0;

ComPtr<ID3D12CommandQueue> queue;
HR_TRY(device->CreateCommandQueue(&desc, IID_PPV_ARGS(&queue)));

Core::SetDebugName(queue.Get(), L"DirectQueue");
```

**名前を付けるのを忘れないでください**(第6章 6.5 節)。デバッグレイヤーの警告に `DirectQueue` と出るのと `<unnamed>` と出るのでは、読みやすさがまったく違います。

---

## 9.4 `Reset` と `Close` のライフサイクル

### 9.4.1 コマンドリストは 2 つの状態を持つ

```
        ┌──────────────────────────────────┐
        │      Reset(allocator, pso)       │
        ▼                                  │
  ┌───────────┐                     ┌──────┴──────┐
  │ Recording │ ──── Close() ─────► │   Closed    │
  │  (記録中)  │                     │  (記録済み)  │
  └───────────┘                     └─────────────┘
                                           │
                                           │ ExecuteCommandLists
                                           ▼
                                      (GPU が実行)
```

**規則は次の通りです。**

| 操作 | 必要な状態 |
|---|---|
| コマンドを記録する(`DrawInstanced` など) | Recording |
| `Close()` | Recording |
| `Reset()` | **Closed** |
| `ExecuteCommandLists` に渡す | **Closed** |

### 9.4.2 生成直後は Recording 状態

**ここが最初の落とし穴です。**

```cpp
ComPtr<ID3D12GraphicsCommandList> list;
device->CreateCommandList(0, type, allocator.Get(), nullptr,
                          IID_PPV_ARGS(&list));
// ← この時点で list は Recording 状態!
```

`CreateCommandList` で作ったコマンドリストは、**すでに記録中**です。「作ったので、まず `Reset` しよう」と考えると失敗します。

古典的な定石は、**作った直後に `Close()` する**というものです。

```cpp
device->CreateCommandList(0, type, allocator.Get(), nullptr,
                          IID_PPV_ARGS(&list));
HR_TRY(list->Close());   // すぐ閉じる
```

こうしておけば、フレームループを常に `Reset()` から始められます。特殊な初回処理が要りません。

### 9.4.3 `CreateCommandList1` を使う

`ID3D12Device4` 以降には、**閉じた状態でコマンドリストを作る**メソッドがあります。

```cpp
ComPtr<ID3D12Device4> device4;
HR_TRY(device->QueryInterface(IID_PPV_ARGS(&device4)));

ComPtr<ID3D12GraphicsCommandList> list;
HR_TRY(device4->CreateCommandList1(
    0,                                  // NodeMask
    D3D12_COMMAND_LIST_TYPE_DIRECT,     // Type
    D3D12_COMMAND_LIST_FLAG_NONE,       // Flags
    IID_PPV_ARGS(&list)));
// ← Closed 状態で生成される。アロケータも不要。
```

**こちらのほうが素直です。**

- 生成時にアロケータを渡す必要がない(まだ記録しないので当然)
- 生成直後に `Close()` する不自然な行が消える

`ID3D12Device4` は Windows 10 バージョン 1703 以降で利用でき、Agility SDK を導入している本書の環境では確実に取得できます。

**本書は `CreateCommandList1` を使います。** 古い `CreateCommandList` も、既存のコードを読む際に必要になるので覚えておいてください。

### 9.4.4 `Reset` は 2 種類ある

**紛らわしいのですが、`Reset` は 2 つのオブジェクトにあり、意味がまったく違います。**

#### `ID3D12GraphicsCommandList::Reset(allocator, pso)`

```cpp
HR_TRY(commandList->Reset(allocator.Get(), nullptr));
```

コマンドリストを Recording 状態に戻し、**どのアロケータに記録するか**を指定します。第 2 引数は初期パイプラインステート(第14章)で、今は `nullptr` です。

**このリストが GPU で実行中でも呼べます。** 前述の通り、リストはメモリを持っていないからです。

#### `ID3D12CommandAllocator::Reset()`

```cpp
HR_TRY(allocator->Reset());
```

**記録されたコマンドのメモリを、まとめて再利用可能にします。**

**このアロケータから記録されたコマンドリストが、すべて GPU で実行完了している必要があります。** 実行中に呼ぶと、GPU が読んでいるメモリを再利用することになります。

> **「解放」ではなく「巻き戻し」**
>
> コマンドアロケータは、記録が進むにつれて内部でメモリを確保していきます。`Reset()` はそれを OS に返すのではなく、**先頭に巻き戻して再利用可能にします。** 実際の解放はアロケータ自体が破棄されるときです。
>
> だからこそ、フレームごとに新しいアロケータを作るのではなく、**`Reset()` して使い回す**のが正しい使い方になります(第12章)。

### 9.4.5 `Close()` の戻り値を必ず確認する

**本節でもっとも実用的な指摘です。**

```cpp
HR_TRY(commandList->Close());   // ← 戻り値を捨ててはいけない
```

`Close()` は `HRESULT` を返します。そして、**記録中に発生したエラーの多くは、ここで初めて表面化します。**

コマンドの記録メソッド(`DrawInstanced`、`ResourceBarrier` など)は、ほとんどが `void` を返します。エラーを返す手段がありません。**不正な記録があった場合、その情報は蓄積され、`Close()` で `E_INVALIDARG` などとして返ってきます。**

`Close()` の戻り値を無視すると、**壊れたコマンドリストをそのまま GPU に投げる**ことになります。結果はデバイスロストです。

**本書のルール:`Close()` は必ず `HR_TRY` で受ける。**

第7章で設定した `SetBreakOnSeverity(ERROR, TRUE)` と組み合わせれば、実際には記録した瞬間にデバッガが止まります。それでも `Close()` のチェックは、最後の防御線として残しておいてください。

---

## 9.5 `ExecuteCommandLists`

### 9.5.1 呼び方

```cpp
ID3D12CommandList* lists[] = { commandList.Get() };
queue->ExecuteCommandLists(static_cast<UINT>(std::size(lists)), lists);
```

**引数の型に注意してください。** 受け取るのは `ID3D12CommandList* const*` であって、`ID3D12GraphicsCommandList**` ではありません。

```cpp
// ❌ コンパイルが通らない
ID3D12GraphicsCommandList* bad[] = { commandList.Get() };
queue->ExecuteCommandLists(1, bad);
```

`ID3D12GraphicsCommandList` は `ID3D12CommandList` を継承していますが、**ポインタの配列は継承関係を引き継ぎません。** 基底クラスのポインタとして配列を作ってください。

### 9.5.2 まとめて投入する

複数のコマンドリストは、**1 回の呼び出しでまとめて渡すほうが効率的です。**

```cpp
// ✅ 望ましい
ID3D12CommandList* lists[] = { list1.Get(), list2.Get(), list3.Get() };
queue->ExecuteCommandLists(3, lists);

// △ 動くが、投入のたびにオーバーヘッドがある
queue->ExecuteCommandLists(1, &raw1);
queue->ExecuteCommandLists(1, &raw2);
queue->ExecuteCommandLists(1, &raw3);
```

配列内の順序が、そのまま実行順序になります。**同じキューに投入されたコマンドリストが並列に実行されることはありません。** 並列に走らせたいなら別のキューを使います(第35章)。

### 9.5.3 戻り値がない

```cpp
void ExecuteCommandLists(UINT NumCommandLists,
                         ID3D12CommandList* const* ppCommandLists);
```

**`void` です。** 失敗を返す手段がありません。

不正なコマンドリストを投入した場合、何が起きるか。**その場では何も起きません。** しばらくして、`Present` や `GetDeviceRemovedReason` が `DXGI_ERROR_DEVICE_REMOVED` を返します。

**これが D3D12 のエラーが追いにくい理由です。** 問題を起こした場所と、症状が出る場所が離れています。

対策は 3 つあり、本書ではすべて使います。

| 対策 | 導入した章 |
|---|---|
| デバッグレイヤーでエラー時にブレーク | 第7章 7.6.1 |
| `Close()` の戻り値を確認 | 本章 9.4.5 |
| Aftermath でクラッシュ位置を特定 | 第8章・第31章 |

### 9.5.4 なぜ、まだフレームループに入れられないのか

**本章のコードは、初期化時に 1 回だけ実行します。** フレームループには入れません。

理由は 9.2.1 節の帰結 2 です。ループにするなら、毎フレーム次のことをしたくなります。

```cpp
while (window.ProcessMessages())
{
    allocator->Reset();                          // ← ここが危険
    commandList->Reset(allocator.Get(), nullptr);
    // ... 記録 ...
    commandList->Close();
    queue->ExecuteCommandLists(1, lists);
}
```

**`allocator->Reset()` を呼んでよいのは、前フレームの実行が GPU 側で完了した後です。** そして、それを知る手段が今の我々にはありません。

CPU は GPU より遥かに速く回ります。何もしないループなら、1 秒間に数万回まわるでしょう。GPU が 1 フレーム分を処理する間に、CPU はアロケータを何千回も `Reset` します。**GPU が読んでいる最中のメモリを、繰り返し上書きすることになります。**

**この問題を解決するのが、次章のフェンスです。**

---

## 9.6 実装

### 9.6.1 コマンドキューのラッパ

```cpp
// src/Graphics/CommandQueue.h
#pragma once
#include "std_import.h"
#include "Core/Error.h"

namespace Graphics
{
    class CommandQueue
    {
    public:
        CommandQueue()  = default;
        ~CommandQueue() = default;

        CommandQueue(const CommandQueue&)            = delete;
        CommandQueue& operator=(const CommandQueue&) = delete;

        Core::Status Initialize(ID3D12Device* device,
                                D3D12_COMMAND_LIST_TYPE type,
                                std::wstring_view name);
        void Shutdown();

        ID3D12CommandQueue*     Get()  const noexcept { return m_queue.Get(); }
        D3D12_COMMAND_LIST_TYPE Type() const noexcept { return m_type; }

        void Execute(ID3D12GraphicsCommandList* list);
        void Execute(std::span<ID3D12CommandList* const> lists);

    private:
        Microsoft::WRL::ComPtr<ID3D12CommandQueue> m_queue;
        D3D12_COMMAND_LIST_TYPE m_type = D3D12_COMMAND_LIST_TYPE_DIRECT;
    };
}
```

```cpp
// src/Graphics/CommandQueue.cpp
#include "pch.h"
#include "std_import.h"
#if USE_STD_MODULE
import std;
#endif
#include "Graphics/CommandQueue.h"
#include "Core/Log.h"
#include "Core/DebugName.h"

namespace Graphics
{
    Core::Status CommandQueue::Initialize(
        ID3D12Device* device,
        D3D12_COMMAND_LIST_TYPE type,
        std::wstring_view name)
    {
        D3D12_COMMAND_QUEUE_DESC desc{};     // ← {} を忘れない(9.3.3)
        desc.Type     = type;
        desc.Priority = D3D12_COMMAND_QUEUE_PRIORITY_NORMAL;
        desc.Flags    = D3D12_COMMAND_QUEUE_FLAG_NONE;
        desc.NodeMask = 0;

        HR_TRY(device->CreateCommandQueue(&desc, IID_PPV_ARGS(&m_queue)));

        m_type = type;
        Core::SetDebugName(m_queue.Get(), name);

        LOG_INFO(L"command queue created: {}", name);
        return {};
    }

    void CommandQueue::Shutdown()
    {
        m_queue.Reset();
    }

    void CommandQueue::Execute(ID3D12GraphicsCommandList* list)
    {
        // 基底クラスのポインタとして配列を作る(9.5.1)
        ID3D12CommandList* lists[] = { list };
        m_queue->ExecuteCommandLists(1, lists);
    }

    void CommandQueue::Execute(std::span<ID3D12CommandList* const> lists)
    {
        if (lists.empty())
        {
            return;
        }
        m_queue->ExecuteCommandLists(
            static_cast<UINT>(lists.size()), lists.data());
    }
}
```

### 9.6.2 アロケータとリストは、まだクラスにしない

**意図的に、コマンドアロケータとコマンドリストはクラス化しません。**

理由は、**まだ寿命の管理方法が決まっていないから**です。第12章でフレーム単位の管理を設計するとき、「アロケータをいくつ持つか」「どのタイミングで `Reset` するか」が決まります。その前に `Begin()` / `End()` のようなメソッドを作ると、**危険な操作(`allocator->Reset()`)を安全そうな名前で隠してしまいます。**

```cpp
// ❌ この段階で書いてはいけない設計
class CommandContext
{
    void Begin()
    {
        m_allocator->Reset();     // いつ安全なのかが名前から消える
        m_list->Reset(m_allocator.Get(), nullptr);
    }
};
```

**抽象化は、対象を理解した後に行うものです。** 本章では素のまま扱い、危険な箇所を目に見える状態にしておきます。

### 9.6.3 初期化コード

```cpp
using Microsoft::WRL::ComPtr;

Graphics::CommandQueue            g_queue;
ComPtr<ID3D12CommandAllocator>    g_allocator;
ComPtr<ID3D12GraphicsCommandList> g_commandList;

Core::Status CreateCommandObjects(ID3D12Device* device)
{
    //--- ① キュー ---
    if (auto r = g_queue.Initialize(
            device, D3D12_COMMAND_LIST_TYPE_DIRECT, L"DirectQueue"); !r)
    {
        return r;
    }

    //--- ② アロケータ ---
    HR_TRY(device->CreateCommandAllocator(
        D3D12_COMMAND_LIST_TYPE_DIRECT,
        IID_PPV_ARGS(&g_allocator)));
    Core::SetDebugName(g_allocator.Get(), L"MainAllocator");

    //--- ③ コマンドリスト(Closed 状態で生成) ---
    ComPtr<ID3D12Device4> device4;
    HR_TRY(device->QueryInterface(IID_PPV_ARGS(&device4)));

    HR_TRY(device4->CreateCommandList1(
        0,
        D3D12_COMMAND_LIST_TYPE_DIRECT,
        D3D12_COMMAND_LIST_FLAG_NONE,
        IID_PPV_ARGS(&g_commandList)));
    Core::SetDebugName(g_commandList.Get(), L"MainCommandList");

    return {};
}
```

### 9.6.4 空のコマンドリストを実行する

```cpp
Core::Status SubmitEmptyCommandList()
{
    //--- 記録を開始 ---
    HR_TRY(g_allocator->Reset());
    HR_TRY(g_commandList->Reset(g_allocator.Get(), nullptr));

    //--- ここに描画コマンドが入る(第11章から) ---

    //--- 記録を終了 ---
    HR_TRY(g_commandList->Close());     // 戻り値を必ず見る(9.4.5)

    //--- 投入 ---
    g_queue.Execute(g_commandList.Get());

    LOG_INFO(L"empty command list submitted");
    return {};
}
```

**この 5 行が、この先すべての描画の骨格になります。** 第13章で三角形を描くときも、第37章でレイトレーシングするときも、構造は同じです。間の「記録」部分が増えていくだけです。

---

## ✅ 本章のゴール:空のコマンドリストを実行する

### Step 1:正常系を確認する

```cpp
int WINAPI wWinMain(...)
{
    ::SetProcessDpiAwarenessContext(DPI_AWARENESS_CONTEXT_PER_MONITOR_AWARE_V2);

    Window window;
    if (!window.Create(L"D3D12Book - Chapter 9", 1280, 720)) return 1;

    Graphics::GraphicsDevice device;
    if (auto r = device.Initialize({}); !r)
    {
        Core::ReportError(r.error());
        return 1;
    }
    device.LogSummary();

    if (auto r = CreateCommandObjects(device.Device()); !r)
    {
        Core::ReportError(r.error());
        return 1;
    }

    if (auto r = SubmitEmptyCommandList(); !r)
    {
        Core::ReportError(r.error());
        return 1;
    }

    while (window.ProcessMessages())
    {
        if (window.IsMinimized()) continue;
        // 第12章まで、ここは空のまま
    }

    // GPU がまだ実行中かもしれないが、
    // 待つ手段は第10章まで手に入らない
    device.Shutdown();
    return 0;
}
```

**期待される出力**

```
[Info ] CommandQueue.cpp(28): command queue created: DirectQueue
[Info ] main.cpp(64): empty command list submitted
```

**そして、それ以外に何も出ないこと。**

第7章で `SetBreakOnSeverity(WARNING, TRUE)` を設定したので、デバッグレイヤーが問題を見つければ必ず止まります。**止まらないということは、状態遷移が正しいということです。**

画面には相変わらず何も出ません。**これで正解です。**

### Step 2:わざと規則を破る

何も起きないことの確認だけでは、配管が本当につながっているのかわかりません。**規則を破ってみて、デバッグレイヤーが検出することを確かめます。**

以下の実験は、**1 つずつ試して、確認したら削除してください。**

#### 実験 A:Recording 状態のまま `Reset` する

```cpp
HR_TRY(g_commandList->Reset(g_allocator.Get(), nullptr));
g_commandList->Reset(g_allocator.Get(), nullptr);   // ❌ 2 回目
```

デバッガが停止し、次のような指摘が出ます。

```
[Error] Log.cpp(60): [D3D12] (id 519) D3D12 ERROR:
  ID3D12GraphicsCommandList::Reset: A command list must be
  in the "closed" state before Reset is called.
```

#### 実験 B:Closed 状態のまま記録しようとする

```cpp
HR_TRY(g_commandList->Close());
g_commandList->ClearState(nullptr);   // ❌ 閉じた後に記録
```

#### 実験 C:閉じていないリストを投入する

```cpp
HR_TRY(g_commandList->Reset(g_allocator.Get(), nullptr));
g_queue.Execute(g_commandList.Get());   // ❌ Close していない
```

```
[Error] Log.cpp(60): [D3D12] (id 1000) D3D12 ERROR:
  ID3D12CommandQueue::ExecuteCommandLists: A command list
  that is currently in the recording state was submitted.
```

**3 つとも検出されれば、デバッグレイヤーは正しく機能しています。**

> **なぜエラー ID が表示されるのか**
>
> 第7章 7.6.2 節でメッセージコールバックを登録し、`D3D12_MESSAGE_ID` をログに含めるようにしたためです。この ID は、7.6.3 節でメッセージを個別に抑制する際に使います。**抑制したくなったときに、調べ直さずに済みます。**

### Step 3:解決できない問題を確認する

最後に、**この章では解決できない問題**を頭に入れておきます。

```cpp
while (window.ProcessMessages())
{
    if (window.IsMinimized()) continue;

    // ❌ これは危険。第10章まで書いてはいけない
    if (auto r = SubmitEmptyCommandList(); !r)
    {
        Core::ReportError(r.error());
        break;
    }
}
```

**実際に動かす必要はありません。** 何が起きるかを理解しておいてください。

- CPU は 1 秒間に数万回このループを回る
- そのたびに `g_allocator->Reset()` が呼ばれる
- GPU は、まだ前のコマンドリストを読んでいるかもしれない
- **読んでいる最中のメモリが再利用される**

空のコマンドリストなので、実際には何も壊れないかもしれません。しかし第13章で三角形を描き始めた瞬間、この構造は破綻します。

**「動いているように見えるが、根拠がない」状態です。** 次章でこれを解決します。

---

### 本章の達成状態

- [ ] `D3D12_COMMAND_QUEUE_DESC` を `{}` 付きで宣言し、4 つのフィールドを埋めた
- [ ] `CreateCommandList1` で Closed 状態のコマンドリストを生成した
- [ ] `Close()` の戻り値を `HR_TRY` で受けている
- [ ] `ExecuteCommandLists` に基底クラスのポインタ配列を渡している
- [ ] キュー・アロケータ・リストすべてに名前を付けた
- [ ] 空のコマンドリストが警告なく実行された
- [ ] 実験 A / B / C でデバッグレイヤーが停止することを確認した
- [ ] アロケータとリストを、まだクラス化していない

---

## トラブルシューティング

| 症状 | 原因 | 対処 |
|---|---|---|
| `E_INVALIDARG`(キュー生成) | `Type` が不正、または `{}` 忘れ | 9.3.3 を確認 |
| キュー生成に失敗する | `GLOBAL_REALTIME` を指定した | `NORMAL` にする(9.3.2) |
| `Reset` で「closed 状態が必要」 | 生成直後は Recording | `CreateCommandList1` を使う(9.4.3) |
| `ExecuteCommandLists` でコンパイルエラー | 派生クラスのポインタ配列を渡した | 基底クラスの配列にする(9.5.1) |
| `Close()` が `E_INVALIDARG` | 記録中に不正なコマンドがあった | デバッグレイヤーの指摘を見る(9.4.5) |
| アロケータとリストの型が違う警告 | `D3D12_COMMAND_LIST_TYPE` の不一致 | 両方を揃える(9.2.2) |
| 終了時に「リソースが使用中」 | GPU の実行を待たずに破棄した | **第10章で解決する** |
| ループにしたら不安定 | アロケータを実行中に `Reset` した | **第10章で解決する** |

---

## まとめ

**1. デバイスは何も実行しない。**
`ID3D12Device` はオブジェクトを作るだけです。実行するのはコマンドキューです。

**2. リストとアロケータの非対称性を理解する。**
コマンドリストは記録の道具にすぎず、投入直後に `Reset` できます。コマンドアロケータは実際のメモリなので、**GPU が読み終わるまで `Reset` できません。**

**3. `{}` を必ず付ける。**
`d3dx12.h` のヘルパーを使わない本書では、これが唯一の初期化漏れ対策です。第14章の巨大な構造体で効いてきます。

**4. `Close()` の戻り値を捨てない。**
記録メソッドの多くは `void` を返します。記録中のエラーは `Close()` で初めて表面化します。

**5. `ExecuteCommandLists` は失敗を返さない。**
問題は後になってデバイスロストとして現れます。だから第7章のデバッグレイヤーと、第8章の Aftermath が要ります。

**6. まだフレームループに入れられない。**
GPU の実行完了を知る手段がないからです。**この不足こそが、次章の主題です。**

次章でフェンスを導入します。`Signal` と `GetCompletedValue`、そしてイベントオブジェクトによる待機。**たった 3 つの API で、本章で行き詰まった問題が解けます。**

---

## 参考リンク

| 内容 | URL |
|---|---|
| `ID3D12CommandQueue` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/nn-d3d12-id3d12commandqueue |
| `ID3D12CommandAllocator::Reset` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/nf-d3d12-id3d12commandallocator-reset |
| `ID3D12GraphicsCommandList::Reset` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/nf-d3d12-id3d12graphicscommandlist-reset |
| `ID3D12Device4::CreateCommandList1` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/nf-d3d12-id3d12device4-createcommandlist1 |
| `D3D12_COMMAND_QUEUE_DESC` | https://learn.microsoft.com/ja-jp/windows/win32/api/d3d12/ns-d3d12-d3d12_command_queue_desc |
| コマンドリストとバンドルの記録 | https://learn.microsoft.com/ja-jp/windows/win32/direct3d12/recording-command-lists-and-bundles |
