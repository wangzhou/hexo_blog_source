---
title: Linux软中断基本概念
tags:
  - Linux内核
description: 本文介绍Linux内核里软中断的基本逻辑，分析基于内核代码v5.19-rc8，体系结构基于 riscv。目前，没有实际案例帮助继续分析细节。
abbrlink: 59215
date: 2022-09-07 18:53:45
categories:
---

基本逻辑
--------

中断是CPU硬件上的基本概念，当一个中断发生的时候，CPU当前的执行流被打断，CPU被强
行切到预设好的地址继续执行，被打断的上下文通过CPU的一组寄存器向软件报告。

Linux内核里模拟硬件的这种行为，在软件层面也实现了类似的机制。内核需要预先定义一些
软中断以及软中断要执行的函数，软中断有是否触发的标记，内核有API配置软中断触发标记，
内核会在特定的点去检查是否有软中断的触发标记，如果有，就去调用软中断对应函数，执行
这个函数可以在当前上下文立即执行，也可以唤醒软中断相关的内核线程(ksoftirqd)，在相关
内核线程里执行软中断的回调函数。

代码分析
--------

硬件中断的调用流程，以riscv举例：
```
 /* linux/arch/riscv/kernel/entry.S */
 handle_exception
   +-> generic_handle_arch_irq
     +-> irq_exit
       +-> __irq_exit_rcu
         +-> if (!in_interrupt() && local_softirq_pending())
               /* 注意这个函数会有两个版本，我们这里只看!CONFIG_PREEMPT_RT的版本 */
           +-> invoke_softirq()
             +-> wakeup_softirqd()
```
如上，在硬中断退出的时候会检测有没有pending的软中断，如果有，就去执行软中断的回调
函数，在具体执行的时候，又会根据条件决定是马上调用软中断的处理函数还是唤醒软中断
内核线程去处理，判断条件是是否内核配置了强制在内核线程处理软中断，以及是否有ksoftirqd
这个per-cpu的结构。如果马上执行软中断处理函数，又用内核配置参数决定使用中断当前使用
的栈，还是使用软中断自己的栈。

软中断也提供了API，支持在内核的线程上下文里设置软中断，以及唤醒软中断内核线程处理
软中断。

内核现在已经不建议新增加软中断类型，目前的类型有：
```
enum {                                                                               
        HI_SOFTIRQ=0,                                                           
        TIMER_SOFTIRQ,                                                          
        NET_TX_SOFTIRQ,                                                         
        NET_RX_SOFTIRQ,                                                         
        BLOCK_SOFTIRQ,                                                          
        IRQ_POLL_SOFTIRQ,                                                       
        TASKLET_SOFTIRQ,                                                        
        SCHED_SOFTIRQ,                                                          
        HRTIMER_SOFTIRQ,                                                        
        RCU_SOFTIRQ,
};                                                                              
```
如果依然要使用软中断，可以用tasklet，它就是一种软中断的封装。

软中断使用local_bh_disable()/local_bh_enable()关闭和打开。
