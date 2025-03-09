---
layout: post
title: "netlink: interactive protocol between kernel & userspace (linux, network)"
author: "melon"
date: 2024-11-10 20:55
categories: "2024"
tags:
  - linux
  - network
---

netlink can be seen as a replacement of ioctl() syscall.
it aims to replace fixed-format c structure supplied to ioctl() with a format allowing add & extend the argument
in an easier way.

in principle, netlink is a protocol that facilitate communication between the kernel and user-space process.
it is primarily used for networking-related tasks and allows for info exchange about network itfs, routing,
and other network configs.

<hr>

### # netlink pkg issue analysis
$ 1 introduction  
nicon is a golang tool as a server to handle the network connections establishment on hostfw project, which is
a docker/kata-container featured framework for mutli-device visulization.
during nicon deploy the topology links for hostfw, an error occured in its upstream netlink pkg:

```text
time="2024-11-12T09:20:05Z" level=debug msg="Checking if link engine-Lwl7B exist in pid 1"
time="2024-11-12T09:20:05Z" level=debug msg="Link engine-Lwl7B(56:09:28:da:09:1f) exist in pid 1"
time="2024-11-12T09:20:05Z" level=debug msg="Link engine-Lwl7B exist in target namespace, needn't migrate, skip"
time="2024-11-12T09:20:05Z" level=debug msg="Setting device(engine-Lwl7B) in pid:[1]"
time="2024-11-12T09:20:05Z" level=debug msg="DevBase set engine-Lwl7B down"
time="2024-11-12T09:20:05Z" level=debug msg="Veth preSetting engine-Lwl7B"
time="2024-11-12T09:20:05Z" level=debug msg="DevBase presetting engine-Lwl7B"
time="2024-11-12T09:20:05Z" level=debug msg="NicOpt preSetting engine-Lwl7B"
time="2024-11-12T09:20:06Z" level=debug msg="Deploying error return: error connecting p2b: error migrating link:
failed to set veth pre setting: failed to get vlans: failed to list links: interrupted system call"
```

the netlink pkg mentioned is used to provide basic network operations for hostfw project, including veth pair 
management, linux bridge manage & config, vlan configuraton, macvlan, macvtap interface management, etc.
from the err "interrupted system call", the netlink message sent to kernel seems interrupted, and return EINTR.

$ 2 follow-up analysis  
2.1 the changeset that introduce the bug:  
when kernel detect a change took place in the resources being dumped (walked-over) by a netlink call,
it sets nlmsg flag NLM_F_DUMP_INTR to indicate: the result returned may be incomplete or inconsistent.

before golang netlink #925 (v1.2.1), the flag was ignored and results were returned without an error.
with the changeset golang netlink #925, response handling is aborted, results are discarded, unix.EINTR
is returned.

2.2 code diff for golang netlink changeset #925:

```text
// nl/nl_linux.go: the #925 changset
@@ -559,6 +559,11 @@ done:
    if m.Header.Pid != pid {
        continue
    }
    +
    +if m.Header.Flags&unix.NLM_F_DUMP_INTR != 0 {       // flag check & err return
    +    return nil, syscall.Errno(unix.EINTR)
    +}
    +
    if m.Header.Type == unix.NLMSG_DONE || m.Header.Type == unix.NLMSG_ERROR {
        native := NativeEndian()
        errno := int32(native.Uint32(m.Data[0:4]))
@@ -661,12 +666,14 @@ func GetNetlinkSocketAt(newNs, curNs netns.NsHandle, protocol int) (*NetlinkSock)
```

2.3 why we dont return err & abort nlmsg when flag NLM_F_DUMP_INTR is returned?  
depending on what the caller is looking for, the incomplete/inconsistent result may be usable.
moreover, it's preferable to retry sending the whole request rather than report err, when recv NLM_F_DUMP_INTR,
though this might take unbounded time to finish the job, with lots of changes in lots of data returned.

with the changeset #925, the netlink pkg user can catch errors.Is(err, netlink.ErrDumpInterrupted), and take
appropriate action. whatever, it's a breaking change for the caller to add the check.

<hr>

### # definition of netlink related structures
1 two type of nlmsg supported in below toy code, following a strategy of parse both the format, but finally
output info according to nlmsg_type.

```text
interface messages:
[nlmsghdr][ifinfomsg][attributes...]
    ^         ^
    |         |
    h         NLMSG_DATA(h)

address messages:
[nlmsghdr][ifaddrmsg][attributes...]
    ^         ^
    |         |
    h         NLMSG_DATA(h)
```

2 key structure definitions for nlmsg request & resolve.

```text
struct iovec {
    void*              iov_base;          // data buff
    __kernel_size_t    iov_len;           // size of the data
};

struct msghdr {
    void*              msg_name;          // client addr (socket name)
    int                msg_namelen;       // length of the client addr
    struct iovec*      msg_iov;           // pointer to the iovec structure with message data
    __kernel_size_t    msg_iovlen;        // count of the data blocks
    void*              msg_control;       // points to a buf for other proto control-related msg or misc aux data
    __kernel_size_t    msg_controllen;    // length of the msg_control
    unsigned           msg_flags;         // flags on received message
};

struct nlmsghdr {
    __u32              nlmsg_len;         // msg size, include this header
    __u16              nlmsg_type;        // msg type
    __u16              nlmsg_flags;       // msg flags
    __u32              nlmsg_seq;         // sequence number
    __u32              nlmsg_pid;         // sender identifier (typically - process id)
};

struct sockaddr_nl {   
    sa_family_t        nl_family;         // always AF_NETLINK
    unsigned short     nl_pad;            // typically filled with zeros

    pid_t              nl_pid;            // client identifier (process id)
                                          // for kernel socket, nl_pid is always 0.
                                          // for userspace socket, typically use current process id,
                                          // however, this may cause problems in multithreading app,
                                          // when multiple threads try create and use nl sock via the same nl_pid.
                                          // as a workaround, we can init each nl_pid with both pid & tid attached:
                                          // pthread_self() << 16 | getpid()

    __u32              nl_groups;         // special bitmask for sender/reciver group,
                                          // used after calling bind() on the nl sock to subscribe to specified
                                          // groups’ events, which is very helpful for network monitoring task.
                                          // group def can be found in netlink header file.
};
```

<hr>

### # code in action: monitor route & itf state & addr state by netlink
1 src code:

```text
#include <errno.h>
#include <stdio.h>
#include <memory.h>
#include <net/if.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <linux/rtnetlink.h>

void parseRtattr(struct rtattr* tb[], int max, struct rtattr* rta, int len){ // helper func to parse rtattrs from nlmsg
    memset(tb, 0, sizeof(struct rtattr*) * (max + 1));                       // zalloc mem for rtattr arr
    while(RTA_OK(rta, len)){                                                 // while still possible route addr to parse
        if(rta->rta_type <= max){
            tb[rta->rta_type] = rta;                                         // set route addr in tb
        }
        rta = RTA_NEXT(rta, len);                                            // get next attr
    }
}

int main(){
    int fd = socket(AF_NETLINK, SOCK_RAW, NETLINK_ROUTE);                    // create netlink socket
    if(fd < 0){
        printf("Failed to create netlink socket: %s\n", (char*)strerror(errno));
        return 1;
    }

    struct sockaddr_nl local;                                                // netlink socket addr
    memset(&local, 0, sizeof(local));

    char buf[8192];                                                          // netlink header msg buffer (nlmsghdr)
    struct iovec iov;                                                        // msg structure as io-vector
    iov.iov_base = buf;                                                      // set msg buffer as io
    iov.iov_len = sizeof(buf);                                               // set size

    local.nl_family = AF_NETLINK;                                            // set protocol family
    local.nl_groups = RTMGRP_LINK | RTMGRP_IPV4_IFADDR | RTMGRP_IPV4_ROUTE;  // set groups we interested in
    local.nl_pid = getpid();                                                 // set out id using current process id

    struct msghdr msg;                                                       // init msg header
    msg.msg_name = &local;                                                   // local addr
    msg.msg_namelen = sizeof(local);                                         // addr size
    msg.msg_iov = &iov;                                                      // io vector
    msg.msg_iovlen = 1;                                                      // io size

    if(bind(fd, (struct sockaddr*)&local, sizeof(local)) < 0){               // bind socket
        printf("Failed to bind netlink socket: %s\n", (char*)strerror(errno));
        close(fd);
        return 1;
    }

    while(1){                                                                // read & parse msg to nlm headers
        ssize_t status = recvmsg(fd, &msg, MSG_DONTWAIT);                    // recv msg from nl socket, non-blocking
        if(status < 0){                                                      // status>0: num of bytes recv, <0: err
            if(errno == EINTR || errno == EAGAIN){                           // for intr err
                usleep(250000);
                continue;
            }
            printf("Failed to read netlink: %s", (char*)strerror(errno));    // for other err
            continue;
        }
        if(msg.msg_namelen != sizeof(local)){                                // filter out nlm nlm from all recv
            printf("Invalid length of the sender address struct\n");
            continue;
        }
        struct nlmsghdr* h;
        for(h = (struct nlmsghdr*)buf; (ssize_t)sizeof(*h) <= status; ){     // iterate all possible nlmsghdr (h) in buf
            int len = h->nlmsg_len;                                          // get len of each nlmsg
            int l = len - sizeof(*h);
            char* ifName;
            if((l < 0) || (len > status)){
                printf("Invalid message length: %i\n", len);
                continue;
            }
            if((h->nlmsg_type == RTM_NEWROUTE) || (h->nlmsg_type == RTM_DELROUTE)){  // for route table changes
                printf("Routing table was changed\n");  
            }
            else{                                                            // for nlmsg type other than route changes
                                                                             // (1) first way to parse the nlmsg
                char*             ifup;                                      // sign of up / down
                char*             ifrun;                                     // sign of running or not
                struct ifinfomsg* ifi;                                       // itf info struct
                struct rtattr*    tb[IFLA_MAX + 1];                          // route attr arr, IFLA: link-level attr

                ifi = (struct ifinfomsg*) NLMSG_DATA(h);                     // cast ptr type & pos to parse itf info
                parseRtattr(tb, IFLA_MAX, IFLA_RTA(ifi), h->nlmsg_len);      // get nlmsg attrs
                
                if(tb[IFLA_IFNAME]){                                         // get network itf name
                    ifName = (char*)RTA_DATA(tb[IFLA_IFNAME]);
                }
                if(ifi->ifi_flags & IFF_UP){                                 // get up flag of the net itf
                    ifup = (char*)"UP";
                }
                else{
                    ifup = (char*)"DOWN";
                }
                if(ifi->ifi_flags & IFF_RUNNING){                            // get run/no-run flag of the net itf
                    ifrun = (char*)"RUNNING";
                }
                else{
                    ifrun = (char*)"NOT RUNNING";
                }
                                                                             // (2) second way to parse the nlmsg
                char ifAddress[256];                                         // net address buf
                struct ifaddrmsg* ifa;                                       // structure for network itf data
                struct rtattr* tba[IFA_MAX+1];                               // route addr attr arr, IFA_MAX: addr attr

                ifa = (struct ifaddrmsg*) NLMSG_DATA(h);                     // start addr of
                parseRtattr(tba, IFA_MAX, IFA_RTA(ifa), h->nlmsg_len);

                if(tba[IFA_LOCAL]){                                          // if local addr attr present in nlmsg
                    inet_ntop(AF_INET, RTA_DATA(tba[IFA_LOCAL]), ifAddress, sizeof(ifAddress)); // binary -> readable
                }                                                            // ipv4 format, get net bin data, buf, size

                switch(h->nlmsg_type){                                       // print nlmsg info according to nlmsg_type
                    case RTM_DELADDR:
                        printf("Interface %s: address was removed\n", ifName);
                        break;
                    case RTM_DELLINK:
                        printf("Network interface %s was removed\n", ifName);
                        break;
                    case RTM_NEWLINK:
                        printf("New network interface %s, state: %s %s\n", ifName, ifup, ifrun);
                        break;
                    case RTM_NEWADDR:
                        printf("Interface %s: new address was assigned: %s\n", ifName, ifAddress);
                        break;
                }
            }
            // #define NLMSG_ALIGN(len) (((len)+NLMSG_ALIGNTO-1) & ~(NLMSG_ALIGNTO-1))
            status -= NLMSG_ALIGN(len);                                      // update status by aligned msg len
            h = (struct nlmsghdr*)((char*)h + NLMSG_ALIGN(len));             // get next msg
        }
        usleep(250000);
    }
    close(fd);                                                               // close netlink socket
    return 0;
}
```

2 compile & test:

```text
$(terminal) gcc -o out z.c
./out
New network interface veth659198e, state: DOWN NOT RUNNING
New network interface veth659198e, state: DOWN NOT RUNNING
New network interface veth659198e, state: UP NOT RUNNING
New network interface veth659198e, state: UP NOT RUNNING
New network interface veth659198e, state: UP RUNNING
New network interface veth659198e, state: UP RUNNING
New network interface veth659198e, state: UP RUNNING
New network interface veth659198e, state: UP RUNNING
^C
```

```text
$(terminal2) sudo ip link set veth659198e DOWN
Error: either "dev" is duplicate, or "DOWN" is a garbage.
$(terminal2) sudo ip link set veth659198e down
$(terminal2) sudo ip link set veth659198e up
```

<hr>

### # reference
1 golang netlink pkg: https://github.com/vishvananda/netlink  
2 golang netlink fix (#1018): https://github.com/vishvananda/netlink/pull/1018   
3 golang netlink issue changeset (#925): https://github.com/vishvananda/netlink/pull/925/files
