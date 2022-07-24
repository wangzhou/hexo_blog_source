---
title: How to assign more than 31 VFs to one VM
tags:
  - QEMU
  - PCIe
description: 本文介绍通过给虚拟机加入PCIe switch扩展总线的方法，通过这样的方法， 可以实现把多个VF挂接给一个虚拟机
abbrlink: 5d08c89f
date: 2021-07-11 23:31:24
categories:
---

用QEMU模拟PCIe设备的时候，一般最多可以在系统中配置31个PCIe设备。比如，我们有这样
的QEMU启动参数配置：
```
       qemu-system-aarch64 \
       -machine virt,gic-version=3 \
       -enable-kvm \
       -cpu host \
       -m 1024 \
       -kernel ./Image \
       -initrd ./minifs.cpio.gz \
       -nographic \
       -net none \
       -device vfio-pci,host=0002:81:10.0,id=net0 \
       -device vfio-pci,host=0002:81:20.0,id=net1 \
```
上面的配置中，我们使用了host上的两个82599网卡的vf: 0002:81:10.0, 0002:81:20.0,
把它们直通到了guest上，可以看到这两个vf在guest上会直接连到pci bus 0上。按照同样
的方法，我们可以给一个虚拟机继续增加网口。但是这样的方式增加最多只能到31个vf[1]

这是因为PCIe中规定一条总线下最多只能接32个设备。

为了在一个虚拟机上接入更多的设备，我们可以接入一个PCIe switch，在switch的下游
端口上再接需要的PCIe设备。比如，我有可以如下启动一个QEMU虚拟机：

```
       qemu-system-aarch64 \
       -machine virt,gic-version=3 \
       -enable-kvm \
       -cpu host \
       -m 1024 \
       -kernel ./Image \
       -initrd ./minifs.cpio.gz \
       -nographic \
       -net none \
       -device ioh3420,id=root_port1 \
       -device x3130-upstream,id=upstream_port1,bus=root_port1 \
       -device xio3130-downstream,id=downstream_port1,bus=upstream_port1,chassis=1,slot=1 \
       -device xio3130-downstream,id=downstream_port2,bus=upstream_port1,chassis=1,slot=2 \
       -device xio3130-downstream,id=downstream_port3,bus=upstream_port1,chassis=1,slot=3 \
       -device xio3130-downstream,id=downstream_port4,bus=upstream_port1,chassis=1,slot=4 \
       -device xio3130-downstream,id=downstream_port5,bus=upstream_port1,chassis=1,slot=5 \
       -device vfio-pci,host=0002:81:10.0,id=net0,bus=downstream_port1 \
       -append 'console=ttyAMA0 root=/dev/vda2' \
       -nographic \
```
上面的命令行参数可以搭建一个如下图所示的PCIe switch
```
   pcie.0 bus
   --------------------------------------------------------------------------
                                |
                          -------------
                          | Root Port1|
                          -------------
       -------------------------|-------------------------------------------
       |                 -----------------------------------------         |
       |    PCI Express  | Upstream Port1                        |         |
       |      Switch     -----------------------------------------         |
       |                  |            |                                   |
       |    -------------------    -------------------                     |
       |    | Downstream Port1|    | Downstream Port2|       ....          |
       |    -------------------    -------------------                     |
       -------------|-----------------------|-------------------------------
              ------------                                                 
              | PCIe Dev | vfio-pci device
              ------------


```
[1] Fix me: 为什么是31个不是32个?
[2] qemu/docs/pcie.txt
