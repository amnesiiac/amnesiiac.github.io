---
layout: post
title: "tcp-ip: ARP (地址解析协议)"
subtitle: '[tcp-ip: illustrated vol1] - chapter4' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-08-07 10:14
lang: ch 
catalog: true 
categories: tcp-ip 
tags:
  - Time 2021
  - tcp-ip illustrated vol1 
  - todo
---
### ARP introduction
<center><img src="/img/in-post/tcp-ip_img/arp_1.pdf" width="80%"></center>

### ARP transmission process 
<center><img src="/img/in-post/tcp-ip_img/arp_2.pdf" width="100%"></center>

### ARP高速缓存
ARP地址解析机制能够高速运行的关键在于，每个主机上都有一个ARP高速缓存。这份缓存记录了最近Internet地址到硬件地址之间的映射关系。高速缓存中每一项的生存时间为20min，起始时间从被创建开始算起。
<center><img src="/img/in-post/tcp-ip_img/arp_3.pdf" width="80%"></center>

### ARP请求&应答报文格式
<center><img src="/img/in-post/tcp-ip_img/arp_4.pdf" width="100%"></center>

### ARP代理(proxy ARP)
<center><img src="/img/in-post/tcp-ip_img/arp_5.pdf" width="80%"></center>
**sun即是目的主机也是140.252.13子网的路由器** <br>
通过netb作为ARP代理路由器，可以将数据报传送给主机sun处。根据ARP请求&应答机制，当ARP请求传递到140.252.13子网时，是如何进行ARP广播以找到目的主机，返回ARP应答呢？<br>
其实，路由选路机制保证了这种ARP请求广播传输路径规划。选路表中在140.252处指定：使得数据报的目的端要么是子网140.252.13，要么是子网中的某个主机。sun主机既是140.252.13子网的主机，同时相当于该子网的路由器，也可以是该子网的ARP代理。

**关于ARP代理的物理线路隐藏机制** <br>
ARP代理也称为混合ARP(promiscuous ARP)或者ARP非正式用法(ARP hack)，通过上图中的ARP代理机制可以对gemini主机隐藏sun主机。同理，通过两个物理网络之间的路由器可以互相隐藏物理网络(两个路由器都作为ARP proxy)。

### 免费ARP(gratuitous ARP)
**免费ARP**：主机发送ARP请求来查询自己的IP地址。通常gratuitous ARP在进行系统引导期间进行接口配置时应用。gratuitous ARP请求中，**发送端IP地址协议字段和目的端IP地址协议字段是相同的**，即该ARP请求的目的地址是自身IP。如果网络上不存在其他主机和自己的IP相同，那么这封ARP请求报文将不会收到ARP应答，否则将会收到拥有相同IP的主机的ARP应答。

**免费ARP的作用**：<br>
**(1)** 一个主机可以通过gratuitous ARP来确定是否有另一个主机设置了相同的IP地址。主机不希望这个gratuitous ARP请求发送出去收到ARP应答，如果收到ARP应答，则在终端日志上记录错误信息：以太网地址 A:B:C:D:E:F发来重复的IP地址。<br>
**(2)** 一个主机可以通过gratuitous ARP来更新它在网络上的信息。网络上接收到ARP请求的主机，如果其ARP高速缓存中已经存在发自该协议地址的"硬件地址-协议地址"表项，则需要按照新接受的ARP协议内容进行更新。如：一台主机由于网卡损坏，关机后安装了接口卡，重新启动，此时可以通过gratuitous ARP向互联网广播以实现信息更新&注册。

### ARP command - todo


## Reference
> \<tcp-ip: illustrated vol1\> chapter4

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
