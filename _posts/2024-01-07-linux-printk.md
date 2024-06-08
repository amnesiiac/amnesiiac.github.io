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

printk is one of the most widely known functions in the kernel.
it’s the standard tool for printing messages and usually the most basic way of
tracing and debugging.

<hr>

### # macro functions: pr_info, pr_debug\...
some macro need to be enable in menuconfig for compilation of some methods:  
ref: https://elixir.bootlin.com/linux/v6.5-rc5/source/include/linux/printk.h#L527

<hr>

### # will printk block the proc?
printk will just try to lock the console (serial port) to print the message,
if the lock cannot be acquired, then the output is queued to ringbuffer.

however, the function invoke the printk will never block. there existed a lock
in the buffer, but it's spinlock, so the printk can also be used in interrupt context.

printk is implemented to be used in anywhere.

<p style="margin-bottom: 20px;"></p>

1 usage of spinlock in buffer from printk is inefficient  
ref: https://lwn.net/Articles/779550/

<p style="margin-bottom: 20px;"></p>

2 mutex vs spinlock  
mutex: when a thread tries to lock a mutex already in locked state,
it will go to sleep immediately to allow another thread to take the cpu.
the process will continue to sleep until got woken up (the mutex is unlocked by other thread).

spinlock: when a thread tries to lock a spinlock but failed,
it will continuously re-try locking it, until it finally succeeds.
thus it will not allow another thread to take its place.
however, the os will forcibly switch to another thread,
once the CPU runtime quantum of the current thread has been exceeded.

<p style="margin-bottom: 20px;"></p>

3 why a spinlock can be enabled in a ISR, but mutex & sleep cannot?  
sleep or mutex will block (execution got stuck & sleep, schedule other thread),
but spinlock wont.

code running in interrupt context is unable to sleep, or block, because interrupt context
does not have a backing process associated to reschedule, so there is nothing for
the scheduler to put to sleep or wake up.
if you sleep or block in ISR, you are slower the whole machine, which is insane.

as a result, interrupt context cannot perform actions to let the kernel putting
current context to sleep: e.g., downing semaphore, copying to/from userspace memory or
non-atomically allocating memory.

<hr>

### # printk output in mess format
problem: only 1 proc can occupy the serial port at the same time, if proc A is taking up
the serial port with lock, and enable printk debugging, thus printk will output logs
both for proc A and other procs to serial port.

reason: proc A is lock to serial port, and cpu is dispatching among proc A, B, C\...
but printk with its buffer inside kernel function are common for each proc.
