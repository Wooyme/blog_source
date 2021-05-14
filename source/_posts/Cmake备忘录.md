---
title: Cmake备忘录
date: 2019-11-04 12:26:39
tags:
    - C/C++
    - Cmake
---
# JNI
```
include_directories(
        /PATH/TO/JDK/include
)
```
其他头文件同理
例：
```
cmake_minimum_required(VERSION 3.14)
project(JNI)

set(CMAKE_CXX_STANDARD 14)
include_directories(
        /usr/local/graalvm/19.1.1/include
)
add_library(JNI SHARED library.cpp JNI.h)
```

# linker flags
```
set (CMAKE_SHARED_LINKER_FLAGS "-lpthread")
```


