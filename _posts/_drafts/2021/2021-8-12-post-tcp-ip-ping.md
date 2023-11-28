---
layout: post
title: "tcp-ip: Ping (计算机网络中的声纳)"
subtitle: '[tcp-ip: illustrated vol1] - chapter7' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-08-12 16:25
lang: ch 
catalog: true 
categories: tcp-ip 
tags:
  - Time 2021
  - tcp-ip illustrated vol1 
  - problem
---
### 引言
**1 ping程序的目的是为了测试另一台主机是否可达**。该程序发送一份ICMP回显请求报文给目的主机，并等待ICMP回显应答报文(可查看chapter6中的ICMP类型表)。一般来说，如果不能ping到某台主机，那么就不能通过telnet、FTP连接到那台主机；反之，如果不能通过telnet、FTP连接到某台主机，也可以用ping程序来定位问题。<br>
随着网络安全意识增强，出现了提供访问控制清单的路由器，以及防火墙，因此两台主机之间是否可达，不单一取决于IP层是否可达，还取决于使用的通信协议以及端口号。两台主机之间不能ping通，但是也可以用telnet远程登录到目的主机25号端口(邮件服务端口)。<br>
**2 ping程序可以测出源主机到目的主机的往返时间**，以判断另一台主机距离本主机有多远。<br>
**3 ping程序可以检测IP记录路由、时间戳选项**。

本文主要使用ping程序作为诊断工具对ICMP进行深入剖析。

### Ping程序
发送ICMP回显请求的ping程序为客户，被ping的主机成为服务器。大多数的TCP/IP实现都在内核中支持ping服务器，这种服务器不是一种用户进程，而是由内核提供支持。

##### 一：ICMP回显请求&应答报文格式
同其它类型的ICMP查询报文一样，ICMP回显请求必须echo请求报文中的标识符、序列号。另外，必须将客户发送的选项数据echo回来(选项字段附属了很多功能)。
<center><img src="/img/in-post/tcp-ip_img/ping_1.pdf" width="60%"></center>

##### 二：LAN(local area network)上的输出
在LAN局域网(网络结构图参考chapter1章节)上运行ping程序，得到的输出一般格式如下：
<center><img src="/img/in-post/tcp-ip_img/ping_2.pdf" width="100%"></center>

在bsdi(BSD/386-0.9.4版系统)主机上使用tcpdump命令，打印其发包的情况如下：
<center><img src="/img/in-post/tcp-ip_img/ping_3.pdf" width="100%"></center>

在sun主机上使用ping程序打印ICMP echo-reply的情况如下图，由于第一次发送ICMP echo-request之前需要进行ARP请求发送、ARP应答接受，因此第一次RTT(round-tritime)相比后面的时间要长：
<center><img src="/img/in-post/tcp-ip_img/ping_4.pdf" width="100%"></center>

在新版系统的bsdi(BSD/386-1.0版系统)上使用ping程序打印ICMP echo-reply，可以看到支持us级分辨率的系统(其ping程序同样支持us级分辨率)打印的ping程序输出的结果：
<center><img src="/img/in-post/tcp-ip_img/ping_5.pdf" width="100%"></center>

##### 三：WAN(wide area network)上的输出
在广域网上使用ping程序查看ICMP echo-reply的输出，结果相比LAN的输出有很大不同。下面ping程序的结果是Internet具有正常通信量的ICMP echo-reply结果。
<center><img src="/img/in-post/tcp-ip_img/ping_6.pdf" width="100%"></center>

##### 四：SLIP(serial line internet protocol)线路上的输出
SLIP串行线路上的数据通常以低速异步的方式进行传输，如9600bit/s或者更低。因此需要对这种特殊传输方式进行研究。本案例使用introduction章节中默认主机网络结构，并选取bsdi主机到slip主机这一段SLIP线路进行研究(bsdi、slip主机传输速率被设置为1200bit/s)。<br>
更多关于同步传输、异步传输的例子可以参考[博客](http://localhost:4000/tcp-ip/2021/08/13/post-tcp-ip-asynchronous-synchronous-transmission/)中的内容。

**理论计算RTT数值** <br> 
我们可以根据串行线路吞吐量计算的知识来对于bsdi主机-\>slip主机这段串行线路的往返时间进行推算：<br>
下图绘制了承载了ICMP回显请求&应答的IP数据报的格式，其中根据前面ping实验得知，ICMP报文数据部分长度为56byte。
<center><img src="/img/in-post/tcp-ip_img/ping_7.pdf" width="70%"></center>
IP数据报在SLIP线路上进行传输时，需要进行SLIP的特殊封装，参考chapter2-link-layer部分内容可知，需要至少增加额外2bit，分别为开头结尾的END字符。<br>
线路上的传输速率为1200bit/s，每个byte被封装后为8bit+2END=10bit。因此以byte为单位的传输速率为120byte/s，即1byte/8.33ms。<br>
因此，我们理论估计传输84byte的往返时间为：84byte × 8.33ms/byte × 2 = 1433ms。

**通过在svr4主机上运行ping查看实际(round-tritime)数值** <br> 
<center><img src="/img/in-post/tcp-ip_img/ping_8.pdf" width="100%"></center>

##### ??????problem😫problem(1)?????? 
为了测量在SLIP线路上ping程序的RTT时间，为什么要使用svr4主机发送ICMP回显数据报?不应该是bsdi主机向slip主机发送回显数据报么?还是说bsdi和slip直接通过SLIP链接，传输时间太短了无法和理论相比较?

##### 五：拨号SLIP(serial line internet protocol)线路上的输出
相比单纯SLIP线路，拨号SLIP线路上运行ping程序的RTT时间估计更加复杂。<br>

**案例基础设施** <br>
本案例使用chapter1中通用主机网络结构中sun主机-\>netb这段拨号SLIP线路进行测试。其中sun-\>netb之间使用调制调节器采用V.32调制方式(传输速率9600bit/s)；采用V.42(别名LAP-M)错误控制方式；采用V.42bis数据压缩方式。

**拨号SLIP线路中影响RTT时间**：<br>
**1** 架设在拨号SLIP线路上的调制解调器会带来时延；**2** 调制解调器中提供的数据压缩技术会改变IP数据报的长度；**3** 使用了错误控制协议，因此IP数据报长度有可能会增加；**4** 接收端调制调节器只能在完成校验和验证后，才释放收到的数据；**5** 对于接收端、发送端串行通信接口，许多系统只能在固定的时间间隔、或预先收到一些字符后才读取这些接口。
<center><img src="/img/in-post/tcp-ip_img/ping_9.pdf" width="100%"></center>

### IP记录路由选项 - 记录每个处理过该数据报路由器地址
大多数不同版本的ping程序都提供\<-R\>选项，以提供记录路由的功能。<br>

##### 一：ping -R程序记录路由的原理
**[1] ping程序打印IP路由地址清单的原理** <br>
**1** 在发送的包含ICMP回显请求报文的IP数据报中填写IP-RR选项；**2** 每个处理这份数据报的路由器都把它的IP地址写入IP数据报的选项字段中；**3** 当IP数据包到达目的端时，IP地址清单(存于IP首部选项字段)从ICMP回显请求数据报被**复制**到ICMP回显应答数据报中；**4** ping程序收到ICMP回显应答，即打印出IP地址清单。<br>
上述流程中，大多数处理都涉及IP数据报的选项功能。大多数系统都支持选项功能，只是有少数系统不将IP地址清单复制到ICMP回显应答中。

**[2] IP地址清单最多能包含IP地址的个数** <br>
**1** IP首部长度字段占用4bit，因此IP数据报的首部最多包含15个32bit字段(15×32/8=60byte)。**2** IP首部的固定部分占用20byte，RR选项占用3byte，因此只剩下37byteIP选项字段用于存放IP地址。**3** 每个Ipv4占用32bit=4byte，因此IP地址清单最多能包含9个IP地址。

**[3] IP-RR选项数据格式简图** <br>
<center><img src="/img/in-post/tcp-ip_img/ping_10.pdf" width="80%"></center>

##### 二：一个使用IP-RR选项运行ping程序的案例
在svr4主机上运行ping程序到slip主机，经过的bsdi作为中间路由器负责处理这个数据报。

**[1] 案例涉及的网络结构图如下(截取自chapter1通用主机网络模型)**
<center><img src="/img/in-post/tcp-ip_img/ping_11.pdf" width="80%"></center>

**[2] ICMP回显请求&应答数据报在使用\<-R\>选项后的地址记录流程**
<center><img src="/img/in-post/tcp-ip_img/ping_12.pdf" width="100%"></center>

**[3] 应用\<-R\>选项的ping程序的实际输出示例**
<center><img src="/img/in-post/tcp-ip_img/ping_13.pdf" width="100%"></center>

**[4] 应用\<tcpdump -v\>选项监测(watch)抓取的数据包的详细头文件**
<center><img src="/img/in-post/tcp-ip_img/ping_14.pdf" width="100%"></center>

##### 三：初探IP选路以及ICMP数据报重定向(案例分析)
**The case:** <br>
使用chapter1通用主机网络结构，从slip主机上运行ping程序到aix主机，打印输出结果如下：
<center><img src="/img/in-post/tcp-ip_img/ping_15.pdf" width="100%"></center>

**The doubt:** <br> 在ping程序打印的ICMP回显应答数据报中记录的IP路由中，gateway路由器也被记录，这说明在IP数据报传输的过程中经过了gateway。结合下面主机网络图，很容易产生疑问，为什么aix主机要将ICMP回显应答传输给gateway而不是直接返回给netb路由器呢？

<center><img src="/img/in-post/tcp-ip_img/ping_16.pdf" width="66%"></center>

**The answer:** <br>
aix主机在返回ICMP回显应答时，没有将数据报返回给netb的原因是：它不知道要将数据报返回给netb主机。<br>
要解释上述问题涉及到IP选路的特点：为ICMP回显应答选路时，aix参考路由表中的默认表项，进行IP路由选择。当不能明确哪个主机作为"next-hop-router"时，选择默认路由进行数据转发。关于路由选择时的查表步骤参考[博客](http://localhost:4000/tcp-ip/2021/08/05/post-tcp-ip-internet-protocol/#ip路由选择)中的内容。<br>
aix的默认路由为gateway，而不是netb。使用gateway作为默认路由的原因是：路由器gateway相比子网140.252.1的任何主机都有更强的选路能力：整个子网140.252.1的主机默认路由表项都指向gateway，这样就无须在子网的每台主机上运行选路守护程序。

**Go deep in this question (ICMP数据报重定向) - ???problem(2)???** <br>
**为什么gateway路由器不发送ICMP报文给aix主机，以更新aix的路由表？** 某些原因这种ICMP重定向数据报并没有产生：原因可能是：gateway的产生重定向报文是一封ICMP回显请求报文???why😫???。如果使用Telnet程序调用远端aix主机daytime服务程序(telent aix daytime)，则ICMP会进行重定向，因此aix主机的路由表随之更新。当我们再次运行带路由选项的ping程序时，ICMP回显报文不再会经过gateway(see chapter9.5 for more details)。


### IP时间戳选项
##### IP时间戳选项的一般格式(需要填充在IP数据报的选项字段)
<center><img src="/img/in-post/tcp-ip_img/ping_17.pdf" width="80%"></center>

##### IP时间戳选项的FL标志字段取值以及相应时间戳标志字段的意义(作用)
<center><img src="/img/in-post/tcp-ip_img/ping_18.pdf" width="66%"></center>

##### 关于IP时间戳功能的评价
**1** 只记录时间戳是没有用处的(FL=0)，因为我们没有标明时间戳与路由器之间的对应关系(除非网络的拓扑关系是固定且已知的)。<br>
**2** 使用标志位FL=3较好，因为我们可以指定插入时间戳的路由器。<br>
**3** 应用IP时间戳功能的基本问题是：无法控制任何给定路由器上的时间戳的正确性。因此即使使用FL=3进行时间戳插入，也不一定能正确计算路由时间(time for hops between routers)。<br>
**4** chapter8提供了traceroute程序能够较好计算这种路由跳站时间。

### 小结
ping程序是对两个TCP/IP系统连通性进行测试的工具。它只需要利用ICMP回显请求&应答数据报进行通信，而无需通过传输层TCP/UDP。ping服务程序属于内核ICMP层的一种实现。


## Reference
> \<tcp-ip: illustrated vol1\> chapter7 <br>

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
