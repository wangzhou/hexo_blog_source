---
title: PCIe学习笔记(三)
tags:
  - PCIe
description: '本文继续pci note 2, 介绍函数pci_scan_child_bus, 内核版本为3.18-rc1'
abbrlink: 9dc5e4ca
date: 2021-07-11 23:56:20
categories:
---
```
pci_scan_child_bus(b)
```
这个函数完成pci总线的枚举，完成整个pci树各个总线号的分配。但是并没有分配各个pci桥，
pci device的BAR和mem, I/O, prefetch mem的base/limit寄存器
```
unsigned int pci_scan_child_bus(struct pci_bus *bus)
{
	unsigned int devfn, pass, max = bus->busn_res.start;
	struct pci_dev *dev;

	/* 开始pci枚举, 一个pci bus上最多有256个设备，每个设备最多有8个function
	 * 所以这里最多可以扫描32个device。实际上每个function也是有配置空间的
	 * 所以function可以看作是个逻辑的设备
	 */
	for (devfn = 0; devfn < 0x100; devfn += 8)
		pci_scan_slot(bus, devfn);

	/* pci虚拟化相关的东西 */
	max += pci_iov_bus_range(bus);

	...
	if (!bus->is_added) {
		dev_dbg(&bus->dev, "fixups for bus\n");
		/* 对于arm64来说，是个空函数 */
		pcibios_fixup_bus(bus);
		bus->is_added = 1;
	}

	for (pass = 0; pass < 2; pass++)
		list_for_each_entry(dev, &bus->devices, bus_list) {
			if (pci_is_bridge(dev))
			    	/* 开始递归的做pci枚举，在下面的函数中会再次
				 * 调用pci_scan_child_bus。总线号的确定也在这个
				 * 函数中
				 */
				max = pci_scan_bridge(bus, dev, max, pass);
		}
	...
	return max;
}
```
```
pci_scan_slot(struct pci_bus *bus, int devfn)
    ...
    --> pci_scan_single_device(bus, devfn)
        ...
	    /* 扫描devfn设备, 探测设备：读vendor id, 读BAR的大小 。
	     * 软件行为：为device分配struct pci_dev; 根据探测设备得到的数据，填
	     * 写pci_dev里的一些值。若是按照笔记1中的硬件拓扑图现在扫描的device
	     * 应该是pcie root port
	     */
	--> pci_scan_device(bus, devfn)
	        /* 读devfn的vendor id, 会调用到pci_scan_root_bus的参数ops传入的read操作 */
	    --> pci_bus_read_dev_vendor_id()
	        /* 分配pci_dev结构 */
	    --> pci_alloc_dev(bus)
	    ...
	        /* 处理pci配置头中的class code, Header Type, Interrupt Pin
		 * Interrupt Line等内容，并设定新分配的pci_dev中的对应项。这里
		 * 也会探测device的BAR大小，并设定pci_dev->resource[]。我们关注
		 * 的重点也是pci_bus,pci_dev中相应的元素怎么变化。
		 */
	    --> pci_setup_device(dev)
	        ...
	--> pci_device_add(dev, bus)	

int pci_setup_device(struct pci_dev *dev)
    ...
        /* 的到device配置空间中head type的值，可以是普通pci device, pci桥等
	 * 这个值在下面的switch-case中决定程序是走哪个分支。如果当前的设备是
	 * pcie root port，那么走PCI_HEADER_TYPE_BRIDGE分支。设备配置空间
	 * head type的值一般是pci host controller驱动中配置过的, 可以参考
	 * /drivers/pci/host/pcie-designware.c
	 */
    --> pci_read_config_byte(dev, PCI_HEADER_TYPE, &hdr_type)
        /* device的parent是上游总线之上的那个pci桥 */
    --> dev->dev.parent = dev->bus->bridge;
        /* 读出class revision的值 */
    --> pci_read_config_dword(dev, PCI_CLASS_REVISION, &class)
	dev->revision = class & 0xff;
	dev->class = class >> 8;		    /* upper 3 bytes */

	dev->current_state = PCI_UNKNOWN;

	class = dev->class >> 8;

	switch (dev->hdr_type) {		    /* header type */
	case PCI_HEADER_TYPE_NORMAL:		    /* standard header */
		if (class == PCI_CLASS_BRIDGE_PCI)
			goto bad;
		pci_read_irq(dev);
		/*
		 * 得到BAR的大小，并存在pci_dev->resource[]中，主要是对pci
		 * device起作用。当输入参数dev是pci桥时，得到的值似乎对后面影响
		 * 不大，这点需要确认下？
		 */
		pci_read_bases(dev, 6, PCI_ROM_ADDRESS);
		pci_read_config_word(dev, PCI_SUBSYSTEM_VENDOR_ID, &dev->subsystem_vendor);
		pci_read_config_word(dev, PCI_SUBSYSTEM_ID, &dev->subsystem_device);

			break;

	case PCI_HEADER_TYPE_BRIDGE:		    /* bridge header */
		if (class != PCI_CLASS_BRIDGE_PCI)
			goto bad;
		pci_read_irq(dev);
		dev->transparent = ((dev->class & 0xff) == 1);
		pci_read_bases(dev, 2, PCI_ROM_ADDRESS1);
		set_pcie_hotplug_bridge(dev);
		pos = pci_find_capability(dev, PCI_CAP_ID_SSVID);
		if (pos) {
			pci_read_config_word(dev, pos + PCI_SSVID_VENDOR_ID, &dev->subsystem_vendor);
			pci_read_config_word(dev, pos + PCI_SSVID_DEVICE_ID, &dev->subsystem_device);
		}
		break;
		...
	}
```
```
/*
 * 直接读硬件配置空间中的interrupt line and pin，然后写到pci_dev->line、pci_dev->irq中
 */
pci_read_irq(dev)

/* 依次读取6个bar的内容，相关内容写入pci_dev下的struct resource resource[] */
pci_read_bases(dev, 6, PCI_ROM_ADDRESS);
    -->__pci_read_bases
        -->mask = type ? PCI_ROM_ADDRESS_MASK : ~0;
        /* 最初探测bar的大小，向bar空间写入全1 */
        -->pci_write_config_dword(dev, pos, l | mask);
        /* 读出提示信息，写入sz缓存 */
	-->pci_read_config_dword(dev, pos, &sz);
        /* 如果全1, 设备工作不正常，或是没有这个bar */
        -->if (sz == 0xffffffff)
		sz = 0;

	-->if (type == pci_bar_unknown) {
                /* 根据上面sz的值，得到是io/mem bar, 32/64bits, prefectch/
                 * non-prefectch。这些信息都会写入res->flags中的对应bit
                 */ 
		/* 如果l = 0, 解析的结果，kernel会认为这是一个32bit,
		 * non-prefetchable, mem bar
		 */
		-->res->flags = decode_bar(dev, l);
		res->flags |= IORESOURCE_SIZEALIGN;
		if (res->flags & IORESOURCE_IO) {
                    ...
		} else {
                        /* 下面先把低32bits写到64bits变量的低32bits */
			/* 如果l = 0, l64 = 0 */
			l64 = l & PCI_BASE_ADDRESS_MEM_MASK;
			/* sz是写入全1后从bar中得到的值，如果是0xf000000f
			 * sz64 = 0xf0000000
			 */
			sz64 = sz & PCI_BASE_ADDRESS_MEM_MASK;
                        /* ul的变量是多少bits? */
			/* 应该是0xfffffff0 */
			mask64 = (u32)PCI_BASE_ADDRESS_MEM_MASK;
		}
	} else {
            ...
	}

	if (res->flags & IORESOURCE_MEM_64) {
		pci_read_config_dword(dev, pos + 4, &l);
		pci_write_config_dword(dev, pos + 4, ~0);
		pci_read_config_dword(dev, pos + 4, &sz);
		pci_write_config_dword(dev, pos + 4, l);

		l64 |= ((u64)l << 32);
		sz64 |= ((u64)sz << 32);
		mask64 |= ((u64)~0 << 32);
	}
        ...
	sz64 = pci_size(l64, sz64, mask64);
		/* size = 0xf000_0000 */
		--> size = sz64 & mask64;
		/* size = 0x0fff_ffff, and return this */
		--> size = (size & ~(size-1)) - 1;

	/* struct pci_bus_region, local struct. indicate a bar's resource? */
	region.start = l64; /* l64 = 0 */
	region.end = l64 + sz64; /* 0x0fff_ffff */

	pcibios_bus_to_resource(dev->bus, res, &region);
		/* 首先找到host中的对应资源 */
		--> resource_list_for_each_entry(window, &bridge->windows) {
			--> if (resource_type(res) != resource_type(window->res))
			/* 找到对应到pci域中的起始地址 */
			--> bus_region.start = window->res->start - window->offset;
			--> bus_region.end = window->res->end - window->offset;

	pcibios_resource_to_bus(dev->bus, &inverted_region, res);
```

现在回到pci_scan_child_bus中的pci_scan_bridge()函数。
```
/*
 * 如果一个pci_dev是pci桥的话，以它的上游总线bus和这个设备dev本身为参数，扫描这个
 * 桥。这个函数中实现pci树的递归扫描，在这个函数中为整个pci树上的各个子总线分配
 * 总线号。这里pci_scan_bridge会调用两次，第一次处理BIOS中已经配置的东西，不清楚
 * intel在BIOS中做了怎样的处理。
 */
pci_scan_bridge(bus, dev, max, pass);
    ...
    /* 这里上来就读桥设备的主bus号，是因为有些体系架构下可能在BIOS中已经对这些
     * 做过配置。在ARM64中暂时没有看到有这样做的情况。
     */
    --> pci_read_config_dword(dev, PCI_PRIMARY_BUS, &buses);
    ...
    /* 配置了在kernel中分配bus号，应该不会进入到if中 */
    --> if ((secondary || subordinate) && !pcibios_assign_all_busses() &&
	    !is_cardbus && !broken) {
	...
	} else {
	        /* 第一次掉用pci_scan_bridge就此返回了 */
		if (!pass) {
		        ...
			goto out;
		}

		/* Clear errors */
		pci_write_config_word(dev, PCI_STATUS, 0xffff);

		/* 确定是否已用max+1这个bus号 */
		child = pci_find_bus(pci_domain_nr(bus), max+1);
		if (!child) {
		        /* 分配新的struct pci_bus */
			child = pci_add_new_bus(bus, dev, max+1);
			if (!child)
				goto out;
			pci_bus_insert_busn_res(child, max+1, 0xff);
		}
		max++;
		/* 合成新的bus号，准备写入pci桥的配置空间 */
		buses = (buses & 0xff000000)
		      | ((unsigned int)(child->primary)     <<  0)
		      | ((unsigned int)(child->busn_res.start)   <<  8)
		      | ((unsigned int)(child->busn_res.end) << 16);
		...
		/* 写入该pci桥primary bus, secondary bus, subordinate bus */
		pci_write_config_dword(dev, PCI_PRIMARY_BUS, buses);

		if (!is_cardbus) {
			child->bridge_ctl = bctl;
			/* 递归调用pci_scan_child_bus, 扫描这个子总线下的设备*/
			max = pci_scan_child_bus(child);
		} else {
		        /* 不关心cardbus */
			...
		}
		/*
		 * Set the subordinate bus number to its real value.
		 * 每次递归结束把实际的subordinate bus写入pci桥的配置空间
		 * subordinate bus表示该pci桥下最大的总线号
		 */
		pci_bus_update_busn_res_end(child, max);
		pci_write_config_byte(dev, PCI_SUBORDINATE_BUS, max);
	}
	...
out:
	pci_write_config_word(dev, PCI_BRIDGE_CONTROL, bctl);

	return max;
```
