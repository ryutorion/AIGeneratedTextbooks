# 第19章 アドレス空間を絵にする

---

## この章のゴール

第2部の最後です。ここまで集めてきた情報は、すべて **文字** でした。数字、表、スタックトレース。

この章では **絵にします**。

```
████████████████░░░░░░░░████████░░░░░░░░░░░░░░░░
Mesh                    Texture
```

板の使用状況を、1ピクセル = 1KB として画像に出力します。使用中は色、未使用は暗く。タグごとに色を変えれば、**メモリの中で何がどこに置かれているか** が一目で分かります。

- 何を描くかを設計する
- タグの境界だけを記録する(バンプアロケーターだからできる工夫)
- 外部ライブラリなしで PNG を書き出す(約70行)
- `Reset()` / `Rewind()` の前後を比較する
- **断片化を数値で表す指標** を定義する

最後の項目が、この章の隠れた主題です。バンプアロケーターに断片化は起きません。**まだ存在しない現象を測る道具を、先に作っておきます。** 第4章でベンチマークを先に作ったのと同じ考え方です。

---

## 19.1 なぜ絵にするのか

第15章のレポートは、こう教えてくれました。

```
Mesh      2.10 MB (44.7%)
Texture   3.00 MB (42.8%)
```

**「どれだけ」は分かります。「どこに」は分かりません。**

しかし、メモリ管理では配置が決定的に重要になる場面があります。

- **断片化**:合計では 10 MB 空いているのに、4 MB の確保が失敗する。空きが細切れだから
- **局所性**:同時に使うデータが離れて配置されている(第32章で扱います)
- **寿命の混在**:長生きするデータと短命なデータが交互に並び、解放しても穴が埋まらない

これらはすべて **配置の問題** です。数値の集計では見えません。

そして人間は、**空間的なパターンを見つけるのが得意** です。「なんとなく歯抜けになっている」「後半に固まっている」といった直感は、表を眺めていても働きません。絵にすれば一瞬です。

---

## 19.2 何を描くか

### 1ピクセルに何バイトを対応させるか

板が 16 MB あるとき、1バイト1ピクセルでは 1600 万ピクセルになります。現実的ではありません。

**1ピクセル = 1 KB** を既定にします。16 MB なら 16,384 ピクセル。幅 256 なら高さ 64 の画像になります。

```
16 MB / 1 KB = 16384 ピクセル
16384 / 256 = 64 行
```

1行が 256 KB を表します。

### 走査の向き

左上から右へ、行の終わりで次の行へ。**テキストと同じ順序** です。

```
アドレス低 →→→→→→→→→→→→→→→→→→→
            ↓
            →→→→→→→→→→→→→→→→→→→
            ↓
            →→→→→→→→→→→→→→→→→→→ アドレス高
```

### 色をどう決めるか

| 状態 | 色 |
|---|---|
| 未使用 | 濃い灰色 |
| General | 白 |
| Mesh | 緑 |
| Texture | 橙 |
| Audio | 紫 |
| Script | 青 |
| Physics | 赤 |
| UI | 黄 |
| Debug | 灰 |

**1ピクセルに複数の状態が混ざる場合** はどうするか。1 KB の中に境界があれば、そのピクセルは2つのタグにまたがります。

厳密にやるなら混色しますが、複雑になるだけです。**先頭バイトの状態を採用** します。粒度が粗いことは承知の上で、傾向をつかむのが目的です。

---

## 19.3 タグの境界だけ記録する

色を塗るには、「このアドレスはどのタグか」を知る必要があります。

確保ごとに範囲を記録すると、100 万件で 100 万エントリになります。重すぎます。

### バンプアロケーターだからできる工夫

思い出してください。**バンプアロケーターは、常に前へ進むだけ** です。アドレスは確保順に単調増加します。

ということは、**タグが切り替わった位置だけ記録すれば十分** です。

```
offset:  0        2MB       5MB       6MB
tag:     |--Mesh--|-Texture-|-Script--|
記録:    (0, Mesh) (2MB, Texture) (5MB, Script)
```

境界の間は、すべて同じタグです。第15章の `TagScope` を使っていれば、切り替わりは数十回程度でしょう。

```cpp
namespace ga
{
    struct TagSpan
    {
        std::size_t begin = 0;
        MemoryTag   tag   = MemoryTag::General;
    };
}
```

### 記録する

```cpp
    void RecordTagSpan(std::size_t offset) noexcept
    {
#if GA_ENABLE_ALLOC_TRACKING
        if (!tagSpans_.empty() && tagSpans_.back().tag == currentTag_)
        {
            return;   // タグが変わっていないので記録不要
        }
        tagSpans_.push_back(TagSpan{ offset, currentTag_ });
#else
        (void)offset;
#endif
    }
```

`Allocate()` の中で、確保位置が確定した直後に呼びます。

### `Rewind()` での後始末

第11章以降、機能を足すたびに `Marker` のフィールドが増えてきました。しかし今回は **増やさずに済みます**。

```cpp
    void Rewind(const Marker& m) noexcept
    {
        // ... 従来の処理 ...

        while (!tagSpans_.empty() && tagSpans_.back().begin >= m.offset)
        {
            tagSpans_.pop_back();
        }
    }
```

`begin` がオフセットそのものなので、**巻き戻し先より後ろのものを捨てるだけ** で正しくなります。マーカーに件数を持たせる必要はありません。

> **記録するデータの形を工夫すると、周辺の処理が簡単になる** という好例です。オフセットではなく通し番号で記録していたら、マーカーにフィールドが必要でした。

---

## 19.4 最小の PNG ライター

外部ライブラリを使わずに PNG を書きます。約70行です。

> **本書の主題ではないので、詳細には立ち入りません。** 実務では `stb_image_write.h` のような1ファイルのライブラリを使うほうが現実的です。ここでは「依存を増やさずに済ませる」ことを優先しました。

### PNG の最小構成

```
シグネチャ(8バイト)
IHDR チャンク  ← 幅・高さ・色形式
IDAT チャンク  ← 画像データ(zlib 圧縮)
IEND チャンク  ← 終端
```

各チャンクは `長さ(4) + 種別(4) + データ + CRC32(4)` の形です。

**圧縮は必須ですが、"無圧縮" という圧縮形式が使えます。** deflate には「そのまま格納する(stored)」というブロック形式があるので、これを使えば圧縮アルゴリズムを実装せずに済みます。

### 実装

```cpp
// Playground/Image.h
#pragma once

#include <cstdint>
#include <cstdio>
#include <vector>

namespace img
{
    struct Rgb { std::uint8_t r = 0, g = 0, b = 0; };

    namespace detail
    {
        inline std::uint32_t Crc32(const std::uint8_t* data, std::size_t n)
        {
            static std::uint32_t table[256];
            static bool initialized = false;

            if (!initialized)
            {
                for (std::uint32_t i = 0; i < 256; ++i)
                {
                    std::uint32_t c = i;
                    for (int k = 0; k < 8; ++k)
                    {
                        c = (c & 1) ? (0xEDB88320u ^ (c >> 1)) : (c >> 1);
                    }
                    table[i] = c;
                }
                initialized = true;
            }

            std::uint32_t crc = 0xFFFFFFFFu;
            for (std::size_t i = 0; i < n; ++i)
            {
                crc = table[(crc ^ data[i]) & 0xFF] ^ (crc >> 8);
            }
            return crc ^ 0xFFFFFFFFu;
        }

        inline std::uint32_t Adler32(const std::uint8_t* data, std::size_t n)
        {
            std::uint32_t a = 1, b = 0;
            for (std::size_t i = 0; i < n; ++i)
            {
                a = (a + data[i]) % 65521;
                b = (b + a) % 65521;
            }
            return (b << 16) | a;
        }

        inline void PushBe32(std::vector<std::uint8_t>& out, std::uint32_t v)
        {
            out.push_back(static_cast<std::uint8_t>(v >> 24));
            out.push_back(static_cast<std::uint8_t>(v >> 16));
            out.push_back(static_cast<std::uint8_t>(v >> 8));
            out.push_back(static_cast<std::uint8_t>(v));
        }

        inline void WriteChunk(std::vector<std::uint8_t>& out, const char (&type)[5],
                               const std::uint8_t* data, std::size_t n)
        {
            PushBe32(out, static_cast<std::uint32_t>(n));

            const std::size_t start = out.size();
            out.insert(out.end(), type, type + 4);
            out.insert(out.end(), data, data + n);

            PushBe32(out, Crc32(out.data() + start, out.size() - start));
        }
    }

    class Image
    {
    public:
        Image(int width, int height)
            : width_(width), height_(height)
            , pixels_(static_cast<std::size_t>(width) * height)
        {
        }

        void Set(int x, int y, Rgb c) noexcept
        {
            if (x < 0 || x >= width_ || y < 0 || y >= height_) { return; }
            pixels_[static_cast<std::size_t>(y) * width_ + x] = c;
        }

        int Width()  const noexcept { return width_; }
        int Height() const noexcept { return height_; }

        bool WritePng(const char* path) const
        {
            // --- 生データ:各行の先頭にフィルタ種別 0 ---
            std::vector<std::uint8_t> raw;
            raw.reserve((static_cast<std::size_t>(width_) * 3 + 1) * height_);

            for (int y = 0; y < height_; ++y)
            {
                raw.push_back(0);
                for (int x = 0; x < width_; ++x)
                {
                    const Rgb& c = pixels_[static_cast<std::size_t>(y) * width_ + x];
                    raw.push_back(c.r);
                    raw.push_back(c.g);
                    raw.push_back(c.b);
                }
            }

            // --- zlib ストリーム(無圧縮の deflate)---
            std::vector<std::uint8_t> z;
            z.push_back(0x78);
            z.push_back(0x01);

            std::size_t pos = 0;
            do
            {
                const std::size_t len  = (raw.size() - pos < 65535) ? (raw.size() - pos) : 65535;
                const bool        last = (pos + len == raw.size());

                z.push_back(last ? 1 : 0);
                z.push_back(static_cast<std::uint8_t>(len & 0xFF));
                z.push_back(static_cast<std::uint8_t>(len >> 8));

                const std::uint16_t nlen = static_cast<std::uint16_t>(~len);
                z.push_back(static_cast<std::uint8_t>(nlen & 0xFF));
                z.push_back(static_cast<std::uint8_t>(nlen >> 8));

                z.insert(z.end(), raw.begin() + pos, raw.begin() + pos + len);
                pos += len;
            } while (pos < raw.size());

            detail::PushBe32(z, detail::Adler32(raw.data(), raw.size()));

            // --- ファイル全体を組み立てる ---
            std::vector<std::uint8_t> out = { 0x89, 'P', 'N', 'G', 0x0D, 0x0A, 0x1A, 0x0A };

            std::uint8_t ihdr[13] = {};
            ihdr[0] = static_cast<std::uint8_t>(width_ >> 24);
            ihdr[1] = static_cast<std::uint8_t>(width_ >> 16);
            ihdr[2] = static_cast<std::uint8_t>(width_ >> 8);
            ihdr[3] = static_cast<std::uint8_t>(width_);
            ihdr[4] = static_cast<std::uint8_t>(height_ >> 24);
            ihdr[5] = static_cast<std::uint8_t>(height_ >> 16);
            ihdr[6] = static_cast<std::uint8_t>(height_ >> 8);
            ihdr[7] = static_cast<std::uint8_t>(height_);
            ihdr[8]  = 8;   // ビット深度
            ihdr[9]  = 2;   // カラータイプ:トゥルーカラー(RGB)
            ihdr[10] = 0;   // 圧縮方式
            ihdr[11] = 0;   // フィルタ方式
            ihdr[12] = 0;   // インターレースなし

            detail::WriteChunk(out, "IHDR", ihdr, sizeof(ihdr));
            detail::WriteChunk(out, "IDAT", z.data(), z.size());
            detail::WriteChunk(out, "IEND", nullptr, 0);

            std::FILE* fp = nullptr;
            if (fopen_s(&fp, path, "wb") != 0 || fp == nullptr) { return false; }

            std::fwrite(out.data(), 1, out.size(), fp);
            std::fclose(fp);
            return true;
        }

    private:
        int              width_;
        int              height_;
        std::vector<Rgb> pixels_;
    };
}
```

---

## 19.5 板を描く

```cpp
// Playground/ArenaMap.h
#pragma once

#include "ga/Allocator.h"
#include "Image.h"

namespace viz
{
    struct MapConfig
    {
        std::size_t bytesPerPixel = 1024;
        int         width         = 256;
        int         scale         = 3;      // 1ピクセルを何倍に拡大するか
    };

    inline img::Rgb ColorOf(ga::MemoryTag tag) noexcept
    {
        switch (tag)
        {
        case ga::MemoryTag::General: return { 220, 220, 220 };
        case ga::MemoryTag::Mesh:    return {  80, 200, 100 };
        case ga::MemoryTag::Texture: return { 240, 150,  60 };
        case ga::MemoryTag::Audio:   return { 180, 110, 220 };
        case ga::MemoryTag::Script:  return {  80, 150, 240 };
        case ga::MemoryTag::Physics: return { 230,  90,  90 };
        case ga::MemoryTag::UI:      return { 230, 220,  90 };
        case ga::MemoryTag::Debug:   return { 150, 150, 150 };
        default:                     return { 255,   0, 255 };   // 異常値は目立たせる
        }
    }

    inline constexpr img::Rgb kColorFree{ 30, 30, 34 };

    inline img::Image Render(const ga::Bump& arena, const MapConfig& cfg = {})
    {
        const std::size_t pixelCount =
            (arena.Capacity() + cfg.bytesPerPixel - 1) / cfg.bytesPerPixel;

        const int rows = static_cast<int>((pixelCount + cfg.width - 1) / cfg.width);

        img::Image image(cfg.width * cfg.scale, rows * cfg.scale);

        // タグ境界を配列に取り出す
        std::vector<ga::TagSpan> spans;
        arena.ForEachTagSpan([&](const ga::TagSpan& s) { spans.push_back(s); });

        const std::size_t used = arena.Used();

        std::size_t spanIndex = 0;

        for (std::size_t i = 0; i < pixelCount; ++i)
        {
            const std::size_t byteOffset = i * cfg.bytesPerPixel;

            img::Rgb color = kColorFree;

            if (byteOffset < used)
            {
                // このオフセットが属するタグを探す(オフセットは単調増加なので前進のみ)
                while (spanIndex + 1 < spans.size() && spans[spanIndex + 1].begin <= byteOffset)
                {
                    ++spanIndex;
                }

                if (!spans.empty())
                {
                    color = ColorOf(spans[spanIndex].tag);
                }
            }

            const int px = static_cast<int>(i % cfg.width);
            const int py = static_cast<int>(i / cfg.width);

            for (int sy = 0; sy < cfg.scale; ++sy)
            {
                for (int sx = 0; sx < cfg.scale; ++sx)
                {
                    image.Set(px * cfg.scale + sx, py * cfg.scale + sy, color);
                }
            }
        }

        return image;
    }
}
```

`spanIndex` を前へ進めるだけなので、全体で O(ピクセル数 + 境界数) です。ピクセルごとに二分探索する必要はありません。**バンプアロケーターの単調性が、ここでも効いています。**

---

## 19.6 見てみる

### ロード後の状態

```cpp
int main()
{
    ga::Bump arena(16 * 1024 * 1024);

    {
        GA_TAG(arena, ga::MemoryTag::Mesh);
        LoadMeshes(arena);          // 約 4 MB
    }
    {
        GA_TAG(arena, ga::MemoryTag::Texture);
        LoadTextures(arena);        // 約 3 MB
    }
    {
        GA_TAG(arena, ga::MemoryTag::Script);
        LoadScripts(arena);         // 約 64 KB
    }

    viz::Render(arena).WritePng("arena_after_load.png");
}
```

得られる画像(文字で表すと):

```
行 0-15  ████████████████████████████████  Mesh(緑)
行 16    ████████████████░░░░░░░░░░░░░░░░  Mesh → Texture
行 17-27 ████████████████████████████████  Texture(橙)
行 28    █████░░░░░░░░░░░░░░░░░░░░░░░░░░░  Texture → Script → 空き
行 29-63 ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  未使用(暗い)
```

**きれいな帯になります。** これがバンプアロケーターの姿です。

- 使用領域が先頭に隙間なく詰まっている
- 空きが1つの連続した塊として後ろにある
- タグごとにまとまっている(`TagScope` の順に並ぶ)

### `Reset()` の前後

```cpp
    viz::Render(arena).WritePng("before_reset.png");
    arena.Reset();
    viz::Render(arena).WritePng("after_reset.png");
```

```
before: ████████████████████████░░░░░░░░
after:  ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
```

**一瞬で全部黒になります。** 第8章で「`Reset()` は O(1)」と言ったことが、絵として理解できます。

### `Rewind()` の前後

```cpp
    {
        GA_TAG(arena, ga::MemoryTag::Mesh);
        LoadMeshes(arena);
    }

    {
        ga::BumpScope scope(arena);
        GA_TAG(arena, ga::MemoryTag::Physics);
        BuildCollisionData(arena);

        viz::Render(arena).WritePng("in_scope.png");
    }

    viz::Render(arena).WritePng("after_scope.png");
```

```
in_scope:    ████████████████▓▓▓▓▓▓▓▓░░░░░░░░   緑=Mesh, 赤=Physics
after_scope: ████████████████░░░░░░░░░░░░░░░░   赤が消えた
```

**後ろから削られる** 様子が見えます。スタックアロケーターの動きそのものです。

---

## 19.7 断片化を数値にする

絵は直感的ですが、**比較には数字が要ります**。「昨日より断片化したか」を絵の目視で判断するのは無理です。

指標を定義します。

```cpp
namespace ga
{
    struct FragmentationStats
    {
        std::size_t capacity       = 0;
        std::size_t used           = 0;   // 実際に配ったバイト
        std::size_t internalWaste  = 0;   // パディングで捨てたバイト
        std::size_t freeTotal      = 0;   // 空きの合計
        std::size_t freeLargest    = 0;   // 最大の連続した空き
        std::size_t freeBlockCount = 0;   // 空きブロックの数

        // 板のうち、実際に配れたバイトの割合
        double Utilization() const noexcept
        {
            return (capacity == 0) ? 0.0
                 : static_cast<double>(used) / static_cast<double>(capacity);
        }

        // 配ったバイトのうち、パディングで無駄になった割合
        double InternalFragmentation() const noexcept
        {
            return (used == 0) ? 0.0
                 : static_cast<double>(internalWaste) / static_cast<double>(used);
        }

        // 空きが細切れになっている度合い(0 = 1つの塊、1 に近いほど細切れ)
        double ExternalFragmentation() const noexcept
        {
            if (freeTotal == 0) { return 0.0; }
            return 1.0 - static_cast<double>(freeLargest) / static_cast<double>(freeTotal);
        }
    };
}
```

### 外部断片化の式について

```
外部断片化 = 1 - (最大の連続空き / 空きの合計)
```

意味を確認します。

| 状況 | 最大 / 合計 | 指標 |
|---|---|---|
| 空きが1つの塊 | 1.0 | **0.0**(断片化なし) |
| 10 MB の空きが 5 MB × 2 | 0.5 | 0.5 |
| 10 MB の空きが 1 MB × 10 | 0.1 | **0.9**(ひどい) |

広く使われている式ですが、**万能ではありません**。

「4 MB を確保したい」という具体的な要求に対しては、指標が 0.9 でも最大空きが 4 MB あれば成功します。逆に指標が 0.3 でも、最大空きが 3 MB なら失敗します。

**実務では「いま確保できる最大サイズ」のほうが直接的に役立つことが多い** ということは、覚えておいてください。この指標は、傾向を追うためのものです。

### `Bump` での実装

```cpp
    FragmentationStats GetFragmentation() const noexcept
    {
        FragmentationStats f;
        f.capacity       = Capacity();
        f.used           = Used();
        f.internalWaste  = Padding();
        f.freeTotal      = Remaining();
        f.freeLargest    = Remaining();                 // 空きは常に1つ
        f.freeBlockCount = (Remaining() > 0) ? 1 : 0;
        return f;
    }
```

**外部断片化は、常に 0 です。**

バンプアロケーターは前へ進むだけなので、空きは必ず末尾の1つの塊になります。**構造的に断片化しません。**

```
=== 断片化 ===
  使用率          : 43.9%
  内部断片化      : 0.06%
  外部断片化      : 0.00%   ← 常にゼロ
  空きブロック数  : 1
  最大の連続空き  : 8.98 MB
```

### なぜ、まだ 0 のものを測るのか

第4章でベンチマークの道具を先に作ったのと、同じ理由です。

第20章から、個別解放を導入します。その瞬間、この数字は 0 でなくなります。**変化を捉えるには、変化する前の値を持っていなければなりません。**

そして、この指標こそが第3部を通しての **評価軸** になります。

| 章 | アロケーター | 期待される外部断片化 |
|---|---|---|
| 第19章 | Bump | 0.00 |
| 第23章 | フリーリスト(合体なし) | **高い** |
| 第24章 | フリーリスト(合体あり) | 下がる |
| 第25章 | サイズ別ビン | ? |
| 第26章 | バディ | 低いが内部断片化と引き換え |
| 第27章 | TLSF | ? |

「どれが最強か」ではなく、**どこにコストを移したか** を読み解くための表です。

---

## 19.8 第3部への橋渡し

この章で作った絵は、いまは退屈です。ただの帯です。

**第23章で、この絵は劇的に変わります。**

個別解放を導入すると、帯の途中に穴が空きはじめます。確保と解放を繰り返すうちに、穴は増え、細かくなります。

```
第19章(バンプ):
████████████████████████░░░░░░░░

第23章(フリーリスト、合体なし):
███░█░███░██░░█░██░█░░███░█░░░░░
     ↑ 穴だらけ

第24章(合体あり):
███████░░░████████░░░░░░░░░░░░░░
        ↑ 穴が結合された
```

**この絵の変化を見ることが、第3部の学習そのものです。**

`Reset()` という魔法を捨てて、個別解放という難問に踏み込む。その代償と、それを和らげる工夫の数々。すべてが、この絵と、`ExternalFragmentation()` の数値に現れます。

---

## 19.9 演習

**演習19-1** `bytesPerPixel` を 64 バイトに変えて、細かい構造を見てください。第17章のガードバイトを有効にすると、絵はどう変わりますか。

**演習19-2** パディング領域だけを別の色で塗ってください。第16章で `0xAB` を使ったのと同じ発想です。どこで無駄が出ているか見えますか。

**演習19-3** 画像に目盛りを描いてください。1 MB ごとに線を引くと、読みやすくなりますか。

**演習19-4** 毎フレーム画像を出力し、連番のファイルに保存してください。動画にすると、メモリの動きがどう見えますか。

**演習19-5** `ExternalFragmentation()` が 0.9 でも問題ない状況と、0.3 でも致命的な状況を、具体的に作ってください。

**演習19-6** 「いま確保できる最大サイズ」を返す関数を `Bump` に追加してください。`Remaining()` との違いは何ですか(ヒント:アラインメント)。

**演習19-7** PNG ではなく BMP を出力する関数を書いてください。何行になりますか。PNG との差はどこにありますか。

**演習19-8** 無圧縮 deflate ではなく、同じバイトの繰り返しを縮める簡単な圧縮を実装してください。ファイルサイズはどれくらい減りますか。

---

## 章末チェックリスト

- [ ] 1ピクセルあたりのバイト数と画像サイズの関係を理解した
- [ ] **タグの境界だけを記録すればよい** 理由を説明できる
- [ ] `Rewind()` の後始末にマーカーのフィールドが不要な理由を説明できる
- [ ] PNG を外部ライブラリなしで出力した
- [ ] ロード後・`Reset()` 後・`Rewind()` 後の絵を比較した
- [ ] 外部断片化の式とその限界を説明できる
- [ ] **バンプアロケーターの外部断片化が常に 0 である** 理由を説明できる

---

## 第2部のまとめ

第14章から第19章まで、`Bump` に「見る目」を与えてきました。

| 章 | 機能 | 見えるようになったもの |
|---|---|---|
| 14 | `source_location` | どこから確保されたか |
| 15 | 統計・タグ | 何がどれだけ使っているか |
| 16 | 塗りつぶし | 間違ったメモリを読んでいないか |
| 17 | ガードバイト | はみ出して書いていないか |
| 18 | スタックトレース | どの経路で呼ばれたか |
| 19 | 可視化・断片化指標 | どこに配置されているか |

すべて Debug 構成で有効、Release では消える機能です。コストは大きいものもありますが、**開発中にバグを見つける価値** と引き換えです。

そして重要なのは、**これらの道具が第3部で本当に必要になる** ということです。

第2部までのアロケーターは、単純すぎて壊しようがありませんでした。`offset_` を進めるだけ。バグの入り込む余地がほとんどない。

第3部で作るものは違います。フリーリストの連結、ブロックの分割と合体、サイズ別のビン、ビットツリー。**どれも、間違えれば静かにメモリを壊します。** そのとき、この6章で作った道具が、あなたを救います。

---

## 次章の予告

第3部が始まります。

第20章では、**19章ぶん先送りにしてきた問題** に、ついて向き合います。

> 個別に解放できるようにすると、何が起きるのか。

穴ができる。穴を記録する必要がある。穴を探す必要がある。隣り合った穴を結合したくなる。そして、合計では足りているのに確保できなくなる。

まずコードを書く前に、**紙の上で** 確保と解放をシミュレートします。first fit と best fit を手で動かし、どちらがどう失敗するかを自分の目で確かめます。

そして、1960年代に Knuth たちが取り組んだ問題が、なぜ今も「万能の答えがない」と言われるのかを理解します。

---

> **コラム:メモリを絵にしてきた人々**
>
> メモリの状態を絵にする試みには、長い歴史があります。
>
> ---
>
> ある年齢以上の Windows 使用者なら、**デフラグの画面** を覚えているでしょう。ディスクの断片化状況を四角いブロックで表示し、最適化が進むにつれてブロックが移動していく——あの画面です。何時間も眺めていた人もいるはずです。
>
> あれはディスクの話ですが、**問題の構造はメモリの断片化とまったく同じ** です。細切れになった空きを、連続した塊にまとめ直す。第46章で扱うコンパクションは、メモリ版のデフラグにほかなりません。
>
> あの画面が広く愛されたのは、**抽象的な概念が、動く絵として理解できたから** でしょう。「断片化」という言葉を知らなくても、絵を見れば何が問題か分かります。
>
> ---
>
> 開発者向けのツールにも、可視化の伝統があります。
>
> **VMMap**(Sysinternals)は、プロセスの仮想アドレス空間を色分けして表示します。どこがコミット済みで、どこが予約だけか、どのモジュールが何を使っているか。第7章で少し触れましたが、第29章で `VirtualAlloc` を扱うときに本格的に使います。
>
> **Massif**(Valgrind)や **heaptrack** は、時間軸に沿ったメモリ使用量のグラフを描きます。「どの時点で何が確保されたか」を追えます。
>
> **Unreal Engine の `memreport`** コマンドは、実行中のメモリ内訳をダンプします。第15章で作ったレポートの本格版です。
>
> ---
>
> ゲーム開発の現場では、もっと素朴な絵もよく使われます。**メモリマップ図** です。
>
> ```
>   0x00000000 ┌─────────────────┐
>              │  システム予約     │
>   0x10000000 ├─────────────────┤
>              │  実行コード       │
>   0x14000000 ├─────────────────┤
>              │  永続データ       │
>   0x20000000 ├─────────────────┤
>              │  レベルアリーナ    │
>   0x60000000 ├─────────────────┤
>              │  フレームアリーナ  │
>   0x64000000 └─────────────────┘
> ```
>
> ホワイトボードや設計書に、こういう図が描かれます。第44章と第49章で扱う「メモリ予算」は、しばしばこの形で議論されます。
>
> **絵は、議論の共通言語になります。** 「テクスチャの枠をここまで広げたい」という要求も、図の上で線を動かせば、何を削る必要があるか全員に伝わります。
>
> ---
>
> この章で作ったものは、それらに比べれば玩具です。しかし **自分のアロケーターの、自分の知りたい情報を、自分の見たい形で** 描けるという点で、既製のツールにはない価値があります。
>
> 第3部で断片化と格闘するとき、この玩具が意外な発見をもたらすはずです。
