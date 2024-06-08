---
layout: post
title: "special addresses (linux)"
author: "melon"
date: 2023-12-20 20:45
categories: "2023"
tags:
  - linux
  - network
  - todo
---

0.0.0.0 typically means anywhere, 127.0.0.1 means this very machine.

<hr>

### # listening on 0.0.0.0 vs listening on 127.0.0.1
listening on 0.0.0.0 means listening from anywhere that
has network access to this computer, e.g., from this very computer, from
local network or from the internet.  
while listening on 127.0.0.1 means only listen from this very computer.

<hr>

### # localhost vs 127.0.0.1
localhost is the machine name or domain name of the host server,
which allows a network connection to loop back on itself.
on almost every machine, the localhost and 127.0.0.1 are functionally the same.

1 the difference between localhost & 127.0.0.1  
1) the localhost is proto indenpendant, both for ipv4 and ipv6.  
2) the localhost tend to be slower than 127.0.0.1 cause the ip resolution of
localhost need to go through a DNS lookup table.

<p style="margin-bottom: 20px;"></p>

2 major config files for DNS host lookup  
1) /etc/hosts is the host lookup table:
```text
$ cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

192.168.0.25 nesc-allinone
135.251.206.229 asblx29
```

2) /etc/host.conf is the central file that controls the resolver setup.
it tells the resolver which services to use, and in what order.
more options please ref: https://tldp.org/LDP/nag/node82.html.
```text
$ cat /etc/host.conf
multi on
```

3) /etc/hostname is for domain definition for host machine:
```text
$ cat /etc/hostname
spine.novalocal
```

<hr>

### # hostfw network connectivity check problem
1 buildup the testcase env:  
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

create veth pair named vethtest, vethtest_p:
```text
$(olt) ip link add vethtest type veth peer name vethtest_p
$(olt) ip link set vethtest up
$(olt) ip link set vethtest_p up
```

check the state of the created veth pair:
```text
$(olt) ip -c link | grep vethtest
101: vethtest_p@vethtest: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode
     DEFAULT group default qlen 1000
102: vethtest@vethtest_p: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode
     DEFAULT group default qlen 1000
```

put the peer into certain sub container netns:
```text
$(olt) ip link set vethtest_p netns ihub_1 

$(olt) ip -c link | grep vethtest   # only one end left in current olt container
102: vethtest@if101: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state LOWERLAYERDOWN
     mode DEFAULT group default qlen 1000
```

<p style="margin-bottom: 20px;"></p>

2 test the traffic connectivity with 0.0.0.0  
send icmp echo to any addr inside olt container from itf vethtest:
```text
$(olt) ping -I vethtest 0.0.0.0
PING 0.0.0.0 (0.0.0.0): 56 data bytes
^C
--- 0.0.0.0 ping statistics ---
126 packets transmitted, 0 packets received, 100% packet loss
```

capture pkts inside sub container ihub_1, nothing deliverd:
```text
$(ihub_1) tcpdump -n -i vethtest_p -vvnnXX
tcpdump: listening on vethtest_p, link-type EN10MB (Ethernet), capture size 262144 bytes
^C
0 packets captured
0 packets received by filter
0 packets dropped by kernel
```

<p style="margin-bottom: 20px;"></p>

3 test the traffic connectivity with 1.1.1.1  
send icmp echo to specific addr inside olt container from itf vethtest:
```text
$(olt) ping -I vethtest 1.1.1.1
PING 1.1.1.1 (1.1.1.1): 56 data bytes
^C
--- 1.1.1.1 ping statistics ---
36 packets transmitted, 0 packets received, 100% packet loss
```

capture pkts inside sub container ihub_1, only arp delivered:
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

<hr>

### # todo analysis
1 the major difference behaviour among ping 0.0.0.0 vs ping 1.1.1.1 is that:  
2 the root cause for that is: todo  
3 the ping operation flow
