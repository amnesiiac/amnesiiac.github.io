---
layout: post
title: "cat /proc/${pid}/cmdline (linux)"
author: "melon"
date: 2024-01-25 22:50
categories: "2024"
tags:
  - linux
---

### # introduction
inside device container:
```text
$ ps aux
USER     PID %CPU %MEM    VSZ   RSS TTY    STAT START    TIME  COMMAND
root       1  0.0  0.0   1044     4 ?      Ss   Jan18    0:04  /sbin/docker-init -- sh /entrypoint.sh tail -f /dev/null
root       7  0.0  0.0  11704  2656 ?      S    Jan18    0:00  sh /docker-entrypoint.sh tail -f /dev/null
tcpdump   13  0.0  0.0  26492  8512 ?      Ss   Jan18    0:10  tcpdump -n -s 2048 -i eth_ip -w xxx.pcap 
tcpdump   25  0.0  0.0  26492  8512 ?      Ss   Jan18    0:05  tcpdump -n -s 2048 -i eth_rpmsg -w out.pcap 
tcpdump   37  0.0  0.0  26492  8404 ?      Ss   Jan18    0:04  tcpdump -n -s 2048 -i eth_higig -w out.pcap 
tcpdump   53  0.0  0.0  26492  8340 ?      Ss   Jan18    0:05  tcpdump -n -s 2048 -i tap_rpmsg -w out.pcap
tcpdump   59  0.0  0.0  26492  8452 ?      Ss   Jan18    0:10  tcpdump -n -s 2048 -i tap_ip -w out.pcap
tcpdump   65  0.0  0.0  26492  8420 ?      Ss   Jan18    0:04  tcpdump -n -s 2048 -i tap_higig -w out.pcap
root      99 21.4  1.5 5598812 1711524 ?   Sl   Jan18  308:18  ./qemu-system-x86_64 -m 5120M -enable-kvm -serial ...
root     153  5.0  0.0  21220  3332 pts/0  Ss   08:04    0:00  bash
root     167  2.0  0.0  53796  3452 pts/0  R+   08:04    0:00  ps aux
```

check qemu process full cmdline using procfs:
```text
$ cat /proc/99/cmdline
./qemu-system-x86_64-m5120M-enable-kvm-serialmon:stdio-nographic/root/overlay/sros-vm.qcow2-snapshot-devicevirtio-net-
pci,netdev=net0,mac=DE:AD:BE:EF:37:30-netdevuser,id=net0,hostfwd=tcp::22-:22,hostfwd=tcp::830-:830-devicevirtio-net-pc
i,netdev=net1,mac=DE:AD:BE:EF:38:31-netdevtap,id=net1,ifname=tap_ip,script=no,downscript=no-devicevirtio-net-pci,netde
v=net2,mac=DE:AD:BE:EF:38:32-netdevtap,id=net2,ifname=tap_rpmsg,script=no,downscript=no-devicevirtio-net-pci,netdev=ne
t3,mac=DE:AD:BE:EF:38:33-netdevtap,id=net3,ifname=tap_higig,script=no,downscript=no-devicevirtio-net-pci,netdev=net4,m
ac=DE:AD:BE:EF:38:34-netdevtap,id=net4,ifname=tap4,script=no,downscript=no-devicevirtio-net-pci,netdev=net5,mac=DE:AD:
BE:EF:38:35-netdevtap,id=net5,ifname=tap5,script=no,downscript=no-devicevirtio-net-pci,netdev=net6,mac=DE:AD:BE:EF:38:
36-netdevtap,id=net6,ifname=tap6,script=no,downscript=no-devicevirtio-net-pci,netdev=net7,mac=DE:AD:BE:EF:38:37-netdev
tap,id=net7,ifname=tap7,script=no,downscript=no
```

however, the space of original cmd is missing, using '\0' to separate, hence the solution is as:
```text
$ cat /proc/99/cmdline | tr "\0" " "
./qemu-system-x86_64 -m 5120M -enable-kvm -serial mon:stdio -nographic /root/overlay/sros-vm.qcow2 -snapshot \
-device virtio-net-pci,netdev=net0,mac=DE:AD:BE:EF:37:30 -netdev user,id=net0,hostfwd=tcp::22-:22,hostfwd=tcp::830-:830 
-device virtio-net-pci,netdev=net1,mac=DE:AD:BE:EF:38:31 -netdev tap,id=net1,ifname=tap_ip,script=no,downscript=no \
-device virtio-net-pci,netdev=net2,mac=DE:AD:BE:EF:38:32 -netdev tap,id=net2,ifname=tap_rpmsg,script=no,downscript=no \
-device virtio-net-pci,netdev=net3,mac=DE:AD:BE:EF:38:33 -netdev tap,id=net3,ifname=tap_higig,script=no,downscript=no \
-device virtio-net-pci,netdev=net4,mac=DE:AD:BE:EF:38:34 -netdev tap,id=net4,ifname=tap4,script=no,downscript=no \
-device virtio-net-pci,netdev=net5,mac=DE:AD:BE:EF:38:35 -netdev tap,id=net5,ifname=tap5,script=no,downscript=no \
-device virtio-net-pci,netdev=net6,mac=DE:AD:BE:EF:38:36 -netdev tap,id=net6,ifname=tap6,script=no,downscript=no \
-device virtio-net-pci,netdev=net7,mac=DE:AD:BE:EF:38:37 -netdev tap,id=net7,ifname=tap7,script=no,downscript=no
```

ref: https://www.kernel.org/doc/Documentation/filesystems/proc.txt
