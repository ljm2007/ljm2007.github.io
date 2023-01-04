---
title: arm64-v8a inject
date: 2023-01-04 12:02:43
tags: 注入so
categories: 注入so
top_img: https://bit.ly/3WkSvF9
cover: https://bit.ly/3WkSvF9
---

经过了几天的学习,终于研究出一些东西来,隔行如隔山,这点玩应研究这么久,为了使自己不会忘记 就有这个编笔记.网上的一些关于  android 注入 一般都基于ndk-build生成可执行文件然后用vscode写代码,我感觉 很麻烦,我希望可以android studio+cmake 来直接生成

## 一 创建Native c++ 项目

​     ![image-20230104121130136](https://s2.loli.net/2023/01/04/KfGaCVqMJ74gnE1.png)

​    android studio 帮我们创建好的项目,由于我只生成arm64-v8a的可执行文件 那么在build.gradle中添加代码如下

```
ndk{
            abiFilters "arm64-v8a"
   }
```

![image-20230104121440774](https://s2.loli.net/2023/01/04/68drjupgeTBGQVq.png)

对Cmakelists.txt 改写

![image-20230104124508692](https://s2.loli.net/2023/01/04/BRvdAWfrTu8Cq9Y.png)

```
cmake_minimum_required(VERSION 3.18.1) # 最低CMake版本要求

##################### ⬇Android设置⬇ #####################
#set(CMAKE_SYSTEM_NAME ANDROID) # 设置目标编译平台参数 Android
#set(CMAKE_SYSTEM_VERSION 28) # 系统版本
#set(ANDROID_PLATFORM 28) # 平台版本
#set(ANDROID_ABI arm64-v8a) # 设置目标构架 armeabi-v7a arm64-v8a x86 x86_64
##
## 由于ANDROID_ABI一次编译只能设置一种架构
## 因此该参数在build.sh脚本中单独设置 并且每种架构编译一次
##
#set(ANDROID_NDK /Users/hongqing/Library/Android/sdk/ndk/ollvm) # 设置ndk路径
#set(CMAKE_TOOLCHAIN_FILE /Users/hongqing/Library/Android/sdk/ndk/ollvm/build/cmake/android.toolchain.cmake) # 设置交叉编译链的cmake配置
##################### ⬆Android设置⬆ #####################

project(testptrace) # 工程名称 + 版本

##################### ⬇项目相关参数设置⬇ #####################
#set(CMAKE_CXX_STANDARD 17) # c++ 标准
#set(CMAKE_CXX_FLAGS "-fno-rtti -fno-exceptions -DNDEBUG -fvisibility=hidden -Wno-narrowing -fdeclspec -pthread -w -s -fexceptions -Wall -O3"
#) # 参数
##################### ⬆项目相关参数设置⬆ #####################

##################### ⬇子模块设置⬇ #####################
add_subdirectory(Inject)
add_subdirectory(Hook)
##################### ⬆子模块设置⬆ #####################
```

对Inject 和Hook 目录下的Cmakelists.txt 改写

![image-20230104124648967](https://s2.loli.net/2023/01/04/p5893OoXiGLjaD1.png)

```

##################### ⬇输出文件重定向⬇ #####################
#set(CMAKE_LIBRARY_OUTPUT_DIRECTORY
#    /${CMAKE_SOURCE_DIR}/outputs/${CMAKE_ANDROID_ARCH_ABI}/
#) # 重定向输出产物(动态库)
##################### ⬆输出文件重定向⬆ #####################


##################### ⬇CMake头文件设置⬇ #####################
FILE(GLOB_RECURSE FILE_INCLUDES # 遍历子目录下所有符合情况的头文件
    ${CMAKE_CURRENT_SOURCE_DIR}/*.h*
)
include_directories( # 设置全局头文件目录 使其他源码文件可在任意目录<头文件.h>
    ${CMAKE_CURRENT_SOURCE_DIR}/include/
)



##################### ⬆CMake头文件设置⬆ #####################


##################### ⬇CMake源文件设置⬇ #####################
FILE(GLOB_RECURSE FILE_SOURCES # 遍历子目录下所有符合情况的源文件
    ${CMAKE_CURRENT_SOURCE_DIR}/*.c*
)
##################### ⬆CMake源文件设置⬆ #####################




##################### ⬇添加产物⬇ #####################
add_library(hook SHARED # 生成动态库
    ${FILE_INCLUDES} # 头文件
    ${FILE_SOURCES} # 源文件
)
##################### ⬆添加产物 ⬆ #####################

##################### ⬇连接库文件⬇ #####################
target_link_libraries(hook PRIVATE
    log
)
##################### ⬆连接库文件⬆ #####################
```

```

##################### ⬇输出文件重定向⬇ #####################
#set(CMAKE_RUNTIME_OUTPUT_DIRECTORY
#    /${CMAKE_SOURCE_DIR}/outputs/${CMAKE_ANDROID_ARCH_ABI}/
#) # 重定向输出产物(可执行文件)
##################### ⬆输出文件重定向⬆ #####################


##################### ⬇CMake头文件设置⬇ #####################
FILE(GLOB_RECURSE FILE_INCLUDES # 遍历子目录下所有符合情况的头文件
    ${CMAKE_CURRENT_SOURCE_DIR}/*.h*
)
include_directories( # 设置全局头文件目录 使其他源码文件可在任意目录<头文件.h>
    ${CMAKE_CURRENT_SOURCE_DIR}/include/
    ${CMAKE_CURRENT_SOURCE_DIR}/include/Utils/
)
##################### ⬆CMake头文件设置⬆ #####################


##################### ⬇CMake源文件设置⬇ #####################
FILE(GLOB_RECURSE FILE_SOURCES # 遍历子目录下所有符合情况的源文件
    ${CMAKE_CURRENT_SOURCE_DIR}/*.c*
)
##################### ⬆CMake源文件设置⬆ #####################


##################### ⬇添加产物⬇ #####################
add_executable(Inject # 生成动态库
    ${FILE_INCLUDES} # 头文件
    ${FILE_SOURCES} # 源文件
)
##################### ⬆添加产物 ⬆ #####################
```

这是我在github上发现的源码,但需要cmake脚本来生成的可执行程序,我给改成android studio生成的

源码地址:https://github.com/SsageParuders/AndroidPtraceInject

生成后可以在build----->intermediates------>cmake----->debug------>obj------>arm64-v8a中生成两个文件

![image-20230104125357664](https://s2.loli.net/2023/01/04/wDqZYCHlBRgfzok.png)

项目地址:https://github.com/ljm2007/AndroidInject

