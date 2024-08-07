---
title: Linux内核spinlock实现分析
tags: [Linux内核]
description: >-
  本文简介Linux内核里spinlock实现逻辑，这里会总结下spinlock各种实现的基础逻辑。
  代码分析基于内核v6,5-rc5，涉及到和体系结构相关的部分，我们采用ARM64来分析。
  知乎上有一个系列的文章已经把这块讲的很好，它的位置在这里：https://zhuanlan.zhihu.com/p/100546935
abbrlink: 60548
date: 2023-08-02 14:40:48
categories:
---

基本逻辑
---------

spinlock的基本行为就是各个CPU core去互斥的处理一个数据，加锁后只有获取锁的core
可以处理数据，解锁后，CPU core又可以去获取锁。整个过程关闭调度。

spinlock最直白的实现方法是多核间用原子的读修改写指令抢一个标记，这个标记原始值
是0，0表示没有core占有这个锁，当一个core原子的检测到这个标记是0，并修改成1时，这个
core占有锁，其它core做这个检测时，这个标记是1，读修改写的原子指令不生效。

这个原子指令大概是这样：CAS(int *val, int old, int new)，如果和old和*val相等，才
把new写入val的地址，把*val的老值保存到new里。

用最直白的逻辑写出的锁实现类似:
```
struct self_spinlock {
	__u32 lock;
};

static inline void self_spinlock(struct self_spinlock *lock)
{
	while (__atomic_test_and_set(&lock->lock, __ATOMIC_ACQUIRE))
		while (__atomic_load_n(&lock->lock, __ATOMIC_RELAXED))
			;
}

static inline void self_unspinlock(struct self_spinlock *lock)
{
	__atomic_clear(&lock->lock, __ATOMIC_RELEASE);
}
```
这样的锁有两个问题：1. 锁的请求顺序和实际获得锁的顺序不一致，因为上面本质上还是
多个core在无序的争抢标记位；2. 多核之间cache会相互影响。
```
   +------+    +------+    +------+    +------+
   | CPU0 |    | CPU1 |    | CPU2 |    | CPU3 |
   +------+    +------+    +------+    +------+
      v           v           v           v
   +------+    +------+    +------+    +------+
   |cache0|    |cache1|    |cache2|    |cache3|
   +------+    +------+    +------+    +------+
            \      \          /     /
             \  +----------------+ /
              \ | Flag in memory |/
                +----------------+
```
对于第二个问题，我们展开看下，在有cache的系统里，系统大概的样子如上，如果CPU0占有
锁，cach0为1(cache0/1/2/3是Flag在各个core上的cache)，CPU1/2/3的cache也会在各个core
读Flag时被设置为1，CPU0释放锁的时候，cache0被写成0，同时CPU1/2/3上对应的cach被无效
化，随后哪个core抢先把Flag写成1，对应的cache就是1。后面重复之前的逻辑。可以看出，
本来在unlock core和lock core之间的通行行为被扩展到了所有参与竞争锁的core，不但锁的
请求顺序和实际获得锁的顺序不一致，而且做了很多无用功。

ticket spinlock
----------------

tick spinlock的实现在ARMv6的内核代码里还有保留，具体的路径在linux/arch/arm/include/asm/spinlock.h。
这里只把它核心的逻辑提取出来。

ticket spinlock锁本身的数据结构如下：
```
struct __raw_tickets {
	u16 next;
	u16 owner;
}
```
获取锁的行为就是原子的增加next值，然后作为自己的ticket，拿着自己的ticket一直和owner
做对比，看看是不是轮到自己拿锁，释放锁的行为就是增加owner的值。
```                  
                        +------+    +------+    +------+    +------+
                        | CPU0 |    | CPU1 |    | CPU2 |    | CPU3 |
                        +------+    +------+    +------+    +------+

        lock               ----------------------------------------> 
  local next:              0           1       +---2---+   +---3---+
                                               |       |   |       |
                                     owner++   ^       v   ^       v
      unlock                         ----->    |       |   |       |
owner in __raw_tickets:    0           1       +-owner-+   +-owner-+
```
如上是一个ticket spinlock的示意图，CPU2/3现在在等待锁，CPU1在释放锁。试图获取锁
的CPU原子的对next加1并在本地保存一个加1后的本地next，作为自己的ticket，释放锁的
CPU把锁里的owner值加1，试图获取锁的CPU，一直在拿自己的ticket和owner做比较，如果
ticket和owner相等自己就拿到了锁。

可以看出，ticket spinlock解决了上面的问题1，但是没有解决问题2，因为竞争锁的core
必须时刻拿着自己的ticket和owner做对比，实际上谁得到锁这个信息只要依次传递就好。

MCS spinlock
-------------

基于以上的认识，后来人们又提出了MCS锁，这个名字是用提出这种锁算法的两个人的名字
命名的。

其实从上面ticket spinlock的图上我们已经可以大概看出来要怎么做，就是把每次lock搞
一个锁的副本出来，然后把这些副本用链表链接起来，加锁时还是spin在自己的副本上，解锁
时顺着链表依次释放锁。

大概的示意图如下:
```
                                               ------------+
                                               | lock tail |
                                             / +-----------+
         +------+    +------+    +------+   /+------+
         | CPU0 |    | CPU1 |    | CPU2 |  / | CPU3 |
         +------+    +------+    +------+ /  +------+
                                       1 /        +-----+
                                        /         |     | 3 spin on owner
         +------+    +------+    +------+    +------+   |
         | owner|    | owner|    | owner| 2  | owner|<--+
         | next |--->| next |--->| next |--->| next |
         +------+    +------+    +------+    +------+
```
如上所示，加锁就是找见当前锁链表结尾(步骤1)，把要加的锁节点挂在上面(步骤2)，然后
就spin在自己的owner标记上等待(步骤3)。

解锁就是把当前锁节点的下一个节点的owner配置成1，这样，spin的core检测到owner为1，
就知道现在自己拥有锁了。

MCS锁可以解决上面的两个问题，但是占用的内存比较多了。我们考虑MCS的实际实现问题，
一个MCS锁需要一个lock tail以及每个core上的mcs node结构，占用内存比较大，实际上
在现在的Linux内核中只有MCS node的定义，并没有MCS锁的实现，只所以MCS的定义还在，
是因为qspinlock里要复用MCS node的定义。

qspinlock
----------

当前的内核里，各个体系构架下的spinlock基本上都使用了qspinlock的实现。我们重点分析
下qspinlock的基本逻辑和代码实现。

spinlock_t的定义在linux/include/linux/spinlock_types.h，封装的数据结结构是struct raw_spinlock，
进一步到arch_spinlock_t, 对于支持qspinlock的构架，arch_spinlock_t的定义就是struct qspinlock。

我们只看小端下qspinlock的定义，基于此分析lock和unlock的逻辑细节。
```
typedef struct qspinlock {
	union {
		atomic_t val;
		struct {
			u8	locked;
			u8	pending;
		};
		struct {
			u16	locked_pending;
			u16	tail;
		};
	};
} arch_spinlock_t;
```

kernel/locking/qspinlock.c中一个CPU上静态分配一个size是4的struct qnode数组，每个
qnode元素是struct mcs_spinlock的封装，每个core上的一个qnode对应一种内核上下文，
所以4个qnode分别对应task, hardirq, softirq, NMI内核上下文。

我们再整理下qspinlock的相关数据结构。如下是qspinlock的位域结构, val状态按照如下描述
(tail, pending, locked):
```
(tail,          pending, locked)
<----16bit----><----16bit-----> 
```
每个qspinlock都会有一个这样的结构，所有qspinlock复用每个core上的mcs node，mcs node
封装成qnode后静态定义如下：
```
static DEFINE_PER_CPU_ALIGNED(struct qnode, qnodes[MAX_NODES]);
```
也就是每个core的per-CPU数据区上，都有一个size是4的qnode的数组。

我们先看qspinlock的基本逻辑，具体代码的分析以代码注释的形式放在最后。qspinlock和
MCS核心的不同在于，qspinlock把锁占用与否的信息放到了qspinlock的locked域段，锁的
排队信息放到了pending和mcs链表。

如果是第一个core进来，就直接把locked set 1:
```
 +-------+------------+----------+
 | tail  |  pending 0 | locked 1 |
 +-------+------------+----------+
```

如果已经有一个core占有锁，就把pending set 1，并在locked上spin等待：
```
                              +------+
                              |      |
 +-------+------------+----------+   |  spin check locked
 | tail  |  pending 1 | locked 1 |<--+
 +-------+------------+----------+
```

如果已经有一个core占有锁并且一个core在排队，就在mcs node上排队，并在pending/locked
上spin等待，直到pending也释放锁，才占有锁。 我们把第二个排队节点，也就是这里的mcs node0
的处理展开看下，这一节描述的是mcs node0抢锁的逻辑。
```
               +-----------+---------+
               |           |         |
 +-------+------------+----------+   |
 | tail  |  pending 1 | locked 1 |<--+
 +-------+------------+----------+
     |
     v
 +-----------+
 | mcs node0 |
 +-----------+
```

如果系统里有两个节点在排队，就是如上第三个core还在mcs node0排队的情况，这时如果
又有core想占有锁，那么它们都在各自的mcs node上spin，并逐个加入mcs node的链表：
```
                     +-----------+---------+
                     |           |         |
       +-------+------------+----------+   |
       | tail  |  pending 1 | locked 1 |<--+
       +-------+------------+----------+
               \          +------+
                v         |      |
 +-----------+   +-----------+   |
 | mcs node0 |-->| mcs node1 |<--+
 +-----------+   +-----------+
     cpu0            cpu3
```
如上，cpu3也想占有锁，那么cpu3上的mcs node被加到mcs node链表上spin等待。这时，系统
里cpu0也在等待锁，如上它等的是pending/locked为0，还有两个cpu和这个锁有关系，那就
当前占有锁的cpu，和在pending位置的cpu，其中pending位置的cpu在locked上spin等待。

当cpu0成功占有锁时，cpu0会解开cpu3的spin，这时cpu3的状态成为了之前cpu0的状体，cpu3
开始做pending/locked的spin等待。

qspinlock的核心逻辑大概就是这样，代码里的其它细节处理在具体代码分析里看吧。可以
看到一个core上相同种类上下文的两把锁是复用对应mcs node的，比如，cpu3上一段代码流程
里先后上两把不同的锁lock1/lock2，lock1如果在mcs node1上排队，lock2上锁的代码还没有
执行到，等到cpu3得到了lock1，cpu3上的mcs node1的位置已经释放了，因为cpu3得到lock1
这个信息以经被记录到lock1 qspinlock的locked域段。

如下是qspinlock加锁函数的代码分析，我们忽略了和虚拟化有关的PV相关的代码:
```
/* linux/include/asm-generic/qspinlock.h */
static __always_inline void queued_spin_lock(struct qspinlock *lock)
{
	int val = 0;

	if (likely(atomic_try_cmpxchg_acquire(&lock->val, &val, _Q_LOCKED_VAL)))
		return;

	queued_spin_lock_slowpath(lock, val);
}
```
如果val是0，对lock->val原子写入_Q_LOCKED_VAL(0x1)，否则把lock->val的值写入val，
有写入lock->val函数返回true。可见对于没有写入的slowpath拿到lock->val当前状态的值，
基于此在slowpath里做处理。如果写入成功val的状态变成(0, 0, 1)。这里acquire的语意是
后续操作不能越过当前操作先执行。

这里release的语意是之前的指令不能乱序到当前操作之后执行。
```
static __always_inline void queued_spin_unlock(struct qspinlock *lock)
{
	/*
	 * unlock() needs release semantics:
	 */
	smp_store_release(&lock->locked, 0);
}
```

qspinlock的加锁逻辑的代码细节分析直接写到代码注释里了, 在[这个位置](https://github.com/wangzhou/linux/blob/bc23a8bd93823eae0cc92dc1f9668558cd4d14f3/kernel/locking/qspinlock.c#L317)
