---
title: Linux巨页的使用
tags:
  - Linux内核
description: linux内核支持两种巨页的使用方法，一种是普通巨页，一种是透明巨页。本文介绍的是普通巨页的使用。
abbrlink: f55c9d41
date: 2021-06-20 23:15:06
---

普通巨页的内核说明文档在linux/Documentation/admin-guide/mm/hugetlbpage.rst

使用巨页需要在内核编译的时候打开相关的配置选项：CONFIG_HUGETLBFS, CONFIG_HUGETLB_PAGE

使能系统的巨页可以通过内核的启动参数或者是/sys/kernel/mm/hugepages/下的文件。
我们具体看下sysfs下的相关目录。
```
hugepages/
├── hugepages-1048576kB
│   ├── free_hugepages
│   ├── nr_hugepages
│   ├── nr_hugepages_mempolicy
│   ├── nr_overcommit_hugepages
│   ├── resv_hugepages
│   └── surplus_hugepages
├── hugepages-2048kB
│   ├── free_hugepages
│   ├── nr_hugepages
│   ├── nr_hugepages_mempolicy
│   ├── nr_overcommit_hugepages
│   ├── resv_hugepages
│   └── surplus_hugepages
├── hugepages-32768kB
│   ├── free_hugepages
│   ├── nr_hugepages
│   ├── nr_hugepages_mempolicy
│   ├── nr_overcommit_hugepages
│   ├── resv_hugepages
│   └── surplus_hugepages
└── hugepages-64kB
    ├── free_hugepages
    ├── nr_hugepages
    ├── nr_hugepages_mempolicy
    ── nr_overcommit_hugepages
    ├── resv_hugepages
    └── surplus_hugepages
```
可以看到在hugepages目录下为每个巨页大小都开了专门的目录。可以看到在我的ARM64
系统上，巨页有64KB，32KB，2MB，1GB四种类型。向nr_hugepages写数值可以创建指定数目
的巨页，读free_hugepages可以得到还没有使用的巨页数目。

一般巨页属于系统配置，我们只去使用，不去更改配置。使用的方法是在mmap的时候在
flags参数中加上MAP_HUGETLB。如果只用MAP_HUGETLB，巨页的分配算法是内核里定的，
比如，你要mmap 128KB的内存，在64KB和2MB都可以满足的时候，我们希望从64KB里搞两
页出来就好了，内核可能是从2MB里分配的，实际上，用5.10-rc2的内核，真的是从2MB的
页里分一页出来。

所以，针对巨页，mmap flags里还有宏可以指定从哪种大小的页里分巨页。man mmap下有：
MAP_HUGE_2MB, MAP_HUGE_1GB，不过直接用这个宏会报没有定义，这个是因为gcc版本比较
低，我们可以直接找见内核里定义的地方: linux/include/uapi/linux/mman.h
```
#define MAP_HUGE_64KB	HUGETLB_FLAG_ENCODE_64KB
#define MAP_HUGE_2MB	HUGETLB_FLAG_ENCODE_2MB

#define HUGETLB_FLAG_ENCODE_64KB	(16 << HUGETLB_FLAG_ENCODE_SHIFT)
#define HUGETLB_FLAG_ENCODE_2MB		(21 << HUGETLB_FLAG_ENCODE_SHIFT)

#define HUGETLB_FLAG_ENCODE_SHIFT	26
```
可以看到16, 21正好是页大小以2为底的对数。所以，举个例子，我们可以如下指定从
64KB的巨页中申请内存：
```
int len = 128 * 1024;
unsigned long *p;
int i;

p = mmap(NULL, len, PROT_READ | PROT_WRITE, MAP_PRIVATE | MAP_ANONYMOUS |
	 MAP_HUGETLB | (16 << 26), -1, 0);

for (i = 0; i < len / sizeof(*p); i++) {
	*(p + i) = 20;
}
```
之后再去看/sys/kernel/mm/hugepages/hugepages-64kB/free_hugepages, 会发现减少了2。
需要注意的是，mmap到的内存要去写下，内存才会真实分配，如果在mmap之后就去看
free_hugepages的值，其中还是原来的值。
