---
title: How to dump acpi tables in CentOS
tags:
  - UEFI
  - ACPI
description: This document shows how to dump ACPI table in a Linux system
abbrlink: e8dc22fe
date: 2021-07-05 22:29:37
categories:
---

My system is ARM64 CentOS7.4

1.yum install acpica-tools.aarch64

2.acpidump > acpidump.out 
  
3.acpixtract -a acpidump.out
```
[root@157 acpi_test]# acpixtract -a acpidump.out

Intel ACPI Component Architecture
ACPI Binary Table Extraction Utility version 20160527-64
Copyright (c) 2000 - 2016 Intel Corporation

Acpi table [SPCR] -      80 bytes written to spcr.dat
Acpi table [MCFG] -     172 bytes written to mcfg.dat
Acpi table [GTDT] -     124 bytes written to gtdt.dat
Acpi table [APIC] -    5348 bytes written to apic.dat
Acpi table [IORT] -    1300 bytes written to iort.dat
Acpi table [SLIT] -      60 bytes written to slit.dat
Acpi table [DSDT] -   26427 bytes written to dsdt.dat
Acpi table [SRAT] -    1400 bytes written to srat.dat
Acpi table [DBG2] -      90 bytes written to dbg2.dat
Acpi table [FACP] -     276 bytes written to facp.dat

10 binary ACPI tables extracted
[root@157 acpi_test]# ls
acpidump.out  apic.dat  dbg2.dat  dsdt.dat  facp.dat  gtdt.dat  iort.dat  mcfg.dat  slit.dat  spcr.dat  srat.dat

```
   After this command, you can see related ACPI tables.

4.iasl -d *.dat

e.g. un-compile mcfg.dat:
```
[root@157 acpi_test]# iasl -d mcfg.dat

Intel ACPI Component Architecture
ASL+ Optimizing Compiler version 20160527-64
Copyright (c) 2000 - 2016 Intel Corporation

Input file mcfg.dat, Length 0xAC (172) bytes
ACPI: MCFG 0x0000000000000000 0000AC (v01 HISI   HIP07    00000000 INTL 20151124)
Acpi Data Table [MCFG] decoded
Formatted output:  mcfg.dsl - 3955 bytes
[root@157 acpi_test]# ls mcfg.dsl
mcfg.dsl
```

5.Then you can view it: vim mcfg.dsl
