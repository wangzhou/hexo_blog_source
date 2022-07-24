---
title: Linux irq domain
tags:
  - Linux内核
description: 本文介绍linux内核irq domain的基本概念，分析相关程序实现
abbrlink: 5f6782db
date: 2021-07-17 11:01:12
categories:
---

 linux中断管理中有irq domain的概念。顾名思义irq domain是针对有中断控制功能的IP模
 块中断管理域，是一个软件概念。以linux kernel中的GPIO举个例子。
```
    pins  +-----+     +-----+     +-----+
        ——|     |     |     |     |     |
        ——|     |---->|     |---->|     |
        ——|     |     |     |     |     |
        ——|     |     |     |     |     |
          +-----+     +-----+     +-----+
           GPIO         GIC       CPU core
         controller    
```
 GPIO，GIC，CPU core的硬件示意图如上。 一个GPIO控制器有多个输入输出管脚, 可以输入
 输出高低电平信号，也可以接收外部信号作为中断输入。GPIO控制器把多个外部输入的中断
 信号转成一个中断信号，通过一条中断线报给GIC。CPU core接收到GIC上报的中断后, 通过
 查询GIC的相关寄存器得到中断来自GIC的哪个输入管脚，再通过查询GPIO控制器的相关寄存
 器得到中断真正来自GPIO的哪个输入管脚。然后调用相应的中断处理程序。

 linux内核中断管理的最核心数据结构是struct irq_desc, 为了使用irq号申请中断，中断
 管理系统要为每个中断建立相应的struct irq_desc结构(之后驱动可以使用request_irq()在
 相应的struct irq_desc中注册中断处理程序)。

 上面的硬件框图中，GPIO controller到GIC的中断信号线是在制作Soc的时候就连死的，Soc
 手册中会有该中断线的中断号。在驱动程序中可以直接使用request_irq()直接注册中断处理
 函数。当然在该中断处理函数中也可以直接去读GPIO controller中的中断相关的寄存器，查
 看实际的中断来自哪个GPIO输入管脚，然后调用相应的函数去处理。这样也就没有irq domain
 什么事情了。

 但是如果像上面这样做，会在GPIO controller的驱动代码中加入GPIO所接下级设备的中断
 处理函数。破坏了程序的结构。合理的处理方式是GPIO controller只需要暴露给下级设备输入
 管脚号，下级设备在其自身的驱动程序中，通过管脚号调用GPIO驱动提供的接口，获得相应的
 irq号，然后通过request_irq()注册自己的中断处理函数。基于这样的逻辑，需要在GPIO驱动
 中加入一个struct irq_domain的结构，该结构的作用是：1. 为GPIO的各个输入管脚，在中断
 管理系统中建立相应的struct irq_desc。2. 建立一个输入管脚号到irq号的映射表。

 下面分析一段具体的代码，以内核代码linux/drivers/gpio/gpio-mvebu.c为例：
 在struct mvebu_gpio_chip中包含struct irq_domain *domain用来管理GPIO controller的
 irq domain。在probe函数中
	mvchip->irqbase = irq_alloc_descs(-1, 0, ngpios, -1);
 分配了ngpios个struct irq_desc结构体，返回第一个结构对应的irq号。

	mvchip->domain = irq_domain_add_simple();
 建立并初始化irq_domain结构, 并建立从管脚号到中断号的映射。

 	mvebu_gpio_to_irq();
 通过管脚号找到中断号。

 本文以一个GPIO控制器的例子介绍了linux内核中irq domain的基本逻辑。只要是相似的地方都
 可以引入irq domain的处理方式。
