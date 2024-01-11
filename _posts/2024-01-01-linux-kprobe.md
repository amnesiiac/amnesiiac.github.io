---
layout: post
title: "kprobe: dynamic anchor with self-defined handler (linux)"
author: "melon"
date: 2024-01-01 19:16
categories: "2024"
tags:
  - linux
  - kernel
---

### # comparison of some kernel debugging methods
1 printk(), simple but not smart & easy.  
2 tracepoint, somthing easies, but rely on static anchor on proc switching\.\.\., if you want somewhere else,
the tracepoint need to be settled by yourself, and recompile the kernel.  
3 kprobe, enable insert dynamic anchor in a running kernel, to execute pre-defined dynamic trace func on compiled kernel.

<hr>

### # handler type of kprobe
1 pre-handler: exec before probed instruction, mainly used to dump register content before the anchor point.  
2 post-handler: exec after the probed instruction, used to dump the register content after the anchor point.  
3 fault-handler: exec when certain fault occurs in pre-handler, post-handler or in the instruction being debugged.

<hr>

### # kprobe implementation principle
kprobe is basically a combination of breakpoint + single-step, like kgdb and gdb.

1 register kprobe: each kprobe registered is corresponding to a struct, mainly to record anchor pos, original_opcode.  
2 replace original_opcode: when enable kprobe, replace all anchor as an exception instruction (BRK), lead cpu trap into exception mode.  
3 in exception mode, use single-step of cpu, do pre_handler, set related reg (exec original_opcode as next intruction), return from exeception mode.  
4 exec original_opcode.  
5 after original_opcode finished, cpu trap into exception mode again (due to reg set by single-step), now clear the related reg, exec post_handler, finally return from exception mode.

the exec path is: kprobe->pre_handler => origin instruction => kprobe->post_handler.

<hr>

### # different arch has different kprobe design
1 mips kprobe design:  
mips contains break instruction, but does not have hardware single-step support. single-step feature is implemented in software using break with different opcode value, related code and function could be found at asm-mips/break.h

2 arm kprobe design:  
arm use undefined instructions for kprobe, which cannot be decoded by processor without participation of coprocessor.

3 ppc32 kprobe design:  
ppc32 use tw instruction to trap into exception mode, so in board development, should close /proc/sys/kernel/panic_on_oops to prevent board reboot when debugging.

<hr>

### # how to use kprobe
kprobe 2 kind of user interfaces: kernel module and debugfs.

1 kernel module interface  
kernel src code dir samples/kprobes has many kprobe use sample code,
we could use them as template for writing our own kprobe module.

taking kprobe_example.c as a case:  
• write a declaration of kprobe struct, then define some key member variables: symbol_name, pre_handler, post_handler\.\.\.
the symbol name is kernel function name. the pre_handler and post_handler are hook functions before exec symbol function
(e.g. in kprobe_example.c it uses do_fork as kernel probe position).  
• register kprobe as a module by register_kprobe function.  
• finally, use insmod to load module in kernel.

hence, we could see that before & after a shell cmd that creates new process (e.g. ls, cat\.\.\.), our designed hook function will be invoked.

• sample output by kprobe_example.c when hitting hook func:
```text
$ pre_hander: p->addr = 0x***, ip = ****.
$ post_handler: p->addr = 0x***, pc = ****.
```
todo: adding code in action examples for this\.\.\.

2 debugfs  
load kernel module sometimes not that convenient for some embedded system without gcc, that is, we need to cross-compile ko, then copy ko to target board, then insmod.
but debugfs (ftrace) provide a simple way to register, enable, disable kprobe.

• mount debugfs
```text
$ mount -t debugfs nodev /sys/kernel/debug
```

• register kprobe event:
```text
$ cd /sys/kernel/debug/tracing
```
inside tracing dir, write following to register kprobe event:
```text
$ echo "p:sys_write_event sys_write" > kprobe_events
```
p: the register event type is kprobe, for retprobe, type should be r.  
sys_write_event: the name for the registered kprobe.  
sys_write: the insert position of the kprobe.

then a new dir named kprobes is created:
```text
$ ls /sys/kernel/debug/tracing/events/kprobes
enable filter sys_write_event
```
we have successfully registered kprobe event.

• enable the registered kprobe:
```text
$ cd /sys/kernel/debug/tracing/events/kprobes/events/sys_write_event
$ echo 1 > enable
$ cd /sys/kernel/debug/tracing
$ cat trace
...
bash-808   [003] d... 42715.347565: sys_write_event: (SyS_write+0x0/0xb0)
...
```
process name is bash, pid is 808, at 42715.347565s after bootup, call the probed function sys_write once.

• disable registered kprobe:
```text
$ cd /sys/kernel/debug/tracing/events/kprobes/events/sys_write_event
$ echo 0 > enable                                                     # shutdown kprobe
$ cd /sys/kernel/debug/tracing/events
$ echo "-:kprobes/sys_write_event" >> kprobe_events                   # un-register kprobe
```
