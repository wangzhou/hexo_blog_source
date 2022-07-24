---
title: Linux dma_map_sg API
tags:
  - Linux内核
  - DMA
  - iommu
description: 本文简介Linux内核DMA接口 dma_map_sg的语意
abbrlink: 463e729a
date: 2021-06-27 18:05:15
---

如linux/Documentation/DMA-API-HOW.txt里提到的:
```
With scatterlists, you map a region gathered from several regions by::

	int i, count = dma_map_sg(dev, sglist, nents, direction);
	struct scatterlist *sg;

	for_each_sg(sglist, sg, count, i) {
		hw_address[i] = sg_dma_address(sg);
		hw_len[i] = sg_dma_len(sg);
	}

where nents is the number of entries in the sglist.

The implementation is free to merge several consecutive sglist entries
into one (e.g. if DMA mapping is done with PAGE_SIZE granularity, any
consecutive sglist entries can be merged into one provided the first one
ends and the second one starts on a page boundary - in fact this is a huge
advantage for cards which either cannot do scatter-gather or have very
limited number of scatter-gather entries) and returns the actual number
of sg entries it mapped them to. On failure 0 is returned.

Then you should loop count times (note: this can be less than nents times)
and use sg_dma_address() and sg_dma_len() macros where you previously
accessed sg->address and sg->length as shown above.

To unmap a scatterlist, just call::

	dma_unmap_sg(dev, sglist, nents, direction);

Again, make sure DMA activity has already finished.

.. note::

	The 'nents' argument to the dma_unmap_sg call must be
	the _same_ one you passed into the dma_map_sg call,
	it should _NOT_ be the 'count' value _returned_ from the
	dma_map_sg call.
```
dma_map_sg对一个sgl做map的时候，返回的map的sge的个数可能是小于输入sgl的sge的个数
的，这是因为连续页做了合并操作，在有IOMMU的情况下，也可能是因为IOMMU的存在把一段
连续的IOVA映射成了一些离散的物理地址块，而这些离散的物理地址块正是前面的sgl输入,
这个连续的IOVA正式dma_map_sg的输出。
