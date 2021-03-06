---
title: Linux PCIe hotplug arch analysis
tags:
  - PCIe
  - Linux内核
description: 本文分析Linux内核里PCIe热插拔的代码架构
abbrlink: 487ff967
date: 2021-07-11 23:37:56
categories:
---

This analysis is based on v4.8-rc5.

There are two kinds of PCIe hotplug: one is PCIe native hotplug which is implemented
just used the codes in linux kernel, second is PCIe hotplug based on ACPI. Here
we just analyze the native hotplug.

PCIe hotplug is implemented as a pcie port service, it is registered in pcie port
driver in drivers/pci/hotplug/pciehp_core.c:
```
--> pcied_init(void)
    --> pcie_port_service_register(&hpdriver_portdrv)
```
so system will finally call pciehp_probe in struct pcie_port_service_driver hpdriver_portdrv,
main flow of this probe:
```
pciehp_probe(struct pcie_device *dev)
        /* create struct controller, struct slot, get slot capability and slot status info */
    --> pcie_init(dev);
            /* will register a delay_work in slot:
             * INIT_DELAYED_WORK(&slot->work, pciehp_queue_pushbutton_work)
             *
             * This will be called, e.g. in hot-insertion, after inserting card
             * and press button and wait 5s.
             */
        --> pcie_init_slot
        /* create struct hotplug_slot, struct hotplug_slot_info, struct hotplug_slot_ops,
         * fill functions in hotplug_slot_ops
         *
         *   controller
         *        +------> slot
         *                   +----> hotplug_slot
         *                               +-------> pci_slot
         *                                         hotplug_slot_ops
         *                                         hotplug_slot_info
         *                   +----> delayed_work
         *                   +----> workqueue_struct
         *
         * then the whole structures will like above.
         *
         * After above structure been built, it will call pci_hp_register
         */
    --> init_slot(ctrl);
            /* add hotplug_slot into pci_hotplug_slot_list */
        --> pci_hp_register(hotplug, ctrl->pcie->port->subordinate, 0, name);
                /* to check: where to init pci_slot */
            --> pci_create_slot
            --> list_add(&slot->slot_list, &pci_hotplug_slot_list);
                /* expose pci_slot related file in sys-fs */
            --> fs_add_slot(pci_slot);
        /* register irq */
    --> pcie_init_notification(ctrl);
            /* when request_irq, will let struct controller to be its private data */
        --> pciehp_request_irq(ctrl)
            /* hardware set here ?? enable some bits in slot control reg,
             *
             * here, in pcie_write_cmd function, we need wait former cmd finished,
             * if needed, we should check the details of this wait function.
             */
        --> pcie_enable_notification(ctrl);
        /* if there is a device on the slot */
    --> pciehp_enable_slot(slot);
            /* after check the status of device, if needed, will enable the device
             *
             * As we already implements linkup in UEFI, if we enable PCIe controller
             * driver, it will not call board_added. If we do not enable PCIe controller,
             * it will call board_added here.
             */
        --> board_added(struct slot *p_slot)
```

```
pcie_isr
        /* to check: where to define the handle of below queue */
    --> wake_up(&ctrl->queue);
        /* create related work_struct, then put work_struct to the work_queue in struct slot,
         * Here, we may have: Attention Button Pressed, Presence Detect Changed
         * Power Fault Detected, Link up/down check.
         *
         * same handle will be added to work_struct: interrupt_event_handler, 
         * in above function, it handle all cases.
         *
         * We can not analyze all cases, the key point to understand the standard
         * hot-insertion and hot-removal is to consider the whole flow together
         * with INIT_DELAYED_WORK(&slot->work, pciehp_queue_pushbutton_work) in
         * pcie_init mentioned above.
         *
         * when you analyze the different cases, please pay great attention on
         * struct slot -> state, which indicates the current status of hot-plug
         * state machine. And we should get clear the map between these state
         * and real physical state.
         *
         * STATIC_STATE: at this state, pcie ep already runs fine or ep already
         * had been remove the system.
         *
         * BLINKINGON_STATE: in hot-insertion, after step1, card insertion; step2,
         * press button, power indicator led will blinking 5s. This state just 
         * indicates this.
         *
         * BLINKINGOFF_STATE: same as above, but in hot-removal.
         *
         * POWERON_STATE: in hot-insertion, in the 5s of power indicator led blinking,
         * if we do not press button again, it will try to bring the ep power on,
         * linkup, then add ep to system. This state just shows this.
         *
         * POWEROFF_STATE: same as above, but in hot-removal.
         */
    --> pciehp_queue_interrupt_event(slot, INT_BUTTON_PRESS);
```

If we only see the flow of "interrupt_event_handler: INT_PRESENCE_ON", it can not
match the stardard hot-inertion flow, which firstly inserting a PCIe card and then
pressing the attention button.

If in a system, when a pcie card been plugged in, a hotplug interrupt will be
triggered and presence detect change bit set 1 in slot status register.
This will trigger the operation in flow 1 below. you can see that now the whole
hot-insertion process has no relationship with attention button press.

pcie device hot insertion, trigger Presence Detect interrupt:
```
       --> interrupt_event_handler: INT_PRESENCE_ON
           --> handle_surprise_event
                   /* if presence detect bit in slot status set 1 */
               --> pciehp_queue_power_work(..., ENABLE_REQ)
                       /* will create work_queue, and add it to struct slot's wq,
                        * the handler function is pciehp_power_thread
                        *
                        * above handler will be called by work queue in slot.
                        */
                   --> pciehp_power_thread: case ENABLE_REQ
                       --> pciehp_enable_slot(p_slot)
                               /* before doing this, we will check slot status */
                           --> board_added
                               --> pciehp_green_led_blink
                                       /* send green led blink */
                                   --> pcie_write_cmd_nowait
                                   /* check linkup, here maximum wait 1s */
                               --> pciehp_check_link_status
                                   /* to check resource assign if has problem */
                               --> pciehp_configure_device
                               --> pciehp_green_led_on
                       --> p_slot->state = STATIC_STATE;
```

Let's see standard pcie hot-removal:
button pressed, trigger Attention Button Pressed interrupt
```
       --> interrupt_event_handler: INT_BUTTON_PRESS
               /* In this function, it depends on hot-plug state machine to decide
                * what to do next. Here assuming, power on, we operate hot-removal.
                */
           --> handle_button_press_event(p_slot)
                   /* if pcie card is running fine, we are in STATIC_STATE,
                    * and at this time, we check power, it is power on, so we know
                    * this operation will be a hot-removal, print:
                    * "powering off due to button press"
                    *
                    * Then we change state to BLINKINGOFF_STATE, which means
                    * PCIe controller/card will wait 5s to turn off power.
                    */
                   /* blink power indicator to show power unstable */
               --> pciehp_green_led_blink(p_slot)
                   /* turn off attention indicator to show everything fine */
               --> pciehp_set_attention_status(p_slot, 0)
                   /* delay work queue 5s to wait if we will cancel this operation */
               --> queue_delayed_work(p_slot->wq, &p_slot->work, 5*HZ)
```
As [1] page 394, step 4, it said, hotplug driver should let the driver of PCIe card
stop, and card driver should handle stop the data/interrupt. We should check
pci_stop_and_remove_bus_device below to get the details.

If in 5s, we do not press button, it will eventually call pciehp_queue_pushbutton_work
which initialized in pcie_init.
```
pciehp_queue_pushbutton_work
        /* when hot-removal, state is BLINKINGOFF_STATE */
    --> pciehp_queue_power_work(p_slot, DISABLE_REQ)
            /* POWEROFF_STATE */
        --> pciehp_power_thread
            --> pciehp_disable_slot
                --> remove_board(p_slot)
                    --> pciehp_unconfigure_device(p_slot)
                            /* the point here is how to removal the ep specific
                             * work. If needed, we should check the details here.
                             */
                        --> pci_stop_and_remove_bus_device
```
above is the code map of standard hot-removal.

If at any time, the pcie card has been surprise hot-removed, linux kernel can
handle it by:
```
       --> interrupt_event_handler: INT_PRESENCE_OFF
           --> handle_surprise_event
               --> pciehp_queue_power_work(p_slot, DISABLE_REQ)
                       /* add this to slot's work queue */
                   --> pciehp_power_threa
                       --> pciehp_disable_slot(p_slot)
                           --> remove_board(p_slot)
                       --> p_slot->state = STATIC_STATE;
```
but the point here is how to handle ep specific work, need more investigation here.

Let's see standard pcie hot-insertion:
If we consider pciehp_queue_pushbutton_work in the flow of standard hot-insertion,
it is easy to get the work flow analysis like standard hot-removal above.

The point here is that only if the power of slot is off before inserting ep, can
we use the button to trigger standard hot-insertion flow. If power of slot is on,
once a ep has been inserted in slot, it will trigger presence ditect change interrupt,
then code in kernel can handle following part of hot-insertion, we indeed do not
need to press a button. We just show this in above section.

Resource assignment problem in linux hotplug:
kernel use below code to add a ep to system.
```
       --> pciehp_enable_slot
           --> board_added(p_slot)
               --> pciehp_configure_device
                   ...
                   --> pci_hp_add_bridge(dev)
```
But how does linux assign bus number ahead for a hotplug capable slot ?
we should pay attention to a value: max in pci enumeration code.
```
       --> pci_scan_child_bus
	   --> if (bus->self && bus->self->is_hotplug_bridge && pci_hotplug_bus_size)
		if (max - bus->busn_res.start < pci_hotplug_bus_size - 1)
			max = bus->busn_res.start + pci_hotplug_bus_size - 1;
```
It will check if this is a hotplug bridge and assign more pci buses for this bridge
according to pci_hotplug_bus_size which can be set in command line of kernel.


note
----

struct slot -> state:
```
(to finish)
                         +-------------------+
                         | BLINKINGOFF_STATE |
                         +-------------------+




+--------------+ slot enable +------------+  slot disable +--------------+
|POWERON_STATE | ----------> |STATIC_STATE|  <----------- |POWEROFF_STATE|
+--------------+             +------------+               +--------------+




                          +----------------+
                          |BLINKINGON_STATE|
                          +----------------+
```

from PCIe spec, power indicator should be a green led:
off means power off, so we can insertion/removal
on means power on, so we can not insertion/removal
blinking means, it is powering up/down or a feedback from attention button pressed
                or hot-plug operation is initiated through software.

attention indicator should be a yellow or amber led:
on means, should be pay attention to
off means, everything fine


Reference
---------

1. PCI.EXPRESS系统体系结构标准教材].(美)Pavi.Budruk,Don.Anderson,Tom.Shanley.扫描版.pdf

