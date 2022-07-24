---
title: SMMU stalled transaction with device
tags:
  - SMMU
description: >-
  This doc shares the logic of SMMU translation terminate with device. We will
  talk hardware operations and software code. This doc is based on code:
  https://github.com/Linaro/linux-kernel-warpdrive branch: zip-devel
abbrlink: 4e21b5c9
date: 2021-06-27 17:57:29
---

Currently when process dies, there is no way(software callback function) to
control device DMA stop. The software flow is as below:
```
      +-> exit_mmap
        +-> mmu_notifier_release
	// iommu-sva.c: 
	iommu_mmu_notifier_ops
	  +-> io_mm_release
	    +-> io_mm->ops->clear
	    // arm-smmu-v3.c:
	    arm_smmu_mm_ops->arm_smmu_mm_clear
	      +-> arm_smmu_write_ctx_desc
	        +-> __arm_smmu_write_ctx_desc
		  +-> if (cd == &invalid_cd) 
		    +-> CD.S = CD.R = 0; EPD0 = 1;
		  +-> arm_smmu_sync_cd
		    +-> CD CFGI
```
This will stop smmu stall and stop recore events in SMMU event q, and prevent
page table walk. Note that: CD.S = CD.R = 0; EPD0 = 1 does not prevent translation.
So if there is a transaction from device can SMMU TLB hit, this transaction can
still be sent out SMMU to system bus. Another Note: CD.S = CD.R = 0; EPD0 = 1
works after CD CFGI.

Above flow may happen together with normal SMMU fault flow, consider this case:
We use "exit" as above flow, and "dma" as normal SMMU fault flow.

  SMMU page fault triggered by device DMA(dma) -> software fault handling(dma) -> 
  SMMU CD modification(exit) -> software sending CMD resume retry(dma).

In above case, the last CMD resume retry will fail, hardware should terminate
former stalled transaction, otherwise there will be transaction which is stalled
in SMMU hardware for ever, which will cause device sending the transaction die.

sva unbind flow:
```
	iommu_sva_unbind_device
	  [...]
	  +-> arm_smmu_write_ctx_desc
	    +-> __arm_smmu_write_ctx_desc
	      +-> if (!cd)
	        +-> cd val = 0;
	      +-> arm_smmu_sync_cd
	        +-> CD CFGI
```
This will invalid CD(CD.valid = 0), if device DMA arrive SMMU here, maybe
a SMMU bad CD event(0xa) will report. however, in uacce driver, the release
callback will firstly stop q, then do sva unbind. So if q is stopped, we will
not see SMMU bad CD event.
