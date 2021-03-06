---
title: Linux syscon and regmap
tags:
  - Linux内核
description: 本文简单介绍Linux内核里regmap和syscon的东西，N年前的笔记了。记得 这些东西主要是处理MMIO有多次map的情况
abbrlink: b3dd80c4
date: 2021-07-17 10:03:10
categories:
---

What is regmap and syscon
----------------------------

regmap was introduced by https://lwn.net/Articles/451789/
From my understanding, it provided us a set API to read/write non memory-map I/O
(e.g. I2C and SPI read/write) at first. Then after introduced regmap-mmio.c,
we can use regmap to access memory-map I/O.

code path: drivers/base/regmap/*

syscon was introduced by https://lkml.org/lkml/2012/9/4/568
It provides a set API to access a misc device(e.g. pci-sas subsystem registers
in P660) based on regmap, explicitly based on regmap-mmio.c I guess.

code path: drivers/mfd/*

arch of regmap and syscon
----------------------------

basic structure of regmap:

struct regmap:                   per base address per regmap
struct regmap_bus:               include read/write callback, different "bus"
                                 (e.g. I2C, SPI, mmio) have different regmap_bus
struct regmap_mmio_context:      don't know...
struct regmap_config:            confiuration info.

regmap-mmio call flow:
```
/* drivers/base/regmap/regmap-mmio.c */
__devm_regmap_init_mmio_clk
    --> __devm_regmap_init
        /* regmap_bus(regmap_mmio), config as input, create regmap */
        --> __regmap_init
	    /* if don't have bus->read or bus->write */
	    --> map->reg_read = _regmap_bus_reg_read;
	    --> map->reg_write = _regmap_bus_reg_write;
	    ...
	    /* if have bus->read */
            --> map->reg_read  = _regmap_bus_read;
	        --> map->bus->read

/* drivers/base/regmap/regmap.c */
regmap_read(struct regmap *map, unsigned int reg, unsigned int *val)
    --> _regmap_read
        /* _regmap_bus_reg_read */
        --> map->reg_read(context, reg, val);
	    --> map->bus->reg_read(map->bus_context, reg, val)

/* drivers/base/regmap/regmap.c */
regmap_update_bits
    --> _regmap_update_bits
        --> _regmap_read
	--> _regmap_write
	    --> map->reg_write
```
basic structure of syscon:
```
struct syscon:                 include a strutct regmap; an element in list below
static LIST_HEAD(syscon_list)

syscon driver init a regmap:
syscon->regmap = devm_regmap_init_mmio(dev, base, &syscon_regmap_config);
```
why we need a syscon to describe a misc device
-------------------------------------------------

To understand this, we shoudl search related discussion in community:
https://lists.ozlabs.org/pipermail/devicetree-discuss/2012-August/018704.html

From my understanding, syscon firstly registers a syscon dts node to syscon_list,
we could find this node when we try to access related registers.

how to use syscon to access a misc device
--------------------------------------------

e.g.
```
need dts node:
	pcie_sas: pcie_sas@0xb0000000 {
		compatible = "hisilicon,pcie-sas-subctrl", "syscon";
		reg = <0xb0000000 0x10000>;
	};
```
use below function read/write:
```
	regmap_read(hisi_pcie->subctrl, PCIE_SUBCTRL_SYS_STATE4_REG +
		    0x100 * hisi_pcie->port_id, &val);

	regmap_update_bits(pcie->subctrl, reg, bit_mask, mode << bit_shift);
```
use below function create struct regmap:
```
	hisi_pcie->subctrl =
		syscon_regmap_lookup_by_compatible("hisilicon,pcie-sas-subctrl");
```
reference:
--------------
1. Documentation/devicetree/bindings/regmap/regmap.txt
2. ../mfd/mfd.txt
3. ./syscon.txt
