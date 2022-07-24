---
title: Linux内核链表笔记
tags:
  - Linux内核
description: 本文介绍linux kernel中的链表实现，用法和其中包含的设计思想。
abbrlink: f3ed511a
date: 2021-07-17 11:02:02
categories:
---

linux kernel中的链表实现在/linux/include/linux/list.h中。我们可以在一个c程序中
很快的实现一个链表，那么kernel中链表有什么独特之处呢？一般的我们这样定义一个链
表的节点：
```
struct list_node_a {
	data_a; 
	struct list_node *next;
}
```
这样写的不足是，链表上的数据和链表本身紧密耦合在了一起。现在链表中每个节点存放的
是data_a, 如果又有一个data_b需要串成一个链表，那么就需要再建立一个struct list_note_b
这样的链表节点。简而言之，就是这样的通用性不强。

linux kernel中的实现是，首先实现一个
```
struct list_head {
	struct list_head *prev;
	struct list_head *next;
}
```
的基础结构,这个是链表最核心的结构。然后再加上一些链表操作的接口函数。OK, 通用的
链表实现就完成了。

但是我要怎么使用呢？现在有一个结构 struct data 需要用一个链表管理起来(用一个链表
把一堆struct data串起来)。就在struct data的定义中嵌入一个struct list_head结构，
看起来是这样的：
```
struct data {
	...;	
	struct list_head node;
}
```
除此之外，一般还会定义一个struct list_head作为这个链表的表头节点。表头节点是一个
struct list_head结构, 代表整个链表，他不嵌入struct data中。最后组成的链表如下图
所示(其中struct A就是struct data)。
```
                    +--------------+        +--------------+        +--------------+    
                    |___struct A___|        |___struct A___|        |___struct A___|
                    | ...          |        | ...          |        | ...          |
 +----------+       | +----------+ |        | +----------+ |        | +----------+ |
 |list node_|       | |list node_| |        | |list node_| |        | |list node_| |
 |  prev    |<--------|  prev    |<-----------|  prev    |<-----------|  prev    |<------\
 |  next    |-------->|  next    |----------->|  next    |----------->|  next    |-----\ |
 +----------+       | +----------+ |        | +----------+ |        | +----------+ |   | |
    |  /|\          +--------------+        +--------------+        +--------------+   | |
    |   \                                                                              | |
    \   -------------------------------------------------------------------------------/ /
     ------------------------------------------------------------------------------------
```
但是一个个的struct data节点是怎么被加入链表中的呢? 我们可去/linux/include/linux/list.h
找到答案。假设我们有空的链表 sample_list:
	LIST_HEAD(sample_list);
这里我们初始化了一个链表的表头节点sample_list, 他代表整个链表。在整个链表中的位置
就相当于上图中的最左面哪个list node, 只不过现在他的prev和next指针都是指向自己。

现在我们有一个struct data sample_data结构要加入sample_list链表, 可以调用list.h中的函数
```
	list_add(struct list_head *new, struct list_head *head);
实现：
	list_add(&sample_data.node, &sample_list);
```
可以想像知道struct data sample_data就知道了他里面的struct list_head node的指针，
又知道链表头节点的指针，把一个节点加入链表中是非常容易的。具体可以分析list.h中
的代码。

现在假设我们知道了链表表头节点的指针，我要遍历链表，读取其中的数据。其中关键的就是
知道struct data sample_data中的struct list_head node的指针p，怎么得到对应的
struct data sample_data的指针。没有关系，我们可以使用：
```
	container_of(p, struct data, node);
```
返回对应struct data sample_data的指针。

基本内容到这里就完了，当你发现一个结构中嵌入了一个struct list_head结构时，他一定
将要被连到某个链表中。当然也可能在一个结构中嵌入了多个不同的struct list_head结构，
那一定是这个结构同时被连入了多个链表中。
