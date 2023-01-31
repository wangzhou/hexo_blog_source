---
title: ftrace学习笔记1
tags:
  - 软件调试
  - ftrace
description: >-
  根据之前的学习笔记整理, 介绍ftrace的一些基本知识，以及几个基础的tracer。不过读者
  要是有时间，建议可以浏览下kernel源码中有关ftrace的使用介绍：linux-src/Documentation/trace/
abbrlink: 57c95d77
date: 2021-07-17 11:16:48
categories:
---

原理
----

1. ftrace是一个内核调试工具，调试内核运行，是一个trace工具。它可以查看函数的
  调用关系、函数的执行时间、系统中断的最大关闭时间等等

2. 需编译内核时进行配置，使用时挂载虚拟文件系统debugfs：
  mount -t debugfs nodev /sys/kernel/debug

3. 通过对该文件系统中文件的读写完成所用功能

配置与编译
-----------
```
Kernel hacking  --->
[*] Tracers --->
-*- Tracers
...(配置相应的跟踪器)
```

具体使用
--------

下面显示的就是ftrace使用时的目录了(pc ubuntu12.04)，所有的操作都可以通过读写这些
文件完成，读写的时候用到的命令多为echo, cat之类进入/sys/kernel/debug/tracing,
ubuntu12.04默认已经把ftrace编译入了内核, ls /sys/kernel/debug/tracing得到:

vailable_events            kprobe_profile      stack_trace
available_filter_functions  options             trace
available_tracers           per_cpu             trace_clock
buffer_size_kb              printk_formats      trace_marker
buffer_total_size_kb        README              trace_options
current_tracer              saved_cmdlines      trace_pipe
dyn_ftrace_total_info       set_event           trace_stat
enabled_functions           set_ftrace_filter   tracing_cpumask
events                      set_ftrace_notrace  tracing_enabled
free_buffer                 set_ftrace_pid      tracing_max_latency
function_profile_enabled    set_graph_function  tracing_on
kprobe_events               stack_max_size      tracing_thresh

可以 cat vailable_tracers 查看现在支持的tracer, 一般的tracer有：

function, 可以打印出内核函数的调用过程
function_graph, 以函数调用的格式打印函数调用过程，看起来要方便很多
irqsoff, 打印出禁止中断的时间，对于系统响应不及时的问题，可以用这个查看
wakeup，这个可以打印进程从ready到run的latency
sched_swich，显示的是关于调度的信息

cat current_tracer 查看当前的tracer是什么，所以要用一个tracer的时候，首先要
把它写到这个文件中，比如要用function，就写 echo function > current_tracer

下面具体看一个一个tracer的用法和由此引出来的东西:

 * function:

因为测试的时候常常挂死（cat trace）, 这里就把function的使用写成了一个脚
本，我们一行一行看下，echo 0 > tracing_on， 暂停跟踪器，参考网上的资料，
一般会有一个tracing_enabled的文件(但是有些系统上可能没有),关于tracing_on
和tracing_enabled的区别，现在的理解是，tracing_on是暂停跟踪器，此时跟踪
器还在跟踪内核的运行，只是不再向文件 trace 中写入跟踪信息，
echo 1 > tracing_on 是可以继续当前的跟踪的，而tracing_enabled用来开关
ftrace。另外，ftrace提供了内核函数tracing_on()，这个东西可以直接写在内
核里，放在你想要停起的地方，当tracing_on()执行的时候，你在外面cat trace，
显示的就是附近的信息，如果你在用户态暂停，trace中的内容会离你想要定位的
地方远
```
#! /system/bin/sh 
# change to /sys/kernel/debug/tracing 

dir="/sys/kernel/debug/tracing/" 
echo 0 > ${dir}tracing_on 
echo function > ${dir}current_tracer          # 写入function
echo 1 > ${dir}tracing_on                     # 运行tracer
sleep 5                                       # 叫tracer运行一段时间
echo 0 > ${dir}tracing_on                     # 暂停tracer
cat ${dir}trace | head -10                    # 显示跟踪的函数内容
```
截取了一段现实的结果放在这里
```
# tracer: function 
#                       
# entries-in-buffer/entries-written: 222082/295723   #P:4 
#                                               （online cpu number: 4）
#                              _-----=> irqs-off 
#                             / _----=> need-resched 
#                            | / _---=> hardirq/softirq 
#                            || / _--=> preempt-depth 
#                            ||| /     delay 
#           TASK-PID   CPU#  ||||    TIMESTAMP  FUNCTION 
#              | |       |   ||||       |         | 
 SurfaceFlinger-255   [001] d... 14810.259416: gic_handle_irq <-__irq_usr 
 SurfaceFlinger-255   [001] d... 14810.259483: irq_find_mapping <-gic_handle_irq 
```
 *  function_graph:

同样的操作，看看function_graph的输出
```
  # tracer: function_graph 
  # 
  # CPU  DURATION                  FUNCTION CALLS 
  # |     |   |                     |   |   |   | 
   3)               |  __do_fault() { 
3)               |    i915_gem_fault() { 
3)               |      i915_mutex_lock_interruptible() { 
3)               |        mutex_lock_interruptible() { 
3)   0.339 us    |          _cond_resched(); 
3)   1.212 us    |        } 
3)   1.887 us    |      } 3)               |      i915_gem_object_unbind() { 
3)   0.114 us    |        i915_gem_object_finish_gpu(); 
3)   0.120 us    |        i915_gem_release_mmap();
```
这里 echo __do_fault > set_graph_function了一下，所以输出是针对
_do_fault()的检测

 * irqsoff:

下面截取一段irqsoff的输出信息
```
# tracer: irqsoff 
# 
# irqsoff latency trace v1.1.5 on 3.9.0-rc7-00004-g70df926-dirty 
# -------------------------------------------------------------------- 
# latency: 17790 us, #323/323, CPU#1 | (M:server VP:0, KP:0, SP:0 HP:0 #P:4) 
#    ----------------- 
#    | task: ProcessStats-502 (uid:1000 nice:0 policy:0 rt_prio:0) 
#    ----------------- 
#  => started at: __lock_task_sighand 
#  => ended at:   _raw_spin_unlock_irqrestore 
# 
# 
                  _------=> CPU#            
 / _-----=> irqs-off        
 | / _----=> need-resched    
 || / _---=> hardirq/softirq 
 ||| / _--=> preempt-depth   
 |||| /     delay             
        # cmd     pid      ||||| time  |   caller      
      \   /          |||||  \       |   /           
ProcessS-502     1d...   49us!: __lock_task_sighand 
ProcessS-502     1d...  154us!: _raw_spin_lock <-__lock_task_sighand 
ProcessS-502     1d...  933us+: thread_group_cputime_adjusted <-do_task_stat 
ProcessS-502     1d... 1016us+: thread_group_cputime <-thread_group_cputime_adjusted 
ProcessS-502     1d... 1106us+: task_sched_runtime <-thread_group_cputime 
ProcessS-502     1d... 1195us+: task_rq_lock <-task_sched_runtime 
        ProcessS-502     1d... 1286us!: _raw_spin_lock_irqsave <-task_rq_lock
```
上面找到的是中断关闭时间最长的进程（ProcessStats-502），具体显示出关闭
和开中断的函数，CPU# 进程运行在哪个CPU上，irqs-off ‘d’表示中断关闭
(上半部)，need-resched设置时中断出来之后，执行一次调度，hardirq/softirq
表示硬中断/软中断正在运行。latency: 17790us表示的是关中断最长的时间

 * Wakeup_rt:
```
  # tracer: wakeup_rt 
  # 
  # wakeup_rt latency trace v1.1.5 on 3.9.0-rc7-00004-g70df926-dirty 
  # -------------------------------------------------------------------- 
  # latency: 4155 us, #269/269, CPU#0 | (M:server VP:0, KP:0, SP:0 HP:0 #P:4) 
  #    ----------------- 
  #    | task: watchdog/0-11 (uid:0 nice:0 policy:1 rt_prio:99) 
  #    ----------------- 
  # 
  #                  _------=> CPU#            
  #                 / _-----=> irqs-off        
  #                | / _----=> need-resched    
  #                || / _---=> hardirq/softirq 
  #                ||| / _--=> preempt-depth   
  #                |||| /     delay             
  #  cmd     pid   ||||| time  |   caller      
  #     \   /      |||||  \    |   /           
    <idle>-0       0d.h.   32us+:      0:120:R   + [000]    11:  0:R watchdog/0 
<idle>-0       0d.h.   99us+: 0 
<idle>-0       0d.h.  114us+: check_preempt_curr <-ttwu_do_wakeup 
<idle>-0       0d.h.  126us+: resched_task <-check_preempt_curr 
<idle>-0       0dNh.  141us+: task_woken_rt <-ttwu_do_wakeup 
<idle>-0       0dNh.  157us+: _raw_spin_unlock_irqrestore <-try_to_wake_up 
<idle>-0       0dNh.  172us+: ktime_get <-watchdog_timer_fn 
…
<idle>-0       0dN.. 4064us+: put_prev_task_idle <-__schedule <idle>-0       0dN.. 4077us+: pick_next_task_stop <-__schedule 
<idle>-0       0dN.. 4090us+: pick_next_task_rt <-__schedule 
<idle>-0       0dN.. 4103us+: dequeue_pushable_task <-pick_next_task_rt 
<idle>-0       0d... 4126us+: __schedule 
<idle>-0       0d... 4138us :      0:120:R ==> [000]    11:  0:R watchdog/0
```
可以看到最长实时进程的latency是4155us, 其中很多时间是消耗在过程函数的统
计上，echo 0 > /sys/kernel/debug/tracing/options/function-trace 
可以关闭对过程函数的统计, 有些系统上没有上面的目录或文件

设定
sysctl  kernel.ftrace_enabled=1   或者 
echo 1 > /proc/sys/kernel/ftrace_enabled
也可以关掉对过程函数的跟踪, 下面是使用wakeup tracer时，不记录过程函数时
的测试结果：
```
# tracer: wakeup 
# 
# wakeup latency trace v1.1.5 on 3.2.0-41-generic 
# -------------------------------------------------------------------- 
# latency: 8307 us, #4/4, CPU#2 | (M:desktop VP:0, KP:0, SP:0 HP:0 #P:4) 
#    ----------------- 
#    | task: alsa-sink-22445 (uid:1002 nice:-11 policy:2 rt_prio:5) 
#    ----------------- 
# 
#                  _------=> CPU#            
#                 / _-----=> irqs-off        
#                | / _----=> need-resched    
#                || / _---=> hardirq/softirq 
#                ||| / _--=> preempt-depth   
#                |||| /     delay             
#  cmd     pid   ||||| time  |   caller      
#     \   /      |||||  \    |   /           
   Xorg-22173   2d.h1    0us :  22173:120:R   + [002] 22445: 94:R alsa-sink 
Xorg-22173   2d.h1    1us!: ttwu_do_activate.constprop.179 <-sched_ttwu_pending 
Xorg-22173   2d... 8306us : probe_wakeup_sched_switch <-__schedule 
Xorg-22173   2d... 8307us :  22173:120:R ==> [002] 22445: 94:R alsa-sink 
```
