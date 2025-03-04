---
layout: post
title: "incorrect checksum of pkt caputred by tcpdump (network, tcpdump)"
author: "melon"
date: 2024-03-10 21:11
categories: "2024"
tags:
  - network
---

why we are seeing lots of incorrect tcp checksum erros in the packets captured by tcpdump?
is it meaning that the packet received is wrong? can we solve or just neglect this?

<hr>

### # how to reproduce tcp checksum error
startup device simulation instance, use telnet to send tcp sync packet through the dut_ip:port:

```text
$(host) start-ls-host.py --set models=setup_mx_fixed,board=rant-c-y,inband=true
$(host) telnet 10.1.11.27 830
```

inside the device, try capture all packets on the destination interface (ip: 10.1.11.27):

```text
$(device) tcpdump -i ${inband_interface} -vvnnXX
[68865.605986] device inband entered promiscuous mode
tcpdump: listening on inband, link-type EN10MB (Ethernet), snapshot length 262144 bytes
03:51:42.132969 IP (tos 0x0, ttl 64, id 46914, offset 0, flags [DF], proto TCP (6), length 60)
    10.1.11.17.45492 > 10.1.11.27.830: Flags [S], cksum 0x2a5c (incorrect -> 0x3260), seq 261057083, win 64240,
    options [mss 1460,sackOK,TS val 3588811739 ecr 0,nop,wscale 7], length 0
        0x0000:  0019 8f5f ab4e 7a97 9c93 952a 0800 4500  ..._.Nz....*..E.
        0x0010:  003c b742 4000 4006 594c 0a01 0b11 0a01  .<.B@.@.YL......
        0x0020:  0b1b b1b4 033e 0f8f 6a3b 0000 0000 a002  .....>..j;......
        0x0030:  faf0 2a5c 0000 0204 05b4 0402 080a d5e8  ..*\............
        0x0040:  ebdb 0000 0000 0103 0307 70ba 4cbc       ..........p.L.
03:51:47.252639 ARP, Ethernet (len 6), IPv4 (len 4), Request who-has 10.1.11.27 tell 10.1.11.17, length 50
        0x0000:  0019 8f5f ab4e 7a97 9c93 952a 0806 0001  ..._.Nz....*....
        0x0010:  0800 0604 0001 7a97 9c93 952a 0a01 0b11  ......z....*....
        0x0020:  0000 0000 0000 0a01 0b1b 0000 0000 0000  ................
        0x0030:  0000 0000 0000 0000 0000 0000 fdea 72cf  ..............r.
03:51:47.252752 ARP, Ethernet (len 6), IPv4 (len 4), Reply 10.1.11.27 is-at 00:19:8f:5f:ab:4e, length 28
        0x0000:  7a97 9c93 952a 0019 8f5f ab4e 0806 0001  z....*..._.N....
        0x0010:  0800 0604 0002 0019 8f5f ab4e 0a01 0b1b  ........._.N....
        0x0020:  7a97 9c93 952a 0a01 0b11                 z....*....
03:52:00.593702 ARP, Ethernet (len 6), IPv4 (len 4), Reply 10.1.11.27 is-at 00:19:8f:5f:ab:4e, length 28
        0x0000:  ffff ffff ffff 0019 8f5f ab4e 0806 0001  ........._.N....
        0x0010:  0800 0604 0002 0019 8f5f ab4e 0a01 0b1b  ........._.N....
        0x0020:  0019 8f5f ab4e 0a01 0b1b                 ..._.N....
03:52:01.595312 ARP, Ethernet (len 6), IPv4 (len 4), Reply 10.1.11.27 is-at 00:19:8f:5f:ab:4e, length 28
        0x0000:  ffff ffff ffff 0019 8f5f ab4e 0806 0001  ........._.N....
        0x0010:  0800 0604 0002 0019 8f5f ab4e 0a01 0b1b  ........._.N....
        0x0020:  0019 8f5f ab4e 0a01 0b1b                 ..._.N....
03:52:02.595498 ARP, Ethernet (len 6), IPv4 (len 4), Reply 10.1.11.27 is-at 00:19:8f:5f:ab:4e, length 28
        0x0000:  ffff ffff ffff 0019 8f5f ab4e 0806 0001  ........._.N....
        0x0010:  0800 0604 0002 0019 8f5f ab4e 0a01 0b1b  ........._.N....
        0x0020:  0019 8f5f ab4e 0a01 0b1b                 ..._.N....
03:52:14.388607 IP (tos 0x0, ttl 64, id 46915, offset 0, flags [DF], proto TCP (6), length 60)
    10.1.11.17.45492 > 10.1.11.27.830: Flags [S], cksum 0x2a5c (incorrect -> 0xb460), seq 261057083, win 64240,
    options [mss 1460,sackOK,TS val 3588843994 ecr 0,nop,wscale 7], length 0
        0x0000:  0019 8f5f ab4e 7a97 9c93 952a 0800 4500  ..._.Nz....*..E.
        0x0010:  003c b743 4000 4006 594b 0a01 0b11 0a01  .<.C@.@.YK......
        0x0020:  0b1b b1b4 033e 0f8f 6a3b 0000 0000 a002  .....>..j;......
        0x0030:  faf0 2a5c 0000 0204 05b4 0402 080a d5e9  ..*\............
        0x0040:  69da 0000 0000 0103 0307 2581 0a70       i.........%..p
03:52:43.279549 IP (tos 0xb8, ttl 64, id 62521, offset 0, flags [DF], proto UDP (17), length 76)
    10.1.11.27.123 > 10.1.11.17.123: [udp sum ok] NTPv4, Client, length 48
        Leap indicator: clock unsynchronized (192), Stratum 0 (unspecified), poll 10 (1024s), precision -20
        Root Delay: 0.000000, Root dispersion: 0.000000, Reference-ID: (unspec)
          Reference Timestamp:  0.000000000
          Originator Timestamp: 0.000000000
          Receive Timestamp:    0.000000000
          Transmit Timestamp:   3934583563.279163409 (2024-09-06T03:52:43Z)
            Originator - Receive Timestamp:  0.000000000
            Originator - Transmit Timestamp: 3934583563.279163409 (2024-09-06T03:52:43Z)
        0x0000:  7a97 9c93 952a 0019 8f5f ab4e 0800 45b8  z....*..._.N..E.
        0x0010:  004c f439 4000 4011 1b82 0a01 0b1b 0a01  .L.9@.@.........
        0x0020:  0b11 007b 007b 0038 f991 e300 0aec 0000  ...{.{.8........
        0x0030:  0000 0000 0000 7f00 0001 0000 0000 0000  ................
        0x0040:  0000 0000 0000 0000 0000 0000 0000 0000  ................
        0x0050:  0000 ea84 fb0b 4777 40d2                 ......Gw@.
```

the first tcp packet received reports "cksum 0x2a5c (incorrect -> 0x3260)" warning, while other packets
in different proto no such warning.

<hr>

### # explanations of the tcp checksum issue
1 who added the tcp checksum to the packets?  
if the pkts with incorrect tcp checksums are sent by the same machine on which wireshark is running,
then the issue is likely to happen because: the network interface used to capture the tcp pkts does tcp
checksum offloading: the tcp checksum is added to the pkt by the network interface, not by the os’s
tcp/ip stack.

2 why the tcp checksum computed by interface is wrong?  
when capturing on an interface, pkts sent by the host (capturing the pkt on) are directly handed to the
capture interface by the os: the pkts are handed to the capture interface without a tcp checksum added.

3 how to avoid the warning?  
a) one way to prevent this can be to disable tcp checksum offloading, which might not possible on some os;
and this will significantly reduce networking performance.  
b) while, you can disable the check that wireshark does of the tcp checksum, so that it won’t report any pkts 
as having tcp checksum errors, and it won’t refuse to do tcp reassembly due to incorrect tcp checksum of a pkt.

4 ways to ignore the tcp checksum error report  
a) wireshark gui:
select "preferences" from the "edit" menu, open up the "protocols" list in the left-hand pane of the
"preferences" dialog box, select "tcp", from that list, turn off the "check the validity of the tcp
checksum when possible" option, click "save" if you want to save that setting in your preference file,
and click "ok".  
b) wireshark or tshark cmdline:
you can add option "-o tcp.check_checksum:false", or manually set "tcp.check_checksum:false line" in config
file.  
c) tcpdump cmdline:

```text
$ tcpdump -i ${dev} -vvnnXX --dont-verify-checksums
```

<hr>

### # reference
ref: https://www.wireshark.org/faq.html#_why_am_i_seeing_lots_of_packets_with_incorrect_tcp_checksums
