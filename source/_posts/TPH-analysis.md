---
title: TPH analysis
tags:
  - PCIe
description: >-
  TPH is a PCIe feature which can be seen in PCIe3.0 spec. This document will
  analysis this feature, the goal here is to find how to support it from
  software (Linux kernel)
abbrlink: 569ba796
date: 2021-05-21 05:15:56
---
TPH analysis
------------

-v0.1 2017.6.1 Sherlock init

```      
                             +-----+
                             | CPU |
                             +--+--+
                                |
                           +----+-----+
                           | L3 cache |
                           +----+-----+
                                |                 +-----+
              ----------+-------+-----------------+ DDR |
                        |                         +-----+
                        |
                      +-+--+ <--- stash table
                      | RP |
                      +-+--+
                        |
                      +-+--+ <--- TH, PH, ST
                      | EP |
                      +----+
```

TPH is a feature by which we can controll if TLP from EP can read/write to DDR
or L3 cache directly. As the attribute of read/write flow is different, e.g.
some data will be read soon by CPU, after a DMA write to DDR/Cache, for this
kind of date, it is better to put them directly to cache.

Physically, RP will do the read/write to DDR/cach. TH, PH, ST in PCIe TLP
will be used to control above operation. TH bit(one bit) in TLP indicates if we
enable TPH feature. PH(2 bits) in TLP indicates which kind of flow it is. ST is
a hardware specific design, which can be stored in MSI-X table or TPH capability.

So the software interface about TPH is:

        PR: Device Capability 2(offset 0x24): TPH Completer Supported(12,13bit RO)
        EP: TPH Requester Capability: Header, capability register(RO), controler register(RW),
            ST table.(ST table can be in MSI-X table)
        EP's DMA descriptor: must have a place to offer PH related information.

In our system, we have above registers, ST table is in EP's MSI-X table. Further
more, we have a stash table in RP. We implement ST table with 8 bit ST entry in EP,
which indicates which core/cluster one flow is expected to go to. The stash table
is a self-defined table, which is used to translate PH/ST info to stash info.


So from software view, we can controll TPH from EP's DMA descriptor, ST table,
stash table. Now ST table and stash table have relationship, it is hard to modify
a PCIe device driver according to our chip's ST table definition :(

But currently there is no clear definition about ST in PCIe spec and ARM spec.

Above is the first consideration, which means we may need private patch to do
optimization in the future.

The second consideration is as stach table will be implemented in BIOS, which
is hard to modified, we only can use ST and DMA descriptor to controll TPH behavior.

Then third is that we should firstly configure cacheable attribute, then enable
TPH to put data to L3 cache directly, e.g. we should firstly enable CCA=1 for RP,
then enable TPH feature(e.g. we can enable SMMU and CCA=1 to enable cacheable attribute).

The last is integrated PCIe devices, e.g. networking, also have TPH like feature.
