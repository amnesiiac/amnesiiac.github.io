---
layout: post
title: "iptables & netfilter (linux, network)"
author: "melon"
date: 2024-01-31 23:12
categories: "2024"
tags:
  - linux
  - network
---

### # introduction to netfilter
todo

<hr>

### # introduction to iptables
iptables, ebtables, arptables are applications based on netfilter.  
iptables are so-called linux original firewall, but actually the function of iptables
are more than a firewall can do: it is a ip packet filter tool integrating most
of the functionality of netfilter have.

<hr>

### # why we need iptables out of netfilter
the functionality of kernel hook function of netfiler is powerfull, but somehow too heavy
(coding) for simple packet path controling.  
so iptables rules are simply to enable a cmd-like way to config the same job as netfilter.

the iptables works in usespace to handle the rules, and the netfilter hook functions will
handle the packet path accordingly.

<hr>

### # relationship between iptables & netfilter 
```txt

         ┌─────────┐             ┌──────────┐
         │ syscall │             │ iptables │
         └────┬────┘             └──+────┬──┘    userspace
──────────────┼─────────────────────┼────┼──────────────
              │           getsockopt│    │setsockopt
  ┌───────────+─────────────────────┴────+──────────┐
  │                kernel interfaces                │
  └───────────+────────────────────────+────────────┘
              │                        │
         ┌────+────┐       ┌───────────+────────────┐
         │ tcp/udp │       │ iptables kernel module │
         └────+────┘       └───────────┬────────────┘
              │                        │      kernel space
         ┌────+────┐  hook func  ┌─────+─────┐
         │   ip    │<----------->│ netfilter │
         └────┬────┘             └───────────┘
              │
         ┌────+────┐
         │ net itf │
         └────┬────┘
              │
──────────────┼───────────────────────────────────────────
              │                               driver layer
      ┌───────+───────┐
      │ device driver │
      └───────────────┘
```

<hr>

### # build-in tables & chains in iptables
the iptables rules cover 3 level of concepts: tables, chain and rules.

different tables are generally used for various purposes:
e.g. NAT is used for handling network address translation (ip addr translation, port forwarding, port mapping\...),
FILTER table is used for judging whether packets can be passed ro filtered out.

the inner rules combination within each table is form into a chain, in which all the packets
passed in certain table get matched & handled in turn.

<hr>

### # functionalities of build-in chain of iptables table
the four build-in table in iptables:
```txt
      ┌────────────┐   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
      │ raw        │   │ mangle      │   │ nat         │   │ filter      │
      │┌──────────┐│   │┌───────────┐│   │┌───────────┐│   │┌───────────┐│
      ││PREROUTING││   ││PREROUTING ││   ││PREROUTING ││   ││INPUT      ││
      │└──────────┘│──>│└───────────┘│──>│└───────────┘│──>│└───────────┘│
      │┌──────────┐│   │┌───────────┐│   │┌───────────┐│   │┌───────────┐│
      ││OUTPUT    ││   ││POSTROUTING││   ││POSTROUTING││   ││FORWARD    ││
      │└──────────┘│   │└───────────┘│   │└───────────┘│   │└───────────┘│
      └────────────┘   │┌───────────┐│   │┌───────────┐│   │┌───────────┐│
                       ││INPUT      ││   ││OUTPUT     ││   ││OUTPUT     ││
                       │└───────────┘│   │└───────────┘│   │└───────────┘│
                       │┌───────────┐│   └─────────────┘   └─────────────┘
                       ││OUTPUT     ││
                       │└───────────┘│
                       │┌───────────┐│
                       ││FORWARD    ││
                       │└───────────┘│
                       └─────────────┘
```

the five build-in chain of iptables:
```txt

      once the received packets get                               pkts generated by localhost pend
      in proto stack, PREROUTING is                               to send or redirection, PREROUTING
      triggerred before any route matching.                       is triggerred after routing judgement.
              ┌─────────────┐                                               ┌─────────────┐
              │ PREROUTING  │                                               │ POSTROUTING │
              │ +---------+ │                                               │ +---------+ │
              │ |  nat    | │       after the pkts pass through route       │ |  nat    | │
              │ |  mangle | │       table, FORWARD is triggerred if         │ |  mangle | │
              │ |  raw    | │       pkt destination is other machine.       │ |  raw    | │
              │ +---------+ │               ┌─────────────┐                 │ +---------+ │
              └─────────────┘               │   FORWARD   │                 └─────────────┘
 destination         │    no                │ +---------+ │                    │   │
    is to            +─────────────────────>│ | filter  | │────────────────────┘   │
  localhost?     yes │                      │ | mangle  | │                        │
              ┌─────────────┐               │ +---------+ │                 ┌─────────────┐
              │    INPUT    │               └─────────────┘                 │   OUTPUT    │
              │ +---------+ │                                               │ +---------+ │
              │ |  filter | │                LOCALHOST                      │ | filter  | │
              │ |  mangle | │───────────────────────────────────────────────│ | nat     | │
              │ +---------+ │                                               │ | mangle  | │
              └─────────────┘                                               │ | raw     | │
      after the pkts pass through route                                     │ +---------+ │
      table, INPUT is triggerred if pkt                                     └─────────────┘
      destination is current host machine.                        pkt generated by localhost pend
                                                                  for send, OUTPUT is triggerred
                                                                  once pkt get in proto stack.
```

<hr>

### # self-defined chains in iptables
iptables rules permits the packets handled by self-defined chain rules, hence self-defined
chain is supported inside iptables.

however, self-defined chain rules cannot registered into netfilter hook, thus all self-defined
chain can only be invoked by other chain (e.g. build-in or another self-defined chain).

self-defined chain can be seen as extension for its invoker, cannot be the source & tail handler though.
the end of self-defined chain could return to kernel netfilter hook, could jump to other self-defined
chain, by which make iptables to be powerful to buildup complicated network rules.

<hr>

### # kube-proxy in kubernetes iptables rules analysis
```txt

             ──────────────────────────────────────────────────────────────────────
             ┌─────────┐  ┌───────────────────┐  ┌─────────┐  ┌───────────────────┐
             │ rule #1 │  │ rule #2           │  │ rule #3 │  │ ACCEPT            │
PREROUTING   │         ├──+ if match; jump to ├──+         ├──+ defined by policy │
             │         │  │ KUBE-SERVICE      │  │         │  │                   │
             └─────────┘  └─────────┬─────────┘  └────+────┘  └───────────────────┘
             ───────────────────────|─────────────────│────────────────────────────
                                    |                 |
                  ┌-----------------┘                 └--┐
                  |                                      |
             ─────|──────────────────────────────────────|─────────────────────────
             ┌────+────┐  ┌───────────────────┐  ┌───────┴────────┐
             │ rule #1 │  │ rule #2           │  │ RETURN         │
KUBE-SERVICE │         ├──+ if match; jump to ├──+ last target is │
             │         │  │ KUBE-SRC-XXX      │  │ always return  │
             └─────────┘  └────────┬──────────┘  └───────+────────┘
             ──────────────────────|─────────────────────|─────────────────────────
                                   |                     |
                  ┌----------------┘                     |
             ─────|──────────────────────────────────────|─────────────────────────
             ┌────+────┐  ┌───────────────────┐  ┌───────┴────────┐
             │ rule #1 │  │ rule #2           │  │ RETURN         │
KUBE-SVC     │         ├──+ if match; jump to ├──+ last target is │
             │         │  │ KUBE-SERVICE      │  │ always return  │
             └─────────┘  └───────────────────┘  └────────────────┘
             ──────────────────────────────────────────────────────────────────────
```
the kube-proxy utilize the self-defined chain in iptables to implement the micro service mechanism.

the overview architecture is as:
```txt
                                                 ┌──────────────┐
                                             ┌---+ KUBE-SEP-XXX │
                                             |   └──────────────┘
                          ┌──────────────┐   |   ┌──────────────┐
                      ┌---+ KUBE-SVC-XXX ├---┼---+ KUBE-SEP-XXX │
                      |   └──────────────┘   |   └──────────────┘
                      |                      |   ┌──────────────┐
                      |                      └---+ KUBE-SEP-XXX │
                      |                          └──────────────┘
                      |                         
   ┌──────────────┐   |   ┌──────────────┐       ┌──────────────┐
   │ KUBE-SERVICE ├---┼---+ KUBE-SVC-XXX ├-------+ KUBE-SEP-XXX │
   └──────────────┘   |   └──────────────┘       └──────────────┘
                      |                         
                      |                          ┌──────────────┐
                      |                      ┌---+ KUBE-SEP-XXX │
                      |   ┌──────────────┐   |   └──────────────┘
                      └---+ KUBE-SVC-XXX ├---┼
                          └──────────────┘   |   ┌──────────────┐
                                             └---+ KUBE-SEP-XXX │
                                                 └──────────────┘
```
KUBE-SERVICE act as the entrance chain of a reverse proxy in kubenetes cluster,
KUBE-SVC-XXX act as the entrance chain of a certain micro service,
KUBE-SEP-XXX is represents the ip & port info of k8s pod (the endpoint chain).

the KUBE-SERVICE will jump to certain KUBE-SVC-XXX chain according to the target service ip
of the msg, then the KUBE-SVC-XXX will jump to certain endpoint chain by applying some
load-balancing algorithms.
