---
layout: post
title: "tcp-ip: dynamic routing protocols <br> (动态选路协议)"
subtitle: '[tcp-ip: illustrated vol1] - chapter10' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-08-24 15:14
lang: ch 
catalog: true 
categories: tcp-ip 
tags:
  - Time 2021
  - tcp-ip illustrated vol1 
  - problem
---
### 一：引言
在chapter9中，主要讨论了静态选路：**i** 在配置接口时，以默认方式为直接连接的接口生成路由表项；**ii** 并且可以通过"route add"命令增加表项，route命令通常位于系统自引导(bootstrap file)程序文件中；**iii** 还可以通过ICMP重定向来生成路由表项，通常在默认方式配置出错情况下应用。

在网络很小，且与其他网络只有单一路由连接点(避免形成环)，并且没有备用路由(当主路由失败，可启用备用路由)时，采用静态的选路方式是可行的，如果上述3个条件不能全部满足，则通常采用动态选路。

本文主要讨论用于路由器之间通信的动态路由协议：**RIP**(routing information protocol)选路信息协议，大多数TCP/IP实现都支持此协议；接着研究**OSPF**(open shortest path first)以及**BGP**(border gateway protocol)；最后研究**无分类域间选路协议**(classes interdomain routing)，现在互联网广泛使用这项技术以保持网络中B类网络的数量。

### 二：动态选路
相邻路由器之间进行通信，以告知对方自己的网络连接情况，成为动态选路。路由器上一般运行成为路由守护程序(routing daemon)的进程，他负责：运行选路协议、与相邻路由器进行通信(如[chapter9-1](http://localhost:4000/tcp-ip/2021/08/18/post-tcp-ip-ip-routing/#一简介)所示)，并根据收到的信息更新内核路由表。 

**选路策略** <br>
路由守护程序在动态选路的过程中应用了选路策略(routing policy)：如果routing daemon发现前往同一信宿有多条路由，它会选择最佳路由加入路由表中；如果routing daemon发现线路已经断开or信号不好，则它可能删除该路由并新增一条alternative route予以代替。

**自治系统AS** <br>
Internet是由很多个"自治系统(Autonomous System)"组成的(如一个公司网络、一个大学网络可以看成一个AS系统)，每个AS由单个实体进行管理。每个AS可以选择内部各个路由器之间的选路通信协议，称之为**IGP(interior gateway protocol)**或者**IRP(intradomain routing protocol)**。

**内部、外部选路协议** <br>
选路信息协议RIP是最常见的IGP，OSPF是较新的一种协议，旨在代替RIP。RFC-[Almquist 1993]规定：实现任何动态选路协议的路由器必须同时支持OSPF和RIP，也可以支持其他的IGP。<br>
外部网关协议EGP(exterior gateway protocol)以及域间选路协议IRP用于不同AS之间的路由器。一个较新的EGP协议成为BGP(border gateway protocol)被用于多个NSFNET网络的backbone之间进行通信，或者NSFNET网络backbone和其他附属regional network进行通信。BGP旨在取代EGP。

### 三：unix选路守护程序
unix系统上通常会运行名为routed的路由守护程序，几乎所以的TCP/IP实现都支持该程序。routed只使用RIP协议进行通信，<br>
unix系统上也会运行另一种名为gated程序，它支持IGP、EGP两种协议进行通信。<br>
关于routed、gated两个程序与相关协议的不同版本支持情况可参考下表：
<center><img src="/img/in-post/tcp-ip_img/dynamic_routing_1.pdf" width="80%"></center>

### 四：选路信息协议RIP(routing information protocol) - 版本1
RIP是一种最广为应用也同时也是诟病较多的选路协议。对于RIP协议的正式描述文件可以参考RFC-1058。

##### 1 封装在udp数据报中的RIP报文格式
**[1] RIP报文在udp数据报中的封装格式**
<center><img src="/img/in-post/tcp-ip_img/dynamic_routing_2.pdf" width="50%"></center>

**[2] 使用IP地址时的的RIP报文格式(不包含udp报文头部)**
<center><img src="/img/in-post/tcp-ip_img/dynamic_routing_3.pdf" width="100%"></center>

##### 2 采用RIP协议的routed程序的基本操作(routing operations)
**(i) 初始化 =>** 启动路由守护程序时，它先判断启动了哪些接口(接口处于"UP"状态)，并在每个接口上发送一封请求报文，要求其他路由器发送完整路由表。对于一个点对点的连接，请求报文直接发送给连接的另一端；对于支持广播的网络，请求以广播形式发送。其他路由器上的路由守护程序udp端口号为520。该请求报文的命令字段设置为1，地址系列字段设置为0，度量字段设置为16；这是一种要求另一端提供完整路由表的请求报文。<br>
**(ii) 路由守护程序收到请求 =>** 如果收到(i)中的完整路由表请求，则路由器将完整的路由表发送给请求者。否则将处理请求表中的每一个表项：如果本机路由表中包含到达请求中指定的IP地址的路由表项，则将度量值(见上面RIP协议格式图)设置为本机路由度量值，否则设置为16(16=infinity:表明当前路由器没有到达指定IP地址的路由)。<br>
**(iii) 路由守护程序收到响应 =>** 本机根据收到的响应对本机路由表进行调整(响应生效)。可能是增加新的表项、可能对已有的表项进行修改、可能将已有表项删除。<br>
**(iv) 定期选路更新 =>** 每过30s，部分或者全部的路由器会将本机完整的路由表发送给每一个相邻的路由器。更新信息的发送方式可能是单播形式的(e.g. 点对点线路)，也可以是广播形式的(e.g. 以太网)。<br>
**(v) 触发单个路由表项更新 =>** 当路由表中特定的表项中的度量(metric)改变后，就会触发路由表更新。这种更新不会发送整个路由表，只是将表项中变化的部分发送出去。<br>
**(vi) 每个路由表项都有相关联的定时器 =>** 如果运行RIP的系统发现一条路由表项在3min内没有被更新，就把该表项的度量设置为16，并标记该表项供删除程序识别(3min意味着本应收到6次关于该项目的更新数据报，但是一封都没收到)。再经过60s后，将从本机路由表中删除该表项，以确保该路由表项失效的消息已经完全通知到直接相连的路由器。<br> 

##### 3 RIP协议中的度量(metric)
RIP协议报文中包含的度量信息是按照"跳数(hops)"进行计算的。<br>
度量的计算方式按照如下几条规则进行实施：**i** 所有直接相连的接口(网络接口、路由器接口)之间的度量数为1；**ii** 对于特定自治系统AS，如果从其中一个路由器到达某个网络有多条路由线路，那么路由器将选择跳数最少的路由，而忽略其他的路由。**iii** 路由的跳数最大值为15，表明RIP协议报文只能用于主机间最大跳数为15的AS内(度量=16表示路由器无到达该地址)。

一个简单的"路由器<->网络"度量广播通知示例：
<center><img src="/img/in-post/tcp-ip_img/dynamic_routing_4.pdf" width="80%"></center>

##### 4 RIP选路信息协议的问题&缺陷(pitfalls) --> (todo:有待进一步整理)
> As simple as this sounds, there are pitfalls. First, RIP has no knowledge of subnet addressing. If the normal 16-bit host ID of a class B address is nonzero, for example, RIP can't tell if the nonzero portion is a subnet ID or if the IP address is a complete host address. Some implementations use the subnet mask of the interface through which the RIP information arrived, which isn't always correct.<br>
> **(i)** RIP选路信息协议的更新不包含子网掩码信息，并且RIP没有子网地址的概念，因此RIP协议本身不能分辨出IP地址中的子网部分、主机部分，从而不能确定IP地址是否一个完整的主机地址???why???。一些RIP的实现使用了RIP信息到达接口上设置的子网掩码信息，但是并不总是正确的???why???

> Next, RIP takes a long time to stabilize after the failure of a router or a link. The time is usually measured in minutes. During this settling time routing loops can occur. There are many subtle details in the implementation of RIP that must be followed to help prevent routing loops and to speed convergence. RFC 1058 [Hedrick 1988a] contains many details on how RIP should be implemented. <br>
**(ii)** 应用了RIP协议的网络中的路由器or链路发生故障后，网络可能需要花费分钟记的时间才能稳定下来。这个特性也可以称之为"RIP的好消息传的快、坏消息传的慢"，当发生问题后，需要等待至少3min才能将相关路由失效信息传递出去。

> **(iii)** 最大网络度量=15限制了采用RIP路由信息协议的网络的大小。 

##### 5 使用sun主机ripquery程序来查询netb路由器中的路由表(案例#1)
**(i)** 截取chapter1中通用主机网络结构模型，以提供本案例实验基础如下：
<center><img src="/img/in-post/tcp-ip_img/dynamic_routing_5.pdf" width="80%"></center>

**(ii)** 在sun主机上应用ripquery程序查询netb主机上的路由表，打印程序输出结果如下图。<br>
其中配置netb使其"认为"所有以太网#2中的主机都直接连接到以太网#2上，netb不知道哪些主机真正地和以太网#2直接相连(netb不知道以太网#2中主机的实际连接情况)。由于以太网#2(140.252.13)只有一个连接点，即所有从netb主机到达以太网#2上主机都需要经过唯一的连接点，因此衡量每个以太网#2主机与netb的度量没有意义 -- 不涉及选路，不需要使用RIP协议根据度量进行选路。
<center><img src="/img/in-post/tcp-ip_img/dynamic_routing_6.pdf" width="100%"></center>

**(iii)** 使用tcpdump命令打印数据报交换信息(packet exchange info)如下图。tcpdump命令显示netb返回的第一封RIP应答为满载状态，根据前面的计算可知：载有25项[IP地址,度量]信息对，占用504byte。第二封RIP应答非满载，载有12项[IP地址,度量]信息对，占用12×20+4=244byte(印证了ripquery命令的输出)。
<center><img src="/img/in-post/tcp-ip_img/dynamic_routing_7.pdf" width="100%"></center>

##### 6 使用sun主机ripquery程序来查询gateway路由器中的路由表(案例#2)
**(i)** 在sun主机上应用ripquery程序查询gateway路由器上的路由表，打印程序输出结果如下图。<br>
由于从sun主机到gateway路由器需要经过netb路由器，因此RIP应答返回的路由表信息中到达gateway的度量为2。
<center><img src="/img/in-post/tcp-ip_img/dynamic_routing_8.pdf" width="100%"></center>

##### 7 另一个例子(chapter10.4.6 - 待整理)

### 五：选路信息协议RIP - 版本2
RFC-1388对于RIP协议的定义进行了扩充，称之为RIP-2。RIP-2并不改变原有的RIP协议的部分，而是利用RIP报文格式中"必须为0"的字段来传递额外信息。另外，如果RIP实现程序忽略这些"必须为0"字段，则RIP数据报和RIP-2数据报可以在该程序中通用。

##### RIP-2的报文格式 
<center><img src="/img/in-post/tcp-ip_img/dynamic_routing_9.pdf" width="100%"></center>
**补充说明** <br>
**(i)** 下一跳IP地址指明了发往32bitIP地址的数据报的下一站应该发往哪个直接相连的路由器。如果下一跳IP地址为"0.0.0.0"，那么发送RIP-2应答数据报的通知路由器就是最佳的路由下一跳地址；如果下一跳IP地址不为"0.0.0.0"，则下一跳地址指明了一个比发送RIP-2应答数据包的通知路由器更好的下一跳地址。<br>
**(ii)** ???problem😫problem??? RIP-2报文提供了一种简单的"authentication scheme"：指定地址系列字段(address family)为0xfff，路由标记字段(route tag)为2，其余16byte存放明文密钥(cleartext password)。<br>
**(iii)** RIP-2提供除了RIP-1中的广播通知机制外，还提供了多播的方式通知相连路由更新它们的路由表项。可以减少无需监听通知路由的机器数量，从而降低路由更新负载。

### OSPF(open shortest path first) - 开放最短路径优先
OSPF是另一种内部网关协议，它克服了RIP的所有限制。RFC-1247对OSPF(version2)进行了详细的描述。<br> 不同于采用距离向量的RIP协议，OSPF是一个链路状态协议。不同于许多路由选路协议，OSPF直接使用IP数据报，而不需要使用TCP或者UDP(RIP-v1/v2需要使用udp数据报)。<br>

##### 1 基于距离向量的协议&基于链路状态的协议
**距离向量**：RIP报文包含路由相关的距离度量信息，每个路由器根据它所接收到RIP数据报中的距离度量信息更新自己的路由表。<br>
**链路状态**：不需要和相邻的路由器交换距离信息，而是每个路由器主动测试与其直接相连的路由器的"link status"，并将此信息转发给与其相连的其他路由器，在自治系统中，每个路由器交换这些"link status"信息，并建立完整的路由表。<br>
**两种方式比较** 从实际应用角度，使用链路状态的协议相比距离度量协议构建路由时收敛更快。收敛的意思是：当自治系统的路由发生变动时(路由器关闭or出现故障)，基于链路状态协议构建的路由表可以更快的稳定下来，而不是一直进行频繁调整。

##### 2 OSPF(链路状态)相比RIP(距离向量)的优势
**(i)** OSPF可以针对每个IP服务类型分别提供不同的路由集合。这意味着：对于任何目的地址的数据报，OSPF机制可以设置多个路由表项，分别用于决定不同IP服务类型的路由去向。<br>
**(ii)** OSPF允许为路由器每个接口设置一个纬度无关的损耗度量(dimensionless cost)。损耗度量可以由吞吐率、RTT往返时间、可靠性以及其他性能参数进行指定(计算)。同样，对于每个接口可以根据不同服务类型设置不同损耗度量。<br>
**(iii)** 当OSPF生成的路由表项中针对同一目的地址有多个相同损耗度量的路由表项时，OSPF在这些路由间平分流量，称之为"流量均衡"。<br>
**(iv)** OSPF支持子网。每一份通知OSPF路由报文有单独的subnet mask与之相互关联(associated)，这允许单一IP地址结合不同的子网掩码后，可以用于不同大小的多个子网中(变长子网掩码)。其中目的地址为主机的路由项是通过"主机IP+全1子网掩码"进行通知的，目的地址为默认路由项(default)是通过"IP地址为0.0.0.0+全0网络掩码"进行通知的。<br>
**(v)** 路由器之间的点对点线路不需要为两端分别配置IP地址(OSPF允许应用于无编号网络)，这样可以节约宝贵的IPv4资源。<br>
**(vi)** ???problem😫problem??? 可以使用简单"authentication scheme"，类似于RIP-2中的方式指定明文密钥。<br>
**(vii)** OSPF采用了多播方式发送IP数据报，相比广播的方式减少不参与OSPF机制的网络部分的负载。

大部分厂商都支持OSPF，很多网络中OSPF将逐渐代替RIP系列协议。

### BGP(border gateway protocol) - 边际网关协议
前面提及的RIP-v1/v2、OSPF都是用于自治系统**内部**路由器之间的通信协议，BGP是一种应用于不同自治系统间的路由器的外部网关通信协议。BGP是ARPANET中使用早期EGP协议的替代品。<br>
RFC-1267对于第三版BGP的细节进行了描述。RFC-1268描述了如何在internet中使用BGP。RFC-1467描述了第四版BGP，此版本支持CIDR。

##### 1 构建网际必经路经连通图
BGP系统与其他BGP系统交换网络可达性信息。这些信息包括流量到达这些网络必须经过的自治系统的全部路径(信息包含BGP系统之间进行通信的必经之路)。这些信息完全能够构建一个AS连通性图。然后，可以从该图中剪除路由环路(简化)，最后制订选路策略。

##### 2 本地流量(local traffic)与通过流量(transit traffic)
一个自治系统中的数据报传输流量可以分成本地流量和通过流量，本地流量是指：信源IP地址**或者**信宿IP地址在当前自治系统中，除本地流量之外的都称为通过流量。<br>
在internet使用BGP协议的目的是减少通过流量。

##### 3 自治系统的分类 
自治系统可以分成如下几种类型：<br>
**残桩自治系统(stub AS)**：该自治系统和其他自治系统只有单一连接，因此该系统只有本地流量(该系统必然包含信源or信宿)。<br>
**多接口自治系统(multihomed AS)** 该自治系统和其他多个自治系统相连，但是拒绝传送通过流量(不做传话人)。<br>
**转送自治系统(transit AS)** 该自治系统和其他多个自治系统相连，在特定准则限制下，可以传送本地流量、通过流量。<br>

给定上述三种自治系统的定义，internet的总的拓扑网络结构可以看成是由"stub AS" + "multihomed AS" + "transit AS"共同连接而形成的。其中"stub AS"和"multihomed AS"不需要使用BGP(大材小用)，它们通过EGP在自治系统之间传递信息。

##### 4 BGP协议的选路策略
BGP允许自定选路策略，自定策略由AS管理员来制定，并通过写配置文件将策略传递给BGP服务程序。选路策略本身不属于BGP协议的内容，但是策略可以允许BGP实现在多条可选路由路径上按照策略进行选路；策略可以控制信息的重发机制；策略同时还受到政治、安全、经济的影响。

##### 5 BGP协议的部分实现细节
**BGP数据报依赖的传输协议** <br> 不同于借助udp数据报的RIP，以及直接利用IP层的OSPF，BGP使用TCP作为其传输协议：两个应用BGP协议的系统首先建立TCP连接，然后再交换整个路由表，并在路由信息变化时，通过TCP发送更新信息。<br>

##### 6 BGP属于距离向量协议 - ???problem😫problem???
RIP属于距离向量协议：它只选择路由器跳数最少的路由线路对应的"目标地址-下一跳路由"。BGP不同于RIP协议，针对每一个目的地址，它都会列举(enumerate)所有的路由线路：使用16bit数字标识自治系统，并且用自治系统的标识记录路由线路。通过这种方式可以避免单纯依赖最短距离度量带来的问题。

##### 7 BGP的链路通信检测
BGP定期发送keepalive报文给相邻路由，以检测链路是否可用、主机是否失效，检测报文发送时间间隔建议值为30s。TCP层的keepalive报文和应用层keepalive报文属于不同层次的报文(chapter23)。

### CIDR(classless interdomain routing)
**背景** <br> 随着回联网B类IP地址逐渐匮乏，开始使用网络号更多，但是主机号更少的C类网络IP地址(关于地址的分类&意义详见[introduction]())。

**引发的问题** <br> 原本一个B类网络能够覆盖的主机系统需要使用很多个C类网络进行弥补，而在整个系统中，路由器存储的路由表中原始的一项B类IP将被许多C类表项代替。考虑到路由表项之间通信，增加路由表项带来的复杂度提升成几何增长。

**解决方案** <br> 
无类型域间选路(CIDR)是一种能够防止互联网膨胀的方法，它也被称为超网(supernetting)。

**CIDR的基本思想** <br> CIDR试图通过按照一定规则分配IP地址，从而能够使得路由器中路由表项可以被整合提炼(summarization)，以减少路由器维护表项。<br>
例如，对于单个站点分配了16个IP地址，那么这16个IP地址可以整合以对应路由表中单个表项；进一步地，对于同一网络服务提供商提供的8个站点，且8个站点都用同一个连接点接入internet，那么这8个站点在internet上可使用单一路由表项。<br>

**使用路由summarization功能的前提条件** <br>
**i** 多个IP地址整合到单一路由表项的前提是：这些IP地址具有相同的高位地址bit。**ii** 路由器中的路由表及其选路算法必须扩展成根据"32bitIP地址+32bit子网掩码"进行选路决策(IP公用，每个接口分配不同子网掩码)。**iii** 用于进行路由表项内容更新&维护的选路协议也必须支持"32bitIP地址+32bit子网掩码"，RIP-v2和OSPF支持BGP-v4所使用的子网掩码。

**CIDR总是采用最长的掩码匹配** <br>
CIDR使用可变长子网掩码(variable length subnet market, VLSM)技术，根据子网主机配置需要对于给定的网络IP为每个主机分配地址。VLSM技术可以在给定IP地址的任意位置划分网络号/主机号，而不是遵循互联网通用规则进行划分。<br>
考虑路由表中两项目的地址分别为："192.168.20.16/28, 192.168.0.0/16"，当在路由表中查询"192.168.20.19"的下一跳去向时，发现由于采用了CIDR/VLSM技术，导致两项都和待查IP相匹配，那么CIDR最终将选择最长的匹配将相应数据报进行转发(子网掩码28比16要长)。


## Reference
> \<tcp-ip: illustrated vol1\> chapter10 <br>

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内 <br>
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
