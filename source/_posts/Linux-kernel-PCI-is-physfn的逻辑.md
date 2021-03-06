---
title: Linux kernel PCI is_physfn的逻辑
tags:
  - Linux内核
  - PCIe
description: 本文梳理Linux PCI驱动里is_physfn的逻辑
abbrlink: 522cb356
date: 2021-06-27 18:04:07
---

Linux内核里struct pci_dev里有一个叫is_physfn的域段, 从名字上来看，这个域段可以
用来表示一个pci设备是不是PF。对应的有一个is_virtfn的域段。

显然在驱动里如果有的操作需要区分PF和VF, 我们可以用这个域段作为判断依据。但是，
实际上如果你去看pci代码的实现，他的语义要以SRIOV cap存在为前提的。

他的调用关系是：
```
	pci_iov_init
	  +-> pci_find_ext_capability(dev, PCI_EXT_CAP_ID_SRIOV)
	    +-> sriov_init
	      +-> dev->is_physfn = 1
```

他的语义是pci设备只有在SRIOV cap存在的时候，才有区分PF和VF的必要, 才会去设定
is_physfn。

所以，如果我们用一套驱动同时支持PF和VF, 需要区分PF和VF的操作的时候，如果这时
系统里有SRIOV cap时，依然可以用is_phyfn这个标记。当系统里没有SRIOV cap时，is_physfn
和is_virtfn都不会设置，这时判断就会错。这样的场景发生在PF和VF公用一套驱动，而且
VF直通到guest里工作之时。

这种情况下，我们一般要在设备驱动里增加新的标记位，明确的表示这是一个PF还是一个VF。
这个标记位只和设备相关，和SRIOV cap无关。
