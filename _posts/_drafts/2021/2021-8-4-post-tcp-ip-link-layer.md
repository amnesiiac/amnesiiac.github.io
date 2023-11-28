---
layout: post
title: "tcp-ip: link layer (链路层协议)"
subtitle: '[tcp-ip: illustrated vol1] - chapter2' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-08-04 21:27
lang: ch 
catalog: true 
categories: tcp-ip 
tags:
  - Time 2021
  - tcp-ip illustrated vol1 
---
### IEEE-802封装以及以太网封装
<center><img src="/img/in-post/tcp-ip_img/link_layer_1.pdf" width="100%">tcp_ip_4.asta</center>

### 串行接口链路层协议1 - 最小封装协议(SLIP)
SLIP是串行线路上对IP数据报进行封装的简单形式，在RFC1055有详细的描述。
<center><img src="/img/in-post/tcp-ip_img/link_layer_2.pdf" width="80%">link_layer.keynote</center>

### 串行接口链路层协议2 - 点对点协议(PPP)
<center><img src="/img/in-post/tcp-ip_img/link_layer_3.pdf" width="80%">link_layer.keynote</center>

### 处理目的地址为本机IP地址的机制 - 环回接口(loopback interface)
<center><img src="/img/in-post/tcp-ip_img/link_layer_4.pdf" width="80%">link_layer.keynote</center>

### 最大传输单元MTU及路径MTU
<center><img src="/img/in-post/tcp-ip_img/link_layer_5.pdf" width="80%">link_layer.keynote</center>

### 关于串行线路吞吐量的计算
**1** 假设线路速率是*9600bit/s*，*1byte=8bit*，另外需要1个起始*bit*和1个停止*bit*(e.g.UART用于数据同步)，因此线路速率是: *9600bit/s÷10bit/s=960byte/s=960B/s*. <br>

**2** 以*960B/s*的线路速率传输*1024byte*需要: *1024B÷960B/s=1066ms* 才能把数据传递出去。<br>

**3** 假设当前使用SLIP运行一个交互式应用程序，同时还运行一个FTP文件发送*1024byte*的程序，则平均等待时间则为*533ms*. <br>
**平均等待时间的计算**(只适用于SLIP或PPP链路):<br> SLIP没有类型字段，因此无法在执行SLIP时同时执行其他协议：因此要么先为SLIP交互式程序发送数据，要么先为FTP程序发送数据，两种情况SLIP等待时间分别为:0ms和1066ms。则平均等待时间为: *(0ms+1066ms)/2=533ms*. <br>

**4** 对于交互式应用程序而言，等待*533ms*是不能接受的。可以通过限制SLIP的*MTU=256byte*来减少等待时间。<br>
**计算**：根据比例关系易知传输一帧数据最长等待时间为*266ms*，平均等待时间为*133ms*。<br> 
**选择*256byte*而不是更小的64或128是因为**：大块数据拥有更好的线路利用率：例如CSLIP报文首部*5byte*，数据帧总长度为*261byte*，因此线路利用率为: *(261byte-5byte)/261byte=98.1%*，因此设置*MTU=256byte*是"非常划算的"。如果使用SLIP的*MTU=64byte*，则线路利用率为: *(64byte-5byte)/64byte=92.2%*。

**5** 关于为什么最大传输单元MTU表格中点对点的传输*MTU=296byte*?<br>
**解释**：应用程序数据为*256byte*，TCP和IP的首部占用*40byte*，因此总共需要*MTU=296byte*。(*应用程序数据+TCP/IP首部=IP-datagram*)。<br>
**补充**：最大传输单元MTU是一个链路层概念，而上述IP-datagram计算是传输层概念，在链路层使用SLIP点对点传输，则IP-datagram中TCP/IP首部可能会被CSLIP压缩成*5byte*。设置*MTU=296byte*而不是*MTU=261byte*的原因是：IP传输层需要根据MTU进行分片，IP层链路层的压缩一无所知，因此只能使用未压缩的MTU数值。

**6** 通过TCP/IP首部压缩，可大幅度减少平均等待时间(*数据往返时间/2*)。<br>
**解释**：线路传输速率为*960B/s*，*1byte*应用程序数据，分别使用*40byte*和*5byte*的原始首部和压缩首部，需要的数据往返时间分别为*85.4ms*和*12.5ms*。

## Reference
> \<tcp-ip: illustrated vol1\> chapter2

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
