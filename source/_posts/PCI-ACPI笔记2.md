---
title: PCI ACPI笔记2
tags:
  - UEFI
  - ACPI
  - PCIe
description: '本文以一个例子介绍和PCI相关的一些ACPI表格, 本文基于ARM64平台'
abbrlink: 10b921
date: 2021-07-11 23:39:26
categories:
---

Basic points about PCI ACPI
---------------------------

There are three APCI tables involved in: DSDT, MCFG and IORT

```
MCFG: should configure ECAM compliant configure space address(CPU address).
|
|    _CBA: MMCONFIG base address

DSDT: we can configure IO/MEM range and some SoC specific info in DSDT.
|
|    _CRS indicates: MEM/IO space, bus range
|
|    ...
|
|    PNP0A03 ?

IORT: build up the map among RID, stream ID and device ID.
|
|    ...

_PRT: PCI routing table for INTx
```

let's see a sample of above tables in [2]

```
// for details, refer to p.42 in [4]
EFI_ACPI_5_0_PCI_EXPRESS_MEMORY_MAPPED_CONFIGURATION_SPACE_TABLE Mcfg=
{
  {
      {
        EFI_ACPI_5_0_PCI_EXPRESS_MEMORY_MAPPED_CONFIGURATION_SPACE_BASE_ADDRESS_DESCRIPTION_TABLE_SIGNATURE,
        sizeof (EFI_ACPI_5_0_PCI_EXPRESS_MEMORY_MAPPED_CONFIGURATION_SPACE_TABLE),
        ACPI_5_0_MCFG_VERSION,
        0x00,                                             // Checksum will be updated at runtime
        {EFI_ACPI_ARM_OEM_ID},
        EFI_ACPI_ARM_OEM_TABLE_ID,
        EFI_ACPI_ARM_OEM_REVISION,
        EFI_ACPI_ARM_CREATOR_ID,
        EFI_ACPI_ARM_CREATOR_REVISION
      },
      0x0000000000000000,                                 //Reserved
  },
  {

    // just a sample, so only list one Segment
    {
      0xb0000000,                                         //Base Address
      0x0,                                                //Segment Group Number
      0x0,                                                //Start Bus Number
      0x1f,                                               //End Bus Number
      0x00000000,                                         //Reserved
    },
  }
};

so structure of MCFG table is:
{
    EFI_ACPI_5_0_MCFG_TABLE_CONFIG
    {
        EFI_ACPI_DESCRIPTION_HEADER
        UINT64 Reserved1
    }
    EFI_ACPI_5_0_MCFG_CONFIG_STRUCTURE[]
}
```

```
Scope(_SB)
{
  // PCIe Root bus, for the ASL grammar, please refer to 19 charpter in [5]
  Device (PCI0)                              // acpi_get_devices("HISI0080", ...) ?
  {
    // for the details of items below, please refer to charpter 6 in [5]
    Name (_HID, "HISI0080")                  // PCI Express Root Bridge
    Name (_CID, "PNP0A03")                   // Compatible PCI Root Bridge, Compatible ID
    Name(_SEG, 0)                            // Segment of this Root complex
    Name(_BBN, 0)                            // Base Bus Number
    Name(_CCA, 1)                            // cache coherence attribute ??
    Method (_CRS, 0, Serialized) {           // Root complex resources, _CRS: current resource setting
                                             // Method is defined in 19.6.82 in [5]
      Name (RBUF, ResourceTemplate () {      // Name: 19.6.87, ResourceTemplate: 19.6.111,
                                             // 19.3.3 in [5]
        WordBusNumber (                      // Bus numbers assigned to this root,
                                             // wordBusNumber: 19.6.144
          ResourceProducer,
          MinFixed,
          MaxFixed,
          PosDecode,
          0,                                 // AddressGranularity
          0x0,                               // AddressMinimum - Minimum Bus Number
          0x1f,                              // AddressMaximum - Maximum Bus Number
          0,                                 // AddressTranslation - Set to 0
          0x20                               // RangeLength - Number of Busses
        )
        QWordMemory (                        // 64-bit BAR Windows
          ResourceProducer,
          PosDecode,
          MinFixed,
          MaxFixed,
          Cacheable,
          ReadWrite,
          0x0,                               // Granularity
          0xb2000000,                        // Min Base Address pci address
          0xb7feffff,                        // Max Base Address
          0x0,                               // Translate
          0x5ff0000                          // Length
        )
        QWordIO (
          ResourceProducer,
          MinFixed,
          MaxFixed,
          PosDecode,
          EntireRange,
          0x0,
          0x0,
          0xffff,
          0xb7ff0000,
          0x10000
        )
      })                                      // Name(RBUF)
      Return (RBUF)
    }                                         // Method(_CRS), this method return RBUF!
  } // Device(PCI0)
}
```

```
// did not include smmu related description, for the detail, please refer to [6]
// IORT Head:
[0004]                          Signature : "IORT"    [IO Remapping Table]
[0004]                       Table Length : 0000029e
[0001]                           Revision : 00
[0001]                           Checksum : BC
[0006]                             Oem ID : "HISI "
[0008]                       Oem Table ID : "D03"
[0004]                       Oem Revision : 00000000
[0004]                    Asl Compiler ID : "INTL"
[0004]              Asl Compiler Revision : 20150410

[0004]                         Node Count : 00000003
[0004]                        Node Offset : 00000034
[0004]                           Reserved : 00000000
[0004]                   Optional Padding : 00 00 00 00

/* ITS 0, for dsa */
[0001]                               Type : 00
[0002]                             Length : 0018
[0001]                           Revision : 00
[0004]                           Reserved : 00000000
[0004]                      Mapping Count : 00000000
[0004]                     Mapping Offset : 00000000

[0004]                           ItsCount : 00000001
[0004]                        Identifiers : 00000000

/* mbi-gen dsa  mbi0 - usb, named component */
[0001]                               Type : 01
[0002]                             Length : 0046
[0001]                           Revision : 00
[0004]                           Reserved : 00000000
[0004]                      Mapping Count : 00000001
[0004]                     Mapping Offset : 00000032

[0004]                         Node Flags : 00000000
[0008]                  Memory Properties : [IORT Memory Access Properties]
[0004]                    Cache Coherency : 00000000
[0001]              Hints (decoded below) : 00
                                Transient : 0
                           Write Allocate : 0
                            Read Allocate : 0
                                 Override : 0
[0002]                           Reserved : 0000
[0001]       Memory Flags (decoded below) : 00
                                Coherency : 0
                         Device Attribute : 0
[0001]                  Memory Size Limit : 00
[0017]                        Device Name : "\_SB_.MBI0"
[0004]                            Padding : 00 00 00 00

[0004]                         Input base : 00000000
[0004]                           ID Count : 00000001
[0004]                        Output Base : 00040080  // device id
[0004]                   Output Reference : 00000034  // point to its dsa
[0004]              Flags (decoded below) : 00000000
                           Single Mapping : 1

/* RC 0 */
[0001]                               Type : 02
[0002]                             Length : 0034
[0001]                           Revision : 00
[0004]                           Reserved : 00000000
[0004]                      Mapping Count : 00000001
[0004]                     Mapping Offset : 00000020

[0008]                  Memory Properties : [IORT Memory Access Properties]
[0004]                    Cache Coherency : 00000001
[0001]              Hints (decoded below) : 00
                                Transient : 0
                           Write Allocate : 0
                            Read Allocate : 0
                                 Override : 0
[0002]                           Reserved : 0000
[0001]       Memory Flags (decoded below) : 00
                                Coherency : 0
                         Device Attribute : 0
[0004]                      ATS Attribute : 00000000
[0004]                 PCI Segment Number : 00000000

// Input, output means BDF of pcie host as the device ID in ITS
[0004]                         Input base : 00000000
[0004]                           ID Count : 00002000       // the number of IDs in range
[0004]                        Output Base : 00000000
// refer to the a node, here refer to ITS node in 0x34 offset
[0004]                   Output Reference : 00000034
[0004]              Flags (decoded below) : 00000000
                           Single Mapping : 0
```

Work flow of ACPI parse
-----------------------

based on kernel v4.8

```
/* from the view of ACPI, this function is the start */
acpi_init
    --> acpi_bus_init

        /* this is a week function */
    --> pci_mmcfg_late_init
            /* parse the MCFG table: drivers/acpi/pci_mcfg.c */
        --> acpi_table_parse(ACPI_SIG_MCFG, pci_mcfg_parse) 
                /* add mcfg_entry which contain info in mcfg to pci_mcfg_list,
                 * this will add all mcfg region to above list.
                 */
            --> pci_mcfg_parse

        /* this function is not in v4.8 now, it is used by PCIe controller to map
         * itself to correct ITS and SMMU. This will be modified in future maybe.
         *
         * Now, it parses iort table in iort_table_detect. other moduldes like ITS
         * will directly call the functions in iort_table_detect.
         */
    --> iort_table_detect 

        /* this function will scan all ACPI resource, including PCI */
    --> acpi_scan_init
	--> acpi_pci_root_init
	--> acpi_pci_link_init

	--> acpi_bus_scan
                /* each Device in DSDT will be a device below, which has a root
                 * bus. normally, we configure related address translate hardware
                 * unit in UEFI for this device(rp).
                 */
            --> acpi_bus_attach(device)

            ...     /* why we call acpi_pci_root_add, please refer to
                     * https://wangzhou.github.io/PCI-ACPI笔记1/
                     * scan PCI info in DSDT
                     */
	        --> acpi_pci_root_add
		    --> pci_acpi_scan_root
		        --> acpi_pci_root_create
			    --> acpi_pci_probe_root_resources
			            /* here parse _CRS info in DSDT */
			        --> acpi_dev_get_resources
```

reference
---------
- [RFC PATCH v3 0/3] Add ACPI support for HiSilicon PCIe Host Controllers
- https://github.com/open-estuary/uefi.git/OpenPlatformPkg/Chips/Hisilicon/Pv660/Pv660AcpiTables/
- https://wangzhou.github.io/PCI-ACPI笔记1/
- PCI Firmware Specification 3.0
- ACPI 6.0 spec
- IORT spec
