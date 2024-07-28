---
layout: post
title: "inter-process communication: raw sockets (linux, ipc)"
author: "melon"
date: 2024-01-11 21:09
categories: "2024"
tags:
  - linux
  - ipc
  - todo
---

a ipc packet disorder issue occurs in hardware access app on device simulator platform recently.
the issue occurs once in 200 testcase batchruns based on simulator platform (smp linux based on
multi-vcpus), but can never be reproduced on the real device env (crafted linux on single cpu core).

the hardware access app (reported the ipc disorder issue) worked as a proxy for logic apps,
major responsible for: device config, inspect and manipulation.
all apps are based on common api lib as their system programing interfaces.
what's more, the ipc communication for the hardware access app is based on rawsocket proto,
and there's no order-preservation userspace proto implemented.

the following part will cover: 1) give a brief intro of raw socket, 2) analysis the root cause of
this issue, 3) illustrate a toy send/recv app based on rawsocket.

<hr>

### # intro to raw-socket communication
in each protostack layer, a packet has two disjoint sections: header and payload.
based on the above knowledge, rawsocket can be categorized into network socket (l3)
and data-link socket (l2).

layer 3 raw socket packet typically look like:

```txt
+─────────────────────────────+───────────+─────────────────────────────+─────────+
│ ethernet (typically) header │ ip header │ your transport layer header │ payload │
+─────────────────────────────+───────────+─────────────────────────────+─────────+
```

by l3 raw socket, the header and payload of a packet in network layer are free to customize,
thus one could for using raw socket to mimicing ipv4, ipv6\...


layer 2 raw socket packet typically look like:

```txt
+─────────────────────────────+────────────────────────────+────────────────────────+─────────+
│ ethernet (typically) header │ your internet layer header │ transport layer header │ payload │
+─────────────────────────────+────────────────────────────+────────────────────────+─────────+
```

by l2 raw socket, the header and payload of a packet in data link layer are free to customize.
thus one could determine everything in both l2/l3 proto.
i.e. determine the header/payload of arp, ppp, pppoe\...

<hr>

### # root cause analysis for raw socket disordering
1 why on target board device, the ipc raw socket order is preserved?  
although the raw socket has not buildin order preservation support, considering the fact
that the target board built on a single cpu system, the raw socket order is ensured due to:  
1) the ring buffer works in fifo, so the order of ipc packets wont be changed when passing by.  
2) the packet receive process before get in proto stack in kernel will not change the order.

<p style="margin-bottom: 20px;"></p>

2 why on device simulator env, the ipc raw socket can be disordered?  
the device simulator platform typically has container wrapped vm for running the board build,
and the vm is running on a multi-vcpu host-vm (developer working machine) managed by openstack.  
for smp linux, the packet steering mechanism is enabled, so in certain case, the first
arrived raw socket pkt reach the vcpu#1 which is busy at that time, so the pkt got dispatched to
vcpu#2, the second pkt received is dispatched to vcpu#3, thus the order of them is un-determinstic.

<p style="margin-bottom: 20px;"></p>

3 smp linux packet process schematic diagram
```txt
                                                               ┌────> subque1 ────> cpu0
                                                 ┌──────> que1 +────> subque2 ────> cpu1
                                                 │             └────> subque3 ────> cpu2
                                                 │
                                                 +             ┌────> subque1 ────> cpu0
    pkts ───> veth ───> vhw_irq ───> cpu ───> sw_irq ───> que2 +────> subque2 ────> cpu1
                                                 +             └────> subque3 ────> cpu2
                                                 │
                                                 │             ┌────> subque1 ────> cpu0
                                                 └──────> que3 +────> subque2 ────> cpu1
                                                               └────> subque3 ────> cpu2
```

<hr>

### # solution to the disorder issue
the following is the diff patch for inco app init script of buildroot repo:

```text
diff -r 1701b7c13468 -r e4c5f03bb577 board/xxxxx/xxxx/features/host/target_skeleton_extras/etc/init.d/S69inco_app
--- a/board/xxxxx/xxxx/features/host/target_skeleton_extras/etc/init.d/S69inco_app	Mon Jun 17 03:49:21 2024 +0200
+++ b/board/xxxxx/xxxx/features/host/target_skeleton_extras/etc/init.d/S69inco_app	Sun Jun 16 13:35:04 2024 +0800
@@ -15,9 +15,10 @@
     exit 0
 fi
 
+RPMSG_ITF="eth_rpmsg"
 ITFS_TO_VLAN="16 32 48 64 80 96 112 128 144 160 176 192 208 224 240 256 272 4000"
 
-ihub_enable=$(ifconfig|grep -ci "eth_rpmsg")
+ihub_enable=$(ifconfig|grep -ci $RPMSG_ITF)                # num of rpmsg itf configured for ihub-nt communication
 
 network_itf_vlan()
 {
@@ -34,6 +35,28 @@
     ifconfig rpmsg$itf up
 }
 
+set_rps_cpus()
+{
+    itf=$1
+    cpu_cores=$(grep -c ^processor /proc/cpuinfo)          # num of cores by count num of matched lines
+
+    if [ "$cpu_cores" -gt 32 ]; then
+        cpu_cores=32
+    fi
+
+    selected_core=$(shuf -i 0-$(($cpu_cores-1)) -n 1)      # random selected core num in (0,num_core-1)
+    cpu_mask=$(printf "%x" $((1 << $selected_core)))       # cpu mask
+
+    rps_path="/sys/class/net/$itf/queues/rx-0/rps_cpus"    # itf rps path
+
+    if [[ -f $rps_path ]]; then                            # enable rps feature for itf
+        echo $cpu_mask > $rps_path
+        echo "RPS bound to CPU core: $selected_core, mask: $cpu_mask, path: $rps_path"
+    else
+        echo "RPS path not found: $rps_path"
+    fi
+}
+
 case "$1" in
     start)
         for itf in $ITFS_TO_VLAN; do
@@ -43,6 +66,7 @@
                 network_itf_stub $itf
             fi
         done
+        set_rps_cpus $RPMSG_ITF                            # enable rps for each itf, random choose core to bound with
         ;;
     stop)
         ;;
```

the above fix diff bind each rpmsg interface rps to certain randomized cpu core, so the order
of the raw socket packet wont be messed up due to cpu load.

<hr>

### # intro to the rps feature
receive packet steering (rps) is a software implementation of receive side scaling (rss)
in the linux networking stack. the below are some basic info of the priciple & usages of rps:

1 the implementation of rps pkt handling feature in kernel src  
rps is called during the bottom half of the receive interrupt handler, after the driver sends
pkt up to the netstack using netif_rx() or netif_receive_skb().
then function get_rps_cpu() select the cpu queue for pkt processing.

2 why still need rps feature over hardware-based rss:  
1) rps can be used with any nic, not just ones that support rss in hardware.  
2) software filters can be easily added to hash over new protocols in cooperation with rps.  
3) enable rps wont increase the rate of hw dev irq, although introducing inter-processor interrupts.

3 how to derive optimal rps config to lift up performance for given setup of cpus & workload?  
1) for low-latency networking, allocating as many queues as there are cpus is recommended.  
2) for high-rate networking, the optimal configuration is the one with the smallest number
of queues where no receive queue overflows due to a saturated cpu.

4 enable rps feature in conjunction with hardware-based rss  
in this case, rps acts as a secondary load-balancing mechanism above the rss queue selection.
modern nics support creating multiple dedicated rss contexts that can be selected based on explicit
matching rules, e.g. direct traffic to specific tcp ports to particular rss contexts and queues.

in summary, the rps settings allow software-based recv-side load balancing on top of or in lieu of
hardware-based rss, providing flexibility in configuring recv-side parallelism for network performance.

ref: https://www.kernel.org/doc/Documentation/networking/scaling.rst

<hr>

### # raw socket communication in action
rawsocket.h: todo
```text
#include <stdio.h>

#define BUFSIZE 8192

struct RawSocket;
struct BaseRequestHandler;

struct RawSocket {
    int  socket;
    char buf[BUFSIZE];                                        // data buf = 8192 bytes
    char interface[16];                                       // itf name/idx

    void (*bind_rawsocket)(struct RawSocket* pthis);          // bind
    int  (*recv_rawsocket)(struct RawSocket* pthis);          // recv
    void (*close_rawsocket)(struct RawSocket* pthis);         // close
};

struct RawSocket* new_RawSocket(char* interface);

void bind_RawSocket(struct RawSocket* pthis);
int  recv_RawSocket(struct RawSocket* pthis);
void close_RawSocket(struct RawSocket* pthis);
```

rawsocket.c: todo

```text
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <net/ethernet.h>
#include <net/if.h>
#include <netpacket/packet.h>

#include "rawsock.h"

struct RawSocket* new_RawSocket(char* interface){
    struct RawSocket* rawsocket = (struct RawSocket*)malloc(sizeof(struct RawSocket));
    if((rawsocket->socket = socket(AF_PACKET, SOCK_RAW, htons(ETH_P_ALL))) < 0){        // create socket
        perror("socket");
        exit(0);
    };
    rawsocket->bind_rawsocket = bind_RawSocket;                                         // register methods
    rawsocket->recv_rawsocket = recv_RawSocket;
    rawsocket->close_rawsocket = close_RawSocket;
    memset(rawsocket->interface, 0x0, sizeof(rawsocket->interface));                    // clear itf string
    strcpy(rawsocket->interface, interface);                                            // register with new itf
    return rawsocket;
}

void close_RawSocket(struct RawSocket* pthis){
    close(pthis->socket);                                                               // close socket
    free(pthis);                                                                        // clean up rawsocket resources
    return;
}

void bind_RawSocket(struct RawSocket* pthis){
    struct sockaddr_ll sockaddr;                                                        // init sockaddr_ll
    memset(&sockaddr, 0x0, sizeof(sockaddr));
    sockaddr.sll_family = AF_PACKET;                                                    // set sll family
    sockaddr.sll_protocol = htons(ETH_P_ALL);                                           // set sll proto
    sockaddr.sll_ifindex = if_nametoindex(pthis->interface);                            // set sll itf idx
    if(bind(pthis->socket, (struct sockaddr*)&sockaddr, sizeof(sockaddr)) < 0){         // bind socket to addr
        perror("bind");
        exit(0);
    }
}

int recv_RawSocket(struct RawSocket* pthis){
    int recv_size;
    memset(pthis->buf, 0x0, sizeof(pthis->buf));                                        // clear buf
    recv_size = recv(pthis->socket, pthis->buf, sizeof(pthis->buf),0);                  // recv to buf
    if(recv_size < 0){
        perror("recv error");
    }
    return recv_size;
};
```

send.c: todo

```text
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <net/ethernet.h>
#include <net/if.h>
#include <netpacket/packet.h>
#include <sys/ioctl.h>

#include "rawsock.h"
#include "ethernet.h"

void ethping(char* destination, char* interface){
    struct RawSocket* rawsocket = new_RawSocket(interface);         // create raw socket, assign send itf

    unsigned char buf[32];                                          // buffer for pkt data
    struct ethhdr_frame* eth_packet = (struct ethhdr_frame*)buf;    // define pkts & cast to ethhdr_frame format
    memset(buf, 0x0, sizeof(eth_packet));
    set_macaddr_from_string(destination, eth_packet->h_dest);       // set recv mac addr
    set_macaddr_from_ifname(interface, eth_packet->h_source);       // set send mac addr by itf name
    eth_packet->h_proto = 0x88b5;                                   // proto type is optional, assign as 0x88b5

    char* data = "Hi";                                              // pyload set as Hi
    memcpy(eth_packet->payload, data, sizeof(data));

    rawsocket->bind_rawsocket(rawsocket);                           // bind
    int send_size = send(rawsocket->socket, &buf, sizeof(buf), 0);  // send
    printf("%dbyte send.\n", send_size);

    rawsocket->close_rawsocket(rawsocket);                          // close
}

int main(int argc, char* argv[]){
    if(argc != 3){
        printf("usage: %s <destination> <interface>\n", argv[0]);
        exit(0);
    }
    char* destination = argv[1];
    char* if_name = argv[2];
    ethping(destination, if_name);
    return 0;
}
```

recv.c: todo
```text
#include <stdio.h>
#include <stdlib.h>
#include <linux/if_ether.h>

#include "rawsock.h"
#include "ethernet.h"

void start_daemon(char* interface){
    struct RawSocket* rawsocket = new_RawSocket(interface);                   // create raw socket on certain itf
    int len;
    rawsocket->bind_rawsocket(rawsocket);                                     // bind
    while(1){                                                                 // recv
        int len = rawsocket->recv_rawsocket(rawsocket);
        struct ethhdr_frame* data = (struct ethhdr_frame*)(rawsocket->buf);
        fflush(stdout);
        if(len > 0){                                                          // display
            printf("src: ");
            print_macaddr(data->h_source);
            printf(", ");
            printf("dst: ");
            print_macaddr(data->h_dest);
            printf(", ");
            printf("type: ");
            printf("%02x", (uint16_t)data->h_proto);
            printf(", ");
            printf("payload: ");
            printf("%s", data->payload);
            printf("\n");
        }
    }
}

int main(int argc, char* argv[]){
    if(argc != 2){
        printf("usage: %s <interface>\n", argv[0]);
        exit(0);
    }
    char* if_name = argv[1];
    start_daemon(if_name);
    return 0;
}
```

<hr>

### # testing raw-socket communication
before program execution, let's check the eth0 & docker0 hardware info firstly:

```text
$ ifconfig eth0
eth0: flags=4419<UP,BROADCAST,RUNNING,PROMISC,MULTICAST>  mtu 8142
        inet 192.168.0.20  netmask 255.255.255.0  broadcast 192.168.0.255
        inet6 fe80::f816:3eff:fe64:ce91  prefixlen 64  scopeid 0x20<link>
        ether fa:16:3e:64:ce:91  txqueuelen 1000  (Ethernet)
        RX packets 394717265  bytes 962929734113 (896.7 GiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 216771679  bytes 55241773383 (51.4 GiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

$ ifconfig docker0
docker0: flags=4419<UP,BROADCAST,RUNNING,PROMISC,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 172.17.255.255
        inet6 fe80::42:2bff:fe80:9784  prefixlen 64  scopeid 0x20<link>
        ether 02:42:2b:80:97:84  txqueuelen 0  (Ethernet)
        RX packets 128638339  bytes 11242038200 (10.4 GiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 258259031  bytes 627529820967 (584.4 GiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

<p style="margin-bottom: 20px;"></p>

1 testcase: send raw socket packet from docker0 or eth0 (src) to docker0 (dest):

```text
$ ./send 02:42:2b:80:97:84 docker0
32byte send.

$ ./send 02:42:2b:80:97:84 eth0
32byte send.

$ ./send 02:42:2b:80:97:84 docker0
32byte send.

$ ./send 02:42:2b:80:97:84 docker0
32byte send.

$ ./send 02:42:2b:80:97:84 docker0
32byte send.

$ ./send 02:42:2b:80:97:84 docker0
32byte send.

$ ./send 02:42:2b:80:97:84 eth0
32byte send.
```

recv raw socket packet from docker0, filter the output by designated payload Hi:

```text
$ ./recv docker0 | grep 'Hi'
src: 02:42:2b:80:97:84, dst: 02:42:2b:80:97:84, type: 88b5, payload: Hi
src: 02:42:2b:80:97:84, dst: 02:42:2b:80:97:84, type: 88b5, payload: Hi
src: 02:42:2b:80:97:84, dst: 02:42:2b:80:97:84, type: 88b5, payload: Hi
src: 02:42:2b:80:97:84, dst: 02:42:2b:80:97:84, type: 88b5, payload: Hi
src: 02:42:2b:80:97:84, dst: 02:42:2b:80:97:84, type: 88b5, payload: Hi
```

at meantime in another terminal, confirm the arrival of raw-socket packet by tcpdump:

```text
$ tcpdump -nettti docker0 '(ether dst host 02:42:2b:80:97:84)' -vvnnXX -l | grep Hi
tcpdump: listening on docker0, link-type EN10MB (Ethernet), capture size 262144 bytes
        0x0000:  0242 2b80 9784 0242 2b80 9784 b588 4869  .B+....B+.....Hi
        0x0000:  0242 2b80 9784 0242 2b80 9784 b588 4869  .B+....B+.....Hi
        0x0000:  0242 2b80 9784 0242 2b80 9784 b588 4869  .B+....B+.....Hi
        0x0000:  0242 2b80 9784 0242 2b80 9784 b588 4869  .B+....B+.....Hi
        0x0000:  0242 2b80 9784 0242 2b80 9784 b588 4869  .B+....B+.....Hi
```

the above test result shows that rawsocket packets send from docker0 to docker0 is received fine,
while the rawsocket packets send from eth0 to docker0 cannot reach docker0.

<p style="margin-bottom: 20px;"></p>

2 testcase: try send packet from docker0 to eth0:

```text
$ ./send fa:16:3e:64:ce:91 docker0
32byte send.

$ ./send fa:16:3e:64:ce:91 docker0
32byte send.

$ ./send fa:16:3e:64:ce:91 eth0
32byte send.

$ ./send fa:16:3e:64:ce:91 eth0
32byte send.

$ ./send fa:16:3e:64:ce:91 eth0
32byte send.
```

recv raw socket pkt from itf eth0, filter the output by the designated payload Hi:

```text
$ ./recv eth0 | grep 'Hi'
src: fa:16:3e:64:ce:91, dst: fa:16:3e:64:ce:91, type: 88b5, payload: Hi
src: fa:16:3e:64:ce:91, dst: fa:16:3e:64:ce:91, type: 88b5, payload: Hi
src: fa:16:3e:64:ce:91, dst: fa:16:3e:64:ce:91, type: 88b5, payload: Hi
```

at meantime in another terminal, confirm the arrival of raw-socket packet by tcpdump:

```text
$ tcpdump -nettti eth0 '(ether dst host fa:16:3e:64:ce:91)' -vvnnXX -l | grep Hi
tcpdump: listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
        0x0000:  fa16 3e64 ce91 fa16 3e64 ce91 b588 4869  ..>d....>d....Hi
        0x0000:  fa16 3e64 ce91 fa16 3e64 ce91 b588 4869  ..>d....>d....Hi
        0x0000:  fa16 3e64 ce91 fa16 3e64 ce91 b588 4869  ..>d....>d....Hi
```

the above test result shows that only rawsocket packet send from eth0 to eth0 is received fine,
the rawsocket packet send from docker0 to eth0 cannot reach eth0.

<p style="margin-bottom: 20px;"></p>

3 analysis of why the no raw socket can go through between eth0 and docker0?  
to pinpoint the problem, recall the docker iptables blog, todo

so far, i think the iptables simply filter out the pkts with ip dest not reaching the allowed ip.
some solid proof needed.
