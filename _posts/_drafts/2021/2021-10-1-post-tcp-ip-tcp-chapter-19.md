---
layout: post
title: "tcp-ip: TCP (交互式数据流处理)"
subtitle: '[tcp-ip: illustrated vol1] - chapter19' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-10-01 09:40
lang: ch 
catalog: true 
categories: tcp-ip 
tags:
  - Time 2021
  - tcp-ip illustrated vol1 
---
### 一：引言
本部分内容主要介绍关于使用TCP进行数据传输的相关问题。

根据相关TCP通信量的研究显示：(i) 按照TCP数据报分组数量统计，一半的TCP报文segment为成块数据(如FTP、Email、Usenet news)，另一半的TCP报文segment包含交互数据(如telnet、Rlogin)；(ii) 按照TCP传输字节数统计，成块数据和交互式数据之间的占比约为90%:10%。

上述结果表明：成块的TCP数据(bulk data)基本都是"满长度"的TCP报文(占用512byte)，只有TCP数据传输的最后一个数据报是"没有填满"的TCP报文。然而用于交互的TCP数据(interactive data)通常是"没有填满"的，且占用的字节数要小很多。

TCP需要能够同时处理这两种数据，针对每种数据相应的处理算法也会不同。本文采用Rlogin程序为例来观察交互数据传输过程，并研究延迟确认技术(delayed acknowledgement)是如何工作的，以及Nagle算法如何较少网络中"小的数据分组碎片"的数量。

### 二：交互式输入
观察Rlogin程序建立的TCP连接上，键入一个交互命令所产生的数据流：

##### 1 Rlogin交互程序的一种处理远程按键回显的方式时序图
<center><img src="/img/in-post/tcp-ip_img/tcp_19_1.pdf" width="80%"></center>
- 一般可以将segment#2和segment#3合并：将按键确认ack信号和按键的回显数据使用同一报文发送；这种合并的技术称为延迟确认技术(delayed acknowledgement)。
- Rlogin程序每次从client到server只发送一个byte的数据，而telnet提供了一个选项，该选项允许client发送一行字符到server，从而一定程度上减少了网络负载。

##### 2 在bsdi主机上发送"date\n"到svr4主机上获得回显输出，用tcpdump分析其中的数据分组交换过程
<center><img src="/img/in-post/tcp-ip_img/tcp_19_2.pdf" width="100%"></center>
- 上面tcpdump打印的输出中省略了用于TCP建立连接的数据分组交换情况，并且省略了服务类型输出(TOS字段)。
- tcpdump第一行输出："P 0:1(1) ack 1"表示：数据输出 数据起始序号ISN:数据终止序号(报文segment数据部分长度) ack确认序号(确认之前用于TCP建立连接的数据分组)。
- tcpdump第二行输出：服务器将按键回显数据segment和接收的数据ack确认序号合并成一个数据报返回，称之为delayed ack(本文下一部分进行介绍)。

### 三：延迟确认技术(delayed acknowledgement)
<center><img src="/img/in-post/tcp-ip_img/tcp_19_3.pdf" width="100%"></center>
- 上图中7个从bsdi发送到svr4的ACK信号称为：延时发送的ACK信号。通常TCP在收到数据时并不是立即发送ACK，而是延迟一段时间后发送，以便将ACK和沿同一方向发送的数据报文合并发送。
- 绝大多数延时发送的ACK信号采用最大200ms延时，即TCP最多等待200ms以查看是否有数据能够和该ACK信号合并发送。
- 上图中7个经过延时发送的ACK信号由于等待了200ms后，仍然没有数据合并发送(所有能够合并的数据至少需要ACK信号等待超过200ms)，因此7个时延ACK信号都是单独发送的。
- 上述7个ACK信号的延时时间分别是：123.5ms、65.6ms、109.0ms、132.2ms、42.0ms、140.3ms、195.8ms，即ACK时延信号的发送延时时间是随机的(≤200ms)。
- 上述7个ACK信号的actual time分别是：139.9ms、539.3ms、940.1ms、1339.9ms、1739.9ms、1930.1ms、2140.1ms。很容易发现这些时间的间隔是200ms的整数倍。这是因为：TCP使用了200ms的timer，该timer以内核bootstrap时间为基点在没固定200ms后溢出(执行ACK信号发送操作)。svr4端返回bsdi的确认信号是随机到达的，因此TCP在内核timer确认信号到达后的下一次发生溢出时进行相应ACK确认信号的发送。
- Host Requirements RFC声明TCP需要实现这种delayed ack，但最大时延需≤500ms。

### 四：Nagle算法
Rlogin程序中的客户端一般会发送1byte数据到服务器上，因此会产生许多41byte的分组(20byteIP首部+20byteTCP首部+1byte数据)。

在局域网上，这些小数据分组通常不会带来麻烦，因为局域网一般不会出现拥塞。但是在广域网上，这些小数据分组会大大增加网络拥塞的可能。

采用RFC-896所建议的Nagle算法可以简单、高效的缓解上述问题。Nagle算法要求：TCP连接上至多只能有一个有未经确认的数据分组，即连接上只要有一个数据分组尚未确认，那么新的小数据分组就不能被发送；并且这些小数据分组被TCP收集，在上一个未被确认的数据分组确认ack信号到来时，将它们合并成一个分组发送出去。

这种算法是自适应的：对于高速网络上，数据分组的确认到达很快，因此新的数据发送的也越快；对于低速广域网，确认到达很慢，新的数据被"充分"合并，导致交换的数据分组数量也较少。

##### 1 使用时序图来分析Nagle算法在广域网中减少小数据分组的作用
通过分析本文"延时确认技术"中的时序图很容易发现：1byte数据的TCP数据在局域网上被发送、确认、回显的平均往返时间约为16ms。为了观察到Nagle算法在发送数据快于确认数据时进行合并小数据分组的操作，1秒钟至少键入&发送的字符个数要超过60个(1/60=0.016667s≈16ms)。通常1s种不太可能键入60个字符，因此局域网上通常用不到Nagle算法。

在广域网上，1byteTCP数据被发送、确认、回显的平均往返时间要慢得多。下面展示了在slip主机和vangogh.cs.berkelay.edu之间使用Rlogin程序快速的发送字符数据得到的时序图。

<center><img src="/img/in-post/tcp-ip_img/tcp_19_4.pdf" width="70%"></center>
- 比较上面用于测试Nagle算法的时序图以及"延时确认技术"测试的时序图，可以发现从slip到vangogh无法单独观测到"delayed acknowledgement"。这是因为，为了测试Nagle算法，字符键入&发送的速度非常快，因此没等到内核定时器溢出就有数据发送，所有"delayed ack"都和数据合并在一起传输。
- 上述测试Nagle算法时序图中：从slip到vangogh单方向发送的数据长度是变化的：1byte、1byte、2byte、1byte、2byte、2byte、3byte、1byte、3byte，这是因为：根据Nagle算法，client只有收到前一个数据分组确认信号才能发送目前已经"收集"的合并数据分组。另外，使用Nalge算法发送16byte数据需要使用9个数据报进行发送而不是16个。
- segment#12、segment#13、segment#14、segment#15看起来和之前的数据分组略有不同。根据"**确认序号表示下个将被发送的数据分组序号**"易知：#14包含了#12所"要求"的序号的数据+#13的确认应答；#13包含了#11所"要求"的序号的数据+#12的确认应答；#15包含了#14所"要求"的序号的数据+#13的确认应答。
- 上述时序图中包含一个"delayed ack"数据分组：segment#12。这是一个不带有任何数据的确认应答报文，虽然键入&传输字符非常快，但是服务器可能当时非常繁忙，因此在到达内核timer溢出时间时，尚未准备好下一份将要发送的数据，于是一封"不带数据的delayed ack"报文被server返回。
- segment#17客户端向server发送3byte数据，收到的应答只包含1byte确认，发送字符的回显并没有包含在返回的确认数据分组中。这表明TCP可以在应用程序进程读取&处理数据前发送所接收数据的确认ack信号。返回的是TCP层面的确认信号，它只表明server端TCP层已经正确接受了数据，但应用进程尚未作回应。另外，最后一个确认信号报文窗口大小为8189而不是8192，说明此时server端应用进程尚未读取TCP接收的3byte数据。

##### 2 关闭Nagle算法
有时需要关闭Nagle算法，例如X窗口系统服务，小数据分组(移动鼠标)此时必须尽可能低延时的传输，以便给予进行某些操作的用户以及时的反馈。

socket程序API的用户可以使用"TCP_NODELAY"选项来关闭Nagle算法。

Host Requirement RFC声明TCP必须支持Nagle算法，但必须为应用程序提供方法or接口以关闭该算法在某个连接上的执行。

##### 3 案例分析
在slip主机和vangogh.cs.berkelay.edu之间建立一个Rlogin应用程序交互连接。首先按下F1功能键，F1需要使用3byte字符进行表示("Escape"、"["、"M")。然后按下F2功能键将产生另外3byte字符表示("Escape"、"["、"N")。需要注意的是，"Escape"字符在回显时需要使用两个字符进行表示：ctrl转义字符"^" + "["。

**[0] F1、F2按键的字符表示 & 及其回显转移字符表示**
<center><img src="/img/in-post/tcp-ip_img/tcp_19_5.pdf" width="70%"></center>

**[1] 使用tcpdump程序对于2次按键的数据分组交换情况进行打印&分析**
<center><img src="/img/in-post/tcp-ip_img/tcp_19_6.pdf" width="100%"></center>

**[2] 上述按键程序的数据分组时序图**
<center><img src="/img/in-post/tcp-ip_img/tcp_19_7.pdf" width="100%"></center>

### 五：窗口大小通告 
本文前面介绍Nagle算法的[时序图](http://localhost:4000/tcp-ip/2021/10/01/post-tcp-ip-tcp-chapter-19/#1-使用时序图来分析nagle算法在广域网中减少小数据分组的作用)中，slip主机的大部分窗口都为4096byte，vangogh主机的大部分窗口都为8192byte。但是segment#5的窗口大小为4095byte，这表明：在TCP的缓冲区中，仍然有1byte数据等待应用程序进程读取。同理，segment#7的通知另一端的窗口大小为4094byte，表明：在TCP buffer中，有2byte数据等待应用程序进程读取。

### 小结 - to be done

## Reference
> \<tcp-ip: illustrated vol1\> chapter19 <br>

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用\`\`将关键字包含在内 <br>
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
