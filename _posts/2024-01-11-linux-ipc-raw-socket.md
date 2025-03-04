---
layout: post
title: "inter-core communication: raw sockets (linux, icc)"
author: "melon"
date: 2024-01-11 21:09
categories: "2024"
tags:
  - linux
  - icc
---

this article focus on an icc packet disorder issue, which occurs in a inter-core interrupt proxy app on
device simulation platform. the following part will cover:  
a) background intro of this case analysis;  
b) give a brief intro of raw socket;  
c) analysis the root cause of this issue;  
d) illustrate a toy app for rawsocket send/recv.

<hr>

### # background of this issue case analysis
the issue occurs 1/200 batch-runs on the simulation platform (the origin smp linux based on multi-vcpus),
but can never be reproduced on the physical target device (a crafted linux on single cpu core).

the inter-core interrupt proxy app (report the icc disorder issue) worked as proxy for amp arch (linux + vxworks)
to facilitate various of interrupts to be proxied as rpmsg to notify the other side.

all apps on target/simulation platform are based on shared api lib as the system programing interfaces.
in this case, the rpc infra sdk for inco-proxy app to send crafted rpmsg is based on rawsocket proto,
thus naturally there's no order-preservation mechanism underneath.

<hr>

### # introduction to rawsocket communication
in each protostack layer, a packet has two disjoint sections: header and payload.
based on the above knowledge, rawsocket can be categorized into network socket (l3)
and data-link socket (l2).

a) layer 3 rawsocket packet typically look like:

```txt
                                          | --------------layer #3----------------|
+─────────────────────────────+───────────+─────────────────────────────+─────────+
│ ethernet (typically) header │ ip header │ your transport layer header │ payload │
+─────────────────────────────+───────────+─────────────────────────────+─────────+
```

using l3 rawsocket, the header and payload of a packet in network layer are free to customize,
thus one could for using raw socket to mimicing ipv4, ipv6\...

b) layer 2 rawsocket packet typically look like:

```txt
                              |---------------------------layer #2----------------------------|
+─────────────────────────────+────────────────────────────+────────────────────────+─────────+
│ ethernet (typically) header │ your internet layer header │ transport layer header │ payload │
+─────────────────────────────+────────────────────────────+────────────────────────+─────────+
```

using l2 rawsocket, the header and payload of a packet in data link layer are free to customize,
one could determine everything in l2/l3 proto: i.e. determine the header/payload of arp, ppp, pppoe\...

<hr>

### # root cause analysis for raw socket disordering
1 why on target board device, the icc raw socket order is preserved?  
although the raw socket has not buildin order preservation support, considering the target board is
built on a single cpu system, the reason can be concluded as:  
a) the ring buffer works in fifo, so the order of icc packets wont be changed when passing by.  
b) the packet receive process before get in proto stack in kernel will not change the order.

<p style="margin-bottom: 20px;"></p>

2 why on device simulator env, the icc raw socket can be disordered?  
the device simulator platform typically has container wrapped vm for running the board build,
and the vm is running on a multi-vcpu host-vm (working machine) managed by openstack.  
for smp linux, pkt steering mechanism is enabled, in certain case, the first arrived rawsocket pkt
reach the vcpu-1 (busy at that time), so the pkt get dispatched to vcpu-2, and 2nd pkt received is
dispatched to vcpu-3\... which result in un-determinstic pkt receive order.

<p style="margin-bottom: 20px;"></p>

3 smp linux packet process schematic diagram

```txt
                                                               ┌────> subque1 ────> cpu0
                                                 ┌──────> que1 ┼────> subque2 ────> cpu1
                                                 │             └────> subque3 ────> cpu2
                                                 │
                                                 │             ┌────> subque1 ────> cpu0
    pkts ───> veth ───> vhw_irq ───> cpu ───> sw_irq ───> que2 ┼────> subque2 ────> cpu1
                                                 │             └────> subque3 ────> cpu2
                                                 │
                                                 │             ┌────> subque1 ────> cpu0
                                                 └──────> que3 ┼────> subque2 ────> cpu1
                                                               └────> subque3 ────> cpu2
```

<hr>

### # solution to the disorder issue
the following is the diff patch for inco app of s6 script from buildroot repo:

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
+        set_rps_cpus $RPMSG_ITF                            # enable rps for each itf, random choose core to bound
         ;;
     stop)
         ;;
```

the fix patch just bind each rpmsg interface rps to certain randomized cpu core, so the order of the rawsocket
packet wont be messed up due to cpu load & pkt steering on smp linux simulation env.

<hr>

### # intro to the rps feature
receive packet steering (rps) is a software implementation of receive side scaling (rss)
in the linux networking stack. the below are some basic info of the priciple & usages of rps:

1 implementation of rps pkt handling feature in kernel src  
rps is called during the bottom half of the irq recv handler, after the driver sends pkt up to the netstack
using netif_rx() or netif_receive_skb(), then function get_rps_cpu() select the cpu queue for pkt processing.

2 why still need rps feature over hardware-based rss:  
a) rps can be used with any nic, not just ones that support rss in hardware.  
b) software filters can be easily added to hash over new protocols in cooperation with rps.  
c) enable rps wont increase the rate of hw dev irq, although introducing inter-core interrupts.

3 how to derive optimal rps config to lift up performance for given setup of cpus & workload?  
a) for low-latency networking, allocating as many queues as there are cpus is recommended.  
b) for high-rate networking, the optimal configuration is the one with the smallest number
of queues where no receive queue overflows due to a saturated cpu.

4 enable rps feature in conjunction with hardware-based rss  
in this case, rps acts as a secondary load-balancing mechanism above the rss queue selection.
modern nics support creating multiple dedicated rss contexts that can be selected based on explicit
matching rules, e.g. direct traffic to specific tcp ports to particular rss contexts and queues.

in summary, the rps settings allow software-based recv-side load balancing on top of or in lieu of
hardware-based rss, providing flexibility in configuring recv-side parallelism for network performance.

ref: https://www.kernel.org/doc/Documentation/networking/scaling.rst

<hr>

### # rawsocket communication in action (l2 rawsocket wrapped in ethernet pkt)
1 rawsocket.h: rawsocket structure data member & operator declarations.

```text
#include <stdio.h>

#define BUFSIZE 8192

struct RawSocket;
struct BaseRequestHandler;

struct RawSocket {                                            // inherited type (specified socket)
    int  socket;                                              // base type
    char buf[BUFSIZE];                                        // rawsocket payload data: 8192 bytes
    char interface[16];                                       // itf name/idx for rawsocket obj to bind on

    void (*bind_rawsocket)(struct RawSocket* pthis);          // bind
    int  (*recv_rawsocket)(struct RawSocket* pthis);          // recv
    void (*close_rawsocket)(struct RawSocket* pthis);         // close
};

struct RawSocket* new_RawSocket(char* interface);             // some customized operation for type rawsocket
void   bind_RawSocket(struct RawSocket* pthis);
int    recv_RawSocket(struct RawSocket* pthis);
void   close_RawSocket(struct RawSocket* pthis);
```

2 rawsocket.c: function operators implementation for rawsocket management.

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

struct RawSocket* new_RawSocket(char* interface){                                       // buildup rawsocket obj
    struct RawSocket* rawsocket = (struct RawSocket*)malloc(sizeof(struct RawSocket));  // allocate mem
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

void bind_RawSocket(struct RawSocket* pthis){                                           // rawsocket bind
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

int recv_RawSocket(struct RawSocket* pthis){                                            // rawsocket recv
    int recv_size;
    memset(pthis->buf, 0x0, sizeof(pthis->buf));                                        // clear buf
    recv_size = recv(pthis->socket, pthis->buf, sizeof(pthis->buf), 0);                 // recv to buf
    if(recv_size < 0){
        perror("recv error");
    }
    return recv_size;
};

void close_RawSocket(struct RawSocket* pthis){
    close(pthis->socket);                                                               // close rawsocket
    free(pthis);                                                                        // clean up rawsocket resources
    return;
}
```

3 ethernet.h: ethernet pkt structure definition & operator function declaration.

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
struct ethhdr_frame {                                      // ethernet frame header
    unsigned char   h_dest[6];                             // dest ether addr
    unsigned char   h_source[6];                           // src ether addr
    __be16          h_proto;                               // proto type: (2byte in big endian form), the kind of data
                                                           // frame it carries: ipv4(0x0800), arp(0x0806), ipv6(0x86DD)
    char            payload[];                             // ethernet frame payload
};

void set_macaddr_from_string(char* str, char* raw);        // set mac addr by str input
void set_macaddr_from_ifname(char* interface, char* raw);  // set mac addr by ifname
void print_macaddr(char* raw);                             // print mac addr as str (helper func)
```

4 ethernet.c:

```text
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <net/ethernet.h>
#include <net/if.h>
#include <netpacket/packet.h>
#include <sys/ioctl.h>

#include "ethernet.h"

void print_macaddr(char* raw){
    int i;
    for(i=0; i<5; i++){
        printf("%02x:", (unsigned char)raw[i]);
    }
    printf("%02x", (unsigned char)raw[5]);
}

void set_macaddr_from_string(char* str, char* raw){               // set raw ethernet mac addr by str
    sscanf(str, "%02x:%02x:%02x:%02x:%02x:%02x",
        (int*)&raw[0], (int*)&raw[1], (int*)&raw[2], (int*)&raw[3], (int*)&raw[4], (int*)&raw[5]);
}

void set_macaddr_from_ifname(char* interface, char* raw){         // set ethernet mac addr by the itf mac
    int fd = socket(AF_INET, SOCK_DGRAM, 0);                      // create tmp udp socket

    struct ifreq ifr;                                             // get mac addr of itf, assign to ifreq (net/if.h)
    ifr.ifr_addr.sa_family = AF_INET;
    strncpy(ifr.ifr_name, interface, IFNAMSIZ-1);
    ioctl(fd, SIOCGIFHWADDR, &ifr);                               // ioctl to get mac addr of the itf
    close(fd);

    *(raw) = (unsigned char)ifr.ifr_hwaddr.sa_data[0];            // setup raw by mac addr from ifreq
    *(raw+1) = (unsigned char)ifr.ifr_hwaddr.sa_data[1];
    *(raw+2) = (unsigned char)ifr.ifr_hwaddr.sa_data[2];
    *(raw+3) = (unsigned char)ifr.ifr_hwaddr.sa_data[3];
    *(raw+4) = (unsigned char)ifr.ifr_hwaddr.sa_data[4];
    *(raw+5) = (unsigned char)ifr.ifr_hwaddr.sa_data[5];
}
```

5 send.c: rawsocker pkt generate & xmit program.

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
    struct RawSocket* rawsocket = new_RawSocket(interface);         // 1 create rawsocket, assign send itf

    unsigned char buf[32];                                          // 2 create ethernet frame header + payload
    struct ethhdr_frame* eth_packet = (struct ethhdr_frame*)buf;    // define pkts & cast to ethhdr_frame format
    memset(buf, 0x0, sizeof(eth_packet));
    set_macaddr_from_string(destination, eth_packet->h_dest);       // set dst mac addr by string
    set_macaddr_from_ifname(interface, eth_packet->h_source);       // set src mac addr by itf name (ioctl)
    eth_packet->h_proto = 0x88b5;                                   // proto type is optional, set 0x88b5 (nonsence)
    char* data = "Hi";                                              // set pyload set as Hi
    memcpy(eth_packet->payload, data, sizeof(data));

    rawsocket->bind_rawsocket(rawsocket);                           // 3 bind
    int send_size = send(rawsocket->socket, &buf, sizeof(buf), 0);  // 4 send
    printf("%dbyte send.\n", send_size);

    rawsocket->close_rawsocket(rawsocket);                          // 5 close
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

4 recv.c: rawsocket receive program.

```text
#include <stdio.h>
#include <stdlib.h>
#include <linux/if_ether.h>

#include "rawsock.h"
#include "ethernet.h"

void start_daemon(char* interface){                                           // start daemon for listening pkt
    struct RawSocket* rawsocket = new_RawSocket(interface);                   // 1 create rawsocket on certain itf
    int len;
    rawsocket->bind_rawsocket(rawsocket);                                     // 2 bind
    while(1){                                                                 // 3 recv
        int len = rawsocket->recv_rawsocket(rawsocket);
        struct ethhdr_frame* data = (struct ethhdr_frame*)(rawsocket->buf);
        fflush(stdout);                                                       // flush before print
        if(len > 0){                                                          // display info of pkt received
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

### # validation on rawsocket communication based on above app
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

1 test: send raw socket packet from docker0/eth0 (src) to docker0 (dest):

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

at meantime in another terminal, confirm the arrival of rawsocket packet by tcpdump:

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

2 test: send packet from docker0 (src) to eth0 (dst):

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

at meantime in another terminal, confirm the arrival of rawsocket packet by tcpdump:

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

3 why no rawsocket can go through between eth0 and docker0?  
in brief, the iptables rules that is responsible for handling traffic from/to docker0 is to blame with.
the netfilter rules simply filter out the pkts with ip dst not matching with the allowed ip in both direction,
to pass the rules and let our rawsocket stream flow between docker0 & eth0, at least we need to implement & config
the layer3 own proto as ip, and assign the correct ip src/dst to enable the connection.

to pinpoint the root cause and get knowledge about the details of how docker facilitate the traffic between docker0
and eth0 (outside world), please ref to blog post:
