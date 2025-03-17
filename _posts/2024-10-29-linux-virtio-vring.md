---
layout: post
title: "vring intro: the virtio data plane design (linux, virtualization)"
author: "melon"
date: 2024-10-29 20:28
categories: "2024"
tags:
  - linux
  - virtualization
  - todo
---

what is vring? why we need it? the virtio standard is a protocol operated on shared memory to transfer data
between the frontend guest kdrv and the backend dev on host, and is designed to improve the io efficiency.
the shared mem to enable virtio msg transport is vring, which facilitate the communication between guest &
host kernel in an efficient, reliable way.

<p style="margin-bottom: 20px;"></p>

2 where is the memory address stored in vring fd point to? what does the mem used for?  
vring (shared mem between guest & host) is applied by guest kernel driver, thus address of vring fd is the
same as guest physical address (GPA), by which the guest kernel is able to manage the mem without awareness of
host memory layout.

<hr>

### # ways to facilitate shared memory between guest & host kernel
1 for device natively supported by hypervisor, the GPA is located in the process addr space of hypervisor, thus
the hypervisor can access it naturally.  

<p style="margin-bottom: 20px;"></p>

2 for customized device (vhost-net dev or vhost-user dev), extra memory mapping infra is needed:

2.1 vhost-net device:  
a) during kernel init process, the hypervisor pass the vring info (GPA, userspace_addr, size) to the vhost-net
guest kernel module by ioctl;  
b) the vhost-net ko save the memory mapping of GPA and userspace_addr (hypervisor ctx);  
c) vhost-net ko start kthread, store the mapping between kthread and hypervisor instance, store the
page table info of the hypervisor process.
in guest kernel thread runtime, the page table of hypervisor is visible to guest kernel, which enable guest ko
access the vaddr of hypervisor process ctx.

2.2 vhost-user device:  
todo!!!

<p style="margin-bottom: 20px;"></p>

3 for real physical device, the input output memory management unit (iommu) is needed to enable guest vaddr &
host physical addr translation (pci-passthrough), to allow vm to access the real physical devices in protected
& isolated way.

<hr>

### # vring definition & implementation (ref: virtio-v1.2-csd01)

```text
struct virtq {
    struct virtq_desc desc[ Queue Size ];             // the actual descriptors (16 bytes each)
    struct virtq_avail avail;                         // a ring of available descriptor heads with free-running idx
    u8 pad[ Padding ];                                // padding to the next queue align boundary
    struct virtq_used used;                           // a ring of used descriptor heads with free-running idx
};
```

<p style="margin-bottom: 20px;"></p>

5 three components of vring:  
1) descriptor table: a series of descriptors, with each associated with a mem buf (virq_desc).

```text
struct virtq_desc {                                   // descriptor table entry
    le64 addr;                                        // guest physical address (GPA)
    le32 len;                                         // buffer length

#define VIRTQ_DESC_F_NEXT      1                      // mark buf as continuing via the next field
#define VIRTQ_DESC_F_WRITE     2                      // mark buf as device write only (otherwise read-only)
#define VIRTQ_DESC_F_INDIRECT  4                      // mark buf contains a list of buffer descriptor

    le16 flags;                                       // control info: flags defined above
    le16 next;                                        // if chained desc, set as next desc id
};
```

2) available ring: used by guest drv to notify external dev of the existence of available descriptors,
the avail fd needn't be used at once.

```text
struct virtq_avail {
#define VIRTQ_AVAIL_F_NO_INTERRUPT  1
    le16 flags;                                       // control info of the que
    le16 idx;                                         // idx of next avail fd entry in ring[] to operate on
    le16 ring[ /* queue size */ ];                    // virtio queue: ring
    le16 used_event;                                  // only if VIRTIO_F_EVENT_IDX
};
```

e.g., virtio-net drv will provide some fd for pkt acceptance, which are used when the packet arrived.

3) used ring: used by external dev to notify guest drv about used descriptors.

```text
struct virtq_used {
#define VIRTQ_USED_F_NO_NOTIFY  1
    le16 flags;                                       // control info of the que
    le16 idx;                                         // idx of next used fd in ring[] to operate on
    struct virtq_used_elem ring[ /* queue size */ ];
    le16 avail_event;                                 // only if VIRTIO_F_EVENT_IDX
};

struct virtq_used_elem {                              // le32 is used for ids (padding reasons)
    le32 id;                                          // idx of start of used descriptor chain
    le32 len;                                         // num of bytes written to the dev writable portion (buf from
};                                                    // descriptor chain)
```

e.g., vhost-user client (dev) received a packet, load it in available decriptor, update used descriptor and
nofity guest drv.

<p style="margin-bottom: 20px;"></p>

6 chained descriptors  
when size of data transfered is larger than single fd (buf) storage, multiple fd will be involved: chained
descriptor.
except for the very last descriptor, all deccriptor before should set NEXT flag and set desc.next as the
id of descriptor.  
note: when updating the chained descriptors into avail ring[], only need add the first descriptor of
the chain to the avail ring[].

<p style="margin-bottom: 20px;"></p>

7 communication between vring frontend & backend  
1) guest driver allocate ring.  
2) guest driver update the descriptors table: following show the desc table with an entry for a write-only
buffer (starts with 0x8000, len 2000).

```txt
                   ┌───────────┐       ┌──────────────────────────────────┐
 init avail que -> │ available │       │         descriptor area          │
                   ├───────────┤       ├────────┬────────┬────────┬───────┤
                   │    idx    │       │ buffer │ len    │ flags  │ next  │
                   ├───────────┤       ├────────┼────────┼────────┼───────┤
     next idx 0 -> │     0     │       │ 0x8000 │ 0x2000 │ W      │ 0     │ <- desc table entry
                   ├───────────┤       ├────────┼────────┼────────┼───────┤
   empty ring[] -> │   ring[]  │       │ ...    │        │        │       │
                   ├───────────┤       └────────┴────────┴────────┴───────┘
                   │    ...    │
                   ├───────────┤
                   │    ...    │
                   └───────────┘
```

3) the guest driver update the available ring que:

```text
avail->ring[idx] = desc;                              // set ring[idx] corresponding desc table entry
idx++;                                                // update the next ring idx to operate on
```

```txt
                   ┌───────────┐       ┌──────────────────────────────────┐
      avail que -> │ available │       │         descriptor area          │
                   ├───────────┤       ├────────┬────────┬────────┬───────┤
                   │    idx    │       │ buffer │ len    │ flags  │ next  │
                   ├───────────┤       ├────────┼────────┼────────┼───────┤
     next idx 1 -> │     1     │   ┌───+ 0x8000 │ 0x2000 │ W|N    │ 1     │
                   ├───────────┤   │   ├────────┼────────┼────────┼───────┤
   empty ring[] -> │   ring[]  │   │   │  ...   │        │        │       │
                   ├───────────┤   │   └────────┴────────┴────────┴───────┘
 buf 0 is avail -> │     0     ├───┘
                   ├───────────┤
                   │    ...    │
                   └───────────┘
```

4) guest driver notify the vhost device about the available ring buffer entry.  
5) vhost device noticed the available ring buffer updates, and after usage, update the used ring que.

```txt
                   ┌───────────┐       ┌──────────────────────────────────┐       ┌───────────┐
                   │ available │       │         descriptor area          │       │ used ring │
                   ├───────────┤       ├────────┬────────┬────────┬───────┤       ├───────────┤
                   │    idx    │       │ buffer │ len    │ flags  │ next  │       │    idx    │
                   ├───────────┤       ├────────┼────────┼────────┼───────┤       ├───────────┤
                   │     1     │   ┌───+ 0x8000 │ 0x2000 │ W|N    │ 1     +───┐   │     1     │ <- next used ring idx
                   ├───────────┤   │   ├────────┼────────┼────────┼───────┤   │   ├───────────┤
                   │   ring[]  │   │   │ 0xD000 │ 0x2000 │ W      │ ...   │   │   │   ring[]  │
                   ├───────────┤   │   ├────────┼────────┼────────┼───────┤   │   ├───────────┤
                   │     0     ├───┘   │  ...   │        │        │       │   └───┤  0|0x3000 │ <- update used
                   ├───────────┤       └────────┴────────┴────────┴───────┘       ├───────────┤    id / buflen
                   │    ...    │                                                  │    ...    │
                   └───────────┘                                                  └───────────┘
```

6) vhost device use v-irq to notify the guest kdrv about the newly-used descriptors, please deal with it.  
7) the communication between host & guest overview architecture.

```txt
           ┌─QEMU PROCESS──────────────────────────────────────────────────────────────────┐
           │                                                                               │
           │  ┌─VM PROCESS──────────────────────────────────────────────────────────────┐  │
           │  │                                                                         │  │
           │  │                                                        guest user space │  │
           │  │ ─────────────────────────────────────────────────────────────────────── │  │
           │  │                                                      guest kernel space │  │
           │  │                         ┌─VIRTIO-NET DRIVER──────────────┐              │  │
           │  │                         │    ┌──────────────────────┐    │              │  │
           │  │                         │    │ ┌──────────────────┐ │    │              │  │
           │  │                         │    │ │ descriptor table │ │    │              │  │
           │  │                         │    │ ├──────────────────┤ │    │<-------┐     │  │
           │  │              ┌--------->│    │ │ avail ring[]     │ │    │-------┐|     │  │
           │  │              |          │    │ ├──────────────────┤ │    │       ||     │  │
           │  │              |          │    │ │ used ring[]      │ │    │       ||     │  │
           │  │              |          │    │ └──────────────────┘ │    │       ||     │  │
           │  │              |          │    └────────+────+────────┘    │       ||     │  │
           │  │              |          └─────────────|────|─────────────┘       ||     │  │
           │  └──────────────|────────────────────────|────|─────────────────────||─────┘  │
           │                 |                        |    |                     ||        │
           │       virtio control plane        virtio shared memory        notifications   │
           │      pci transport protocol            (data path)          vmexit & vcpu irq │
           │          (control path)                  |    |                     ||        │
           │                 |          ┌─────────────|────|─────────────┐       ||        │
           │                 |          │    ┌────────+────+────────┐    │       ||        │
           │                 |          │    │ ┌──────────────────┐ │    │       ||        │
           │                 |          │    │ │ descriptor table │ │    │       ||        │
           │                 |          │    │ ├──────────────────┤ │    │       ||        │
           │                 └--------->│    │ │ avail ring[]     │ │    │<---┐  ||        │
           │                            │    │ ├──────────────────┤ │    │---┐|  ||        │
           │                            │    │ │ used ring[]      │ │    │   ||  ||        │
           │                            │    │ └──────────────────┘ │    │   ||  ||        │
           │                            │    └──────────────────────┘    │   ||  ||        │
           │                            └─VIRTIO-NET DEVICE──────────────┘   ||  ||        │
           └─────────────────────────────────────────────────────────────────||──||────────┘
                                                                             ||  ||             host user space
 ────────────────────────────────────────────────────────────────────────────||──||────────────────────────────
                                                                             ||  ||           host kernel space
                                      ┌──────────────────────────────────────┼┼──┼┼────────┐
                                      │                                      |└--┘|        │
                                      │              kernel virtual machine  └----┘        │
                                      │                                                    │
                                      └────────────────────────────────────────────────────┘
```
