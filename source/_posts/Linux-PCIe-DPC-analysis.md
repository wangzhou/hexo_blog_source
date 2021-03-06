---
title: Linux PCIe DPC analysis
tags:
  - PCIe
  - Linux内核
description: 本文分析Linux内核PCIe DPC特性的软件架构
abbrlink: 702cc8e8
date: 2021-07-11 23:38:25
categories:
---

linux implements PCIe dpc feature as a service, which is registed in PCIe port
driver subsystem in drivers/pci/pcie/pcie-dpc.c

to check: if dpc_probe will be called for any port type, as in
          struct pcie_port_service_driver dpcdriver, port_type = PCIE_ANY_PORT.

let's assume dpc_probe will be called for any port type, then the flow of probe:
```
    --> INIT_WORK(&dpc->work, interrupt_event_handler);
    --> devm_request_irq(&dev->device, dev->irq, dpc_irq, IRQF_SHARED, "pcie-dpc", dpc);

        /* set PCI_EXP_DPC_CTL_EN_NONFATAL, 0x6 offset in dpc cap: 0x02
         * PCI_EXP_DPC_CTL_INT_EN
         */
    --> pci_write_config_word(pdev, dpc->cap_pos + PCI_EXP_DPC_CTL, ctl);
```
so it is quiet easy above, just enable the non-fatal and top interrupt.

In the irq handler:
```
        /* just read some data from dpc cap, then print info to user, we can
         * see which kind of error happened: unconrrectable, non-fatal, fatal.
         */
    --> pci_read_config_word(pdev, dpc->cap_pos + PCI_EXP_DPC_STATUS, &status);
    --> pci_read_config_word(pdev, dpc->cap_pos + PCI_EXP_DPC_SOURCE_ID, &source);

        /* run work queue register above, to check: how long to wait */
    --> schedule_work(&dpc->work);

            /* This handler will try to stop and remove all devices under this port.
             * Then wait link inactive, and set dpc cap's status register.
             *
             * If your rp implement dpc, and there is just one ep connnected to
             * rp directly, then this ep will be stopped and removed. when stopping
             * it, pme/aspm functions will be called.
             *
             * If your rp connects a switch, and there are some devices connected
             * to dp of this switch, when this rp 
             * under this rp will be stopped and removed.
             */
        --> interrupt_event_handler
                /* If your rp implements dpc, then the parent below should be
                 * the subordinate bus. Then for devices under this bus, it will
                 * call pci_stop_and_remove_bus_device(dev) to them one by one
                 */
            --> struct pci_bus *parent = pdev->subordinate;
            --> pci_stop_and_remove_bus_device(dev);
                    /* below function will be called in a recursive way, so the
                     * first device to be stopped and removed will be the deepest one.
                     * Then one by one, all devices will be stopped and removed.
                     */
                --> pci_stop_bus_device(dev);
                        /* to check: it will call pme/aspm functions in below function */ 
                    --> pci_stop_dev(dev);
                    /* in a same recursive way in below function */
                --> pci_remove_bus_device(dev);
                        /* to check: will call pm functions */
                    --> pci_bridge_d3_device_removed(dev);
                        /* all software structures destroy */
                    --> pci_destroy_dev(dev);

                /* here stop maximum 1s to wait Data Link Layer Link Active bit
                 * in Link status register to become 0.
                 * If this bit is 1, it indicates the DL_Active state ?
                 */
            --> dpc_wait_link_inactive(pdev);
                /* set dpc trigger status bit and dpc interrupt status bit in
                 * dpc status register.
                 */
            --> pci_write_config_word(pdev, dpc->cap_pos + PCI_EXP_DPC_STATUS,
		                      PCI_EXP_DPC_STATUS_TRIGGER | PCI_EXP_DPC_STATUS_INTERRUPT);

```

