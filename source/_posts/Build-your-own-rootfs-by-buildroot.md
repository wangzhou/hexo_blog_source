---
title: Build your own rootfs by buildroot
tags:
  - 软件调试
  - rootfs
description: 本文简单记录下编译rootfs的过程
abbrlink: 962c911d
date: 2021-06-20 23:16:36
---

1. git clone git://git.buildroot.net/buildroot

2. cd buildroot

3. make defconfig

4. make menuconfig to choose special tools, e.g. numactl

5. make menuconfig to choose BR2_TARGET_ROOTFS_CPIO=y and
   Compression method to gzip.

6. make menuconfig to choose your arch, e.g. my arch is BR2_aarch64
```
   Note: if you choose a wrong arch, after changing to right arch config,
         you should make clean, than make, otherwise you may meet something
	 wrong like: "No working init found".
```

7. wait and you will find built minirootfs in buildroot/output/images/
   you can use rootfs.cpio.gz as rootfs here.

Note: 编译glib(BR2_PACKAGE_LIBGLIB2), 依赖BR2_TOOLCHAIN_BUILDROOT_WCHAR, BR2_USE_WCHAR

如上是之前编译buildroot的一个笔记，其实buildroot在交叉编译构建文件系统的情况还是
非常好用的，因为buildroot里包含了很多基本库，如果你的app依赖了第三方的库，在交叉编译
的时候，一般你要先交叉编译依赖的库，然后再交叉编译app的时候链接之前交叉编译出来的
库，一两个这样的依赖库还好，要是有比较多的依赖库就会比较麻烦。如果使用buildroot，
只要在buildroot配置的时候打开依赖库的配置就好。

自己的app如何集成在buildroot编译生成的小系统里。一个好用的方法是使用buildroot的
override功能。它的基本逻辑是，现在buildroot的编译配置体系里加如你想编译的包的配置，
如下的patch中，我们加了一个叫devmmu的包的配置，这个包的配置基本是空的，然后我们
在buildroot的根目录下发放一个local.mk的文件，并在里指明devmmu这个包的源码目录，
相关的写法一定要按照<package_name>_OVERRIDE_SRCDIR = <package_path>的写法。
全部配置好后，在buildroot make menuconfig的时候把devmmu选上，make编译的时候就会
去指定的目录里找devmmu的代码，并编译安装到生成的小文件系统里。如果后面改了devmmu
的源码，使用make devmmu-rebuild all可以只便宜安装devmmu。
```
diff --git a/local.mk b/local.mk
new file mode 100644
index 0000000000..2775c16f6b
--- /dev/null
+++ b/local.mk
@@ -0,0 +1 @@
+DEVMMU_OVERRIDE_SRCDIR = /home/wangzhou/devmmu-user/
diff --git a/package/Config.in b/package/Config.in
index 82b28d2835..bcb1649da7 100644
--- a/package/Config.in
+++ b/package/Config.in
@@ -2524,4 +2524,6 @@ menu "Text editors and viewers"
 	source "package/vim/Config.in"
 endmenu
 
+source "package/devmmu/Config.in"
+
 endmenu
diff --git a/package/devmmu/Config.in b/package/devmmu/Config.in
new file mode 100644
index 0000000000..4c7f9edcc4
--- /dev/null
+++ b/package/devmmu/Config.in
@@ -0,0 +1,4 @@
+config BR2_PACKAGE_DEVMMU
+	bool "devmmu"
+	help
+		DevMMU testsuit
diff --git a/package/devmmu/devmmu.mk b/package/devmmu/devmmu.mk
new file mode 100644
index 0000000000..c02eedf5c3
--- /dev/null
+++ b/package/devmmu/devmmu.mk
@@ -0,0 +1,12 @@
+################################################################################
+#
+# devmmu
+#
+################################################################################
+
+DEVMMU_VERSION = 0.1
+DEVMMU_SOURCE=
+DEVMMU_INSTALL_STAGING=NO
+DEVMMU_DEPENDENCIES=
+
+$(eval $(autotools-package))
```
