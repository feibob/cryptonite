dnl Process this file with autoconf to produce a configure script.

AC_INIT(encfs/encfs.h) dnl a source file from your sub dir
AC_CONFIG_AUX_DIR([build-aux])
AM_INIT_AUTOMAKE(encfs, 1.8.0) dnl searches for some needed programs
AC_CONFIG_MACRO_DIR([m4])

AC_CANONICAL_HOST
AM_CONDITIONAL([DARWIN],
               [case $host_os in darwin*) true;; *) false;; esac])

dnl without this order in this file, automake will be confused!
dnl
AM_CONFIG_HEADER(config.h)

dnl This ksh/zsh feature conflicts with `cd blah ; pwd`
unset CDPATH

AC_LANG_CPLUSPLUS
AC_PROG_CXX
    
dnl almost the same like KDE_SET_PEFIX but the path is /usr/local
dnl
unset CDPATH
dnl make /usr/local the default for the installation
AC_PREFIX_DEFAULT(/usr/local)

AM_GNU_GETTEXT([external])
AM_GNU_GETTEXT_VERSION([0.17])
dnl AC_LIB_LINKFLAGS([asprintf]) # use internal copy
LIBINTL=-lgettextlib

dnl create only shared libtool-libraries
dnl These can be overridden by command line arguments
AC_ENABLE_SHARED(yes)
AC_ENABLE_STATIC(no)

AM_CONDITIONAL( BUILD_STATIC, test "x$enable_static" = "xyes" )
dnl only build either static or shared, not both..
if test "x$enable_static" = "xyes"; then
    enable_shared=no
    AC_DEFINE(BUILD_STATIC, [1], [Building static library])
fi

AC_PROG_LIBTOOL

AX_PTHREAD

AC_CHECK_HEADERS([tr1/memory tr1/unordered_map tr1/unordered_set tr1/tuple])

dnl Need to include any user specified flags in the tests below, as they might
dnl specify required include directories..
FUSE_FLAGS="-D_FILE_OFFSET_BITS=64 -DFUSE_USE_VERSION=26"
CPPFLAGS="$CPPFLAGS $USER_INCLUDES $FUSE_FLAGS -D__STDC_FORMAT_MACROS"
CXXFLAGS="$CXXFLAGS $PTHREAD_CFLAGS $USER_INCLUDES"
LDFLAGS="$LDFLAGS $PTHREAD_LIBS $USER_LDFLAGS $FUSE_LIBS"

dnl Look for fuse headers.
AX_EXT_HAVE_HEADER(fuse.h, /usr/include/fuse /usr/local/include/fuse \
                   /opt/include/fuse /opt/local/include/fuse \
                   /usr/include/osxfuse /usr/local/include/osxfuse)

dnl Ensure the necessary paths are added to LDPATH
AX_EXT_HAVE_LIB(/usr/lib /usr/local/lib /opt/lib /opt/local/lib, fuse,
                fuse_new, [])
AX_EXT_HAVE_LIB(/usr/lib /usr/local/lib /opt/lib /opt/local/lib, osxfuse,
                fuse_new, [])

if test "$GXX" = "yes"; then
    CXXFLAGS="-W -Wall -Wpointer-arith -Wwrite-strings $CXXFLAGS"
    dnl CXXFLAGS="$CXXFLAGS -Wformat=2 -Wconversion"
fi

if test -z "${DARWIN_TRUE}"; then
    dnl Prefer OSXFuse, but fall back to libfuse.
    AC_CHECK_LIB(osxfuse, fuse_new, [FUSE_LIBS="$FUSE_LIBS -losxfuse"],
        AC_CHECK_LIB(fuse, fuse_new, [FUSE_LIBS="$FUSE_LIBS -lfuse"],
                    AC_MSG_ERROR([Unable to find libfuse or libosxfuse.])))
else
    AC_CHECK_LIB(fuse, fuse_new, [FUSE_LIBS="$FUSE_LIBS -lfuse"],
                 AC_MSG_ERROR([Unable to find libfuse.]))
fi

# check for a supported FUSE_MAJOR_VERSION.
dnl AC_MSG_CHECKING([For supported FUSE API version])
dnl AC_RUN_IFELSE([ 
dnl     AC_LANG_PROGRAM([[#include "fuse.h"]],
dnl [[
dnl     if(FUSE_MAJOR_VERSION == 2 && FUSE_MINOR_VERSION >= 5)
dnl     {
dnl 	return 0;
dnl     } else
dnl 	return -1;
dnl ]])], 
dnl     [AC_MSG_RESULT([yes])],
dnl     [AC_MSG_RESULT([no])
dnl     AC_MSG_FAILURE([
dnl Encfs 1.3 requires FUSE 2.5 or newer.  Please check config.log for errors.  If
dnl you cannot determine the problem, mail encfs-users@lists.sourceforge.net
dnl and include the config.log file])
dnl     ]
dnl )

dnl fuse_operations.setxattr was added 2004-03-31
dnl only enable it if setxattr function is found..
AC_CHECK_HEADERS([attr/xattr.h sys/xattr.h])

dnl xattr functions take additional arguments on some systems (eg Darwin).
AC_CACHE_CHECK([whether xattr interface takes additional options], 
        smb_attr_cv_xattr_add_opt, [
            old_LIBS=$LIBS
            LIBS="$LIBS $ACL_LIBS"
            AC_TRY_COMPILE([
                #include <sys/types.h>
                #if HAVE_ATTR_XATTR_H
                #include <attr/xattr.h>
                #elif HAVE_SYS_XATTR_H
                #include <sys/xattr.h>
                #endif
            ],[
               getxattr(0, 0, 0, 0, 0, 0);
               ],
           [smb_attr_cv_xattr_add_opt=yes],
           [smb_attr_cv_xattr_add_opt=no;LIBS=$old_LIBS])
])
if test x"$smb_attr_cv_xattr_add_opt" = x"yes"; then
    AC_DEFINE(XATTR_ADD_OPT, 1, [xattr functions have additional options])
fi

dnl Check for valgrind headers..
AC_ARG_ENABLE(valgrind,
    AC_HELP_STRING([--enable-valgrind],
                   [build with valgrind support.]),
       AC_CHECK_HEADERS([valgrind/valgrind.h valgrind/memcheck.h])
       )

# allow user option of not using ssl..
AC_ARG_ENABLE(openssl,
    AC_HELP_STRING([--disable-openssl],
		   [disables openssl library usage.]),
	with_openssl=$enableval, with_openssl="yes" )

# try checking for openssl using 
if test "x$with_openssl" = "xyes"; then
    # look for openssl using pkg-config first..
    PKG_CHECK_MODULES(OPENSSL, openssl >= 0.9.7,
	with_openssl="yes", with_openssl="old-test")

    # If that fails, try checking via old methods - which isn't as robust when
    # it comes to extra include paths, etc..
    if test "x$with_openssl" = "xold-test"; then
	AC_CHECK_HEADER(openssl/ssl.h,
	    AC_CHECK_LIB(ssl,SSL_new,
		[with_openssl="yes"]))
	OPENSSL_LIBS="-lssl"
    fi

    # if we have openssl, then examine available interfaces.
    if test "x$with_openssl" = "xyes"; then
	AC_DEFINE(HAVE_SSL, [1], [Linking with OpenSSL])

	# add in the libs just for the test..
	oldflags=$CXXFLAGS
	oldlibs=$LIBS
	CXXFLAGS="$CXXFLAGS $OPENSSL_CFLAGS"
	LIBS="$LIBS $OPENSSL_LIBS"
	AC_CHECK_FUNCS(EVP_aes_128_cbc EVP_aes_192_cbc EVP_aes_256_cbc,
	    AC_DEFINE(HAVE_EVP_AES, [1], [Have EVP AES interfaces]))
	AC_CHECK_FUNCS(EVP_bf_cbc,
	    AC_DEFINE(HAVE_EVP_BF, [1], [Have EVP Blowfish interfaces]))
	AC_CHECK_FUNCS(EVP_CIPHER_CTX_set_padding, 
	    with_opensslevp=yes,
	    AC_MSG_WARN([New SSL cipher code only enabled for OpenSSL 0.9.7 or later]))
	AC_CHECK_FUNCS(HMAC_Init_ex,
	    AC_DEFINE(HAVE_HMAC_INIT_EX, [1], [Have HMAC_Init_ex function]))
	
	CXXFLAGS="$oldflags"
	LIBS="$oldlibs"

	AC_SUBST(HAVE_EVP_AES)
	AC_SUBST(HAVE_EVP_BF)
	AC_SUBST(HAVE_HMAC_INIT_EX)
    fi
fi
AM_CONDITIONAL( BUILD_OPENSSL, test "x$with_openssl" = "xyes" )
AM_CONDITIONAL( BUILD_SSLCIPHER, test "x$with_opensslevp" = "xyes" )
AC_SUBST(HAVE_SSL)

if test "x$with_openssl" != "xyes"; then
    AC_MSG_ERROR( [Encfs requires OpenSSL])
fi

# check for RLOG
PKG_CHECK_MODULES(RLOG, librlog >= 1.3, with_rlog="yes", with_rlog="test")

# manual check for rlog, unless environment variable already set
if test "$with_rlog" = "test" && test "x$RLOG_LIBS" = "x"; then
    AC_MSG_WARN([Checking for librlog the hard way])
    AC_CHECK_LIB(rlog, RLogVersion, [RLOG_LIBS="-lrlog"],
        [AC_MSG_ERROR([EncFS depends on librlog])]) 
fi

# find Protocol Buffers
PKG_CHECK_MODULES(PROTOBUF, protobuf >= 2.0)
AC_PATH_PROG(PROTOC, protoc, [no])
if test "$PROTOC" == "no"; then
    AC_MSG_FAILURE([Protocol Buffers compiler 'protoc' is required to build.])
fi

# find TinyXML
AC_LANG_PUSH([C++])
CPPFLAGS="$CPPFLAGS -DTIXML_USE_STL"
AC_CHECK_HEADER([tinyxml.h],,[AC_MSG_ERROR([tinyxml.h not found])])
AC_CHECK_LIB([tinyxml],[main],,
             [AC_MSG_ERROR([you must install libtinyxml dev])])
AC_LANG_POP([C++])

# look for pod2man program for building man pages
AC_PATH_PROG(POD2MAN, pod2man, [no])
AC_PATH_PROG(POD2HTML, pod2html, [no])
AM_CONDITIONAL( BUILD_MAN, test "x$POD2MAN" != "xno" )
AM_CONDITIONAL( BUILD_MANHTML, test "x$POD2HTML" != "xno" )
AM_CONDITIONAL( BUILD_NLS, test "x$USE_NLS" != "xno" )


AC_CONFIG_FILES([Makefile] \
	[encfs/Makefile] \
	[encfs.spec] \
	[makedist2.sh] \
	[m4/Makefile] \
	[po/Makefile.in] \
	[po/Makefile])

AC_OUTPUT

