---
layout: post
title: "ftrace: tracer for kernel function (linux)"
author: "melon"
date: 2023-12-29 20:57
categories: "2023"
tags:
  - linux
  - kernel
---

ftrace is a tracer framework (part of debugfs), used for tracing linux kernel funciton callstacks.

<hr>

### # ftrace principle
ftrace a tracing framework, includes some powerfull tracers for inspect function calls.

before code compiling, it use gcc -pg option to insert a special func mcount() before each function call.
mcount() is implemented in clib, but kernel code dont link clib, that's why gcc -pg feature is utilized.
mcount should be implemented using asm, the caller wont call it by c API.

when compiling, use a script to replace all mcount call as NOP instruction, then record mcount pos in a table.
when user hot enable tracing, will replace NOP with mcount() call.

however, inserting mcount before each function will have bad impact on performance,
so the kernel support dynamic tracing: CONFIG_DYNAMIC_FTRACE.

<hr>

### # basic usages
check whether current kernel support ftrace feature by existence of debugfs:
```text
ls /sys/kernel/debug/tracing/
```

show basic readme about this debugfs
```text
$ cat /sys/kernel/debug/tracing/README
...
```

check available tracers
```text
$ cat /sys/kernel/debug/tracing/available_tracers
blk function_graph wakeup_dl wakeup_rt wakeup function nop
```

show current active tracer
```text
$ cat /sys/kernel/debug/tracing/current_tracer
nop
```

<hr>

### # start & pause tracing
```text
$ echo 0 > /sys/kernel/debug/tracing/tracing_on
$ echo 1 > /sys/kernel/debug/tracing/tracing_on
```

<hr>

### # setup tracer options
todo

<hr>

### # kernel function tracer usages
1 generic way to perform trace debug
```text
echo 1 > tracing_on  && \
setup_tracer_options && \
echo 0 > tracing_on  && \
startup app          && \
echo 1 > tracing_on
```
the above method will output huge bulk of things to trace buffer, which is hard to
safari to our interested part.

2 precise way to perform trace debug
enable kernel function tracer in our interested code segment to enable precise tracing.  
the userspace programs src can handle the trace debugfs system by:
enable trace when entering into function body, and disable it right before leaving.

<hr>

### # trace marker (todo: add practical example)
the marker is used to make search easier to focus on the interested part:
```text
$ echo 'melon_trace' > trace_marker
```

<hr>

### # per_cpu trace debug
per cpu info are in debugfs path as follows (including the trace):
```text
$ ls /sys/kernel/debug/tracing/per_cpu/cpu$i
buffer_size_kb  snapshot  snapshot_raw  stats  trace  trace_pipe  trace_pipe_raw

$ cat /sys/kernel/debug/tracing/per_cpu/cpu$i/trace
```

<hr>

### # tracing certain function call inside kernel code
the principle is the same as above user space code trace part, but kernel dont use echo.
the functions tracing_on() and tracing_off() can be used for switch on/off.
trace_printk() is flexible to print out the interested info.


printk vs trace_printk:
printk write output to console, which consuming a mount of cpu cycle (a few ms),
trace_printk write output to ringbuffer, which cost less cpu cycles (1/10 us),
result in less performance impact.
the output from trace_printk will write to tracer, which can be derived by cat trace.

<hr>

### # ftrace usages
```text
$ echo function > /sys/kernel/debug/tracing/current_tracer
$ cat /sys/kernel/debug/tracing/current_tracer
function
```

check function trace result, log is continously growing fast, ony show top 20 lines.
```text
$ cat trace | head -n 20
# tracer: function
#
# entries-in-buffer/entries-written: 1850399/165139411104   #P:36
#
#                                _-----=> irqs-off
#                               / _----=> need-resched
#                              | / _---=> hardirq/softirq
#                              || / _--=> preempt-depth
#                              ||| /     delay
#           TASK-PID     CPU#  ||||   TIMESTAMP  FUNCTION
#              | |         |   ||||      |         |
           <...>-13267   [034] d... 6368755.192329: check_cfs_rq_runtime <-put_prev_entity
           <...>-13267   [034] d... 6368755.192329: put_prev_entity <-put_prev_task_fair
           <...>-13267   [034] d... 6368755.192329: check_cfs_rq_runtime <-put_prev_entity
           <...>-13267   [034] d... 6368755.192329: put_prev_entity <-put_prev_task_fair
           <...>-13267   [034] d... 6368755.192329: check_cfs_rq_runtime <-put_prev_entity
           <...>-13267   [034] d... 6368755.192329: set_next_task_idle <-pick_next_task_idle
           <...>-13267   [034] d... 6368755.192330: enter_lazy_tlb <-__schedule
          <idle>-0       [034] d... 6368755.192330: __switch_to_xtra <-__switch_to
          <idle>-0       [034] d... 6368755.192330: finish_task_switch <-__schedule
```
some explainations of the ftrace log columns:
taks name, pid, execution cpu num, timestamp for the function call,
function call as A <- B (func B invoke func A)

<hr>

### # get ftrace output for certain CPU
```text
$ cat trace | grep '\[034\]' | head -n 20
           <...>-13267   [034] d... 6368755.192329: check_cfs_rq_runtime <-put_prev_entity
           <...>-13267   [034] d... 6368755.192329: put_prev_entity <-put_prev_task_fair
           <...>-13267   [034] d... 6368755.192329: check_cfs_rq_runtime <-put_prev_entity
           <...>-13267   [034] d... 6368755.192329: put_prev_entity <-put_prev_task_fair
           <...>-13267   [034] d... 6368755.192329: check_cfs_rq_runtime <-put_prev_entity
           <...>-13267   [034] d... 6368755.192329: set_next_task_idle <-pick_next_task_idle
           <...>-13267   [034] d... 6368755.192330: enter_lazy_tlb <-__schedule
          <idle>-0       [034] d... 6368755.192330: __switch_to_xtra <-__switch_to
          <idle>-0       [034] d... 6368755.192330: finish_task_switch <-__schedule
          <idle>-0       [034] .... 6368755.192330: do_idle <-cpu_startup_entry
          <idle>-0       [034] .... 6368755.192331: tick_nohz_idle_enter <-do_idle
          <idle>-0       [034] d... 6368755.192331: ktime_get <-tick_nohz_idle_enter
          <idle>-0       [034] d... 6368755.192331: arch_cpu_idle_enter <-do_idle
          <idle>-0       [034] d... 6368755.192331: tsc_verify_tsc_adjust <-arch_cpu_idle_enter
          <idle>-0       [034] d... 6368755.192331: local_touch_nmi <-arch_cpu_idle_enter
          <idle>-0       [034] d... 6368755.192331: rcu_nocb_flush_deferred_wakeup <-do_idle
          <idle>-0       [034] d... 6368755.192331: tick_check_broadcast_expired <-do_idle
          <idle>-0       [034] d... 6368755.192332: cpuidle_get_cpu_driver <-do_idle
          <idle>-0       [034] d... 6368755.192332: cpuidle_not_available <-do_idle
          <idle>-0       [034] d... 6368755.192332: tick_nohz_idle_stop_tick <-do_idle
```

try to get the process info using the pid printed:

```text
$ ps -p 13267
  PID TTY          TIME CMD
```

ps cmd find nothing, which means the pid in kernel space is different from the user space one shown by top.

<hr>

### # enable tgid display when using ftrace
kernel pid is actually thread id, userspace pid is process id (thread group id).
see blog post tid vs tgid for more details.

try checking the ftrace options, enable it if closed:

```text
$ cat /sys/kernel/debug/tracing/options/record-tgid
0
$ echo 1 > /sys/kernel/debug/tracing/options/record-tgid
```

problem: the col tgid is shown, but no actual digits in it:

```text
$ cat trace | head -n 20
# tracer: function
#
# entries-in-buffer/entries-written: 1850168/942599234198   #P:36
#
#                                          _-----=> irqs-off
#                                         / _----=> need-resched
#                                        | / _---=> hardirq/softirq
#                                        || / _--=> preempt-depth
#                                        ||| /     delay
#           TASK-PID       TGID    CPU#  ||||   TIMESTAMP  FUNCTION
#              | |           |       |   ||||      |         |
           <...>-13267   (-------) [034] d... 6368755.192329: check_cfs_rq_runtime <-put_prev_entity
           <...>-13267   (-------) [034] d... 6368755.192329: put_prev_entity <-put_prev_task_fair
           <...>-13267   (-------) [034] d... 6368755.192329: check_cfs_rq_runtime <-put_prev_entity
           <...>-13267   (-------) [034] d... 6368755.192329: put_prev_entity <-put_prev_task_fair
           <...>-13267   (-------) [034] d... 6368755.192329: check_cfs_rq_runtime <-put_prev_entity
           <...>-13267   (-------) [034] d... 6368755.192329: set_next_task_idle <-pick_next_task_idle
           <...>-13267   (-------) [034] d... 6368755.192330: enter_lazy_tlb <-__schedule
          <idle>-0       (-------) [034] d... 6368755.192330: __switch_to_xtra <-__switch_to
          <idle>-0       (-------) [034] d... 6368755.192330: finish_task_switch <-__schedule
```

<hr>

### # function_graph tracer usages
```text
$ echo function_graph > /sys/kernel/debug/tracing/current_tracer

$ cat /sys/kernel/debug/tracing/current_tracer
function_graph
```

show trace output for cpu 34, a code-like call graph is shown:
```text
$ cat trace | grep -e '34)' | head -n 100
cpu     duration              function calls
--------------------------------------------------------------------------
 34) + 97.344 us   |          } /* hrtimer_interrupt */
 34)               |          irq_exit() {
 34)   0.331 us    |            ksoftirqd_running();
 34)               |            __do_softirq() {
 34)               |              run_timer_softirq() {
 34)   0.293 us    |                _raw_spin_lock_irq();
 34)   0.301 us    |                _raw_spin_lock_irq();
 34)   1.589 us    |              }
 34)   2.182 us    |            }
 34)   0.280 us    |            idle_cpu();
 34)   0.286 us    |            rcu_irq_exit();
 34)   4.570 us    |          }
 34) ! 104.299 us  |        } /* smp_apic_timer_interrupt */
 34)   <========== |
 34) ! 105.641 us  |      } /* handle_external_interrupt_irqoff [kvm_intel] */
 34) ! 106.284 us  |    } /* vmx_handle_exit_irqoff [kvm_intel] */
 34)   0.544 us    |    __srcu_read_lock();
 34)               |    vmx_handle_exit [kvm_intel]() {
 34)   0.252 us    |      handle_external_interrupt [kvm_intel]();
 34)   0.944 us    |    }
 34) ! 317.132 us  |  } /* vcpu_enter_guest [kvm] */
```
if the duration more than 100us, then ! is added.

<hr>

### # stack tracing usages
```text
$ cat /proc/sys/kernel/stack_tracer_enabled        # check stack tracer status
0

$ echo 1 > /proc/sys/kernel/stack_tracer_enabled   # enable it

$ cat /sys/kernel/debug/tracing/stack_trace        # show stack depth, size
      Depth    Size   Location    (5 entries)
      -----    ----   --------
0)     3792      64   __accumulate_pelt_segments+0x5/0x90
1)     3728     160   __update_load_avg_se+0x23b/0x320
2)     3568      72   enqueue_entity+0x5e/0x6a0
3)     3496    2088   return_to_handler+0x0/0x36
4)     1408    1408   __writeback_single_inode+0x40/0x300

$ cat stack_max_size                               # check the max stack size used
3968

$ echo 0 > /sys/kernel/debug/tracing/stack_trace   # clear the tracer
```

if certain kernel code segment using its own stack, then the common stack tracer wont be
able to follow it. e.g., INTs has its own stack, which common stack tracer can not follow.
