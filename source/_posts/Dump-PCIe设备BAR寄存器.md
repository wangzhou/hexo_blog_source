---
title: Dump PCIe设备BAR寄存器
tags:
  - 软件调试
  - PCIe
description: >-
  调试的时候需要dump PCIe设备BAR里各个寄存器的内容，一般我们可以读
  /sys/devices/<device_path>/resource[n]这个文件得到。

  Martoni写了一个小软件可以方便resource里内容的读取。这个软件在： https://github.com/Martoni/pcie_debug。

  本文介绍这个工具的使用。
abbrlink: 8674802a
date: 2021-06-27 18:03:34
---

下载后编译即可，可能需要安装readline和curses库。
gcc pci_debug.c -lreadline -lcurses -o pci_debug

这个小工具的用法很简单，看help就可以懂:

pci_debug -s <BDF>
可以进入一个命令行交互界面。
(这个工具有一个bug，就是不支持设备有domain号, 对于有domain号的设备可以打上如
 下的补丁)

输入命令：d addr len 查看地址addr开始的len个单位的数值，这里的单位default数值是
          32bit。

	  q退出工具。

          e改变大小端。
	  ...

这个工具需要sudo或者root权限。

```
diff --git a/pci_debug.c b/pci_debug.c
index f746840..30f4d45 100644
--- a/pci_debug.c
+++ b/pci_debug.c
@@ -177,9 +177,9 @@ int main(int argc, char *argv[])
 	 */
 
 	/* Extract the PCI parameters from the slot string */
-	status = sscanf(slot, "%2x:%2x.%1x",
-			&dev->bus, &dev->slot, &dev->function);
-	if (status != 3) {
+	status = sscanf(slot, "%4x:%2x:%2x.%1x",
+			&dev->domain, &dev->bus, &dev->slot, &dev->function);
+	if (status != 4) {
 		printf("Error parsing slot information!\n");
 		show_usage();
 		return -1;
```
