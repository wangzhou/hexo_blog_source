---
title: Unix/Linux编程实践教程笔记(11-15)
tags:
  - Linux用户态编程
description: >-
  本文档是Unix/Linux编程实践教程最后五章的总结笔记，这五章主要围绕socket，讲了各种 进程、线程间通信的方法。大家可以从这个链接git
  clone该书对应的代码:
  https://github.com/yuzhidi/Understanding-UNIXLINUX-Programming.git 简单编译后就
  可以运行。
abbrlink: db3a18be
date: 2021-06-27 18:00:40
---

从获取服务的服务的角度讲，一个进程可以直接调用函数，也可以通过和另个一个进程互动
获取相关的服务。从进程间通信的角度讲，socket是一种不同物理机器上进程间通信的方式。

本机间的进程间通信的方式有匿名管道，命名管道(mkfifo), Unix domain socket, 通过
相同的文件通信，共享内存。不同物理机器之间的进程见通信一般用socket。

匿名管道
--------

匿名管道的一般用法是，在父进程中创建管道，fork出子进程，这样子进程也继承了父进程
中的管道的读文件描述符和写文件描述符，如果是父进程往子进程写数据，那么父进程需要
需要关闭读读描述符，子进程需要关闭写描述符，反之亦然，通信的时候父进程向写文件
描述符写入信息，子进程读读文件描述符读出信息。

命名管道
--------

命名管道的方式，需要首先使用mkfifo创建管道文件，然后写进程向管道里写入数据，读
进程从管道里读出数据。命名管道可以使用在独立的进程之间，而匿名管道必须在父子进程
之间使用。(如何查看系统里的管道?)

文件通信
--------

文件通信需要使用文件锁保证一致性

共享内存
--------

使用ipcs -m可以查看系统里现在的共享内存，共享内存是用户态的一块内存，两个进程可以
通过一块共享内存交换信息。使用共享内存会有同步问题，需要对共享内存加锁，做到各个
进程对共享内存访问的互斥，这个可以用信号量机制，信号量是一个系统机制。

  产生数据的进程：
```
	seg_id = shmget(TIME_MEM_KEY, SEG_SIZE, IPC_CREAT | 0777);

	mem_ptr = shmat(seg_id, NULL, 0);

	for (n = 0; n < 60; n++) {
		time(&now);			/* get the time	*/
		strcpy(mem_ptr, ctime(&now));	/* write to mem */
		sleep(1);			/* wait a sec   */
	}
		
	/* now remove it */
	shmctl(seg_id, IPC_RMID, NULL);
```
  消耗数据的进程：
```
	seg_id = shmget(TIME_MEM_KEY, SEG_SIZE, 0777);

	mem_ptr = shmat(seg_id, NULL, 0);

	printf("The time, direct from memory: ..%s", mem_ptr);

	shmdt( mem_ptr );
```

socket
------

关于socket通信，这几章介绍了基于TCP的socket，数据报socket(基于UDP), 和Unix domain
socket。可以使用select和poll响应多个阻塞的文件。

 1. 流式socket

    client端连接建立的流程：
```
	struct sockaddr_in  servadd;        /* the number to call */
	struct hostent      *hp;            /* used to get number */
	int    sock_id, sock_fd;            /* the socket and fd  */

	/*
	 * AF_INET表示Internet域，SOCK_STREAM表示流式socket
	 */

	sock_id = socket(AF_INET, SOCK_STREAM, 0);    /* get a line */
	if (sock_id == -1) 
		oops( "socket" );

	/* servadd用来描述server的地址, 地址由IP和port组成 */
	bzero(&servadd, sizeof(servadd));   /* zero the address */

	/*
	 * 这个函数返回host对应的IP地址，入参是host name或者域名。测试返现，
	 * 如果是host name，得到的是/etc/hosts下对应的IP; 如果是域名，会得到对应
	 × 的IP。
	 */
	hp = gethostbyname(av[1]);          /* lookup host's ip # */
	if (hp == NULL) 
		oops(av[1]);                /* or die */
	bcopy(hp->h_addr, (struct sockaddr *)&servadd.sin_addr, hp->h_length);

	servadd.sin_port = htons(atoi(av[2]));  /* fill in port number */

	servadd.sin_family = AF_INET ;          /* fill in socket type */

	/* now dial */
	if (connect(sock_id,(struct sockaddr *)&servadd, sizeof(servadd)) != 0)
	       oops( "connect" );
```
   server端(省略了一些错误处理)：
```
	struct  sockaddr_in   saddr;   /* build our address here */
	struct	hostent		*hp;   /* this is part of our    */
	char	hostname[HOSTLEN];     /* address 	         */
	int	sock_id,sock_fd;       /* line id, file desc     */
	FILE	*sock_fp;              /* use socket as stream   */

	sock_id = socket(PF_INET, SOCK_STREAM, 0);    /* get a socket */
	bzero((void *)&saddr, sizeof(saddr)); /* clear out struct     */

	gethostname(hostname, HOSTLEN);         /* where am I ?         */
	hp = gethostbyname(hostname);           /* get info about host  */
	                                        /* fill in host part    */
	bcopy((void *)hp->h_addr, (void *)&saddr.sin_addr, hp->h_length);
	saddr.sin_port = htons(PORTNUM);        /* fill in socket port  */
	saddr.sin_family = AF_INET ;            /* fill in addr family  */

	if (bind(sock_id, (struct sockaddr *)&saddr, sizeof(saddr)) != 0)
	       oops("bind");

	if (listen(sock_id, 1) != 0) 
		oops("listen");

	while (1) {
		sock_fd = accept(sock_id, NULL, NULL); /* wait for call */
	        if (sock_fd == -1)
	                oops("accept");         /* error getting calls  */

		/* fdopen把一个文件fd和一个文件流建立联系 */
	        sock_fp = fdopen(sock_fd,"w");  /* we'll write to the   */
	        if (sock_fp == NULL)            /* socket as a stream   */
	                oops("fdopen");         /* unless we can't      */

	        thetime = time(NULL);           /* get time             */
						/* and convert to strng */
	        fprintf(sock_fp, "The time here is ..");
	        fprintf(sock_fp, "%s", ctime(&thetime)); 
	        fclose(sock_fp);              /* release connection   */
	}

```

 2. 数据报socket

    数据报socket不需要提前建立链路，所以流式socket里的connect，listen，accept都
    没有。数据报socket用sendto和recvfrom来发送和接收数据, recvfrom可以得到发送
    者的地址，这样就可以用sendto给发送者发送信息。

    (数据报可以直接对socket fd做read操作么？)

 3. Unix domain socket

    和上面的使用IP加端口号的地址表示方式不同，Uninx domain socket用主机上的一个
    文件表示地址。

    client:
```
	int	           sock;
	struct sockaddr_un addr;

	sock = socket(PF_UNIX, SOCK_DGRAM, 0);

	addr.sun_family = AF_UNIX;
	/* sockname是一个文件名字 */
	strcpy(addr.sun_path, sockname);
	addrlen = strlen(sockname) + sizeof(addr.sun_family);

	sendto(sock, msg, strlen(msg), 0, (struct sockaddr *)&addr, addrlen);
```
    server:
```
	int	sock;			/* read messages here	*/
	struct sockaddr_un addr;	/* this is its address	*/

	/* build an address */
	addr.sun_family = AF_UNIX;		/* note AF_UNIX */
	strcpy(addr.sun_path, sockname);	/* filename is address */
	addrlen = strlen(sockname) + sizeof(addr.sun_family);

	sock = socket(PF_UNIX, SOCK_DGRAM, 0);	/* note PF_UNIX  */

	/* bind the address */
	bind(sock, (struct sockaddr *) &addr, addrlen);

	while (1) {
		/* 直接read socket fd */
		l = read(sock, msg, MSGLEN);	/* read works for DGRAM	*/
		msg[l] = '\0';			/* make it a string 	*/

		/* do your job... */
	}
```

 4. select poll
