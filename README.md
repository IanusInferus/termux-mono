# How to install Mono and MSBuild inside Termux

## Description

[Termux](https://github.com/termux/termux-app) is an Android terminal app and Linux environment.

[Mono](https://www.mono-project.com) is an implementation of .Net Framework on various platforms, including Android.

[MSBuild](https://github.com/microsoft/msbuild) is the default build system in Mono (and .Net).

Go to the last section if you want to use the prebuilt binaries.

## Build Mono from source

There is no pre-compiled binary for Mono on Termux now, except for prooted Arch Linux and Ubuntu. ([40](https://github.com/termux/termux-packages/issues/40))

Here I describe a process to cross-compile Mono 6.10 from source on Linux.

Termux now defaults to Android 7.0 (API Level 24, /data/data/com.termux/files/usr/include/android/api-level.h). I cross-compile on Ubuntu(on WSL) and target Android 7.0.

    # at first, make sure you have mono installed on the host machine
    # https://www.mono-project.com/download/stable/#download-lin

    sudo apt install wget perl python3 cmake unzip git ninja-build g++

    # get Android NDK from https://developer.android.com/ndk/downloads for Linux
    # say android-ndk-r21d-linux-x86_64.zip
    unzip android-ndk-r21d-linux-x86_64.zip

    wget https://download.mono-project.com/sources/mono/mono-6.10.0.104.tar.xz

    # compile mono for PC with .Net Class Libraries
    mkdir mono-6.10.0.104-PC
    tar xf mono-6.10.0.104.tar.xz -C mono-6.10.0.104-PC --strip-components=1
    cd mono-6.10.0.104-PC
    export CC=
    export CXX=
    export LDFLAGS=
    ./configure
    make
    make install DESTDIR=~/mono-6.10.0.104-bin-PC
    cd ..

    # compile mono for Android without .Net Class Libraries
    tar xf mono-6.10.0.104.tar.xz
    cd mono-6.10.0.104
    # patch for log output (defaults to adb log rather than terminal)
    sed -i 's|#if HOST_ANDROID|#if 0|g' mono/eglib/goutput.c
    export CC="$(realpath ..)/android-ndk-r21d/toolchains/llvm/prebuilt/linux-x86_64/bin/clang --target=aarch64-linux-androideabi24 --sysroot=$(realpath ..)/android-ndk-r21d/toolchains/llvm/prebuilt/linux-x86_64/sysroot"
    export CXX="$(realpath ..)/android-ndk-r21d/toolchains/llvm/prebuilt/linux-x86_64/bin/clang --target=aarch64-linux-androideabi24 --sysroot=$(realpath ..)/android-ndk-r21d/toolchains/llvm/prebuilt/linux-x86_64/sysroot"
    export LDFLAGS="-lz -Wl,-rpath='\$\$ORIGIN/../lib'"
    ./configure --host=aarch64-linux-androideabi24 --prefix=/data/data/com.termux/files/usr/local --disable-mcs-build --with-btls-android-ndk="$(realpath ..)/android-ndk-r21d" --with-btls-android-api=24 --with-btls-android-cmake-toolchain="$(realpath ..)/android-ndk-r21d/build/cmake/android.toolchain.cmake"
    make
    make install DESTDIR=~/mono-6.10.0.104-bin
    cd ..

    # copy .Net Class Libraries
    cp -rf ~/mono-6.10.0.104-bin-PC/usr/local/lib/mono ~/mono-6.10.0.104-bin/data/data/com.termux/files/usr/local/lib/

    # Some libraries may use libc.so.6, which doesn't exist on Android. We can create a symbol link to libc<span></span>.so.
    ln -s /system/lib64/libc.so ~/mono-6.10.0.104-bin/data/data/com.termux/files/usr/local/lib/libc.so.6

## Install MSBuild

Don't try to build from source, it depends on a .Net Core version which is not shipped with Visual Studio and is very difficult to compile both on Windows or on Linux. Just download the pre-compiled binary from Mono. It contains only portable .Net binaries.

    sudo apt install p7zip-full
    wget http://download.mono-project.com/repo/ubuntu/pool/main/m/msbuild/msbuild_16.6+xamarinxplat.2020.04.29.14.43-0xamarin5+ubuntu2004b1_all.deb
    7z x msbuild_16.6+xamarinxplat.2020.04.29.14.43-0xamarin5+ubuntu2004b1_all.deb
    tar xf data.tar
    cp -R usr/* ~/mono-6.10.0.104-bin/data/data/com.termux/files/usr/local/
    rm -rf data.tar usr

Change the content of ~/mono-6.10.0.104-bin/data/data/com.termux/files/usr/local/bin/msbuild into

    #!/bin/sh
    MONO_GC_PARAMS="nursery-size=64m,$MONO_GC_PARAMS" exec /data/data/com.termux/files/usr/local/bin/mono --assembly-loader=strict $MONO_OPTIONS /data/data/com.termux/files/usr/local/lib/mono/msbuild/15.0/bin/MSBuild.dll "$@"

Roslyn is buggy on Mono on arm64 as it assumes the legacy x86/x86_64 memory model which guarantees memory access order in some situations on the metal.

* https://preshing.com/20120930/weak-vs-strong-memory-models/
* https://github.com/dotnet/roslyn/issues/24932
* https://github.com/mono/mono/issues/12632
* https://xamarin.github.io/bugzilla-archives/56/56546/bug.html
* https://news.ycombinator.com/item?id=14318877

This problem can not really be fixed in Mono, as there is a huge performance penalty. To workaround this problem, we have to force Roslyn to run single-threaded.

    sed -i 's|"@(Compile)"|"@(Compile);/parallel-"|g' ~/mono-6.10.0.104-bin/data/data/com.termux/files/usr/local/lib/mono/msbuild/Current/bin/Roslyn/Microsoft.CSharp.Core.targets
    sed -i 's|"@(Compile)"|"@(Compile);/parallel-"|g' ~/mono-6.10.0.104-bin/data/data/com.termux/files/usr/local/lib/mono/msbuild/Current/bin/Roslyn/Microsoft.VisualBasic.Core.targets

## Transfer to device

Pack all files.

    cd ~/mono-6.10.0.104-bin/data/data/com.termux/files/usr
    tar cfJ mono-termux.6.10.0.104.tar.xz local

Extract files on device.

    cd /data/data/com.termux/files/usr
    tar xf mono-termux.6.10.0.104.tar.xz

Add usr/local/bin to system environment variable PATH if it isn't already there.

    echo export PATH=/data/data/com.termux/files/usr/local/bin:$PATH >> ~/.bash_profile
