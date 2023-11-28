---
layout: post
title: "kernel lock mechanism (linux kernel architecture ch5)"
author: "melon"
date: 2023-09-21 08:09
categories: "2023"
tags:
  - linux
  - todo
---

### # introduction
Kernel has unrestricted access to the full address space. On multiprocessor system, if several processors are in kernel mode at the same time, this will cause the race condition problem.

In the first SMP system, only one processor could be kernel mode. Recently kernel use fine-grained network of locks for explicit protection of certain data structures. 

<hr>

### # kernel lock mechanisms
1) Atomic Operations — These are the simplest locking operations. They guarantee that simple operations, such as incrementing a counter, are performed atomically without interruption even if the operation is made up of several assembly language statements.

2) Spinlocks — These are the most frequently used locking option. They are designed for the short- term protection of sections against access by other processors. While the kernel is waiting for a spinlock to be released, it repeatedly checks whether it can acquire the lock without going to sleep in the meantime (busy waiting). Of course, this is not very efficient if waits are long.

3) Semaphores — These are implemented in the classical way. While waiting for a semaphore to be released, the kernel goes to sleep until it is woken. Only then does it attempt to acquire the semaphore. Mutexes are a special case of semaphores — only one user at a time can be in the critical region protected by them.

4) Reader/Writer Locks — These are locks that distinguish between two types of access to data structures. Any number of processors may perform concurrent read access to a data structure, but write access is restricted to a single CPU. Naturally, the data structure cannot be read while it is being written.
