---
title: Parsing ranges item in pcie-designware.c
tags:
  - PCIe
description: >-
  This document tries to analysis codes about parsing ranges item in PCIe dts
  node. kernel version: v4.0.
abbrlink: da21670b
date: 2021-07-11 23:45:47
categories:
---

```
int __init dw_pcie_host_init(struct pcie_port *pp)
{
	...
	/*
	 * read out value of "#address-cells" as na and #size-cells as ns.
	 * as you can see in dw pcie binding document, address-cells should
	 * be set as 3 and size-cells should be set as 2 which are we already
	 * done in pcie dts node. so here na = 3, np = 2.
	 *
	 * in fact, address-cells indicates how many bytes do we use to describe
	 * a pci address: of cause thit is 3, one byte is bits flage which
	 * indicates if it is a mem/io/cfg/pref, other two bytes are for a 64
	 * bits address(pci address can be a 64 bits address). 
	 * size-cells indicate how many bytes do we use to describe a size of
	 * above pci address, of cause it should be at least 2 to describe a
	 * 64 bits address.
	 */
	of_property_read_u32(np, "#address-cells", &na);
	ns = of_n_size_cells(np);

	cfg_res = platform_get_resource_byname(pdev, IORESOURCE_MEM, "config");
	if (cfg_res) {
		pp->cfg0_size = resource_size(cfg_res)/2;
		pp->cfg1_size = resource_size(cfg_res)/2;
		pp->cfg0_base = cfg_res->start;
		pp->cfg1_base = cfg_res->start + pp->cfg0_size;

		/* Find the untranslated configuration space address */
		index = of_property_match_string(np, "reg-names", "config");
		addrp = of_get_address(np, index, NULL, NULL);
		pp->cfg0_mod_base = of_read_number(addrp, ns);
		pp->cfg1_mod_base = pp->cfg0_mod_base + pp->cfg0_size;
	} else {
		dev_err(pp->dev, "missing *config* reg space\n");
	}

	if (of_pci_range_parser_init(&parser, np)) {
		/* na indicates pci address address cells, ns is related size */
		--> const int na = 3, ns = 2;
		/*
		 * so parser->pna is cpu address cell number which can be found
		 * in parent dts node. take hip05.dtsi as an example, it should
		 * be 2, as:
		 *   	peripherals {
		 *		compatible = "simple-bus";
		 *		#address-cells = <2>;
		 *		#size-cells = <2>;
		 *		...
		 * note: pcie dts nodes should under peripherals node.
		 * above size-cells and address-cells should be both 2 for
		 * 64bits cpu address. all are wrong in hulk-3.19/hulk-4.1
		 * hip05.dtsi.
		 *
		 * for a 32bits Soc, above address-cells and size-cells both
		 * are 1, so parser->pna is 1
		 */
		--> parser->pna = of_n_addr_cells(node);
			--> if (np->parent)
				np = np->parent;
			--> ip = of_get_property(np, "#address-cells", NULL);
		/*
		 * ranges =
		 * <0x82000000 0 0xb5100000 0x240 0x00000000 0 0x00f00000>;
		 * |----> pci address <----|--> cpu addr <--|--> size <--|
		 */
		--> parser->np = parser->pna + na + ns;
		/*
		 * ranges = <0x00000800 0 0x01f00000 0x01f00000 0 0x00080000
 		 * 	     0x81000000 0 0          0x01f80000 0 0x00010000
 		 *	     0x82000000 0 0x01000000 0x01000000 0 0x00f00000>;
		 *
		 * like above ranges, parser->range will point to 0x00000800,
		 * parser->end will point to 0x00f00000.
		 */
		--> parser->range = of_get_property(node, "ranges", &rlen);
		--> parser->end = parser->range + rlen / sizeof(__be32);

		dev_err(pp->dev, "missing ranges property\n");
		return -EINVAL;
	}

	/* Get the I/O and memory ranges from DT */
	for_each_of_pci_range(&parser, &range) {
	  --> of_pci_range_parser_one(parser, range)
	    /* first byte in ranges in each line */
	    --> range->pci_space = parser->range[0];
	    /*
	     * from below function, we can see only mem/io/32bits/64bits/
	     * prefetch bits are valid. so even we set other bits in dts,
	     * code does not parse them.
	     *
	     * for each bit's meaning, you can refer to:
	     * http://www.devicetree.org/Device_Tree_Usage
	     */
	    --> range->flags = of_bus_pci_get_flags(parser->range);
	    /* point to pci address */
	    --> range->pci_addr = of_read_number(parser->range + 1, ns);
	    /* point to cpu address, not very clear below function */
	    --> range->cpu_addr = of_translate_address(parser->node,
				  parser->range + na);
	    /* point to size section */
	    --> range->size = of_read_number(parser->range + parser->pna + na, ns);
	    /*
	     * after below, parser->range points to 0x81000000 for example,
	     * next time, we start to parse the second line of ranges.
	     */
	    --> parser->range += parser->np;
	    /*
	     * in below loop, it find the ranges sub-item which has same type 
	     * and are contiguous.
	     */
	    --> while (parser->range + parser->np <= parser->end) {
			...

		}
		...
		if (restype == IORESOURCE_MEM) {
			of_pci_range_to_resource(&range, np, &pp->mem);
			pp->mem.name = "MEM";
			pp->mem_size = resource_size(&pp->mem);
			pp->mem_bus_addr = range.pci_addr;

			/*
			 * take mem as example, ranges item as below:
			 * ranges =
			 * <0x82000000 0 0xb5100000 0x240 0x00000000 0 0x00f00000>;
			 * now parser.range points to 0x00f00000,
			 * parser.rang - parser.np shifts points to 0x82000000.
			 * "+ na" will add pci address offset, so now point to
			 * start of cpu address: 0x240
			 *
			 * the problem of v3 patch: of_n_addr_cells(np) - 5 + na
			 * is ok, but there went wrong in parser_range_end.
			 * if there is only one line in ranges item which is
			 * my test case, it is ok. but if there are more than
			 * one line in ranges item, parser_range_end points to
			 * the end of ranges item which should points to each
			 * line end.
			 *
			 * pp->mem_mod_base = of_read_number(parser_range_end -
			 *		of_n_addr_cells(np) - 5 + na, ns);
			 *
			 * there is one problem: mem_mod_base is same with
			 * mem_base, why do we use mem_base directly??
			 */
			pp->mem_mod_base = of_read_number(parser.range -
							  parser.np + na, ns);

		}
		if (restype == 0) {
			of_pci_range_to_resource(&range, np, &pp->cfg);
			pp->cfg0_size = resource_size(&pp->cfg)/2;
			pp->cfg1_size = resource_size(&pp->cfg)/2;
			pp->cfg0_base = pp->cfg.start;
			pp->cfg1_base = pp->cfg.start + pp->cfg0_size;

			/* Find the untranslated configuration space address */
			pp->cfg0_mod_base = of_read_number(parser.range -
							   parser.np + na, ns);
			pp->cfg1_mod_base = pp->cfg0_mod_base +
					    pp->cfg0_size;
		}
	}
	...
}
```
