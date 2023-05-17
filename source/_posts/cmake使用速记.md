---
title: cmake使用速记
tags:
  - cmake
  - 编译链接
description: >-
  本文是学习cmake的一个笔记。测试的平台是Ubuntu 20.04，arm64，使用纯C代码测试。
  相关的测试代码放在：https://github.com/wangzhou/tests/cmake。
abbrlink: 32593
date: 2023-05-11 09:23:05
categories:
---


cmake基本逻辑
-------------

cmake根据cmake的配置文件生成Makefile, 然后用户可以用make命令进行软件的构建。
我们先从软件构件的角度自己想下cmake要描述怎么样的场景，然后一一看看cmake的描述
方法。我们需要：

1. 指定要编译生成的二进制文件，以及生成这些二进制文件的源文件。这些二进制文件可以
   是可执行文件，静态库和动态库。

2. 指定编译参数，指定链接参数。链接参数里要可以指定要链接的库。

3. 支持多目录层次的构建。

4. 支持Makfile里伪目标的创建，比如，make clean/install/uninstall等。

5. 支持系统环境的检测，类似autoconfigure/automake里的功能。

6. 支持环境参数的定义，以及条件构建。

7. cmake的基本用法。

描述二进制文件的构建
--------------------
```
project(hello_demo)

add_executable(hello hello.c hello.h)

add_subdirectory(lib)

target_link_libraries(hello add)
```
如上，在工程目录里的CMakeLists.txt文件里描述如何构建。project指定工程的名字，
add_executable指定要编译的二进制文件和它对应的源文件，这里是hello和hello.c, hello.h。
add_subdirectory指定要包含的下一级构建的目录，比如我们这里加了一个lib的目录，里面
是一个实例的库，lib目录下同样要有CMakeLists.txt：
```
add_library(add SHARED add.c)
```
add_library指定要编译生成一个libadd的动态库，SHARED就是指定生成动态库，这个动态
库的源代码是add.c。

继续看顶层的CMakeLists.txt, target_link_libraries指定hello要链接libadd这个库。

如上是一个最简单的认识，那么cmake是怎么知道使用哪个编译器编译的呢？比如系统里有
默认的gcc编译器，之后又在其他地方安装了其他版本的gcc，通过什么控制cmake对编译器
的选择。首先，cmake将选择系统默认的gcc编译器，路径在/usr/bin，库的路径在/lib，
通过修改CMAKE_C_COMPILER或者CMAKE_CXX_COMPILER的配置，可以改变默认的编译器：
```
set(CMAKE_C_COMPILER "/opt/gcc/bin/gcc")
```
如上，会把编译器设定为/opt/gcc下的gcc。但是，链接还是标准路径的glibc库，如何把glibc
库也修改成自定义的库？

指定编译链接参数
----------------
```
set(CMAKE_C_FLAGS "-O2 -g")
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fprofiles-arcs -ftest-coverage")
```
如上，先设置编译参数是-O2 -g，然后在此基础上加上-fprofiles-arcs -ftest-coverage

下面来看链接一个第三方库的方法，比如hello.c里使用了一个numa_max_node()的函数，这个
函数是libnuma里的库函数。现在考虑把libnuma链接进来。
```
target_link_libraries(hello add numa)
```
如上，可以把-lnuma这个参数加到链接参数里。dpdk -L libnuma1可以发现，当前测试系统
上这个库的位置在/usr/lib/aarch64-linux-gnu/libnuma.so.1.0.0，所以，编译器使用默认
路径就可以搜索到。可以使用如下的方式把更多的链接参数配置给cmake:
```
set(CMAKE_EXE_LINKER_FLAGS "-L/opt -I/opt/include/")
```

多层次目录构建
--------------

如上，使用add_subdirectory()把下下一级目录加进来。
