---
title: PCI/SMMU ATS analysis note
tags:
  - 计算机体系结构
  - PCIe
description: This doc is an analysis note about ATS feature in PCIe/SMMU
abbrlink: e89bca2c
date: 2021-07-05 22:42:41
categories:
---

ATS
------

ATS is the short of Access Translation Service. In the PCIe spec, the ATS section
includes ATS, PRI and PASID. Here we only consider ATS.

Generally speaking, ATS just prefetch the PA(physical address) to store in PCIe
device's side. Once this PCIe device sending a memory read/write operation
(e.g memory read/write TLP), it will bring a flag which means that the address
of this operation had already been translated. When this operation passes SMMU,
SMMU will parse this flag and know that this address is a PA and SMMU will do
nothing about this operation, SMMU then will just forward this operation to
read/write memory.

The hardware will do the prefetch automatically. But software must be involved
when we want to disable/invalidate a prefetched PA in PCIe device.
```
          +-------+
          |  CPU  |
          +---+---+
              |
      --------+-----------------------------+----
              |        PA   ^               |
          +---+---+    ^    |            +--+--+
          | SMMU  |    |    |            | DDR |
          +---+---+    |    |            +-----+
              |        VA   |
          +---+---+         |
          |  RP   |         |
          +---+---+         |
              |             |
          +---+---+         PA
          |  EP   |
          +-------+
```

From the above figure, with SMMU enable and without ATS enable, read/write
operation from EP will be with a VA, SMMU help to do the translation from VA to
PA. However, when enabling ATS, PA can be stored in EP ahead, EP can send
read/write operation with a PA directly, which will improve the performance.

Hareware support
-------------------

 For EP that supports ATS must have ATS Extended Capability in its CFG. In ATS
 cap, we have ATS capability registers and ATS controller registers. Now all
 bits in ATS capability registers are RO, STU and Enable in ATS control registers
 are RW. STU means smallest translation unit, which should be matched with SMMU
 supported page size.

 For SMMU, we should enable ATS by IDR0_ATS in SMMU_IDR0.ATS.

 - ATS enable

   We should enable SMMU_IDR0.ATS and enable bit in PCIe device's ATS cap.

 - ATS request

   I think the ATS request which will be followed by a ATS completion is triggered
   automatically. PCIe ATS spec says "Host system software can not modify the ATC",
   which is used to store the PA in PCIe device.

   I think the ATS request will be device specific and be implemented together
   with the DMA of PCIe device. There is only one PCIe EP card(Mellanox ConnectX-5)
   supported ATS, ATS in PCIe device(EP) maybe will implemented together with
   EP's DMA, which means DMA will still get a VA from SMMU as target address,
   however, before DMA operation, EP will firstly send a ATS request to get the
   related PA of above VA, then EP will send DMA read/write operation with this
   PA. But how many does EP send ATS request? For example, if 8K size DMA range
   has been requested, and STU is 4K for system, will EP send 2 ATS request 2 PA?
   This also has relationship with the process of ATS invalidation. From the code
   below, it will invalidate a range of iova as the inputs of arm_smmu_atc_inv_to_cmd
   include iova and size. If the range of iova to iova + size covers multiple
   stu/page, what will happen in the hardware?

 - ATS invalidate

   When VA -> PA changed in SMMU, PA in PCIe device's ATC is not the correct
   address, so host software should invalidate the cached PA by leting SMMU
   send an ATS invalidation request. ATS invalidation completion will be sent
   from the PCIe device later.

   SMMU will use command CMD_ATC_INV to invalidate the cached PA in EP.

software support
-------------------

 For analysis will base on [PATCH 0/7] Add PCI ATS support to SMMUv3, we can
 find this in git://linux-arm.org/linux-jpb.git branch: svm/ats-v1.

 - ATS enable:
```
   arm_smmu_add_device
       --> arm_smmu_enable_ats
               /* will configure stu in this function, stu here comes from smmu */
           --> pci_enable_ats(pdev, stu)
```
 - ATS invalidate:

   There are two places to call ATC_INV:
```
   *    arm_smmu_detach_dev
            --> arm_smmu_atc_inv_master_all

   *    arm_smmu_unmap(struct iommu_domain *domain, unsigned long iova, size_t size)
            --> arm_smmu_atc_inv_domain(smmu_domain, 0, iova, size)
                    /* it has an algrithm to caculate the invalidate page */
                --> arm_smmu_atc_inv_to_cmd(ssid, iova, size, &cmd)
```
```
arm_smmu_unmap input: iommu domain, iova, size
       /*
        * we pick each device in this iommu domain to do the invalidation operation,
        * normally, we have one device in one iommu domain.
        * 
        * here struct arm_smmu_master_data is for every device under a SMMU, and
        * it has been created in arm_smmu_add_device.
        */
    -> arm_smmu_atc_inv_domain  input: smmu_domain, ssid, iova, size
        --> arm_smmu_atc_inv_master: 
```
        
