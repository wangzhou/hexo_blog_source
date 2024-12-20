---
title: SMMU TLBI分析
tags:
  - SMMU
  - Linux内核
  - 计算机体系结构
description: 本文分析SMMU-v3 tlbi相关的硬件语义和linux相关驱动代码的实现。
abbrlink: e96c7a44
date: 2021-06-19 10:28:42
---

SMMU E2H支持
---------------

 SMMU SVA支持的patchset里有一个补丁：iommu/arm-smmu-v3: Add support for VHE
 这个补丁的基本逻辑是, CPU支持VHE，且内核支持时，host kernel工作在EL2，在SVA
 下，因为需要做DVM tlbi广播，且tlbi广播在同一个EL里生效，所以需要SMMU这边的TLB
 也打上EL2的标记。这个是通过设置设备对应的STE里的STRW生效的，这个配置对SMMU tlb
 影响是全局的。

 这里面还有三个逻辑：

 1. 要使得DVM tlbi的广播生效，还要设置下SMMU CR2.E2H寄存器(使得ASID标记生效)。
    这个补丁里配置了E2H。

 2. 因为STRW的配置是STE全局的，SMMU管理的内核地址的TLB也打上了EL2的标记，这就
    需要把原来对内核地址的tlbi命令从TLBI_NH改成TLBI_VA。

 3. 原来invalidate内核地址的tlbi命令又可以分为TLBI_VA和TLBI_ASID，所以，上面
    一步还要把原来的TLBI_NH_VA/TLBI_NH_ASID都改成TLBI_EL2_VA/TLBI_EL2_ASID。

 这个补丁做了上述的修改，使得SVA和内核地址的tlbi在VHE使能的时候都可以正常使用。

SMMU tlbi相关命令
--------------------

 SMMU spec 4.4章节介绍了tlbi相关的command, 上面提到的相关tlbi命令在这一章节中
 都有详细的描述。EL2和NH的标记在上面已经介绍。关于VA和ASID标记的区别在于做tlb
 无效化的时候作用的范围是不一样的。简单的讲带ASID的tlbi会无效化对应asid的tlb，
 理论上讲，SVA需要用这个，但是现在SVA用的是DVM，还没有用到带ASID的tlbi，在实际
 分析的时候，还看到no-strict时使用的是带ASID的tlbi，这个下面结合代码来分析。
 直接带VA的tlbi可以对一段地址做tlb无效化，相关的command域段里的值有: scale,
 num, asid, address, tg, ttl, leaf。当SMMU硬件支持tlbi by range(IDR3_RIL = 1)时，
 还可以用带VA的tlbi实现tlbi by range.

linux SMMU驱动里的实现
-------------------------

 因为SVA没有用tlbi，所以我们主要分析内核dma内存中的tlbi就好了。
```
  /* drivers/iommu/dma-iommu.c */
  iommu_dma_free
    +-> __iommu_dma_unmap
      /* 用来init iotlb_gather，为以后做tlbi by range做准备 */
      +-> iommu_iotlb_gather_init
      +-> iommu_unmap_fast
        ...
          +-> arm_smmu_unmap
            /* drivers/iommu/io-pgtable-arm.c, 这一步跳的有点大:) */
            +-> arm_lpae_unmap 
              +-> __arm_lpae_unmap
                +-> io_pgtable_tlb_add_page
                  /* drivers/iommu/arm/arm-smmu-v3/arm-smmu-v3.c */
                  +-> arm_smmu_tlb_inv_page_nosync 
                    /*
                     * 不断的收集tlbi的range，遇到不相连的range就做tlbi
                     */
                    +-> iommu_iotlb_gather_add_page
                      +-> iommu_tlb_sync
                        /* arm-smmu-v3.c */
                        +-> arm_smmu_iotlb_sync
                          /* 使用VA tlbi，range invalidate的逻辑也在这里 */
                          +-> arm_smmu_tlb_inv_range

      +-> iommu_tlb_sync (如果没有fq_domain，即使用strict mode)
        +-> arm_smmu_iotlb_sync
      ...
```
 使用no-strict模式会起一个定时器, 在队列满或者定时器到时做tlbi。
```
 /* drivers/iommu/iova.c */
 iova_domain_flush
   /* drivers/iommu/dma-iommu.c */
   +-> iommu_dma_flush_iotlb_all
     /* arm-smmu-v3.c */
     +-> arm_smmu_flush_iotlb_all
       /* 这个函数里会下发带ASID的tlb */
       +-> arm_smmu_tlb_inv_context
```
