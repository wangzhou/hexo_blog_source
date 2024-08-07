---
title: PCIe学习笔记(二)
tags:
  - PCIe
description: '本文继续pci note 1, 介绍pci_create_root_bus函数, 内核版本为3.18-rc1'
abbrlink: e336c4e1
date: 2021-07-11 23:48:15
categories:
---
 调用关系：
```
 pci_scan_root_bus
     --> pci_create_root_bus
```
```
/*
 * device说明见下文，bus是根总线号，ops是配置空间读写函数的接口，需要驱动作者
 * 传入回调函数, 会在pci_scan_child_bus->pci_scan_slot->pci_scan_single_device->
 * pci_scan_device->pci_bus_read_dev_vendor_id调用到该ops中的read函数。sysdata
 * 传入私有数据。resources链表的元素是struct pci_host_bridge_window, 是dts上
 * 读上来的总线号，mem空间，I/O空间的信息, 一般一个pci_host_bridge_window对应
 * 一个信息
 */
struct pci_bus *pci_create_root_bus(struct device *parent, int bus,
		struct pci_ops *ops, void *sysdata, struct list_head *resources)
{
	...
	/*
	 * 分配 struct pci_host_bridge, 初始化其中的windows链表
	 * windows链表上的存的结构是：struct pci_host_bridge_window
	 * struct pci_host_bridge_window {
	 * 	struct list_head list;
	 * 	struct resource *res;	/* host bridge aperture (CPU address) */
	 * 	resource_size_t offset;	/* bus address + offset = CPU address */
	 * };
	 */
	bridge = pci_alloc_host_bridge();

	/*
	 * 输入参数parent来自pci host驱动中pci host核心结构的struct device *dev,
	 * dev来自 platform_device 中的dev。可以以drivers/pci/host下的pci-mvebu.c
	 * 作为例子, 其中所谓的pci host核心结构是：struct mvebu_pcie
	 */
	bridge->dev.parent = parent;

	/* 分配 struct pci_bus */
	b = pci_alloc_bus(NULL);

	b->sysdata = sysdata;
	b->ops = ops;
	b->number = b->busn_res.start = bus;
	/* 在pcie dts节点中找见domain字段, 加入pci_bus的domain_nr */
	pci_bus_assign_domain_nr(b, parent);
	/*
	 * 在pci_root_buses全局链表中找相应domain下的bus, 首次调用的时候返回NULL
	 * 上面分配的pci_root_buses是在当前函数的最后才加入pci_root_buses中的，现在该
	 * 全局链表为空
	 */
	b2 = pci_find_bus(pci_domain_nr(b), bus);
	/*
	 * 上面两行处理有关pci domain的信息，kernel pci子系统怎么处理pci domain
	 * 的呢？ 首先数据结构是全局的链表：pci_root_buses, 局部链表：pci_domain_busn_res_list
	 * pci_root_buses中存放每个pci domain的根总线，根总线在pci_create_root_bus
	 * 函数的结尾被添加到pci_root_buses链表中。pci_domain_busn_res_list存放
	 * 各个domain的信息, 包括domain号、domain包含的bus号范围, 该链表上存放
	 * 存放的结构是：struct pci_domain_busn_res, 在函数get_pci_domain_busn_res
	 * 中查找相应domain号的pci_domain_busn_res, 如果没有就分配一个新的
	 * pci_domain_busn_res, 然后加到pci_domain_busn_res_list上
	 */

	bridge->bus = b;
	dev_set_name(&bridge->dev, "pci%04x:%02x", pci_domain_nr(b), bus);
	error = pcibios_root_bridge_prepare(bridge);

	error = device_register(&bridge->dev);

	b->bridge = get_device(&bridge->dev);
	device_enable_async_suspend(b->bridge);
	pci_set_bus_of_node(b);

	if (!parent)
		set_dev_node(b->bridge, pcibus_to_node(b));

	b->dev.class = &pcibus_class;
	/* b->bridge 为对应pci_host_bridge中struct device dev的指针 */
	b->dev.parent = b->bridge;
	dev_set_name(&b->dev, "%04x:%02x", pci_domain_nr(b), bus);
	error = device_register(&b->dev);

	pcibios_add_bus(b);

	/* Create legacy_io and legacy_mem files for this bus */
	pci_create_legacy_files(b);

	...
	/*
	 * 取出pci_create_root_bus函数传入的链表resources中的pci_host_bridge_window,
	 * 把每个pci_host_bridge_window加入pci_host_bridge中的window链表中
	 */
	list_for_each_entry_safe(window, n, resources, list) {
		list_move_tail(&window->list, &bridge->windows);
		res = window->res;
		offset = window->offset;
		if (res->flags & IORESOURCE_BUS)
		    	/*
			 * 一般的，resources链表上有bus number, MEM space, I/O
			 * space的节点，如果是bus number节点则调用以下函数。该
			 * 函数会找到当前pci_bus的父结构，生成pci_bus中的busn_res
			 * 并和父结构中的struct resource busn_res建立联系。
			 * 如果父子在bus号上存在冲突，则返回冲突的bus号[1]
			 */
			pci_bus_insert_busn_res(b, bus, res->end);
		else
			/*
			 * 向pci_bus中的链表resources中加入struct pci_bus_resource
			 * 记录mem, io的资源
			 */
			pci_bus_add_resource(b, res, 0);
		if (offset) {
			if (resource_type(res) == IORESOURCE_IO)
				fmt = " (bus address [%#06llx-%#06llx])";
			else
				fmt = " (bus address [%#010llx-%#010llx])";
			snprintf(bus_addr, sizeof(bus_addr), fmt,
				 (unsigned long long) (res->start - offset),
				 (unsigned long long) (res->end - offset));
		} else
			bus_addr[0] = '\0';
		dev_info(&b->dev, "root bus resource %pR%s\n", res, bus_addr);
	}

	down_write(&pci_bus_sem);
	/* 把创建的pci_bus加入全局链表pci_root_buses中 */
	list_add_tail(&b->node, &pci_root_buses);
	up_write(&pci_bus_sem);

	return b;
	...
}
```
[1] 关于linux中resource结构对资源的管理可以参看: http://www.linuxidc.com/Linux/2011-09/43708.htm
