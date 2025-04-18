---
title: Linux内核riscv entry.S分析
tags:
  - riscv
  - Linux内核
  - 计算机体系结构
description: >-
  本文分析Linux内核里riscv构架下的entry.S这个文件，内核代码使用的版本是5.12-rc8。
  采用直接在代码里加注释的方式写分析的内容，注释用“n:”开头。目前只分析主流程， 一些相关的细节还需要持续更新。
abbrlink: fb0987b1
date: 2022-07-24 22:07:59
categories:
---

```
/* n: 在非抢占的情况下，下面两个符号等价 */
#if !IS_ENABLED(CONFIG_PREEMPTION)
.set resume_kernel, restore_all
#endif

/* n: 在head.S里配置给CSR_TVEC */
ENTRY(handle_exception)
	/*
	 * If coming from userspace, preserve the user thread pointer and load
	 * the kernel thread pointer.  If we came from the kernel, the scratch
	 * register will contain 0, and we should continue on the current TP.
	 */
	/*
	 * n: 交换tp寄存器和scratch寄存器里的值。如果exception从用户态进来，scratch
	 * 的值应该是task_struct(下面在离开内核的时候，会把用户进程的task_struct
	 * 指针赋给tp)，如果exception从内核态进来，tp应该是当前线程的task_struct
	 * 指针，scratch应该是0。
	 *
	 * 所以，下面紧接着的处理中，如果从内核进来，要恢复下tp。
	 *
	 * 总结下在用户态和内核态时tp和scratch的值是什么：
	 *
	 *  +----------+--------------+---------------+
	 *  |          | user space   |   kernel      |
	 *  +----------+--------------+---------------+
	 *  | tp:      | tls base     |   task_struct |
	 *  +----------+--------------+---------------+
	 *  | scratch: | task_struct  |   0           |
	 *  +----------+--------------+---------------+
	 *
	 * 注意：配置tp为tls地址的函数是：copy_thread，如果用户态没有使用TLS，tp
	 *       在用户态的值是？
	 */
	csrrw tp, CSR_SCRATCH, tp
	/* n: tp不为0，异常来自用户态，直接跳到上下文保存的地方 */
	bnez tp, _save_context

_restore_kernel_tpsp:
	/*
	 * n: csrr伪指令，把scratch寄存器的值写入tp，上面为了判断是否在内核把tp
	 * 的值和CSR_SCRATCH的值做了交换。这里恢复tp寄存器。
	 */
	csrr tp, CSR_SCRATCH
	/* n: 把内核sp保存到内核thread_info上 */
	REG_S sp, TASK_TI_KERNEL_SP(tp)
_save_context:
	/*
	 * n: 保存sp到thread_info用户态指针的位置，如果是就在内核态，那么把内核栈
	 * sp保存到了thread_info用户态栈指针的位置。
	 *
	 * 异常或中断来自内核或者用户态，再下面合并处理。当来自用户态时，tp的值和
	 * scratch寄存器的值是一样的，所以这里不需要恢复tp。
	 */
	REG_S sp, TASK_TI_USER_SP(tp)
	/* n: sp换成内核栈 */
	REG_L sp, TASK_TI_KERNEL_SP(tp)
	/*
	 * n: 扩大栈的范围，扩大的范围用来保存相关的寄存器。移动sp其实就相当于在栈上分配空间。
	 *    sp移动之前的值是中断或者异常打断的上下文，也就是中断或异常处理完后要恢复的值。
	 */
	addi sp, sp, -(PT_SIZE_ON_STACK)
	/*
	 * n: 下面的一段代码把各个系统寄存器保存到栈上刚刚开辟出来的空间, 注意需要
	 * 特殊处理的是sp(x2)和tp(x4)。当前的sp，由于上面的变动已经不是需要保存的sp，
	 * 但是，之前我们已经把需要保存的sp放到了thread_info里，所以下面把thread_info
	 * 里的sp取出后再入栈。
	 */
	REG_S x1,  PT_RA(sp)
	REG_S x3,  PT_GP(sp)
	REG_S x5,  PT_T0(sp)
	REG_S x6,  PT_T1(sp)
	REG_S x7,  PT_T2(sp)
	REG_S x8,  PT_S0(sp)
	REG_S x9,  PT_S1(sp)
	REG_S x10, PT_A0(sp)
	REG_S x11, PT_A1(sp)
	REG_S x12, PT_A2(sp)
	REG_S x13, PT_A3(sp)
	REG_S x14, PT_A4(sp)
	REG_S x15, PT_A5(sp)
	REG_S x16, PT_A6(sp)
	REG_S x17, PT_A7(sp)
	REG_S x18, PT_S2(sp)
	REG_S x19, PT_S3(sp)
	REG_S x20, PT_S4(sp)
	REG_S x21, PT_S5(sp)
	REG_S x22, PT_S6(sp)
	REG_S x23, PT_S7(sp)
	REG_S x24, PT_S8(sp)
	REG_S x25, PT_S9(sp)
	REG_S x26, PT_S10(sp)
	REG_S x27, PT_S11(sp)
	REG_S x28, PT_T3(sp)
	REG_S x29, PT_T4(sp)
	REG_S x30, PT_T5(sp)
	REG_S x31, PT_T6(sp)

	/*
	 * Disable user-mode memory access as it should only be set in the
	 * actual user copy routines.
	 *
	 * Disable the FPU to detect illegal usage of floating point in kernel
	 * space.
	 */
	/* n: 配置先放到t0寄存器里，SR_SUM为1容许S mode接入U mode地址，反之不容许 */
	li t0, SR_SUM | SR_FS

	/*
	 * n: 上面已经保存了寄存器现场，下面可以使用系统寄存器了，s0保存用户态sp。
	 * 把task_struct里保存的用户态栈指针提取出来，然后在后面保存到内核栈上。
	 */
	REG_L s0, TASK_TI_USER_SP(tp)
	/* n: 把中断异常相关的csr寄存器读出来, ~t0 & csr_status旧值，然后写入csr_status，相当于清SR_SUM和SR_FS */
	csrrc s1, CSR_STATUS, t0
	csrr s2, CSR_EPC
	csrr s3, CSR_TVAL
	csrr s4, CSR_CAUSE
	csrr s5, CSR_SCRATCH
	/* n: 把中断异常相关的csr寄存器以及用户态sp保存到栈上 */
	REG_S s0, PT_SP(sp)
	REG_S s1, PT_STATUS(sp)
	REG_S s2, PT_EPC(sp)
	REG_S s3, PT_BADADDR(sp)
	REG_S s4, PT_CAUSE(sp)
	REG_S s5, PT_TP(sp)

	/*
	 * Set the scratch register to 0, so that if a recursive exception
	 * occurs, the exception vector knows it came from the kernel
	 */
	csrw CSR_SCRATCH, x0

	/* Load the global pointer */
.option push
.option norelax
	la gp, __global_pointer$
.option pop

	/* n: todo */
#ifdef CONFIG_TRACE_IRQFLAGS
	call trace_hardirqs_off
#endif

	/* n: todo */
#ifdef CONFIG_CONTEXT_TRACKING
	/* If previous state is in user mode, call context_tracking_user_exit. */
	li   a0, SR_PP
	and a0, s1, a0
	bnez a0, skip_context_tracking
	call context_tracking_user_exit
skip_context_tracking:
#endif

	/*
	 * MSB of cause differentiates between
	 * interrupts and exceptions
	 */
	/* n: 最高位为1会解释为一个负数，如果是异常最高位为0，会跳到lable 1f */
	bge s4, zero, 1f

	/* n: 如果是中断，会走到这里，先把返回地址保存到ra */
	la ra, ret_from_exception

	/* Handle interrupts */
	/* n: 上面可以知道sp就是pt_regs的地址，a0存放pt_regs的地址 */
	move a0, sp /* pt_regs */
	/* n: a1存handle_arch_irq的地址 */
	la a1, handle_arch_irq
	/* n: handle_arch_irq看起来像一个数组，这个在哪里定义？*/
	REG_L a1, (a1)
	/* n: 返回地址已经配置到ra里。这里是中断处理函数的入口，下面的处理都和异常有关了 */
	jr a1
1:
	/*
	 * Exceptions run with interrupts enabled or disabled depending on the
	 * state of SR_PIE in m/sstatus.
	 */
	/* n: s1之前被存入sstatus的值，这里把SR_PIE的值存入t0 */
	andi t0, s1, SR_PIE
	/*
	 * n: PIE表示进入中断或者异常之前IE的配置情况，如果之前就是关中断，现在
	 * 继续关中断跑，如果之前是开中断的，需要打开中断继续跑，但是ebreak异常中
	 * 不能开中断，所以，如果之前是开中断的，我们继续检测是不是ebreak异常，
	 * 如果是ebreak异常，在关中断下继续跑，如果不是ebreak异常，会走到csrs CSR_STATUS, SR_IE，
	 * 这里会把中断打开。
	 *
	 * 注意，这里是在处理异常。硬件在处理中断或异常时会改变SPIE/SPP/SIE的值。
	 * SPP会保存中断或异常时CPU的mode，SPIE会保存sstatus里的SIE值，SIE表示S
	 * mode中断是否被enable，最后硬件会关中断就是把SIE配置成0。
	 */
	beqz t0, 1f
	/* kprobes, entered via ebreak, must have interrupts disabled. */
	li t0, EXC_BREAKPOINT
	beq s4, t0, 1f
	/* n: todo */
#ifdef CONFIG_TRACE_IRQFLAGS
	call trace_hardirqs_on
#endif
	/*
	 * n: csrs是csrrs x0, csr, rs, 意思是把csr的值存入x0, csr的旧值和rs做或运算后写入csr。
	 * 这里是把S mode全局中断打开。注意，上面的逻辑是如果检测到是ebreak异常是直接跳过开中断的。
	 */
	csrs CSR_STATUS, SR_IE

	/* n: 开始处理异常，先配置返回地址。非ebreak，且之前是开中断的，异常处理前开中断 */
1:
	la ra, ret_from_exception
	/* Handle syscalls */
	li t0, EXC_SYSCALL
	/* n: 如果是ecall异常就跳到系统调用的地方处理 */
	beq s4, t0, handle_syscall

	/* Handle other exceptions */
	/* n: 向左移动3bit就是得到函数指针的偏移，其他异常的入口函数排到了excp_vect_table这个表里 */
	slli t0, s4, RISCV_LGPTR
	la t1, excp_vect_table
	la t2, excp_vect_table_end
	move a0, sp /* pt_regs */
	/* n: t1是异常向量表的基地址，再加上对应异常处理函数的偏移，t0的值就是对应异常处理函数的地址 */
	add t0, t1, t0
	/* Check if exception code lies within bounds */
	/* n: 计算的异常处理函数的地址超出了异常向量表的结尾 */
	bgeu t0, t2, 1f
	/* n: t0是存放函数指针的地址，这里把t0更新成函数指针 */
	REG_L t0, 0(t0)
	/* n: 跳到对应异常处理函数执行，返回地址还是之前配置的ret_from_exception */
	jr t0
1:
	tail do_trap_unknown

	/* 这个是handle_syscall这个函数的独立代码了, 异常的处理被分为系统调用和其他异常 */
handle_syscall:
	/* n: todo */
#ifdef CONFIG_RISCV_M_MODE
	/*
	 * When running is M-Mode (no MMU config), MPIE does not get set.
	 * As a result, we need to force enable interrupts here because
	 * handle_exception did not do set SR_IE as it always sees SR_PIE
	 * being cleared.
	 */
	csrs CSR_STATUS, SR_IE
#endif
	/* n: todo */
#if defined(CONFIG_TRACE_IRQFLAGS) || defined(CONFIG_CONTEXT_TRACKING)
	/* Recover a0 - a7 for system calls */
	REG_L a0, PT_A0(sp)
	REG_L a1, PT_A1(sp)
	REG_L a2, PT_A2(sp)
	REG_L a3, PT_A3(sp)
	REG_L a4, PT_A4(sp)
	REG_L a5, PT_A5(sp)
	REG_L a6, PT_A6(sp)
	REG_L a7, PT_A7(sp)
#endif
	/* save the initial A0 value (needed in signal handlers) */
	/* n: 把a0保存到栈的PT_ORIG_A0这个位置，a0为什么要存都这里？(a0会作为系统调用的返回值，估计这里是为了做区分) */
	REG_S a0, PT_ORIG_A0(sp)
	/*
	 * Advance SEPC to avoid executing the original
	 * scall instruction on sret
	 */
	addi s2, s2, 0x4
	REG_S s2, PT_EPC(sp)
	/* Trace syscalls, but only if requested by the user. */
	/* n: 读一个thread_info里的flag，判断是否要走syscall trace */
	REG_L t0, TASK_TI_FLAGS(tp)
	andi t0, t0, _TIF_SYSCALL_WORK
	bnez t0, handle_syscall_trace_enter
check_syscall_nr:
	/* Check to make sure we don't jump to a bogus syscall number. */
	li t0, __NR_syscalls
	/* n: ni的意思是no implemented，下面可以看出如果系统调用号大于等于最大值了，就会进入这个函数 */
	la s0, sys_ni_syscall
	/*
	 * Syscall number held in a7.
	 * If syscall number is above allowed value, redirect to ni_syscall.
	 */
	bgeu a7, t0, 1f
	/* Call syscall */
	/*
	 * n: sys_call_table这个表定义在arch/riscv/kernel/syscall_table.c，又include
	 * 到了其他地方，最终系统调用都在linux/include/uapi/asm-generic/unistd.h
	 * 里定义。
	 *
	 * 注意riscv系统调用的参数使用a0-a7传递，a7里放系统调用号。
	 */
	la s0, sys_call_table
	slli t0, a7, RISCV_LGPTR
	add s0, s0, t0
	REG_L s0, 0(s0)
	/* n: 从这里返回，返回到ret_from_syscall, 之前的几个跳转返回，会返回到之前配置的ra，就是ret_from_exception */
1:
	jalr s0

ret_from_syscall:
	/* Set user a0 to kernel a0 */
	/* n: a0保存返回值，这里把a0存入内核栈保存的寄存器是为了后面恢复？*/
	REG_S a0, PT_A0(sp)
	/*
	 * We didn't execute the actual syscall.
	 * Seccomp already set return value for the current task pt_regs.
	 * (If it was configured with SECCOMP_RET_ERRNO/TRACE)
	 */
ret_from_syscall_rejected:
	/* Trace syscalls, but only if requested by the user. */
	REG_L t0, TASK_TI_FLAGS(tp)
	andi t0, t0, _TIF_SYSCALL_WORK
	bnez t0, handle_syscall_trace_exit

ret_from_exception:
	/* n: 把sstatus的值加载到s0里 */
	REG_L s0, PT_STATUS(sp)
	/* n: 关中断。处理异常的时候，除了ebreak异常，其他异常处理可能是开中断的 */
	csrc CSR_STATUS, SR_IE
#ifdef CONFIG_TRACE_IRQFLAGS
	call trace_hardirqs_off
#endif
#ifdef CONFIG_RISCV_M_MODE
	/* the MPP value is too large to be used as an immediate arg for addi */
	li t0, SR_MPP
	and s0, s0, t0
#else
	/* n: 恢复进入S mode之前的mode, 并保存到s0 */
	andi s0, s0, SR_SPP
#endif
	/* n: s0不是0，即不是user mode，那只恢复内核的上下文就好, 否则要先恢复用户态上下文 */
	bnez s0, resume_kernel

	/* n: 用户态和内核态恢复唯一的区别就在resume_userspace这一段 */
resume_userspace:
	/* Interrupts must be disabled here so flags are checked atomically */
	REG_L s0, TASK_TI_FLAGS(tp) /* current_thread_info->flags */
	/* n: todo: 哪里会配置这个值？(这个估计要结合调度看下) 定义在arch/riscv/include/asm/thread_info.h */
	andi s1, s0, _TIF_WORK_MASK
	bnez s1, work_pending

#ifdef CONFIG_CONTEXT_TRACKING
	call context_tracking_user_enter
#endif
	/* n: 这里把相当于sp向上移动，相当于退栈，释放保存寄存器上下文的栈空间 */
	/* Save unwound kernel stack pointer in thread_info */
	addi s0, sp, PT_SIZE_ON_STACK
	/* n: 把当前内核栈sp保存到task_struct */
	REG_S s0, TASK_TI_KERNEL_SP(tp)

	/*
	 * Save TP into the scratch register , so we can find the kernel data
	 * structures again.
	 */
	/*
	 * n: 每次离开内核的时候把当前进程的tp存入scratch寄存器。注意，只有进入
	 * 用户态的时候才这样，这里也和一开始的分析呼应。
	 */
	csrw CSR_SCRATCH, tp

restore_all:
#ifdef CONFIG_TRACE_IRQFLAGS
	REG_L s1, PT_STATUS(sp)
	andi t0, s1, SR_PIE
	beqz t0, 1f
	call trace_hardirqs_on
	j 2f
1:
	call trace_hardirqs_off
2:
#endif
	/* n: 注意，前面虽然已经"退栈"，但是sp的指向还没有变，这里依然可以继续使用sp */
	REG_L a0, PT_STATUS(sp)
	/*
	 * The current load reservation is effectively part of the processor's
	 * state, in the sense that load reservations cannot be shared between
	 * different hart contexts.  We can't actually save and restore a load
	 * reservation, so instead here we clear any existing reservation --
	 * it's always legal for implementations to clear load reservations at
	 * any point (as long as the forward progress guarantee is kept, but
	 * we'll ignore that here).
	 *
	 * Dangling load reservations can be the result of taking a trap in the
	 * middle of an LR/SC sequence, but can also be the result of a taken
	 * forward branch around an SC -- which is how we implement CAS.  As a
	 * result we need to clear reservations between the last CAS and the
	 * jump back to the new context.  While it is unlikely the store
	 * completes, implementations are allowed to expand reservations to be
	 * arbitrarily large.
	 */
	/* n: 处理和lr/sc相关的逻辑? */
	REG_L  a2, PT_EPC(sp)
	REG_SC x0, a2, PT_EPC(sp)

	csrw CSR_STATUS, a0
	csrw CSR_EPC, a2

	REG_L x1,  PT_RA(sp)
	REG_L x3,  PT_GP(sp)
	REG_L x4,  PT_TP(sp)
	REG_L x5,  PT_T0(sp)
	REG_L x6,  PT_T1(sp)
	REG_L x7,  PT_T2(sp)
	REG_L x8,  PT_S0(sp)
	REG_L x9,  PT_S1(sp)
	REG_L x10, PT_A0(sp)
	REG_L x11, PT_A1(sp)
	REG_L x12, PT_A2(sp)
	REG_L x13, PT_A3(sp)
	REG_L x14, PT_A4(sp)
	REG_L x15, PT_A5(sp)
	REG_L x16, PT_A6(sp)
	REG_L x17, PT_A7(sp)
	REG_L x18, PT_S2(sp)
	REG_L x19, PT_S3(sp)
	REG_L x20, PT_S4(sp)
	REG_L x21, PT_S5(sp)
	REG_L x22, PT_S6(sp)
	REG_L x23, PT_S7(sp)
	REG_L x24, PT_S8(sp)
	REG_L x25, PT_S9(sp)
	REG_L x26, PT_S10(sp)
	REG_L x27, PT_S11(sp)
	REG_L x28, PT_T3(sp)
	REG_L x29, PT_T4(sp)
	REG_L x30, PT_T5(sp)
	REG_L x31, PT_T6(sp)

	REG_L x2,  PT_SP(sp)

#ifdef CONFIG_RISCV_M_MODE
	mret
#else
	sret
#endif

/* n: 开抢占的时候，在恢复上下文之前要先处理下抢占 */
#if IS_ENABLED(CONFIG_PREEMPTION)
resume_kernel:
	/*
	 * n: 取thread_info里的preempt_count, 如果是非0，那么是禁止抢占的, 就直接
	 *    跳到restore_all, 不做后面的调度, 否则就继续往下走，查看是否需要调度，
	 *    需要做调度，就执行下调度。
	 *
	 *    所谓抢占就是一个线程执行被被动的中断，然后CPU上换另一个线程的上下文
	 *    执行，所以，一个线程主动放弃CPU，换另一个线程上CPU执行肯定不是抢占，
	 *    所以，抢占具体实施的点就是像在中断或异常的时候，处理完中断或异常，
	 *    然后调度一下，这个时候就可能调度其他的线程到CPU上跑。
	 *
	 *    内核里又加了一个禁止抢占的标记，就是上面preempt_count这个变量，当
	 *    这个变量是0的时候，是可以抢占的，当这个变量大于0，是不能抢占的。
	 *    这样，就有可能出现，中断处理完但是不能抢占的情况。
	 */
	REG_L s0, TASK_TI_PREEMPT_COUNT(tp)
	bnez s0, restore_all
	REG_L s0, TASK_TI_FLAGS(tp)
	andi s0, s0, _TIF_NEED_RESCHED
	beqz s0, restore_all
	call preempt_schedule_irq
	j restore_all
#endif

work_pending:
	/* Enter slow path for supplementary processing */
	la ra, ret_from_exception
	andi s1, s0, _TIF_NEED_RESCHED
	/* n: 异常或中断返回时主动调度 */
	bnez s1, work_resched
work_notifysig:
	/* Handle pending signals and notify-resume requests */
	csrs CSR_STATUS, SR_IE /* Enable interrupts for do_notify_resume() */
	move a0, sp /* pt_regs */
	move a1, s0 /* current_thread_info->flags */
	tail do_notify_resume
work_resched:
	tail schedule

/* Slow paths for ptrace. */
handle_syscall_trace_enter:
	move a0, sp
	call do_syscall_trace_enter
	move t0, a0
	REG_L a0, PT_A0(sp)
	REG_L a1, PT_A1(sp)
	REG_L a2, PT_A2(sp)
	REG_L a3, PT_A3(sp)
	REG_L a4, PT_A4(sp)
	REG_L a5, PT_A5(sp)
	REG_L a6, PT_A6(sp)
	REG_L a7, PT_A7(sp)
	bnez t0, ret_from_syscall_rejected
	j check_syscall_nr
handle_syscall_trace_exit:
	move a0, sp
	call do_syscall_trace_exit
	j ret_from_exception

END(handle_exception)

ENTRY(ret_from_fork)
	la ra, ret_from_exception
	tail schedule_tail
ENDPROC(ret_from_fork)

ENTRY(ret_from_kernel_thread)
	call schedule_tail
	/* Call fn(arg) */
	la ra, ret_from_exception
	move a0, s1
	jr s0
ENDPROC(ret_from_kernel_thread)


/*
 * Integer register context switch
 * The callee-saved registers must be saved and restored.
 *
 *   a0: previous task_struct (must be preserved across the switch)
 *   a1: next task_struct
 *
 * The value of a0 and a1 must be preserved by this function, as that's how
 * arguments are passed to schedule_tail.
 */
/*
 * n: sched/core.c里的context_switch会调用到这里。__switch_to这个函数的第一个入参
 * 是前task_struct, 后一个是要switch到的线程的task_struct。task_struct里有一个架
 * 构相关的结构thread_struct。riscv的定义在arch/riscv/include/asm/processor.h, 
 * TASK_THREAD_RA是这个结构在task_struct里的偏移，所以a3、a4得到thread_struct在
 * 切出、切入线程task_struct里thread_struct的地址。
 * 
 */
ENTRY(__switch_to)
	/* Save context into prev->thread */
	li    a4,  TASK_THREAD_RA
	add   a3, a0, a4
	add   a4, a1, a4
	/* n: 保存这些就够了，剩下的是caller save寄存器，已经保存到对应的栈了 */
	REG_S ra,  TASK_THREAD_RA_RA(a3)
	REG_S sp,  TASK_THREAD_SP_RA(a3)
	REG_S s0,  TASK_THREAD_S0_RA(a3)
	REG_S s1,  TASK_THREAD_S1_RA(a3)
	REG_S s2,  TASK_THREAD_S2_RA(a3)
	REG_S s3,  TASK_THREAD_S3_RA(a3)
	REG_S s4,  TASK_THREAD_S4_RA(a3)
	REG_S s5,  TASK_THREAD_S5_RA(a3)
	REG_S s6,  TASK_THREAD_S6_RA(a3)
	REG_S s7,  TASK_THREAD_S7_RA(a3)
	REG_S s8,  TASK_THREAD_S8_RA(a3)
	REG_S s9,  TASK_THREAD_S9_RA(a3)
	REG_S s10, TASK_THREAD_S10_RA(a3)
	REG_S s11, TASK_THREAD_S11_RA(a3)
	/* Restore context from next->thread */
	REG_L ra,  TASK_THREAD_RA_RA(a4)
	REG_L sp,  TASK_THREAD_SP_RA(a4)
	REG_L s0,  TASK_THREAD_S0_RA(a4)
	REG_L s1,  TASK_THREAD_S1_RA(a4)
	REG_L s2,  TASK_THREAD_S2_RA(a4)
	REG_L s3,  TASK_THREAD_S3_RA(a4)
	REG_L s4,  TASK_THREAD_S4_RA(a4)
	REG_L s5,  TASK_THREAD_S5_RA(a4)
	REG_L s6,  TASK_THREAD_S6_RA(a4)
	REG_L s7,  TASK_THREAD_S7_RA(a4)
	REG_L s8,  TASK_THREAD_S8_RA(a4)
	REG_L s9,  TASK_THREAD_S9_RA(a4)
	REG_L s10, TASK_THREAD_S10_RA(a4)
	REG_L s11, TASK_THREAD_S11_RA(a4)
	/* Swap the CPU entry around. */
	/* n: 交换两个task_struct里的thread_info里的cpu域段，cpu的语意是？*/
	lw a3, TASK_TI_CPU(a0)
	lw a4, TASK_TI_CPU(a1)
	sw a3, TASK_TI_CPU(a1)
	sw a4, TASK_TI_CPU(a0)
	/* The offset of thread_info in task_struct is zero. */
	/* n: tp指向新的task_struct */
	move tp, a1
	ret
ENDPROC(__switch_to)

#ifndef CONFIG_MMU
#define do_page_fault do_trap_unknown
#endif

	.section ".rodata"
	.align LGREG
	/* Exception vector table */
ENTRY(excp_vect_table)
	RISCV_PTR do_trap_insn_misaligned
	RISCV_PTR do_trap_insn_fault
	RISCV_PTR do_trap_insn_illegal
	RISCV_PTR do_trap_break
	RISCV_PTR do_trap_load_misaligned
	RISCV_PTR do_trap_load_fault
	RISCV_PTR do_trap_store_misaligned
	RISCV_PTR do_trap_store_fault
	RISCV_PTR do_trap_ecall_u /* system call, gets intercepted */
	RISCV_PTR do_trap_ecall_s
	RISCV_PTR do_trap_unknown
	RISCV_PTR do_trap_ecall_m
	RISCV_PTR do_page_fault   /* instruction page fault */
	RISCV_PTR do_page_fault   /* load page fault */
	RISCV_PTR do_trap_unknown
	RISCV_PTR do_page_fault   /* store page fault */
excp_vect_table_end:
END(excp_vect_table)

#ifndef CONFIG_MMU
ENTRY(__user_rt_sigreturn)
	li a7, __NR_rt_sigreturn
	scall
END(__user_rt_sigreturn)
#endif
```
```
        -+- task_struct           low address      <--- tp
         |  (thread_info)
         |
         |  struct thread_struct thread
 stack   |
 size    |
         |
         |
         |
         |
         |
         |
         |
         |
         |
         +-+-
         | |                                          ^
         | |  PT_SIZE_ON_STACK                        |
         | |                                          |
        -+-+- sp end                high address      |
```
 如上是一个线程对应的内核栈和task_struct在内存中的存储格式。栈底在sp end的位置，
 task_struct在sp end - 栈大小的位置，内核栈会分出PT_SIZE_ON_STACK的大小，用来保存
 中断异常时的寄存器上下文，PT_XX的宏就是这个区里的偏移。thread_info放在task_struct
 的一开始，用来暂存体系结构相关的一些数据。thread也在task_struct里，在__switch_to
 里使用，用TASK_THREAD_XX_XX表示相关的偏移。
