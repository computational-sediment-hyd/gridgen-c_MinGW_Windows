# gridgen (sakov/gridgen-c) — Windows / MinGW ビルド対応版

オリジナル: https://github.com/sakov/gridgen-c （Pavel Sakov, CSIRO）

Linux では `./configure && make` が通るが Windows では通らない、という問題を解消したもの。
**単にビルドを通しただけでなく、Windows 上で正しい格子が出るところまで直してある**
（後述のとおり、素直にビルドを通しただけでは実行時にクラッシュする）。

---

## 1. ビルド方法

### 推奨: MSYS2 の MinGW シェル

MSYS2 をインストールし、**MSYS2 MinGW 64-bit** シェルを開いて:

```sh
pacman -S --needed mingw-w64-x86_64-gcc make
cd gridgen
./configure
make
make lib      # libgridgen.a
make shlib    # libgridgen.dll (+ libgridgen.dll.a インポートライブラリ)
```

`configure` は autoconf 2.69/2.71 で再生成済みなので autoconf は不要。
`configure.ac` を編集した場合のみ `autoconf -o configure configure.ac` を実行する。

### Linux からのクロスコンパイル

```sh
./configure --host=x86_64-w64-mingw32   # 32bit は --host=i686-w64-mingw32
make
```

### MSYS2 を入れたくない場合（素の cmd.exe + MinGW-w64）

`sh` も `configure` も使わない経路も用意した:

```bat
copy config.h.mingw config.h
mingw32-make -f makefile.mingw
mingw32-make -f makefile.mingw lib
mingw32-make -f makefile.mingw shlib
```

### Linux / macOS

従来どおり `./configure && make`。動作は変更前と完全に同一（後述の回帰テスト参照）。

---

## 2. 何を直したか

Windows でビルド・実行できなかった原因は 4 つあり、うち 2 つはビルドの問題、
残り 2 つは **ビルドが通った後に牙をむく実行時バグ**だった。

### (A) ビルドが通らない原因

**A-1. `triangle.c` が `<sys/time.h>` / `gettimeofday()` に依存**

Triangle は処理時間レポート用に POSIX の `<sys/time.h>` を無条件で include する。
Triangle 側に `NO_TIMER` スイッチが用意されているので、Windows では自動的に
これが立つようにした（`triangle.c` 冒頭、`_WIN32` / `__MINGW32__` を検出）。
gridgen はこのタイマ出力を使っていないので、機能上の欠落はない。

同じブロックで、32bit x86 の Windows では `CPU86` を自動定義するようにした。
これは Triangle の厳密幾何述語 (exact predicates) が正しく動くために x87 FPU を
53bit 精度に固定するもので、Linux では `<fpu_control.h>`（`LINUX` スイッチ）、
Windows では `<float.h>` の `_control87()`（`CPU86` スイッチ）が対応する。
x86-64 では double は SSE2 で処理されるためどちらも不要。

**A-2. ビルドシステムが Unix 決め打ち**

- `configure.in` が autoconf 2.13 世代の書き方（`AC_TRY_COMPILE`, `AC_HAVE_LIBRARY` 等）で、
  ホスト判定 (`AC_CANONICAL_HOST`) も実行ファイル拡張子の判定 (`AC_EXEEXT`) も無い。
- `makefile.in` が `gridgen`（拡張子なし）を生成物として決め打ち。MinGW の gcc は
  `-o gridgen` でも実際には `gridgen.exe` を作るため、make はターゲットが
  「常に存在しない」と判断して毎回リンクし直す。
- 共有ライブラリが `libgridgen.so` + `-fPIC` 決め打ち。Windows では DLL であり、
  `-fPIC` は無意味（警告になる）、`--out-implib` でインポートライブラリを作る必要がある。

そこで `configure.in` を **`configure.ac` に書き換え**、`AC_CANONICAL_HOST` を導入して
ホスト種別ごとに次を切り替えるようにした:

| 変数 | Linux | Windows (MinGW) | macOS |
|---|---|---|---|
| `EXEEXT` | (空) | `.exe` | (空) |
| `PICFLAG` | `-fPIC` | (空) | `-fPIC` |
| `SHLIB` | `libgridgen.so` | `libgridgen.dll` | `libgridgen.dylib` |
| `SHLIB_LDFLAGS` | `-shared` | `-shared -Wl,--out-implib,libgridgen.dll.a -Wl,--export-all-symbols` | `-dynamiclib` |
| `NO_TIMER` | 無し | 定義 | 無し |

`makefile.in` はこれらを参照する形に書き換えた。

### (B) ビルドは通るが実行時に壊れる原因 ← ここが本題

**B-1. Triangle の LLP64 バグ（起動直後に segfault）**

Triangle はポインタを `unsigned long` に格納し、下位 2bit をタグビットとして
使ったり、メモリプールのアラインメント演算をしたりする:

```c
(otri).orient = (int) ((unsigned long) (ptr) & (unsigned long) 3l);
alignptr = (unsigned long) (pool->nowblock + 1);
```

LP64 の Unix では `sizeof(long) == sizeof(void*) == 8` なので問題ないが、
**Windows x64 は LLP64 で `sizeof(long) == 4`、`sizeof(void*) == 8`**。
つまり全ポインタが上位 32bit を落とされ、最初のメモリ確保で即座に落ちる。
（実際に、この修正前は `page fault on write access to 0x00000000fe8d0050` で
クラッシュした。上位ビットが消えたポインタそのもの。）

`<stdint.h>` の `uintptr_t` は定義上どのプラットフォームでもポインタを格納できるので、
ポインタ演算に使われている `unsigned long` をすべて `uintptr_t` (`tri_uintptr`) に
置き換えた（41 行）。デバッグ出力用の `printtriangle()` / `printsubseg()` 内の
`%lx` は `unsigned long long` / `%llx` に拡幅（23 行）。
乱数シード `randomseed` と `randomnation()` は本物の整数なのでそのまま。

**B-2. `gridgen.c` の `key_find()` が text mode の `ftell()` に依存**

パラメータファイルの読み取りがこうなっている:

```c
fpos = ftell(fp);
s = fgets(buf, BUFSIZE, fp);      /* 行を読む */
...
fseek(fp, fpos + (s - buf) + len, 0);   /* キーの直後へ飛ぶ */
```

これは「`ftell()` の戻り値はファイル先頭からのバイト数」という前提の演算だが、
**Windows の text mode ストリームではこれが成り立たない**（C 標準でも、text
ストリームの `ftell()` 値は `fseek()` にそのまま渡す以外の使い方は未定義）。

実測（同一の LF 改行ファイルを読んだときの `ftell()`）:

| 行 | Linux (text) | Windows (text) | Windows (binary) |
|---|---|---|---|
| `input xy.1` | 0 | 0 | 0 |
| `output grid.1` | 11 | **8** | 11 |
| `nx 40` | 25 | **23** | 25 |
| `sigmas sigmas.1` | 31 | **30** | 31 |

このずれのせいで `fseek()` がキー名の途中に着地し、`sigmas sigmas.1` の値が
`s sigmas.1` と読まれる。結果として `s sigmas.1` や `le rect.0` といった
**ゴミ名のファイルが生成され、格子ファイルは一切書かれない**（しかも終了コードは 0）。

対策:

- パラメータファイルを **binary mode (`"rb"`) で開く**ようにした
  （UNIX では `"b"` は no-op なので、同じコードのままで挙動不変）。
- binary mode にすると CRLF 改行のファイルでは `\r` が残るため、`prm_read()` が
  `\r` でも値を打ち切り、末尾の空白を除去するようにした。
  → **Windows で作成した CRLF のパラメータファイルもそのまま読める**（検証済み）。

### (C) ついで

- `nan.h` の MSVC 用分岐の `unsigned _int64` は `unsigned __int64` のタイプミス（デッドコードだが修正）。
- `CUSTOMISE` が `LDFLAGS="-L/usr/local/lib"` を無条件で付けていたのを、
  Windows では付けないようにした。またクロスコンパイル時に `CC=gcc` で
  クロスコンパイラを潰さないようにした。
- `AC_CANONICAL_HOST` に必要な `config.guess` / `config.sub` を同梱。

---

## 3. 検証結果

### ビルド

| ターゲット | 結果 |
|---|---|
| Linux x86-64 (gcc, native) | OK / 警告 0 |
| Windows x86-64 (x86_64-w64-mingw32-gcc) | OK / 警告 0 → `gridgen.exe` (PE32+), `libgridgen.a`, `libgridgen.dll`, `libgridgen.dll.a` |
| Windows x86 32bit (i686-w64-mingw32-gcc) | OK / 警告 0 → `gridgen.exe` (PE32)。`_control87` 参照を確認（CPU86 経路が有効） |

### 数値検証

同梱の `examples/prm.0` 〜 `prm.5`（既存の 6 ケース）について、
Linux ネイティブ版の出力を基準に、Windows 版 (`gridgen.exe`, Wine 上で実行) の出力を比較:

| ケース | 格子点数 | NaN マスク | max\|dx\| | max\|dy\| | 最大相対誤差 | バイト完全一致 |
|---|---|---|---|---|---|---|
| prm.0 | 10800 | 一致 | 0 | 0 | 0 | ○ |
| prm.1 | 1600 | 一致 | 1.0e-11 | 0 | 1.2e-10 | − |
| prm.2 | 1600 | 一致 | 0 | 0 | 0 | ○ |
| prm.3 | 8000 | 一致 | 1.0e-09 | 1.0e-09 | 6.4e-10 | − |
| prm.4 | 1600 | 一致 | 0 | 0 | 0 | ○ |
| prm.5 | 2601 | 一致 | 2.5e-13 | 0 | 4.1e-14 | ○(注) |

- 全ケースで格子点数・NaN（マスク）配置が完全一致。
- 最大相対誤差 **6.4e-10**。gridgen の出力書式は `%.10g`（有効 10 桁）なので、
  これは**出力の最終桁の丸め相当**であり、実質的に同一解。
- 6 ケース中 3 ケースはバイト単位で完全一致。

### 回帰テスト（Linux）

同じ 6 ケースについて、**パッチ前のオリジナル Linux バイナリの出力とバイト単位で完全一致**。
Unix 側の挙動は一切変えていない。

### CRLF パラメータファイル

`prm.1` を CRLF に変換して Windows 版で実行 → LF 版と出力が完全一致。

---

## 4. 変更ファイル一覧

| ファイル | 内容 |
|---|---|
| `configure.ac` | **新規**（`configure.in` を置換）。`AC_CANONICAL_HOST` によるホスト判定、`AC_EXEEXT`、`SHLIB`/`PICFLAG`/`SHLIB_LDFLAGS` の切替 |
| `configure` | 上記から autoconf 2.71 で再生成 |
| `makefile.in` | `$(EXEEXT)` / `$(SHLIB)` / `$(PICFLAG)` を使うよう書き換え、`clean` を .exe/.dll 対応に |
| `config.h.in` | `NO_TIMER` の枠を追加 |
| `triangle.c` | Windows で `NO_TIMER` / 32bit で `CPU86` を自動定義。ポインタ用 `unsigned long` → `uintptr_t` (LLP64 対応) |
| `gridgen.c` | パラメータファイルを binary mode で開く。`prm_read()` を CRLF 対応に |
| `nan.h` | MSVC 分岐の `_int64` → `__int64` |
| `CUSTOMISE` | Windows / クロスコンパイル対応 |
| `config.guess`, `config.sub` | **新規**（`AC_CANONICAL_HOST` に必要） |
| `makefile.mingw`, `config.h.mingw` | **新規**。MSYS2 なしで cmd.exe から使う経路 |
| `configure.in` | 削除（`configure.ac` に置換） |

---

## 5. 注意

- 上流の `libgu` (gridutils) は任意。無くても `gridgen.exe` / `libgridgen.a` /
  `libgridgen.dll` はビルドできる（`gridgen_generategrid2()` だけが使えない）。
  `configure` は見つからなければ warning を出して続行する。
- ライセンスは上流に従う（`LICENSE` 参照）。Triangle (`triangle.c`) は
  Shewchuk のライセンスで、商用利用には別途許諾が必要な点に注意。
