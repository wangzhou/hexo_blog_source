---
title: Linux动态链接模块间函数调用逻辑分析
tags:
  - 编译链接
description: 文本整理Linux动态链接时模块间函数调用的逻辑，分析基于RISCV构架。
abbrlink: 25503
date: 2023-09-18 18:39:24
categories:
---

本文使用如下的测试程序，观察动态链接时printf相关的链接逻辑实现。
```
#include <stdio.h>

static int aaa = 0x123456; 

int main()
{
	printf("hello: %d\n", aaa);
	return 0;
}
```

riscv64-linux-gnu-gcc test.c编译后，riscv64-linux-gnu-objdump -D ./a.out > test.s做反汇编。
test.s里关键的内容如下。main函数里对printf的调用是:
```
 646:	f0bff0ef          	jal	ra,550 <printf@plt>
```
printf@plt是.plt段的一段，整个.plt如下:
```
0000000000000520 <.plt>:
 520:	00002397          	auipc	t2,0x2
 524:	41c30333          	sub	t1,t1,t3
 528:	af03be03          	ld	t3,-1296(t2) # 2010 <__TMC_END__>
 52c:	fd430313          	addi	t1,t1,-44 # ffffffffffff9fd4 <__global_pointer$+0xffffffffffff77d4>
 530:	af038293          	addi	t0,t2,-1296
 534:	00135313          	srli	t1,t1,0x1
 538:	0082b283          	ld	t0,8(t0) # ffffffffffffd008 <__global_pointer$+0xffffffffffffa808>
 53c:	000e0067          	jr	t3 # 1a000 <__global_pointer$+0x17800>

0000000000000540 <__libc_start_main@plt>:
 540:	00002e17          	auipc	t3,0x2
 544:	ae0e3e03          	ld	t3,-1312(t3) # 2020 <__libc_start_main@GLIBC_2.27>
 548:	000e0367          	jalr	t1,t3
 54c:	00000013          	nop

0000000000000550 <printf@plt>:
 550:	00002e17          	auipc	t3,0x2
 554:	ad8e3e03          	ld	t3,-1320(t3) # 2028 <printf@GLIBC_2.27>
 558:	000e0367          	jalr	t1,t3
 55c:	00000013          	nop
```
我们依次看下printf@plt里的指令，auipc和ld计算地址后把0x2028的数据load到t3，这里0x2028
是.got里printf函数的地址，这个位置存放动态链接完成后，printf的地址，但是第一次运行
到这里，可以看到0x2028上的内容是0x520，这个地址正好是.plt的基地址，所以printf@plt
中的jalr跳转到.plt开始处。.plt开始处准备t0-t3的数据，并跳到t3地址上执行，依次看下
t0-t3上保存的数据，0x530处的指令得到t0为.got表的基地址，所以0x538得到.got + 8处的
数据，linkmap的地址？0x524处t1 = t1 - t3, 得到0x55c - 0x520的差值，0x52c处把这个
差值再减去44(0x2c)得到0x10，这个值是什么？t2的值是0x2520，_dl_runtime_resolve没有
用到，t3是.got基地址上的数据，ld在装载时会用_dl_runtime_resolve的地址覆盖.got的
这个位置，所以0x53c就是直接跳到_dl_runtime_resolve函数执行。

如下是反汇编.got的内容:
```
Disassembly of section .got:

0000000000002010 <__TMC_END__>:
    2010:	ffff                	0xffff
    2012:	ffff                	0xffff
    2014:	ffff                	0xffff
    2016:	ffff                	0xffff
	...
    2020:	0520                	addi	s0,sp,648
    2022:	0000                	unimp
    2024:	0000                	unimp
    2026:	0000                	unimp
    2028:	0520                	addi	s0,sp,648
    202a:	0000                	unimp
    202c:	0000                	unimp
    202e:	0000                	unimp
    2030:	1e10                	addi	a2,sp,816
	...
```

_dl_runtime_resolve在glibc里定义，具体位置在glibc/sysdeps/riscv/dl-trampoline.S，
这个函数是体系架构相关的，这里是riscv定义的地方。这个函数计算需要动态链接的函数
的地址，更新.got里对应的地址，然后把控制返回到.plt的对应域段，比如这里的printf@plt
域段，可见这里再次load到t3的值就是printf实际地址了。

需要注意的是，printf的入参被保存到a0-a7这样的参数寄存器上，而且没有保存恢复的动作，
所以这里.plt里两次过程调用都没有使用a0-a7寄存器，而是使用t0-t3这样的临时寄存器，
临时寄存器是caller save寄存器，只要过程调用后不再使用，caller就可以不必保存和恢复。

这里.plt里无法使用正常的使用a0-a7的过程调用，因为这里不是一个返回调用点后续指令的
调用行为。这里的调用从printf@plt进入，在0x558发生一圈调用后，又跳回到printf@plt
重复执行，而正常的调用应该返回到0x55c。

