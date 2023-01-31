---
title: Linux内核模块编译
tags:
  - Linux内核
description: 介绍linux内核模块的编译步骤
abbrlink: be0ad29e
date: 2021-07-17 11:21:03
categories:
---

采用源码树外编译的方式
-----------------------

1. 编写模块代码：hello.c[1]

2. 在同一目录下编写Makefile[2]

3. 在同一目录下编译：
```
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- -C /vm/wangzhou/linux-iks  M=`pwd` modules
```
   ARCH：平台, CROSS_COMPILE：交叉编译器, -C：kernel源码路径, M=：module代码的路径, 
   modules：指明编译模块。这里编译arm平台下的module

4. 在hello.c同一目录下将会生成hello.ko

在kernel的源码树中编译
-----------------------

1. 把写好的模块代码放入源码树的一个目录：linux-src/drivers/char/hello.c

2. 在.../char/下的Makefile中加入：obj-m += hello.o

   可以看到Makefile中的项目多是: obj-$(CONFIG_VIRTIO_CONSOLE)+= virtio_console.o

3. 在linux-src/下：

   make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- zImage -j 8 modules
   内核中所有配置成模块的程序将得到编译, 本例子上在linux-src/drivers/char/下将会
   看到编译好的hello.ko

模块由多个文件组成，放在一个目录里编译
---------------------------------------

1. 把文件夹放在源码树的一个目录下：

   linux-src/drivers/char/hello/hello.c  ...hello/add.c[3]
   为了测试的目的, 需要在hello.c中加入调用add()的语句...

2. 在.../char/的Makefile文件中加：obj-m += hello/ 表示该module在hello目录下

3. 在.../hello/下创建Makefile文件[4]

4. 同上面第三步, 在hello/下会生成hello_add.ko模块

利用Kconfig图形化配置kernel、module: 
-------------------------------------

 用make menuconfig可以图形化配置kernel。可以使用下面的方式, 把自己写的module加入
 配置菜单：

1. 把.../char/下的Makefile中之前加的行修改成：
   obj-${CONFIG_HELLO_ADD} += hello/ 表示在最后在.config文件中显示的配置项是CONFIG_HELLO_ADD

2. 把.../char/hello/下的Makefile修改成：
   obj-${CONFIG_HELLO_ADD} += hello_add.o
   hello_add-objs := hello.o add.o

   可以看到, 编译时会根据CONFIG_HELLO_ADD的值去决定编译成什么, 下面要说的Kconfig
   即用来设置CONFIG_HELLO_ADDde值

3. 在.../char/下的Kconfig文件中加入：
```
config HELLO_ADD
	tristate "HELLO_ADD MODUEL(try the Kconfig)"
	default n
	help
	  something about help information of hello_add module
```
 tristate表示可以配置成n, y, m三种状态(不编译，直接编译入kernel，编译成module),
 后面的字符串最后会显示到菜单上, default 指明默认的情况, config 指明配置的变量,
 即CONFIG_HELLO_ADD

4. make menuconfig 依照下面路径, 将会看到HELLO_ADD模块的配置菜单
```
  Device Drivers --->
          Character devices --->
          ...
          <> HELLO_ADD MODULE(try the Kconfig)
          ...
```

5. 同(二)中第三步，在对应目录下即可得到hello_add.ko

附录
-----

[1] 编写模块代码：hello.c
```
/* hello.c */
#include <linux/init.h>
#include <linux/module.h>
MODULE_LICENSE("Dual BSD/GPL");

static int hello_init(void)
{
	printk(KERN_ALERT "Hello, begin to test ...\n");
	return 0;
}

static void hello_exit(void)
{
	printk(KERN_ALERT"Goodbye, cruel world\n");
}

module_init(hello_init);
module_exit(hello_exit);

MODULE_AUTHOR("***");
MODULE_DESCRIPTION("A simple test Module");
MODULE_ALIAS("a simplest module");
```

[2] Makefile文件
```
    obj-m := hello.o
    clean:
        rm  hello.ko hello.mod.c hello.mod.o hello.o modules.order Module.symvers
```

[3] 编写文件: add.c
```
    /* add.c */
    int add(int a, int b)
    {
	return (a+b);
    }
```

[4] Makefile文件
```
    obj-m := hello_add.o  /* 指明module的名字 */
    hello_add-objs := add.o hello.o /* 指明hello_add是由几个文件链接到一起的*/
```
