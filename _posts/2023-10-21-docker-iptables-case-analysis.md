---
layout: post
title: "docker iptables case analysis (docker & iptables)"
author: "melon"
date: 2023-10-21 11:22
categories: "2023"
tags:
  - container
  - network
---

this article focus the steps to buildup layer 3 network connection between two containers of different subnets,
each subnet is created & isolated using docker default bridge driver.

<hr>

### # experiment env setup
a) setup basic subnet & container instance, attach each container to their own subnet for isolation:

```text
$(host) docker network create testnet1                                      # create subnet1
$(host) docker network create testnet2                                      # create subnet2

$(host) docker network ls                                                   # validation
NETWORK ID     NAME       DRIVER    SCOPE
97392a75e5a1   bridge     bridge    local
11789d7cb259   host       host      local
8c7ae9ad8eb5   none       null      local
4f0537e1b2cd   testnet1   bridge    local                                   # *
149d732ffd90   testnet2   bridge    local                                   # *

$ docker run --rm -it --network testnet1 --name test1 ubuntu:22.04 bash     # container1 attached to subnet1
$ docker run --rm -it --network testnet2 --name test2 ubuntu:22.04 bash     # container2 attached to subnet2
```

b) show default config info of the subnet testnet1 & the corresponding container test1:

```text
$(host) docker inspect test1                                                # show container test1 info
 "Gateway": "172.18.0.1",
 "IPAddress": "172.18.0.2",
 "IPPrefixLen": 16,
 "IPv6Gateway": "",
 "GlobalIPv6Address": "",
 "GlobalIPv6PrefixLen": 0,
 "MacAddress": "02:42:ac:12:00:02",

$(host) docker network inspect testnet1                                     # show network testnet1 info
[
    {
        "Name": "testnet1",
        "Id": "4f0537e1b2cd9306a7ad93c0d5ed5ec68af107d00fe0cb13ce74604f9521245f",
        "Created": "2023-10-20T16:48:55.156821681+08:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.18.0.0/16",
                    "Gateway": "172.18.0.1"                                 # the gateway ip
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "707b84080324548d216ff8e191b5a0b412254bbd49d9706a7af08b45fd1f7e43": {
                "Name": "test1",
                "EndpointID": "5ec4f12b050ddd3a5c980bf8809140c0ea43f72641ef334688e1bde0647805e5",
                "MacAddress": "02:42:ac:12:00:02",
                "IPv4Address": "172.18.0.2/16",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {}
    }
]
```

c) show the docker gateway bridge info by its ip (for isolatation of the container test1):

```text
$(host) ip addr show | grep -C 5 172.18.0.1
148: br-4f0537e1b2cd: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:6b:73:de:a0 brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.1/16 brd 172.18.255.255 scope global br-4f0537e1b2cd
       valid_lft forever preferred_lft forever
    inet6 fe80::42:6bff:fe73:dea0/64 scope link
       valid_lft forever preferred_lft forever
```

d) show default config info of the subnet testnet2 & the corresponding container test2:

```text
$(host) docker inspect test2                                                # show container test2 info
 "Gateway": "172.19.0.1",
 "IPAddress": "172.19.0.2",
 "IPPrefixLen": 16,
 "IPv6Gateway": "",
 "GlobalIPv6Address": "",
 "GlobalIPv6PrefixLen": 0,
 "MacAddress": "02:42:ac:13:00:02",


$(host) docker network inspect testnet2                                     # show network testnet2 info
[
    {
        "Name": "testnet2",
        "Id": "149d732ffd90906dbdf149e4e8bc9bc30045cb5805301b96a2510fe253d9f7fe",
        "Created": "2023-10-20T16:50:43.396909905+08:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.19.0.0/16",
                    "Gateway": "172.19.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "53cf2cd678c42fdd02245879a8c33b72a21ec70709d201efdecc9278b13c4fed": {
                "Name": "test2",
                "EndpointID": "d691c134703b42902c9a6254eec9f706616ce4730dd3ba925572897596ed5d25",
                "MacAddress": "02:42:ac:13:00:02",
                "IPv4Address": "172.19.0.2/16",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {}
    }
]
```

e) show the docker gateway bridge info by its ip (for isolatation of the container test2):

```text
$(host) ip addr show | grep -C 5 172.19.0.1          # check the interface created by docker for container2
149: br-149d732ffd90: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:2e:d5:a3:ee brd ff:ff:ff:ff:ff:ff
    inet 172.19.0.1/16 brd 172.19.255.255 scope global br-149d732ffd90
       valid_lft forever preferred_lft forever
    inet6 fe80::42:2eff:fed5:a3ee/64 scope link
       valid_lft forever preferred_lft forever
```

f) confirm the ip_forward is enabled on host machine:

```text
$(host) cat /proc/sys/net/ipv4/ip_forward
1
```

<hr>

### # failed attempt: config the redirection iptables, but still no connection between
a) ping container test2 ip inside container test1, currently there's no iptable rules established
for packet redirection from testnet1 to testnet2 yet:

```text
$(container1) ping 172.19.0.2                           # ping container test2 from container test1
  PING 172.19.0.2 (172.19.0.2) 56(84) bytes of data.
  ^C
  --- 172.19.0.2 ping statistics ---
  81 packets transmitted, 0 received, 100% packet loss, time 81916ms

$(host) tcpdump -n -i br-4f0537e1b2cd -vvnnXX           # capture pkts on docker network test1, the icmp got catched
tcpdump: listening on br-4f0537e1b2cd, link-type EN10MB (Ethernet), capture size 262144 bytes
17:26:33.598357 IP (tos 0x0, ttl 64, id 13327, offset 0, flags [DF], proto ICMP (1), length 84)
    172.18.0.2 > 172.19.0.2: ICMP echo request, id 2, seq 1, length 64
        0x0000:  0242 6b73 dea0 0242 ac12 0002 0800 4500  .Bks...B......E.
        0x0010:  0054 340f 4000 4001 ae70 ac12 0002 ac13  .T4.@.@..p......
        0x0020:  0002 0800 055c 0002 0001 c947 3265 0000  .....\.....G2e..
        0x0030:  0000 2f21 0900 0000 0000 1011 1213 1415  ../!............
        0x0040:  1617 1819 1a1b 1c1d 1e1f 2021 2223 2425  ...........!"#$%
        0x0050:  2627 2829 2a2b 2c2d 2e2f 3031 3233 3435  &'()*+,-./012345
        0x0060:  3637                                     67
        ...

$(host) tcpdump -n -i br-149d732ffd90 -vvnnXX           # capture pkts on docker network test2, nothing got
tcpdump: listening on br-149d732ffd90, link-type EN10MB (Ethernet), capture size 262144 bytes
```

b) config iptables rules to enable layer2 connection between network testnet1 & testnet2:

```text
# accepts all traffic from br-4f0537e1b2cd to br-149d732ffd90
$(host) iptables -A FORWARD -i br-4f0537e1b2cd -o br-149d732ffd90 -j ACCEPT

# accepts all traffic from br-149d732ffd90 to br-4f0537e1b2cd from an existed & established connection
$(host) iptables -A FORWARD -i br-149d732ffd90 -o br-4f0537e1b2cd -m state --state RELATED,ESTABLISHED -j ACCEPT

# masquerade all traffic outbound from br-149d732ffd90
$(host) iptables -t nat -A POSTROUTING -o br-149d732ffd90 -j MASQUERADE
```

c) check the forward chain, the output show that the changes are applied at the bottom of this chain:

```text
$(host) iptables -nvL FORWARD
Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target                   prot opt in              out              source     destination
29710   30M DOCKER-USER              all  --  *               *                0.0.0.0/0  0.0.0.0/0
29710   30M DOCKER-ISOLATION-STAGE-1 all  --  *               *                0.0.0.0/0  0.0.0.0/0
    0     0 ACCEPT                   all  --  *               br-149d732ffd90  0.0.0.0/0  0.0.0.0/0   ctstate RELATED...
    0     0 DOCKER                   all  --  *               br-149d732ffd90  0.0.0.0/0  0.0.0.0/0
    0     0 ACCEPT                   all  --  br-149d732ffd90 !br-149d732ffd90 0.0.0.0/0  0.0.0.0/0
    0     0 ACCEPT                   all  --  br-149d732ffd90 br-149d732ffd90  0.0.0.0/0  0.0.0.0/0
13564   29M ACCEPT                   all  --  *               br-4f0537e1b2cd  0.0.0.0/0  0.0.0.0/0   ctstate RELATED...
    0     0 DOCKER                   all  --  *               br-4f0537e1b2cd  0.0.0.0/0  0.0.0.0/0
 9618  505K ACCEPT                   all  --  br-4f0537e1b2cd !br-4f0537e1b2cd 0.0.0.0/0  0.0.0.0/0
    0     0 ACCEPT                   all  --  br-4f0537e1b2cd br-4f0537e1b2cd  0.0.0.0/0  0.0.0.0/0
            ...                               ...
    0     0 ACCEPT                   all  --  br-4f0537e1b2cd br-149d732ffd90  0.0.0.0/0  0.0.0.0/0
    0     0 ACCEPT                   all  --  br-149d732ffd90 br-4f0537e1b2cd  0.0.0.0/0  0.0.0.0/0   state RELATED...
```

d) what's more, the masquerade rule is applied also:

```text
$(host) iptables -nvL -t nat
            ...
Chain POSTROUTING (policy ACCEPT 2 packets, 130 bytes)
 pkts bytes target                   prot opt in       out                source          destination
    0     0 MASQUERADE               all  --  *        !br-149d732ffd90   172.19.0.0/16   0.0.0.0/0
   15   922 MASQUERADE               all  --  *        !br-4f0537e1b2cd   172.18.0.0/16   0.0.0.0/0
12810  810K MASQUERADE               all  --  *        !docker0           172.17.0.0/16   0.0.0.0/0
            ...                                        ...
    0     0 MASQUERADE               all  --  *        br-149d732ffd90    0.0.0.0/0       0.0.0.0/0
            ...
```

e) once again, ping ip of container test2 inside container test1, however, still no traffic on subnet2:

```text
$(host) tcpdump -n -i br-149d732ffd90 -vvnnXX
tcpdump: listening on br-149d732ffd90, link-type EN10MB (Ethernet), capture size 262144 bytes
^c
0 packets captured
0 packets received by filter
0 packets dropped by kernel
```

<hr>

### # solution: -A vs -I, the difference?
a) convert the action from -A to -I, to insert the added rules on top of each chain:

```text
$(host) iptables -I FORWARD -i br-4f0537e1b2cd -o br-149d732ffd90 -j ACCEPT
```

once again, ping ip of container test2 inside container test1, capture pkt on container 2 bridge:

```text
$(host) tcpdump -n -i br-149d732ffd90 -vvnnXX
tcpdump: listening on br-149d732ffd90, link-type EN10MB (Ethernet), capture size 262144 bytes
19:15:12.058300 IP (tos 0x0, ttl 63, id 56621, offset 0, flags [DF], proto ICMP (1), length 84)
    172.19.0.1 > 172.19.0.2: ICMP echo request, id 3, seq 6121, length 64
        0x0000:  0242 ac13 0002 0242 2ed5 a3ee 0800 4500  .B.....B......E.
        0x0010:  0054 dd2d 4000 3f01 0652 ac13 0001 ac13  .T.-@.?..R......
        0x0020:  0002 0800 2297 0003 17e9 4061 3265 0000  ....".....@a2e..
        0x0030:  0000 8be3 0000 0000 0000 1011 1213 1415  ................
        0x0040:  1617 1819 1a1b 1c1d 1e1f 2021 2223 2425  ...........!"#$%
        0x0050:  2627 2829 2a2b 2c2d 2e2f 3031 3233 3435  &'()*+,-./012345
        0x0060:  3637                                     67
        ...
1 packets captured
1 packets received by filter
0 packets dropped by kernel
```

b) config the return pkt redirection rules from container test2 back to container test1:

```text
$(host) iptables -I FORWARD -i br-149d732ffd90 -o br-4f0537e1b2cd -m state --state RELATED,ESTABLISHED -j ACCEPT
```

ping container test2 inside container test1, the result shows the 2 added docker bridge connected:

```text
$(container1) ping 172.19.0.2
PING 172.19.0.2 (172.19.0.2) 56(84) bytes of data.
64 bytes from 172.19.0.2: icmp_seq=6308 ttl=63 time=0.091 ms
64 bytes from 172.19.0.2: icmp_seq=6309 ttl=63 time=0.076 ms
64 bytes from 172.19.0.2: icmp_seq=6310 ttl=63 time=0.108 ms
...
```

<hr>

### # root cause: iptables -I option for setting the newly-added rules matched at the highest priority
if use -A option to append new rules, by default the DOCKER-ISOLATION-STAGED-1 is hitted, rather than the
rule added at bottom, thus the newly added rules wont take effect when pkt arrived:

```text
$ iptables -nvL FORWARD
Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target                   prot opt in              out              source    destination
30601   30M DOCKER-USER              all  --  *               *                0.0.0.0/0 0.0.0.0/0  <- hit this, go next
30601   30M DOCKER-ISOLATION-STAGE-1 all  --  *               *                0.0.0.0/0 0.0.0.0/0  <- hit this,
            ...
    0     0 ACCEPT                   all  --  br-4f0537e1b2cd br-149d732ffd90  0.0.0.0/0 0.0.0.0/0  <- newly add
    0     0 ACCEPT                   all  --  br-149d732ffd90 br-4f0537e1b2cd  0.0.0.0/0 0.0.0.0/0  <- newly add

$ iptables -nvL DOCKER-ISOLATION-STAGE-1    # check the isolation chain added by docker
Chain DOCKER-ISOLATION-STAGE-1 (1 references)
 pkts bytes target                   prot opt in              out              source    destination
  218 18312 DOCKER-ISOLATION-STAGE-2 all  --  br-149d732ffd90 !br-149d732ffd90 0.0.0.0/0 0.0.0.0/0  <- for isolation
15803 1025K DOCKER-ISOLATION-STAGE-2 all  --  br-4f0537e1b2cd !br-4f0537e1b2cd 0.0.0.0/0 0.0.0.0/0
 743K   65M DOCKER-ISOLATION-STAGE-2 all  --  docker0         !docker0         0.0.0.0/0 0.0.0.0/0
2172K 5807M RETURN                   all  --  *               *                0.0.0.0/0 0.0.0.0/0
```

something off topic about the isolation rules added by docker engine: the purpose of the extra isolation stage
is to forward any traffic into container1 to DOCKER-ISOLATION-STAGE-2, used to isolate the container out of the
unwanted traffic.
