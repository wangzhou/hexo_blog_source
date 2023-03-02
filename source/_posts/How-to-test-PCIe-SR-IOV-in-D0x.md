---
title: How to test PCIe SR-IOV in D0x
tags:
  - 软件测试
  - PCIe
description: >-
  This doc helps to introduce how to test PCIe SR-IOV in HiSilicon D05 board. We
  can use same way to test SR-IOV in Hi1620.
abbrlink: 9fd37943
date: 2021-07-11 23:34:19
categories:
---

preparation
--------------

kernel: https://github.com/hisilicon/kernel-dev.git branch: private-topic-sriov-v3-4.10

UEFI: openlab1.0 101 server /home/wangzhou/repo/plinth_uefi/uefi

qemu: https://github.com/eauger/qemu.git branch: v2.7.0-passthrough-rfc-v5

hardware topology:
```
   +---------------+       +----------------------+
   |  D05 I        |       |         D05 II       |
   |               |       |                      |
   |   +-----------+       +------------------+   |
   |   |1P NA PCIe2|<----->|any 10G networking|   |
   |   +-----------+       +------------------+   |
   +---------------+       +----------------------+
```
compile kernel and UEFI
--------------------------

configure kernel: Add SMMU_V3=y, 82599 PF driver = m, 82599 VF driver = m,
                  VFIO PCI driver = m (p.s. ACPI boot)
compile kernel image and ko
compile UEFI using: ./uefi-tools/uefi-build.sh -c ./LinaroPkg/platforms.config d05


basic test
-------------

    1. boot up host OS (firstly update UEFI above)[1]

    2. copy modules to host OS:
       ixgbe.ko mdio.ko vfio_iommu_type1.ko vfio.ko vfio-pci.ko
       vfio_virqfd.ko irqbypass.ko

    3. prepare host environment:

       mkdir /lib/modules/`uname -r`
       touch /lib/modules/`uname -r`/modules.order
       touch /lib/modules/`uname -r`/modules.builtin
       depmod ixgbe.ko  mdio.ko vfio_iommu_type1.ko vfio.ko vfio-pci.ko
              vfio_virqfd.ko irqbypass.ko
 
       /*
        * here we insert ixgbe as it will use function in ixgbe driver to triggre
        * VF. If we do no insert ixgbe here, we will not get VF's sysfs interface,
        * and lspci will not show information of VF.
        *
        * here we need not insert ixgbevf(VF driver), as we will try to bind vfio-pci
        * to VF device. However, if we want to use VF in host, we should insert
        * ixgbevf here. After inserting ixgbevf, if we want to use vfio-pci driver,
        * we should firstly unbind ixgbevf driver for VF device using:
        * echo 0002:81:10.0 > /sys/bus/pci/drivers/ixgbevf/unbind, then we can
        * use  echo vfio-pci > /sys/bus/pci/devices/0002:81:10.0/driver_override
        *      echo 0002:81:10.0 > /sys/bus/pci/drivers_probe
        * to bind vfio-pci driver with VF device.
        */
       modprobe ixgbe
 
       /* trigger one VF, 0002:81:00.0 is the PF in which you want trigger a VF */
       echo 1 > /sys/devices/pci0002:80/0002:80:00.0/0002:81:00.0/sriov_numvfs
 
       modprobe -v vfio-pci disable_idle_d3=1
       modprobe -r vfio_iommu_type1
       modprobe -v vfio_iommu_type1 allow_unsafe_interrupts=1
 
       /* set related PF up */
       ifconfig eth26 up
 
       /* 0002:81:10.0 is BDF of VF */
       echo vfio-pci > /sys/bus/pci/devices/0002:81:10.0/driver_override
       echo 0002:81:10.0 > /sys/bus/pci/drivers_probe

    4. run qemu[2]
 
       qemu-system-aarch64 \
       -machine virt,gic-version=3 \
       -enable-kvm \
       -cpu host \
       -m 1024 \
       -kernel ./Image \
       -initrd ./minifs.cpio.gz \
       -nographic \
       -net none -device vfio-pci,host=0002:81:10.0,id=net0 \
       -D trace_log

    5. set networking configurations in guest machine and remote machine
       
       run ping and iperf to test

more scenarios
-----------------

    1. enable multiple VFs, assigned to one VM
    2. enable multiple VFs, assigned to different VMs
    3. use VF directly
    4. VF and PF communicate

performance
--------------
    1. should at least reach the performance of PF[3]

reference:
[1] should add pcie_acs_override=downstream in kernel command line:
    As our PCIe RP does not support ACS capbility, we should add this command line
    and related patch(https://lkml.org/lkml/2013/5/30/513) which will never be
    merged into mainline to tell kernel our PCIe RP indeed did something to
    provide address isolation just like ACS does.
    we will upstream in ACS quirk patch to fix this.
[2] how to compile qemu locally in D05:
    https://wangzhou.github.io/ARM64-qemu-native-build/
[3] should add 82599 patch to test, firstly make sure performance of 82599 PF is
    good.
