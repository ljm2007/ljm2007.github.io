---
title: 利用Cmake对NDK进行Android的交叉编译
date: 2022-11-11 18:18:40
tags: ndk
categories: ndk
top_img: https://s2.loli.net/2022/11/11/roHQRqmDbCuGS9J.png
cover: https://s2.loli.net/2022/11/11/roHQRqmDbCuGS9J.png	
---

 我是由一个windows系统开发,只会C++ MFC 编程同学转到 ndk编程,由于对这方面没什么经验,所以学习起来有点艰难,对Cmake的认识少之又少,用了一个星期的时间才把这个东西搞定.也就有了这编笔记

## 1.[下载NDK](https://developer.android.com/ndk/downloads) 

 在官网选择你对应系统的软件包,我的是kali 系统,所以我选择 linux版本的

![](https://s2.loli.net/2022/11/12/l5gHBbsomTtY6kz.png)

## 2.配置VS Code对C/C++的支持

这里我选择了[C/C++插件](https://github.com/microsoft/vscode-cpptools)，官方提供的，应该值得信赖。

![C/C++ plugin](https://s2.loli.net/2022/11/11/3FlRny62pVCjzIO.png)

Cmake支持

编译工具这里我选择了cmake，所以安装[Cmake](https://github.com/twxs/vs.language.cmake)和[Cmake Tools](https://github.com/microsoft/vscode-cmake-tools)这两个插件。

![Cmake plugin](https://s2.loli.net/2022/11/11/fk4P2chX8CKp6dT.png)

Cmake插件是让VS Code支持Cmake语言。

![Cmake Tools plugin](https://s2.loli.net/2022/11/11/eVUjGqYCPmtHkIs.png)

而Cmake Tools插件则是能让VS提供各种Cmake编译相关的小工具，包括在底部状态栏显示一些快捷工具。

安装上述三个插件后，重启VS Code让插件生效。

------

##    4. [下载CMake](https://cmake.org/download/) 

- ​    下表介绍在将CMake和NDK搭配使用时，可以配置的部分变量

|               编译参数               |                             说明                             |
| :----------------------------------: | :----------------------------------------------------------: |
|         **ANDROID_PLATFORM**         | **指定目标Android平台的名称，如android-18指定Android 4.3(API级别18)** |
|             ANDROID_STL              |             指定CMake应使用的STL，默认c++_static             |
|             ANDROID_PIE              | 指定是否使用位置独立的可执行文件(PIE)。Android动态链接器在Android 4.1(API级别16)及更高级别上支持PIE，可设置为On、OFF |
|         ANDROID_CPP_FEATURES         | 指定CMake编译原生库时需使用的特定C++功能，可设置为rtti(运行时类型信息)、exceptions(C++异常) |
|   ANDROID_ALLOW_UNDEFINED_SYMBOLS    | 指定CMake在构建原生库时，如果遇到未定义的引用，是否会引发未定义的符号错误。默认FALSE |
|           ANDROID_ARM_NEON           | 指定CMake是否应构建支持NEON的原生库。API级别为23或更高级别时，默认值为true，否则为false |
| ANDROID_DISABLE_FORMAT_STRING_CHECKS | 指定是否在编译源代码时保护格式字符串。启用保护后，如果在printf样式函数中使用非常量格式字符串，则编译器会引发错误。默认false |

下表介绍在Android进行交叉编译时，可以使用的具体构建参数，将有助于调试CMake构建问题：

|            编译参数            |                             说明                             |
| :----------------------------: | :----------------------------------------------------------: |
|        **ANDROID_ABI**         | **目标ABI，可设置为armeabi-v7a、arm64-v8a、x86、x86_64，默认armeabi** |
|        **ANDROID_NDK**         |                **安装的NDK根目录的绝对路径**                 |
|    **CMAKE_TOOLCHAIN_FILE**    | **进行交叉编译的android.toolchain.cmake文件的路径，默认在$NDK/build/cmake/目录** |
|       ANDROID_TOOLCHAIN        |             CMake使用的编译器工具链，默认为clang             |
|        CMAKE_BUILD_TYPE        |             配置构建类型，可设置为Release、Debug             |
|    ANDROID_NATIVE_API_LEVEL    |                CMake进行编译的Android API级别                |
| CMAKE_LIBRARY_OUTPUT_DIRECTORY |       构建LIBRARY目标文件之后，CMake存放这些文件的位置       |

看着编译的变量和参数都不少，那如何来抉择呢？

其实，一般情况下，只需要配置`ANDROID_ABI`、`ANDROID_NDK`、`CMAKE_TOOLCHAIN_FILE`、`ANDROID_PLATFORM`四个变量即可。

**ANDROID_ABI是CPU架构，ANDROID_NDK是NDK的根目录，CMAKE_TOOLCHAIN_FILE是工具链文件，ANDROID_PLATFORM是支持的最低Android平台**。

为什么指定ANDROID_PLATFORM？如果不指定，Android平台版本较低，此时ANDROID_PIE默认为OFF，可执行程序无法执行。

1.我用的是kaili系统,直接用命令下载 

```shell
sudo apt-get install cmake
```

##   5.创建工程 

![](https://s2.loli.net/2022/11/11/z5hJnad9U2VueKM.png)

## 6. 编写CMakeLists.txt

```shell
cmake_minimum_required(VERSION 3.24.2)

project(camketest)

if(CMAKE_SOURCE_DIR STREQUAL CMAKE_BINARY_DIR)
    message(FATAL_ERROR
      "\n"
      "In-source builds are not allowed.\n"
      "Instead, provide a path to build tree like so:\n"
      "cmake -S . -B <destination>\n"
      "\n"
      "To remove files you accidentally created execute:\n"
      "please delete CMakeFiles and CMakeCache.txt\n"
    )
endif()

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
add_subdirectory(src)
```

在src目录里也要创建一个CMakeLists.txt

```shell
add_executable(main main.cpp)
```

## 7.编写build.sh自动化脚本

```shell
#/bin/bash

export ANDROID_NDK=/opt/env/android-ndk-r14b

rm -r build
mkdir build && cd build 

cmake -DCMAKE_TOOLCHAIN_FILE=$ANDROID_NDK/build/cmake/android.toolchain.cmake \
	-DANDROID_ABI="armeabi-v7a" \
	-DANDROID_NDK=$ANDROID_NDK \
	-DANDROID_PLATFORM=android-22 \
	..

make && make install

cd ..
```



##   8. 测试代码 

​     在src目录里面创建main.cpp 并写入测试代码

```c++
#include <iostream>

#include<sys/ptrace.h>
int main(int argc,const char * argv[])
{
  std::cout<<"Hello cpp:"<<__cplusplus<<std::endl;
}

```

## 9.用 adb push 输入手机测试

```shell
adb push test /data/local/tmp/
```

## 10.参考资料

【Cmake】利用NDK进行Android的交叉编译（附实例）https://blog.csdn.net/qq_38410730/article/details/103622813

【CMake】CMakeLists.txt的超傻瓜手把手教程（附实例源码）https://blog.csdn.net/qq_38410730/article/details/102477162

VS Code + Cmake Tools, 搭建C/C++跨平台（NDK、iOS）开发环境 https://jay-dh.github.io/blog/vscode-cmake-cross-compile

Using Visual Studio Code as an Android C++ editor https://donaldmunro.github.io/VSCode-Android-CC/
