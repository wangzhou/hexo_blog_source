---
title: Linux vfio driver arch analysis
tags:
  - 虚拟化
  - vfio
description: 本文分析Linux内核里vfio驱动的架构
abbrlink: a1d7a5a7
date: 2021-07-11 23:40:18
categories:
---

The whole vfio subsystem should support 3 sub-features:

1. cfg/mem/io support: user space can access cfg/mem/io of vf.
2. dma support: data in vf can be translated to user space dma memory range.
3. interrupt from vf can be routed to VM OS.

From view of code, we can see whole vfio driver as:

1. init base vfio arch in drivers/vfio/vfio.c
2. init pci/platform vfio device driver in drivers/vfio/pci/vfio_pci.c, drivers/vfio/platform/vfio_platform.c
3. init vfio iommu driver and register to vfio system in drivers/vfio/vfio_iommu*

This vfio system will create /dev/vfio/vfio as a vfio container, which indicates
an address space share by multiple devices. It will also create /dev/vfio/<group_number>
as a vfio group, which indicates a group shared by multiple devices using a iommu
or smmu unit. when we open a /dev/vfio/<group_number>, we will get a fd, which
indicates a device handled by vfio system. Device can be controlled by this fd.

vfio system does not create new bus, however, we should unbind original device
driver, and bind device with vfio device driver. So for a PCI device, we need
vfio pci driver to handle this device. This vfio pci driver becomes the agent of
this device and export all its resource to user space.


The interfaces for userspace:


vfio init in vfio.c
----------------------

vfio registers a misc device in /dev/vfio/vfio.

initialize items in vfio: 
```
register vfio_dev(miscdevice) in misc sub-system, file: /dev/vfio/vfio
              |
              |--> fops(vfio_fops(file_operations))
                             |
                             |--> open: create vfio_container and
                                        set this as file's private data
                                  unlocked_ioctl:
                                          /* bind vfio_container and
                                           * vfio_iommu_driver which had been
                                           * registered in vfio.iommu_driver_list
                                           * in specific iommu file, like:
                                           * vfio_iommu_type1.c
                                           */
                                      --> vfio_ioctl_set_iommu
                                   read/write/mmap: will call functions in vfio_iommu_driver
```

vfio.c creat a vfio class, this will work together with device_create in
vfio_create_group. vfio creates a vfio group indeed is creating a device in this
vfio class, vfio group file will be /dev/vfio/<group_number>.

vfio_create_group is called in vfio_pci_probe and vfio_platform_probe. In the probe,
we get the devices which we want to handle by vfio system, then find which iommu group
these devices belong to, then create the related vfio_group to help to store related
iommu group. Here just use device_creat to create a file under /dev/vfio/ to refer to
the vfio_group. At last, we creat vfio_pci/vfio_platform_device for the devices
which we want vfio system to take care of. For details, please refer to part2.
```
vfio.class = class_create(THIS_MODULE, "vfio")
```
when we operate /dev/vfio/<group_number>, indeed we will call
functions in vfio_group_fops.
```
register chr device: vfio.group_cdev(struct cdev)
                              |
                              |--> file_operations(vfio_group_fops)
                                               |
                                               |--> open
                                                    unlocked_ioctl
                                                            /* register ops of
                                                             * vfio_device
                                                             */
                                                        --> vfio_group_get_device_fd
```
so what happen if we call above callback:
```
open: find vfio_group --> share vfio_group to private_data of related struct file.
      above vfio_group was created and added in vfio_pci_probe.
unlocked_ioctl: 
    VFIO_GROUP_GET_DEVICE_FD:
	-->vfio_group_get_device_fd
    VFIO_GROUP_SET_CONTAINER: 
        --> vfio_group_set_container
```
An ioctl of vfio_group can get a fd for the device.

We already get the iommu_group of a device, why do we use vfio_group_set_container
to add this vfio_group to a vfio container?

The concept of vfio container is to build an address space shared by multiple
devices.


      vfio container


              ------+--------------+--------------+-------
                    |              |              |
                  +-+--+         +-+--+         +-+--+
                  |smmu|         |smmu|         |smmu|
                  +-+--+         +-+--+         +-+--+
                    |              |              |
                  +-+--+         +-+--+         +-+--+
                  |dev |         |dev |         |dev |
                  +----+         +----+         +----+

When vfio_group is added to vfio container, mappings in this vfio_group will be
added to other smmus physically. So all smmus above have same mapping if vfio_groups
have been added into same vfio container. All mappings are maintained in vfio
container. 

how to add vfio_group to vfio_container:
```
vfio_ioctl_set_iommu
    --> __vfio_container_attach_groups(container, driver, data)
            --> driver->ops-attach_group(vfio_iommu, group->iommu_group)
```

probe of vfio_pci.c/vfio_platform.c
---------------------------------------------------------------

All working in vfio system will help build below vfio struct:
```
global: vfio
            --> group_list(vfio_group)
                               |
                               |--> iommu_group
                                    device_list(vfio_device)
                                                     |-->group(vfio_group)
                                                         ops(vfio_device_ops)
                                                                   |--> open
                                                                        ...
            --> device_list(vfio_device)
                               |
                               |--> ops(vfio_device_ops)
                                    group(vfio_group)
         
            --> iommu_drivers_list(vfio_iommu_driver)
                                          |
                                          |--> ops(vfio_iommu_driver_ops)
```
Here we analyze the flows in vfio_pci.

in vfio_pci_init, use pci_register_driver(&vfio_pci_driver) to probe the PCIe
devices in the whole PCIe domain, which devices we had already build up in
standard PCIe enumeration process.
```
vfio_pci_probe
        /* get iommu_group from device, iommu_group had been added to device's
         * iommu_group during the init of iommu(smmu)
         */
    --> vfio_iommu_group_get
    --> allocate memory for vfio_pci_device, which in vfio system refers to above device
        /* this will be called in pci or platform device file to create vfio_group,
         * vfio_device. ops for specific kind of vfio_device is different.
         */
    --> vfio_add_group_dev
            /* create vfio_group, and add it to vfio.group_idr and vfio.group_list.
             * and add iommu_group to this vfio_group.
             */
        --> vfio_create_group
                /* create vfio_group file as /dev/vfio/<group number> */
            --> device_create
            /* create vfio_device and add it to vfio_group.device_list, ops will be
             * called at:
             *
             * vfio_group_fops_unl_ioctl can get a fd which refers to related
             * devcie's fd. the operations to this fd will route to the operations
             * in vfio_device_fops(struct file_operations) which will call the
             * callbacks in vfio_device.
             *
             * vfio_pci_device is vfio_device's private data.
             */
        --> vfio_group_create_device
```

vfio_register_iommu_driver in specific iommu file
----------------------------------------------------

Physically we can use different iommu implementation, e.g. SMMU in ARM, IOMMU for
Intel. This vfio iommu driver is used to control this.

register vfio_iommu_driver to vfio:
```
vfio
    --> iommu_drivers_list(vfio_iommu_driver)
                                   |
                                   |--> ops(vfio_iommu_driver_ops)
                                                   |--> open: create vfio_iommu.

```
This ops is called by vfio_container->vfio_smmu_driver->ops, we bind
vfio_container and vfio_smmu_driver together in unlocked_ioctl(using VFIO_SET_IOMMU)
of /dev/vfio/vfio. Here we can register specific iommu driver to vfio, now there are
vfio iommu driver from X86(vfio_iommu_type1.c) and POWER(vfio_iommu_spapr_tce.c).

how to bind vfio_container and vfio_smmu_driver:
for a /dev/vfio/vfio container fd, its ioctl VFIO_SET_IOMMU will set specific
IOMMU for the container:
```
vfio_fops_unl_ioctl
    --> VFIO_SET_IOMMU
        --> vfio_ioctl_set_iommu(container, arg)
```

how to call the ops in vfio_smmu_driver:
vfio_container->ops will call the ops in vfio_smmu_driver.


how to access cfg/mem/io of VFs
----------------------------------
```
/* ioctl of vfio_group to get a fd of device */
vfio_group_get_device_fd

/* open in vfio_pci_device */
--> device->ops->open
    --> anon_inode_getfile("[vfio-device]", &vfio_device_fops, device, O_RDWR);

/* to do: So it seems we can use both ways to access vf cfg/mem/io range */
read/write: use fd's read/write to access vf cfg/mem/io.
mmap: map cfg/mem/io range in pci_dev to user space.
```

Reference:
----------
- https://www.ibm.com/developerworks/community/blogs/5144904d-5d75-45ed-9d2b-cf1754ee936a/entry/20160605?lang=en
- http://blog.csdn.net/qq123386926/article/details/47757089
- https://zhuanlan.zhihu.com/p/35489035
