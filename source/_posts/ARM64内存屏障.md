---
title: ARM64内存屏障
tags:
  - 计算机体系结构
  - ARM64
  - 内存屏障
description: >-
  本文试图梳理aarch64构架下的内存屏障的逻辑，细节的东西还要去看ARMv8的手册。其实， 《ARM Cortex-A Series
  Programmer’s Guide for ARMv8-A》这本书的第13章，memory order已经对内存屏障的内容做了比较入门的讲解。
abbrlink: 26672f2c
date: 2021-07-05 22:27:54
categories:
---

 要理解内存屏障，要知道memory的一些特性，这里的memory不是只指内存。而是从CPU角度
 看到的存储空间。如果CPU对一系列的指令执行是严格串行的，我们是不用外加上面内存
 屏障的。比如：

 	str x1 [x2]   // A
	str x2 [x3]   // B

 A指令完全执行完，B指令才执行。这样根本不需要内存屏障指令的介入。

 但是现在处理器和内存系统之间的速度差距已经非常大，如果要上面的A执行完成才执行B
 cpu要消耗大量的时间干等在那里。为了缓解CPU和内存系统间的速度差距，同时也为了不断
 提升CPU的效率，现代CPU存在很多指令执行上的技术，导致的结果是指令执行的结果和之前
 的逻辑已经不一样了。

 CPU上存在着多种指令执行的技术，其中导致需要加上内存屏障执行的是CPU上的指令乱序
 执行。

 为了解释清楚CPU执行乱序执行需要引入两个基本的概念，一个是内存类型(memory type),
 另外一个是master/slave.
 
 内存类型定义的CPU和内存相互作用时的一些性质，内存类型只有normal和device两种,
 normal基本上可以对应DDR，device基本上可以对应设备的MMIO. normal的内存可以配置成
 cacheable和non-cacheable, 其中cacheable的内存属性又可以进一步配置shareability的属性。
 device的内存的属性使用GRE表述，G是gather，R是re-order, E是write early aknowledge,
 每种属性可以是nX(e.g. nG), 就是不支持的意思。关于normal和device下面的诸多属性,
 可以查阅本文开始时提到的ARM编程手册13章中的内容。

 master和slave是芯片里面的概念，master是一个动作的发起者，slave是一个动作的接收
 者。一般的一个芯片里，master有各个core，以及外设的DMA控制器。slave有内存和MMIO.

 基于以上的概念，我们看内存屏障为啥有存在的意义。master(CPU, 不确定DMA是不是这样)
 对normal内存的操作，存在很多提升效率的处理(多发射，乱序执行，执行预取...).
 我们回到乱序执行上来。ARMv8的CPU对指令乱序执行的几条规则是这样的:

 1. 在单一的core上，指令完成的顺序是串行的。注意，master收到slave的(有时不一定是
    slave, 比如，device内存的E属性)完成信息为一个执行完成。

 2. 在单一的core上下发的指令，slave上完成的顺序是无法保证串行的。

 3. 多个master上看到的操作是不保序的。

 这样就会导致:

 在一个core上:

     等待A条件出现
     A条件出现后执行B动作

  如果B动作先被执行，则可能出错。所以要在之间加上内存屏障：

     等待A条件出现
     内存屏障
     A条件出现后执行B动作

  在多个core上:
  
     core 1           core 2

     DMA              check flag
     set flag         get date from DMA range
      
  core1的逻辑是先用DMA搬数据，搬完数据后设立一个标记位。core2不断的在检测标记
  位，当检测到标记位的时候，core2就可以使用DMA搬好的数据了。

  但是，core1的DMA和set flag两个操作可能是乱序执行的，可能在core2看来flags已经
  置位了，但是其实DMA的数据还没有搬完, 这时，如果core2去使用数据，就有可能出错。
  正确的做法是在DMA和set flag执行加上内存屏障，确保DMA完成了，再去set flag.
  注意，这里其实就是上面的第三点。另外，这里和cache一致性没有关系, 如果，我们
  单独看dma，或者单独看flag，两者各自都是硬件保证cache一致性的。

  具体来说ARMv8上的内存屏障执行有ISB，DMB，DSB。ISB和指令相关，后面两个和内存
  访问相关。具体的区别可以查文章开头提到的书。

  具体驱动代码, 可以参考下arm64下几组read/write的实现，代码在arch/arm64/include/asm/io.h.
  可以看到同一个write函数有一下三种类型:

	__raw_writeb
	writeb_relaxed(v,c)
	writeb(v,c)	

  第一个是str指令的封装，第二个用于I/O内存也是str的封装，第三个可以用于normal的
  内存读写，其实是真是write之前加了一个DSB的内存屏障指令。

  未完待续...
