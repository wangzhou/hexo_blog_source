---
title: riscv特权级模型
tags:
  - riscv
  - 计算机体系结构
description: >-
  本文分析riscv特权级spec，尝试依照spec总结下riscv的特权级模型。有时直接看 spec，可能过于分散，本文分析的时候结合qemu tcg
  riscv的代码。基于riscv的spec 是20190608，基于的qemu版本是5.1.50。
abbrlink: 55311
categories:
---

特权级
-------

 这一节先梳理CPU运行的基本逻辑。CPU在PC处取指令，然后执行指令，指令改变CPU状态，
 这个状态周而复。指令改变CPU状态，被改变的有CPU的寄存器以及内存，被改变的寄存器
 包括PC，这样CPU就可以完成计算以及指令跳转。

 这个同步变化的模型可能被中断或者异常打断。实际上异常的整体运行情况也符合上面的
 模型，指令在执行的时候，会遇到各种各样的异常，当异常发生的时候，硬件通过一组寄存器
 向软件提供异常发生时的硬件状态，同时硬件改变PC的状态，使其指向一段预先放好的代码。

 中断异步的打断CPU正在执行的指令，通过一组寄存器向软件提供中断发生时的硬件状态，
 同时硬件改变PC的状态，使其指向一段预先放好的代码。

 当中断或异常发生的时候，硬件可以把CPU切换到一个正交的状态上，这些状态对于CPU的
 寄存器有着不同的访问权限。目前在riscv处理器上，有U, S, M等CPU模式(本文先不看虚拟化
 相关的CPU状体)，它们的编码分别是：0，1，3。

寄存器以及访问方式
-------------------

 CPU有通用寄存器，也有系统寄存器，其中系统寄存器也叫CSR，control system register。
 一般指令可以访问通用寄存器，但是只有特殊的指令才能访问系统寄存器，在riscv上，有
 一堆这样的csr指令。这些指令以csrxxx的方式命名，通过寄存器编号做各种读写操作。

M mode寄存器
-------------

 M mode下的寄存器大概可以分为若干类，一类是描述机器的寄存器，一类是和中断异常以及
 内存相关的寄存器，一类是时间相关的寄存器，另外还有PMU相关的寄存器。

 描述机器的寄存器有：misa，mvendorid，marchid，mimpid，mhartid。从名字就可以看出，
 这些寄存器是机器相关特性的描述，其中最后一个是core的编号。misa描述当前硬件支持的特性，
 riscv很多特性是可选的，这个寄存器就相当于一个能力寄存器。

 中断异常相关的寄存器有：mstatus, mepc, mtvec, mtval, medeleg, mideleg, mie, mip，mcause。
 S mode的相关寄存器和这些寄存器基本一样，只是以s开头。这些寄存器我们可以用中断异常
 处理的流程把他们串起来，在下面S mode里描述。比较特殊的是medeleg/mideleg寄存器，默认
 情况下riscv的中断和异常都在M mode下处理，可以通过配置这两个寄存器把异常或中断委托
 在S mode下处理。

 内存管理相关的寄存器有: PMP相关寄存器。

 时间管理相关的寄存器：mtime，mtimecmp。CPU内时钟相关的寄存器。

S mode寄存器
-------------

 S mode存在和中断异常以及内存相关的寄存器。我们主要从S mode出发来整理梳理下。

 中断异常相关的寄存器有：sstatus, sepc, stvec, stval, sie, sip，scratch，scause。
 内存管理相关的寄存器有: satp

 sstatus是机器相关的全局状态寄存器。riscv上，虽然各个mode的status寄存器是独立，但是
 各个mode的status的各个域段是正交的，基本上是每个域段在每个mode上都有一份。

  SIE   SPIE   SPP

 SIE表示全局中断的使能情况，SPIE表示特权级切换之前全局中断的使能情况，SPP表示特权级
 切换之前CPU所处在的mode。其他的域段，后续遇见具体使用场景可以补充上来。

 sie是riscv上定义的各种具体中断的使能状态，每种中断一个bit。riscv定义的中断可以
 参考riscv的spec。

 sip是对应各种中断的pending状态，每种中断一个bit。

 stvec是存放中断异常的跳转地址，当中断或异常发生时，硬件在做相应状态的改动后，就
 直接跳到stvec保存的地址，从这里取指令执行。基于这样的定义，软件可以把中断异常向量表
 的地址配置到这个寄存器里，中断或者异常的时候，硬件就会把PC跳到中断异常向量表。
 当然软件也可以把其他的地址配置到stvec，借用它完成跳转的功能。

 sepc是S mode exception PC，就是硬件用来给软件报告发生异常或者中断时的PC的，当异常
 发生时，sepc就是异常指令对应的PC，当中断发生的时候，sepc是被中断的指令，比如有A、B
 两条指令，中断发生在AB之间、或者和B同时发生导致B没有执行，sepc保存B指令的PC。

 scause报告中断或异常发生的原因，当这个寄存器的最高bit是1时表示中断，0表示发成的
 是异常。

 stval报告中断或异常的参数，当发生非法指令异常时，这个寄存器里存放非法指令的指令编码，
 当发生访存异常时，这个寄存器存放的是被访问内存的地址。

 scratch寄存器是留给软件使用的一个寄存器。Linux内核使用这个寄存器判断中断或者异常
 发生的时候CPU是在用户态还是内核态，当scratch是0时，表示在内核态，否则在用户态。

 satp是存放页表的基地址，riscv内核态和用户态分时使用这个页表基地址寄存器，这个寄存器
 的最高bit表示是否启用页表。如果启用页表，硬件在执行访存指令的时候，在TLB没有命中
 时，就会通过satp做page table walk，以此来找虚拟地址对应的物理地址。

 到此为止，中断或异常发生时需要用到的寄存器都有了。我们下面通过具体的中断或者异常
 流程把整个过程串起来。

中断或异常流程
---------------

 中断异常向量表的地址会提前配置到stvec。medeleg/mideleg寄存器需要提前配置好，把
 需要在S mode下处理的异常和中断对应的bit配置上。

 当中断或异常发生的时候，硬件把SIE的值copy到SPIE，当前处理器mode写入SPP，SIE清0。
 sepc存入异常指令地址、或者中断指令地址，scause写入中断或者异常的原因，stval写入
 中断或异常的参数，然后通过stvec得到中断异常向量的地址。随后，硬件从中断异常向量
 地址取指令执行。(可以参考qemu代码：qemu/target/riscv/cpu.c riscv_cpu_do_interrupt函数)

 以Linux内核为例，riscv的中断异常处理流程，先保存中断或异常发生时的寄存器上下文，
 然后根据scause的信息找见具体的中断或异常处理函数执行。具体的软件流程分析可以参考
 本站的Linux内核riscv entry.S分析。

 当异常或中断需要返回时，软件可以使用sret指令，sret指令在执行的时候会把sepc寄存器
 里保存的地址作为返回地址，使用SPP寄存器里的值作为CPU的mode，把SPIE的值写入SIE，
 SPIE写1，SPP写入U mode编号。所以，在调用sret前，软件要配置好sepc、SPP、SPIE寄存器
 的值。

特权级相关的指令
-----------------

 异常相关的指令：ecall、ebreak、sret、mret。ecall和ebreak比较相似，就是使用指令触发
 ecall或者ebreak异常。ecall异常又可以分为ecall from U mode、ecall from S mode，分别
 表示ecall是在CPU U mode还是在S mode发起的。在Linux上，从U mode发起的ecall就是一个
 系统调用，软件把系统调用需要的参数先摆到系统寄存器上，然后触发ecall指令，硬件依照
 上述的异常流程改变CPU的状态，最终软件执行系统调用代码，参数从系统寄存器上获取。
 Linux上，特性体系构架的系统调用ABI是软件约定的。sret、mret只是从S mode或者从M mode
 返回的不同，其他的逻辑是相同的。

 机器相关的指令：reset、wfi。reset复位整个riscv机器。wfi执行的时候会挂起CPU，直到
 CPU收到中断，一般是用来降低功耗的。

 内存屏障相关的指令：sfence.vma。sfence.vma和其他体系构架下的TLB flush指令类似，
 用来清空TLB，这个指令可以带ASID或address参数，表示清空对应参数标记的TLB，当ASID
 或者address的寄存器是X0时，表示对应的参数是无效的。
