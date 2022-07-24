---
title: qemu iommu模拟思路分析
tags:
  - QEMU
  - iommu
description: 本文介绍qemu对iommu模拟的思路，我们以qemu里smmuv3设备驱动为例。
abbrlink: 5230085a
date: 2021-08-21 09:55:36
categories:
---

smmuv3的父设备的代码里有：
```
 smmu_base_realize
   +-> pci_setup_iommu
     +-> bus->iommu_fn = smmu_find_add_as;
     +-> bus->iommu_opaque = SMMUState;
```
把一个获取as的方法放到了PCIBus里，一般的这个方法是有IOMMU驱动定义，IOMMU驱动为
IOMMU管理的设备在IOMMU驱动里建立一个数据结构存储相关信息，这个信息里就有设备对应
的AS。通过这个方法，使用设备的bdf作为输入，后续设备可有拿到他自己对应的AS。
设备随后发dma的时候通过他自己的AS得到smmu里的translate函数然后做翻译。

我们可以看一个具体的intel e1000的虚拟网卡：qemu/hw/net/e1000.c
在这个设备注册的时候会调用：
```
 /* qemu/hw/pci/pci.c */
 do_pci_register_device
   +->pci_init_bus_master
     +-> AddressSpace *dma_as = pci_device_iommu_address_space
           /* 这个iommu_fn就是上面的smmu_find_add_as */
       +-> iommu_bus->iommu_fn(PCIBus, SMMUState, devfn)
       /* 在哪里把 dma_as存在dev->bus_master_as里的？*/
```

设备模拟一个dma读写的实现：
```
 pci_dma_read/write
   ...
     /* pci_get_address_space_space得到dev->bus_master_as */
     +-> dma_memory_rw(pci_get_address_space(dev), ...)
```
可以看到这个函数后面的调用链里会最终调用到iommu里的translate函数，然后对翻译
都的地址读写。

对于普通的DMA，如上已经够了。但是，对于带PASID的翻译请求，现在还没有支持的逻辑。
我们查看具体的translate函数的入参：
```
    IOMMUTLBEntry (*translate)(IOMMUMemoryRegion *iommu, hwaddr addr,
                               IOMMUAccessFlags flag, int iommu_idx);
```
我们可以用最后一个参数iommu_idx去传递pasid给具体的翻译函数，这个iommu_idx不能直接
使用，在使用前需要通过attrs_to_index这个函数把MemTxAttrs attrs翻译成iommu_idex。
所以，想要在当前的smmu驱动里支持pasid，就要在smmu驱动的imrc里实现attrs_to_index
的回调函数。如果我们的输入把attrs数值上相等映射成iommu_idx，attrs_to_index可以
是如下这样：
```
static int smmuv3_attrs_to_index(IOMMUMemoryRegion *iommu, MemTxAttrs attrs)
{
    return attrs.pasid & 0xffff;
}
```
对应的我们给设备驱动可以提供这样的带PASID的DMA读写函数：
```
static inline int pci_dma_rw_pasid(PCIDevice *dev, dma_addr_t addr,
                             void *buf, dma_addr_t len, DMADirection dir,
                             uint32_t pasid)
{
    MemTxAttrs attr = { .pasid = 0xffff & pasid };

    return dma_memory_rw_attr(pci_get_address_space(dev), addr, buf, len,
			      dir, attr);
}
```
