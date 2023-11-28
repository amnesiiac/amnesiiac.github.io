---
layout: post
title: "device drivers (linux kernel architecture ch6)"
author: "melon"
date: 2023-05-28 12:40
categories: "2023"
tags:
    - linux
    - ongoing
---

### # layer model for addressing peripherals
```txt
            ┌─────────────────┐
            │   application   │
            └──────+───┬──────┘
                   │   │
              ┌────┴───+────┐device special file      user space
--------------│  /dev/xyz   │-----------------------------------
              └────+───┬────┘                       kernel space
                   │   │
         ┌─────────┴───+──────────┐
         │  virtual file system   │   
         ├────────────────────────┤
         │         driver         │   
         ├────────────────────────┤
         │        hardware        │   
         └────────────────────────┘
```

<hr>

### # link example of the bus system
```txt
      ┌───────┐   memory   ┌───────┐
      │  RAM  +────────────+  CPU  │
      └───────┘    bus     └───+───┘
                               │
  ────+────────────────────────+─────────────────── PCI#0
      │ pci bridge       slot      slot      slot
  ────+─────────+─────────+─────────+─────────+──── PCI#1
                │                   │          pci slots
         ┌──────+──────┐     ┌──────+───────┐
         │usb controler│     │SCSI controler+──+───scanner
         └──────+──────┘     └──────+───────┘  │
                │                   │       harddisk
            ┌───+───┐    harddisk───+
            │roothub│               │
            └─+───+─┘             CD ROM
              │   │
             hub  webcam
             / \
          mouse keyboard
```

<hr>

### # methods to interaction with peripherals
(1) I/O ports  
(2) I/O memory mapping  
(3) polling and interrupts  
