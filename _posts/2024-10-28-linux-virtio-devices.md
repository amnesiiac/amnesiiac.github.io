---
layout: post
title: "virtio device & driver abstraction (linux, virtualization)"
author: "melon"
date: 2024-10-28 20:43
categories: "2024"
tags:
  - linux
  - virtualization
  - todo
---

### # relationship between qemu & kvm
when working together, kvm arbitrates access to the cpu and memory, and qemu emulates the hardware resources
(hard disk, video, usb, etc.).
when working alone, qemu emulate both cpu and hardware.

<hr>

### # virtio spec & vhost protocol
1 virtio spec: maintained by oasis, define how to create the control & data plane between guest & host:
the buffer & vring layout in data plane is described in details in the specification.

2 vhost protocol: allow virtio data plane implementation to be offloaded to somewhere for
performance enhancement (implemented as user or host process).

<hr>

### # virtio networking components
1 control plane: for capability negotiation between the host & guest, for data plane management.  
it is implemented in qemu process based on the virtio spec, while the data plane is out of qemu, why?
reason: if data plane is implemented in qemu (as a linux userspace process), the operation to exchange
data (pkt) between userspace & kernel will be too expensive!

2 data plane: used for transferring the actual data (pkt) between host and guest.  
it is designed to be as efficient as possible to move the packet fast, while the control plane
is designed to be as flexible as possible to support different device and vendor in future architectures.

3 data plane implementation offloading (vhost protocol's function): to help us implement a data plane
going directly from the host to the guest (bypass the qemu process); by vhost offloading, we eliminate
the latency of data copy & process to enhance the performance.

<hr>

### # virtio device introduction
virtio device can be discovered by pci, mmio, channel io.

virtio dev is a device that exposes virtio interface for software to manage & exchange info.
virtio dev can be exposed to the emulated environment by pci, memory mapping io (expose the dev in a mem region)
and s/390 channel io.
part of the communication need to be delegated to mechanism like device discovery.

virtio device's main task: convert the signal from the form outside the virtual env (container, vm\...),
to the form needed for data exchange on the virtio dataplane (vring\...).
the signal could be physical (electricity or light from a nic) or already virtual (just a representation of pkt).

<hr>

### # virtio interface: the mandatory component of virtio device
this section will introduce the basic components of virtio interface, and describe how virtio dev & drv
communicate with each other by them.

$ 1 device status field (the traffic light)  
the dev status field is a sequence of bits the dev & drv used to for their init,
which act like traffic lights, the status of dev is indicated by set and clear the bits:

1.1 guest drv set the bit ACKNOWLEDGE (0x1) in device status field to indicate the ack for the dev,
and set the bit DRIVER (0x2) to indicate the init is in progress.  
1.2 guest drv start feature negotiation using the feature bits, and sets bit DRIVER_OK (0x4) and
FEATURES_OK (0x8) to acknowledge the features, so the following communication can start.  
1.3 however, if the dev want to indicate a fatal failure, it can set bit DEVICE_NEEDS_RESET (0x40),
and the drv can do the same with bit FAILED (0x80).

the device communicate the location of these bits using transport specific method, like pci scanning or knowing
the address for mmio.

<p style="margin-bottom: 20px;"></p>

$ 2 feature bits (set communication agreement points)  
device feature bits are used to communicate what feature it support, and to agree with the driver about
what of the feature bits will be used.

the feature bits can be either device-generic or device-specific:
for dev-generic feature, a bit can acknowledge the drv what memory mode can be used.
for dev-specific feature, a bit can represent different kinds of offloads supported: checksum or
scatter-gather (as a nic dev).

after the dev initialied, the guest drv read the feature bits the dev offers, and send back the subset
which it can handle.
if they agree with each other, the drv will allocate and inform the virtqueues established to the dev,
with all other config needed.

<p style="margin-bottom: 20px;"></p>

$ 3 notifications (the peer got work to do)  
by notification, dev & drv is able to notify each other that they got info to communicate.
the notification semantic is specified in the standard, while the implementation are transport specific:
can either by a irq (pci) or a write to a specific memory location (mmio).
the dev & drv need to expose one notification method at least.

<p style="margin-bottom: 20px;"></p>

$ 4 device configuration space (for dev initialization)  
configuration space is used for dev init, which generally wont frequently change.
the config space is organized as little-endian byte order, and the driver must read & write in 32-bit format.
some config are optional, which is determined by corresponding feature bits effectiveness.

<p style="margin-bottom: 20px;"></p>

$ 5 virtqueues (the communication media)  
virtqueue is a queue of guest’s buffer that the host consume (r/w to guest) and return msg from host to guest.
the memory layout of a virtqueue is like a circular ring (vring).
each vqueue including: 1 descriptor table; 2 available ring; 3 used ring.
all the three parts are stored in guest memory, with consecutive physical address.
requirements of the alignment & size of the three parts are as:

virtqueue part    | alignment  | size in bytes
---               | ---        | ---
descriptor table  | 16         | 16 * vque_size
available ring    | 2          | 6 + 2 * vque_size
used ring         | 4          | 6 + 8 * vque_size

<p style="margin-bottom: 20px;"></p>

$ 6 steps for the guest driver to send a buffer to the virtio dev  
6.1 the guest driver fill one or serveral chained slot in the descriptor table.  
6.2 the guest driver write the descriptor index to the available ring.  
6.3 the guest driver write notification to the device.  
6.4 after device use the buffer, it write the descriptor index to the used ring.  
6.5 the device send an irq to the driver.

<p style="margin-bottom: 20px;"></p>

$ 7 more details can refer to blog post vring intro.

<p style="margin-bottom: 20px;"></p>

$ 8 the steps for virtio driver to initialize a device  
8.1 reset the device.  
8.2 set the ACKNOWLEDGE status bit: the guest kernel has notice the device.  
8.3 set the DRIVER status bit: the guest kernel know how to drive the device.  
8.4 read device feature bits, and write the subset of feature bits understood by the kernel and
    driver to the device.
    during this step the driver MAY read (but MUST NOT write) the device-specific config fields
    to check that it can support the device before accepting it.  
8.5 set the FEATURES_OK status bit. the driver MUST NOT accept new feature bits after this step.  
8.6 re-read device status to ensure the FEATURES_OK bit is still set: otherwise, the dev does not support our
    subset of features and the dev is unusable.  
8.7 perform dev-specific setup, including discovery of virtqueues for the dev, optional per-bus setup,
    reading and possibly writing the dev’s virtio config space, and population of virtqueues.  
8.8 set the DRIVER_OK status bit. at this point the device is 'live'.

<hr>

### # virtio drivers: the software avatar
the virtio driver is the software in the virtual environment, used to talk with the virtio device through
relevant parts of the virtio spec.
generally speaking, the virtio driver tasks for the control plane are:
1 look for the device; 2 allocate shared memory in the guest for the communication.

<hr>

### # building blocks of virtio-based vm architecture
1 hypervisor: pool resources like: processor, memory, and storage, then reallocate them among guests.

2 kvm: allow the kernel to function as a hypervisor, so the host machine can run multiple isolated guests,
manage & support for creation of them.
kvm only provide device abstraction but no processor emulation, and it emulate very little hardware,
instead defer the dev emulation to higher level app like: qemu, crosvm, and firecracker.

3 qemu: a hosted virtual machine monitor (vmm), which provides a set of different hardware and
device models for the guest machine by emulation.
qemu can be used with kvm to run virtual machines at near native speed leveraging hardware extensions.

4 libvirt: an interface to translate xml-formatted config to qemu cli calls.
it also provides an admin daemon to configure child process such as qemu (no root privilege needed in this case).
openstack nova also use libvirt to spin up a qemu process for establishment of each vm.

<hr>

### # a few scenarios about the interaction between dev & drv
$ 1 qemu emulated device diagram

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
           │  │              ┌--------->│    │ │ avail ring[]     │ │    ├-------┐|     │  │
           │  │              |          │    │ ├──────────────────┤ │    │  (1)  ||     │  │
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
           │                            │    │ ├──────────────────┤ │    ├---┐|  ||        │
           │                            │    │ │ used ring[]      │ │    │(2)||  ||        │
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

1.1 the driver notifications are routed usually by pci to kvm irq, used to stop the guest instruction execution
and return the control to the host qemu process (by triggering vmexit event).  
1.2 the device notifications can be sent by qemu to kvm device using a special ioctl, then the kvm will arbitrate
the access to vcpu & memory, and generate vcpu irq accordingly.

<p style="margin-bottom: 20px;"></p>

$ 2 host kernel emulated virtio-net device diagram  
in scenario I, the notification need to travel from the guest kernel, to qemu, and then to the host kernel for
forwarding of the network frame (too long the control plane path).
thus, we can maintain a thread in host kernel, sharing memory with the guest kernel virtio drv to handle virtio
dataplane.

```txt
           ┌─QEMU PROCESS──────────────────────────────────────────────────────────────────┐
           │                                                                               │
           │  ┌─VM PROCESS──────────────────────────────────────────────────────────────┐  │
           │  │                                                                         │  │
           │  │                                                        guest user space │  │
           │  │ ─────────────────────────────────────────────────────────────────────── │  │
           │  │                                                      guest kernel space │  │
           │  │                 ┌─VIRTIO-NET DRIVER──────────────┐                      │  │
           │  │                 │    ┌──────────────────────┐    │                      │  │
           │  │   ┌------------>│    │ ┌──────────────────┐ │    │                      │  │
           │  │   |             │    │ │ descriptor table │ │    │                      │  │
           │  │   |             │    │ ├──────────────────┤ │    │<---------------┐     │  │
           │  │   |        ┌--->│    │ │ avail ring[]     │ │    ├---------------┐|     │  │
           │  │   |        |    │    │ ├──────────────────┤ │    │               ||     │  │
           │  │   |        |    │    │ │ used ring[]      │ │    │               ||     │  │
           │  │   |        |    │    │ └──────────────────┘ │    │               ||     │  │
           │  │   |        |    │    └────────+────+────────┘    │               ||     │  │
           │  │   |        |    └─────────────|────|─────────────┘               ||     │  │
           │  └───|────────|──────────────────|────|─────────────────────────────||─────┘  │
           │      |        |                  |    |                             ||        │
           │      |virtio control plane  virtio shared mem                 notifications   │
           │      |   pci protocol          (data path)   (2)                (eventfd) (3) │
           │      |  (control path)           |    |                             ||        │
           │      |        |          ┌───────|────|────────┐                    ||        │
           │      |        |          │  virtio-net|device  │                    ||        │
           │      |        └----------│       |model        │                    ||        │
           │ (data path)              └───────|────|────────┘                    ||        │
           │      |                           |    |   vhost-net setup           ||        │
           └──────|───────────────────────────|────|──────────+──────────────────||────────┘
                  |                           |    |    (control path)           ||             host user space
 ─────────────────|───────────────────────────|────|──────────|──────────────────||────────────────────────────
                  |                           |    |          |                  ||           host kernel space
                  |   ┌─VIRTIO-NET DEVICE─────|────|──────────+──┐               ||
                  |   │              ┌────────+────+────────┐    │     ┌─────────┼┼────────┐
                  |   │              │ ┌──────────────────┐ │<---│-----│---------┘|        │
                  |   │              │ │ descriptor table │ ├----│-----│----------┘        │
                  |   │              │ ├──────────────────┤ │    │     │        kvm        │
                  └-->│  vhost-net   │ │ avail ring[]     │ │    │     └───────────────────┘
                      │   process    │ ├──────────────────┤ │    │
                  (1) │              │ │ used ring[]      │ │    │
                      │              │ └──────────────────┘ │    │
                      │              └──────────────────────┘    │
                      └──────────────────────────────────────────┘
```

1.1 to increase performance, use in-kernel virtio-net device (vhost-net) to offload the data plane out of qemu
to the kernel, where packet forwarding takes place.  
1.2 qemu init the device through virtio dataplane (shared mem), then forward the virtio dev status
to vhost-net, and delegate the data plane to it.  
1.3 kvm will use the eventfd for dev irq communication, and expose another one to receive cpu irqs.
the guest kdrv does not need to be aware of this change, it will operate the same as scenario I.

<p style="margin-bottom: 20px;"></p>

$ 3 host userspace emulated virtio-user device diagram  
the virtio device in scenario II can be moved from the kernel to userspace on host machine, where the packet
forwarding framework like DPDK can be established. the protocol to set this up is virtio-user.

```txt
           ┌─QEMU PROCESS──────────────────────────────────────────────────────────────────┐
           │                                                                               │
           │  ┌─VM PROCESS──────────────────────────────────────────────────────────────┐  │
           │  │                                                                         │  │
           │  │                                                        guest user space │  │
           │  │ ─────────────────────────────────────────────────────────────────────── │  │
           │  │                                                      guest kernel space │  │
           │  │                 ┌─VIRTIO-NET DRIVER──────────────┐                      │  │
           │  │                 │    ┌──────────────────────┐    │                      │  │
           │  │   ┌------------>│    │ ┌──────────────────┐ │    │                      │  │
           │  │   |             │    │ │ descriptor table │ │    │                      │  │
           │  │   |             │    │ ├──────────────────┤ │    │<---------------┐     │  │
           │  │   |        ┌--->│    │ │ avail ring[]     │ │    ├-------------->┐|     │  │
           │  │   |        |    │    │ ├──────────────────┤ │    │               ||     │  │
           │  │   |        |    │    │ │ used ring[]      │ │    │               ||     │  │
           │  │   |        |    │    │ └──────────────────┘ │    │               ||     │  │
           │  │   |        |    │    └────────+────+────────┘    │               ||     │  │
           │  │   |        |    └─────────────|────|─────────────┘               ||     │  │
           │  └───|────────|──────────────────|────|─────────────────────────────||─────┘  │
           │      |        |                  |    |                             ||        │
           │      |virtio control plane  virtio shared mem                 notifications   │
           │      |   pci protocol          (data path)                      (eventfd)     │
           │      |  (control path)           |    |                             ||        │
           │      |        |          ┌───────|────|────────┐                    ||        │
           │      |        |          │  virtio-net|device  │                    ||        │
           │      |        └--------->│       |model (?)    │                    ||        │
           │ (data path)              └───────|────|────────┘                    ||        │
           │      |                           |    |   vhost-net setup           ||        │
           └──────|───────────────────────────|────|──────────+──────────────────||────────┘
                  |                           |    |    vhost | uds              ||
                  |                           |    |   (control path)            ||
                  |   ┌─VIRTIO-NET DEVICE─────|────|──────────+──┐               ||
                  |   │              ┌────────+────+────────┐    │               ||
                  |   │              │ ┌──────────────────┐ │    │               ||
                  |   │              │ │ descriptor table │ │    │               ||
                  |   │              │ ├──────────────────┤ │    │               ||
                  └-->│  vhost-user  │ │ avail ring[]     │ │    │               ||
                      │   process    │ ├──────────────────┤ │    │               ||
                      │              │ │ used ring[]      │ │    │               ||
                      │              │ └──────┬────+──────┘ │    │               ||
                      │              └────────|────|────────┘    │               ||
                      └───────────────────────|────|─────────────┘               ||
                                              |    |                             ||             host user space
 ─────────────────────────────────────────────|────|─────────────────────────────||────────────────────────────
                                              |    |                   ┌─────────┼┼────────┐  host kernel space
                                              |    └-------------------│<--------┘|        │
                                              └----------------------->│----------┘        │
                                                                       │        kvm        │
                                                                       └───────────────────┘
```

<p style="margin-bottom: 20px;"></p>

$ 4 virtio-user device with userland virtio driver in guest kernel diagram  
it even allows guest to run virtio drivers in guest’s userland, instead of the kernel, in this case, virtio-net
driver is the process that managing the memory and the vring.

```txt
           ┌─QEMU PROCESS──────────────────────────────────────────────────────────────────┐
           │                                                                               │
           │  ┌─VM PROCESS──────────────────────────────────────────────────────────────┐  │
           │  │                                                                         │  │
           │  │                 ┌─VIRTIO-NET DRIVER──────────────┐                      │  │
           │  │                 │    ┌──────────────────────┐    │                      │  │
           │  │   ┌------------>│    │ ┌──────────────────┐ │    │                      │  │
           │  │   |             │    │ │ descriptor table │ │    │                      │  │
           │  │   |             │    │ ├──────────────────┤ │    │<---------------┐     │  │
           │  │   |        ┌--->│    │ │ avail ring[]     │ │    ├-------------->┐|     │  │
           │  │   |        |    │    │ ├──────────────────┤ │    │               ||     │  │
           │  │   |        |    │    │ │ used ring[]      │ │    │               ||     │  │
           │  │   |        |    │    │ └──────────────────┘ │    │               ||     │  │
           │  │   |        |    │    └────────+────+────────┘    │               ||     │  │
           │  │   |        |    └─────────────|────|─────────────┘               ||     │  │
           │  │   |        |                  |    |                   guest user|space │  │
           │  │ ──|────────|──────────────────|────|─────────────────────────────||──── │  │
           │  │   |        |                  |    |                 guest kernel|space │  │
           │  └───|────────|──────────────────|────|─────────────────────────────||─────┘  │
           │      |        |                  |    |                             ||        │
           │      |virtio control plane  virtio shared mem                 notifications   │
           │      |   pci protocol         (data path)                       (eventfd)     │
           │      |  (control path)           |    |                             ||        │
           │      |        |       ┌──────────|────|─────────┐                   ||        │
           │      |        └------>│ virtio-net device model │                   ||        │
           │ (data path)           └──────────|────|─────────┘                   ||        │
           │      |                           |    |   vhost-net setup           ||        │
           └──────|───────────────────────────|────|──────────+──────────────────||────────┘
                  |                           |    |    vhost | uds              ||
                  |                           |    |   (control path)            ||
                  |   ┌─VIRTIO-NET DEVICE─────|────|──────────+──┐               ||
                  |   │              ┌────────+────+────────┐    │               ||
                  |   │              │ ┌──────────────────┐ │    │               ||
                  |   │              │ │ descriptor table │ │    │               ||
                  |   │              │ ├──────────────────┤ │    │               ||
                  └-->│  vhost-user  │ │ avail ring[]     │ │    │               ||
                      │   process    │ ├──────────────────┤ │    │               ||
                      │              │ │ used ring[]      │ │    │               ||
                      │              │ └──────┬────+──────┘ │    │               ||
                      │              └────────|────|────────┘    │               ||
                      └───────────────────────|────|─────────────┘               ||
                                              |    |                             ||             host user space
 ─────────────────────────────────────────────|────|─────────────────────────────||────────────────────────────
                                              |    |                   ┌─────────┼┼────────┐  host kernel space
                                              |    └-------------------│<--------┘|        │
                                              └----------------------->│----------┘        │
                                                                       │        kvm        │
                                                                       └───────────────────┘
```

<p style="margin-bottom: 20px;"></p>

$ 5 virtio real hardware device passthrough diagram  
finally, the virtio device support passthrough with a proper hardware (support virtio data plane).

the proper physical nic device can be exposed directly to guest kernel with certain hardware
(iommu: translate the mem address between guest and device) and software (e.g., vfio linux driver:
enable the host kernel to pass the control of a pci dev to guest kernel) provided.
thus, the virtio device will use true hardware signals for notifications to infra, like pci and cpu irq.

if a hardware nic choose to go this way, the easiest approach is to build its driver on top of virtio data-path
acceleration framework (vdpa).

```txt
           ┌─QEMU PROCESS──────────────────────────────────────────────────────────────────┐
           │                                                                               │
           │  ┌─VM PROCESS──────────────────────────────────────────────────────────────┐  │
           │  │                                                                         │  │
           │  │                                                        guest user space │  │
           │  │ ─────────────────────────────────────────────────────────────────────── │  │
           │  │                                                      guest kernel space │  │
           │  │                 ┌─VIRTIO-NET DRIVER──────────────┐                      │  │
           │  │                 │    ┌──────────────────────┐    │                      │  │
           │  │                 │    │ ┌──────────────────┐ │    │                      │  │
           │  │                 │    │ │ descriptor table │ │    │                      │  │
           │  │                 │    │ ├──────────────────┤ │    │<---------------┐     │  │
           │  │            ┌--->│    │ │ avail ring[]     │ │    ├---------------┐|     │  │
           │  │            |    │    │ ├──────────────────┤ │    │               ||     │  │
           │  │            |    │    │ │ used ring[]      │ │    │               ||     │  │
           │  │            |    │    │ └──────────────────┘ │    │               ||     │  │
           │  │            |    │    └────────+────+────────┘    │               ||     │  │
           │  │            |    └─────────────|────|─────────────┘               ||     │  │
           │  └────────────|──────────────────|────|─────────────────────────────||─────┘  │
           │               |                  |    |                             ||        │
           │      virtio control plane   virtio shared mem                 notifications   │
           │          pci protocol          (data path)                      (eventfd)     │
           │         (control path)           |    |                             ||        │
           │               |       ┌──────────|────|─────────┐                   ||        │
           │               └------>│ virtio-net device model │                   ||        │
           │                       └──────────|────|─────────┘                   ||        │
           │                                  |    |                             ||        │
           └──────────────────────────────────|────|──────────+──────────────────||────────┘
                                              |    |     vDPA | setup            ||             host user space
 ─────────────────────────────────────────────|────|──────────|──────────────────||────────────────────────────
                                              |    |(pass dev |cntl to guest)    ||           host kernel space
                                              |    |  ┌───────+────────┐   ┌─────||─────┐
                                              |    |  │      VFIO      │   │     ||     │
                                              |    |  └───+─────────+──┘   │    kvm     │
                                              |    |      |         |      │     ||     │
                                              |    |  ┌───+───┐     |      └─────||─────┘
                                              |    |  │ IOMMU │     |            ||
                                              |    |  └───+───┘     |            ||
                                              |    |    (control path)           ||
                                              |    |      |         |            ||           host kernel space
 ─────────────────────────────────────────────|────|──────|─────────|────────────||────────────────────────────
                                           (data path)    |         |            ||             hardware blocks
                                         ┌────+────+───┐  |         |            ||
                      (addr translation) │(1) IOMMU|   +--┘         |            ||
                                         └────+────+───┘            |            ||
                                              |    |                |            ||
                      ┌─VIRTIO-NET HARDWARE───|────|─────────────┐  |            ||
                      │              ┌────────+────+────────┐    │  |            ||
                      │              │ ┌──────────────────┐ │    │  |            ||
                      │              │ │ descriptor table │ ├----│--┘            ||
                      │  virtio-net  │ ├──────────────────┤ │    │               ||
                      │   hardware   │ │ avail ring[]     │ │    │               ||
                      │  (physical)  │ ├──────────────────┤ │<---│---------------┘|
                      │              │ │ used ring[]      │ ├----│----------------┘
                      │              │ └──────────────────┘ │    │
                      │              └──────────────────────┘    │
                      └──────────────────────────────────────────┘
```

1.1 iommu: manage the mem mapping between linux dev address & physical addresses; handling DMA request from
linux dev to corresponding physical using the established mapppings; raise page-fault or do error handling
when linux dev try to access unauthorized mem.

<hr>

### # reference
https://docs.oasis-open.org/virtio/virtio/v1.2/cs01/virtio-v1.2-cs01.pdf
