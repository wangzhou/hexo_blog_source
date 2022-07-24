---
title: Linux kernel poll
tags:
  - Linux内核
description: Linux内核poll系统调用实现的分析。
abbrlink: 4142aa42
date: 2021-06-20 23:17:48
---
```
/* fs/select.c */
SYSCALL_DEFINE3(poll, struct pollfd __user *, ufds, unsigned int, nfds,
		int, timeout_msecs)
  +-> do_sys_poll
    +-> do_poll(head, &table, end_time)

    /* poll的主流程在do_poll这个函数里。可以看到这个函数是一个大for循环，基本逻辑是：*/

    for () {
	/*
	 * poll系统调用可能是poll一组fd，所以这里循环调用这组fd对应的底层poll
	 * 回调函。以uacce驱动为例, 驱动的poll回调函数里，一般先调用poll_wait
	 * 把自己挂到一个等待队列上，注意这个操作就是在等待队列的链表上加上一个
	 * entry, 函数会马上返回，这之后会检测一下是否已经poll到数据。注意，真正
	 * 把当前进程睡眠的点在下面的poll_schedule_timeout函数处，所以如果第一次
	 * poll的时候已经有任务完成，程序走到下面if count的地方会直接返回，不会
	 * 睡眠。
	 */
	for () {
		do_pollfd();
		  +-> vfs_poll();
		    +-> file->f_op->poll(file, pt);
	}

	/* poll到有event发生或者超时就退出大循环 */
	if (count || timed_out)
		break;

	/*
	 * 睡眠等待event发生，以uacce驱动为例，当硬件有任务完成的时候，会在中断
	 * 处理中唤醒等待队列上的进程。进程被唤醒后，从这里继续执行，在上面的
	 * 内部for循环里依次调用各个fd的poll函数查找有event发生的fd。
	 */
	poll_schedule_timeout();

    };
```

uacce驱动里的poll回调函数:
```
static __poll_t uacce_fops_poll(struct file *file, poll_table *wait)
{
	struct uacce_queue *q = file->private_data;
	struct uacce_device *uacce = q->uacce;

	poll_wait(file, &q->wait, wait);
	if (uacce->ops->is_q_updated && uacce->ops->is_q_updated(q))
		return EPOLLIN | EPOLLRDNORM;

	return 0;
}
```
