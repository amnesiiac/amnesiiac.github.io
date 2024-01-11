---
layout: post
title: "cpu affinity (linux)"
author: "melon"
date: 2023-12-25 20:41
categories: "2023"
tags:
  - linux
---

### # taskset & numactl
taskset & numactl are about cpu affinity settings & kernel schedual part, which both rely on syscall cpu_setaffinity.

ref: https://docs.kernel.org/admin-guide/cgroup-v1/cpusets.html

<hr>

### # cpuset/cset vs taskset
ref: https://serverfault.com/questions/625146/difference-between-taskset-and-cpuset

<hr>

### # code in action
a toy code for set/get cpu range of proc:
```text
#!/bin/bash

set_all_proc_cpurange() {
    for pid in /proc/[0-9]*; do
        taskset -apc 16-31 ${pid##*/};   # get all proc from filesys, and set affinity
    done
}

run_proc_with_given_cpurange() {
    procname=top
    taskset -c 0-15 "${procname}"
}

set_proc_cpurange() {
    pid=17219
    taskset -pc 0,3,7-11 "${pid}"
}

get_proc_cpurange() {
    pid=17219
    taskset -p "${pid}"
}
```

set_proc_cpurange:
```text
pid 17219's current affinity list: 0-35
pid 17219's new affinity list: 0,3,7-11
```

get_proc_cpurange:
```text
pid 17219's current affinity mask: f89
```

run_proc_with_given_cpurange:
```text
top - 17:34:58 up 64 days, 18:20, 25 users,  load average: 9.72, 7.75, 7.13
Tasks: 4128 total,   8 running, 3918 sleeping,   0 stopped,   1 zombie
%Cpu(s): 12.0 us,  5.4 sy,  0.0 ni, 82.3 id,  0.1 wa,  0.0 hi,  0.1 si,  0.0 st
KiB Mem : 10715187+total,  7057012 free, 40233980 used, 59860876 buff/cache
KiB Swap:        0 total,        0 free,        0 used. 38942056 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
32053 root      20   0 4244692   2.1g   2.1g S  33.7  2.0   1152:20 cloud-hyperviso
32073 root      20   0 4244676   2.1g   2.1g S  31.1  2.0   1150:40 cloud-hyperviso
 8930 root      20   0 4248920   2.1g   2.1g S  30.8  2.0 389:23.98 cloud-hyperviso
32422 root      20   0 4248844   2.0g   2.0g S  30.8  2.0 349:41.68 cloud-hyperviso
32309 root      20   0 4248844   2.1g   2.1g S  27.6  2.0 340:57.72 cloud-hyperviso
 8912 root      20   0 4248848   2.0g   2.0g S  27.3  2.0 389:37.90 cloud-hyperviso
31692 root      20   0 5598808   1.6g  13208 S  24.8  1.6 324:58.63 qemu-system-x86
25746 root      20   0 4244696   2.0g   2.0g S  20.3  1.9   2208:20 cloud-hyperviso
31564 root      20   0 5598792   1.6g  13004 S  16.2  1.6 334:47.54 qemu-system-x86
 3679 root      20   0   33144   3884   2176 R  15.2  0.0   0:00.48 system_manageme
32267 root      20   0 4244740   1.6g   1.6g S  13.0  1.6   1652:50 cloud-hyperviso
 8957 root      20   0 4240608   1.0g   1.0g S   9.8  1.0 159:21.56 cloud-hyperviso
34065 root      20   0 5607004   1.6g  12272 S   9.5  1.5 381:39.52 qemu-system-x86
20281 root      20   0 5598808   1.6g  13428 S   9.2  1.5 394:32.99 qemu-system-x86
34016 root      20   0 5601480   1.6g  12212 S   9.2  1.5 383:07.24 qemu-system-x86
32386 root      20   0 4240776   1.0g   1.0g S   8.9  1.0 456:58.44 cloud-hyperviso
 3690 root      20   0   33144   3944   2232 R   8.6  0.0   0:00.27 system_manageme
20280 root      20   0 5596368   1.6g  13332 S   8.6  1.5 395:32.28 qemu-system-x86
21275 root      20   0 5595340   1.7g   1904 S   8.6  1.7  11465:31 qemu-system-x86
32150 root      20   0 4240572 894628 893972 S   7.6  0.8 123:19.23 cloud-hyperviso
32384 root      20   0 4240512   1.0g   1.0g S   7.3  1.0 137:53.56 cloud-hyperviso
32287 root      20   0 4241016 943844 943032 S   7.0  0.9 403:32.10 cloud-hyperviso
 8980 root      20   0 4240512 892040 891404 S   6.3  0.8 146:50.04 cloud-hyperviso
 4223 root      20   0   30852   2232      0 R   5.7  0.0   0:00.18 system_manageme
17219 metung    20   0  168540   8612   3724 R   4.4  0.0   0:18.73 top
 4279 root      20   0    4276    592    504 R   4.1  0.0   0:00.13 cat
 1789 root      20   0  659352 320172   6604 S   1.6  0.3 537:15.18 switch_hwa_app
```
