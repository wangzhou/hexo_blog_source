---
title: how to test RoCE driver
tags:
  - RDMA
description: >-
  This document shares how to do a sanity test for a RoCE driver. If you are a
  new guy for RoCE, it is for you.
abbrlink: 79c3f639
date: 2021-05-22 12:43:48
---
how to test RoCE driver
-----------------------

-v0.1 2018..27 Sherlock init

This document shares how to do a sanity test for a RoCE driver. If you are a
new guy for RoCE, it is for you.

Here is what I had done to test RoCE driver:

1. I had a ARM64 based machine installed a Redhat system in it.

2. download perftest, this is a open source RoCE test cases:
	
	git clone https://github.com/lsgunth/perftest.git

3. download RoCE's user space library, rdma-core:

	git clone https://github.com/linux-rdma/rdma-core.git

4. download kernel which includes the RoCE driver to test.

5. build perftest tools followed by the README.

   Here in a Redhat(CentOS) system, when you did ./configure, it maybe show
   that you lack ibverbs head files. you can install rdma-core-devel to solve it:

	yum install rdma-core-devel.aarch64

   if it is successful, you will get ib_send_bw .... tools in perftest.

6. build rdma-core followed by its README. In a Redhat(CentOS) system, maybe you
   need:

	yum install libnl-devel.aarch64

   if it is successful, you will get user space library under rdma-core/build/lib

7. Copy the libraries under rdma-core/build/lib/* to /lib64 in your system.
   (I also did the test in ubuntu, you should copy the libraries to /lib)

8. build your kernel together with your RoCE drivers.
   In my case, RoCE driver will be built into some kernel modules.

   install modules: make modules_install
   install kernel image: make install
   (before build kernel, you can add a localversion file to help to tell your kernel)

9. reboot system, then boot up using your built kernel.

10. in my case, RoCE kernel can not be loaded automatically, so I need to do:

	modprobe hns-roce-hw-v2

11. then you can find your roce device in /sys/class/infiniband. Or use the tools
    in rdma-core/build/bin: ibv_devices, ibv_devinfo to see your RoCE devices.

12. up your RoCE's networking interface, and keep its link status as: yes
    you can find RoCE's networking interface by searching, e.g.:

    /sys/class/infiniband/hns_0/ports/1/gid_attrs/ndevs

13. in a redhat system, you need do:

	echo "driver hns" > /etc/libibverbs.d/hns.driver

    but in a ubuntu system, you need do:

	echo "driver hns" > /usr/local/etc/libibverbs.d/hns.driver

14. finally you can use ib_send_bw, ib_read_bw, ib_write_bw... to do the sanity
    test. Every time you do the test, you should create a RoCE server and create
    a RoCE client to access and send date to the server.

    you can do as:
```
[root@localhost perftest]# ./ib_write_bw -n 5 -d hns_0 &
[1] 10352
[root@localhost perftest]# 
************************************
* Waiting for client to connect... *
************************************

[root@localhost perftest]# ./ib_write_bw -n 5 -d hns_0 192.168.2.198
---------------------------------------------------------------------------------------
                    RDMA_Write BW Test
---------------------------------------------------------------------------------------
 Dual-port       : OFF		Device         : hns_0
                    RDMA_Write BW Test
 Number of qps   : 1		Transport type : IB
 Dual-port       : OFF		Device         : hns_0
 Connection type : RC		Using SRQ      : OFF
 Number of qps   : 1		Transport type : IB
 CQ Moderation   : 5
 Connection type : RC		Using SRQ      : OFF
 Mtu             : 1024[B]
 TX depth        : 5
 Link type       : Ethernet
 CQ Moderation   : 5
 Gid index       : 0
 Mtu             : 1024[B]
 Max inline data : 0[B]
 Link type       : Ethernet
 rdma_cm QPs	 : OFF
 Gid index       : 0
 Data ex. method : Ethernet
 Max inline data : 0[B]
---------------------------------------------------------------------------------------
 rdma_cm QPs	 : OFF
 Data ex. method : Ethernet
---------------------------------------------------------------------------------------
hr_qp->port_num= 0x1
hr_qp->port_num= 0x1
 local address: LID 0000 QPN 0x0022 PSN 0x16566e RKey 0x000300 VAddr 0x00ffff93c9b000
 local address: LID 0000 QPN 0x0023 PSN 0x1fd9e RKey 0x000400 VAddr 0x00ffffbe24d000
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:02:198
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:02:198
 remote address: LID 0000 QPN 0x0023 PSN 0x1fd9e RKey 0x000400 VAddr 0x00ffffbe24d000
 remote address: LID 0000 QPN 0x0022 PSN 0x16566e RKey 0x000300 VAddr 0x00ffff93c9b000
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:02:198
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:02:198
---------------------------------------------------------------------------------------
---------------------------------------------------------------------------------------
 #bytes     #iterations    BW peak[MB/sec]    BW average[MB/sec]   MsgRate[Mpps]
 #bytes     #iterations    BW peak[MB/sec]    BW average[MB/sec]   MsgRate[Mpps]
 65536      5                8169.77            8161.23		   0.130580
---------------------------------------------------------------------------------------
 65536      5                8169.77            8161.23		   0.130580
---------------------------------------------------------------------------------------
[1]+  Done                    ./ib_write_bw -n 5 -d hns_0


[root@localhost perftest]# ./ib_read_bw -n 5 -d hns_0 &
[1] 10339
[root@localhost perftest]# ---------------------------------------------------------------------------------------
Device not recognized to implement inline feature. Disabling it

************************************
* Waiting for client to connect... *
************************************

[root@localhost perftest]# ./ib_read_bw -n 5 -d hns_0 192.168.2.198
---------------------------------------------------------------------------------------
Device not recognized to implement inline feature. Disabling it
---------------------------------------------------------------------------------------
---------------------------------------------------------------------------------------
                    RDMA_Read BW Test
                    RDMA_Read BW Test
 Dual-port       : OFF		Device         : hns_0
 Dual-port       : OFF		Device         : hns_0
 Number of qps   : 1		Transport type : IB
 Number of qps   : 1		Transport type : IB
 Connection type : RC		Using SRQ      : OFF
 Connection type : RC		Using SRQ      : OFF
 CQ Moderation   : 5
 TX depth        : 5
 Mtu             : 1024[B]
 CQ Moderation   : 5
 Link type       : Ethernet
 Mtu             : 1024[B]
 Gid index       : 0
 Link type       : Ethernet
 Outstand reads  : 128
 Gid index       : 0
 rdma_cm QPs	 : OFF
 Outstand reads  : 128
 Data ex. method : Ethernet
 rdma_cm QPs	 : OFF
---------------------------------------------------------------------------------------
 Data ex. method : Ethernet
---------------------------------------------------------------------------------------
hr_qp->port_num= 0x1
hr_qp->port_num= 0x1
 local address: LID 0000 QPN 0x0020 PSN 0xe6d890 OUT 0x80 RKey 0x000300 VAddr 0x00ffffb0250000
 local address: LID 0000 QPN 0x0021 PSN 0x87bda4 OUT 0x80 RKey 0x000400 VAddr 0x00ffff9dbe8000
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:02:198
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:02:198
 remote address: LID 0000 QPN 0x0020 PSN 0xe6d890 OUT 0x80 RKey 0x000300 VAddr 0x00ffffb0250000
 remote address: LID 0000 QPN 0x0021 PSN 0x87bda4 OUT 0x80 RKey 0x000400 VAddr 0x00ffff9dbe8000
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:02:198
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:02:198
---------------------------------------------------------------------------------------
---------------------------------------------------------------------------------------
 #bytes     #iterations    BW peak[MB/sec]    BW average[MB/sec]   MsgRate[Mpps]
 #bytes     #iterations    BW peak[MB/sec]    BW average[MB/sec]   MsgRate[Mpps]
 65536      5                8002.35            8002.35		   0.128038
---------------------------------------------------------------------------------------
 65536      5                8002.35            8002.35		   0.128038
---------------------------------------------------------------------------------------
[1]+  Done                    ./ib_read_bw -n 5 -d hns_0

[root@localhost perftest]# ./ib_send_bw -n 5 -d hns_0 &
[1] 10300
[root@localhost perftest]# 
************************************
* Waiting for client to connect... *
************************************


[root@localhost perftest]# ./ib_send_bw -n 5 -d hns_0 192.168.2.198
---------------------------------------------------------------------------------------
---------------------------------------------------------------------------------------
                    Send BW Test
                    Send BW Test
 Dual-port       : OFF		Device         : hns_0
 Dual-port       : OFF		Device         : hns_0
 Number of qps   : 1		Transport type : IB
 Number of qps   : 1		Transport type : IB
 Connection type : RC		Using SRQ      : OFF
 Connection type : RC		Using SRQ      : OFF
 RX depth        : 6
 TX depth        : 5
 CQ Moderation   : 5
 CQ Moderation   : 5
 Mtu             : 1024[B]
 Mtu             : 1024[B]
 Link type       : Ethernet
 Link type       : Ethernet
 Gid index       : 0
 Gid index       : 0
 Max inline data : 0[B]
 Max inline data : 0[B]
 rdma_cm QPs	 : OFF
 rdma_cm QPs	 : OFF
 Data ex. method : Ethernet
 Data ex. method : Ethernet
---------------------------------------------------------------------------------------
---------------------------------------------------------------------------------------
hr_qp->port_num= 0x1
hr_qp->port_num= 0x1
 local address: LID 0000 QPN 0x001e PSN 0xab3cfa
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:02:198
 local address: LID 0000 QPN 0x001f PSN 0xdefae7
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:02:198
 remote address: LID 0000 QPN 0x001f PSN 0xdefae7
 remote address: LID 0000 QPN 0x001e PSN 0xab3cfa
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:02:198
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:02:198
---------------------------------------------------------------------------------------
---------------------------------------------------------------------------------------
 #bytes     #iterations    BW peak[MB/sec]    BW average[MB/sec]   MsgRate[Mpps]
 #bytes     #iterations    BW peak[MB/sec]    BW average[MB/sec]   MsgRate[Mpps]
 65536      5                8054.00            8051.92		   0.128831
---------------------------------------------------------------------------------------
 65536      5                0.00               15340.76		   0.245452
---------------------------------------------------------------------------------------
[1]+  Done                    ./ib_send_bw -n 5 -d hns_0
```
