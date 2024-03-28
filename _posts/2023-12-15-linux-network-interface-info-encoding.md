---
layout: post
title: "encoding information through linux interface (linux, network)"
author: "melon"
date: 2024-01-30 21:56
categories: "2024"
tags:
  - linux
  - network
---

### # background
a application scenarios of nested containers in device simulation is to setup
some veth pairs for network connectivity between multiple netns. imagine a scenario as follows:

from the parent netns, we can get the basic info of a certain interface like:
topo connection, functionalities of the itf, etc.

however, in the child netns (container), the info cannot be retrieved. thus the applications
inside the child netns cannot get enough info to startup.

here are some basic methods to implement this.

<hr>

### # method-1: use simple interface name for encoding
use veth interface naming to encode the functionality of each interface in olt container,
then scan all available interface inside ihub container & dynamic assign to qemu process,
hence the image inside ihub could decode.

see code in devtools/hostfw before this changset:
```text
changeset:   20925:89870ca95ac1
user:        metung <melon.tung@xxxxx-sbell.com>
date:        Thu Jan 04 09:40:19 2024 +0800
summary:     hostfw: fix chassis reset, ihub with marvell connection are lost
```
files to check: devtools/hostfw/models/setup/ihub.yaml, devtools/hostfw/dockerfile/ihub/docker-entrypoint.sh.
the itf name scanning logic:
```text
# scan for names like: eth_lt1_up_p, eth_lt2_up_p, eth_pcta_up1_p, eth_pcta_up2_p
buildup_qemu_postfix() {
    qemu_cmd_postfix=""
    eth_lst=$(ifconfig -s | awk '{print $1}' | grep "^eth" | grep "_p$")
    ...
}
```
this method have limitations in kernel, which the itf name should be less than 16 characters.

<hr>

### # method-2: use eth alias support inside sysfs of kernel
create a veth pair for test:
```text
$ ip link add my_veth_itf type veth peer name my_veth_itf_p
```

set the alias for certain interface:
```text
$ ip link set dev my_veth_itf alias this_is_a_alias_for_my_veth_itf
```

check the existence & state of the links (ip -c link is same as ip link show):
```text
$ ip -c link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
   link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
...
9: my_veth_itf_p@my_veth_itf: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default
   qlen 1000 link/ether 76:9a:2d:a3:b5:c4 brd ff:ff:ff:ff:ff:ff
10: my_veth_itf@my_veth_itf_p: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default
    qlen 1000 link/ether 62:3c:b2:3a:9a:cd brd ff:ff:ff:ff:ff:ff
```

check the alias info encoded to veth: my_veth_itf
```text
$ ip link show my_veth_itf
10: my_veth_itf@my_veth_itf_p: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 62:3c:b2:3a:9a:cd brd ff:ff:ff:ff:ff:ff
    alias this_is_a_alias_for_my_veth_itf
```

another way to do this:
```text
$ ip -o -d link show my_veth_itf
10: my_veth_itf@my_veth_itf_p: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 62:3c:b2:3a:9a:cd brd ff:ff:ff:ff:ff:ff promiscuity 0 veth addrgenmode eui64 numtxqueues 1 numrxqueues 1
    gso_max_size 65536 gso_max_segs 65535 alias this_is_a_alias_for_my_veth_itf
```
however, the alias name cannot be used as a replica of the real net device name: this alias name cannot
be used in iproute cmd for interface operations.  
ref https://unix.stackexchange.com/a/391547

related doc in kernel: https://www.kernel.org/doc/Documentation/ABI/testing/sysfs-class-net

code in action, see the hostfw version after this changset:
```text
todo Maxime
```

<hr>

### # method-3: using altname by iproute of kernel
this feature is only available after the iproute suit version after v5.4.0 came after 2019-11-25.  
the altname solution for encoding info inside linux netdev is:
```text
$ ip link add name brx type bridge
$ ip link altname add brx name mypersonalsuperspecialbridge
$ ip link set someotherveryveryveryverylongname master mypersonalsuperspecialbridge
$ ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
   link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
3: dummy0: <BROADCAST,NOARP> mtu 1500 qdisc noop master brx state DOWN mode DEFAULT group default qlen 1000
   link/ether 7e:a2:d4:b8:91:7a brd ff:ff:ff:ff:ff:ff
   altname someothername
   altname someotherveryveryveryverylongname
4: brx: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
   link/ether 7e:a2:d4:b8:91:7a brd ff:ff:ff:ff:ff:ff
   altname mypersonalsuperspecialbridge
```
add ipv4 address to the bridge using alternative name:
```text
$ ip addr add 192.168.0.1/24 dev mypersonalsuperspecialbridge
$ ip addr show mypersonalsuperspecialbridge
4: brx: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
   link/ether 7e:a2:d4:b8:91:7a brd ff:ff:ff:ff:ff:ff
   altname mypersonalsuperspecialbridge
   inet 192.168.0.1/24 scope global brx
   valid_lft forever preferred_lft forever
```

currently the version of iproute pkg on my host is:
```text
$ yum info iproute
Loaded plugins: fastestmirror, ovl
Repodata is over 2 weeks old. Install yum-cron? Or run: yum makecache fast
Determining fastest mirrors
 * base: mirror.zetup.net
 * epel: ftp.fau.de
 * extras: mirror.fysik.dtu.dk
 * updates: mirror.fysik.dtu.dk
Installed Packages
Name        : iproute
Arch        : x86_64
Version     : 4.11.0
Release     : 30.el7
Size        : 1.8 M
Repo        : installed
From repo   : base
Summary     : Advanced IP routing and network device configuration tools
URL         : http://kernel.org/pub/linux/utils/net/iproute2/
License     : GPLv2+ and Public Domain
Description : The iproute package contains networking utilities (ip and rtmon, for example)
            : which are designed to use the advanced networking capabilities of the Linux
            : kernel.
```
code patch to implement this in kernel, please search in the google linux kernel archives with keyword:
patch net-next rfc 0/7: introduce alternative names for network interfaces.

ref: https://lwn.net/Articles/794289/
