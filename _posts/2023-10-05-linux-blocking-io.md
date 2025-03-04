---
layout: post
title: "blocking io (linux io)"
author: "melon"
date: 2023-10-05 12:44
categories: "2023"
tags:
  - linux
  - ongoing
---

### # introduction to blocking io model
the blocking is a general operation that a process give up cpu & hang-up due to waiting for some event.

inside networking IO semantics, when a process waiting for data from certain socket, 
if the data is not comming, will set the process state from TASK_RUNNING to TASK_INTERRUPTABLE,
and give up cpu initiatively, let dispatcher to arrange other procs in ready state.

ref: BV1qX4y1H7gm

<hr>

### # the expenses for synchronous blocking
1 when receiving data from a socket using recv syscall, if no data arrived, current process will get off from 
cpu occupytion, and left cpu to another process, leading to extra expense of context switch.  

2 when the connection data is arrived, previous sleep process is wakeup by the notification from ksoftirq context,
in which another expenses for process context switch.  

3 what's more, sync blocking IO model means one process for one socket.
in high-concurrency application scenarios, many processes are spawned, which will takeup untolerable memory usage.

if cpu is busying doing context switches, without actually doing the business logics, that's nonsense.
if one process for one socket connection, then tons of thousands of connection might be established for a 
company, which will lead to memory exhausted.

<hr>

### # blocking io case study I: read/write a large data to stdout/file (apue)
```text
#include <errno.h>
#include <fcntl.h>                                                    // file flag
#include "apue.h"                                                     // apue header file

char buf[500000];                                                     // read up to 500kb content

int main(void){
    int     ntowrite;                                                 // num of byte to write
    int     nwrite;                                                   // num of byte writen by cur write
    char*   ptr;
    ntowrite = read(STDIN_FILENO, buf, sizeof(buf));                  // read: up to 500kb from stdin into buf
    fprintf(stderr, "read %d bytes\n", ntowrite);                     // print num of byte to read
    ptr = buf;
                                                                      // default, write op is in blocking mode
    while(ntowrite > 0){                                              // continously writing to stdout
        errno = 0;
        nwrite = write(STDOUT_FILENO, ptr, ntowrite);                 // write to stdout, setup nwrite
        fprintf(stderr, "nwrite = %d, errno = %d\n", nwrite, errno);  // print: nwrite, errno
        if(nwrite > 0){                                               // reset start ptr if cur write is valid
            ptr += nwrite;
            ntowrite -= nwrite;
        }
    }
    exit(0);
}
```

download apue code and compile the above code using:

```text
$ gcc -o out test.c -I /Users/mac/c3/apue/apue.3e/include -L /Users/mac/c3/apue/apue.3e/lib -lapue
```

1 using file /etc/services as stdin, only redirect stderr to file, left stdin to terminal:

```text
$ ./out < /etc/services/ 2>err
# Network services, Internet style
#
# Note that it is presently the policy of IANA to assign a single well-known
# port number for both TCP and UDP; hence, most entries here have two entries
# even if the protocol doesn't support UDP operations.
#
# The latest IANA port assignments can be gotten from
#
#	http://www.iana.org/assignments/port-numbers
...
```

check the err file content as follows, the write to console is done in only 1 write calls,
in blocking mode, the write will block on the terminal io till the whole content are written out successfully.

```text
read 500000 bytes
nwrite = 500000, errno = 0      # the write is done in 1 try
```

2 using file /etc/services as stdin, redirect stderr to file, redirect stdin to file:

```text
$ ./out < /etc/services/ 2>err 1>info
```

check the stderr of the above test case, the stdin is a regular file, the write operation is expected to be executed once. When stdout is regular file, the file i/o is responsible for intaking the output, the buffer is enough for accepting 500kb in.

```text
$ cat err
read 500000 bytes
nwrite = 500000, errno = 0
```

check the stdout of the above test case, which is same as test case 1.

```text
$ cat info
# Network services, Internet style
#
# Note that it is presently the policy of IANA to assign a single well-known
# port number for both TCP and UDP; hence, most entries here have two entries
# even if the protocol doesn't support UDP operations.
#
# The latest IANA port assignments can be gotten from
#
#	http://www.iana.org/assignments/port-numbers
...
```

<hr>

### # blocking io case study II: the representation of socket inside kernel
a simple but typical code for network io in sync blocking mode:

```text
int main(){
    int sock = socket(AF_INET, SOCK_STREAM, 0);
    connect(sock, ...);
    recv(sock, ...);
}
```

the related kernel socket obj with respect to above socket syscall is as:

```txt
              struct socket                    struct socket
        ┌───────────────────────┐        ┌────────────────────────────┐
        │┌─────────────────┐    │        │┌─────────────────────────┐ │
        ││struct file* file│    │        ││sk_prot (protocol handle)├─┼───┐
        │└─────────────────┘    │        │└─────────────────────────┘ │   │
        │┌───────────────┐      │        │┌──────────────────────────┐│   │    struct socket_wq
        ││struct sock* sk├──────┼────────+│sk_receive_queue (rev que)││   │  ┌────────────────────────┐
        │└───────────────┘      │        │└──────────────────────────┘│   │  │┌──────────────────────┐│
        │┌─────────────────────┐│        │┌─────────────────┐         │   │  ││wait_queue_head_t wait││
        ││struct proto_ops* ops││        ││*sk_wq (wait_que)├─────────┼───│──+└──────────────────────┘│
        │└──────────┬──────────┘│        │└─────────────────┘         │   │  └────────────────────────┘
        └───────────┼───────────┘        │┌───────────────────────┐   │   │  ┌─────────────────┐
                    │                    ││void (*sk_data_ready)()├───┼──────+sock_def_readable│
                    │                    │└───────────────────────┘   │   │  └─────────────────┘
                    │                    └────────────────────────────┘   │   callback function
                    │                                                     │
        ┌───────────+───────────────┐    ┌─────────────────────────────┐  │
        │┌───────┐    ┌───────────┐ │    │┌────────┐   ┌──────────────┐│  │
        ││.accept├────+inet_accept│ │    ││.connect├───+tcp_v4_connect││  │
        │└───────┘    └───────────┘ │    │└────────┘   └──────────────┘│  │
        │┌────────┐   ┌────────────┐│    │┌────────┐   ┌────────────┐  │  │
        ││.sendmsg├───+inet_sendmsg││    ││.sendmsg├───+tcp_sendmsg │  +──┘
        │└────────┘   └────────────┘│    │└────────┘   └────────────┘  │
        │┌────────┐   ┌────────────┐│    │┌────────┐   ┌────────────┐  │
        ││.recvmsg├───+inet_recvmsg││    ││.recvmsg├───+tcp_recvmsg │  │
        │└────────┘   └────────────┘│    │└────────┘   └────────────┘  │
        └───────────────────────────┘    └─────────────────────────────┘
       struct proto_ops inet_stream_ops      struct protol tcp_prot

```

the code related to create the above structure:  
1) the syscall entrance: sock_create.

```text
// file: net/socket.c
SYSCALL_DEFINE3(socket, int, family, int, type, int, protocol){
    ...
    retval = sock_create(family, type, protocol, &sock);
    ...
}
```

2) the inner implementor: __sock_create.

```text
// file: net/socket.c
int __sock_create(struct net* net, int family, ...){
    struct socket* sock;
    const struct net_proto_family* pf;
    ...
    sock = sock_alloc();                          // alignment of socket obj
    pf = rcu_dereference(net_families[family]);   // get operation table of all proto family
    err = pf->create(net, sock, protocol, kern);  // call each proto family's creator, AF_INET -> inet_create
}
```

3) the AF_INET family creator: inet_create.

```text
// file: net/ipv4/af_inet.c
static int inet_create(struct net* net, struct socket* sock, int protocol, int kern){
    struct sock* sk;
    ...
lookup_protocol:                                                 // look for the requested type/protocol pair
    rcu_read_lock();
    list_for_each_entry_rcu(answer, &inetsw[sock->type], list){
        ...err checks...
    }
    ...err checks...
    sock->ops = answer->ops;                                     // assign inet_stream_ops to socket->ops
    answer_prot = answer->prot;                                  // get tcp_prot
    answer_flags = answer->flags;                                // get tcp_flags
    rcu_read_unlock();
    sk = sk_alloc(net, PF_INET, GFP_KERNEL, answer_prot);        // allocate sock obj, assign tcp_prot to sock->sk_prot
    sock_init_data(sock, sk);                                    // init sock obj
}
```

4) the struct inet_protosw definition.

```text
// when startup, insert all the elements in inetsw_array[] into the linked list inetsw
static struct inet_protosw inetsw_array[] = {
    {
        .type =       SOCK_STREAM,  // tcp
        .protocol =   IPPROTO_TCP,
        .prot =       &tcp_prot,
        .ops =        &inet_stream_ops,
        .flags =      INET_PROTOSW_PERMANENT | INET_PROTOSW_ICSK,
    },
    {
        .type =       SOCK_DGRAM,
        .protocol =   IPPROTO_UDP,
        .prot =       &udp_prot,
        .ops =        &inet_dgram_ops,
        .flags =      INET_PROTOSW_PERMANENT,
    },
    {
        .type =       SOCK_DGRAM,
        .protocol =   IPPROTO_ICMP,
        .prot =       &ping_prot,
        .ops =        &inet_sockraw_ops,
        .flags =      INET_PROTOSW_REUSE,
    },
    {
        .type =       SOCK_RAW,
        .protocol =   IPPROTO_IP,	/* wild card */
        .prot =       &raw_prot,
        .ops =        &inet_sockraw_ops,
        .flags =      INET_PROTOSW_REUSE,
    }
};
```

5) regist sk_data_ready callback function. 

```text
// file: net/core/sock.c
void sock_init_data(struct socket* sock, struct sock* sk){
    sk->sk_data_ready   = sock_def_readable;                // set socket data ready callback
    sk->sk_write_space  = sock_def_write_space;
    sk->sk_error_report = sock_def_error_support;
}
```

<hr>

### # sync blocking io case study III: cooperation between user and kernel
0 sample code for client-end networking BIO:

```text
int main(){
    int sk = socket(AF_INET, SOCK_STREAM, 0);
    connect(sk, ...);
    recv(sk, ...);
}
```

1 overview: the sync blocking way of accepting packets.

```txt
   ┌────────────────────────┐            ┌───────────┐         ┌──────────────┐
   │ user process(blocking) │<----(10)---┤ ksoftirqd │<--(7)---┤ software irq │<---------┐
   └─────────────┬───┬──────┘            └─────┬─────┘         └──────────────┘          |
                 |   |                         |                                         |
                 |   |  (2) blocking           |                                         |
              (1)|   └-----------------┐       |(8)                                      |
       ┌─────────┼─────────────────────┼───────┼──────┐                                  |
       │┌────────+────────┐     ┌─┬─┬─┬+┬───┬─┐|      │                                  |
       ││socket kernel obj├─────+ │ │ │ │   │ │|      │                                  |
       │└────────┬────────┘     └─┴─┴─┴─┴───┴─┘|      │                                  |
       │         |               proc wait que |      │                                  |
       │         |                             |      │                                  |(6)
       │┌─┬─┬─┬─┬+──┬─┬─┬─┐  (9) ┌───┐   ┌─────+────┐ │ ┌─────┐  ┌─────────────┐      ┌──┴──┐
       ││ │ │ │ │...│ │ │ │<-----┤skb│<--┤ringbuffer├─┼─┤ mem ├──┤bridge/memory├──────┤ cpu │
       │└─┴─┴─┴─┴───┴─┴─┴─┘      └───┘   └─────+────┘ │ └─────┘  │ controller  │      └──+──┘
       │  socket recv que                   (4)|      │          └─────────────┘  ┌──────┴───────┐
       └───────────────────────────────────────┼──────┘                           │ hardware irq │
                                               |                                  └──────+───────┘
───────────────────────────────────────────────|─────────────────────────────────────────|────────── [PCIe]
                                            ┌──┴──┐                                      |
            ------------(3)---------------->│ nic ├------------------(5)-----------------┘
                                            └─────┘

       1 create sock                         2 wait for recv  
       3 packets send to nic                 4 DMA frames to mem  
       5 issue hardware irq to notify cpu    6 handling hardware irq  
       7 issue software irq                  8 get packets from ringbuffer  
       9 push into socket recv que           10 wakeup the thread in waiting que (context switch)
```

2 procedure of syscall recv: the workflow from syscall to blocking the proc  
using strace cmd for tracking the syscall path, the procedure is as follows:

```txt
                                               │
                                            ┌──+───┐
                                            │ recv │
                                            └──┬───┘
                                        ┌──────+───────┐1.syscall                   user space
        --------------------------------┤   recvfrom   ├--------------------------------------
                                        └──────┬───────┘                          kernel space
            runtime of proc A in kernel space  │
           ┌───────────────────────────────────│──────────────────────────────┐
           │  struct socket                    │                              │
           │  ┌─────────┐               ┌──────+───────┐                      │
           │  │   ops   ├──────────────>│ inet_recvmsg │2.inet_stream_ops     │
           │  └─────────┘               └──────┬───────┘                      │
           │  struct sock                      │                              │
           │  ┌─────────┐                      │                              │
           │  │┌───────┐│               ┌──────+───────┐                      │
           │  ││sk_prot├┼──────────────>│  tcp_recvmsg │3.tcp_prot            │
           │  │└───────┘│               └───┬──┬──┬────┘                      │
           │  │┌───────┐│4.check the recvque|  │  │5.reset cur proc state     │
           │  ││recvque├┼─┐┌----------------┘  │  └──────────────┐            │
           │  │└───────┘│ │|                   │         ┌───────+──────┐     │
           │  │┌───────┐│┌++┬──┬──┬──┐         │    from │ TASK_RUNNING │     │
           │  ││ sk_wq │││  │  │  │  │(empty)  │         └───────┬──────┘     │
           │  │└───┬───┘│└──┴──┴──┴──┘         │       ┌─────────+─────────┐  │
           │  └────┼────┘sock recvque          │    to │ TASK_INTERRUPTABLE│  │
           │       │                           │       └─────────────┬─────┘  │
           │       │              ┌────────────┘                     │        │
           │       │              │add cur proc to wait que          │        │ proc A is peeled off
           │       │sock wait que┌+┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┐       │        │ from CPU, context switch
           │       └────────────>│ │ │ │ │ │ │ │ │ │ │ │ │ │ │       │        │ for the first time.
           └─────────────────────┤ │ │ │ │ │ │ │ │ │ │ │ │ │ ├───────│────────┘
                                 └─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┘ ┌─────+───────┐
                                  process wait que (blocking)  │  process B  │
                                                               └─────────────┘ 
                                                            6.proc A giveup cpu, 
                                                            kernel dispatch proc B
```

3 softirq module: the workflow when data arrived at nic  
a softirq is a software interrupt request in the kernel. it is a way for the kernel to defer works
which can be delayed without causing problems, and it's a way to improve kernel performance as this
free up the main kernel thread, let it focus on user-space applications.

softirqs are implemented using a special type of kernel thread called a softirqd.
the softirqd thread runs at lower priority than the main kernel thread, so it does not interfere with
the processing of user-space applications.

softirqs are used for tasks such as network packet processing, disk I/O, and timer management.

```txt
           ┌─────────────────────────────────────────────────────────┐
           │ [runtime of process A in kernel model]                  │
           │                                                         │
           │  struct socket  struct sock     current proc state      │
           │  ┌─────────┐    ┌─────────┐   ┌────────────────────┐    │
           │  │┌───────┐│    │┌───────┐│   │ TASK_INTERRUPTIBLE │    │
           │  ││  ops  │├────+│sk_prot││   └────────────────────┘    │
           │  │└───────┘│    │└───────┘│                             │
           │  └─────────┘    │┌───────┐│                             │
           │    ┌────────────┼┤recvque││                             │
           │    │            │└───────┘│                             │
           │    │            │┌───────┐│                             │
           │    │            ││ sk_wq ├┼────┐                        │
           │    │            │└───────┘│    │                        │  proc A is in CPU again,
           │  sock recv que  └─────────┘    │proc wait que(blocking) │  context switch for the
           │  ┌─+┬──┬──┬──┐                ┌+┬─┬─┬─┬─┬─┬─┬─┐         │  second time.
           │  │  │  │  │  │                │ │ │ │ │ │ │ │ │         │  ┌──────────────────┐
           │  │  │  │  │  │                │ │ │ │ │ │ │ │ ├───────────>│ proc running que │
           │  └─+┴──┴──┴──┘                └+┴─┴─┴─┴─┴─┴─┴─┘         │  └──────────────────┘
           └────│───────────────────────────│────────────────────────┘
                │                           │ only wakeup 1 proc even more than 1 block on the same sock
           ┌────│───────────────────────────│────────────────────────┐
           │ 2.save data to sock recvque   3.wakeup proc at waitque  │
           │  ┌─┴─────────────┐           ┌─┴─────────────────┐      │
           │  │ tcp_queue_rcv │           │ sock_def_readable │      │
           │  └───+───────────┘           └────────────+──────┘      │
           │      │   first call tcp_queue_rcv         │             │
           │      │   ┌─────────────────────┐   ┌──────┴────────┐    │
           │      └───┤ tcp_rcv_established ├──>│ sk_data_ready │    │
           │          └──────────+──────────┘   └───────────────┘    │
           │             ┌───────┴───────┐    then call sk_data_ready│
           │             │ tcp_v4_do_rcv │                           │
           │             └───────+───────┘                           │
           │              ┌──────┴─────┐                             │
           │              │ tcp_v4_rcv │                             │
           │              └──────+─────┘                             │
           │                     │   [software irq context: ksoftirq]│
           └─────────────────────│───────────────────────────────────┘
       1.packets arrive at nic┌──┴──┐
       ---------------------->│ nic │
                              └─────┘
```

the advantages of using softirq are: improved performance (defer works that can be delayed);
reduced latency (handling pkts & disk io in background); increased scalability (running on smp linux).

the disadvantages of using softirq are: complexity to implement & manage
(careful coordination with main kthread & other components); overhead introduced (the kernel state save & restore);
security issue (can be used to inject code in kernel, kernel privilege abuse)
 
<hr>

### # conclusion
so, from the above workflow chart, each time a client-end waiting for & receive a packets, the related process
will be peeled off from cpu & get the cpu back, in which 2 times of context switch is needed. 
if the user program is "networking IO-bound" workload, then CPU will be busy doing useless context switch again and again.

the networking IO model (just name it as single channel without reuse) cannot be used in server-end programs,
cause the above model are assuming each socket for each connection proc, and all others will blocking util they
are selected by the cpu dispatcher.

this BIO networking model can be used in client-end, in which the process will block to wait for incoming data
to perform the next steps.
but nowadays, some well encapsulated networking framework like golang/net are abandoned BIO like above.

<hr>

### # reference
ref: APUE  
ref: dive into linux networking
