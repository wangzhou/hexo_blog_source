---
title: GIC ITS学习笔记(一)
tags:
  - GIC
description: N年前分析ARM GIC代码的一个笔记
abbrlink: f2188acc
date: 2021-07-11 23:43:41
categories:
---

current GIC ITS code in v4.2/v4.3
------------------------------------
```
/* this is the arch of interrupt sub-system */
struct irq_desc
    --> struct irq_data
        --> struct irq_chip
        --> struct irq_domain
            --> struct irq_domain_ops

/* this is the arch of ITS sub-system, for v4.1, for v4.2, kernel changed a lot */
struct its_node
    --> struct irq_domain
        --> struct irq_domain_ops (its_domain_ops)
	        /* alloc put struct irq_chip (its_irq_chip) to related
		 * struct irq_data
		 *
		 * Fix me for other work alloc does
		 */
	    --> .alloc (its_irq_domain_alloc)
	    --> .activate (its_irq_domain_activate)
	    --> ...
    --> struct msi_controller
            /* father irq_domain is irq_domain above */
        --> struct irq_domain
	        /* msi_domain_ops defined in kernel/irq/msi.c */
	    --> struct irq_domain_ops (struct irq_domain_ops msi_domain_ops)
	    --> void *host_data (struct msi_domain_info its_pci_msi_domain_info)
                --> struct msi_domain_ops (its_pci_msi_ops)
                    --> .msi_prepare (its_msi_prepare)
                    --> ...
                --> struct irq_chip (its_msi_irq_chip)
```
/* this is the arch of ITS sub-system for v4.2, ITS driver changed a lot in v4.2 */
there is no irq_domain in struct its_node. In drivers/irqchip/irq-gic-v3-its.c,
just build up below irq_domain.
```
		gic irq_domain --> irq_domain_ops(gic_irq_domain_ops)
		      ^                --> .alloc(gic_irq_domain_alloc)
		      |
		its irq_domain --> irq_domain_ops(its_domain_ops)
		      ^                --> .alloc(its_irq_domain_alloc)
		      |                --> ...
		      |        --> host_data(struct msi_domain_info)
		      |            --> msi_domain_ops(its_msi_domain_ops)
		      |                --> .msi_prepare(its_msi_prepare)
		      |            --> irq_chip, chip_data, handler...
		      |            --> void *data(struct its_node)
```
In drvers/irqchip/irq-gic-v3-its-pci-msi.c,
   drvers/irqchip/irq-gic-v3-its-platform-msi.c, it seems that we create two
other irq_domain:
```
pci_its irq_domain                      platform_its irq_domain
        /* kernel/irq/msi.c */                  /* kernel/irq/msi.c */
    --> irq_domain_ops(msi_domain_ops)      --> irq_domain_ops(msi_domain_op)
        /* irq-gic-v3-its-pci-msi.c             /* irq-gic-v3-its-platform-msi.c
	 * (its_pci_msi_domain_info)             * (its_pmsi_domain_info)
	 */                                      */
    --> void *host_data                     --> void *host_data
        --> .ops(its_pci_msi_ops)               --> .ops(its_pmsi_ops)
	        /* its_pci_msi_prepare */   	    /* its_pmsi_prepare */
	    --> .msi_prepare                    --> .msi_prepare
	--> .chip(its_msi_irq_chip)             --> .chip(its_pmsi_irq_chip)
```
msi domain struct
--------------------

basic struct:
```
struct msi_domain_info
    --> struct msi_domain_ops
        --> .msi_prepare
    --> struct irq_chip
        --> .irq_write_msi_msg
```
msi related struct should be part of interrupt sub-system as showed in part 1.
struct msi_domain_info will be stored in void *host_data of a irq_domain.
```
/* will find below funtion in drvers/irqchip/irq-gic-v3-its-pci-msi.c */
pci_msi_create_irq_domain(device_node, msi_domain_info, parent)
        /* core function */
    --> msi_create_irq_domain(node, info, parent);
            /* msi_domain_ops is irq_domain_ops in kernel/irq/msi.c
             * info below will be stored in host_data of irq_domain
             *
             * both pci_its irq_domain and platform_its irq_domain use
             * same msi_domain_ops, but different msi_domain_info.
             * above msi_domain_ops is a struct irq_domain_ops.
             */
        --> irq_domain_add_hierarchy(parent, 0, 0, node, &msi_domain_ops, info)
```
pci msi struct 
-----------------

in part 2, it creats related domain, then we will see how to use callbacks
in above domain. This part shows how to allocate irqs in a msi domain.
```
/* this is the work flow of PCI MSI */
/* kernel/drivers/pci/msi.c */
pci_msi_setup_msi_irqs(struct pci_dev *dev, int nvec, int type)
        /* so this irq_domain is pci_its irq_domain ? */
    --> pci_msi_domain_alloc_irqs(domain, dev, nvec, type);
        --> msi_domain_alloc_irqs(domain, &dev->dev, nvec);
	        /* should be its_pci_msi_prepare ?
                 * if below function, first get dev_id, then call parent domain
                 * msi_prepare which is its domain msi_prepare and will build
                 * up device table of ITS.
                 */
	    --> ps->msi_prepare(domain, dev, nvec, &arg);
	       /* domain, virq, desc->nvec_used, dev_to_node(dev), &arg, false */
	    --> __irq_domain_alloc_irqs()

/* details of how ITS work, this prepare function just allocat device table and
 * related structure in ITS
 */
/* in pci_msi irq_domain */
its_pci_msi_prepare
    --> ... (get dev_ip)
        /* in its irq_domain */
    --> its_msi_prepare
        --> its_create_device(its, dev_id, nvec);
```
```
/* there is an important point: how to configure MSI capability registers of one
 * PCI device, this action takes place in irq_write_msi_msg mentioned in part 2's
 * "basic struct". how to call this function ? obviouly, this function is stored
 * in msi_domain_info, so we must use some msi layer function to access this
 * function to interrup sub-system, in fact, it uses msi_domain_ops to do this.
 */
/* it stored irq_chip in msi_domain_info to interrupt sub-system in 
 * pci_msi_create_irq_domain, details as below:
 */
pci_msi_create_irq_domain
    --> msi_create_irq_domain
        -->  msi_domain_update_dom_ops
                 /* (struct msi_domain_ops msi_domain_ops_default -->
                  * msi_domain_ops_init
                  */
             --> ops->msi_init = msi_domain_ops_default.msi_init;
                     /* para: domain, virq, hwirq, info->chip, info->chip_data */
                 --> irq_domain_set_hwirq_and_chip
```
in function request_irq, it will call .active in irq_domain struct, here is will
call function msi_domain_activate in struct irq_domain_ops msi_domain_ops.
```
request_irq
    ...
    --> msi_domain_activate
        --> irq_chip_write_msi_msg
                /* here is irq_write_msi_msg in struct msi_domain_info */
            --> data->chip->irq_write_msi_msg(data, msg);
```
