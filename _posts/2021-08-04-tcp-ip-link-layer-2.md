---
layout: post
title: "link layer protocol (tcp-ip vol 1)"
author: "melon"
date: 2021-08-04 22:18
categories: "2021"
tags:
  - network
  - todo
---

### # IEEE 802.2/802.3 & Ethernet packet encapsulation

<hr>

### # slip (serial line internet protocol): min encapsulation

<hr>

### # ppp (point to point protocol)

<hr>

### # loopback interface: for packet with destination to itself

<hr>

### # max transmission unit (mtu)

```txt
─────────────────────────────────────────────────
 network                         | mtu (bytes)
─────────────────────────────────────────────────
 super channel                   | 65535
 16mb/s token ring (IBM)         | 17914
 4mb/s token ring (IEEE 802.5)   | 4464
 FDDI                            | 4352
 ethernet                        | 1500
 IEEE 802.3/802.2                | 1492
 X.25                            | 576
 ppp (low latency)               | 296
─────────────────────────────────────────────────

1 mtu (maximum transmission unit) link layer protocol is used as restrictions for link-layer data frame length.
2 mtu is logical restrictions but not physical, in order to obtain relative fast response time.
3 if len of ip datagram is larger than link-layer mtu, the datagram is segmented, to make each ip segment len < mtu.
4 mtu is fixed in certain network, for cross-network networking, the path-mtu is dominent (network-path min mtu).
5 path-mtu is not a static value, depending on the actual data transmission path; thus path-mtu (A->B) need not be
  same as path-mtu (B->A).
6 RFC-1191 describes mtu discover mechanism in all situations.
```

<hr>

### # todo
