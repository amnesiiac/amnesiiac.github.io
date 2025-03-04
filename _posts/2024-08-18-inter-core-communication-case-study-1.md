---
layout: post
title: "inter-core communication on amp arch I (overview) (rust, rpc, amp)"
author: "melon"
date: 1111-08-18 21:51
categories: "2024"
tags:
  - virtualization
  - driver
  - icc
  - nokia
---

this article introduce a rust app used in inter-core communication scenarios between a smp linux os
and single core vsworks os.

the inter-core communication mechanism is enabled only based on simulation builds of them, and is designed for
mimicing the inter-core interrupt happen between hardware boards.

<hr>

### # background: how does core 0 (nt board) co-work with core 1 (ihub) on real world
1 overview of asymmetric multiprocessor systems (amp) architecture: smp linux + vxworks:

```txt
                  (core 0/2/3)                                       |                      (core 1)
┌─────────────────────────────────────────────┐                      .  ┌─────────────────────────────────────────────┐
│ ┌-----------------------------------------┐ │                      |  │ ┌-----------------------------------------┐ │
│ |               SHELF-NE APP              | │                      .  │ |                IHUB-NE APP              | │
│ |                                         | │                      |  │ |                                         | │
│ | ┌────────────┐                          | │                      .  │ | ┌────────────┐                          | │
│ | │ redun-cntl │<-------------------------|-│-------------------------│-|>│ redun-cntl │                          | │
│ | └─────+──────┘       .           .      | │                      |  │ | └─────+──────┘       .           .      | │
│ └-------┼--------------┼-----------┼------┘ │                      .  │ └-------┼--------------┼-----------┼------┘ │
│ ┌-------┼--------------┼-----------┼------┐ │                      |  │ ┌-------┼--------------┼-----------┼------┐ │
│ | ┌─────+─────┐        │           │      | │                      .  │ | ┌─────+──────┐       │           │      | │
│ | │ redun-hwa │        │           │      | │  access hw controled |  │ | │ switch-hwa │       │           │      | │
│ | └─────+─────┘  ┌─────+────┐      │      | │     by other core    .  │ | └─────+──────┘ ┌─────+────┐      │      | │
│ |       │        │ eqpt-hwa │<-----│------|-│-------------------------│-|-------│------->│ eqpt-hwa │      │      | │
│ |       │        └─────+────┘  ┌───+────┐ | │                      |  │ |       │        └─────+────┘  ┌───+────┐ | │
│ |       │              │       │ fs-hwa │<|-│-------------------------│-|-------│--------------│------>│ fs-hwa │ | │
│ |       │              │       └───+────┘ | │  fs access should go |  │ |       │              │       └───+────┘ | │
│ |       │        SHELF-NE HWA      │      | │    through shelf-ne  .  │ |       │        SHELF-NE HWA      │      | │
│ └-------┼--------------┼-----------┼------┘ │                      |  │ └-------┼--------------┼-----------┼------┘ │
└─────────┼──────────────┼───────────┼────────┘                      .  └─────────┼──────────────┼───────────┼────────┘
----------┼--------------┼-----------┼--------------------------------------------┼--------------┼-----------┼----------
┌─────────+──────────────+───────────+────────┐   init core 1 boot   |  ┌─────────+──────────────+───────────+────────┐
│              PLATFORM HWA (linux)           │<----------------------->│              PLATFORM HWA (vxworks)         │
└─────────────────────────────────────────────┘     flash access     |  └─────────────────────────────────────────────┘
------------------------------------------------------------------------------------------------------------------------
┌──────────┐      ┌─────────┐       ┌─────────┐                      |                    ┌───────────┐
│ redun-hw │      │ cf-card │       │ eqpt-hw │                      |                    │ switch-hw │
└──────────┘      └─────────┘       └─────────┘                      .                    └───────────┘
```

<p style="margin-bottom: 20px;"></p>

2 ihub-ne bootup sequence on target:

```txt

                (core 0/2/3)                                                                     (core 1)
┌────────────────────────────────────────────┐                                     ┌───────────────────────────────────┐
│                                            │                                     │ ┌-------------------------------┐ │
│ ┌────────────────────────────────────────┐ │                                     │ |            vxworks            | │
│ │             file system hwa            │ │                                     │ |                               | │
│ └─────────+──────────────────+───────────┘ │                                     │ |                               | │
│           │                  │             │                                     │ |                               | │
│ ┌---------┼------------------┼-----------┐ │    ┌───────────────────────────┐    │ |                               | │
│ |         │      linux       │           | │    │            RAM            │    │ |                               | │
│ | ┌───────+─────┐ ┌──────────+─────────┐ | │    │ ┌───────────────────────┐ │    │ |                               | │
│ | │ file system │ │                    │ | │ $1 │ │ load ihub uboot       │ │ $2 │ | ┌────────────┐$3┌───────────┐ | │
│ | └───────+─────┘ │ ihub boot assistor ├-|-│----│-+ load ihub             ├-│----│-|-+ ihub uboot ├--+ ihub apps │ | │
│ |         │       │        (*)         │ | │    │ │ decompressed sw image │ │    │ | └────────────┘  └───────────┘ | │
│ |         │       └────────────────────┘ | │    │ └───────────────────────┘ │    │ |                               | │
│ └---------┼------------------------------┘ │    └───────────────────────────┘    │ └-------------------------------┘ │
└───────────┼────────────────────────────────┘                                     └───────────────────────────────────┘   
            │                          
      ┌─────+──────────────────────────┐      *: executed in core 0 linux userspace
      │         compact flash          │      *: executed after linux platform services are available
      └────────────────────────────────┘      *: use filesystem hwa to access the cf card to load ihub sw image

$1: core 0 load ihub uboot to RAM, core 0 load decompressed sw image to RAM, then startup core 1 by the uboot.  
$2: core 1 ihub uboot start execution, do the basic initialization.  
$3: ihub uboot execute the first instruction to startup sw application.
```

notes:  
a) core 0 is the master core, which loads dedicated boot-code to ram for core 1 and release it.  
b) in core 0 kernel space, there's a remoteproc driver framework responsible for eloading firmware,
booting and managing these remote processors (core 1 ihub).  
c) in core 0 user space, the ihub boot assistor app is used to manage ihub bootup by sysfs provided
by remoteproc driver.

<p style="margin-bottom: 20px;"></p>

3 hardware resource ownership overview:

```txt
───────────────────────────────────────────────────────────────────────────────────────────────────────────────────
  soc resource  device            role of core 0/2/3    role of core 1   share mode
───────────────────────────────────────────────────────────────────────────────────────────────────────────────────
                cf card           part owner            part owner       $1 each side has a dedicated area for it,
                                                                         mutex needed for any access to cf.
                nor flash         owner                 -                $2 owned & managed by core 0/2/3, no
  local bus                                                              direct access from core 1.
                cpld              owner                 -                $2 ...
                                  
                ddr               part owner            part owner       $3 each side has a dedicated area for it,
                                                                         mutex needed for only simultaneous access
───────────────────────────────────────────────────────────────────────────────────────────────────────────────────
  mpic          msg interrupt     part owner            part owner       $3 ...
───────────────────────────────────────────────────────────────────────────────────────────────────────────────────
                uart port         part owner            part owner       $3 ...
  uart          uart interrupt    owner                 -                all uart irq are managed by core 0/2/3,
                                                                         notify core 1 if needed.
───────────────────────────────────────────────────────────────────────────────────────────────────────────────────
  pci-e         switch            -                     owner            data-plane
───────────────────────────────────────────────────────────────────────────────────────────────────────────────────
                RI                owner                 -                $4 owned & managed by core 0/2/3, and core
                ACS9510           owner                 -                0/2/3 provide hwa access proxy service for
  spi           DS26503           owner                 -                core 1 to access resources here.
                IDT82V3285        owner                 -
                TS x 3            owner                 -
───────────────────────────────────────────────────────────────────────────────────────────────────────────────────
  xaui          BCM56843          owner                 -                $4 ...
───────────────────────────────────────────────────────────────────────────────────────────────────────────────────
  tsec          for other nt      owner                 -                $4 ...
───────────────────────────────────────────────────────────────────────────────────────────────────────────────────
                sfp x 4           owner                 -                $4 ...
  i2c           to ntio           owner                 -
                alarm             owner                 -
───────────────────────────────────────────────────────────────────────────────────────────────────────────────────
```

notes:  
a) all the board peripherals is initialized at core 0/2/3 under a common board init module:
i2c, hw watchdog, duart, cpld, ram, compact flash.  
b) ihub should init the bcm switch and peripherals controlled exclusively by core 1.  
c) ihub need handle its file system requirements through core-0/2/3 (remote access).  
d) core 0/2/3 is the master for all the board peripherals that shared across cores.  
e) equipment hardware abstraction proxy is a slave module running in core 1 to access the shared peripherals.

<p style="margin-bottom: 20px;"></p>

4 inco-proxy application to enable inter-core communication between core 0 & core 1:  
virtio/rpmsg is chosen as the inter-core communication mechanism between core 0 & core 1.
rpmsg is a virtio-based messaging bus that allows kernel drivers to communicate with remote processors
available on the system.
in turn, drivers could then expose appropriate user space interfaces if needed.

```txt
                                                                                   (core 1)
                                                                       ┌────────────────────────────────┐
                                                                       │ ┌─────equipment hwa proxy────┐ │
                                                                       │ │   ┌────────────────────┐   │ │
                                                                       │ │   │      ion api       │   │ │
               (core 0/2/3)                                            │ │   └────────────────────┘   │ │
┌───────────────────────────────────────┐                              │ │   ┌────────────────────┐   │ │
│ ┌────────────applications───────────┐ │                              │ │   │       mapper       │   │ │
│ │            ┌────────────────────┐ │ │                              │ │   └────────────────────┘   │ │
│ │            │  google proto buf  │ │ │                              │ │   ┌────────────────────┐   │ │
│ │            └────────────────────┘ │ │                              │ │   │ google proto buf $1│   │ │
│ │            ┌────────────────────┐ │ │                              │ │   ├────────────────────│   │ │
│ │            │   yipc-lib socket  │ │ │                              │ │   │    grpc-rpmsg $2   │   │ │
│ │            └────────────────────┘ │ │                              │ │   └─────────+──────────┘   │ │
│ │            ┌────────────────────┐ │ │                              │ └─────────────┼──────────────┘ │
│ │            │ ethernet interface │ │ │                              │ ┌─────────────┼──────────────┐ │
│ │            │      rpmsg 0       │ │ │                              │ │ ┌───────────+────────────┐ │ │
│ │            └──────────+─────────┘ │ │                              │ │ │ PAL api for inter-core │ │ │
│ └──────+────────────────┼───────────┘ │                              │ │ │     communication $3   │ │ │
│        |                |             │                              │ │ └───────────+────────────┘ │ │
│ ┌──────┼────────────────┼───────────┐ │                              │ │             |              │ │
│ │      |        ┌-------┴--------┐  │ │                              │ │     ┌-------+--------┐     │ │
│ │      |        |  virtio stack  |  │ │                              │ │     |  virtio stack  |     │ │
│ │      |        |┌──────────────┐|  │ │                              │ │     |┌──────────────┐|     │ │
│ │      |        |│   endpoint   │|  │ │                              │ │     |│   endpoint   │|     │ │
│ │      |        |├──────────────┤|  │ │                              │ │     |├──────────────┤|     │ │
│ │ ┌────+────┐   |│ rpmsg driver │|  │ │ shared mem & inter-core irqs │ │     |│ rpmsg tx/rx  │|     │ │
│ │ │ drivers │   |├──────────────┤|  │<-------------------------------->│     |├──────────────┤|     │ │
│ │ └────+────┘   |│ virtio ring  │|  │ │                              │ │     |│ virtio ring  │|     │ │
│ │      |        |├──────────────┤|  │ │                              │ │     |├──────────────┤|     │ │
│ │      |        |│  interrupts  │|  │ │                              │ │     |│  interrupts  │|     │ │
│ │      |        |└──────────────┘|  │ │                              │ │     |└──────────────┘|     │ │
│ │      |        └----------------┘  │ │                              │ │     └----------------┘     │ │
│ └──────┼────────linux───────────────┘ │                              │ └----vxworks platform hwa----┘ │
└────────┼──────────────────────────────┘                              └────────────────────────────────┘
---------┼------------------------------------------------------------------------------------------------
      ┌──+─────────────────────────────┐ $1: ...
      │       equipment hardware       │ $2: invoke PAI api to transport the GPB encapsulated req to remote eqpt manager
      └────────────────────────────────┘ $3: invoke rpmsg api to send/recv msg over virtio channel
```

notes for the core 0 side:  
a) the core 0 driver already implement different rpmsg channels as different eth interfaces, such as
rpmsg16, rpmsg32, and more than 10 others.  
b) each interface will serve a separate app independently, eqpt_hwa_app (rpmsg32), fsproxy_app (rpmsg16).  
c) core 0 platform infra already possess a yipc connection mechanism with both standard sockets and
unix domain sockets supported.
to manage the communication between core0 and ihub, yipc need to be extended to support rpmsg/virtio based
communication.

4.1 core 0 related code:

```blurtext
linux-isam-drivers:
./misc/rpmsg_eth_ifc.c
./misc/rpmsg_platform_driver.c
./misc/rpmsg_misc_dev.c
./misc/rpmsg_interface.c

buildroot:
./board/Nokia/isam-reborn/board/lant-a/lanta.dts
./board/Nokia/isam-reborn/board/fant-f-y/fant-f-y-3041.dts
./board/Nokia/isam-reborn/board/fant-f-y/fant-f-y-4040.dts
./board/Nokia/isam-reborn/board/fant-g-y/fanth-y.dts
./board/Nokia/isam-reborn/board/fant-g-y/fantg-y.dts

sw:
./vobs/dsl/sw/y/src/yAPI/src/yipc.c
./vobs/dsl/sw/y/src/yAPI/src/yipc_rpmsg.c
./vobs/dsl/sw/y/src/yAPI/src/yipc_send_recv.c
./vobs/dsl/sw/y/src/yAPI/src/yipc_uds.c
./vobs/dsl/sw/y/src/yAPI/src/yipc_ysocket.c
```

details of the code walk go 
rpmsg device dts in buildroot/board/Nokia/isam-reborn/board/lant-a/lanta.dts:

```blurtext
@soc {
    ...
	rpmsg{
		#address-cells = <0x2>;
		#size-cells = <0>;

		compatible = "fsl,nokia-rpmsg";
		input-msgirq-num = <3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3 3>;  /* all channel use msg irq 3 */
		output-msgirq-num = <7 7 7 7 7 7 7 7 7 7 7 7 7 7 7 7 7>; /* all channel use msg irq 7 */
		input-dstcpu-id = <0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0>;
		output-dstcpu-id = <2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2 2>;
		rpmsg-role = <1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1 1>;        /* 1:MASTER, 2:REMOTE */
		channel-num = <0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16>;
		channel-src = <0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16>;
		channel-dst = <0x100 0x200 0x300 0x400 0x500 0x600 0x700 0x800 0x900 0xA00 0xB00 0xC00 0xD00 0xE00 0xF00 0x1000 0x1100 0x1200>;
		channel-name =  "nokia-fsproxy-channel", "nokia-eqpthwa-channel",
				"nokia-redundancy-channel", "nokia-swm1-channel",
				"nokia-swm2-channel", "nokia-nat-channel",
				"nokia-debug-channel", "nokia-clk1-channel",
				"nokia-clk2-channel", "nokia-typeb1-channel",
				"nokia-typeb2-channel", "nokia-typeb3-channel",
				"nokia-swm-ntio-channel", "nokia-x2-channel",
				"nokia-x3-channel", "nokia-x4-channel",
				"nokia-x5-channel";
		eth-dev-endpoints = <0x10 0x20 0x30 0x40 0x50 0x60 0x70 0x80 0x90 0xA0 0xB0 0xC0 0xD0 0xE0 0xF0 0x100 0x110>;
		misc-dev-endpoints = <0x12 0x22 0x32 0x42 0x52 0x62 0x72 0x82 0x92 0xA2 0xB2 0xC2 0xD2 0xE2 0xF2 0x102 0x112>;
		shmem-rx-vring   = <0x0 0xB8000000 0x0 0xB8210000 0x0 0xB8260000 0x0 0xB82B0000
				    0x0 0xB8300000 0x0 0xB8350000 0x0 0xB83A0000 0x0 0xB83F0000
				    0x0 0xB8440000 0x0 0xB8490000 0x0 0xB84E0000 0x0 0xB8530000
				    0x0 0xB8580000 0x0 0xB85D0000 0x0 0xB8620000 0x0 0xB8670000
				    0x0 0xB86C0000>;
		shmem-tx-vring   = <0x0 0xB8008000 0x0 0xB8218000 0x0 0xB8268000 0x0 0xB82B8000
				    0x0 0xB8308000 0x0 0xB8358000 0x0 0xB83A8000 0x0 0xB83F8000
				    0x0 0xB8448000 0x0 0xB8498000 0x0 0xB84E8000 0x0 0xB8538000
				    0x0 0xB8588000 0x0 0xB85D8000 0x0 0xB8628000 0x0 0xB8678000
				    0x0 0xB86C8000>;
		data-buffer-pool = <0x0 0xB8010000 0x0 0xB8220000 0x0 0xB8270000 0x0 0xB82C0000
				    0x0 0xB8310000 0x0 0xB8360000 0x0 0xB83B0000 0x0 0xB8400000
				    0x0 0xB8450000 0x0 0xB84A0000 0x0 0xB84F0000 0x0 0xB8540000
				    0x0 0xB8590000 0x0 0xB85E0000 0x0 0xB8630000 0x0 0xB8680000
				    0x0 0xB86D0000>;
		rpmsg-num-bufs   = <256 512 512 512 512 512 512 512 512 512 512 512 512 512 512 512 512>;
		rpmsg-buf-size   = <8192 512 512 512 512 512 512 512 512 512 512 512 512 512 512 512 512>;
		status = "okay";
	};
    ...
    msgirq3 {
        compatible = "alu,msgirq";
        msg-num = <0x3>;
        mivpr-offset = <0x51660>;
        midr-offset = <0x51670>;
        rpmsg-use = <0x1>;
    };

    ...

    msgirq7 {
        compatible = "alu,msgirq";
        msg-num = <0x7>;
        mivpr-offset = <0x516e0>;
        midr-offset = <0x516f0>;
        rpmsg-use = <0x1>;
    };
};
```

4.2 code 1 related code:

```txt
┌─────────────────────────────┐  transport layer
│  rpmsg (channel, endpoint)  │ -----------------+
└─────────────────────────────┘                  |  ported to vxworks for core 1
┌─────────────────────────────┐     mac layer    |
│  virtio (virtqueue, vring)  │ -----------------+
└─────────────────────────────┘
┌─────────────────────────────┐
│  shared mem and inter-core  │   phycial layer     existing ihub inco module will
│     interrupt handling      │ -----------------+  be reused for providing shared memory
└─────────────────────────────┘                     and interrupt handling
```

4.3 usecase: file system access (cf card) for core 1 ihub.

```txt
                                                                                       (core 1)
                                                                            ┌─────────────────────────────┐
                                                                            │ ┌─────────────────────────┐ │
                (core 0/2/3)                                                │ │        ion apps         │ │
┌────────────────────────────────────────────┐                              │ └───────────+─────────────┘ │
│ ┌────────────────────────────────────────┐ │                              │ ┌───────────+─────────────┐ │
│ │             file system hwa            │ │                              │ │    file system proxy    │ │
│ └─────+─────────────+────────+───────+───┘ │                              │ └───+───────+─────────────┘ │
│       |             |        |       |     │                              │     |       |               │
│       |             |     ┌──+───┐   |     │                              │  ┌──+───┐   |               │
│       |             |     │ yipc │   |     │                              │  │ yipc │   |               │
│       |             |     └──+───┘   |     │                              │  └──+───┘   |               │
│       |             |        |       |     │                              │     |       |               │
│       |             |     ┌──+───────+──┐  │  virtual io data channel $1  │  ┌──+───────+──┐            │
│       |             |     │             │<-│------------------------------│->│             │            │
│       |             |     │     PAL     │  │                              │  │     PAL     │            │
│       |             |     │             │<-│------------------------------│->│             │            │
│       |             |     └─────────────┘  │ virtio io control channel $2 │  └─────────────┘            │
│ ┌-----┼-------------┼--------------------┐ │                              │ ┌-------------------------┐ │
│ | ┌───+───┐   ┌─────+───────┐            | │                              │ |                         | │
│ | │ rsync +---+ file system │            | │                              │ |         vxworks         | │
│ | └───────┘   └─────+───────┘    linux   | │                              │ |                         | │
│ └-------------------┼--------------------┘ │                              │ └-------------------------┘ │
└─────────────────────┼──────────────────────┘                              └─────────────────────────────┘
----------------------┼-------------------------------------------------------------------------------------
      ┌───────────────+────────────────┐
      │    cf card (owned by core 0)   │
      └────────────────────────────────┘
$1: file system calls are handled directly over rpmsg/virtio data channel.
$2: the control channel is used by yipc session.
```

to access the cf card, the core 1 ihub needs to rely on the rpmsg channel to request assistance from core0,
cf card direct access from core 1 is not possible.

<p style="margin-bottom: 20px;"></p>

5 inter-core interrupts definitions (functionalities, irq notify direction, index)

```txt
─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
                          inter-core message interrupt usage (between core 0 and core 1)
───────────┬───────────────────┬───────────┬───────────────┬───────────┬─────────────┬───────────┬───────────┬───────────
 function  │ notify ihub crash │ uart irq  │ typeb failure │ virtio    │ core1 reset │ core1     │ swo       │ virtio
           │ log collect       │ forward   │ notification  │ msg       │ request     │ watchdog  │ notify    │ msg
───────────┼───────────────────┼───────────┼───────────────┼───────────┼─────────────┼───────────┼───────────┼───────────
 direction │ core 0->1         │ core 0->1 │ core 1->0     │ core 1->0 │ core 1->0   │ core 1->0 │ core 0->1 │ core 0->1
───────────┼───────────────────┼───────────┼───────────────┼───────────┼─────────────┼───────────┼───────────┼───────────
 irq index │ 0                 │ 1         │ 2             │ 3         │ 4           │ 5         │ 6         │ 7
───────────┴───────────────────┴───────────┴───────────────┴───────────┴─────────────┴───────────┴───────────┴───────────
```

<p style="margin-bottom: 20px;"></p>

6 internal/external communications

```txt
todo: get the full image here
```

<hr>

### # implementations on hostfw simulation framework env
for the ihub core 1 integration in simulation framework (hostfw), the core 1 ihub qemu image is needed basically.

we have simulated the rpmsg for inter-core communication, ip for internal communication, and higig for redundancy channels
at the framework level.
ihub also adapted the rpmsg protocol to ensure communication between both sides.
we also simulated the inter-core interrupts for notification of redundancy and watchdog.

<p style="margin-bottom: 20px;"></p>

the development can be progressed in four stages:

1 core 1 ihub bootup sequences:  
a) start the core 1 ihub wrapper container as an isolated env for its qemu image.  
b) prepare qemu cmdline and start core 1 image stored in the wrapper container,
provide port mapping for ihub login.  
c) currently there's no connection with other NT/LT boards.


host kernel & guest kernel architecture
```txt
         ┌──────────────────────qemu-x86-64 emulator─────────────────────┐
         │                                                               │
         │ ┌────────────────────────vxworks 5.x────────────────────────┐ │
         │ │                                                           │ │
         │ │ ┌──────┐  ┌───────┐  ┌─────ipd stack─────┐ ┌───npapi────┐ │ │
         │ │ │ uart │  │ timer │  │ ┌───────────────┐ │ │ ┌────────┐ │ │ │
         │ │ └──────┘  └───────┘  │ │ bsp/pcpentium │ │ │ │ npqemu │ │ │ │
         │ │                      │ └───────────────┘ │ │ └────────┘ │ │ │
         │ │                      └───────────────────┘ └────────────┘ │ │
         │ │                                                           │ │
         │ │                        (guest os)    ┌─────────────┐      │ │
         │ └──────────────────────────────────────│     pci     │──────┘ │
         │                                        └─────────────┘        │
         │   ┌──────┐  ┌───────┐                                         │
         │   │ uart │  │ timer │                                         │
         │   └──────┘  └───────┘                                         │
         │                            (vmm)       ┌─────────────┐        │
         └────────────────────────────────────────│ virtual net │────────┘
         ┌────────────────────────linux kernel────│    itf-1    │────────┐
         │   ┌───────────────┐  ┌───────┐         └─────────────┘        │
         │   │ linux console │  │ timer │                                │
         │   └───────────────┘  └───────┘                                │
         │                           (host)       ┌─────────────┐        │
         └────────────────────────────────────────│  net itf-1  │────────┘
                                                  └─────────────┘
```


qemu commandline:

```text
$ todo
```

hostfw code changesets:

```blurtext
13146:f9c382433634 hostfw: intergration ihub host
13174:b2a8d5bbe95c hostfw: ihub inband support
14342:b7873101156d hostfw: support ihub image download
```

<p style="margin-bottom: 20px;"></p>

2: communication between core 0 and core 1  
inter-core communication support
ihub qemu shall use user mode backend for net0 which response for traffic management (ssh port 22 and netconf port 830)

```txt
todo: add hostfw topo with rpmsg highlighted here.
```

a) ihub qemu shall use tap mode backend for rpmsg and ip channels.  
b) vlan interfaces (rpmsg16\...rpmsg272) created with same name of target to simulate rpmsg interfaces,
so that the moswa apps could use these rpmsg interfaces without code changes for rpmsg channel,
raw socket packets between core0 and ihub should be tagged with 802.1q header (ether type 0x8100, vlan id 16\...272),
ether type shall be 0x8800.  
c) layer2 header (14bytes + 4bytes vlan tag + 2bytes padding for align) should be add/del before yipc frame,
this ensure raw socket packets could be forwarded when it go through linux bridges inter-core related interfaces
from core0 and ihub should connect to br_interCore_1.  
d) for internal communication, 169.254.1.251 should be configured by ihub qemu.  
e) raw socket frame format:

```txt
         (padding 2bytes for totally 4 bytes align)
           ┌────────────────────────────────────┬─────────────┬──────────────┬───────────────────┐
           │ customized layer2 header (20bytes) │ yipc header │ yipc payload │ padding for align │
           └────────────────────────────────────┴─────────────┴──────────────┴───────────────────┘

		                          customized layer 2 header details
    ├------------------------------------------------------------------------------------------------┤
    ┌──────────────────┬──────────────────┬───────────────────┬───────────────────┬──────────────────┐
    │ dst mac (6bytes) │ src mac (6bytes) │ vlan tag (4bytes) │ eth type (2bytes) │ padding (2bytes) │
    └──────────────────┴──────────────────┴───────────────────┴───────────────────┴──────────────────┘
```
