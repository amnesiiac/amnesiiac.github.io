---
layout: post
title: "nic offloading feature intro (network, nic, offloading)"
author: "melon"
date: 2024-03-11 20:44
categories: "2024"
tags:
  - network
---

most modern os support some form of network offloading, where some network processing happens on the nic
instead of the cpu.

normally this feature is useful, as it can free up resources on the rest of the system and let the cpu handle
more connections. however, if you're trying to capture traffic, it can result in false errors and strange or
even missing traffic.

<hr>

### # checksum offloading
1 show the offloading config of certain nic:

```text
$ ethtool -K ${dev}
$ ethtool --show-offload ${dev}
```

2 disable the offloading feature of certain nic:

```text
$ ethtool -K ${dev} rx off tx off
$ ethtool --offload ${dev} rx off tx off
```

3 enable the offloading feature of certain nic:

```text
$ ethtool -K ${dev} rx on tx on
$ ethtool --offload ${dev} rx on tx on
```

<hr>

### # segmentation offloading
1 mtu limitation vs transmission efficiency  
the default ethernet maximum transfer unit (mtu) is 1500 bytes, which is the largest frame size can be transmitted,
which can cause system resources to be underutilized: i.e. if there are 3200 bytes of data to transmit, pkt will be
segmented and restored accordingly merely for transport purpose.

2 how to level-up the transmission efficency  
there's an option called offloads, to allow the relevant protocol stack to transmit packets larger than normal mtu.
packets as large as the maximum allowable 64k can be created, with options for both transmit (tx) and recv (rx).

3 the benefits of enable offloading feature  
when sending or receiving large amounts of data, with offload feature, the os can handle large packet rather than
multiple smaller pkts for every 64k of data sent or received, which means fewer irq requests generated, less overhead
for splitting or combining traffic, and more effective transmission, leading to an overall increase in throughput.

<hr>

### # segmentation offloading types
1 tcp segmentation offload (tso, use tcp to send large pkt):
rely on the nic to handle segmentation, and then adds the tcp, ip and data link layer headers to each segment.

2 udp fragmentation offload (ufo, use udp to send large pkt):
rely on the nic to handle ip fragmentation into mtu sized packets for large udp datagrams.

3 generic segmentation offload (gso, use tcp/udp to send large pkt):
if the segmentation & fragmentation cannot be handled by the nic, gso performs the same operations to bypass
the nic hardware. this is achieved by delaying segmentation as late as possible, i.e. until the pkt processed
by device driver.

4 large receive offload (lro, use tcp):
all incoming packets are re-segmented as they are received, reducing the number of segments for system to process.
the segments can be merged either in the driver or using the nic.
a problem with lro is: it tends to re-segment all incoming packets, often ignoring differences between headers
and other part.  
what's more, it is not possible to use lro when ip forwarding is enabled, cause lro in combination with ip
forwarding can lead to checksum errors.

```text
$ cat /proc/sys/net/ipv4/ip_forward          # ip forwarding enabled
1
```

5 generic receive offload (gro, use tcp/udp):
gro is more rigorous than lro when resegmenting packets, for example it checks the mac headers of each packet,
which must match, only a limited number of tcp or ip headers can be different, and the tcp timestamps must match.
re-segmenting can be handled by either the nic or the gso code.

<hr>

### # case study in hostfw device simulation framework project
1 unexpected traffic test error when enable pkt offloading on certain itf  
by default, the linux bridge used by hostfw is configured in hub mode (layer 2 switch), and the pkt offloading
feature is enabled on them, which means the pkts with size >= mtu will be processed by the bridge rather than
the kernel proto stack.

test batch intro: the test framework will send jumbo pkts passing the bridge, and capture & validate
the traffic on recv-end to decide whether the pkt is correct or will drop it & return error.

```txt
         |                                       |                                   |
  send   |                                       |--------> segmented pkt 1 -------->|
 traffic |--------------> jumbo pkt ------------>|--------> segmented pkt 2 -------->| validate traffic
         |                                       |--------> segmented pkt 3 -------->|
      ───o───────────────────────────────────────o───────────────────────────────────o───  [layer 2 physical link]
        src                        linux bridge work in hub mode                    dst

```

test framework try validate the number & checksum of the pkt received, but found the statistics are unexpected
then report error.

2 the changset to fix the above issue:  
please refer to changeset: 20522:f75a0a7b7333 for details.
