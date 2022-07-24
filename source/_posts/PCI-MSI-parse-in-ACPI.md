---
title: PCI MSI parse in ACPI
tags:
  - PCIe
  - ACPI
description: >-
  In server system, we often use ACPI table in UEFI to stort parametres of
  system. For a PCIe subsystem, IORT table will be used to store configurations
  of ITS, SMMU and RC. This document analyzes how to parse ITS configuration in
  kernel.
abbrlink: 21b6df7b
date: 2021-07-11 23:36:48
categories:
---

parse IORT table and build ITS interrupt domain
--------------------------------------------------
During PCI enumeration, pci_device_add will be called for each PCIe devices.
In pci_device_add, we will get the ITS irq domain for this PCIe device and store
this domain in this device's struct(pci_dev).
```
pci_device_add
    --> pci_set_msi_domain(dev)
        --> pci_dev_msi_domain(dev)
            --> pci_msi_get_device_domain
                --> iort_get_device_domain(&pdev->dev, rid)
                    --> iort_dev_find_its_id(dev, req_id, 0, &its_id)
                            /* key point to find the RC node in IORT table */
                        --> iort_find_dev_node(dev)
                            /*
			     * if the device is a pci device, we get its related
                             * root bus, and we scan the IORT table to get the
                             * PCI_ROOT_COMPLEX node of this root bus.
                             * 
                             * point is we use iort_match_node_callback to confirm
                             * that we find the right PCI_ROOT_COMPLEX. however,
                             * above function uses "segment value" to do the check
                             * 
                             * if we configure IORT as:
                             * RC0: segment0: ITS map1
                             * RC1: segment0: ITS map2
                             * 
                             * when we try to find a pcie device under RC1, finally
                             * we will get ITS map1 under RC0.[2]
                             * 
                             */
                    ...
                        /* return irq domain */
                    --> irq_find_matching_fwnode(handle, DOMAIN_BUS_PCI_MSI)


iort_dev_find_its_id
       /* find related RC node in IORT table */
    -->iort_find_dev_node
       /*
        * find related ITS node in MADT table here. Is there a bug?
        *
        * This function supports multiple mappings in one RC[1]:
        * 1. multiple ITS mappings.
        * 2. multiple SMMU mappings?
        * 3. multiple ITS/SMMU mappings
        *
        * One RC node in IORT table maps to one PCIe segment(one PCIe domain),
        * we can have multiple PCIe host bridges in one PCIe segment.
        *
        */
    -->iort_node_map_rid(node, req_id, NULL, IORT_MSI_TYPE)
```
get an interrtup in PCIe device driver
-----------------------------------------
When PCIe device wants to apply interrupts, it will call some functions like: pci_enable_msi.
these kind of function will first get irq domain stored in pci_dev, then will
call ITS driver to get an interrupt resource.


Reference:
----------

[1]
```
/* PCIe0 */
[0001]                               Type : 02
[0002]                             Length : 0034
[0001]                           Revision : 00
[0004]                           Reserved : 00000000
[0004]                      Mapping Count : 00000001
[0004]                     Mapping Offset : 00000020

[0008]                  Memory Properties : [IORT Memory Access Properties]
[0004]                    Cache Coherency : 00000001
[0001]              Hints (decoded below) : 00
                                Transient : 0
                           Write Allocate : 0
                            Read Allocate : 0
                                 Override : 0
[0002]                           Reserved : 0000
[0001]       Memory Flags (decoded below) : 00
                                Coherency : 0
                         Device Attribute : 0
[0004]                      ATS Attribute : 00000000
[0004]                 PCI Segment Number : 00000000

[0004]                         Input base : 00000000
[0004]                           ID Count : 00004000
[0004]                        Output Base : 00000000
[0004]                   Output Reference : 0000007c
[0004]              Flags (decoded below) : 00000000
                           Single Mapping : 0

[0004]                         Input base : 00006000
[0004]                           ID Count : 00000100
[0004]                        Output Base : 00006000
[0004]                   Output Reference : 0000007c
[0004]              Flags (decoded below) : 00000000
                           Single Mapping : 0
```
[2]

This is our orignal design. But now it seems it is wrong. Now I think the concept
of RC should be mapped to PCIe domain or a segment. So multiple ITS maps for
one RC can be configured as in [1].

Same rule can be applied for SMMU configuration.
