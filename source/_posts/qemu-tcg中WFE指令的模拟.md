---
title: qemu_tcg中WFE指令的模拟
tags: [QEMU, ARM64]
description: 本文分析qemu tcg中对ARM64 WFE指令模拟的基本逻辑，分析使用的qemu版本是v8.2.50。
abbrlink: 59376
date: 2024-04-14 16:53:46
categories:
---

WFE指令的基本逻辑
------------------

ARM64中的WFE指令的基本描述可以参考[这里](https://wangzhou.github.io/ARM构架下原子操作相关指令总结/)。

模拟逻辑
---------

可以看到WFE指令的基本逻辑是要触发CPU进入低功耗模式挂起执行，然后还需要模拟各种
触发CPU继续执行的激励。

在我们当前分析的qemu版本中，user mode和system mode对WFE的模拟是不一样的，不过两
者都没有完整模拟出WFE的相关功能。

首先，qemu翻译执行的主循环在遇到WFE指令时，断开tb，配置EXCP_YIELD异常，使用长跳转
跳出翻译执行主循环。

对于user mode，针对EXCP_YIELD异常没有做处理，模拟执行又继续到WFE指令，于是模拟程
序就一直在这里循环了?

对于system mode，跳出翻译执行主循环后，会在qemu_wait_io_event里把vCPU线程挂起等待。
```
qemu_wait_io_event
  +-> qemu_cond_wait(cpu->halt_cond, &bql)
```

ARM spec里描述了多种触发CPU执行的激励，但是，当前的qemu里只实现了中断触发CPU继续
执行的逻辑。qemu tcg里模拟中断的基本逻辑可以参考[这里](https://wangzhou.github.io/qemu-tcg中断模拟/)。在整个中断模拟的逻辑里，会调用
到cpu_interrupt这个函数，对WFE的唤醒逻辑就在其中。
```
/* system/cpus.c */
cpu_interrupt
     /* accel/tcg/tcg-accel-ops.c */
  +-> cpus_accel->handle_interrupt(cpu, mask) // tcg_handle_interrupt
        /* 
         * 代码位置在system/cpus.c，可以看到对于不是当前CPU，走qemu_cpu_kick的
         * 分支，如果中断就是发给当前CPU的走下面的qatomic_set分支。
         */
    +-> qemu_cpu_kick
      +-> qemu_cond_broadcast(cpu->halt_cond)
      +-> cpus_accel->kick_vcpu_thread(cpu) // mttcg_kick_vcpu_thread
        +-> cpu_exit
          +-> qatomic_set(&cpu->exit_request, 1)
          +-> smp_wmb() 
          +-> qatomic_set(&cpu->neg.icount_decr.u16.high, -1)

    +-> qatomic_set(&cpu->neg.icount_decr.u16.high, -1)
```
如上可以看到，对于发给其它CPU的中断，首先触发对应的CPU从挂起的状态继续运行，然后
使对应的CPU退出翻译执行主流程，进而可以去处理中断。如果这里的唤醒在对应CPU睡眠之
前，本次中断中的唤醒是无效的，那么对应CPU的睡眠就只能等后续的中断去唤醒。

这里为什么还要调用cpu_exit一下？cpu_exit中，先配置exit_request为1，再配置icount_decr.
u16.high为-1，在翻译执行的主流程里会检测后者，-1会触发vCPU跳出翻译执行的主逻辑，
进入中断异常处理逻辑，在cpu_handle_interrupt里会检测exit_request，exit_request
为1会触发vCPU彻底跳出翻译执行主逻辑(包括中断异常处理)。所以，cpu_exit的语意是明确
的，就是使vCPU退出运行。这里kick_vcpu_thread是要使vCPU继续运行，不清楚为什么还要
调用下cpu_exit。
