---
title: dpdk mempool的逻辑
tags:
  - dpdk
description: 本文分析dpdk里自带的块内存池的实现
abbrlink: '57323647'
date: 2021-06-20 23:15:26
---

dpdk里有块内存池的支持，用户可以调用相关接口创建固定block大小的内存池，然后从这
个内存池里申请和释放内存。

这个内存池叫做rte_mempool, 相关的实现代码在dpdk/lib/librte_mempool/*. dpdk里
提供了rte_mempool相关的测试app，在这个地方app/test/test_mempool.c, 顺着这个测试
app可以大概看出如何使用rte_mempool。

相关的接口有：
-------------

1. rte_mempool_create

2. rte_mempool_cache_create

3. rte_mempool_generic_get

4. rte_mempool_generic_put

基本的构架是：
-------------
```
               +------------+
     cpu0      |local_cache |----+
               +------------+    |
                                 |
               +------------+    +-> +---------+           +-------------+
     cpu1      |local_cache |------> |rte_ring |---------->|memory block |
               +------------+    +-> +---------+           +-------------+
                                 |
                                 |
                                 |
                                 |
                                 |
               +------------+    |
     cpuN      |local_cache |----+
               +------------+
```
支持mempool的基本数据结构式，local_cache, rte_ring, memzone。memzone是分配的
内存(包括管理需要的内存)，一般是大页内存。rte_ring是一个无锁队列，用来管理内存,
rte_ring的entry个数等于memory里block的个数。rte_ring用cmp and change的原子指令
实现无锁队列，但是，在多核的时候，这个指令的开销也很大，所以为每一个cpu建立了
block内存的cache，cpu先从自己的cache里取block内存，不够的时候再去rte_ring里批量取。

具体实现的关键点：
-----------------
```
rte_mempool_create
  /*
   * flags指示具体挂哪个ops, ops的定义在dpdk/drivers/mempool/ring/rte_mempool_ring.c
   * ring的初始化调用alloc，从ring里申请释放内存调用dequeue/enqueue
   */
  +-> rte_mempool_set_ops_name
  +-> rte_mempool_populate_default
    +-> mempool_ops_alloc_onece
      +-> rte_mempool_ops_alloc
        +-> ops->alloc
  /* 被管理的memory memzone的申请 */
  +-> rte_mempool_populate_default 

在dpdk/drivers/mempool/ring/rte_mempool_ring.c里有alloc的实现：
ring_alloc
  +-> rte_ring_create
    +-> rte_ring_create_elem
       /* ring memzone的申请在这个地方 */
       +-> rte_memzone_reserve_aligned
       /* ring的初始化 */
       +-> rte_ring_init

rte_mempool_generic_get
  +-> /* get from local cache */
  ...
  +-> /* ring_dequeue: 这个地方是从ring里分配内存 */
    +-> rte_mempool_ops_dequeue_bluk
      +-> ops->dequeue

在dpdk/drivers/mempool/ring/rte_mempool_ring.c里有dequeue的实现：
e.g. rte_ring_sc_dequeue
       +-> rte_ring_sc_dequeue_bulk
       ...
         /* 具体申请bluck的逻辑 */
         +-> __rte_ring_do_dequeue_elem
```
