---
title: PCI FLR analysis
tags:
  - PCIe
  - 计算机体系结构
description: >-
  PCI协议里有function level reset的定义，实现FLR的设备可以提供function级别的复位。
  本文梳理linux内核里和FLR相关的流程。本文可以作为设备复位设计的一个参考。
abbrlink: faf884c6
date: 2021-06-29 00:00:56
categories:
---

关于FLR的硬件操作比较简单, 相关的硬件有:
  - 配置空间里device cap里的FLR capability bit, 这个表示设备是否支持FLR。
  - 配置空间里device control里的BCR_FLR bit, 写这个bit可以触发FLR。

检测是否支持FLR
```
/* drivers/pci/pci.c */
pcie_has_flr(struct pci_dev *dev)
```

Linux kernel里pcie_flr会被下面的三个函数调用到, 触发FLR
```
/* drivers/pci/pci.c: below tree functions will call __pci_reset_function_locked */
pci_reset_function(struct pci_dev *dev)
pci_reset_function_locked(struct pci_dev *dev)
pci_try_reset_function(struct pci_dev *dev)
    => __pci_reset_function_locked(struct pci_dev *dev)
        -> pcie_flr(struct pci_dev *dev)
```

下面来看上面的三个函数在哪里会用到。

首先通过这个设备的sysfs接口可以触发FLR:
```
/* drivers/pci/pci-sysfs.c */
reset_store
    -> pci_reset_function(struct pci_dev *dev)
```

另外，vfio里也提供的接口，可以供用户触发FLR。这些接口包括，vfio设备的enable,
disable, 以及一个vfio设备相关的ioctl。
```
/* drivers/vfio/pci/vfio_pci_config.c */
vfio_exp_config_write
    -> pci_try_reset_function

/* drivers/vfio/pci/vfio_pci.c */
vfio_pci_enable
vfio_pci_disable
vfio_pci_ioctl (cmd == VFIO_DEVICE_RESET)
    => pci_try_reset_function(pdev);
```

单独的FLR操作需要配合整个reset流程工作, 在上面的调用pcie_flr的函数里，他们基本
的处理流程都是, 先做reset_prepare, 再触发FLR，最后做reset后的恢复:
```
the logic of pci_reset_function and its brother functions are:
	- reset_prepare
	- flr operation if supporting flr
	- reset_done
```

reset_prepare and reset_done callbacks are stored in pci_driver's pci_error_handlers,
these callbacks should be offered by your device driver:
```
struct pci_driver {
	...
	const struct pci_error_handlers {
		...
		void (*reset_prepare)(struct pci_dev *dev);
		void (*reset_done)(struct pci_dev *dev);
		...
	}
	...
}
```

从上面的代码分析中可以看出，linux内核中的flr流程并不涉及到软件中设备结构的销毁。
所以，只要在发生flr这段时间不去下发硬件请求，用lspci看，设备一直都在。一般硬件
在实现flr的时候，在硬件层面都有pf到vf的通知方式, 这样可以保证在pf flr时候通知到
flr做必要的处理，当pf flr完成后，可以通知vf驱动做必要的硬件配置上的恢复。
