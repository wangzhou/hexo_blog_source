---
title: qemu tcg模拟原子指令
date: 2022-01-26 23:11:35
tags: [qemu, 虚拟化]
description: "qemu tcg中原子指令学习的一个速记，以riscv为平台。"
categories:
---

```
/* target/riscv/insn_trans/trans_rva.c.inc */
trans_amoadd_w
  -> gen_amo
    -> tcg_gen_atomic_fetch_add_tl
      -> tcg_gen_atomic_fetch_add_i64
```
如上最后一个函数的定义在：tcg/tcg-op.c
```
#define GEN_ATOMIC_HELPER(NAME, OP, NEW)                                \
static void * const table_##NAME[(MO_SIZE | MO_BSWAP) + 1] = {          \
    [MO_8] = gen_helper_atomic_##NAME##b,                               \
    [MO_16 | MO_LE] = gen_helper_atomic_##NAME##w_le,                   \
    [MO_16 | MO_BE] = gen_helper_atomic_##NAME##w_be,                   \
    [MO_32 | MO_LE] = gen_helper_atomic_##NAME##l_le,                   \
    [MO_32 | MO_BE] = gen_helper_atomic_##NAME##l_be,                   \
    WITH_ATOMIC64([MO_64 | MO_LE] = gen_helper_atomic_##NAME##q_le)     \
    WITH_ATOMIC64([MO_64 | MO_BE] = gen_helper_atomic_##NAME##q_be)     \
};                                                                      \
void tcg_gen_atomic_##NAME##_i32                                        \
    (TCGv_i32 ret, TCGv addr, TCGv_i32 val, TCGArg idx, MemOp memop)    \
{                                                                       \
    if (tcg_ctx->tb_cflags & CF_PARALLEL) {                             \
        do_atomic_op_i32(ret, addr, val, idx, memop, table_##NAME);     \
    } else {                                                            \
        do_nonatomic_op_i32(ret, addr, val, idx, memop, NEW,            \
                            tcg_gen_##OP##_i32);                        \
    }                                                                   \
}                                                                       \
void tcg_gen_atomic_##NAME##_i64                                        \
    (TCGv_i64 ret, TCGv addr, TCGv_i64 val, TCGArg idx, MemOp memop)    \
{                                                                       \
    if (tcg_ctx->tb_cflags & CF_PARALLEL) {                             \
        do_atomic_op_i64(ret, addr, val, idx, memop, table_##NAME);     \
    } else {                                                            \
        do_nonatomic_op_i64(ret, addr, val, idx, memop, NEW,            \
                            tcg_gen_##OP##_i64);                        \
    }                                                                   \
}
```
do_atomic_op_i64里会调用gen_helper_atomic_add_xxx，这个函数的定义在:
accel/tcg/atomic_template.h
```
#define GEN_ATOMIC_HELPER(X)                                        \
ABI_TYPE ATOMIC_NAME(X)(CPUArchState *env, target_ulong addr,       \
                        ABI_TYPE val, MemOpIdx oi, uintptr_t retaddr) \
{                                                                   \
    DATA_TYPE *haddr = atomic_mmu_lookup(env, addr, oi, DATA_SIZE,  \
                                         PAGE_READ | PAGE_WRITE, retaddr); \
    DATA_TYPE ret;                                                  \
    atomic_trace_rmw_pre(env, addr, oi);                            \
    ret = qatomic_##X(haddr, val);                                  \
    ATOMIC_MMU_CLEANUP;                                             \
    atomic_trace_rmw_post(env, addr, oi);                           \
    return ret;                                                     \
}
```
可以看到，里面还是使用的host平台上的基本的原子语义函数做的。

如果需要模拟多条指令拼起来的原子指令，我们就考虑用锁保护。要保护的对象是内存的状态。
之所以需要保护，是多CPU可能会去改相同的内存位置。qemu使用一个线程模拟一个CPU，
所以一个CPU对本CPU的寄存器的更新总是顺序的，所以CPU的寄存器状态是不需要做互斥的。

对于无法映射到host上原子指令的情况，其实qemu里已经做了处理，我们也可以直接使用
qemu中的方式处理。我们可以参考qemu对i386 cmpxchg16b指令的处理：qemu/target/i386/tcg/mem_helper.c
```
helper_cmpxchg16b
  +-> cpu_loop_exit_atomic
    +-> cpu->exception_index = EXCP_ATOMIC;
    +-> cpu_loop_exit_restore(cpu, pc);
      +-> cpu_restore_state
      +-> cpu_loop_exit(cpu);
```
如上，把CPU的状态设置为atomic异常，回退当前guest PC，这个使得下次再进来的时候可以
使指令再次执行。最后用长跳转跳出整个tb翻译执行的大循环。可以从
accel/tcg/tcg-accel-ops-mttcg.c中的CPU线程代码看相关调用：
```
mttcg_cpu_thread_fn
  +-> tcg_cpus_exec
  +-> cpu_exec_step_atomic
     ...
```
可以看到cpu_exec_step_atomic里有tb的翻译执行的小循环。这里需要注意的地方有，tb
翻译执行是在一个互斥区里，执行tb翻译执行之前把这个tb配置成了只容许有一条guest指令，
这样做是为了使临界区尽量小。相应的cmpxchg16b翻译执行跑两遍，第一遍触发发原子异常，
第二遍跑到同样的位置会进入一个无锁的实现里执行一遍：target/i386/tcg/translate.c:
gen_helper_cmpxchg16b_unlocked，控制进哪个分支的逻辑是tb cflags的CF_PARALLEL，
在进入cpu_exec_step_atomic的时候会把这个标记为去掉，翻译的时候就会进入相应的代码，
产生相应的tb，如果是直接lookup执行这个tb，注意tb lookup的参数里也包含了
tb的cflags。
