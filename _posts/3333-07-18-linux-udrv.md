---
layout: post
title: "udrv: basic concepts & design pattern (linux, udrv)"
author: "melon"
date: 2024-07-17 22:00
categories: "2024"
tags:
  - linux
  - todo
---

a driver is software that directly controls a particular device attached to a computer.
os segregate the system's virtual memory into two kinds of addresses, based on privilege level:
kernel space and user space.
the memory separation is aided by features on the cpu called protection rings.

typically, drivers run in kernel space, but there existed drivers in userspace as the
direct interface of the hardware device, which is named as udrv.

<hr>

### # requirements for udrv to work: uio & vfio
for udrv to control a device, it's a necessary to let kernel relinquish control of device:  
1) unbind kdrv from a device by a file in sysfs.  
2) rebind kdrv to some special device drivers already bundled with linux: uio or vfio.

the uio/vfio are two dummy drivers in the sense that they are indicates to kernel that
the device already has a driver bound to it, so the kernel wont try to rebind default drv.
they wont actually init the hw in any way, nor do they understand what type the device is.

what's the difference between uio and vfio?  
vfio is capable of programming the platform's iommu, a critical piece of hardware for ensuring
memory safety in user space drivers. see userspace dma for details.

what happens when unbind dev from kernel?  
1) once the device is unbound, the kernel can't use it anymore.
i.e, if you unbind an NVMe device from linux, the devices corresponding to it such as
/dev/nvme0n1 will disappear.  
2) it means that filesystems mounted on the dev will also be removed and kernel
filesystems can no longer interact with the dev.
in fact the entire kernel block storage stack is no longer involved.

<hr>

### # why introduce udrv? the benefit?
before udrv is introduced, the platform software code is orgranized as follows:

```txt
                                          ┌──────────────┐
┌---------------------------┐             │   board-a    │
|                           |             │ ┌──────────┐ │
|  ./software_source_code   |             │ │ xxxx-app │ │
|  │                        |             │ └──────────┘ │
|  ├── product_common_code  |             │ ┌──────────┐ │
|  │   └── drivers -----------┬--------+--->│ driver-a │ │
|  │       ├── file_1.c     | |        |  │ └──────────┘ │
|  │       └── file_2.c     | |        |  └──────────────┘
|  │                        | |        |                  
|  ├── board_a              | |        |  ┌──────────────┐
|  │   └── drivers --------------------┘  │   board-b    │
|  │       ├── file_1.c     | |           │ ┌──────────┐ │
|  │       └── file_2.c     | |           │ │ xxxx-app │ │
|  │                        | |           │ └──────────┘ │
|  └── board_b              | └--------┐  │ ┌──────────┐ │
|      └── drivers --------------------+--->│ driver-b │ │
|          ├── file_c1.c    |             │ └──────────┘ │
|          └── file_c2.c    |             └──────────────┘
|                           |
└---------------------------┘
```

which typically has several drawbacks:  
1) software code base is huge, with lots of duplications and hard to be reused.  
2) when new product board hardware is on schedual, there will exist too much duplicate works
for code migration & adaption, which make its hard to debug & maintainence.

hence, there's strong need to simplify the above project organization, the main idea is:  
1) create a new device model to cover legacy & incoming product hardware.  
2) make the device model data driven, and the code can automatically adapt to the board difference data input.  
3) the code base structure should be designed with extreme reusable, with min changes needed for new board intake.  
4) the abstraction need to provide unified api interface to platform-level application for hardware abstraction.

<hr>

### # design flow among different product stage
the wholesome board platform architecture design has 3 stages, with more and more powerful chips integrated,
more and more functionalities supported, the platform software has evolved into more layers of abstraction
levels:

```txt
                                        ┌──────────────────────────┐           ┌──────────────────────────┐
                                        │   equipment logic apps   │           │   equipment logic apps   │
                                        └──────────────────────────┘           └──────────────────────────┘
                                        ┌──────────────────────────┐           ┌──────────────────────────┐
                                        │   hardware access apps   │           │   hardware access apps   │
 ┌──────────────────────────┐           └──────────────────────────┘           └──────────────────────────┘
 │       product-apps       │           ┌──────────────────────────┐           ┌──────────────────────────┐ --+
 └──────────────────────────┘           │                          │           │     platform service     │   |
 ┌──────────────────────────┐           │     userspace driver     │           ├--------------------------┤   |
 │     userspace driver     │           │                          │           │     userspace driver     │   |
 └──────────────────────────┘           └──────────────────────────┘           └──────────────────────────┘   | platform
------------------------------  ────>  ------------------------------  ────>  ------------------------------  |  domain
 ┌──────────────────────────┐           ┌──────────────────────────┐           ┌──────────────────────────┐   |
 │    kernelspace driver    │           │    kernelspace driver    │           │    kernelspace driver    │   |
 └──────────────────────────┘           └──────────────────────────┘           └──────────────────────────┘ --+
 ┌──────────────────────────┐           ┌──────────────────────────┐           ┌──────────────────────────┐
 │         hardware         │           │         hardware         │           │         hardware         │
 └──────────────────────────┘           └──────────────────────────┘           └──────────────────────────┘
      legacy architecture                middle state architecture              current architecture design
```


<hr>


### # udrv architecture: the detailed design
the overview architecture of udrv is as follows, roughly it can be seen as a proxy to delegate the
requests from low level hardware access apps to access the physical hardware resources.

with udrv integrated, the apps developers could focus on the more about bussiness logic rather than
conduct meaningless code changes to just follow the pace of system/platform-level evolutions & bug fixes,
hence, the udrv will try make the service interface stable and adapt to low-level changes for different
hw-board-type.

```txt
┌───────────────────────────────────────────────────────────┐
│                product userspace application              │
└───────────────────────────────────────────────────────────┘
                              |
┌─────────────────────────────┼────────────────────────────────────────────────────────┐
│  udrv                       +                                                        │
│ ┌─────────────────────────────────────────────────────────┐     ┌───────┐  ┌───────┐ │
│ │                device service interfaces                │<----│       │--│       │ │
│ └─────────────────────────────────────────────────────────┘     │       │  │       │ │
│ ┌─────────────────────────────────────────────────────────┐     │       │  │       │ │
│ │                    interface mapping                    │     │       │  │       │ │
│ └─────────────────────────────────────────────────────────┘     │       │  │       │ │
│ ┌─────────────────────────────────────────────────────────┐     │       │  │       │ │
│ │                device lowlevel interface                │<----│       │--│       │ │
│ └─────────────────────────────────────────────────────────┘     │       │  │       │ │
│           │                             |                       │ tests │  │ trace │ │
│           │                             +                       │       │  │ debug │ │
│           │                ┌───────────┐ ┌────────────┐         │       │  │       │ │
│           +                │┌───────────┐│┌────────────┐        │       │  │       │ │
│ ┌────────────────────┐     ││┌───────────┐│┌────────────┐       │       │  │       │ │
│ │ device tree parser │     │││           │└│   driver   │       │       │  │       │ │
│ │       (json)       │     │││    bus    │┌└────────────┘       │       │  │       │ │
│ └────────────────────┘     └││           ││┌────────────┐       │       │  │       │ │
│                             └│           │└│   device   │       │       │  │       │ │
│                              └───────────┘ └────────────┘       └───────┘  └───────┘ │
│ ┌──────────────────────────────────────────────────────────────────────────────────┐ │
│ │                      operating system adaption layer (OSAL)                      │ │
│ └──────────────────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

the kernel abstraction of udrv is bus-drv-dev model inside, which is buildup with the information from
user customized driver, device info json file; the core model is initialized in order of bus->drv->dev
(according to the abstraction level from high to low).

after the core model builtup, any incoming request from hardware access apps could then delegated by udrv
through the system adaption level towards the kdrv exposed interfaces.

it's attractive to inherit the kernel device model into udrv, because it's classic design & a clean abstraction
for hw resources.

<hr>

### # device driver bus triplet model
linked relationship happens between bus & dev, bus & drv, dev & dev; while the matched relationship only happens
between dev & drv.

```txt
┌───────┐
│       │ link ┌────────┐      ┌────────┐      ┌────────┐
│       *------* device *------* device *------* device │      linked:  *--*
│       │      └────+───┘      └────+───┘      └──+─────┘      matched: +--+
│  bus  │           |               | ┌-----------┘
│       │ link ┌────+───┐      ┌────+─+─┐
│       *------* driver *------* driver │
│       │      └────────┘      └────────┘
└───*───┘
    |
    |   ┌───────┐
    |   │       │ link ┌────────┐      ┌────────┐      ┌────────┐
    |   │       *------* device *------* device *------* device │
    |   │       │      └─────+──┘      └──+─────┘      └──+─────┘
    └---*  bus  │           ┌|------------┘               |
        │       │           |└------------┐   ┌-----------┘
        │       │ link ┌────+───┐      ┌──+───+─┐
        │       *------* driver *------* driver │
        └───*───┘      └────────┘      └────────┘
            |
            |   ┌───────┐      struct list device
            |   │       │ link ┌────────┐      ┌────────┐
            |   │       *------* device *------* device │
            |   │       │      └────+───┘      └────+───┘
            └---*  bus  │           └-------------┐ |
                │       │ link ┌────────┐      ┌──+─+───┐
                │       *------* driver *------* driver │
                │       │      └────────┘      └────────┘
                └───────┘      struct list driver
```

<hr>

### # two-layered device management model: the controller device and the servicing device
✓两级设备管理模型
✓ 第一级，作为控制器级别，用于管理第二级设备，以及提供通用的操作
✓ 第二级，真正的提供服务的设备
✓不进行更深层级联设备，解除理解障碍
✓设备扁平化需要内核驱动(Kdrv)支持

taking udrv/tests/json/udrv.json as case:

```text
 "intrc": {
     "compatible": "udrv interrupt controller",                             // layer#1: controller

     "intr_ntio_presence": {
         "compatible": "udrv uio interrupt device",                         // layer#2: irq uio device
         "uioname": "uio_pipe_ntio_presence",
         "auto-enable": false,
         "regs": {
             "storm": { "reg": "intr_storm.storm", "privatedata": "0x1" }
         }
     },

     "intr_sfp0_presence": {                                                // layer#2: irq uio device
         "compatible": "udrv uio interrupt device",
         "uioname": "uio_pipe_sfp0_presence"
     },

     "intr_sfp0_rx_los": {
         "compatible": "udrv uio interrupt device",
         "uioname": "uio_pipe_sfp0_rx_los"
     },

     "intr_sfp0_tx_fault": {
         "compatible": "udrv uio interrupt device",
         "uioname": "uio_pipe_sfp0_tx_fault"
     },

     "intr_alarm0": {
         "compatible": "udrv uio interrupt device",
         "uioname": "uio_pipe_alarm0"
     },

     "intr_alarm1": {
         "compatible": "udrv inotify device",
         "pathname": "vroot/inotify/inotify_alarm1",
         "events": ["IN_CLOSE_WRITE"],
         "regs": {
             "storm": { "reg": "intr_storm.storm", "privatedata": "0x1" }
         }
     },

     "intr_inotify_test": {
         "compatible": "udrv inotify device",
         "pathname": "vroot/inotify/inotify_test",
         "auto-enable": false
     }
 },
```

which shows the intrc as the abstract controller device: udrv interrupt controller, and the sub uio interrupt devices.

<hr>

### # flat device model
a sample graphic illustration of layered platform model for a i2c-cpld-dev_xxx device:

```txt
      ┌----------------------------------------------------------------------┐
      |                                udrv                                  |
      └----------------------------------------------------------------------┘
                                        |                           |
                                        |                           |
                                        +                           +
                               /dev/i2c-x,addr_of_cpld        /dev/spi_xxxx      user space (flat)
						       /dev/miscdev_xxx
-------------------------------------------------------------------------------------------------------
                                  ┌────────────┐-----------------------------┐
                                  │ spi-master │            |   spi-device   |
      ┌---------------------------┼------------│-----------------------------┘  kernel space (nested)
      │ i2c-adaptor               │ i2c-client │
      └---------------------------┴────────────┘
-------------------------------------------------------------------------------------------------------
      ┌───────────────┐          ┌──────────────┐           ┌────────────────┐
      │      soc      *----------*     cpld     *-----------*     device     │   hardware (linked)
      └───────────────┘          └──────────────┘           └────────────────┘
```

flat device model can enable flat interrupt handling design based on udrv abstraction:

```txt
            interrupt-1                             interrupt-1
                |                         interrupt-0    |     interrupt-2
interrupt-0 ----┼---- interrupt-2              |         |          |
                |                              |         |          |       application xxx
       check cpld registers                    |         |          |
                |                              |         |          |
----------------|------------------------------|---------|----------|----------------------------
                |                              |         |          |
            /dev/uio0                     /dev/uio0  /dev/uio1  /dev/uio2   user space
                |                              |         |          |
----------------|------------------------------|---------|----------|-----------------------------
        ┌───────+───────┐                  ┌───+─────────+──────────+──┐
        │  cpld driver  │                  │        cpld driver        │    kernel space
        └───────────────┘                  └───────────────────────────┘
        nested interrupts                    flat interrupt for device
```

<hr>

### # management of hot-plugable devices with udrv
NTIO & NTIO sfp


<hr>

### # integration udrv package in buildroot
1 udrv package dir: /buildroot/package/Nokia/udrv/

```text
.
├── Config.in
└── udrv.mk
```

for udrv feature enablement, inside config, select BR2_PACKAGE_UDRV option.
the udrv software is released & used as a dynamic linked libs: libudrv.so, and can be included
in the compilation stage of the applications on demand.

2 Config.in

```text
config BR2_PACKAGE_UDRV
	bool "udrv"
	depends on BR2_PACKAGE_JANSSON
	help
	  Udrv is a common user sapce driver library, it manages the
	  board devices configured in json file(s), then provides
	  generic interfaces to application. An application does not to
	  know the device details.

	  This package builds the libudrv library.

if BR2_PACKAGE_UDRV
config BR2_PACKAGE_UDRV_FEATURE_CHRDEV_GPIO_DRIVER
	bool "udrv: GPIO driver based on character device interface"
	help
	  After all ARCHs Linux kernel is upgraded to a new version that
	  GPIO supports character device interfaces, this option can be
	  safely removed.

config BR2_PACKAGE_UDRV_FEATURE_PSEUDO_SPI_FULL_DUPLEX
	bool "udrv: pseudo spi full duplex transfer"
	help
	  For kernel 5.4, you have to send a spi full duplex transfer in
	  a two length spi_ioc_transfer. See kernel source spidev_fdx.c.

config BR2_PACKAGE_UDRV_SHELL
	bool "install udrv_shell"
	help
	  Additionally install udrv_shell

endif
```

3 udrv.mk

```text
################################################################################
#
# udrv
#
################################################################################

UDRV_VERSION = 0919ed523ced3a7a774c78db201bfaa0a4a5e1fb
UDRV_SITE = ssh://$(ISAM_HG_SERVER)/all/reborn/udrv
UDRV_SITE_METHOD = hg
UDRV_INSTALL_STAGING = YES
UDRV_LICENSE = PROPRIETARY

UDRV_DEPENDENCIES += jansson libuio
HOST_UDRV_DEPENDENCIES += host-jansson

UDRV_FEATURES += UDRV_FEATURE_CHRDEV_GPIO_DRIVER=$(BR2_PACKAGE_UDRV_FEATURE_CHRDEV_GPIO_DRIVER)
UDRV_FEATURES += UDRV_FEATURE_PSEUDO_SPI_FULL_DUPLEX=$(BR2_PACKAGE_UDRV_FEATURE_PSEUDO_SPI_FULL_DUPLEX)

UDRV_ADDITIONAL_INSTALL_TARGETS-$(BR2_PACKAGE_UDRV_SHELL) = install_shell

$(call ISAM_LINUX_APP,BR2_PACKAGE_ISAM_LINUX_APPS_NSTP,nstp)
define UDRV_BUILD_CMDS
	$(MAKE) $(TARGET_CONFIGURE_OPTS) $(UDRV_FEATURES) -C $(@D)
endef

define UDRV_INSTALL_TARGET_CMDS
	$(MAKE) $(TARGET_CONFIGURE_OPTS) $(UDRV_FEATURES) DESTDIR=$(TARGET_DIR) -C $(@D) install $(UDRV_ADDITIONAL_INSTALL_TARGETS-y)
endef

define UDRV_INSTALL_STAGING_CMDS
	$(MAKE) $(TARGET_CONFIGURE_OPTS) $(UDRV_FEATURES) DESTDIR=$(STAGING_DIR) -C $(@D) install
endef

define HOST_UDRV_BUILD_CMDS
	$(HOST_MAKE_ENV) $(MAKE) $(HOST_CONFIGURE_OPTS) $(UDRV_FEATURES) -C $(@D)
endef

define HOST_UDRV_INSTALL_CMDS
	$(HOST_MAKE_ENV) $(MAKE) $(HOST_CONFIGURE_OPTS) $(UDRV_FEATURES) DESTDIR=$(HOST_DIR) -C $(@D) install
endef

$(eval $(generic-package))
$(eval $(host-generic-package))
```

<hr>


### # compilation relationship of udrv & product apps
the data-drvien udrv (dynamic linked lib) is compiled and used in board platform service.

```text
┌---------------------------┐        ┌──────────────┐
|                           |        │   board-c    │
|  ./software_source_code   |        │ ┌──────────┐ │
|  │ ┌-------------------┐  |        │ │ xxxx-app │ │
|  ├─┼ board_a           |  |        │ └──────────┘ │
|  │ | └── jsons         |  |        │ ┌──────────┐ │
|  │ |     ├── udrv.json |  |        | │   udrv   │ │
|  │ |     └── aaa.json  |  |        │ └─────+────┘ │
|  │ └-------------------┘  |        └───────|──────┘
|  ├── board_b              |                |
|  │   └── jsons            |                |
|  │       ├── udrv.json    |     ┌----------┴-------┐
|  │       └── bbb.json     |     |                  |
|  │ ┌-------------------┐  |     |  ┌───────────────|─────┐
|  └─┼ board_c           |  |     |  │ ./buildroot   |     |
|    | └── jsons         ├--------┘  │ | ┌-----------┴---┐ │
|    |     ├── udrv.json |  |        │ ├─┼ udrv          | │
|    |     └── ccc.json  |  |        │ │ | └── libudrv.so| |
|    └-------------------┘  |        │ │ └---------------┘ |
└---------------------------┘        └─────────────────────┘
```


 14 Upper left corner ┌
 15 Upper right corner ┐
 16 Lower left corner └
 17 Lower right corner ┘
 18 Tee pointing right ├
 19 Tee pointing left ┤
 20 Tee pointing up ┴
 21 Tee pointing down ┬
 22 Horizontal line ─
 23 Vertical line │
 24 Large Plus or cross over ┼
 25 Scan Line 1 ⎺
 26 Scan Line 3 ⎻
