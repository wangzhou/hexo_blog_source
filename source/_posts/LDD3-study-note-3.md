---
title: LDD3 study note 3
tags:
  - 读书笔记
  - LDD3
description: '这篇笔记记录驱动程序中mmap的写法，以及相关的调试过程。同样代码在： https://github.com/wangzhou/scull.git'
categories: read
abbrlink: 65bd8c5
date: 2021-07-05 22:40:00
---

mmap
--------

 在linux用户态调用mmap函数可以把文件内容直接映射到内存，这样用户态程序可以像访问
 内存一样访问文件。同样，使用mmap也可以把设备的一段IO空间映射到用户态，用户态程序
 可以直接访问这个设备的寄存器。当然，要在程序驱动里添加mmap的对应支持。

驱动实现
-----------
为了支持把设备的IO空间映射到用户态，驱动里要实现.mmap的回调，struct vm_area_struct
会把用户态想要映射到的用户态虚拟地址传到内核。在.mmap回调里需要把这些参数，连同
想要映射的实际物理地址传递给remap_pfn_range函数，这个函数帮助建立虚拟地址到物理
地址的映射。

下面的例子是在scull驱动里，申请了一段内核态的内存，我们可以通过下面的操作把它映射
到用户态。
```
int scull_mmap(struct file *file, struct vm_area_struct *vma)
{
        unsigned long page = virt_to_phys(scull_device->mmap_memory);
        unsigned long start = (unsigned long)vma->vm_start;
        unsigned long size = (unsigned long)(vma->vm_end - vma->vm_start);
        vma->vm_flags |= (VM_IO | VM_LOCKED | VM_DONTEXPAND | VM_DONTDUMP);

        if (remap_pfn_range(vma, start, page >> PAGE_SHIFT, size, PAGE_SHARED)) {
                printk(KERN_ALERT "remap_pfn_range failed!\n");
                return -1;
        }

        return 0;
}
```
LDD3上讲，remap_pfn_range只能映射IO空间和系统保留内存，但是上面例子的mmap_memory
实际在就是用get_free_page分配的(把get_free_page换成kmalloc也是可以做映射的), 测试
的结果是用get_free_page分配的内存也是可以被映射到用户态的。所以，实际上
remap_pfn_range也可以把系统内存映射到用户态。

内核用struct vm_area_struct管理进程空间的各个虚拟地址区域。cat /proc/pid/maps
可以看到一个进程所有的虚拟地址区域。在下面的例子中，我们可以看到测试程序(read.c)
的进程的各个地址区域。可以看到调用mmap创建起来的一个地址区域/dev/scull0.

```
estuary:/$ cat /proc/1251/maps
00400000-00401000 r-xp 00000000 00:01 882            /a.out
00410000-00411000 rw-p 00000000 00:01 882            /a.out
ffffae624000-ffffae634000 rw-p 00000000 00:00 0 
ffffae634000-ffffae764000 r-xp 00000000 00:01 629    /lib/aarch64-linux-gnu/libc-2.21.so
ffffae764000-ffffae773000 ---p 00130000 00:01 629    /lib/aarch64-linux-gnu/libc-2.21.so
ffffae773000-ffffae777000 r--p 0012f000 00:01 629    /lib/aarch64-linux-gnu/libc-2.21.so
ffffae777000-ffffae779000 rw-p 00133000 00:01 629    /lib/aarch64-linux-gnu/libc-2.21.so
ffffae779000-ffffae77d000 rw-p 00000000 00:00 0 
ffffae77d000-ffffae799000 r-xp 00000000 00:01 575    /lib/aarch64-linux-gnu/ld-2.21.so
ffffae7a1000-ffffae7a2000 rw-s 00000000 00:01 1385   /dev/scull0
ffffae7a2000-ffffae7a7000 rw-p 00000000 00:00 0 
ffffae7a7000-ffffae7a8000 r--p 00000000 00:00 0      [vvar]
ffffae7a8000-ffffae7a9000 r-xp 00000000 00:00 0      [vdso]
ffffae7a9000-ffffae7aa000 r--p 0001c000 00:01 575    /lib/aarch64-linux-gnu/ld-2.21.so
ffffae7aa000-ffffae7ac000 rw-p 0001d000 00:01 575    /lib/aarch64-linux-gnu/ld-2.21.so
fffff5f89000-fffff5faa000 rw-p 00000000 00:00 0      [stack]
```
