---
layout: post
title: "Conda-forge: creating Windows compatible recipes for Autotools based Python projects"
date: 2017-10-01 10:44:30 +0100
comments: true
author: Tom Schoonjans
published: false
categories: [python, conda-forge, anaconda, autotools, windows, python-bindings]
---

It is probably fair to say that Anaconda has become the most popular Python distribution in scientific circles. This should not come as a surprise as it comes with many features that make it stand out against it competitors:

* a convenient installer
* allows for installing several versions of the Python interpreters side-by-side using environments
* has access to the most recent versions of many important (scientific) packages
* does not require admin privileges!

Though Linux and macOS ship with their own Python interpreters, these are in many cases insufficient for development as access to the latest releases of Python packages is usually not possible with the default package managers (outdated packages only) or using pip, especially when binary extensions are required that depend on non-Python libraries.

With Anaconda, it is quite easy to write and build new recipes for Python and non-Python software packages and upload them to a personal distribution channel. Conda-forge makes this even easier: just write the recipe, open a PR on GitHub, and wait for the continuous-integration buildbots to generate binary packages for Linux (Circle-CI), macOS (Travis-CI) and Windows (AppVeyor). After merging the PR, the binary packages will be automatically uploaded to the conda-forge channel, at which point they become available for all Anaconda users.

This procedure works fine as long as the installation script for the Python package uses distutils or setuptools, as it ensures that all files are installed in the appropriate location and that any binary extensions are compiled with the same compiler that was used to compile the Python interpreter. But what happens when the software package uses a different installation system?

Enter my personal flagship project xraylib, a library providing convenient access to physical databases relevant in the field of X-ray physics. This library consists of a core shared library, written in ANSI-C, and comes with bindings for about a dozen of other languages such as Perl, IDL, Ruby, Lua, Fortran, and of course Python. In fact there are two Python bindings: the first extension module is generated with SWIG (deals with scalar arguments) and the second one with Cython (deals with NumPy arrays as arguments).
In order to support building the core library as well as the many bindings using a single build and installation system (and thereby ignoring the recommended build systems each of these languages bindings has for binary extensions), I use autotools, which consists of autoconf, automake and libtool. Libtool is key in this setup, as it enables building both shared libraries (the core C library as well as the Fortran bindings) as well as dynamical loadable plugins (Python, Perl, Lua, Ruby and IDL bindings), in a platform-independent manner! See also my very first blogpost about this.

Such an autotools project can be used via three commands (after unpacking the source tarball):

./configure
make
make install

The configure script figures out where the Python extensions need to be installed in the last command, thereby ensuring they will get picked up by the interpreter without fooling around with the PYTHONPATH environment variable. This works well in a Conda recipe on both macOS and Linux, and has led to several people uploading xraylib packages to their personal conda channels for these two platforms.

Windows however is an entirely different beast.

Over the years, many have asked me to come up with providing better Python support for xraylib on Windows, in particular via pip and conda, as the Python binary extensions I provide in my xraylib Windows SDKs do not integrate easily with Python interpreters, in particular due to their dependency on specific NumPy versions.


MORE

Several major obstacles needed to be addressed before successfully generating xraylib’s Python extensions on Windows with a conda recipe. The AppVeyor buildbots offer only a Windows cmd.exe shell to perform the build, as well as the compilers and linkers that are provided by Visual Studio, Microsoft’s integrated development environment, and of course the Python interpreters.

The first major obstacle one has to overcome when running the three commands on Windows is that it does not come with a Bash compatible shell. This is a big problem since the configure script is in fact a Bash shell script, which also expects several basic UNIX utilities to be present such as cat, rm, mkdir, grep, sed,…, but which of course are missing.
Secondly, there is the compiler problem. Microsoft has its own development suite Visual Studio, which contains a compiler cl.exe suitable for both C and C++ code as well as a linker link.exe, which links the object code together in an executable of library. This compiler and linker are not supported by autotools, and there is no way around this (to the best of my knowledge). So we need to use a different compiler, one which plays nicely with autotools: gcc (MinGW-w64!) or clang, neither of which is present on a Windows system by default.
This brings us immediately to the next problem: the ld linker that is part of MinGW-w64 does not like the import libraries (libs\pythonxy.lib) for the Python DLLs and refuses to link the extensions with these libraries. This is due to the fact that the official Python releases for Windows are compiled with Visual Studio, and that Python really only supports building extensions on this platform using the cl.exe compiler, so they didn’t bother including an ld.exe compatible import library.

Until recently, there was no way to solve any of this. But then Ray Donnelly (@mingwandroid) of Continuum Analytics (the company behind Anaconda) generated conda recipes for the MinGW-w64 compilers and toolchain and added them to the default conda channel. For this effort, he relied extensively on his previous work on the MSYS2 project, which provides Windows users to a Bash shell as well as access to a large number of Unix/Linux utilities and software package via pacman. Fortunately, he also added a conda recipe for the Bash shell, thereby solving my first problem, as I can now launch bash.exe from the Windows cmd.exe that is used by the AppVeyor buildbot.

Now after updating the conda recipe (meta.yaml) with the required build dependencies, the next step was to generate the correct bld.bat file containing the commands necessary to generate the bindings.

Paste bld.bat here

Let’s have a look at the lines in this file.

Set environment variable GCC_ARCH to ensure that the compiler is invoked for the right architecture. The gcc package provided by conda compiles by default to 64-bit, so this needs to be overridden for 32-bit python builds. Note also the presence of an additional flag -DMS_WIN64, necessary to avoid a linker error when compiling 64-bit plugins. (https://stackoverflow.com/questions/2842469/python-undefined-reference-to-imp-py-initmodule4)

IF "%ARCH%" == "64" (
set GCC_ARCH=x86_64-w64-mingw32
set EXTRA_FLAGS=-DMS_WIN64
) else (
set GCC_ARCH=i686-w64-mingw32
)

Ensure there’s a /tmp folder. Noticed that a lot of the Unix utilities complain when this folder is absent.

bash -lc "ln -s ${LOCALAPPDATA}/Temp /tmp"

Download, compile and install swig, which is a hard requirement to compile the bindings, but unfortunately was not added by Ray Donnelly to conda.

bash -lc "curl -L -O https://downloads.sourceforge.net/project/swig/swig/swig-3.0.12/swig-3.0.12.tar.gz && tar xfz swig-3.0.12.tar.gz && cd swig-3.0.12 && ./configure --build=$GCC_ARCH --host=$GCC_ARCH --target=$GCC_ARCH --without-perl5 --without-guile --disable-ccache --prefix=/mingw-w64 && make && make install"

Generate a definitions file out of the python dll with gendef and turn it into an import library with dlltool. This will allow our binary plugins to link against the python dll when using the ld linker (problem 3!)

bash -lc "gendef ${PREFIX}/python${CONDA_PY}.dll && dlltool -l libpython${CONDA_PY}.a -d python${CONDA_PY}.def -k -A"

Autoreconf and configure xraylib, while setting the appropriate options. Notice the presence of the cygpath command, which is necessary to convert conda’s environment variables with Windows style paths to Unix style paths. The last option LIBS=-L$PWD is used to ensure that the linker finds the python import library that was created by the previous command.

bash -lc "autoreconf -fi && ./configure --disable-static --build=$GCC_ARCH --host=$GCC_ARCH --target=$GCC_ARCH --enable-python-integration --disable-fortran2003 --enable-python --prefix=`cygpath -u $PREFIX` --bindir=`cygpath -u $LIBRARY_BIN` --libdir=`cygpath -u $LIBRARY_LIB` PYTHON=`which python` SWIG=`which swig` CPPFLAGS=$EXTRA_FLAGS LIBS=-L$PWD"

Compile and install the bindings

bash -lc "make"
bash -lc "make install"

Move the xraylib core dll to a folder that is picked up by the PATH variable of an Anaconda installation.

bash -lc "mv $PREFIX/Library/usr/bin/libxrl-7.dll $PREFIX/Library/bin/"

Remove redundant files, to avoid them from being packaged into the conda tarballs that will get uploaded.

bash -lc "rm -r $PREFIX/include/xraylib"
bash -lc "rm -rf $PREFIX/Library/tmp"
bash -lc "rm `cygpath -u $LIBRARY_BIN`/xraylib"
bash -lc "rm `cygpath -u $LIBRARY_LIB`/libxrl.la"
bash -lc "rm `cygpath -u $LIBRARY_LIB`/pkgconfig/libxrl.pc"
bash -lc "rm `cygpath -u $SP_DIR`/*.la"
bash -lc "rm `cygpath -u $SP_DIR`/*.dll.a"
bash -lc "rm `cygpath -u $SP_DIR`/xrayhelp.py*"
bash -lc "rm `cygpath -u $SP_DIR`/xraymessages.py*"
bash -lc "rm -r $PREFIX/share/xraylib"
bash -lc "cd swig-3.0.12 && make uninstall"
bash -lc "rm -rf /mingw-w64/share/swig"

It should also be mentioned that considerable changes were necessary to the configure.ac autoconf script of xraylib, as well as to some of its auxiliary macros (in the m4 subdirectory). This was mostly to ensure that the Windows style paths as returned by the Python interpreter when querying for installation information, would be turned into Unix style paths using cygpath. https://github.com/tschoonj/xraylib/commit/dcc2b83004e95064a4f6a5d3ca8727f3819b6bfa.

For those interested in more information, please have a look at the conda-forge xraylib feedstock repository, as well as the xraylib repository itself.

