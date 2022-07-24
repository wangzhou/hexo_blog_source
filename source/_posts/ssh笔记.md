---
title: ssh笔记
tags:
  - shell
description: 本文用来收集ssh使用的一些技巧
abbrlink: 6501f3e2
date: 2021-07-11 23:34:41
categories:
---

我们在日常工作中常常使用ssh连接远程服务器。

一般我们这样使用: ssh user_name@remote_host
其中remote_host可以是远程服务器的域名或者是IP. 之后输入密码就可以登录远程服务器

利用ssh通信的其他命令(e.g. scp)的格式也和上面的相似。

对于想不每次手动输入密码登录, 我们可以把ssh-keygen生成的公钥放到远程服务器对应
home目录下的.ssh/authorized_keys这个文件里。这样直接输入上面的命令就可以登录远程
服务器。

如何把公钥里的内容放到远程服务器的authorized_keys里, 我们可以使用命令:
```
ssh user_name@remote_host "cat >> ~/.ssh/authorized_keys" < ~/.ssh/id_rsa.pub
```
如果服务端的ssh server不在常用的22端口上。我们可以用-p指定所用的端口:
```
ssh -p port_number user_name@remote_host
```
如果远程服务器没有图形化界面，可以使用 ssh -X user_name@remote_host "command"
(e.g. ssh -X user_name@remote_host "thunderbird"), 在远程服务器上运行命令command,
而在本地机器上显示command的图形化输出。
