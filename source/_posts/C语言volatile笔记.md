---
title: C语言volatile笔记
tags:
  - C语言
description: 本文介绍C语言里的volatile关键词的使用方法。
abbrlink: b7523820
date: 2021-07-05 22:35:27
categories:
---

C语言里volatile用来修饰一个变量，告诉编译器怎么去编译一个变量。编译器编译一个变量，
看起来应该是很直白的：一个变量在内存里存放，读的话就读对应的内存，写这个变量的
话就向这个内存写值就可以了，为什么还要volatile这个词来修饰下？

在这之前我们要搞清楚两个问题.

第一，代码要编译成汇编指令才可以执行，编译成的汇编指令可能不是从内存里取这个值，
当然也可能写内存的时候不会立即生效。注意我这里只提到了内存，没有说cache，这里我
假设代码运行的机器是可以在硬件层面保证内存/cache的数据一致性的，就是说cache在软
件层面是透明的。其实现在大部分机器也确实是这样的。

第二，读写一个值的时候，不要认为它对应的物理存储一定是内存(DDR), 它还可以是设备
的一段IO寄存器(MMIO), 比如，我把一个物理设备的一段IO寄存器直接mmap出来，你在
用户态用mmap的到一个地址，你访问这个地址，其实访问的是这个设备的寄存器。IO寄存器
的属性和内存的完全不一样，你写下去一个值是为了触发一个操作，你再写下去一个相同的
值是为了再触发一次这个操作, 第二个操作是不能代替第一个操作的；硬件还可以通过IO
寄存器给你返回状态，也就是一个“内存”里的值，可能自己会改变 :)

还有一种情况和上面的情况相似，就是多个独立的执行流改变一个变量的情况。当一个写
操作已经改变了内存里的值的时候，如果读代码是去读一个缓存的值，这样就会出错。再次
强调这里不包括cache和内存不一致的问题，也就是说，我们假设如果写已经更新了内存，
读的时候，即使读的是对应的cache，硬件也会识别到cache里的数据和内存的数据是不一样
的，硬件会自动把内存里的新数据填到cache里，之后读是可以读到正确的值。那什么时候
会出问题呢: 当读操作把数据缓存在寄存器里的时候。

基于以上的认识，我们看一段代码：
```
/* for test 1 */
int test;

/* for test 2 */
volatile int test_v;

/* for test 3 */
int test_3, tmp1, tmp2, tmp3;


void volatile_test_1()
{
	test = 111;	
	test = 222;
	test = 333;
}

void volatile_test_2()
{
	test_v = 111;	
	test_v = 222;
	test_v = 333;
}

void volatile_test_3_read()
{
	test = test_3;	
	tmp1 = test_3;
	tmp2 = test_3;
	tmp3 = test_3;
}

void volatile_test_3_write()
{
	test_3 = 111;	
}
```

对这段代码普通的情况下，编译成的汇编代码是：(使用x86-arm64 gcc交叉编译器)
```
	.arch armv8-a
	.file	"test.c"
	.comm	test,4,4
	.comm	test_v,4,4
	.comm	test_3,4,4
	.comm	tmp1,4,4
	.comm	tmp2,4,4
	.comm	tmp3,4,4
	.text
	.align	2
	.global	volatile_test_1
	.type	volatile_test_1, %function
volatile_test_1:
	adrp	x0, test
	add	x0, x0, :lo12:test
	mov	w1, 111
	str	w1, [x0]
	adrp	x0, test
	add	x0, x0, :lo12:test
	mov	w1, 222
	str	w1, [x0]
	adrp	x0, test
	add	x0, x0, :lo12:test
	mov	w1, 333
	str	w1, [x0]
	nop
	ret
	.size	volatile_test_1, .-volatile_test_1
	.align	2
	.global	volatile_test_2
	.type	volatile_test_2, %function
volatile_test_2:
	adrp	x0, test_v
	add	x0, x0, :lo12:test_v
	mov	w1, 111
	str	w1, [x0]
	adrp	x0, test_v
	add	x0, x0, :lo12:test_v
	mov	w1, 222
	str	w1, [x0]
	adrp	x0, test_v
	add	x0, x0, :lo12:test_v
	mov	w1, 333
	str	w1, [x0]
	nop
	ret
	.size	volatile_test_2, .-volatile_test_2
	.align	2
	.global	volatile_test_3_read
	.type	volatile_test_3_read, %function
volatile_test_3_read:
	adrp	x0, test_3
	add	x0, x0, :lo12:test_3
	ldr	w1, [x0]
	adrp	x0, test
	add	x0, x0, :lo12:test
	str	w1, [x0]
	adrp	x0, test_3
	add	x0, x0, :lo12:test_3
	ldr	w1, [x0]
	adrp	x0, tmp1
	add	x0, x0, :lo12:tmp1
	str	w1, [x0]
	adrp	x0, test_3
	add	x0, x0, :lo12:test_3
	ldr	w1, [x0]
	adrp	x0, tmp2
	add	x0, x0, :lo12:tmp2
	str	w1, [x0]
	adrp	x0, test_3
	add	x0, x0, :lo12:test_3
	ldr	w1, [x0]
	adrp	x0, tmp3
	add	x0, x0, :lo12:tmp3
	str	w1, [x0]
	nop
	ret
	.size	volatile_test_3_read, .-volatile_test_3_read
	.align	2
	.global	volatile_test_3_write
	.type	volatile_test_3_write, %function
volatile_test_3_write:
	adrp	x0, test_3
	add	x0, x0, :lo12:test_3
	mov	w1, 111
	str	w1, [x0]
	nop
	ret
	.size	volatile_test_3_write, .-volatile_test_3_write
	.ident	"GCC: (Linaro GCC 6.3-2017.05) 6.3.1 20170404"
	.section	.note.GNU-stack,"",@progbits

```

但是加上-O3的选项，叫编译器帮忙给我优化下编译结果，最后的代码是：
```
	.arch armv8-a
	.file	"test.c"
	.text
	.align	2
	.p2align 3,,7
	.global	volatile_test_1
	.type	volatile_test_1, %function
volatile_test_1:
	adrp	x0, test
	mov	w1, 333
	str	w1, [x0, #:lo12:test]
	ret
	.size	volatile_test_1, .-volatile_test_1
	.align	2
	.p2align 3,,7
	.global	volatile_test_2
	.type	volatile_test_2, %function
volatile_test_2:
	adrp	x0, test_v
	mov	w3, 111
	mov	w2, 222
	mov	w1, 333
	str	w3, [x0, #:lo12:test_v]
	str	w2, [x0, #:lo12:test_v]
	str	w1, [x0, #:lo12:test_v]
	ret
	.size	volatile_test_2, .-volatile_test_2
	.align	2
	.p2align 3,,7
	.global	volatile_test_3_read
	.type	volatile_test_3_read, %function
volatile_test_3_read:
	adrp	x0, test_3
	adrp	x4, test
	adrp	x3, tmp1
	adrp	x2, tmp2
	adrp	x1, tmp3
	ldr	w0, [x0, #:lo12:test_3]
	str	w0, [x4, #:lo12:test]
	str	w0, [x3, #:lo12:tmp1]
	str	w0, [x2, #:lo12:tmp2]
	str	w0, [x1, #:lo12:tmp3]
	ret
	.size	volatile_test_3_read, .-volatile_test_3_read
	.align	2
	.p2align 3,,7
	.global	volatile_test_3_write
	.type	volatile_test_3_write, %function
volatile_test_3_write:
	adrp	x0, test_3
	mov	w1, 111
	str	w1, [x0, #:lo12:test_3]
	ret
	.size	volatile_test_3_write, .-volatile_test_3_write
	.comm	tmp3,4,4
	.comm	tmp2,4,4
	.comm	tmp1,4,4
	.comm	test_3,4,4
	.comm	test_v,4,4
	.comm	test,4,4
	.ident	"GCC: (Linaro GCC 6.3-2017.05) 6.3.1 20170404"
	.section	.note.GNU-stack,"",@progbits
```

可以看出普通编译的时候都是没有问题的。问题出现在编译器做优化的时候:

test_1多次写一个变量的时候, 编译器认为，只要最后一次写就可以了。如果，test1对应
的是一个IO寄存器，或者test_1的值，会被另一个执行流程识别，这里就会出问题。解决
办法就是给test_1加上volatile，告诉编译器每次写操作都不能被编译器优化掉。

test_3多次读一个变量的时候，编译器后两次的读都没有去正真读内存，而是使用了第一次
缓存在寄存器里的值。如果，test_3对应的是一个IO寄存器，或者如代码里一样test_3在
另一个执行流里被修改了(volatile_test_3_write), 但是read操作还是读的寄存器里的值，
这样就会出问题。要修改也是加volatile。

在上面的第一里提到，写内存的时候不是马上生效的, 这里也和cache没有啥关系，在硬件
保证cache一致性的系统里，对于单个操作，可以认为是写内存就马上生效的。这里说的
不生效是指CPU的多个写操作写下去的写完成的先后是不保证的。要想保证一个写操作的先后
顺序，就需要使用内存屏障指令。这个不是本问讨论的问题, 另外再说。这里只是说明，
volatile只是告诉编译器不做额外优化，它是不能保证写生效顺序的。
