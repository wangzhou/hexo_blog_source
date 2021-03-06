---
title: APUE学习笔记(第三章)
tags:
  - APUE
description: '本章讨论create, open, read, write，lseek等文件I/O函数。'
categories: read
abbrlink: 4f2f584c
date: 2021-07-17 10:48:01
---


1. 文件相关的几个数据结构：文件描述符，文件表，i node.
   每个进程有一个文件描述符表(用户态中), 用来指代这个进程打开的文件，文件描述符
   是0，1，2...这样的数字，对应的是一个个打开的文件。read(), write(), lseek()
   中指代文件的入参用文件描述符表。

   文件表是一个内核数据结构，每个进程打开的每个文件在内核里都有对应的一个结构
   来描述，所有的这样的结构组成文件表。每个这样的结构中包括，文件状态标志，文件
   偏移量，i node指针。

   open(path, oflag, ...); open函数的第二个入参，可以设置只读，只写等参数。这
   些参数最终被写到上面的文件状态标志中。当再调用read, write时，文件状态标志
   将起到判定的作用。

   这三个数据结构的关系见APUE上的图。

2. 缓冲概念：从硬盘上读数据，文件系统会做缓冲，这个在内核中。用户态的I/O也会坐
   缓冲，这个存在用户态的buffer中。数据写入硬盘，会经过用户态缓冲，内核文件系统
   缓冲，添加到写队列等步骤。

3. 多个进程写一个文件：如果是都在文件末尾写入，则lseek到文件末尾，再write写入，
   会出现竞争。用open(..,O_APPEND,..)可以避免。O_APPEND会设置文件状态标志，write
   时会查看文件状态标志，如果有O_APPEND则在文件末尾写入。

4. ./a.out 6>test
   在文件描述符6上打开文件test. 编译下面的代码，运行上面的命令。可以开到文件
   test中写入“12345”。以此类推：ls 2>>log, 在文件描述符2上打开文件，文件描述
   符2一般是错误错误输出，所以这样的命令把ls命令的错误输出写入log文件。
   command 6<>log, 是运行command的时候，在文件描述符6上，以读写的方式打开文件log。
```
	#include <unistd.h>
	#include <errno.h>

	char buffer[5] = {'1','2','3','4','5'};

	int main(void)
	{
		int ret;
		int file;

		/* 把下面的5改成其他大于5的数，写入文件时会出错
                 * 貌似是要输出几个buffer中的数据，这里的count就是多少，
                 * 否则会出错
                 */		
		ret = write(6, buffer, 5);
		if (ret == -1) {
			perror("read read_test");
			return -1;
		}
	}
```
5. fcntl函数, 提供对文件描述符标志和文件状态标志的查询和修改。
   不清楚文中所说的文件描述符是什么? 下面得到的结果均是0
```
	fd = fcntl(0, F_GETFD);	
	printf("fd: %d\n", fd);
	fd = fcntl(1, F_GETFD);	
	printf("fd: %d\n", fd);
	fd = fcntl(2, F_GETFD);	
	printf("fd: %d\n", fd);
```
   文件状态标志位和上面的描述一致。fcntl可以复制文件描述符：fcntl(fd, F_DUPFD, fd_min)，
   大于或等于fd_min的文件描述符将指向fd所指向的文件表项，相当于把fd复制到了fd_min
   
