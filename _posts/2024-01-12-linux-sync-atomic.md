---
layout: post
title: "syncronization: atomic (linux, sync)"
author: "melon"
date: 2024-01-12 20:12
categories: "2024"
tags:
  - linux
  - sync
  - todo
---

atomic operation is a set of instructions that are executed without interrupts
physically or logically.
moreover, read and write to the memory involved in atomic operation is forbidden.

<hr>

### # features of an atomic operation
1 atomicity: ensures that an operation completes without any interference from other threads.  
2 visibility: changes made by one thread are visible to other threads.  
3 ordering: ensures the order of a operation sequence execution.  
4 level: machine instruction level atomic, asm instruction level atomic,
application code level atomic. application code or asm code are atomic or not
is up to the atomic of translated machine code.

<hr>

### # what is a atomic operation (todo: refine the title?)
in view of machine instruction level, most machine instructions are non-atomic,
and only some machine instructions are atomic (inherently atomic machine instructions).

<hr>

### # the relationship between "atomic" and "interruptable" machine instructions:
1 for single core system, non-interruptible machine instructions are atomic,
and interruptible machine instructions are not necessarily atomic.  
2 for multicore system, machine instructions interruptible or not are both not
necessarily atomic.

for example: a non-interruptible instruction is executed on the cpu core#1,
but the intermediate state of the instruction can be observed by the instruction
executed on the cpu core#2, which break the atomicity.

<hr>

### # lock with atomic operation (example needed)
locking can atomize non-atomic operation. there are mainly two ways for this:  
1 memory locking: during execution of the prefixed non-atomic operation,
a lock signal get declared on memory bus to lock the memory to prevent r/w by rest of world,
which enable atomic in very large cost.  
2 cache locking: during execution of the prefixed non-atomic operation,
the cache line involved in the lock is locked, and other cpu cores cannot access
the corresponding cache line in the cache.

<hr>

### # cache coherency protocol with atomic operation
cache coherence protocols (e.g. MESI) can only be used to ensure cache coherence
within multiple cpu cores, but cannot guarantee atomicity of instruction operations.

cache coherence only guarantees the consistency between SMP system cpu caches,
but wont lock the memory/cache line for certain operation, thus enable operation from
other cpu to access the same mem/cache to get & use the same variable, which break
the atomicity.

<hr>

### # lock mechism with atomic operation
a lock must be used to achieve atomic operations, but using locks cannot guarantee
atomic operations:  
1 on asm level: LOCK command prefix + CMPXCHG asm instruction to form a CAS
atomic instrcution.  
2 machine code level: implicit lock for native atomic machine code operation.

<hr>

### # kernel implementation of atomic
ref: https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/atomic.h
