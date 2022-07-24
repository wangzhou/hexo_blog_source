---
title: PCI parse MEM/IO range in _CRS in ACPI table
tags:
  - PCIe
  - Linux内核
description: >-
  Linux PCI code parses MEM/IO range, which are described in _CRS method in ACPI
  DSDT table. This document tries to share an analysis about this part of code.
abbrlink: 68a70987
date: 2021-07-11 23:36:08
categories:
---

```
pci_acpi_scan_root(struct acpi_pci_root *root)
    --> root_ops->prepare_resources = pci_acpi_root_prepare_resources
        --> acpi_pci_root_create
            ...
            --> ops->prepare_resources(info)
            ...

pci_acpi_root_prepare_resources
    --> acpi_pci_probe_root_resources(ci)
        ...
            /* this is the key function */
        --> acpi_dev_get_resources
        ...

/* will analyze acpi_dev_get_resources below */
acpi_dev_get_resources(device, list, acpi_dev_filter_resource_type_cb, (void *)flags);
    --> c.list = list;
	c.preproc = preproc; /* acpi_dev_filter_resource_type_cb */
	c.preproc_data = preproc_data; /* (void *)flags */
	c.count = 0;
	c.error = 0;
        /*
         * will parse _CRS method for above device, acpi_walk_resources is defined
         * in drivers/acpi/acpica/rsxface.c
         *
         * acpi_dev_process_resource will be called for every resources in _CRS,
         * it parses a resource and adds it into the list in c.list.
         *
         * this means e.g. if there are 3 MEM/IO range, acpi_dev_process_resource
         * will be called three times.
         */
    --> acpi_walk_resources(adev->handle, METHOD_NAME__CRS, acpi_dev_process_resource, &c);

/* details of acpi_dev_process_resource */
static acpi_status acpi_dev_process_resource(struct acpi_resource *ares, void *context)
        /* filter the resource type to some kinds */
    --> c->preproc(ares, c->preproc_data)
    ...
        /*
         * still do not know the difference of below two functions, prefetchable
         * attribute will be parsed in acpi_decode_space function.
         */
    --> acpi_dev_resource_memory(ares, res)
    --> acpi_dev_resource_address_space(ares, &win)
        --> acpi_decode_space
            --> if (addr->info.mem.caching == ACPI_PREFETCHABLE_MEMORY)
    ...
        /* add resource to list */
    --> acpi_dev_new_resource_entry(&win, c)
        --> resource_list_add_tail(rentry, c->list);
```

Some ideas about PCI prefetch window and memory type/attribute of AMRv8[1]
-----------------------------------------------------------------------

PCI prefetch and memory type/attribute has no relationship with each other,
as PCI prefetch is an attribute in PCI domain, and memory type/attribute is for
ARM bus domain. Further more, ATU(address translate unit: translate a CPU address
to PCI address) only translate address, do nothing about prefetch support.

Generally, when we get cpu physical address(PA) for a BAR, PCI device driver will
use ioremap(or other ioremap_*) to get a VA. memory type/attribute will be set
to MMU in ioremap kind of functions to support memory operations in ARM bus domain.

For ECAM, we use below work flow to get VA of ECAM, we can see nGnRE will be
set for ECAM memory type/attribute.
```
/* will ioremap ECAM in below ioremap using nGnRE */
pci_acpi_scan_root
    --> pci_acpi_scan_root
        --> pci_ecam_create
            --> cfg->win = ioremap
```
Reference:
[1] ARM Cortex-A Series Programmer's Guide for ARMv8-A(13: Memory Ordering) 
