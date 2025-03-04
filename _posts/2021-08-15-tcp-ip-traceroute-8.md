---
layout: post
title: "traceroute: powerful sonar for tcp/ip (tcp-ip vol 1)"
author: "melon"
date: 2021-08-15 19:51
categories: "2021"
tags:
  - network
  - todo
---

traceroute is a tool to explore the tcp/ip protocol in depth, which allow us to see ip datagram route from one
host to another.
it even support the source routing option to record the route taken in ip path.

$ 1 why use traceroute instead of the ip RR option?  
1.1 not all router support the record route option in the early days, so ip RR option cannot be used on all paths.
    the traceroute does not require any special or optional features of the intermediate routers.  
1.2 record route is generally a one-way option. if the sender set the ip RR option, the receiver must extract
    all info from the ip header and return it to the sender.
    on the same ip path, the receiver will copy the ip RR option, thus ip will be recorded on the return path,
    which is be redundant.  
1.3 last but not least: the space reserved for option in ip header is limited, can only store up to 9 ip addr,
    which may be sufficient for ARPANET, but far from enough for today's networks.

<hr>

### # introduction to traceroute
traceroute use icmp msg as carrier and use the ttl (time to live) field in ip header to limit max num of route
that the transmission pass through.

$ 1 ttl field basics  
todo

$ 2 ttl field purposes  
todo

$ 3 how does traceroute derive routing path info  
3.1 the src host send an ip datagram with a ttl=1 to the dst host.
    the 1st hop router of the ip datagram return the ttl-1, discard the datagram,
    and return a timeout icmp msg. in this way, the src host obtain the "1st hop router ip" in routing path.  
3.2 the traceroute send an ip packet with a ttl=2 to the dst host, so the "2nd hop router" can be obtained.
    and so on, until the next hop target is the destination host.  
3.3 even when the dst host receive a datagram with a ttl=1, it will not discard the datagram,
    nor will it generate a timeout icmp msg (the data has reached the dst).
    this time, the traceroute program send a udp datagram to the dst host and select a "not likely number (>30000)"
    as the udp port number (no program on dst host to use this port),
    by this chance, the dst host udp service module will generate a port unreachable icmp msg and return it to the
    src host, and in this way return its own ip addr.  
3.4 the traceroute determine when to end by judging the type of icmp msg received (timeout icmp, port unreachable icmp).

$ 4 basic requirement of traceroute  
todo

<hr>

### # usecase: traceroute on local area network

<hr>

### # usecase: traceroute on wide area network

<hr>

### # ip source routing options
$ 1 ip source routing basics  
generally ip routing is dynamic: each router must determine which router the datagram is forwarded to.
app do not control or concerned about the exact route, but can discover the actual route via traceroute.

source routing is that the datagram sender specifies the route, mainly come in two forms:  
$ a strict source routing: the sender specify the exact routing path for ip datagram to take.
during transmission of the ip datagram, if a router find that the next router specified by the source
route is not on the network directly connected to it, it will return "source route failed" msg.  
b) loose source routing: the sender specify a list of ip addr that must be passed, but other routers are
allowed to exist in between any two of them.

$ 2 ip source routing datagram option format  

$ 3 execution process for ip source routing  
intro + graph explanation

$ 4 host requirement rfc: for tcp client to support source routing

$ 5 loose source routing by traceroute

$ 6 strict source routing by traceroute

$ 7 loose source round-trip routing by traceroute

<hr>

### # conclusion
in network using the tcp/ip protocol, traceroute is an indispensable research tool.
the traceroute tool has simple and clear cli, but powerful functionalities, and is very useful for
routing path analysis.
