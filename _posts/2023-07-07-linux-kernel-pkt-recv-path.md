---
layout: post
title: "packets transmission from nic to proto stack (pkts receive path)"
author: "melon"
date: 2023-07-07 22:22
categories: "2023"
tags:
  - network
---

### # network proto stack: osi networking layer model
```txt
                       ┌───────────────────┐
                       │ application layer +────> HTTP/FTP
     user              └──────┬────+───────┘
     space                  ┌─+────┴─┐
     -----------------------┤ socket ├------------------------
                            └─┬────+─┘
                        ┌─────+────┴──────┐
                        │ transport layer ├─────> TCP/UDP
                        └─────┬────+──────┘
           ┌ protocol stack   │    │  
           │            ┌─────+────┴──────┐
           │            │  network layer  ├─────> IP/ICMP/IGMP
     kernel│            └─────┬────+──────┘
     space │ -----------------┼----┼--------------------------
           │            ┌─────+────┴──────┐
           └ driver     │ data link layer ├─────> ATM/ARP/SLIP/CSLIP/LLC/MAC/PPP
                        └─────┬────+──────┘
     -------------------------┼----┼--------------------------
                        ┌─────+────┴──────┐
                        │ physical layer  ├─────> IEEE 802.3/nic/
                        └─────────────────┘
```

<hr>

### # linux packets processing path
```txt
                                                  ┌────────────┐
                                                  │user process│
                                                  └─────────+──┘
          ┌─────┐ (4)                                    (8)|
   ┌------+ CPU ├---------┐     ┌───────────────────────────|─────┐
   |      └──+──┘         |     │ user process in kernel space    │
   |         │            |     │                           |     │
   |         │            |     │              ┌────────────┴─┐   │
   |         │           (5)    │              │protocol layer│(7)│
   | ┌───────┴───────┐ ksoftirq │              └────────────+─┘   │
(3)| │ bridge/memory │    |     │                           |     │
   | │  controller   +────|─────+───────────────────────────|─────┤
   | └───────┬───────┘    |     │                           |     │
   |         │            |     │   ┌───────────┐         ┌─┴─┐   │
   |         │            └---------+ring buffer├---(6)---+skb│   │
   |         │                  │   └─────+─────┘         └───┘   │
   |         │                  │         |                       │
   |         │                  │ kernel process in kernel space  │
   |         │                  └─────────|───────────────────────┘
   |         │                           (2)                          [PCIe]
───|─────────┴─────────────────────────┬──|─────────────────────────────────────
   |                                   │  |
   |                          ┌────────┴──┴────────────┐ (1) ┌─────────┐
   └-----------(3)------------+ network interface card +-----┤ network │
                              └────────────────────────┘     └─────────┘

1) data frame transfer from network to nic.
2) nic use DMA to send data frame to memory.
3) nic notify CPU by hardware irq. 
4) CPU response for hardware irq, issues a software irq after a short procedure.
5) ksoftirqd thread processing softirq, invoking the poll func registed by nic driver to receive pkts.
6) data frames are derived from ring buffer to save as a skb.
7) protocol layer processing data frame, the derived data is placed in socket RX queue.
8) kernel wakeup user process.
```

<hr>

### # prepararions of linux kernel before receiving packets
before linux kernel could receive packets, a lot of preparations need to be done:  
creation of ksoftirqd threads, registration of handlers of each protocol layer, initialization of net subsystems, 
nic need to be ready.
the following steps shows details of some preparations.

1) the creation of ksoftirqd threads(N) by linux kernel  
```txt
                                                                        ┌-------(3)-------┐
                                                               ┌────────┴────────────┐    |
                 ┌───────────────────────────────┐             │ kernel/softirq.c:   │    |
    -----(1)-----+ kernel/smpboot.c:             ├-----(2)-----+ spawn_ksoftirqd     +----┘
                 │ smpboot_register_percpu_thread│             │ ksoftirqd_should_run│
                 └───────────────────────────────┘             │ run_ksoftirqd       │
                                                               └─────────────────────┘
                                                               
   1 system init          2 spawn_ksoftirqd threads         3 regist threads looping functions

```                  

2) after the ksoftirqd threads are created, it will run into self-looping function: 
ksfotirqd_should_run and run_ksoftirqd, in which waiting for incoming softirq to handle.  
softirq types and definition in linux kernel (network irq & other types of irq) is as:
```text
// file: include/linux/interrupt.h
enum {
    HI_SOFTIRQ=0,
    TIMER SOFTIRQ,
    NET_TX SOFTIRQ,
    NET_RX_SOFTIRQ,
    BLOCK SOFTIRO,
    BLOCK_IOPOLL_SOFTIRQ,
    TASKLET_SOFTIRQ,
    SCHED_SOFTIRQ,
    HRTIMER_SOFTIRQ,
    RCU_SOFTIRQ,
    NR_SOFTIRQS
};
```

3) network subsystem initialization  
```txt
                            3. regist NET_RX_SOFTIRQ handler: net_rx_action
                                      NET_TX_SOFTIRQ handler: net_tx_action 
subsystem init ┌─────────────────┐             ┌───────────────────┐
------(1)----->│ net/core/dev.c: ├-----(3)---->│ kernel/softirq.c: │
(net_dev_init) └────────+────────┘             └──────────+────────┘
                        | 2. init softnet_data            |(4)     4. record softirq -> handler
                        |    for each CPU.         ┌─┬─┬─┬+┬───┬─┐ pair info into soft_irq_vec
     ┌------------------+-------------------------┐│ │ │ │ │...│ │    
     |// file: include/linux/netdevice.h          |└─┴─┴─┴─┴───┴─┘
     |struct softnet_data{                        |   soft_vec
     |    struct Qdisc        *output_queue;      |
     |    struct Qdisc        *output_queue_tailp;|
     |    struct list_head     poll_list;         |
     |    struct sk_buff      *completion_queue;  |
     |    struct sk_buff_head  process_queue;     |
     |    ...                                     |
     |}                                           |
     └--------------------------------------------┘
 
```
step 2: linux kernel invokes subsys_initcall() to init all subsystems. 
to init network subsystem, need pass net_dev_init to subsys_initcall():
```text
// file: net/core/dev.c
static int __init net_dev_init(void){
    ...
    for_each_possible_cpu(i){
        // allocate softnet_data for each cpu
        struct softnet_data* sd = &per_cpu(softnet_data, i);
        memset(sd, 0, sizeof(*sd));
        skb_queue_head_init(&sd->input_pkt_queue);
        skb_queue_head_init(&sd->process_queue); 
        sd->completion_queue = NULL;
        // poll_list in softnet_data is placeholder for poll func registed by nic driver
        INIT_LIST_HEAD(&sd->poll_list);
        ...
    }
    // regist handler function "net_(t/r)x_action" for each soft irq (TX/RX)
    open_softirq(NET_TX_SOFTIRQ, net_tx_action);  
    open_softirq(NET_RX_SOFTIRQ, net_rx_action);
}
subsys_initcall(net_dev_init);
```
step 4: using softirq_vec to hold the handler for TX/RX soft irqs.
```text
// file: kernel/softirq.c
void open_softirq(int nr, void (*action)(struct softirq_action *)){ // the handler func is registed in softirq vector
    softirq_vec[nr].action = action;
}
```

4) network protocol stack registration process  
linux kernel implemented network layer IP protocol, and transport layer TCP/UDP protocol.
the implentation functions are 'ip_rcv()', 'tcp_v4_rcv()', 'udp_rcv()'.
these function implementations includes protocol processing procedure.  
the fs_initcall() is similar as sybsys_initcall(), which are also an entrance to initialize kernel modules.
```txt
                                     ┌───────────────────┐
                                     │ inet_add_protocol │  1. regist udp_rcv() func
                               ┌---->│  (&udp_protocol)  ├------┐       inet_protos vector
                               |     └───────────────────┘      |       ┌─┬─┬─┬─┬───┬─┐
                               |      net/ipv4/protocol.c       ├------>│ │ │ │ │...│ │
                               |                                |       └─┴─┴─┴─┴───┴─┘
fs_initcall ┌─────────────┐    |     ┌───────────────────┐      |       net/core/dev.c
----------->│ inet_init() ├----+---->│ inet_add_protocol ├------┘    
            └─────────────┘    |     │  (&tcp_protocol)  │  2. regist tcp_v4_rcv() func
           net/ipv4/af_inet.c  |     └───────────────────┘
                               |      net/ipv4/protocol.c
                               |                                        ptype_base hash table
                               |     ┌───────────────────┐   (3)        ┌─┬─┬─┬─┬───┬─┐
                               └---->│   dev_add_pack()  ├------------->│ │ │ │ │...│ │
                                     └───────────────────┘              └─┴─┴─┴─┴───┴─┘
                                       net/core/dev.c                   net/core/dev.c 
                                                          3. regist ip_rcv() func
```

5) network interface card driver initialization
every driver program will regist its init function to kernel by 'module_init'.
when driver program is loaded, then kernel will call the registed init function.  
the igb nic driver program:
```text
// file: drivers/net/ethernet/intel/igb/igb_main.c
static struct pci_driver igb_driver = { // handlers
    .name     = igb_driver_name,
    .id_table = igb_pci_tbl,
    .probe    = igb_probe,              // get nic state ready after nic is recognized by kernel
    .remove   = igb_remove,
    ...
};
```
the igb driver program register function:
```text
static int __init igb_init_module(void){
    ...
    ret = pci_register_driver(&igb_driver);
    return ret;
}
```
the operations of 'igb_probe' function:
```txt
                                                     4. DMA initialization
                          (4) (5) (6) (7)            5. regist ethtool functions                       
                 ┌---------------------------------┐ 6. regist net_device_ops, netdev variables        
                 |                                 | 7. NAP initialization, poll function registration 
                 |                                 |  
1. startup   ┌───+────┐2. call probe method ┌──────┴─────┐3. derive mac address ┌─────┐
------------>│ kernel ├-------------------->│ nic driver ├--------------------->│ nic │
             └────────┘                     └────────────┘                      └─────┘
```
step5: when ethtool uses syscall, kernel will find the registered nic callback functions.
For igb, the ethtools callbacks are defined in drivers/net/ethernet/intel/igb/igb_ethtool.c.
That is, ethtool itself cannot check nic RX/TX statistics or adjust the nic RX/TX queue size... 
But with the help of nic driver api, ethtool can achieve that.  

step6: register handler in net_device_ops data structure.
The handler igb_open will be invoked when nic startup.
```txt
// file: drivers/net/ethernet/intel/igb/igb_main.c
static const struct net_device_ops igb_netdev_ops = {  // all callback functions
    .ndo_open            = igb_open,                   // set nic up (ifconfig eth0 up)
    .ndo_stop            = igb_close,                  // set nic down (ifconfig eth0 down)
    .ndo_start_xmit      = igb_xmit_frame,             //
    .ndo_get_stats64     = igb_set_stats64,
    .ndo_set_rx_mode     = igb_set_rx_mode,            // 
    .ndo_set_mac_address = igb_set_mac,                // set mac addr
    .ndo_change_mtu      = igb_change_mtu,             // set nic mtu
    .ndo_do_ioctl        = igb_ioctl,
    // ...
};
```
step7: NAPI todo


6) network interface card startup (regist net_device_ops)
```txt
                   ┌-------------------------------------------------------┐
                   |      3. allocate RX/TX queue memory                   |
                   |                                                       |
1. nic startup ┌───+────┐ 2. call registed method in net_device_ops ┌──────┴─────┐
-------------->│ kernel ├------------------------------------------>│ nic driver │
               └───+────┘    e.g. igb_open (nic up)                 └──────┬─────┘
                   |                                                       |
                   |      4. regist irq handler                            |
                   └-------------------------------------------------------┘
                          5. open hardirq, wait for pkts
```
The igb_open source code:
```text
// file: drivers/net/ethernet/intel/igb/igb_main.c
static int _igb_open(struct net_device* netdev, bool resuming){
    err = igb_setup_all_tx_resources(adaptor);         // allocate tx descriptor vector
    err = igb_setup_all_rx_resources(adaptor);         // allocate rx descriptor vector (ringbuffer)
                                                       // buildup mapping between ringbuffer and memory
    err = igb_request_irq(adaptor);                    // regist irq handler
    if(err){
        goto err_req_irq;
    }
    for(i=0; i<adaptor->num_q_vectors; i++){           // enable napi
        napi_enable(&(adaptor->q_vector[i]->napi));
    }
}
```
specially, the igb_setup_all_rx_resources code:
```text
static int igb_setup_all_rx_resources(struct igb_adaptor* adaptor){
    ...
    for(i=0; i<adaptor->num_rx_resources; i++){             // more than one ring buffer is created
        err = igb_setup_rx_resources(adaptor->rx_ring[i]);
    }
}
```
the created ringbuffers (receive queue):
```txt
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│     ┌─┐  ┌─┐               ┌─┐  ┌─┐                ┌─┐  ┌─┐                ┌─┐  ┌─┐     │
│  ┌──└─┘──└─┘──┐         ┌──└─┘──└─┘──┐          ┌──└─┘──└─┘──┐          ┌──└─┘──└─┘──┐  │
│ ┌┴┐          ┌┴┐       ┌┴┐          ┌┴┐        ┌┴┐          ┌┴┐        ┌┴┐          ┌┴┐ │
│ └┬┘          └┬┘       └┬┘          └┬┘        └┬┘          └┬┘        └┬┘          └┬┘ │
│  │ ringbuffer │         │ ringbuffer │          │ ringbuffer │          │ ringbuffer │  │
│ ┌┴┐          ┌┴┐       ┌┴┐          ┌┴┐        ┌┴┐          ┌┴┐        ┌┴┐          ┌┴┐ │
│ └┬┘          └┬┘       └┬┘          └┬┘        └┬┘          └┬┘        └┬┘          └┬┘ │
│  └──┌─┐──┌─┐──┘         └──┌─┐──┌─┐──┘          └──┌─┐──┌─┐──┘          └──┌─┐──┌─┐──┘  │
│     └─┘  └─┘               └─┘  └─┘                └─┘  └─┘                └─┘  └─┘     │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```
the ringbuffer creation:
```text
// file: drivers/net/ethernet/intel/igb/igb_main.c
// igb_setup_rx_resources - allocate Rx resources (Descriptors)
// @rx_ring: Rx descriptor ring (for a specific queue) to setup
// Returns 0 on success, neg on failure
int igb_setup_rx_resources(struct igb_ring *rx_ring){
    ...
    // allocate mem for igb_rx_buffer
    size = sizeof(struct igb_rx_buffer) * rx_ring->count;
    rx_ring->rx_buffer_info = vmalloc(size);
    if(!rx_ring->rx_buffer_info){
        goto err;
    }
    // allocate mem for e1000 DMA array /* Round up to nearest 4K */
    rx_ring->size = rx_ring->count * sizeof(union e1000_adv_rx_desc);
    rx_ring->size = ALIGN(rx_ring->size, 4096);
    rx_ring->desc = dma_alloc_coherent(dev, rx_ring->size, &rx_ring->dma, GFP_KERNEL);
    if (!rx_ring->desc){
        goto err;
    }
    // init ring buffer member
    rx_ring->next_to_alloc = 0;
    rx_ring->next_to_clean = 0;
    rx_ring->next_to_use = 0;
    rx_ring->xdp_prog = adapter->xdp_prog;
    return 0;
err:
    xdp_rxq_info_unreg(&rx_ring->xdp_rxq);
    vfree(rx_ring->rx_buffer_info);
    rx_ring->rx_buffer_info = NULL;
    dev_err(dev, "Unable to allocate memory for the Rx descriptor ring\n");
    return -ENOMEM;
}
```
for each 'ringbuffer', there are mainly two array buffer to be initiated, one is igb_rx_buffer which is used by kernel, 
the other is e1000_adv_rx_desc which is used by nic hardware and it is allocated by dma_alloc_coheret.
```txt
                          ┌─────────────────────────────────────────┐
     ┌─┐  ┌─┐             │ ptr_vec(kernel)     bd_vec(nic)         │
  ┌──└─┘──└─┘──┐          │ igb_rx_buffer       e1000_adv_rx_desc[] │
 ┌┴┐          ┌┴┐         │                                         │
 └┬┘          └┬┘         │ ┌───────┐     ┌───┐     ┌───────┐       │
  │ ringbuffer │  ----->  │ │   1   ├---->│skb│<----┤   1   │       │
 ┌┴┐          ┌┴┐         │ ├───────┤     ├───┤     ├───────┤       │
 └┬┘          └┬┘         │ │   2   ├---->│skb│<----┤   2   │       │
  └──┌─┐──┌─┐──┘          │ ├───────┤     ├───┤     ├───────┤       │
     └─┘  └─┘             │ │   3   ├---->│skb│<----┤   3   │       │
                          │ ├───────┤     ├───┤     ├───────┤       │ 
                          │ │   4   ├---->│skb│<----┤   4   │       │
                          │ └───────┘     └───┘     └───────┘       │
                          │    ...                     ...          │ 
                          │ ├───────┤               ├───────┤       │
                          │ │  512  │               │  512  │       │
                          │ └───────┘               └───────┘       │
                          └─────────────────────────────────────────┘
```
7) regist irq handler functions
```text
// igb_request_irq - initialize interrupts
// @adapter: board private structure to initialize
// 
// Attempts to configure interrupts using the best available
// capabilities of the hardware and kernel.
static int igb_request_irq(struct igb_adapter *adapter){
    struct net_device *netdev = adapter->netdev;
    struct pci_dev *pdev = adapter->pdev;
    int err = 0;

    if(adapter->flags & IGB_FLAG_HAS_MSIX){
        // call igb_request_msix
        err = igb_request_msix(adapter);
        if(!err){
            goto request_done;
        }
        ...
    }
    ...
request_done:
	return err;
}
```
each RX queue has its own irq handler: igb_rx_ring
```text
static int igb_request_msix(struct igb_adapter *adapter){
    ...
    for(i=0; i<num_q_vectors; i++){
        ...
        // the actual irq handler is 'igb_msix_ring'
        err = request_irq(adapter->msix_entries[vector].vector,
                          igb_msix_ring, 0, q_vector->name, q_vector);
    }
    ...
}
```

<hr>

### # accepting data packets
1) hardware irq handling by functions registed by nic driver
```txt
                        3. send hw irq ┌─────┐ 4. invoke hw irq handlers
                              ┌------->│ nic │<-┐ registed by driver
                              |        └─────┘  |
                              |                 |      5. startup NAPI, ┌───────────────────┐
┌─────────┐1. pkts arrived ┌──┴──┐          ┌───┴────┐    send sw irq.  │ ksoftirqd/0(cpu0) │
│ network ├--------------->│ nic │          │ driver ├------(5)-------->└───────────────────┘
└─────────┘                └──┬──┘          └────────┘                  ┌───────────────────┐
                              |       ┌────────┐                        │ ksoftirqd/1(cpu1) │
                              └-(2)-->│ memory │ (ringbuffer)           └───────────────────┘
              2. nic DMA frame to mem └────────┘
```

2) software irq handling by kernel threads: ksoftirqd
```txt
                                            2. judge softirq_pending sign, execute _do_softirq
           ┌───────────────────┐     ┌───────────────┐   ┌──────────────┐   ┌───────────────┐
           │ ksoftirqd/0(cpu0) ├---->│ run_ksoftirqd ├-->│ _do_softirqd ├-->│ net_rx_action │
           └───────────────────┘     └───────────────┘   └──────────────┘   └────────┬──────┘
1. startup ┌───────────────────┐                                                     |
---------->│ ksoftirqd/1(cpu1) ├--┐                           softnet_data_poll_list | 
           └───────────────+───┘  |kernel thread loop           ┌─┬─┬─┬─┬───┬─┐      | 3. invoke poll funcs  
                           └------┘                             └─┴─┴─┴─┴───┴─┘      |    registed by driver
                                                                          ┌──────────+───────────┐
                           5. igb_clean_rx_irq send pkts to proto stack   │        driver        │
                          ┌────────────────┐     ┌──────────────────┐     │   ┌─────────────┐    │    ┌─────────┐
                          │ run_ksoftirqd()│<----┤ napi_gro_reveive │<----┤   │  igb_poll() │    ├--->│ packets │
                          └───────┬────────┘     └──────────────────┘     │   └──────┬──────┘    │    └─────────┘
                            net/core/dev.c          net/core/dev.c        │          |           │ 4. get pkts
                                  |                                       │ ┌────────+─────────┐ │ from ringbuf
                          ┌───────+───────────┐    ┌─────┐                │ │ igb_clean_rx_irq │ │
                          │ netif_reveive_skb ├--->│ skb │                │ └──────────────────┘ │
                          └───────────────────┘    └─────┘                └──────────────────────┘
                            net/core/dev.c                           drivers/net/ethernet/intel/igb/igb_main.c
```

3) skb handled by network protocol stack
```txt
                                                        ┌─────────┐
                                                 ┌----->│ arp_rcv │
                                                 |      └─────────┘
┌───────────────────┐    ┌───────────────────────┴──┐
│ netif_receive_skb ├--->│ __netif_receive_skb_core │                   ┌────────────┐       skb accept que
└───────────────────┘    └───────────────────────┬──┘               ┌-->│ tcp_v4_rcv ├---┐   of user process
                               /net/core/dev.c   |      ┌────────┐  |   └────────────┘   |   ┌─┬─┬─┬─┬───┬─┐
                                                 └----->│ ip_rcv ├--┤                    ├-->| | | | |   | ├-->wakeup user
                                                        └────────┘  |   ┌─────────┐      |   └─┴─┴─┴─┴───┴─┘   process
                                                                    └-->│ udp_rcv ├------┘
                                                                        └─────────┘
```

4) packets handling by ip layer

<hr>

### # questions as conclusion
1) what is ringbuffer? why ringbuffer is dropping packets?  
2) what is hardware interrupt? what is software interrupt?  
3) what does kernel threads ksoftirqd do?  
4) why enable multi-queue on nic can levelup network speed?  
5) how does tcpdump work? which layer does it work on?  
6) how does iptable/netfilter work? which layer does them work on?  
7) can tcpdump capture packets blocked by iptable?  
the procedure of accepting network packets:
```txt
        (hardware)                                   (kernel mode)
┌──────────────────────────┐  ┌──────────────────────────────────────────────────────────────┐
│┌─────┐   ┌──────────────┐│  │┌──────────────┐  ┌────────┐  ┌─────────┐  ┌─────────────────┐│
││ nic ├-->│ hardware irq ├-->││ software irq ├->│ driver ├->│ network ├->│ protocol stack  ││
│└─────┘   └──────────────┘│  │└──────────────┘  └────────┘  │ devices │  │┌───────────────┐││     (user mode)
└──────────────────────────┘  │                              └─────────┘  ││transport layer│││    ┌────────────┐
                              │                                           │└───────────────┘├---->│user process│
                              │                                           │┌──────────────┐ ││    └────────────┘
                              │                                           ││netowrk layer │ ││
                              │                                           │└──────────────┘ ││
                              │                                           └─────────────────┘│
                              └──────────────────────────────────────────────────────────────┘
--------------------------->  -------------------------------> tcpdump ---> netfilter ----------->
```
the procedure of sending network packets:
```txt
                                         (kernel mode)
                  ┌──────────────────────────────────────────────────────────┐
                  │┌─────────────────┐                                       │           (hardware)          
                  ││ protocol stack  │                                       │   ┌──────────────────────────┐
 (user mode)      ││┌───────────────┐│  ┌─────────┐   ┌────────┐   ┌───────┐ │   │┌─────┐   ┌──────────────┐│
┌────────────┐    │││transport layer││  │neighbor ├-->│network ├-->│drivers├---->││ nic ├-->│ hardware irq ││
│user process├--->││└───────────────┘├->│subsystem│   │devices │   └───────┘ │   │└─────┘   └──────────────┘│
└────────────┘    ││┌─────────────┐  │  └─────────┘   └────────┘             │   └──────────────────────────┘
                  │││netowrk layer│  │                                       │
                  ││└─────────────┘  │                                       │
                  │└─────────────────┘                                       │
                  └──────────────────────────────────────────────────────────┘
------------------> netfilter ------------------------> tcpdump --------------------------->
```
8) how to check the cpu load when accepting packets?  
9) why using dpdk? what does dpdk used for?  

