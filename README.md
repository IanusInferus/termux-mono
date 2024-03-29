﻿# How to install Mono and MSBuild inside Termux

## Description

[Termux](https://github.com/termux/termux-app) is an Android terminal app and Linux environment.

[Mono](https://www.mono-project.com) is an implementation of .Net Framework on various platforms, including Android.

[MSBuild](https://github.com/microsoft/msbuild) is the default build system in Mono (and .Net).

Skip `Build Mono from source` and `Install MSBuild` if you want to use the prebuilt binaries from `Releases`.

## Build Mono from source

There is no pre-compiled binary for Mono on Termux now, except for prooted Arch Linux and Ubuntu. ([40](https://github.com/termux/termux-packages/issues/40))

Here I describe a process to cross-compile Mono 6.12 from source on Linux.

Termux now defaults to Android 7.0 (API Level 24, `/data/data/com.termux/files/usr/include/android/api-level.h`). I cross-compile on Ubuntu 20.04 x86_64 and target Android 7.0.

    # at first, make sure you have mono installed on the host machine
    # https://www.mono-project.com/download/stable/#download-lin

    sudo apt install wget perl python3 cmake unzip git ninja-build g++

    # get Android NDK from https://developer.android.com/ndk/downloads for Linux
    # say android-ndk-r22-linux-x86_64.zip
    unzip android-ndk-r22-linux-x86_64.zip

    wget https://download.mono-project.com/sources/mono/mono-6.12.0.122.tar.xz

    # compile mono for PC with .Net Class Libraries
    mkdir mono-6.12.0.122-PC
    tar xf mono-6.12.0.122.tar.xz -C mono-6.12.0.122-PC --strip-components=1
    cd mono-6.12.0.122-PC
    export CC=
    export CXX=
    export LDFLAGS=
    ./configure --disable-system-aot
    make
    make install "DESTDIR=$(realpath ..)/mono-6.12.0.122-bin-PC"
    cd ..

    # compile mono for Android without .Net Class Libraries
    tar xf mono-6.12.0.122.tar.xz
    cd mono-6.12.0.122
    arch=aarch64 # arch can be any of aarch64, armv7a
    api_level=24 # api level can be any of 21-24,26-30
    # patch for log output (defaults to adb log rather than terminal)
    sed -i 's|#if HOST_ANDROID|#if 0|g' mono/eglib/goutput.c
    sed -i 's/#if defined(HOST_ANDROID) || !defined(HAVE_ATEXIT)/#if 0/g' mono/utils/mono-proclib.c
    if [ "$arch" = "armv7a" ]; then
      sed -i 's|#define USE_TKILL_ON_ANDROID 1||g' mono/metadata/threads.c
      sed -i 's|#define USE_TKILL_ON_ANDROID 1||g' mono/utils/mono-threads-posix.c
    fi
    if [ "$arch" = "armv7a" ]; then
      target=$arch-linux-androideabi
    else
      target=$arch-linux-android
    fi
    export CC="$(realpath ..)/android-ndk-r22/toolchains/llvm/prebuilt/linux-x86_64/bin/clang --target=$target$api_level --sysroot=$(realpath ..)/android-ndk-r22/toolchains/llvm/prebuilt/linux-x86_64/sysroot"
    export CXX="$(realpath ..)/android-ndk-r22/toolchains/llvm/prebuilt/linux-x86_64/bin/clang --target=$target$api_level --sysroot=$(realpath ..)/android-ndk-r22/toolchains/llvm/prebuilt/linux-x86_64/sysroot"
    if [ "$api_level" -ge "24" ]; then
      export LDFLAGS="-lz -Wl,-rpath='\$\$ORIGIN/../lib' -Wl,--enable-new-dtags"
    else
      export LDFLAGS="-lz" # see https://github.com/termux/termux-packages/issues/2071
    fi
    if [ "$arch" = "armv7a" ]; then
      extra_flags=--with-btls-android-ndk-asm-workaround
    else
      extra_flags=
    fi
    ./configure --host=$target --prefix=/data/data/com.termux/files/usr/local --disable-mcs-build --with-btls-android-ndk="$(realpath ..)/android-ndk-r22" --with-btls-android-api=$api_level --with-btls-android-cmake-toolchain="$(realpath ..)/android-ndk-r22/build/cmake/android.toolchain.cmake" $extra_flags
    make
    make install "DESTDIR=$(realpath ..)/mono-6.12.0.122-bin"
    cd ..

    # copy .Net Class Libraries
    cp -rf mono-6.12.0.122-bin-PC/usr/local/lib/mono mono-6.12.0.122-bin/data/data/com.termux/files/usr/local/lib/

    # Some libraries may use libc.so.6, which doesn't exist on Android. We can create a symbol link to libc.so.
    if [ "$arch" = "armv7a" ]; then
      ln -s /system/lib/libc.so mono-6.12.0.122-bin/data/data/com.termux/files/usr/local/lib/libc.so.6
    else
      ln -s /system/lib64/libc.so mono-6.12.0.122-bin/data/data/com.termux/files/usr/local/lib/libc.so.6
    fi

## Install MSBuild

Don't try to build MSBuild from source, it depends on a .Net Core version which is not shipped with Visual Studio and is very difficult to compile both on Windows or on Linux. Just download the pre-compiled binary from Mono. It contains only portable .Net binaries.

    sudo apt install p7zip-full
    wget http://download.mono-project.com/repo/ubuntu/pool/main/m/msbuild/msbuild_16.6+xamarinxplat.2021.01.15.16.11-0xamarin1+ubuntu2004b1_all.deb
    7z x msbuild_16.6+xamarinxplat.2021.01.15.16.11-0xamarin1+ubuntu2004b1_all.deb
    tar xf data.tar
    cp -R usr/* mono-6.12.0.122-bin/data/data/com.termux/files/usr/local/
    rm -rf data.tar usr

Modify the path of msbuild command.

    sed -i 's|/usr/bin|/data/data/com.termux/files/usr/local/bin|g' mono-6.12.0.122-bin/data/data/com.termux/files/usr/local/bin/msbuild
    sed -i 's|/usr/lib|/data/data/com.termux/files/usr/local/lib|g' mono-6.12.0.122-bin/data/data/com.termux/files/usr/local/bin/msbuild

<del>

Roslyn is buggy on Mono on arm64 as it assumes the legacy x86/x86_64 memory model which guarantees memory access order in some situations on the metal.

* https://preshing.com/20120930/weak-vs-strong-memory-models/
* https://github.com/dotnet/roslyn/issues/24932
* https://github.com/mono/mono/issues/12632
* https://xamarin.github.io/bugzilla-archives/56/56546/bug.html
* https://news.ycombinator.com/item?id=14318877

This problem can not really be fixed in Mono, as there is a huge performance penalty. To workaround this problem, we have to force Roslyn to run single-threaded.

    sed -i 's|"@(Compile)"|"@(Compile);/parallel-"|g' mono-6.12.0.122-bin/data/data/com.termux/files/usr/local/lib/mono/msbuild/Current/bin/Roslyn/Microsoft.CSharp.Core.targets
    sed -i 's|"@(Compile)"|"@(Compile);/parallel-"|g' mono-6.12.0.122-bin/data/data/com.termux/files/usr/local/lib/mono/msbuild/Current/bin/Roslyn/Microsoft.VisualBasic.Core.targets

</del>

In spite of the huge performance penalty, mono has [implemented](https://github.com/mono/mono/pull/17136) legacy x86/x86_64 memory model (total store order). The preceding patch is not necessary.

## Transfer to device

Pack all files.

    cd mono-6.12.0.122-bin/data/data/com.termux/files/usr
    tar cfJ mono-termux.6.12.0.122-arm64-android24.tar.xz local

Transfer file to /data/data/com.termux/files/usr on device.

Extract files on device.

    cd /data/data/com.termux/files/usr
    tar xf mono-termux.6.12.0.122-arm64-android24.tar.xz

Add usr/local/bin to system environment variable PATH if it isn't already there.

    echo export PATH=/data/data/com.termux/files/usr/local/bin:$PATH >> ~/.bash_profile

    # only do this if you use a version built for Api Level < 24
    # echo export LD_LIBRARY_PATH=/data/data/com.termux/files/usr/local/lib:$LD_LIBRARY_PATH >> ~/.bash_profile

## Build AOT cache for system libraries

For our binaries to run faster, we can build AOT cache for some system libraries on device.

    cd /data/data/com.termux/files/usr/local/lib/mono/4.5
    mono --aot csc.exe
    mono --aot mcs.exe
    mono --aot Microsoft.CodeAnalysis.CSharp.dll
    mono --aot Microsoft.CodeAnalysis.dll
    mono --aot mscorlib.dll
    mono --aot System.Collections.Immutable.dll
    mono --aot System.Reflection.Metadata.dll

## SSL certificate store

To use SSL, a certificate store is required.

Android provides a system certificate store at `/system/etc/security/cacerts` with read-only access.

Mono can use Android system certificate store, but requires Mono for Android (Xamarin Android).

But we can also use a separate certificate store like on Linux. Just copy `/usr/share/.mono` from a Linux device with Mono installed to `~/.config/.mono` on the Android device.
