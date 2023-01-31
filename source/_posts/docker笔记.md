---
title: docker笔记
tags:
  - 虚拟化
  - 容器
description: docker的三个最基本的概念是镜像(Image)，容器(Container)和仓库(Repository)。本文简单介绍有关他们的基础操作。
abbrlink: 4417b09c
date: 2021-07-11 23:33:02
categories:
---

docker的三个最基本的概念是镜像(Image)，容器(Container)和仓库(Repository)。本文
简单介绍有关他们的基础操作。

Image
--------

Image可以看成是一个配置过的发行版(e.g. 带apach配置的ubuntu发行版)。我们可以自己
生成一个镜像，或者是直接下载已有的镜像使用。比如下载最新的ubuntu docker iamge:
```
wangzhou@EstBuildSvr1:~$ docker pull ubuntu
Using default tag: latest
latest: Pulling from library/ubuntu
8aec416115fd: Pull complete 
695f074e24e3: Pull complete 
946d6c48c2a7: Pull complete 
bc7277e579f0: Pull complete 
2508cbcde94b: Pull complete 
Digest: sha256:71cd81252a3563a03ad8daee81047b62ab5d892ebbfbf71cf53415f29c130950
Status: Downloaded newer image for ubuntu:latest
```
注意上面下载的Image和ubuntu官方发行版还是有一定的不同的, 希望一直用docker image
的同学要对这个问题有准备。

查看系统中已有的docker image可以使用docker images, 比如：
```
wangzhou@EstBuildSvr1:~$ docker images
REPOSITORY              TAG                 IMAGE ID            CREATED             SIZE
ubuntu                  latest              f49eec89601e        3 weeks ago         129.5 MB
ubuntu                  14.04.1             4bf30b29cec4        4 weeks ago         284.6 MB
ubuntu                  14.04               3f755ca42730        8 weeks ago         188 MB
compile/ubuntu/server   14.04               09637d6f7c6f        3 months ago        540.2 MB
<none>                  <none>              bd3d4369aebc        5 months ago        126.6 MB
build/ubuntu/server     14.04               ab01bc6b6a57        7 months ago        1.663 GB
```

container
------------

假设我们现在已经有了ubuntu的image，可以运行以下的命令在容器中运行这个镜像。

sudo docker run -t -i ubuntu:latest /bin/bash

在容器中:
```
root@518aa47b44c4:~#   /* here 518aa47b44c4 is the ID of this container */
```
ubuntu的环境下可以用apt-get安装相应的软件：
apt-get update
then install basic programs: ping, vim, mutt, fetchmail, msmtp, procmail

但是，上面的改动只是在容器中的，容器退出我们的改动就不存在了，所以，容器退出后
要把相关的改动提交，从而形成一个新的docker image, 下次我们在容器里运行这个image,
就包含了这次提交的信息。具体如下节。

build a docker image
-----------------------
```
wangzhou@EstBuildSvr1:~$ docker commit -m "Add: basic tools: vim, mutt, fetchmail, procmail, msmtp" -a "Docker basic" 518aa47b44c4 ubuntu/latest
sha256:c8ea18e453cf756dcbb5acaa21a96e78baf18f7b7fbdd5cf8b44edd606702bf0
wangzhou@EstBuildSvr1:~$ docker images
REPOSITORY              TAG                 IMAGE ID            CREATED             SIZE
ubuntu/latest           latest              c8ea18e453cf        14 seconds ago      301.3 MB
ubuntu                  latest              f49eec89601e        3 weeks ago         129.5 MB
ubuntu                  14.04.1             4bf30b29cec4        4 weeks ago         284.6 MB
ubuntu                  14.04               3f755ca42730        8 weeks ago         188 MB
compile/ubuntu/server   14.04               09637d6f7c6f        3 months ago        540.2 MB
<none>                  <none>              bd3d4369aebc        5 months ago        126.6 MB
build/ubuntu/server     14.04               ab01bc6b6a57        7 months ago        1.663 GB
```
下次再运行image的时候，可以指定image的ID运行，可以用repo:tag的方式运行, 比如：
```
sudo docker run -t -i c8ea18e453cf7
sudo docker run -t -i ubuntu/latest
sudo docker run -t -i ubuntu/latest:latest
```
更改docker image的tag:
```
sudo docker tag 6b46dcca842d docker_test/with_net_tools:add_ping_change_tag
sherlock@T440:~/notes$ sudo docker images
REPOSITORY                   TAG                   IMAGE ID            CREATED             SIZE
docker_test/with_net_tools   add_ping              6b46dcca842d        4 hours ago         170.9 MB
docker_test/with_net_tools   add_ping_change_tag   6b46dcca842d        4 hours ago         170.9 MB
docker_test/with_net_tools   latest                4b48ccb6135c        4 hours ago         167.2 MB
ubuntu                       latest                f753707788c5        2 weeks ago         127.2 MB
ubuntu                       12.04                 6e0ef8cc1b8a        12 months ago       136 MB
```

从一个容器退出后，容器还没有彻底消亡。用sudo docker ps -a 可以查看现在系统里的容器,
STATUS一栏显示容器的状态，'Up XXX days'表示这个容器已经运行XXX天了，现在还在运行，
Exited (0) XXX days ago表示这个容器现在是退出状态。对于退出状态的容器，可以
docker start -i 容器id重新启动这个容器。对于运行状态的容器，可以docker attach 容器id
继续接入这个容器, 这时接入这个容器的所有终端将同步显示容器中运行的程序。
```
/* remove a contrainor */
sudo docker rm
/*
 * remove a image, if a image was used by a stopped container, should firstly
 * remove related stopped container
 */
sudo docker rmi
```
upload a docker image
------------------------

为了保存，传播image，我们可以把一个docker image上传到DockerHub的仓库。当然首先
要在DockerHub上注册有账户。在上传之前，先在本机上用docker login登录下DockerHub
账户, 用户名、密码用对答的方式输入。然后就是运行：docker push repo_name[:tag]
的方式上传了。值得注意的是，repo_name要用DockerHub_account/XXX的形式, 不然可能
无法上传。
