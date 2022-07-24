---
title: 使用ftrace跟踪函数
tags:
  - Linux内核
  - 软件调试
  - ftrace
description: >-
  有些时候在调试内核代码时，我们想跟踪下内核代码的执行流程，以及函数执行时间。
  这个时候我们可以用Linux内核自带的ftrace来跟踪。本文简介具体的跟踪方法
abbrlink: 27c5bc60
date: 2021-06-27 18:05:57
---

有些时候在调试内核代码时，我们想跟踪下内核代码的执行流程，以及函数执行时间。
这个时候我们可以用Linux内核自带的ftrace来跟踪。

 - 确定内核打开了ftrace的编译选项: CONFIG_FTRACE。这个Tracers目录下的子编译
   选项可以都打开，我的经验是自测试的几个可以不开，不然开机过程加上自测会时间
   很长，其他的都可以开打。

 - cd /sys/kernel/debug/tracing

 - echo 0 > tracing_on //关闭trace

 - echo 0 > trace //把之前跟踪buffer里的数据清空

 - echo function_graph > current_tracer //设置当前的跟踪器是function_graph

 - echo funcgraph-proc > trace_options

 - echo funcgraph-abstime > trace_options //在输出结果里增加每个函数的时间戳

 - echo your_func > set_ftrace_filter //把你要跟踪的函数配置进ftrace

 - grep -i your_func available_filter_functions //可以查看上个步骤的函数有没有设置好

 - echo 1 > tracing_on //开始trace

 - 启动你要跟踪的程序执行

 - echo 0 > tracing_on //关闭trace

 - cat trace > your_trace_result //输出得到的跟踪信息，可以将信息重定向到文件

经过上面，你可以大概得到一个这样的输出信息:
```
# tracer: function_graph
#
#     TIME        CPU  TASK/PID         DURATION                  FUNCTION CALLS
#      |          |     |    |           |   |                     |   |   |   |
[...]
 9520.457571 |     2)  dma0cha-1201  |   0.940 us    |  hisi_dma_issue_pending [hisi_dma]();
 9520.457583 |     0)    <idle>-0    |   0.680 us    |  hisi_dma_irq [hisi_dma]();
 9520.457584 |     0)    <idle>-0    |   0.490 us    |  hisi_dma_desc_free [hisi_dma]();
 9520.457589 |     2)  dma0cha-1201  |   0.420 us    |  hisi_dma_tx_status [hisi_dma]();
 9520.457666 |     2)  dma0cha-1201  |   0.930 us    |  hisi_dma_prep_dma_memcpy [hisi_dma]();
 9520.457668 |     2)  dma0cha-1201  |   0.900 us    |  hisi_dma_issue_pending [hisi_dma]();
 9520.462827 |     2)  dma0cha-1201  |   0.910 us    |  hisi_dma_issue_pending [hisi_dma]();
 9520.462833 |     0)    <idle>-0    |   0.620 us    |  hisi_dma_irq [hisi_dma]();
[...] 
```
duration这一栏得到的是每个函数的执行时间。通过time一栏的时间戳，可以得到函数和
函数之间的时延, 比如可以得到从发起一个dma传输task(hisi_dma_issue_pending)到
中断函数执行(hisi_dma_irq)的大概时延是：9520.457583 -  9520.457571 = 12us。

如果要跟踪一个函数的调用链，可以使用set_graph_function。具体的使用设置是:

 - echo function_graph > current_tracer

 - echo your_func > set_graph_function
