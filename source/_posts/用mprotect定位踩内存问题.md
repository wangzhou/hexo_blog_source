---
title: 用mprotect定位踩内存问题
tags:
  - 软件调试
  - mprotect
description: 本文以一个实际的例子介绍如何使用mprotect定位踩内存的问题
abbrlink: 3bbd52f3
date: 2021-06-27 18:13:40
---

Linux用户态程序踩内存时可以用mprotect定位，mprotect本身是linux系统上的一个系统
调用，这个系统调用可以改变一段内存的读写属性, 当有非法的访问访问对应的内存的时候
会给进程发一个SIGSEGV信号，进程可以在信号处理函数中加调试信息进行定位。

mprotect的参数为要保护的虚拟地址，保护地址的大小，和保护地址空间的属性。这里
地址size必须是已页对齐的，地址空间的属性有读、写、执行和不可接入。

显然，当被踩内存本身就是只读的时候，我们一开始就可以用mprotect把这段内存保护起来,
别的执行流踩了这段内存就会触发信号。如果，被踩的内存是一段可读可写的内存，我们
可以在正常执行的时候调用mprotect设置为读写，正常执行完后用mprotect设置为只读。

在信号处理函数中，可以调用backtrace, backtrace_symbols相关函数把调用栈打出来。
如下的测试代码，在X86上用gcc -rdynamic test.c编译运行是OK的，可以打出调用栈，
加-rdynamic是为了打出调用栈里的函数名。但是在ARM64的环境下，需要用
gcc -rdynamic -funwind-tables test.c来编译测试代码，否则只能打出模块的名字。

```
#include <execinfo.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <signal.h>
#include <malloc.h>
#include <sys/mman.h>

/*
 * test in x86 with gcc -rdynamic test.c is OK.
 *
 * however, in aarch64, return of backtrace is alway 1.
 * use gcc -rdynamic -funwind-tables test.c to solve this problem.
 */

void fun_3(void);

void handler(int sig, siginfo_t *si, void *unused)
{
	fun_3();
}

void fun_3(void)
{
#define SIZE 10
	void *buffer[SIZE];
	char **strings;
	int n, i;

	n = backtrace(buffer, SIZE);

	strings = backtrace_symbols(buffer, n);

	for (i = 0; i < n; i++) {
		printf("%s\n", strings[i]);
	}

	free(strings);

	exit(EXIT_FAILURE);
}

void fun_2(void)
{
	fun_3();
}

void fun_1(void)
{
	fun_2();
}

void fun_0(void)
{
	fun_1();
}

int main()
{
	struct sigaction sa;
	char *buffer;
	int pagesize;

	sa.sa_flags = SA_SIGINFO;
	sigemptyset(&sa.sa_mask);
	sa.sa_sigaction = handler;
	sigaction(SIGSEGV, &sa, NULL);

	pagesize = sysconf(_SC_PAGE_SIZE);
	buffer = memalign(pagesize, 4 * pagesize);

	if (mprotect(buffer, pagesize, PROT_READ | PROT_WRITE) == -1)
		printf("fail to set mprotect\n");

	printf("write a in buffer a\n");
	*buffer = 'a';
	printf("write a in buffer b\n");
	sleep(2);
	printf("write a in buffer c\n");

	if (mprotect(buffer, pagesize, PROT_READ) == -1)
		printf("fail to set mprotect\n");

	*buffer = 'b';

	//fun_0();	

	exit(EXIT_SUCCESS);
}
```
