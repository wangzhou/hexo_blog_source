---
title: Linux内核NMI-watchdog基本逻辑
tags:
  - Linux内核
  - 软件调试
description: 本文梳理Linux内核基于NMI的watchdog的基本逻辑。分析依赖的Linux内核版本是v6.8-rc5
abbrlink: 13222
date: 2024-04-28 19:43:01
categories:
---

基本逻辑
---------

一个计算机系统可能支持各种watchdog，比如可以使用一个专用的watchdog硬件设备，这个
硬件设备是一个计数器，计数器超时会触发告警，我们每隔一段时清理下这个计数器，这样
在系统正常时，总不会触发计数器超时告警，反过来讲，当计数器超时告警时，我们推断系
统发生了异常，异常导致无法及时清理这个计数器。

Linux内核基于PMU counter实现了watchdog，这种实现要求PMU counter的中断要是不可屏
蔽中断(NMI)。基本逻辑是，soft lockup的hrtimer超时处理里会更新相关的标记(hrtimer_interrupts)，
PMU counter的益处中断处理函数里检测hrtimer_interrupts有没有被更新，如果一定时间
没有被更新，那么说明hrtime的中断得不到处理，就判断发生了异常。

Linux内核把这种异常叫做hard lockup，基于PMU counter的watchdog是hard lockup的一种
实现方式。对应的，Linux内核也有soft lockup的特性，具体逻辑可以参考[这里](https://wangzhou.github.io/Linux内核soft-lockup检测机制/)。

代码分析
---------

内核初始化时会调用lockup_detector_init，这里是soft lockup和hard lockup的入口，
```
lockup_detector_init
      /* kernel/watchdog_perf.c */
  +-> watchdog_hardlockup_probe

        /* 先检测当前硬件平台上有没有支持NMI中断的PMU */
    +-> arch_perf_nmi_is_available

        /* 有NMI中断的PMU, 使用PMU的counter中断(配置成NMI)检测hard lockup */
    +-> hardlockup_detector_event_create
          /* 根据watchdog_thresh计算counter的采样间隔 */
      +-> wd_attr->sample_period = hw_nmi_get_sample_period(watchdog_thresh)
          /* 注册PMU event */
      +-> perf_event_create_kernel_counter(wd_attr, cpu, NULL, watchdog_overflow_callback, NULL)
          /* 更新watchdog_ev为如上得到perf_event*/
      +-> this_cpu_write(watchdog_ev, evt)

        /* 对于soft lockup打开的情况，在soft lockup后重新启动hard lockup */
    +-> lockup_detector_setup
      +-> if (watchdog_enabled && watchdog_thresh)
            softlockup_start_all()
          /* 目前是空函数？*/
      +-> watchdog_hardlockup_start()
          /* ？*/
      +-> __lockup_detector_cleanup()

watchdog_overflow_callback
  +-> watchdog_check_timestamp
  +-> watchdog_hardlockup_check
        /* 
         * 通过检测hrtimer_interrupts变量是否更新来判断hard lockup，而hrtimer_interrupts
         * 这个变量是在soft lockup的hrtimer处理函数中更新的。如果这个变量不变化，
         * 表示一段时间都没有hrtimer中断，判断发生了hard lockup。
         * 
         * 可见关中断的时间如果太长就会触发hard lockup。
         */
    +-> is_hardlockup(cpu)
```
