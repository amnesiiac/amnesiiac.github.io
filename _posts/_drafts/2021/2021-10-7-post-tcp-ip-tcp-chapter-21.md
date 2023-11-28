---
layout: post
title: "tcp-ip: TCP (超时与重传)"
subtitle: '[tcp-ip: illustrated vol1] - chapter21' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: true 
date: 2021-10-07 22:45
lang: ch 
catalog: true 
categories: tcp-ip 
tags:
  - Time 2021
  - tcp-ip illustrated vol1 
  - problem
---
### 一：引言
TCP借助返回确认接收ack信号来实现"可靠的"数据传输，但是数据本身和确认信号都可能丢失，因此TCP在发送时，设置一个定时器，当定时器发生溢出时(计时结束)仍然没有收到确认ack信号，那么就会重传该数据。

TCP关于超时和重传的实现的关键在于：怎样决定超时间隔(什么情况下执行超时策略)，以及制定什么样的超时策略(当超时发生时应当执行什么动作)。

**对于每个连接，TCP需要管理4个不同的定时器：**
- 重传定时器(retransmission timer)：用于界定另一端返回确认信号的时间限度。
- 持续定时器(persist timer)：用于即使对端已经关闭接受窗口，持续定时器仍然能够保持窗口信息不断流动(window size information flowing)。
- 保活定时器(keepalive timer)：用于检测一个闲置连接(idle connection)的对端何时崩溃或重启。
- 2MSL定时器(2MSL timer)：用于计算一个连接处于TIME_WAIT状态的时间。

**本文主要包含&介绍如下几部分内容：**
- 首先介绍一个简单的TCP超时重传案例，然后再介绍一个更加复杂的例子。
- 通过上述2个案例，可以研究TCP时钟管理的相关细节。
- 通过上述2个案例，可以研究TCP的典型实现是如何确定往返时间(RTT)的，以及如果利用往返时间等信息为后续传输的报文段建立超时重传时间。
- 然后对于TCP拥塞避免相关机制进行研究：即当传输的数据分组丢失时，TCP的应对措施。并且提供一个分组丢失的案例以进行分析。
- 最后介绍较新的快速重传、快速恢复算法，并讨论为什么该算法在TCP检测分组丢失的过程中比"等待时钟超时"效率更高。


### 二：超时和重传的简单案例

##### 1 在bsdi主机上运行telent程序建立TCP连接，发送字符数据"hello world"，拔掉电缆再发送字符数据"and hi"
<center><img src="/img/in-post/tcp-ip_img/tcp_21_1.pdf" width="100%"></center>

##### 2 使用tcpdump程序对于上述TCP连接的建立、数据传输、超时重传的过程进行分析(省略bsdi设置的TOS服务类型字段信息)
<center><img src="/img/in-post/tcp-ip_img/tcp_21_2.pdf" width="100%"></center>
- 检查tcpdump打印的输出中连续的超时重传报文的时间差，忽略小数部分，得到的"重传间隔"分别为1、3、6、12、24、48、64、64、64(s)...
- 实际上，TCP连接超时重传的首次重传时间为1.5s，而不是tcpdump程序所显示的1s，其中的原因参考[chapter18TCP连接和中止部分的内容](http://localhost:4000/tcp-ip/2021/09/25/post-tcp-ip-tcp-chapter-18/#3-第一次超时时间为58s而不是预设的6s的原因分析)。
- 超时重传时间在第一次超时RTO时间(1.5s)的基础上，每次增加1倍，直到超时时间到达64s。超时时间间隔每次变为前一次超时间隔的2倍的这种设置被称为"指数退避(exponential backoff)"。对于简单文件传输协议TFTP，每次超时重传都是在上一次超时重传5s之后发生。
- 从首次分组传输(tcpdump输出第6行，24.480s)，到TCP执行复位信号的传输(tcpdump输出第19行，566.488s)，整个过程时间间隔大约9min，该时间和特定TCP实现相关，对于大多数TCP的实现而言是不可变的。但solaris2.2允许管理员通过"tcp_ip_abort_interval"变量来控制整个超时重传时间(该系统整个时间默认设置为2min)。


### 三：往返时间测量
TCP超时与重传的一个重要的部分是：计算给定TCP连接的传输往返时间(round-trip time, RTT)。另外，TCP连接的传输路径经过的路由器经常变动，网络流量(net traffic)也随时变化，因此TCP确定往返时间的机制应当能够跟踪这些变化，并对超时重传时间进行调整。

##### 1 RFC-793推荐的RTO(retransmission timeout)计算方式
- **step#1:获得实际RTT** TCP测量：从发送一个带有特定序号的数据，到接收到包含该序号数据确认ack信号之间传输往返时间RTT，将这个往返时间使用M进行表示。需要注意，数据发送的序号和确认接收的序号必须匹配，因为数据分组发送和数据接收确认ack信号之间没有必然的1v1对应关系。
- **step#2:获得smoothed RTT** 使用"低通滤波"的方式重新定义往返时间，$R \leftarrow \alpha R + (1-\alpha)M$，其中$\alpha$为平滑因子，默认值取0.9。使用这种"平滑的往返时间"的意义是：每次按照step#1测量RTT后，新的smoothed RTT中90%来自之前的估计值，10%来自新的测量M值。
- **step#3:获得重传超时时间RTO** RFC-793推荐使用RTO，RTO的数值应当计算为：$RTO=\beta R$，其中$\beta$为默认值为2的delay variance factor。
- **关于超时重传时间这种计算方式的讨论** 当网络波动较大时，step#1计算得到的M值变动很大，但是使用smoothed RTT计算方式无法"紧跟"这种RTT变化(90%受之前估计值的影响)，则容易引发很多不必要的超时重传。例如：之前计算的估计值R较小，而后续网络波动较大，RTT测量值较大，使用smoothed算法导致新的RTO数值计算偏小，因此超时重传时间相比理想值要短的多。

##### 2 Jacobson 1988推荐的新的RTO计算方式 - smoothed D如何计算? ???problem😫problem???
- 根据Jacobson，均值偏差(mean deviation)是对标准偏差(standard deviation)的一种很好的逼近，并且更容易计算。
- **step#1:首先计算均值偏差Err** 利用当前**测量**的RTT数值M，以及smoothed RTT估计数值R(看作平均RTT的一种近似)，计算均值偏差为：$Err = M - R$。
- **step#2:使用均值偏差更新smoothed RTT数值R** 使用step#1计算得到的均值偏差更新R数值如：$A \leftarrow A + gErr$，其中g=0.125。
- **step#3:使用均值偏差更新smoothed 标准偏差D** $D \leftarrow D + h(\mid Err \mid - D)$，其中h=0.25表示偏差增益，D表示经过平滑的均值偏差数值。
- **step#4:使用smoothed RTT(R)和smoothed 标准偏差D计算新的重传时间RTO** $RTO = A + 4D$
- **关于计算公式中g、h的取值问题探究** 在选值时，$g=0.125$，$h=0.25$，$4g=0.5=2^{-1}$，$4h=1=2^0$，即g和h选值满足：数值的4倍为2的次方。这样设计是为了计算时，使用移位来代替乘、除运算。

##### 3 karn算法(解决重传多义性的问题)
在分组重传时会产生这样的问题：一个数据分组被发送，出现超时，超时重传时间执行类似[本文前面的案例](http://localhost:4000/tcp-ip/2021/10/07/post-tcp-ip-tcp-chapter-21/#2-使用tcpdump程序对于上述tcp连接的建立数据传输超时重传的过程进行分析省略bsdi设置的tos服务类型字段信息)的方式，即按照"指数退避"的方式进行传输；分组每次按照2倍于之前的RTO进行重传，此时收到一个确认ack报文。那么这个这个确认ack信号是这对第一个数据分组还是针对第二个分组呢？<br>
这就是所谓的"超时重传二义性问题"。

[karn and partridge 1987]规定：当超时和重传发生时，在重传数据最终到达之前，RTT估计器不能更新新的smoothed RTT估计值，因为我们不知道收到的ack对应哪个发送的数据分组。


### 四：往返时间RTT的案例
本部分内容将通过一些案例来了解：TCP超时与重传、慢启动、拥塞避免等机制的实现细节。

##### 1 在slip主机上使用sock程序发送32个1024byte数据到互联网中vangogh主机discard服务
<center><img src="/img/in-post/tcp-ip_img/tcp_21_3.pdf" width="100%"></center>
- sock命令相应应用进程执行32个"write"操作，每次写入MSS=1024byte。
- slip主机到达bsdi的SLIP链路的MTU=296byte，因此上面的写入的数据将被TCP分成128×256byte的方式进行分组(segment)发送。
- 整个传输过程持续45s，可以观察到1次超时和3次重传。

##### 2 slip主机在局域网内的连接情况
<center><img src="/img/in-post/tcp-ip_img/tcp_21_4.pdf" width="80%"></center>
- slip(左下角)需要通过两个传输速率为9600bit/s的SLIP链路，并通过Cisco路由器连接到互联网上。
- 由于经过两个传输速率较慢的SLIP链路，因此希望能借此在数据传输过程中产生一些可测量的时延(measurable delay)。

##### 3 往返时间RTT的测量
<center><img src="/img/in-post/tcp-ip_img/tcp_21_5.pdf" width="90%"></center>
- 在[chapter24 TCP时间戳选项](http://localhost:4000/tcp-ip/2021/10/24/post-tcp-ip-tcp-chapter-24/#五时间戳选项)中关于RTT时间测量、窗口大小、混叠效应对于RTT测量&估计以及超时重传的影响分析中进行补充&引用。

##### 4 往返时间RTT的估计值计算
- 上图中3个大括号用于标记那些报文segment用于一次往返时间RTT计时，并不是所有传输的数据分组都参与了计时。
- 需要注意：大多数伯克利的TCP实现中，对于一个连接，在任何情况下，只进行一次RTT时间测量。当一份数据分组被发送时，对于该给定连接的定时器已经被启动，那么该分组不会参与定时???
- **计时器(tick counter)的工作原理**：开始计时后，每过500ms时间，计时器中的计数器+1。例如：一个给定数据分组，它的确认ack信号在数据分组被发送后的550ms返回，那么计时器中的计数器为1tick(500ms)或者2tick(1000ms)，1还是2取决于计数器是否从0开始计数。
- **有效的往返时间RTT数值测量的条件**：除了为每个TCP连接记录一个tick counter，开始计时后发送的起始数据分组的序号也被记录，当包含该数据分组序号的确认ack信号返回时，定时器关闭。只要该数据分组的确认ack信号返回时，对应的数据分组没有执行超时重传(即确认ack返回的足够及时不需要超时重传)，那么smoothed RTT数值和smoothed mean deviation数值就基于这个新的RTT测量值进行更新。
- **结合上面的案例时序图进行分析#1** 上图中建立的TCP连接的timer在发送segment#1时启动，并在segment#2到达slip时中止。时序图显示这个往返时间间隔是1.061s(tcpdump输出)，但是socket debug information显示该过程经历了3个TCP时钟tick，即RTT=1500ms。
- **结合上面的案例时序图进行分析#2** 建立的TCP连接的timer在segment#6时再次被启动，并在segment#10返回slip时中止。时序图显示往返时间间隔是1.015s(tcpdump)，因此测量的RTT数值是2tick=1000ms。注意，当slip收到segment#8时，并不会更新timer，因为该确认ack不是针对timer启动时的数据分组返回的；另外，slip发送segment#7和segment#9时不能启动timer，这是因为此时timer已经被使用。

##### 5 实际往返时间RTT和时钟tick计数之间的"换算"关系
<center><img src="/img/in-post/tcp-ip_img/tcp_21_6.pdf" width="100%"></center>

##### 6 全部128个数据分组得到的18个RTT测量值以及相应RTO计算值图表
<center><img src="/img/in-post/tcp-ip_img/tcp_21_7.pdf" width="60%"></center>
- 上图中横轴时间从0开始，横轴0时刻表示传输数据分组segment#1的时刻，并不是建立TCP连接时传输第一个SYN信号的时刻。
- TCP计算出的RTO数值来自socket debug information的输出，测量的往返时间RTT数值来自tcpdump中程序记录报文传输时刻。
- 测量的RTT数值点对应了：本文前面部分的"时序图"以及"往返时间RTT和tick计数关系图"中的3个往返时间示例。
- 从上图很容易发现：在横轴时刻为10、14、21时，测量RTT的时间间隔相比其他的间隔要长，这是因为在这些时刻，TCP发生超时重传数据分组导致RTT测量间隔增加。即根据karn算法：当超时重传发生时，在重传的数据的确认ack信号到达之前，不能更新RTT估计器，即不会产生测量RTT数值、和估计RTO数值。 
- 从上图很容易发现：RTO估计数值总是500ms(1个时钟tick)的整数倍。它们之间的换算关系见前一部分往返时间RTT和时钟tick之间的换算关系图。

##### 7 RTT估计器相关计算(结合上述案例进行计算)
- **step-1** 变量R和D分别被初始化为0s和3s，根据$RTO=R+2D=0+2\times 3=6s$来计算初始的RTO，这个RTO数值被用于对TCP建立连接时发送的SYN信号的RTO数值进行估计。注意乘数2只在RTO初始化计算中使用，后续RTO估计使用$RTO=R+4D$来更新。
- **step-2** 初始SYN分组在传输过程中丢失，发送端等待确认信号超时并发生重传，下图给出tcpdump打印的TCP建立连接过程发生的超时重传前4行：<center><img src="/img/in-post/tcp-ip_img/tcp_21_8.pdf" width="100%"></center>
- **step-3** 上图中，5.8s后数据重传发生，此时计算RTO应该为$RTO=R+4D=0+4\times 3=12s$。其中smoothed RTT(R)和smoothed mean deviation(D)仍然使用初始的0和3s。因此指数退避的基础值取为12s，下一次超时时间间隔为12×2=24s，再下一个超时时间间隔为24×2=48s。
- **step-4** 第2行中重传数据分组对应的确认ack信号(第3行)返回，并不会更新R和D的数值，这是因为特定TCP的karn算法实现对于是否使用重传数据更新R和D比较模糊(ambiguous)，有时允许使用重传数据进行更新，有时不使用重传数据更新R&D。

<details>
    <summary><b>[点击展开]</b> 往返时间RTT的测量图</summary>
    <center><img src="/img/in-post/tcp-ip_img/tcp_21_5.pdf" width="90%"></center>
</details>
- **step-1** 上面的测量图中第一个数据分组的确认ack应答返回发送端后，根据本文前面绘制的RTT往返时间和时钟tick换算关系图，第一个RTT测量对应3个tick，所以$smoothed RTT(R)=1.5s(3×tick=3×500ms)$；将smoothed RTT估计值初始化范围$R+0.5s=2s$，至于为什么没有采用"低通滤波的方式初始化的原因不明"。然后将smoothed mean deviation(D)初始化为$R/2=1s$。
- **step-2** 利用step-1初始化的R&D计算RTO：$RTO=A+4D=2+4\times 1=6s$
- **step-3** 上面测量图中第二个数据分组确认ak应答返回发送端后，根据RTT往返时间和时钟tick换算关系图，第二个RTT测量对应1tick(500ms=0.5s)。因此，按照Jacobson 1988方法：首先获得测量结果和RTT估计器估计值的误差：$Err=M-R=0.5-2=-1.5s$；然后获得更新的smoothed RTT(R)数值：$R \leftarrow R + gErr=2 - 0.125 \times 1.5=1.8125$；再更新smoothed mean deviation(D)数值：$D=D+h(\mid Err\mid -D)=1+0.25\times (1.5-1)=1.125s$；最后得到超时重传时间RTO：$RTO=R+4D=1.8125+4\times 1.125=6.3125s$。
- **注意** 虽然上述计算出的超时重传时间RTO为6.3125s，但是实际RTO采用6s而不是计算值(如"18个RTT测量值及RTO估计值图表"中第二个点所示)，即实际和理论计算略有偏差。

##### 8 观察超时重传案例中传输数据的"慢启动"现象
<details>
    <summary><b>[点击展开]</b> 往返时间的测量时序图</summary>
    <center><img src="/img/in-post/tcp-ip_img/tcp_21_5.pdf" width="90%"></center>
</details>
- 在上图所示案例的数据分组传输过程可以看到慢启动机制：segment#1分组发送时，只允许传输一个报文段，在发送端收到该分组的确认ack应答后，则可以发送2个数据分组segment#3以及segment#4。
- 更多关于慢启动的内容，可以参考[前一章](http://localhost:4000/tcp-ip/2021/10/03/post-tcp-ip-tcp-chapter-20/#六慢启动slow-start)中对慢启动机制的简单介绍。


### 五：拥塞举例(congestion example & simple analysis)
##### 1 数据传输过程分组序号与该分组发送时间图表
<center><img src="/img/in-post/tcp-ip_img/tcp_21_9.pdf" width="80%"></center>
- 上图显示了slip主机向vangogh主机发送32768byte的数据。
- 由于超时重传、和快速重传两种算法都会重新发送已经发送但是未能收到确认ack的分组，因此重发会引起发送分组序号曲线出现"反折"。上图中出现了3次"反折"，因此在数据分组传输过程中，出现了3次分组丢失&执行快速重传。
- 3次快速重传分别发生在时刻10、14、21附近。

##### 2 数据传输过程从从segment#43开始到segment#71的分组交换、超时快速重传时序图
<center><img src="/img/in-post/tcp-ip_img/tcp_21_10.pdf" width="100%"></center>
- 从上述数据分组传输时序图可知：数据分组seg#45可能丢失了or在传输过程中损坏了。对于数据分组seg#45的具体情况无法通过上面时序图进行分析判断。
- 注意，在发送端发送segment#63重传数据分组后，可以直接继续剩余数据分组的传输(seg#67、seg#69、seg#71)，而无需等待接收端返回重传数据报的确认ack信号(seg#72)。
- 注意，seg#46到seg#59都是"失序"的数据分组，它们并不是接收端期望收到的6657，TCP接收端并不会为它们返回相应确认ack信号，而是暂时保存它们并返回重复的丢失序号的ack信号。
- 当丢失重传的数据分组到达接收端时，接收端TCP在缓存中保存6657~8960byte的数据，并将这2304byte数据发给应用进程。这些数据在seg#72中被统一确认。
- 注意seg#72返回的确认ack信号中，接收端通告窗口大小为5888(8192-2304)byte，这是因为应用进程在接收端返回确认ack信号时，尚未接收TCP缓存中的2304数据。

### 六：拥塞避免算法(congestion avoidance algorithm)
在chapter20中，曾经介绍了慢启动算法，它是一种用于控制TCP连接发送端发起数据流的方法。在数据的传输过程中，有时中间路由器会达到缓存的极限，对于后续到达的数据报将作丢弃处理。

拥塞避免算法是一种能够处理中间路由器丢失数据分组的算法，[Jacobson 1988]是对于该方法进行了具体的描述。

中间路由器发生分组丢失意味着在源主机和目的主机之间的某处网络中发生了拥塞，故拥塞避免算法假定由于分组收到损坏引起的丢失是非常少见的(≪1%)。针对数据分组丢失，有两种指示方式：**发生超时**和**接收到重复的确认信号**。

拥塞避免算法和慢启动算法是两个相互独立的算法，它们的目的是不同的(一个用于启动流量控制、一个用于拥塞避免)。当拥塞发生时，通常希望降低数据分组发送到网络中的速率，因此可以通过调用慢启动算法来实现。在实际应用中，这两个算法通常在一起实现。

##### 1 拥塞避免算法和慢启动算法在拥塞发生时的实现方式
- **(1)** 算法需要为每个TCP连接维护两个变量：拥塞窗口cwnd、慢启动门限ssthresh。对于给定的TCP连接，初始化cwnd=1segment，ssthresh=65535byte。
- **(2)** TCP output routine不能输出超过$min(cwnd,接收端通告窗口大小)$的数据分组。拥塞窗口是发送端流量控制，而通告窗口大小是接收端流量控制。拥塞窗口的设置来源于发送发对于网络拥堵情况的估计，通告窗口的设置和接收端在该TCP连接上的可用缓存大小相关。
- **(3)** 当拥塞发生时(timeout or reception of duplicate ack)，ssthresh被设置成$\frac{1}{2}min(cwnd,receiver's \; advertised \; window)$，并且至少为2个报文段。如果拥塞是因为timeout导致，则cwnd被设置为1报文段，此时为慢启动。
- **(4)** 在通信中，新的数据被对方确认时，增加cwnd。cwnd的增加方式依赖于当前的状态：如果cwnd≤ssthresh，则应当按照slow start的方式增加cwnd，否则按照拥塞避免的方式增加cwnd。
- **(4.1)** 慢启动算法初始设置cwnd为1个报文段，此后每收到一个确认ack就+1，这会使得cwnd成指数方式增长1、2、4、8...
- **(4.2)** 拥塞避免算法每收到一个确认ack将cwnd增加$\frac{1}{cwnd}\times cwnd$。和慢启动相比，这是一种"additive increase"，即一个往返时间内，不管在此期间收到多少分组的确认ack信号，最多为cwnd增加一个报文段。

##### 2 慢启动算法和拥塞避免算法的图形化描述
<center><img src="/img/in-post/tcp-ip_img/tcp_21_11.pdf" width="70%"></center>
- 从上图得到的直观结果可知："slow start"并不是一个很精确的描述。慢启动只是采取了一个比引起网络拥塞更慢一些的传输速率，但是需要注意，在慢启动过程中，进入网络的数据分组的被发送的速率仍然是增加的。只有发送速率(cwnd)到达ssthresh时，分组发送速率才会保持恒定。


### 七：快速重传与快速恢复算法(fast retransmit & fast recovery algorithms)
拥塞避免算法的改进版本在[Jacobson 1990]被提出。

**收到时序数据分组，立即返回duplicate ack信号** <br>
在介绍[Jacobson 1990]具体对拥塞避免算法做出那些修改时，需要注意到：当接收端收到一个失序的报文段时，TCP需要**立即**返回一个"duplicate ack"信号，这个信号用于告知分组发送端：当前收到的数据分组为一个"out-of-order"分组，并通知发送端自己希望收到的正确的序号。

**判断产生duplicate ack信号的原因** <br>
有时我们不知道duplicate ack信号是由于"丢失的报文段引起"，还是由于"报文分段的重排序(reordering)导致"的。为了判断&区分这两种导致收到重复确认信号的起因，需要等待&接收duplicate ack信号。<br>
如果duplicate ack是由于少量报文的重新排序导致的，则在重新排序的报文段被处理并产生新的ack之前，重复报文段的个数只能是1或2。如果连续收到3个或者3个以上的duplicate ack信号，则大概率在数据传输过程中发生分组丢失。<br>
如果已经判断出分组丢失，可以重传丢失的报文数据分组，而无需等待超时定时器发生溢出再执行重传，这就是所谓的快速重传算法。<br>
紧接着，执行完快速重传后，执行拥塞避免算法而不是慢启动算法，这种处理方式称为快速恢复算法。

##### 1 结合[拥塞避免]部分的时序图分析快速重传、快速恢复算法
<details>
    <summary><b>[点击展开] 拥塞避免案例数据传输时序图</b></summary>
    <center><img src="/img/in-post/tcp-ip_img/tcp_21_10.pdf" width="100%"></center>
</details>
- 从上图容易看出：接收端收到3个duplicate ack之后，并没有执行慢启动，而是判断相应序号数据分组已经丢失，执行快速重传。
- **关于duplicate ack信号** seg#45数据分组丢失，发送端继续发送其余序号的数据分组，接收端只有收到一个其他序号的新的数据分组后，才能返回丢失序号的duplicate ack报文。
- **当一个数据分组丢失后，不执行慢启动的原因** 收到duplicate ack信号，不仅仅是说明特定序号的数据分组丢失了，而且说明了在发送端、接收端两端建立的TCP连接中仍然有流动的数据，因此没有"充分的"理由执行慢启动以突然减少传输中的数据流。

##### 2 结合[拥塞避免]部分的时序图分析快速重传算法的步骤
- **step#1** 当收到第3个duplicate ack信号后(判断为发生分组丢失)，将慢启动门限(slow-start thresh, ssthresh)设置为当前拥塞窗口的一半(1\2×cwnd)。发送端重传丢失的数据分组。设置拥塞窗口为ssthresh+3×segment大小。
- **step#2** 发送端每收到一份接收端返回的duplicate ack信号(时序图右侧的9个数据分组)，拥塞窗口cwnd+=1segment(1segment=1MSS)，并发送后续的一个数据分组。
- **step#3** 当对于新的数据分组确认ack信号(seg#72)到达发送端时，设置cwnd=ssthresh=step#1中设置的拥塞窗口数值。该确认ack信号(seg#72)是在发送端执行重传的第一个RTT时间内对重传的数据分组的确认信号。进一步地，这个确认ack信号也应该是对丢失的数据分组以及收到的所有后续数据分组(图中右侧9个ack分别对应的分组)的确认。在分组发生丢失时，尽管重传数据分组并收到该数据分组的确认信号，但考虑到避免网络拥塞，减轻传输路由负担，使用拥塞避免算法设置cwnd，以限制发送端在接下来(seg#72之后的分组传输)发送分组的速率。


### 八：拥塞举例(continued) - cwnd & ssthresh的计算过程
##### 1 从建立TCP连接到发送7个数据分组的cwnd和ssthresh数值图表
<center><img src="/img/in-post/tcp-ip_img/tcp_21_12.pdf" width="70%"></center>
- 通过使用tcpdump程序中的socket debug option，可以用于观测一个TCP连接在发送每个分组时的cwnd和ssthresh数值，进而可以对于TCP从"慢启动"到"拥塞避免"两种状态的转换进行观测。
- 上图展示了本文使用的往返时间RTT案例的TCP连接中两种状态转换时的cwnd和ssthresh的变化。
- **(1)** TCP建立连接发送SYN信号时，最大分组长度MSS=256byte，则cwnd被设置为256byte，ssthresh被设置成65536byte。
- **(2)** 在接下来的分组传输中，每收到一个数据分组确认ack信号，cwnd+=1segment，即cwnd+=1MSS。假如网络不会发生拥塞，后续的cwnd取值为512(+256)、768(+2×256)、1024(+3×256)、1280(+4×256)...最后cwnd将会超过接收端通告窗口大小，将由通告窗口对于数据流量进行限制。实际tcpdump打印的cwnd和ssthresh并没有按照预想的情况增加，这是由于在分组传输中，TCP连接达到了ssthresh后，进入"拥塞避免"状态。
- 实际上，由于上图对应的TCP建立连接的SYN信号发生超时，因此ssthresh被设置为最小数值512byte(2MSS=2segment)；TCP建立连接后，进入慢启动状态，因此cwnd初始化为1segment(256byte)。
- **(3)** 收到接收端返回的SYN和ACK信号后，并不改变cwnd和ssthresh的数值。cwnd和ssthresh数值只有收到数据分组确认ack信号后，才进行数值修正。
- **(4)** 当ack257到达发送端，此时cwnd\<ssthresh，处于"慢启动"状态，因此cwnd+=1MSS=512byte。
- **(5)** 当ack513到达发送端，此时cwnd\<ssthresh，处于"慢启动"状态，因此cwnd+=1MSS=768byte。
- **(6)** 当ack769到达发送端，此时cwnd\>ssthresh，处于"拥塞避免"状态，此时cwnd数值按照如下方式进行计算(即本文前面部分所提到的"增加1/cwnd")：
$$
cwnd \leftarrow cwnd + \frac{segsize\times segsize}{cwnd} + \frac{segsize}{8}
$$
，将segsize=256byte、cwnd=768byte带入，得到ack769到达后的cwnd=885byte。
- **(7)** 当ack1025到达发送端，将segisze=256byte、cwnd=885byte带入，得到新的cwnd=991byte。

##### 2 完整数据传输过程中数据发送序号和cwnd的取值折线图
<center><img src="/img/in-post/tcp-ip_img/tcp_21_13.pdf" width="90%"></center>

##### 3 数据传输过程中发生拥塞避免部分的cwnd和ssthresh数值分析图表
<center><img src="/img/in-post/tcp-ip_img/tcp_21_14.pdf" width="70%"></center>
- 注意，segment#60和segment#61两个duplicate ack信号到达时，cwnd并不会改变(尚不能构成出发快速重传的条件)；当segment#62的duplicate ack信号到达时，达到快速重传的条件，此时ssthresh被设置为cwnd的一半(2426/2=1213≈256×4=1024)，注意ssthresh总是被设置为segment长度的整数倍；此时cwnd被设置为ssthresh+3×MSS=1024+3×256=1792byte。
- 对于接下来收到的5个报文段：segment#64、segment#65、segment#66、segment#68、segment#70，每收到一个duplicate ack信号，cwnd+=1MSS。
- 当收到新的数据的确认应答时(segment#72)，cwnd的数值被设置为ssthresh=1024byte，并进入正常拥塞避免进程：收到segment#72确认ack后，cwnd+=1MSS=1024byte+256byte=1280byte(如图表中所示)。
- segment#65到达时，cwnd=2048(尚未增加至表中的2304)，此时未确认的数据分组有：segment#46、segment#48、segment#50、segment#52、segment#54、segment#55、segment#57、segment#59、segment#63，一共2304byte(9×256)，因此在收到segment#65后，并不能发送任何数据(参考[chapter-20](http://localhost:4000/tcp-ip/2021/10/03/post-tcp-ip-tcp-chapter-20/#1-数据发送端滑动窗口协议示意图)，已发送未确认的分组+实际可用的窗口大小=cwnd)，此时未确认的分组大小\>cwnd。
- segment#65到达后，cwnd+=1MSS=2304byte，此时根据窗口关系计算易知：实际可用的窗口大小=0byte，即仍然不能进行数据发送。
- segment#66到达后，cwnd=2560byte，此时实际可用窗口大小=256byte，可以发送1个数据分组。类似的，当segment#68到达后，cwnd=2816byte，可以发送数据分组。segment#70到达后，处理情况同理。


### 九：为每条路由维护路由表项用于保存相应路由度量信息(per-route metrics)
较新的TCP实现中，在路由表项中维护了许多TCP连接相关的指标信息。

如果一个TCP连接关闭后，已经发送了**足够多的数据**，获得了许多有价值统计信息，并且路由表项entry对应的TCP连接目的地址不是路由器表中的默认表项，那么下面列举的信息将保存在路由表项中以备下次使用：smoothed RTT(平滑的往返时间)、smoothed mean deviation(平滑的均值偏差)、ssthresh(慢启动门限)。<br>
**足够多的数据**：16个窗口的数据，用来获得16个往返时间RTT采样，从而使得smoothed RTT对于理想的RTT估计误差控制在一定范围内(5%)。

网络管理员可以使用route(8)命令来设置给定路由表项的度量信息，包括：smoothed RTT(平滑的往返时间)、smoothed mean deviation(平滑的均值偏差)、ssthresh(慢启动门限)、MTU(链路最大传输单元)、outbound bandwidth-delay product(输出带宽时延乘积)、inbound bandwidth-delay product(输入带宽时延乘积)。

当建立一个新的TCP连接时，无论该连接是主动建立还是被动建立，如果将使用的路由表项已经存储了上述"per-route metrics"，则直接使用存储的度量值进行初始化。


### 十：TCP处理给定连接返回的ICMP差错信息的方式
TCP遇到的最常见的ICMP差错是源站抑制(source quench)、主机不可达差错(host unreachable)、网络不可达差错(network unreachable)。

##### 1 基于伯克利的TCP实现对于上述差错的处理方式
- TCP收到源站抑制报文，会将拥塞窗口cwnd设置为1来运行慢启动，但是不改变慢启动门限ssthresh。拥塞窗口将一直保持打开状态直到它**开放了所有通路(open all the way)(通路个数受到拥塞窗口大小和往返时间的限制)**或者**发生了拥塞**。???problem😫problem???什么叫open all the way?
- 收到的主机不可达、网络不可达信息都被忽略，因为这两种差错信息被认为是短暂的现象(极有可能会很快恢复)。例如：路由中某个中间路由器被关闭，导致数分钟后选路协议才能将TCP连接经过的路由"稳定地"替换成另一条使用代替路由的线路，这个过程可能引发上述2种ICMP不可达差错之一，但是TCP连接不需要被关闭。

##### 2 结合案例分析TCP处理ICMP差错信息的方式
本部分内容通过将TCP连接中SLIP链路断开来产生ICMP主机不可达报文，并观察该报文是如何被TCP处理的。

**(i) 建立从slip主机到aix主机的TCP连接，slip和aix之间的网络拓扑关系如下所示**
<center><img src="/img/in-post/tcp-ip_img/tcp_21_15.pdf" width="80%"></center>

**(ii) 在slip主机上启动sock程序连接到aix主机的"回显服务程序"，并通过挂断SLIP链路的方式测试TCP处理ICMP不可达报文的方式**
<center><img src="/img/in-post/tcp-ip_img/tcp_21_16.pdf" width="100%"></center>

**(iii) 使用tcpdump程序打印上述sock程序案例对应的数据分组交换信息**
<center><img src="/img/in-post/tcp-ip_img/tcp_21_17.pdf" width="100%"></center>
- 从上图容易发现：每次发送端slip主机向aix主机发送数据分组，由于SLIP链路断开，中间路由sun主机每次都会返回给发送端"host unreachable"。
- sock程序并不会对应每次超时重传都会打印"连接超时"信息，而是暂时保存收到的"host unreachable"差错，并只在最后放弃重传时才打印"No route to host"信息。这是因为：ICMP主机不可达、ICMP网络不可达信息通常恢被忽略，TCP将它们视作是短暂的现象，认为连接很快就会恢复(这种机制能够避免由于中间路由断开导致的ICMP报文"风暴"导致网络拥塞)。
- 最后，注意tcpdump打印输出中，第6～14行和第22～46行之间重传数据的间隔不同。在发送'line number 3'(tcpdump输出第17～19行)时，TCP更新了它的往返时间估计器(estimators)：初始重传时间为3s，后续取值依次为6s、12s、24s、48s直至上限64s。

### 十一：重分组(Repacketization)
Repacketization：当TCP发生超时并执行重传时，不一定要重传和发送超时的数据分组一样的数据报。

TCP允许对于超时的多个数据分组进行重新分组，从而可以一次性发送较大的报文段(不能超过TCP连接的MSS数值)。

Repacketization在一定程度上可以提高该TCP连接的性能。TCP可以使用字节序号代替数据分组序号对于它发送的数据和相应确认ack进行划分。

##### 1 使用sock程序对于TCP连接的重分组机制进行实现 & 分析
<center><img src="/img/in-post/tcp-ip_img/tcp_21_18.pdf" width="100%"></center>

##### 2 使用tcpdump程序打印上述sock程序对应的数据分组交换信息
<center><img src="/img/in-post/tcp-ip_img/tcp_21_19.pdf" width="100%"></center>

## Reference
> \<tcp-ip: illustrated vol1\> chapter21 <br>

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
