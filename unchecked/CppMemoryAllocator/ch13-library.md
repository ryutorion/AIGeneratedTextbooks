# 第13章 ここまでをライブラリに切り出す

---

## この章のゴール

`Playground.cpp` は、そろそろ 600 行を超えているはずです。中身は雑多です。

- アラインメント計算のユーティリティ
- エラー型
- 破棄リストの実装
- `Bump` クラス本体
- ベンチマークヘルパー
- テスト関数が10個以上
- `main`

このまま第20章まで進むと、プールアロケーター、フリーリスト、バディ、TLSF が加わります。**1ファイルでは確実に破綻します。**

この章では整理します。

- ソリューションに静的ライブラリ `AllocatorLib` を追加する
- 名前空間 `ga` を導入する
- ヘッダを役割ごとに分割し、公開ヘッダと内部ヘッダを区別する
- **プロパティシート** で、ビルド設定をプロジェクト間で共有する
- プリコンパイル済みヘッダを設定し、効果を測る
- 循環インクルードを避ける依存関係の作り方を身につける

コードは1行も変わりません。**構造だけを変えます。**

---

## 13.1 完成形を先に見る

目指すフォルダ構成です。

```
GrowingAllocator/
├─ GrowingAllocator.sln
│
├─ build/
│   └─ Common.props              ← 全プロジェクト共通のビルド設定
│
├─ AllocatorLib/
│   ├─ AllocatorLib.vcxproj
│   ├─ include/
│   │   └─ ga/
│   │       ├─ Allocator.h       ← まとめてインクルードする用
│   │       ├─ Core.h            ← 基本的な型と関数
│   │       ├─ AllocError.h      ← エラー型
│   │       ├─ Bump.h            ← Bump アロケーター
│   │       └─ detail/
│   │           └─ Finalizer.h   ← 内部実装(直接使わない)
│   └─ src/
│       ├─ pch.h
│       ├─ pch.cpp
│       └─ AllocatorLib.cpp
│
└─ Playground/
    ├─ Playground.vcxproj
    ├─ pch.h
    ├─ pch.cpp
    ├─ Bench.h                   ← ベンチマークヘルパー
    ├─ Tests.h
    ├─ Tests.cpp                 ← テスト関数
    └─ main.cpp
```

`include/ga/` という2階層になっているのが要点です。理由は 13.5 節で説明します。

---

## 13.2 静的ライブラリプロジェクトを追加する

### プロジェクトの作成

1. ソリューションエクスプローラーで **ソリューション「GrowingAllocator」を右クリック**
2. **追加 → 新しいプロジェクト**
3. テンプレートから **「スタティック ライブラリ」**(C++、Windows)を選択
4. プロジェクト名:`AllocatorLib`
5. 場所がソリューションフォルダ直下になっていることを確認して **作成**

自動生成されるファイル(`AllocatorLib.cpp`、`pch.h`、`pch.cpp`、`framework.h`)は、あとで整理します。

### プラットフォームを確認する

新しいプロジェクトのプラットフォームが `x64` になっていることを確認してください。`Playground` と揃っていないとリンクできません。

---

## 13.3 プロパティシートで設定を共有する

ここで、第1章とは違うやり方を導入します。

第1章では、プロジェクトのプロパティを直接編集しました。しかしプロジェクトが2つになった今、同じ設定を2回書くことになります。第20章以降でテスト用のプロジェクトなどを足せば、3回、4回と増えていきます。

**設定を1か所にまとめる仕組み** が Visual Studio にはあります。プロパティシート(`.props` ファイル)です。

### 作成する

1. メニューの **表示 → その他のウィンドウ → プロパティ マネージャー**
2. `Playground` を右クリック → **新しいプロジェクト プロパティ シートの追加**
3. 名前:`Common.props`
4. 保存先:ソリューションフォルダの下の `build` フォルダ(なければ作る)

### 設定を書き込む

追加された `Common.props` をダブルクリックすると、見慣れたプロパティページが開きます。ここに第1章の設定を入れます。

| 項目 | 値 |
|---|---|
| 全般 → C++ 言語標準 | `/std:c++latest` |
| C/C++ → コマンド ライン → 追加のオプション | `/utf-8` `/Zc:__cplusplus` |
| C/C++ → 全般 → 追加のインクルード ディレクトリ | `$(SolutionDir)AllocatorLib\include` |

**構成は「すべての構成」「すべてのプラットフォーム」** にしてください。プロパティシートでも、この注意は変わりません。

### もう一方のプロジェクトにも適用する

1. プロパティ マネージャーで `AllocatorLib` を右クリック
2. **既存のプロパティ シートの追加**
3. `build\Common.props` を選択

これで、両方のプロジェクトが同じ設定を共有します。**以降、設定を変えるときは `Common.props` を1回編集するだけです。**

> **プロパティシートは、プロジェクトを追加するたびに適用してください。** 忘れると「なぜかこのプロジェクトだけ `<print>` が見つからない」という事態になります。

### 元の設定を消す

`Playground` のプロジェクトプロパティに直接書いた設定は、プロパティシートより優先されます。二重管理を避けるため、**プロジェクト側の設定は「継承する」状態に戻して** ください。

各項目の値のドロップダウンから「継承」を選ぶか、値を空にします。

---

## 13.4 名前空間を導入する

ファイルを分ける前に、名前空間で包みます。

```cpp
namespace ga
{
    class Bump { ... };
}
```

`ga` は "growing allocator" の略です。短い名前を選んだのは、`ga::Bump` のように毎回書くことになるからです。

### なぜ必要か

いま定義している名前を見てください。

```cpp
AlignUp
IsAlignedTo
AllocError
Bump
Marker
Finalizer
kDefaultAlignment
```

**どれも一般的すぎます。** ライブラリとして配布したら、他の誰かの `AlignUp` と衝突します。ゲームエンジンのコードベースには、たいてい同名の関数が既にあります。

### 内部実装は `ga::detail` へ

利用者が直接触るべきでないものは、さらに内側に入れます。

```cpp
namespace ga::detail
{
    struct Finalizer { ... };

    template <class T>
    void DestroyThunk(void* p, std::size_t n) noexcept { ... }
}
```

`detail` という名前は、C++ の世界で「これは内部実装だから触るな」という慣習的な合図です。Boost をはじめ、多くのライブラリが使っています。

**強制力はありません。** `ga::detail::Finalizer` と書けば使えてしまいます。しかし「これは公開 API ではない」という意図は明確に伝わります。

---

## 13.5 ヘッダを分割する

### `include/ga/Core.h`

基本的なユーティリティです。

```cpp
#pragma once

#include <bit>
#include <cassert>
#include <cstddef>
#include <cstdint>

namespace ga
{
    inline constexpr std::size_t kDefaultAlignment = __STDCPP_DEFAULT_NEW_ALIGNMENT__;

    constexpr std::uintptr_t AlignUp(std::uintptr_t value, std::size_t alignment) noexcept
    {
        assert(std::has_single_bit(alignment));

        const std::uintptr_t mask = static_cast<std::uintptr_t>(alignment) - 1;
        return (value + mask) & ~mask;
    }

    inline bool IsAlignedTo(const void* p, std::size_t alignment) noexcept
    {
        return (reinterpret_cast<std::uintptr_t>(p) % alignment) == 0;
    }
}
```

### `include/ga/AllocError.h`

```cpp
#pragma once

#include <expected>

namespace ga
{
    enum class AllocError
    {
        OutOfMemory,
        InvalidAlignment,
        SizeTooLarge,
    };

    constexpr const char* ToString(AllocError e) noexcept
    {
        switch (e)
        {
        case AllocError::OutOfMemory:      return "OutOfMemory";
        case AllocError::InvalidAlignment: return "InvalidAlignment";
        case AllocError::SizeTooLarge:     return "SizeTooLarge";
        }
        return "Unknown";
    }

    using AllocResult = std::expected<void*, AllocError>;
}
```

### `include/ga/detail/Finalizer.h`

```cpp
#pragma once

#include <cstddef>
#include <memory>

namespace ga::detail
{
    struct Finalizer
    {
        void      (*destroy)(void*, std::size_t) noexcept;
        void*       object;
        std::size_t count;
        Finalizer*  next;
    };

    template <class T>
    void DestroyThunk(void* p, std::size_t n) noexcept
    {
        T* array = static_cast<T*>(p);
        for (std::size_t i = n; i > 0; --i)
        {
            std::destroy_at(array + (i - 1));
        }
    }
}
```

### `include/ga/Bump.h`

```cpp
#pragma once

#include "ga/Core.h"
#include "ga/AllocError.h"
#include "ga/detail/Finalizer.h"

#include <limits>
#include <span>
#include <type_traits>
#include <utility>
#include <vector>

namespace ga
{
    template <class T>
    using ArrayResult = std::expected<std::span<T>, AllocError>;

    class Bump
    {
        // (第12章までの実装をそのまま)
    };

    class BumpScope
    {
        // (第9章の実装をそのまま)
    };
}
```

### `include/ga/Allocator.h`

利用者が「とりあえずこれ1つ」で済ませるためのヘッダです。

```cpp
#pragma once

#include "ga/Core.h"
#include "ga/AllocError.h"
#include "ga/Bump.h"
```

第20章以降でアロケーターが増えたら、ここに足していきます。

### 山括弧か、二重引用符か

```cpp
#include "ga/Core.h"     // ← 本書ではこちら
#include <ga/Core.h>
```

規格上の違いは「実装依存の場所を先に探すかどうか」だけで、実際にはどちらでも動きます。

慣習としては、**自分のプロジェクトのヘッダは `"..."`、外部のライブラリは `<...>`** と使い分けるのが一般的です。本書もこれに従います。

### `ga/` を付ける理由

インクルードパスは `AllocatorLib\include` に通してあるので、`#include "Core.h"` でも届きます。しかし `ga/` を付けます。

**理由は衝突の回避です。** `Core.h` という名前のヘッダは、世の中に無数にあります。複数のライブラリを使うプロジェクトで、どれが読まれるか分からない状況は避けたい。

`#include "ga/Core.h"` なら、どのライブラリのものか一目で分かります。**ヘッダのインクルードパスにも名前空間を作る**、という発想です。

---

## 13.6 `.cpp` は必要か

`Bump` はクラステンプレートではありませんが、`New<T>()` や `NewArray<T>()` といったメンバ関数テンプレートを持っています。**テンプレートの定義はヘッダに置く必要があります。**

非テンプレートのメンバだけを `.cpp` に分けることは可能ですが、クラス定義が2か所に散らばって読みにくくなります。

**結論:現時点では、ヘッダオンリーで進めます。**

### では静的ライブラリは無意味か

今のところ、`AllocatorLib` はヘッダを配るだけで、`.lib` の中身はほぼ空です。実際、`.cpp` が1つもないと次の警告が出ます。

```
warning LNK4221: このオブジェクト ファイルは、以前に定義されたシンボルを定義していません
```

**それでもプロジェクトを作る理由は3つあります。**

**1. 第29章以降で必要になる。** `VirtualAlloc` を扱う章から、Windows API を呼ぶ実装コードが出てきます。それを `.cpp` に置きます。`<Windows.h>` を公開ヘッダに含めたくないので、実装を隠す場所が要ります。

**2. 依存関係が明示される。** `Playground` から `AllocatorLib` への一方向の依存が、プロジェクト構成として表現されます。逆向きの依存(ライブラリがテストコードを参照する、など)を作ろうとすれば、すぐ気づけます。

**3. ヘッダの独立性が検査される。** 次項のとおりです。

### ヘッダの独立性を検査する

`AllocatorLib.cpp` を、こう書きます。

```cpp
#include "pch.h"

// 各ヘッダが単独でコンパイルできることを検査する
#include "ga/Core.h"
#include "ga/AllocError.h"
#include "ga/Bump.h"
#include "ga/detail/Finalizer.h"

namespace ga
{
    // LNK4221 を避けるためのシンボル
    const char* LibraryVersion() noexcept
    {
        return "GrowingAllocator 0.8";
    }
}
```

これで、**すべてのヘッダが必ず1回はコンパイルされます。**

ヘッダの独立性——「そのヘッダを単独でインクルードしてもコンパイルが通ること」——は、意識しないとすぐ壊れます。`Bump.h` が `<vector>` のインクルードを忘れていても、`main.cpp` が先に `<vector>` を読んでいれば気づけません。

このファイルがあれば、順序に依存しない形で全ヘッダが検査されます。

---

## 13.7 プロジェクト参照を追加する

`Playground` から `AllocatorLib` を使えるようにします。

1. ソリューションエクスプローラーで **`Playground` を右クリック**
2. **追加 → 参照**
3. `AllocatorLib` にチェックを入れて **OK**

これだけです。Visual Studio が自動的に、

- ビルド順序を調整する(ライブラリを先にビルド)
- `AllocatorLib.lib` をリンクする

をやってくれます。**「追加の依存ファイル」に `.lib` を手書きする必要はありません。**

インクルードパスは 13.3 節でプロパティシートに設定済みなので、そのまま `#include "ga/Allocator.h"` と書けます。

---

## 13.8 プリコンパイル済みヘッダ

### 何が問題なのか

`Bump.h` は `<vector>`、`<span>`、`<expected>`、`<memory>` などを取り込みます。これらは巨大です。プリプロセッサを通した後のソースは、数万行から数十万行に膨れ上がります。

そして、それが **`.cpp` の数だけ繰り返されます**。`main.cpp` と `Tests.cpp` の両方で `<vector>` を1から解析するのは、明らかに無駄です。

### 設定する

プリコンパイル済みヘッダ(PCH)は、この共通部分を1回だけ解析して、結果をファイルに保存する仕組みです。

**`Playground/pch.h`:**

```cpp
#pragma once

// 変更されない、重いヘッダをここに集める
#include <algorithm>
#include <chrono>
#include <cstddef>
#include <cstdint>
#include <expected>
#include <memory>
#include <print>
#include <span>
#include <string>
#include <vector>
```

**`Playground/pch.cpp`:**

```cpp
#include "pch.h"
```

**プロジェクト設定(構成:すべての構成):**

| 対象 | 設定項目 | 値 |
|---|---|---|
| プロジェクト全体 | C/C++ → プリコンパイル済みヘッダー → プリコンパイル済みヘッダー | **使用 (/Yu)** |
| プロジェクト全体 | 同 → プリコンパイル済みヘッダー ファイル | `pch.h` |
| **`pch.cpp` のみ** | 同 → プリコンパイル済みヘッダー | **作成 (/Yc)** |

`pch.cpp` だけ設定が違う点に注意してください。ファイルを右クリック → プロパティで個別に設定します。

**すべての `.cpp` の先頭に** `#include "pch.h"` を書きます。

```cpp
#include "pch.h"       // ← 必ず最初の行

#include "ga/Allocator.h"
#include "Tests.h"
```

### ⚠ 落とし穴

**`/Yu` を使うと、`#include "pch.h"` より前の行はすべて無視されます。**

```cpp
#define _CRTDBG_MAP_ALLOC   // ← 無視される!
#include "pch.h"
```

コンパイラは `pch.h` が現れるまでの内容を読み飛ばします。マクロ定義を先頭に置いたつもりが効かない、という事故が起きます。マクロは `pch.h` の中に書いてください。

**ライブラリの公開ヘッダに `pch.h` を含めてはいけません。**

```cpp
// ga/Bump.h
#include "pch.h"    // ← 絶対にダメ
```

公開ヘッダは、利用者のプロジェクトからインクルードされます。相手のプロジェクトに `pch.h` がある保証はありませんし、あったとしても中身は違います。

**公開ヘッダは、自分が必要とするものを自分でインクルードする。** これが原則です。

### 効果を測る

PCH の有無でビルド時間を比べてみましょう。

**測り方:** ビルド → **ソリューションのリビルド** を実行し、出力ウィンドウの最後に表示される経過時間を見ます。

| | フルリビルド |
|---|---|
| PCH なし | 4.2 秒 |
| PCH あり | **1.8 秒** |

ファイルが2つの段階でも 2 倍以上の差です。ファイルが増えるほど効果は大きくなります。

### あわせて `/MP` も

複数の `.cpp` を並列にコンパイルするオプションです。

`C/C++ → 全般 → マルチプロセッサ コンパイル` を **はい (/MP)** にします。

`.cpp` が増えてくると、これも効いてきます。プロパティシートに書いておきましょう。

---

## 13.9 循環インクルードを避ける

ヘッダを分けると、必ずこの問題が出てきます。`A.h` が `B.h` を含み、`B.h` が `A.h` を含む状態です。

`#pragma once` があるので無限ループにはなりませんが、**片方が不完全な状態でコンパイルされ、意味不明なエラーが出ます。**

### 依存を一方向に保つ

現在の依存関係を図にします。

```
        Allocator.h
             │
    ┌────────┼────────┐
    ▼        ▼        ▼
  Core.h  AllocError.h  Bump.h
                          │
              ┌───────────┴───────┐
              ▼                   ▼
           Core.h          detail/Finalizer.h
```

**矢印がすべて下向きです。** 上位のヘッダが下位を知っていて、逆はありません。

この形を保つコツは、**「階層」を意識すること** です。

| 階層 | 内容 | 何に依存してよいか |
|---|---|---|
| 0 | `Core.h`、`AllocError.h` | 標準ライブラリのみ |
| 1 | `detail/Finalizer.h` | 階層 0 |
| 2 | `Bump.h` | 階層 0、1 |
| 3 | `Allocator.h` | すべて |

**新しいヘッダを足すときは、まず階層を決めてください。** 同じ階層のヘッダ同士がインクルードし合ったら、それは設計の見直しの合図です。

### 前方宣言を使う

ポインタや参照としてしか使わない型は、インクルードせずに宣言だけで済みます。

```cpp
// ga/Bump.h
namespace ga::detail { struct Finalizer; }   // 前方宣言

namespace ga
{
    class Bump
    {
        struct Marker
        {
            detail::Finalizer* finalizers = nullptr;   // ポインタなので OK
        };
    };
}
```

ただし、`Bump` の実装は `Finalizer` のメンバにアクセスするので、結局インクルードが必要です。**前方宣言で済むのは、型の中身を知らなくてよい場合だけ** です。

判断基準:

| 使い方 | 前方宣言で足りるか |
|---|---|
| ポインタ・参照のメンバ | ○ |
| 関数の引数・戻り値(宣言のみ) | ○ |
| 値としてのメンバ | × (サイズが必要) |
| メンバへのアクセス | × |
| 継承 | × |

### ヘッダに書いてはいけないもの

```cpp
using namespace std;              // ← 絶対にダメ
#define MAX(a, b) ...             // ← マクロは名前空間を無視する
```

ヘッダは他人のコードに取り込まれます。そこに `using namespace` を書けば、**インクルードした全員に押し付ける** ことになります。マクロも同様で、名前空間の保護が効きません。

これは、後のコラムで触れる「モジュールが解決しようとした問題」の1つでもあります。

---

## 13.10 テストとベンチを分離する

`Playground` 側も整理します。

**`Bench.h`**(ベンチマークヘルパー):

```cpp
#pragma once

#include <chrono>
#include <cstdint>
#include <vector>

namespace bench
{
    // (第4章の実装をそのまま)
}
```

**`Tests.h`**:

```cpp
#pragma once

namespace tests
{
    void RunAll();
}
```

**`Tests.cpp`**:

```cpp
#include "pch.h"
#include "Tests.h"

#include "ga/Allocator.h"

namespace tests
{
    namespace
    {
        // 個々のテストは、このファイルの外から見えなくてよい
        void Test_AlignUp() { ... }
        void Test_UsedIncreases() { ... }
        // ...
    }

    void RunAll()
    {
        Test_AlignUp();
        Test_UsedIncreases();
        // ...
    }
}
```

個々のテスト関数を **無名名前空間** に入れています。これでこのファイルの外からは見えなくなり、名前の衝突を気にせず済みます。公開するのは `RunAll()` だけです。

**`main.cpp`**:

```cpp
#include "pch.h"

#include "ga/Allocator.h"
#include "Bench.h"
#include "Tests.h"

int main()
{
    tests::RunAll();

    // ベンチマークはここに
}
```

`main.cpp` が 20 行になりました。

---

## 13.11 ビルドを通す

移行作業の手順です。

1. `AllocatorLib/include/ga/` フォルダを作る
2. ヘッダを4つ作り、`Playground.cpp` から該当部分を切り取って貼る
3. すべてを `namespace ga { ... }` で包む
4. `detail` に入れるものを `ga::detail` に移す
5. `Playground.cpp` を `main.cpp` に改名し、テストを `Tests.cpp` へ移す
6. `Bench.h` を作る
7. プロパティシートを両プロジェクトに適用する
8. プロジェクト参照を追加する
9. PCH を設定する
10. ビルドする

### よくあるエラー

**`error C2065: 'Bump': 定義されていない識別子です`**

名前空間を付け忘れています。`ga::Bump` と書くか、`.cpp` の中で `using namespace ga;` を書きます(ヘッダには書かないこと)。

**`error C1010: プリコンパイル ヘッダーの検索中に、予期しない EOF が見つかりました`**

その `.cpp` に `#include "pch.h"` がありません。すべての `.cpp` の先頭に必要です。

**`error C2039: 'span': 'std' のメンバーではありません`**

`Bump.h` が `<span>` をインクルードし忘れています。13.6 節の `AllocatorLib.cpp` を作ってあれば、この種の漏れは必ず検出されます。

**`fatal error C1083: include ファイルを開けません 'ga/Core.h'`**

インクルードパスが通っていません。プロパティシートの「追加のインクルード ディレクトリ」を確認してください。プロパティシートを `AllocatorLib` 側に適用し忘れているケースが多いです。

### 動作確認

```cpp
int main()
{
    tests::RunAll();

    ga::Bump bump(1024);
    auto r = bump.New<int>(42);

    std::println("値: {}", **r);
    std::println("使用量: {} / {}", bump.Used(), bump.Capacity());
}
```

第12章までと同じ結果が出れば成功です。

---

## 演習

**演習13-1** `AllocatorLib.cpp` から `#include "ga/Bump.h"` を消し、`Bump.h` から `<vector>` のインクルードを削ってみてください。どのタイミングでエラーになりますか。

**演習13-2** `ga/Core.h` に `using namespace std;` を書いて、何が起きるか観察してください(観察したら必ず消してください)。

**演習13-3** PCH を無効にして、ビルド時間を測ってください。`.cpp` の数を増やすと、差はどう変わりますか。

**演習13-4** `pch.h` に `ga/Bump.h` を入れると、ビルドはさらに速くなります。しかし本書ではそうしていません。なぜでしょうか。

**演習13-5** `Bump.h` を単独でインクルードするだけの `.cpp` を作り、コンパイルが通ることを確認してください。他のヘッダについても同様に作ると、どんな利点がありますか。

**演習13-6** `ga::detail::Finalizer` を `main.cpp` から直接使ってみてください。コンパイルは通りますか。通るとしたら、`detail` に入れる意味は何でしょうか。

**演習13-7** 静的ライブラリではなく DLL にすると、何が変わりますか。テンプレートを含むライブラリを DLL にする際の問題を調べてください(第41章の予習になります)。

---

## 章末チェックリスト

- [ ] `AllocatorLib` プロジェクトを追加した
- [ ] プロパティシート `Common.props` を作り、両プロジェクトに適用した
- [ ] 名前空間 `ga` と `ga::detail` を導入した
- [ ] ヘッダを役割ごとに分割した
- [ ] インクルードパスに `ga/` を付ける理由を説明できる
- [ ] プロジェクト参照を追加した
- [ ] PCH を設定し、ビルド時間の差を測った
- [ ] 依存関係が一方向であることを図で確認した
- [ ] 第12章までと同じ動作をすることを確認した

---

## 次章の予告

器が整いました。第2部からは、**アロケーターを「見えるようにする」** 作業に入ります。

第14章では、C++20 の `std::source_location` を使って、「どこから確保されたか」を記録します。既定引数として受け取るので、呼び出し側のコードは1文字も変わりません。

```cpp
auto r = bump.Allocate(100);
// → ログに "main.cpp:42 で 100 バイト確保" と出る
```

そこから、統計、タグ付け、メモリパターン、ガードバイト、コールスタック、可視化へと進みます。第3部で個別解放という難問に取り組む前に、**中で何が起きているかを見る目** を手に入れておきます。

---

> **コラム:モジュールを使わない理由**
>
> C++20 で導入されたモジュールは、この章で扱った問題の多くを、根本から解決するものです。
>
> - `#include` のようにテキストを貼り付けないので、**マクロが漏れない**
> - 1回コンパイルした結果を再利用するので、**PCH が要らない**
> - `export` しないものは本当に見えないので、`detail` 名前空間のような慣習が不要
> - インクルード順序に依存しない
> - ODR 違反が起きにくい
>
> 13.9 節で「ヘッダに `using namespace` を書くな」「マクロは名前空間を無視する」と注意しました。モジュールなら、そもそも問題になりません。
>
> **では、なぜ本書は使わないのか。**
>
> ---
>
> **理由1:周辺環境がまだ追いついていない。**
>
> モジュールはビルドシステムに大きな変更を要求します。ヘッダは「どの順にコンパイルしてもよい」のに対し、モジュールは **依存関係の順にビルドしなければなりません**。ビルドシステムがモジュール間の依存を解析し、正しい順序を決める必要があります。
>
> Visual Studio は比較的よく対応していますが、CMake や他のビルドシステム、静的解析ツール、IDE の補完機能まで含めると、成熟しているとは言いがたいのが現状です。「ビルドは通るがインテリセンスが効かない」といった状態は珍しくありません。
>
> **理由2:外の世界はヘッダでできている。**
>
> ゲーム開発で使うライブラリの大半は、いまだにヘッダで配布されています。DirectX、各種ミドルウェア、社内の既存コード。それらを取り込む以上、ヘッダの世界とは付き合い続けることになります。
>
> モジュールとヘッダを混在させることは可能ですが、その境界の扱い(グローバルモジュールフラグメント、`import <vector>;` の可否など)を説明し始めると、それだけで1章が必要になります。**本書の主題はメモリアロケーターであって、ビルドシステムではありません。**
>
> **理由3:読者の環境を想定しにくい。**
>
> 本書のコードを、既存のプロジェクトに組み込みたい読者がいるはずです。そのプロジェクトがモジュールに対応している保証はありません。ヘッダで書いておけば、どんな環境にも持ち込めます。
>
> ---
>
> **ただし、無関係とは考えていません。**
>
> 本書のコードは、モジュール化しやすい形を保つよう気をつけています。
>
> - ヘッダに `using namespace` を書かない
> - マクロをほとんど使わない
> - 依存関係を一方向に保つ
> - 公開と内部を名前空間で分けている
>
> これらはモジュールへの移行を楽にする性質であると同時に、**ヘッダ方式でも単純に良い設計** です。第45章でライブラリとして仕上げるときにも、この方針を維持します。
>
> ---
>
> 最後に、公平のために言い添えておきます。
>
> モジュールを避ける理由の多くは「今はまだ」という時期の問題であり、技術的な優劣ではありません。数年後にこの本を読んでいる読者にとっては、状況が変わっているかもしれません。そのときは、`ga/Bump.h` を `ga.bump` モジュールに書き換えてみてください。**この章で整理した依存関係の階層が、そのままモジュールの構造になるはずです。**
