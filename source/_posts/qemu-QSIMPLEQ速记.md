---
title: qemu QSIMPLEQ速记
tags:
  - QEMU
  - 数据结构
description: 本文是对qemu代码里QSIMPLEQ的一个速记，这个是qemu里自带的一个简单链表的实现。
abbrlink: 4562
date: 2023-05-16 14:06:37
categories:
---

数据结构
---------

核心的数据结构是：链表头和链表元素，链表头就两个数据：sqh_first和sqh_last，sqh_first
是第一个链表元素的指针，sqh_last是链表最后一个元素里指向下一个元素的指针的地址。

链表元素类型用户自定义，但是里面必须用QSIMPLEQ_ENTRY定义一个链表元素的指针。整体
示意图如下：
```
 list head              list elm                 list elm
 +------------+         +--------------+         +--------------+
 | sqh_first -+-------->| ...          |    +--->| ...          |
 |            |         |              |    |    |              |
 | sqh_last  -+---+     | entry/field: |    |    | entry/field: |
 +------------+   |     |              |    |    |              |
                  |     |   sqe_next --+----+    |   sqe_next   |
                  |     |              |         |      ^       |
                  |     +--------------+         +------+-------+
                  +-------------------------------------+        
```

基本操作示例
-------------

有了上面的基本数据结构，我们就很容易理解关于这个链表的各种操作。

链表head动态初始化，sqh_last在初始情况下先指向head里的sqh_first。
```
#define QSIMPLEQ_INIT(head) do {                                        \       
    (head)->sqh_first = NULL;                                           \       
    (head)->sqh_last = &(head)->sqh_first;                              \       
} while (/*CONSTCOND*/0)                                                        
```

在链表结尾插入一个新节点。因为sqh_last已经保存了链表最后一个元素里的sqe_next的
地址，所以直接把sqe_next指向新的元素，再更新head里的sqh_last就好。
```
#define QSIMPLEQ_INSERT_TAIL(head, elm, field) do {                     \       
    (elm)->field.sqe_next = NULL;                                       \       
    *(head)->sqh_last = (elm);                                          \       
    (head)->sqh_last = &(elm)->field.sqe_next;                          \       
} while (/*CONSTCOND*/0)                                                        
```

找链表最后一个元素。因为可以直接找到最后一个元素的sqe_next即field结构的地址，通过
类似Linux内核container_of的方式就可以反查到元素的地址。
```
#define QSIMPLEQ_LAST(head, type, field)                                \       
    (QSIMPLEQ_EMPTY((head)) ?                                           \       
        NULL :                                                          \       
            ((struct type *)(void *)                                    \       
        ((char *)((head)->sqh_last) - offsetof(struct type, field))))           
```

遍历链表元素。这个没有什么特殊的。
```
#define QSIMPLEQ_FOREACH(var, head, field)                              \       
    for ((var) = ((head)->sqh_first);                                   \       
        (var);                                                          \       
        (var) = ((var)->field.sqe_next))                                        
```

在链表头插入一个新节点。首先更新新节点的sqe_next指针，然后更新sqh_first使其指向
新插入的节点，这两个操作对所有情况都是一样的。对于空链表的情况，因为新插入的节点
也是末尾节点，所以还要更新下head里的sqh_last。
```
#define QSIMPLEQ_INSERT_HEAD(head, elm, field) do {                     \       
    if (((elm)->field.sqe_next = (head)->sqh_first) == NULL)            \       
        (head)->sqh_last = &(elm)->field.sqe_next;                      \       
    (head)->sqh_first = (elm);                                          \       
} while (/*CONSTCOND*/0)                                                        
```

在一个元素后插入新节点。基本逻辑和在链表头插入一个新节点的逻辑一样，都要额外处理
下插入节点是末尾节点的情况。
```
#define QSIMPLEQ_INSERT_AFTER(head, listelm, elm, field) do {           \       
    if (((elm)->field.sqe_next = (listelm)->field.sqe_next) == NULL)    \       
        (head)->sqh_last = &(elm)->field.sqe_next;                      \       
    (listelm)->field.sqe_next = (elm);                                  \       
} while (/*CONSTCOND*/0)                                                        
```

从链表头上删除一个节点。基本逻辑和在链表头插入一个新节点的逻辑一样，都要额外处理
下插入节点是末尾节点的情况。
```
#define QSIMPLEQ_REMOVE_HEAD(head, field) do {                          \       
    typeof((head)->sqh_first) elm = (head)->sqh_first;                  \       
    if (((head)->sqh_first = elm->field.sqe_next) == NULL)              \       
        (head)->sqh_last = &(head)->sqh_first;                          \       
    elm->field.sqe_next = NULL;                                         \       
} while (/*CONSTCOND*/0)                                                        
```
