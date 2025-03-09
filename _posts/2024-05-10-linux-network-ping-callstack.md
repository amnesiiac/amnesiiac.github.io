---
layout: post
title: "ping cmd underlying callstack in various application scenarios (linux, network, tool)"
author: "melon"
date: 2024-05-10 21:49
categories: "2024"
tags:
  - linux
  - network
  - todo
---

### # application scenarios
two machine a (5.5.5.11) and b (5.5.5.22), connected directly via a physical net interfaces (40G mellanox cx4).
test the ping cmd in machine b in following ways:  
1 ping the peer end of connection:

```text
$ ping ${physical_ip_of_a}
```

2 ping itself:

```text
$ ping ${self_ip}
```

3 ping localhost:

```text
$ ping ${ip_of_localhost}
```

finally, we examine the details of the underlying func callstack of ping by perf.

<hr>

### # cmd to record the callstack
1 use perf to sample on the ping localhost's callgraph (collecting 1-2min data):

```text
$ sudo perf record -g ping localhost
```

2 resolve & analysis on the data collected using flamegraph:

```text
$ sudo perf script | ${path_to_flamegraph}/stackcollapse-perf.pl | ${path_to_flamegraph}/FlameGraph/flamegraph.pl \
  > plocalhost.svg
```

<hr>

### # conclusion
rough process of ping is as: ping utility use sys_socket to create a socket, then use sys_sendto to send pkt,
use sys_recvmsg to receive pkt.
the kernel protocol stack does the main job for it.

<center><img src="/assets/images/2024/ping_callstack_1.png" width="100%"></center>

<center><img src="/assets/images/2024/ping_callstack_2.png" width="100%"></center>
