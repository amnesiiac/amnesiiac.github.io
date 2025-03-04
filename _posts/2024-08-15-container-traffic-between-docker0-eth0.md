---
layout: post
title: "container traffic flow between docker0 & eth0 (container, netfilter)"
author: "melon"
date: 2024-08-15 16:31
categories: "2024"
tags:
  - container
  - network
  - todo
---

this article focus on how the following traffic flow path get established:

```text
container -> docker0 -> eth0 -> outside world
```

<hr>

### # combine this with the next section
1 when a packet flows from container to docker0, how does it know it will be forwarded to eth0,
and then to the outside world?

docker uses table NAT chain MASQUERADE to process outbound traffic from containers, and next the pkts will follow
the standard outbound routing on host machine, which may or may not involve eth0.

```text
$ iptables -t nat -vnL POSTROUTING
Chain POSTROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts  bytes  target        prot opt in    out         source           destination
    0      0  MASQUERADE    all  --  *     !docker0    172.17.0.0/16    0.0.0.0/0 
```

<hr>

### # iptable rules path (docker0 <─> eth0) on the host machine
ongoing\... further cleaning up the iptables path between docker0 & eth0, pending to adding all necessary
iptables rules in here.

```text
$ iptables -L -v
Chain INPUT (policy ACCEPT 17414 packets, 4889K bytes)
 pkts bytes target     prot opt in     out    source      destination
    0     0 ACCEPT     udp  --  virbr0 any    anywhere    anywhere             udp dpt:domain
    0     0 ACCEPT     tcp  --  virbr0 any    anywhere    anywhere             tcp dpt:domain
    0     0 ACCEPT     udp  --  virbr0 any    anywhere    anywhere             udp dpt:bootps
    0     0 ACCEPT     tcp  --  virbr0 any    anywhere    anywhere             tcp dpt:bootps
```

```text
Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target                    prot opt in               out              source            destination
  52M   83G DOCKER-USER               all  --  any              any              anywhere          anywhere     // 1
  52M   83G DOCKER-ISOLATION-STAGE-1  all  --  any              any              anywhere          anywhere     // 3
  35M   81G ACCEPT                    all  --  any              docker0          anywhere          anywhere
16551  992K DOCKER                    all  --  any              docker0          anywhere          anywhere
  18M 2010M ACCEPT                    all  --  docker0         !docker0          anywhere          anywhere
    0     0 ACCEPT                    all  --  docker0          docker0          anywhere          anywhere
    0     0 ACCEPT                    all  --  any              br-37105304fb02  anywhere          anywhere
    0     0 DOCKER                    all  --  any              br-37105304fb02  anywhere          anywhere
    0     0 ACCEPT                    all  --  br-37105304fb02 !br-37105304fb02  anywhere          anywhere
    0     0 ACCEPT                    all  --  br-37105304fb02  br-37105304fb02  anywhere          anywhere
    0     0 ACCEPT                    all  --  any              virbr0           anywhere          192.168.122.0/24
    0     0 ACCEPT                    all  --  virbr0           any              192.168.122.0/24  anywhere
    0     0 ACCEPT                    all  --  virbr0           virbr0           anywhere          anywhere
    0     0 REJECT                    all  --  any              virbr0           anywhere          anywhere
    0     0 REJECT                    all  --  virbr0           any              anywhere          anywhere
```

```text
Chain OUTPUT (policy ACCEPT 14898 packets, 2458K bytes)
 pkts bytes target     prot opt in     out     source               destination
    0     0 ACCEPT     udp  --  any    virbr0  anywhere             anywhere             udp dpt:bootpc
```

docker chain: all pkt from any itf other than docker0 to reach the 'gateway' interface of docker0, from any src
to certain container ip:dpt, will be forwarded to ACCEPT chain accordingly:

```text
Chain DOCKER (2 references)
 pkts bytes target     prot opt in                out              source    destination
    0     0 ACCEPT     tcp  --  !br-37105304fb02  br-37105304fb02  anywhere  172.20.0.2   tcp dpt:http
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.2   tcp dpt:vmware-fdm
   84  5040 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.3   tcp dpt:tungsten-https
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.3   tcp dpt:irdmi
  216 11936 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.5   tcp dpt:EtherNet/IP-1
   13   756 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.6   tcp dpt:ddi-tcp-1
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.27  tcp dpt:61000
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.27  tcp dpt:10022
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.27  tcp dpt:4222
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.27  tcp dpt:tnp1-port
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.27  tcp dpt:semaphore
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.27  tcp dpt:down
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.27  tcp dpt:tunstall-pnc
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.27  tcp dpt:exp2
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.4   tcp dpt:62003
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.4   tcp dpt:62002
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.4   tcp dpt:62001
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.4   tcp dpt:62000
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.4   tcp dpt:16022
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.4   tcp dpt:15022
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.4   tcp dpt:14022
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.4   tcp dpt:13022
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.4   tcp dpt:10222
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.4   tcp dpt:10024
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.4   tcp dpt:teamcoherence
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.4   tcp dpt:swa-2
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.4   tcp dpt:8222
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.4   tcp dpt:8024
    4   240 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.4   tcp dpt:7222
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.4   tcp dpt:vmsvc
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.4   tcp dpt:semaphore
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.4   tcp dpt:silkp2
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.4   tcp dpt:down
   27  1620 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.4   tcp dpt:slp-notify
    7   420 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.4   tcp dpt:tunstall-pnc
   17  1020 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.4   tcp dpt:altalink
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.4   tcp dpt:exp2
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.8   tcp dpt:62003
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.8   tcp dpt:62002
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.8   tcp dpt:62001
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.8   tcp dpt:62000
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.8   tcp dpt:16022
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.8   tcp dpt:15022
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.8   tcp dpt:14022
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.8   tcp dpt:13022
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.8   tcp dpt:10222
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.8   tcp dpt:10024
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.8   tcp dpt:teamcoherence
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.8   tcp dpt:swa-2
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.8   tcp dpt:8222
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.8   tcp dpt:8024
    4   240 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.8   tcp dpt:7222
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.8   tcp dpt:vmsvc
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.8   tcp dpt:6022
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.8   tcp dpt:mice
    3   180 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.8   tcp dpt:4830
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.8   tcp dpt:dnox
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.8   tcp dpt:cernsysmgmtagt
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.8   tcp dpt:semaphore
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.8   tcp dpt:csregagent
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.8   tcp dpt:fjdocdist
    5   300 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.8   tcp dpt:slp-notify
    5   300 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.8   tcp dpt:tunstall-pnc
    5   300 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.8   tcp dpt:altalink
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.10  tcp dpt:10022
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.10  tcp dpt:4222
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.10  tcp dpt:tnp1-port
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.10  tcp dpt:semaphore
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.10  tcp dpt:down
 7768  466K ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.10  tcp dpt:tunstall-pnc
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.10  tcp dpt:exp2
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.12  tcp dpt:10022
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.12  tcp dpt:4222
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.12  tcp dpt:tnp1-port
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.12  tcp dpt:semaphore
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.12  tcp dpt:down
 7766  466K ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.12  tcp dpt:tunstall-pnc
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.12  tcp dpt:exp2
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.7   tcp dpt:12022
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.7   tcp dpt:11022
    4   240 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.7   tcp dpt:radmind
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.7   tcp dpt:6024
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.7   tcp dpt:xmpp-client
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.7   tcp dpt:scpi-telnet
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.7   tcp dpt:semaphore
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.7   tcp dpt:csregagent
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.7   tcp dpt:down
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.7   tcp dpt:fjdocdist
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.7   tcp dpt:slp-notify
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.7   tcp dpt:tunstall-pnc
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.13  tcp dpt:10022
   18  1080 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.13  tcp dpt:4222
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.13  tcp dpt:tnp1-port
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.13  tcp dpt:semaphore
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.13  tcp dpt:down
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.13  tcp dpt:tunstall-pnc
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.13  tcp dpt:exp2
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.11  tcp dpt:62003
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.11  tcp dpt:62002
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.11  tcp dpt:62001
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.11  tcp dpt:62000
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.11  tcp dpt:16022
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.11  tcp dpt:15022
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.11  tcp dpt:14022
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.11  tcp dpt:13022
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.11  tcp dpt:10222
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.11  tcp dpt:10024
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.11  tcp dpt:teamcoherence
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.11  tcp dpt:swa-2
    1    60 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.11  tcp dpt:8222
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.11  tcp dpt:8024
    8   480 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.11  tcp dpt:7222
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.11  tcp dpt:vmsvc
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.11  tcp dpt:6022
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.11  tcp dpt:mice
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.11  tcp dpt:4830
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.11  tcp dpt:dnox
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.11  tcp dpt:cernsysmgmtagt
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.11  tcp dpt:semaphore
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.11  tcp dpt:csregagent
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.11  tcp dpt:fjdocdist
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.11  tcp dpt:slp-notify
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.11  tcp dpt:tunstall-pnc
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.11  tcp dpt:altalink
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.9   tcp dpt:10022
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.9   tcp dpt:4222
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.9   tcp dpt:tnp1-port
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.9   tcp dpt:down
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.9   tcp dpt:tunstall-pnc
    0     0 ACCEPT     tcp  --  !docker0          docker0          anywhere  172.17.0.9   tcp dpt:exp2
```

docker isolation chain: classic rules to isolate traffic between 2 docker subnet interfaces (docker0, br-xxx):

```text
Chain DOCKER-ISOLATION-STAGE-1 (1 references)
 pkts bytes target                     prot opt in               out              source    destination
  18M 2010M DOCKER-ISOLATION-STAGE-2   all  --  docker0         !docker0          anywhere  anywhere     // 4
    0     0 DOCKER-ISOLATION-STAGE-2   all  --  br-37105304fb02 !br-37105304fb02  anywhere  anywhere     // 4'
  52M   83G RETURN                     all  --  any              any              anywhere  anywhere     // 7

Chain DOCKER-ISOLATION-STAGE-2 (2 references)
 pkts bytes target     prot opt in     out              source    destination
    0     0 DROP       all  --  any    docker0          anywhere  anywhere       // 5
    0     0 DROP       all  --  any    br-37105304fb02  anywhere  anywhere       // 5'
  18M 2010M RETURN     all  --  any    any              anywhere  anywhere       // 6
```

```text
Chain DOCKER-USER (1 references)
 pkts bytes target     prot opt in     out     source               destination
  52M   83G RETURN     all  --  any    any     anywhere             anywhere     // 2
```

<hr>

### # route path (docker0 <─> eth0) on the host machine
1 show configured ip route rules on host machine:

```text
$(host) ip route show | column '-t'
default           via  192.168.0.1      dev    eth0    proto  dhcp  metric  100                            // 1
169.254.169.254   via  192.168.0.1      dev    eth0    proto  dhcp  metric  100                            // 2
172.17.0.0/16     dev  docker0          proto  kernel  scope  link  src     172.17.0.1                     // 3
172.20.0.0/16     dev  br-37105304fb02  proto  kernel  scope  link  src     172.20.0.1                     // 4
192.168.0.0/24    dev  eth0             proto  kernel  scope  link  src     192.168.0.20   metric    100
192.168.122.0/24  dev  virbr0           proto  kernel  scope  link  src     192.168.122.1  linkdown
```

2 show pretty route rules use routel tool (iproute2):

```text
$(host) routel
           target            gateway          source    proto   scope       dev             tbl
          default        192.168.0.1                     dhcp              eth0                            // 1
  169.254.169.254        192.168.0.1                     dhcp              eth0                            // 2
    172.17.0.0/16                         172.17.0.1   kernel    link   docker0                            // 3
    172.20.0.0/16                         172.20.0.1   kernel    link   br-37105304fb02                    // 4
   192.168.0.0/24                       192.168.0.20   kernel    link      eth0
 192.168.122.0/24                      192.168.122.1   kernel    link    virbr0
        127.0.0.0          broadcast       127.0.0.1   kernel    link        lo           local
      127.0.0.0/8              local       127.0.0.1   kernel    host        lo           local
        127.0.0.1              local       127.0.0.1   kernel    host        lo           local
  127.255.255.255          broadcast       127.0.0.1   kernel    link        lo           local
       172.17.0.0          broadcast      172.17.0.1   kernel    link   docker0           local
       172.17.0.1              local      172.17.0.1   kernel    host   docker0           local
   172.17.255.255          broadcast      172.17.0.1   kernel    link   docker0           local
       172.20.0.0          broadcast      172.20.0.1   kernel    link   br-37105304fb02   local
       172.20.0.1              local      172.20.0.1   kernel    host   br-37105304fb02   local
   172.20.255.255          broadcast      172.20.0.1   kernel    link   br-37105304fb02   local
      192.168.0.0          broadcast    192.168.0.20   kernel    link      eth0           local
     192.168.0.20              local    192.168.0.20   kernel    host      eth0           local
    192.168.0.255          broadcast    192.168.0.20   kernel    link      eth0           local
    192.168.122.0          broadcast   192.168.122.1   kernel    link    virbr0           local
    192.168.122.1              local   192.168.122.1   kernel    host    virbr0           local
  192.168.122.255          broadcast   192.168.122.1   kernel    link    virbr0           local
...
```

the explanations of rules related to docker0 & eth0 are as follows:  
a) for route rule 1, all traffic without specified rule will match this rule: redirect the pkt to gateway
(192.168.0.1) by interface eth0; proto dhcp: current rule is added to route table by dhcp.  
b) for route rule 3, all traffic targeted to subnet 172.17.0.0/16 will be redirect to docker0 interface,
with pkt src ip masked as 172.17.0.1 (docker0).  
c) for route rule 4, all traffic targeted to subnet 172.20.0.0/16 will be redirect to br-37105304fb02 interface,
with pkt src ip masked as 172.20.0.1 (br-37105304fb02).

<hr>

### # pkt route path from the view inside container
1 prepare container env, and show ip route info:

```text
$ docker run -it --rm alpine sh
$(container) apk update && apk add iproute2 && apk add python3

$(container) ip route show | column '-t'
$(container) ip route show
default        via  172.17.0.1  dev    eth0                                                  // 1
172.17.0.0/16  dev  eth0        proto  kernel scope link src 172.17.0.15                     // 2

$(container) routel
Dst             Gateway         Prefsrc         Protocol Scope   Dev              Table
default         172.17.0.1                                       eth0                        // 1
172.17.0.0/16                   172.17.0.15     kernel   link    eth0                        // 2
127.0.0.0                       127.0.0.1       kernel   link    lo               local
127.0.0.0/8                     127.0.0.1       kernel   host    lo               local
127.0.0.1                       127.0.0.1       kernel   host    lo               local
127.255.255.255                 127.0.0.1       kernel   link    lo               local
172.17.0.0                      172.17.0.15     kernel   link    eth0             local
172.17.0.15                     172.17.0.15     kernel   host    eth0             local
172.17.255.255                  172.17.0.15     kernel   link    eth0             local
```

some explanations about the above information derived:  
a) for route rule 1, all pkt without specified rule will match this rule: redirect the pkt to gateway
(172.17.0.1) by interface eth0.  
b) for route rule 2, all pkt dst to 172.17.0.0/16 subnet will be send through the container's default interface
eth0, with preferred src set as 172.17.0.15 (eth0).

2 check interface info inside the container:

```text
$(container) ifconfig
eth0    Link encap:Ethernet  HWaddr 02:42:AC:11:00:0F
        inet addr:172.17.0.15  Bcast:172.17.255.255  Mask:255.255.0.0
        UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
        RX packets:3099 errors:0 dropped:0 overruns:0 frame:0
        TX packets:1729 errors:0 dropped:0 overruns:0 carrier:0
        collisions:0 txqueuelen:0
        RX bytes:18737031 (17.8 MiB)  TX bytes:124904 (121.9 KiB)

lo      Link encap:Local Loopback
        inet addr:127.0.0.1  Mask:255.0.0.0
        UP LOOPBACK RUNNING  MTU:65536  Metric:1
        RX packets:0 errors:0 dropped:0 overruns:0 frame:0
        TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
        collisions:0 txqueuelen:1000
        RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```

<hr>

### # examine the traffic connection between docker network bridge & containers by veth
1 check the bridge & interface attachement info:

```text
$ brctl show
bridge name         bridge id               STP enabled     interfaces
br-37105304fb02     8000.02422aba30a5       no              veth8dc8d4c    // 2 extra customized bridge
docker0             8000.024228c009a4       no              veth029c8ed    // 1 docker default bridge
                                                            veth0360bfc
                                                            veth2afe731
                                                            veth3a3f891
                                                            veth55c7e18
                                                            veth57b3f76
                                                            veth638def7
                                                            veth780339f
                                                            vethac3d6d5
                                                            vethb442da2
                                                            vethb46e47b
                                                            vethc484df6
                                                            vethe792b6d
                                                            vethf7d53ac
virbr0              8000.5254000009d1       yes             virbr0-nic
```

2 except for the default bridge docker0 created by docker, there's extra docker network driver enabled:

```text
$ ifconfig br-37105304fb02
br-37105304fb02: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
    inet 172.20.0.1  netmask 255.255.0.0  broadcast 172.20.255.255
    inet6 fe80::42:2aff:feba:30a5  prefixlen 64  scopeid 0x20<link>
    ether 02:42:2a:ba:30:a5  txqueuelen 0  (Ethernet)
    RX packets 0  bytes 0 (0.0 B)
    RX errors 0  dropped 0  overruns 0  frame 0
    TX packets 0  bytes 0 (0.0 B)
    TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

3 check all configured docker network driver:

```text
$ docker network ls
NETWORK ID     NAME             DRIVER    SCOPE
44e9126b7756   bridge           bridge    local      // 1 default docker bridge
11789d7cb259   host             host      local
37105304fb02   mirror_default   bridge    local      // 2 extra customized bridge
8c7ae9ad8eb5   none             null      local
```

4 inspect the extra customized bridge info:

```text
$ docker network inspect mirror_default
[
    {
        "Name": "mirror_default",
        "Id": "37105304fb02b799db9482b9d5cf2ede66c5dd7e80ff8308c1a86b88b6e6ca84",
        "Created": "2024-03-13T23:34:22.867774399+08:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.20.0.0/16",
                    "Gateway": "172.20.0.1"           // * (this ip is the br-xxx)
                }
            ]
        },
        "Internal": false,
        "Attachable": true,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "3d3f72aa1633ad5313e373cf2c222a11a1ad488336d22f1ebfe51659fbeb98ce": {
                "Name": "alpine_fileserver",
                "EndpointID": "185d65f0ad724739c2cbf5b6bdb265b174fdd576102775e7ac3f8e5d44f4a865",
                "MacAddress": "02:42:ac:14:00:02",
                "IPv4Address": "172.20.0.2/16",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {
            "com.docker.compose.network": "default",
            "com.docker.compose.project": "mirror",
            "com.docker.compose.version": "1.29.0"
        }
    }
]
```
