---
title: ARM构架下原子操作相关指令总结
tags:
  - ARM64
  - 计算机体系结构
description: 本文总结ARM架构下和原子操作相关的一些指令，后续相关内容都总结到这里。 涉及到代码分析时，使用的内核版本是v6.5-rc3。
abbrlink: 51246
date: 2023-08-21 18:45:44
categories:
---

LDXR/STXR
----------

ARM提供load-reserved/store-conditional的方式实现原子操作，其中相关的一组指令有：
LDXR/STXR/LDXRB/STXRB/LDXRH/STXRH/LDXP/STXP，这些指令基本功能一样，只是操作的地址
位宽不一样，XR是寄存器位宽，XRB是8bit，XRH是16bit，XP是两个寄存器位宽。load-acquire/
store-release这种barrier属性也可以加到这个指令上，其中前者加入A，后者加入L，比如
LDXR/STXR加了load-acquire/store-release属性的指令是LDAXR/STLXR。

ARM里叫这种指令是load-exclusive/store-exclusive的指令，其实和RISCV上的
load-reserved/store-conditional指令是基本一样的逻辑，学术界最开始提出这种指令时，
用的也是LR/SC(load-reserved/store-conditional)这样的叫法。

ARM的这组指令的用法和RISCV的LR/SC指令的逻辑是一样的，基本逻辑我们可以大概参考RISCV
LR/SC的逻辑，[这里](https://wangzhou.github.io/riscv原子指令分析/)是一个RISCV LR/SC的细节分析。

ARM上使用LDXR/STXR的例子可以查看内核的这个位置：linux/arch/arm64/include/asm/atomic_ll_sc.h
和其它体系结构类似，ARM上的各种原子操作都可以用这两个指令实现，其中就包括了普通
的原子运算和CAS(compare and swap)的原子操作。

ARM上LDXR和WFE结合还有一个巧妙的用法，LDXR会设置一个exclusive的标记，当有其它store
操作把这个标记去掉时，ARM规定系统会触发event register的set操作，这个操作是解开WFE
的其中一个信号。这样，一个core反复读一个地址直到读到特定值的条件读操作可以使用
LDXR+WFE的方式实现，内核里的核心实现代码如下，我们把注释直接写在下面的代码里。
```
/* linux/arch/arm64/include/asm/cmpxchg.h */
#define __CMPWAIT_CASE(w, sfx, sz)					\
static inline void __cmpwait_case_##sz(volatile void *ptr,		\
				       unsigned long val)		\
{									\
	unsigned long tmp;						\
									\
	asm volatile(							\

	/* sevl和紧接着的wfe去掉之前可能已经set的event register */

	"	sevl\n"							\
	"	wfe\n"							\

	/* 读检测地址上的数据，并把exclusive标记配置上 */

	"	ldxr" #sfx "\t%" #w "[tmp], %[v]\n"			\

	/* 检测下读到的值，如果是等待的值，就跳出循环 */

	"	eor	%" #w "[tmp], %" #w "[tmp], %" #w "[val]\n"	\
	"	cbnz	%" #w "[tmp], 1f\n"				\

	/*
	 * 否则睡眠等待, 当其它core写被检测的地址时，会触发这个地址的global monitor
	 * 被清理，从何触发WFE wakeup event唤醒WFE。注意，这里并不需要stxr之类的
	 * 条件写指令，只要普通store指令就可以触发global monitor被清理。
	 */

	"	wfe\n"							\
	"1:"								\
	: [tmp] "=&r" (tmp), [v] "+Q" (*(u##sz *)ptr)			\
	: [val] "r" (val));						\
}
```
在循环中反复调用如上的检测逻辑即可以完成条件读的功能，条件读在qspinlock的实现里
有大量的使用。

LSE(Large System Extensions)中的原子指令
-----------------------------------------

ARMv8.1的LSE里新增加了单指令的原子操作，大概分为：1. 对内存里的值做原子运算的指令；
2. 把一个值和内存里的值做交换的指令；3. CAS原子指令。

第一类指令数量庞大，命名逻辑是LD/ST + 运算种类，LD前缀表示有返回值，返回值是内存
里的旧值，ST表示没有返回值。比如LDADD x1, x2, [x3]表示，把x2和x3地址上值相加的和
写入x3地址，x3地址上的旧值写入x1，STADD x1, [x3]只有原子加这个操作，没有返回旧值。

第二类指令有: SWP/SWPB/SWPH，把输入寄存器里的值写入内存，并把内存上的旧值写到输出
寄存器。

第三类指令有：CAS/CASB/CASH/CASP，这些就是经典的CAS原子指令，比如CAS Xs, Xt, [Xn]
的功能是比较Xs和Xn内存上的值，如果相等就把Xt的值写入到Xn内存上。需要注意的是如上
的CASP指令，其中的P是pair的意思，虽然这个指令的寄存器还是Xs/Xt/Xn，但是它语意上
扩展到了128bit，逻辑是比较Xs Xs+1和Xn为基地值的128bit的值，如果相等就把Xs Xs+1的
值写入Xn为基地址的128bit内存上。这里用64bit寄存器举例介绍了，32bit寄存器的CASP指令
是一样的逻辑。

Linux内核里ARM上LSE实现的原子操作的代码在这里: linux/arch/arm64/include/asm/atomic_lse.h
可以看出基本上就是对相应LSE原子指令的封装。

Single copy 64B load/store
---------------------------

ARMv8.7引入的这组指令支持原子的对MMIO读写64Byte数据，相关的指令有：LD64B/ST64B/
ST64BV/ST64BV0，这些指令只能操作Non-cacheable或者device的地址空间，带V后缀的表示
有返回值，带0的表示可以工作在EL0。举一个例子看下具体使用的逻辑，比如ST64B Xt [Xn]
表示把Xt-X(t+7)这八个64bit寄存器里的值写入Xn为基地值的64Byte地址空间。

这组指令主要用来原子给设备的MMIO空间发请求，地址用来唯一的标识设备，64Byte的数据
是请求参数。因为地址是设备的MMIO空间，所以这个指令的功能其实是设备自定义的，广义
上看，设备的side effect都是这个指令的功能。

WFI/WFE/SEV/SEVL
-----------------

如上提到了WFE指令，这里把相关的指令都展开看下。 ARM里一开始只有WFI，顾名思义WFI
会使core进入低功耗模式，直到core上有中断，才唤醒core继续运行。

WFE(wait for event)是睡眠core等待event发生，这里的event的定义就有很多了，WFE里还
定义了一个event register的概念，这个是一个虚拟register，只能由特定的指令或硬件行为
set。spec里规定set event register的行为有：SEV/SEVL指令，前者set全部PE，后者set
当前PE；异常返回；清理PE的global monitor; Generic Timer event stream。event register
只能由WFE指令清理掉。

event register如果没有set，WFE正常进入低功耗状态，event register如果已经被set，
WFE指令并不能触发core进入低功耗，而且event register会被清理掉。可以看到LDXR/STXR
这一节提到使用sevl + wfe清理掉之前可能set的event register，这里不管之前event register
有没有set，这样操作总能把event register清理掉。

唤醒WFE的evnet有: SEV指令; 当前core收到并可以处理中断；asynchronous external request
debug event; timer event stream; global monitor被清理掉。其中最后一种event就是我们
上面在LDXR/STXR一节中提到的情况。

如上，SEV(send Event)和SEVL(send Event Local)，都可以set event register，只不过
作用core的范围不一样。SEV还可以一把唤醒所有core上WFE，如果一个core已经WFE了，语意
上就不可能再执行SEVL唤醒，所以可以看到spec里也没有相关的定义。

可以看到一个core WFE睡眠后被唤醒，触发其唤醒的事件可能不是它期望的，所以，被唤醒
的core需要执行检测代码，看看是否自己等待的事件发生，如果不是被自己等待的事件唤醒，
就需要再次执行WFE去睡眠。

PRFM
-----

CASP指令的示例代码里提到了PRFM，这里把预取相关的指令都展开看下。PRFM完成对地址上
数据的预取，具体硬件可以把它实现成提前加载数据到cache，这个指令可以带不同的参数。

customized instruction
-----------------------

ARM支持其它厂商可以自定义指令，相关的预留指令空间的定义可以查看ARM用户手册的这个
位置：Reserved encoding space for IMPLEMENTATION DEFINED instructions
