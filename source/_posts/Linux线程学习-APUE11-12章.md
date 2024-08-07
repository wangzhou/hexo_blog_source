---
title: Linux线程学习- APUE11/12章
tags:
  - Linux用户态编程
  - APUE
description: Linux线程学习- APUE11/12章笔记
categories: read
abbrlink: 76f6c9bc
date: 2021-06-19 09:43:55
---

线程的创建和退出
-------------------

 Linux系统下线程和进程的概念是比较模糊的。一般来说，线程是调度的单位，进程是资源
 的单位。本质上来说，内核看到都是一个个线程，但是线程之间可以通过共享资源，相互
 之间又划分到不同的进程里。

 从内核视角上看各种不同的ID会比较清楚。之前的[这篇文章](https://wangzhou.github.io/APUE学习笔记-第九章/)对进程ID, 线程ID，进程组，
 线程组，会话已经有了简单描述。

 在一个进程的多个线程里调用getpid，这个是一个c库对getpid系统调用的封装，所以这
 个函数返回调用线程的进程ID, 这个ID其实就是这个线程组的线程组ID，也是这个线程组
 的主线程的内核pid，这个从kernel代码kernel/sys.c getpid的系统调用可以看的很清楚。
 使用gettid系统调用可以得到一个线程的对应内核pid。

 而所谓pthread库的pthread_self()得到的只是pthread库自定义的线程ID，这个东西和上
 面内核里真正的各种ID是完全不同的。

 线程的创建用pthread_create, 用的系统调用是clone。我们考虑线程结束的方法，线程执
 行完线程函数后自己会退出，这其中也包括线程函数自己调用pthread_exit把自己结束。
 线程也可以被进程中的其他线程取消，取消一个线程使用的函数是：pthread_cancel(pthread_t tid)。
 用strace跟踪下，可以发现pthread_cancel中的系统调用是tgkill(tgid, tid, SIGRTMIN),
 这个系统调用向线程组tgid里的tid线程发送SIGRTMIN信号。需要注意的是，pthread_cancel
 并不是同步的终止线程，而是给线程发送一个终止信号就返回，线程的终止行为还取决于
 线程自身的配置，首先线程自己可以配置是否响应pthread_cancel，默认情况下是可以响应
 的，可以通过pthread_setcancelstate配置这个状态。在这个基础上，线程需要在执行到
 一些特定的函数才能停下来，这些函数叫做取消点，通过pthread_setcanceltype可以配置
 对应的行为，PTHREAD_CANCEL_DEFERRED是在取消点cancel掉线程，默认是这样的配置，
 PTHREAD_CANCEL_ASYNCHRONOUS是立即cancel掉线程。

 线程可以用pthread_jorn在阻塞等待相关线程结束，并且得到线程结束所带的返回值。

 线程可以注册退出的时候要调用的函数。使用pthread_cleanup_push, pthread_cleanup_pop。
 注册的函数在线程调用pthread_exit或者是线程被pthread_cancel的时候执行，线程正常
 执行结束时不执行。用strace -f跟踪进程中所有线程的系统调用可以发现，对应的系统调用
 是set_robust_list。

线程同步的锁
---------------

 线程之间共享变量的时候需要加锁，以pthread库为例，我们有pthread_mutex, pthead_spinlock,
 pthread读写锁。此外我们还可以自己实现锁，这个的好处是不受具体线程库的限制，
 不好的地方是，我们自己实现的锁一定没有pthread库实现的性能高。如下的测试代码中
 比较了pthread mutex/spinlock，以及自己实现的spinlock的性能:
 https://github.com/wangzhou/tests/blob/master/lock_test/test.c

 apue这里的例子11-5很好的展示了一个引用计数的实现方式。 
 一定要避免A-B/B-A这种交叉加锁的情况，但是注意这里只是加锁，减锁的顺序可以随意。

 如果一定要出现交叉加锁的情况，要做成如果第二把锁没有抢到(所以，第二把锁要用try
 锁)，那么已经锁上的第一把锁也要放开，要留出一条通道把可能的死锁情况放过去。过
 一段时间再锁上第一把锁，再try第二把锁。

 11-6的代码对于交叉加锁的处理方式是，需要上第二把锁的时候，直接放开第一把锁，
 然后先上第二把锁，再上第一把锁，由于这个时候有个时间的空隙，可能不满足之前的
 条件了，所以要再check下之前需要第二把锁的条件。当然apue举这个例子的目的这里是
 为了说明可以简化加锁，需要在复杂度和性能之间权衡。
 
条件变量和信号量
-------------------

 条件变量和信号量都是为了线程/进程之间同步用的，提供了线程/进程之间相互等待、
 通知的机制。简单写一个测试:
 https://github.com/wangzhou/tests/blob/master/pthread/thread.c
 https://github.com/wangzhou/tests/blob/master/pthread/sem.c
 在thread.c和sem.c里都会出现同一个问题，如果不在enqueue_msg函数里unlock和发信号
 之间增加时延，会出现发送线程一直持有锁，接收线程得不到锁从而无法及时接收的情况。
 测试发现，在unlock和发送信号(比如thread.c的pthread_mutex_unlock和pthread_cond_signal)
 之间增加usleep(1)的时延就可以使如上的这种情况不出现。

线程控制
-----------

 pthread库还有很多控制接口，这些接口可以改变如上接口的语义，或者增减新的功能。
 比如，每种锁的初始化接口都可以控制这些锁在嵌套加锁、不对称加锁等的语义，遇到
 这样的情况，可以配置成容许或者是返回错误值; 可以配置锁在线程之间还是进程之间
 共享; 可以配置线程栈的大小等; 可以申请线程的私有数据; 可以使得一个函数只执行
 一次。

 简单写一个线程私有数据的测试:
 https://github.com/wangzhou/tests/blob/master/pthread/priv.c
 可以看到，pthread_key_t key1的create没有和线程绑定。基本的逻辑是，pthread_key_create
 可以在任意一个线程里，pthread_setspecific和pthread_getspecific是在线程里对应的，
 在线程A里set的数据，可以在线程A里通过get拿到，但是在另外的一个线程B里对全局的
 key1使用pthread_getspecific只能拿到NULL。apue里的例子把pthread_key_create放到
 了每个线程里，为了只调用一次pthread_key_create, 还介绍只跑一个函数一次的接口：
 int pthread_once(pthread_once_t *initflag, void (*initfn)(void))，可以想象这个
 函数的实现是先上锁，然后检查initflag，initflag没有置上已经跑过的flag就调用initfn，
 如果已经跑过就不用跑了。这个只跑一次的接口在某些库的设计里是很有用的，比如一个
 库需要在进程使用的时候初始化一次，这里就可以用相似的设计或者直接用这个API。

 关于信号可以参考[这里](https://wangzhou.github.io/Linux信号笔记/)，线程和信号的细节内容需要另外描述。

 线程在fork系统调用上遇到的问题，我们自己写一个独立的应用的时候比较难遇到，因为
 全局受自己的控制，我们只要一开始fork进程，做好规划就好。但是，当我们要写一个库，
 这个库可以被其他上层代码调用，我们的库里又要起独立的线程时，这个问题就会出来。
 这个问题的本质是自己写的库里向上层export出了全局的资源，比如，如果库里只出函数
 接口，就没有问题，所有库里的资源都在函数的栈上，但是，如果库里有了全局资源，相当
 于，我们的库向调用进程里增加了全局的资源，调用进程将需要考虑这些全局资源(当然库
 可以向上层的调用者提出诉求或者接口)。具体看，子进程会从父进程那里继承:
 全局变量(子进程在没有写之前，如果去读这个变量，依然得到的是父进程里的值，如果拿
 这个值去做判断就有可能出错)、各种锁、信号量和条件变量。如果fork出来的进程不是
 调用exec系列函数去执行一个新的程序，那么子进程里拥有和父进程一样的锁、信号量和
 条件变量。可以通过pthread_atfork提前挂上fork时候的回调函数进行处理，即一定是在
 prepare回调里先获取所有的锁，在parent、child里再释放所有的锁，注意这里的获取释放
 的操作和父进程里的可能获取锁的行为做了互斥。
 
 preaed/pwrite可以保证多线程对一个fd的操作是原子的。
