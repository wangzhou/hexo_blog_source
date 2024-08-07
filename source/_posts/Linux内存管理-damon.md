---
title: Linux内存管理-damon
tags:
  - Linux内核
  - 内存管理
description: 本文分析Linux内核里内存管理中damon子系统的基本逻辑，分析基于v6.7-rc6内核。
abbrlink: 37109
date: 2024-02-21 18:44:21
categories:
---

基本逻辑
---------

damon是Linux内核里的一个内存管理的子系统，这个子系统对用户暴露了一组sysfs接口出
来，用户通过配置这些接口可以定制内存回收、内存冷热页管理等策略。damon子系统会根
据用户定制的策略，使用专门的内核线程进行管理。

damon子系统的代码在内核mm/damon目录下。内核中有对应的使用文档，路径在Documentation/
mm/damon，Documentation/admin-guide/mm/damon。

总体上看damon子系统有三个大的使用方式，第一个是做内存监控，第二个是做内存回收，
第三个是做LRU链表上冷热页的调整。

做内存监控的sysfs接口在/sys/kernel/mm/damon目录，用户可以配置监控的目标以及策略，
也可以配置某些事件发生时要做的处理。

内存回收和LRU冷热页调整的sysfs接口在/sys/module/damon_reclaim和/sys/module/damon_lru_sort。
用户通过这些接口定义内存回收和LRU冷热页调整的规则。

damon还是使用PTE的access bit来判断CPU对物理页的访问情况，damon核心的创新是它把
若干page放到一起，只用其中一个page的access bit做为这一堆page访问情况的度量，当
这一堆page(一个区域)的访问情况相仿时，这个优化是合理的。在此基础上，damon会做区
域的合并和拆分，相邻的不活跃的区域会合并成一个大区域，活跃的区域会拆分成多个较
小的区域。

damon还提供一个用户态工具：https://github.com/awslabs/damo，这个工具是damon sysfs
接口的封装，方便用户直接使用。

实际使用
---------

如下是damon做内存监控实际使用时的一个sysfs的文件树，我们把注释直接写到里面。
```
`-- admin
    `-- kdamonds                            <-- 创建了两个kdamond线程，只看0
        |-- 0
        |   |-- contexts                    <-- 目前一个kdamond只支持一个context
        |   |   |-- 0
        |   |   |   |-- avail_operations
        |   |   |   |-- monitoring_attrs
        |   |   |   |   |-- intervals
        |   |   |   |   |   |-- aggr_us
        |   |   |   |   |   |-- sample_us
        |   |   |   |   |   `-- update_us
        |   |   |   |   `-- nr_regions
        |   |   |   |       |-- max
        |   |   |   |       `-- min
        |   |   |   |-- operations
        |   |   |   |-- schemes
        |   |   |   |   `-- nr_schemes      <-- 这里可以展开，配置context的监控策略
        |   |   |   `-- targets             <-- 一个context下创建了3个target，
        |   |   |       |-- 0                   每个target可以跟踪不同进程
        |   |   |       |   |-- pid_target
        |   |   |       |   `-- regions     <-- target0下创建了4个人region
        |   |   |       |       |-- 0
        |   |   |       |       |   |-- end
        |   |   |       |       |   `-- start
        |   |   |       |       |-- 1
        |   |   |       |       |   |-- end
        |   |   |       |       |   `-- start
        |   |   |       |       |-- 2
        |   |   |       |       |   |-- end
        |   |   |       |       |   `-- start
        |   |   |       |       |-- 3
        |   |   |       |       |   |-- end
        |   |   |       |       |   `-- start
        |   |   |       |       `-- nr_regions
        |   |   |       |-- 1
        |   |   |       |   |-- pid_target
        |   |   |       |   `-- regions
        |   |   |       |       `-- nr_regions
        |   |   |       |-- 2
        |   |   |       |   |-- pid_target
        |   |   |       |   `-- regions
        |   |   |       |       `-- nr_regions
        |   |   |       `-- nr_targets
        |   |   `-- nr_contexts
        |   |-- pid
        |   `-- state
        |-- 1
        |   |-- contexts
        |   |   `-- nr_contexts
        |   |-- pid
        |   `-- state
        `-- nr_kdamonds
```

如下是damon做内存回收的sysfs接口：
```
# pwd
/sys/module/damon_reclaim/parameters
# ls
aggr_interval                nr_reclaim_tried_regions
bytes_reclaim_tried_regions  nr_reclaimed_regions
bytes_reclaimed_regions      quota_ms
commit_inputs                quota_reset_interval_ms
enabled                      quota_sz
kdamond_pid                  sample_interval
max_nr_regions               skip_anon
min_age                      wmarks_high
min_nr_regions               wmarks_interval
monitor_region_end           wmarks_low
monitor_region_start         wmarks_mid
nr_quota_exceeds
```

如下是damon做LRU冷热页移动的sysfs接口：
```
# pwd
/sys/module/damon_lru_sort/parameters
# ls
aggr_interval                      nr_cold_quota_exceeds
bytes_lru_sort_tried_cold_regions  nr_hot_quota_exceeds
bytes_lru_sort_tried_hot_regions   nr_lru_sort_tried_cold_regions
bytes_lru_sorted_cold_regions      nr_lru_sort_tried_hot_regions
bytes_lru_sorted_hot_regions       nr_lru_sorted_cold_regions
cold_min_age                       nr_lru_sorted_hot_regions
commit_inputs                      quota_ms
enabled                            quota_reset_interval_ms
hot_thres_access_freq              sample_interval
kdamond_pid                        wmarks_high
max_nr_regions                     wmarks_interval
min_nr_regions                     wmarks_low
monitor_region_end                 wmarks_mid
monitor_region_start
```

代码分析
---------

damon子系统用一堆__init修饰的初始化函数在内核启动的时候做damon相关的初始化。damon
子系统的数据结构和运行大逻辑比较简单，复杂度在和业务相关的参数上。

damon的内部数据结构基本和内存检测sysfs接口展示出来的一致，也是context/target/region
的基本数据结构，context中抽象出了具体业务在整个检测流程里的操作函数，这一堆操作
函数统一放到struct damon_operations里。

如上的三种业务，每种业务运行的时候都对应一个kdamond的内核线程。每个kdamon线程都
在一个循环里执行：检测内存访问，对应内存操作，调整检测区域的步骤，具体业务在
damon_operations的各个回调函数里执行。

下面用damon reclaim串联下如上的逻辑。damon reclaim的代码在mm/damon/reclaim，现在
是一个只能编译进内核的内核模块。
```
damon_reclaim_init
      /*
       * 生成context和target(全局变量), 为context添加上damon_operations，其中
       * damon_operations是DAMON_OPS_PADDR，为context生成对应的scheme，可见这里
       * 同样有scheme的概念，只不过没有对外接口，直接写死了scheme的对应参数。
       */
  +-> damon_modules_new_paddr_ctx_target
```

用户对/sys/module/damon_reclaim/enabled写1触发damon收拢相关参数，然后拉起kdamond
内核线程。
```
damon_reclaim_enabled_store
  +-> damon_reclaim_turn
    +-> damon_reclaim_apply_parameters
      +-> damon_start
      ...
```

reclaim场景下kdamond的基本逻辑。
```
kdamond_fn
  while (!kdamond_need_stop(ctx)) {
    +-> prepare_access_checks // damon_pa_prepare_access_checks
    |     /*
    |      * 对于每个region都随机挑选一个物理页做预处理，所谓预处理就是对这个物
    |      * 理页做逆向映射查找(rmap_walk)，对于找到的每个虚拟页都清除页表上对应
    |      * 的访问标记(access bit)。
    |      */
    | +-> damon_pa_mkold
    |
    |   /* 等待sample_interval us */
    +-> kdamond_usleep(sample_interval)
    |
    +-> check_accesses // damon_pa_check_accesses
    |     /*
    |      * 如上相同的逻辑，rmap_walk检查所有虚拟页上的访问标记是否有被写过，
    |      * 由此得出相应物理页在这期间有没有被访问过，并记录得到的统计数据。
    |      *
    |      * 这个函数返回相应物理页有没有被访问过。
    |      */
    | +-> damon_pa_young
    |     /* 更新统计结果 */
    | +-> damon_update_region_access_rate
    |
    +-> kdamond_apply_schemes
          /*
           * 根据scheme->action里的配置做对应的处理，如上初始化scheme时给reclaim
           * scheme->action的参数是DAMOS_PAGEOUT，所以这里进入对应的流程。
           *
           * 针对每个target、每个region都做这样的处理。
           */
      +-> ctx->ops.apply_schemes // damon_pa_apply_schemes
            /* 没有看懂这里的逻辑，是否做reclaim的依据是什么？*/
        +-> damon_pa_pageout
          +-> reclaim_pages
  }
```
