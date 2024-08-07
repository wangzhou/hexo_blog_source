---
title: gdb用法收集
tags:
  - 软件调试
  - gdb
description: 一直没有系统的看看gdb的使用方法，用这个文档持续记录下gdb的使用技巧。
abbrlink: 5a0482f1
date: 2022-01-26 23:24:34
categories:
---

最简单的用法就是在本地用gdb, gdb + 被调试的程序，注意被调试程序的参数要通过
set args xxx这样传给gdb。

我们启动一个实际的调试看看，这里的调试程序选qemu-riscv64，这个是一个qemu的用户态
模式命令行工具，用他可以跨平台的执行用户态程序，它的代码在qemu/linux-user下。
```
sherlock@m1:~/repos/qemu/build$ gdb qemu-riscv64 
GNU gdb (Ubuntu 9.2-0ubuntu1~20.04) 9.2
Copyright (C) 2020 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "aarch64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from qemu-riscv64...
warning: File "/home/sherlock/repos/qemu/.gdbinit" auto-loading has been declined by your `auto-load safe-path' set to "$debugdir:$datadir/auto-load".
To enable execution of this file add
        add-auto-load-safe-path /home/sherlock/repos/qemu/.gdbinit
line to your configuration file "/home/sherlock/.gdbinit".
To completely disable this security protection add
        set auto-load safe-path /
line to your configuration file "/home/sherlock/.gdbinit".
For more information about this security protection see the
--Type <RET> for more, q to quit, c to continue without paging--
"Auto-loading safe path" section in the GDB manual.  E.g., run from the shell:
        info "(gdb)Auto-loading safe path"
(gdb) set args ~/a.out
(gdb) b tcg_gen_code
Breakpoint 1 at 0x179e68: file ../tcg/tcg.c, line 4159.
(gdb) r
Starting program: /home/sherlock/repos/qemu/build/qemu-riscv64 ~/a.out
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/aarch64-linux-gnu/libthread_db.so.1".
[New Thread 0xfffff7971150 (LWP 548388)]

Thread 1 "qemu-riscv64" hit Breakpoint 1, tcg_gen_code (s=0xaaaaaae88e88 <tcg_init_ctx>, tb=tb@entry=0xffffe8000080 <code_gen_buffer+52>)
    at ../tcg/tcg.c:4159
warning: Source file is more recent than executable.
4159    {
(gdb) bt
#0  tcg_gen_code (s=0xaaaaaae88e88 <tcg_init_ctx>, tb=tb@entry=0xffffe8000080 <code_gen_buffer+52>) at ../tcg/tcg.c:4159
#1  0x0000aaaaaac1a3b4 in tb_gen_code (cpu=cpu@entry=0xaaaaaaec7d90, pc=pc@entry=66376, cs_base=cs_base@entry=0, flags=flags@entry=24832, cflags=0)
    at ../accel/tcg/translate-all.c:1496
#2  0x0000aaaaaac563fc in cpu_exec (cpu=cpu@entry=0xaaaaaaec7d90) at ../accel/tcg/cpu-exec.c:945
#3  0x0000aaaaaab5e224 in cpu_loop (env=0xaaaaaaed0110) at ../linux-user/riscv/cpu_loop.c:37
#4  0x0000aaaaaab484b8 in main (argc=<optimized out>, argv=<optimized out>, envp=<optimized out>) at ../linux-user/main.c:885
(gdb)
```
如上，使用gdb qemu-riscv64启动调试，set args ~/a.out设置qemu-riscv64的参数，b tcg_gen_code
在tcg_gen_code上打一个断点，r把程序跑起来，程序会在断点处停下来，bt可以把这个时候
的调用栈打出来。c可以叫程序继续跑起来，一直到下一个断点处停下来。

我们也可以用.gdbinit把gdb启动后需要执行的命令先写到这个配置文件里，gdb启动后就会
自动调用。.gdbinit可以放到自己的home目录作为默认配置文件，也可以放到运行gdb的当前
目录，.gdbinit里可以用file把要跑的命令加上，用set args把命令的参数加上：
```
file ./qemu_path/qemu-system-riscv
set args -m 2048M  \
	qemu parameters \
break main
run
```

打断点的方式还有很多变种：

1. b file:line      文件名 + 文件行号
2. b line           直接用行号
3. b function       直接用函数名
4. b file:function  文件名 + 函数名
5. b xxxx if cond   满足后面的条件才会断住程序

info命令可以查看gdb的信息，比如info b查看当前所有设定的断点，info registers查看
寄存器的值。

delete number可以删去对应的断点，这里的number就是info b里显示的断点的编号。

gdb可以单步执行程序，或者跳入跳出函数。用n单步执行，这个是以函数为单位的，要想
调入函数执行需要用s，退出函数回到上层执行用finish。如果单步执行到一个循环里，可以
用u直接把循环一次都执行完。在gdb输入enter会重复输入上次的命令。

用p可以打出参数的值，如果是一个地址，可以p *arg打印地址上的数据，对于静态数组可以
直接用"p 数组名"打印数组成员，对于动态申请的内存可以"p 地址@size"打印这个地址开始
的长度为size的数据，p file::variable、p function::variable可以给变量限定作用域。

watch cond可以监控变量/表达式/地址上的值有没有改变，只要改变，程序就会暂停并且打
印被监控对象的新值和旧值。注意，对于一个指针，比如，int *p, watch p是监控指针变量，
watch *p是监控指针指向的地址。

p打印格式, p/x 十六进制。
