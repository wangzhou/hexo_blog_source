---
title: qemu PCIe设备增加pasid capability
tags:
  - QEMU
  - PCIe
description: 本文介绍给一个qemu pcie设备增加pasid capability要注意的问题。
abbrlink: f921b9b3
date: 2021-07-26 14:10:39
categories:
---

首先pasid cap是一个PCIe extended的cap，它的位置应该在PCIe配置空间0x100开始(包括)
往后的空间上。

在qemu的启动命令里直接加一个PCI设备，qemu把它看作的是一个PCI设备，用lspci看到的
配置空间只有0x0~0xff。qemu里对于PCI和PCIe设备是分开对待的，如果要接入一个PCIe设备，
需要先在根总线下接一个pcie_port，然后在pcie_port下在接入PCIe设备:
```
	-device pcie-root-port,id=root_port,bus=pcie.0 \
	-device ghms_pci,bus=root_port
```

为了使的接入的设备是一个PCIe设备，设备的TypeInfo中的接口应该定义成PCIe:
```
	static InterfaceInfo ghms_pci_if[] = {
	    { INTERFACE_PCIE_DEVICE },
	    { },
	};
```

下面就在设备class的realize函数里增加pasid cap的初始化代码，目前qemu代码(5.1.50)
里还没有直接可以调用的函数，我们仿照其他的cap，在hw/pci/pcie.c里给pasid加上一个
初始化函数：
```
	void pcie_pasid_init(PCIDevice *dev, uint16_t offset)
	{
	    pcie_add_capability(dev, PCI_EXT_CAP_ID_PASID, PCI_PASID_VER, offset,
	                        PCI_PASID_SIZEOF);
	    dev->exp.pasid_cap = offset;
	
	    /* 把pasid max bit配置成了2^4 - 1 = 15 */
	    pci_set_word(dev->config + offset + PCI_PASID_CAP, 4 << 8 | 0x6);
	    pci_set_word(dev->wmask + offset + PCI_PASID_CTRL, 0);
	}
```

在设备class的realize函数里调用如上函数加上pasid cap:
```
	#define GHMS_PCI_EXPRESS_CAP_OFFSET         0xe0
	#define GHMS_PCI_PASID_CAP_OFFSET           0x100
	pcie_endpoint_cap_init(pdev, GHMS_PCI_EXPRESS_CAP_OFFSET);
	pcie_pasid_init(pdev, GHMS_PCI_PASID_CAP_OFFSET);
```
pcie_add_capability里会用PCI_EXPRESS_CAP检测是不是PCIe设备，所以要先加上PCI_EXPRESS_CAP，
在0x40(0x34是capabilities pointer)到0xff这段地址选一个位置加上，如上选的是0xe0(避开
之前的MSI cap)。如上的函数内部会找到cap list尾，然后把新加的cap挂上去，所以我们这里
不需要做额外处理。
