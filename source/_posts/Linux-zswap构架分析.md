---
title: Linux zswap构架分析
tags:
  - Linux内核
  - zswap
description: >-
  本文简单分析zswap的软件构架，为在zswap框架中添加crypto acomp的支持做准备。 关于zswap的基本介绍和使用可以参考:
  https://wangzhou.github.io/使用linux-zswap/。本文的分析基于Linux主线5.5-rc1。
abbrlink: 3bd4e7ca
date: 2021-06-27 18:02:12
---

基本模型
-----------

 - zswap初始化流程

     1. 为系统里的每一个cpu core分配per cpu dst内存。

     2. 为每个cpu core分配内核压缩解压缩的上下文。

     3. 创建zpool, 并根据用户态模块参数选择zpool后端使用的内存分配器。创建
        zpool的时候需要给zpool注册一个evict的回调函数，这个函数用于在zpool的
	后端内存分配器没有内存的时候, 把zpool里的压缩内存向swap设备写入。可以
	看到现在的evict在向swap设备写数据的时候还要先把压缩的数据解压缩，如果写
	入swap设备的数据将来被使用，重新加载回内存的代码路径是标准的缺页流程，
	和zswap没有关系。这个步骤整体上把创建出来的各种基础数据结构封装在一个
	struct zswap_pool的结构中, 上面步骤里的per cpu压缩解压缩上下文也放在了
	zswap_pool。

     4. 注册frontswap的回调函数。根据内核Documentation/vm/frontswap.rst，store
        用于把swap页存入zpool, load用于从zpool重新加载swap的页，invalidate_*
	用于把zpool里存放的压缩页面释放。

 - 核心数据结构
```
       struct zswap_pool

       swap page ---> frontswap
                         .store
			 .load
			 .invalidate_page
			 .invalidate_area
			 .init
```
   zswap初始化的时候注册frontswap的一个实例，frontswap的各个回调中使用zpool接口
   存储压缩的内存，zpool接口的后端可以是不同的专用于存储压缩内存的内存分配器。

 - 对用户态的接口

   1. zswap用多个模块参数用来配置zswap的参数: zpool的后端内存分配器是可以选的
      (默认是zbud); 压缩解压算法(默认是LZO)，测试硬件offload的时候，这里要选
      则相应的算法; zpool占内存的大小; 对相同页的优化处理。

      这些参数在/sys/module/zswap/parameters/下也可以配置。

   2. 在/sys/kernel/debug/zswap/下有zswap的相关统计项。

 - 对内核crypto comp的接口

   1. 当前代码使用的是内核crypto comp压缩解压缩接口，没有使用crypto acomp接口。

实现细节
-----------
  不清楚这里的逻辑，为什么要一个zswap pool的链表。
```
  __zswap_pool_create_fallback(void)
    +-> zswap_pool_create(char *type, char *compressor)
      +-> cpuhp_state_add_instance(CPUHP_MM_ZSWP_POOL_PREPARE, &pool->node)

  list_add(&pool->list, &zswap_pools);
```
  这里选一个zswap_frontswap_store的实现分析:
```
  zswap_frontswap_store
    +-> entry = zswap_entry_cache_alloc(GFP_KERNEL)
    	分配一个zswap_entry, 用来描述要压缩存储的一个页。
    +-> crypto_comp_compress
        调用crypto comp API做压缩。
    +-> zpool_malloc(entry->pool->zpool, hlen + dlen, gfp, &handle)
        从zpool里分配一段内存, 用来存压缩的页。
    +-> zpool_map_handle(entry->pool->zpool, handle, ZPOOL_MM_RW)
    +-> memcpy(buf, &zhdr, hlen)
    +-> memcpy(buf + hlen, dst, dlen)
        把压缩后的swap页存入zpool。
    +-> zswap_rb_insert(&tree->rbroot, entry, &dupentry)
        把对应的swap页的entry插入一个红黑树，以后load，invalidate等可以从这个
	红黑树查找对应的swap页。
```
    从上面store函数的分析中可以看到，因为整个设计都是基于per cpu的，所以做
    crypto_comp_compress的时候都是关闭抢占的(是否要关闭调度), 这和acomp的基本
    使用方式是不兼容的，在crypto testmgr.c里acomp的test case是wait等待任务完成
    的。还有一点，压缩完的数据需要copy到zpool的内存里。
