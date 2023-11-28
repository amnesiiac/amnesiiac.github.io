---
layout: post
title: "tcp-ip: TCP (未来发展和性能优化)"
subtitle: '[tcp-ip: illustrated vol1] - chapter24' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: true 
date: 2021-10-24 15:35
lang: ch 
catalog: true 
categories: tcp-ip 
tags:
  - Time 2021
  - tcp-ip illustrated vol1 
---
### 一：引言
在1980s到1990s初期，TCP/IP的主要数据链路是以太网。虽然TCP在相比以太网传输速率更高的环境中(e.g. T3 phone lines, FDDI, gigabit networks)也能够正常运行，但是在这些环境下运行的TCP可能会暴露出一些限制。

本文中讨论的主要内容是关于TCP的一些proposed modifications，主要为了让TCP能够在高速率传输环境中获得最大的吞吐量(maximum throughput)。
- 首先讨论**路径MTU发现机制**，本文主要关注该机制和TCP的协同工作方式，以允许TCP的nonlocal连接使用≥536byte的MTU，从而增加网络吞吐量。
- 接着介绍**"long-fat pipe"，即具有很大的"带宽×网络时延"的网**络，并介绍TCP在这些网络上所具有的局限性。和"long-fat pipe"相关的两个TCP选项在本文中进行介绍：**"窗口扩大选项**(用于增加TCP最大窗口使其≥65535byte)"、**时间戳选项**(用于对于TCP的往返时间进行更加精确的测量，以及在高传输速率下可能会发生的"序号回绕(wrapped sequence numbers)"提供保护措施)。RFC-1323对于这两个选项进行了定义。
- 还将介绍**proposed T/TCP(modifications to TCP for transactions)**。transaction mode of communication的特性是：由服务端回复客户端的请求，这是一种最普通的通信范式(paradigm)；T/TCP的目标是尽量减少两端交换的分组数，避免3次握手建立连接和4次握手关闭连接，使得客户端能够在"1个RTT时间+处理客户端请求的时间"收到服务端的响应。
- **路径MTU发现选项、窗口扩大选项、时间戳选项、T/TCP**能够和现存的TCP版本实现向后兼容，较新版本系统也能够和老版本的系统进行interoperate。除了路径MTU发现选项需要在ICMP报文中增加一个额外的字段，4个选项只需要在设置该选项一端的TCP中实现相应机制即可。
- 最后，通过展示一些关于TCP性能图表作为本文结束。


### 二：路径MTU发现
**(a) 回顾路径MTU发现相关概念** <br> 当前两个主机之间的路径上的任何网络的最小MTU。路径MTU发现需要在IP首部设置"不要分片(DF)"标志位，以用来发现当前路径上的任何路由器是否需要对正在传输的IP数据报进行分片。<br>
chapter11中介绍：当一个带转发的IP数据分组设置了DF标志位，其长度却超过了路径MTU数值，则中间路由器会返回ICMP不可达差错。<br>
chapter12中提到：某版本的traceroute程序使用上述返回ICMP差错报文的机制来获取目的路径MTU。

本部分重点介绍路径MTU发现机制是如何按照RFC-1191中的规定在TCP中实现的。

**(b) TCP的路径MTU发现的数值设定&机制相关的基本规则** <br>
- **(1)** 在建立TCP连接时，发送端采用输出接口or对端声明的最大数据分组尺寸中的最小MTU作为初始发送的数据分组大小。
- 路径发现MTU机制不允许TCP收发超过"对端声明的的最大报文长度MSS"的数据分组。
- 如果TCP连接对端没有指定一个MSS数值，则发送端使用默认MTU数值为536byte。
- 另外，部分TCP实现可以按照chapter21中提到的那样，为每个路由表项单独保存一份路径MTU信息。
- **(2)** 一旦选定了初始发送的数据分组大小，则该TCP连接上的所有发送的IP数据分组都将被设置DF(don't fragment)标志位。如果某个中间路由器需要对于设置了DF标志位的数据分组进行分片，那么它会丢弃该数据分组，并返回ICMP-"不能分片差错"。
- **(3)** 当发送端收到中间路由返回的ICMP-不能分片差错，TCP就减少数据分组大小并执行重传。**(3.1)** 如果产生的ICMP不能分片报文格式为"newer form"，待发送数据分组长度被设置成："下一跳的MTU数值" - "IP首部长度" - "TCP首部长度"。**(3.2)** 如果收到的ICMP不能分片报文格式为"older form"，待发送数据分组长度通过查表选择下一个最小的表项MTU数值(参考[chapter2-MTU常见链路数值表](http://localhost:4000/tcp-ip/2021/08/04/post-tcp-ip-link-layer/#最大传输单元mtu及路径mtu))。
- **(4)** 当发送端收到ICMP-不能分片差错导致数据分组重传时，发送端拥塞窗口不需要变化，但是需要执行慢启动重发被丢弃的数据分组。
- **(5)** 对于一个TCP连接，路由线路可以发生变化(路径MTU发现机制得到的MTU具有时效性)，因此记录每次主动降低路径MTU的时刻t，在t+10min尝试使用一个更大的MTU数值(该数值≤对端声明的MSS && 该数值≤数据外出端口的MTU)发送数据分组(RFC-1191推荐使用10min作为间隔，Solaris2.2使用30min作为间隔)。
- 对于"nonlocal"的目的端，默认的MSS一般为536byte，借助路由发现机制可以避免通过MTU≤576byte的中间链路时被分片；对于"local"目的端，同理可以避免分片。但是为了更好的充分利用MTU≥536byte的广域网，TCP的相应实现必须停止使用针对"non-local"目的端设置的默认536byteMTU数值，一个更好的MTU选择是默认采用输出端口的MTU数值。

##### 1 路径MTU发现案例分析
当TCP连接传输中的某个中间路由器的MTU比连接两端的结构MTU数值小时，通过传输数据分组观察solaris2.2系统中的路径MTU发现机制的具体工作原理。

**(a) 路径MTU发现案例涉及的主机网络拓扑结构**
<center><img src="/img/in-post/tcp-ip_img/tcp_24_1.pdf" width="80%"></center>

**(b) 从solaris主机到slip主机建立一条TCP连接，并在solaris系统上运行sock程序向slip标准丢弃服务程序执行512byte写操作**
<center><img src="/img/in-post/tcp-ip_img/tcp_24_2.pdf" width="100%"></center>

**(c) 在中间路由器sun的SLIP链路接口上运行tcpdump程序打印分组交换相关信息** 
<center><img src="/img/in-post/tcp-ip_img/tcp_24_3.pdf" width="100%"></center>

##### 2 大的数据分组or小的数据分组?
通常情况下，在分组大小不足以在传输中发生分片的情况下，按照较大的数据分组发送数据较好，这是因为较大的数据分组拥有：更小的数据分组首部负荷(网络)、选路决定负荷(路由器)、协议的处理和设备的中断(主机)等因素有关。<br>
但是，并不是所有场景下采用较大的数据分组都好于小的数据分组。

考虑如下案例：通过4个路由器传输8192byte数据，每个路由器和一个T1电话线路(1544000b/s)相连接：<br>
**(1) 将8192byte数据使用2个4096byte数据分组进行发送，传输分组时序流程图如下**
<center><img src="/img/in-post/tcp-ip_img/tcp_24_4.pdf" width="100%"></center>

**(2) 将8192byte数据使用16个512byte数据分组进行发送，传输分组时序流程图如下**
<center><img src="/img/in-post/tcp-ip_img/tcp_24_5.pdf" width="100%"></center>


### 三：long-fat pipe - 又长又肥的通信管道
回顾chapter20，可以将一个TCP连接的通道的容量(也称为管道的大小)计算为带宽时延的乘积：$capacity(b)=bandwidth(b/s)\times round-trip\, time(s)$。当这个乘积越来越大时，TCP的某些局限性就会暴露出来。

##### 1 几种网络信道的带宽时延乘积表
<center><img src="/img/in-post/tcp-ip_img/tcp_24_6.pdf" width="70%"></center>

具有较大的带宽时延乘积的网络被称为长肥网络(long-fat network, LFN)，运行在一个LFN上的TCP连接被称为长肥管道(long-fat pipe)。
回顾[chapter20通道容量计算](http://localhost:4000/tcp-ip/2021/10/03/post-tcp-ip-tcp-chapter-20/#3-通道容量的计算pipeline-capacity---带宽时延的乘积bandwidth-delay-product)易知，管道可以被水平拉长(往返时间RTT增加)，也可以被垂直拉高(使用较高带宽的信道)。

##### 2 使用长肥管道进行TCP通信可能会遇到的问题
- **(1)** TCP首部中窗口大小字段占用16bit，因此将窗口尺寸限制在65535byte范围内。但是从上面"网络信道带宽时延乘积表"最后一列可以看出，对于"long-fat pipe"而言，需要更大的窗口以进一步提升在长肥管道中通信的吞吐量。本文将介绍的窗口扩大选项可以解决这个问题。
- **(2)** 在长肥管道中的通信，如果发生数据分组丢失会使得吞吐量急剧减少。当只有一个数据分组丢失时，可以采用[chapter21快速重传、快速恢复算法](http://localhost:4000/tcp-ip/2021/10/07/post-tcp-ip-tcp-chapter-21/#七快速重传与快速恢复算法fast-retransmit--fast-recovery-algorithms)来避免信道"draining"。但是，当一个窗口内发生多个分组丢失时，即使采用快速重传、恢复算法也仍然会导致信道进入"枯竭"状态。此后需要执行慢启动经过多个RTT时间再将信道重新填满。
- **(3)** 许多TCP实现执行"one RTT measurement per window"，而不是"one RTT measurement per segment"。对于一个长肥信道而言需要使用更好的往返时间测量机制：时间戳选项，它允许对于窗口内的多个segment进行计时，也可以对重传segment进行计时。
- **(4)** TCP对于传输中的每个1byte数据使用32bit序号进行标识。当网络中有一个"delayed"报文段，它所在的TCP连接已经被释放，并发送端和接收端已经重新建立了新的连接，怎样防止这种"delayed"报文短出现在新的连接中呢？
- **(4-a)** IP首部的TTL字段为每个IP数据分组规定了生存时间上限：255s或者255hops，当"delayed segment"到达上限后被自动丢弃。
- **(4-b)** MSL(maximum segment lifetime)同样限制了报文分段的生命期，另外[chapter18中介绍的TCP平静时间概念](http://localhost:4000/tcp-ip/2021/09/25/post-tcp-ip-tcp-chapter-18/#3-平静时间概念quiet-time-concept)避免了主机故障重启后，由于收到之前建立连接传输的"delayed segment"而产生混淆的情况。
- **(4-c)** TCP的序号空间有限，当传输了**4,294,967,296byte(4Gbyte)**数据后会被重用。如果在一条传输速率较快的信道上，特定序号的报文分组出现延迟，但是仍然在连接有效期间到达，即网络足够快以至于在不到一个MSL的时间内序号就发生了"回绕"(发送端再次发送相同序号的报文，但延迟到达的报文尚未接收)。
- **(5) 关于不同链路的"回绕"时间的计算** T3电话线路(45Mb/s)将在12min发生回绕，FDDI(100Mb/s)回绕时间为5min，千兆网络(gigabit network:1000Mb/s)只需34s就会发生回绕。回绕时间取决于不同链路的带宽，而不再是带宽和时延的乘积。
- **(6) 数据分组发生序号回绕的解决方式** TCP时间戳选项中的PAWS保护回绕序号算法(protection against wrapped sequence numbers)可以应对这类情况。

##### 3 千兆网络案例分析(gigabit network)
当网络传输达到一定速率后，things change。[Partridge 1994]详细介绍了gigabit network的细节。本部分内容重点关注时延(latency)和带宽(bandwidth)之间的差别。

考虑在一条穿越整个美国的链路上传输大小为1,000,000byte文件的情况，假定时延固定为30ms，下图展示了使用两个不同传输带宽的链路对于该文件进行传输的情况。左边使用T1电话线路(1544000bit/s)传输这些数据，右边使用1Gb/s网络传输这些数据。
<center><img src="/img/in-post/tcp-ip_img/tcp_24_7.pdf" width="100%"></center>
- 图中黄色部分表示正在信道中传输的数据所处的位置(是从开始传输经过30ms后的状态)。
- **[1] 不同链路，不同带宽下，传输相同数据占用信道带宽情况分析** 经过30ms后，传输数据的第一个byte都已经到达对端。**(i)** 对于T1电话线链路而言，管道容量计算为：$\frac{1544000bit/s \times 30}{8\times 1000}=5790byte$，即管道只能容纳5790byte数据，则发送端仍然有994210byte数据待发送。**(ii)** 对于1Gb/s网络链路而言，管道容量计算为：$\frac{1,000,000,000bit/s \times 30}{8\times 1000}=3,750,000byte$，即管道可以容纳3,750,000byte数据，即整个传输的文件只占用25%的带宽。
- **[2] 不同带宽下，影响总数据传输时间的主要因素分析** 使用T1电话线路传输该文件的总时间为5.211s，如果使用T3电话线路(45,000,000bit/s)传输该文件，则总时间减少为0.208s；即增加29倍带宽可以将总时间减少到原来的25分之一(增加带宽能够显著影响传输时间)。使用1Gb/s链路传输该文件的总时间大约为0.038s(30ms时延+8ms传输数据时间)，假如能够将带宽增加到2Gb/s，将只能将总传输时间减少到0.034s(30ms时延+8ms传输数据时间)；即当前带宽加倍却只能将总时间减少10%。因此，当传输速率为1Gb/s下，对于总传输时间而言，时延是主要影响因素，带宽并不成为限制。
- **[3] 关于传输链路中的时延时间分析** 时延时间的长短主要和光速以及通信链路长度相关，光速是不能够改变的(unless Einstein was wrong)。
- "The effect of this fixed latency becomes worse when we consider the packets required to establish and terminate a connection." - Tcp-ip illustrated vol1原文，难以理解，缺乏结合案例分析。


### 四：窗口扩大选项(window scale option)
窗口扩大选项将TCP的窗口定义从16bit增加到32bit。窗口扩大选项并不改变TCP首部格式，而是通过定义一个选项用于对16bit窗口执行"scaling operation"来完成。

[**chapter18中TCP选项格式**](http://localhost:4000/tcp-ip/2021/09/25/post-tcp-ip-tcp-chapter-18/#1-rfc-793--rfc-1323定义的选项格式)部分介绍了窗口扩大选项的格式：其中1byte移位数(shift count)取值介于0～14之间，表示考虑窗口扩大因子的窗口范围介于65535×(2^0~2^16)byte之间。

##### 1 窗口扩大选项的几个基本特性&用法
- **(1) 选项只能用于SYN报文中** 窗口扩大选项只能出现在SYN报文中，因而当TCP连接"3次握手"完成后，不再能改变窗口扩大选项，每个连接方向的扩大因子是固定的。
- **(2) 连接的两个方向上的选项相互独立** 主动建立连接的一端在其发送SYN信号时发送这个选项，被动建立连接的一端只有收到带有该选项的SYN信号后，才能在返回的SYN信号中发送该选项；因此因此，连接中每个方向上的窗口扩大因子可以不同。
- **(3) 选项的兼容性** 当主动连接的一端(active open)发送了一个非0的窗口扩大因子(window scale factor)，但是没有收到另一端返回的窗口扩大因子选项，它就将发送和接受的shift count置0。该机制保证支持window scale option的新系统和不支持该选项的旧系统进行交互操作(interoperate)。Host Requirements RFC规定TCP强制接受最大报文大小(MSS - only appear in SYN)，并且它允许TCP忽略所有它不支持的(understand)的选项；这个规定增加了TCP对于选项的兼容性。
- **(4) 举例分析窗口扩大选项的使用方式** 发送端设置shift count=S，接收端返回的shift count=R。因此，每次收到16bit advertised window，都向左移动R位以获得实际窗口大小；每次发送32bit advertised window到另一端，都将其右移S位，用于填充TCP首部中16bit窗口的数值。
- **窗口扩大因子(shift count)具体数值是如何确定的** TCP根据接受缓存大小自动选择shift count数值，但是通常应当为应用进程提供修改接口。

##### 2 窗口扩大选项相关案例分析
**从vangogh(4.4BSD:支持窗口扩大选项)使用sock程序建立TCP连接 & 观察其使用窗口扩大选项的情况**
<center><img src="/img/in-post/tcp-ip_img/tcp_24_8.pdf" width="100%"></center>
- 从vangogh到bsdi主机共建立两次TCP连接，指定两次连接的接收缓存分别为128000byte和220000byte。

**使用tcpdump程序打印上面sock程序的数据分组交换信息 & 简单分析**
<center><img src="/img/in-post/tcp-ip_img/tcp_24_9.pdf" width="100%"></center>
- **[connection#1]** vangogh通告一个65535byte的窗口，并在SYN信号分组设置窗口扩大因子为1。vangogh通告窗口(65535)是一个小于接收端缓存大小(128000)的最大可能数值。
- **[connection#1]** vangogh通告窗口(65535)以及扩大因子(1)表明，vangogh主机"would like to send window advertisement up to 65535×2"。
- **[connection#1]** 上述带有窗口扩大因子的窗口通告将会对接收端缓存大小起到调节作用。但是本案例中，由于接收端bsdi主机返回的SYN信号中没有发送窗口扩大选项，因此vangogh通告的窗口扩大因子没有被使用，后续的传输中仍然使用初始通告窗口(65535)进行通信。
- **[connection#2]** vangogh通告了65535byte的窗口，并设置窗口扩大因子为2，这表明它"希望"发送的通告窗口范围是0~65535×4(同样大于接受缓存)。


### 五：时间戳选项(timestamp options: for RTT computing)
时间戳选项允许发送端在每个报文分组(every segment)中记录一个时间戳数值(timestamp)，接收端在确认ack中reflect这个数值，从而允许发送端根据每个收到的确认ack信号计算一个RTT数值。需要注意的是：上述计算的RTT数值是针对1个ack来计算的，并不一定是针对单个segment计算的，因为1个ack可能用于确认多个报文segment。

##### 1 RTT测量方式 & 混叠效应 & 产生的影响
RFC-1323-3.1给出较大窗口需要更优的窗口计算方式的理由：通常，往返时间RTT在较低的采样率下测量得到(once per window)，导致所估计的RTT数值带有一定的窗口混叠(aliasing)。对于"8 segmemts per window"而言，采样率为$\frac{1}{8} \times 数据传输速率$是可以接受的(混叠效应不明显)；但对于"100 segments per window而言"，采样率为$\frac{1}{100} \times 数据传输速率$，混叠效应将显著影响RTT的测量准确性。

由于混叠效应，窗口中数据分组数量越多，测量得到的RTT数值越小，因此超时重传时间计算([参考chapter21超时与重传](http://localhost:4000/tcp-ip/2021/10/07/post-tcp-ip-tcp-chapter-21/#三往返时间测量))越短，将会引起很多不必要的重传。如果用于测量RTT数值的窗口中，有部分报文分组丢失，则测量得到的RTT数值相比实际准确往返时间更小，情况会"get worse"。


##### 2 "n segments per window"中RTT测量的混叠效应(n数值越大，混叠效应越明显，测量的RTT数值越小)
<center><img src="/img/in-post/tcp-ip_img/tcp_24_10.pdf" width="100%"></center>
- 上图是根据[chapter21-往返时间RTT测量图](http://localhost:4000/tcp-ip/2021/10/07/post-tcp-ip-tcp-chapter-21/#3-往返时间rtt的测量)稍加修改得到的，用于说明为什么当单个窗口包含多个数据分组时，测量得到的RTT数值会偏小。
- 第一个RTT时间测量是最准确的，因为慢启动初始单个窗口只包含1个数据分组。这也是为什么smoothed RTT数值计算中，新的RTT估计值中90%来自之前估计的RTT数值，10%取决于当前值。
- 第二个RTT时间测量中，单个窗口包含2个数据分组：第二个数据分组通常不等到第一个分组到达对端就已经被发送，因此两个数据分组之间存在"aliasing"部分，这也是导致RTT测量误差的原因之一。
- 随着慢启动单个窗口发送数据分组数量越来越多，上述数据分组混叠效应越来越明显，当达到"8 segment per window"时，混叠误差尚可接受，但当达到"100 segment per window"时，混叠误差将导致RTT数值相比实际值明显偏小，后续计算的得到的重传时间也会偏小(将引起不必要的超时重传)。

##### 3 时间戳选项格式 & 相关信息
- **[chapter18 TCP选项格式](http://localhost:4000/tcp-ip/2021/09/25/post-tcp-ip-tcp-chapter-18/#1-rfc-793--rfc-1323定义的选项格式)展示了时间戳选项的基本格式**，发送端在"timestamp value"字段设置一个32bit数值，接收端在应答时回显该数值。
- 所有包含TCP时间戳选项的首部长度从20byte增加到32byte。
- 时间戳是一个单调递增的数值，接收端只需要回显收到的内容，因此无需关注时间戳的单位是什么。
- 时间戳选项不需要在两个主机之间进行任何形式的时钟同步，因为往返时间的计算只在同一台主机上进行。
- RFC-1323推荐在1ms~1s之间选取一个固定时间，将时间戳数值+1。e.g. 4.4BSD在启动时将时间戳设置为0，每隔500ms将时间戳时钟+1。
- 时间戳选项只能在SYN信号中倍初始化设置。主动发起端(active open)在SYN报文中指定时间戳选项，但是只有当对端收到这个选项后，该选项在后续的数据分组交换过程中才被设置(启用)。

<details>
      <summary><b>[点击展开] 窗口扩大选项tcpdump程序输出案例中，利用两个时间戳计算时间间隔</b></summary>
      <center><img src="/img/in-post/tcp-ip_img/tcp_24_9.pdf" width="100%"></center>
</details>
- 其中第一行SYN信号的TCP选项中时间戳数值为995351，第十一行SYN信号的TCP选项中的时间戳数值为995440。相差89个时间戳单元，每个单元500ms，所以实际时间相差44.4s。

##### 4 TCP时间戳选项的更新算法
当发送端连续发送2份数据分组，接收端返回单个确认ack同时确认这2个分组时，返回的ack信号应当回显哪个时间戳报文呢？下面介绍的时间戳更新算法能够解决这个问题。

为了减少任意一端需要维持的状态数量，每个TCP连接只维护(maintain)**一个**时间戳数值。下面介绍这种时间戳的更新算法：
- **(1)** TCP跟踪将在下一个ack信号中发送的时间戳的数值(tsrecent)，以及上一份发送的ack信号中的确认序号数值(lastack)。
- **(2)** 当一个包含lastack序号的数据分组到达时，该数据分组中的时间戳数值被保存在tsrecent中。
- **(3)** 每当返回时间戳选项时，tsrecent被作为时间戳回显(echo)被返回，期望收到的序号被保存在lastack中。
- **(2)-(3)** 不断循环，直到完成整个数据的传输。

该算法能够处理下面两种情况：
- **(1)** 如果发送端发送的数据分组对应的确认ack信号被接收端延迟返回(1个ack对多个数据分组进行确认应答)，则回显的时间戳数值应为最早被确认确认的数据分组(回答了之前的疑问)。e.g. 接收端收到序号分别为1～1024和1025~2048两个数据分组，每个分组都带有一个时间戳选项，接收端产生一个确认序号为2049的ack信号对它们进行确认，此时确认ack信号中包含的时间戳应为1～1024数据分组的时间戳。
- **(2)** 如果接收端收到一个数据分组，分组序号在窗口范围内，但它是一个失序报文(out-of sequence)，即前面的数据分组可能已经丢失。当丢失or时延的数据分组到达接收端后，丢失分组而不是失序分组的时间戳将被回显。e.g. 假定有3个分别包含1024byte的数据分组，它们按照如下顺序被接收：segment#1(1~1024)、segment#3(2049~3072):失序、segment#2(1025~2048):丢失。返回的确认ack信号依次为：ack 1025(timestamp for seg#1)：一个正常的针对分组1返回的确认信号；duplicate ack 1025(timestamp for seg#1)：一个异常的针对丢失的分组2的请求信号；ack 3073(timestamp for seg#2)：一个特殊处理的带有丢失分组时间戳的确认信号。

**正常&非正常的数据分组传输过程中的timestamp处理方式**
<center><img src="/img/in-post/tcp-ip_img/tcp_24_11.pdf" width="100%"></center>
- **explain for (2):解释为什么丢失分组到达时，回显丢失分组时间戳而不是失序分组时间戳？结合"abnormal data transmission"的时序图分析**：回显丢失分组(segement#2)，根据发送segment#2和回显分组计算得到的往返时间如图中"RTT#1"所示；如果回显失序分组(segment#3)，根据发送segment#3和回显分组计算得到的往返时间如图中"RTT#2"所示。
- **RTT时间估计值过高要比估计过低要好** RTT时间估计过高(回显丢失分组的时间戳)，会使得TCP超时重传时间过高，通常这是可以接受的；如果RTT时间估计过低(回显失序分组的时间戳)，会使得TCP超时重传时间过低，通常这将导致一些不必要的超时重传，无形中增大了网络拥堵的可能性。


### 六：PAWS(protection against wrapped sequence numbers):序号回绕防止措施
##### 1 一个假定的TCP连接(用于说明PAWS防止序号回绕算法的原理)
不妨假设存在一个使用了窗口扩大选项的TCP连接，其最大可能的窗口是1000M byte(1G byte)，并假定该连接使用时间戳选项，并且在发送端指定时间戳更新间隔(每发送一个窗口大小的数据时间戳增加1)。<br>

注意，根据最大窗口和窗口扩大因子进行计算，理论上最大的窗口大小为$65535\times 2^{14} = (2^{16} -1) \times 2^{14}$，而不是$1000M=2^{30}=2^{16}\times 2^{14}$。

##### 2 使用上述假定的TCP连接传输6000M byte(6G byte)数据时，连接两端的传输数据流
<center><img src="/img/in-post/tcp-ip_img/tcp_24_12.pdf" width="100%"></center>
- **[假定#1]** 假定一个数据分组在时间B丢失并引起了重传，在时间E该丢失的数据分组重新出现(由于网络延时，导致其传输时间超时)。
- **[假定#2]** 假定该传输链路速度很快，丢失的报文段在MSL的时间内重新出现，否则这个delayed segment将会在它的TTL到期时被中间路由器丢弃。
- **[问题的提出]** TCP的序号空间是有限的，总共能存储4G个序号，因此上述数据传输在时间E发生**序号回绕**。发送的序号为1G:2G的"丢失的"报文段恰好在"序号回绕"时返回，此时时间F刚好发送序号为1G:2G，那么在收到序号相同的报文后，如何区分它是"delayed segment"还是时间F发送的数据分组？
- **[PAWS算法解决上述问题的方式]** 虽然序号发生回绕，单纯从两个数据分组的序号无法将"delayed segment"和时间F发送的分组区分，但是结合它们的时间戳则很容易区分："delayed segment"的时间戳为2，而时间F发送的分组时间戳为5or6。
- **[关于PAWS算法]** PAWS算法不需要TCP连接两端任何形式的时间同步，之需要接收端返回的时间戳数值单调递增，并且每发送一个窗口的数据至少增加1(保证不同窗口的数据分组时间戳不同)。


### 七：T/TCP(a TCP extension for transactions) - TCP协议的拓展协议
TCP提供的是一种"virtual-circuit transport sevice"，这种service类型非常适合e.g. "remote login"和"file transfer"等应用场景。一个TCP连接的生命期有3个不同的阶段：连接建立、数据传输、连接中止。

##### 1 什么是transactions service (client request followed by a server response)？
transactions service的特征如下所示：
- **(1)** 应当避免连接建立和连接中止的overhead。when possible，直接发送一个请求报文并接受一个应答报文进行服务通信。
- **(2)** TCP连接的时延(latency)应减少至RTT+SPT：其中RTT(round-trip time)为往返时间，SPT(server processing time)为服务端处理请求的时间。
- **(3)** 服务端应具有检查duplicate request的能力，并且当收到duplicate request时，不会重复处理transactions，而是返回已保存的、和该dup request对应的response。

一个使用上述transactions service的实例是：chapter14介绍的域名服务(尽管DNS和处理重复请求无关)。

##### 2为什么需要本节介绍的T/TCP？
- 如果应用程序设计人员在选择TCP或UDP时，TCP提供了过多的transaction features，UDP提供的transaction features的不足。那么通常应当使用UDP来构造，欠缺的features(e.g. dynamic timeout and retransmission, congestion avoidance)交给应用层来构建。
- 使用UDP+应用层的实现方式存在的问题：不同的应用程序需要对于一些通用的功能(动态超时重传、拥塞避免etc.)不断进行重复实现，每设计一个应用程序就需要至少实现一次。一个较好的解决方案是：提供一个处理足够多的事物功能的运输层，即T/TCP。

##### 3 T/TCP避免建立连接的3次握手机制
- **[1] T/TCP维护一个全局计数器(用于对建立的TCP连接计数)** T/TCP为打开的(active open & passive open)指定一个连接计数(connection count, CC)。主机的CC数值由一个全局计数器分配，该计数器每次使用后，计数+1。
- **[2] 新的T/TCP选项:CC** 在使用T/TCP的两端之间传输的**每个数据分组**都包含一个新的TCP option:CC。该选项占用6byte，包含了发送端的对应该连接的32bit CC数值。
- **[3] 为每个对端维护一个存储CC数值的cache** 主机为每个对端主机分别维护一个cache，该cache存储**上一个连接**接受的数据分组的CC数值，存入cache的CC数值来自对端主机的可接受的SYN信号数据分组。
- **[4] 有选择的执行TCP三次握手** 当通过initial SYN数据分组收到CC option时，接收端需要比较收到的CC数值和维护的cache中CC数值(last CC value received in SYN)。如果接收的CC数值比cache中的CC数值大，则说明该SYN用于传输新的数据分组，报文中所有数据分组被传递给接收端应用进程。这样建立的连接称为"half-synchronized"连接。如果接收的CC数值比cache中的CC数值小，或者主机上找不到该对端的cache，则执行正常TCP三次握手。
- **[5] 关于CCECHO选项** 作为对initial SYN报文的应答，对端发送的SYN+ACK报文使用另一个称为"CCECHO选项"的字段回显initial SYN报文的CC数值。
- **[6] 非SYN报文分组中的CC数值的作用** 主要用于detect & reject任何来自previous incarnations of the same connection的duplicate segments。

上述T/TCP的"accelerated open"避免了建立连接的3次握手，除非客户端or服务端已经崩溃并重新启动。而T/TCP这种机制的代价只是服务端需要维护一个cache用于保存客户端收到的分组的CC数值。

##### 4 T/TCP缩短处于TIME_WAIT状态的时间(通过动态测量连接两端的RTT数值实现)
TIME_WAIT状态时延一般被设置为8倍的重传超时时间RTO(retransmission timeout value)。RTO数值的计算和测量的RTT时间相关。

缩短TCP连接处于TIME_WAIT的2MSL等待的时间，能够极大程序增加TCP连接的"transaction rate"。根据习题18.14可知：TIME_WAIT的2MSL等待时间将会使得TCP连接两端的"transaction rate"降低至268次/s。

**minimal transaction sequence包含如下三个报文段**
<center><img src="/img/in-post/tcp-ip_img/tcp_24_13.pdf" width="100%"></center>
- 从客户端应用进程的角度而言：该进程对于自身发送的请求的响应时间为"RTT(round-trip time)+SPT(segment processing time)"。

##### 5 Tcp-ip illustrated vol1的参考文献中关于T/TCP的CC选项的具体实现的instructions
- **(1)** 服务端的SYN信号和ACK信号需要被延时发送，以等待服务端进程处理完client-data后将服务端应答数据(reply)和它们一起发送。但需要注意时延的时间不能超过客户端超时时限，否则引起超时重传。
- **(2)** 客户端发送的请求(request=client-data)可以分成多个数据分组进行发送，但服务端必须能够对这些request分组失序到达的情况进行处理。通常情况下，如果一个数据分组在SYN信号之前到达，则该分组被丢弃并产生一个复位信号；如果使用T/TCP，则这些失序报文将放入队列中被依次处理。
- **(3)** T/TCP提供的API必须允许服务端进程能够通过单个operation完成数据发送&关闭连接，从而使得服务端的FIN信号和reply(server-data)能够一同发送。(通常是服务端应用进程执行"write the reply"，然后关闭连接，导致发送FIN信号)
- **(4)** T/TCP实现中，由于client-data和client-SYN信号合并发送，导致在收到server返回的MSS数值通告前，就会发送数据。为了避免在不知道对端MSS数值的情况下，默认发送≤536byte长度的数据，一个给定主机的MSS数值应当和其CC数值一同被缓存(cached)。
- **(5)** T/TCP实现中允许：客户端在尚未收到服务端的通告窗口时，也可以向服务端发送数据。T/TCP建议的默认窗口大小为4096byte，并为为服务端缓存(cache)拥塞门限(congestion threshold)。
- **(6)** 当T/TCP使用"minimal three-exchange"报文段通信时，每个连接方向只能计算一个RTT数值。另外还需要注意到：客户端测量的RTT数值包含了服务端应用进程处理client-data的时间；这意味着：smoothed RTT数值和其方差也需要被缓存(cached)。

##### 6 Fancy features of T/TCP
- T/TCP协议对于现有的协议做了最小的修改，但实现了对现有的TCP实现的兼容。
- T/TCP协议利用了TCP中现有的工程特性(e.g. 动态超时重传、拥塞避免机制etc)，而不是强制将这些必要组件交给应用进程处理。

##### 7 Alternative protocols of T/TCP
一种可代替T/TCP的协议是VMTP(versatile message transaction protocol)，即通用报文事务协议。RFC-1045对VMTP进行了详细的描述。和T/TCP不同的是：VMTP是一个完全依赖IP传输层的协议(和original TCP有较大不同)。VMTP支持error detection、retransmission、duplicate suppression，它也支持多播通信。

### 八：TCP的性能度量&评价
1980s出版的数据统计信息显示：TCP在以太网上能够以100,000~200,000byte/s的速率传输。截止tcp-ip illustrated vol1第一版发行，TCP实现可以按照800,000byte/s的速率传输甚至更快。

##### 1 一个典型的以太网传输的TCP数据报文、确认ACK应答报文大小
<center><img src="/img/in-post/tcp-ip_img/tcp_24_14.pdf" width="50%"></center>

##### 2 在10Mb/s传输速率的以太网上计算理论TCP最大吞吐量
**[1]** 假定发送端传输2个back-to-back的full-sized数据报文段，接收端为这2个数据报文返回一个确认ACK应答报文，计算TCP最大吞吐量如下：
$$
throughout = \frac{实际有效传输数据长度(byte)}{总数据报文长度(byte)} \times 线路传输速率= \frac{2\times 1460byte}{2\times 1538byte + 84byte} \times \frac{10,000,000bit/s}{8b/byte} = 1,555,064byte/s
$$
- 实际有效传输数据长度(byte) = 传输的2份数据报中userdata部分 = $2\times 1460byte$。
- 总数据报文长度(byte) = 传输的2份数据包的总长度 + 确认ACK信号部分 = $2\times 1538byte + 84byte$。

**[2]** 如果允许发送端发送数据的窗口开到最大值65535byte(不使用窗口扩大选项)，可以容纳44个1460byte报文段(注意窗口只限制数据部分的大小)。如果接收端对于每22个数据分组返回一个确认ACK信号，那么计算此过程TCP最大吞吐量如下：
$$
throughout = \frac{实际有效传输数据长度(byte)}{总数据报文长度(byte)} \times 线路传输速率= \frac{22\times 1460byte}{22\times 1538byte + 84byte} \times \frac{10,000,000bit/s}{8b/byte} = 1,183,667b/s
$$

**[3]** 上述**[1][2]**只是理论计算的TCP最大吞吐量限制，并且需要做出如下假定：
- 接收端返回的确认ACK信号没有和发送端发送的数据分组在传输中出现"collide"现象。
- 发送端可以按照"minimum Ethernet spacing"发送数据报；接收端可以在"minimum Ethernet spacing"时间内产生确认ACK报文。

**[4]** 实际运行的TCP吞吐量比上述理论计算的数值要低。实际应用中TCP吞吐量的限制因素有：
- 实际运行中，整条线路传输速率将受到最慢的传输链路的限制。
- 实际运行中，数据的处理速度将受到整条线路中内存运算速度最慢的限制。
- 实际运行中，整条线路的传输速率将会受到$\frac{接收端提供的窗口}{往返时间RTT}$的限制。即"带宽-时延乘积方程"：用最大允许窗口作为带宽和时延的乘积，以求解带宽速率。

上述计算的一个重要的意义在于：揭示了TCP的最高运行速率上限是由TCP的窗口大小和光速决定的。如[Partridge and Pink 1993]中计算的那样：许多协议性能问题在于实现中的固有缺陷而不是协议本身的限制。


## Reference
> \<tcp-ip: illustrated vol1\> chapter24 <br>

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用\`\`将关键字包含在内 <br>
> 8 `<center><img src="/img/in-post/economics_4/xxx.png" width="60%"></center>` <br>
> 9 使用html设置图片文字环绕方式：<br>
    `<div>` <br>
        `<img src="/img_path" align="left" width="40%" hspace="" vspace=""/>` <br>
        `<p>paragraph1 around the picture</p>` <br>
        `<p>paragraph2 around the picture</p>` <br>
        `<p>paragraph3 around the picture</p>` <br>
    `</div>` <br>
> 10 `<font style="color:red; font-weight:bold">加粗蓝色</font>`用来设置字体颜色 <br>
> 11 使用html设置可折叠部分内容：<br>
  `<details>` <br>
      `<summary><b>[点击展开] xxx</b></summary>` <br>
      `<center><img src="/img/in-post/tcp-ip_img/tcp_20_9.pdf" width="100%"></center>` <br>
  `</details>` <br>
> 12 问题脚注: ???problem😫problem???
