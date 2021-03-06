---
title: Linux/UEFI memory block note
tags:
  - 计算机体系结构
description: This doc shows how UEFI reports memory blocks to OS
abbrlink: 6572c16f
date: 2021-07-05 22:29:05
categories:
---

D05 CPU and memory hardware topology
```
      |||| +--------+  |||| ||||  +---------+  ||||
      |||| |        |  |||| ||||  |         |  ||||
      |||| |  CPU   |  |||| ||||  |   CPU   |  ||||
      |||| |        |  |||| ||||  |         |  ||||
      |||| +--------+  |||| ||||  +---------+  ||||


       Dim A        Dim B
         |     | |    |   | |
         +---> | | <--+   | |
               | |        | |
               | |        | |
       +-------+-+--------+-+---------+
       |      ++-+-+     ++-+-+       | <-- CPU
       |      |ddrc|     |ddrc|       |
       |     ++----+-+  ++----+-+     |
       |     |       |  |       |     |
       |     |       +--+       |     |
       |     | C Die |  | C Die |     |
       |     |       +--+       |     |
       |     |       |  |       |     |
       |     ++----+-+  ++----+-+ ... |
       |      |ddrc|     |ddrc|       |
       |      ++-+-+     ++-+-+       |
       +-------+-+--------+-+---------+
               | |        | |
               | |        | |
               | |        | |
               | |        | |
```
ACPI SRAT table will discribe this topology. In SRAT table CPU will be discribed
statically, but memory block will be discribed drynamically. UEFI will detect
the DDR in the Dims and report memory blocks in SRAT table to OS.

The logic of how UEFI defines a memmroy block is based on specific UEFI. 
Now D05 UEFI will assigned memory range for each ddrc, e.g. 0 ~ 32G. ddr plugging
in specific dim will get the cpu address from its ddrc.

D05 UEFI can connect two memory rangs of different ddrc to one memory block and
report this to OS, if these two memory ranges' ddr is contiguous:

  contiguous:      ddrc 0 ~ 32G, plugging 2 16G ddr.

  not contiguous:  ddrc 0 ~ 32G, plugging 1 16G ddr.


In D05, run dmesg | grep SRAT:
```
[    0.000000] ACPI: SRAT 0x00000000397F0000 000578 (v03 HISI   HIP07    00000000 INTL 20151124)
[    0.000000] ACPI: NUMA: SRAT: PXM 0 -> MPIDR 0x10000 -> Node 0
[    0.000000] ACPI: NUMA: SRAT: PXM 0 -> MPIDR 0x10001 -> Node 0
[    0.000000] ACPI: NUMA: SRAT: PXM 0 -> MPIDR 0x10002 -> Node 0
[    0.000000] ACPI: NUMA: SRAT: PXM 0 -> MPIDR 0x10003 -> Node 0
[    0.000000] ACPI: NUMA: SRAT: PXM 0 -> MPIDR 0x10100 -> Node 0
[    0.000000] ACPI: NUMA: SRAT: PXM 0 -> MPIDR 0x10101 -> Node 0
[    0.000000] ACPI: NUMA: SRAT: PXM 0 -> MPIDR 0x10102 -> Node 0
[    0.000000] ACPI: NUMA: SRAT: PXM 0 -> MPIDR 0x10103 -> Node 0
[    0.000000] ACPI: NUMA: SRAT: PXM 0 -> MPIDR 0x10200 -> Node 0
[    0.000000] ACPI: NUMA: SRAT: PXM 0 -> MPIDR 0x10201 -> Node 0
[    0.000000] ACPI: NUMA: SRAT: PXM 0 -> MPIDR 0x10202 -> Node 0
[    0.000000] ACPI: NUMA: SRAT: PXM 0 -> MPIDR 0x10203 -> Node 0
[    0.000000] ACPI: NUMA: SRAT: PXM 0 -> MPIDR 0x10300 -> Node 0
[    0.000000] ACPI: NUMA: SRAT: PXM 0 -> MPIDR 0x10301 -> Node 0
[    0.000000] ACPI: NUMA: SRAT: PXM 0 -> MPIDR 0x10302 -> Node 0
[    0.000000] ACPI: NUMA: SRAT: PXM 0 -> MPIDR 0x10303 -> Node 0
[    0.000000] ACPI: NUMA: SRAT: PXM 1 -> MPIDR 0x30000 -> Node 1
[    0.000000] ACPI: NUMA: SRAT: PXM 1 -> MPIDR 0x30001 -> Node 1
[    0.000000] ACPI: NUMA: SRAT: PXM 1 -> MPIDR 0x30002 -> Node 1
[    0.000000] ACPI: NUMA: SRAT: PXM 1 -> MPIDR 0x30003 -> Node 1
[    0.000000] ACPI: NUMA: SRAT: PXM 1 -> MPIDR 0x30100 -> Node 1
[    0.000000] ACPI: NUMA: SRAT: PXM 1 -> MPIDR 0x30101 -> Node 1
[    0.000000] ACPI: NUMA: SRAT: PXM 1 -> MPIDR 0x30102 -> Node 1
[    0.000000] ACPI: NUMA: SRAT: PXM 1 -> MPIDR 0x30103 -> Node 1
[    0.000000] ACPI: NUMA: SRAT: PXM 1 -> MPIDR 0x30200 -> Node 1
[    0.000000] ACPI: NUMA: SRAT: PXM 1 -> MPIDR 0x30201 -> Node 1
[    0.000000] ACPI: NUMA: SRAT: PXM 1 -> MPIDR 0x30202 -> Node 1
[    0.000000] ACPI: NUMA: SRAT: PXM 1 -> MPIDR 0x30203 -> Node 1
[    0.000000] ACPI: NUMA: SRAT: PXM 1 -> MPIDR 0x30300 -> Node 1
[    0.000000] ACPI: NUMA: SRAT: PXM 1 -> MPIDR 0x30301 -> Node 1
[    0.000000] ACPI: NUMA: SRAT: PXM 1 -> MPIDR 0x30302 -> Node 1
[    0.000000] ACPI: NUMA: SRAT: PXM 1 -> MPIDR 0x30303 -> Node 1
[    0.000000] ACPI: NUMA: SRAT: PXM 2 -> MPIDR 0x50000 -> Node 2
[    0.000000] ACPI: NUMA: SRAT: PXM 2 -> MPIDR 0x50001 -> Node 2
[    0.000000] ACPI: NUMA: SRAT: PXM 2 -> MPIDR 0x50002 -> Node 2
[    0.000000] ACPI: NUMA: SRAT: PXM 2 -> MPIDR 0x50003 -> Node 2
[    0.000000] ACPI: NUMA: SRAT: PXM 2 -> MPIDR 0x50100 -> Node 2
[    0.000000] ACPI: NUMA: SRAT: PXM 2 -> MPIDR 0x50101 -> Node 2
[    0.000000] ACPI: NUMA: SRAT: PXM 2 -> MPIDR 0x50102 -> Node 2
[    0.000000] ACPI: NUMA: SRAT: PXM 2 -> MPIDR 0x50103 -> Node 2
[    0.000000] ACPI: NUMA: SRAT: PXM 2 -> MPIDR 0x50200 -> Node 2
[    0.000000] ACPI: NUMA: SRAT: PXM 2 -> MPIDR 0x50201 -> Node 2
[    0.000000] ACPI: NUMA: SRAT: PXM 2 -> MPIDR 0x50202 -> Node 2
[    0.000000] ACPI: NUMA: SRAT: PXM 2 -> MPIDR 0x50203 -> Node 2
[    0.000000] ACPI: NUMA: SRAT: PXM 2 -> MPIDR 0x50300 -> Node 2
[    0.000000] ACPI: NUMA: SRAT: PXM 2 -> MPIDR 0x50301 -> Node 2
[    0.000000] ACPI: NUMA: SRAT: PXM 2 -> MPIDR 0x50302 -> Node 2
[    0.000000] ACPI: NUMA: SRAT: PXM 2 -> MPIDR 0x50303 -> Node 2
[    0.000000] ACPI: NUMA: SRAT: PXM 3 -> MPIDR 0x70000 -> Node 3
[    0.000000] ACPI: NUMA: SRAT: PXM 3 -> MPIDR 0x70001 -> Node 3
[    0.000000] ACPI: NUMA: SRAT: PXM 3 -> MPIDR 0x70002 -> Node 3
[    0.000000] ACPI: NUMA: SRAT: PXM 3 -> MPIDR 0x70003 -> Node 3
[    0.000000] ACPI: NUMA: SRAT: PXM 3 -> MPIDR 0x70100 -> Node 3
[    0.000000] ACPI: NUMA: SRAT: PXM 3 -> MPIDR 0x70101 -> Node 3
[    0.000000] ACPI: NUMA: SRAT: PXM 3 -> MPIDR 0x70102 -> Node 3
[    0.000000] ACPI: NUMA: SRAT: PXM 3 -> MPIDR 0x70103 -> Node 3
[    0.000000] ACPI: NUMA: SRAT: PXM 3 -> MPIDR 0x70200 -> Node 3
[    0.000000] ACPI: NUMA: SRAT: PXM 3 -> MPIDR 0x70201 -> Node 3
[    0.000000] ACPI: NUMA: SRAT: PXM 3 -> MPIDR 0x70202 -> Node 3
[    0.000000] ACPI: NUMA: SRAT: PXM 3 -> MPIDR 0x70203 -> Node 3
[    0.000000] ACPI: NUMA: SRAT: PXM 3 -> MPIDR 0x70300 -> Node 3
[    0.000000] ACPI: NUMA: SRAT: PXM 3 -> MPIDR 0x70301 -> Node 3
[    0.000000] ACPI: NUMA: SRAT: PXM 3 -> MPIDR 0x70302 -> Node 3
[    0.000000] ACPI: NUMA: SRAT: PXM 3 -> MPIDR 0x70303 -> Node 3
[    0.000000] ACPI: SRAT: Node 0 PXM 0 [mem 0x00000000-0x3fffffff]
[    0.000000] ACPI: SRAT: Node 1 PXM 1 [mem 0x1400000000-0x17ffffffff]
[    0.000000] ACPI: SRAT: Node 0 PXM 0 [mem 0x1000000000-0x13ffffffff]
[    0.000000] ACPI: SRAT: Node 3 PXM 3 [mem 0x8800000000-0x8bffffffff]
[    0.000000] ACPI: SRAT: Node 2 PXM 2 [mem 0x8400000000-0x87ffffffff]
```
