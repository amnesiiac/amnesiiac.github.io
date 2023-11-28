---
layout: post
title: "tcp-ip: ICMP (internet控制报文协议)"
subtitle: '[tcp-ip: illustrated vol1] - chapter6' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-08-10 09:51
lang: ch 
catalog: true 
categories: tcp-ip 
tags:
  - Time 2021
  - tcp-ip illustrated vol1 
---
### 引言
**定义：** ICMP(internet message control protocol)，常被认为是IP的一个组成部分，用于传递差错报文、其他需要注意的信息。

**应用：** ICMP通常被IP层，或者更高层协议TCP、UDP等进行使用；也有一些ICMP报文将差错报文返回给用户进程。<br>

**ICMP数据报的封装** <br>
ICMP数据报是封装在IP数据报内部进行传输的。
<center><img src="/img/in-post/tcp-ip_img/icmp_1.pdf" width="60%"></center>

### ICMP数据报的类型
<center><img src="/img/in-post/tcp-ip_img/icmp_2.pdf" width="80%"></center>
**下面几种情况均不会产生ICMP差错报文**：<br>
**•** ICMP差错报文不会产生另一份ICMP差错报文。但ICMP查询报文可能会产生ICMP差错报文。<br> 
**•** 目的地址是广播地址、多播地址(D类地址)的IP数据报。<br>
**•** 用于链路层广播的数据报(RARP请求数据报)。<br>
**•** 不是IP分片的第一片的报文不会产生差错报文。只有IP数据报分片第一片能产生ICMP差错数据报。<br>
**•** 源地址不是单个主机的数据报。源地址为零地址、环回地址、广播地址、多播地址的IP数据报不能产生ICMP差错报文。<br>
**•** 以上种种限制是为了防止由于广播分组机制导致产生ICMP差错数据报的广播风暴。

### ICMP地址掩码请求与应答
ICMP地址掩码请求的作用：无盘系统通过引导过程获取自己的子网掩码。和RARP请求用于获取自身IP地址的方式类似，图盘系统通过广播自己的ICMP请求报文来获得子网掩码。无盘系统获取自身子网掩码的另一种方式是通过BOOTP协议(chapter16)。

##### 一：ICMP地址掩码请求&应答的数据报格式
<center><img src="/img/in-post/tcp-ip_img/icmp_3.pdf" width="60%"></center>

##### 二：ICMP地址掩码请求&应答案例分析
**案例场景** <br>
sun主机希望通过广播发送ICMP子网掩码请求的方式获取自身的子网掩码。位于同一网络的sun主机(self)、bsdi主机、svr4主机收到了该请求，使用tcpdump、ifconfig命令对于它们之间的ICMP通信进行研究。

**[1] ICMP地址掩码请求&应答涉及的sun主机、bsdi主机、svr4主机网络结构图**
<center><img src="/img/in-post/tcp-ip_img/icmp_4.pdf" width="80%"></center>

**[2] sun主机广播方式发送ICMP地址掩码请求 - 收到本机环回应答、bsdi主机、svr4主机的应答如下**
<center><img src="/img/in-post/tcp-ip_img/icmp_5.pdf" width="100%"></center>

**[3] 在svr4主机的emd0接口上运行ifconfig命令，获取其自身配置的子网掩码信息(用于错误排除)**
<center><img src="/img/in-post/tcp-ip_img/icmp_6.pdf" width="100%"></center>
由于sun主机、bsdi主机、svr4主机都位于同一个以太网上，因此它们主机配置的子网掩码应该相同。根据ifconfig命令查询结果显示，svr4自身配置的子网掩码是对的，但是它发送给sun主机的ICMP子网掩码应答出现了错误，因此可以断定是svr4的ICMP子网掩码应答部分产生了问题。

**[4] 在bsdi主机上使用tcpdump -e来查看bsdi发包情况 (BSD/386操作系统产生了一个错误)**
<center><img src="/img/in-post/tcp-ip_img/icmp_7.pdf" width="100%"></center>

**[5] 关于特定主机发送ICMP子网掩码请求&应答的"资质条件"** <br>
RFC规定：除非主机系统被特殊配置为地址掩码的授权代理，否则它不能发送地址掩码应答。详细可以参考tcp-ip: illustrated vol1 appendix-E。<br>

**[6] 通过ICMP子网掩码请求&应答来进一步证明：向本机IP地址发送IP数据报需要通过环回接口**
<center><img src="/img/in-post/tcp-ip_img/icmp_8.pdf" width="100%"></center>

### ICMP时间戳请求与应答
ICMP时间戳允许：一个系统向另外一个系统查询当前时间。<br>
ICMP时间戳返回的时间格式为：自0:00 00 000计算的毫秒数Coordinated universal time(UTC)。<br>
ICMP时间戳返回的时间数值始终小于(86 400 000ms = 24h × 60min × 60s × 1000ms)。<br>
使用ICMP时间戳请求&应答报文机制的好处是：它提供了毫秒级分辨率。而其他方法从别的主机获取时间(如某些unix系统的rdate命令)只能提供"秒"级分辨率。

##### 一：ICMP时间戳请求&应答的基本格式
<center><img src="/img/in-post/tcp-ip_img/icmp_9.pdf" width="60%"></center>

##### 二：ICMP时间戳请求&应答案例
**[1] 编写程序icmptime实现ICMP时间戳请求的发送并打印ICMP时间戳请求应答**
<center><img src="/img/in-post/tcp-ip_img/icmp_10.pdf" width="100%"></center>

**[2] ICMP时间戳相关计算中涉及的时间概念关系图**
<center><img src="/img/in-post/tcp-ip_img/icmp_11.pdf" width="50%"></center>

**[3] 计算sun和bsdi的主机时间差值需要假定的基本条件** <br>
**1** 选择相信RTT(往返时间)是准确的。相信相对时间相比相信绝对时间更加靠谱。<br>
**2** 相信RTT(往返时间)中的一半用于ICMP时间戳请求的传输，一半用于ICMP时间戳应答的传输。<br>
**3** 相信difference(recv-orig)，但是不相信sun、bsdi主机上的所有时间。<br>

**[4] 计算sun和bsdi的主机时间差值** <br>
<center><img src="/img/in-post/tcp-ip_img/icmp_12.pdf" width="100%"></center>

**[5] ICMP时间戳接受报文主机的recv时间可以判断其系统软件的时间分辨率** <br>
<center><img src="/img/in-post/tcp-ip_img/icmp_13.pdf" width="100%"></center>

**[6] 特殊的ICMP时间戳 - 采用非0:00 00 000起计时的时间**
<center><img src="/img/in-post/tcp-ip_img/icmp_14.pdf" width="100%"></center>

### 获取目的主机系统日期、时间的其他方法
**[1] 调用通用基本简单服务(daytime、time)** <br>
在sun主机上使用telnet命令远程获取bsdi主机日期 - 调用daytime服务
<center><img src="/img/in-post/tcp-ip_img/icmp_15.pdf" width="100%"></center>

其他标准简单服务如下表
<center><img src="/img/in-post/tcp-ip_img/icmp_16.pdf" width="70%"></center>

**[2] 严格时间计数器使用网络时间协议(network time protocol, NTP)** <br>
NTP协议采用了先进的技术保证了LAN、WAN上的一组时钟误差在ms以内。更多关于计算机精确时间的内容详见RFC-1305。

**[3] 开放软件基金会(OSF)的分布式计算环境(DCE)定义了分布式时间服务(DTS)** <br>
这种服务提供了计算机间的时钟同步。详情参考文献[Rosenberg, Kenney and Fisher1992]。

**[4] 伯克利的unix系统提供了一种timed服务** <br>
timed服务可用来同步局域网上的时钟，但是与NTP、DTS服务不同，timed不在广域网范围内工作。

### 查看&分析ICMP差错报文中附加的信息 (以ICMP端口不可达报文为例)
ICMP端口不可达报文是ICMP目的不可达报文的一种，希望通过它对于ICMP差错报文中附加信息进行研究。下面主要通过运行UDP协议进程来产生ICMP ICMP不可达报文，并进行研究。

##### 一：生成一份ICMP端口不可达差错报文
生成ICMP不可达报文需要利用udp的一项规则：[如果目的主机收到一个udp数据报，而目的端口不对应于某个进程正在使用的端口，udp就会以ICMP端口不可达的方式作出回应]。可以使用TFTP强制生成一份ICMP端口不可达报文：
<center><img src="/img/in-post/tcp-ip_img/icmp_17.pdf" width="100%"></center>

使用tcpdump命令打印bsdi主机和svr4主机udp报文交换的结果：
<center><img src="/img/in-post/tcp-ip_img/icmp_18.pdf" width="100%"></center>
上表中需要注意的是：<br> 
**1** ICMP端口不可达差错是立即返回的(每次svr4收到udp数据报间隔0.0035s左右即发送ICMP应答)；<br> 
**2** TFTP客户端重发算法忽略ICMP端口不可达报文，每隔5s重发udp；<br> 
**3** ICMP端口不可达报文是主机之间(svr4-\>bsdi)进行交换的报文，本身依赖于TCP/IP无需附加端口号；<br> 
**4** udp数据报从特定的端口(2924)发送到另一个特定的端口号(8888)。<br> 
**5** 20byte字节长度指udp数据报中数据的长度，包含TFTP2字节操作码、9个字节的文件名(末尾空字符)"temp.foo"、9个字节的字符串(末尾空字符)"netascii"，详见TFTP报文格式。<br>
**6** BSD系统不会将从socket接收到的ICMP报文中UDP数据通知TFTP用户进程，除非BSD TFTP客户端中进程发送"connect"命令给该端口(标准BSD系统不使用connect命令)。本案例中显式使用了connect命令给端口8888，因此ICMP差错报文通知了TFTP进程，触发了重传算法(每隔5s重发，相比TCP的重发机制，not so good)。

##### 二：ICMP端口不可达差错报文格式
**[1]** 使用"tcpdump -e"命令打印结果，就可以看到每个返回ICMP端口不可达报文的完整长度(70byte)。下图展示了ICMP端口不可达("UDP端口不可达")报文的数据结构。
<center><img src="/img/in-post/tcp-ip_img/icmp_19.pdf" width="70%"></center>
• ICMP不可达报文通用形式中规定包含的"至少包含差错IP数据报后的8byte"是必要的：UDP/TCP首部中前8个字节包含重要信息(源端口号、目的端口号)，ICMP端口不可达差错报文是由于目的端口没有被目的主机上的任何继承占用，而返回的，因此通知这个"未被进程监控的unreachable port"给ICMP发送方是至关重要的。<br>
• ICMP不可达报文通用形式中规定包含的"差错数据报中的IP首部(包含选项)"是必要的：IP首部中包含了协议字段，使得ICMP能够知道如何解释上一条中包含的8byte信息。

**[2] ICMP不可达报文的通用数据报格式**
<center><img src="/img/in-post/tcp-ip_img/icmp_20.pdf" width="70%"></center>
尽管ICMP协议规范允许返回差错IP数据报首部后续的至少8byte，但是大多数BSD系统都只返回8byte。


### ICMP报文在BSD 4.4系统上的处理
ICMP报文覆盖的范围较广，从信息差错到致命差错，因此对于不同的ICMP报文，系统给出的处理方法迥异；对于不同的系统针对ICMP报文也有不同的处理规范。下表展示了BSD 4.4系统针对ICMP报文的处理方式。
<center><img src="/img/in-post/tcp-ip_img/icmp_21.pdf" width="80%"></center>

## Reference
> \<tcp-ip: illustrated vol1\> chapter6 <br>

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
> 10 `<font style="color:red; font-weight:bold">加粗蓝色</font>`用来设置字体颜色
