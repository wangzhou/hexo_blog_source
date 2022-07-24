---
title: Linux kernel kthread
tags:
  - Linux内核
description: This doc shares the basic logic of kthread usage in Linux kernel.
abbrlink: 71cebe4e
date: 2021-06-28 23:55:34
categories:
---

Linux kernel has kthread_* APIs to create/stop thread in kernel.
You can use kthread_run to create a thread and put it into running.
Normally, after thread function running over, created thread will stop. However,
what is the logic of the stop of thread. We can write a small driver to test it.

In test case 1 beblow, we create a kthread which is alway running. We will find
that kthread_stop can not stop it.

In test case 2 below, we use kthread_stop to stop a thread which is created
righ now. You can find that the "created thread" will never put into running,
and kthread_stop will return -4.

In test case 3 below, created thread will use thread_should_stop to check
if it should stop. If it finds it should stop, it will go to return by itself.
This test case can work well.

So the logic of thread stop should be:

 1. If thread function is finished, related thread will stop.

 2. If thread function is not finished, there should be another code who tells
    the thread that others want it to be stopped. Other code uses kthread_stop()
    to set related flag for the thread. This is not an async message, which
    means thread should check this flag to find if others want it to stop. If
    thread does not to check this flag, it will be running alway. So it seems
    even kernel schedule will not stop a thread whose "should stop flag" is
    already set by "other code".

    So normally you can see the usage in test case 3. Or we can do thread
    function, and at the end of function wait "other code" to stop it like:
```
	kthread_function ()
	{
		ret = do_thread_job();		

		while (!kthread_should_stop())
			schedule();

		return ret;
	}
```
    In this way, "other code" can get the return value of thread.

 3. We can not use kthread_stop right after the creation of a thread, it seems
    kernel thinks that maybe we do not want the thread to be put into running.

    (to do: when can we use kthread_stop after kthread_run?)

 4. Last but not least, as always mentioned in other blogs, kthread_stop should
    be called when thread is running.

Test driver:
```
#include <linux/kernel.h>
#include <linux/kthread.h>
#include <linux/module.h>
#include <linux/delay.h>

MODULE_LICENSE("Dual BSD/GPL");
static int case_num = 1;
module_param(case_num, int, S_IRUGO);

static int thread_fun(void *date)
{
	unsigned long i;

	while (1) {
		i++;
	}

	return 0;
}

/* test if kthread_stop can stop a thread right now */
static int test_kthread_stop_a_thread(void)
{
	struct task_struct *thread;

	thread = kthread_run(thread_fun, NULL, "kthread_test");
	if (IS_ERR(thread)) {
		pr_info("test: fail to create a kthread\n");
		return -EPERM;
	}

	msleep(400);

	kthread_stop(thread);

	return 0;
}

static int thread_hello(void *date)
{
	pr_info("hello world!\n");
	
	while (!kthread_should_stop())
		schedule();

	return 0;
}

/* test to find when to call thread_stop */
static int test_kthread_stop_righ_after_run(void)
{
	struct task_struct *thread;

	thread = kthread_run(thread_hello, NULL, "kthread_hello");
	if (IS_ERR(thread)) {
		pr_info("test: fail to create a kthread\n");
		return -EPERM;
	}

	kthread_stop(thread);

	return 0;
}

static int thread_loop_hello(void *date)
{
	int i = 0;

	while (!kthread_should_stop()) {
		pr_info("hello world: %d!\n", i++);
		msleep(1000);
	}

	pr_info("bye :)\n");

	return 0;
}

/* normal */
static int test_kthread_stop_normal(void)
{
	struct task_struct *thread;

	thread = kthread_run(thread_loop_hello, NULL, "kthread_loop_hello");
	if (IS_ERR(thread)) {
		pr_info("test: fail to create a kthread\n");
		return -EPERM;
	}
	
	msleep(4000);

	kthread_stop(thread);

	return 0;
}

static int __init kthread_init(void)
{
	switch (case_num) {
	case 1:
		test_kthread_stop_a_thread();
		break;
	case 2:
		test_kthread_stop_righ_after_run();
		break;
	case 3:
		test_kthread_stop_normal();
		break;
	default:		
		pr_err("no test case found!\n");
	}

        return 0;
}

static void __exit kthread_exit(void)
{
}

module_init(kthread_init);
module_exit(kthread_exit);

MODULE_AUTHOR("Sherlock");
MODULE_DESCRIPTION("The driver for kthread study");
```
