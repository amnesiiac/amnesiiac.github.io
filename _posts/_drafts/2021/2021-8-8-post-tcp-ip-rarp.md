---
layout: post
title: "tcp-ip: RARP (逆地址解析协议)"
subtitle: '[tcp-ip: illustrated vol1] - chapter5' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-08-08 15:13
lang: ch 
catalog: true 
categories: tcp-ip 
tags:
  - Time 2021
  - tcp-ip illustrated vol1 
---
### 引言
具有本地磁盘的系统引导时，其IP地址可以从磁盘中的配置文件中读取。但是，对于无盘机、X终端、无盘工作站，需要使用其他方式来获得IP地址。<br>
> RARP is now an obsolete protocol that was used to allow a host to determine it's IP address based on the host's MAC address. RARP协议是一种允许host主机基于自身MAC地址决定其IP地址的机制。不过现在已经过时了。<br>
> The protocol was rendered obsolete by more modern techniques and protocols such as BOOTP and DHCP (Dynamic Host Configuration Protocol). RARP已经被跟多的现代技术如：BOOTP和DHCP技术代替。

无盘系统通过RARP获取自身IP地址的方式大致如下：1 从接口卡中读取唯一的硬件地址。2 发送一份RARP请求在网络上以广播的方式传送，以请求某个主机响应该无盘系统的IP地址，并通过单播的方式返回RARP应答。<br>
上述RARP请求和应答的机制流程听起来比较简单，但是具体实现起来相对ARP困难许多。关于RARP规范的说明详见RFC903。

关于无盘系统(simply borrowed from wikipedia)：无盘系统是一种应用于网吧、卡拉OK、办公室的一种**网络传输技术**。使用无盘系统的电脑不使用本地磁盘获得启动系统，而是通过网络的特定服务器来获得启动镜像，并下载到本地进行启动；同时亦不需要使用传统磁盘读取资料，而是从局域网的服务器读取资料。

### RARP的数据报(分组)格式
RARP的数据报(也称为分组)格式同ARP的数据报格式大同小异。不同之处在于RARP请求&应答的帧类型代码为0x8035；并且RARP的请求操作码为3，应答操作码为4。
<center><img src="/img/in-post/tcp-ip_img/arp_6.pdf" width="100%"></center>
同ARP一致，RARP请求以广播的方式发送，RARP应答以单播的方式发送。

### RARP应用举例
**1 sun主机通过RARP从网络上引导获取自身IP - 收到bsdi主机应答的情况**
<center><img src="/img/in-post/tcp-ip_img/arp_7.pdf" width="100%"></center>

**2 sun主机通过RARP从网络上引导获取自身IP - 但是没有收到应答的情况** <br>
<center><img src="/img/in-post/tcp-ip_img/arp_8.pdf" width="100%"></center>
超时重发采用一定规律下变动的时间间隔要比固定的时间间隔要好。相似的例子在TCP超时重发机制中也可以看到。

### RARP服务器的设计及其复杂性衡量
**1 虽然RARP服务从概念上理解很简单，但想要实现RARP服务器相比ARP服务器要复杂**：ARP服务器通常是TCP/IP在内核中实现的一部分，内核掌握自身的协议地址、硬件地址，所以进行ARP应答(返回自身IP)时非常容易。RARP服务器提供的服务不属于内核层面的应用(内核一般不负责读取分析磁盘文件)，而是用户进程层面的应用。<br>
**2 RARP服务器一般为网络上的多个主机(网络上的所有无盘系统)提供硬件地址到协议地址的映射**。在unix系统中，该映射一般位于"/etc/ethers"目录中。<br>
**3 RARP服务器的实现是和系统绑定在一起的**。RARP请求是作为一种特殊的以太网帧进行传送的(帧类型字段为0x8035)。作为RARP服务器的电脑需要支持发送、接受这种以太网帧，而收发特殊类型的以太网帧需要系统层面的支持。补充：BSD分组过滤器、网络接口栓、SVR4数据提供者接口可以作为RARP服务器。<br>
**4 RARP请求是在网络中的硬件层以广播的方式完成的**，它们不经过路由器(服务于网络层)进行转发(ARP请求很容易实现是因为它知道对面的IP地址，可以通过网络层进行广播)。为了让无盘系统能够在特定RARP服务器关机下也能引导获取自身IP，一条电缆需要配置多个RARP服务器。由于每个RARP请求通过广播的方式进行发送，每个RARP服务器可能在同一时间进行回复，增加了以太网传输冲突的几率。


## Reference
> \<tcp-ip: illustrated vol1\> chapter5 <br>
> https://stackoverflow.com/questions/11669034/application-of-rarp-protocol

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
