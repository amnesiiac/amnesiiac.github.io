---
layout: post
title: "tcp-ip: TCP (保活定时器)"
subtitle: '[tcp-ip: illustrated vol1] - chapter23' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: true 
date: 2021-10-20 21:31
lang: ch 
catalog: true 
categories: tcp-ip 
tags:
  - Time 2021
  - tcp-ip illustrated vol1 
---
### 一：引言
一个空闲(idle)的TCP连接允许没有任何数据流动，即TCP连接的双方都没有向对面发送数据，则TCP连接端之间不交换任何信息。

**TCP协议中不存在轮询(polling)机制(包括长轮训、短轮询)，TCP使用连接的方式进行通信(长连接、短连接)** <br>
这意味着：在建立了客户端到服务端的连接后，可以使其空闲几小时、几天、几星期或者几个月，但TCP连接依然能够保持"UP"状态；甚至中间路由器可能发生崩溃重启，电话线路可能被挂断再连通，只要建立TCP连接的两端的主机没有重启，在该TCP连接依然保持"UP"状态。<br>

**TCP连接两端的应用进程没有使用"application level timer"来检测连接的"inactivity"状态(以关闭TCP连接两端任何一端的应用进程)** <br>
回顾chapter10中介绍的BGP相关内容，BGP每隔30s向对端发送一个应用级的探查(probe)，该探查就是一个独立于TCP保活定时器(keep-alive timer)之外的定时器。


**Host Requirements RFC提供了3个不实现TCP保活定时器的理由** <br>
- 在出现转瞬即逝的错误(transient errors)时，可能导致一个perfectly good connections dropped。
- 实现这种机制需要耗费不必要的带宽。
- 在按照数据分组计费的网络中，需要耗费更多的钱。

TCP中的保活定时器是一个非常有争议的功能，许多观点认为：如果需要这种类似功能，不应该在TCP层面提供，而是应该交给应用程序来完成。尽管如此，许多TCP实现都提供了保活定时器机制。

**为什么需要为TCP实现保活定时器** <br>
许多情况下，保活定时器功能主要是为服务器的应用程序提供的。服务器希望知道客户主机是否崩溃关机，或者崩溃又重新启动，从而能够代替客户端进程来对申请的资源进行"占用(tie up)"操作。此时需要在TCP实现中提供"keep-alive timer"来提供这种能力，许多版本的Telnet和Rlogin服务端默认使用这个选项。

**举例分析需要实现保活定时器的应用场景** <br>
当个人计算机用户登录到一个使用Telnet应用进程的服务器时，建立了TCP连接，此时该PC用户直接关闭其电源，没有注销掉和服务端Telnet应用进程之间的TCP连接，则会产生一个"half-open"连接。

[chapter18中关于half-open连接](http://localhost:4000/tcp-ip/2021/09/25/post-tcp-ip-tcp-chapter-18/#3-半打开连接的检测detecting-of-half-open-connections)曾经提到，客户端通过"half-open"连接发送数据，在服务端重新上线后会返回"reset"复位信号。

但是，如果客户端因为发生异常而消失，使得服务器上留下一个"half-open"的连接，客户端即使重新上线也不会发送"reset"复位信号，因此服务器进程将会永远等待下去。久而久之，导致服务器上遗留很多半打开的无用连接。TCP保活定时器机制就是为了检测到这种服务端的"half-open"连接。


### 二：保活定时器机制描述
在本部分内容中对保活定时器的描述中，只有服务端设置&使用保活定时器，TCP连接的对端则为客户端。

**保活定时器选项不强制使用** <br>
注意，并没有强制要求客户端不能使用该选项，但实际应用中都是服务端设置该功能(chapter26 Telnet和Rlogin中，只有服务端设置此选项)，如果连接两端都特别需要确认另一端主机的状态，则两端都可以设置这个选项(chapter29 NFS使用TCP时，客户端和服务端都需要设置此选项)。

**保活定时器机制在客户端4种状态下的具体应用&影响** <br>
如果一个给定的连接在2h时间内没有任何activity，则服务端就向客户端发送一个探查报文(probe segment)。
客户端必须处理一下4种状态之一：
- **状态#1**  客户端主机正常运行，并且从服务端可达。客户端的TCP能够正常响应，服务端也知道对端处于正常工作状态。服务端将在2h之后将保活定时器复位；如果在2h定时器expired之前，有应用程序的通信流量使用该TCP连接进行通信，则定时器在该通信交换数据后的未来2h再执行复位。此状态下，服务端应用进程不会知道服务端保活探查报文相关信息，因为保活探查属于TCP层次，除非客户端变成状态#2、#3、#4，保活侦查对于应用层都是透明的(不可见的)。
- **状态#2** 客户端主机已经崩溃，并且正处于"down"或者"rebooting(not rebooted)"的状态。无论客户端主机位于哪种状态，它的TCP都处于"no responding"状态。服务端将不能收到已发送的探查报文的回应。服务端总共发送10个探查报文，每份报文定时75s后超时。如果最终没有收到任何一个响应，则服务端判定客户端主机处于"down"状态，并中止与其的TCP连接。服务端应用进程将收到来自它的TCP的差错报告：e.g. 连接超时之类的信息(作为其应用进程"读请求"的应答)。
- **状态#3** 客户端主机崩溃并已经重新启动。此时客户端将对服务端发送的probe segment予以回应，此客户端返回的响应是一个复位信号，使服务端中止该连接。服务端应用进程将收到来自它的TCP的差错报告：连接被对端复位(作为其应用进程"读请求"的应答)。
- **状态#4** 客户端主机运行正常，但是从服务端不可达。TCP服务端处理方式同**状态#2**，因为TCP并不能区分出处于状态#2和状态#4的客户端主机的区别，它只能判断出是否收到已发送探查报文的响应。服务端应用进程将收到来自它的TCP的差错报告：e.g. 连接超时信息或者根据服务端TCP收到的其他ICMP差错返回相应差错(作为其应用进程"读请求"的应答)。


**服务端无需关注客户端被主动关闭&主动重启(主动指客户端操作员执行相应操作)** <br>
当客户端系统被操作员主动关闭&主动重启时，所有应用进程被中止，客户端会在中之前发送一个FIN信号。服务端收到FIN信号后，其TCP将发送"end-of-file"给服务端应用进程，使得服务端应用层能进行相应处理。

**保活定时器的2h空闲时间是否可以设置&修改** <br>
参考tcp-ip illustrated appendix E，这个数值可以修改。但是需要注意的是，保活定时器的空闲时间属于系统级的变量，改变它将会对所有使用该选项选项的用户。<br>
Host Requirements RFC提到：如果实现了保活定时器的功能，除非应用进程指明启用该功能，否则它被禁止使用。并且保活间隔必须可配置，但其默认值必须≥2h。

### 三：保活定时器应用案例
##### 1 状态#2 客户端主机已经崩溃，处于"down"或者"rebooting(not rebooted)"的状态，模拟步骤&分组交换分析如下：
- **(1)** 在客户端主机bsdi上运行sock程序和主机svr4上的标准回显服务进程建立TCP连接(使用"-K"选项启用保活功能)。
- **(2)** 进行通信，以验证建立TCP连接能够正常通过数据。
- **(3)** 观察客户端TCP每隔2h发送保活分组，并观察该分组被服务端TCP确认。
- **(4)** 将服务端以太网电缆拔掉，模拟server crash(直到本案例结束)，来使得客户端认为服务端主机已经崩溃。
- **(5)** 预期客户端判定连接中断之前，将发送10个间隔为75s的数据分组。

**(a) 步骤(1)&(2)：使用sock程序在客户端和服务端建立TCP连接** <br>
<center><img src="/img/in-post/tcp-ip_img/tcp_23_1.pdf" width="100%"></center>

**(b) 去掉建立TCP连接、和窗口通告报文分段的tcpdump程序输出**
<center><img src="/img/in-post/tcp-ip_img/tcp_23_2.pdf" width="100%"></center>

##### 2 状态#3 客户端主机已经崩溃并重新启动，模拟步骤&分组交换分析如下：
- **(1)** 在客户端主机bsdi上运行sock程序和主机svr4上的标准回显服务进程建立TCP连接(使用"-K"选项启用保活功能)。
- **(2)** 进行通信，以验证建立TCP连接能够正常通过数据。
- **(3)** 观察客户端TCP每隔2h发送保活分组，并观察该分组被服务端TCP确认。
- **(4)** 将服务端以太网电缆拔掉，模拟server crash，然后将服务器重启，然后再连接到以太网上。
- **(5)** 预期客户端发送的保活探查报文将导致服务端返回"reset"复位信号。

**(a) 步骤(1)&(2)：使用sock程序在客户端和服务端建立TCP连接** <br>
<center><img src="/img/in-post/tcp-ip_img/tcp_23_3.pdf" width="100%"></center>

**(b) 去掉建立TCP连接、窗口通告报文分段部分信息的tcpdump程序输出** <br>
<center><img src="/img/in-post/tcp-ip_img/tcp_23_4.pdf" width="100%"></center>

##### 3 状态#4 客户端主机运行正常，但从服务端不可达，模拟步骤&分组交换分析如下：
- **(1)** 在客户端slip主机上运行sock程序经过一个SLIP链路和网络上的主机vangogh.cs.berkelay.edu建立连接(使用"-K"选项启用保活功能)。
- **(2)** 进行通信，以验证建立TCP连接能够正常通过数据。
- **(3)** 观察客户端TCP每隔2h发送保活分组。
- **(4)** 在某个时刻断开客户端slip和服务端vangogh之间的SLIP链路。
- **(5)** 预期客户端发送的保活探查报文无法到达服务端，因此返回服务端不可达差错信息"no route to host"。

**(a) 步骤(1)&(2)：使用sock程序在客户端和服务端建立TCP连接** <br>
<center><img src="/img/in-post/tcp-ip_img/tcp_23_5.pdf" width="100%"></center>

**(b) 去掉建立TCP连接、窗口通告报文分段部分信息的tcpdump程序输出** <br>
<center><img src="/img/in-post/tcp-ip_img/tcp_23_6.pdf" width="100%"></center>


## Reference
> \<tcp-ip: illustrated vol1\> chapter23 <br>

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
