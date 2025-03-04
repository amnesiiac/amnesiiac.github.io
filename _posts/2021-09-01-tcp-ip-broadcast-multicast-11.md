---
layout: post
title: "broadcast & multicast (tcp-ip vol 1)"
author: "melon"
date: 2021-09-01 21:04
categories: "2021"
tags:
  - network
  - todo
---

iP addr can be roughly divided into three types: unicast addr, broadcast addr, and multicast addr.
the broadcast and multicast way to send data are only applicable to udp datagrams rather than tcp.
because, tcp is a connection-oriented protocol, which means there is a tcp connection between each side.

$ 1 unicast & boardcast & multicast  
normally, each eth frame is sent to only one dst, and the dst addr ref to certain port on a single host (unicast).
when unicast is used, communication between the src and dst will not affect other communications,
but only competing for shared physical path with them.

however, if the dst of an eth frame is all other hosts on the network, it's called a broadcast.
the sending mechanism of broadcast can be observed by arp & rarp requests by tcpdump.

the transmission method between unicast & broadcast is called multicast, in which data is only sent to
hosts within the multicast group.

$ 2 defects of broadcast transmission  
broadcast is targeted at all host on network, but not all host on network should respond to it,
which increase the network load of non-related host.
for example, there are 50 hosts in network, but only 20 of them participate in the udp broadcast app,
the remaining hosts have to process the broadcast datagram until the udp layer and discard it due to
port number mismatch.

$ 3 advantages of multicast over broadcast  
multicast can solve the above problem as: host on network join one or more multicast groups.
after certain network card learn the group of all hosts, it will perform multicast according to group record,
greatly reducing the redundant eth frame processing burden.

$ 4 the proto stack process path of ethernet data frame

```txt
                    ┌──────────────────────────┐
                    │                          │
 application layer  │   telnet / ftp / email   │
                    │                          │
                    └────────────+─────────────┘
                                 │ yes
                    ┌────────────┴─────────────┐
                    │                          │ udp stack layer filter:
 transmission layer │           udp            │ is src/dst port match?
                    │                          │-----------------------------> discard & return icmp unreachable
                    └────────────+─────────────┘            no
                                 │ yes
                    ┌────────────┴─────────────┐
                    │                          │ ip stack layer filter:
 network layer      │  internet proto layer    │ is src/dst ip address match?
                    │                          │-----------------------------> discard
                    └────────────+─────────────┘             no
                                 │ yes
                    ┌────────────┴─────────────┐
                    │                          │ frame filter: frame type match with designated type (ip,arp)?
                 ┌--│  device driver program   │ multicast filter: is current host belong to multicast group?
                 |  │                          │--------------------------------------------------------------> discard
                 |  └────────────+─────────────┘                             no
 data link layer-|               │ yes
                 |  ┌────────────┴─────────────┐
                 |  │                          │ is ethernet frame dst addr match up with nic mac addr?
                 └--│  network interface card  │ is ethernet frame dst addr broadcast addr? checksum correct?
                    │                          │--------------------------------------------------------------> discard
                    └────────────+─────────────┘                             no
 ethernet                        │
─────────────────────────────────┴────────────────────────────────────────────────────────────────────────────
```

<hr>

### # broadcast

$ 1 restricted broadcast  
1.1 definition  
the router on the network cannot forward broadcast datagram with restricted dst addr under any circumstances.
that is, restricted broadcast datagram can only be propagated in local network rather than pass through the router.
the restricted local network broadcast address is 255.255.255.255 (the dst addr of the datagram from src).

1.2 restricted broadcast datagram transmission by multi-interface host  
todo

$ 2 restricted broadcast (network-oriented)  
todo

$ 3 restricted boardcast (subnet-oriented)  
todo

$ 4 restricted boardcast (all-subnet-oriented)  
todo

<hr>

### # broadcast usecase analysis
todo

<hr>

### # multicast
$ 1 ip multicast is mainly responsible for two types of services:  
1.1 data transmission to multiple dst addr (deliver info to multiple recipients).
    for example: interactive conference systems, group mailing, news. without multicast, tcp can also be used
    to deliver a copy of the datagram to each multicast dst addr (many app use tcp to enable multicast to
    ensure transmission reliability).  
1.2 client requests to servers are sent using multicast (solicitation of server by clients),
    for example: a diskless workstation send requests to multiple boot servers, while currently, this service
    is completed using broadcast (chapter16 bootp). multicast is used to reduce the burden of hosts in network,
    for those who do not support diskless system boot service.

$ 2 multicast group address  
todo

$ 3 conversion from multicast group address to ethernet address  
todo

$ 4 multicast in fddi & token ring network  
todo

<hr>

### # conclusion
