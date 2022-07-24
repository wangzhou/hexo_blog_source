---
title: Add CentOS yum repo in Redhat system
tags:
  - 包管理
  - RPM/YUM
description: This document shows how to add yum repo resource for a Redhat/CentOS system
abbrlink: a3197202
date: 2021-07-05 22:30:31
categories:
---

In Redhat system, you should buy it to use. However, CentOS is
free and almost same with related Redhat system. So after we
add the CentOS yum repo, then we can directly use Redhat system.
(If you already have a CentOS, please use it directly)

We can add the repo configuration file in /etc/yum.repos.d/ like:
```
# CentOS-Base.repo
#
# The mirror system uses the connecting IP address of the client and the
# update status of each mirror to pick mirrors that are updated to and
# geographically close to the client.  You should use this for CentOS updates
# unless you are manually picking other mirrors.
#
# If the mirrorlist= does not work for you, as a fall back you can try the 
# remarked out baseurl= line instead.
#
#

[base]
name=CentOS-$releasever - Base
baseurl=http://mirror.centos.org/altarch/$releasever/os/$basearch/
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
       file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7-aarch64

#released updates 
[updates]
name=CentOS-$releasever - Updates
baseurl=http://mirror.centos.org/altarch/$releasever/updates/$basearch/
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
       file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7-aarch64

#additional packages that may be useful
[extras]
name=CentOS-$releasever - Extras
baseurl=http://mirror.centos.org/altarch/$releasever/extras/$basearch/
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
       file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7-aarch64
```
but it also goes wrong.

here we should replace the variable "$releasever" and "$basearch" to the
specific one which we want to use. you can search in http://mirror.centos.org/altarch/
to see which repo you will use.

In my case, I will use:
```
[base]
name=CentOS-base
baseurl=http://mirror.centos.org/altarch/7.4.1708/os/aarch64/
```
and I also remove gpgcheck related lines.

It works fine with me :)
