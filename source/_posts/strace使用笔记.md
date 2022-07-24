---
title: strace使用笔记
tags:
  - 软件调试
description: stracestrace a.out 可以输出a.out中依次调用的系统调用，和gdb一样strace使 用系统调用pstrace实现其功能。
abbrlink: e222383f
date: 2021-07-17 11:23:29
categories:
---

# 基本功能：

1. strace ./a.out 依次显示各个系统调用

2. strace -c ./a.out 可以打印出a.out中各个系统调用的使用次数和时间等信息

3. strace -ff -o strace_log ./a.out 可以把输出的结果存在strace_log.<pid>中

4. strace -f 可以在fork之后跟踪各进程内的系统调用

5. strace -i ls 显示系统调用时的instruction pointer

6. strace -r|-t|-tt|-T 现实相对时间戳、绝对时间、微秒时间和系统调用消耗时间

7. strace -e 可以跟踪指定的事件(strace -e open ./a.out)

8. strace -p <pid> 可以对指定进程进行跟踪

9. strace -O <**ms> 可以把**ms的时间认为是strace的开销，strace在统计时间的时
  候会减去**ms时间，这个时间一般是由程序运行时间和strace跟踪情况下程序的运
  行时间相减得到, 是个经验值
