---
title: hns3各个模块的关系
tags:
  - Linux内核
  - 软件架构
description: 分析一个问题，这个问题和hns3有关系，这里分析下hns3各个模块的逻辑，以及主要 数据结构的逻辑。分析完全依赖Linux主线代码v6.8-rc5。
abbrlink: 29971
date: 2024-04-23 19:24:48
categories:
---

模块
-----

HNS3是海思Hipxxx系列Soc的网络加速器驱动，编译成模块后有hclge，hclgevf，hnae3，
hns3这几个内核模块组成。

hnae
-----

hnae模块只是提供了一组注册接口出来，其它的模块可以利用这组注册接口注册ae_dev/ae_algo/
client进来，hnae模块里分别用hnae_ae_dev_list/hnae_ae_algo_list/hnae_client_list
做对应对象的管理。

ae_dev代表一个具有某种网络功能的设备，client表示一个ae_dev上的一组功能，比如，对
于一个网口(ae_dev)，以太网口(NIC)和RoCE就是其上的两个client，NIC和RoCE复用这个ae_dev
的底层硬件资源，但是从业务层面看又是相互独立的。

hnae3_register_ae_dev把一个ae_dev向hnae注册，并检测是否有ae_algo与之匹配，若匹配
则调用算法中对应的init_ae_dev函数初始化ae_dev，同时检测是否有client和ae_dev匹配，
若匹配则调用ae_algo中的init_client_instance函数初始化client。

hnae3_register_ae_algo和如上设备注册的接口一样，也有一样的和设备以及client匹配检测、
以及对应的初始化流程。

hnae3_register_client也和设备注册的接口一样，也有ae_dev<->client匹配检测以及对应
client的实例初始化流程。注意，client和client的实例是不同的。

总的来看，hnae是一个核心的库，是模块依赖的最底层，其它模块不remove，hnae不会被
remove掉。

hclgevf/hclge
--------------

hclgevf/hclge的初始化接口只做了一件事，就是把这个模块中定义的ae_algo注册到hnae。

hclgevf/hclge通过hnae的注册接口把一堆回调函数提供给了ae_dev和client，而hclgevf/
hclge是可以随时卸载的，如果ae_dev或者client还在使用这些回调函数，hclgevf/hclge又
被卸载掉，这里就会出问题。hnae_unregister_ae_algo里做了针对ae_dev和client的uninit
操作，在hclgevf/hclge离开的时候做对应的清理，就不会有问题。

总的来看，hclgevf/hclge就是一堆傻傻的回调，它们只是被驱使，自己并不能决定自己的
命运，只能在hclgevf/hclge的register/unregister里做对应的处理。

hns3
-----

hns3整体上是一个标准的PCIe设备驱动。在hns3模块的init里使用hnae3_register_client
注册client，注意这个发生在pci_register_driver之前，因为pci_register_driver之后，
触发的probe才注册ae_dev设备，所以init里client注册逻辑只把client注册到hnae里的链表，
并不会触发ae_dev和client的匹配。

注意，这里的client并没有实例的概念，只是一种网络服务的描述，比如NIC client结构里
描述的都是以太网口相关的操作函数。所以，放在hns3驱动init里注册是合理的。

这个PCIe驱动的probe函数里使用hnae3_register_ae_dev注册ae_dev。由如上可知，这个过
程会初始化ae_dev，以及创建client的实例。

深入client实例创建过程看下：
```
  hnae3_init_client_instance
    +-> ae_dev->ops->init_client_instance // 注意，这个在hclge里定义。
      +-> hclge_init_nic_client_instance  // switch区分NIC和ROCE。
        +-> client->ops->init_instance    // 又调会client里的回调。
```
初始化一个client的实例，client难道提供不了足量的信息？没有理解为什么要走到hclge。

基本的数据结构是：
```
pci_dev->priv(ae_dev)->priv(hclge_dev)->vport  
                                          |
                                          +-- nic(struct hnae3_handle)
                                          |
                                          +-- roce(struct hnae3_handle)
```

总结下hns3的逻辑。hns3是一个PCIe设备驱动，所谓ae_dev是为了抽象而多加的一层，client
实例是hns3和NIC业务相关的封装，ae_algo是PCIe设备驱动本来就有的业务相关的代码，只
不过为了抽象，把它们放到ae_algo，然后独立成模块。

hns3和hclge独立成模块后，hns3可以独立hclge做remove，有可能出现hns3已经remove，
但是hclge中的回调还在运行。但是hclge中的回调只是被动的调用，控制逻辑应该在hns3中，
也就是说hns3在remove的时候(ae_dev/client remove)，应该保证：1. 拒绝外部流量，2.
排空内部流量，3. 停住对应硬件。这样，应该不会出现hns3 remove掉，hclge中的回调还
在运行的情况。

RoCE
-----

RoCE对应的模块是hns_roce_hw_v2，在这个模块的init通过hnae3_register_client对hnae
注册一个client，注册过程触发对应client实例的初始化。
