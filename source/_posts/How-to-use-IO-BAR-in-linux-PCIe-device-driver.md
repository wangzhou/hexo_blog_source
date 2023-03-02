---
title: How to use IO BAR in linux PCIe device driver
tags:
  - 计算机体系结构
  - PCIe
description: >-
  This document shows how to use an IO BAR in PCIe device. Currently there is
  few devices supported IO BAR, however, when we have a card which needs to use
  IO BAR, this document shares a basic logic about it
abbrlink: ba71ce37
date: 2021-07-05 22:43:23
categories:
---

IO window parse analysis
--------------------------

   In an ACPI based system, we parse the IO window configured in DSDT table, as
   showed in this [link](https://wangzhou.github.io/PCI-parse-MEM-IO-range-in-CRS-in-ACPI-table/).

   We can see in pci_acpi_root_prepare_resources:
```
        pci_acpi_root_prepare_resources
            --> acpi_pci_probe_root_resources
                --> acpi_pci_root_remap_iospace
```
   in acpi_pci_root_remap_iospace, CPU address of one PCIe IO window will be
   mapped to PCI_IOBASE based system IO space, like below picture:

```
     PCI_IOBASE            PCI_IOBASE + PCIE_IO_SIZE - 1
           |<------------->|
   --------+---------------+---------------------------   <- CPU VA in kernel
       |   |      |\      \
       |   |      | \      \                  
       v   |      |  \      \                  maps supported by MMU
           |      |   \      \       
           |      |    \      \
           |<---->|     |<---->|   <- CPU PA
       |    \      \     \      \
       |     \      \     \      \             maps supported by ATU
       v      \      \     \      \
               \      \     \      \
                \      \     \      \
                 |<---->|     |<---->|  <- PCI address

                IO window 1   IO window 2
               host bridge 1  host bridge 2
```

   and offset between CPU VA and PCI_IOBASE will be stored in resource_entry,
   which will be passed to PCI enumeration. The offset in resource_entry will be
   offset(above) - PCI address. In the process of the enumeration, IO BAR will
   be allocated in related IO window.

   the base of IO BAR will be stored in (pci_dev -> resource[IO BAR].start), which
   is the offset to PCI_IOBASE in CPU VA.


How to use in PCIe device driver
----------------------------------

   In one hardware arch, we use inb/outb, inw/outw ... function to access IO
   space. In ARM64, these functions are defined in linux/include/asm-generic/io.h,
   like: 
```
        #ifndef outb
        #define outb outb
        static inline void outb(u8 value, unsigned long addr)
        {
                writeb(value, PCI_IOBASE + addr);
        }
        #endif
```
   So when we want to read/write IO BAR in PCIe device driver, we should:

   1. get the base of one IO BAR by: addr = pci_resource_start(dev, bar)
   2. use outb(value, addr) for an example to do port input by byte-width.

