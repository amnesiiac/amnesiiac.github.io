---
layout: post
title: "tcp-ip: TCP (保持定时器)"
subtitle: '[tcp-ip: illustrated vol1] - chapter22' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: true 
date: 2021-10-18 17:06
lang: ch 
catalog: true 
categories: tcp-ip 
tags:
  - Time 2021
  - tcp-ip illustrated vol1 
---
### 一：引言
**接收端的窗口更新确认ack信号的作用** <br>
TCP可以通过接收端返回通告窗口来限制发送端数据流量，当通告窗口被设置为0时，相当于阻止发送端发送数据，直到窗口变为非0值为止。
通告窗口为0的情况可以在[chapter20中fast sender和slow receiver之间的数据通信时序图](http://localhost:4000/tcp-ip/2021/10/03/post-tcp-ip-tcp-chapter-20/#4-从fast-sender发送数据给slow-receriver的数据分组传输时序图)中找到相应情况：segment#8设置通告窗口为0，segment#9更新通告窗口为4096以允许后续数据传输。

**接收端使用确认ack信号更新通告窗口的机制可能出现的问题** <br>
确认ack信号分组的传输并不总是能成功的(即使它借助"可靠的"TCP协议进行传输)，TCP只确认那些包含数据的确认ack信号分组，而不会确认纯粹确认ack信号分组；因此TCP的实现必须考虑处理"用于打开通告窗口"的确认ack信号分组丢失的情况。

**"用于窗口通告更新"的ack报文分组的丢失处理机制的TCP实现是必要的** <br>
假如一个确认ack信号丢失了，则TCP连接的双方可能因为等待对方的回应而使得TCP连接传输中止：接收端等待发送端发送的数据(已经向发送端发送通告窗口更新报文，但是该报文在传输过程中丢失)，发送端在等待允许它能继续发送数据的通告窗口更新。如果TCP没有相应机制处理通告窗口确认ack信号丢失的情况，那么将产生死锁(deadlock)。

**保持定时器(persist timer)** <br>
TCP连接的发送端可以使用一个保持定时器来周期性的向接收端发送查询，以"主动地"发现接收端通告的窗口是否增大。从发送端发出的查询报文称为"窗口探查(window probe)"。

**内容梗概** <br>
本文主要讨论窗口探查机制以及TCP的保持定时器，并且还将讨论和persist timer相关的"糊涂窗口综合症"。


### 二：保持定时器案例分析
##### 1 使用sock程序构建"休眠"服务器
为了观察TCP实现中的保持定时器机制，在服务端启动一个接收进程，用于监听客户的连接请求。

具体地，接收进程首先接收客户端的连接请求，然后休眠很长一段时间后，再从网上读取数据。可以使用"sock -P"来使服务端在接收TCP连接和执行第一次读取动作之间进入休眠状态，下面的程序指定该休眠时间为100000s(27.8h)。
<center><img src="/img/in-post/tcp-ip_img/tcp_22_1.pdf" width="100%"></center>

客户端进程运行在bsdi主机上，向服务端5555端口执行1024byte的写操作。

##### 2 使用tcpdump程序对于案例数据分组交换情况进行打印&分析
<center><img src="/img/in-post/tcp-ip_img/tcp_22_2.pdf" width="100%"></center>
- 从客户端bsdi主机向服务端svr4主机执行1024byte写操作，tcpdump输出如上图所示(已经省略掉建立TCP连接的过程)。
- segment#1~segment#13数据分组交换情况展示了：客户端bsdi主机和服务端svr4主机的正常数据传输过程。共发送了9216byte的数据，确认了9216byte数据。服务端通告窗口大小为4096byte，默认的socket缓存大小为4096byte，但是实际上接收了9216byte的数据，**这是svr4主机的TCP实现代码和流子系统之间某种形式交互的结果**。???problem😫problem??? 没搞懂什么交互原理
- segment#13中，服务端确认了前面发送的4个数据分组，然后通告窗口为0，以使客户端停止发送其他数据。
- segment#13报文导致客户端设置保持定时器(persist timer)，如果保持定时器expired后，仍然没有收到服务端窗口更新报文，客户端就发送探查数据报，以确认服务端窗口更新报文是否丢失。本案例中，服务端进程处于休眠状态，因此TCP将会缓存所有收到的9216byte，以等待进程读取，所以，案例中窗口更新报文不会丢失，而是总是返回通告窗口为0的报文。
- **观察每次客户端重发的"窗口探查"报文的时间间隔**。在收到通告窗口大小为0的报文后(segment#13)，第一个探查报文(segment#14)的间隔为4.949s，下一个探查报文(segment#16)的间隔是4.996s，随后的几个探查报文间隔分别是：5.996s、11.996s、23.997s、47.997s、59.998s。这些报文间隔总是比整数秒数：6、12、24、48、60少一点，这是因为：TCP的探查报文的重发被TCP的500ms超时进程所控制，每次当超时进程定时器expired，即发送窗口探查，在大约4ms后收到应答，每次收到应答都将tcpdump程序的"报文间隔计时器timer"重启，因此下一个TCP超时进程的tick被tcpdump标注的时间将为"500ms-4ms"。
- **保持定时器窗口探查重发时间的设置** 保持定时器的窗口探查重发时间间隔应用了TCP的指数退避时间(首次超时时间为1.5s，第二次超时时间增加1倍为3s，第三次再增加一倍为6s...)，唯一不同的是，保持定时器的时间间隔总是设置在[5,60]的区间范围内(入上图tcpdump输出所示)。
- **窗口探查报文只包含1byte的数据** 注意，窗口探查报文分组后面的确认ack信号并不是对窗口探查发送的1byte数据的确认ack，这个确认ack针对的是包含序号为9216在内的所有数据。因此，由于窗口探查报文不能得到确认，因此它总是被重传，直到服务端发送一个窗口更新分组&对于保持定时期间所有窗口探查数据进行确认应答。
- **窗口探查报文和chapter21中的超时处理的区别** 不同于TCP超时处理(在一定时间后将放弃超时重传)，TCP会一直进行窗口探查的发送，经过指数退避后每隔60s重新发送一次，直到客户端收到窗口更新or客户端应用进程使用的连接被中止。

##### 3 TCP超时进程定时器的tick和tcpdump程序中计算分组间隔计时器之间的计算关系(为什么tcpdump输出的间隔总是相比指数避退时间少)
<center><img src="/img/in-post/tcp-ip_img/tcp_22_3.pdf" width="100%"></center>


### 三：糊涂窗口综合症(silly window syndrome)
之前介绍的基于窗口的发送、接收流量控制机制，一定程度上会导致一种称为"糊涂窗口综合症(silly window syndrome, SWS)"的状况。当SWS发生时，只有small amounts of data通过TCP连接进行交换，而不是full-sized segments。

**SWS的现象可以发生在TCP连接的任何一端**<br>
此时接收端**总是**通告很小的通告窗口，而不是等待缓存充分释放，允许通知一个较大窗口时才进行通告；此时发送端总是发送很少数量的数据，而不是等其他数据准备好一次性发送一个较大的报文段。

**可以在TCP连接的任何一端采取相应措施避免出现SWS现象** <br>
- 接收端不通告小窗口。The normal algorithm: 接收端从不通告一个比当前的通告窗口更大的窗口(当前通告窗口可以为0)，除非将要通告的窗口=当前通告窗口+1MSS或者将要通告的窗口=当前窗口+接收方缓存空间的1/2(不论实际能够增加多少)。即限制接收端不能随意的发送新的通告窗口，而是只有当满足一定条件时，才能进行通告。
- 发送端只有满足下列条件之一时才能发送数据分组：**(a)** 可以发送一个full-sized segment；**(b)** 可以发送至少为接收方通告窗口一半大小的报文分段；**(c)** 如果当前不处于"expecting an ack"的状态(当前没有尚未确认的发送数据)，或者该连接上已经禁用了Nagle算法(chapter19)，可以发送everything we have。
- 发送端(b)条款主要限制那些总是通告小窗口(\<1segment)的主机；发送端(c)条款限制了：在有尚未被确认的数据时以及不能使用Nagle算法的条件下，不发送小的报文分组。如果应用进程在执行小数据的写操作，条款(c)可以避免出现SWS。

##### 1 糊涂窗口综合症案例
本部分通过一个详细的案例来观察实际数据传输过程中出现糊涂窗口综合症的情况，该案例同时也包含了保持定时器。

**(a)** 在sun主机上运行sock程序，向网络中的bsdi主机写6个1024byte的数据，sun主机运行的sock程序如下所示：
<center><img src="/img/in-post/tcp-ip_img/tcp_22_4.pdf" width="100%"></center>
**(b)** 在主机bsdi接收数据时加入一些pauses：第一次读取数据时暂停4s，之后每次读取数据前暂停2s。接收端bsdi主机每次执行256byte的读取操作。
<center><img src="/img/in-post/tcp-ip_img/tcp_22_5.pdf" width="100%"></center>
- 接收端应用进程第一次读取数据前的暂停是为了让接收端缓存被填满，迫使数据发送端停止数据发送。
- 随后接收端从网络中进行小数据量的读取操作(256byte)，我们预期能够观察到接收端为避免发生糊涂窗口综合症而采取的措施。

##### 2 发送端、接收端避免糊涂窗口综合症机制的案例数据分组交换时序图
<center><img src="/img/in-post/tcp-ip_img/tcp_22_6.pdf" width="100%"></center>

##### 3 发送端、接收端避免糊涂窗口综合症的案例数据分组交换事件序列表
<center><img src="/img/in-post/tcp-ip_img/tcp_22_7.pdf" width="80%"></center>


## Reference
> \<tcp-ip: illustrated vol1\> chapter22 <br>

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用\`\`将关键字包含在内 <br>
> 8 `<center><img src="/img/in-post/economics_4/xxx.png" width="60%"></center>` <br>
> 9 使用html设置图片文字环绕方式：<br>
    `<div>` <br>
        `<img src="/img_path" align="left" width="40%" hspace="" vspace=""/>` <br>
        `<p>paragraph1 around the picture</p>` <br>
        `<p>paragraph2 around the picture</p>` <br>
        `<p>paragraph3 around the picture</p>` <br>
    `</div>` <br>
> 10 `<font style="color:red; font-weight:bold">加粗蓝色</font>`用来设置字体颜色 <br>
> 11 使用html设置可折叠部分内容：<br>
  `<details>` <br>
      `<summary><b>[点击展开] xxx</b></summary>` <br>
      `<center><img src="/img/in-post/tcp-ip_img/tcp_20_9.pdf" width="100%"></center>` <br>
  `</details>` <br>
> 12 问题脚注: ???problem😫problem???
