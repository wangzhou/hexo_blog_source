---
title: Linux内核scatterlist用法
tags:
  - Linux内核
  - scatterlist
description: >-
  本文主要讲linux kernel里scatterlist的使用方法，有的设备驱动需要使用这个接口编程。 本文从设备的硬件结构，kernel
  scatterlist的原理，以及scatterlist API说明应该怎么 用这个接口,
  很多是自己的初步理解。下面的两篇文章都是讲scatterlist的，可以参考看下:
  http://www.wowotech.net/memory_management/scatterlist.html
  https://lwn.net/Articles/234617/ lwn的文章讲的已经比较好了，wowo的文章讲了数据结构，但是基本上没有把逻辑讲完整。
abbrlink: a94be268
date: 2021-07-05 22:33:36
categories:
---

为什么有scatterlist
---------------------

   这里说的scatterlist，说的是用聚散表管理的内存，这是一个一般的概念。这个概念主要
   是配合硬件产生的。从物理上讲，我们可以分配一段连续的内存使用，但是，我们也
   可以分配很多段不连续的内存，然后用一个软件数据结构把这些数据结构串联起来使用。

   那在什么时候会用呢？CPU看到的内存可以通过MMU做映射，一段连续的虚拟内存可以被
   映射到不连续的物理内存上, 显然这里是不需要scatterlist这个概念支持的。当有了
   IOMMU这个概念的时候，设备看到的地址可以是一个连续的虚拟地址，这块地址通过IOMMU
   可以映射到不连续的物理地址上，这里也不需要scatterlist这个概念。可以注意到上面
   说的设备的DMA必须是支持连续地址的。

   但是有的时候，有的设备的DMA还支持scatterlist这种数据结构提供的内存模型。简单
   来讲，就是你可以配置这个设备，使得它的DMA可以一次DMA接入一串不连续的物理地址。

   这个时候内核的scatterlist数据结构就出现了。

硬件结构
---------

   如上所述，就是硬件寄存器可以配置一组离散的内存，设备启动DMA可以接入这组内存。

API使用
---------

   在使用硬件的这个特性的时候，需要首先分配好各个内存区域，当然，这个时候还只知道
   各个内存区域的cpu地址(cpu虚拟地址)

   然后把各个内存区域的信息填到scatterlist里，组成相应的内核数据结构。

   把scatterlist这个结构用dma_map_sg处理下，得到每段内存区域对应的dma地址(总线地址),
   注意设备发起DMA操作的时候用的就是这个地址。

   使用各个内存区域的dma地址, 配置具体的硬件。

内核支持
---------
   ...
