---
layout: post
title: "tcp-ip: TCP (连接和终止)"
subtitle: '[tcp-ip: illustrated vol1] - chapter18' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-09-25 23:30
lang: ch 
catalog: true 
categories: tcp-ip 
tags:
  - Time 2021
  - tcp-ip illustrated vol1 
---
### 一：引言
TCP是面向连接的协议，任何一方发送数据之前，都必须先在双方之间建立连接。本文主要介绍TCP连接是如何建立，以及通信结束后是如何终止的。

### 二：连接的建立&终止
##### 1 使用telnet程序建立 & 终止TCP连接
<center><img src="/img/in-post/tcp-ip_img/tcp_18_1.pdf" width="100%"></center>

##### 2 使用tcpdump程序对于上述TCP连接 & 终止过程分组交换情况进行分析(使用tcpdump -S选项)
<center><img src="/img/in-post/tcp-ip_img/tcp_18_2.pdf" width="100%"></center>

##### 3 tcpdump程序打印的标识符缩写 & 全称 & 描述信息对应关系图表
<center><img src="/img/in-post/tcp-ip_img/tcp_18_3.pdf" width="40%"></center>

##### 4 TCP建立连接 & 终止连接及其相应的时间序列图
<center><img src="/img/in-post/tcp-ip_img/tcp_18_4.pdf" width="100%"></center>

TCP建立连接部分简析：
- 发送端首先发送SYN信号以执行主动打开(active open)，接收端接受该SYN信号并执行被动打开(passive open)。TCP支持两端都主动打开，在本文后续部分进行介绍。
- TCP连接某端口为了建立连接而发送SYN信号时，会选择一个初始序号(initial sequence number, ISN)。ISN随时间而变化，因此不同的连接有不同的ISN。RFC793指出ISN相当于一个32bit计数器，每隔4ms自动加1。
- segment#3到segment#4之间的4.1s的时间是从建立了TCP连接到键入'quit'准备中断该连接的时间。

TCP终止连接部分简析：
- 建立一个TCP连接需要3次握手，而终止TCP连接需要4次握手。终止连接需要4次握手的原因是，TCP为全双工通信，即数据在两个方向上可以同时通信，因此每个方向上必须进行独立关闭。
- 对于全关闭：首先发送FIN信号的一端执行主动关闭，另一端(接受此FIN信号)执行被动关闭。TCP连接的双方可以都执行主动关闭，在本文后续部分进行介绍。
- TCP连接发送端telnet应用程序关闭时，发送FIN信号，用于终止TCP连接。当接收端收到该FIN信号，就返回ACK信号，此时接收端TCP服务程序将会向telnet应用程序发送一个文件结束符(EOF)。接收端应用程序telnet关闭，相应TCP端返回一个FIN信号，发送端返回ACK信号，确认接收端关闭。


##### 5 使用普通的tcpdump程序打印的输出(不使用-S选项)
默认情况下，tcpdump只在收发SYN信号时显示完整的序号，后面的报文收发只显示与初始ISN序号的相对偏移值。

### 三：TCP连接建立超时
现实中有多种情况导致TCP无法正常建立连接，一种典型情况是：TCP服务端主机没有处于正常状态。

##### 1 模拟TCP无法建立连接场景并观测重发SYN信号时间间隔
现模拟这种情况：断开服务端主机的电缆线，并通过telnet程序向他发起连接，打印tcpdump输出如下。
<center><img src="/img/in-post/tcp-ip_img/tcp_18_5.pdf" width="100%"></center>
上述tcpdump程序打印的输出中出现符号"[tos 0x10]"，这是IP数据报中服务类型字段(TOS)。BSD/386系统中的telnet进程将该字段设置为"最小时延"。

##### 2 对于telnet应用程序进行计时以观测TCP连接超时重发最长时间限制
大多数伯克利软件系统将建立一个TCP连接的最长时间限制设置为75s，上述telnet超时重传测量的最大时间间隔为76s。使用date记录telnet命令从建立TCP连接到放弃建立连接的时间。
<center><img src="/img/in-post/tcp-ip_img/tcp_18_6.pdf" width="100%"></center>

##### 3 第一次超时时间为5.8s而不是预设的6s的原因分析
下图展示了应用程序的tick和TCP连接两次定时重连的时间之间的关系。第一次定时6s进行TCP重连，第二次定时24s进行TCP重连。<br>
第一次超时重传为5.8s的原因是：第一次定时时刻可能在应用程序两个tick之间的某时刻，但是TCP定时程序每经过一个tick就将定时计数器减一。因此由于起始定时时刻不同而产生0～0.5s的误差。
<center><img src="/img/in-post/tcp-ip_img/tcp_18_7.pdf" width="80%"></center>

### 四：最大报文长度
**[1] 引言** 最大报文长度(maximum segment size, MSS)表示TCP传往另一端的最大数据部分长度。当TCP连接建立时，通信双方需告知对方各自的MSS。目前涉及的TCP连接设置的MSS都是1024，即相应IP数据报长度为40byte(20byteIP首部+20byteTCP首部)+1024byte。

**[2] 最大报文长度MSS不是一个可以协商的选项(连接建立时必须确定)**，MSS数值必须在发送SYN信号报文段中，即建立TCP连接时必须确定通信双方都接受的MSS数值。如果其中一方不接受另一方发送的MSS数值，那么MSS取默认536byte(20byte IP首部 + 20byteTCP首部 = 576byte IP数据报总长度)。

**[3] 一般来说，MSS越大越好**，这是因为较大的MSS有较高的网络利用率，更少的几率产生网络数据碎片。本地应用进程发起一个TCP连接，则它能够设置的最大的MSS数值为外出接口链路上的MTU长度减去固定的IP首部长度以及TCP首部长度：对于以太网外出接口而言，MSS数值可以最大设为1500byte-20byte-20byte=1460byte；对于IEEE-802.3封装而言，MSS最大设为1492byte-40byte=1452byte。

**[4] 目的IP地址为非本地主机(TCP连接不同子网中的主机)，MSS默认数值一般为536byte**。TCP连接两端是否为本地连接可以通过子网号进行判断：(i) 如果目的IP地址的网络号和子网号和当前主机相同，则TCP为本地连接；(ii) 如果目的IP地址的网络号和当前主机不同，那么为非本地连接；(iii) 如果目的IP地址和当前主机网络号相同，但子网号不同，则可能是本地连接，也可能是非本地连接。针对第三种情况，大多数TCP实现都提供了一种配置选项，该选项允许系统管理员说明每个子网属于local还是non-local。

##### 1 结合案例分析最大报文长度对于TCP通信的影响
从sun主机向slip主机发起一项TCP连接，sun主机和slip主机的网络关系拓扑图如下所示。
<center><img src="/img/in-post/tcp-ip_img/tcp_18_8.pdf" width="50%"></center>

使用tcpdump程序对于上述sun主机和slip主机建立TCP连接中数据分组交换情况进行分析，打印交换的分组信息如下所示。
<center><img src="/img/in-post/tcp-ip_img/tcp_18_9.pdf" width="100%"></center>

sun主机不会发送超过256byte的segment，因为它收到的最大报文长度MSS=256byte。<br>
需要注意的是，虽然sun主机在建立TCP连接发送SYN信号时，通知slip主机最大报文长度MSS为1460byte，但是slip主机也不会发送超过256byte的segment，这是因为slip主机知道自身外出接口的MTU为256。

##### 2 最大报文长度MSS & 避免报文分段
当TCP连接两端通过多个网络链路进行连接，只有当中间传输网络的MTU能够兼容TCP连接两端的"min(MTU,MSS)"数值，才能保证数据传输过程中不会被分段。如果中间网络的MTU小于TCP连接两端的"min(MTU,MSS)"数值，那么数据传输过程中会产生分段。<br>
使用路径MTU发现机制(path MTU discovery)是避免数据分段的唯一方式。

### 五：TCP的半关闭 
TCP的半关闭是指：即使TCP连接中的一端已经结束了数据发送，但它仍然能够保持接受来自TCP连接另一端发送的数据的能力。只有很少一部分应用程序需要使用TCP半关闭机制。<br>
用生动形象的语言去描述TCP半关闭机制：我已经完成了我的数据传送，因此发送FIN信号给TCP连接另一端，但是在对方完成数据发送并返回FIN信号之前，我仍然想要接受对方发来的数据。

##### 1 半关闭TCP连接通信时序图
<center><img src="/img/in-post/tcp-ip_img/tcp_18_10.pdf" width="100%"></center>
- segment#4调用的不是close，而是shutdown，因此TCP连接执行半关闭。shutdown只关闭当前端，而close将关闭整个TCP连接。
- 当segment#9中的ACK确认信号被服务端接收后，整个TCP连接彻底关闭。
- 上述TCP半关闭只显示了一个报文端数据通信情况，实际应用中在TCP连接半关闭情况下，可能需要多次数据传输&确认。

##### 2 半关闭的应用案例
<center><img src="/img/in-post/tcp-ip_img/tcp_18_11.pdf" width="70%"></center>
- unix系统中的rsh命令是应用TCP连接半关闭通信的典型命令。
- rsh首先建立主机sun和主机bsdi之间的连接，然后将datafile复制给TCP连接并传输copy到TCP连接另一端。
- 在bsdi主机上，rshd服务进程将对接收到的datafile执行sort命令，并将sort命令输出结果通过TCP连接返回客户端sun主机。
- 这个过程中，需要注意TCP是全双工的，即TCP连接的两端可以同时进行数据收发。
- 主机bsdi上的sort程序只有读取到datafile文件全部内容才会产生输出。在datafile文件传输完成后，sun主机和bsdi主机之间的TCP连接变成半关闭状态。sun主机不再向TCP连接另一端发送数据，但sun主机上的rsh客户端继续接受来自TCP连接另一端的返回的数据。

### 六：TCP状态变迁图
<center><img src="/img/in-post/tcp-ip_img/tcp_18_12.pdf" width="100%"></center>

##### 1 状态变迁图注意要点
- 状态转移图中所有状态转移都针对同一端，即只绘制client端或者server端自身的状态转移，而不绘制client和server之间的状态转移。
- 图中实线箭头表示普通客户端状态的变更(normal client transitions)，虚线箭头表示普通服务端状态的变更(normal server transitions)。
- ESTABLISH状态为：TCP连接双方能够进行双向数据传递的状态。
- 图中两条"进入"ESTABLISHED状态的实线、虚线分别表示建立一个TCP连接所需要的客户端、服务端发送、接受的信号分组。
- 图中两条"离开"ESTABLISHED状态的实线、虚线分别表示关闭一个TCP连接所需要的客户端、服务端发送、接受的信号分组。
- 图中两个虚线框分别表示主动关闭(active close)、被动关闭(passive close)。
- 图中CLOSED并不是一个TCP程序运行中的真实状态，它仅表示这个状态图的假想起点、终点。

##### 2 TCP建立&终止连接的状态转化时序图
<center><img src="/img/in-post/tcp-ip_img/tcp_18_13.pdf" width="100%"></center>
- TCP建立连接时涉及的状态转移：SYN_SENT =\> ESTABLISHED =\> FIN_WAIT_1 =\> FIN_WAIT_2 =\> TIME_WAIT。
- TCP终止连接时涉及的状态转移：SYN_RCVD =\> ESTABLISHED =\> CLOSE_WAIT =\> LAST_ACK =\> CLOSED。
- 上述TCP建立连接&终止连接的状态转化时序可以在TCP状态变迁图中对应的线路部分进行验证。

##### 3 2MSL等待状态
状态转移图中TIME_WAIT状态也称为2MSL等待状态。每个TCP具体实现都必须选择报文最大生存时间(maximum segment lifetime, MSL)，MSL表示每个报文segment在被丢弃前在网络内存在的最长时间。TCP报文借助IP数据报进行传输，IP数据报具有限制其生存时间的TTL字段，因此很容易推断出MSL是一个有限的数值。

RFC-793指出MSL为2分钟。但通常实际应用中常用数值为30s、1min、2min。

当TCP发送端执行主动关闭，并发回最后一个ACK信号，该TCP连接必须在TIME_WAIT状态停留至少为2MSL的时间，用于防止：最后发送端发送的ACK丢失，另一端等待超时后重发最后一个FIN信号，2MSL的等待时间允许发送端针对超时重发的FIN信号进行应答。

**[1] 2MSL等待时间的注意事项**
- 在TCP连接等待的2MSL的时间内，该连接占用的socket(client的IP地址、端口号以及server的IP地址、端口号)不能被使用(该连接不能被使用)。
- 大多数伯克利实现的TCP加强了上述限制：在TCP连接等待的2MSL时间内，该socket(插口)占用的本地端口不能再被使用(不同被其他应用进程占用)。
- TCP连接在2MSL等待时，此时该socket到达的任何报文都会被丢弃。

**[2] TCP客户端执行主动关闭和TCP服务端执行主动关闭的区别**
- 在分析TCP连接两端分别执行主动关闭之前，需要明确：TCP连接客户端的端口随机可用的本地端口即可；但TCP连接服务端需要使用公知端口进行通信。
- TCP客户端执行主动关闭，则客户端会进入TIME_WAIT状态(客户端占用的socket将等待2MSL的时间才能重用)，此时服务端属于被动关闭，服务端并不会进入TIME_WAIT状态。因此在2MSL等待时间内，如果客户端、服务端尝试再次建立连接，客户端只能使用不同的本地端口，而服务端不要更换端口。
- TCP服务端执行主动关闭，则服务端会进入TIME_WAIT状态(服务端占用的socket等待2MSL的时间才能重用)。由于服务端使用公知端口，因此在2MSL的时间内，该服务端将不能再次建立任何TCP连接，通常需要等待2MSL=1～4min的时间。

**[3] 使用sock程序，观察TCP服务端公知服务端口被占用而无法建立连接的现象** <br>
**(a)** 在sun主机上启动server进程，占用本地端口6666；在bsdi主机上启动client进程与server进程建立连接，占用本地端口1081。
<center><img src="/img/in-post/tcp-ip_img/tcp_18_14.pdf" width="100%"></center>
**(b)** 在上面的连接中止后的2MSL等待时间内，在sun主机相同端口(6666)再次启动同样的服务端进程。
<center><img src="/img/in-post/tcp-ip_img/tcp_18_15.pdf" width="100%"></center>
**(c)** 使用netstat命令检查sun主机6666端口状态信息。
<center><img src="/img/in-post/tcp-ip_img/tcp_18_16.pdf" width="100%"></center>

**[4] 使用sock程序，观察TCP客户端使用的本地端口被占用而无法建立连接的现象** <br>
**(a)** 在sun主机上的1162端口启动client端进程，与bsdi主机上的echo回显服务进程相连接。
<center><img src="/img/in-post/tcp-ip_img/tcp_18_17.pdf" width="100%"></center>
**(b)** 在上面连接中止后的2MSL等待时间内，在sun主机(client)相同端口1162再次建立TCP连接到bsdi-echo。
<center><img src="/img/in-post/tcp-ip_img/tcp_18_18.pdf" width="100%"></center>

**[5] 使用sock程序，观察TCP服务端公知端口被占用而无法建立连接的现象；使用-A选项设置SO_REUSEADDR尝试重建❮同一❯连接并观察结果** <br>
**(a)** 在sun主机上启动server进程，占用本地端口6666；在bsdi主机上启动client进程与server进程建立连接，占用本地端口1081。
<center><img src="/img/in-post/tcp-ip_img/tcp_18_19.pdf" width="100%"></center>

**(b)** 在上述连接中止后的2MSL等待时间内，在sun主机上尝试再次绑定本地端口6666和bsdi主机建立❮同一❯连接。
<center><img src="/img/in-post/tcp-ip_img/tcp_18_20.pdf" width="100%"></center>

**(c)** 设置SO_REUSEADDR选项允许server端重用处于2MSL等待时间的端口6666，在sun主机上尝试再次和bsdi主机建立❮同一❯连接。
<center><img src="/img/in-post/tcp-ip_img/tcp_18_21.pdf" width="100%"></center>

**(d)** 使用sock程序，使用-A选项，在其他主机上(bsdi)使用TCP和server端处于2MSL等待状态的端口6666建立❮另一条❯连接 <br>
在bsdi主机上使用sock程序尝试与sun主机上处于2MSL等待状态的端口建立TCP连接：可以连接成功。这是不符合TCP连接规范的，但是大多数伯克利实现的TCP都支持该功能：即允许一个新的连接请求到达仍然处于TIME_WAIT状态的连接，❮只要新的连接建立的序号大于之前建立的连接的最后使用的序号即可❯。本案例中的程序将建立新连接的序号设置为之前连接的"最后序号+128000"。<br>
<center><img src="/img/in-post/tcp-ip_img/tcp_18_22.pdf" width="100%"></center>

RFC-1185指出了上述技术可能存在的缺陷。该技术能够让客户端和服务端程序能够连续的使用相同的端口号(连续建立连接无需在client、server两端每次都更换端口号)，需要注意的是，这项技术需要每次通信都在服务端执行"主动关闭"，即服务端必须进入TIME_WAIT状态。

##### 3 平静时间概念(quiet time concept)
TCP重新启动后初始的MSL的时间内，不能够建立任何连接，这段时间称为TCP的平静时间。RFC-793给出了平静时间的相关规范。

TCP平静时间概念是为了避免如下场景中可能出现的misinterpret错误：<br>
当处于2MSL等待状态的主机出现故障，它会在MSL的时间内重新启动。如果它在重启动完成后，立即利用当前处于2MSL状态的socket建立一个新的连接，那么对于主机故障前发出的由网络问题而延迟到达的报文segment将会被misinterpret成重新连接的报文segment。通过选择重启后ISN序号并不能完全避免这个错误。

为了避免这种连接前后混淆的错误，主机发生故障重启后，需要等待MSL的时间才能建立新的连接，以确保延迟到达的网络数据segment能够到达，或者数据报丢失而返回超时信息，从而能够确保区分数据报的归属。

##### 4 FIN_WIN_2状态的无限等待 & 及其避免方式
在进入FIN_WIN_2状态时，客户端已经发送了FIN信号并收到确认该FIN信号的ACK应答信号，此时除非执行TCP半关闭，否则客户端将保持FIN_WIN_2状态以等待另一端应用层收到文件结束EOF，并向客户端返回FIN信号来关闭另一端连接。<br>
上述过程中，应用层对于文件EOF信号处理中由于某种原因未能返回FIN信号，则导致客户端一直保持FIN_WIN_2状态，而另一端将一直保持CLOSE_WAIT状态。

为了避免上述问题，伯克利实现的TCP规定：如果执行主动关闭应用层决定采用全关闭而不是半关闭，那么就要设置一个定时器，如果该TCP连接空闲**10min+75s**，那么TCP将重置为CLOSED状态。

### 七：复位报文段(reset segments)
首先介绍"referenced connection"表示从特定源IP:源端口号到目的IP:目的端口号的一条连接，RFC793称之为"socket"。
TCP首部中RST字段表示复位(reset)。当"referenced connection"连接中传输的报文segment不正确时，TCP将发送复位信号。<br>
下面将分几种情况分别对于产生复位信号的场景进行分析。

##### 1 向一个不存在的端口发送连接请求
当建立TCP连接请求的报文到达目的端口时，没有任何进程对该端口进行监听，此时会产生复位报文。同之前提到的UDP机制类似：如果UDP数据报到达目的端，但该端口没有被任何进程监听，那么将会返回ICMP不可达报文信息；与之对应，TCP将会返回复位报文。

在bsdi主机通过telnet程序和svr4主机上一个未被任何进程占用的端口建立TCP连接，以产生TCP复位报文。
<center><img src="/img/in-post/tcp-ip_img/tcp_18_23.pdf" width="100%"></center>

使用tcpdump程序对于上述过程的报文分组交换情况进行抓包，并打印输出如下。
<center><img src="/img/in-post/tcp-ip_img/tcp_18_24.pdf" width="100%"></center>

##### 2 由于异常而终止TCP连接
正常方式终止一个TCP连接是通过：连接中的某一端等待全部数据发送完成后，再发送FIN信号执行主动关闭，这可以被称为"有序释放(orderly release)"，这种情况下没有任何数据丢失。但是，有时TCP通过发送复位(reset)报文而不是FIN信号来终止一个连接，此时称为"异常释放(abortive release)"。

采取"异常释放"方式来终止连接为应用程序提供了两个特性：**(i)** 应用程序借助"abortive release"机制实现：丢弃所有待发送报文端、立即发送复位报文段。**(ii)** RST复位报文接收方需要具有区分另一端执行异常关闭or正常关闭的功能；换一种角度，应用程序必须提供执行异常关闭&正常关闭两种手段。

使用sock -L程序，并设置"linger on close"选项(SO_LINGER)以实现在关闭连接时执行异常关闭，而不是返回FIN信号进行正常关闭。
<center><img src="/img/in-post/tcp-ip_img/tcp_18_25.pdf" width="100%"></center>

使用tcpdump程序对于上述sock程序执行异常关闭时的数据分组交换情况进行分析，并打印相关信息如下所示。第6行，主机在关闭时发送一个RST复位报文，而不是发送FIN正常关闭信号。<br>
RST报文中包含一个序号和一个确认序号。另外，RST报文不会对接收方产生任何影响，接受RST报文一端无需进行确认，只需终止连接，并通知应用层连接已经复位。
<center><img src="/img/in-post/tcp-ip_img/tcp_18_26.pdf" width="100%"></center>

在服务端主机svr4上，运行sock程序，客户端主动执行异常关闭后，打印相关错误信息以验证连接发生异常终止。
<center><img src="/img/in-post/tcp-ip_img/tcp_18_27.pdf" width="100%"></center>

##### 3 半打开连接的检测(detecting of half-open connections)
如果TCP连接中的一方已经关闭or异常关闭，但是连接另一端却不知道，则称这种状态下的TCP连接为"半打开"的。

**[1] 发生TCP"半打开"连接的2个原因**
原因#1：TCP连接中任意一端的主机发生异常均可能导致half-open的TCP连接。只要不在half-open的TCP连接上传输数据，那么处于连接状态的一端就不会检测到另一端已经出现异常。

原因#2：当客户端主机突然断电而不是正常结束客户端程序后再关机。例如：在客户端和服务端主机之间建立TCP连接，但是用户将客户端主机关机前，没有退出应用程序客户端的TCP连接，此时只要客户端不再使用该连接向服务端发送数据，服务端永远不会知道客户端程序已经消失。用户第二天建立新的telnet程序。长此以往，服务端会遗留很多半打开的TCP连接。

chapter23中将介绍TCP的keepalive选项能够使得TCP连接的一端能够检测出另一端是否还在线。

**[2] 模拟服务端主机出现异常，建立"半打开"TCP连接** <br>
在bsdi主机上运行telnet程序，和svr4主机上的discard标准服务建立TCP连接。然后断开服务器主机和以太网之间的电缆(防止TCP程序向外发送FIN导致连接正常关闭)，并重启服务器主机，以此来模拟服务器主机出现异常。

在bsdi主机(client)上和svr4主机(server)进行通信，然后按照上述方式模拟服务端主机出现异常而单方面丢失TCP连接(进入"半打开"状态)，然后重启svr4主机，再次在bsdi主机上使用之前的TCP连接向svr4发送数据，观测svr4服务端主机返回信息，并打印输出。
<center><img src="/img/in-post/tcp-ip_img/tcp_18_28.pdf" width="100%"></center>

使用tcpdump程序对于上述从"初始建立bsdi主机到svr4主机的连接"，到"模拟svr4服务端主机发生异常"，再到"svr4服务端主机重新建立arp高速缓存并返回TCP复位信息"的整个过程进行抓包分析，打印相应数据分组信息如下。
<center><img src="/img/in-post/tcp-ip_img/tcp_18_29.pdf" width="100%"></center>

### 八：同时打开状态(simultaneous open)
同时打开状态的定义：两个应用程序彼此同时执行主动打开(active open)是可行的，尽管这种情况发生的概率极小。执行同时打开的应用程序中每一端都需要发送SYN信号，并将SYN信号传递给对方。每端使用对方熟知的端口作为连接使用的本地端口。

TCP进行了特殊设计以支持同时打开，对于simultaneous open只建立一条连接，而不是建立两个连接；不同于OSI数据传输层：OSI针对同时打开状态需要建立两个连接。

TCP同时打开连接的建立需要经过"4"次握手而不是"3"次握手。另外，TCP同时打开的两端不能被分成客户端或者服务端，它们既是客户端又是服务端。

##### 1 同时打开TCP建立连接时序图
<center><img src="/img/in-post/tcp-ip_img/tcp_18_30.pdf" width="70%"></center>

##### 2 同时打开的案例&分析
现在考虑如何模拟TCP连接两端同时打开：两端必须互相发送SYN信号，并且要求：在收到另一端ACK信号之前两端的SYN信号发送完成，因此只需要使SYN信号传输时间很长就能够实现(两端之间数据传递有较长的往返时间)。<br>
因此，考虑从bsdi主机上连接到vangogh.cs.berkelay.edu上，两端通信线缆需要通过传输速率很慢的拨号链路SLIP，因此保证了双方同步收到SYN信号时间足够长(几百ms)，从而在一端输入sock命令执行主动打开后，有充足的时间在另一端执行sock程序也进行主动打开。

(i) 在bsdi主机(client & server)上运行sock程序执行主动打开。
<center><img src="/img/in-post/tcp-ip_img/tcp_18_31.pdf" width="100%"></center>

(ii) 在vangogh主机(client & server)上运行sock程序执行主动打开。
<center><img src="/img/in-post/tcp-ip_img/tcp_18_32.pdf" width="100%"></center>

(iii) 使用tcpdump程序对于上述同时打开的过程中交换的数据分组过程进行抓包，并打印相关信息如下。<br>
其中1～4行打印的数据分组信息为建立"同时打开"TCP连接所需的必要数据segment交换；第5～8行打印的数据分组信息为使用"同时打开"的TCP连接进行数据分组交换涉及的数据segment；第9～12行打印的数据分组信息为关闭该TCP连接所需必要的数据segment交换。
<center><img src="/img/in-post/tcp-ip_img/tcp_18_33.pdf" width="100%"></center>

### 九：同时关闭状态(simultaneous close)
之前的所有案例都是TCP连接的一端执行主动关闭，而另一方执行被动关闭。本部分内容主要想要说明：TCP连接两端都执行主动关闭也是可能的，并且TCP规范也为这种"主动关闭"方式提供支持。

位于TCP连接的两端近似同时发出关闭指令，因此连接的两端均从ESTABLISHED状态变为FIN_WAIT_1状态。因此连接两端将向对方发送发送FIN信号，在收到收到来自对方的FIN信号后均变为CLOSING状态并发送最后的ACK信号。在TCP连接双反收到最后的ACK信号后，状态变成TIME_WAIT。

##### 1 同时关闭TCP建立连接时序图
<center><img src="/img/in-post/tcp-ip_img/tcp_18_34.pdf" width="70%"></center>

### 十：TCP选项
最初的TCP选项只包含：选项表结束(the end of option list)、无操作(no operation)、最大报文长度(maximum segment size)。新的RFC(尤其是RFC-1323)定义了新的RFC选项。

##### 1 RFC-793 & RFC-1323定义的选项格式
<center><img src="/img/in-post/tcp-ip_img/tcp_18_35.pdf" width="80%"></center>
- 每个选项格式以1字节kind字段开始，用于说明选项的类型。
- 选项表结束、无操作选项都只占用1byte，除选项类型没有其他字节。
- 最大报文长度、窗口扩大因子、时间戳都包含一个len字段，用于表示该选项占用的字节数。
- 设置无操作选项(NOP)的原因是，方便TCP发送端将选项部分报文padding成4byte的倍数。
- 其他选项kind=4、5、6、7称为"可选的ACK选项"和回显选项。回显选项已经被时间戳选项代替，截止TCP-vol1出书时可选的ACK选项(selective ACK options)尚未定论，相关信息没有包含在RFC-1323中。

##### 2 BSD(4.4)系统建立TCP连接时发送的SYN信号TCP首部选项
<center><img src="/img/in-post/tcp-ip_img/tcp_18_36.pdf" width="80%"></center>

### 十一：TCP服务器的设计(to be added)
##### 1 TCP服务器的端口号
##### 2 受限制的服务器本地IP
##### 3 受限制的服务器远端IP
##### 4 处理TCP连接请求队列

### 十二：小结(to be added)

## Reference
> \<tcp-ip: illustrated vol1\> chapter18 <br>

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用\`\`将关键字包含在内 <br>
> 8 `<center><img src="/img/in-post/economics_4/xxx.png" width="60%"></center>` <br>
> 9 使用html设置图片文字环绕方式: <br>
    `<div>` <br>
        `<img src="/img_path" align="left" width="40%" hspace="" vspace=""/>` <br>
        `<p>paragraph1 around the picture</p>` <br>
        `<p>paragraph2 around the picture</p>` <br>
        `<p>paragraph3 around the picture</p>` <br>
    `</div>` <br>
> 10 `<font style="color:red; font-weight:bold">加粗蓝色</font>`用来设置字体颜色 <br>
> 问题脚注: ### ???problem😫problem???
