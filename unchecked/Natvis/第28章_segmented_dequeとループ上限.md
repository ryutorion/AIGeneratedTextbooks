# 第28章 segmented deque とループ上限

第6部の締めくくりです。二段構造のコンテナを題材に、**専用要素と `CustomListItems` の境界**をもう一度確認し、あわせて**大量要素と打ち切り**の扱いを整理します。

## 28.1 型を作る

`vizkit\include\vizkit\seq\deque.hpp`

```cpp
#pragma once

#include <cstddef>
#include <new>

namespace vizkit {

// ブロック（固定長チャンク）の配列で構成される両端キュー。
//   _blocks[i] が 1 ブロック（要素 BlockSize 個）を指す
//   論理インデックス i の要素は
//     _blocks[(_offset + i) / BlockSize][(_offset + i) % BlockSize]
template <class T, std::size_t BlockSize = 8>
class deque {
public:
    static constexpr std::size_t block_size = BlockSize;

    deque() = default;
    ~deque();

    deque(const deque&) = delete;
    deque& operator=(const deque&) = delete;

    void push_back(const T& v);           // 実装省略
    void push_front(const T& v);          // 実装省略

    [[nodiscard]] std::size_t size() const noexcept { return _size; }

    [[nodiscard]] T& operator[](std::size_t i) noexcept {
        const std::size_t k = _offset + i;
        return _blocks[k / BlockSize][k % BlockSize];
    }

private:
    T**         _blocks      = nullptr;   // ブロックへのポインタの配列
    std::size_t _block_count = 0;
    std::size_t _offset      = 0;         // 先頭要素の論理位置
    std::size_t _size        = 0;
};

} // namespace vizkit
```

`main.cpp`：

```cpp
#include <vizkit/seq/deque.hpp>
// ...
    vizkit::deque<int, 4> dq;
    for (int i = 1; i <= 10; ++i) dq.push_back(i * 10);
    dq.push_front(5);
    // 論理: 5, 10, 20, ..., 100
```

素の表示は、ポインタの配列なので二重に読めません。

```
▷ dq         {_blocks=0x000001f4a2c33c00 {0x000001f4a2c33c40} _block_count=4 _offset=3 _size=11 }
```

## 28.2 まず IndexListItems で書けるか考える

第23章 23.1節の判断基準に戻ります。**添字の変換で位置が求まるか。**

`operator[]` の実装を見てください。

```cpp
const std::size_t k = _offset + i;
return _blocks[k / BlockSize][k % BlockSize];
```

求まります。`$i` から純粋な計算だけで要素にたどり着けます。**`IndexListItems` で書けます。**

```xml
<Type Name="vizkit::deque&lt;*,*&gt;">
  <DisplayString>{{ size={_size} }}</DisplayString>
  <Expand>
    <Item Name="[size]" ExcludeView="simple">_size</Item>
    <IndexListItems>
      <Size>_size</Size>
      <ValueNode>_blocks[($i + _offset) / $T2][($i + _offset) % $T2]</ValueNode>
    </IndexListItems>
  </Expand>
</Type>
```

```
▷ dq             { size=11 }
     [size]      11
     [0]         5
     [1]         10
     [2]         20
     ...
     [10]        100
```

C++ の `operator[]` の式を、そのまま `ValueNode` に写しただけです。第17章 17.3節でリングバッファに対してやったことと同じ手順です。

### std::deque も同じ

第27章 27.7節で見た `STL.natvis` の定義と比べてください。

```xml
<ValueNode>
  _Mypair._Myval2._Map[(($i + _Mypair._Myval2._Myoff) / _EEN_DS) % _Mypair._Myval2._Mapsize]
                      [($i + _Mypair._Myval2._Myoff) % _EEN_DS]
</ValueNode>
```

`% _Mapsize` が余分に付いているのは、`std::deque` のブロック配列が循環バッファになっているためです。それ以外は同じ形です。

**MSVC の標準ライブラリも、二段構造に `IndexListItems` を使っている。** 「二段だから `CustomListItems`」ではありません。

## 28.3 CustomListItems が必要になる条件

では、いつ `CustomListItems` が必要になるのでしょうか。

**ブロックのサイズが一定でない場合**です。

```cpp
// ブロックごとに長さが違う実装
struct block { T* data; std::size_t count; };
block*      _blocks;
std::size_t _block_count;
```

この場合、論理インデックス `$i` からブロック番号を求めるには、先頭から `count` を積算していくしかありません。純粋な計算では求まらないので、`IndexListItems` は使えません。

```xml
<CustomListItems>
  <Variable Name="b"     InitialValue="0" />
  <Variable Name="j"     InitialValue="0" />
  <Variable Name="guard" InitialValue="0" />
  <Size>_size</Size>

  <Loop Condition="b &lt; (int)_block_count">
    <Break Condition="guard &gt; 1000000" />
    <Exec>j = 0</Exec>
    <Loop Condition="j &lt; (int)_blocks[b].count">
      <Item>_blocks[b].data[j]</Item>
      <Exec>j++</Exec>
      <Exec>guard++</Exec>
    </Loop>
    <Exec>b++</Exec>
  </Loop>
</CustomListItems>
```

第26章の二重ループとまったく同じ形です。**「外側の配列を回り、内側の可変長を辿る」という構造は、チェイン法のハッシュマップと共通**です。

第6部を通じての結論を、ここで確定させます。

> **`CustomListItems` が必要なのは、走査に「可変」が入るときだけ。**
> 飛ばす要素の数が可変（第25章）、チェーンの長さが可変（第26章）、ブロックの長さが可変（本章）。
> 可変がなければ、添字の計算で書ける。

## 28.4 大量要素と打ち切り

第6部で扱ってきたコンテナは、どれも数万〜数百万要素になりえます。表示コストの問題が現実になります。

### MaxItemsPerView

`CustomListItems` の `MaxItemsPerView` 属性は、既定 5000、範囲 1〜50000 でした（第23章 23.8節）。

```xml
<CustomListItems MaxItemsPerView="1000">
```

これを超えると、末尾に「続きを見る」ノードが作られます。**要素が消えるわけではありません。** 一度に描画する量を制限しているだけです。

### 自分で打ち切る

`MaxItemsPerView` はデバッガ側の描画制御です。**ループそのものを止めたい**場合は、自分で `Break` を書きます。

```xml
<CustomListItems>
  <Variable Name="i"     InitialValue="0" />
  <Variable Name="shown" InitialValue="0" />
  <Size>_size &lt; 200 ? (int)_size : 200</Size>

  <Loop Condition="i &lt; (int)_size">
    <Break Condition="shown &gt;= 200" />
    <Item Name="[{i}]">_blocks[(i + _offset) / $T2][(i + _offset) % $T2]</Item>
    <Exec>shown++</Exec>
    <Exec>i++</Exec>
  </Loop>
</CustomListItems>
```

**打ち切ったことを表示に残す**のが親切です。何も言わずに 200 件で止まると、要素数が 200 だと誤解されます。

```xml
<Synthetic Name="[...]" Condition="_size &gt; 200">
  <DisplayString>残り {(int)_size - 200} 件は表示されていません（,view(all) で全件）</DisplayString>
</Synthetic>
```

第11章 11.2節の「説明行」の実用例です。**Natvis が嘘をつかないようにする**ための1行です。

そして、全件を見るためのビューを別に用意します。

```xml
<IndexListItems IncludeView="all">
  <Size>_size</Size>
  <ValueNode>_blocks[($i + _offset) / $T2][($i + _offset) % $T2]</ValueNode>
</IndexListItems>
```

`dq,view(all)` で全件。既定は 200 件で打ち切り、という構成です。

### どこにコストがかかるのか

第14章 14.8節・第17章 17.7節で触れた性能の話を、ここで整理します。

| 列挙方法 | 1要素あたりのコスト |
|---|---|
| `ArrayItems` | アドレス計算のみ（式の評価なし） |
| `IndexListItems` | `ValueNode` の式を1回評価 |
| `LinkedListItems` / `TreeItems` | `NextPointer` / `ValueNode` を1回ずつ評価 |
| `CustomListItems` | ループ中のすべての `Exec` と `Item` を評価 |

`CustomListItems` は、**要素を1件出すたびに複数の式を評価します**。しかもデバッガの式エバリュエーターは、コンパイルされたコードではなくインタプリタです。1要素あたりのコストは `ArrayItems` の数十倍になりえます。

デバッガは画面に見えている範囲だけを評価する仕組み（仮想化）を持っていますが、**`CustomListItems` は先頭から順に辿るしかない**ため、100 万件目を見るには 100 万回のループが必要です。`ArrayItems` なら即座にジャンプできます。

したがって実務的な指針はこうなります。

> **要素数が数千を超えうるコンテナで `CustomListItems` を既定にしない。**
> 既定は専用要素（あるいは打ち切り付き）、キー表示や統計は別ビューに置く。

第24章 24.6節で `flat_map` に `fast` ビューを用意したのは、この判断です。

## 28.5 Skip 要素

`CustomListItems` には `<Skip>` という要素もあります。ユーザーがウォッチウィンドウをスクロールしたときに、**要素を高速に読み飛ばす**ためのロジックを書きます。

前節で述べた「先頭から順に辿るしかない」という制約を、部分的に緩和するものです。たとえばブロック単位で飛ばせる構造なら、`Skip` にその処理を書くことで、100 万件目へのジャンプが速くなります。

ただし利用例は少なく、環境依存の面もあります。**まずは打ち切りとビュー分割で対処し、それでも足りない場合の選択肢**と考えてください。

## 28.6 第6部のまとめ

第23章から第28章までで、`CustomListItems` を扱いました。

```
CustomListItems  MaxItemsPerView="5000"
 ├─ Variable      ループ変数（外で宣言、型は InitialValue から推論）
 ├─ Size          出力件数（ループ回数ではない）
 ├─ Exec          代入・算術・論理。関数呼び出し不可
 ├─ Loop          Condition 付き、または Break で脱出。入れ子可
 │   ├─ If / Elseif / Else
 │   ├─ Break Condition="..."
 │   └─ Item Name="[{式}]">式</Item>     ← キーを名前に出せる唯一の手段
 ├─ Item          ループの外にも置ける（統計値）
 └─ Skip          スクロール時の高速スキップ（利用例は少ない）
```

第6部で立てた原則を並べます。

1. **`CustomListItems` が必要なのは、走査に「可変」が入るときだけ。**（第28章 28.3節）
2. **`<Size>` は出力件数であって、ループの回数ではない。**（第25章 25.3節）
3. **`Item` は `If` で囲む。`Condition` 属性に頼らない。**（第24章 24.5節）
4. **`Variable` の `InitialValue` は型推論のためのダミーでよい。**（第26章 26.3節）
5. **カウンタによる `Break` を必ず入れる。二重ループなら内側にこそ。**（第23章 23.7節、第26章 26.6節）
6. **ループを回して `Item` を1件出すパターンで、診断値を作れる。**（第25章、第26章）
7. **要素数が多いコンテナで既定にしない。ビューで使い分ける。**（第24章 24.6節、第28章 28.4節）

そして第6部全体を貫く発見が、第27章にありました。**データ構造の設計そのものが、Natvis の書きやすさを決める。** `std::unordered_map` の Natvis が1行で済むのは、Natvis が優れているからではなく、実装が優れているからです。

次の第7部では、ここまで場当たり的に使ってきたワイルドカードと `$T1` を正面から扱い、テンプレートクラスへの対応を体系化します。

## 28.7 この章のまとめ

- 二段構造でも、**添字の計算で位置が求まるなら `IndexListItems` で書ける**。`std::deque` も同じ。
- `CustomListItems` が要るのは、**ブロック長やチェーン長が可変**のとき。構造は第26章の二重ループと同一。
- `MaxItemsPerView` は描画量の制御（既定 5000、範囲 1〜50000）。**要素が消えるわけではない。**
- ループ自体を止めたいなら `Break` で自分で打ち切る。**打ち切ったことを `Synthetic` で明示する。** Natvis が嘘をつかないようにする。
- 全件を見るための `IncludeView="all"` を別に用意する。
- `CustomListItems` は要素ごとに複数の式を評価し、**先頭から順に辿るしかない**ため、大量要素では極端に遅い。専用要素は数十倍速い。
- **要素数が数千を超えうるコンテナでは、`CustomListItems` を既定にしない。**
- `<Skip>` はスクロール時の高速スキップ用。まずは打ち切りとビュー分割で対処する。

## 28.8 演習

1. `deque<int, 4>` に 10000 件入れ、`IndexListItems` 版と `CustomListItems` 版で末尾までスクロールする速度を比べてください。
2. `<Size>` を `_size` のままにして `<Break Condition="shown &gt;= 200" />` を入れると、どんな不整合が起きるか観察してください。`Size` と実際の出力件数が食い違う場合の挙動です。
3. ブロック長が可変な `deque` を実装し、28.3節の `CustomListItems` 版が正しく動くことを確認してください。
4. `MaxItemsPerView="10"` を指定して 100 件のコンテナを開き、末尾のノードを展開して続きが見られることを確認してください。
5. `_offset` をデバッガから壊れた値（`_block_count * BlockSize` を超える値）に書き換え、`IndexListItems` 版がどうなるか確認してください。そのうえで、範囲チェックの `Condition` をどこに入れるべきか考えてください。
