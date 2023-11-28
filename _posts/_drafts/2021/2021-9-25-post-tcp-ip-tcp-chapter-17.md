---
layout: post
title: "tcp-ip: TCP (传输控制协议)"
subtitle: '[tcp-ip: illustrated vol1] - chapter17' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-09-25 10:50
lang: ch 
catalog: true 
categories: tcp-ip 
tags:
  - Time 2021
  - tcp-ip illustrated vol1 
---
### 一：引言
本文主要介绍TCP在应用层提供的传输服务，以及TCP数据报首部字段的格式、意义。

### 二：TCP提供的服务
尽管TCP和UDP都借助相同的网络层(IP)，但是TCP却和UDP层提供了完全不同的服务。<br> UDP提供的是无连接的、不可靠的传输服务，而TCP提供的是面向连接的、可靠的字节流传输服务。面向连接是指：两个使用TCP进行通信的应用(client和server)在交换数据之前需要建立TCP连接。

##### 1 TCP传输保证可靠性的方式
- 应用程序的数据首先被分成"best-sized data chunks"来进行发送。这和UDP完全不同，使用UDP发送数据的应用程序产生的UDP数据报长度保持不变。TCP传递给网络IP层的信息单位称为数据段(segment)。对于TCP如何确定传输的segment长度，在下一章中进行介绍。
- 当TCP发送一个segment后，启动一个定时器，等待目的端返回的针对该segment的ACK信号，如果不能及时收到ACK信号，TCP将会重发该数据段。
- 当TCP收到发送自TCP连接另一端的数据，它将发送一个确认。该确认不是立即发送，通常将推迟一定时间(1/?秒)再进行发送。
- TCP为其首部和数据部分分别维护校验和，这是一个端到端的检验和，用于监测数据在传输过程中的是否出现差错。如果校验和出现错误，则接收端会丢弃该数据报，不会发送ACK确认接受数据报，等待接收端超时重发。
- TCP使用IP数据报进行传输，IP数据报在传输过程中可能失序，因此导致TCP在传输中可能失序。接收端TCP负责将失序的数据报重新整理并传递给应用程序。
- TCP使用IP协议进行传输，IP数据包在传输过程中可能发生重复，TCP接收端应负责将重复的数据报丢弃。
- TCP具有流量控制功能，TCP连接的两端都有固定大小的缓冲区空间。TCP接收端只允许另一端发送不超过接收端缓冲区能够容纳的数据。该功能用于防止处理数据报较慢的主机发生缓冲溢出。

##### 2 TCP连接通过字节流进行通信
两个应用程序通过TCP连接交换8bit字节流(byte stream)，称之为字节流服务(byte stream service)。需要注意的是TCP并不会在字节流中插入记录标识(record marker)，因此数据发送端通过TCP分3次写入10byte，20byte，50byte；接收端通过TCP无法了解发送端每次发送的字节数，可以分4次接受每次接受20byte。<br>
TCP并不会对于传输的字节流进行解析，因此TCP传输程序无法获悉传输的数据是二进制数据，还是ASCII字符，还是EBCDIC字符或者任何其他类型数据。对于字节流的解释应该交给TCP连接双方应用层进行。这种处理方式和linux内核对于读写的内容不作任何解析，读写内容的解析全部交给应用程序处理。


### 三：TCP首部
##### 1 TCP在IP数据报中的封装方式
<center><img src="/img/in-post/tcp-ip_img/tcp_17_1.pdf" width="50%"></center>

##### 2 TCP首部数据格式
<center><img src="/img/in-post/tcp-ip_img/tcp_17_2.pdf" width="70%"></center>
- 每个TCP连接必须包含**[源端口号、目的端口号]**，用于寻找发送端和接收端进程。源端IP、端口以及目的端IP、端口唯一确定了一条TCP连接。
- RFC793：将IP地址和端口号称为一个插口(socket)，后来socket也被作为伯克利编程插口。将一条TCP连接的双方称为一组插口对。
- **[32bit序号字段]**用于标识TCP发送端发送到接收端的字节流中的字节数(从发送端-\>接收端的第一个byte开始计数)。序号是32bit无符号数，序号到达2^32-1后再从0开始。
- 初始建立一个新的连接时，SYN标志设置为1，此时序号字段数值为主机选择的该连接的初始序号(initial sequence number, ISN)。SYN标志位需要消耗一个序号，因此正式发送的数据的第一个序号为ISN+1。
- **[32bit确认序号字段]**的含义是：发送确认端(数据接收端)所希望接受的下一个数据报的序号，即该序号是**"上一个已经被接受的数据报的序号"+"上一个被接受的数据部分长度"**。注意：仅当ACK标志为1时确认序号才有效，即仅当上一份数据分组被成功接受ACK才能继续下一封数据分组的发送。
- 发送ACK信号数据报无需任何依赖，直接设置即可。但是32bit确认序号是否有效需要ACK=1，因此一旦TCP连接建立，那么ACK标志位总是为1，32bit确认序号总是有效的。
- TCP连接支持应用层面的全双工通信(full-duplex)，即数据可以在两个方向上独立进行传输，但需要连接的每一端维护每个方向上的传输序号。
- **[首部长度字段]**给出TCP首部包含的32bit(4byte)的数目(普通首部字段+选项字段)。这个数值的作用是标识出选项字段的长度数值(选项字段的长度是可变的)。TCP首部长度字段占4bit(1111=1+2+4+8=15)，因此TCP首部最大长度为15×4byte=60byte，即选项部分最大长度为40byte。
- TCP首部中的**[6个标志位字段]**的简单用法如下：URG(紧急指针有效)、ACK(确认序号有效)、PSH(接收方应当尽快将此数据报交给应用层)、RST(重建连接)、SYN(同步序号以发起一个连接)、FIN(发送端完成发送任务)。
- **[紧急指针字段]**用于发送紧急数据的标志字段。URG=1，紧急指针有效。紧急指针的数值表示正向偏移量，该偏移量和序号字段中的数值相加表示所发送的紧急数据的最后一个字节相应的序号。
- **[最长报文大小(maximum segment size, MSS)]** 通常每个TCP连接端在建立通信的第一个报文段(设置SYN标志)指定MSS选项：用于向另一端通知自身能够接受的最大报文长度。
- **[关于TCP通信的窗口大小]** TCP连接传输的流量控制通过发送端和接收端声明各自的窗口大小来实现，以防止处理、传输数据较慢的一方缓冲区溢出。窗口大小按照byte计算。从确认序号作为窗口的左边沿，向右移动窗口大小的byte到达窗口的右边沿。窗口大小的是一个16bit的字段，因此最大窗口大小(max window size)为65535byte。窗口刻度选项(window scale option)可以允许max window size按照比例变化以提供更大的窗口。


### 四：小结
许多流行的应用：Telnet、Rlogin、FTP、SMTP都使用TCP作为通信方式。


## Reference
> \<tcp-ip: illustrated vol1\> chapter17 <br>
>
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
