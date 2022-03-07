---
title: qemu user mode速记
date: 2021-12-25 23:56:24
tags: [虚拟化, qemu]
description: "本文是qemu user mode的一个速记，简单分析相关的代码、构建以及使用方法。
      具体以riscv为例说明。"
categories:
---

基本逻辑
--------

qemu有两种的使用方法，一种是system mode，一种是user mode。前者模拟整个machine，
其上可以运行一个完成的guest OS，后者可以在host上运行一个guest的程序，这个时候他
通过tcg用软件模拟guest CPU的状态，当guest程序里有系统调用的时候，user mode会直接
在host上模拟系统调用。

配置编译：configure --target-list=riscv64-linux-user; make
在build下就会编译生成相关的命令行工具：qemu-riscv64

用这个工具就可以直接运行riscv64的程序：qemu-riscv64 riscv64_app

代码分析
--------

user mode的代码在qemu/linux-user/*，相关tcg的代码在qemu/tcg/*、qemu/accel/*，
qemu/target/riscv/*。

```
/* qemu/linux-user/main.c */
main
  -> cpu_loop
       /* 这个函数里就是tcg相关的翻译执行过程了 */
    -> cpu_exec
       /* 指令执行遇到tcg处理不了的，就落到swith里执行 */
    -> swith (trapnr)
         /* 具体看下遇到系统调用的处理办法 */
      -> RISCV_EXCP_U_ECALL
           /* qemu/linux-user/syscall.c */
        -> do_syscall
	     /* 可以看到这里把所有的系统调用都模拟了下 */
	  -> do_syscall1
```

使用方法
--------

使用qemu-riscv --help可以查到全部参数设定，进一步用qemu-riscv -d --help可以看debug
log的配置。其中-d可以配置打印出中间码、guest和host反汇编、guest cpu的寄存器等值，
这些东西在调试tcg代码相关代码的时候比较有用，-singlestep可以对guest的汇编逐条打印
出调试信息。这里可以看一个 -d cpu,op,in_asm的输出log。
```
qemu-riscv64 -singlestep -d cpu,op,in_asm ~/a.out &> ~/log

[...]
IN: main
0x0000000000010430:  1101              addi            sp,sp,-32

OP:
 ld_i32 tmp0,env,$0xfffffffffffffff8
 brcond_i32 tmp0,$0x0,lt,$L0

 ---- 0000000000010430
 mov_i64 tmp2,x2/sp
 add_i64 tmp2,tmp2,$0xffffffffffffffe0
 mov_i64 x2/sp,tmp2
 mov_i64 pc,$0x10432
 call lookup_tb_ptr,$0x6,$1,tmp2,env
 goto_ptr tmp2
 set_label $L0
 exit_tb $0xffff980ab5c3

 pc       0000000000010430
 x0/zero 0000000000000000 x1/ra 0000000000010606 x2/sp 0000004000800370 x3/gp 0000000000071028
 x4/tp 0000000000072710 x5/t0 0000000000072000 x6/t1 2f2f2f2f2f2f2f2f x7/t2 0000000000072000
 x8/s0 0000000000010940 x9/s1 00000000000109d0 x10/a0 0000000000000001 x11/a1 00000040008004b8
 x12/a2 00000040008004c8 x13/a3 0000000000000000 x14/a4 0000004000800398 x15/a5 0000000000010430
 x16/a6 0000000000071138 x17/a7 0112702f5b5a4001 x18/s2 0000000000000000 x19/s3 0000000000000000
 x20/s4 0000000000000000 x21/s5 0000000000000000 x22/s6 0000000000000000 x23/s7 0000000000000000
 x24/s8 0000000000000000 x25/s9 0000000000000000 x26/s10 0000000000000000 x27/s11 0000000000000000
 x28/t3 ffffffffffffffff x29/t4 000000000006ead0 x30/t5 0000000000000000 x31/t6 0000000000072000
----------------
IN: main
0x0000000000010432:  ec06              sd              ra,24(sp)

OP:
 ld_i32 tmp0,env,$0xfffffffffffffff8
 brcond_i32 tmp0,$0x0,lt,$L0

 ---- 0000000000010432
 mov_i64 tmp2,x2/sp
 add_i64 tmp2,tmp2,$0x18
 mov_i64 tmp3,x1/ra
 qemu_st_i64 tmp3,tmp2,leq,0
 mov_i64 pc,$0x10434
 call lookup_tb_ptr,$0x6,$1,tmp2,env
 goto_ptr tmp2
 set_label $L0
 exit_tb $0xffff980ab703

 pc       0000000000010432
 x0/zero 0000000000000000 x1/ra 0000000000010606 x2/sp 0000004000800350 x3/gp 0000000000071028
 x4/tp 0000000000072710 x5/t0 0000000000072000 x6/t1 2f2f2f2f2f2f2f2f x7/t2 0000000000072000
 x8/s0 0000000000010940 x9/s1 00000000000109d0 x10/a0 0000000000000001 x11/a1 00000040008004b8
 x12/a2 00000040008004c8 x13/a3 0000000000000000 x14/a4 0000004000800398 x15/a5 0000000000010430
 x16/a6 0000000000071138 x17/a7 0112702f5b5a4001 x18/s2 0000000000000000 x19/s3 0000000000000000
 x20/s4 0000000000000000 x21/s5 0000000000000000 x22/s6 0000000000000000 x23/s7 0000000000000000
 x24/s8 0000000000000000 x25/s9 0000000000000000 x26/s10 0000000000000000 x27/s11 0000000000000000
 x28/t3 ffffffffffffffff x29/t4 000000000006ead0 x30/t5 0000000000000000 x31/t6 0000000000072000
[...]
```
IN: main表示当前在guest app的main里，下面是guest汇编代码，OP是tcg中间码，pc是这个
指令执行后guest寄存器里的值。
