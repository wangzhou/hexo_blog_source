---
title: Linux workqueue分析
tags:
  - Linux内核
description: >-
  本文介绍Linux内核里workqueue的使用方法，分析workqueue具体的代码实现。
  并且基于qemu虚拟机做简单的测试。本文的分析基于Linux主线代码v5.5, 分析参考了蜗窝科技的分析文章:
  http://www.wowotech.net/irq_subsystem/workqueue.html
  http://www.wowotech.net/irq_subsystem/cmwq-intro.html
  http://www.wowotech.net/irq_subsystem/alloc_workqueue.html
  http://www.wowotech.net/irq_subsystem/queue_and_handle_work.html
abbrlink: 6f5b0f9e
date: 2021-06-27 18:01:35
---

workqueue基本使用方法
------------------------

 workqueue机制可以异步执行内核其他模块的任务。内核其他模块有任务需要异步执行的
 时候，可以调用workqueue提供的接口，把任务丢给一个对应的内核线程去执行。这里
 提到的任务定义是一个函数。

 这里我们提到的workqueue是内核最新的版本，它的正规名字叫做Concurrency Managed
 Workqueue。代码路劲在：linux/kernel/workqueue.c, linux/include/linux/workqueue.h

 workqueue相关的基本概念有: work, workqueue, pool workqueue, worker pool, worker,
 per-cpu worker pool, unbound worker pool, per-cpu workqueue, unbound workqueue.

 work, workqueue是workqueue对其他模块的接口。使用者要先创建需要执行的任务, 即work。
 然后调用workqueue API把这个任务放入相关的workqueue里执行。

 整个workqueue又分per-cpu workqueue和unbound workqueue。per-cpu workqueue是系统
 启动的时候就分配好的(to do: 需要分析)。unbound workqueue需要用户自己创建。

 如果用户使用per-cpu workqueue，只需要调用schedule_work(struct work_struct *work)
 即可。这个API会把work放到当前cpu的normal kworker pool中的一个worker上跑。
 使用queue_work_on(cpu, system_highpri_wq, work)可以把一个work放到指定cpu 高优先
 级pool上跑，使用queue_work_on(cpu, system_wq, work)可以把work放到指定cpu normal
 kworker pool上跑。

 如果用户使用unbound workqueue, 需要先使用
 alloc_workqueue(const char *fmt, unsigned int flags, int max_active, ...)申请
 workqueue, 其中flags中要使用WQ_UNBOUND。随后使用
 queue_work(struct workqueue_struct *wq, struct work_strct *work)把work放到wq
 里执行，也可以用queue_work_on(int cpu, struct workqueue_struct *wq, struct work_strct *work)，
 但是，上面的这个函数执行的时候并不能精确的把work放在cpu上执行。这两个函数的
 效果都是，把work放到当前cpu对应的numa node上的一个cpu跑。

 可以看到，虽然workqueue的用户接口是work定义和work queue，但是最新workqueue的设计
 把前端的用户接口和后端的执行线程解耦开来，而且进一步加入了work pool即线程池的
 概念，把具体的执行线程即worker的创建和销毁变成了一个动态的过程。所以，我们的讨论
 要先基于线程池来。对于per-cpu workqueue, 在workqueue初始化的过程中，为每一个cpu
 都创建一个normal work pool和高优先级的work pool。对于unbound workqueue使用的
 unbound work pool，系统建立一个全局的hash表保存所有不同种类的unbound work pool,
 所谓的种类由nice、cpumask、no numa三个参数决定，如果发现需要的unbound work pool
 在系统里已经有了，那么直接使用已有的unbound work pool, 一个work可以显示的放在
 一个unbound workqueue上跑，但是真正调度到哪个unbound work pool、还是新建立一个
 unbound work pool, 这个需要运行时才能决定。

 如上，现在workqueue的设计分了前端和后端, 前端是workqueue，后端是kworker pool，
 worker。中间靠一个pool workqueue的东西连接。这里kworker pool就是一个线程池，
 worker就是一个个线程，pool workqueue的pool是一个动作，这个动作把前端的workqueue
 和后端的一个线程池建立起联系。

 至此，我们还需要知道线程池里线程创建和销毁的机制。首先创建线程池的时候会默认
 创建一个线程，注意，这里不是在创建队列的时候。也就是说，对于一个unbound work
 queue, 在复用系统已经的unbound线程池的时候(绝大多数是复用已有的, 从调试结果看,
 系统一开始就会给每个numa node上创建一个normal unbound work pool和一个高优先级
 unbound work pool), 是完全可能不新建线程的。当线程池里的线程没有被使用的时候，
 会自动进入idle状态，idle装态的线程在一定时间后会被销毁。线程池里至少要保持有一个
 idle线程在，即使超时也不会被销毁。所以，当一个线程池里只有一个idle线程，这时我们
 又queue work到这个线程池时，workqueue的代码会首先wake up这个idle线程，这个idle
 线程起来后首先要做的就是看看有没有idle线程，如果没有，就要创建一个线程出来，可以
 看到idle线程被wake up后，这个线程池里已经没有idle线程，所以这里就会在线程池里
 创建新线程。当线程池里的线程被阻塞，这时又有新的work被调度的到这个线程池里时，
 也会创建新的线程(to do: 分析code)。处于运行状态的线程会把work queue里缓存的任务
 都执行一遍。当一个任务需要执行的时候，如果work pool里有运行状态的线程时，workqueue
 代码不会wake up idle线程。

 当频繁把work放到一个unbound work pool上时，会有新worker创建被创建出来。但是
 当频繁的把work放到per-cpu work pool上的时候，任务只会在一个相同的worker上执行。
 (to do: 分析code)

 以上的数据结构大致是这样的:
```
     +----------+
     |unbound wq+-+- kworker pool(node0)-+- worker0 (u<work pool id>:<worker num>)
     +----------+ |                      +- worker1
                  |                      `- ...
                  |
                  `- kworker pool(node1)-+- worker0
                                         +- worker1
                                         `- ...
     +------------+
     |numa node 0 |
     |            |  .-- kworker pool    -+- worker0 (<cpu num>:<worker num>)
     |   cpu 0 ---+--+-- kworker pool[H]  +- worker1 
     |            |  .-- kworker pool     `- ...
     |   cpu 1 ---+--+-- kworker pool[H] -+- worker0 (<cpu num>:<worker num>H)
     +------------+                       +- worker1
                                          `- ...
     +------------+
     |numa node 1 |
     |            |
     |   cpu 2    |      ...
     |            |
     |   cpu 3    |
     +------------+
```

workqueue代码的基本结构
--------------------------

 详见简介中的连接, 这四篇文章已经讲的很好。

workqeueu API再挖掘
----------------------
 
 如上，workqueue的work不会直接和worker绑定，而是在queue_work的时候选择一个
 kworker pool。

 对于unbound workqueue, alloc_workqueue中会判断系统里是否有相同的kworker pool,
 如果有，就用已有的kworker pool。判断相同的依据是nice，cpumask，no numa。对于
 per-cpu workqueue，直接选择当前cpu上的kworker pool。

 创建一个新的kworker pool肯定至少要创建一个内核线程。所以，如果alloc_workqueue
 复用旧的kworker pool可能会用之前kworker pool里的线程，如果alloc_workqueue里创建
 新的kworker pool，那么这个步骤就会新建一个内核线程。

 kworker pool的worker是动态调整的，pool里的线程都处于阻塞状态的时候，pool就会
 新起一个线程执行work。下面的测试中，我们给一个unbound wq里发work，每个work里
 sleep一段时间，可以看到，这个unbound wq是复用的系统已经有的kworker pool，在用完
 该kworker pool里本来已有的线程后，该worker pool会起新的线程执行work, 过一会
 可以发现，该worker pool里的线程有一部分被回收。(case 3)

 注意, 在申请workqueue的时候加上WQ_SYSFS参数，可以把该workqueue的信息通过sysfs
 暴露到用户态：e.g. /sys/bus/workqueue/devices/writeback

测试
-------

 如下附录中的测试代码, 以下是测试得到的结论:

 case 1: queue_work_on, 对unbound work queue操作时只能把work放在对应numa node上
         的cpu，无法具体到cpu。

 case 2: schedule_work可以把work放到当前的cpu上。

 case 3: 当kwork pool里的线程都处于block状态的时候，如果有work需要执行，对应
         的pool就会新分配内核线程。(这个对于per-cpu和unbound work queue都是一样的)

 case 4: schedule_work会把work放到当前cpu上跑，block的时候，pool会新分配线程。

 case 5: schedule_work_on会只把work放到指定cpu上，block的时候，pool会新分配线程。

 case 6: 使用unboud pool, 频繁queue_work_on, 即使没有block，pool里也会创建新
         线程。

 case 7: 使用queue_work_on(cpu, system_highpri_wq, work)可以把一个work放到指定
         cpu上的高优先级pool上跑。如果有block, pool里会分配新线程，如果没有block，
	 即使频繁调用queue_work_on pool也不会创建新线程。

附录
-----
 wq_test.c测试代码：https://github.com/wangzhou/tests/blob/master/kwq/wq_test.c
