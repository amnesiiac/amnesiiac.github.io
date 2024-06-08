---
layout: post
title: "tracepoint: static hooks in kernel (linux)"
author: "melon"
date: 2024-01-02 21:37
categories: "2024"
tags:
  - linux
  - kernel
---

trackpoint are general hooks inside kernel src code, they could be triggered when
kernel reached certain code point.

this feature could be utilized be many trace/debug tools:
e.g. perf collect tracepoint timestamp to produce report,
help us to do program performance analysis.

<hr>

### # why need tracepoint
we could use printk() inside kernel to get wanted info with self-compiled kernel,
but there existed some tricky point to mention:  
1 all printk statement are managed by kernel, printk from all module are printed as whole,
and hard for us to find interested part of info.  
2 if want to change the behavior of kernel printk(), all occurance of it should be involved,
which is a great deal of work.  
3 in embeded system, if console has large amount output, will make user hard to interact
with board system (inputs got flushed out).

<hr>

### # tracepoint: kernel replica for printk
kernel use a stub method for tracepoint log capture, the stubs in kernel are called
tracepoint also.  
each tracepoint has a name, a switch to enable/disable, function for regist stub,
function to uninstall stub.  
the stub functionality is nearly the same as printk(), but output msg to kernel's
ring buffer rather than print msg to console.  
the output msg finally present on debugfs, which could be enable & check by user.

<hr>

### # how to enable tracepoint for kernel
1 add config option for kernel:
```text
CONFIG_TRACEPOINTS=y
CONFIG_DEBUG_FS=y
```

2 after kernel bootup, mount debugfs:
```text
$ mount -t debugfs none /sys/kernel/debug/
```

<hr>

### # how to use debugfs interface for tracepoint
1 list all available events can be traced by debugfs:  
```text
$ ls /sys/kernel/debug/tracing/events
alarmtimer        header_page  neigh           skb
block             huge_memory  net             smbus
bpf_test_run      hwmon        nfsd            sock
bridge            hyperv       nmi             spi
cgroup            i2c          oom             sunrpc
clk               initcall     page_isolation  swiotlb
compaction        intel_iommu  pagemap         syscalls
context_tracking  iommu        page_pool       task
cpuhp             irq          percpu          tcp
devfreq           irq_matrix   power           thermal
devlink           irq_vectors  printk          thermal_power_allocator
dma_fence         jbd2         qdisc           timer
drm               kmem         ras             tlb
enable            kvm          raw_syscalls    udp
exceptions        kvmmmu       rcu             vmscan
ext4              kyber        regmap          vsyscall
fib               libata       regulator       wbt
fib6              mce          rpcgss          workqueue
filelock          mdio         rpm             writeback
filemap           migrate      rseq            x86_fpu
fs_dax            module       rtc             xdp
ftrace            msr          sched           xen
gpio              napi         scsi            xhci-hcd
header_event      nbd          signal
```

2 find out which events are enable/disable:
```text
$ find . -name 'enable' | xargs grep '0'    # find all tracepoint event disabled
./nbd/nbd_header_sent/enable:0
./nbd/nbd_payload_sent/enable:0
...
./kvmmmu/kvm_mmu_pagetable_walk/enable:0
./kvmmmu/kvm_mmu_paging_element/enable:0
./kvmmmu/kvm_mmu_set_accessed_bit/enable:0
./kvmmmu/kvm_mmu_set_dirty_bit/enable:0
...
./kvm/kvm_entry/enable:0
./kvm/kvm_hypercall/enable:0
./kvm/kvm_hv_hypercall/enable:0
./kvm/kvm_pio/enable:0
...

$ find . -name 'enable' | xargs grep '1'    # find all tracepoint event enabled
```

3 enable kernel proc schedual related output:
```text
echo 1 > /sys/kernel/debug/tracing/events/sched/enable
```

4 the output show default tracer is set as function_graph set by previous test,
but undesired for now:
```text
$ cat /sys/kernel/debug/tracing/trace  | head -n 10
# tracer: function_graph
#
# CPU  DURATION         FUNCTION CALLS
#
  0)   0.163 us           } /* __accumulate_pelt_segments */
  0)   0.552 us         } /* __update_load_avg_se */
  0)   0.183 us         __update_load_avg_cfs_rq();
  0)   0.167 us         update_cfs_group();
  0)   0.182 us         account_entity_enqueue();
  0)   0.163 us         place_entity();
  0)   0.180 us         __enqueue_entity();
```

confirm current tracer:
```text
$ cat /sys/kernel/debug/tracing/current_tracer
function_graph
```

set current_tracer as nop, test again:
```text
$ echo nop > ./current_tracer

$ cat trace | head -n 20
# tracer: nop
#
# entries-in-buffer/entries-written: 945438/3165598   #P:36
#
#                                          _-----=> irqs-off
#                                         / _----=> need-resched
#                                        | / _---=> hardirq/softirq
#                                        || / _--=> preempt-depth
#                                        ||| /     delay
#           TASK-PID       TGID    CPU#  ||||   TIMESTAMP  FUNCTION
#                                        ||||
    kworker/25:2-22093   (  22093) [025] d... 6729576.500141: sched_switch: prev_comm=kworker/25:2 prev_pid=22093 ...
          <idle>-0       (-------) [025] d.s. 6729576.507139: sched_waking: comm=kworker/25:2 pid=22093 prio=120 ...
          <idle>-0       (-------) [025] dNs. 6729576.507141: sched_wakeup: comm=kworker/25:2 pid=22093 prio=120 ...
          <idle>-0       (-------) [025] d... 6729576.507146: sched_switch: prev_comm=swapper/25 prev_pid=0 ...
    kworker/25:2-22093   (  22093) [025] d... 6729576.507149: sched_stat_runtime: comm=kworker/25:2 pid=22093 ...
    kworker/25:2-22093   (  22093) [025] d... 6729576.507342: sched_switch: prev_comm=kworker/25:2 prev_pid=22093 ...
          <idle>-0       (-------) [025] d.s. 6729576.511149: sched_waking: comm=kworker/25:2 pid=22093 prio=120 ...
          <idle>-0       (-------) [025] dNs. 6729576.511152: sched_wakeup: comm=kworker/25:2 pid=22093 prio=120 ...
          <idle>-0       (-------) [025] d... 6729576.511162: sched_switch: prev_comm=swapper/25 prev_pid=0 ...
```
hence, we could check tracepoint captures by trace buffer output, but TLDR.  

get tracepiont info for certain proc by:
```text
$ ps -p 11385
  PID TTY          TIME CMD
11385 ?        1-13:07:24 qemu-system-x86

$ echo 11385 > set_event_pid
```

clear the trace buffer, show newly captured trace logs:
```text
$ echo 0 > trace
$ cat trace | head -n 20
# tracer: nop
#
# entries-in-buffer/entries-written: 5203/5203   #P:36
#
#                                          _-----=> irqs-off
#                                         / _----=> need-resched
#                                        | / _---=> hardirq/softirq
#                                        || / _--=> preempt-depth
#                                        ||| /     delay
#           TASK-PID       TGID    CPU#  ||||   TIMESTAMP  FUNCTION
#              | |           |       |   ||||      |         |
       _net2_qp0-26374   (  36411) [003] d.s1 6736852.843825: sched_waking: comm=qemu-system-x86 pid=11385 ...
       _net2_qp0-26374   (  36411) [003] d.s1 6736852.843842: sched_wakeup: comm=qemu-system-x86 pid=11385 ...
          <idle>-0       (-------) [009] d... 6736852.843919: sched_switch: prev_comm=swapper/9 prev_pid=0 ...
 qemu-system-x86-11385   (  11385) [009] d... 6736852.844030: sched_waking: comm=qemu-system-x86 pid=12325 ...
 qemu-system-x86-11385   (  11385) [009] d... 6736852.844103: sched_stat_runtime: comm=qemu-system-x86 ...
 qemu-system-x86-11385   (  11385) [009] d... 6736852.844110: sched_switch: prev_comm=qemu-system-x86 ...
       _net1_qp0-27919   (  35909) [007] d.s1 6736852.861180: sched_waking: comm=qemu-system-x86 pid=11385 ...
       _net1_qp0-27919   (  35909) [007] d.s1 6736852.861202: sched_wakeup: comm=qemu-system-x86 pid=11385 ...
          <idle>-0       (-------) [009] d... 6736852.861257: sched_switch: prev_comm=swapper/9 prev_pid=0 ...
```
explanation of tracepoint in above log:  
sched_switch: when schedulor decide to switch to another task, the cur task is flush out.  
sched_wakeup/sched_waking: when kernel try to wakeup cur task by try_to_wake_up.  

sched_waking triggered at the beginning of wake up, sched_wakeup is triggered at the end
of wake up. usually, there's a little gap between the trigger of the 2 tracepoint,
but has exceptions.

the timestamp shows the absolute time since cpu bootup, the date time in the log can be
deduced as:
```text
$ date
Tue Jan  2 22:47:33 CST 2024

$ uptime
22:47:36 up 77 days, 23:33, 19 users,  load average: 12.22, 8.99, 8.54
```
Tue Jan 22:47:33 - 77 days = cpu absolute bootup time
log first line absolute time = cpu absolute bootup time + TIMESTAMP(6736852.843825)

<hr>

### # steps to setup & customize the tracepoint method inside debugfs
1 check available net related tracepoint event:
```text
$ pwd
/sys/kernel/debug/tracing/events/net

$ ls
enable                napi_gro_receive_entry  net_dev_xmit             netif_receive_skb_exit        netif_rx_entry
filter                napi_gro_receive_exit   net_dev_xmit_timeout     netif_receive_skb_list_entry  netif_rx_exit
napi_gro_frags_entry  net_dev_queue           netif_receive_skb        netif_receive_skb_list_exit   netif_rx_ni_entry
napi_gro_frags_exit   net_dev_start_xmit      netif_receive_skb_entry  netif_rx                      netif_rx_ni_exit
```

2 examine one of the supported static anchor:
```text
$ cd netif_receive_skb
$ ls
enable  filter  format  id  trigger
```

3 the trace log output format is determined by filter file:
```text
$ cat format
name: netif_receive_skb
ID: 1256
format:
    field:  unsigned short common_type;          offset:0;  size:2;  signed:0;
    field:  unsigned char common_flags;          offset:2;  size:1;  signed:0;
    field:  unsigned char common_preempt_count;  offset:3;  size:1;  signed:0;
    field:  int common_pid;                      offset:4;  size:4;  signed:1;

    field:  void * skbaddr;                      offset:8;  size:8;  signed:0;
    field:  unsigned int len;                    offset:16; size:4;  signed:0;
    field:  __data_loc char[] name;              offset:20; size:4;  signed:1;

print fmt: "dev=%s skbaddr=%p len=%u", __get_str(name), REC->skbaddr, REC->len
```
from above, we could get the filter field keyeord for filter input.

4 check the log collected by tracepoint
```text
$ cat /sys/kernel/debug/tracing/trace | grep 'xxx'
...
```

<hr>

### # usecases: using tracepoint for kernel pkts rx/tx analysis
case 1: set tracepoint filter for eqpt_xxx process, and enable traceing.
```text
$ ps | grep eqpt_xxx
0    22289      214  -31 R    0     S eqpt_xxx_app

$ echo "common_pid == 22289" > filter
$ cd /sys/kernel/debug/tracing/events/net/netif_receive_skb && cat enable
0
$ echo 1 > enable
```

the tracepoint netif_receive_skb for eqpt_xxx_app proc is shown as:
```text
$ cat /sys/kernel/debug/tracing/trace
# tracer: nop
#
# entries-in-buffer/entries-written: 6/6   #P:4
#
#                              _-----=> irqs-off
#                             / _----=> need-resched
#                            | / _---=> hardirq/softirq
#                            || / _--=> preempt-depth
#                            ||| /     delay
#           TASK-PID   CPU#  ||||    TIMESTAMP  FUNCTION
#              | |       |   ||||       |         |
    eqpt_xxx_app-22289 [000] ..s1  2912.455748: netif_receive_skb: dev=lo skbaddr=ffffffc0bba7b6e0 len=124
    eqpt_xxx_app-22289 [000] ..s1  2912.455777: netif_receive_skb: dev=lo skbaddr=ffffffc0bc1f4300 len=52
    eqpt_xxx_app-22289 [000] ..s1  2917.455532: netif_receive_skb: dev=lo skbaddr=ffffffc0f4c512e0 len=124
    eqpt_xxx_app-22289 [000] ..s1  2917.455566: netif_receive_skb: dev=lo skbaddr=ffffffc0d95ec000 len=52
    eqpt_xxx_app-22289 [000] ..s1  2922.456378: netif_receive_skb: dev=lo skbaddr=ffffffc0bc347ce0 len=124
    eqpt_xxx_app-22289 [000] ..s1  2922.456401: netif_receive_skb: dev=lo skbaddr=ffffffc0d9575500 len=52

# echo "(common_pid == 1 || common_flags == 1)" > filter
```

all tracepoint (static anchor) supported for kernel networking:
```text
$ cd /sys/kernel/debug/tracing/events/net && ls
enable                   napi_gro_frags_entry     net_dev_queue            net_dev_xmit
netif_receive_skb_entry  netif_rx_entry           filter                   napi_gro_receive_entry
net_dev_start_xmit       netif_receive_skb        netif_rx                 netif_rx_ni_entry
```

description for several tracepoint related to pkts acception:  
1) netif_rx_ni_entry: accepting pkts at the entry of non-interrupt context by nic.  
2) netif_rx_entry: accepting pkts at the entry of interrupt context by nic.  
3) netif_rx: pkts entering function netif_rx_internal.  
4) netif_receive_skb_entry: pkts entering function netif_receive_skb.  
5) netif_receive_skb: pkts entering function __netif_receive_skb_core, pkts into protocol layer.

<p style="margin-bottom: 20px;"></p>

case 2: set tracepoint filter for eqpt_logic process, and enable tracing.
```text
$ ps | grep eqpt_logic                                          # check the pid of certain app
1    22283      213  -31 R    0     S eqpt_logic_olt

$ cd /sys/kernel/debug/tracing/events/net/net_dev_xmit && ls
enable   filter   format   id       trigger

$ cd net_dev_xmit && echo "common_pid == 22283" > filter        # set filter for certain process
$ echo 1 > enable                                               # enable
$ echo 0 > /sys/kernel/debug/tracing/trace                      # clear trace buffer
```

the captured trace log for net_dev_xmit is as:
```text
$ cat /sys/kernel/debug/tracing/trace | head -n 15
# tracer: nop
#
# entries-in-buffer/entries-written: 4/4   #P:4
#
#                              _-----=> irqs-off
#                             / _----=> need-resched
#                            | / _---=> hardirq/softirq
#                            || / _--=> preempt-depth
#                            ||| /     delay
#           TASK-PID   CPU#  ||||    TIMESTAMP  FUNCTION
#              | |       |   ||||       |         |
  eqpt_logic_xxx-22283 [000] ....  6018.619851: net_dev_xmit: dev=lo skbaddr=ffffffc0d8c0f2e0 len=138 rc=0
  eqpt_logic_xxx-22283 [000] ..s2  6018.619888: net_dev_xmit: dev=lo skbaddr=ffffffc0ba8e4400 len=66 rc=0
  eqpt_logic_xxx-22283 [001] ....  6023.620002: net_dev_xmit: dev=lo skbaddr=ffffffc0d8d58ae0 len=138 rc=0
  eqpt_logic_xxx-22283 [001] ..s2  6023.620022: net_dev_xmit: dev=lo skbaddr=ffffffc0b96c3300 len=66 rc=0
```
