---
title: pstack使用笔记
tags:
  - 软件调试
description: >-
  pstack的功能是显示当前进程中函数的调用栈的关系，若是多线程的情况下，
  会显示各个线程中函数调用的关系。脚本用了gdb中的bt(backtrace)功能，在gdb中输入bt即可 打印出程序当前的栈中的函数调用关系。
abbrlink: 8141fc42
date: 2021-07-17 11:21:31
categories:
---

使用方法：pstack pid, 就是说要先把程序跑起来，查见进程号，再使用pstack pid

1. 在ubunbu系统上，使用sudo apt-get install pstack可以直接安装pstack，注意这里
   安装上的是二进制的程序。之前对这样安装上的pstack作了测试，无法显示函数的调用
   关系。根据man pstack的说明，现在的pstack只支持32bit ELF binaries

2. 网上的一些文章指出，pstack只是一个脚本程序，在相关网站上下载了pstack.sh脚本
   测试。使用时总是提示出错，但是表现看不出来错误，全部删去，就留下前几行，还是
   显示有错，新建一个脚本文件，照抄那几行过来（这时两个文件看起来一样），但是用
   diff ***.sh ***.sh测试一下，竟然不一样。用ghex查看二进制的文件，看出是字符行
   尾的时候的编码不一样。想到下载的脚本可以是在windows下的脚本，下载安装 dos2unix
   编码转换工具，然后dos2unix pstack.sh, 工具可以使用。

   pstack.sh脚本简单分析:
```
/* pstack.sh */
#! /bin/sh
# 输入参数以及pid是否存在的判断
if test $# -ne 1; then
    echo "Usage: `basename $0 .sh` <process-id>" 1>&2
    exit 1 
fi
if test ! -r /proc/$1; then
    echo "Process $1 not found." 1>&2
    exit 1
fi

# GDB doesn't allow "thread apply all bt" when the process isn't
# threaded; need to peek at the process to determine if that or the
# simpler "bt" should be used.
# 先设置参数bt, 要是多线程的情况，那么设置为：thread apply all bt
backtrace="bt"
if test -d /proc/$1/task ; then
    # Newer kernel; has a task/ directory.
    if test `/bin/ls /proc/$1/task | /usr/bin/wc -l` -gt 1 2>/dev/null ; then
        backtrace="thread apply all bt"
    fi
elif test -f /proc/$1/maps ; then
    # Older kernel; go by it loading libpthread.
    if /bin/grep -e libpthread /proc/$1/maps > /dev/null 2>&1 ; then
        backtrace="thread apply all bt"
    fi
fi
 
GDB=${GDB:-/usr/bin/gdb}

# echo $GDB -> /usr/bin/gdb

# -nx: Do not execute commands from any `.gdbinit' initialization files.  Normal
   ly, the commands in these files are executed after all the command options  
   and  argu‐ments have been processed.
# --quiet: Do not print the introductory and copyright messages.  These message
   s are also suppressed in batch mode
# --batch: ...
# --readnever: ...
if $GDB -nx --quiet --batch --readnever > /dev/null 2>&1; then
    readnever=--readnever
else
    readnever=
fi
# 单步运行/usr/bin/gdb -nx --quiet --batch --readnever时,--readnever会出错
 
# Run GDB, strip out unwanted noise.
# 原来用/proc/$1/exe找到pid对应的命令行，但是发现下面命令中的是/proc/$1/exe
# 改成下面的'echo /proc/$1/exe'就可以了
$GDB --quiet $readnever -nx 'echo /proc/$1/exe' $1 <<EOF 2>&1 |
$backtrace 
EOF
```

这里，运行pstack [pid]一次会显示当时的函数调用，不能显示指定位置的堆栈情况。
可以先运行gdb --quiet -nx /proc/command/exe command, 待gdb起来后，在想要的地
方设立断点，程序停住之后，使用bt，显示出当时的堆栈函数调用情况
 
整理显示输出格式
/bin/sed -n \
    -e 's/^(gdb) //' \
    -e '/^#/p' \
    -e '/^Thread/p'

附：测试程序和堆栈情况（单线程和多线程）

单线程：
```
#include <stdio.h>

int func_called(int a, int b)
{
	int c;
	c = a + b;
	return c;
}

void func_calling(void)
{
	int a = 1, b = 2;
	int r;
	r = func_called(a, b);
	printf("r = %d\n", r);
}

int main()
{
	while(1) {
		func_calling();
	}
}
```

测试输出：
```
29788 pts/5    00:00:34 test2
29819 pts/4    00:00:00 ps
***@A101107831:/vm/***/notes$ ./pstack.sh 29788
[sudo] password for ***: 
#0  0x00007f6d7d366910 in __write_nocancel ()
#1  0x00007f6d7d2f9883 in _IO_new_file_write (f=0x7f6d7d639260, 
#2  0x00007f6d7d2f974a in new_do_write (fp=0x7f6d7d639260, 
#3  0x00007f6d7d2faeb5 in _IO_new_do_write (fp=<optimized out>, 
#4  0x00007f6d7d2fa025 in _IO_new_file_xsputn (n=1, data=<optimized out>, 
#5  _IO_new_file_xsputn (f=0x7f6d7d639260, data=<optimized out>, n=1)
#6  0x00007f6d7d2ca4a7 in _IO_vfprintf_internal (s=<optimized out>, 
#7  0x00007f6d7d2d38d9 in __printf (format=<optimized out>) at printf.c:35
#8  0x000000000040054d in func_calling () at test2.c:15
#9  0x0000000000400563 in main () at test2.c:27
```
注：使用pc上已经在运行的应用程序调试时，无法显示具体的函数名和相关参数，原因是pc
    上的应用程序没有加入调试信息

多线程：
```
# include<stdio.h>
# include<pthread.h>
# include<string.h>

pthread_t tid1;
pthread_t tid2;

void* thread1(void* arg)
{
	while (1) {
		printf("thread_id = %d\n", (int)tid1);
		sleep(1);
	} 
}


void* thread2(void* arg)
{
	while (1) {
		printf("thread_id = %d\n", (int)tid2);
		sleep(1);
	}
}


int main()
{
	int err;
	
	err = pthread_create(&tid1, NULL, thread1, NULL);
	if (err != 0){
		fprintf(stderr, "can't create thread: %s\n", strerror(err));
	}
	err = pthread_create(&tid2, NULL, thread2, NULL);
	if (err != 0){
		fprintf(stderr, "can't create thread: %s\n", strerror(err));
	}
	
	while(1); 
}
```
测试输出：
```
*** 30280 22661 30280 99    3 21:51 pts/5    00:00:12 ./multi_threads
*** 30280 22661 30281  0    3 21:51 pts/5    00:00:00 ./multi_threads
*** 30280 22661 30282  0    3 21:51 pts/5    00:00:00 ./multi_threads
*** 30284  3758 30284  0    1 21:51 pts/4    00:00:00 ps -eLf
***@A101107831:/vm/***/notes$ ./pstack.sh 30280
[sudo] password for ***: 
Thread 3 (Thread 0x7f2a351ae700 (LWP 30281)):
#0  0x00007f2a3526e84d in nanosleep () at ../sysdeps/unix/syscall-template.S:82
#1  0x00007f2a3526e6ec in __sleep (seconds=0)
#2  0x00000000004006fc in thread1 (arg=0x0) at multi_threads.c:14
#3  0x00007f2a35575e9a in start_thread (arg=0x7f2a351ae700)
#4  0x00007f2a352a2ccd in clone ()
#5  0x0000000000000000 in ?? ()
Thread 2 (Thread 0x7f2a349ad700 (LWP 30282)):
#0  0x00007f2a3526e84d in nanosleep () at ../sysdeps/unix/syscall-template.S:82
#1  0x00007f2a3526e6ec in __sleep (seconds=0)
#2  0x0000000000400736 in thread2 (arg=0x0) at multi_threads.c:22
#3  0x00007f2a35575e9a in start_thread (arg=0x7f2a349ad700)
#4  0x00007f2a352a2ccd in clone ()
#5  0x0000000000000000 in ?? ()
Thread 1 (Thread 0x7f2a35988700 (LWP 30280)):
#0  main () at multi_threads.c:39
```
注：按照下面gdb multi-thread debug方法，在pc上无法显示有关多线程的信息(被调试进
    程使用pc已经在运行的应用程序进程)

gdb多线程debug:
https://sourceware.org/gdb/onlinedocs/gdb/Threads.html#Threads
