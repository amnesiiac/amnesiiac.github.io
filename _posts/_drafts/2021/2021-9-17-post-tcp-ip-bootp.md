---
layout: post
title: "tcp-ip: BOOTP (引导程序协议)"
subtitle: '[tcp-ip: illustrated vol1] - chapter16' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-09-17 00:17
lang: ch 
catalog: true 
categories: tcp-ip 
tags:
  - Time 2021
  - tcp-ip illustrated vol1 
---
### 一：引言 
chapter5中提到了无盘系统，它可在不知道自身IP情况下，通过逆地址解析协议(RARP)在bootstrap时获取自身IP地址。然而，使用RARP会存在2个问题：**(i)** 自身IP地址是唯一的返回结果(返回的信息量有限)；**(ii)** RARP属于链路层的协议，因此RARP请求不会被router转发，因此每个网络都需要架设一个RARP服务器专门用于RARP通信。<br>
本章将介绍另外一种用于系统bootstrap的协议：引导程序协议(BOOTP)。BOOTP使用UDP作为传输媒介，并通常**需要和TFTP一起协同工作**。RFC-951是BOOTP的正式规范，RFC-1542对它做了阐述说明。

### 二：BOOTP数据报的分组格式

##### 1 BOOTP分组在UDP数据报中的封装格式
<center><img src="/img/in-post/tcp-ip_img/bootp_1.pdf" width="40%"></center>

##### 2 BOOTP请求&应答分组的具体格式
<center><img src="/img/in-post/tcp-ip_img/bootp_2.pdf" width="60%"></center>
- **"操作码"字段**为1表示请求，2表示应答。
- "硬件类型"字段为1表示10Mb/s的以太网(和ARP请求&应答报文中的同名字段的含义相同)。
- 对于以太网链路，**"硬件地址长度"**字段为6byte。
- **"跳数"**字段被客户端设置为0，但也可以被代理服务器所使用。
- **"事务标识"**字段：由客户端设置并由服务端返回的32bit整数。客户端用它对请求&应答进行匹配。客户端将每份BOOTP请求报文的该字段设为随机数。 
- 客户端开始进行引导时即设置**"秒数"**字段。所有服务端能够检查该时间值，备用服务器(secondary server)在等待时间超过此设定值后才响应客户端请求(主服务器未响应)。
- 如果客户端已经知道自身的IP地址，它将该数值写入**"client IP addr"**字段；否则(unknown about self-IP)，客户端将该字段设置为0。
- 如果"client IP addr"为0(客户端不知道自身IP)，服务器将该客户的IP地址填入**"your IP addr"**字段。
- **"server IP addr"**字段由服务端填写。
- 如果使用了proxy server，则该proxy负责将自己的网关信息填入**"gateway IP addr"**字段。
- 客户端必须设置它自身的**"client hardware addr"**字段(即使该数值和以太网帧首部中的数值相同)。在UDP-BOOTP数据报内部重复设置该字段的意义在于：方便进程能够通过UDP层来获取"client hardware addr"信息，because通常进程通过查看UDP数据报以获得存储在以太网帧中的信息是几乎不可能的。
- **"server hostname"**字段是一个"null terminate string"，由服务端填写。
- 服务器还将在**"bootstrap filename"**字段填入：用于系统引导的文件名及其路径。
- **"vendor-specific information"**字段用于对BOOTP进行不同的扩展。

##### 3 使用BOOTP进行系统引导 - IP地址&端口号
**[1] IP地址** <br>
客户端使用BOOTP操作码=1进行自身系统引导时，BOOTP通常采用链路层广播的方式发送。引导的IP数据报通常以limited broadcast方式发送，即目的IP地址为:255.255.255.255，源IP地址通常为0.0.0.0。

**[2] 端口号** <br>
BOOTP有两个相关公知端口号：67(服务端)和68(客户端)。BOOTP客户端不会像TFTP一样选择unused临时端口进行通信，只使用68端口进行通信。

**[3] 为什么BOOTP服务端和接收端需要使用两个端口号，而不是一个？** <br>
因为BOOTP请求通过广播形式发送BOOTP报文(UDP数据报)，并且BOOTP应答报文也可以通过广播的形式发送数据报(尽管通常不使用广播方式发送)。因此无论是服务端还是接收端，都需要具备接受广播数据报分组的能力。

回顾广播相关知识：广播需要具备一个广播组，只有特定广播组内的节点才能接受到发送到该广播组的信息。广播组是接受广播数据报文的端口号决定的：局域网内的一个节点如果设置了广播属性并监听了端口号A时，该节点就加入了A广播组。<br>
如果一个节点想要接受A广播数据信息，需要事先绑定IP地址&端口A，然后设置该socket属性为广播属性。如果一个节点不想接受广播数据信息，只发送广播数据信息，则只需要事先为socket设置广播属性，无需绑定端口，直接向广播地址A端口发送UDP数据报即可。

综上所述：为了能够让BOOTP服务端、接收端都能接受广播数据报，则必须为两端都设置公知端口。

**[4] 客户端引导自身系统时需要对收到的广播数据报进行过滤** <br>
如果多个客户端同时进行系统引导，并且BOOTP服务器将所有BOOTP应答进行广播，那么每个客户都会收到本属于其他客户端的应答。<br>
客户端可以通过BOOTP首部中"事务标识"字段对于收到的所有应答数据报进行遴选，或者可以检查返回的客户硬件地址对收到的应答数据报进行区分。

### 三：使用BOOTP引导一个X终端的案例
使用BOOTP程序引导X终端系统(proteus)，提供引导相关文件的系统为mercury。proteus待引导系统为BOOTP客户端程序，mercury引导文件提供系统为BOOTP服务端程序。使用TFTP简单文件传输协议进行通信，并通过TFTP传输引导文件。

##### 1 使用tcpdump程序分析引导过程数据分组交换情况
下图中的tcpdump程序输出是在不同的网络上打印出的。
<center><img src="/img/in-post/tcp-ip_img/bootp_3.pdf" width="100%"></center>
- BOOTP请求&应答需要重复3次之后，才能够使用TFTP协议进行具体引导文件传输。
- 在整个BOOTP引导过程中涉及两次ARP请求&应答。第一次在首次接收到BOOTP应答数据报后，通过ARP机制确认BOOTP应答返回的IP在网络上是唯一的(不存在其他主机和客户端主机共享相同的IP)；第二次在正式和服务端主机通过TFTP传输正式文件之前，通过ARP获取服务端主机硬件地址用于通信。
- 从第13行到第18行，可以看出引导文件的传输总共用了9s的时间，共传输的总数据量是：516byte × 2463 + 228byte = 1271136byte，其中总共传输的引导文件相关数据量为：512byte × 2463 + (228byte - 4byte) = 1261280byte。
- X终端在通过BOOTP引导后，还需要使用TFTP传输终端相关字体文件、DNS name server queries，最后执行X终端协议初始化。上述过程需要花费6s，加上X终端引导过程需要15s左右，总共的引导时间大约为21s。

### 四：BOOTP服务器的设计 - ditto, omitted

### 五：BOOTP经过路由器(through a router)
RARP协议的缺点是它的数据报传播依赖于链路层广播，这种广播不会被路由器(IP层)进行转发，因此需要在每个物理网络中架设一个RARP服务器。<br>
如果路由器支持BOOTP协议，BOOTP协议数据报可以被路由器转发(它依赖于UDP数据报)，但是通常没有必要将BOOTP请求转发到另一个网络中的服务器上。


### 六：厂商专属信息(vendor-specific info)
关于BOOTP数据报格式中的64byte厂商专属信息字段进行补充介绍。RFC-1533对于该字段进行规范描述。


## Reference
> \<tcp-ip: illustrated vol1\> chapter16 <br>
> https://blog.csdn.net/leonwei/article/details/6202976

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
