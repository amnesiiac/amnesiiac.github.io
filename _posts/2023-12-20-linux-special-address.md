---
layout: post
title: "special addresses (linux)"
author: "melon"
date: 2023-12-20 20:45
categories: "2023"
tags:
  - linux
  - network
  - ongoing
---

### # what is 0.0.0.0
ref: https://serverfault.com/a/78058

<hr>

### # what is localhost

<hr>

### # what is 127.0.0.1

<hr>

### # hostfw network connectivity check problem
check all sub container (netns) in wrapper container olt:
```text
$(olt) ip netns list
nt_2 (id: 1)
ihub_2 (id: 5)
ihub_1 (id: 6)
lt_1 (id: 2)
nt_1 (id: 3)
lt_2 (id: 4)
```

create & setup veth pair name (vethtest, vethtest_p):
```text
$(olt) ip link add vethtest type veth peer name vethtest_p
$(olt) ip link set vethtest up
$(olt) ip link set vethtest_p up
```

check the state of created veth pair just now
```text
$(olt) ip -c link | grep vethtest
101: vethtest_p@vethtest: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode
     DEFAULT group default qlen 1000
102: vethtest@vethtest_p: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode
     DEFAULT group default qlen 1000
```

put the peer into sub container netns
```text
$(olt) ip link set vethtest_p netns ihub_1 
```

check cur olt container, only one end left
```text
$(olt) ip -c link | grep vethtest
102: vethtest@if101: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state LOWERLAYERDOWN
     mode DEFAULT group default qlen 1000
```

send icmp echo to __any addr__ inside parent container olt:
```text
$(olt) ping -I vethtest 0.0.0.0
PING 0.0.0.0 (0.0.0.0): 56 data bytes
^C
--- 0.0.0.0 ping statistics ---
126 packets transmitted, 0 packets received, 100% packet loss
```

capture pkts inside sub container ihub_1, __nothing deliverd__:
```text
$(ihub_1) tcpdump -n -i vethtest_p -vvnnXX
tcpdump: listening on vethtest_p, link-type EN10MB (Ethernet), capture size 262144 bytes
^C
0 packets captured
0 packets received by filter
0 packets dropped by kernel
```

send icmp echo to __specific addr__ inside parent container olt:
```text
$(olt) ping -I vethtest 1.1.1.1
PING 1.1.1.1 (1.1.1.1): 56 data bytes
^C
--- 1.1.1.1 ping statistics ---
36 packets transmitted, 0 packets received, 100% packet loss
```

capture pkts inside sub container ihub_1, __only arp delivered__:
```text
$(ihub_1) tcpdump -n -i vethtest_p -vvnnXX
tcpdump: listening on vethtest_p, link-type EN10MB (Ethernet), capture size 262144 bytes
15:49:24.861552 ARP, Ethernet (len 6), IPv4 (len 4), Request who-has 1.1.1.1 tell 172.100.0.3, length 28
        0x0000:  ffff ffff ffff 2284 9daa 1db7 0806 0001  ......".........
        0x0010:  0800 0604 0001 2284 9daa 1db7 ac64 0003  ......"......d..
        0x0020:  0000 0000 0000 0101 0101                 ..........
15:49:25.882232 ARP, Ethernet (len 6), IPv4 (len 4), Request who-has 1.1.1.1 tell 172.100.0.3, length 28
        0x0000:  ffff ffff ffff 2284 9daa 1db7 0806 0001  ......".........
        0x0010:  0800 0604 0001 2284 9daa 1db7 ac64 0003  ......"......d..
        0x0020:  0000 0000 0000 0101 0101                 ..........
15:49:26.906233 ARP, Ethernet (len 6), IPv4 (len 4), Request who-has 1.1.1.1 tell 172.100.0.3, length 28
        0x0000:  ffff ffff ffff 2284 9daa 1db7 0806 0001  ......".........
        0x0010:  0800 0604 0001 2284 9daa 1db7 ac64 0003  ......"......d..
        0x0020:  0000 0000 0000 0101 0101                 ..........
^C
3 packets captured
3 packets received by filter
0 packets dropped by kernel
```

the major difference behaviour among ping 0.0.0.0 vs ping 1.1.1.1 is that:  
the root cause for that is: todo  
the ping inner most mechanism: todo on blog
