dnl configure.ac for gridgen
dnl (MinGW/MSYS2 対応版 -- 旧 configure.in を autoconf 2.69/2.71 用に書き換え)

AC_INIT([gridgen],[1.51.5])
AC_CONFIG_SRCDIR([CUSTOMISE])
AC_CONFIG_HEADERS([config.h])

dnl ---------------------------------------------------------------------
dnl ホスト種別の判定 (Windows/MinGW を識別するために必須)
dnl ---------------------------------------------------------------------
AC_CANONICAL_HOST

dnl Get the shell variable to override local customisations.
AC_DEFUN([AC_CUSTOMISE],
[dnl
    AC_BEFORE([$0], [AC_DEFAULT])dnl
    if test -r CUSTOMISE; then
        . ./CUSTOMISE
    fi
])

AC_CUSTOMISE

dnl Checks for programs.
AC_PROG_CC
AC_PROG_INSTALL
AC_CHECK_TOOL(AR, ar, ar)
AC_CHECK_TOOL(RANLIB, ranlib, :)
AC_CHECK_TOOL(STRIP, strip, :)

dnl EXEEXT を確定させる (MinGW では .exe)。
dnl これを行わないと make が gridgen.exe を毎回作り直そうとする。
AC_EXEEXT
AC_OBJEXT

AC_CHECK_HEADER(math.h, , AC_MSG_ERROR([unable to find header]))
AC_SEARCH_LIBS(cos, m, , AC_MSG_ERROR([unable to find library libm]))

AC_CHECK_FUNCS(tgamma)
if test "$ac_cv_func_tgamma" = no; then
  AC_CHECK_LIB(m, tgamma, AC_DEFINE(HAVE_TGAMMA))
fi

if test "$ac_cv_lib_m_tgamma" = no; then
  AC_CHECK_FUNCS(lgamma)
  if test "$ac_cv_func_lgamma" = no; then
    AC_CHECK_LIB(m, lgamma, AC_DEFINE(HAVE_LGAMMA), AC_MSG_ERROR([Unable to locate both tgamma and lgamma]))
  fi
fi

dnl ---------------------------------------------------------------------
dnl 複素数サポート
dnl MinGW-w64 (GCC) は complex.h / _Complex / clog / cexp をすべて備えるので
dnl NOCOMPLEX 経路には落ちない。
dnl ---------------------------------------------------------------------
AC_CHECK_HEADER(complex.h, complex_h=yes, complex_h=no)
AC_CHECK_FUNCS(clog, , complex_h=no)
AC_CHECK_FUNCS(cexp, , complex_h=no)
if test "$complex_h" = yes; then
  AC_MSG_CHECKING([for keyword _Complex])
  AC_COMPILE_IFELSE([AC_LANG_PROGRAM([[#include <complex.h>]],
                                     [[_Complex double z = 1.0 + 1.0 * I; (void) z;]])],
                    [_Complex=yes], [_Complex=no])
  AC_MSG_RESULT([$_Complex])

  if test "$_Complex" = yes; then
    AC_DEFINE(HAVE_COMPLEX)
  else
    AC_MSG_CHECKING([for keyword __complex__])
    AC_COMPILE_IFELSE([AC_LANG_PROGRAM([[#include <complex.h>]],
                                       [[__complex__ double z = 1.0 + 1.0 * I; (void) z;]])],
                      [__complex__=yes], [__complex__=no])
    AC_MSG_RESULT([$__complex__])
    if test "$__complex__" = no; then
      AC_MSG_ERROR([keywords _Complex or __complex__ not supported])
    else
      AC_DEFINE(HAVE__COMPLEX__)
    fi
  fi
else
  echo '  using code in gridgen/c99-min for complex arithmetics support...'
  echo '  warning: this may slow down things by a factor of two or so compared'
  echo '           to a native implementation'
  NOCOMPLEX=yes
  AC_DEFINE(NOCOMPLEX)
fi

dnl ---------------------------------------------------------------------
dnl gridutils (libgu) -- 任意。無くても gridgen 本体はビルドできる。
dnl ---------------------------------------------------------------------
AC_CHECK_HEADER(gridnodes.h, gridnodes_h=yes, gridnodes_h=no)
if test "$gridnodes_h" = yes; then
   AC_DEFINE(HAVE_GRIDNODES_H)
else
   AC_MSG_WARN([unable to find gridnodes.h. You will not be able to compile the library function gridgen_generategrid2().])
fi

AC_CHECK_LIB(gu, gridnodes_readnextpoint, , AC_MSG_WARN([unable to find library libgu; you will not be able to build the shared library]))

dnl ---------------------------------------------------------------------
dnl ホスト依存の設定 (ここが MinGW 対応の中核)
dnl   PICFLAG        : Windows の DLL では -fPIC は無意味 (警告になる)
dnl   SHLIB          : 共有ライブラリのファイル名
dnl   SHLIB_LDFLAGS  : 共有ライブラリのリンクオプション
dnl   CFLAGS_TRIANGLE: triangle.c は Windows では NO_TIMER が必要
dnl                    (sys/time.h / gettimeofday に依存しないようにする)
dnl ---------------------------------------------------------------------
case "$host_os" in
  mingw* | msys* | cygwin*)
    WIN32=yes
    PICFLAG=
    SHLIB=libgridgen.dll
    SHLIB_LDFLAGS="-shared -Wl,--out-implib,libgridgen.dll.a -Wl,--export-all-symbols"
    CFLAGS="$CFLAGS -DNO_TIMER"
    CFLAGS_TRIANGLE="$CFLAGS_TRIANGLE -DNO_TIMER"
    AC_DEFINE(NO_TIMER)
    ;;
  darwin*)
    WIN32=no
    PICFLAG=-fPIC
    SHLIB=libgridgen.dylib
    SHLIB_LDFLAGS="-dynamiclib"
    ;;
  *)
    WIN32=no
    PICFLAG=-fPIC
    SHLIB=libgridgen.so
    SHLIB_LDFLAGS="-shared"
    ;;
esac
AC_MSG_CHECKING([host type])
AC_MSG_RESULT([$host_os (shared library: $SHLIB)])

AC_SUBST(NOCOMPLEX)
AC_SUBST(CC)
AC_SUBST(CFLAGS)
AC_SUBST(AR)
AC_SUBST(RANLIB)
AC_SUBST(STRIP)
AC_SUBST(LDFLAGS)
AC_SUBST(CFLAGS_TRIANGLE)
AC_SUBST(HAVE_GRIDNODES_H)
AC_SUBST(EXEEXT)
AC_SUBST(OBJEXT)
AC_SUBST(WIN32)
AC_SUBST(PICFLAG)
AC_SUBST(SHLIB)
AC_SUBST(SHLIB_LDFLAGS)

AC_CONFIG_FILES([makefile])
AC_OUTPUT
