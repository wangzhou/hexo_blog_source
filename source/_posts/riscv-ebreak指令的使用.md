---
title: riscv ebreak指令的使用
abbrlink: 23091
date: 2022-07-25 15:46:36
tags: [riscv, ISA, 计算机体系结构]
description: "本文分析riscv ebreak指令在Linux内核中的使用，分析基于的代码时Linux主线v5.12-rc8，
      主要分析BUG_ON宏、kprobe、uaccess实现。"
categories:
---

ebreak指令
-----------

 riscv上的ebreak指令和ecall指令的功能基本一致，都是用指令触发一个异常，ebreak使用
 指令出发一个ebreak异常。

BUG_ON宏实现
------------

 BUG_ON()是内核里常用的一个断言，当不满足BUG_ON的条件时，内核就会打印出当前的CPU
 寄存器上下文以及相关的内核调用栈。

 riscv上BUG_ON内部实现是BUG()，它的定义在arch/riscv/include/asm/bug.h
```
#define BUG() do {						\
	__BUG_FLAGS(0);						\
	unreachable();						\
} while (0)

#define __BUG_FLAGS(flags)					\
do {								\
	__asm__ __volatile__ (					\
		"1:\n\t"					\
			"ebreak\n"				\
			".pushsection __bug_table,\"aw\"\n\t"	\
		"2:\n\t"					\
			__BUG_ENTRY "\n\t"			\
			".org 2b + %3\n\t"                      \
			".popsection"				\
		:						\
		: "i" (__FILE__), "i" (__LINE__),		\
		  "i" (flags),					\
		  "i" (sizeof(struct bug_entry)));              \
} while (0)
```
如上的.pushsection指示编译器把下面label2开头的一段二进制在链接的时候
放到名叫__bug_table的段里。

ebreak指令的执行最终会执行到ebreak的异常处理函数: do_trap_break，这个函数的定义在
linux/arch/riscv/kernel/traps.c，这个函数的实现包含了kprobe、uprobe、kgdb以及BUG
的逻辑。

BUG的逻辑实现在report_bug函数，核心的逻辑是使用异常PC在__bug_table这个段里查找对应
的信息，然后打印查找到的内容。

uaccess实现
------------

 riscv的对应实现在linux/arch/riscv/include/asm/uaccess.h，用户态内核态之间传递信息
 的逻辑基本上在uaccess里实现，比如，copy_from_user、copy_to_user等等。

 这里只看一个copy_to_user的实现:
```
 /* linux/include/linux/uaccess.h */
 copy_to_user
       /* linux/lib/usercopy.c */
   +-> _copy_to_user
         /* 下面的体系构架相关的函数在arch/riscv/include/asm/uaccess.h */
     +-> access_ok
     +-> raw_copy_to_user
           /* arch/riscv/lib/uaccess.S */
       +-> __asm_copy_to_user(void __user *to, const void *from, unsigned long n)
```
__asm_copy_to_user的代码分析如下，直接以注释的形式写在下面的代码之间:
```
ENTRY(__asm_copy_to_user)
ENTRY(__asm_copy_from_user)

	/* Enable access to user memory */
	li t6, SR_SUM
	csrs CSR_STATUS, t6

	/* 如上，a0是用户态目的地址，a1是内核buffer的地址, a2是要copy的size，a3是源数据buffer的结尾 */
	add a3, a1, a2
	/* Use word-oriented copy only if low-order bits match */
	/* 64bit的系统，SZREG是8，t0得到目的地址的低3bit，t1得到源地址的低3bit */
	andi t0, a0, SZREG-1
	andi t1, a1, SZREG-1
	/* 
         * 目的地址和源地址低3bit不相等的时候，跳到label 2，有数据要拷贝的时候，
         * 会直接跳到label 5, label 5本来是拷贝非8Byte数据的，但是，在源数据buffer
         * 和目的数据buffer地址不对应的情况，直接在label 5用单Byte拷贝的方式复制
         * 整段内存。
         */
	bne t0, t1, 2f

	/*
	 * 对于源数据buffer，下面每一段表示8B，那么t0和t1的志向的位置如下：
         *
	 * |  8B   |       |       |       |       |
	 * +-------+-------+-------+-------+-------+
	 * |   ^   |       |       |    ^  |       |
         *     |                        |
         *     +------------------------+
         *     |   ^               ^    | 源数据buffer
         *         |               |
         *     a1                       a3
         *         t0              t1
         *
         * 如下这块代码的逻辑是找到源数据buffer中整8Byte的部分，t0指向起始地址，
         * t1指向结尾地址。
	 */
	addi t0, a1, SZREG-1
	andi t1, a3, ~(SZREG-1)
	andi t0, t0, ~(SZREG-1)
	/*
	 * a3: terminal address of source region
	 * t0: lowest XLEN-aligned address in source
	 * t1: highest XLEN-aligned address in source
	 */
        /*
         * t0 >= t1是一些边角的情况，跳到label 2，有数据要拷贝的时候，再跳到label 5
         * 做单Byte拷贝。过了这个点后，下面是拷贝算法的主题逻辑。
         */
	bgeu t0, t1, 2f
        /* a1 < t0说明有t0前面多出来的那一截，跳到label 4，单Byte拷贝结尾这段非对齐数据 */
	bltu a1, t0, 4f
1:
	/*
	 * fixup宏的定义是：
	 *
	 *      .macro fixup op reg addr lbl
	 * 100:
	 *	\op \reg, \addr
	 *	.section __ex_table,"a"
	 *	.balign RISCV_SZPTR
	 *	RISCV_PTR 100b, \lbl
	 *	.previous
	 *	.endm
	 *
	 * 这个宏生成了一个指令，上下文看来，一般是生成load或者store指令，然后在
	 * __ex_table这个段生成一个dword大小(8B)的数据，这个数据的格式是低4B是之前
	 * 生成指令的地址，高4B是一个跳转的label。当load/store有异常发生的时候，
         * 异常处理函数在__ex_table中找到对应的地址，高32bit保存就是要继续跳转的
         * 地址，这里是继续跳转到label 10的地方，函数返回值配置成数据搬移的size，
         * 然后函数返回。
	 *
	 * 相关的流程是：各种访存异常处理函数(e.g. do_trap_load_misaligned)->do_trap_error->fixup_exception
	 * 在fixup_exception里配置regs->epc为fixup代码地址(label 10地址)，然后调用
	 * sret跳到对应地址执行。这里怎么和异常恢复结合起来？还会有什么异常？
	 */
	fixup REG_L, t2, (a1), 10f
	fixup REG_S, t2, (a0), 10f
	addi a1, a1, SZREG
	addi a0, a0, SZREG
	bltu a1, t1, 1b
2:
	bltu a1, a3, 5f

3:
	/* Disable access to user memory */
	csrc CSR_STATUS, t6
	li a0, 0
	ret
4: /* Edge case: unalignment */
	fixup lbu, t2, (a1), 10f
	fixup sb, t2, (a0), 10f
	addi a1, a1, 1
	addi a0, a0, 1
	bltu a1, t0, 4b
        /* 拷贝完前面一截非8Byte对齐的数据，跳到label 1以8Byte为单位复制数据 */
	j 1b
5: /* Edge case: remainder */
	fixup lbu, t2, (a1), 10f
	fixup sb, t2, (a0), 10f
	addi a1, a1, 1
	addi a0, a0, 1
	bltu a1, a3, 5b
	j 3b
ENDPROC(__asm_copy_to_user)
ENDPROC(__asm_copy_from_user)
EXPORT_SYMBOL(__asm_copy_to_user)
EXPORT_SYMBOL(__asm_copy_from_user)
[...]
10:
	/* Disable access to user memory */
	csrs CSR_STATUS, t6
	mv a0, a2
	ret
```

kprobe实现
-----------

 基本原理是插入一个ebreak指令，ebreak触发kprobe的调用。代码入口在do_trap_break。

GDB场景
--------

 执行到ebreak指令的时候，如果在用户态就向用户态发一个SIGTRAP的信号，如果在内核态
 并且使能KGDB，就处理相关逻辑。代码入口在do_trap_break。
