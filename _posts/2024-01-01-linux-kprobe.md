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

### # printk vs tracepoint vs kprobe (kernel debugging)
1) printk: easy to use but not easy & smart for debug issues.  
2) tracepoint: easier for debug, but rely on static anchor, if you want somewhere else,
the tracepoint need to be relocated, then recompile the kernel for activation.  
3) kprobe: support inserting dynamic anchor in a running kernel for the execution of
pre-defined dynamic trace func on a compiled kernel.

<hr>

### # kprobe handlers
1) pre-handler: exec before probed instruction, for dump register content before anchor.  
2) post-handler: exec after the probed instruction, for dump register content after anchor.  
3) fault-handler: exec when certain fault occurs in pre/post-handler or the instruction under debugging.

<hr>

### # kprobe implementation details
kprobe is basically a combination of breakpoint + single-step, like kgdb and gdb:  
1) register kprobe: kprobes registered is related to a struct, to record anchor pos,
original_opcode.  
2) replace original_opcode: when enable kprobe, replace all anchor with exception instruction
(BRK), let cpu trap into exception mode.  
3) during exception, cpu exec in single-step mode, do pre_handler, set related reg
(set original_opcode as next intruction), finally return from exeception mode.  
4) exec original_opcode.  
5) after original_opcode finished, cpu trap into exception mode again
(due to reg set by single-step), then clear the related reg, exec post_handler,
finally return from exception mode.

exec path: kprobe->pre_handler => origin instruction => kprobe->post_handler.

<hr>

### # different arch has different kprobe design
1) mips kprobe design: mips contains break instruction, but does not have hardware
single-step support. mips single-step feature is implemented in software using break
with different opcode value, related code and function could be found at asm-mips/break.h.  
2) arm kprobe design: arm use undefined instructions for kprobe, which cannot be decoded
by processor without participation of coprocessor.  
3) ppc32 kprobe design: ppc32 use tw instruction to trap into exception mode,
so in board development, should close /proc/sys/kernel/panic_on_oops to prevent board reboot
when debugging oops problems.

<hr>

### # how to write kprobe module & use it?
kprobe has two kinds of user interfaces: kernel module and debugfs.

1 kernel module interface (ko)  
kernel src code dir samples/kprobes has many kprobe usecases, we could use them as template
for writing our own kprobe module. the main steps to implement a ko for kprobe are as:

1) write a declaration of kprobe struct, then define the key member variables:
symbol_name (kernel function name), pre_handler (hook), post_handler (hook)\...  
2) register kprobe as a module by register_kprobe function.  
3) finally, use insmod to load module in kernel.

usecase: linux/samples/kprobles/kprobe_example.c

```text
// Here's a sample kernel module showing the use of kprobes to dump a
// stack trace and selected registers when kernel_clone() is called.
//
// For more information on theory of operation of kprobes, see
// Documentation/trace/kprobes.rst
//
// You will see the trace data in /var/log/messages and on the console
// whenever kernel_clone() is invoked to create a new process.

#define pr_fmt(fmt) "%s: " fmt, __func__

#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/kprobes.h>

static char symbol[KSYM_NAME_LEN] = "kernel_clone";                         // for fork, vfork, clone
module_param_string(symbol, symbol, KSYM_NAME_LEN, 0644);                   // params to select func for probe
                                                                            // e.g. insmod mymodule.ko symbol=printk
static struct kprobe kp = {                                                 // specific for each probe
    .symbol_name	= symbol,
};

static int __kprobes handler_pre(struct kprobe* p, struct pt_regs* regs){   // pre handler
#ifdef CONFIG_X86
    pr_info("<%s> p->addr = 0x%p, ip = %lx, flags = 0x%lx\n", p->symbol_name, p->addr, regs->ip, regs->flags);
#endif
    // call dump_stack() here will give a stack backtrace
    return 0;
}

static void __kprobes handler_post(struct kprobe* p, struct pt_regs* regs, unsigned long flags){  // post handler
#ifdef CONFIG_X86
    pr_info("<%s> p->addr = 0x%p, flags = 0x%lx\n", p->symbol_name, p->addr, regs->flags);
#endif
}

static int __init kprobe_init(void){                                        // init ko
    kp.pre_handler = handler_pre;
    kp.post_handler = handler_post;

    int ret = register_kprobe(&kp);
    if(ret < 0){
        pr_err("register_kprobe failed, returned %d\n", ret);
        return ret;
    }
    pr_info("Planted kprobe at %p\n", kp.addr);
    return 0;
}

static void __exit kprobe_exit(void){                                       // exit ko
    unregister_kprobe(&kp);
    pr_info("kprobe at %p unregistered\n", kp.addr);
}

module_init(kprobe_init)
module_exit(kprobe_exit)
MODULE_LICENSE("GPL");
```

after loading the above ko to kernel, the kprobe handler is invoked before or after new
process is created by clone, which can be tested by creating new shell process (ls):

```text
$ ls
$ pre_hander: p->addr = 0x***, ip = ****.
$ blog        dockerfile
$ post_handler: p->addr = 0x***, pc = ****.
```

<p style="margin-bottom: 20px;"></p>

2 debugfs  
while in some situations, loading kernel module is not convenient for some embedded system
without gcc, as we need to cross-compile ko, then copy ko to target board, then insmod.
so we can resort to debugfs (ftrace), which provide a simple way to register, enable,
disable kprobe.

1) mount debugfs

```text
$ mount -t debugfs nodev /sys/kernel/debug
```

2) register kprobe event:

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

3) enable the registered kprobe:

```text
$ cd /sys/kernel/debug/tracing/events/kprobes/events/sys_write_event
$ echo 1 > enable
$ cd /sys/kernel/debug/tracing
$ cat trace
...
bash-808   [003] d... 42715.347565: sys_write_event: (SyS_write+0x0/0xb0)
...
```

process name is bash, pid is 808, at 42715.347565s after bootup, call the probed function
sys_write once.

4) disable registered kprobe:

```text
$ cd /sys/kernel/debug/tracing/events/kprobes/events/sys_write_event
$ echo 0 > enable                                                     # shutdown kprobe
$ cd /sys/kernel/debug/tracing/events
$ echo "-:kprobes/sys_write_event" >> kprobe_events                   # un-register kprobe
```
