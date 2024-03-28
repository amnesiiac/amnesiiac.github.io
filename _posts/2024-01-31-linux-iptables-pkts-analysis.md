---
layout: post
title: "iptables pkts route analysis (linux, network)"
author: "melon"
date: 2024-01-31 22:12
categories: "2024"
tags:
  - linux
  - network
---

### # find out which iptables rule the pkts are hitting
monitor pkts counter changes that hitting certain iptable chain
```text
$ watch -n1 -d "iptables -t filter -vnxL | grep -vE 'pkts|Chain' | sort -nk1hr | column -t"
```
illustrate the above script word by word:
```text
1 watch -n1 -d: watch the content change with 1s as interval, -d option will highlight the changes between each frame.
  iptables -t filter -vnxL: monitor the filter tables pkts count changes.
2 grep -vE 'pkts|Chain': using reverse grep to get rid of pkts/chain title lines.
3 sort -nk1hr: sorts the remaining lines based on the first column in numeric order, the k1 specifies that the first
  column should be used as the sorting key, -h ensures that human-readable num (e.g., 1K) are sorted correctly.
```

this monitor script might cause too long content, which makes the interested part hard to be captured.
```text
$ watch -n1 -d "iptables -t filter -vnxL | grep -vE 'pkts|Chain' | head -n ${endline} | tail -n ${interval} | column -t"
```
the above script will monitor the pkts count changes for specific line range (certain chain of current table),
any variant on the numbers will be highlighted with reverse color.
```text
$ watch -n1 -d "iptables -t filter -vnxL | head -n 20 | tail -n 20 | column -t"
Every 1.0s: iptables -t filter -vnxL | head -n 20 | tail -n 20 | column -t                     Wed Feb 21 16:11:56 2024

Chain   INPUT     (policy         ACCEPT  400905  packets,        4260289042      bytes)
pkts    bytes     target          prot    opt     in              out             source           destination
0       0         ACCEPT          udp     --      virbr0          *               0.0.0.0/0        0.0.0.0/0 udp dpt:53
0       0         ACCEPT          tcp     --      virbr0          *               0.0.0.0/0        0.0.0.0/0 tcp dpt:53
0       0         ACCEPT          udp     --      virbr0          *               0.0.0.0/0        0.0.0.0/0 udp dpt:67
0       0         ACCEPT          tcp     --      virbr0          *               0.0.0.0/0        0.0.0.0/0 tcp dpt:67

Chain   FORWARD   (policy         ACCEPT  0       packets,        0               bytes)
pkts    bytes     target          prot    opt     in              out             source           destination
10      840       ACCEPT          all     --      br-149d732ffd90 br-4f0537e1b2cd 0.0.0.0/0        0.0.0.0/0
573     48132     ACCEPT          all     --      br-4f0537e1b2cd br-149d732ffd90 0.0.0.0/0        0.0.0.0/0
143447  29425498  DOCKER-USER     all     --      *               *               0.0.0.0/0        0.0.0.0/0
143447  29425498  DOCKER-ISOLA... all     --      *               *               0.0.0.0/0        0.0.0.0/0
994046  27180729  ACCEPT          all     --      *               docker0         0.0.0.0/0        0.0.0.0/0
35416   2122592   DOCKER          all     --      *               docker0         0.0.0.0/0        0.0.0.0/0
436302  27149606  ACCEPT          all     --      docker0         !docker0        0.0.0.0/0        0.0.0.0/0
0       0         ACCEPT          all     --      docker0         docker0         0.0.0.0/0        0.0.0.0/0
0       0         ACCEPT          all     --      *               virbr0          0.0.0.0/0        192.168.122.0/24
0       0         ACCEPT          all     --      virbr0          *               192.168.122.0/24 0.0.0.0/0
0       0         ACCEPT          all     --      virbr0          virbr0          0.0.0.0/0        0.0.0.0/0
```

<hr>

### # find out which iptables rules are responsible for dropping certain packets
(1) live monitoring  
before sending packets, print filter table every 1s, drop pkts chains lines, sort from high to low
```text
$ watch -n1 -d "iptables -nvxL -t filter | grep -vE 'pkts|Chains' | sort -nk1r"
```
then send the packets, the rules dropping the packets will incremented.

(2) method-2: diff comparation  
before sending the packets, record statics output:
```text
$ iptables -vnL > sample1
```

sending the packets, and it's dropped by certain rules, then snapshot the result:
```text
$ iptables -vnL > sample2
```

compare the snapshot statistics, the rules that drop the packets pkts count will get incremented
```text
$ diff sample1 sample2
```

(3) using iptables trace extension  
append a output rule to raw table, trace all packets output to ip/subnet:port using tcp protocol:
```text
$ iptables -t raw -A OUTPUT -p tcp --destination 192.168.0.0/24 --dport 80 -j TRACE
Warning: Extension TRACE revision 0 not supported, missing kernel module?
iptables: No chain/target/match by that name.
```
the output above means no necessary iptables extension installed, try:
```text
$ apt list | grep iptables-extension-trace
```
or can use to check the installation status:
```text
$ yum list | grep iptables-extensions
```
install the track extension by:
```text
$ sudo modprobe ip_conntrack_trace
```

after config & run, the output result(/var/log/kern.log):
```text
$ cat /var/log/kern.log | grep 'TRACE:'
Mar 24 22:41:52 enterprise kernel: [885386.325658] TRACE: raw:PREROUTING:policy:2 IN=eth0 OUT=
MAC=00:1d:7d:aa:e3:4e:00:04:4b:05:b4:dc:08:00 SRC=192.168.32.18 DST=192.168.12.152 LEN=52 TOS=0x00
PREC=0x00 TTL=128 ID=30561 DF PROTO=TCP SPT=53054 DPT=80 SEQ=3653700382 ACK=0 WINDOW=8192 RES=0x00
SYN URGP=0 OPT (020405B40103030201010402)

Mar 24 22:41:52 enterprise kernel: [885386.325689] TRACE: mangle:PREROUTING:policy:1 IN=eth0 OUT=
MAC=00:1d:7d:aa:e3:4e:00:04:4b:05:b4:dc:08:00 SRC=192.168.32.18 DST=192.168.12.152 LEN=52 TOS=0x00
PREC=0x00 TTL=128 ID=30561 DF PROTO=TCP SPT=53054 DPT=80 SEQ=3653700382 ACK=0 WINDOW=8192 RES=0x00
SYN URGP=0 OPT (020405B40103030201010402)
```

explain above output result word by word:
```text
raw:PREROUTING   Hitting the raw PREROUTING chain
policy:2         Matching rule #2
IN               incoming on eth0
SRC              From 192.168.32.18
DST              To port 80 on 192.168.12.152
PROTO            With TCP flags and options
```
