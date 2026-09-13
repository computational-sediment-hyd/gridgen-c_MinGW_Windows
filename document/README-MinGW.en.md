# gridgen (sakov/gridgen-c) — Windows / MinGW build port

Upstream: https://github.com/sakov/gridgen-c (Pavel Sakov, CSIRO)

This fixes the fact that `./configure && make` works on Linux but fails on
Windows. **It's not just a "make it compile" fix — it's fixed all the way
through to producing correct grids on Windows** (as detailed below, a naive
"just get it to build" fix still crashes at runtime).

---

## 1. How to build

### Recommended: MSYS2 MinGW shell

Install MSYS2, open the **MSYS2 MinGW 64-bit** shell, then:

```sh
pacman -S --needed mingw-w64-x86_64-gcc make
cd gridgen
./configure
make
make lib      # libgridgen.a
make shlib    # libgridgen.dll (+ libgridgen.dll.a import library)
```

`configure` has already been regenerated with autoconf 2.69/2.71, so
autoconf itself is not required. Only run
`autoconf -o configure configure.ac` if you edit `configure.ac`.

### Cross-compiling from Linux

```sh
./configure --host=x86_64-w64-mingw32   # 32-bit: --host=i686-w64-mingw32
make
```

### If you don't want to install MSYS2 (plain cmd.exe + MinGW-w64)

A path that needs neither `sh` nor `configure` is also included:

```bat
copy config.h.mingw config.h
mingw32-make -f makefile.mingw
mingw32-make -f makefile.mingw lib
mingw32-make -f makefile.mingw shlib
```

### Linux / macOS

Unchanged: `./configure && make`. Behavior is byte-for-byte identical to
before the patch (see the regression test results below).

---

## 2. What was broken and what was fixed

There were four reasons the Windows build/run didn't work: two were build
problems, and the other two were **runtime bugs that only bite after the
build succeeds**.

### (A) Why it wouldn't build

**A-1. `triangle.c` depends on `<sys/time.h>` / `gettimeofday()`**

Triangle unconditionally includes the POSIX `<sys/time.h>` header for its
timing report. Triangle already has a `NO_TIMER` switch for this, so it's
now defined automatically on Windows (detected via `_WIN32` /
`__MINGW32__` at the top of `triangle.c`). gridgen doesn't use this timing
output at all, so nothing functional is lost.

In the same block, `CPU86` is now defined automatically for 32-bit x86
Windows builds. This pins the x87 FPU to 53-bit precision, which Triangle's
exact geometric predicates require to work correctly — on Linux this is
done via `<fpu_control.h>` (the `LINUX` switch), and on Windows via
`_control87()` from `<float.h>` (the `CPU86` switch). Neither is needed on
x86-64, since doubles are handled by SSE2 there.

**A-2. The build system hard-codes Unix assumptions**

- `configure.in` was written in autoconf-2.13-era style (`AC_TRY_COMPILE`,
  `AC_HAVE_LIBRARY`, etc.) with no host detection (`AC_CANONICAL_HOST`) and
  no executable-suffix detection (`AC_EXEEXT`).
- `makefile.in` hard-codes `gridgen` (no extension) as the build product.
  MinGW's gcc actually produces `gridgen.exe` even when you pass
  `-o gridgen`, so make would decide the target "never exists" and relink
  every time.
- The shared library was hard-coded to `libgridgen.so` + `-fPIC`. On
  Windows it needs to be a DLL, `-fPIC` is meaningless there (and triggers
  a warning), and an import library needs to be produced via
  `--out-implib`.

So `configure.in` was **rewritten as `configure.ac`**, introducing
`AC_CANONICAL_HOST` to switch the following per host:

| Variable | Linux | Windows (MinGW) | macOS |
|---|---|---|---|
| `EXEEXT` | (empty) | `.exe` | (empty) |
| `PICFLAG` | `-fPIC` | (empty) | `-fPIC` |
| `SHLIB` | `libgridgen.so` | `libgridgen.dll` | `libgridgen.dylib` |
| `SHLIB_LDFLAGS` | `-shared` | `-shared -Wl,--out-implib,libgridgen.dll.a -Wl,--export-all-symbols` | `-dynamiclib` |
| `NO_TIMER` | not defined | defined | not defined |

`makefile.in` was rewritten to reference these substitutions.

### (B) Why it built but crashed at runtime ← the real story

**B-1. Triangle's LLP64 bug (segfault right at startup)**

Triangle stores pointers in `unsigned long`, using the low 2 bits as tag
bits and doing memory-pool alignment arithmetic on them:

```c
(otri).orient = (int) ((unsigned long) (ptr) & (unsigned long) 3l);
alignptr = (unsigned long) (pool->nowblock + 1);
```

On LP64 Unix, `sizeof(long) == sizeof(void*) == 8`, so this is fine. But
**Windows x64 is LLP64: `sizeof(long) == 4` while `sizeof(void*) == 8`**.
Every pointer gets its upper 32 bits truncated, and it crashes on the very
first memory allocation. (Before this fix, it actually crashed with
`page fault on write access to 0x00000000fe8d0050` — that value is a
pointer with its high bits chopped off, in the flesh.)

`uintptr_t` from `<stdint.h>` is guaranteed by definition to hold a pointer
on any platform, so every pointer-arithmetic use of `unsigned long` was
replaced with `uintptr_t` (aliased as `tri_uintptr`) — 41 lines. The debug
printers `printtriangle()` / `printsubseg()` had their `%lx` widened to
`unsigned long long` / `%llx` (23 lines). The random seed `randomseed` and
`randomnation()` are genuine integers, not pointers, and were left alone.

**B-2. `gridgen.c`'s `key_find()` relies on `ftell()` in text mode**

The parameter-file reader does this:

```c
fpos = ftell(fp);
s = fgets(buf, BUFSIZE, fp);      /* read a line */
...
fseek(fp, fpos + (s - buf) + len, 0);   /* seek to just past the key */
```

This assumes `ftell()`'s return value is a byte offset from the start of
the file — but **that assumption does not hold for a text-mode stream on
Windows** (per the C standard, a text-stream `ftell()` value is only
defined for feeding straight back into `fseek()`, not for arithmetic).

Measured (`ftell()` while reading the same LF-terminated file):

| Line | Linux (text) | Windows (text) | Windows (binary) |
|---|---|---|---|
| `input xy.1` | 0 | 0 | 0 |
| `output grid.1` | 11 | **8** | 11 |
| `nx 40` | 25 | **23** | 25 |
| `sigmas sigmas.1` | 31 | **30** | 31 |

Because of this drift, `fseek()` lands in the middle of a key name, so
`sigmas sigmas.1`'s value gets read as `s sigmas.1`. The result: **garbage
filenames like `s sigmas.1` or `le rect.0` get created, and no grid file
is ever written** — while the process still exits with status 0.

Fix:

- The parameter file is now opened in **binary mode (`"rb"`)**
  (`"b"` is a no-op on Unix, so behavior there is unchanged).
- Opening in binary mode leaves a trailing `\r` on CRLF files, so
  `prm_read()` now also stops a value at `\r` and strips trailing
  whitespace. → **Parameter files with Windows (CRLF) line endings can now
  be read as-is** (verified).

### (C) While we were at it

- `nan.h`'s MSVC branch had `unsigned _int64`, a typo for
  `unsigned __int64` (dead code on this toolchain, but fixed anyway).
- `CUSTOMISE` unconditionally appended `LDFLAGS="-L/usr/local/lib"`; this
  is now skipped on Windows. Also, `CC=gcc` is no longer forced when
  cross-compiling, so it doesn't clobber the cross compiler.
- `config.guess` / `config.sub`, required by `AC_CANONICAL_HOST`, are now
  bundled.

---

## 3. Verification results

### Build

| Target | Result |
|---|---|
| Linux x86-64 (gcc, native) | OK / 0 warnings |
| Windows x86-64 (x86_64-w64-mingw32-gcc) | OK / 0 warnings → `gridgen.exe` (PE32+), `libgridgen.a`, `libgridgen.dll`, `libgridgen.dll.a` |
| Windows x86 32-bit (i686-w64-mingw32-gcc) | OK / 0 warnings → `gridgen.exe` (PE32). Confirmed `_control87` references (CPU86 path active) |

### Numerical verification

Using the bundled `examples/prm.0` through `prm.5` (the six existing test
cases), the Windows build's output (`gridgen.exe`, run under Wine) was
compared against the native Linux build's output as the reference:

| Case | Grid points | NaN mask | max\|dx\| | max\|dy\| | Max relative error | Byte-identical |
|---|---|---|---|---|---|---|
| prm.0 | 10800 | match | 0 | 0 | 0 | yes |
| prm.1 | 1600 | match | 1.0e-11 | 0 | 1.2e-10 | no |
| prm.2 | 1600 | match | 0 | 0 | 0 | yes |
| prm.3 | 8000 | match | 1.0e-09 | 1.0e-09 | 6.4e-10 | no |
| prm.4 | 1600 | match | 0 | 0 | 0 | yes |
| prm.5 | 2601 | match | 2.5e-13 | 0 | 4.1e-14 | yes (see note) |

- Grid-point counts and the NaN mask pattern match exactly in every case.
- Maximum relative error is **6.4e-10**. Since gridgen's output format is
  `%.10g` (10 significant digits), this is consistent with rounding in the
  last printed digit — i.e., effectively the same solution.
- 3 of the 6 cases are byte-identical.

### Regression test (Linux)

For the same 6 cases, output is **byte-identical to the original,
unpatched Linux binary**. Unix-side behavior is unchanged.

### CRLF parameter file

`prm.1` converted to CRLF and run on the Windows build → output identical
to the LF version.

---

## 4. Files changed

| File | What changed |
|---|---|
| `configure.ac` | **New** (replaces `configure.in`). Host detection via `AC_CANONICAL_HOST`, `AC_EXEEXT`, switching of `SHLIB`/`PICFLAG`/`SHLIB_LDFLAGS` |
| `configure` | Regenerated from the above with autoconf 2.71 |
| `makefile.in` | Rewritten to use `$(EXEEXT)` / `$(SHLIB)` / `$(PICFLAG)`; `clean` now handles .exe/.dll |
| `config.h.in` | Added the `NO_TIMER` slot |
| `triangle.c` | Auto-defines `NO_TIMER` on Windows, `CPU86` on 32-bit. Pointer-holding `unsigned long` → `uintptr_t` (LLP64 fix) |
| `gridgen.c` | Opens the parameter file in binary mode. `prm_read()` made CRLF-safe |
| `nan.h` | MSVC branch `_int64` → `__int64` |
| `CUSTOMISE` | Windows / cross-compile support |
| `config.guess`, `config.sub` | **New** (required by `AC_CANONICAL_HOST`) |
| `makefile.mingw`, `config.h.mingw` | **New**. A path for plain cmd.exe, no MSYS2 needed |
| `configure.in` | Removed (replaced by `configure.ac`) |

---

## 5. Notes

- The upstream `libgu` (gridutils) dependency is optional. `gridgen.exe` /
  `libgridgen.a` / `libgridgen.dll` all build without it (only
  `gridgen_generategrid2()` becomes unavailable). `configure` prints a
  warning and continues if it isn't found.
- Licensing follows upstream (see `LICENSE`). Note that Triangle
  (`triangle.c`) is under Shewchuk's license, which requires separate
  permission for commercial use.
