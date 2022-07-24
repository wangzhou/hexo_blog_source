---
title: rpm yum使用笔记
tags:
  - 包管理
  - RPM/YUM
description: This document is used to collect and share rpm/yum related commands
abbrlink: cf4b4360
date: 2021-07-05 22:29:58
categories:
---
rpm
---
  1. install a rpm package: rpm -ivh *.rpm

  2. query something:

       rpm -q kernel  --> query a package name kernel.

       rpm -qf file   --> query a file to find which package it belogs to
                          so we can use this to find the package of one shell
                          command, like: rpm -qf /usr/sbin/lspci, it will get:
                          pciutils-3.5.1-2.el7.aarch64.

       rpm -ql package -> list all files in this package.

       rpm -qa        --> query all installed package in system.

       rpm -qi package -> list related info of this package.

       rpm -qc package -> find config file of this package, e.g.
                          rpm -qc openssh-clients-xxxx, will get
                          /etc/ssh/ssh_config.

       rpm -qd package -> find related document.

       rpm -qR package -> find related required libs.

yum
---
       yum search kernel      --> search in yum datebase.

       yum provides software  --> find which package contains this software in
                                  yum database. similar with rpm -qf file, but
                                  rpm searchs packages locally.
                                  yum provides fio, we will get:

	Loaded plugins: fastestmirror
	Loading mirror speeds from cached hostfile
	fio-3.1-1.el7.aarch64 : Multithreaded IO generation tool
	Repo        : epel

       yum info package       --> show information of one package
       				  e.g. yum info kernel will show all kernel
				       packages with different versions

       yum list package       --> list all version of this package


rpmbuild
--------

	1. from source to build rpm package

		...

	P.S. For linux kernel, you can directly run: "make rpm" to do so,
	     it will create kernel/kernel-dev/kernel-header rpm packages.

	2. downlaod a rpm, modify it, and re-build a new rpm package

		...

download rpm
------------

	This document shares the way to download rpm package using yum tools:
	https://www.ostechnix.com/download-rpm-package-dependencies-centos/

	"yum install --downloadonly" will also install package.

	"yumdownloader" just downloads the package.

download source
---------------

       1. yum install yum-utils

          /* should be the name of package, just showed when yum search XXX */
       2. yumdownloader --source xorg-x11-drv-ati

       3. yum install mock
          useradd -s /sbin/nologin mockbuild

       4. rpm -hiv xorg-x11-drv-ati-7.7.1-3.20160928git3fc839ff.el7.src.rpm
          it will create ~/rpmbuild/SOURCE and install above package there.

       5. cd /root/rpmbuild/SOURCES

       6. xz -d *
          tar -xf *
