---
title: PCIe学习笔记(四)
tags:
  - PCIe
description: '本文继续pci note 3, 介绍PCI枚举中资源分配相关的函数, 内核版本为3.18-rc1'
abbrlink: 67860bc5
date: 2021-07-11 23:49:52
categories:
---
```
pci_assign_unassigned_bus_resources(b)
void pci_assign_unassigned_bus_resources(struct pci_bus *bus)
{
	struct pci_dev *dev;
	LIST_HEAD(add_list); /* list of resources that
					want additional resources */

	down_read(&pci_bus_sem);
	list_for_each_entry(dev, &bus->devices, bus_list)
		if (pci_is_bridge(dev) && pci_has_subordinate(dev))
		    		/* 这里传入的参数是bus 1对应的pci_bus */
				__pci_bus_size_bridges(dev->subordinate,
							 &add_list);
	up_read(&pci_bus_sem);
	/* 配置各个桥和设备的BAR，配置桥的MEM，I/O，prefetch MEM base/limit */
	__pci_bus_assign_resources(bus, &add_list, NULL);
	BUG_ON(!list_empty(&add_list));
}

__pci_bus_size_bridges(struct pci_bus *bus, struct list_head *realloc_head)
    ...
       /* 在当前的连接状态下，list中的代码不会执行。下面的代码层层递归，直到
        * 最底层设备的上的pci_bus，这时最底层设备是没有下一级的pci_bus的。
        * 所以，继续执行后面的代码。
        */
    -->list_for_each_entry(dev, &bus->devices, bus_list) {
	    ...
	case PCI_CLASS_BRIDGE_PCI:
	default:
		__pci_bus_size_bridges(b, realloc_head);
		break;
       }
    ...
       /* 当前pci_bus的桥对应的pci_dev, 这里应该是host */
    -->switch (bus->self->class >> 8) {
        ...
        case PCI_CLASS_BRIDGE_PCI:
            /* 会执行这里 */
    	    pci_bridge_check_ranges(bus);
	    ...
	default:
		/* 这个函数改变了pci_bus->resource[]中的值。start对齐4K，size是该bus下
		 * 所有I/O空间的总和。但是似乎realloc_head
		 * list似乎没有node添加上去
		 */
		pbus_size_io(bus, realloc_head ? 0 : additional_io_size,
			     additional_io_size, realloc_head);

		/*
		 * If there's a 64-bit prefetchable MMIO window, compute
		 * the size required to put all 64-bit prefetchable
		 * resources in it.
		 */
		b_res = &bus->self->resource[PCI_BRIDGE_RESOURCES];
		mask = IORESOURCE_MEM;
		prefmask = IORESOURCE_MEM | IORESOURCE_PREFETCH;
		if (b_res[2].flags & IORESOURCE_MEM_64) {
			prefmask |= IORESOURCE_MEM_64;
			ret = pbus_size_mem(bus, prefmask, prefmask,
				  prefmask, prefmask,
				  realloc_head ? 0 : additional_mem_size,
				  additional_mem_size, realloc_head);

			/*
			 * If successful, all non-prefetchable resources
			 * and any 32-bit prefetchable resources will go in
			 * the non-prefetchable window.
			 */
			if (ret == 0) {
				mask = prefmask;
				type2 = prefmask & ~IORESOURCE_MEM_64;
				type3 = prefmask & ~IORESOURCE_PREFETCH;
			}
		}

		/*
		 * If there is no 64-bit prefetchable window, compute the
		 * size required to put all prefetchable resources in the
		 * 32-bit prefetchable window (if there is one).
		 */
		if (!type2) {
			prefmask &= ~IORESOURCE_MEM_64;
			ret = pbus_size_mem(bus, prefmask, prefmask,
					 prefmask, prefmask,
					 realloc_head ? 0 : additional_mem_size,
					 additional_mem_size, realloc_head);

			/*
			 * If successful, only non-prefetchable resources
			 * will go in the non-prefetchable window.
			 */
			if (ret == 0)
				mask = prefmask;
			else
				additional_mem_size += additional_mem_size;

			type2 = type3 = IORESOURCE_MEM;
		}

		/*
		 * Compute the size required to put everything else in the
		 * non-prefetchable window.  This includes:
		 *
		 *   - all non-prefetchable resources
		 *   - 32-bit prefetchable resources if there's a 64-bit
		 *     prefetchable window or no prefetchable window at all
		 *   - 64-bit prefetchable resources if there's no
		 *     prefetchable window at all
		 *
		 * Note that the strategy in __pci_assign_resource() must
		 * match that used here.  Specifically, we cannot put a
		 * 32-bit prefetchable resource in a 64-bit prefetchable
		 * window.
		 */
		pbus_size_mem(bus, mask, IORESOURCE_MEM, type2, type3,
				realloc_head ? 0 : additional_mem_size,
				additional_mem_size, realloc_head);
		break;
	}
}

__pci_bus_assign_resources(bus, &add_list, NULL)
        /* bus:0, 会对该bus上的所有device分别调用__dev_sort_resource
	 * 然后统一调用一个__assign_resources_sorted()。之后程序进入
	 * 下面的list中，又会嵌套进入__pci_bus_assign_resources, 这时
	 * bus:1。重复上面的过程。在bus:1是__pci_bus_assign_resources
	 * 在处理处理完pbus_assign_resources_sorted后不回往下执行,返回
	 * 上层。这时bus:0, 进入pci_setup_bridge执行。
	 * 其中，会在pbus_assign_resources_sorted中配置BAR，在
	 * __pci_setup_bridge中配mem, I/O, prefetch mem的base/limit
	 */
    --> pbus_assign_resources_sorted(bus, realloc_head, fail_head);
    --> list_for_each_entry(dev, &bus->devices, bus_list)
        --> __pci_bus_assign_resources(b, realloc_head, fail_head);
	--> switch (dev->class >> 8)
		case PCI_CLASS_BRIDGE_PCI:
		    --> pci_setup_bridge(b);

static void pbus_assign_resources_sorted()
    --> LIST_HEAD(head);
    --> list_for_each_entry(dev, &bus->devices, bus_list)
		__dev_sort_resources(dev, &head);
                    /* 把pci_dev中的resource[]从大到小排序, 存入链表head中 */
		    --> pdev_sort_resources(dev, head);
    --> __assign_resources_sorted(&head, realloc_head, fail_head);
            /* 因为realloc_head为空链表，所以直接到requested_and_reassign */
	--> if (!realloc_head || list_empty(realloc_head))
		goto requested_and_reassign;
	...
        requested_and_reassign:
	--> assign_requested_resources_sorted(head, fail_head);
	        /* dev_res(pci_dev_resource)存储一个device中一个配置空间
		 * 的资源(一个设备可有多个mem或I/O配置空间)
		 */
	    --> list_for_each_entry(dev_res, head, list)
		--> resource_size(res) &&
		pci_assign_resource(dev_res->dev, idx)
	           --> _pci_assign_resource(dev, resno, size, align);
		   --> pci_update_resource(dev, resno);

        --> reassign_resources_sorted(realloc_head, head);
```
