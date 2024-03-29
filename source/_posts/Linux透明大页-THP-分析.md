---
title: Linux透明大页(THP)分析
tags:
  - Linux内核
  - 内存管理
description: 本文简单介绍Linux内核透明大页的概念、使用方式和代码
abbrlink: 7bfba555
date: 2021-06-19 10:05:48
---

THP是什么
------------

 Linux有两种使用大页的方式，一种是普通大页，另一种是透明大页。本文说的就是透明
 大页，透明大页的本意是系统在可以搞成大页的时候，自动的给你做大页的映射，这样
 有两个好处，一是减少缺页的次数，另一个是减少TLB miss的数量。

 可以在这里找见THP的说明文档：
```
linux/Documentation/vm/transhuge.rst
linux/Documentation/admin-guide/mm/transhuge.rst
```

THP的使用方式
----------------

 上面的内核文档已经详细介绍了THP的使用方式。简单总结，就是sysfs中提供了三大类
 接口去控制THP的使用方式。

 第一类接口配置THP的使用范围: /sys/kernel/mm/transparent_hugepage/enabled
 alway是指在整个系统里使用THP, madvise是指可以用madvise指定使用THP的地址范围，
 never是关掉THP。内核提供了CONFIG_TRANSPARENT_HUGEPAGE_ALWAYS以及
 CONFIG_TRANSPARENT_HUGEPAGE_MADVISE去配置如上enable里的默认值。

 第二类接口配置生成大页的方式：/sys/kernel/mm/transparent_hugepage/defrag
 always是在内存申请或者madvise的时候直接stall住内存申请，直到内核通过各种手段
 把大页搞出来。内存申请返回或者是madvise返回的时候，已经是大页了; defer是随后
 内核会异步的把大页给你搞好; defer+madvise是只对madvise是立即搞定大页，其他的
 情况还是按照defer的来; madvise是只对madvise立即搞定大页。

 第三类接口，配置内核khugepaged线程的一些参数，比如多少长时间做一次扫描, 具体
 可以参考上面的内核文档：/sys/kernel/mm/transparent_hugepage/khugepaged/*

 内核文档里也介绍了THP相关的一些统计参数，比如/proc/vmstat里的thp_*的各个参数，
 /proc/meminfo、/proc/PID/smap里的AnonHugePages。从内核代码里还可以看到有debugfs
 下的接口/sys/kernel/debug/split_huge_pages。

 整个系统使用THP，打开enabled always就好。单独一段内存使用THP，需要用madvise打
 上一个MADV_HUGEPAGE, 类似:
```
	p = mmap(NULL, MEM_SIZE, PROT_READ | PROT_WRITE, MAP_PRIVATE |
		 MAP_ANONYMOUS, -1, 0);
	if (p == MAP_FAILED)
		exit(1);

	ret = madvise(p, MEM_SIZE, MADV_HUGEPAGE);
	if (ret)
		exit(1);

	munmap(p, MEM_SIZE);
```

 实际上, 现在的内核代码和defrag中的定义已经没有严格对应。比如在enable: always,
 defrag: madvise的时候，vma merge还是会把大于2MB的vma扔给khugepaged线程去扫描、
 然后触发huge page collapse。

THP内核代码分析
------------------

 透明巨页的内核配置是CONFIG_TRANSPARENT_HUGEPAGE，代码主要在mm/khugepages.c，
 mm/huge_memory.c，相关的头文件在include/linux/huge_mm.h，include/linux/khugepaged.h。

 THP的初始化在huge_memory.c的hugepage_init(), 这个函数初始化THP, 并且start
 khugepaged内核线程。/sys/kernel/mm/transparent_hugepage/enabled里写always或者
 madvise也可以start khugepaged内核线程。

 我们跟踪代码的从三个地方入手，一个是__transparent_hugepage_enabled，这个是内核
 缺页流程__handle_mm_fault里, 这里会判断系统有没有使能透明大页，如果使能了，会直
 接分配大物理页。另外一个是，系统会启动一个内核线程持续扫描系统里的页，进行大页
 的合并。代码分析依赖5.11-rc4。第三个地方是madvise系统调用。

 mm/memory.c
```
	__handle_mm_fault
	  +-> __transparent_hugepage_enabled
	    /* 忽略细节，可以看到在下面的函数里申请page、安装页表 */
	    +-> create_huge_pmd/pud
	      +-> do_huge_pmd_anonymous_page
	        /* 分配内存 */
	        +-> alloc_hugepage_vma
		/* 安装页表并且处理和其他mm部件的关系 */
		+-> __do_huge_pmd_anonymous_page

	  /* ? */
	  +-> handle_pte_fault
```
 
 khugepaged的线程函数：
```
	khugepaged
	  +-> khugepaged_do_scan
	    /* 一般情况不会进去这里 */
	    +-> collapse_pte_mapped_thp
	    /*
	     * 有这两个入口做小页换成大页的操作, 第一个和file-back的内存有关系，
	     * 下面调用的是: collapse_file。第二个处理匿名页或者是堆上的内存，
	     * 下面调用的是：collapse_huge_page。
	     */
	    +-> khugepaged_scan_file
	    /* 先扫描整个pmd下的页，可以做collapase的时候，最后走到collapse_huge_page */
	    +-> khugepaged_scan_pmd
	      /*
	       * 这个是小页换成大页的核心函数，先分配2MB连续页面，然后断开pmd，
	       * 同时做tlb invalidate，然后把小页的内存copy到大页，然后把pmd页表
	       * 安装上。其中涉及的同步逻辑在最后collapse_huge_page页表同步里
	       * 展开分析。
	       */
	      +-> collapse_huge_page
```
 
 khugepaged的扫描需要内核的其他部分提供需要扫描的对象，这个入口函数是
 khugepaged_enter。可以发现整个系统里有mm/mmap.c、mm/huge_memory.c、mm/shmem.c
 里调用了。

 madvise系统调用，mm/madvise.c
```
	madvise_behavior
	  +-> hugepage_madvise
	   ...
	     /*
	      * 这个函数为对应的mm生成一个mm_slot，把mm_slot添加到khugepaged的
	      * 扫描链表里，然后wakeup khugepaged线程。
	      */
	   +-> __khugepaged_enter
```
 mmap.c里的调用是在mmap或者brk系统调用，在不断的做vma_merge的时候，如果发现有
 vma的range跨越了2MB的连续va，就会把对应的vma交给khugepaged扫描，做大页的替换。

 collapse_huge_page页表同步分析:
```
      /* 分配大页 */
  +-> khugepaged_alloc_page

  +-> mmap_write_lock
  
      /* 做secondory mmu tlbi */
  +-> mmu_notifier_invalidate_range_start

      /* 清空pmd页表项，做cpu tlbi */
  +-> pmdp_collapse_flush

  +-> mmu_notifier_invalidate_range_end

      /* 清空pte页表, free相关页面 */
  +-> __collapse_huge_page_isolate

      /* 把小页里的内容copy到大页 */
  +-> __collapse_huge_page_copy

  [...]

      /* 装大页页表 */
  +-> set_pmd_at

  +-> mmap_write_unlock
```
  注意其他的cpu或者设备可以这个过程中还在写原来的小页内存，这些操作和如上页表
  变动的同步点在上面两个tlbi处，tlbi和随后的barrier可以保证之前正在总线上的相关
  地址操作都完成。这样在barrier后的新访存操作都会触发fault，这些fault都会在
  mmap_write_lock上排队。等到新访问的fault可以执行的时候，发现已经有大页，就可以
  做后续的处理。
