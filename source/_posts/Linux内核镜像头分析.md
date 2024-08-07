---
title: Linux内核镜像头分析
tags:
  - Linux内核
  - opensbi
  - uboot
description: 本文分析Linux内核镜像中Image头部格式的一些相关问题，使用的硬件构架是riscv， 分析中使用的内核版本使用v6.1
abbrlink: 36540
date: 2023-01-07 21:07:42
categories:
---

Linux内核镜像头部格式
----------------------

 Linux内核镜像的格式和启动它的bootloader有一定的关系。如果bootloader只是简单的跳到
 内核Image的第一条指令开始执行，那么内核Image直接从开始放指令就可以，如果bootloader
 对内核Image有一定的要求，一般是Image的头部符合一定的格式，那么内核Image就必须按照
 相关的格式排布，这意味着这样的bootloader可以解析内核Image的格式，内核的入口地址也
 不一定是内核Image的首地址。

 Linux和windows上程序的格式是不一样的，Linux是ELF格式、windows上是PE/COFF格式，这里
 说的是应用程序，我们看下Linux内核的Image。编译Linux内核的log可以看见，编译内核时
 先生成vmlinux，然后用objcopy生成Image，vmlinux是ELF格式，bootloader需要理解ELF
 才能引导Linux内核，所以还要把ELF转变成bootloader可以理解的二进制格式。

 如上，如果bootloader直接从第一条指令执行，直接把内核的入口地址放在Image的开始就
 可以，现在可以看到opensbi就是这样的简单bootloader。对于有些复杂的bootloader，比如
 uboot或者UEFI，它们支持的启动方式更多，有的启动方式对被启动的Image的格式就有约束。

 UEFI中使用的EFI启动方式，uboot也支持这种启动方式。这种启动方式一开始在UEFI/windows
 中使用，Image head要求是PE/COFF格式的，Linux内核为了支持EFI启动，Image head也要
 是PE/COFF格式。

 PE/COFF格式的head大概是这样的:
 ![PE/COFF head](PE_COFF_head.png)

 其中，开头两个字节的二进制编码必须是0x5A4D，对应的ascii字符就是‘MZ’，0x3C偏移处
 是真实PE头在Image中的偏移，PE/COFF的格式为了兼容DOS的格式，在真实PE头前加了如上
 的DOS head、DOS STUB。AddressOfEntryPoint处放Image的入口信息，内容是入口地址相对
 Image起始的偏移。

 下面具体看下riscv内核Image开始部分的代码。
```
  #ifdef CONFIG_EFI
          /*
           * This instruction decodes to "MZ" ASCII required by UEFI.
           */
          c.li s4,-13
          j _start_kernel
  #else
          /* jump to start kernel */
          j _start_kernel
          /* reserved */
          .word 0
  #endif
```
 编译好的riscv的Image要么支持EFI启动要么不支持。支持EFI启动的Image一开始放了一条
 指令，这条指令的二进制编码就是0x5A4D，后面在DOS head的空间直接放了一条跳转指令，
 这样EFI的内核Image也兼容了类似opensbi这种直接从Image第一条指令启动的bootloader，
 c.li s4,-13这样的指令不会有什么副作用，第二条指令就直接跳到_start_kernel了，如果
 bootloader按照EFI启动，会从PE头里的AddressOfEntryPoint域段得到入口地址，根本不会
 去执行j _start_kernel指令。

 对于EFI的Image，直接在如下的地方把这个PE头塞进来:
```
#ifdef CONFIG_EFI
	.word pe_head_start - _start
pe_head_start:

	__EFI_PE_HEADER
#else
	.word 0
#endif
```
 可以看到宏__EFI_PE_HEADER把PE头加了进来，AddressOfEntryPoint放的是__efistub_efi_pe_entry
 的偏移，所以EFI内核的代码执行路径和非EFI内核在这里还是不同的。

 riscv上的__efistub_efi_pe_entry是定义在drivers/firmware/efi/libstub/efi-stub-entry.c里
 的efi_pe_entry这个函数，在编译的时候给这个函数加上了__efistub_这个前缀：
```
/* drivers/firmware/efi/libstub/Makefile */
STUBCOPY_FLAGS-$(CONFIG_RISCV)  += --prefix-alloc-sections=.init \
                                   --prefix-symbols=__efistub_
```

opensbi启动内核
----------------

 如上，opensbi只是简单跳到被启动对象的起始地址。

uboot启动内核
--------------

 Linux内核已经支持EFI方式启动，当初的内核patch的链接是https://lwn.net/Articles/813589/
 这个patch是使用uboot + qemu来测试EFI启动的。

 用这个命令可以启动uboot:
```
qemu-system-riscv64 -m 256m -machine virt -nographic -kernel path_to_u-boot/u-boot
```
 启动的效果大概是：
```
OpenSBI v1.1
   ____                    _____ ____ _____
  / __ \                  / ____|  _ \_   _|
 | |  | |_ __   ___ _ __ | (___ | |_) || |
 | |  | | '_ \ / _ \ '_ \ \___ \|  _ < | |
 | |__| | |_) |  __/ | | |____) | |_) || |_
  \____/| .__/ \___|_| |_|_____/|____/_____|
        | |
        |_|

Platform Name             : riscv-virtio,qemu
Platform Features         : medeleg
Platform HART Count       : 1
Platform IPI Device       : aclint-mswi
Platform Timer Device     : aclint-mtimer @ 10000000Hz
Platform Console Device   : uart8250
Platform HSM Device       : ---
Platform Reboot Device    : sifive_test
Platform Shutdown Device  : sifive_test
Firmware Base             : 0x80000000
Firmware Size             : 288 KB
Runtime SBI Version       : 1.0

Domain0 Name              : root
Domain0 Boot HART         : 0
Domain0 HARTs             : 0*
Domain0 Region00          : 0x0000000002000000-0x000000000200ffff (I)
Domain0 Region01          : 0x0000000080000000-0x000000008007ffff ()
Domain0 Region02          : 0x0000000000000000-0xffffffffffffffff (R,W,X)
Domain0 Next Address      : 0x0000000080200000
Domain0 Next Arg1         : 0x000000008fe00000
Domain0 Next Mode         : S-mode
Domain0 SysReset          : yes

Boot HART ID              : 0
Boot HART Domain          : root
Boot HART Priv Version    : v1.12
Boot HART Base ISA        : rv64imafdch
Boot HART ISA Extensions  : time,sstc
Boot HART PMP Count       : 16
Boot HART PMP Granularity : 4
Boot HART PMP Address Bits: 54
Boot HART MHPM Count      : 16
Boot HART MIDELEG         : 0x0000000000001666
Boot HART MEDELEG         : 0x0000000000f0b509


U-Boot 2023.01-rc4-00059-g8d6cbf5e6b (Jan 05 2023 - 15:59:10 +0800)

CPU:   rv64imafdch_zicsr_zifencei_zihintpause_zba_zbb_zbc_zbs_sstc
Model: riscv-virtio,qemu
DRAM:  256 MiB
Core:  25 devices, 12 uclasses, devicetree: board
Flash: 32 MiB
Loading Environment from nowhere... OK
In:    serial@10000000
Out:   serial@10000000
Err:   serial@10000000
Net:   No ethernet found.
Working FDT set to 8f734950
Hit any key to stop autoboot:  0 
=> 
```
 (todo: 怎么继续启动内核)
