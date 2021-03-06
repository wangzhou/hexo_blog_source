---
title: 使用TAILQ
tags:
  - 数据结构
description: TAILQ是一个套公共的链表实现，我们可以直接使用。本文分析TAILQ的实现
abbrlink: a6b302af
date: 2021-06-20 23:16:01
---

TAILQ是BSD里实现的一套简单的链表，一般linux系统只要include sys/queue.h就可以使用。
在我使用的ARM版本的ubuntu系统，这个文件在/usr/include/aarch64-linux-gnu/sys/queue.h.

TAILQ作为链表使用的基本数据结构就是两个: TAILQ_ENTRY, TAILQ_HEAD。一个表示链表
的节点，一个表示链表头, 当使用的时候需要用TAILQ宏定义出相应的结构。具体使用的
方法和内核里链表的使用方式基本一致。在需要链表连起来的元素里每个埋一个TAILQ_ENTRY
的结构体。随后的各种操作宏里会大量的用到node这个变量。其实TAILQ_ENTRY就是定义了
一个包含指向本节点前后节点的指针结构。
```
struct your_list_node {
	int data;
	TAILQ_ENTRY(your_list_node) node;
};

#define	_TAILQ_ENTRY(type, qual)					\
struct {								\
	qual type *tqe_next;		/* next element */		\
	qual type *qual *tqe_prev;	/* address of previous next element */\
}
#define TAILQ_ENTRY(type)	_TAILQ_ENTRY(struct type,)
```
使用TAILQ_HEAD定义一个类型是struct list_head, 所管理的链表元素类型是
truct your_list_node的链表头结构体。所以如下的宏其实只是定义了一个类型：
struct list_head。使用的时候还要有链表头实例的定义。
```
TAILQ_HEAD(list_head, your_list_node);	
```

有了这两个基本数据结构，我们看几个基本的操作。
```
struct list_head head;
TAILQ_INIT(&head);
```
初始化一个链表头。

```
struct your_list_node e1;
TAILQ_INSERT_TAIL(&head, &e1, node);
```
把一个节点插入head链表头表示的链表的尾部。第二个参数是链表元素的指针。

```
TAILQ_FOREACH(tmp, &head, node) {
	printf("---> %d\n", tmp->data);
}
```
遍历head链表头表示的链表的所有元素，每次得到的元素指针放到tmp里。

```
struct your_list_node e4;
struct your_list_node e5;
TAILQ_INSERT_BEFORE(&e4, &e5, node);
```
把e5插到e4的前面，第一个, 第二个参数都是链表元素指针。

```
struct your_list_node *tmp;
tmp = TAILQ_LAST(&head, list_head);
```
得到head链表头表示的链表的最后一个元素的指针。head是链表头实例的名字，list_head
则是链表头的类型。

```
struct your_list_node e2;
TAILQ_REMOVE(&head, &e2, node);
```
从head链表头表示的链表中删除e2节点，第二个参数是要删除节点的指针。

其他的宏基本逻辑和上面介绍的基本一样。
完整的测试代码可以参考: https://github.com/wangzhou/tests/blob/master/tailq/test.c
