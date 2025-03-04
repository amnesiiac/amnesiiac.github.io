---
layout: post
title: "cat /proc/meminfo && free (linux)"
author: "melon"
date: 2023-12-27 21:14
categories: "2023"
tags:
  - linux
---

### # free outputs
free cmd supported display unit:
```text
$ free -b (byte)
$ free -k (kilo)
$ free -m (mega)
$ free -g (giga)
```

typical free output in kbytes:
```text
$ free -k
              total        used        free      shared  buff/cache   available
Mem:      107151868    31966516    13121096    15520008    62064256    58613996
Swap:             0           0           0
```
total = used + free + buff/cache.  
used: taken up by proc + kernel.  
free: mem that totally free, not taken up be proc, kernel or buffer/cache.  
available = free + recollectable part in buffer/cache, the max available mem could be allocated.  
shared = tmpfs(temporary fs in mem, lost on reboot) + shmem(shared mem between proc).

<hr>

### # free outputs corresponding keywords in cat /proc/meminfo
```text
$ free -k
           MemTotal     MemTotal-MemFree    MemFree       Shmem    Buffers/Cache   MemAvailable
Mem:      107151868             31966516   13121096    15520008         62064256       58613996
          SwapTotal   SwapTotal-SwapFree   SwapFree
Swap:             0                    0          0
```

<hr>

### # cat /proc/meminfo
kernel docs: Documentation/filesystems/proc.txt and Documentation/vm/hugetlbpage.txt
```text
# high-level statistics
MemTotal:         107151868 kB  (total mem)
MemFree:           10772220 kB  (mem not in use by any proc)
MemAvailable:      59120812 kB  (mem available for allocation of any proc)

Buffers:           10663016 kB  (temporary storeage element in mem)
Cached:            45436420 kB  (page cache size: cache for file read from disk: tmpfs+shmem, excluding swapcached)
SwapCached:               0 kB  (recently used swapped cache)
SwapTotal:                0 kB  (total swap space available)
SwapFree:                 0 kB  (no used swap space)

# detailed statistics
Active:            47650364 kB  (memory used recently, not suitable to reclaim for new proc or swap out)
Inactive:          34963112 kB  (memory not used recently, can reclaim for new proc or swap out)
Active(anon):      29186112 kB  (annonymous mem used recently, usually not suitable for swap out)
Inactive(anon):    12799348 kB  (annonymous mem not used recently, can swap out)
Active(file):      18464252 kB  (page cache mem used recently, usually not suitable for reclaim)
Inactive(file):    22163764 kB  (page cache mem not used recently, can reclaim for no big impact on performance)
Unevictable:           1596 kB  (unevictable pages cannot be swapped out for some reasons)
Mlocked:                 60 kB  (page cache locked to mem by mlock() syscall, part of uevictable)

# mem statistics
Dirty:                 2104 kB  (mem to be written back to disk)
Writeback:                0 kB  (mem is actively writting back to disk)
AnonPages:         25864844 kB  (mem used by pages that are not backed by files and mapped into userspace page table)
Mapped:            13011960 kB  (mem used for files that have been mmaped, such as libraries)
Shmem:             15473412 kB  (mem used by shared memory: shmem and tmpfs)

KReclaimable:       8798172 kB  (kernel allocated mem, reclaimable under mem pressure, including SReclaimable)
Slab:              11608552 kB  (mem used by the kernel to cache data structures for its own use,
                                 contiguous pages of cached allocated by slab allocator)
SReclaimable:       8798172 kB  (part of slab that can be reclaimed, such as caches)
SUnreclaim:         2810380 kB  (part of slab that cannot be reclaimed even when lacking memory)
KernelStack:         234496 kB  (mem used by the kernel stack allocations done for each task in the system)
PageTables:          330572 kB  (mem dedicated to the lowest page table level)
NFS_Unstable:             0 kB  (mem of NFS pages sent to the server but not yet committed to the stable storage)
Bounce:                   0 kB  (mem used for the block device "bounce buffers")
WritebackTmp:             0 kB
CommitLimit:       53575932 kB
                                              overcommit_ratio
([total RAM pages] - [total hugTLB pages]) * ────────────────── + [total swap pages] = CommitLimit
                                                    100
Committed_AS:     141450660 kB
VmallocTotal:   34359738367 kB  (mem of total allocated virtual address space)
VmallocUsed:         364300 kB  (mem of used virtual address space)
VmallocChunk:             0 kB  (contiguous block mem of available virtual address space)
Percpu:              282080 kB
HardwareCorrupted:        0 kB
AnonHugePages:     19386368 kB
ShmemHugePages:           0 kB
ShmemPmdMapped:           0 kB
FileHugePages:            0 kB
FilePmdMapped:            0 kB
HugePages_Total:          0     (num of total hugepages for the system)
HugePages_Free:           0     (num of total available hugepages for the system)
HugePages_Rsvd:           0     (num of unused huge pages reserved for hugetlbfs)
HugePages_Surp:           0     (num of surplus huge pages)
Hugepagesize:          2048 kB  (size for each huge page unit)
Hugetlb:                  0 kB
DirectMap4k:       14022504 kB
DirectMap2M:       93980672 kB
DirectMap1G:        3145728 kB
```

<hr>

### # dirty mem
dirty memory is memory representing data on disk that has been changed but has not yet been written out to disk,
including:  
1 memory containing buffered writes that have not been flushed to disk yet.  
2 regions of memory mapped files that have been updated but not written out to disk yet.  
3 pages that are in the process of being written to swap space but have changed since the system started
writing them to swap space.

having a few MB of dirty memory is normal on any reasonably busy system,
and even spikes up to a few hundred MB are not unusual.  
the only time to really be worried about it is if it's consistently very high,
which is usually a sign that your disks are a performance bottleneck for your system.
