---
layout: post
title: "tmpfs: file system with all things in mem/swap (linux, tmpfs)"
author: "melon"
date: 2023-12-16 22:48
categories: "2023"
tags:
  - linux
  - kernel
  - todo
---

tmpfs is a fs which keep all files in virtual memory.

everything in tmpfs is temporary in the sense that no files will be created on your hard drive.
if you unmount a tmpfs instance, everything stored there is lost.

tmpfs put everything into the kernel internal caches and grows and shrinks to accommodate the files it contains
and is able to swap unneeded pages out to swap space.
it has max size limit which can be adjusted on the fly via:

```text
$ mount -o remount ...
```

when comparing tmpfs with ramfs (the template to create tmpfs), the tmpfs support swap and limit check.
when comparing tmpfs with ram disk (/dev/ram*), intend to simulate a fixed size hard disk in physical ram,
where you have to create an ordinary fs on top.
ramdisk cannot swap and you do not have the possibility to resize them.

since tmpfs lives completely in the page cache and on swap.
all tmpfs pages are shown as Shmem in /proc/meminfo and Shared in free cmd ouput.
note: the counters of /proc/meminfo & free include shared mem part also, thus the most reliable way
to get net count for tmpfs is df and du.

<hr>

### # tmpfs usecases
$ 1 there is always a invisible kernel internal mount, which is used for shared anonymous mappings and system-v shared mem.
this annonymous mount doesnt strictly depend on CONFIG_TMPFS.
if CONFIG_TMPFS not set, the user visible part of tmpfs is not build, while internal mechanism are always present.

$ 2 in glibc>=2.2 expect tmpfs to be mounted at /dev/shm for posix shared mem (shm_open, shm_unlink),
which can be done by adding lines in /etc/fstab:

```text
$ cat /etc/fstab
# /etc/fstab
# Created by anaconda on Thu Dec 12 14:18:51 2019

# uuid to ensure the correctness of partition mounted regardless of device name
# enable level-1 dump command backup: 1
# check the fs in first order when boot: 1

UUID=2d42bb95-2009-9a5ef58ff6bc    /           ext4    defaults    1 1
/dev/vdb1                          /repo1      ext4    defaults    0 0
tmpfs	                           /dev/shm    tmpfs   defaults    0 0   <--- added lines for tmpfs
```

note: remember to create target dir that you intend to mount tmpfs on.
the mount is not needed for system-v shared mem, while the internal mount is used for that.
in the kernel=2.3, it was possible to mount the predecessor of tmpfs (shm fs) to use system-v shared mem.

$ 3 its very convenient to mount tmpfs on /tmp and /var/tmp (big partition typically), benefits are as:  
3.1 enable fast /tmp file access for software;  
3.2 automatically cleanup after usage;  
3.3 reduce disk i/o when app access /tmp;  
3.4 dynamic resize: the tmpfs is dynamically adjusted according to ram left;  
3.5 compaitible with intiramfs;  
3.6 big swap partition: allow swap from ram to disk when system in heavy load.

loop mount of tmpfs file do work (mount a file of type tmpfs as a block device), thus mkinitrd shipped by most
distribution should succeed with a tmpfs /tmp.
the mkinitrd is a tool to create initial ram disk (initrd), which is loaded in mem for linux boot.

<hr>

### # reference
https://www.kernel.org/doc/html/v5.9/filesystems/tmpfs.html  
ramfs, rootfs, initramfs see: https://docs.kernel.org/filesystems/ramfs-rootfs-initramfs.html.
