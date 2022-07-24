---
title: PCIe memory base and memory limit
tags:
  - PCIe
description: 本文分析PCIe协议中mem base/limit的相关语意
abbrlink: 77399f99
date: 2021-07-11 23:39:53
categories:
---

在PCIe typeI 配置空间中有mem base, mem limit, prefetch mem base, prefetch mem limit,
upper 32 bit prefetch base, upper 32 bit prefetch limit，io base, io limit等一些
寄存器。

这些寄存器都是在PCI域中路由TLP包的时候用的一些窗口寄存器。如果一个TLP mem 读/写
的地址正好落入mem base ~ mem base + mem limit的范围内，则该TLP包从上游总线转到下游
总线上；如果一个TLP包到达桥的下游总线时，其访问的地址不在mem base ~ mem base + mem limit
的范围内时，该TLP包被传送到上游总线。一个访问从CPU域被转换到PCI域，应该使用和具体
实现相关的硬件转换模块，比如ATU。

prefetch的情况和上面的是类似的，只是prefetch可以是32bit地址也可以是64bit地址。
而非prefetch只有是32bit地址的。

所有的BAR分配的地址都是在上面的窗口中的，其实更准确的讲是先确定BAR的类型和大小然后
再根据一条总线下各类BAR占的总资源，填写上面的寄存器。

也就说BAR的类型可以有：non-prefetch 32 bit mem bar.
                       
                       prefetch 64 bit mem bar
                       prefetch 32 bit mem bar ?

                       16 bit io bar  io都是non-prefetch
                       32 bit io bar

在PCIe中，prefetch的bar(prefetch bit 置位)必须是64bit mem bar[1](22.1.16), 和上面
prefetch 32 bit mem bar是冲突的(Intel 82599上有该类型的BAR)

non-prefetch bar只可以被分到non-prefetch的window里，prefetch bar可以被分到non-prefetch
的window里。64bit bar可以被分到32bit window里。这里的窗口只的都是PCI域的窗口。


[1] [PCI.EXPRESS系统体系结构标准教材].(美)Pavi.Budruk,Don.Anderson,Tom.Shanley
