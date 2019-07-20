---
title: Building PCL for Android
date: 2019-04-05 07:14:06
category: wiki
tags:
  - Android
---

This is a short guide about building PCL for android using [pcl-superbuild](https://github.com/patmarion/pcl-superbuild).

Several problems have been solved (with minimal change to existing source code) to make sure that PCL can be successfully build.

However, there is **NO GUARANTEE** that the built binaries will actually run on Android.

## 1.1 Add 32-bit support for your system

The pcl-superbuild script is old. It requires legacy toolchains which does not have a 64-bit binary release.

Newer Linux distros (e.g. Ubuntu 18.04) do not include 32-bit support by default, thus you have to enable it manually.

```bash
sudo dpkg --add-architecture i386
sudo apt update
sudo apt install multiarch-support libc6:i386 libstdc++6:i386 zlib1g:i386
```

## 1.2 Download build tools

Building PCL requires Android NDK and cmake.

Recent releases of these tools are not compatible with pcl-superbuild. Use [ndk-r8d](https://dl.google.com/android/ndk/android-ndk-r8d-linux-x86.tar.bz2) and [cmake 2.8.12.2](https://github.com/Kitware/CMake/releases/download/v2.8.12.2/cmake-2.8.12.2-Linux-i386.tar.gz) instead.

## 1.3 Build PCL

Run the pcl-superbuild script to build PCL.

```bash
mkdir build && cd build
/path/to/cmake-2.8.12.2/bin/cmake -DBUILD_IOS_DEVICE:BOOL="OFF" ../
export ANDROID_NDK=/path/to/ndk-r8d
make
```

Note that you will encounter errors when running this script. Read the following solutions.

## 1.4 Manually download dependencies

The script may complain about a file hash mismatch after downloading eigen. That is because our cmake does not have openssl support, and unfortunately, the download link is redirected to HTTPS.

The quickest way to solve this is to download manually using `wget`, and run `make` again.

```bash
wget http://www.vtk.org/files/support/eigen-3.1.0-alpha1.tar.gz -O /path/to/pcl-superbuild/build/CMakeExternals/Download/eigen/eigen-3.1.0-alpha1.tar.gz
```

## 1.5 Compile errors

1. When compiling PCL, you will encounter GCC internal compiler error. The reason is not yet clear, but by replace `powf()` with `std::pow()` will prevent this error.
2. Before compiling VTK, you will encounter errors concerning its cmake script. Modify `${_gcc_version}` to `_gcc_version` will make it work.
3. When compiling VTK, GCC will complain about `std::ostream` cannot accept `std::ostream&` as argument of its overloaded operator `<<`.

See this [blog post](https://blog.csdn.net/LongZh_CN/article/details/61463911) for (1.) and (2.) patch details.

For (3.):

* Delete `vtkOStreamWrapper& operator << (ostream&);` in "/path/to/pcl-superbuild/build/CMakeExternals/Source/vtk/Common/Core/vtkOStreamWrapper.h"
* Delete `VTKOSTREAM_OPERATOR(ostream&);` in "/path/to/pcl-superbuild/build/CMakeExternals/Source/vtk/Common/Core/vtkOStreamWrapper.cxx"
* Delete `vtkWarningMacro("Replacing existsing mapping for attribute " << attributeName);` in "/path/to/pcl-superbuild/build/CMakeExternals/Source/vtk/Rendering/Core/vtkGenericVertexAttributeMapping.cxx"

Now you may successfully compile PCL and its dependencies.

