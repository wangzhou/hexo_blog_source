---
title: Some tips to help to boot an installed Linux distribution manually
tags:
  - UEFI
description: >-
  If your Linux system fails to boot up normally, this may be a way to boot it
  up from UEFI command manually
abbrlink: 4014887b
date: 2021-07-05 22:32:24
categories:
---

1. enther UEFI shell: D06>

2. enter "start fs1:\EFI\redhat\grubaa64.efi"

   this step is to load grubaa64.efi. fs1 here is the partition
   in which there is a grubaa64.efi. you maybe need to try other
   partitions like: fs0, fs1, fs2...

   for linux, the EFI partition will be mounted under /boot/efi/

3. then you will enter the shell of grub: grub>
   (or your grub will load grub.conf directly, then you can directly
    go into grub menu)

4. find where is grub.cfg in grub shell, then enter:

   configfile grub.cfg
   (my case is: configfile (hd1,gpt1)/efi/redhat/grub.conf)

   in this step, you may need go around to see where is the grub.conf,
   you can use "ls" in grub shell, and "tab" works also in grub shell,
   path can be completed by "tab"

   then enter grub menu

5. choose which kernel to boot
