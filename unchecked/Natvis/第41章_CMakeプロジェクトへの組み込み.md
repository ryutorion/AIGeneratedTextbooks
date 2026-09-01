# 第41章 CMake プロジェクトへの組み込み

MSBuild ではなく CMake でビルドしている場合の扱いです。

CMake の Natvis サポートは**ジェネレータによって挙動が違う**ため、そこを理解しないと「Visual Studio では効くのに Ninja ビルドでは効かない」という状況に陥ります。

## 41.1 Visual Studio ジェネレータ ―― target_sources で足りる

CMake 3.7 以降、Visual Studio ジェネレータは `.natvis` を認識し、生成する `.vcxproj` に**正しい項目の種類で配置します**。

```cmake
add_library(vizkit STATIC
    src/vizkit.cpp
)

target_sources(vizkit PRIVATE
    vizkit.natvis
)

target_include_directories(vizkit PUBLIC include)
```

これだけで、第40章 40.1節の `<Natvis Include="vizkit.natvis" />` が生成されます。ソリューションに含まれるので、デバッグ時に読み込まれます（第39章の優先順位2）。

**Visual Studio ジェネレータを使っている限り、これで十分**です。

## 41.2 Ninja ジェネレータ ―― 無視される

問題はこちらです。

> CMake は Visual Studio に対してのみ natvis ファイルを識別させる方法を知っています。他のジェネレータはこのファイルを無視します。

`target_sources` に `.natvis` を書いても、Ninja ジェネレータは**何もしません**。エラーも警告も出ません。

Visual Studio の「フォルダーを開く」機能（CMake 統合）は既定で Ninja を使うことが多いので、**気づかないうちにこの状況になっている**ことがあります。

### 解決 ―― /NATVIS リンカーフラグを明示する

第40章 40.2節のリンカーオプションを、CMake から直接渡します。

```cmake
target_link_options(vizkit PRIVATE
    /NATVIS:${CMAKE_CURRENT_SOURCE_DIR}/vizkit.natvis
)
```

ただし、`vizkit` が**静的ライブラリの場合、これは効きません**。第40章 40.3節で述べたとおり、`/NATVIS` はリンカーのオプションであり、静的ライブラリの作成には `lib.exe` が使われるからです。

## 41.3 INTERFACE で利用者に伝播させる

静的ライブラリやヘッダーオンリーライブラリでは、**リンクする側にフラグを伝播させる**必要があります。`INTERFACE` を使います。

```cmake
target_link_options(vizkit INTERFACE
    /NATVIS:$<TARGET_PROPERTY:vizkit,SOURCE_DIR>/vizkit.natvis
)
```

これで、`target_link_libraries(myapp PRIVATE vizkit)` と書いた側の実行可能ファイルに `/NATVIS` が渡ります。第40章 40.4節の `.props` と同じ効果を、CMake で実現した形です。

`$<TARGET_PROPERTY:vizkit,SOURCE_DIR>` を使っているのは、**利用者のビルドディレクトリからでも正しいパスが解決されるようにする**ためです。`${CMAKE_CURRENT_SOURCE_DIR}` は、`INTERFACE` として伝播したときに評価される文脈が変わる可能性があるので、ジェネレータ式のほうが安全です。

## 41.4 MSVC 以外を守る

`/NATVIS` は MSVC のリンカー固有のオプションです。Clang や GCC に渡すとエラーになります。クロスプラットフォームのプロジェクトでは、ジェネレータ式で守ります。

```cmake
target_link_options(vizkit INTERFACE
    $<$<CXX_COMPILER_ID:MSVC>:/NATVIS:$<TARGET_PROPERTY:vizkit,SOURCE_DIR>/vizkit.natvis>
)
```

`clang-cl`（MSVC 互換モードの Clang）を使う場合は、`CXX_COMPILER_ID` が `Clang` になりますが、リンカーは `lld-link` や `link.exe` で `/NATVIS` を受け付けることがあります。環境に応じて条件を調整してください。

より丁寧に書くなら、リンカーの種類で判定します。

```cmake
if(MSVC)
    target_link_options(vizkit INTERFACE
        /NATVIS:$<TARGET_PROPERTY:vizkit,SOURCE_DIR>/vizkit.natvis
    )
endif()
```

`if(MSVC)` は `clang-cl` でも真になるので、実務ではこちらのほうが扱いやすいことが多いです。

## 41.5 まとめて関数にする

複数のライブラリで同じことを書くなら、関数に切り出します。

```cmake
# natvis ファイルをターゲットに関連付ける。
#   - Visual Studio ジェネレータ: ソースとして追加（IDE に表示される）
#   - MSVC リンカー: /NATVIS で PDB に埋め込む（INTERFACE で利用者に伝播）
function(vizkit_add_natvis target natvis_file)
    get_filename_component(_abs "${natvis_file}" ABSOLUTE)

    target_sources(${target} PRIVATE "${_abs}")

    if(MSVC)
        target_link_options(${target} INTERFACE "/NATVIS:${_abs}")
    endif()
endfunction()
```

```cmake
add_library(vizkit STATIC src/vizkit.cpp)
target_include_directories(vizkit PUBLIC include)
vizkit_add_natvis(vizkit vizkit.natvis)
```

`target_sources` と `target_link_options` の両方を行っているので、**どのジェネレータでも動きます**。Visual Studio ジェネレータでは両方の経路が有効になりますが、第39章の優先順位により、プロジェクト内のファイルが PDB のものより優先されるので、開発中は `.natvisreload` が使えます（第40章 40.7節と同じ両立）。

## 41.6 install() で同梱する

ライブラリをインストールして配布する場合、`.natvis` も一緒に配ります。

```cmake
install(TARGETS vizkit
    EXPORT vizkitTargets
    ARCHIVE DESTINATION lib
    LIBRARY DESTINATION lib
    RUNTIME DESTINATION bin
)

install(DIRECTORY include/ DESTINATION include)

# natvis ファイル
install(FILES vizkit.natvis DESTINATION lib)

# PDB（MSVC の場合）
if(MSVC)
    install(FILES $<TARGET_PDB_FILE:vizkit> DESTINATION lib OPTIONAL)
endif()
```

### インストール後のパス解決

問題は、インストール後の `/NATVIS` のパスです。ビルドツリーのパスをそのまま `INTERFACE` に埋め込むと、インストール先では存在しません。

`BUILD_INTERFACE` と `INSTALL_INTERFACE` で分けます。

```cmake
if(MSVC)
    target_link_options(vizkit INTERFACE
        "$<BUILD_INTERFACE:/NATVIS:${CMAKE_CURRENT_SOURCE_DIR}/vizkit.natvis>"
        "$<INSTALL_INTERFACE:/NATVIS:${CMAKE_INSTALL_PREFIX}/lib/vizkit.natvis>"
    )
endif()
```

`INSTALL_INTERFACE` で `${CMAKE_INSTALL_PREFIX}` を直接使うのは推奨されないため、より正確にはパッケージ設定ファイル（`vizkitConfig.cmake`）の中で `${PACKAGE_PREFIX_DIR}` を使って組み立てます。

```cmake
# vizkitConfig.cmake.in
@PACKAGE_INIT@
include("${CMAKE_CURRENT_LIST_DIR}/vizkitTargets.cmake")

if(MSVC)
    target_link_options(vizkit::vizkit INTERFACE
        "/NATVIS:${PACKAGE_PREFIX_DIR}/lib/vizkit.natvis"
    )
endif()
```

**PDB 埋め込みまで含めた配布は、ここまでの手間がかかります。** ヘッダーと `.natvis` だけを配り、「利用者が `/NATVIS` を自分で設定する」という運用にとどめるのも、規模によっては合理的な判断です。

## 41.7 vcpkg ポート

vcpkg でライブラリを配布する場合、ポートの `portfile.cmake` で `.natvis` をインストールします。

```cmake
vcpkg_cmake_install()
vcpkg_cmake_config_fixup(PACKAGE_NAME vizkit CONFIG_PATH lib/cmake/vizkit)

# natvis を share に置く
file(INSTALL "${SOURCE_PATH}/vizkit.natvis"
     DESTINATION "${CURRENT_PACKAGES_DIR}/share/${PORT}")
```

vcpkg 経由で入れたライブラリは**ソリューションに含まれない**ので、第39章 39.4節で述べたとおり、プロジェクト内の `.natvis` としては読み込まれません。したがって次のどちらかが必要です。

- **PDB 埋め込み**（`vcpkg` は Debug ビルドの PDB を同梱するので、これが効く）
- **利用者がユーザーフォルダにコピーする**（手作業になる）

vcpkg で配布するライブラリでは、**PDB 埋め込みの重要度が特に高い**と言えます。

## 41.8 この章のまとめ

- **Visual Studio ジェネレータなら `target_sources` に `.natvis` を書くだけでよい**（CMake 3.7 以降）。
- **Ninja ジェネレータは `.natvis` を無視する。** エラーも警告も出ない。Visual Studio の「フォルダーを開く」は既定で Ninja のことが多いので注意。
- Ninja で埋め込むには **`target_link_options` で `/NATVIS:` を渡す**。
- **静的ライブラリでは `PRIVATE` では効かない。`INTERFACE` で利用者に伝播させる。**
- パスは `$<TARGET_PROPERTY:target,SOURCE_DIR>` で解決する。`${CMAKE_CURRENT_SOURCE_DIR}` より安全。
- **MSVC 以外に `/NATVIS` を渡さない。** `if(MSVC)` かジェネレータ式で守る。
- `target_sources` と `target_link_options` を両方行う関数にまとめると、どのジェネレータでも動き、開発中は `.natvisreload` も使える。
- `install()` では `.natvis` と PDB を同梱し、パスは `BUILD_INTERFACE` / `INSTALL_INTERFACE` で分ける。**パッケージ設定ファイルで組み立てるのが正確。**
- **vcpkg 経由のライブラリはソリューションに含まれない。** PDB 埋め込みの重要度が特に高い。

## 41.9 演習

1. `vizkit` を CMake プロジェクトに移し、Visual Studio ジェネレータ（`cmake -G "Visual Studio 18 2026"`）で生成した `.vcxproj` に `<Natvis Include=...>` が入っていることを確認してください。
2. 同じプロジェクトを Ninja（`cmake -G Ninja`）で生成してビルドし、Natvis が効かないことを確認してください。診断メッセージで読み込まれていないことも確かめます。
3. 41.5節の `vizkit_add_natvis()` 関数を実装し、Ninja でも効くようになることを確認してください。
4. `target_link_options` を `PRIVATE` にした状態で静的ライブラリをビルドし、効かないことを確認してください。`INTERFACE` に変えて効くようになることも確かめます。
5. `install()` してから別のプロジェクトで `find_package(vizkit)` して使い、インストール後のパス解決が正しく動くか確認してください。
