---
title: PCIe enumeration for SR-IOV PCIe device
tags:
  - Linux内核
  - PCIe
description: >-
  There are a lot things should be done in software to support PCIe SR-IOV.
  Using qemu(for example) to use a VF, at least, we should:

  1. host PCI subsystem finds VF and allocate resources for VF. 2. vfio driver
  exports VF's configure space, MEM/IO range to user space. 3. vfio driver
  allocate DMA for VF by IOMMU. 4. vfio driver builds the interrupt flow for VF.
  5. qemu vfio driver simulates PCIe device for guest kernel.

  This document just tries to share an outline about point 1.
abbrlink: 43f2d057
date: 2021-07-11 23:35:34
categories:
---

Parse VF/PF BARs
-------------------
```
/* In PCI enumeration, once we find SR-IOV capability, sriov_init will be called */
pci_device_add
    --> pci_init_capabilities
        --> pci_iov_init
                /* parse sr-iov capability in pf, get bar sizef of vf */
            --> sriov_init(struct pci_dev *dev, int pos)
                ...
                    /* get bar_i size of vf */
                --> __pci_read_base(dev, pci_bar_unknown, res, pos + PCI_SRIOV_BAR + i * 4);
                ...
                    /* get bar_i size for all vf */
                --> res->end = res->start + resource_size(res) * total - 1;
```
so at this step, we will see BAR0\~BAR5, ROM_BAR, VF_BAR0\~VF_BAR5..., the sizes
of them will be known. VF_BAR0 includes BAR0 in VF1,2,.... the resouces assignment
below will allocate resources for VF BARs ahead. Here all BAR0 in VF1,2... has
been seen as one block. you can see pci_enable_sriov below will directly assign
related BAR of certain VF to related pci_dev, as we already caculate and allocate
the resouces here.
```
/*
 * so if we get the size of bar of vf here, assignment will happen in standard
 * assignment in PCI subsystem.
 */

...
```
```
/*
 * this function will be called in specific PCIe device driver. e.g.
 * drivers/net/ethernet/intel/ixgbe/ixgbe_sriov.c, when user in userspace triggers
 * vf by sysfs or loads modules with vf parameters.
 */
pci_enable_sriov
    --> sriov_enable(dev, nr_virtfn)
        --> pci_iov_add_virtfn
            ...
                /*
                 * As we already got all VFs' BARs been assigned above, so here
                 * just fill the BAR resource to each VF's pci_dev.
                 */
            --> virtfn->resource[i].start = res->start + size * id
```
Note:                            
---------                        
1) BARs in PF/VF
```
     +--------------+    +-------->+----------+
     | PF CFG       |    |         | VF1 BAR0 |
     +--------------+    |         | VF2 BAR0 |
     | PCI cfg head:|    |         | ...      |
     | ...          |    |         | VF5 BAR0 |
     | BAR0 ~ BAR5  |    |         +----------+
     | ...          |    |  +----->+----------+
     +--------------+    |  |      | VF1 BAR1 |
     | SR-IOV cap:  |    |  |      | VF2 BAR1 |
     | ...          |    |  |      | ...      |
     | VF BAR0      +----+  |      | VF5 BAR1 |
     | VF BAR1      +-------+      +----------+
     | ...          |              ...              
     | VF BAR5      +------------->+----------+
     +--------------+              | VF1 BAR5 |
                                   | VF2 BAR5 |
                                   | ...      |
                                   | VF5 BAR5 |
                                   +----------+
```                         
