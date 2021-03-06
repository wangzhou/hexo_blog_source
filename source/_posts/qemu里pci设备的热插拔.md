---
title: qemu里pci设备的热插拔
tags:
  - QEMU
  - PCIe
  - 虚拟化
description: 本文讨论linux系统中pci设备直通给qemu中涉及的pci热插拔问题。
abbrlink: 9a1b7375
date: 2021-06-28 23:56:18
categories:
---

场景是一个pcie设备的vf通过vfio直通给qemu使用，这时如果我们在host上通过sysfs把
对应的vf disable掉。

正常来讲，qemu里的vf pci设备会表现为一个pci设备热拔出的行为。与之相对应的设置为：

 1. guest kernel的配置里要打开pci hotplug: CONFIG_HOTPLUG_PCI_PCIE.

 2. guest kernel的启动cmdline里要是能pci native hotplug, 加上pcie_port=native

 3. 启动qemu的时候，需要把直通上来的pci vf挂到一个支持pci热插拔的pci桥下面:
    比如在qemu里挂接一个ioh3420的pci桥，然后再把直通的vf挂在这个桥下。

 4. 本文的测试是在主线linux v5.0-rc6上做的，这个版本有一个pci hotplug的bug，这个
    bug会导致虚拟机里vf无法被热拔。相关的fix补丁已经被pci maintainer ack, 会合
    入v5.1主线版本。如果是在v5.0, 以及之前的内核的版本上测试，需要确认这个补丁
    是否合入:
        [PATCH RESEND] PCI: pciehp: Assign ctrl->slot_ctrl before writing it to hardware

 综合以上，如下的qemu启动命令，配合正确的kernel，可以支持qemu里直通vf的pci热拔
 操作:(这里已ARM64平台为例)
```
	qemu-system-aarch64 -machine virt,gic_version=3 -enable-kvm -cpu host \
	-m 1024 -kernel ./Image -initrd ./minifs.cpio.gz -nographic -append \
	"rdinit=init console=ttyAMA0 earlycon=pl011,0x9000000 pcie_ports=native" \
	-device ioh3420,id=root_port \
	-device vfio-pci,host=0000:75:00.1,bus=root_port
```

具体可以这样测试:(已HiSilicon D06 zip engine为例)

 1. 在host上把vf和vfio驱动绑定:
```
	echo 1 > /sys/devices/pci0000:74/0000:74:00.0/0000:75:00.0/sriov_numvfs
	echo 0000:75:00.1 > /sys/bus/pci/drivers/hisi_zip/unbind
	echo vfio-pci > /sys/devices/pci0000:74/0000:74:00.0/0000:75:00.1/driver_override
	echo 0000:75:00.1 > /sys/bus/pci/drivers_probe
```

 2. 启动qemu: 同上面的命令

 3. 在host上disable vf:
```
	echo 0 > /sys/devices/pci0000:74/0000:74:00.0/0000:75:00.0/sriov_numvfs
```
    
 可以看到在qemu里，vf表现为一个pci热拔的动作:
```
    (fix me: add log, 目前看上面的命令会在host上挂住)
```

为了使得这篇介绍完整，对于qemu里pci设备的热插，可以这样来做:

 1. 启动qemu后按ctrl a + c 进入qemu monitor(启动qemu的时候带ioh3420但是不带VF设备)

 2. 在qemu monitor里: device_add vfio-pci,host=0000:75:00.1,bus=root_port

 这样可以把已经和vfio驱动绑定的VF PCI热插到qemu
 (fix me: lspci看不到新设备，但是在qemu monitor里info pci可以看到新插入的设备)
