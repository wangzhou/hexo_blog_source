---
title: Intel QAT ZIP初步分析
tags:
  - ZIP
  - QAT
description: 本文分析Intel QAT技术对应软件栈的支持，主要的关注点在压缩解压缩的软件栈，本文只是寻找资料时候的一个笔记，还很粗糙。
abbrlink: 24427f5f
date: 2021-06-27 18:04:39
---

1. 基本信息 
-----------

 QAT的官网: https://www.intel.cn/content/www/cn/zh/architecture-and-technology/intel-quick-assist-technology-overview.html

 QAT相关的代码在01.org网站: https://01.org/zh/intel-quickassist-technology?langredirect=1

 基本QAT相关的用户手册，代码都可以从上面的URL获得。

2. 用户APP、QATzip、libqat库的关系
----------------------------------

 我们这里看看QAT里支持的压缩解压缩是怎么最终叫用户APP使用到的。

 QAT整个压缩解压缩的软件栈由: 内核驱动，libqat用户态基础库，QATzip用户态库组成。
 我们这里以ceph作为APP，一起看下，ceph里的压缩解压缩可以直接调用QATzip库提供的
 接口使用QAT的硬件压缩解压缩引擎。

 以上软件的位置在：1. QAT的内核态驱动和libqat用户态基础库合在一起放在上面的01.org
 的这个包里：Intel® QuickAssist Technology Driver for Linux* - HW version 1.7.
 2. QATzip的代码在：https://github.com/intel/QATzip. 3. ceph的代码：github.com/ceph.

 整个调用链的逻辑是：

 1. 内核态QAT crypto驱动除了向crypto子系统上注册外，也会向内核UIO子系统注册，
    通过UIO把QAT的硬件资源暴露给用户态。由于UIO存在安全上的问题，可以看到主线
    内核里注册到UIO的驱动很少，这也是QAT内核驱动中注册UIO这部分无法上传到主线
    的原因。

 2. libqat用户态基础库封装UIO用户态接口，向上提供一组基础的API。这个在01.org
    网站的接口说明文档中有介绍。

 3. QATzip这个库调用libqat API对外提供QAT的压缩解压缩基本接口。提供的接口在
    QATzip/include/qatzip.h这个头文件中。

 4. Ceph代码里压缩解压缩的部分ceph/src/compressor/有QatAccel.cc, 这部分代码
    调用QATzip的接口封装ceph里的压缩解压缩接口供同目录下的zlib/zlibCompressor.cc
    使用。(目前竟然是HAVE_QATZIP这个宏隔开的 :( )

3. 接口分析
-----------

 1. libqat有一个qat_service的服务，这个服务可能和/dev/qat_adf_ctl的这个字符设备
    配合提供一些管理工作。

 2. qat卡可以load不同的固件，从而改变qat卡自身的功能。
