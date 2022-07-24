---
title: pci设备直通qemu相关的RAS处理
tags:
  - PCIe
  - QEMU
  - RAS
description: 本文介绍linux系统上当一个pf设备有pci aer错误的时候，对应的直通到qemu里的vf的行为。分析基于当前的主线内核v5.0-rc6。
abbrlink: d34d1e70
date: 2021-06-28 23:57:03
categories:
---

v5.0-rc6里pci aer的处理逻辑是，当pf有aer时会扫描pf父总线下的所有function, 并
调用对应function的pci_driver->err_handler->err_detected/slot_reset函数
(如果有这些回调，这里不展开描述)。

基于上面的逻辑，如果正好有vf通过vfio驱动直通到qemu, 那么vfio-pci驱动里的
pci_driver->err_handler->err_detected(具体函数是: vfio_pci_aer_err_detected)
将被调用到。vfio中的err_detected函数向qemu发一个eventfd消息。qemu收到该消息的
时候会终止qemu进程(abort)
