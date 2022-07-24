---
title: APUE学习笔记(第一章)
tags:
  - APUE
description: APUE第一章笔记，好久以前的事了
categories: read
abbrlink: 423f3a3d
date: 2021-07-17 10:48:13
---

1. 每个进程有一个工作目录，所有相对目录都从这个工作目录开始解释。可以用chdir改
   变这个工作目录。

2. 用户态程序出错处理：
   GNU c库中会定义一个errno的全局变量(ubuntu下在/usr/include/errno.h中)。函数
   发生错误的时候可以把错误值写入这个errno，用以指名是什么错误。比如read()错误
   时返回-1，我们去查errno的值，知道发生了什么错误。

   #include <string.h>
   char *strerror(int errnum); 输入errno，输出和errno相关的字符串的指针。

   #include <stdio.h>
   void perror(const char *msg);
   输入是一个自定义的字符串，输出输入的字符串 + "：" + errno对应的字符串。
