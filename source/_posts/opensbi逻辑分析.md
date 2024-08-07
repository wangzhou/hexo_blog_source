---
title: opensbi逻辑分析
tags:
  - opensbi
  - riscv
description: 本文分析opensbi的基本逻辑，分析基于opensbi 1.1版本。
abbrlink: 21819
date: 2022-10-21 14:41:22
categories:
---

基本逻辑
--------

 riscv构架定义了一套SBI标准(supervisor binary interface)，顾名思义，就是特权态软件
 和运行机器的一套二进制接口，这个标准其实是把机器行为抽象了，特权态软件通过这套
 二进制标准向底层机器请求服务，这个特权态软件包括host上的内核，也包括跑在guest上
 的内核。

 riscv SBI标准的链接是：https://github.com/riscv-non-isa/riscv-sbi-doc

 opensbi是这个标准的一个实现。opensbi的代码组织上，把sbi公共代码抽象出来形成两个
 库：libsbi, libsbiutils，不同平台调用这两个库的代码，和平台相关代码构建出具体平台
 上的bios二进制。

 对于qemu的virt平台，相关的代码在：opensbi/platform/generic, 编译生成的二进制在
 opensbi/build/platform/generic/firmware/。

 编译会生成三种类型的固件，分别是：fw_jump.elf, fw_load.elf, fw_dynamic.elf。
 这三种固件的区别是：fw_jump.elf跳到指定地址运行下一个阶段的代码，fw_load.elf会
 把下一个阶段的二进制，通常是Linux内核，一起打包到fw_load.elf固件里，fw_dynamic.elf
 定义了上一个程序以及下一个程序和fw_dynamic.elf的接口，如下的代码分析可以清楚的
 看到这一点。

编译运行
--------

 我们可以这样编译出qemu virt平台的opensbi二进制:
```
make all PLATFORM=generic CROSS_COMPILE=riscv64-linux-gnu- PLATFORM_RISCV_XLEN=64
```
 
 如果系统上已经有riscv的qemu，我们可以这样运行下编译好的固件：
```
~/repos/qemu/build/qemu-system-riscv64 -M virt -m 256M -nographic -bios /home/sherlock/repos/opensbi/build/platform/generic/firmware/fw_payload.elf
```

 大概的运行效果大概是：
```
OpenSBI v1.1-2-gcaa5eea
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
Firmware Size             : 284 KB
Runtime SBI Version       : 1.0

Domain0 Name              : root
Domain0 Boot HART         : 0
Domain0 HARTs             : 0*
Domain0 Region00          : 0x0000000002000000-0x000000000200ffff (I)
Domain0 Region01          : 0x0000000080000000-0x000000008007ffff ()
Domain0 Region02          : 0x0000000000000000-0xffffffffffffffff (R,W,X)
Domain0 Next Address      : 0x0000000080200000
Domain0 Next Arg1         : 0x0000000082200000
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

Test payload running
QEMU: Terminated
```

代码分析
--------

 我们把代码分析直接以注释的形式写到代码里，固件的代码是从fw_base.S开始执行的，我们
 也从这个文件开始注释，需要其他文件的时候，我们直接嵌入其他文件的注释，我们加的
 注释以“n:”开头。
```
/*
 * SPDX-License-Identifier: BSD-2-Clause
 *
 * Copyright (c) 2019 Western Digital Corporation or its affiliates.
 *
 * Authors:
 *   Anup Patel <anup.patel@wdc.com>
 */

#include <sbi/riscv_asm.h>
#include <sbi/riscv_encoding.h>
#include <sbi/riscv_elf.h>
#include <sbi/sbi_platform.h>
#include <sbi/sbi_scratch.h>
#include <sbi/sbi_trap.h>

#define BOOT_STATUS_RELOCATE_DONE	1
#define BOOT_STATUS_BOOT_HART_DONE	2

.macro	MOV_3R __d0, __s0, __d1, __s1, __d2, __s2
	add	\__d0, \__s0, zero
	add	\__d1, \__s1, zero
	add	\__d2, \__s2, zero
.endm

.macro	MOV_5R __d0, __s0, __d1, __s1, __d2, __s2, __d3, __s3, __d4, __s4
	add	\__d0, \__s0, zero
	add	\__d1, \__s1, zero
	add	\__d2, \__s2, zero
	add	\__d3, \__s3, zero
	add	\__d4, \__s4, zero
.endm

/*
 * If __start_reg <= __check_reg and __check_reg < __end_reg then
 *   jump to __pass
 */
.macro BRANGE __start_reg, __end_reg, __check_reg, __jump_lable
	blt	\__check_reg, \__start_reg, 999f
	bge	\__check_reg, \__end_reg, 999f
	j	\__jump_lable
999:
.endm

	.section .entry, "ax", %progbits
	.align 3
	.globl _start
	.globl _start_warm
_start:
	/* n: 把a0,a1,a2的值先保存到s0,s1,s2, 下面的fw_boot_hart要用a0,a1,a2 */
	/* Find preferred boot HART id */
	MOV_3R	s0, a0, s1, a1, s2, a2
	/*
	 * n: 调用这个函数获取boot hart的id，对于fw_dynamic, fw_jump, fw_payload
	 *    三种类型的bios, boot hart是不一样的，fw_jump和fw_payload这里直接返回
	 *    -1, fw_dynamic这种bios情况有点不一样，启动fw_dynamic的上一级程序会把
	 *    参数放到struct fw_dynamic_info里，用a2传给opensbi，根据具体代码可以
	 *    看出，根据fw_dynamic_info version不同，boot hart是不一样的，version1
	 *    是固定的boot hart, version2返回-1，没有定义boot hart。
	 */
	call	fw_boot_hart
	/* n: boot hart id放到a6里 */
	add	a6, a0, zero
	/* n: 这里恢复a0,a1,a2的值 */
	MOV_3R	a0, s0, a1, s1, a2, s2
	li	a7, -1
	/* n: 如果boot hart id是-1，使用lottery的方式选择boot hart */
	beq	a6, a7, _try_lottery
	/*
	 * n: 如果是固定的boot hart, a6是上一级定义的boot hart，a0是当前hart id,
	 *    这里可以参考qemu的代码：hw/riscv/boot.c, reset_vec定义的地方，第三条
	 *    指令获取当前hart的hart id值。两者不相等，说明不是指定boot hart，跳到
	 *    重定位结束的位置。
	 */
	/* Jump to relocation wait loop if we are not boot hart */
	bne	a0, a6, _wait_relocate_copy_done
_try_lottery:
	/*
	 * n: 多个核执行到了这里，只要启动核去做二进制的重定位就好。其他核到重定位
	 *    结束的位置就好。
	 * 
	 *    lla和la有什么区别?
	 */
	/* Jump to relocation wait loop if we don't get relocation lottery */
	lla	a6, _relocate_lottery
	li	a7, 1
	/*
	 * n: a6指向的地址上的值(_relocate_lottery的地址)做原子加1，_relocate_lottery
	 *    的老值写入a6。
	 */
	amoadd.w a6, a7, (a6)
	/*
	 * n: _relocate_lottery不等于0，就跳到boot hart做完重定位的地方。如果多个
	 *    核一起启动执行，只有最先执行上面原子加指令的核的a6(_relocate_lottery
	 *    初始值)是0，所以，后续执行到这里的核都是从核，直接跳到重定位完成的
	 *    地址。
	 */
	bnez	a6, _wait_relocate_copy_done

	/* n: t0是_load_start这个符号的地址，这个地址存的是_fw_start的值 */
	/* Save load address */
	lla	t0, _load_start
	/*
	 * n: t1是_fw_start这个符号的地址，三种fw中，_fw_start这个符号都在二进制
	 *    的起始位置，所以，这里取_fw_start的运行时的地址，其实就是二进制实际
	 *。  被加载到的地址。
	 */
	lla	t1, _fw_start
	/*
	 * n: 保存实际二进制实际加载位置到_load_start, qemu virt平台上的链接地址
	 *    和实际加载地址是一样的，都是内存0x80000000这个地址，但是，如果bios
	 *    是加载到其他地方，比如flash里，链接地址和实际加载位置就是不一样的。
	 *    这个时候就需要做relocate，把二进制从实际加载位置拷贝到链接地址。
	 */
	REG_S	t1, 0(t0)

#ifdef FW_PIC
	/*
	 * n: opensbi编译生成最终bios的二进制以及sbi相关的库，最终二进制和sbi库
	 *    可以是动态链接的，那么这就需要对于动态链接的符号做下重定位。
	 *
	 *    这块的内容相对独立，可以先跳过。
	 */
	/* relocate the global table content */
	lla	t0, _link_start
	REG_L	t0, 0(t0)
	/* n: 实际加载地址 - 链接地址 */
	/* t1 shall has the address of _fw_start */
	sub	t2, t1, t0
	lla	t3, _runtime_offset
	REG_S	t2, (t3)
	/* n: 对于PIC的情况，只看这一个段就好了？*/
	lla	t0, __rel_dyn_start
	lla	t1, __rel_dyn_end
	/* n: 这两个符号的地址想等，表示没有这个段 */
	beq	t0, t1, _relocate_done
	j	5f
2:
	REG_L	t5, -(REGBYTES*2)(t0)	/* t5 <-- relocation info:type */
	li	t3, R_RISCV_RELATIVE	/* reloc type R_RISCV_RELATIVE */
	bne	t5, t3, 3f
	REG_L	t3, -(REGBYTES*3)(t0)
	REG_L	t5, -(REGBYTES)(t0)	/* t5 <-- addend */
	add	t5, t5, t2
	add	t3, t3, t2
	REG_S	t5, 0(t3)		/* store runtime address to the GOT entry */
	j	5f

3:
	lla	t4, __dyn_sym_start

4:
	REG_L	t5, -(REGBYTES*2)(t0)	/* t5 <-- relocation info:type */
	srli	t6, t5, SYM_INDEX	/* t6 <--- sym table index */
	andi	t5, t5, 0xFF		/* t5 <--- relocation type */
	li	t3, RELOC_TYPE
	bne	t5, t3, 5f

	/* address R_RISCV_64 or R_RISCV_32 cases*/
	REG_L	t3, -(REGBYTES*3)(t0)
	li	t5, SYM_SIZE
	mul	t6, t6, t5
	add	s5, t4, t6
	REG_L	t6, -(REGBYTES)(t0)	/* t0 <-- addend */
	REG_L	t5, REGBYTES(s5)
	add	t5, t5, t6
	add	t5, t5, t2		/* t5 <-- location to fix up in RAM */
	add	t3, t3, t2		/* t3 <-- location to fix up in RAM */
	REG_S	t5, 0(t3)		/* store runtime address to the variable */

5:
	/* n: 循环处理.rel.dyn这个段里的信息 */
	addi	t0, t0, (REGBYTES*3)
	ble	t0, t1, 2b
	j	_relocate_done
_wait_relocate_copy_done:
	j	_wait_for_boot_hart
#else
	/* Relocate if load address != link address */
_relocate:
	lla	t0, _link_start
	REG_L	t0, 0(t0)
	lla	t1, _link_end
	REG_L	t1, 0(t1)
	/* n: 加载地址在上面已经得到，这里把加载地址取出，放到t2 */
	lla	t2, _load_start
	REG_L	t2, 0(t2)
	sub	t3, t1, t0
	add	t3, t3, t2
	/* n: 如果link address和load address相等，不需要relocate了 */
	beq	t0, t2, _relocate_done
	/* n: 否则，开始做relocate */
	lla	t4, _relocate_done
	/* n: t4是_relocate_done - _load_start */
	sub	t4, t4, t2
	/* n: t4现在是link下，_relocate_done应该在的地址 */
	add	t4, t4, t0
	/* n: _load_start比_link_start小 */
	blt	t2, t0, _relocate_copy_to_upper
	/*
	 * n: _load_start比_link_start大, 往低地址拷贝数据, 可以看到t2和t0是拷贝
	 *    时的指针。 所以，_link_end(t1)要比_load_start(t2)小，不然拷贝的时候会
	 *    出现重叠的情况。
	 *
	 *     _link_start <--- t0
	 *
	 *
	 *     _link_end   <--- t1
	 *
	 *     _load_start <--- t2
	 *
	 *
	 *     _load_end   <--- t3
	 */
_relocate_copy_to_lower:
	ble	t1, t2, _relocate_copy_to_lower_loop
	/* n: 出现重叠的情况，_relocate_lottery这个位置有什么特殊的意义？*/
	lla	t3, _relocate_lottery
	/* n: t2在[t1, t3)内就跳到_start_hang, 异常处理逻辑没有看懂? */
	BRANGE	t2, t1, t3, _start_hang
	lla	t3, _boot_status
	BRANGE	t2, t1, t3, _start_hang
	lla	t3, _relocate
	lla	t5, _relocate_done
	BRANGE	t2, t1, t3, _start_hang
	BRANGE	t2, t1, t5, _start_hang
	BRANGE  t3, t5, t2, _start_hang
_relocate_copy_to_lower_loop:
	/* n: load address向link address拷贝 */
	REG_L	t3, 0(t2)
	REG_S	t3, 0(t0)
	add	t0, t0, __SIZEOF_POINTER__
	add	t2, t2, __SIZEOF_POINTER__
	blt	t0, t1, _relocate_copy_to_lower_loop
	/* n: 循环拷贝完数据就跳到_relocate_done */
	jr	t4
_relocate_copy_to_upper:
	/*
	 * n: _load_start比_link_start小, 往高地址拷贝数据, 可以看到t2和t0是拷贝
	 *    时的指针。 所以，_load_end(t3)要比_link_start(t0)小，不然拷贝的时候会
	 *    出现重叠的情况。
	 *
	 *     _load_start <--- t2
	 *
	 *
	 *     _load_end   <--- t3
	 *
	 *     _link_start <--- t0
	 *
	 *
	 *     _link_end   <--- t1
	 */
	ble	t3, t0, _relocate_copy_to_upper_loop
	/* n: 异常处理 */
	lla	t2, _relocate_lottery
	BRANGE	t0, t3, t2, _start_hang
	lla	t2, _boot_status
	BRANGE	t0, t3, t2, _start_hang
	lla	t2, _relocate
	lla	t5, _relocate_done
	BRANGE	t0, t3, t2, _start_hang
	BRANGE	t0, t3, t5, _start_hang
	BRANGE	t2, t5, t0, _start_hang
_relocate_copy_to_upper_loop:
	/* n: 倒着拷贝 */
	add	t3, t3, -__SIZEOF_POINTER__
	add	t1, t1, -__SIZEOF_POINTER__
	REG_L	t2, 0(t3)
	REG_S	t2, 0(t1)
	blt	t0, t1, _relocate_copy_to_upper_loop
	jr	t4
_wait_relocate_copy_done:
	lla	t0, _fw_start
	lla	t1, _link_start
	REG_L	t1, 0(t1)
	/*
	 * n: 没有动态链接的情况下，非启动核看到加载地址和链接地址一样时，可以
	 *    直接跳到等待的地址，实际上relocate已经完成。对于没有relocate完成的
	 *    情况，下面先计算出relocate完成后wait_for_boot_hart的地址，但是没有
	 *    relocate完之前还不能跳过去，先把计算出的地址存到t3，然后在一个循环
	 *    里反复检测_boot_status的值
	 */
	beq	t0, t1, _wait_for_boot_hart
	lla	t2, _boot_status
	lla	t3, _wait_for_boot_hart
	/* n: 计算wait_for_boot_hart - _fw_start */
	sub	t3, t3, t0
	/* n: 计算出链接后wait_for_boot_hart的地址(t3) */
	add	t3, t3, t1
1:
	/* waitting for relocate copy done (_boot_status == 1) */
	li	t4, BOOT_STATUS_RELOCATE_DONE
	REG_L	t5, 0(t2)
	/* Reduce the bus traffic so that boot hart may proceed faster */
	nop
	nop
	nop
	/*
	 * n: t5表示当前hart的状态，t4是relocate完成状态, hart一开始_boot_status
	 *    是0，relocate done的状态是1，当relocate拷贝完时，_boot_statue的值
	 *    变成1，跳出这个等待。
	 *
	 *    主核在完成relocate后会把_boot_statue赋1。
	 */
	bgt     t4, t5, 1b
	/* n: 跳到wait_for_boot_hart。从核的视角，它先等待主核做完relocate， */
	jr	t3
#endif
_relocate_done:
	/*
	 * n: 要从多核的角度去看这个代码，主核，也就是做relocate的核，做完relocate
	 *    后会跳到这里，它配置_boot_status这个全局变量为BOOT_STATUS_RELOCATE_DONE，
	 *    但是从核会在上面_wait_relocate_copy_done这里循环等待，直到主核完成
	 *    relocate。
	 */
	/*
	 * Mark relocate copy done
	 * Use _boot_status copy relative to the load address
	 */
	lla	t0, _boot_status
	/* n: 计算_boot_status的地址，为啥PIC的时候的逻辑不一样？*/
#ifndef FW_PIC
	lla	t1, _link_start
	REG_L	t1, 0(t1)
	lla	t2, _load_start
	REG_L	t2, 0(t2)
	sub	t0, t0, t1
	add	t0, t0, t2
#endif
	li	t1, BOOT_STATUS_RELOCATE_DONE
	REG_S	t1, 0(t0)
	fence	rw, rw

	/* At this point we are running from link address */

	/*
	 * n: 下面的逻辑就比较直白了，如下是主核的启动逻辑, 从核会跳到wait_for_boot_hart，
	 *    等主核把如下的这些公共逻辑执行一下，等到主核完全启动后，从核从
	 *    _start_warm继续跑。
	 *
	 *    为啥要等主核完全启动？
	 */
	/* Reset all registers for boot HART */
	li	ra, 0
	/* n: ra, a0, a1, a2不清，其他都清0 */
	call	_reset_regs

	/* n: 循环清0 BSS段 */
	/* Zero-out BSS */
	lla	s4, _bss_start
	lla	s5, _bss_end
_bss_zero:
	REG_S	zero, (s4)
	add	s4, s4, __SIZEOF_POINTER__
	blt	s4, s5, _bss_zero

	/* Setup temporary trap handler */
	lla	s4, _start_hang
	csrw	CSR_MTVEC, s4

	/* Setup temporary stack */
	lla	s4, _fw_end
	/* n: s5设置成8KB */
	li	s5, (SBI_SCRATCH_SIZE * 2)
	/* n: 从opensbi二进制加载结束位置开始的8K设置为栈，栈向低地址生长 */
	add	sp, s4, s5

	/* n: 保存a0-a4, fw_save_info会用到 */
	/* Allow main firmware to save info */
	MOV_5R	s0, a0, s1, a1, s2, a2, s3, a3, s4, a4
	/*
	 * n: 把上一个阶段的启动信息，放到这个阶段和下一个阶段通信的数据结构里，
	 *    目前，只有fw_dynamic会通过一个数据结构向内核传递启动参数。
	 */
	call	fw_save_info
	MOV_5R	a0, s0, a1, s1, a2, s2, a3, s3, a4, s4

	/* n: 看起来opensbi支持配置dtb进来 */
#ifdef FW_FDT_PATH
	/* Override previous arg1 */
	lla	a1, fw_fdt_bin
#endif

	/*
	 * Initialize platform
	 * Note: The a0 to a4 registers passed to the
	 * firmware are parameters to this function.
	 */
	MOV_5R	s0, a0, s1, a1, s2, a2, s3, a3, s4, a4
	/* n: 返回值是dtb的地址, generic platform也就是qemu virt有这个函数，貌似没有干什么 */
	call	fw_platform_init
	add	t0, a0, zero
	MOV_5R	a0, s0, a1, s1, a2, s2, a3, s3, a4, s4
	add	a1, t0, zero

	/*
	 * n: platform是平台相关的数据结构，qemu virt的定义在:
	 *    platform/generic/platform.c, 这个是opensbi里针对qemu virt平台定义
	 *    的配置。最大的核数是128，栈大小是8KB。
	 */
	/* Preload HART details
	 * s7 -> HART Count
	 * s8 -> HART Stack Size
	 */
	lla	a4, platform
#if __riscv_xlen > 32
	lwu	s7, SBI_PLATFORM_HART_COUNT_OFFSET(a4)
	lwu	s8, SBI_PLATFORM_HART_STACK_SIZE_OFFSET(a4)
#else
	lw	s7, SBI_PLATFORM_HART_COUNT_OFFSET(a4)
	lw	s8, SBI_PLATFORM_HART_STACK_SIZE_OFFSET(a4)
#endif

	/* Setup scratch space for all the HARTs */
	lla	tp, _fw_end
	/* n: 算出全部核需要的栈内存大小 */
	mul	a5, s7, s8
	/* n: tp,t3都指向全部栈内存的起始基地址 */
	add	tp, tp, a5
	/* Keep a copy of tp */
	add	t3, tp, zero
	/* Counter */
	li	t2, 1
	/* hartid 0 is mandated by ISA */
	li	t1, 0
_scratch_init:
	/*
	 * The following registers hold values that are computed before
	 * entering this block, and should remain unchanged.
	 *
	 * t3 -> the firmware end address <--- 程序会从二进制加载的结尾分配栈内存, t3指向最后的结尾。
	 * s7 -> HART count
	 * s8 -> HART stack size
	 */
	/* n: tp拿到全部栈内存的基地址 */
	add	tp, t3, zero
	/*
	 * n: t1是首次是0，所以，首次计算a5是0，这个t1是hart的编号，s8是每个核的栈大小，
	 *    所以，a5是每个hart的栈的偏移。
	 */
	mul	a5, s8, t1
	/*
	 * n: 可以看到每个hart的栈是全部栈内存的基地址开始，随hart id增加，依次
	 *    向小地址分配的，每个核的scratch内存是4KB，是从栈的中间开始的。    
         *    tp这里是scratch的开始地址。整个分布的示意图如下。
	 */
	sub	tp, tp, a5
	li	a5, SBI_SCRATCH_SIZE
	/*
	 * n: tp这里是scratch开始地址，scratch的基本排布大概类似：
	 *
	 *    _fw_start 
	 *
	 *
	 *    _fw_end
	 *                                          |
	 *                                          |  地址增大
	 *                                          |
	 *                                          v
	 *    stack                                      
	 *    ^                                      
	 *    | 8KB                                     
	 *    |                                      
	 *    |  scratch start(hart 0)     <--- tp
	 *    |    ^
	 *    |    | 4KB
	 *    |    v
	 *    v  scratch end               <--- t3
	 */    
	sub	tp, tp, a5

	/* Initialize scratch space */
	/* Store fw_start and fw_size in scratch space */
	lla	a4, _fw_start
	/*
         * n: t3是包含栈内存的fw的结尾地址，所以，a5是包含栈内存的fw的大小，这个值
         *    要保存到scratch内存。
         */
	sub	a5, t3, a4
	/* n: 把各种信息存到scratch内存里 */
	REG_S	a4, SBI_SCRATCH_FW_START_OFFSET(tp)
	REG_S	a5, SBI_SCRATCH_FW_SIZE_OFFSET(tp)
	/* Store next arg1 in scratch space */
	MOV_3R	s0, a0, s1, a1, s2, a2
	/*
	 * n: 从opensbi的上一个bootloader拿opensbi启动下一个系统(一般是内核)。可以
	 *    看到，三种fw的函数个不相同，fw_dynamic对上和对下都有固定的参数传递
	 *    数据结构；fw_jump和fw_payload的处理一样，可以是编译配置的参数，也可以
	 *    是从a1传递的参数。a0是如下函数的返回值。
         *
         *    可以看到_dynamic_next_arg1相关的内存和scratch内存是两个不同的概念。
         *    下面就是填scratch的各个域段。
	 */
	call	fw_next_arg1
	REG_S	a0, SBI_SCRATCH_NEXT_ARG1_OFFSET(tp)
	MOV_3R	a0, s0, a1, s1, a2, s2
	/* Store next address in scratch space */
	MOV_3R	s0, a0, s1, a1, s2, a2
	/* n：如下的逻辑都和上面的一样，只是配置的信息不一样, 这里是配置下个系统的初始地址 */
	call	fw_next_addr
	REG_S	a0, SBI_SCRATCH_NEXT_ADDR_OFFSET(tp)
	MOV_3R	a0, s0, a1, s1, a2, s2
	/* Store next mode in scratch space */
	MOV_3R	s0, a0, s1, a1, s2, a2
	/* n: 配置下个系统的CPU mode, 三种fw都是直接在opensbi代码里配死成S mode */
	call	fw_next_mode
	REG_S	a0, SBI_SCRATCH_NEXT_MODE_OFFSET(tp)
	MOV_3R	a0, s0, a1, s1, a2, s2
	/* Store warm_boot address in scratch space */
	lla	a4, _start_warm
	REG_S	a4, SBI_SCRATCH_WARMBOOT_ADDR_OFFSET(tp)
	/* Store platform address in scratch space */
	lla	a4, platform
	REG_S	a4, SBI_SCRATCH_PLATFORM_ADDR_OFFSET(tp)
	/* n: _hartid_to_scratch函数的功能？*/
	/* Store hartid-to-scratch function address in scratch space */
	lla	a4, _hartid_to_scratch
	REG_S	a4, SBI_SCRATCH_HARTID_TO_SCRATCH_OFFSET(tp)
	/* n: 可以看到_trap_exit就是恢复异常现场，然后调用mret回到异常点 */
	/* Store trap-exit function address in scratch space */
	lla	a4, _trap_exit
	REG_S	a4, SBI_SCRATCH_TRAP_EXIT_OFFSET(tp)
	/* Clear tmp0 in scratch space */
	REG_S	zero, SBI_SCRATCH_TMP0_OFFSET(tp)
	/* Store firmware options in scratch space */
	MOV_3R	s0, a0, s1, a1, s2, a2
#ifdef FW_OPTIONS
	li	a0, FW_OPTIONS
#else
	call	fw_options
#endif
	REG_S	a0, SBI_SCRATCH_OPTIONS_OFFSET(tp)
	MOV_3R	a0, s0, a1, s1, a2, s2
	/* Move to next scratch space */
	add	t1, t1, t2
	/* n: 循环处理所有核的scratch内存 */
	blt	t1, s7, _scratch_init

	/*
	 * Relocate Flatened Device Tree (FDT)
	 * source FDT address = previous arg1    <--- previous arg1和next arg1一样么？
	 * destination FDT address = next arg1
	 *
	 * Note: We will preserve a0 and a1 passed by
	 * previous booting stage.
	 */
	/* n: 貌似a1是0表示device tree relocate搞定了, 跳到_fdt_reloc_done做warm boot? */
	beqz	a1, _fdt_reloc_done
	/* Mask values in a4 */
	li	a4, 0xff
	/* t1 = destination FDT start address */
	MOV_3R	s0, a0, s1, a1, s2, a2
	call	fw_next_arg1
	add	t1, a0, zero
	MOV_3R	a0, s0, a1, s1, a2, s2
	/* n: 哪里配置是0的？*/
	beqz	t1, _fdt_reloc_done
	beq	t1, a1, _fdt_reloc_done
	/* t0 = source FDT start address */
	/* n: 开始dts relocate */
	add	t0, a1, zero
	/* t2 = source FDT size in big-endian */
#if __riscv_xlen == 64
	lwu	t2, 4(t0)
#else
	lw	t2, 4(t0)
#endif
	/* n: 把大端的数据转成小端的？*/
	/* t3 = bit[15:8] of FDT size */
	add	t3, t2, zero
	srli	t3, t3, 16
	and	t3, t3, a4
	slli	t3, t3, 8
	/* t4 = bit[23:16] of FDT size */
	add	t4, t2, zero
	srli	t4, t4, 8
	and	t4, t4, a4
	slli	t4, t4, 16
	/* t5 = bit[31:24] of FDT size */
	add	t5, t2, zero
	and	t5, t5, a4
	slli	t5, t5, 24
	/* t2 = bit[7:0] of FDT size */
	srli	t2, t2, 24
	and	t2, t2, a4
	/* t2 = FDT size in little-endian */
	or	t2, t2, t3
	or	t2, t2, t4
	or	t2, t2, t5
	/* n: 为啥dtb的地址会不一样？*/
	/* t2 = destination FDT end address */
	add	t2, t1, t2
	/* FDT copy loop */
	ble	t2, t1, _fdt_reloc_done
_fdt_reloc_again:
	REG_L	t3, 0(t0)
	REG_S	t3, 0(t1)
	add	t0, t0, __SIZEOF_POINTER__
	add	t1, t1, __SIZEOF_POINTER__
	blt	t1, t2, _fdt_reloc_again
_fdt_reloc_done:

	/* mark boot hart done */
	li	t0, BOOT_STATUS_BOOT_HART_DONE
	lla	t1, _boot_status
	REG_S	t0, 0(t1)
	fence	rw, rw
	j	_start_warm

	/* n: 非启动核都等在这里 */
	/* waiting for boot hart to be done (_boot_status == 2) */
_wait_for_boot_hart:
	li	t0, BOOT_STATUS_BOOT_HART_DONE
	lla	t1, _boot_status
	REG_L	t1, 0(t1)
	/* Reduce the bus traffic so that boot hart may proceed faster */
	nop
	nop
	nop
	/*
	 * n: 非启动核都等在这里，不断的检测_boot_status，启动核完成启动后会把
	 *    _boot_status配置成BOOT_STATUS_BOOT_HART_DONE?
	 */
	bne	t0, t1, _wait_for_boot_hart

	/*
	 * n: 主从核启动都要跑这个，无非是从核先在_wait_for_boot_hart那里等待，
	 *    直到主核继续执行完一些公共的处理逻辑，比如上面的bss清理，scratch内存
	 *    初始化，dtb重定位等。
	 */
_start_warm:
	/* Reset all registers for non-boot HARTs */
	li	ra, 0
	call	_reset_regs

	/* Disable and clear all interrupts */
	csrw	CSR_MIE, zero
	csrw	CSR_MIP, zero

	/* Find HART count and HART stack size */
	lla	a4, platform
#if __riscv_xlen == 64
	lwu	s7, SBI_PLATFORM_HART_COUNT_OFFSET(a4)
	lwu	s8, SBI_PLATFORM_HART_STACK_SIZE_OFFSET(a4)
#else
	lw	s7, SBI_PLATFORM_HART_COUNT_OFFSET(a4)
	lw	s8, SBI_PLATFORM_HART_STACK_SIZE_OFFSET(a4)
#endif
	/*
	 * n: 似乎是从sbi_platform这个数据结构里取出hart_index2id这个表里定义的
	 *    数组index到hard id的映射，当没有这个数组，也就是hart_index2id域段
	 *    是0的时候，表示一一映射。
	 */
	REG_L	s9, SBI_PLATFORM_HART_INDEX2ID_OFFSET(a4)

	/* n: 获取当前核的hart id */
	/* Find HART id */
	csrr	s6, CSR_MHARTID

	/* n: hart_index2id是NULL，s6中已经是得到hart_id */
	/* Find HART index */
	beqz	s9, 3f
	li	a4, 0
1:
	/* n: hart_index2id不是NULL, 第一次执行时，a5是hart_index2id[0]的值 */
#if __riscv_xlen == 64
	lwu	a5, (s9)
#else
	lw	a5, (s9)
#endif
	/* n: 为啥要在hart_index2id这个表里搜索当前hart id? */
	beq	a5, s6, 2f
	/* n: s9指向hart_index2id数组的下一个位置 */
	add	s9, s9, 4
	add	a4, a4, 1
	blt	a4, s7, 1b
	/* n: 遍历hart id已经异常 */
	li	a4, -1
2:	add	s6, a4, zero
	/* n: 判断是否当前hart id有异常 */
3:	bge	s6, s7, _start_hang

	/* n: tp指向的是scratch内存的基地址 */
	/* Find the scratch space based on HART index */
	lla	tp, _fw_end
	mul	a5, s7, s8
	add	tp, tp, a5
	mul	a5, s8, s6
	sub	tp, tp, a5
	li	a5, SBI_SCRATCH_SIZE
	sub	tp, tp, a5

	/* update the mscratch */
	csrw	CSR_MSCRATCH, tp

	/* n: 这里比较奇怪，上面分配了8K, 这里只用了4KB? */
	/* Setup stack */
	add	sp, tp, zero

	/* n: 配上M mode的异常处理向量 */
	/* Setup trap handler */
	lla	a4, _trap_handler
#if __riscv_xlen == 32
	csrr	a5, CSR_MISA
	srli	a5, a5, ('H' - 'A')
	/* n: rv32? 如果支持虚拟化扩展，中断异常配置不同的向量？*/
	andi	a5, a5, 0x1
	beq	a5, zero, _skip_trap_handler_rv32_hyp
	lla	a4, _trap_handler_rv32_hyp
_skip_trap_handler_rv32_hyp:
#endif
	csrw	CSR_MTVEC, a4

#if __riscv_xlen == 32
	/* n: 这里的逻辑还是不清楚，对于虚拟化扩展，异常退出的逻辑也不一样？*/
	/* Override trap exit for H-extension */
	csrr	a5, CSR_MISA
	srli	a5, a5, ('H' - 'A')
	andi	a5, a5, 0x1
	beq	a5, zero, _skip_trap_exit_rv32_hyp
	lla	a4, _trap_exit_rv32_hyp
	csrr	a5, CSR_MSCRATCH
	REG_S	a4, SBI_SCRATCH_TRAP_EXIT_OFFSET(a5)
_skip_trap_exit_rv32_hyp:
#endif

	/* Initialize SBI runtime */
	csrr	a0, CSR_MSCRATCH
	call	sbi_init

	/* We don't expect to reach here hence just hang */
	j	_start_hang

	.data
	.align 3
#ifdef FW_PIC
_runtime_offset:
	RISCV_PTR	0
#endif
	/* n: 定义一个dword的空间，整体上看，这里相当于定义了一些全局变量 */
_relocate_lottery:
	RISCV_PTR	0
_boot_status:
	RISCV_PTR	0
_load_start:
	RISCV_PTR	_fw_start
_link_start:
	RISCV_PTR	FW_TEXT_START
_link_end:
	RISCV_PTR	_fw_reloc_end

	.section .entry, "ax", %progbits
	.align 3
	.globl _hartid_to_scratch
_hartid_to_scratch:
	/*
	 * a0 -> HART ID (passed by caller)
	 * a1 -> HART Index (passed by caller)
	 * t0 -> HART Stack Size
	 * t1 -> HART Stack End
	 * t2 -> Temporary
	 */
	lla	t2, platform
#if __riscv_xlen == 64
	lwu	t0, SBI_PLATFORM_HART_STACK_SIZE_OFFSET(t2)
	lwu	t2, SBI_PLATFORM_HART_COUNT_OFFSET(t2)
#else
	lw	t0, SBI_PLATFORM_HART_STACK_SIZE_OFFSET(t2)
	lw	t2, SBI_PLATFORM_HART_COUNT_OFFSET(t2)
#endif
	sub	t2, t2, a1
	mul	t2, t2, t0
	lla	t1, _fw_end
	add	t1, t1, t2
	li	t2, SBI_SCRATCH_SIZE
	sub	a0, t1, t2
	ret

	.section .entry, "ax", %progbits
	.align 3
	.globl _start_hang
_start_hang:
	wfi
	j	_start_hang

	.section .entry, "ax", %progbits
	.align 3
	.weak fw_platform_init
fw_platform_init:
	add	a0, a1, zero
	ret

	/* Map implicit memcpy() added by compiler to sbi_memcpy() */
	.section .text
	.align 3
	.globl memcpy
memcpy:
	tail	sbi_memcpy

	/* Map implicit memset() added by compiler to sbi_memset() */
	.section .text
	.align 3
	.globl memset
memset:
	tail	sbi_memset

	/* Map implicit memmove() added by compiler to sbi_memmove() */
	.section .text
	.align 3
	.globl memmove
memmove:
	tail	sbi_memmove

	/* Map implicit memcmp() added by compiler to sbi_memcmp() */
	.section .text
	.align 3
	.globl memcmp
memcmp:
	tail	sbi_memcmp

.macro	TRAP_SAVE_AND_SETUP_SP_T0
	/* Swap TP and MSCRATCH */
	csrrw	tp, CSR_MSCRATCH, tp

	/* Save T0 in scratch space */
	REG_S	t0, SBI_SCRATCH_TMP0_OFFSET(tp)

	/*
	 * Set T0 to appropriate exception stack
	 *
	 * Came_From_M_Mode = ((MSTATUS.MPP < PRV_M) ? 1 : 0) - 1;
	 * Exception_Stack = TP ^ (Came_From_M_Mode & (SP ^ TP))
	 *
	 * Came_From_M_Mode = 0    ==>    Exception_Stack = TP
	 * Came_From_M_Mode = -1   ==>    Exception_Stack = SP
	 */
	csrr	t0, CSR_MSTATUS
	srl	t0, t0, MSTATUS_MPP_SHIFT
	and	t0, t0, PRV_M
	slti	t0, t0, PRV_M
	add	t0, t0, -1
	xor	sp, sp, tp
	and	t0, t0, sp
	xor	sp, sp, tp
	xor	t0, tp, t0

	/* Save original SP on exception stack */
	REG_S	sp, (SBI_TRAP_REGS_OFFSET(sp) - SBI_TRAP_REGS_SIZE)(t0)

	/* Set SP to exception stack and make room for trap registers */
	add	sp, t0, -(SBI_TRAP_REGS_SIZE)

	/* Restore T0 from scratch space */
	REG_L	t0, SBI_SCRATCH_TMP0_OFFSET(tp)

	/* Save T0 on stack */
	REG_S	t0, SBI_TRAP_REGS_OFFSET(t0)(sp)

	/* Swap TP and MSCRATCH */
	csrrw	tp, CSR_MSCRATCH, tp
.endm

.macro	TRAP_SAVE_MEPC_MSTATUS have_mstatush
	/* Save MEPC and MSTATUS CSRs */
	csrr	t0, CSR_MEPC
	REG_S	t0, SBI_TRAP_REGS_OFFSET(mepc)(sp)
	csrr	t0, CSR_MSTATUS
	REG_S	t0, SBI_TRAP_REGS_OFFSET(mstatus)(sp)
	.if \have_mstatush
	csrr	t0, CSR_MSTATUSH
	REG_S	t0, SBI_TRAP_REGS_OFFSET(mstatusH)(sp)
	.else
	REG_S	zero, SBI_TRAP_REGS_OFFSET(mstatusH)(sp)
	.endif
.endm

.macro	TRAP_SAVE_GENERAL_REGS_EXCEPT_SP_T0
	/* Save all general regisers except SP and T0 */
	REG_S	zero, SBI_TRAP_REGS_OFFSET(zero)(sp)
	REG_S	ra, SBI_TRAP_REGS_OFFSET(ra)(sp)
	REG_S	gp, SBI_TRAP_REGS_OFFSET(gp)(sp)
	REG_S	tp, SBI_TRAP_REGS_OFFSET(tp)(sp)
	REG_S	t1, SBI_TRAP_REGS_OFFSET(t1)(sp)
	REG_S	t2, SBI_TRAP_REGS_OFFSET(t2)(sp)
	REG_S	s0, SBI_TRAP_REGS_OFFSET(s0)(sp)
	REG_S	s1, SBI_TRAP_REGS_OFFSET(s1)(sp)
	REG_S	a0, SBI_TRAP_REGS_OFFSET(a0)(sp)
	REG_S	a1, SBI_TRAP_REGS_OFFSET(a1)(sp)
	REG_S	a2, SBI_TRAP_REGS_OFFSET(a2)(sp)
	REG_S	a3, SBI_TRAP_REGS_OFFSET(a3)(sp)
	REG_S	a4, SBI_TRAP_REGS_OFFSET(a4)(sp)
	REG_S	a5, SBI_TRAP_REGS_OFFSET(a5)(sp)
	REG_S	a6, SBI_TRAP_REGS_OFFSET(a6)(sp)
	REG_S	a7, SBI_TRAP_REGS_OFFSET(a7)(sp)
	REG_S	s2, SBI_TRAP_REGS_OFFSET(s2)(sp)
	REG_S	s3, SBI_TRAP_REGS_OFFSET(s3)(sp)
	REG_S	s4, SBI_TRAP_REGS_OFFSET(s4)(sp)
	REG_S	s5, SBI_TRAP_REGS_OFFSET(s5)(sp)
	REG_S	s6, SBI_TRAP_REGS_OFFSET(s6)(sp)
	REG_S	s7, SBI_TRAP_REGS_OFFSET(s7)(sp)
	REG_S	s8, SBI_TRAP_REGS_OFFSET(s8)(sp)
	REG_S	s9, SBI_TRAP_REGS_OFFSET(s9)(sp)
	REG_S	s10, SBI_TRAP_REGS_OFFSET(s10)(sp)
	REG_S	s11, SBI_TRAP_REGS_OFFSET(s11)(sp)
	REG_S	t3, SBI_TRAP_REGS_OFFSET(t3)(sp)
	REG_S	t4, SBI_TRAP_REGS_OFFSET(t4)(sp)
	REG_S	t5, SBI_TRAP_REGS_OFFSET(t5)(sp)
	REG_S	t6, SBI_TRAP_REGS_OFFSET(t6)(sp)
.endm

.macro	TRAP_CALL_C_ROUTINE
	/* Call C routine */
	add	a0, sp, zero
	call	sbi_trap_handler
.endm

.macro	TRAP_RESTORE_GENERAL_REGS_EXCEPT_A0_T0
	/* Restore all general regisers except A0 and T0 */
	REG_L	ra, SBI_TRAP_REGS_OFFSET(ra)(a0)
	REG_L	sp, SBI_TRAP_REGS_OFFSET(sp)(a0)
	REG_L	gp, SBI_TRAP_REGS_OFFSET(gp)(a0)
	REG_L	tp, SBI_TRAP_REGS_OFFSET(tp)(a0)
	REG_L	t1, SBI_TRAP_REGS_OFFSET(t1)(a0)
	REG_L	t2, SBI_TRAP_REGS_OFFSET(t2)(a0)
	REG_L	s0, SBI_TRAP_REGS_OFFSET(s0)(a0)
	REG_L	s1, SBI_TRAP_REGS_OFFSET(s1)(a0)
	REG_L	a1, SBI_TRAP_REGS_OFFSET(a1)(a0)
	REG_L	a2, SBI_TRAP_REGS_OFFSET(a2)(a0)
	REG_L	a3, SBI_TRAP_REGS_OFFSET(a3)(a0)
	REG_L	a4, SBI_TRAP_REGS_OFFSET(a4)(a0)
	REG_L	a5, SBI_TRAP_REGS_OFFSET(a5)(a0)
	REG_L	a6, SBI_TRAP_REGS_OFFSET(a6)(a0)
	REG_L	a7, SBI_TRAP_REGS_OFFSET(a7)(a0)
	REG_L	s2, SBI_TRAP_REGS_OFFSET(s2)(a0)
	REG_L	s3, SBI_TRAP_REGS_OFFSET(s3)(a0)
	REG_L	s4, SBI_TRAP_REGS_OFFSET(s4)(a0)
	REG_L	s5, SBI_TRAP_REGS_OFFSET(s5)(a0)
	REG_L	s6, SBI_TRAP_REGS_OFFSET(s6)(a0)
	REG_L	s7, SBI_TRAP_REGS_OFFSET(s7)(a0)
	REG_L	s8, SBI_TRAP_REGS_OFFSET(s8)(a0)
	REG_L	s9, SBI_TRAP_REGS_OFFSET(s9)(a0)
	REG_L	s10, SBI_TRAP_REGS_OFFSET(s10)(a0)
	REG_L	s11, SBI_TRAP_REGS_OFFSET(s11)(a0)
	REG_L	t3, SBI_TRAP_REGS_OFFSET(t3)(a0)
	REG_L	t4, SBI_TRAP_REGS_OFFSET(t4)(a0)
	REG_L	t5, SBI_TRAP_REGS_OFFSET(t5)(a0)
	REG_L	t6, SBI_TRAP_REGS_OFFSET(t6)(a0)
.endm

.macro	TRAP_RESTORE_MEPC_MSTATUS have_mstatush
	/* Restore MEPC and MSTATUS CSRs */
	REG_L	t0, SBI_TRAP_REGS_OFFSET(mepc)(a0)
	csrw	CSR_MEPC, t0
	REG_L	t0, SBI_TRAP_REGS_OFFSET(mstatus)(a0)
	csrw	CSR_MSTATUS, t0
	.if \have_mstatush
	REG_L	t0, SBI_TRAP_REGS_OFFSET(mstatusH)(a0)
	csrw	CSR_MSTATUSH, t0
	.endif
.endm

.macro TRAP_RESTORE_A0_T0
	/* Restore T0 */
	REG_L	t0, SBI_TRAP_REGS_OFFSET(t0)(a0)

	/* Restore A0 */
	REG_L	a0, SBI_TRAP_REGS_OFFSET(a0)(a0)
.endm

	.section .entry, "ax", %progbits
	.align 3
	.globl _trap_handler
	.globl _trap_exit
_trap_handler:
	TRAP_SAVE_AND_SETUP_SP_T0

	TRAP_SAVE_MEPC_MSTATUS 0

	TRAP_SAVE_GENERAL_REGS_EXCEPT_SP_T0

	TRAP_CALL_C_ROUTINE

	/* n: opensbi/lib/sbi/sbi_trap.c里的sbi_trap_exit会调用这里 */
_trap_exit:
	TRAP_RESTORE_GENERAL_REGS_EXCEPT_A0_T0

	TRAP_RESTORE_MEPC_MSTATUS 0

	TRAP_RESTORE_A0_T0

	mret

#if __riscv_xlen == 32
	.section .entry, "ax", %progbits
	.align 3
	.globl _trap_handler_rv32_hyp
	.globl _trap_exit_rv32_hyp
_trap_handler_rv32_hyp:
	TRAP_SAVE_AND_SETUP_SP_T0

	TRAP_SAVE_MEPC_MSTATUS 1

	TRAP_SAVE_GENERAL_REGS_EXCEPT_SP_T0

	TRAP_CALL_C_ROUTINE

_trap_exit_rv32_hyp:
	TRAP_RESTORE_GENERAL_REGS_EXCEPT_A0_T0

	TRAP_RESTORE_MEPC_MSTATUS 1

	TRAP_RESTORE_A0_T0

	mret
#endif

	.section .entry, "ax", %progbits
	.align 3
	.globl _reset_regs
_reset_regs:

	/* flush the instruction cache */
	fence.i
	/* Reset all registers except ra, a0, a1 and a2 */
	li sp, 0
	li gp, 0
	li tp, 0
	li t0, 0
	li t1, 0
	li t2, 0
	li s0, 0
	li s1, 0
	li a3, 0
	li a4, 0
	li a5, 0
	li a6, 0
	li a7, 0
	li s2, 0
	li s3, 0
	li s4, 0
	li s5, 0
	li s6, 0
	li s7, 0
	li s8, 0
	li s9, 0
	li s10, 0
	li s11, 0
	li t3, 0
	li t4, 0
	li t5, 0
	li t6, 0
	csrw CSR_MSCRATCH, 0

	ret

#ifdef FW_FDT_PATH
	.section .rodata
	.align 4
	.globl fw_fdt_bin
fw_fdt_bin:
	.incbin FW_FDT_PATH
#ifdef FW_FDT_PADDING
	.fill FW_FDT_PADDING, 1, 0
#endif
#endif
```
