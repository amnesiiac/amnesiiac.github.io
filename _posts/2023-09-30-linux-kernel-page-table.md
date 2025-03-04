---
layout: post
title: "page table (linux, kernel)"
author: "melon"
date: 2023-09-30 21:54
categories: "2023"
tags:
  - linux
  - kernel
---


### # why need multilayer page mapping? (ref: bilibili)
1-level page mapping: the single page table must be allocated in memory;

2-level page mapping: the first page table must be allocated in memory, while left pte page allocated in mem
on need.

<hr>

### # code in action
pagetable_example.c

```text
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/mm.h>
#include <linux/highmem.h>
#include <linux/slab.h>

static int __init pagetable_example_init(void) {
    struct page* page;
    void* vaddr;
    unsigned long paddr;

    page = alloc_page(GFP_KERNEL);                                  // physical page allocation
    if (!page) {
        pr_err("failed to allocate page\n");
        return -ENOMEM;
    }

    paddr = page_to_phys(page);                                     // get physical page addr
    pr_info("allocated page at physical address: 0x%lx\n", paddr);

    vaddr = vmap(&page, 1, VM_MAP, PAGE_KERNEL);                    // map physical page to virtual mem space
    if (!vaddr) {
        pr_err("failed to map page to virtual address\n");
        __free_page(page);
        return -ENOMEM;
    }
    pr_info("mapped page to virtual address: %p\n", vaddr);

    strcpy(vaddr, "hello, kernel page!");                           // write data to virtual mem

    pr_info("data in mapped page: %s\n", (char *)vaddr);            // read data from virtual mem

    vunmap(vaddr);                                                  // unmap virtual mem for process

    __free_page(page);                                              // release physical page allocated

    return 0;
}

static void __exit pagetable_example_exit(void) {
    pr_info("page table example module unloaded\n");
}

module_init(pagetable_example_init);
module_exit(pagetable_example_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Your Name");
MODULE_DESCRIPTION("A simple example of page table mapping in the Linux kernel");
```

makefile:

```text
obj-m += pagetable_example.o

all:
    make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
    make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

load kernel module:

```text
$ sudo insmod pagetable_example.ko
```

uninstall kernel module

```text
$ sudo rmmod pagetable_example
```

<hr>

### # reference
https://mp.weixin.qq.com/s/USS7-7zCEUgho6W6Zxz_QQ  
https://www.baeldung.com/cs/multi-level-page-tables  
BV1ug411S7Da
