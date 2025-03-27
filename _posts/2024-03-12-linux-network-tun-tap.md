---
layout: post
title: "tun/tap devices (network)"
author: "melon"
date: 2024-03-12 22:29
categories: "2024"
tags:
  - network
---

in computer networking, tun and tap are kernel virtual netdev.
being netdev supported entirely in software, they differ from ordinary netdev which are backed by physical
network adapters.

tun, namely network tunnel, simulate a network layer device and operate in layer 3 carrying ip packets.
tap, namely network tap, simulate a link layer device and operate in layer 2 carrying ethernet frames.
tun is used with routing.
tap can be used to create a userspace network bridge.
though both are for net tunnel purpose, tun and tap cannot be used together because they tx and rx packets
at different layer of the net stack.

packet sent by an os via a tun/tap device are delivered to a userspace program which attach itself to the device.
a userspace program may also pass packets into a tun/tap device.
in this case the tun/tap device deliver (or say inject) these packet to the os network stack thus emulating their reception from an external source.

```txt
                            ip pkts
         layer 3 <----------- tun -----------> layer 3

                           eth frames
         layer 2 <----------- tap -----------> layer 2
```

<hr>

### # usecase analysis
the requirement is to create multiple tap interface, each assigned a ip addr on same subnet:

```txt
                   br0 (192.168.1.199)
                    │
         ┌──────────┴─────────┬───────┬──────┬──────┬─────┐
         │                    |       |      |      |     |
       eth0                  tap0    tap1   tap2   tap3   tap4
                    (192.168.1.150)  (.151) (.152) (.153) (.154)
```

by connecting them to a linux bridge, all the tap interface is assumed to be reachable from external pc.
however, when ping from tap0 to external computer say 192.168.1.200, it's not working:

```text
$ ping -I tap0 192.168.1.200
```

when ping from 192.168.1.200 (bridge) to 192.168.1.150 (tap0), it is working, and the mac addr of the br0 is retrieved.

1 how to ping from tap interface to external?

```text
$ ping -I tap0  # tell ping to send the ping packet out on tap0
```

which will bypass the linux bridge and really only send on the specified "physical" interface.
so, effectively, you're not "ping from" the tap interface, you're "ping to" it.
if want to "ping from" the tap interface, need to attach something to it (e.g. openvpn) and
send the ping from the other end of the virtual cable the tap interface is connected to.

2 how to get the mac address of the right tap interface, when pinged from outside?

```text
$ arp -i br0 -Ds 192.168.1.150 tap0 pub
```

<hr>

### # hostfw case analysis
test on hostfw instance, inside nested ihub container (setup -> olt -> ihub):

```txt
                 br_ihub_rpmsg
                      │
         ┌────────────┴────────────┐
         │                         │
     eth_rpmsg                 tap_rpmsg
```

check the bridge layouts:

```text
$ brctl show
bridge name       bridge id               STP enabled     interfaces
br_ihub_higig     8000.06717b013c76       no              eth_higig
                                                          tap_higig
br_ihub_ip        8000.0a9fd1cfc98e       no              eth_ip
                                                          tap_ip
br_ihub_rpmsg     8000.2eb2f2dfb21f       no              eth_rpmsg    <--- veth
                                                          tap_rpmsg    <--- tap
```

intend to ping via the tap interface to arbitary ip:

```text
$ ping -I tap_rpmsg 1.1.1.1 &
[1] 70
$ ping: Warning: source address might be selected on device other than tap_rpmsg.
PING 1.1.1.1 (1.1.1.1) from 172.101.0.4 tap_rpmsg: 56(84) bytes of data.
```

capture on the bridge of all protos:

```text
$ tcpdump -i br_ihub_rpmsg -nevvXX
tcpdump: listening on br_ihub_rpmsg, link-type EN10MB (Ethernet), capture size 262144 bytes
08:07:56.094646 00:01:02:03:04:05 > Broadcast, ethertype 802.1Q (0x8100), length 112: vlan 32, p 0, ethertype 0x8800,
        0x0000:  ffff ffff ffff 0001 0203 0405 8100 0020  ................
        0x0010:  8800 0000 0000 0100 0000 0000 0000 0030  ...............0
        0x0020:  0000 0001 0000 0004 0000 0b59 0000 0002  ...........Y....
        0x0030:  0000 0004 0000 0000 0000 0003 0000 0004  ................
        0x0040:  0000 0018 0000 0005 0000 0004 0000 0000  ................
        0x0050:  0000 0004 0000 0018 0a00 1080 021a 0608  ................
        0x0060:  0110 0118 006a 0908 a201 1000 1860 200a  .....j.......`..
08:07:56.096397 e2:7b:5e:22:32:cc > Broadcast, ethertype 802.1Q (0x8100), length 132: vlan 32, p 0, ethertype 0x8800,
        0x0000:  ffff ffff ffff e27b 5e22 32cc 8100 0020  .......{^"2.....
        0x0010:  8800 0000 0000 0100 0000 0000 0000 0030  ...............0
        0x0020:  0000 0001 0000 0004 0000 0b59 0000 0002  ...........Y....
        0x0030:  0000 0004 0000 0001 0000 0003 0000 0004  ................
        0x0040:  0000 002a 0000 0007 0000 0004 0000 0000  ...*............
        0x0050:  0000 0004 0000 002a 0a02 0800 1080 021a  .......*........
        0x0060:  0608 0110 0118 0072 1908 a201 1000 1860  .......r.......`
        0x0070:  200a 2a0e 2380 0181 01a8 0107 b001 0000  ..*.#...........
        0x0080:  0000 00a0                                ....
^C
2 packets captured
2 packets received by filter
0 packets dropped by kernel
2 packets dropped by interface
```

the result shows nothing related to the ping is catched, just pkt between ihub-nt through predefined vlan interfaces.

actually, by using ping -I tap, we are 'ping to it' rather than 'ping from it', so the attached linux bridge
wont get the packets.
bridge work on layer 3, while tap are layer 2, if try to see packet on the linux bridge from the tap,
need to setup local network on the tap.

<hr>

### # reference
ref: https://www.kernel.org/doc/Documentation/networking/tuntap.txt
