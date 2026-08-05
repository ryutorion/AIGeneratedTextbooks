# 第31章 Aftermath を使いこなす

**本書の設計上の要となる章です。**

第2章 2.4.3 節で Aftermath SDK をダウンロードし、第3章 3.6 節でリンク設定を済ませ、第8章で最小構成を組み込みました。**まだ三角形すら出ていない段階で、クラッシュダンプの仕組みを用意したのは、この章のためです。**

第13章 13.6.3 節では、こう書きました。

> **後から設定しても手遅れです。** デバッグ情報なしでビルドしたシェーダーがクラッシュしても、そのときのダンプには機械語のアドレスしか残りません。

**本章で、その投資が回収されます。**

やることは 3 つです。

| 作業 | 得られるもの |
|---|---|
| **機能フラグを有効にする** | リソース追跡、呼び出し履歴 |
| **イベントマーカーを仕込む** | GPU がどこまで実行したか |
| **シェーダーデバッグ情報を紐付ける** | **クラッシュ位置を HLSL の行番号で特定** |

そして最後に、**意図的にクラッシュさせて、原因を特定する実習**を行います。

**本章のゴール**
GPU クラッシュの原因を、HLSL のソースコード上の行として特定できるようになる。

---

## 31.1 機能フラグを有効にする

### 31.1.1 第8章の初期化を拡張する

**第8章 8.3 節で、次の順序を確認しました。**

```
GFSDK_Aftermath_EnableGpuCrashDumps   ← デバイス生成の
        ↓
D3D12CreateDevice
        ↓
GFSDK_Aftermath_DX12_Initialize       ← デバイス生成の
```

**本章では、後者に渡す機能フラグを増やします。**

```cpp
typedef enum GFSDK_Aftermath_FeatureFlags
{
    GFSDK_Aftermath_FeatureFlags_Minimum                = 0x00000000,
    GFSDK_Aftermath_FeatureFlags_EnableMarkers          = 0x00000001,
    GFSDK_Aftermath_FeatureFlags_EnableResourceTracking = 0x00000002,
    GFSDK_Aftermath_FeatureFlags_CallStackCapturing     = 0x40000000,
    GFSDK_Aftermath_FeatureFlags_GenerateShaderDebugInfo= 0x00000008,
    GFSDK_Aftermath_FeatureFlags_EnableShaderErrorReporting = 0x00000010,
} GFSDK_Aftermath_FeatureFlags;
```

### 31.1.2 各フラグの意味とコスト

| フラグ | 得られるもの | コスト |
|---|---|---|
| **`EnableMarkers`** | イベントマーカーの記録 | 中(31.2 節で軽減) |
| **`EnableResourceTracking`** | ページフォルト時のリソース特定 | 小 |
| **`CallStackCapturing`** | マーカー設定時の CPU 呼び出し履歴 | **大** |
| **`GenerateShaderDebugInfo`** | シェーダーの行番号対応表 | 中 |
| **`EnableShaderErrorReporting`** | シェーダー内のエラー報告 | 中 |

**`CallStackCapturing` のコストが突出しています。**

マーカーを設定するたびに CPU のスタックトレースを取得するので、**マーカーが多いと目に見えて遅くなります。** ドローコールごとにマーカーを打つ設計では、実用に耐えません。

### 31.1.3 本書の設定

```cpp
// src/Graphics/AftermathConfig.h
#pragma once

namespace Aftermath
{
    //-----------------------------------------------------------
    // 開発時の設定。
    //-----------------------------------------------------------
    inline constexpr std::uint32_t kDevelopmentFlags =
          GFSDK_Aftermath_FeatureFlags_EnableMarkers
        | GFSDK_Aftermath_FeatureFlags_EnableResourceTracking
        | GFSDK_Aftermath_FeatureFlags_GenerateShaderDebugInfo
        | GFSDK_Aftermath_FeatureFlags_EnableShaderErrorReporting;

    //-----------------------------------------------------------
    // 製品版の設定(31.6 節)。
    // マーカーは残すが、呼び出し履歴は取らない。
    //-----------------------------------------------------------
    inline constexpr std::uint32_t kShippingFlags =
          GFSDK_Aftermath_FeatureFlags_EnableMarkers
        | GFSDK_Aftermath_FeatureFlags_EnableResourceTracking
        | GFSDK_Aftermath_FeatureFlags_GenerateShaderDebugInfo;

    //-----------------------------------------------------------
    // クラッシュを追っているときだけ有効にする。
    //-----------------------------------------------------------
    inline constexpr std::uint32_t kDeepDebugFlags =
          kDevelopmentFlags
        | GFSDK_Aftermath_FeatureFlags_CallStackCapturing;
}
```

**`CallStackCapturing` を既定から外しました。**

**第7章 7.1.3 節で GPU-Based Validation を既定で無効にしたのと同じ判断です。** 強力だが重いものは、必要なときだけ有効にします。

```cpp
Core::Status Aftermath::InitializeDevice(ID3D12Device* device,
                                         std::uint32_t featureFlags)
{
    const auto result = GFSDK_Aftermath_DX12_Initialize(
        GFSDK_Aftermath_Version_API,
        featureFlags,
        device);

    if (!GFSDK_Aftermath_SUCCEED(result))
    {
        LOG_ERROR(L"Aftermath DX12 init failed: {:#x}",
                  static_cast<std::uint32_t>(result));
        return std::unexpected(Core::MakeError(
            E_FAIL, L"GFSDK_Aftermath_DX12_Initialize"));
    }

    LOG_INFO(L"Aftermath initialized (flags {:#x})", featureFlags);
    return {};
}
```

---

## 31.2 イベントマーカー

### 31.2.1 何のためにあるか

**GPU がクラッシュしたとき、知りたいのは「どこまで実行したか」です。**

**マーカーは、コマンドの流れの中に埋め込まれた目印です。**

```
[マーカー: Shadow Pass]
  ドローコール ×48
[マーカー: Opaque]
  ドローコール ×312
[マーカー: Bloom]
  ドローコール ×7        ← ここでクラッシュ
[マーカー: Composite]
```

**クラッシュダンプには、「どのマーカーまで完了し、どのマーカーが実行中か」が記録されます。**

**第29章 29.1.2 節で入れた `GPU_EVENT` とは別物です。**

| | `GPU_EVENT`(第29章) | **Aftermath マーカー(本章)** |
|---|---|---|
| 用途 | **ツールでの可視化** | **クラッシュ時の位置特定** |
| 認識するもの | Nsight Graphics、PIX | **Aftermath** |
| クラッシュ時 | 記録されない | **記録される** |

**両方入れる必要があります。** 幸い、同じ場所に置けます。

### 31.2.2 コンテキストハンドル

**マーカーは、コマンドリストごとのコンテキストに設定します。**

```cpp
GFSDK_Aftermath_ContextHandle contextHandle{};

const auto result = GFSDK_Aftermath_DX12_CreateContextHandle(
    commandList,
    &contextHandle);
```

**コマンドリストと 1 対 1 で対応します。**

**本書はコマンドリストが 1 本なので(第12章 12.2.2 節)、ハンドルも 1 つです。** 第35章で並列記録を導入したら、リストごとに作ります。

**破棄も必要です。**

```cpp
GFSDK_Aftermath_ReleaseContextHandle(contextHandle);
```

### 31.2.3 文字列を毎回渡さない設計

**素朴な実装は、こうなります。**

```cpp
const char* name = "Shadow Pass";
GFSDK_Aftermath_SetEventMarker(
    contextHandle,
    name,
    static_cast<unsigned int>(std::strlen(name) + 1));
```

**これには問題があります。**

**マーカーのデータは、Aftermath が内部でコピーして保持します。** 毎フレーム数百個のマーカーを打つと、**その分のメモリコピーが発生します。**

**解決策が用意されています。**

> **マーカーのデータ長を 0 にすると、ポインタ値そのものがマーカー ID として扱われます。**

```cpp
GFSDK_Aftermath_SetEventMarker(
    contextHandle,
    reinterpret_cast<const void*>(markerId),
    0);                                        // ← サイズ 0
```

**コピーが発生しません。**

**代わりに、ダンプを解析するとき「この ID が何を意味するか」を教える必要があります。** そのためのコールバックが `ResolveMarkerCallback` です。

### 31.2.4 マーカー ID の管理

```cpp
// src/Graphics/AftermathMarkers.h
#pragma once
#include "std_import.h"

namespace Aftermath
{
    //-----------------------------------------------------------
    // マーカー名を ID に対応づける。
    // ID は、名前を登録した順の連番。
    //-----------------------------------------------------------
    class MarkerRegistry
    {
    public:
        //--- 名前を登録し、ID を返す(初回のみ登録)---
        [[nodiscard]] std::uint64_t Register(std::string_view name);

        //--- ID から名前を引く(解析時に使う)---
        [[nodiscard]] std::string_view Lookup(std::uint64_t id) const;

        static MarkerRegistry& Instance();

    private:
        mutable std::mutex m_mutex;
        std::vector<std::string> m_names;
        std::unordered_map<std::string, std::uint64_t> m_lookup;
    };
}
```

```cpp
std::uint64_t MarkerRegistry::Register(std::string_view name)
{
    const std::lock_guard lock(m_mutex);

    const std::string key(name);
    if (const auto it = m_lookup.find(key); it != m_lookup.end())
    {
        return it->second;
    }

    //--- ID は 1 から。0 は「マーカーなし」を表す ---
    const auto id = static_cast<std::uint64_t>(m_names.size()) + 1;
    m_names.push_back(key);
    m_lookup.emplace(key, id);
    return id;
}
```

**ID を 1 から始めるのは、第10章 10.3.2 節でフェンス値を 1 から始めたのと同じ理由です。** 0 を「未設定」として区別できます。

**`std::mutex` で保護しているのは、第35章の並列記録に備えてのことです。** 現時点では 1 スレッドですが、後で書き換えずに済みます。

### 31.2.5 RAII でスコープを表現する

**第29章 29.1.2 節の `ScopedEvent` と同じ形にします。**

```cpp
// src/Graphics/AftermathMarkers.h

namespace Aftermath
{
    class ScopedMarker
    {
    public:
        ScopedMarker(GFSDK_Aftermath_ContextHandle context,
                     std::string_view name)
            : m_context(context)
        {
            const auto id = MarkerRegistry::Instance().Register(name);

            //--- サイズ 0 でポインタを ID として渡す(31.2.3 節)---
            GFSDK_Aftermath_SetEventMarker(
                m_context,
                reinterpret_cast<const void*>(id),
                0);
        }

        ~ScopedMarker()
        {
            //--- スコープを抜けたことを示す ---
            // 親スコープの ID に戻すのが理想だが、
            // 単純に「なし」を設定する
            GFSDK_Aftermath_SetEventMarker(m_context, nullptr, 0);
        }

        ScopedMarker(const ScopedMarker&)            = delete;
        ScopedMarker& operator=(const ScopedMarker&) = delete;

    private:
        GFSDK_Aftermath_ContextHandle m_context;
    };
}
```

**マーカーは「最後に設定されたもの」だけが有効です。** 入れ子構造を厳密に表現するには、スタックを自前で管理する必要があります。

**本書は単純な形にします。** クラッシュ位置の特定には、これで十分です。

### 31.2.6 第29章のマクロと統合する

**同じ場所に 2 種類のマーカーを打ちたいので、マクロをまとめます。**

```cpp
// src/Graphics/DebugMarker.h を更新

//---------------------------------------------------------------
// ツール用マーカー(第29章)と Aftermath マーカー(第31章)を
// 同時に設定する。
//---------------------------------------------------------------
class ScopedGpuMarker
{
public:
    ScopedGpuMarker(ID3D12GraphicsCommandList* commandList,
                    GFSDK_Aftermath_ContextHandle aftermathContext,
                    std::string_view name)
        : m_toolMarker(commandList, Core::ToWide(name))
        , m_aftermathMarker(aftermathContext, name)
    {
    }

private:
    Graphics::ScopedEvent      m_toolMarker;       // 第29章
    Aftermath::ScopedMarker    m_aftermathMarker;  // 第31章
};

#define GPU_MARKER(cmdList, ctx, name)                              \
    ScopedGpuMarker GPU_EVENT_CONCAT(gpuMarker_, __LINE__)          \
        { (cmdList), (ctx), (name) }
```

**使い方は変わりません。**

```cpp
{
    GPU_MARKER(m_commandList.Get(), m_aftermathContext, "Shadow Pass");
    RenderShadowPass(m_shadowCasters);
}
```

### 31.2.7 粒度をどうするか

**マーカーは細かいほど、クラッシュ位置が正確に分かります。** しかしコストも増えます。

| 粒度 | 特定できる範囲 | コスト |
|---|---|---|
| **パスごと** | 「ブルームパスで落ちた」 | **小** |
| メッシュごと | 「このモデルの描画で落ちた」 | 中 |
| ドローコールごと | 「312 個目のドローで落ちた」 | 大 |

**本書はパスごとを既定とし、切り替えられるようにします。**

```cpp
#if defined(AFTERMATH_FINE_GRAINED_MARKERS)
    for (const RenderObject* object : visible)
    {
        GPU_MARKER(cmdList, ctx,
                   std::format("Draw {}", object->name));
        // ...
    }
#endif
```

**クラッシュがどのパスで起きるか分かったら、そのパスだけ細かくします。** 段階的に絞り込むのが実用的です。

---

## 31.3 シェーダーデバッグ情報を紐付ける

### 31.3.1 3 つの識別子

**ここが本章で最も混乱しやすい部分です。** 似た名前のものが 3 つあります。

| 識別子 | 取得方法 | 何を指すか |
|---|---|---|
| **シェーダーハッシュ** | `GFSDK_Aftermath_GetShaderHash` | シェーダーバイナリの識別子 |
| **DebugName** | `GFSDK_Aftermath_GetShaderDebugName` | **PDB ファイル名に対応** |
| **ShaderDebugInfoIdentifier** | コールバックで受け取る | ドライバが生成する情報の識別子 |

**それぞれ別のコールバックで使います。**

```
クラッシュダンプの解析

├─ ShaderDebugInfoLookupCallback
│    → ShaderDebugInfoIdentifier で検索
│    → ドライバ生成のデバッグ情報(31.3.2 節)
│
├─ ShaderLookupCallback
│    → シェーダーハッシュで検索
│    → .cso ファイルの内容
│
└─ ShaderSourceDebugDataLookupCallback
     → DebugName で検索
     → .pdb ファイルの内容(第13章)
```

**3 つすべてを提供して初めて、HLSL の行番号が表示されます。**

### 31.3.2 ドライバ生成のデバッグ情報

**`GenerateShaderDebugInfo` フラグを有効にすると、コールバックが呼ばれます。**

```cpp
void GpuCrashTracker::OnShaderDebugInfo(const void* debugInfo,
                                        std::uint32_t size)
{
    //--- 識別子を取得する ---
    GFSDK_Aftermath_ShaderDebugInfoIdentifier identifier{};

    const auto result = GFSDK_Aftermath_GetShaderDebugInfoIdentifier(
        GFSDK_Aftermath_Version_API,
        debugInfo,
        size,
        &identifier);

    if (!GFSDK_Aftermath_SUCCEED(result))
    {
        return;
    }

    //--- メモリに保持する ---
    const std::lock_guard lock(m_mutex);

    std::vector<std::byte> data(size);
    std::memcpy(data.data(), debugInfo, size);
    m_shaderDebugInfo[identifier] = std::move(data);

    //--- ファイルにも書き出しておく ---
    // クラッシュ後に別プロセスで解析する場合に必要
    const auto fileName = std::format(L"shader-{:016x}-{:016x}.nvdbg",
                                      identifier.id[0], identifier.id[1]);
    WriteFile(m_dumpDirectory / fileName, data);
}
```

**このコールバックは、シェーダーが最初に使われたときに呼ばれます。** クラッシュ時ではありません。

**したがって、常に保持しておく必要があります。**

> **`DeferDebugInfoCallbacks` フラグ**
>
> `GFSDK_Aftermath_EnableGpuCrashDumps` に渡すフラグに、次のものがあります。
>
> ```cpp
> GFSDK_Aftermath_GpuCrashDumpFeatureFlags_DeferDebugInfoCallbacks
> ```
>
> **これを指定すると、Aftermath 側がデバッグ情報をキャッシュし、クラッシュ時にまとめてコールバックを呼びます。**
>
> **アプリ側で保持する必要がなくなります。** メモリ効率が良いので、本書はこちらを使います。

### 31.3.3 シェーダーバイナリの登録

**第13章で出力した `.cso` を、ハッシュで引けるようにします。**

```cpp
void ShaderRegistry::Register(const std::filesystem::path& csoPath)
{
    const auto data = ReadFile(csoPath);
    if (data.empty()) return;

    const D3D12_SHADER_BYTECODE bytecode{ data.data(), data.size() };

    //--- ① シェーダーハッシュ ---
    GFSDK_Aftermath_ShaderBinaryHash hash{};
    GFSDK_Aftermath_GetShaderHash(
        GFSDK_Aftermath_Version_API, &bytecode, &hash);

    //--- ② DebugName(PDB のファイル名に対応)---
    GFSDK_Aftermath_ShaderDebugName debugName{};
    GFSDK_Aftermath_GetShaderDebugName(
        GFSDK_Aftermath_Version_API, &bytecode, &debugName);

    const std::lock_guard lock(m_mutex);

    m_binaries[hash.hash]          = data;
    m_debugNames[debugName.name]   = csoPath;

    LOG_TRACE(L"shader registered: {} (hash {:016x}, debug name {})",
              csoPath.filename().wstring(),
              hash.hash,
              Core::ToWide(debugName.name));
}
```

**`GFSDK_Aftermath_ShaderDebugName::name` は、文字列です。**

**そして、この文字列が PDB のファイル名と一致します。**

**第13章 13.6.2 節で `-Fd` にディレクトリを渡した**ので、PDB はハッシュ名で出力されています。

```
build/x64/Debug/shaders/pdb/
  ├─ 1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d.pdb
  └─ 9f8e7d6c5b4a39281706f5e4d3c2b1a0.pdb
```

**`GetShaderDebugName` が返すのが、まさにこの名前です。**

> **DXC のハッシュと Aftermath のハッシュは別物**
>
> 混同しやすい点です。
>
> | ハッシュ | 用途 |
> |---|---|
> | **DXC が生成するハッシュ** | **PDB のファイル名**(第13章 13.6.2 節) |
> | `GFSDK_Aftermath_GetShaderHash` | Aftermath 内部でのシェーダー識別 |
>
> **`GetShaderDebugName` は、前者を取り出す関数です。** シェーダーバイナリの中に記録されている DXC のハッシュを読み取ります。
>
> `GetShaderHash` は別の値を返すので、**PDB を探すのに使ってはいけません。**

### 31.3.4 3 つのルックアップコールバック

**クラッシュダンプを解析するとき、Aftermath が「このデータをくれ」と要求してきます。**

```cpp
//---------------------------------------------------------------
// ① ドライバ生成のデバッグ情報
//---------------------------------------------------------------
void GpuCrashTracker::OnShaderDebugInfoLookup(
    const GFSDK_Aftermath_ShaderDebugInfoIdentifier& identifier,
    PFN_GFSDK_Aftermath_SetData setShaderDebugInfo) const
{
    const std::lock_guard lock(m_mutex);

    const auto it = m_shaderDebugInfo.find(identifier);
    if (it == m_shaderDebugInfo.end())
    {
        return;      // 見つからなければ何もしない
    }

    setShaderDebugInfo(it->second.data(),
                       static_cast<std::uint32_t>(it->second.size()));
}

//---------------------------------------------------------------
// ② シェーダーバイナリ(.cso)
//---------------------------------------------------------------
void GpuCrashTracker::OnShaderLookup(
    const GFSDK_Aftermath_ShaderBinaryHash& hash,
    PFN_GFSDK_Aftermath_SetData setShaderBinary) const
{
    const auto data = ShaderRegistry::Instance().FindBinary(hash.hash);
    if (data.empty()) return;

    setShaderBinary(data.data(), static_cast<std::uint32_t>(data.size()));
}

//---------------------------------------------------------------
// ③ シェーダーの PDB
//---------------------------------------------------------------
void GpuCrashTracker::OnShaderSourceDebugInfoLookup(
    const GFSDK_Aftermath_ShaderDebugName& debugName,
    PFN_GFSDK_Aftermath_SetData setShaderBinary) const
{
    //--- DebugName = PDB のファイル名(31.3.3 節)---
    const auto pdbPath = m_pdbDirectory /
        std::filesystem::path(std::string(debugName.name));

    const auto data = ReadFile(pdbPath);
    if (data.empty())
    {
        LOG_WARN(L"shader pdb not found: {}", pdbPath.wstring());
        return;
    }

    setShaderBinary(data.data(), static_cast<std::uint32_t>(data.size()));
}
```

**③ が最も重要です。** これがなければ、HLSL のソースコードは表示されません。

**そして、PDB のファイル名を推測する必要がありません。** `debugName.name` がそのままファイル名です。**第13章の設計が、ここで完全に噛み合いました。**

### 31.3.5 マーカーの解決コールバック

**31.2.3 節で、マーカーをポインタ値として渡しました。** 解析時に、それが何を意味するかを教えます。

```cpp
void GpuCrashTracker::OnResolveMarker(
    const void* markerData,
    std::uint32_t markerDataSize,
    PFN_GFSDK_Aftermath_SetData setMarkerData) const
{
    //--- サイズ 0 で渡したので、ポインタ値が ID ---
    if (markerDataSize != 0)
    {
        return;      // 通常のマーカーは何もしなくてよい
    }

    const auto id = reinterpret_cast<std::uint64_t>(markerData);
    const auto name = MarkerRegistry::Instance().Lookup(id);

    if (name.empty()) return;

    setMarkerData(name.data(), static_cast<std::uint32_t>(name.size()));
}
```

**これがないと、ダンプには生のポインタ値しか記録されません。**

---

## 31.4 ダンプの中身を読む

### 31.4.1 Nsight Graphics で開く

**生成された `.nv-gpudmp` を、Nsight Graphics で開きます。**

```
File → Open → crash.nv-gpudmp
```

**第29章 29.1.3 節で書いた通り、キャプチャとは別の操作です。** ダンプを「開く」だけなので、デバッガのアタッチは不要です。

### 31.4.2 何が表示されるか

**主要な情報は 4 つです。**

#### ① クラッシュの種類

```
GPU Exception: Page Fault
```

| 種類 | 意味 |
|---|---|
| **Page Fault** | 無効なメモリアドレスへのアクセス |
| **Timeout (TDR)** | 一定時間内に完了しなかった(第8章 8.1.1 節) |
| **Illegal Instruction** | 不正な命令 |

#### ② イベントマーカーの状態

```
Markers:
  Queue 0, CommandList 0:
    [Completed]  Shadow Pass
    [Completed]  Opaque
    [Executing]  Bloom          ← ここで止まっている
```

**31.2 節で仕込んだマーカーが、ここに現れます。**

**`Executing` のマーカーが、クラッシュ地点です。**

#### ③ ページフォルトの詳細

```
Page Fault:
  GPU Virtual Address: 0x0000000204A00000
  Access Type: Read
  Resource: 'BloomTexture[0]'         ← 名前が出る
    Size: 2073600 bytes
    Dimension: TEXTURE2D
    Format: R16G16B16A16_FLOAT
```

**リソース名が表示されています。**

**第6章 6.5.2 節で予告した内容が、ここで実現しました。**

> **名前を付けていない場合**
> ```
> Resource: <unnamed>  (size 8388608, D3D12_RESOURCE_DIMENSION_BUFFER)
> ```
> アドレスとサイズだけです。数十のバッファを作っていたら、どれのことか特定できません。

**`EnableResourceTracking` フラグ(31.1.2 節)を有効にしたので、リソースが特定できています。**

#### ④ シェーダーの実行位置

```
Active Shaders:
  Pixel Shader (hash: 1a2b3c4d5e6f7a8b)
    Source: BloomBlur.hlsl

    Line 47:  color += gSource.Sample(gSampler, input.uv + delta).rgb * weight;
              ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
              Crash location
```

**HLSL のソースコードと行番号が表示されています。**

**これが本章の目標です。** そして、これが表示されるためには次のすべてが必要でした。

| 必要なもの | 出典 |
|---|---|
| `-Zi` でコンパイル | 第13章 13.6.4 節 |
| `-Fd` で PDB を分離出力 | 第13章 13.6.2 節 |
| `GenerateShaderDebugInfo` フラグ | 31.1.3 節 |
| 3 つのルックアップコールバック | 31.3.4 節 |
| **PDB ファイルの保管** | 第13章 13.6.3 節のコラム |

**1 つでも欠けると、機械語のアドレスしか出ません。**

### 31.4.3 自前で解析する

**Nsight Graphics を使わず、プログラムから解析することもできます。**

```cpp
Core::Status AnalyzeCrashDump(std::span<const std::byte> dumpData)
{
    //--- ① デコーダを作る ---
    GFSDK_Aftermath_GpuCrashDump_Decoder decoder{};

    auto result = GFSDK_Aftermath_GpuCrashDump_CreateDecoder(
        GFSDK_Aftermath_Version_API,
        dumpData.data(),
        static_cast<std::uint32_t>(dumpData.size()),
        &decoder);

    if (!GFSDK_Aftermath_SUCCEED(result))
    {
        return std::unexpected(Core::MakeError(E_FAIL, L"CreateDecoder"));
    }

    //--- 終了時に必ず破棄する ---
    struct DecoderGuard
    {
        GFSDK_Aftermath_GpuCrashDump_Decoder handle;
        ~DecoderGuard() { GFSDK_Aftermath_GpuCrashDump_DestroyDecoder(handle); }
    } guard{ decoder };

    //--- ② 基本情報 ---
    GFSDK_Aftermath_GpuCrashDump_BaseInfo baseInfo{};
    GFSDK_Aftermath_GpuCrashDump_GetBaseInfo(decoder, &baseInfo);

    LOG_INFO(L"crash dump: pid {}, gpu {}",
             baseInfo.pid,
             Core::ToWide(baseInfo.deviceDebugName));

    //--- ③ ページフォルト情報 ---
    GFSDK_Aftermath_GpuCrashDump_PageFaultInfo pageFault{};
    if (GFSDK_Aftermath_SUCCEED(
            GFSDK_Aftermath_GpuCrashDump_GetPageFaultInfo(
                decoder, &pageFault)))
    {
        LOG_ERROR(L"page fault at {:#018x}", pageFault.faultingGpuVA);

        if (pageFault.bHasResourceInfo)
        {
            LOG_ERROR(L"  resource: {} x {}, format {}",
                      pageFault.resourceInfo.width,
                      pageFault.resourceInfo.height,
                      pageFault.resourceInfo.format);
        }
    }

    //--- ④ JSON として出力する ---
    const std::uint32_t decoderFlags =
          GFSDK_Aftermath_GpuCrashDumpDecoderFlags_ALL_INFO;

    std::uint32_t jsonSize = 0;
    result = GFSDK_Aftermath_GpuCrashDump_GenerateJSON(
        decoder,
        decoderFlags,
        GFSDK_Aftermath_GpuCrashDumpFormatterFlags_UTF8_OUTPUT,
        ShaderDebugInfoLookupCallback,
        ShaderLookupCallback,
        ShaderSourceDebugInfoLookupCallback,
        this,
        &jsonSize);

    if (GFSDK_Aftermath_SUCCEED(result) && jsonSize > 0)
    {
        std::vector<char> json(jsonSize);
        GFSDK_Aftermath_GpuCrashDump_GetJSON(
            decoder, jsonSize, json.data());

        WriteTextFile(m_dumpDirectory / L"crash.json", json);
        LOG_INFO(L"crash dump JSON written ({} bytes)", jsonSize);
    }

    return {};
}
```

**JSON 出力が有用です。**

**CI に組み込めば、自動テストで発生したクラッシュを機械的に解析できます。**

```json
{
  "GPUCrashDump": {
    "PageFault": {
      "FaultingGpuVA": "0x204A00000",
      "ResourceInfo": { "Width": 960, "Height": 540, ... }
    },
    "ActiveWarps": [ ... ],
    "ShaderInfo": [ ... ]
  }
}
```

---

## 31.5 意図的にクラッシュさせる

**本章の実習です。**

**3 種類のクラッシュを起こし、それぞれのダンプがどう見えるかを確認します。**

> **重要:実験の前に**
>
> - **Nsight Graphics や PIX をアタッチしないでください**(第29章 29.1.3 節)
> - **作業中のファイルは保存してください**
> - **TDR が発火するので、数秒間画面が固まります**
> - **稀にドライバの復帰に失敗し、再起動が必要になることがあります**

### 31.5.1 実験 A:無限ループ

**最も単純なクラッシュです。**

```hlsl
//=====================================================
// shaders/CrashTest.hlsl
//=====================================================

RWStructuredBuffer<uint> gOutput : register(u0);

[numthreads(64, 1, 1)]
void CSInfiniteLoop(uint3 id : SV_DispatchThreadID)
{
    uint counter = 0;

    //--- 終了しないループ ---
    // コンパイラに最適化されないよう、外部データに依存させる
    while (gOutput[0] != 0xFFFFFFFF)
    {
        ++counter;
    }

    gOutput[id.x] = counter;
}
```

**実行すると、数秒後に TDR が発火します。**

```
[Fatal] Renderer.cpp(412): === DEVICE LOST ===
[Fatal] Renderer.cpp(413): reason: DXGI_ERROR_DEVICE_HUNG (0x887A0006)
[Info ] Aftermath.cpp(178): waiting for crash dump...
[Info ] Aftermath.cpp(185): crash dump written: crash-20260731-142317.nv-gpudmp
```

**第10章 10.5.4 節で実装した「ダンプ生成の完了を待つ」処理が、ここで働いています。**

**待たなければ、ファイルは残りません。**

**ダンプを開くと、こう表示されます。**

```
GPU Exception: Timeout

Markers:
  [Executing]  Crash Test

Active Shaders:
  Compute Shader
    Source: CrashTest.hlsl
    Line 14:  while (gOutput[0] != 0xFFFFFFFF)
              Crash location
```

**無限ループの行が特定されました。**

### 31.5.2 実験 B:範囲外アクセス

**ページフォルトを起こします。**

```hlsl
RWStructuredBuffer<uint> gOutput : register(u0);

cbuffer CrashConstants : register(b0)
{
    uint elementCount;
    uint outOfBoundsOffset;
};

[numthreads(64, 1, 1)]
void CSOutOfBounds(uint3 id : SV_DispatchThreadID)
{
    //--- バッファのサイズを大きく超えたアクセス ---
    const uint index = id.x + outOfBoundsOffset;
    gOutput[index] = id.x;
}
```

```cpp
//--- 定数バッファに巨大なオフセットを設定 ---
constants.elementCount      = 1024;
constants.outOfBoundsOffset = 0x10000000;   // ❌ 極端に大きい
```

**ダンプの内容:**

```
GPU Exception: Page Fault

Page Fault:
  GPU Virtual Address: 0x0000000740000000
  Access Type: Write
  Resource: 'CrashTestBuffer'
    Size: 4096 bytes
    GPU VA: 0x0000000204A00000 - 0x0000000204A01000

  Note: Faulting address is outside any known resource.

Active Shaders:
  Compute Shader
    Source: CrashTest.hlsl
    Line 18:  gOutput[index] = id.x;
              Crash location
```

**「どのリソースの、どの範囲を超えたか」まで分かります。**

**`EnableResourceTracking` フラグの効果です**(31.1.2 節)。

### 31.5.3 実験 C:解放済みリソースへのアクセス

**最も実践的なケースです。**

**第16章 16.3.4 節で警告した内容を、意図的に再現します。**

```cpp
//--- ❌ GPU の完了を待たずに中間バッファを解放する ---
{
    ComPtr<ID3D12Resource> staging;
    // ... 中間バッファを作り、CopyBufferRegion を記録 ...
}   // ← ここで解放される

commandList->Close();
queue->ExecuteCommandLists(1, lists);
// GPU がコピーを実行するとき、staging は既に解放済み
```

**症状は不安定です。**

| 結果 | 頻度 |
|---|---|
| 何も起きない | **よくある**(たまたま間に合った) |
| 絵が壊れる | ときどき |
| ページフォルト | ときどき |

**「何も起きない」のが最も危険**だと、第16章で書きました。

**ページフォルトが起きた場合のダンプ:**

```
Page Fault:
  GPU Virtual Address: 0x0000000204B00000
  Access Type: Read
  Resource: <freed>
    Note: This address was previously mapped to a resource
          that has been destroyed.
```

**`<freed>` と表示されます。** 解放済みリソースへのアクセスだと分かります。

**この情報がなければ、原因の特定は困難です。**

### 31.5.4 実験のためのフレームワーク

**実験を切り替えられるようにしておきます。**

```cpp
enum class CrashTest
{
    None,
    InfiniteLoop,
    OutOfBounds,
    UseAfterFree,
};

void Renderer::RunCrashTest(CrashTest test)
{
    if (test == CrashTest::None) return;

    LOG_WARN(L"=== CRASH TEST: {} ===", ToString(test));
    LOG_WARN(L"The GPU will hang. This is intentional.");

    GPU_MARKER(m_commandList.Get(), m_aftermathContext, "Crash Test");

    switch (test)
    {
    case CrashTest::InfiniteLoop:
        m_commandList->SetPipelineState(m_crashInfiniteLoopPso.Get());
        m_commandList->Dispatch(1, 1, 1);
        break;

    case CrashTest::OutOfBounds:
        m_commandList->SetPipelineState(m_crashOutOfBoundsPso.Get());
        m_commandList->Dispatch(1, 1, 1);
        break;

    // ...
    }
}
```

**マーカーを打っておく**のを忘れないでください。**どのテストで落ちたかが、ダンプに記録されます。**

---

## 31.6 製品版に組み込む

### 31.6.1 何を残すか

| 機能 | 開発時 | **製品版** |
|---|---|---|
| クラッシュダンプ生成 | ○ | **○** |
| イベントマーカー | ○ | **○** |
| リソース追跡 | ○ | **○** |
| シェーダーデバッグ情報 | ○ | **○** |
| **呼び出し履歴** | 必要時 | **×** |
| デバッグレイヤー | ○ | × |
| GPU-Based Validation | 必要時 | × |

**マーカーとリソース追跡は残します。** コストが小さく、価値が大きいためです。

**呼び出し履歴だけは外します**(31.1.2 節)。

### 31.6.2 PDB を保管する

**第13章 13.6.3 節のコラムで予告した内容です。**

> **エンドユーザーからクラッシュダンプを回収する設計にするなら、リリースビルドの PDB は保管しなければなりません。** ダンプだけあっても、対応する PDB がなければ解読できません。

**ビルドごとに、次を保管します。**

```
releases/
└─ v1.2.3/
   ├─ shaders/
   │   ├─ *.cso           ← シェーダーバイナリ
   │   └─ pdb/
   │       └─ *.pdb       ← シェーダーの PDB
   ├─ D3D12Book.pdb       ← C++ の PDB
   └─ build-info.json     ← バージョン、日時、コミットハッシュ
```

**ビルド番号をダンプに埋め込みます。**

```cpp
void GpuCrashTracker::OnCrashDumpDescription(
    PFN_GFSDK_Aftermath_AddGpuCrashDumpDescription addDescription)
{
    addDescription(
        GFSDK_Aftermath_GpuCrashDumpDescriptionKey_ApplicationName,
        "D3D12Book");

    addDescription(
        GFSDK_Aftermath_GpuCrashDumpDescriptionKey_ApplicationVersion,
        kBuildVersion);

    //--- 任意のキーも追加できる ---
    addDescription(
        GFSDK_Aftermath_GpuCrashDumpDescriptionKey_UserDefined,
        kGitCommitHash);

    addDescription(
        GFSDK_Aftermath_GpuCrashDumpDescriptionKey_UserDefined + 1,
        GetCurrentSceneName());
}
```

**ダンプを受け取ったとき、どのビルドか、どのシーンかが分かります。**

### 31.6.3 ダンプを回収する

**エンドユーザーの環境で発生したクラッシュを集める仕組みです。**

```cpp
void GpuCrashTracker::OnCrashDump(const void* dumpData,
                                  std::uint32_t size)
{
    //--- ① ファイルに保存 ---
    const auto fileName = std::format(L"crash-{:%Y%m%d-%H%M%S}.nv-gpudmp",
                                      std::chrono::system_clock::now());
    const auto path = GetCrashDumpDirectory() / fileName;

    WriteFile(path, std::span{
        static_cast<const std::byte*>(dumpData), size });

    LOG_FATAL(L"crash dump written: {}", path.wstring());

    //--- ② ユーザーに通知 ---
    m_crashDumpPath = path;
    m_crashDumpReady = true;
}
```

**次回起動時に、送信するかを尋ねます。**

```cpp
void CheckPendingCrashDumps()
{
    const auto dumps = FindCrashDumps(GetCrashDumpDirectory());
    if (dumps.empty()) return;

    const int result = ::MessageBoxW(
        nullptr,
        L"前回の実行中に問題が発生しました。\n"
        L"診断情報を開発元に送信しますか?",
        L"D3D12Book",
        MB_YESNO | MB_ICONQUESTION);

    if (result == IDYES)
    {
        UploadCrashDumps(dumps);
    }

    DeleteCrashDumps(dumps);
}
```

**必ず同意を求めてください。** クラッシュダンプには、実行環境の情報が含まれます。

### 31.6.4 ダンプに含まれる情報

**送信する前に、何が含まれるかを把握しておくべきです。**

| 含まれるもの | 備考 |
|---|---|
| GPU の型番、ドライバのバージョン | |
| クラッシュ時の GPU の状態 | |
| リソースの名前とサイズ | **第6章で付けた名前** |
| イベントマーカーの名前 | 31.2 節 |
| シェーダーのハッシュ | ソースコードそのものは含まれない |
| プロセス ID、アプリ名 | |

**シェーダーのソースコードは含まれません。** PDB は開発元が持っているものです。

**ただし、リソース名やマーカー名は文字列としてそのまま入ります。** 内部的な名称を使っている場合は、注意してください。

---

## ✅ 本章のゴール:わざと落として、原因行を特定できる

### Step 1:機能フラグを確認する

```
[Info ] Aftermath.cpp(92): Aftermath initialized (flags 0x1b)
[Info ] Aftermath.cpp(95):   markers            : yes
[Info ] Aftermath.cpp(96):   resource tracking  : yes
[Info ] Aftermath.cpp(97):   shader debug info  : yes
[Info ] Aftermath.cpp(98):   call stack         : no
```

### Step 2:シェーダーを登録する

```
[Trace] ShaderRegistry.cpp(48): shader registered: Lit.VS.cso
        (hash 3f2a1b8c9d0e4f5a, debug name 1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d.pdb)
```

**DebugName が PDB のファイル名と一致していることを確認してください。**

```
build/x64/Debug/shaders/pdb/
  └─ 1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d.pdb   ← 一致
```

**一致しない場合、第13章 13.6.2 節の `-Fd` 設定を確認してください。**

### Step 3:マーカーが記録されているか確認する

**この段階では、まだクラッシュさせません。**

**Nsight Graphics でキャプチャして、マーカーが見えることを確認します。**

**ただし、第29章 29.1.3 節の通り、Aftermath マーカーはツールでは見えません。** 見えるのは第29章の `GPU_EVENT` のほうです。

**Aftermath マーカーの確認は、Step 4 で行います。**

### Step 4:実験 A(無限ループ)

**31.5.1 節のシェーダーを実行します。**

**確認すること:**

- [ ] TDR が発火し、数秒後にデバイスロストが検出される
- [ ] ダンプファイルが生成される
- [ ] **ダンプ生成の完了を待ってから終了している**(第10章 10.5.4 節)

**ダンプを Nsight Graphics で開き、次を確認します:**

- [ ] `GPU Exception: Timeout` と表示される
- [ ] マーカー `Crash Test` が `Executing` 状態
- [ ] **`CrashTest.hlsl` の行番号が表示される**

### Step 5:実験 B(範囲外アクセス)

**確認すること:**

- [ ] `GPU Exception: Page Fault`
- [ ] **フォルトアドレスが表示される**
- [ ] **リソース名が表示される**
- [ ] シェーダーの行番号が表示される

### Step 6:PDB を隠してみる

**Step 5 の状態で、PDB ディレクトリの名前を変えてください。**

```
shaders/pdb/  →  shaders/pdb_hidden/
```

**再度クラッシュさせ、ダンプを開きます。**

```
Active Shaders:
  Compute Shader (hash: 1a2b3c4d5e6f7a8b)
    Source: <not available>

    0x00000138:  IMAD R2, R0, c[0x0][0x0], RZ
    0x00000148:  STG.E [R2], R3
                 ^^^^^^^^^^^^^^ Crash location
```

**機械語のアドレスしか出ません。**

**第13章 13.6.3 節で「後から設定しても手遅れ」と書いた理由が、これで完全に分かります。**

**PDB を元に戻してください。**

### Step 7:リソース追跡を無効にしてみる

```cpp
constexpr std::uint32_t flags =
      GFSDK_Aftermath_FeatureFlags_EnableMarkers
    | GFSDK_Aftermath_FeatureFlags_GenerateShaderDebugInfo;
    // EnableResourceTracking を外す
```

**実験 B を再実行します。**

```
Page Fault:
  GPU Virtual Address: 0x0000000740000000
  Resource: <unknown>
```

**リソースが特定できなくなります。**

**フラグを戻してください。**

### Step 8:デバッグ名を消してみる

```cpp
#define D3D12_ENABLE_DEBUG_NAMES 0
```

**第6章 6.5.3 節で作ったマクロです。**

**実験 B を再実行します。**

```
Resource: <unnamed>
  Size: 4096 bytes
```

**名前が出なくなります。**

**第6章から続けてきた習慣の価値が、ここで最も明確になります。**

**元に戻してください。**

### Step 9:JSON 解析を試す

**31.4.3 節の自前解析を実行します。**

```
[Info ] CrashAnalyzer.cpp(88): crash dump JSON written (48213 bytes)
```

**`crash.json` を開いて、構造を確認してください。**

**CI に組み込む場合、この JSON を解析することになります。**

---

### 本章の達成状態

- [ ] 機能フラグを設定した(`CallStackCapturing` は既定で無効)
- [ ] `DeferDebugInfoCallbacks` を使っている
- [ ] コンテキストハンドルを作成・破棄している
- [ ] マーカー ID をレジストリで管理している
- [ ] サイズ 0 でマーカーを設定している
- [ ] `ResolveMarkerCallback` を実装した
- [ ] シェーダーバイナリをハッシュで登録した
- [ ] `GetShaderDebugName` で PDB 名を取得している
- [ ] 3 つのルックアップコールバックを実装した
- [ ] 第29章のマーカーと統合した
- [ ] **実験 A で無限ループの行を特定できた**
- [ ] **実験 B でページフォルトのリソースを特定できた**
- [ ] Step 6 で PDB がない場合の見え方を確認した
- [ ] Step 8 でデバッグ名の価値を確認した
- [ ] 製品版の設定を用意した

---

## トラブルシューティング

| 症状 | 原因 | 対処 |
|---|---|---|
| ダンプが生成されない | デバッガがアタッチ中 | 第29章 29.1.3 節 |
| 同上 | 初期化順序が違う | 第8章 8.3 節 |
| 同上 | 生成完了を待っていない | 第10章 10.5.4 節 |
| マーカーが出ない | `EnableMarkers` が無効 | 31.1.3 |
| マーカーが数値のまま | `ResolveMarkerCallback` 未実装 | 31.3.5 |
| リソース名が `<unnamed>` | `SetName` を呼んでいない | 第6章 6.5 節 |
| リソースが `<unknown>` | `EnableResourceTracking` 無効 | 31.1.3 |
| シェーダーのソースが出ない | PDB が見つからない | 31.3.4 の ③ |
| 同上 | `-Zi` を付けていない | 第13章 13.6.4 節 |
| 同上 | `GetShaderHash` で PDB を探した | `GetShaderDebugName` を使う(31.3.3) |
| 極端に遅い | `CallStackCapturing` が有効 | 31.1.2 |
| ドライバが復帰しない | TDR の限界 | PC を再起動する |

---

## まとめ

**1. 3 つの識別子を混同しない。**
シェーダーハッシュ、DebugName、ShaderDebugInfoIdentifier。**PDB を探すのは `GetShaderDebugName` です。**

**2. マーカーはサイズ 0 で渡す。**
ポインタ値が ID として扱われ、コピーが発生しません。**代わりに `ResolveMarkerCallback` で名前を教えます。**

**3. HLSL の行番号が出るには、5 つの条件が揃う必要がある。**
`-Zi`、`-Fd`、機能フラグ、3 つのコールバック、PDB の保管。**1 つでも欠ければ機械語だけです。**

**4. 第13章の設計が、ここで完全に噛み合った。**
`-Fd` にディレクトリを渡したので PDB がハッシュ名になり、`GetShaderDebugName` が返す名前と一致します。**推測が不要です。**

**5. 第6章のデバッグ名が、最大の価値を発揮した。**
`Resource: 'BloomTexture[0]'` と `Resource: <unnamed>` の差は、原因特定に要する時間の差そのものです。

**6. 第10章の「ダンプ生成を待つ」が、なければ何も残らない。**
検出してすぐ終了すると、書き込み途中でプロセスが消えます。

**7. 製品版でも、マーカーとリソース追跡は残す。**
コストが小さく、エンドユーザーの環境で発生したクラッシュを解析できる価値は大きいためです。

---

## 第4部を終えて

**デバッグの道具が揃いました。**

| 章 | 得たもの | 使う場面 |
|---|---|---|
| 第29章 | Nsight Graphics | **絵がおかしい** |
| 第30章 | Enhanced Barriers と GBV | **たまに壊れる** |
| 第31章 | Aftermath | **GPU がクラッシュする** |

**第2章 2.4.4 節で示したツールの使い分けが、実際の経験を伴って理解できたはずです。**

**そして、ここまでの準備がすべて回収されました。**

```
第4章  Agility SDK      → 最新機能の解析
第6章  デバッグ名        → リソースの特定
第7章  デバッグレイヤー   → API 誤用の検出
第13章 シェーダー PDB    → 行番号の特定
第8章  Aftermath 最小構成 → クラッシュダンプの生成
```

**「まだ三角形も出ていないのに、なぜこんな準備を」と思われた設定が、すべてここに集約しました。**

**第5部からは、モダンな Direct3D 12 の機能に進みます。** コンピュートシェーダー、バインドレス、メッシュシェーダー、レイトレーシング。**どれも「書いたつもりが動かない」ことが日常的に起こる領域です。**

**だからこそ、道具を先に揃えました。**

---

## 参考リンク

| 内容 | URL |
|---|---|
| Nsight Aftermath SDK ガイド | https://docs.nvidia.com/nsight-aftermath/SDK/ |
| API の使い方 | https://docs.nvidia.com/nsight-aftermath/SDK/topics/api_usage.html |
| シェーダーのコンパイル | https://docs.nvidia.com/nsight-aftermath/SDK/topics/shader_compilation.html |
| クラッシュダンプの読み方 | https://docs.nvidia.com/nsight-aftermath/SDK/topics/reading_dumps.html |
| 公式サンプル | https://github.com/NVIDIA/nsight-aftermath-samples |
