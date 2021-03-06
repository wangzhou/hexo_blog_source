---
title: 'PCIe userspace tools: lspci, setpci and sysfs'
tags:
  - 软件调试
  - PCIe
description: >-
  From mj.ucw.cz/sw/pciutils we can get the code of lspci/setpci, which are two
  useful tools to debug PCIe related problems. This doc just introduces these
  two tools and also the sysfs interfaces which can be used in PCIe problem
  debug.
abbrlink: f1aca621
date: 2021-07-05 22:41:09
categories:
---

lspci
-----

 lspci
```
00:00.0 Host bridge: Intel Corporation Xeon E7 v3/Xeon E5 v3/Core i7 DMI2 (rev 02)
00:01.0 PCI bridge: Intel Corporation Xeon E7 v3/Xeon E5 v3/Core i7 PCI Express Root Port 1 (rev 02)
00:02.0 PCI bridge: Intel Corporation Xeon E7 v3/Xeon E5 v3/Core i7 PCI Express Root Port 2 (rev 02)
00:02.2 PCI bridge: Intel Corporation Xeon E7 v3/Xeon E5 v3/Core i7 PCI Express Root Port 2 (rev 02)
00:03.0 PCI bridge: Intel Corporation Xeon E7 v3/Xeon E5 v3/Core i7 PCI Express Root Port 3 (rev 02)
00:03.2 PCI bridge: Intel Corporation Xeon E7 v3/Xeon E5 v3/Core i7 PCI Express Root Port 3 (rev 02)
```
A general show of PCIe devices in system.

 lspci -t
```
[...]
 \-[0000:00]-+-00.0
             +-01.0-[01]----00.0
             +-02.0-[02]--+-00.0
             |            +-00.1
             |            +-00.2
             |            \-00.3
             +-02.2-[03]--
             +-03.0-[04]--
             +-03.2-[05]--
[...]
```
Show PCIe topology in tree picture. As showed in above picture, 0000:00 means
domain 0 bus 0; 01.0 means a device under bus 0, so its BDF should be 0000:00:01.0;
01.0-[01] here [01] means the bus number under device 00:01.0, some times we may
get [xx-yy] which means the bus range under this device, apparently this device
is a PCIe bridge; 01.0-[01]----00.0 here 00.0 means a device which device:function
is 00.0, so together with its father bus, its BDF should be 0000:01:00.0.
  
 lspci -s ff:0f.1
```
ff:0f.1 System peripheral: Intel Corporation Xeon E7 v3/Xeon E5 v3/Core i7 Buffered Ring Agent (rev 02)
```
To see the specific device information, use lspci -s BDF

 lspci -vvv
```
80:05.4 PIC: Intel Corporation Xeon E7 v3/Xeon E5 v3/Core i7 I/O APIC (rev 02) (prog-if 20 [IO(X)-APIC])
	Subsystem: Device 19e5:2060
	Control: I/O- Mem+ BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx-
	Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
	Latency: 0
	Region 0: Memory at c8000000 (32-bit, non-prefetchable) [size=4K]
	Capabilities: <access denied>
```
To see more information, use -vvv/-vv/-v. You can see BAR in "Region", different
bridge window range and different capabilities.

 lspci -xxx
```
7f:13.1 System peripheral: Intel Corporation Xeon E7 v3/Xeon E5 v3/Core i7 Integrated Memory Controller 0 Target Address, Thermal & RAS Registers (rev 02)
00: 86 80 71 2f 00 00 10 00 02 00 80 08 00 00 80 00
10: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
20: 00 00 00 00 00 00 00 00 00 00 00 00 e5 19 60 20
30: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
```
To see configure space as bit value, use -xxx/-xx/-x


setpci
------

 setpci -s BDF 20.L=0x12345678

 Above command set 0x20 offset of configure space of a device which BDF is BDF
 to 0x12345678. L here means 32bit, W will mean 16bit, B will mean 8bit.


sysfs
-----

 - remove

   We can remove a device by writing 1 to its remove file.

 - rescan

   We can rescan a device/bus by writing 1 to its rescan file, we can also rescan
   whole PCIe system by echo 1 > /sys/bus/pci/rescan (need to check...).

   When we do rescan a device, we find its father bus, and pass this bus to PCIe
   enumeration process.

   However, if we rescan a RP or PCIe bridge, as the structure of related RP or
   PCIe bridge is still there, Linux kernel will do nothing about MEM/IO window
   of related RP or PCIe bridge.(however, writing/reading MEM/IO operation will
   be done to checkout if this bridge's MEM/IO window register is working)

 - reset
   
   (to do...)

