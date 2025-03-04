---
layout: post
title: "introduction (tcp-ip vol 1)"
author: "melon"
date: 2021-08-02 21:53
categories: "2021"
tags:
  - network
  - todo
---

### # osi 7 layer model

```txt
 ┌─────────────────────────────────────────────────┐   ┌─────────────────────────────────┐
 │ 1 application layer                             │   │                                 │
 │ human interaction layer,                        │   │                                 │
 │ app can access the network                      │   │                                 │
 ├─────────────────────────────────────────────────┤   │                                 │
 │ 2 presentation layer                            │   │ 1 application layer             │
 │ ensure data in usable format &                  │   │ telnet, ssh, http, smtp, ftp,   │
 │ where data encryption occurs                    │   │ ssl/tls, html ,snmp...          │
 ├─────────────────────────────────────────────────┤   │                                 │
 │ 3 session layer                                 │   │                                 │
 │ maintain connections &&                         │   │                                 │
 │ responsible for port & session controlling      │   │                                 │
 ├─────────────────────────────────────────────────┤   ├─────────────────────────────────┤
 │ 4 transport layer                               │   │ 2 transport layer               │
 │ transmit data by protocols like tcp/udp         │   │ tcp, udp, sctp, dccp...         │
 ├─────────────────────────────────────────────────┤   ├─────────────────────────────────┤
 │ 5 network layer                                 │   │ 3 network layer                 │
 │ decide which physical path to take              │   │ arp, ipv4, ipv6, icmp, ipsec... │
 ├─────────────────────────────────────────────────┤   ├─────────────────────────────────┤
 │ 6 data-link layer                               │   │                                 │
 │ define the data format on the network           │   │ 4 link layer                    │
 ├─────────────────────────────────────────────────┤   │ ethernet, wlan, ppp...          │
 │ 7 physical layer                                │   │                                 │
 │ transmit raw bit stream over the physical media │   │                                 │
 └─────────────────────────────────────────────────┘   └─────────────────────────────────┘
```

<hr>

### # encapsulation process of tcp packet (network proto stack)

```txt
                                                        ┌──────┐
                                                        │ data │
                                                        └──────┘                  ┌─────────────┐
                                                               +------------------│ application │
                                            ┌───────────┬──────┐                  └─────────────┘
                                            │   header  │ data │
                                            └───────────┴──────┘                  ┌───────────┐
                                                               +------------------│ tcp layer │
                               ┌────────────┬──────────────────┐                  └───────────┘
                               │ tcp header │ application data │
                               └────────────┴──────────────────┘
                               ├───────────────────────────────┤
                                         tcp segment                              ┌──────────┐
                                                               +------------------│ ip layer │
                   ┌───────────┬────────────┬──────────────────┐                  └──────────┘
                   │ ip header │ tcp header │ application data │
                   └───────────┴────────────┴──────────────────┘
                   ├───────────────────────────────────────────┤
                                    ip datagram                                   ┌─────────────────┐
                                                               +------------------│ ethernet driver │
 ┌─────────────────┬───────────┬────────────┬──────────────────┬───────────────┐  └─────────────────┘
 │ ethernet header │ ip header │ tcp header │ application data │ ethernet tail │
 └─────────────────┴───────────┴────────────┴──────────────────┴───────────────┘
         14             20           20          6 - 1460              4        
 ├─────────────────────────────────────────────────────────────────────────────┤
                      ethernet frame (64 byte ~ 1518 byte)
```
