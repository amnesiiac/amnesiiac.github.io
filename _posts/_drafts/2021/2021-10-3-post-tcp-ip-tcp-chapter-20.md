---
layout: post
title: "tcp-ip: TCP (成块数据流处理)"
subtitle: '[tcp-ip: illustrated vol1] - chapter20' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: true 
date: 2021-10-03 12:55
lang: ch 
catalog: true 
categories: tcp-ip 
tags:
  - Time 2021
  - tcp-ip illustrated vol1 
---
### 一：引言
在chapter15中，TFTP简单文件传输协议客户端和服务端之间的通信需要依据"停止等待协议"，即发送端在发送下一个数据块之前需要等待接收方返回的对上一份数据的确认信号。

本章中将介绍另一种流量控制协议：滑动窗口协议(sliding window protocol)，即发送端在等待接收端返回的数据接受确认之前可以连续发送多个数据分组，发送端无需每发送一个分组就停下来确认，因此相比"停止等待协议"能够加速数据传输。

另外，本章还将介绍TCP的PUSH标志位，慢启动，以及成块数据流的吞吐量相关知识内容。


### 二：普通TCP传输数据流(normal data flow)
##### 1 使用sock命令在bsdi主机、svr4主机启动TCP服务端、客户端程序
<center><img src="/img/in-post/tcp-ip_img/tcp_20_1.pdf" width="100%"></center>

##### 2 使用前一部分的sock程序从svr4到bsdi的TCP数据传输数据分组交换时序图
<center><img src="/img/in-post/tcp-ip_img/tcp_20_2.pdf" width="100%"></center>
- segment#7为什么同时包含对于segment#4和segment#5的数据部分的确认？在服务端接收到客户端svr4发来的segment#4数据时，将该TCP连接设置为"产生delayed ack"的连接，因此对于接收到的segment#4数据的确认ack信号并不会立即发送，而是等待下一次内核timer定时器溢出时再将合并的所有数据确认ack信号返回客户端。
- segment#8单独返回了segment#6的数据确认信号，而不是等到segment#6和segment#9数据接收后，在统一发送它们的确认ack信号。这是因为：在收到segment#9数据前，内核timer溢出，因此直接返回"现有的未确认"数据的确认ack。
- segment#11~segment#16和上述segment#4~segment#7的过程类似：segment#11的数据到达，将该TCP连接设置为"产生delayed ack"的连接，当下一次内核timer溢出时，返回segment#14合并确认报文；segment#13数据到达后，再次将该TCP连接设置为"产生delayed ack"连接，当下一次内核timer溢出时，返回segment#16合并确认报文。
- segment#7、segment#14、segment#16中针对多个数据分组都进行了"合并的确认"。使用TCP滑动窗口协议时，TCP连接接收方不必对每个收到的数据分组进行确认。这种合并的确认表示"接收端已经正确收到了发送的确认序号-1之前的所有数据"。

##### 3 图20-2 svr4到bsdi之间使用TCP滑动窗口协议进行通讯的另一个例子 - ditto, omitted

##### 4 从fast sender发送数据给slow receriver的数据分组传输时序图
<center><img src="/img/in-post/tcp-ip_img/tcp_20_3.pdf" width="100%"></center>
- 发送端sun主机以"back-to-back"方式连续发送4个数据分组segment#4~segment#7
- segment#8返回对接收到的4个数据分组的确认ack信号，但是由于slow receiver的应用程序进程尚未读取这些数据，及receiver接受的数据都在TCP缓冲区，因此其通告窗宽为0。
- segment#9发送一个确认信号，虽然它看起来像一个用于确认数据接受的报文，但实际上它的用处是用来通知另一端增加窗宽，即它实际上是一个窗宽更新报文。
- 注意segment#13中包含了两种标志信号(FIN和PUSH)，随后的两份报文segment#17和segment#18都对该报文中的FIN信号进行确认。


### 三：滑动窗口(sliding window)
##### 1 数据发送端滑动窗口协议示意图
<center><img src="/img/in-post/tcp-ip_img/tcp_20_4.pdf" width="60%"></center>

##### 2 数据接收端滑动窗口协议示意图
<center><img src="/img/in-post/tcp-ip_img/tcp_20_5.pdf" width="60%"></center>

##### 3 "配合"上述接收端、发送端滑动窗口的一种数据传输时序图
<center><img src="/img/in-post/tcp-ip_img/tcp_20_6.pdf" width="60%"></center>

##### 4 滑动窗口协议相关内容整理&分析
- 窗口大小是：无需等待ack确认信号，可以直接发送的数据分组最大值。
- 系统开辟缓存空间是因为：发送端主机在等到数据确认应答之前，必须在缓存区中保留该部分数据。如果如期收到数据确认应答，则可从缓存区中将该数据删除。
- 当发送端一次性将接收端可以接收的大小的数据发出后，此时发送端实际可用的窗口大小为0，称之为0窗口，表明在收到下一个数据确认应答之前，发送端不在能发送任何数据。
- 当发送端已经发送的数据被接收端确认&返回确认ack信号时，窗口的左边沿向右边沿移动，称为**窗口合拢**。
- 当接收端应用程序进程已经读取了已经返回确认ack信号的数据时，将数据占用的接收端TCP缓存中删除；此时发送端收到该部分数据的确认ack信号，将窗口右边沿向右移动所确认的数据个数，称为**窗口张开**。
- 当窗口的右边沿向左移动时，称为**窗口收缩**。Host Requirement RFC强烈建议不使用窗口收缩方式控制传输流量。

##### 5 以滑动窗口形式展示的[TCP数据传输分组交换过程](http://localhost:4000/tcp-ip/2021/10/03/post-tcp-ip-tcp-chapter-20/#2-使用前一部分的sock程序从svr4到bsdi的tcp数据传输数据分组交换时序图)
<center><img src="/img/in-post/tcp-ip_img/tcp_20_7.pdf" width="100%"></center>
- 发送端不一定必须发送全窗口大小的数据，多数情况会预留一定空间(发送的数据分组个数少于接收端通告的窗口大小)。
- 接收端返回给发送端的确认ack信号报文段的作用主要有：确认部分接收到的数据、将接收端窗口整体向右滑动。即窗口的大小、相对位置是依确认序号而定的。
- 上述segment#7到segment#8中揭示：滑动窗口大小可以减小，但是窗口的左边沿却不可以向左移动。即窗口收缩是被禁止的。
- 接收端在发送一个确认接收ack信号之前，不必等到接收端窗口被填满。如segment#10确认之前接收的数据时，并不是等到接收的数据充满接收端"全部可用窗口"后才返回(将返回win 0)，而是留有一定"余地"。

### 四：窗口大小
接收端的窗口大小通常可以收到应用进程控制，因此不同的接收端窗口设置方式将影响TCP数据传输的性能。

4.2 BSD系统默认设置发送&接收缓冲区大小为2048byte；4.3 BSD系统中发送&接收缓冲区大小被增加到4096byte；Sun 4.1.3、BSD/386、svr4仍然使用默认4096byte大小的缓冲区；Solaris 2.2、4.4 BSD、AIX 3.2使用更大的缓冲区大小：8192byte或16384byte。

使用socket API可以设置接收&发送缓冲区大小。接收端缓存大小是该TCP连接上接收端向发送端能够通告的最大的窗口大小，即接收端缓存区大小决定了通告窗口的上限。一些应用程序进程通过修改socket缓存大小来增加应用性能。例如，以太网传输链路中的两个工作站进行文件传输，默认的4096byte窗口大小并不是最佳选择，使用16384byte的窗口可以增加约40%左右的吞吐量。

##### 1 改变窗口大小影响案例分析
**[1] 使用"sock -R"控制接收端的缓存大小，在bsdi和sun主机上分别启动服务端、客户端程序**
<center><img src="/img/in-post/tcp-ip_img/tcp_20_8.pdf" width="100%"></center>
**[2] sock程序数据传输时序图**
<center><img src="/img/in-post/tcp-ip_img/tcp_20_9.pdf" width="100%"></center>
- segment#2在初始建立TCP连接时，提供了一个较大的窗口6144byte，因此，发送端连续发送了6次数据(共6144byte)，然后停止等待接收端返回确认应答(下一段发送数据的起始序号 & 新的窗口大小)。
- segment#10返回了确认序号，并提供了新的窗口大小2048byte。新的可用窗口大小缩小的原因可能是：接收端应用程序进程没有时间、额外空间或者没有必要再读取超过2048byte的数据。注意，接收端通告的窗口大小是根据自身应用程序运行情况指定的。
- segment#12将数据传输和FIN信号发送合并。
- segment#13的确认序号和segment#10相同，返回该报文的目的只是想通告一个更大的可用窗口大小。
- segment#14用于对于segment#11和segment#12发送的数据的确认。
- segment#15和segment#16仅仅用于通告一个可行的更大窗口。
- segment#17和segment#18用于正常的TCP连接关闭。

### 五：PUSH标志
**定义** PUSH标志是：发送端用于通知接收端将所收到的数据全部提交给接收进程；将被提交给接收进程的数据包括：和PUSH标志位一起传送的数据，以及接收端TCP已经接受的尚未提交给接收进程的数据。

**最初的TCP规范中**，一般规定编程接口允许发送端进程告诉它的TCP何时设置PUSH标志，即编程接口支持"受进程控制的TCP设置PUSH标志"。因此可以通过编程接口主动控制接收端不要因为等待额外的数据而使得已经接受的数据在接收端缓存区中滞留。

**目前的实现方式** 然而，目前大多数编程接口API都没有为进程提供控制TCP设置PUSH标志的方法。许多实现程序认为：一个好的TCP实现应该能够自行决定何时设置PUSH标志。

**伯克利的TCP实现(自动设置PUSH的例子)** 如果等待发送的数据分段是send buffer最后一条数据(将其发送后将导致empty send buffer)，大多数源于伯克利的TCP实现在这种情况下会自动设置PUSH标志。<br>
因此，我们可以观察到：每次应用进程执行写操作时都会设置PUSH标志，这是因文每次应用程序完成写操作后，数据会被立即发送，send buffer立即被清空，PUSH标志被自动添加。

##### 1 结合本文之前的几个时序图案例进行分析
<details>
    <summary> <b>[点击展开]</b> #1 TCP数据分组交换过程时序图 - tcp vol1 图20-1 </summary>
    <center><img src="/img/in-post/tcp-ip_img/tcp_20_2.pdf" width="100%"></center>
</details>
- 发送端发送的所有8个数据报文，segment#4~segment#6、segment#9、segment#11~segment#13、segment#15的PUSH标识位都被设置为1，这是因为发送端应用进程执行了8次"write"操作，每次写入1024byte数据，每次执行写操作后立即清空发送缓存，因此自动设置PUSH标志位。

<details>
    <summary> <b>[点击展开]</b> #2 TCP数据分组交换过程时序图 - tcp vol1 图20-7 </summary>
    <center><img src="/img/in-post/tcp-ip_img/tcp_20_9.pdf" width="100%"></center>
</details>
- segment#7报文段中的PUSH标志位置1，为什么数据没有完全发送完成，就设置PUSH标志位呢？这是因为sock程序进程指定发送8192byte长度的数据，而我们的接收端通告的缓存只有4096byte，因此发送端不等待接收端缓存占满前就设置PUSH标志，以"提前"清空接收端缓存，方便接收端接收下一部分数据。
- segment#12报文段中的PUSH标志置1，因为segment#12是发送端整个发送数据的最后一个报文段，因此负责将接收端的缓冲区清空(将接收端缓存的数据全部提交用户进程)。
- segment#14、segment#15、segment#16出现了三个连续的确认报文段。首先分析segment#14：产生该报文的原因是，当窗口大小增加了2×最大报文段长度(MSS=2×1024=2048)时or窗口增大了最大可能窗口的50%时，通告对端窗口被更新。其次考虑segment#15、segment#16：它们是两个不必要的窗口更新报文段，此时已经收到对端发送的FIN标志位(中止连接)，不会再发生数据传输。

<details>
    <summary> <b>[点击展开]</b> #3 TCP数据分组交换过程时序图 - tcp vol1 图20-3 </summary>
    <center><img src="/img/in-post/tcp-ip_img/tcp_20_3.pdf" width="100%"></center>
</details>
- segment#4~segment#7的4个报文端中PUSH标志位都被设置，这是因为它们中的每一个都使得TCP层产生一个segment，并接着单独发送给IP层。这种发送数据的方式称为"back-to-back"方式。
- 发送端TCP连接需要等待接收端的反馈，segment#8和segment#9用于通告接收端窗口更新。接着segment#10~segment#13是发送端用于传输剩余的数据(4096byte)。发送方知道segment#13是最后的数据传输报文，因此需要为其设置PUSH标志；另外，为了中止TCP连接，同时在segment#13中还设置了FIN标志。

### 六：慢启动(slow start)
**前面案例传输数据方式中存在的问题** 截止目前，本文中所有使用的案例中，发送端和接收端建立连接后，一开始总是发送多个报文段，直到第一次收到接收端返回的窗口大小通知后，才进行"有计划的收发数据"。这种方式在接收端、发送端同在相同局域网内时可行，但是如果在发送端和接收端之间存在很多路由器，或者存在速率较慢的网络时，可能出现问题：一些中间路由器需要缓存传输的数据，如果每个TCP连接一开始都发送多个大量的报文段，那么无形中增加了中间路由器负担，并且这种启动方式已被证实严重降低了网络吞吐量。

**慢启动和拥塞窗口的关系** 慢启动为发送端增加了另一个窗口：拥塞窗口(congestion window, cwnd)，当发送端和接收端主机建立连接时，cwnd被初始化为1segment宽，segment的大小即是接收端通告的报文段大小；发送端每收到一个ACK信号，拥塞窗口就增加1segment宽。发送端采用min(cwnd，接收端通告窗口)作为发送上限。

**拥塞窗口cwnd是发送端的流量控制手段，而通告窗口是接收端的流量控制手段。**

**关于拥塞窗口的增加和控制**：发送端开始发送一个报文段，然后等待ACK信号，此时cwnd=1segment；当发送端收到第一个ACK信号时，cwnd从1segment增加到2segment，此时可以发送2个报文段。当收到这2个报文段的ACK信号时，cwnd+=2即拥塞窗口变为4segment。<br>
以此类推，拥塞窗口的增加是一种近似指数增加的方式，但是，当某时刻传输的数据达到互联网容量最大值时，此时中间路由器开始丢弃新发送的数据分组，并将通过TCP超时、重传机制通知拥塞窗口过大；这部分内容将在下一章中介绍。

##### 1 从sun主机到vangogh的远程TCP连接的数据分组交换时序图(cwnd窗口的变化)
<center><img src="/img/in-post/tcp-ip_img/tcp_20_10.pdf" width="100%"></center>
- 上面的时序图展示了从sun主机到vangogh.cs.berkelay.edu的主机之间建立的TCP连接上的数据传输时序图。从sun主机到vangogh主机需要经过一个传输速率很慢的SLIP线路，该链路是该连接数据传输的瓶颈。
- 在建立TCP连接之后(MSS数值已经被设置)，发送端首先发送一封长度为MSS的数据分组(segment#1)，然后等待接收端返回的ACK信号。
- 在收到接收端返回的ACK确认信号后，拥塞窗口变为2MSS，并再发送两份数据报。收到segment#5的ACK确认信号后，拥塞窗口变为3MSS大小。
- 需要注意的是，慢启动的拥塞窗口并不是可以无限制增加的，当拥塞窗口大小达到"慢启动门限时"，就会采用"拥塞避免算法"来替代"慢启动"算法计算拥塞窗口大小；"拥塞避免算法"可以使得拥塞窗口成线性增长，更多相关的内容详见下一章中的内容。

### 七：成块数据的吞吐量(计算)
这一部分内容主要研究**窗口大小**、**窗口流量控制**、**慢启动**对于传输"bulk data"的TCP连接吞吐量的影响。

##### 1 16个离散时间单元时刻对应的数据传输snap
<center><img src="/img/in-post/tcp-ip_img/tcp_20_11.pdf" width="100%"></center>
- 在time0时刻，发送方发送一个报文段(标记为a)，此时发送端处于慢启动过程中，cwnd=1，在发送端继续发送数据分段前，必须等待该数据分段的确认。
- 经过time0、time1、time2、time3，所发送的报文段到达接收端，再经过time4、time5、time6、time7，发送端收到来自接收端返回的接收确认ACK信号(标记为$a)。这提示了该TCP连接的RTT时间为time0~time7。
- 当发送端收到来自接收端的确认ack信号后，cwnd+=1，因此在time8、time9连续发送两个报文段(将它们标记为b和c)。报文b和c传输到达接收端，产生两个确认信号(标记为$b和$c)，返回给发送端。$b和$c到达发送端之间的间隔和b和c到达接收端之间的间隔一致，将这种现象称为**"TCP self-clocking"行为**。实际情况下，返回路径上的消息队列处理机制会改变ack的到达速率。
- 上述每个离散时间snap中，数据分组画的相对较大，返回的确认ack信号绘制的相对较小，以符合它们占用空间的实际情况。数据分组包含IP首部、TCP首部、数据部分；确认ack信号包含IP首部、TCP首部。
- 注意上面绘制的数据分组、确认ack信号snap图是基于它们在TCP连接中传输速率相同而绘制的，实际情况并不总是如此(要复杂的多)。

##### 2 紧接上面16个离散时间单元数据传输snap的另外16个传输snap
<center><img src="/img/in-post/tcp-ip_img/tcp_20_12.pdf" width="100%"></center>
- 随着发送端数据继续向接收端传输，接收端不断返回数据接收确认ack信号，因此，发送端拥塞窗口不断增加，在time27时刻，拥塞窗口大小cwnd增加到8个MSS大小。
- 在time31时刻及其后续时间段的数据传输过程，发送端到接收端的TCP连接管道被数据&确认ack信号充满，此时无论拥塞窗口和通告窗口大小为多少，该连接都不能容纳更多的数据。此时，接收端每接收一个数据分组(从pipeline中移除一个数据)，发送端就会补充发送一个数据分组再将pipeline充满。这个状态称为TCP连接的理想稳定状态(ideal steady state)。

##### 3 通道容量的计算(pipeline capacity) - 带宽时延的乘积(bandwidth-delay product)
TCP连接建立的通道容量可以按照如下方式进行计算：
$capacity(bit) = bandwidth(b/s) \times round-trip time(s)$。<br>
即TCP连接通道容量取决于网络传输速率和TCP传输的往返时间。

以一条穿越美国的T1电话通信线路为例，已知其往返时间RTT=60ms，线路的传输速率为1,544,000bit/s，可以计算得该线路的通道容量为11580byte。

**原始传输速率(raw bit rate)和实际传输速率(actual bit rate)**：T1电话线路的原始传输比特率为1,544,000bit/s，由于传输的数据中，每193bit中需要占用1bit用于帧同步(framing)，因此该线路的实际传输比特率为1,536,000bit/s。

**改变传输带宽以及往返时间对于TCP连接的通道传输容量影响的示意图**
<center><img src="/img/in-post/tcp-ip_img/tcp_20_13.pdf" width="90%"></center>

##### 4 拥塞(congestion)
当数据从一个较大的连接通道向一个较小的连接通道传输时，便会发生拥塞。另外，当多个输入数据流到达一个路由器，但路由器的输出数据流小于输入数据流的总和时也会发生拥塞。

**一个简单的大连接通道(big pipe)向相对较小的连接通道(smaller pipe)发送报文的情况**
<center><img src="/img/in-post/tcp-ip_img/tcp_20_14.pdf" width="100%"></center>
- 大多数主机都连接在局域网上，并通过一个路由器和传输速率相对较低的广域网相连。
- 上图中，将路由器R1标记为"瓶颈"是因为它是拥塞发生的地方。它连接着两个传输速率不同的传输通道。
- R1和R3通常是相同的路由器，R2和R4通常是相同的路由器。路由器的传输不必须使用对称的路由路径。
- 上图中，假定发送端不使用慢启动，则发送端按照局域网的带宽尽可能快地发送序号1～20的报文段。如果瓶颈路由器的缓存不能容纳这20个数据分组，那么后续多余的数据分组将会被路由器丢弃。
- 另外，接收端返回的ACK信号的速率(报文之间的间隔时间)和传输线路上最慢的链路上一致。

### 八：紧急模式(urgency mode)
TCP提供了一种"紧急模式(urgenct mode)"机制，它可以使得TCP连接的一端通知另一端：以某种格式组织的紧急数据已经被放入普通传输数据流中。另一端接到这个通知后，由接收端决定如何处理这些特殊数据。

**在TCP数据报首部中设置和紧急模式相关选项** <br> 通过设置TCP首部中的"URG"标识位(URG=1)和16bit紧急指针来启动该功能。16bit指针需要置为一个正的偏移量，该偏移量必须和TCP首部中的序号字段相加以得到urgent data的最后一个字节的序号。<br>
关于紧急指针offset+TCP首部序号指向urgent data的最后一个字节还是指向urgent data的最后一个字节的下一个字节，最初的TCP规范中有两种解释，但是Host Requirements RFC指定该数值为urgent data的最后一个序号(虽然很多伯克利的实现都将其指向urgent data的下一个字节)。

**关于应用进程处于紧急模式和正常模式的判定** <br>
只要接收端当前读取数据的位置到16bit紧急指针指向的位置之间有数据存在，就可以认为接收端的应用程序处于紧急模式(urgent mode)；在读取位置经过紧急指针指向的位置之后，应用程序便转到正常模式。

**关于TCP的紧急模式和采用带外数据(out-of-band data)传输的区别&联系** <br>
许多TCP实现都将紧急模式和TCP带外数据通信相互混淆，导致它们混淆的原因是：主要的编程接口(socket API)将TCP的紧急方式传输的数据映射成socket所谓的带外数据。如果一个应用程序确实需要独立的带外信道进行通信，那么建立第二个TCP连接是最好的方案。

**TCP的紧急模式的常见应用：Telnet、Rlogin以及FTP** <br>
当使用交互程序的用户键入中断键时，TCP会采用紧急模式发送键入的中止字符到对端，采用紧急模式能够允许对端"冲洗"所有未处理的输入，并丢弃所有尚未发送的输出。

TCP紧急模式还有一个用途：如果服务端到发送端的数据传输可能由于客户端读取数据太慢而中止(TCP连接位于理想稳定状态)，如果此时它进入紧急模式，那么即使它不能发送数据，也可以通过发送紧急指针&设置URG=1给客户端，使客户端读取数据、打开窗口，使TCP通信pipeline中的数据流动起来。

FTP同样，允许使用带外数据传输方式来传输&执行对一个文件的传输中断，从而响应交互用户放弃一个文件传输的操作。

##### 1 紧急模式案例
将sun主机作为TCP通信发送端，bsdi主机作为TCP通信接收端，并通过sock程序传输数据，以观察在接收端bsdi主机的窗口关闭情况下，TCP连接是如何发送紧急数据的。

**(i) 发送端、接收端sock程序**
<center><img src="/img/in-post/tcp-ip_img/tcp_20_15.pdf" width="100%"></center>
- 设置发送端缓存为8192byte，以让发送端应用进程能够立即写入全部需要发送的数据。

**(ii) tcpdump对上述数据分组传输过程、进入紧急模式后的传输过程进行分析 & 打印相关分组信息**
<center><img src="/img/in-post/tcp-ip_img/tcp_20_16.pdf" width="100%"></center>
- 在接收端应用进程执行第一个读取操作之前，发送端的整个数据分组发送过程持续了几毫秒即中止。这是发送端sock程序设置缓存区大小为8192byte的原因：让TCP缓存区能够容纳所有待发送的数据，防止由于发送端应用进程中止导致数据不能完整完成传输过程。
- TCP发送端会将所有缓存区中的数据构成发送队列，并随时在允许的情况下发送出去。
- 每次发送端应用进程执行一个写操作，TCP输出函数都被调用。在紧急模式下，虽然不可能写任何数据，但是TCP输出函数仍然被调用(用来发送通告URG=1、紧急指针的数据分组)。

**(iii) 上述案例应用进程执行的"write"操作 & 传输的6145byte数据的序号 & 及其报文段分组情况**
<center><img src="/img/in-post/tcp-ip_img/tcp_20_17.pdf" width="80%"></center>
- 应用进程执行"write"操作，每次向发送端缓存区写入数据。注意，序号为4097的数据为TCP进入紧急模式下发送的紧急数据。
- 示意图下方的报文段为TCP连接每次发送的数据报分段对应的序号部分，不难发现，应用进程执行"write"操作写入的数据块和TCP每次发送的报文段不是完全对应关系。即当应用进程写入紧急数据时，需要借助URG标志位在每个报文段中标识出"该报文段是否包含urgent data"，还需要借助紧急指针指示紧急数据最后一个byte或者紧急数据的尾后byte。
- 根据tcpdump中紧急指针指示的数据序号为4098可知：sun主机系统的TCP实现将紧急指向urgent data的尾后byte。
- 上图显示：序号为4097的紧急数据是和正常数据合并在同一个segment中发送的，当TCP发现最后剩余1byte普通数据未发送，于是将剩余部分打包发送(见tcpdump打印的输出的第17行)。


## Reference
> \<tcp-ip: illustrated vol1\> chapter20 <br>

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
      `<summary>点击时的区域标题：点击查看详细内容</summary>` <br>
      `<center><img src="/img/in-post/tcp-ip_img/tcp_20_9.pdf" width="100%"></center>` <br>
  `</details>` <br>
> 12 问题脚注: ???problem😫problem???
