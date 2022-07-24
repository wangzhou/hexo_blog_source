---
title: '程序configure, compile, install的逻辑'
tags:
  - 软件开发
description: '本文讲述linux环境下用户态程序的配置,编译和安装的整个逻辑过程。具体的细节需要 在另外的文章中说明'
abbrlink: d5d9f373
date: 2021-07-17 11:03:09
categories:
---

 拿到一个linux用户态程序的源码，到使用该程序分为一下几步：

 1. 配置(configure)，在源码目录下一般会有configure脚本文件，执行该文件可以检测
    源码需要的编译链接环境，然后相应的生成makefile文件。

    使用./configure --help可以得到该脚本的使用帮助, 一般的有一下几个参数：
    ./configure --host=*** --prefix=*** CC=*** LDFLAGS=*** LIBS=***
    其中--host指定编译生成的可执行文件的执行环境， --prefix指定make install
    的安装路径，CC指定用到的编译器，LDFLAGS指定链接时标准的库搜索路径之外的
    库搜索路径。

 2. 编译链接(make)
    根据Makefile文件中的配置，编译链接成可执行程序。
    
 3. 安装(make install)
    第一步中(configure)中--prefix会把程序的安装路径写入到makefile中。在这一步会
    依照该路径把相应的文件拷贝到相应的目录。
    一个程序可以就只有一个可执行文件。也可能除了可以执行文件外，还需要一些静态
    库或者是动态库的支持, 这时安装程序就包括把可执行文件和动态库文件拷贝到相应
    的目录。
