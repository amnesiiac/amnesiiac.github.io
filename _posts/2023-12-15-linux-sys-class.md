---
layout: post
title: "sys/class dir (linux)"
author: "melon"
date: 2023-12-15 20:41
categories: "2023"
tags:
  - linux
  - ongoing
---

### # introduction to /sys/class dir
/sys/class is a directory in the Linux filesystem that provides a way to interact with the 
kernel and access information about various classes of devices and subsystems.

it is part of the sysfs virtual filesystem, which exposes information about the kernel, 
devices, and their attributes.

in the context of /sys/class, the directory contains subdirectories, each corresponding to 
a particular class of devices or subsystems. each class directory contains information and 
control files that allow users and applications to query and modify attributes of devices.

check the supported devices:
```text
$ pwd
/sys/class

$ ls
ata_device  bsg          dmi             graphics     leds      net            raw          spi_master  vc
ata_link    cpuid        drm             hidraw       mdio_bus  pci_bus        regulator    thermal     vtconsole
ata_port    dax          drm_dp_aux_dev  hwmon        mem       pcmcia_socket  rfkill       tpm         wakeup
backlight   devcoredump  extcon          i2c-adapter  misc      phy            rtc          tpmrm       watchdog
bdi         devfreq      firmware        input        msr       power_supply   scsi_device  tty
block       dma          gpio            iommu        nd        ppdev          scsi_host    usbmon
```

annotate them as a tree:
```text
$ tree -L 1
.
├── ata_device
├── ata_link
├── ata_port
├── backlight
├── bdi
├── block
├── bsg
├── cpuid
├── dax
├── devcoredump
├── devfreq
├── dma
├── dmi
├── drm
├── drm_dp_aux_dev
├── extcon
├── firmware
├── gpio                # provide access to gpio pins on the systems
├── graphics
├── hidraw
├── hwmon
├── i2c-adapter
├── input
├── iommu
├── leds
├── mdio_bus
├── mem
├── misc
├── msr
├── nd
├── net                # contains info for network devices
├── pci_bus
├── pcmcia_socket
├── phy
├── power_supply       # contains information about power supplies, such as batteries or AC adapters.
├── ppdev
├── raw
├── regulator
├── rfkill
├── rtc
├── scsi_device
├── scsi_host
├── spi_master
├── thermal
├── tpm
├── tpmrm
├── tty
├── usbmon
├── vc
├── vtconsole
├── wakeup
└── watchdog

52 directories, 0 files
```
