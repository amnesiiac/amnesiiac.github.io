---
layout: post
title: "rpmsg kdrv issue: rcu stall, scheduling while atomic (kdrv, rpmsg, sync)"
author: "melon"
date: 1111-03-04 20:24
categories: "2025"
tags:
  - driver
  - icc
  - kernel
---

this article mainly describe an kernel hang issue: report scheduling while atomic, caused by using mutex as sync
promitive in __dev_queue_xmit context when implement the rpmsg platform kdrv.

<hr>

### # issue description
lsfx series board fant-h loaded with 2203.052 build is stuck in cold-standby state. code-standby means hw in lower
power state, while sw not started compared to hot-standby.
when checked the nt-b core-0 (the standby compared to nt-a), see the below trace being printed repeatedly.
the log snippet are as:

$ 1 nt-a core-0: report standby board state as none.

```blurtext
isam-reborn# show hardware-protection-element-state
hardware-protection-element-state component
STANDBY
PROTECTION PROTECTION STATUS
HARDWARE GROUP ELEMENT STANDBY CHANGE STANDBY STATE CHANGE STANDBY STATE CHANGE TROUBLE SHOOTING
REFERENCE REFERENCE STATUS REASON TIME INFORMATION
-----------------------------------------------------------------------------------------------------------------
Board-Nta NT-Protection PROVIDING-SERVICE NONE 2022-01-27T16:21:07+00:00 Active nt
Board-Ntb NT-Protection COLD-STANDBY NONE 2022-01-27T16:21:07+00:00 IHUB/Database/File/Software are not available
```

$ 2 nt-b core-0: report rcu stall problem, leading to system hang.

```text
[15441.599359] rcu: INFO: rcu_preempt detected stalls on CPUs/tasks:
[15441.605455] rcu: 3-...!: (1 GPs behind) idle=d7e/1/0x4000000000000000 softirq=52215/52216 fqs=33
[15441.614418] rcu: 5-...!: (13 GPs behind) idle=dce/0/0x1 softirq=42241/42241 fqs=33
[15441.622163] (detected by 2, t=15268234 jiffies, g=60521, q=99)
[15441.628080] Task dump for CPU 3:
[15441.631303] clock_hwa_app R running task 0 5271 336 0x00100004
[15441.638353] Call Trace:
[15441.640795] [c00000007c04f620] [c00000007c04f6d0] 0xc00000007c04f6d0 (unreliable)
[15441.648280] Task dump for CPU 5:
[15441.651503] swapper/5 R running task 0 0 1 0x0000080c
[15441.658553] Call Trace:
[15441.660997] [c0000000b022f9f0] [c0000000000d18c4] .clockevents_program_event+0x138/0x150 (unreliable)
[15441.670220] [c0000000b022fa80] [0000000000000001] 0x1
[15441.675272] rcu: rcu_preempt kthread starved for 15268154 jiffies! g60521 f0x2 RCU_GP_WAIT_FQS(5) ->state=0x0 ->cpu=0
[15441.685884] rcu: RCU grace-period kthread stack dump:
[15441.690932] rcu_preempt R running task 0 10 2 0x00000808
[15441.697982] Call Trace:
[15441.700423] [c0000000b01ff7b0] [c0000000b7a00688] 0xc0000000b7a00688 (unreliable)
[15441.707910] [c0000000b01ff990] [c000000000008878] .__switch_to+0xbc/0xc8
[15441.714614] [c0000000b01ffa30] [c00000000073d6f4] .__schedule+0x494/0x56c
[15441.721404] [c0000000b01ffb00] [c00000000073d86c] .schedule+0xa0/0xd8
[15441.727845] [c0000000b01ffb80] [c000000000740610] .schedule_timeout+0xec/0x10c
[15441.735069] [c0000000b01ffc40] [c0000000000b701c] .rcu_gp_kthread+0x5d0/0x834
[15441.742207] [c0000000b01ffd70] [c000000000075fbc] .kthread+0x15c/0x164
[15441.748736] [c0000000b01ffe20] [c0000000000009ec] .ret_from_kernel_thread+0x58/0x6c
```

<hr>

### # analysis
reproduce the above issue by enable rcu torture module running in duplex fant-h, the log is as:
rpmsg platform driver should not use mutex in net tx context, which may bring cpu in rcu stall state:

```text
$ cat /sys/class/reborn/nt_status/status
active

$ check kernel logs via dmesg or /var/log/kern.log
[50314.952711] rcu_torture_fwd_prog_nr: Duration 14640 cver 879 gps 1460
[50315.161884] rcu_torture_fwd_prog_cr Duration 39 barrier: 19 pending 15628 n_launders: 31487 n_launders_sa: 8223...
[50315.175993] rcu_torture_fwd_cb_hist: Callback-invocation histogram (duration 72 jiffies): 1s/10: 57683:11

[50365.835721] rcu-torture: rtc: 00000000e312d68d ver: 1952831 tfle: 0 rta: 1952832 rtaf: 0 rtf: 1952822 rtmbe: 0 rtbe: 0...
[50365.851868] rcu-torture: Reader Pipe: 37733716978 5723571 0 0 0 0 0 0 0 0 0
[50365.859125] rcu-torture: Reader Batch: 37728415901 11024629 0 0 0 0 0 0 0 0 0
[50365.866389] rcu-torture: Free-Block Circulation: 1952832 1952831 1952830 1952829 1952828 1952827 1952826 1952823 0
[50391.842708] rcu_torture_fwd_prog_nr: Duration 13712 cver 873 gps 1409
[50392.042817] rcu_torture_fwd_prog_cr Duration 42 barrier: 21 pending 20056 n_launders: 34776 n_launders_sa: 102...
[50392.056789] rcu_torture_fwd_cb_hist: Callback-invocation histogram (duration 77 jiffies): 1s/10: 61445:10
[50427.275731] rcu-torture: rtc: 00000000cef4102f ver: 1956629 tfle: 0 rta: 1956629 rtaf: 0 rtf: 1956620 rtmbe: 0 rtbe: 0...
[50427.291783] rcu-torture: Reader Pipe: 37805260077 5734583 0 0 0 0 0 0 0 0 0
[50427.298864] rcu-torture: Reader Batch: 37799948846 11045794 0 0 0 0 0 0 0 0 0
[50427.306114] rcu-torture: Free-Block Circulation: 1956628 1956628 1956627 1956626 1956625 1956624 1956623 1956620 0
[50467.650712] rcu_torture_fwd_prog_nr: Duration 11693 cver 718 gps 1199
[50467.878163] rcu_torture_fwd_prog_cr Duration 53 barrier: 20 pending 8851 n_launders: 41193 n_launders_sa: 8851...
[50467.892445] rcu_torture_fwd_cb_hist: Callback-invocation histogram (duration 87 jiffies): 1s/10: 70173:13
[50488.715745] rcu-torture: rtc: 00000000ccf990cc ver: 1960304 tfle: 0 rta: 1960305 rtaf: 0 rtf: 1960290 rtmbe: 0 rtbe: 0...
[50488.732182] rcu-torture: Reader Pipe: 37872815388 5745250 0 0 0 0 0 0 0 0 0
[50488.739284] rcu-torture: Reader Batch: 37867494157 11066462 0 0 0 0 0 0 0 0 0
[50488.746584] rcu-torture: Free-Block Circulation: 1960305 1960304 1960301 1960300 1960298 1960297 1960296 1960295 0
[50494.227093] BUG: scheduling while atomic: clock_hwa_app/5079/0x00000203
[50494.233733] Modules linked in: rcutorture torture fnio_ab_cpld_spi(O) fnio_ab_cpld(O) i2c_mux_reg(O) remoteproc_ppc(O)...
[50494.308422] Preemption disabled at:                         // prempt is not allowed in the rcu ctx (__dev_queue_xmit)
[50494.308438] [<c0000000005ab5dc>] .__dev_queue_xmit+0x80/0x5f0
[50494.317717] CPU: 1 PID: 5079 Comm: clock_hwa_app Tainted: G O 5.4.72 #2  // rcu ctx of clock-hwa-app is polluted
[50494.325637] Call Trace:                                                  // rcu ctx callstack
[50494.328085] [...] .dump_stack+0xa0/0xd4 (unreliable)                     // start callstack dump
[50494.335839] [...] .__schedule_bug+0xcc/0xe8                              // schedule ctx: bug report
[50494.342806] [...] .__schedule+0x80/0x56c
[50494.349510] [...] .schedule+0xa0/0xd8
[50494.355955] [...] .schedule_preempt_disabled+0x20/0x38                   // schedule ctx when preemption is disabled
[50494.363879] [...] .__mutex_lock.isra.6+0x114/0x328                       // mutex lock ctx
[50494.371453] [...] .mutex_lock+0x4c/0x70
[50494.378082] [...] .nokia_rpmsg_notify+0x28/0xb8 [rpmsg_platform_driver]  // plt drv: rpmsg notify ctx
[50494.387485] [...] .virtqueue_notify+0x44/0x68                            // virtque data plane changed notify ctx
[50494.394629] [...] .rpmsg_send_offchannel_raw+0x2a4/0x31c                 // rpmsg kernel ctx
[50494.402726] [...] .rpmsg_trysendto+0x70/0x94
[50494.409788] [...] .rpmsg_ether_xmit+0xf0/0x188 [rpmsg_ifc]               // rpmsg drv kernel ctx
[50494.418058] [...] .netdev_start_xmit+0x60/0x90                           // netdev xmit ctx
[50494.425284] [...] .dev_hard_start_xmit+0x134/0x22c
[50494.432860] [...] .sch_direct_xmit+0xd4/0x288                            // netdev xmit ctx
[50494.440000] [...] .__qdisc_run+0x48c/0x564                               // net dev queing ctx
[50494.446880] [...] .qdisc_run.part.99+0x34/0x50
[50494.454106] [...] .__dev_queue_xmit+0x2a4/0x5f0                          // net dev queing ctx
[50494.461423] [...] .packet_sendmsg+0xd84/0xee0                            // high-level pkt send
[50494.468565] [...] .sock_sendmsg_nosec+0x2c/0x50
[50494.475880] [...] .sock_write_iter+0xc4/0x110
[50494.483026] [...] .do_iter_readv_writev+0x10c/0x160
[50494.490697] [...] .do_iter_write+0x90/0xb8
[50494.497583] [...] .compat_writev+0x80/0xc0                  // compatible layer for different arch or upper invokers
[50494.504463] [...] .do_compat_writev+0x70/0xb4               // compatible layer
[50494.511603] [...] system_call+0x60/0x6c                     // app syscall for ptrace/setre
[50494.518454] -----------[ cut here ]-----------
[50494.523204] DEBUG_LOCKS_WARN_ON(val > preempt_count())
[50494.523241] WARNING: CPU: 5 PID: 5079 at kernel/sched/core.c:3899 .preempt_count_sub+0x7c/0xfc
[50494.536991] Modules linked in: rcutorture torture fnio_ab_cpld_spi(O) fnio_ab_cpld(O) i2c_mux_reg(O) remoteproc_ppc(O)...
[50494.611654] CPU: 5 PID: 5079 Comm: clock_hwa_app Tainted: G W O 5.4.72 #2
[50494.619575] NIP: c00000000007de4c LR: c00000000007de48 CTR: 0000000000000000
[50494.626712] REGS: c000000099f33130 TRAP: 0700 Tainted: G W O (5.4.72)
[50494.634544] MSR: 000000008202b002 <VEC,CE,EE,FP,ME> CR: 24422442 XER: 20000000
[50494.642034] IRQMASK: 0
...
```

<hr>

### # solution
replace mutex with spinlock in rpmsg platform driver context:

```blurtext
$ hg diff -c 2156
diff --git a/misc/rpmsg_platform_driver.c b/misc/rpmsg_platform_driver.c
--- a/misc/rpmsg_platform_driver.c
+++ b/misc/rpmsg_platform_driver.c
@@ -17,10 +17,16 @@

-#define DRIVER_VERSION      "0.0.2"
+#define DRIVER_VERSION      "0.0.3"

 struct nokia_rpmsg_vproc {
-    struct virtio_device vdev;
-    u64 vring[VRING_NUM_PER_VIRTDEV];
-    u64 data_buffer_pool;
-    struct mutex lock;
-    struct virtqueue *vq[VQUEUE_NUM_PER_VIRTDEV];
-    u32 rpmsg_role;
-    s32 input_dst_cpuid;
-    s32 output_dst_cpuid;
-    struct rpmsg_irq_data irq_data;
-    int base_vq_id;
-    int num_of_vqs;
-    struct kobject kobj;
-    __u32 tx_count;
-    __u32 rx_count;
-    int log_level;
-    struct rpmsg_channel_info chinfo;
-    struct delayed_work delay_work_info;
-    struct list_head extra_endpoint_list;
-    u64 features;
-    unsigned int rpmsg_num_bufs;
-    unsigned int rpmsg_buf_size;
+	struct virtio_device vdev;
+	u64 vring[VRING_NUM_PER_VIRTDEV];
+	u64 data_buffer_pool;
+	/* used in virtqueue_notify, it's in network tx context, preemption maybe disable, so should only spinlock here */
+	spinlock_t spin_lock;
+	struct virtqueue *vq[VQUEUE_NUM_PER_VIRTDEV];
+	u32 rpmsg_role;
+	s32 input_dst_cpuid;
+	s32 output_dst_cpuid;
+	struct rpmsg_irq_data irq_data;
+	int base_vq_id;
+	int num_of_vqs;
+	struct kobject kobj;
+	__u32 tx_count;
+	__u32 rx_count;
+	int log_level;
+	struct rpmsg_channel_info chinfo;
+	struct delayed_work delay_work_info;
+	struct list_head extra_endpoint_list;
+	u64 features;
+	unsigned int rpmsg_num_bufs;
+	unsigned int rpmsg_buf_size;
 };

 struct nokia_rpmsg_vq_info {
@@ -197,13 +204,18 @@ static bool nokia_rpmsg_notify(struct vi
    struct virtio_device *vdev = &rpvq->rpdev->vdev;
    int ret;

-	mutex_lock(&rpvq->rpdev->lock);
+	/*
+	 * In network tx context(__dev_queue_xmit), spin lock is used to disable preempt.
+	 * So mutex should not be used here, may cause CPU schedule issue
+	*/
+	spin_lock(&rpvq->rpdev->spin_lock);
    /* message payload could be virtqueue id, now no use as there is only one queue for tx or rx */

    ret = rpmsg_irq_trigger(rpvq->rpdev->irq_data.output_irq_num, rpvq->rpdev->irq_data.channel_num);
    rpvq->rpdev->tx_count++;
    if(rpvq->rpdev->log_level > LOG_NONE) dev_info(&vdev->dev, "notify remote, txcount = %u !\n", rpvq->rpdev->tx_count);
-	mutex_unlock(&rpvq->rpdev->lock);
+	spin_unlock(&rpvq->rpdev->spin_lock);
    if (ret) {
        pr_err("nokia_rpmsg_notify() failed: %d\n", ret);
    }
@@ -268,7 +280,7 @@ static struct virtqueue *rp_find_vq(stru
    /* system-wide unique id for this virtqueue */
    rpvq->vq_id = rpdev->base_vq_id + index;
    rpvq->rpdev = rpdev;
-	mutex_init(&rpdev->lock);
+	spin_lock_init(&rpdev->spin_lock);

    return vq;
```

<hr>

### # rcu torture how-to
$ 1 how to config kernel to enable rcu-torture ko?  
hierarchical rcu: https://lwn.net/Articles/305782/#Testing  
kernel config param for rcu: https://lwn.net/Articles/777214/  
rcu-torture doc: https://github.com/torvalds/linux/tree/master/tools/testing/selftests/rcutorture/doc

$ 2 how to operate the torture?  
ref: kernel Documentation/RCU/torture.txt

$ 3 rcu torture testcases:  
https://github.com/torvalds/linux/tree/master/tools/testing/selftests/rcutorture/configs/rcu
