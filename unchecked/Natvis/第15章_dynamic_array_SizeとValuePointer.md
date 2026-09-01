# 第15章 dynamic_array<T> ―― Size / ValuePointer / LowerBound

第1章の冒頭で「読めない型」の例として挙げた `dynamic_array` を、いよいよ実装して可視化します。

`std::vector` と同じ三ポインタ表現を採り、要素数も容量も**実行時にしか分からない**という、コンテナの標準的な形です。

## 15.1 型を作る

`vizkit\include\vizkit\seq\dynamic_array.hpp`

```cpp
#pragma once

#include <cstddef>
#include <new>
#include <utility>

namespace vizkit {

// std::vector 相当の最小実装。三ポインタ表現。
//   _begin  : 確保領域の先頭
//   _end    : 最後の要素の次（size = _end - _begin）
//   _cap    : 確保領域の終端（capacity = _cap - _begin）
template <class T>
class dynamic_array {
public:
    dynamic_array() = default;

    ~dynamic_array() {
        clear();
        ::operator delete(static_cast<void*>(_begin));
    }

    dynamic_array(const dynamic_array&) = delete;
    dynamic_array& operator=(const dynamic_array&) = delete;

    void push_back(const T& v) {
        if (_end == _cap) grow();
        new (_end) T(v);
        ++_end;
    }

    void clear() noexcept {
        while (_end != _begin) { --_end; _end->~T(); }
    }

    [[nodiscard]] std::size_t size()     const noexcept { return static_cast<std::size_t>(_end - _begin); }
    [[nodiscard]] std::size_t capacity() const noexcept { return static_cast<std::size_t>(_cap - _begin); }

private:
    void grow() {
        const std::size_t old_cap = capacity();
        const std::size_t new_cap = old_cap == 0 ? 4 : old_cap * 2;
        T* p = static_cast<T*>(::operator new(sizeof(T) * new_cap));
        const std::size_t n = size();
        for (std::size_t i = 0; i < n; ++i) {
            new (p + i) T(std::move(_begin[i]));
            _begin[i].~T();
        }
        ::operator delete(static_cast<void*>(_begin));
        _begin = p;
        _end   = p + n;
        _cap   = p + new_cap;
    }

    T* _begin = nullptr;
    T* _end   = nullptr;
    T* _cap   = nullptr;
};

} // namespace vizkit
```

`main.cpp`：

```cpp
#include <vizkit/seq/dynamic_array.hpp>
// ...
    vizkit::dynamic_array<int> v;
    v.push_back(10);
    v.push_back(20);
    v.push_back(30);

    vizkit::dynamic_array<vizkit::point2> path;
    path.push_back(vizkit::point2{ 0.0, 0.0 });
    path.push_back(vizkit::point2{ 3.0, 4.0 });

    vizkit::dynamic_array<int> untouched;   // 一度も push_back していない
```

第1章で見た素の表示を、あらためて確認しておいてください。

```
▷ v            {_begin=0x000001f4a2c31240 {10} _end=... _cap=... }   vizkit::dynamic_array<int>
```

要素が3つあることも、20 と 30 が入っていることも見えません。

## 15.2 Size をポインタ差で書く

```xml
<Type Name="vizkit::dynamic_array&lt;*&gt;">
  <DisplayString>{{ size={_end - _begin} }}</DisplayString>
  <Expand>
    <ArrayItems>
      <Size>_end - _begin</Size>
      <ValuePointer>_begin</ValuePointer>
    </ArrayItems>
  </Expand>
</Type>
```

```
▷ v             { size=3 }              vizkit::dynamic_array<int>
     [0]        10
     [1]        20
     [2]        30
   ▷ [Raw View] {_begin=... _end=... _cap=... }
```

読めるようになりました。第1章 1.3節で先出しした表示に一歩近づいています。

前章と違うのは、`Size` が**実行時の値から計算される**点だけです。`ArrayItems` の書き方そのものは変わりません。

### size() は呼べないが、Intrinsic は書ける

`dynamic_array` には `size()` メンバー関数がありますが、Natvis から呼ぶことはできません（第5章）。しかし `Intrinsic` で同名の「Natvis 内関数」を定義できます。

```xml
<Type Name="vizkit::dynamic_array&lt;*&gt;">
  <Intrinsic Name="size"     Expression="(int)(_end - _begin)" />
  <Intrinsic Name="capacity" Expression="(int)(_cap - _begin)" />
  <DisplayString>{{ size={size()} }}</DisplayString>
  <Expand>
    <ArrayItems>
      <Size>size()</Size>
      <ValuePointer>_begin</ValuePointer>
    </ArrayItems>
  </Expand>
</Type>
```

C++ 側の `size()` とは無関係な、Natvis の中だけの定義です。しかし**同じ名前を付けておく**と、Natvis を読む人（＝将来の自分）にとって意味が明らかになります。`STL.natvis` も同じ命名をしています。

### (int) キャストについて

`_end - _begin` の型は `ptrdiff_t`（64ビット環境では `__int64`）です。そのままでも動きますが、`(int)` にキャストしておくと表示が素直になります。

- `[size]` の行に `3` と出る（`__int64` のまま何かの拍子に `3L` のように出ることを避けられる）
- 他の整数と比較する式で、符号や幅の警告じみた挙動に悩まされない

要素数が 20 億を超えるコンテナを扱う予定がなければ、`(int)` で問題ありません。`STL.natvis` も多くの箇所で同様のキャストを入れています。

## 15.3 [size] と [capacity]

`fixed_array` と違い、**要素数が型名から読めません**。`[size]` の `Item` が必要です。容量も、再確保のタイミングを調べるときに欲しくなります。

```xml
<Expand>
  <Item Name="[size]"     ExcludeView="simple">size()</Item>
  <Item Name="[capacity]" ExcludeView="simple">capacity()</Item>
  <ArrayItems>
    <Size>size()</Size>
    <ValuePointer>_begin</ValuePointer>
  </ArrayItems>
</Expand>
```

```
▷ v             { size=3 }
     [size]     3
     [capacity] 4
     [0]        10
     [1]        20
     [2]        30
```

`ExcludeView="simple"` を付けている理由は第13章のとおりです。`dynamic_array` を 200 個並べたときに、`[size]` と `[capacity]` が邪魔になる場面があります。

これは `STL.natvis` の `std::vector` とまったく同じ設計です。

```xml
<Item Name="[size]" ExcludeView="simple">_Mylast - _Myfirst</Item>
<Item Name="[capacity]" ExcludeView="simple">_Myend - _Myfirst</Item>
```

## 15.4 空とnullをガードする

`untouched`（一度も `push_back` していないもの）は、3つのポインタがすべて `nullptr` です。

```
▷ untouched     { size=0 }
     [size]     0
     [capacity] 0
```

`nullptr - nullptr` は 0 なので、実は破綻しません。しかし表示としては素っ気ないので、状態を明示しておきます。

```xml
<DisplayString Condition="_begin == nullptr">[unallocated]</DisplayString>
<DisplayString Condition="size() == 0">[empty] (capacity={capacity()})</DisplayString>
<DisplayString>{{ size={size()} }}</DisplayString>
```

「まだ確保していない」と「確保済みだが空」を区別できるようになりました。メモリ確保のタイミングを追っているときには、この区別が効きます。

さらに、破損検出も入れておきます（第9章 9.5節の考え方の再利用）。

```xml
<DisplayString Condition="_end &lt; _begin || _cap &lt; _end">[corrupt]</DisplayString>
```

`_begin <= _end <= _cap` は `dynamic_array` の不変条件です。**不変条件を Natvis に書いておくと、デバッガが不変条件チェッカーになります。** ムーブ後の抜け殻や、解放済みオブジェクトを踏んだときに、一目で分かります。

条件の順序に注意してください。`[corrupt]` は最も限定的な条件なので、**最上位**に置きます（第7章 7.3節）。

## 15.5 LowerBound ―― 添字を 0 以外から始める

`ArrayItems` の子要素には `<LowerBound>` があります。既定値は 0 で、表示される添字の開始値を変えられます。

```xml
<ArrayItems>
  <Size>size()</Size>
  <ValuePointer>_begin</ValuePointer>
  <LowerBound>1</LowerBound>
</ArrayItems>
```

```
     [1]        10
     [2]        20
     [3]        30
```

使いどころは限られますが、次のような場面で効きます。

- Fortran 由来のコードや、数学的な記法に合わせた 1 始まり配列
- 「先頭にセンチネルがあり、実データは 1 から」という設計
- リングバッファで論理インデックスを表示したい場合（第17章では別の方法を採ります）

注意点として、`LowerBound` は**表示上の添字を変えるだけ**です。`ValuePointer` が指す位置は変わりません。「`[1]` と表示されている要素は `_begin[0]`」という対応になります。混乱を招きやすいので、実際に 1 始まりの型でない限り使わないほうが無難です。

## 15.6 未使用領域を覗く ―― detail ビューの活用

コンテナのバグを追っているとき、「`_end` から `_cap` までの領域に何が残っているか」を見たくなることがあります。ムーブ後の状態や、デストラクタの呼び忘れを調べる場面です。

第13章のビューを使って、詳細ビューにだけ出します。

```xml
<Synthetic Name="[raw storage]" IncludeView="detail" Condition="_begin != nullptr">
  <DisplayString>{capacity()} slots ({size()} used)</DisplayString>
  <Expand>
    <ArrayItems>
      <Size>capacity()</Size>
      <ValuePointer>_begin</ValuePointer>
    </ArrayItems>
  </Expand>
</Synthetic>
```

```
v,view(detail)
  ├ [size]         3
  ├ [capacity]     4
  ├ [raw storage]  4 slots (3 used)
  │    [0]         10
  │    [1]         20
  │    [2]         30
  │    [3]         -842150451      ← 未初期化領域
  ├ [0]            10
  ├ [1]            20
  └ [2]            30
```

`[3]` に `0xCDCDCDCD`（MSVC のデバッグヒープが未初期化ヒープメモリを埋める値）が見えています。**未初期化領域を「意図的に」見せる**という、Natvis の使い方の一例です。

既定のビューには絶対に出さないでください。未構築のオブジェクトを表示させると、要素型の Natvis が不正なメモリを読んでデバッガが不安定になることがあります。`IncludeView="detail"` に閉じ込めるのが安全です。

## 15.7 STL の vector を読む

ここまで書いてきたものと、MSVC の `std::vector` の定義を比べてみてください。ウォッチウィンドウで `std::vector<int>` を作り、`[Raw View]` を開けば、こういう構造が見えます。

```
_Mypair
 └ _Myval2
     ├ _Myfirst    ← 本書の _begin
     ├ _Mylast     ← 本書の _end
     └ _Myend      ← 本書の _cap
```

`_Mypair` は「アロケータと値を空基底最適化で詰め込むための入れ物」です。`STL.natvis` はこれを次のように書いています。

```xml
<Item Name="[size]" ExcludeView="simple">_Mylast - _Myfirst</Item>
<Item Name="[capacity]" ExcludeView="simple">_Myend - _Myfirst</Item>
```

`_Mypair._Myval2.` という前置きが省かれているのは、`<Type Name="std::vector&lt;*&gt;">` の直下に `<Intrinsic>` や `<ExpandedItem>` を置いて文脈を移しているためです。第27章でこのファイルを本格的に読みます。

**構造としては、本章で書いたものと同じ**だと分かれば十分です。三ポインタ表現に対する `ArrayItems` の当て方は、実装が誰であっても変わりません。

## 15.8 現時点の dynamic_array の Natvis

```xml
  <Type Name="vizkit::dynamic_array&lt;*&gt;">
    <Intrinsic Name="size"     Expression="(int)(_end - _begin)" />
    <Intrinsic Name="capacity" Expression="(int)(_cap - _begin)" />

    <DisplayString Condition="_end &lt; _begin || _cap &lt; _end">[corrupt]</DisplayString>
    <DisplayString Condition="_begin == nullptr">[unallocated]</DisplayString>
    <DisplayString Condition="size() == 0">[empty] (capacity={capacity()})</DisplayString>
    <DisplayString>{{ size={size()} }}</DisplayString>

    <Expand>
      <Item Name="[size]"     ExcludeView="simple">size()</Item>
      <Item Name="[capacity]" ExcludeView="simple">capacity()</Item>
      <Synthetic Name="[raw storage]" IncludeView="detail" Condition="_begin != nullptr">
        <DisplayString>{capacity()} slots ({size()} used)</DisplayString>
        <Expand>
          <ArrayItems>
            <Size>capacity()</Size>
            <ValuePointer>_begin</ValuePointer>
          </ArrayItems>
        </Expand>
      </Synthetic>
      <ArrayItems>
        <Size>size()</Size>
        <ValuePointer>_begin</ValuePointer>
      </ArrayItems>
    </Expand>
  </Type>
```

## 15.9 この章のまとめ

- 三ポインタ表現では `Size` を **ポインタ差**で書く。`ArrayItems` の書き方自体は固定長のときと同じ。
- C++ の `size()` は呼べないが、**同名の `Intrinsic` を定義**すると Natvis が読みやすくなる。`STL.natvis` も同じ命名。
- ポインタ差は `(int)` にキャストしておくと表示が素直になる。
- 要素数が型名から読めないので、**`[size]` / `[capacity]` の `Item` は必要**。`ExcludeView="simple"` を付ける。
- 「未確保」「空」「破損」を `Condition` で区別する。**不変条件（`_begin <= _end <= _cap`）を書けば、Natvis が不変条件チェッカーになる**。
- `<LowerBound>` は表示上の添字だけを変える。使いどころは限定的。
- 未初期化領域を見せる `[raw storage]` は `IncludeView="detail"` に閉じ込める。既定ビューに出してはいけない。

## 15.10 演習

1. `v` に `push_back` を5回して、`[capacity]` が 4 から 8 に変わる瞬間をステップ実行で観察してください。Natvis なしでこれを追う手間を想像してみてください。
2. デバッガのウォッチウィンドウで `v._end` の値を直接書き換え、`_begin` より小さくして `[corrupt]` が出ることを確認してください。
3. `v,view(detail)` と `v,view(simple)` を並べて、3種類のビューの差を確認してください。
4. `<LowerBound>1</LowerBound>` を追加し、`[1]` と表示される要素が `_begin[0]` であることをメモリウィンドウで確認してください。
5. `dynamic_array<vizkit::dynamic_array<int>>`（入れ子）を作り、2階層のコンテナが正しく展開されることを確認してください。
