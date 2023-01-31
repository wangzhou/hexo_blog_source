---
title: PCI ACPI笔记1
tags:
  - PCIe
  - ACPI
description: This note talks about some basic knowledge about ACPI and PCI.
abbrlink: 9919e89b
date: 2021-07-17 10:47:28
categories:
---

ACPI
-------

For the basic knowledge, you can refer to "PCI Express 体系结构导读" by
WangQi, in charpter 14, there are related stuffs about ACPI.

And you can also refer to kernel source code linux/Documentation/acpi/ to find
ACPI related documents. ACPI Definition Blocks is as below, this figure is
copied from linux/Documentation/acpi/namespace.txt.
```
     +---------+    +-------+    +--------+    +------------------------+
     |  RSDP   | +->| XSDT  | +->|  FADT  |    |  +-------------------+ |
     +---------+ |  +-------+ |  +--------+  +-|->|       DSDT        | |
     | Pointer | |  | Entry |-+  | ...... |  | |  +-------------------+ |
     +---------+ |  +-------+    | X_DSDT |--+ |  | Definition Blocks | |
     | Pointer |-+  | ..... |    | ...... |    |  +-------------------+ |
     +---------+    +-------+    +--------+    |  +-------------------+ |
                    | Entry |------------------|->|       SSDT        | |
                    +- - - -+                  |  +-------------------| |
                    | Entry | - - - - - - - -+ |  | Definition Blocks | |
                    +- - - -+                | |  +-------------------+ |
                                             | |  +- - - - - - - - - -+ |
                                             +-|->|       SSDT        | |
                                               |  +-------------------+ |
                                               |  | Definition Blocks | |
                                               |  +- - - - - - - - - -+ |
                                               +------------------------+
                                                           |
                                              OSPM Loading |
                                                          \|/
                                                    +----------------+
                                                    | ACPI Namespace |
                                                    +----------------+
```
PCI using ACPI
-----------------

a. work flows:
```
/* just register struct acpi_bus_type acpi_pci_bus to list: bus_type_list */
acpi_pci_init() /* arch_initcall(acpi_pci_init) */

acpi_init() /* subsys_initcall(acpi_init) */
    --> acpi_scan_init()
        /*
         * register pci_root_handler to list: acpi_scan_handlers_list.
	 * the attach will be called in acpi_scan_attach_handler().
	 * there attach is assigned as acpi_pci_root_add()
	 */
        --> acpi_pci_root_init()
	/*
         * register pci_link_handler to list: acpi_scan_handlers_list.
	 * this handler has relationship with PCI IRQ.
	 */
	--> acpi_pci_link_init()
	/* we facus on PCI-ACPI, ignore other handlers' init */
	...
        --> acpi_bus_scan()
	    /* create struct acpi_devices for all device in this system */
	    --> acpi_walk_namespace()
	    --> acpi_bus_attach()
	        --> acpi_scan_attach_handler()
		    --> acpi_scan_match_handler()
		    --> handler->attach /* attach is acpi_pci_root_add */

acpi_pci_root_add()
    /*
     * in kernel, there are two pci_acpi_scan_root, they are in
     * arch/ia64/pci/pci.c and arch/x86/pci/acpi.c.
     * if we will implement PCI using ACPI in ARM64, we should implement
     * another this kind of function in arch/arm64/kernel/pci.c.
     * in pci_acpi_scan_root, will allocate struct pci_controller and
     * struct pci_root_info.
     */
    --> pci_acpi_scan_root()
        --> probe_pci_root_info()
	    /*
	     * will called twice, first for count_window, second for add window.
	     * this function will get infomation from ACPI table.
	     */
	    --> acpi_walk_resources() /* drivers/acpi/acpica/rsxface.c */
```
b. basic structs:
```
global list: acpi_scan_handlers_list /* drivers/acpi/scan.c */
element: struct acpi_scan_handler

static list: bus_type_list /* drivers/acpi */
element: struct acpi_bus_type

struct acpi_scan_handler:
     struct acpi_device_id *ids;
     attach;
     ...

struct acpi_bus_type
struct acpi_device

struct pci_controller
struct pci_root_info:
    struct pci_controller *controller;
```
