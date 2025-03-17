---
layout: post
title: "ping underlying callstack in various application scenarios (linux, network, tool)"
author: "melon"
date: 2024-01-10 22:57
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

1 ping peer callstack

<center><img src="/assets/images/2024/ping_callstack_1.png" width="70%"></center>

packet transmit path of ping (syscall into proto stack path):

<center><img src="/assets/images/2024/ping_callstack_2.png" width="70%"></center>

the syscall start from sys_sendto, end with mlx5e_sq_xmit of certain physical netcard.

```txt
host b                                                     host a
ping
sys_sendto
sock_sendmsg
ip_finish_output
dev_hard_start_xmit
mlx5e_sq_xmit
                           ----icmp request--->

sleep on skb receive (on behalf of ping thread in kernel)
                                                           handling
                           <---icmp reply------
mlx driver recv packet
wake up recv thread
skb_recv_datagram
sock_recvmsg
sys_recvmsg (wake up ping thread)
ping receive packet
host b                                                     host a
```

2 ping self callstack

<center><img src="/assets/images/2024/ping_callstack_3.png" width="70%"></center>

core callstack path is as:

<center><img src="/assets/images/2024/ping_callstack_4.png" width="70%"></center>

which also start from sys_sendto, however, when callstack reach ip_finish_output2, it will skip left process,
goto icmp req recv process directly (proto stack know the packet is for itself).

the ping self use device lo corresponding driver to recv pkt (standard linux pkt recv process):

```txt
host b ping                                         host b kernel
ping
sys_sendto
sock_sendmsg
ip_finish_output
                                ---icmp request--->
                                                    do_softirq
                                                    ip_rcv
                                                    icmp_rcv
                                                    icmp_echo
                                                              (go through ip send process again)
                                                    ip_output
                                                    dev_hard_start_xmit
                                                          (send pkt through loopback)
                                                    loopback_xmit
                                <---icmp reply-----
sleep on skb receive (on behalf of ping thread in kernel)
loopback driver recv pkt
wake up recv thread
skb_recv_datagram
sock_recvmsg
sys_recvmsg (wakeup ping thread)
ping recv pkt
host b ping                                         host b kernel
```

3 ping localhost

<center><img src="/assets/images/2024/ping_callstack_5.png" width="70%"></center>

<hr>

### # conclusion
1 ping peer, ping self, ping localhost roughly has same pkt recv callstack: ping invoke sys_recvmsg, and then
kernel waiting on skb_recv_datagram (give up cpu); while the nic driver ctx to handle the received pkt might
run on other cpu, which is out of reach of perf (perf sample by certain thread num).  
2 ping peer operation will actually use the nic.  
3 ping self & ping localhost roughly has the same send process. while the pkt is not actually sent, the pkt
will goto recv process at ip stack, and finally icmp-reply send by localhost dev.

<hr>

### # latency

ping    | latency (ms)   | delta
---     | ---            | ---
local   | 0.077          | 0
peer    | 0.134          | 0.057

time delta between ping peer & ping local is 0.057ms, the reason can be concluded as: ping peer need 4
times physical nic driver handling then ping local (host b nic send, host a nic recv, host a nic send,
host b nic recv).
thus the time delta lies in nic transmission and drv pkt handling.
