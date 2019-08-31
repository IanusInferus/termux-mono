# How to install Mono and MSBuild inside Termux

## Description

[Termux](https://github.com/termux/termux-app) is an Android terminal app and Linux environment.

[Mono](https://www.mono-project.com) is an implementation of .Net Framework on various platforms, including Android.

[MSBuild](https://github.com/microsoft/msbuild) is the default build system in Mono (and .Net).

## Build Mono from source

There is no pre-compiled binary for Mono on Termux now, except for prooted Arch Linux and Ubuntu. ([40](https://github.com/termux/termux-packages/issues/40))

Here I describe a process to build Mono 6.0 from source on an Android arm64 device.

Termux now defaults to Android 7.0 (API Level 24, /data/data/com.termux/files/usr/include/android/api-level.h), so anything requires an API level higher than that need to be disabled. I compile on an Android 8.0 device personally. I have tried to convince autoconf to use -D__ANDROID_API__=24 in function checking (for existence of pthread_getname_np, ...) but failed, so on other Android versions, more patches may be needed if you experience linkage problems.

Also, it's not possible to build the class libraries on device as you need a runnable Mono or Monolite to bootstrap.

    pkg install wget perl python cmake clang
    wget https://download.mono-project.com/sources/mono/mono-6.0.0.319.tar.xz

    tar xf mono-6.0.0.319.tar.xz
    cd mono-6.0.0.319

    ./configure --prefix=/data/data/com.termux/files/usr/local --disable-mcs-build

    sed -i 's|#define HAVE_PTHREAD_GETNAME_NP 1|/\* #undef HAVE_PTHREAD_GETNAME_NP \*/|g' config.h
    sed -i 's|#define HAVE_GETPWENT 1|/\* #undef HAVE_GETPWENT \*/|g' config.h
    sed -i 's|#define HAVE_SETPWENT 1|/\* #undef HAVE_SETPWENT \*/|g' config.h
    sed -i 's|#define HAVE_LUTIMES 1|/\* #undef HAVE_LUTIMES \*/|g' config.h
    sed -i 's|#define HAVE_FUTIMES 1|/\* #undef HAVE_FUTIMES \*/|g' config.h
    sed -i 's|#define HAVE_GETDOMAINNAME 1|/\* #undef HAVE_GETDOMAINNAME \*/|g' config.h
    sed -i 's|#define HAVE_SETDOMAINNAME 1|/\* #undef HAVE_SETDOMAINNAME \*/|g' config.h

    sed -i 's|-lpthread|-lpthread -llog|g' mono/mini/Makefile
    sed -i 's|-lpthread|-lpthread -llog|g' mono/dis/Makefile
    sed -i 's|-lpthread|-lpthread -llog|g' tools/pedump/Makefile
    sed -i 's|\tgint64       st_atime_nsec;|#undef st_atime_nsec\n#undef st_mtime_nsec\n#undef st_ctime_nsec\n\tgint64       st_atime_nsec;|g' support/map.h
    sed -i 's|return L_cuserid;|return -1;|g' support/stdio.c

    make
    make install
    cd ..

Copy class libraries from Mono directory on a PC (/usr/local/lib/mono on Linux) to /data/data/com.termux/files/usr/local/lib/mono.

Add usr/local/bin to system environment variable PATH

    echo export PATH=/data/data/com.termux/files/usr/local/bin:$PATH >> ~/.bash_profile

Exit and reenter bash.

## Install MSBuild

Don't try to build from source, it depends on a .Net Core version which is not shipped with Visual Studio and is very difficult to compile both on Windows or on Linux. Just download the pre-compiled binary from Mono. It contains only portable .Net binaries.

    pkg install p7zip
    wget http://download.mono-project.com/repo/ubuntu/pool/main/m/msbuild/msbuild_16.1+xamarinxplat.2019.06.05.11.19-0xamarin9+ubuntu1804b1_all.deb
    7z x msbuild_16.1+xamarinxplat.2019.06.05.11.19-0xamarin9+ubuntu1804b1_all.deb
    tar xf data.tar
    cp -R usr/* /data/data/com.termux/files/usr/local/
    rm -rf data.tar usr

Change the content of /data/data/com.termux/files/usr/local/bin/msbuild into

    #!/bin/sh
    MONO_GC_PARAMS="nursery-size=64m,$MONO_GC_PARAMS" exec /data/data/com.termux/files/usr/local/bin/mono $MONO_OPTIONS /data/data/com.termux/files/usr/local/lib/mono/msbuild/15.0/bin/MSBuild.dll "$@"

Roslyn is buggy on Mono on arm64 as it assumes the legacy x86/x86_64 memory model which grants memory access order in some situations on the metal.

* https://preshing.com/20120930/weak-vs-strong-memory-models/
* https://github.com/dotnet/roslyn/issues/24932
* https://github.com/mono/mono/issues/12632
* https://xamarin.github.io/bugzilla-archives/56/56546/bug.html
* https://news.ycombinator.com/item?id=14318877

This problem can not really be fixed in Mono, as there is a huge performance penalty. To workaround this problem, we have to force Roslyn to run single-threaded.

    sed -i 's|"@(Compile)"|"@(Compile);/parallel-"|g' /data/data/com.termux/files/usr/local/lib/mono/msbuild/Current/bin/Roslyn/Microsoft.CSharp.Core.targets
    sed -i 's|"@(Compile)"|"@(Compile);/parallel-"|g' /data/data/com.termux/files/usr/local/lib/mono/msbuild/Current/bin/Roslyn/Microsoft.VisualBasic.Core.targets

## Backup and Recover

All these can be packed for later use.

    cd /data/data/com.termux/files/usr
    tar cfJ mono-termux.6.0.0.319.tar.xz local

To recover, execute the following commands and don't forget to add usr/local/bin to PATH.

    cd /data/data/com.termux/files/usr
    tar xf mono-termux.6.0.0.319.tar.xz
