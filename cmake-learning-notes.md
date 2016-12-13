---
title: cmake学习笔记
date: 2016-09-12 15:27:01
categories: 语言
tags: [cmake, 编译]
---

在mac上
``` bash
brew install cmake
```

官方最简单的CMakeLists.txt文件:

``` bash
cmake_minimum_required (VERSION 2.6)
PROJECT (Tutorial)
SET(SRC_LIST hello.c)
ADD_EXECUTABLE(Tutorial ${SRC_LIST})
```



