---
layout: post
title: "tcp-ip: traceroute (TCP/IP协议的高级声纳)"
subtitle: '[tcp-ip: illustrated vol1] - chapter8' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-08-15 14:52
lang: ch 
catalog: true 
categories: tcp-ip 
tags:
  - Time 2021
  - tcp-ip illustrated vol1 
---
### 引言
traceroute程序是一个能够较为深入的探索TCP/IP协议的工具。traceroute程序可以让我们看到IP数据报从一台主机发送到另一台主机所经过的路由。tracerout程序甚至可以让我们使用源路由选项以记录IP路径中经过的路由。

**为什么不使用chapter7中介绍的IP-RR选项而选择开发traceroute应用程序?** <br>
1) 早期并不是所有路由器都支持记录路由选项，因此IP-RR选项在某些路径上不能使用。所开发的traceroute程序不需要中间路由器具备任何特殊or可选的功能。<br>
2) 记录路由一般是单向选项，发送端设置了IP-RR选项，则接收端必须从IP首部中提取所有信息返回发送端。在同一条IP路径上，接收端会复制IP-RR选项，因此在返回路径上同样会记录IP，使得记录下来的IP地址增加了一倍(可能是冗余的)。<br>
3) "last but not least"：IP首部中留给选项的空间是有限的(IP首部长度选项)，最多只能存放9个IP地址，在ARPANET可能是足够的，但是对于现在的网络是远远不够的。

### traceroute程序机制简单介绍
traceroute使用ICMP报文作为载体，使用IP首部中的TTL(time to live)字段对于最长传输经过的路由数进行限制。

**[1] TTL字段的基本概念** <br> 
TTL字段是由发送端设置的一个8bit字段，TTL初始值由"Assigned Numbers RFC"进行指定，早期版本为15或32，当前(1997)值为64。在chapter7中ICMP回显应答数据报中，TTL被通常被设置成最大值255。

一般情况下，每个处理IP数据报的服务器都需要把TTL数字减一or减去在路由器中停留的秒数(RFC1009)。大多数路由器处理数据报的时延小于1s，因此TTL成为"跳站计数器"。很少有路由器支持上述这种实现，在RFC[Almquist 1993]将它指定为可选的功能，将TTL规范成一个纯粹的"跳站计数器"。

当路由器收到一份IP数据报，其TTL字段是0或1的时，路由器将不再转发该数据报，接收该数据报的主机可将其交给应用程序。正常情况下，路由器不应接受TTL字段为0的数据报，相反应将它直接丢弃，并返回给源主机一份超时ICMP。[traceroute程的关键在与目的主机通信中途的router返回的这封超时ICMP报文包含了目的主机的信源地址]。

**[2] TTL字段的目的** <br>
单纯用TTL记录网络路径中记录路由器跳数没有太多实际意义，TTL字段根本目的是防止数据报选路时无休止地在网络中流动。例如：当路由器瘫痪or两个路由器之间的链接丢失，选路协议可能检测丢失的路由并一直进行下去，TTL字段为这些循环的数据报加上一个生存上限，使其不能无休止传递。

**[3] traceroute程序获取路由信息的操作流程** <br>
1) 源主机发送一份TTL字段为1的IP数据报给目的主机。IP数据报发送的第一站路由将返回TTL值减1，丢弃该数据报，并返回一封超时ICMP报文，这样源主机得到了路由路径中的"1st hop router IP"。<br>
2) traceroute程序发送一封TTL字段为2的IP数据包给目的主机，这样就可以得到"2nd hop router"。以此类推，直到下一跳目标为目的主机。<br>
3) 即使当目的主机在接收到TTL值为1的数据报时，也不会丢弃该数据报，更不会产生超时ICMP报文(数据已经到达目的地)。此时，traceroute程序发送一封udp数据报给目的主机，并选择一个"not likely number(\>30000)"作为udp的端口号，使得目的主机上的任何程序都不可能使用该端口。这样目的主机udp服务模块将会产生一份端口不可达ICMP报文返回给源主机，并以此方式返回自身的IP地址。<br>
4) traceroute程序通过判断收到的ICMP报文类型(超时ICMP、端口不可达ICMP)来决定何时结束。

**[4] 运行traceroute程序的基本要求** <br>
traceroute程序需要通过设置TTL字段，才能正常运行，并非所有针对TCP/IP协议的"programming interface"都支持这项功能，在程序层面也不是所有的implementation都支持设置TTL字段的能力。目前大部分系统支持设置TTL字段的能力，并可以运行traceroute程序。不过运行traceroute程序通常需要超级用户权限。

### LAN (local area network)上的输出
案例使用chapter1中通用的主机网络结构中的一个局域网主机部分进行traceroute程序测试，并通过tcpdump程序对于输出进行分析。进一步地，计算源主机svr4到目标主机slip之间的理论往返时间(round-tritime,RTT)，并与traceroute程序得到的时间相对比。

##### [1] 案例所使用的主机网络结构
其中，bsdi主机和slip主机之间为9600bit/s的SLIP线路。
<center><img src="/img/in-post/tcp-ip_img/traceroute_0.pdf" width="60%"></center>

##### [2] traceroute命令行输出分析
<center><img src="/img/in-post/tcp-ip_img/traceroute_1.pdf" width="100%"></center>

##### [3] tcpdump命令打印数据报发送情况
tcpdump抓包所得到的结果证实：在进行IP数据报传输的时候确实需要先进行ARP通信。
<center><img src="/img/in-post/tcp-ip_img/traceroute_2.pdf" width="100%"></center>

##### [4] 超时ICMP数据报的格式
有两种不同的超时ICMP数据报，它们在报文格式上的区别在于8bit代码字段的数值不同。<br>
**1** 本文所应用的报文类型8bit代码为0，这种报文是在TTL字段值为0的时候产生的。**2** 主机在组装分组的IP数据报时可能发生超时，此时主机将发送一份"组装报文超时"的ICMP报文(8bit代码为1)，将会在chapter11.5进行讨论。

<center><img src="/img/in-post/tcp-ip_img/traceroute_3.pdf" width="60%"></center>

##### [5] SLIP线路理论往返时间
**发送的UDP数据报** 发送的数据报共包含42byte: 12byte数据部分+8byteUDP首部+20byteIP首部+2byteSLIP终止符(至少)。<br>
**返回的超时ICMP差错报文** 返回的数据报长度为58byte: 20byte返回IP数据报首部+8byte超时ICMP首部+20byte产生差错的ICMP首部+8byte产生差错数据部分前8byte+2byteSLIP终止符(至少)。<br>
**计算RTT理论往返时间** 理论往返收发字节总数/线路理论传输速率=(42byte+58byte)/960byte/s=104ms。

##### [6] traceroute程序注意事项
1) 不能保证当前采用的路由也是将来所采用的路由，因此关于线路传输时间的估计等测算都是临时的。甚至两份连续发送的IP数据报也可能采用不同的路由。<br>
使用traceroute可以可以观察到这种路由变化：对于一个给定的TTL数值，其路由发生变化，traceroute将打印出新的IP地址。<br>
2) 不能保证traceroute程序发送的udp数据报和路由路径上的路由返回的超时ICMP、端口不可达ICMP采用相同的路由。由于发送和返回可能采用了两条路由线路，因此traceroute程序上的往返时间RTT可能并不能体现出特定路径上数据从发出到返回需要的时间。<br>
3) traceroute程序返回的信源IP地址是udp数据报到达的路由器接口的IP地址。和IP记录路由(IP-RR)选项总是记录ICMP发送端口的地址。<br>
4) 由于路由器至少具有2个接口的关系，从A路由器到B路由器和从B路由器到A路由器两个方向上运行traceroute程序，得到的结果也可能是不同的。例如，从slip主机上运行traceroute程序到bsdi主机得到如下结果。
<center><img src="/img/in-post/tcp-ip_img/traceroute_4.pdf" width="100%"></center>
对比之前的输出发现：即使经过了相同的bsdi主机，由于使用的端口不一样，其IP地址发生了改变。每个不同的端口可以配置不同的主机名，因此两个方向同一条线路所记录的路由主机名也可能不同。

##### [7] DNS在IP路由路径表示中的作用
参考主机网络结构如下图所示，两个LAN网络通过两个路由器相连，路由器之间的网络为点到点链路。
如果从左边网络1中的主机运行traceroute程序到网络2中的主机2上，程序将发现路由上的IP地址为f1、f3；从主机2上运行traceroute程序到主机1上，程序将发现路由上的IP地址为f2、f4。<br>
<center><img src="/img/in-post/tcp-ip_img/traceroute_5.pdf" width="40%"></center>
需要注意的是：两次路由发现的记录结果中，f2和f3的网络号相同(处于同一网段)，但是f1和f4网络号不同。这样，两次运行traceroute程序得到的结果虽然在相同的线路上传递，却得到了"非常不直观、难以理解"的显示。<br>
如果traceroute能输出易读的域名，那么将有助于对程序结果的理解。但当traceroute程序收到ICMP报文时，所能获取的唯一信息即IP地址，因此在给定IP地址的情况下，程序需要进行一个"反向域名查看"以从IP获得域名。这项服务涉及DNS，DNS需要路由器、主机的管理员进行配置。关于DNS的内容在chapter14.5进行介绍。

### WAN (wide area network)上的输出
LAN部分的内容对于理解traceroute程序中的简单机制、路由记录相关协议原理已经足够。这部分内容主要展示了traceroute程序在worldwide internet上的运行结果，并进行简单的解析。<br>
在sun主机上运行traceroute程序到NIC(network information center)的情况如下图所示：
<center><img src="/img/in-post/tcp-ip_img/traceroute_6.pdf" width="100%"></center>
**Gist-1** TTL字段为3的第一个往返时间RTT数值比TTL字段为2的第一个往返时间要小。RTT时间时从发送主机到某一路由器，再返回发送主机的时间，由于两次通信使用的端口可能不同，以及每个端口的拥塞程度不同，因此这种情况是可解释的。<br>
**Gist-2** TTL字段为6的第2个RTT数值为590ms，几乎为其他两份发送的数据报的RTT数值的两倍(234ms、262ms)。这种情况表明：IP路由不同显著影响通信的效率。

### IP源站选路选项
##### [1] 基本概念
通常IP路由是动态的，即每个路由器都要判断特定数据报要下面应该转发到哪个路由器。应用程序不会对需要经过的路由进行控制，通常也不concerned with路由。应用程序借助类似traceroute的程序来发现实际的路由。<br>

源站选路(source routing)的思想是：由数据报发送端指定路由，通常有两种形式：<br>
**严格的源站选路** 发送端指明IP数据报必须采用的exact routing path。在IP数据报传输过程中，如果某个路由器发现source route指定的下一个路由器不在与其直接连接的网络上，那么它就返回一个"source route failed"信息。<br>
**宽松的源站选路** 发送端指明了一份必须要经过的IP地址清单(路由端口)，但是清单上任何两个IP地址之间允许通过其他路由器。

##### [2] 源站选路选项的格式
可以在IP数据报首部中指定源站路由选项。另外，traceroute程序提供了一种查看源站路由的方法，可以通过该程序检查源站路由机制的运行情况。<br>
源站路由选项的全称应该为"源站以及记录路由"(source and record route)。源站以及记录路由的配置方式如下图所示：
<center><img src="/img/in-post/tcp-ip_img/traceroute_7.pdf" width="100%"></center>
需要注意的是：对于源站选路选项而言，需要在IP数据报发送之前填充源站选路选项的表单(通常其数量小于9)；而对于记录路由选项而言，份必须尽可能分配空间以占满9个地址。

##### [3] 源站选路的基本运行过程
1) 发送主机从应用程序中接受源站路由清单(格式如上图所示)，将第一个表项去掉(存放了数据报最终目的地址)，将剩余的IP地址清单左移一位，将存放数据报目的地址的表项放在清单的最后一项。ptr指针仍然指向地址清单的第一项(ptr=4)。<br>
2) 每个处理IP数据报的路由器检查当前source address是否为数据报中源站选路选项的destination IP address，如果不是destination IP address，则按照路由器正常转发机制对数据报进行转发(必须指定宽松的源站选路，否则目的主机接受不到发送的数据报，而是返回给源主机"source route failed"信息)。<br>
3) 每个处理IP数据报的路由器检查当前source address是否为数据报中源站选路选项的destination IP address，如果是destination IP address，且ptr不大于路径的长度，那么：a) 源站选路清单中下一个地址(where ptr point)就是数据报的new destination address。b) 用路由器outgoing interface IP address作为new source address。c) ptr指针向右移动4bit。

##### [4] 源站选路的案例分析
<center><img src="/img/in-post/tcp-ip_img/traceroute_8.pdf" width="100%"></center>

##### [5] 关于源站选路概念的补充
Host Requirments RFC指明：TCP客户端需要具备源站选路的能力，也同时应该具备接受一份源站选路的数据报的能力，并且对于所有segments on that TCP connection都可以提供reverse route用于应答。另外，当它收到一份来自不同路由选路的数据报时，新的源站选路应该override旧的源站选路。

##### [6] 宽松的源站选路traceroute程序示例
使用"traceroute -g"命令可以为宽松的源站选路指定一些路由器，该选项最多指定8个路由器(最后一个源站选路空间位置留给目的主机)。使用"traceroute -g"命令的程序示例如下：
<center><img src="/img/in-post/tcp-ip_img/traceroute_9.pdf" width="100%"></center>

##### [7] 严格的源站选路traceroute程序示例
使用"traceroute -G"命令可以为严格的源站选路制定一些路由器，该选项最多指定8个路由器(最后一个源站选路空间位置留给目的主机)。使用"traceroute -G"的程序示例如下：
<center><img src="/img/in-post/tcp-ip_img/traceroute_9.pdf" width="100%"></center>

使用"tcpdump -v"命令以用来显示出源站路由信息，但是同时-v选项还会输出一些如数据报ID的不必要的信息，不妨将多余的信息手动删除。在源主机sun上运行该命令得到的输出结果如下：
<center><img src="/img/in-post/tcp-ip_img/traceroute_10.pdf" width="100%"></center>

##### [8] 宽松的源站选路traceroute程序往返路由
通常情况下，从主机A到主机B的路由和从主机B到主机A的路由不一定完全一致。为了验证两个方向的路由是否相同，只能在主机A和主机B上分别运行traceroute程序，否则很难发现两条路径是否不同。<br>
但是，采用宽松的源站选路就可以决定两个方向上的路由：

<center><img src="/img/in-post/tcp-ip_img/traceroute_11.pdf" width="100%"></center>

### 小结
在一个应用TCP/IP协议簇的网络中，traceroute是不可或缺的研究工具。traceroute工具操作命令简单明了，但功能强大，针对网络路由路径分析非常有用。更多关于traceroute命令的文档资源、常用命令有待整理。

## Reference
> \<tcp-ip: illustrated vol1\> chapter8 <br>

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
> 问题脚注: ### ??????problem😫problem??????
