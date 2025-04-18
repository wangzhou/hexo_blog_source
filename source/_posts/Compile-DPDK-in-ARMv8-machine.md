---
title: Compile DPDK in ARMv8 machine
tags:
  - ARM64
  - DPDK
description: 'This doc is a quirk note about the process of DPDK compile in ARMv8 machine. '
abbrlink: 87441d3f
date: 2021-06-29 00:00:15
categories:
---

Hardware: D05 board.
System: Linux 157 4.11.0-45.el7.aarch64 aarch64 GNU/Linux
		CentOS Linux release 7.4.1708 (AltArch) 
	
0. download dpdk codes
	git clone git://dpdk.org/dpdk

1. install kernel header files
	yum install kernel-headers.aarch64

2. install libpcap
	yum install libpcap-devel.aarch64

3. install numa header files
	yum install numactl-devel.aarch64

4. config and make
	make config T=arm64-armv8a-linuxapp-gcc
	make

	Note: above will put compiled files to dpdk/build

	"make install T=arm64-armv8a-linuxapp-gcc" will create a directory
	named as arm64-armv8a-linuxapp-gcc under dpdk and put compiled files
	in it.

5. build the dpdk helloworld example:

	export RTE_SDK=~/repos/dpdk
	export RTE_TARGET=arm64-armv8a-linuxapp-gcc
	cd examples/helloworld/
	make
```
	note: should use "make install T=arm64-armv8a-linuxapp-gcc" in step4
	      above to avoid compile error in step5.
```

6. test helloworld in above build:

	(1) enable hugepage
```
		echo 2 > /sys/kernel/mm/hugepages/hugepages-524288kB/nr_hugepages
		grep -i huge /proc/meminfo
		AnonHugePages:         0 kB
		ShmemHugePages:        0 kB
		HugePages_Total:       2
		HugePages_Free:        1
		HugePages_Rsvd:        0
		HugePages_Surp:        0
		Hugepagesize:     524288 kB
```
	(2) run ./helloworld -c 3 -n 2 in examples/helloworld
```
		EAL: Detected 64 lcore(s)
		EAL: Detected 4 NUMA nodes
		EAL: Multi-process socket /var/run/dpdk/rte/mp_socket
		EAL: No free hugepages reported in hugepages-524288kB
		EAL: No free hugepages reported in hugepages-524288kB
		EAL: 10 hugepages of size 2097152 reserved, but no mounted hugetlbfs found for that size
		EAL: Probing VFIO support...
		EAL: PCI device 0002:e9:00.0 on NUMA socket -1
		EAL:   Invalid NUMA socket, default to 0
		EAL:   probe driver: 8086:10fb net_ixgbe
		EAL: PCI device 0002:e9:00.1 on NUMA socket -1
		EAL:   Invalid NUMA socket, default to 0
		EAL:   probe driver: 8086:10fb net_ixgbe
		hello from core 1
		hello from core 0
```
