---
title: linux kernel vfio mdev arch
tags:
  - Linux内核
  - vfio
description: This document shares the basic logic of linux vfio mdev driver
abbrlink: e9c69e8f
date: 2021-07-05 22:26:32
categories:
---

Linux kernel adds a vfio mdev system in /driver/vfio/mdev. This sub-system can
create virtual devices and export their DMA outside to user space.

This subsystem creates one new bus called: struct bus_type mdev_bus_type.
And it uses a gobal list to store parent device, here we call orginal device as
parent device and virtual device created from orginal device as mdev device,
e.g. a NIC is a parent device and one of NIC's queue can be a mdev device to
fullfil a seperated package sending/receiving work.

vfio mdev exports a API:
```
int mdev_register_device(struct device *dev, const struct mdev_parent_ops *ops)
```
to let a device register to itself.

And vfio mdev exports a set of sysfs files to help user to create a mdev device,
here we call a mdev device as child device.

A child device will be added into mdev_bus_type, and a driver which belongs to
the mdev_bus_type in vfio_mdev.c will bind with related child device.

The probe of this vfio_mdev is very simple, it just call:
```
vfio_add_group_dev(dev, &vfio_mdev_dev_ops, mdev)
```
It just create a vfio_group and related vfio_device. So the vfio mdev will reuse
all the concept of vfio system.

The probe of mdev_bus_type will help to create a new iommu_group for child
device. Above vfio_add_group_dev creates iommu_group's vfio_group and add this
vfio_group to vfio's group list.

If user want to use above vfio_group together with a vfio container, it should
attach this vfio_group to related vfio container. When doing this attach
operation. vfio system will replay all mappings in vfio container to this
vfio group, which means this child's DMA can see the address space managed by
vfio container.
