---
title: riscv call conversion速览
tags:
  - riscv
  - 计算机体系结构
description: 本文是对riscv call conversion的一个速记，其中会反汇编简单的代码，然后分析 和call conversion相关的地方。
abbrlink: 48273
date: 2023-04-18 21:18:32
categories:
---

基本逻辑
---------

riscv call conversion的协议在[这里](https://github.com/riscv-non-isa/riscv-elf-psabi-doc/releases/download/v1.0/riscv-abi.pdf)。所谓call conversion是指函数调用过程中二进制
接口的定义。如果一个程序都相同的编译器构建，自然二进制的接口是一样，但是，程序难免
要使用各种库，这样程序构建的时候就是直接链接库的二进制，双方的二进制接口就要对齐。

其实，只要是不同的二进制程序之间有相互关联都存在二进制接口对齐的问题，必须定义相关
的准则，比如一个内核模块和内核之间的ABI问题，在一个内核版本上编译好的内核模块在不
同的内核上可能就不能使用，这就需要发行版(这里讨论的是Linux的发行版)在一定版本内
ABI是要做到兼容的，比如redhat的发行版在大版本是一定会做到ABI兼容的。

call conversion定义的就是函数调用之间的准则，比如函数调用的时候怎么传递函数入参，
怎么保存寄存器等等。下面的章节逐一介绍下。

寄存器相关的约定
-----------------

函数调用到了新函数，要使用通用寄存器进行计算，但是这些寄存器上还有发起调用函数的
数据，必须把通用寄存器上的值先保存起来，等新函数使用完寄存器后，返回发起调用函数
执行后续代码的时候再恢复回来原先寄存器上的值。这就有两种办法，一种是发起调用的函
数负责恢复和保存寄存器，一种是在被调函数里使用寄存器之前先保存旧值，使用完再把把
寄存器的旧值恢复回去，一般CPU都会把整个通用寄存器(GPR)分成这两种寄存器，前者叫作
caller saved registers, 后者叫做callee saved registers。

我们先看caller saved registers，从被调函数看，它可以放心的使用caller saved registers，
因为调用函数已经做了保存，调用函数负责在调用被调函数之前保存caller saved registers，
调用函数并不需要保存所有的caller saved registers，它只要保存自己使用的caller saved
registers。

被调函数只需要保存以及恢复它使用的callee saved registers就好，对于callee saved
registers需要先保存再使用。

riscv定义的caller saved register有ra、t0-t6、a0-a7，定义的callee saved register
有sp、s0-s11，浮点也有对应的caller/callee saved registers。t0-t6、a0-a7和s0-s11
的属性是直接定义的，但是ra和sp的属性是天然自带的。ra保存函数的返回地址，涉及到的
指令是: jal rd, offset或者jalr rd, offset(rs1)，比如用jal ra, offset实现函数调用，
jal把pc跳到pc + offset，并把jal的下一条指令的地址保存到ra，子函数要靠ra返回到父
函数的调用点，同理父函数的ra保存的是父函数的返回地址，所以在父函数在调用子函数之
前要保存父函数自己的返回地址。sp是栈指针，进入函数首先就要开栈，为函数的临时变量
准备存储空间，离开函数之前退栈，撤销函数的临时变量存储空间。

riscv定义的函数参数传递方式是使用a0-a7作为入参，使用a0-a1传递返回值，如果函数入参
寄存器放不下，使用调用函数栈传递参数，多余的参数从栈顶到栈底依此排列。

写个小程序，然后反汇编后对照的看下：
```
int add(int a, int b)
{
	return a + b;
}

int main()
{
	int a = 1, b = 2, c = 5;

	c += add(a, b);

	return c;
}

0000000000000612 <main>:
 612:	1101                	addi	sp,sp,-32    开栈
 614:	ec06                	sd	ra,24(sp)    ra是caller save，下面638行会覆盖ra，所以这里比如caller save
 616:	e822                	sd	s0,16(sp)    s0 callee save, 保存完就可以用
 618:	1000                	addi	s0,sp,32     main函数上下文使用s0作为帧指针
 61a:	4785                	li	a5,1
 61c:	fef42223          	sw	a5,-28(s0)
 620:	4789                	li	a5,2
 622:	fef42423          	sw	a5,-24(s0)
 626:	4795                	li	a5,5
 628:	fef42623          	sw	a5,-20(s0)   a/b/c逐个入栈，其实没有必要。还有很多没有必要的入栈
 62c:	fe842703          	lw	a4,-24(s0)
 630:	fe442783          	lw	a5,-28(s0)
 634:	85ba                	mv	a1,a4        准备add入参
 636:	853e                	mv	a0,a5
 638:	fb3ff0ef          	jal	ra,5ea <add> 函数调用
 63c:	87aa                	mv	a5,a0        函数返回值a0
 63e:	873e                	mv	a4,a5
 640:	fec42783          	lw	a5,-20(s0)   从栈上读到c
 644:	9fb9                	addw	a5,a5,a4     c和函数返回值做加法
 646:	fef42623          	sw	a5,-20(s0)
 64a:	fec42783          	lw	a5,-20(s0)
 64e:	853e                	mv	a0,a5        准备main的返回值
 650:	60e2                	ld	ra,24(sp)    恢复caller save的ra，为ret做准备
 652:	6442                	ld	s0,16(sp)    恢复callee save的s0
 654:	6105                	addi	sp,sp,32     退栈
 656:	8082                	ret                  函数返回

00000000000005ea <add>:
 5ea:	1101                	addi	sp,sp,-32    开栈
 5ec:	ec22                	sd	s0,24(sp)    callee save, 然后在函数上下文才能用s0做帧指针
 5ee:	1000                	addi	s0,sp,32     上面保存了s0，所以这里把s0用做帧指针
 5f0:	87aa                	mv	a5,a0        
 5f2:	872e                	mv	a4,a1
 5f4:	fef42623          	sw	a5,-20(s0)
 5f8:	87ba                	mv	a5,a4
 5fa:	fef42423          	sw	a5,-24(s0)
 5fe:	fec42703          	lw	a4,-20(s0)
 602:	fe842783          	lw	a5,-24(s0)
 606:	9fb9                	addw	a5,a5,a4
 608:	2781                	sext.w	a5,a5
 60a:	853e                	mv	a0,a5
 60c:	6462                	ld	s0,24(sp)    恢复callee save寄存器
 60e:	6105                	addi	sp,sp,32     退栈
 610:	8082                	ret                  函数返回
```

函数参数用栈传递的情况：
```
int add(int a, int b, int c, int d, int e, int f, int g, int h, int i, int j)
{
	return a + b + c + d + e + f + g + h + i + j;
}

int main()
{
	int a = 1, b = 2, c = 5;
	
	c += add(a, b, 3, 4, 5, 6, 7, 8, 9, 10);

	return c;
}

0000000000000680 <main>:
 680:	7179                	addi	sp,sp,-48
 682:	f406                	sd	ra,40(sp)
 684:	f022                	sd	s0,32(sp)
 686:	1800                	addi	s0,sp,48
 688:	4785                	li	a5,1
 68a:	fef42223          	sw	a5,-28(s0)
 68e:	4789                	li	a5,2
 690:	fef42423          	sw	a5,-24(s0)
 694:	4795                	li	a5,5
 696:	fef42623          	sw	a5,-20(s0)
 69a:	fe842583          	lw	a1,-24(s0)
 69e:	fe442503          	lw	a0,-28(s0)
 6a2:	47a9                	li	a5,10
 6a4:	e43e                	sd	a5,8(sp)
 6a6:	47a5                	li	a5,9
 6a8:	e03e                	sd	a5,0(sp)   <--- 可以看出超过入参寄存器的参数
 6aa:	48a1                	li	a7,8            放到调用函数(caller)栈一开始的位置
 6ac:	481d                	li	a6,7
 6ae:	4799                	li	a5,6
 6b0:	4715                	li	a4,5
 6b2:	4691                	li	a3,4
 6b4:	460d                	li	a2,3
 6b6:	f35ff0ef          	jal	ra,5ea <add>
 6ba:	87aa                	mv	a5,a0
 6bc:	873e                	mv	a4,a5
 6be:	fec42783          	lw	a5,-20(s0)
 [...]
```
