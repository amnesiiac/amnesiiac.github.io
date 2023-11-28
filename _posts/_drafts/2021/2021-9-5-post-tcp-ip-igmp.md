---
layout: post
title: "tcp-ip: IGMP (internet组管理协议)"
subtitle: '[tcp-ip: illustrated vol1] - chapter13' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-09-05 10:26
lang: ch 
catalog: true 
categories: tcp-ip 
tags:
  - Time 2021
  - tcp-ip illustrated vol1 
---
### 一：引言
chapter12介绍了IP多播的基本概念，并简要说明了单个物理网络内部的多播过程。当多播涉及多个网络且多播需要经过路由器转发时，情况会复杂很多。<br>
本章介绍支持主机or路由器进行多播的internet组管理协议(internet group management protocol, IGMP)。IGMP允许网络上所有系统知道当前主机所在多播组。路由器知道关于网络上的主机所在的多播组信息以确定多播数据报应该向哪个接口转发。
IGMP在RFC1112中被定义。

### 二：IGMP报文的格式
IGMP报文数据格式如下图所示。和ICMP一样，IGMP也属于IP层的一部分，IGMP数据报通过IP数据报进行传播。但是同之前提到的其他协议不同，IGMP报文有固定的长度(8byte)，它没有可选数据。
<center><img src="/img/in-post/tcp-ip_img/igmp_1.pdf" width="80%"></center>

上图中IGMP版本号为1，到目前为止IGMP共有3个版本(v1,v2,v3)，每个版本的实现方式不同(各个版本之间的演化有待整理)。<br>
对于IGMP-v1而言，当IGMP4bit类型值为1时，表明当前IP数据报是支持多播的路由器发出的查询报文；IGMP4bit类型值为2时，表明当前IP数据报是主机发送的IGMP-report报文。校验和的计算方式同ICMP相同。在发送查询报文时，将32bit组地址设置为0，当返回IGMP-report报文时，将32bit组地址设置为要加入的组地址。 

### 三：IGMP协议
##### 1 加入一个多播组
加入多播组这一动作的概念可以解释为：操作系统中运行一种进程，进程在一个主机某个接口上加入多播组。因此，在一个主机上的某个接口的多播组是动态的，它随着进程的加入、离开多播组而变动。这种进程需要具备"加入多播组"、"离开已经加入的多播组"两种操作。这是支持多播的主机中任何API都必须支持的操作。<br>
多播组中的成员是与接口相关联的，一个进程可以通过不同接口加入相同的多播组；即相同进程通过不同接口加入多播组后，属于多播组不同的成员。<br>
一个主机通过多播组地址、接口来识别特定多播组。主机应当保留所有"有主机进程加入"的多播组，即使该多播组只有一个主机进程加入。主机应当保存每个多播组上的主机进程个数。

##### 2 IGMP的查询&报告(report&queries)
多播路由器使用ICMP报文来跟踪每个与该路由器相连的网络上的多播组成员的变化情况。

**[1] 多播路由器的IGMP报文使用规则**<br>
**(i)** 当该主机的所有进程第一次加入一个多播组，主机会发送一封IGMP-report报文。后续该主机其他进程加入同一多播组不会发送IGMP-report报文。无论主机有多少进程加入某一多播组，只会发送一封IGMP-report报文。IGMP-report报文在主机进程加入多播组所使用接口上被发送。<br>
**(ii)** 主机上的进程离开一个多播组，主机不会主动发送IGMP-report。即使该主机最后一个进程离开多播组，主机也不会主动发送报文。当主机获悉某个多播组中不再包含该主机的进程，那么此后收到的IGMP-queries报文后返回的IGMP-report报文就不会包含该多播组。<br>
**(iii)** 多播路由器定时发送IGMP-queries报文来查询任何直接相连的主机上有加入到任意多播组的进程(多播路由器想要获悉周围相连主机的各种进程加入的多播组情况)。多播路由器从每个接口都发送一封IGMP-queries报文。IGMP-queries的多播组地址设置为0，因为路由器希望主机对它加入的每个多播组都返回一封IGMP-report报文。<br>
**(iv)** 主机通过相应IGMP-report报文来响应多播路由器发来的IGMP-queries报文。主机对于加入的每一个多播组均返回一份IGMP-report报告。<br>

**[2] 路由器转发多播数据报的机制简介** <br>
通过IGMP-queries报文和IGMP-report报文，多播路由器对每个接口都维护一个关联的多播组表，表中记录活跃的多播组(表中至少包含一个主机)。<br>
当路由器转发多播数据报时，查询所有接口上活跃的多播组表，根据多播数据报相关的多播链路层地址进行匹配，并转发多播数据报到匹配成功的表项对应的所有接口上。

**[3] 主机和多播路由器通过IGMP-report&IGMP-queries报文进行通信的简单模型**
<center><img src="/img/in-post/tcp-ip_img/igmp_2.pdf" width="90%"></center>

##### 3 改善IGMP协议效率的实现细节
**[1] IGMP报文具有间隔重发机制** <br>
当主机上的第一个进程加入多播组时(首次发送IGMP-report报文)并不能保证该报文被可靠的传送，这是因为IGMP采用IP数据报作为发送的载体(IP数据报不保证数据报传递成功性、不保证数据报的按序传递、不保证数据报的传输完整性)。为了让IGMP-report报文更可靠，间隔一定时间(0-10s范围内随机选择)再次发送一次IGMP-report报文。<br>
**[2] 主机在收到IGMP-queries报文，经过随机时间间隔后才做出响应** <br>
主机返回的响应报文一般不止1封，主机对于它所在的每一个多播组都发送一个IGMP-report报文。<br>
加入多播组的每个主机都采用随机时延发送IGMP-report报文；根据主机-多播路由器通信简单模型，IGMP-report报文的目的IP为所加入的组地址，因此多播组中的某主机将收到来自同一多播组中其他主机的IGMP-report报文。<br>
上述多播IGMP-report报文通知机制表明：如果一个等待特定随机时延准备发送IGMP-report报告的主机收到同组其他主机发送的相同报文，那么该主机的IGMP-report报文可以不必发送了；因为多播路由器不关心某一接口关联的多播组有多少个主机，只关心该多播组是否至少拥有一台主机。<br>
**[3] 发送IGMP-report采取随机时延是为了避免网络拥塞** <br>
**[4] 补充内容** <br>
在没有多播路由器的单个物理网络内部，仅有的IGMP通信流量是主机加入新的多播组时，所发送的IGMP-report所占用的流量。

##### 4 生存时间字段(TTL)
IGMP-report和IGMP-queries报文中的TTL(time to live)生存时间字段均设置为1，该字段位于IP数据报的首部。
TTL=0的多播数据报只能在本地主机上转送；TTL=1的多播数据报局限在同一子网中转送；TTL>1的多播数据报能够在不同子网间被多播路由器转送。<br>
一个发往多播地址的数据报从不会产生ICMP差错：当TTL=0时，多播路由器不会返回"ICMP超时报文"。

通常情况下，发送多播数据报的用户进程不关心发出多播数据报的TTL数值。但是，chapter8介绍的traceroute程序通过设置不断增加的TTL数值来探测&获取路由信息。<br>
通过逐渐增加发送的多播路由器数据报TTL数值，应用程序可实现对于网络上特定服务器的扩展环搜索(expanding ring search)：第一份多播数据报以TTL=1发送，没有收到响应则设置TTL=2发送多播数据报，以此类推，最终可以找到以跳数度量的最近的服务器。

特殊地址范围224.0.0.0 ~ 224.0.0.255是专门用于多播范围不超过1hop的应用程序。如果发送多播数据报时目的地址位于该区间，则不管IP数据报首部TTL数值设置为多少，多播路由器均不会对该数据报进行转发。

##### 5 "全部主机"多播组(all-hosts group) - 特殊的多播组
之前的主机-路由器通信简单模型图中，多播路由器返回的IGMP-queries数据报的目的IP地址为224.0.0.1，该地址称为"全部主机"多播组地址。<br>
"全部主机"指的是网络上所有支持多播的主机or路由器。每台主机的每个支持多播的接口自动加入此多播组。<br>
"all-hosts group"中的主机接口成员从不会发送IGMP-report报文(只要支持多播它就属于该多播组)。

### 四：一个例子
本文前面的部分主要介绍了关于多播理论相关的知识，这部分主要通过程序进行一些验证。首先配置sun主机使其支持多播，并通过"netstat"、"tcpdump"程序显示sun主机上和多播相关的信息。

##### 1 netstat命令打印sun主机上所有支持多播的接口的信息
使用经过修改的netstat命令以打印sun主机上所有支持多播的接口相关信息如下图所示。
<center><img src="/img/in-post/tcp-ip_img/igmp_3.pdf" width="100%"></center>

上面打印的接口信息显示：sun主机上共有3个接口：le0(以太网接口)、s10(SLIP链路接口)、lo0(环回接口)支持多播，并且它们三个都属于"所有主机"多播组(它们的地址均包含224.0.0.1特殊地址)。<br>
打印信息的第4行输出为"01:00:5e:00:00:01"，参考chapter12 广播&多播的内容易知：改地址为接口多播IP地址(224.0.0.1)映射而成的以太网多播地址。

##### 2 netstat命令打印sun主机上路由表信息
使用\<r\>选项来打印sun主机的路由表如下图。
<center><img src="/img/in-post/tcp-ip_img/igmp_4.pdf" width="100%"></center>

路由表还可以看到目的多播地址的下一跳网关、状态标志位、正在使用该路由的活动进程计数、通过该路由发送的数据报分组总数以及该路由相关的接口。

##### 3 使用测试程序在特定接口上加入一个多播组 & netstat命令打印相关信息
测试程序的内容不是重点，本部分重点关注特定接口上加入一个多播组后(以太网接口140.252.13.33加入多播组224.1.2.3)，接口相关信息的变化。<br>
接口加入新的多播组后，使用netstat命令打印输出如下图。
<center><img src="/img/in-post/tcp-ip_img/igmp_5.pdf" width="100%"></center>

上面打印的接口信息显示：sun主机上的以太网接口le0已经新加入了测试程序指定的224.1.2.3多播组；并且224.1.2.3多播组IP地址被映射成以太网多播地址01:00:5e:01:02:03。<br>
需要注意的是：只有以太网接口le0加入指定多播组，而其他接口s10和lo0不受影响；即加入多播组是一个和接口相关联的概念。

##### 4 使用tcpdump抓包程序对进程加入该多播组进行观测
使用tcpdump命令截获IGMP数据报并打印相关信息如下图。
<center><img src="/img/in-post/tcp-ip_img/igmp_6.pdf" width="100%"></center>

**观察上图可以得到如下几点信息** <br>
**(i)** 当主机第一次加入多播组地址224.1.2.3时，发送IGMP-report信息，如上图第一行所示。<br>
**(ii)** 分析上图中第二行易知：sun主机再次发送IGMP-report报文给多播组地址224.1.2.3。第二次发送IGMP-report相比第一次经过了6.94s时延，根据之前内容的描述，这个时延不是固定的，而是0-10s之间的随机数值。<br>
**(iii)** 图中两次发送IGMP-report报文的ttl=1被"[]"括起来，其原因是："[]"用于标识特殊值，ttl字段在正常情况下都大于1，因此对于ttl=0/1，tcpdump程序将其特殊标记。<br>
**(iv)** 多播路由器必须接受它的所有接口上的任何多播数据报，路由器无法确定主机加入哪一个多播组。

##### 5 在多播选路路由器上观察IGMP-report & IGMP-queries
在sun主机上启动一个多播选路守护程序，以用来观测IGMP-report和IGMP-queries报文的交换(本部分内容不研究多播选路协议)。sun主机运行多播选路程序，相当于sun主机在"模拟"路由器在收到IGMP-report和IGMP-queries报文时的行为。

下图展示了sun主机上多播选路程序运行时的tcpdump抓包输出结果。需要注意的是：在启动选路守护程序之前，sun主机已经加入了一个额外的地址为224.9.9.9的多播组。
<center><img src="/img/in-post/tcp-ip_img/igmp_7.pdf" width="100%"></center>

**关于上述tcpdump命令打印伪造的"多播路由器"输出的几点注意事项** <br>
**(i)** 注意上述路由守护程序启动的时间位于第一行和第二行tcpdump命令输出之间。<br>
**(ii)** 224.0.0.4是一个特殊的多播组地址，它被用于多播选路协议：DVMRP(distance vector multicast routing protocol)；只要sun主机的多播路由程序启动，sun主机就加入该多播组，直到sun退出路由守护程序才退出该组(implicitly join)。<br>
**(iii)** "explicitly join"表示sun主机显示加入的分组，"implicitly join"表示sun主机在启动路由守护程序时即刻加入的多播分组。<br>

### 小结
在许多应用中，它比广播更好，因为多播降低了不参与通信的主机的负担。<br>
本章主要介绍的技术用于：在一个局域网中或跨越邻近局域网的多播。广播通常局限在单个局域网中，对目前许多使用广播的应用来说，可采用多播来替代广播。<br>
然而，多播尚未解决的一个问题是广域网内的多播。<br>


## Reference
> \<tcp-ip: illustrated vol1\> chapter13 <br>

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
