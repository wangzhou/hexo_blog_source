---
title: Linux IO DMA地址映射流程分析
tags:
  - Linux内核
  - DMA
  - iommu
  - SMMU
  - 页表
description: >-
  本文分析Linux内核中IO DMA地址映射的流程，其中涉及到的具体iommu硬件以及驱动， 我们分析ARM
  SMMUv3的相关的实现。分析内容基于Linux主线5.13-rc4。
abbrlink: eecc1f14
date: 2021-06-27 23:17:20
---

SMMU页表以及相关配置的初始化流程
-----------------------------------

 iommu_ops里的attach_dev回调用来在SMMU一侧为master设备建立各种数据结构。如下是
 arm_smmu_attach_dev的流程:
```
arm_smmu_attach_dev
  +-> arm_smmu_domain_finalise
        /*
         * 目的是创建smmu_domain里的pgtbl_ops， 这个结构的原型是struct io_pgtable_ops
         * struct io_pgtable_ops
         *   +-> map   
         *   +-> unmap
         *   +-> iova_to_phys
         */
    +-> alloc_io_pgtable_ops
        /*
         * 以64bit s1为例子， 如下函数初始化页表的pgd， 并且初始化页表map/unmap的
         * 操作函数
         */
      +-> arm_64_lpae_alloc_pgtable_s1

            /* 主要创建页表操作函数 */
        +-> arm_lpae_alloc_pgtable
          +-> map = arm_lpae_map
          +-> unmap = arm_lpae_unmap
          +-> iova_to_phys = arm_lpae_iova_to_phys

            /* 创建pgd */
        +-> __arm_lpae_alloc_pages

            /* 得到页表基地址 */
        +-> cfg->arm_lpae_s1_cfg.ttbr = virt_to_phys(data->pgd);

        /* 收尾的配置再搞下，目前是用来配置CD表 */
    +-> finalise_stage_fn
        /* 得到的io_pgtable_ops存放到smmu_domain中 */
    +-> smmu_domain->pgtbl_ops

  +-> arm_smmu_install_ste_for_dev
```

map流程分析
--------------

 我们从内核DMA API接口向下跟踪，观察dma内存的申请和map的流程。以dma_alloc_coherent
 为例分析，这个接口按照用户的请求申请内存，返回CPU虚拟地址和iova。
```
 dma_alloc_coherent
   +-> dma_alloc_attrs /* kernel/dma/mapping.c */
     +-> iommu_dma_alloc /* drivers/iommu/dma-iommu.c */
           /*
            * 如下是内存分配和map的主体逻辑，大概可以分成两块。第一块是iomm_dma_alloc_remap，
            * 这个内存分配和map在这个函数里一起完成，第二块是其余的逻辑，这部分逻辑把分配
            * 内存和map分开了。第二部分里又有从dma pool里分内存和直接分内存，我们不分析
            * dma pool里的case。
            *
            * 如上的case1和case3的核心区别是有没有开DMA_REMAP的内核配置，对应到具体的实现
            * 是，REMAP的情况可以申请不连续的物理页面，调用remap函数得到连续的CPU虚拟地址。
            * 可以看到REMAP的情况才真正支持大范围的dma地址。如果REMAP没有开，也就是case3，
            * iommu_dma_alloc_pages中实际是调用伙伴系统的接口(不考虑CMA的情况)，受MAX_ORDER
            * 的影响，一次可分配的连续物理内存是有限制的。
            */
       +-> iommu_dma_alloc_remap
             /*
              * 根据size分配物理页面，多次调用伙伴系统接口分配不连续的物理页面块。同时
              * 这个函数还做了iommu的map。我们仔细看下这个函数的细节。
              */
         +-> __iommu_dma_alloc_noncontiguous
               /* 这个风骚的bit运算取到的是最小一级的数值， 一般最小一级就是系统页大小 */
           +-> min_size = alloc_sizes & -alloc_sizes;
               /*
                * 分配的算法在如下的函数里，count是需要分配页面的个数，这里的页是指系统
                * 页大小。order_mask是页表里各级block的大小的mask，显然拿到这个信息是为了
                * 分配的时候尽量从block分配，这个信息从iommu_domain的pgsize_bitmap中得到，
                * pgsize_bitmap和具体的页表实现有关，在具体的iommu驱动里赋值，比如ARM的
                * SMMUv3在4K页大小下，他的各级block大小是4K、2M和1G，所以，pgsize_bitmap
                * 是 SZ_4K | SZ_2M | SZ_1G。
                */
           +-> __iommu_dma_alloc_pages(...， count， order_mask， ...)
                 /*
                  * 这段while循环是分配的主逻辑，通过位运算计算每次分配内存的大小。
                  * (2U << __fls(count) - 1)得到count的mask，比如count是0b1000，
                  * mask就是0b1111， mask和order_mask相与，取出最高bit，就是针对
                  * 当前count可以分配内存的最大block size，然后调用伙伴系统的接口
                  * 去分配连续的物理内存。然后，跳出循环，更新下次需要分配的count，
                  * 把本次分配的物理内存一页一页的放到输出pages数组里。虽然分配
                  * 出的可以是物理地址连续的一个block，但是输出还是已page保存的。
                  */
             +-> while (count) {...}
               /* 分配iova */
           +-> iommu_dam_alloc_iova
               /*
                * 把如上分配的物理页用一个sgl的数据组合起来，注意连续的物理也会
                * 又合并到一个sgl节点里。后面的iommu_map就可以把一个block映射到
                * 一个页表的block里。不过具体的map逻辑还要在具体iommu驱动的map
                * 回调函数中实现。从这里的分析可以看出，iommu驱动map回调函数输入
                * 的size值并不是一定是page size大小。
                */
           +-> sg_alloc_table_from_pages
               /* 把iova到物理页面的map建立好 */
           +-> iommu_map_sg_atomic
        
             /* 把离散的物理页面remap到连续的CPU虚拟地址上 */
         +-> dma_common_pages_remap 

       +-> iommu_dma_alloc_pages

       +-> __iommu_dma_map
         [...]
             /* 可以看到这个函数的while循环里也是如上从最大block分配的类似算法 */
         +-> __iommu_map
```
```
 /* drivers/iommu/io-pgtbl-arm.c */
 /*
  * 这个函数就是具体做页表映射的函数，函数的输入iova，paddr， size，iomm_prot已经
  * 表示了要映射地址的va， pa， size和属性。这里iova和paddr的具体分配在上层的dma
  * 框架里已经搞定。lvl是和ARM SMMUv3页表level相关的一个参数，不同页大小、VA位数
  * stage对应的页表level以及起始level有不一样的情况。比如，如下的48bit、4K page
  * size的情况，就是有level0/1/2/3四级页表。__arm_lpae_map具体要做的把一个给定
  * map参数的翻译添加到页表里。
  */
 arm_smmu_map
   +-> arm_lpae_map
     +-> __arm_lpae_map(...， iova， paddr， size， prot， lvl， ptep， ...)
```
 __arm_lpae_map的实现比较直白，就是递归的创建页表。完全按照上层给的page或者block
 的map来做页表映射。

页表相关的细节
-----------------
```
      level              0       1        3        3
      block size                 1G       2M       4K
  +--------+--------+--------+--------+--------+--------+--------+
  |63    56|55    48|47    39|38    30|29    21|20    12|11     0|
  +--------+--------+--------+--------+--------+--------+--------+
   |                 |         |         |         |         |
   |                 |         |         |         |         v
   |                 |         |         |         |   [11:0]  in-page offset
   |                 |         |         |         +-> [20:12] L3 index
   |                 |         |         +-----------> [29:21] L2 index
   |                 |         +---------------------> [38:30] L1 index
   |                 +-------------------------------> [47:39] L0 index
   +-------------------------------------------------> [63] TTBR0/1
```
 如上是一个ARM64(SMMUv3)48bit、4K page size的VA用来做每级页表索引的划分, 这个划分
 比较常见riscv sv39也是这样的划分，只不过是少了最高的一级。在这样的划分下，每一级
 页表都有512个entry，如果一个页表项是64bit，每一级页表的每个table就正好占用4KB内存。
