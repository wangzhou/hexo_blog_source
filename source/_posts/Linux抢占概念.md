---
title: Linux抢占概念
tags:
  - Linux内核
description: >-
  这篇文章把linux内核里的preempt这个概念介绍的比较清楚:
  http://news.eeworld.com.cn/mp/ymc/a52661.jspx 本文再把一些关键的点总结下。
abbrlink: abd7cd0e
date: 2021-06-20 23:18:11
---

抢占的概念
-------------

 linux内核支持抢占, 可以通过抢占相关的编译宏打开或者禁止抢占。

 一个线程在一个cpu上运行, 不是这个线程主动让出cpu导致的其他线程跑到这个cpu上
 的情况都是抢占。关抢占就是禁止这样的行为发生。

 线程主动让出cpu的行为有，比如一个线程发送了一个I/O任务，然后调用
 wait_for_completion睡眠等待completion。其他的各种行为我们都认为是内核抢占行为，
 比如，周期性的tick中断中调度器把另外一个线程调度到cpu上跑；各种中断程序里，
 触发了调度把另一个线程调度到cpu上跑。

 所以，内核里一段程序之前关了内核抢占之后又把内核抢占打开，如果这块代码中没有
 主动让出cpu的行为，我们可以认为这段代码在一个cpu上会持续的执行完，中间不会被
 打断。但是，这里说的是单个cpu上的情况, 当这段代码里有访问全局的数据结构的时候，
 对于那个全局的数据结构，还是可能和其他cpu上的程序并发访问的。

开关抢占的实现
-----------------

 可以看到禁止抢占这个宏就是给task_struct结构里的preempt count加引用计数。
```
#define preempt_disable() \
do { \
	preempt_count_inc(); \
	barrier(); \
} while (0)
```

 开启抢占是先减引用计数, 然后执行调度。
```
#define preempt_enable() \
do { \
	barrier(); \
	if (unlikely(preempt_count_dec_and_test())) \
		__preempt_schedule(); \
} while (0)
```

 是否可以抢占的判断标准除了如上preempt的计数外，还有中断的情况。如果中断是关闭的
 同样认为是不能抢占的。
```
#define preemptible()	(preempt_count() == 0 && !irqs_disabled())
```
 但是，从上面可以看出来，禁止抢占并没有关中断，在禁止抢占的上下文里，中断依然可以
 进来，中断处理完后内核会进行调度，在禁止抢占时，内核依然调度被中断打断的线程运行，
 这样从总体上看之前的线程没有被其它线程抢占。

关抢占的用途
---------------

 关抢占的目的就是叫一段代码不被内核调度打断下执行完，因为有的时候换一个程序进来
 跑可能把原来程序的数据破坏掉。比如内核zswap里，在每一个cpu上分配了压缩解压缩
 的tfm，然后用per-cpu变量保存相应的内容，他在使用这些per-cpu变量的时候就需要
 关闭内核抢占, 不然中途换另一个线程在cpu上跑就会有数据不一致的问题。

 可以看到在使用per-cpu变量的宏里，一进来就把抢占关了:
```
#define get_cpu_ptr(var)						\
({									\
	preempt_disable();						\
	this_cpu_ptr(var);						\
})
```
