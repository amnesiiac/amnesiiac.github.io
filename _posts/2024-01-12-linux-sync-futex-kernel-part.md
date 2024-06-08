---
layout: post
title: "syncronization: futex the kernel part (linux, sync)"
author: "melon"
date: 2024-01-12 21:25
categories: "2024"
tags:
  - linux
  - sync
  - todo
---

simply stated, a futex is a queue the kernel manages for userspace convenience.
it lets userspace code ask the kernel to suspend until a certain condition is satisfied,
and lets other userspace code signal that condition and wake up waiting processes.

earlier we've menioned busy-looping as one approach to wait on success of atomic operations;
a kernel-managed queue is the much more efficient alternative, absolving userspace code from
the need to burn billions of cpu cycles on pointless spinning.

in the linux kernel, futexes are implemented in kernel/futex.c. the kernel keeps a hash table
keyed by the address to quickly find the proper queue data structure and adds the calling process
to the wait queue. there's quite a bit of complication, of course, due to using fine-grained locking
within the kernel itself and the various advanced options of futexes.

use cursor to analysis this implementation (todo)
use github part folder download site for this kernel sub module analysis

ref: https://github.com/torvalds/linux/tree/master/kernel/futex
