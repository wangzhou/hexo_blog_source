---
title: Linux中的调度域和调度组
tags:
  - Linux内核
  - 调度
description: >-
  本文总结Linux内核里调度域和调度组的逻辑关系，Linux的调度子系统基于调度域和
  调度组的概念进行负载均衡，本文是后续分析具体负载均衡逻辑的基础。分析使用的
  内核版本是6.8-rc5，使用qemu的版本是v8.2.50的ARM64构架版本。
abbrlink: 56056
date: 2024-03-09 17:45:19
categories:
---

基本逻辑
---------

Linux内核把CPU核按照调度域和调度组范围进行管理，调度域/调度组是一个层级结构，并
不是只有一层。之所以要把CPU核分组管理，是因为在进行线程负载均衡的时候，“距离”较
近的核的附属资源一般集中在一起，当线程在这些核之间迁移时，因为线程迁移带来的开销
更小。所以，如果线程均衡会优先在“距离”近的核上进行，在达成均衡的目的的同时，尽量
减小开销。

具体的看，当硬件具有SMT时，在同一个物理核的不同虚拟核上迁移时，因为虚拟核共享L1
cache，迁移引发的cache开销是很小的；当线程在cluster内迁移时，因为cluster内的CPU
核可能共享L3 cache，迁移引发的cache开销比较小；当线程在NUMA节点内部迁移时，NUMA
balancing只是重新补上NUMA balancing扫描时断开的页表项，而在不同NUMA节点之间迁移
时，NUMA balancing会进行物理页面的迁移(NUMA balancing的具体逻辑可以参考[这里](https://wangzhou.github.io/Linux内存管理-NUMA-balance中的内存迁移))。

调度域/调度组的概念在内核文档中已经有描述：Documentation/scheduler/sched-domains.rst。
下面是一个包含SMT/cluster/NUMA的CPU系统拓扑图，我们用这个图做具体说明。
```
+------------------------ NUMA0 ------------------------+-------------------------- NUMA1 ----------------------------+
|                                                       |                                                             |
+--------- cluster0 --------+--------- cluster1 --------+--------- cluster2 ----------+---------- cluster3 -----------+
|                           |                           |                             |                               |
+-------------+-------------+-------------+-------------+-------------+---------------+---------------+---------------+
|  (p_core0)  |  (p_core1)  |  (p_core2)  |  (p_core3)  |  (p_core4)  |  (p_core5)    |  (p_core6)    |  (p_core7)    |
|             |             |             |             |             |               |               |               |
| core0 core1 | core2 core3 | core4 core5 | core6 core7 | core8 core9 | core10 core11 | core12 core13 | core14 core16 |
+-------------+-------------+-------------+-------------+-------------+---------------+---------------+---------------+
```
对于上面的系统，描述调度域的数据结构是这样的，对于每个core，在每个层级上都有一个
sched_domain的数据结构描述调度范围，比如core0在最底层(SMT)的sched_domain里包含
core0和core1，在再上一层(cluster)的sched_domain里包含core0-core3，在NUMA这一层的
sched_domain里包含core0-core7，在最上层的sched_domain里包含全部core。

调度域在一层中收拢所有的CPU核，调度组则为负载均衡增加一个从上到下的视角。比如，
core0的最底层sched_domain里就包含两个调度组，其中一个调度组里包含core0，另一个
调度组里包含core1；core0的cluster这一层的调度域里包含两个调度组，一个调度组包含
core0/core1，另一个调度组里包含core2/core3。

注意，调度里还有一个task_group的概念，task_group是线程的集合，为的是把一组线程作
为一个调度实体参与调度，利用task_group也可以控制线程使用CPU资源的情况。

代码分析
---------

schedule domain和group创建的基本逻辑如下：
```
start_kernel
  ...
  +-> kernel_init_freeable
    +-> sched_init_smp
      +-> sched_init_domains(cpu_active_mask)
        +-> doms_cur = alloc_sched_domains(1)
            /*
             * doms_cur[0]是系统中active cpu的mask，sched domain和group都是在
             * 这个函数里创建起来的。
             */
        +-> build_sched_domains(doms_cur[0], NULL)

build_sched_domains
      /*
       * 为各级和各个CPU分配domain/group/domain_shared/group_capacity等的内存，
       * 也就是对于每个CPU核在每一个调度层级都有一个domain和group的数据结构(我们
       * 先聚焦分析domain和group)。我们在下面把所有的数据结构都写出来，显示到一
       * 张图里。
       */
  +-> __visit_domain_allocation_hell

  +-> for_each_cpu(i, cpu_map) {
        for_each_sd_topology(tl) {
          +-> build_sched_domain(tl, cpu_map, attr, sd, i)
        }
      }
  +-> for_each_cpu(i, cpu_map) {
        for (sd = *per_cpu_ptr(d.sd, i); sd; sd = sd->parent) {
          +-> (build_sched_groups(sd, i)
        }
      }
  +-> for_each_cpu(i, cpu_map) {                                              
        /* 没有理解LLC imbalanced这里的逻辑? */
      }
  +-> for_each_cpu(i, cpu_map) {                                              
        +-> cpu_attach_domain(sd, d.rd, i)
              /* root domain是什么？*/
          +-> rq_attach_root(rq, rd)
              /* 把core的最底层domain放到core的rq上 */
          +-> rcu_assign_pointer(rq->sd, sd)
      }
```

如下的这个图展示如上系统拓扑中所有CPU核相关的sched_domain/sched_group的数据结构，
用d(xxx)表示domain，括号中的数字表示domain里包含的CPU核，用g(xxx)表示group，括号
里的数字表示group里包含的CPU核，d(xxx)下的多个g(xxx)表示一个domain里包含多个group。
如下只画出来一个NUMA中CPU核上的相关数据结构，另外一个NUMA上的数据结构是类似的。
```
      -----------------------------------------------------------------
NUMA:  d(0-15) d(0-15) d(0-15) d(0-15) d(0-15) d(0-15) d(0-15) d(0-15) 
       g(0-7)  g(0-7)  g(0-7)  g(0-7) 
       g(8-15) g(8-15) g(8-15) g(8-15) ...
       
      -----------------------------------------------------------------
MC:    d(0-7)  d(0-7)  d(0-7)  d(0-7)  d(0-7)  d(0-7)  d(0-7)  d(0-7)  
       g(0-3)  g(0-3)  g(0-3)  g(0-3)
       g(3-7)  g(3-7)  g(3-7)  g(3-7)  ...
       
      -----------------------------------------------------------------
CLS:   d(0-3)  d(0-3)  d(0-3)  d(0-3)  d(4-7)  d(4-7)  d(4-7)  d(4-7)  
       g(0-1)  g(0-1)  g(0-1)  g(0-1)
       g(2-3)  g(2-3)  g(2-3)  g(2-3)  ...
       
      -----------------------------------------------------------------
SMT:   d(0/1)  d(0/1)  d(2/3)  d(2/3)  d(4/5)  d(4/5)  d(6/7)  d(6/7)  
       g(0)    g(0)    g(2)    g(2)
       g(1)    g(1)    g(3)    g(3)    ...
       
       core0   core1   core2   core3   core4   core5   core6   core7   
```

Debug信息
----------

我们采用目前最新的ARM64版本的qemu(v8.2.50)构造一个如上拓扑结构的系统出来。大概的
命令是这样：
```
qemu-system-aarch64 \
        -smp 16,sockets=2,clusters=2,threads=2 \
        -cpu cortex-a57 \
        -machine virt \
        -append "console=ttyAMA0" \
        -nographic -m 4096m \
        -kernel ~/repos/linux/arch/arm64/boot/Image \
        -initrd ~/tests/kernel_debug_using_qemu/rootfs.cpio.gz \
        -object memory-backend-ram,id=mem0,size=2048M \
        -object memory-backend-ram,id=mem1,size=2048M \
        -numa node,memdev=mem0,nodeid=0,cpus=0-7 \
        -numa node,memdev=mem1,nodeid=1,cpus=8-15
```
如上命令表示系统里有16个core，两个socket(一个socket对应一个NUMA)，一个NUMA里包含
两个cluster，一个物理CPU核包含两个逻辑CPU core。

在schedule的debugfs里可以看到如下和domain相关的debug信息：
```
# pwd
/sys/kernel/debug/sched
# echo Y > verbose 
# cd domains/
# ls
cpu0   cpu10  cpu12  cpu14  cpu2   cpu4   cpu6   cpu8
cpu1   cpu11  cpu13  cpu15  cpu3   cpu5   cpu7   cpu9
# cd cpu2/
# tree 
.
├── domain0
│   ├── busy_factor             16
│   ├── cache_nice_tries        0
│   ├── flags
│   ├── groups_flags
│   ├── imbalance_pct           110
│   ├── max_interval            4
│   ├── max_newidle_lb_cost
│   ├── min_interval            2
│   └── name               <--- SMT
├── domain1
│   ├── busy_factor             16
│   ├── cache_nice_tries        1
│   ├── flags
│   ├── groups_flags
│   ├── imbalance_pct           117
│   ├── max_interval            8
│   ├── max_newidle_lb_cost
│   ├── min_interval            4
│   └── name               <--- CLS
├── domain2
│   ├── busy_factor             16
│   ├── cache_nice_tries        1
│   ├── flags
│   ├── groups_flags
│   ├── imbalance_pct           117
│   ├── max_interval            16
│   ├── max_newidle_lb_cost
│   ├── min_interval            8
│   └── name               <--- MC
└── domain3
    ├── busy_factor             16
    ├── cache_nice_tries        2
    ├── flags
    ├── groups_flags
    ├── imbalance_pct           117
    ├── max_interval            32
    ├── max_newidle_lb_cost
    ├── min_interval            16
    └── name               <--- NUMA
```
