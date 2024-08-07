---
title: C语言嵌入ARM64汇编
tags:
  - ARM64
  - 汇编
  - C语言
description: >-
  本文介绍在C语言嵌入汇编语言的方法，linux kernel里有很多这样的例子。这里在aarch64
  平台下介绍，所有例子都是这个平台下的。网上的这篇文章已经有很好的介绍，大家可以 参阅：
  http://www.ethernut.de/en/documents/arm-inline-asm.html,
  http://ericw.ca/notes/a-tiny-guide-to-gcc-inline-assembly.html
  https://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html
abbrlink: c8d7dd82
date: 2021-07-05 22:34:15
categories:
---

使用场景
---------

   嵌入汇编一般用在c语言无法使用的地方，比如要操作系统寄存器了，比如要直接调用
   汇编指令完成功能了(内存屏障，刷cache), 还有就是要优化性能。Linux内核里和硬件
   直接接触的体系架构相关的代码有很多是直接用汇编写的。
   

嵌入汇编的语法
---------------

   asm volatile(code : output operand list : input operand list: clobber list);

   其中volatile告诉编译器不要去优化这里的汇编代码。防止你辛苦写的汇编代码一下被
   编译器优化的面目全非。output/input的定义是针对C语言里的变量，在汇编里是输入
   还是输出，下面的input/output operand中，汇编里的变量都和C里的变量有绑定，这
   很容易看出汇编里是只读某个C变量(input)，还是要对一个C变量做修改(output)。

   有时你会看到这样的嵌入汇编代码：
   ```
   asm("mov %[result], %[value], ror #1" : [result] "=r" (y) : [value] "r" (x) :)
   ```
   但是一般的，比如在linux内核里你看到的是这样的：
   ```
   asm("mov %0, %1, ror #1" : "=r" (result) : "r" (value));
   ```
   唯一不同的是，code里和operand里参数的对应关系的写法。我们这里用第二种方法，
   它的对应关系是，从output operand list到input operand list, 参数的命名从0开始，
   依次是%0，%1...

   operand list的一般格式是："常量" (参数)。参数是和C语言里变量的接口，比如：
   ```
	static inline void __raw_writel(u32 val, volatile void __iomem *addr)
	{
		asm volatile("str %w0, [%1]" : : "rZ" (val), "r" (addr));
	}
   ```
   这里的val和addr就是C代码里的变量。常量的意义可以参考:
   https://gcc.gnu.org/onlinedocs/gcc/Constraints.html, 简单讲这个常量可以是r,
   Q, m等，也可以再次被=，+， &等修饰。r, m, Q讲的是这个变量可以被编译器编译在
   寄存器里，内存里，或者是通过指针再间接寻址下，=表示这个变量的老值可以被覆盖，
   +表示这个变量可读可写。这里code里的w表示这个变量的位宽，w表示是32bit的，x表示
   是64bit的。其中=/+不可能出现在input operand list里，因为它们都有写的语意。这里
   再次注意，我们现在讨论的是aarch64下的汇编。

   可以随便写一个程序，aarch64-linux-gnu-gcc -S xx.c编译下，看看生成的汇编代码
   是怎么样的：
   ```
	void test()
	{
	        int y = 0;
		int x = 1;
	
		asm volatile("add %w0, %w1, 1" : "=r" (y) : "r" (x) :);
	}
   ```
   下面是编译成的汇编代码：
   ```
		.arch armv8-a
		.file	"embedded_asm.c"
		.text
		.align	2
		.global	test
		.type	test, %function
	test:
		sub	sp, sp, #16
		str	wzr, [sp, 12]
		mov	w0, 1
		str	w0, [sp, 8]
		ldr	w0, [sp, 8]
	#APP
	// 6 "embedded_asm.c" 1
		add w0, w0, 1
	// 0 "" 2
	#NO_APP
		str	w0, [sp, 12]
		nop
		add	sp, sp, 16
		ret
		.size	test, .-test
		.ident	"GCC: (Linaro GCC 6.3-2017.05) 6.3.1 20170404"
		.section	.note.GNU-stack,"",@progbits
   ```
   把r改成m后是这样的：
   ```
	test:
		sub	sp, sp, #16
		str	wzr, [sp, 12]
		mov	w0, 1
		str	w0, [sp, 8]
	#APP
	// 6 "embedded_asm.c" 1
		add [sp, 12], [sp, 8], 1
	// 0 "" 2
	#NO_APP
		nop
		add	sp, sp, 16
		ret
   ```
   把r改成Q后是这样的：
   ```
	   test:
		sub	sp, sp, #16
		str	wzr, [sp, 12]
		mov	w0, 1
		str	w0, [sp, 8]
		add	x0, sp, 12
		add	x1, sp, 8
	#APP
	// 6 "embedded_asm.c" 1
		add [x0], [x1], 1
	// 0 "" 2
	#NO_APP
		nop
		add	sp, sp, 16
		ret
   ```
   
   clobber的解释gcc上说是：告知编译器嵌入的汇编代码除了影响output operand里的
   变量外，还对哪些寄存器或者是对内存会产生影响。把他们可以列到clobber list里面
   来。寄存器直接列名字，两个特殊的clobber参数是"memory"和"cc", cc表示会影响到
   flags register，memory就会影响到input operant参数指向的内存。

举例
-----

   ...
