---
layout: post
title: "introduction and overview (linux kernel architecture ch1)"
author: "melon"
date: 2023-05-02 12:17
categories: "2023"
tags:
  - linux
  - ongoing
---

### # high-level overview of linux kernel structure
```txt
          ┌────────────────────────┐
          │      applications      │
          └─────────┬────+─────────┘
user space          │    │                               
          ┌─────────+────┴─────────┐
          │       c-library        │   
          └─────────┬────+─────────┘
--------------------┼----┼------------------------ sys call / sys call return
      ┌─────────────+────┴────────────────┐
      │ ┌───────────┐    ┌──────────────┐ │
      │ │core kernel│<──>│device drivers│ │
      │ └───────────┘    └──────────────┘ │
      └─────────────┬────+────────────────┘
kernel space        │    │  
          ┌─────────+────┴─────────┐
          │        hardware        │   
          └────────────────────────┘
```

<hr>

### # the layers of complete linux system
```txt
      ┌──────────────────────────────────┐
      │ ┌──────────┐    ┌──────────────┐ │
      │ │networking│<──>│device drivers│ │
      │ └──────────┘    └────┬────+────┘ │
      │                      │    │      │
      │  ┌─────────┐    ┌────+────┴────┐ │
      │  │   VFS   │<──>│ filesystems  │ │
      │  └─────────┘    └──────────────┘ │
      └─────┬───+────────────┬────+──────┘
            │   │            │    │
      ┌─────+───┴──────┬─────+────┴──────┐
      │   Memory mgnt  │  Process mgnt   │
      ├────────────────┴─────────────────┤
      │    Architecture specific code    │
      └──────────────────────────────────┘
```

<hr>

### # process with/without threads
```txt
    process without threads   process with threads
    ┌─────────┐               ┌─────────┐
    │    ^    │               │ ^  ^  ^ │
    │    │    │               │ │  │  │ │
    │    │    │               │ │  │  │ │
    └─────────┘               └─────────┘
```

<hr>

### # virtual address space division
The division does not depend on how much RAM is available. Each user space process 'think' itself has 3GB of memory.
```txt
    ┌──────────────────────────┐ <-- 2^32 bytes(4Gb) / 2^64 bytes(8Gb) in 1byte=8bit system
    │   kernel address space   │   
    ├──────────────────────────┤ <-- TASK_SIZE (an architecture specific constant)
    │                          │     e.g. IA32, TASK_SIZE=3Gb
    │    user address space    │ 
    │                          │ 
    └──────────────────────────┘ <-- 0
```

<hr>

### # execution in kernel mode & user mode
```txt
    ┌────────┐             ┌────────┐             ┌────────┐                ┌────────┐
    │ kernel │      ┌----->│ kernel │------┐      │ kernel │        ┌------>│ kernel │ interrupt context
    ├────────┤      |      ├────────┤      |      ├────────┤        |       ├────────┤
    │        │      |      │        │      |      │        │        |       │--------│ user space code 
    │  user  │------┘      │  user  │      └----->│  user  │--------┘       │--user--│ cannot be accessed
    │        │ system call │        │ return from │        │ async hardware │--------│ by kernel
    └────────┘             └────────┘ system call └────────┘ interrupt      └────────┘ 
```

<hr>

### # physical address space & virtual address space
• in most cases, the virtual address space are bigger then physical RAM spaces to the system.  
• phsical pages are often called page frames. the virtual address units are called pages, the pages & page frames are of equal size.  
• process a has 8 page, the process b has 7 page, the RAM has 4 page frames.  
• the mapping between physical space to virtual address can lift the separation between processes: the page 4 of process a and the page 2 of process are mapped to the same RAM frame. It's kernel responsiblility to which area to be shared between proccesses.
```txt
      proc a          RAM           proc b
    ┌────────┐__   ┌────────┐_____┌────────┐
    ├────────┤  |__├────────┤   __├────────┤
    ├────────┤   __├────────┤__|  ├────────┤
    ├────────┤__|  ├────────┤__   ├────────┤
    ├────────┤     └────────┘  |  ├────────┤
    ├────────┤                 |  ├────────┤
    ├────────┤                 |__├────────┤
    ├────────┤                    └────────┘
    └────────┘  
```

### # page tables
• the following is a 3-level page table, while linux typically use 4-level page table.  
• MMU (memory management unit) is used to optimize the memory access operations.  
• TLB (translation lookaside buffer) is a fast CPU cache used to access the content without using page tables.  
```txt
    page global     page middle   page table     
    directory       directory     entry
    ┌─────────────┬─────────────┬─────────────┬────────────────┐
    │     PGD     │     PMD     │     PTE     │     offset     │ virtual address
    └─┬───────────┴──┬──────────┴────┬────────┴───┬────────────┘     (index)
      |              |               |            |
      |  ┌───────┐   |__┌───────┐    | ┌───────┐  |   ┌───────────┐
      |  │       │  ┌──>│pointer├──┐ | │       │  |   │           │
      |__├───────┤  │   ├───────┤  │ | │       │  ├──>│byte offset│
         │pointer├──┘   │       │  │ | │       │  │   │           │
         ├───────┤      │       │  │ |_├───────┤  │   └───────────┘
         │       │      │       │  └──>│pointer├──┘    page frame
         └───────┘      └───────┘      └───────┘      
         global         middle         page table
         page table     page table
```

<hr>

### # memory mappings
• Mappings are also used directly in the kernel when implementing device drivers. The input and output areas of peripheral devices can be mapped into virtual address space; reads and writes to these areas are then redirected to the devices by the system, thus greatly simplifying driver implementation.  
• ioremap: a function in linux kernel to map physical memory(hardware register/other I/O devices) into kernel virtual address space.  
• mmap: a system call to map files/devices into memory, which allows a process to acess the contents of file/devices as if in RAM.  
• code example to illustrate ioremap usages:
```c
# AUTHOR: GPT-3.5 turbo
#include <linux/module.h>
#include <linux/init.h>
#include <linux/io.h>

#define MY_DEVICE_BASE 0x10000000
#define MY_DEVICE_SIZE 0x1000

void __iomem *my_device_memory;

static int __init my_device_init(void){
    my_device_memory = ioremap(MY_DEVICE_BASE, MY_DEVICE_SIZE);
    if (!my_device_memory){
        printk(KERN_ERR "Failed to map device memory\n");
        return -ENOMEM;
    }
    // initialize device registers
    return 0;
}

static void __exit my_device_exit(void){
    // cleanup device resources
    iounmap(my_device_memory);
}

module_init(my_device_init);
module_exit(my_device_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("Your Name");
MODULE_DESCRIPTION("Example driver using ioremap");
```

<hr>

### # allocation of physical memory
1 background:  
• kernel keep track of the pages allocated/free, and prevent 2 processes from using the same area in RAM.  
• kernel also make sure the allocation/free action of memory is as fast as possible.  
• kernel can only allocate whole page frames.  
• dividing memory into smaller portions is delegated to the stdlib in userspace.  

<hr>

### # tricks of allocating the physical memory inside kernel:
#### (1) the functionality of the buddy system algorithm:  
• reduce memory fragmentation and improve memory utilization.  
• reduce the amount of wasted space caused by small gaps between allocated blocks.  
#### (2) buddy system allocation algorithm
• initializing memory pool: the memory pool is divided into blocks of fixed sizes, each of which is a power of two.
```txt
    ┌───────────────────────────────────────────────────────┐
    │                         8-page                        │ (block-1)
    └───────────────────────────────────────────────────────┘
    ┌───────────────────────────┬───────────────────────────┐
    │           4-page          │           4-page          │ (block-2)
    └───────────────────────────┴───────────────────────────┘
    ┌─────────────┬─────────────┬─────────────┬─────────────┐
    │    2-page   │    2-page   │    2-page   │    2-page   │ (block-3)
    └─────────────┴─────────────┴─────────────┴─────────────┘
    ┌──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┐
    │1-page│1-page│1-page│1-page│1-page│1-page│1-page│1-page│ (block-4)
    └──────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┘
```
• allocating memory: search for approprite size of free block to use, if the block size is larger then twice the required, split it to halfs, return one of them and add the left to memory pool.  
• free memory: when a block of memory is free, check the existence of if its buddy, if existed, merge them and add back to memory pool.  
• coalescing(merge): if two adjacent blocks of memory are both free, they can be merged into a single larger block.  
• splitting: when a block of memroy is splitted into 2 halfs, each half is assigned an identifier encoding its size/address in memory.  
#### (3) slab cache
• use slab cache for memory blocks much smaller than a whole page frame, it cannot invoke userspace stdlib.  
• not only performs allocation but also implements a generic cache for frequently used small objects.  
```txt
          ┌───────────────────────────┐
          │    generic kernel code    │
          └──┬────+────────┬────+─────┘
             │    │        │    │ - allocation of blocks(<=frames)
allocation of│    │        │    │ - provide generic cache 
page frames  │    │  ┌─────+────┴─────┐
             │    │  │ slab allocator │
       ┌─────+────┴──┴────────────────┴────────┐
       │       the buddy system allocator      │
       └───────────────────────────────────────┘
       ┌──┬──┬──┬──┬──┬──┬───────────────┬──┬──┐
       │  │  │  │  │  │  │  page frames  │  │  │
       └──┴──┴──┴──┴──┴──┴───────────────┴──┴──┘
```
#### (4) page swapping & reclaim
• swapping enables RAM to be enlarged virtually by using disk space as extended memory.  
• infrequently used pages can be written to hard disk when the kernel requires more RAM, once they are use, they can be swapped back to RAM.  
• swapped-out pages are identified by a special entry in the page table, when a process attempts to access a page of this kind, the CPU initiates a page fault that is intercepted by the kernel. Because it is unaware of the page fault, swapping in and out of the page is totally invisible to the process.

<hr>

### # timing

<hr>

### # system call
• process management.  
• signals.  
• files.  
• directory and filesystems.  
• protection mechanisms.  
• timer functions.  
#### (1) why system calls cannot implmented in usespace (user lib)
• Because special protection mechanisms are needed to ensure that system stability/security.  
• Some system calls are reliant on kernel-internel structures/functions to yield data/results.
#### (2) system call implementation is a architecture-specific & processor-specific concept
• The processor must change the privilege level and switch from user mode to system mode.  
There no standard way of doing this because each hardware platform offers specific mechanisms. Even on the same architecture, these operations depend on processor type.

<hr>

### # device drivers, block and character devices
• Everything is a file.  
• The task of a device driver is to support application communication via device files (enable data to be read from and written to a device in a suitable way).
#### (1) peripheral devices
• character devices: deliver a continuous stream of data that applications read sequentially. Random access is not possible, allow data to be read/written byte-by-byte or character-by-character.  
```txt
/dev/tty: represents a terminal device (e.g. serial port, console...)
/dev/null: a device that discard all data written to it, return EOF when read.
/dev/full: a device that return null bytes when read, return error when written to. 
/dev/random: a device that generates random numbers.
/dev/urandom: a device that gnerate pseudo random numbers.
```
• block devices: allow applications to address their data randomly, and to freely select the position at which they want to read/write data.  

<hr>

### # networking

<hr>

### # filesystems
The kernel provide an additional software layer to abstract the special features of the various low-level filesystems from the application layer and from the kernel itself.
```txt
        ┌─────────────────────┐ 
        │applications and libc│ 
        └───────┬────+────────┘ 
----------------┼----┼--------------------- sys call / sys call return
   ┌────────────+────┴─────────────┐ 
   │      virtual file systems     │ (abstract layer: unified api)
   └─┬──+────────┬─+──────────┬──+─┘ 
    ┌+──┴┐      ┌+─┴┐       ┌─+──┴─┐
    │ExtN│      │XFS│       │ProcFS│ (different FS supported by linux)
    └┬──+┘      └┬─+┘       └─┬──+─┘
  ┌──+──┴────────+─┴──────────+──┴──┐
  │ ┌──────────┐    ┌────────────┐  │
  │ │page cache│<──>│buffer cache│  │ (improve performance & reduce disk accesses)
  │ └──────────┘    └────────────┘  │
  └─────┬───+───────────────────────┘
        │   │
    ┌───+───┴───┐    ┌──────────────┐    ┌─────────┐
    │block layer│<──>│device drivers│<──>│hard disk│
    └───────────┘    └──────────────┘    └─────────┘
```

<hr>

### # modules and hot-plugging

<hr>

### # cache
• The kernel implements access to block devices by means of page memory mappings, thus related caches are also organized into pages. When caching, the whole pages are cached, thus name page cache.  
• The buffer cache is far less important, which is used to cache data that are not organized into pages. Now, the buffer cache is mostly superseded by page cache.  
• page cache: stores pages of files to optimize file I/O performance.  
• buffer cache: stores disk blocks to optimize block I/O performance.  

<hr>

### # list handling
