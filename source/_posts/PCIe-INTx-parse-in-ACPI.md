---
title: PCIe INTx parse in ACPI
tags:
  - PCIe
  - ACPI
description: 本文分析Linux系统里PCIe INTx中断的ACPI解析过程
abbrlink: fbb77401
date: 2021-07-11 23:17:22
categories:
---

When probing a PCIe device, it will assign INTx to it.
```
pci_device_probe
        /* arch/arm64/kernel/pci.c */
    --> pcibios_alloc_irq
        --> acpi_pci_irq_enable(dev)
            --> pin = dev->pin
                /* From a EP, we find the related INTx in the RP.
                 * It will search from the EP which wants to be assigned a INTx
                 * to find a host bridge which has the INTx configure(will be in
                 * the host bridge's DSDT device)
                 * The device(RP or IEP) which directly connects the logic host
                 * bridge and the pin number after swizzling will be passed to
                 * check function to find correct irq number
                 * (check function: acpi_pci_irq_check_entry in
                 * acpi_pci_irq_find_prt_entry)
                 */
            --> acpi_pci_irq_lookup(dev, pin)
                --> acpi_pci_irq_find_prt_entry(dev, pin, &entry)
                --> while (bridge) {
                        --> pin = pci_swizzle_interrupt_pin(dev, pin)
                        --> acpi_pci_irq_find_prt_entry(bridge, pin, &entry)
                    }
                /* get irq number in _PRT */
            --> gsi = entry->index
                /* driver/acpi/irq.c */
            --> rc = acpi_register_gsi(&dev->dev, gsi, triggering, polarity)
                --> fwspec.fwnode = acpi_gsi_domain_id
                    /* This is the API to get virq */
                --> irq_create_fwspec_mapping(&fwspec)
                /* here rc is the related virq */
            --> dev->irq = rc
```
If we use GICv3:
```
gic_acpi_init
    --> acpi_set_irq_model(ACPI_IRQ_MODEL_GIC, domain_handle)
        --> acpi_irq_model = model
        --> acpi_gsi_domain_id = fwnode
```
We can get GICD domain like above.

dev->pin had been set in below flow:
```
pci_setup_device
    --> pci_read_irq
        --> pci_read_config_byte(dev, PCI_INTERRUPT_PIN, &irq)
        --> dev->pin = irq
```

We can add INTx configure in ACPI DSDT as described in 6.2.13.1 in ACPI spec 6.1.
```
        Name(_PRT, Package{
                Package{0xFFFF,0,0,640},   // INT_A
                Package{0xFFFF,1,0,641},   // INT_B
                Package{0xFFFF,2,0,642},   // INT_C
                Package{0xFFFF,3,0,643}    // INT_D
        })                |    | |  |
            +-------------+    | |  +---------------+
            |       -----------+ |                  |
         All_BDF   pin   interrup_controller   hw_irq_number
```
If interrup_controller field is 0, we will use "global interrupt pool" mentioned
in ACPI spec. In ARM world, this "global interrupt pool" will be GICD which is
defined in above gic_acpi_init

If our hardware topology is like this:
```
        system bus  
             |         
             +----- RP0(device number is 0)
             |
             +----- RP1(device number is 1)
             |
             +----- RP2(device number is 2)
             |
             +----- IEP(device number is y)
             |
```
We can configure the INTx _RPT as:
```
        Name(_PRT, Package{
                Package{0xFFFF,0,0,RP0_INTA},
                Package{0xFFFF,1,0,RP0_INTB},
                Package{0xFFFF,2,0,RP0_INTC},
                Package{0xFFFF,3,0,RP0_INTD},
                Package{0x1FFFF,0,0,RP1_INTA},
                Package{0x1FFFF,1,0,RP1_INTB},
                Package{0x1FFFF,2,0,RP1_INTC},
                Package{0x1FFFF,3,0,RP1_INTD},
                Package{0x2FFFF,0,0,RP2_INTA},
                Package{0x2FFFF,1,0,RP2_INTB},
                Package{0x2FFFF,2,0,RP2_INTC},
                Package{0x2FFFF,3,0,RP2_INTD} 
                Package{0xyFFFF,1,0,IEP_INTA} 
        })
```

NOTE:
 把dts下INTx中断的解析也放到这个文档里吧，不想另外起文档了。调用链大致是：
```
pci_device_add()
    -->pcibios_add_device(dev)
        -->dev->irq = of_irq_parse_and_map_pci(dev, 0, 0)
	    -->of_irq_parse_and_map_pci()
	           /* check if pci_dev has a device_node */
	        -->pci_device_to_OF_node(pdev); ?
		   /* what is pin for? */
		-->pci_read_config_byte(pdev, PCI_INTERRUPT_PIN, &pin)
		-->of_irq_parse_raw
		      /* just parse interrupt-cell, interrupt-map,
		       * interrupt-map-mask
		       */
		   --> of_get_property(...,"interrupt-cell",...")

	    -->irq_create_of_mapping()
```
可见，PCI的核心代码pci_device_add()会扫面dts中的信息，然后给对应的中断分配
中断号资源。分配好中断号(virq)会写到pci_dev->irq中，供pci设备驱动注册中断handler
的时候使用。各个pci设备中注册的中断handler有时会共享一个INTx中短线(e.g. INTa)。
这时一旦一个INTx中断被触发，不同设备上的中断handler都会被调用到。可见注册的时候，
这些中断handler都应该是shareable的。
