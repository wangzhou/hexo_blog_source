---
title: PCI SMMU parse in ACPI
tags:
  - PCIe
  - Linux内核
  - ACPI
description: >-
  This document shares a code analysis based on Linux-v4.9 about how to parse
  SMMU information in ACPI IORT table, and how does a PCIe device get its SMMUv3
  device.
abbrlink: 9dde42f2
date: 2021-07-11 23:36:35
categories:
---

acpi smmu parse
------------------
```
acpi_init
    --> acpi_iort_init
            /* parse the SMMU node in IORT table */
        --> iort_init_platform_devices
                /* create a platform_device for SMMU, and add it to platform_bus */
            --> iort_add_smmu_platform_device
                    /* why call this function here? */
                --> acpi_dma_configure
                    --> iort_iommu_configure

            /* call functions in section: __iort_acpi_probe_table ~ __iort_acpi_probe_table_end */
        --> acpi_probe_device_table(iort)
```
put arm_smmu_init in above section in compile phase
------------------------------------------------------
```
IORT_ACPI_DECLARE(arm_smmu_v3, ACPI_SIG_IORT, acpi_smmu_v3_init);
arm_smmu_v3_init 
        /*
         * add SMMU driver to platform_bus, this will trigger probe function to
         * bind this driver and above SMMU platform_device.
         */
    --> platform_driver_register(&arm_smmu_driver)
```
PCIe device get its SMMU device
----------------------------------
```
pci_device_add
    --> pci_dma_configure(dev)
        --> acpi_dma_configure(&dev->dev, attr)

acpi_dma_configure
       /* if dev is a pci device, first find related RC node in IORT table */
    -->iommu = iort_iommu_configure 
            /* RC node */
        --> node = iort_scan_node
                /* parent here is SMMU node */
            --> parent = iort_node_map_rid
```
