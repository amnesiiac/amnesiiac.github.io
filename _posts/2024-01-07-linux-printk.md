---
layout: post
title: "printk: print things interested in kernel (linux)"
author: "melon"
date: 2024-01-07 11:32
categories: "2024"
tags:
  - linux
  - kernel
---

### # introduction
todo

<hr>

### # macro functions: pr_info, pr_debug\.\.\.
some macro need to be enable in menuconfig for compilation of some methods
ref: https://elixir.bootlin.com/linux/v6.5-rc5/source/include/linux/printk.h#L527

<hr>

### # will printk block the proc?
printk will just try to lock the console(serial port) to print the message, if the lock cannot be acquired,
then the output is queued to ringbuffer but the function invoke the printk will never block.
there existed a lock in the buffer, but it's spinlock, so the printk can also be used in interrupt context.

printk is implemented to be used in anywhere.

however, for the spinlock usage in printk to buffer, someone complained as inefficient: https://lwn.net/Articles/779550/

<hr>

### # printk output mess problem
another problem is: only 1 proc can occupy the serial port at the same time, if proc A is taking up the serial port (with lock),
and enable printk debugging, thus printk will output logs both for proc A and other procs to serial port.

reason: proc A is lock to serial port, and cpu is dispatching among proc A,B,C\.\.\.
but printk inside kernel function are common for each proc.
