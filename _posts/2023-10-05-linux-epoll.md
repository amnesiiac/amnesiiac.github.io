---
layout: post
title: "epoll (linux io multiplexing)"
author: "melon"
date: 2023-10-05 09:33
categories: "2023"
tags:
  - linux
  - ongoing
---

### # introduction
In blocking network IO, the user process is blocked waiting for a single socket connection.
The linux process frequent creation & context switches for wait & wakeup is expensive for supporting massive connection concurrency scenario.

Hence, we should let a single process to handle many connections, named as multiplexing. 
But then the problem becomes: how to find out which connection is "active"?

We could use a single non-blocking process with a inner for loop to traverse all sockets to checkout the active connection to keep going, but will meet the problem of low efficiency & hard to implement.

The epoll networking IO model is a way out.

<hr>

### # todo
0) sample code for epoll networking IO model (todo: add detailed client-end code):
```text
int main(){
    listen(fd, ...);
    cfd1 = accept(...);
    cfd2 = accept(...);
    efd = epoll_create(...);                   // create a epoll obj
    epoll_ct1(efd, EPOLL_CTL_ADD, cfd1, ...);  // add a connection to epoll for management
    epoll_ct2(efd, EPOLL_CTL_ADD, cfd2, ...);
    epoll_wait(efd, ...);                      // blocking wait for IO event from connection managed
}
```

<hr>

### # overview of epoll networking model
```txt
    
        ┌────────────┐      ┌─────────┐                  ┌──────────┐
        │epoll_create│      │epoll_ctl│                  │epoll_wait│
        └─────┬──────┘      └────┬────┘                  └───┬──+───┘              user space
    ----------┼------------------┼---------------------------┼--┼------------------------------
     ┌────────┼──────────────────┼───────────────────────────┼──┼────────────────┐
     │   (1.1)│             (1.2)│               (1.3)┌──────┘  ├------------------------┐
     │   ┌────+────┐             │                    │         └──────┐         │  (1.4)|
     │   │┌───────┐│             │                   ┌+┐     ┌─┐       │         │  ┌────+────┐
     │   ││rdllist├┼─────────────│───────────────────+ ├─────+ │       │(2.5)    │  │process B│
     │   │└───────┘│             │    list of epitems│ +─────┤ │       │         │  └─────────┘
     │   │         │             │        of ready fd└+┘     └─┘       │         │
     │   │┌───────┐│             │                    │               ┌┴┐     ┌─┐│
     │   ││wq     ├┼─────────────│────────────────────│───────────────+ ├─────+ ││list of user processes 
     │   │└───────┘│             │                    │               │ +─────┤ ││blocked on epoll obj
     │   │┌───────┐│            ┌+┐                   │               └+┘     └─┘│
     │   ││rbr    ├┼────────────+b│                   │(2.3)           │         │
     │   │└───────┘│            └┬┘                   │                │         │
     │   └─────────┘        ┌────┴────┐               │                │         │
     │    eventpoll        ┌┴┐       ┌┴┐         ┌────┴───┐    (2.4)   │         │
     │                     │r│       │r│       ┌-+ epitem ├────────────┘         │
     │                     └┬┘       └┬┘       | └────┬───┘                      │
     │                   ┌──┴──┐   ┌──┴──┐     |      │(2.2)                     │
     │                  ┌┴┐   ┌┴┐ ┌┴┐   ┌┴┐    | ┌────┴───┐                      │
     │                  │b│   │b│ │b│   │b│    | │ socket │                      │
     │                  └┬┘   └─┘ └─┘   └┬┘    | └────┬───┘                      │
     │                   └──┐         ┌──┴──┐  |      │(2.2)                     │
     │                     ┌┴┐       ┌┴┐   ┌┴┐ | ┌────┴───┐                      │
     │                     │r│       │r│   │r│ | │  irqs  │                      │
     │                     └─┘       └┬┘   └─┘ | └────┬───┘                      │
     │                                └--------┘      │                          │
     └────────────────────────────────────────────────┼──────────────────────────┘
                                 (2.1)           ┌────┴───┐
                        ------------------------>│  nics  │
                                                 └────────┘

1) as the recv pipeline of epoll model in kernel mode, the preparation steps are:
1.1 epoll_create: create an eventpoll obj.
1.2 epoll_ctl: add socket for listening, and insert the listened epitem obj into self-maintained rb-tree.
1.3 epoll_wait: check the rdllist of eventpoll obj, findout if any socket listened by cur proc reach ready state.
1.4 epoll_wait: enter block state and giveup cpu occupytion if none of its listening sockets are ready within timeout.

2) as lower layer accepting pkts and 'notify' epoll model, the steps are:
2.1 the data packets arrived at nic.
2.2 the hw-irq & sw-irq context handler module put the data into socket recvque, and find socket related epitem.
2.3 check the listening epitem rb-tree of current process to findout whether current data pkts is waited by this process.
2.3 insert the epitem into rdllist, if the data meet the need.
2.4 check if any process is blocking on wq.
2.5 wakeup another user process if needed, and return the eventpoll obj back.
```

<hr>

### # relationship between eventpoll & process task_struct
in epoll model, a single user process could listen for many sockets, the mechanism to associate them together is as follows: 
```txt

      struct task_struct
     ┌───────────────────────────┐
     │┌─────────────────────────┐│
     ││volatile long state      ││                     ┌───────────┐                 ┌─────────────┐
     │└─────────────────────────┘│                ┌───>│struct file├────────────────>│struct socket│
     │┌─────────────────────────┐│     ┌───────┐  │    └───────────┘                 └─────────────┘
     ││pid_t pid                ││  ┌──+ *file │  │                                
     │└─────────────────────────┘│  │  ├───────┤  │    ┌───────────┐                 ┌─────────────┐
     │┌─────────────────────────┐│  │  │ 0     │  │ ┌─>│struct file├────────────────>│struct socket│
  ┌──┼┤struct file_struct* files││  │  ├───────┤  │ │  └───────────┘                 └─────────────┘
  │  │└─────────────────────────┘│  │  │ 1     │  │ │                 
  │  └───────────────────────────┘  │  ├───────┤  │ │   struct file                   struct eventpoll
  │                                 │  │ ....  │  │ │  ┌───────────────────────┐     ┌──────────────────────────┐
  │   struct files_struct           │  ├───────┤  │ │  │┌─────────────────────┐│     │┌────────────────────────┐│
  │  ┌─────────────────────┐        │  │ 5000  ├──┘ │  ││struct path f_path   ││  ┌─>││struct file* file       ││
  └─>│┌───────────────────┐│        │  ├───────┤    │  │└─────────────────────┘│  │  │└────────────────────────┘│
  ┌──┼┤struct fdtable* fdt││        │  │ 5001  ├────┘  │┌─────────────────────┐│  │  │┌────────────────────────┐│
  │  │└───────────────────┘│        │  ├───────┤       ││struct inode* f_inode││  │  ││struct list_head rdllist││
  │  └─────────────────────┘        │  │ 5002  ├──────>│└─────────────────────┘│  │  │└────────────────────────┘│
  │                                 │  ├───────┤       │┌─────────────────────┐│  │  │┌────────────────────────┐│
  │   struct fdtable                │  │ ....  │       ││         ...         ││  │  ││struct rb_root rbr      ││
  │  ┌──────────────────┐           │  └───────┘       │└─────────────────────┘│  │  │└────────────────────────┘│
  └─>│┌────────────────┐│           │                  │┌─────────────────────┐│  │  └──────────────────────────┘
     ││struct file** fd├┼───────────┘                  ││void* private_data   ├┼──┘
     │└────────────────┘│                              │└─────────────────────┘│ 
     └──────────────────┘                              └───────────────────────┘

```

<hr>

### # details of epoll key data structure obj: eventpoll
```txt

      struct eventpoll
     ┌────────────────────┐     listnode
     │┌──────────────────┐│     ┌─────┐     ┌─────┐
     ││wait_queue_head_t ├┼─────+     ├─────+     │ process wait que used by sys_epoll_wait
     ││       wq         +┼─────┤     +─────┤     │
     │└──────────────────┘│     └─────┘     └─────┘
     │┌──────────────────┐│     ┌─────┐     ┌─────┐
     ││ struct list_head ├┼─────+     ├─────+     │ descriptor que in ready state for acception
     ││     rdllist      +┼─────┤     +─────┤     │
     │└──────────────────┘│     └─────┘     └─────┘
     │┌──────────────────┐│     ┌─────────────────┐                       ┌─────────────┐
     ││struct rb_root rbr├┼─────+ red-black tree  │ each rbtree node is a │struct epitem│
     │└──────────────────┘│     └─────────────────┘                       └─────────────┘
     └────────────────────┘     each epoll obj maintains a rbtree

1) wq: processes wait que implemented by list, recording all the processes that blocked on current epoll obj.
   when data comes through nic to trigger irq context, the corresponding user process can be founded by wq.

2) rbr: red-black tree to support for efficient search, insert, delete operations of epitems, 
   in order to manage all the socket connections added by user process.

3) rdllist: a linked list of file descriptors from the socket connections in ready state, 
   thus the user process only need to traversal this list to find out which connection is in ready state, without search into the rbtree.
```

<hr>

### # epoll_ctl: add socket connection into epoll obj (eventpoll) listening tree
```txt
                                                                                 struct epitem
  ┌───────┐     ┌───────────────┐    struct sock      sock wait que             ┌────────────────────┐
  │ *file │ ┌──>│ struct file   │    ┌─────────┐     ┌─────────────┐            │┌──────────────────┐│
  ├───────┤ │   └───────┬───────┘    │┌───────┐│  ┌──┼>func | base─┼────────────+│ epoll_filefd ffd ││
  │ 0     │ │   ┌───────+───────┐    ││ sk_wq ├┼──┘  ├─────────────┤            │└──────────────────┘+---┐
  ├───────┤ │   │ struct socket ├────+└───────┘│     │      |      │            │┌──────────────────┐│   |
  │ 1     │ │   └───────────────┘    └─────────┘     └─────────────┘            ││ eventpoll *ep    ││   |
  ├───────┤ │                                   (func: ep_poll_callback)        │└──────────────────┘│   |
  │ ....  │ │                                                                   └────────────────────┘   |
  ├───────┤ │                                                                                            |
  │ 5000  ├─┘                                                                    struct epitem           |
  ├───────┤     ┌───────────────┐    struct sock      sock wait que             ┌────────────────────┐   |
  │ 5001  ├────>│ struct file   │    ┌─────────┐     ┌─────────────┐            │┌──────────────────┐│   |
  ├───────┤     └───────┬───────┘    │┌───────┐│  ┌──┼>func | base─┼────────────+│ epoll_filefd ffd ││   |
  │ 5002  ├─┐   ┌───────+───────┐    ││ sk_wq ├┼──┘  ├─────────────┤            │└──────────────────┘+-┐ |
  ├───────┤ │   │ struct socket ├────+└───────┘│     │      |      │            │┌──────────────────┐│ | |
  │ ....  │ │   └───────────────┘    └─────────┘     └─────────────┘    ┌───────┼┤ eventpoll *ep    ││ | |
  └───────┘ │                                   (func: ep_poll_callback)│       │└──────────────────┘│ | |
            │                                                           │       └────────────────────┘ | |
            │    struct file             struct eventpoll               │               ┌─┐            | |
            │   ┌────────────────────┐  ┌──────────────────────────┐    │  ┌────────────+b├------------┘ |
            │   │┌──────────────────┐│  │┌────────────────────────┐│    │  │            └┬┘              |
            └───+│void* private_data├┼──+│struct file* file       │+────┘  │        ┌────┴────┐          |
                │└──────────────────┘│  │└────────────────────────┘│       │       ┌┴┐       ┌┴┐         |
                │┌──────────────────┐│  │┌────────────────────────┐│       │       │r│       │r├---------┘
                ││       ...        ││  ││struct list_head rdllist││       │       └┬┘       └┬┘        
                │└──────────────────┘│  │└────────────────────────┘│       │     ┌──┴──┐   ┌──┴──┐      
                └────────────────────┘  │┌────────────────────────┐│       │    ┌┴┐   ┌┴┐ ┌┴┐   ┌┴┐     
                                        ││struct rb_root rbr      ├┼───────┘    │b│   │b│ │b│   │b│     
                                        │└────────────────────────┘│            └┬┘   └─┘ └─┘   └┬┘
                                        └──────────────────────────┘             └──┐         ┌──┴──┐ 
                                                                                   ┌┴┐       ┌┴┐   ┌┴┐
                                                                                   │r│       │r│   │r│
                                                                                   └─┘       └─┘   └─┘
1) allocate a rb-tree node obj struct epitem.

2) insert the event (the socket that user process waiting for) into socket waitque wq, regist callback 
   function ep_poll_callback, if certain socket is ready will use ep_poll_callback to wake up the process.

3) insert epitem into red-black tree. The rb-tree is chosen for its balance behavior among search, insert, mem usages.
   hashtable is best for O(1) search, but not as balanced as rb-tree.
```

<hr>

### # epoll_wait: wait for incoming event
```txt
                                     │ 
                                ┌────┴─────┐                              user space
   -----------------------------│epoll_wait│----------------------------------------
                                └────┬─────┘                            kernel space
    ┌────────────────────────────────┼───────────────────────────────────────────┐
    │                                │                                           │
    │                                │ 1 check in rdllist whether eventpoll      │
    │   struct eventpoll             │ has socket in ready state                 │
    │  ┌───────────────────┐         │                                           │
    │  │┌─────────────────┐│       ┌─+─┐───────────┐   2 define waiting event    │
    │  ││struct list_head +┼───────┤   │           │   and add current proc into │
    │  ││    rdllist      ├┼───────+   │         ┌─+─┐ proc wait que (wq).       │
    │  │└─────────────────┘│       └───┘         │xxx│                           │
    │  │┌─────────────────┐│         ┌───────────┤xxx│                           │
    │  ││wait_queue_head_t+┼────┐    │           └───┘                           │
    │  ││       wq        ├┼───┐│    │                                           │
    │  │└─────────────────┘│   ││    │ 3 insert into wq of eventpoll             │
    │  │┌─────────────────┐│   ││  ┌─+─┐         ┌───┐                           │
    │  ││ struct rb_root  ││   │└──┤xxx+─────────┤   │                           │
    │  ││      rbr        ││   └───+xxx├─────────+   │                           │
    │  │└─────────────────┘│       └─┬─┘   (wq)  └───┘                           │
    │  └───────────────────┘         │                                           │
    │                                │ 4 give away cpu (blocking)                │
    └────────────────────────────────┼───────────────────────────────────────────┘
                              ┌──────┴─────┐
                              │another proc│
                              └────────────┘
 1) epoll_ctl and epoll_wait create a waique both, but the difference is that: the wait que (wq) is
    attached to the eventpoll obj, the wait que(for socket) created by epoll_ctl is attached to socket obj.
 
 2) wait que of eventpoll is used to connect a single eventpoll obj and all processes blocking on.
    wait que of sock is used to connect all sockets and corresponding epitems.
 
 3) epoll will block current process if no IO event related is active, but usually set the sockets which wait
    for incoming data as non-block. Block / non-block mode is meaningfull only we make its subject clear.
```

<hr>

### # data is coming
preparations: the data structure & callback waiting for data to invoke hw-irq -> sw-irq:
```txt
                       ┌-------------------------------------------------------------------------------┐
                       |                             sock wait que                                     |
                       |                               ┌──────┐                                        |
                       |                               │ null │                                        |
                       |                               └─+──┬─┘                    struct epitem       |
 ┌───────┐     ┌───────+───────┐    struct sock          │  │                  ┌────────────────────┐  |
 │ *file │ ┌──>│ struct file   │    ┌─────────┐     ┌────┴──+─────┐            │┌──────────────────┐│  |
 ├───────┤ │   └───────┬───────┘    │┌───────┐│  ┌──+  *private   │            ││ epoll_filefd ffd ├┼--┘
 │ 0     │ │   ┌───────+───────┐    ││ sk_wq ├┼──┘  ├─────────────┤            │└──────────────────┘│
 ├───────┤ │   │ struct socket ├────+└───────┘│  ┌──┼>func | base─┼────────────+┌──────────────────┐│
 │ 1     │ │   └───────────────┘    └─────────┘  │  └───+─────┬───┘            ││ eventpoll *ep    ││
 ├───────┤ │                                     │      │     │                │└────────┬─────────┘│
 │ ....  │ │                         ┌───────────┘      │     │                └─────────┼──────────┘
 ├───────┤ │               ┌─────────+────────┐     ┌───┴─────+───┐                      |
 │ 5000  ├─┘               │ ep_poll_callback │     │  *private   │                      |
 ├───────┤                 └──────────────────┘     ├─────────────┤                      |
 │ 5001  │                                          │ func | base │                      |
 ├───────┤                                          └───+─────┬───┘                      |
 │ 5002  ├─┐                                            │     │                          |
 ├───────┤ │                                              ...                            |
 │ ....  │ │                                                                             |
 └───────┘ │    struct file             struct eventpoll <-------------------------------┘
           │   ┌────────────────────┐  ┌──────────────────────────┐   proc wait que
           │   │┌──────────────────┐│  │┌────────────────────────┐│   ┌──────────┐   ┌──────────────┐
           └───+│void* private_data├┼──+│wait_queue_head_t wq    ├┼───+ *private ┼───+ current proc │
               │└──────────────────┘│  │└────────────────────────┘│   ├──────────┤   └──────────────┘
               └────────────────────┘  │┌────────────────────────┐│   │  *func   ┼─┐ ┌───────────────────────┐
                                       ││struct list_head rdllist││   └──────────┘ └─+ default_wake_function │
                                       │└────────────────────────┘│                  └───────────────────────┘
                                       │┌────────────────────────┐│
                                       ││struct rb_root rbr      ││
                                       │└────────────────────────┘│
                                       └──────────────────────────┘ 

```
data arrived, the pipeline of softirq context:
```txt

     ┌──────────────────────────────────────────────────────────────────────────────┐
     │   struct socket               struct sock                                    │
     │  ┌─────────────┐             ┌────────────┐                                  │
     │  │┌───────────┐│             │┌──────────┐│                                  │
     │  ││   *sk     │├─────────────+│ sk_prot  ││                                  │
     │  │└───────────┘│             │└──────────┘│                  sock wait que   │
     │  │┌───────────┐│             │┌──────────┐│                 ┌──────────────┐ │
     │  ││   ops     ││      ┌──────┼┤ recvque  ││   ┌─────────────+  *private    │ │
     │  │└───────────┘│      │      │└──────────┘│   │             ├──────────────┤ │
     │  └─────────────┘      │      │┌──────────┐│   │             │ *func | base │ │
     │                       │      ││ sk_wq    +┼───┘             └───+─────┬────┘ │
     │          ┌────────────┘      │└──────────┘│                     │     │      │
     │          │ sock recv que     └────────────┘                 ┌───┴─────+────┐ │
     │         ┌+┬─┬─┬─┬─┬─┬─┬─┬─┐         (set by epoll_ctl) null─┼──*private    │ │
     │         │ │ │ │ │ │ │ │ │ │         ┌──────────────────┐    ├──────────────┤ │
     │         │ │ │ │ │ │ │ │ │ │         │ ep_poll_callback +────┼─*func | base │ │
     │         └+┴─┴─┴─┴─┴─┴─┴─┴─┘         └───────────+──────┘    └──────────────┘ │
     └──────────┼──────────────────────────────────────┼────────────────────────────┘
                │                                      │ 4.exec the cb func registed in sock waitque
           ┌────│──────────────────────────────────────│─────────────┐
           │ 2.save data to sock recvque   3.judge if waitque empty  │
           │  ┌─┴─────────────┐           ┌────────────┴──────┐      │
           │  │ tcp_queue_rcv │           │ sock_def_readable │      │
           │  └───+───────────┘           └────────────+──────┘      │
           │      │   first call tcp_queue_rcv         │             │
           │      │   ┌─────────────────────┐   ┌──────┴────────┐    │
           │      └───┤ tcp_rcv_established ├──>│ sk_data_ready │    │(software irq context: ksoftirq)
           │          └──────────+──────────┘   └───────────────┘    │             
           │             ┌───────┴───────┐    then call sk_data_ready│
           │             │ tcp_v4_do_rcv │                           │
           │             └───────+───────┘                           │
           │              ┌──────┴─────┐                             │
           │              │ tcp_v4_rcv │                             │
           │              └──────+─────┘                             │
           └─────────────────────│───────────────────────────────────┘
       1.packets arrive at nic┌──┴──┐
       ---------------------->│ nic │
                              └─────┘

```
execute ep_poll_callback registered by epoll_ctl when adding sock for listening:
```txt

     ┌────────────────────────────────────────────────────────────────────────────────────────┐
     │  struct file             struct eventpoll          proc wait que                       │
     │ ┌────────────────────┐  ┌──────────────────────┐  ┌────────────┐                       │
     │ │┌──────────────────┐│  │┌────────────────────┐│  │┌──────────┐│  ┌──────────────┐     │
     │ ││void* private_data├┼──+│wait_queue_head_t wq├┼──+│ *private ├┼──+ current proc +───┐ │
     │ │└──────────────────┘│  │└────────────────────┘│  │├──────────┤│  └──────────────┘   │ │
     │ └────────────────────┘  └──────────────────────┘  ││  *func   ├┼───────┐             │ │
     │                                 (2,3,4)           │└──────────┘│       │          (6)│ │
     │                                                   └─────+──────┘       │             │ │
     │                                                         │  ┌───────────+───────────┐ │ │
     │                                                      (5)│  │ default_wake_function ├─┘ │
     │                                                         │  └───────────────────────┘   │
     └─────────────────────────────────────────────────────────┼──────────────────────────────┘
                                                   ┌───────────┴──────┐
                                                (1)│ ep_poll_callback │
                                                   └──────────────────┘
  1) execute callback function: ep_poll_callback.
  2) ep_poll_callback will find related epitem by the base ptr from socket wait que
  3) it can find the related eventpoll obj by ptr to eventpoll obj (ep) in epitem.
  4) then add related epitem into ready list (rdllist) of the eventpoll.
  5) finally check the proc wait que of the eventpoll, if any proc waiting for this epitem.
     if no proc waiting for cur epitem, ksoftirq's business is done.
     if some proc is waiting for it, will find the callback func (default_wake_function).
  6) inside default_wake_function, will find the proc descriptor (blocking), then wake it up.
```
the proc pointed by *private of the proc wait que is blocking on the execution of epoll_wait syscall, 
which will push the current proc into RUNNABLE queue, waiting for kernel re-dispatch.  
when the blocked process is recovered, then will continue running from the epoll_wait() func call, 
then return the available event inside rdllist to user process.
