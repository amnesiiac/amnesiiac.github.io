---
layout: post
title: "static routing support in ip layer (tcp-ip vol 1)"
author: "melon"
date: 2021-08-18 20:36
categories: "2021"
tags:
  - network
  - todo
---

routing is one of the most important function of ip.
datagram that need to be routed can be generated locally or by other hosts.
the latter require the host to be configured as a router, otherwise if the dst host of the datagram received
is not the current host, the datagram will be silently discarded.

ip routing daemon work process in ip stack diagram (refine needed):

```txt
                                                                            ┌─────┐                   ┌─────┐
                                                                            │ udp │                   │ tcp │
                                                                            └──+──┘                   └──+──┘
                                                                               │                         │
                              ┌------------------------------------------------│-------------------------│-------------┐
                              |                        ┌──────┐                └─────────────────────────┤             |
                              |                        │ icmp │<─────────────────────────────────────────┤y (3 choice) |
                              |                        └───┬──┘ ┌────────────────────────────────────────┴───────────┐ |
                              |                            │    │ is the ip datagram sent to current host?           │ |
┌───────────────┐             |                            │    │ a: the ip datagram dst addr is one of cur host ip? │ |
│ routing table │             |                            │    │ b: the ip datagram a broadcast packet?             │ |
│ daemon        ├-----------┐ |                            │    └────────────────────────────────────────+───────────┘ |
└───────────────┘           | |   ┌─────────┐   ┌──────────+──────────┐                                  │             |
┌───────────────────┐       └---->│ routing │   │ ip stack egress     │  if cur pkt is source  ┌─────────┴───────────┐ |
│ routing table cli ├--route add->│ table   ├──>│ compute next hop    │<-----------------------┤ handling ip options │ |
└───────────────────┘       ┌-----┤ entries │   │ router addr if need │  routing datagram?     └─────────+───────────┘ |
┌───────────────────┐       | |   └─────────┘   └──────────┬──────────┘                                  │             |
│ netstat for print │<------┘ |                            │                                             │             |
│ routing table     │         |                            │                                   ┌─────────┴────────┐    |
└───────────────────┘         |                            │                                   │ ip ingress queue │    |
                              |IP STACK                    │                                   └─────────+────────┘    |
                              └--------------------------------------------------------------------------│-------------┘
                                                           │                                             │
                                                  ┌────────+─────────────────────────────────────────────┴─────────────┐
                                                  │                          network interface card                    │
                                                  └────────────────────────────────────────────────────────────────────┘
```

<hr>

### # ip routing principles
intro...

$ 1 simple routing table analysis by netstat -rn

1.1 subnet mask & interface ip addr together determine the host num

1.2 factors to impact on routing table complexity 

$ 2 ip routing usecase analysis

$ 3 initialization of routing table

$ 4 netstat & ifconfig output on svr4 host

$ 5 what if no matching table entry for certain pkt, and no default entry provided?

<hr>

### # icmp datagram: host unreachable & destination unreachable

<hr>

### # whether allow forwarding ip packets

<hr>

### # icmp redirecton error datagram
when a router receive an ip datagram that should not be sent to it, it will send an icmp redirect error datagram
to the sender of the ip datagram.

$ 1 icmp redirection datagram case analysis
todo

$ 2 icmp redirection datagram application scenarios  
icmp redirect datagram are used to allow host using little routing info to buildup a complete routing table.
it allow host to use dumb logic to build routing table (without the need for complex function support),
and offload the part requiring flexible and intelligent implementation to the router side.

for example, all host connected to a lan only need to use a default route when starting up,
and then all other item in the routing table are learned through icmp redirection mechanism.

$ 3 a more complex case analysis

$ 4 icmp redirection datagram format

<hr>

### # icmp routing discovery datagram
one way to update routing table is 'ifconfig & route add', which is used to init routing table, add
default routing table entries in config file.
another way to update ip routing table is by icmp notification & request datagrams.

generally, after the host complete system bootstrap, it send an icmp solicitation msg by broadcast or multicast.
one or more router respond to the solicitation msg and add the notification info to the response msg.
in addition, the router will periodically send their notification msg by broadcast or multicast,
allowing each listening host to regularly update their routing table entries.

$ 1 icmp routing request datagram format (RFC-1256, broadcast/multicast)

$ 2 icmp routing notification datagram format (RFC-1256, broadcast/multicast)

$ 3 router operations

$ 4 host operations
