---
layout: post
title: "vss & rss & pss & uss (linux memory)"
author: "melon"
date: 2024-01-15 08:15
categories: "2024"
tags:
  - linux
  - ongoing
---

### # intro
generally, the relationship between 4 memory statistics are as: vss >= rss >= pss >= uss.

<hr>

### # vss (virtual set size)
definition: virtual statistics memory occupytion.

<hr>

### # rss (resident set size)
definition: rough physical memory occupytion.  
for shared lib loaded into the mem, rss will just take all of them into account for each proc invoked the lib.
will add duplicate part mem when computing all procs mem usage.

<hr>

### # pss (proportional set size)
definition: approximately the real physical memory occupytion.  
for shared lib loaded into the mem, pss will split them into different proc that invoked the lib.
which will be realy useful for computing total mem for all procs (no dup part considered).

<hr>

### # uss (unique set size)
definition: physical private memory taken up by proc sololy.

<hr>

### # memory model graph cross running proc with shared memory
given a live 5 processes memory model as follows:
```txt
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         memory occupied by shared libs                          │
└─────────────────────────────────────────────────────────────────────────────────┘
┌──────────────────────────────────────┐   ┌──────────────────────────────────────┐
│      process #1 allocated memory     │   │      process #2 allocated memory     │
│ ┌─────────────┐ ┌──────────────────┐ │   │ ┌─────────────┐ ┌──────────────────┐ │
│ │ free memory │ │ memory under use │ │   │ │ free memory │ │ memory under use │ │
│ └─────────────┘ └──────────────────┘ │   │ └─────────────┘ └──────────────────┘ │
└──────────────────────────────────────┘   └──────────────────────────────────────┘
┌──────────────────────────────────────┐   ┌──────────────────────────────────────┐
│      process #3 allocated memory     │   │      process #4 allocated memory     │
│ ┌─────────────┐ ┌──────────────────┐ │   │ ┌─────────────┐ ┌──────────────────┐ │
│ │ free memory │ │ memory under use │ │   │ │ free memory │ │ memory under use │ │
│ └─────────────┘ └──────────────────┘ │   │ └─────────────┘ └──────────────────┘ │
└──────────────────────────────────────┘   └──────────────────────────────────────┘

     (no shared lib memory utilized)
┌──────────────────────────────────────┐
│      process #5 allocated memory     │
│ ┌─────────────┐ ┌──────────────────┐ │
│ │ free memory │ │ memory under use │ │
│ └─────────────┘ └──────────────────┘ │
└──────────────────────────────────────┘
```
process 1-4 using common shared lib, while process 5 dont.  
let x denote proc 1-4, hence, the four memory can be computed as:  
vss_x = memory_allocated_x + common_shared_lib_memory
```txt
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         memory occupied by shared libs                          │
└─────────────────────────────────────────────────────────────────────────────────┘
┌──────────────────────────────────────┐
│      process #1 allocated memory     │
│ ┌─────────────┐ ┌──────────────────┐ │
│ │ free memory │ │ memory under use │ │
│ └─────────────┘ └──────────────────┘ │
└──────────────────────────────────────┘
```
rss_x = memory_under_use_x + common_shared_lib_memory
```txt
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         memory occupied by shared libs                          │
└─────────────────────────────────────────────────────────────────────────────────┘
┌--------------------------------------┐
|      process #1 allocated memory     |
| ┌-------------┐ ┌──────────────────┐ |
| | free memory | │ memory under use │ |
| └-------------┘ └──────────────────┘ |
└--------------------------------------┘
```
pss_x = memory_under_use_x + common_shared_lib_memory / num_proc_using_shared_mem
```txt
┌─────────────────────------------------------------------------------------------┐
│ portion used by #1 |    memory occupied by shared libs                          |
└─────────────────────------------------------------------------------------------┘
┌--------------------------------------┐
|      process #1 allocated memory     |
| ┌-------------┐ ┌──────────────────┐ |
| | free memory | │ memory under use │ |
| └-------------┘ └──────────────────┘ |
└--------------------------------------┘
```
uss_x = memory_under_use_x
```txt
┌---------------------------------------------------------------------------------┐
|                         memory occupied by shared libs                          |
└---------------------------------------------------------------------------------┘
┌--------------------------------------┐
|      process #1 allocated memory     |
| ┌-------------┐ ┌──────────────────┐ |
| | free memory | │ memory under use │ |
| └-------------┘ └──────────────────┘ |
└--------------------------------------┘
```

<hr>

### # how to compute uss/pss/rss for certain proc
uss = sum of /proc/${pid}/smaps private_clean + private_dirty  
pss = sum of /proc/${pid}/smaps pss  
rss = sum of /proc/${pid}/smaps rss

program to automatically compute the statistics:
```text
todo
```

<hr>

### # illustration of /proc/${pid}/smaps fields (todo)
```text
/proc/13 # cat smaps | head -n 60
55ffec366000-55ffec36c000 r--p 00000000 fe:01 3685107                    /bin/busybox
Size:                 24 kB
KernelPageSize:        4 kB
MMUPageSize:           4 kB
Rss:                  24 kB
Pss:                   3 kB
Shared_Clean:         24 kB
Shared_Dirty:          0 kB
Private_Clean:         0 kB
Private_Dirty:         0 kB
Referenced:           24 kB
Anonymous:             0 kB
LazyFree:              0 kB
AnonHugePages:         0 kB
ShmemPmdMapped:        0 kB
FilePmdMapped:         0 kB
Shared_Hugetlb:        0 kB
Private_Hugetlb:       0 kB
Swap:                  0 kB
SwapPss:               0 kB
Locked:                0 kB
THPeligible:    0
VmFlags: rd mr mw me
55ffec36c000-55ffec406000 r-xp 00006000 fe:01 3685107                    /bin/busybox
Size:                616 kB
KernelPageSize:        4 kB
MMUPageSize:           4 kB
Rss:                 192 kB
Pss:                  41 kB
Shared_Clean:        192 kB
Shared_Dirty:          0 kB
Private_Clean:         0 kB
Private_Dirty:         0 kB
Referenced:          192 kB
Anonymous:             0 kB
LazyFree:              0 kB
AnonHugePages:         0 kB
ShmemPmdMapped:        0 kB
FilePmdMapped:         0 kB
Shared_Hugetlb:        0 kB
Private_Hugetlb:       0 kB
Swap:                  0 kB
SwapPss:               0 kB
Locked:                0 kB
THPeligible:    0
VmFlags: rd ex mr mw me
55ffec406000-55ffec428000 r--p 000a0000 fe:01 3685107                    /bin/busybox
Size:                136 kB
KernelPageSize:        4 kB
MMUPageSize:           4 kB
Rss:                 136 kB
Pss:                  22 kB
Shared_Clean:        136 kB
Shared_Dirty:          0 kB
Private_Clean:         0 kB
Private_Dirty:         0 kB
Referenced:          136 kB
Anonymous:             0 kB
LazyFree:              0 kB
AnonHugePages:         0 kB
...
```
