---
layout: post
title: "tcp-ip: Telnet & Rlogin (远程登录协议)"
subtitle: '[tcp-ip: illustrated vol1] - chapter26' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: true 
date: 2021-11-08 10:55
lang: ch 
catalog: true 
categories: tcp-ip 
tags:
  - Time 2021
  - tcp-ip illustrated vol1 
---
### 一：引言
远程登录(remote login)是internet上最广泛的应用之一。借助远程登录技术，我们可以通过先登录到一台主机上，然后通过internet登录到其他主机上去，并不需要在每台主机上都通过hardwired的方式建立一个通信终端(老式计算机之间的远程登录方式)。

在TCP/IP网络上，有两种应用提供了远程登录功能：
- ➀ Telnet：标准的提供远程登录功能的应用，几乎每个TCP/IP的实现都提供这个功能。支持运行在不同操作系统的主机之间，Telnet通过client和server之间的选项协商机制(option negotiation)，从而确定通信双方可以提供的特性。
- ➁ Rlogin：起源于伯克利版本的unix系统。最初只能运行在unix各版本之间，现在也可以在其他操作系统上运行。

**❮本章内容概述❯** <br>
本文主要介绍Telnet和Rlogin。首先将介绍Rlogin，因为Rlogin机制较为简单。然后介绍Telnet，下面对Telnet进行简单介绍：

**❮Telnet(telecommunication network protocol)的client和server之间的典型连接图❯** <br>
<center><img src="/img/in-post/tcp-ip_img/telnet_rlogin_1.pdf" width="70%"></center>
- ➀ Telent client同时和user at a terminal以及TCP/IP协议模块进行交互：通常情况下，user键入的信息的传输都通过TCP/IP连接，TCP/IP连接返回的任何信息都输出到terminal上。
- ➁ Telnet server经常会和pseudo-terminal打交道，至少在unix系统是这样。该机制使得在server端login shell进程、以及运行在login shell上的进程"认为"它们在直接和一个terminal交互。实际上，使server端的login shell"认为"它在和一个terminal交互是编写远程登录服务器软件最难的部分之一。
- ➂ 只使用一条TCP连接。由于在某些情况下，Telnet client必须和Telnet server进行命令传输、数据交互，因此需要某种方式来对TCP连接上传输的命令、用户数据进行描述(delineate)/区分。在本文后续部分进行介绍。
- ➃ 上图中虚线框中的部分包括：terminal、pseudo-terminal以及TCP/IP的实现部分，它们都属于系统内核部分的内容。虚线框外的Telnet server/client部分属于user applications。
- ➄ 上图中将server端的login shell部分单独绘制，以重申(reiterate)我们必须拥有server端的系统账户并登录到server上，才能完成上述远程控制操作。登录方式可以为Telnet或者Rlogin。

**❮Telnet & Rlogin client端、server端源码行数对比图表❯** <br>
<center><img src="/img/in-post/tcp-ip_img/telnet_rlogin_2.pdf" width="80%"></center>

远程登录不是具有high-volume data传输的应用，相反通常情况下有许多小数据分组在client & server两端之间进行交换。[Paxson 1993]给出了client发送到server的数据(bytes)和从server端返回的数据(bytes)之间的比例大约为1:20；即user通常键入很短的命令，server通常返回较多的字符。


### 二：Rlogin协议
Rlogin第一次发布是在4.2BSD中，且只支持unix系统之间的远程登录，由于应用两端都知道对端的操作系统类型，不需要实现选项协商机制(option negotiation)。后来，Rlogin也派生出支持非unix环境的版本。

RFC-1282[Kantor 1991]详细说明了Rlogin协议。[Stevens 1990]-chapter15介绍了远程登录的client端进程、server端进程的编写，并给出了源代码。

##### 1 应用进程的启动(application startup)
Rlogin应用程序在client和server之间使用单个TCP连接。在该TCP连接建立完成后，client和server之间根据application protocol将执行如下操作：
- ➀ client向server写入4个字符串(string)：**(i)** a byte of 0; **(ii)** 用户在client host上的登录名，以a byte of 0结尾; **(iii)** 用户在server host上的登录名，以a byte of 0结尾; **(iv)** 用户terminal类型名+斜杠(slash)+终端速率(terminal speed)+a byte of 0。❮对于为什么需要上述字符串进行说明❯：同时需要client/server host的登录名，因为两者可能会不一样。需要terminal类型名，因为server端全屏应用程序可能需要此参数。需要terminal speed，因为一些应用根据终端速率不同执行不同的操作流程：e.g. vi编辑器在终端速率较低时，其窗口也会变小(减少窗口redraw的时间)。
- ➁ server向client返回a byte of 0。
- ➂ server有一个请求user键入password的选项。用户发送的password采用普通数据的方式被传送，没有针对它的特殊协议。server向client发送一个string(通常是"Password:")，如果在指定时间内(通常为60s)没有键入password，server执行主动关闭该连接。❮避免每次登录输入password的方法❯ 我们可以在server端根目录下创建一个文件(将其命名为".rhosts")，文件中包含hostname和username。如果我们使用指定的host并以指定的username进行登录，则不需要键入密码。由于这种免密Rlogin登录存在安全漏洞，不推荐使用。❮password采用cleartext的方式传输❯ cleartext方式即：password中每个字符在传输中不会被加密、改变；因此从raw network packets中的数据部分可以直接读取我们发送的password内容。较新版本的系统(e.g. 4.4BSD)实现的Rlogin，使用Kerberos来提高登录的安全性。
- ➃ server通常向client发送请求，以询问terminal窗口的大小。

Rlogin应用程序每次发送1byte数据到server，server端返回应答数据。在数据发送时，启用了Nagle算法：在传输速率较慢的网络上，将多个输入byte合并成单个TCP数据报进行发送。client端用户键入的内容全部被发送到server上，server返回的应答信息将会被回显在client端terminal上。

另外，client和server之间可以互相发送指令，在介绍这些指令前，首先介绍应用这些指令的几种场景。

##### 2 流量控制(flow control)
默认情况下，流量控制是由Rlogin client端完成的：client端可以识别user键入的STOP(control+Q)或者START(control+S)字符，并停止or启动terminal输出。

每次我们键入control+S来中止terminal输出时，control+S字符将会通过网络发送到server，使得server停止写操作。但是注意到此时server写入网络的内容可能已经将信道填满(取决于window size)，这部分内容在client端输出停止前仍然会显示在terminal中(可能有hundreds or thousands of bytes of data)。

**❮Rlogin连接中通信管道中的已发送尚未接收的数据❯** <br>
<center><img src="/img/in-post/tcp-ip_img/telnet_rlogin_3.pdf" width="80%"></center>
对于交互方式使用Rlogin的user而言，对于user键入control+S的这种"延迟反应"是非常不好的。

**❮client端如何区分server端返回的字符为普通回显字符or流控制信息❯** <br>
有时，server端运行的应用进程需要对于进程输入字符进行逐byte进行interpret，但是又不希望client端获得关于server端应用进程的输入字符信息(通过server回显)，并将这些字符中的字符作为流控制信息(control+Q、control+S)来处理。为了解决这个问题，server端需要通知client端何时执行流控制，何时只是回显server返回的字符。

##### 3 客户中断键(client interrupt)
当我们为了中断一个在server端运行的进程而键入一个中断字符时(通常是DELETE或CONTROL-C)，会导致和上一节流量控制提到的相同的问题：当server端向client发送大量数据时，client端键入中断按键希望server端进程尽快中止，并尽快停止在client上的输出；但通常期待的效果会"延迟反应"。

**❮客户中断键、流量控制是否必须使用TCP紧急模式❯** <br>
在流量控制机制很少中止从client端到server端的数据流，因为该传输方向只包含user键入的字符。因此从client发送到server的特殊字符(e.g. CONTROL-S或interrupt)的传输，it's not necessary to use TCP[紧急模式](http://localhost:4000/tcp-ip/2021/10/03/post-tcp-ip-tcp-chapter-20/#八紧急模式urgency-mode)。

##### 4 窗口大小改变(window size changes)
通过将application运行在窗口化的显示框内(windowed display)，可以在application运行同时改变窗口大小。一些application需要获取到这些窗口大小改变的信息，尤其是一些full-screen editor获取窗口尺寸信息以调整自己输出格式。目前大多数unix系统提供了这种功能：通知application窗口大小的变化。

对于本文远程登录这个application而言，窗口大小的变动发生在client，而运行在server端的进程同样需要被告知窗口大小的变动(以调整自身运行方式)。因此Rlogin client端需要使用某些方式来通知server端窗口大小是否变动、新窗口大小。

##### 5 从server=>client的命令
本节内容将介绍Rlogin应用程序从server通过TCP/IP连接发送到client的4条命令。

**❮如何区分server=>client发送的内容是command还是normal data❯** <br>
从server端向client返回命令的过程中的问题核心是：传输过程只建立一条TCP连接，该连接需要承担命令传输、数据传输两部分职责。因此server端需要将返回client的命令"做上标记"，以使client能够区分数据和命令，并正确的interpret命令而不是将它们作为普通数据回显在terminal中。标记方式为TCP紧急模式(urgent mode)。

**❮server=>client发送命令后，server端和client端处理过程❯** <br>
server向client发送一条command后，server进入紧急模式，紧急数据的最后一个byte是command byte。当client端收到了紧急模式通知后，从TCP/IP连接读取数据并保存这些数据直到遇到command byte。client端接受并保存的数据可能在terminal中显示、被丢弃，具体执行的操作取决于执行的command类型。

**❮Rlogin命令表：server=>client❯** <br>
<center><img src="/img/in-post/tcp-ip_img/telnet_rlogin_4.pdf" width="80%"></center>
左边是1byte命令数据，右边是client在收到server发送的"紧急数据"(command)后的响应情况描述。

**❮为什么从server=>client的数据需要窗口流控制机制❯** <br>
因为运行在server上的进程产生输出的速度通常要比client端terminal显示数据的速度要快，所以需要使用流控制机制来限制从server=\>client的数据流。Conversely，从client=\>server的数据很少被流控制中止，因为这个方向的数据包含了user键入的字符。

**❮为什么需要使用urgent mode发送server=>client命令❯** <br>
根据[chapter-20 紧急模式案例](http://localhost:4000/tcp-ip/2021/10/03/post-tcp-ip-tcp-chapter-20/#1-紧急模式案例)可知：即使接收端窗口为0，包含紧急数据的通告数据分组仍然能够正常传输。将上述4条命令通过TCP紧急模式发送的原因是：即使从server=\>client的数据流被TCP窗口流控制机制中止(e.g.接收端缓存buffer已满，因此通告发送端win=0)，command#1(flush output)仍然需要被发送到client端，此时只有通过紧急数据能够通过连接。command#2~4不是time critical的命令，但为了实现简单起见它们同样按urgent mode发送。

##### 6 从client=>server的命令
从client到server端只定义了一条命令：发送当前terminal窗口大小。当且仅当client收到server发送的0x80(请求窗口大小)，才会执行此命令。

**❮如何区分client=>server发送的内容是command还是normal data❯** <br>
区分从client发来的内容属于command还是normal data很重要：可以避免server端将client发送的command当作normal data而被传输到server端应用进程中。client端采用给command数据做标记的方式帮助server进行内容区分：2byte "0xff" + 2byte "special flag"。

**❮关于当前terminal窗口大小的表示方式：two special flag + four 16-bit数值❯** <br>
两个flag bytes都是ASCII 字符"s"。两个flag bytes后有4个16-bit数值：**[**行数(e.g. 25)，每行包含的字符数(e.g. 80)，X方向上的像素数，Y方向上的像素数**]**。通常情况下，Rlogin server端调用的应用进程使用以字符(character)为单位来度量terminal窗口尺寸，因此后2个16-bit数值设置为0。

**❮关于in-band signaling的概念及其潜在问题❯** <br>
上述描述的从client=\>server之间的命令传输形式称为"in-band signaling"，即command bytes在normal data数据流中传输；normal data也被称为"in-band data"。<br>
使用字符0xff来指示"in-band command"是因为：正常client端user输入不太可能产生0xff字符的normal data，因此用于区分normal data和command。<br>
但是这个方式并不完美：如果client端的输入产生了2byte 0xff字符，紧跟的字符恰好为2个ASCII字符s，则后续的8bytes数据将会被interpret成client端窗口大小信息。

Rlogin远程登录程序中从server=\>client的命令姑且称之为"out-of-band signaling"，因为大多数应用程序针对较重要的数据采用"out-of-band"方式传输而得名。 <br>
但是chapter20关于TCP紧急模式的讨论曾提到：TCP在紧急模式下传输的数据不属于"out-of-band data"，而是将command放在normal data数据流中，通过urgent pointer来指示urgent data所在的精确位置。因此上述称呼不够准确。<br>

**❮关于重要数据(e.g. comamnd)的传输方式选择问题：in-band signaling/out-band signaling❯** <br>
从client=\>server传输command的方式使用"in-band signaling"，因此server端必须逐byte检查从client端接受的数据，寻找2bytes的0xff字符(command标识符)。<br>
从server=\>client传输command的方式使用"out-band signaling"，client端无须对收到的数据进行逐byte检查，除非server端进入紧急模式(发送了包含紧急数据的分组)。即使在紧急模式下，client端之需要检查urgent pointer指向的byte即可。

从client=\>server传输的数据量(bytes)和从server=\>client传输的数据量(bytes)之间的比例大约为1:20。因此client=\>server方向上传输数据需要使用"in-band signaling"方式，小数据传输量可以允许逐byte查找0xff字符；server=\>client方向上的传输需要使用"out-band signaling"方式，较大的数据量需要使用TCP urgent mode机制来标识重要command数据的位置。

##### 7 客户端转义符(client escape)
通常情况下，在Rlogin client端键入的所有内容都会被发送到server上。有时，我们只希望和Rlogin的client端进程进行交互，并不需要将键入的内容发送到server上。这种情况下，就需要借助客户端转义符来完成：键入tlide("~")作为键入的行首字符，紧接着键入下面4种字符之一：
- ➀ a period(".")：用于结束client端进程。
- ➁ end of file character(EOF:通常是Control-D)：用于关闭client端进程。 
- ➂ job control suspend character(通常是Control-Z)：用于将client进程挂起。
- ➃ job control delayed-suspend character(通常是Control-Y)：只将client端输入挂起，此时，我们在client键入的输入都将由运行在client端的进程(无论该进程是什么)来interpret(而不会发送到server端)，但从server=\>client发送的任何信息仍会显示在client端terminal上。这种机制非常适合我们需要在server端运行一个long-time running程序的场合：我们既需要在client端获悉该程序的输出，又想在client端运行其他程序。

仅当client端的unix系统版本支持任务流程控制(job control)时，后两个命令才有效。


### 三：Rlogin的案例分析
本部分内容包含2个案例，第一个案例展示了：Rlogin会话(session)初始建立时，client端和server端的协议交互；第二个案例展示了：当我们键入中断键以中止一个正在运行的、产生很多输出信息的server端进程时将会发生什么。我们还将展示穿过Rlogin session的normal data flow。

##### 1 初始的client-server协议(initial client-server protocol)
下图中展示的是从bsdi(client)到svr4(server)之间Rlogin应用程序建立一个连接的时序图，图中已经将通常TCP连接的建立过程、窗口通告信息、TOS服务类型字段信息省略。
<center><img src="/img/in-post/tcp-ip_img/telnet_rlogin_5.pdf" width="100%"></center>
- 本文前面[Rlogin应用程序的启动](http://localhost:4000/tcp-ip/2021/11/08/post-tcp-ip-tcp-chapter-26/#1-应用进程的启动application-startup)部分的内容对应了时序图中segment#1～#9的数据分组交换过程。
- segment#10、segment#12、segment#14、segment#16是server在建立Rlogin连接成功的问候语。
- segment#18为shell提示符(prompts)。
- 注意到，建立Rlogin连接时，client端进程使用的端口号为1023，该端口由IANA负责分配。Rlogin协议有reserved port的机制，即要求client端使用的端口号≤1024。在unix系统中，client端进程不能占用reserved port，除非该进程拥有superuser privilege。这属于client和server之间authentication机制中的部分，借助该身份认证机制可以实现免密登录。[Stevens 1990]描述了在client和server之间关于reserved port和authentication的详细的信息。

<details>
    <summary>[点击展开] 使用Rlogin程序从client向server发送"data/n"，使用tcpdump打印分组交换信息</summary>
    <center><img src="/img/in-post/tcp-ip_img/tcp_19_2.pdf" width="100%"></center>
</details>
- Rlogin程序每次从client到server只会发送1byte数据。
- Rlogin建立的TCP连接可以被任何一端关闭。如果我们在键入了一个命令使得server端运行的shell进程关闭，那么server端将执行active close；如果我们在client端键入一个"escape"(通常是tlide)，并紧接着键入一个period或者end-of-file字符，则client端执行active close。

##### 2 客户键入中断键案例&分析
本部分中的案例涉及许多TCP相关算法：TCP的紧急模式、糊涂窗口避免、窗口流量控制、保持定时器。我们sun主机上启动client端进程，并使用Rlogin登录到bsdi主机上，向client的terminal输出一个大文件，然后使用Control-S组合键中断文件输出，当输出停止后，再键入interrupt key(DELETE)来中止Rlogin程序。

**❮client端sun主机的terminal中的案例调用&输出❯** 
<center><img src="/img/in-post/tcp-ip_img/telnet_rlogin_6.pdf" width="100%"></center>

**❮案例中关键组件模式框图 & 数据流概述❯** <br>
<center><img src="/img/in-post/tcp-ip_img/telnet_rlogin_7.pdf" width="100%"></center>
图中TCP receive buffer和TCP send buffer中的数据分别对应后面的时序图中segment#1~7、segment#14~18。

**❮案例中关键组件的运行状态分析❯** <br>
- ➀ 我们通过键入Control-S来停止client端继续输出文件内容。
- ➁ Rlogin程序的client端被中止向terminal中写入数据，但是client端的output buffer仍然能够从TCP的receive buffer中提取数据，因此很快terminal的output buffer将会被填满。
- ➂ 由于terminal的 output buffer填满，所以Rlogin应用进程将不会再从client端TCP receive buffer读取数据，TCP继续从连接中接受文件数据，导致receive buffer填满。
- ➃ TCP receive buffer填满，于是TCP client向TCP server通告receiver窗口为0使sender停止发送数据。
- ➄ 由于收到receiver通告窗口为0，TCP server的output被停止，导致其send buffer很快就会填满。
- ➅ 由于send buffer被填满，Rlogin server因此被中止，从而Rlogin server不能从运行在server端的cat应用进程读取数据。
- ➆ 当cat进程的output buffer填满后，其进行本身也会中止。
- ➇ 我们在Rlogin client端键入interrupt key来中断运行在Rlogin server上的cat应用进程。interrupt key可以从TCP client发送到TCP server上，因为这个方向的数据传输没有被"窗口流控制"所中止。
- ➈ server端cat进程收到中断指令，停止运行，将导致cat进程的output buffer被flush(清洗)。这个flush操作将唤醒Rlogin server进程，使其进入紧急模式，并向Rlogin client发送"flush output"指令(0x02)。

**❮案例数据分组交换时序图❯** - 从client端terminal进程键入Control-S到使用interrupt key中止server端cat应用进程的过程 <br>
<center><img src="/img/in-post/tcp-ip_img/telnet_rlogin_8.pdf" width="100%"></center>
相关知识跳转：[关于糊涂窗口综合症](http://localhost:4000/tcp-ip/2021/10/18/post-tcp-ip-tcp-chapter-22/#三糊涂窗口综合症silly-window-syndrome)


### 四：Telnet协议
Telnet协议可以运行在任何主机(operating system)之间，也可运行在任何终端之间。RFC-854定义了Telnet的规范。[Comer and Stevens 1993]-chapter25/26给出了Telnet实现细节&源代码。

##### 1 NVT(network virtual terminal)
网络虚拟终端NVT是一种位于连接两端的虚拟设备(imaginary device)。client端操作系统需要将任何terminal类型映射成NVT类型；server端操作系统必须将NVT类型映射成任何server支持的terminal类型。

NVT是一个带有键盘和打印机的字符设备(character device)。用户通过键盘键入的数据被发送到server端，client端从server端接受的数据被输出到打印机上。默认情况下，client端将会在打印机上回显用户键入的数据(即打印用户的输入)，但是通常可以设置选项来改变处理用户键入数据的方式。

##### 2 NVT ASCII
NVT ASCII表示一种7-bit variant ASCII的字符集，NVT ASCII被所有网络协议簇(Internet protocol suite)使用。每个7-bit字符按照8-bit(byte)的形式传输，最高位bit被设置为0。行结束(end-of-line)字符以2-character sequence方式进行传输："CR(carriage return)"+"LF(linefeed)"的方式来传输，使用"\r\n"来表示。单独的"CR"也是以2-character sequence方式进行表示："\r\0"，相当于"CR"+"NUL"。

在后续章节内容可以看到：FTP、SMTP、Finger、Whois等应用程序都使用NVT ASCII字符集来表示client端命令以及server端response。

##### 3 Telnet命令
Telnet在两个通信方向上都使用"in-band signaling"(带内信令)的方式，即使用2byte "0xff"用于标识命令数据，称之为IAC(interpret as command)符号。

"0xff"作为命令数据标识符，为了和数据255作区分，当需要发送数据255时使用"0xff"+"0xff"。前一小节提到Telnet数据流使用7-bit ASCII字符进行传输，因此不能直接传输数据255，而是通过IAC+IAC的方式进行传输。另外，在RFC-856中定义了一个选项：允许Telnet数据以8bit的方式进行传输。

**❮Telnet IAC prefix command table❯** <br>
<center><img src="/img/in-post/tcp-ip_img/telnet_rlogin_9.pdf" width="60%"></center>
IAC+IAC=数据255，使用这种方式能够实现使用7-bit ASCII来传输数据255。<br>
描述中编号1～4为选项协商，关于option negotiation机制的更多细节参考下一小节内容。<br>
IAC+SB表示子选项起始信号，IAC+SE表示子选项结束信号，关于suboption negotiation机制的更多细节参考下下小节的内容。<br>

##### 4 选项协商(option negotiation)
尽管Telnet程序启动时假定两端都使用NVT(network virtual terminal)，但通常Telnet连接建立后的第一个交换的分组是选项协商。选项协商是对称的，即连接两端都可以向对端发送option negotiation request。

**❮Telnet选项协商的4种request类型❯** <br>
对于任何给定的option，任意一端都可以发送下面4个的request中的任意一个：
- ➀ WILL：sender(client/server)想要激活option。
- ➁ DO：sender(client/server)想让receiver激活option。
- ➂ WONT：sender(client/server)想要禁止option。
- ➃ DONT：sender(client/server)想让receiver禁止option。

**❮Telnet选项协商的6种sender-receiver状态类型❯** <br>
Telnet规定：对于激活option请求，receiver可以选择同意or不同意；对于禁止option请求，receiver必须同意。因此综合sender & receiver在option negotiate过程中不同的状态，共有6种情况如下表所示：
<center><img src="/img/in-post/tcp-ip_img/telnet_rlogin_10.pdf" width="60%"></center>

**❮支持option negotiation机制的选项❯** <br>
选项协商过程需要3byte数据：IAC byte、4种request类型之一(WILL/DO/WONT/DONT)、a ID byte用于指定需要启动/禁用的选项。

目前有超过40个不同的option支持option negotiation机制。Assigned Numbers RFC指定了支持option negotiation机制的option ID，以及描述option的相关RFC。本文中涉及的支持option negotiation机制的option如下表所示：
<center><img src="/img/in-post/tcp-ip_img/telnet_rlogin_11.pdf" width="40%"></center>

**❮Telnet远程登录应用中的option negotiation机制❯** <br>
和大多数Telnet协议内容类似，Telnet option negotiation机制也是对称的，即连接的两端都可以发起对某个option的协商。但是，Telnet作为一种远程登录的应用而言，功能上并不是对称的：client端执行某种类型的操作，server端执行其他类型的操作。在后续案例部分将会看到：某些Telnet option仅适用于client端(e.g. enable option linemode)，一些只适用于server端。

##### 5 子选项协商(suboption negotiation)
一些选项不仅需要enable/disable信息：对于terminal type选项而言，client还要发送一个ASCII字符串用于标识terminal type。为了解决这些选项的协商，需要使用子选项协商机制。

RFC-1091定义了terminal type选项协商的规范：
- ➀ 连接任意一端(通常是client端)发送3-byte sequence \<IAC, WILL, 24\> 以请求enable terminal type option。24是terminal type option的选项ID。
- ➁ 如果server端允许"enable the option"，则回应3-byte sequence \<IAC, DO, 24\>。
- ➂ 然后server端向client端发送6-byte sequence \<IAC, SB, 24, 1, IAC, SE\> 来请求client端的terminal type。其中IAC SB表示"suboption begin"；24表示这是"terminal type选项的suboption"；1表示"send your terminal type"；IAC SE表示"suboption end"。
- ➃ 如果client端terminal type为"ibmpc"，则回应11-byte sequence \<IAC, SB, 24, 0, 'I', 'B', 'M', 'P', 'C', IAC, SE\>。其中4th byte 0表示"my terminal type is:"。terminal type在Telnet suboption均以uppercase字符表示(通常被server转换成lowercase)。Assigned Numbers RFC中包含了"official acceptable terminal types"，但是在unix系统中，任何server能够接受的terminal type都是合理的，只要这些名字在termcap或terminfo数据库中。

##### 6 半双工、一次一个字符、一次一行、行模式
对于绝大多数Telnet client端、server端而言，共有4种操作模式(operation mode)：
- ➀ 半双工(half-duplex)。Telnet的默认模式，但现在很少使用。默认的NVT(network virtual terminal)是一个半双工设备，在接受user input之前需要一个GO AHEAD(GA)命令。user input在Telnet client本地端被回显(从NVT keyboard=\>NVT printer)，因此client端到server只能发送整行的数据。<br>
尽管这种模式被所有terminal type所支持，但它不能够充分应对支持full-duplex的terminal和host之间的通信任务，而这种通信现已成为标准模式。RFC-857定义了ECHO选项，RFC-858定义了SUPPRESS GO AHEAD选项。联合这两个选项可以为模式➁提供支持：character-at-a-time with remote echo。

- ➁ 一次一个字符(character-at-a-time)。这种模式和本文前面介绍的Rlogin应用运行方式类似：client端每个键入的character都被单独发送到server上。Telnet server回显收到的character，除非server端应用进程将回显功能关闭。在较长的延时的网络、较繁忙的网络中对每个client端键入的character进行回显带来的问题是显而易见的(perceptible)；然而我们将看到character at a time被现在许多Telnet实现用作默认模式。<br>
进入这种模式只需要server端启用SUPPRESS GO AHEAD选项即可：可以通过client端发送DO SUPPRESS GO AHEAD来完成(client请求server开启选项)；也可以通过server发送一个WILL SUPPRESS GO AHEAD来完成(server请求client批准开启选项)。<br>
另外，在开启character-at-a-time模式后，通常server端还会发送WILL EHCO来请求允许回显。

- ➂ 一次一行(line-at-a-time)。这种模式通常被称为"kludge line mode"，该模式根据RFC-858实现。RFC-858规定：为了实现带有远程回显的character-at-a-time，ECHO和SUPPRESS GO AHEAD选项必须都启用。当ECHO和SUPPRESS GO AHEAD选项中任何一个没有被启用时，则Telnet进入line-at-a-time模式。在本文后续案例部分将看到这个选项是如何协商确定的，以及当server端应用进程需要接受每次keystroke信息时，该选项是如何被禁用的。

- ➃ 行模式(linemode)。这种模式通常被称为"real line mode"。使用linemode来表示启用linemode选项，该选项在RFC-1184中定义。这个选项由client进程和server进程之间协商确定，该模式弥补了kludge-linemode中所有的缺陷(deficiencies)。较新版本的Telnet支持此选项。

**❮不同系统下Telnet client进程和server进程之间默认操作方式❯** <br>
<center><img src="/img/in-post/tcp-ip_img/telnet_rlogin_12.pdf" width="80%"></center>
- ➀ char表示character-at-a-time模式，kuldge表示line-at-a-time模式，linemode表示linemode模式。
- ➁ 只有当client端、server端主机系统都是BSD/386或4.4BSD时，才支持linemode作为默认模式；当server端系统为BSD/386或4.4BSD时，如果client端不支持linemode，则server尝试**协商**使用"kludge line mode"模式。
- ➂ 表中所有的client、server的操作系统都支持kuldge line mode，但并不会将其作为默认模式，只在server端"主动协商"时才会使用。

##### 7 同步信号(synch signal)
Telnet同步信号使用IAC DM(data mark)作为标记，并采用TCP紧急数据的方式发送。在数据流中的DM命令标记用于通知receiver回归到正常处理流程(DM被用于标记同步信号结束)。同步信号可以在Telnet连接的任何方向上发送。<br>
当任何一端收到"对端已经进入紧急模式"的通知后，它将开始读取数据流，并将除Telnet命令之外的所有数据部分丢弃。紧急数据的最后一个byte是DM byte。使用TCP紧急模式的原因是：即使TCP数据流已经被窗口流控制机制中止，仍然允许Telnet命令通过连接进行传输。

本文后续Telnet案例部分将介绍synch signal有关案例。

##### 8 客户端转义符(client escape)
同Rlogin client端一样，Telnet client也有"仅和自身通信"的功能，即不将user在client端键入的内容发送到server上。通常客户端转义符为"Control-]"，通常被打印成"^]"。<br>
在Telnet client端键入转义符后，client进程通常会显示提示符"telnet>"，在这种状态下有很多可以键入的命令，有的用于改变session的属性，有的用于在session中打印信息；其中help命令在绝大多数unix系统中用于显示可用的命令。

本文后续Telnet案例部分将介绍关于client escape的用法，以及在"telnet>"状态下，可以键入的命令。


### 五：Telnet举例
这一部分将介绍在三种不同的操作模式下Telnet选项协商过程。并观察交互式程序的user使用interrupt key中止server端正在运行的进程后的系统状态。

##### 1 单字符模式：character-at-a-time mode
在character-at-a-time模式下，在terminal键入的每个character都会单独被发送到server端，随后server将回显该character。在选项协商过程中，当client端系统版本较新时(BSD/386)，将会尝试协商开启许多较新的选项，但是由于server端系统版本老旧(SVR4)，这些协商尝试将会被server拒绝。

将激活client进程的一个选项，以看到client和server之间的选项协商的内容。另外，使用tcpdump来获取选项协商过程中的数据分组交换及其时序信息。

**❮从bsdi主机建立Telnet连接到svr4主机，显示选项协商过程❯** <br>
<center><img src="/img/in-post/tcp-ip_img/telnet_rlogin_13.pdf" width="100%"></center>
- **line**➀ client端发起SUPPRESS GO AHEAD选项的协商。该选项以DO开始，表示client端希望server端启用该选项；通常GO AHEAD命令由server发送到client(使用DO选项时会禁止GO AHEAD(GA)命令的发送)。在line➀⓪可以看到server端返回允许启用选项应答。
- **line**➁ client端希望按照RFC-1091 [VanBokkelen 1989]中的定义发送terminal type。这个协商过程是unix系统client端的通用过程。该选项的协商过程通过client端发送WILL命令来完成(主动通知对端terminal type)。
- **line**➂ RFC-1073 [Waitzman 1988]定义了negotiate about window size(NAWS)。如果server端同意启用该选项，则client将通过suboption的方式发送terminal window的行数、列数。另外，后续terminal window大小发生改变，client端随时发送这个suboption选项(这个机制同本文前面介绍的Rlogin的0x80命令相似)。
- **line**➃ TSPEED选项允许sender(通常是client)发送它的terminal speed，该选项在RFC 1079 [Hedrick 1988b]中定义。如果server端允许启用该选项，则client通过suboption的方式发送它的发送速率(transmit speed)和接受速率(receive speed)。
- **line**➄ LFLOW选项表示"local flow control"，该选项在RFC-1372 [Hedrick and Borman 1992]中定义。client端通过发送这个选项到server端以通知对端：它想要enable/disable通过命令方式进行流量控制。如果server接受该选项，则每当client/server两端需要处理Control-S(START流量控制)和Control-Q(STOP流量控制)按键信息时，就向client端发送suboption(此option同本文前面介绍的Rlogin的0x10、0x20命令相似)。由client进程进行流量控制的效果要比server端进行流量控制要好。
- **line**➅ LINEMODE选项表示之前讨论过的"real line mode"。所有terminal character的处理都由Telnet client负责(包括backspace、erase line, etc.)，处理得到的完整的lines被发送到server端。关于此选项本文后续有相关案例。
- **line**➆ ENVIRON选项用于client端将user environment variables发送到server端，该选项在RFC-1408 [Borman 1993a]中定义。unix系统中的环境变量通常由name(uppercase)、=(equals sign)、string value构成，不过这只是一个惯例而已。对于BSD/386系统，Telnet client只有2个环境变量可以发送(DISPLAY、PRINTER)，前提是这两个环境变量被定义&被启用。Telnet user也可以定义其他需要发送的环境变量。
- **line**➇ STATUS选项运行Telnet连接的一端询问对端对于Telnet选项的当前取值(current status)的理解(perception)，该选项在RFC-859 [Postel and Reynolds 1983c]中定义。如果STATUS选项被启用(enabled)，客户端进程可以请求server端以子选项的方式发送对应suboption的状态值。
- **line**➈ 这是来自server端的第一个响应。server端同意启用terminal type选项。但是client端只有等到server发送请求terminal type suboption时(line➀➆)才能发送自己的terminal type - line➁。
- **line**➀⓪ server端同意suppress sending GO AHEAD命令 - line➀。
- **line**➀➀ server端不同意client端发送其terminal window size - line➂。
- **line**➀➁ server端不同意client端发送其terminal speed - line➃。
- **line**➀➂ server端不同意client端进程执行流量控制 - line➄。
- **line**➀➃ server端不同意client端启用linemode(real line mode)选项 - line➅。
- **line**➀➄ server端不同意client端发送其环境变量 - line➆。
- **line**➀➅ server端不会发送status状态信息 - line➇。
- **line**➀➆ server端发送的suboption，用来请求client发送其terminal type - line➁。
- **line**➀➇ client端发送自己的terminal type：6-character string "IBMPC3" - line➁。
- **line**➀➈ server端请求client端执行回显。这是案例中server端第一次主动发起选项协商。
- **line**➁⓪ client端同意接受&显示server端的回显数据。
- **line**➁➀ server端请求client端执行回显。在client和server之间已经进行了line➀➈和line➁⓪的命令交换后，这条命令是多余的(superfluous)。不过，这条命令可以用于unix Telnet servers判断client端系统是否为4.2BSD或者是较新的BSD版本：如果client对此命令回应WILL ECHO，那么client端可能是一个载有4.2BSD系统的较老主机，并且不能支持TCP的紧急模式。
- **line**➁➁ client端回应WONT ECHO，表明client端不是一个4.2BSD主机。
- **line**➁➂ server端回应DONT ECHO作为line➁➁的响应。

**❮从bsdi主机建立Telnet连接到svr4主机，选项协商过程数据分组交换时序图❯** <br>
<center><img src="/img/in-post/tcp-ip_img/telnet_rlogin_14.pdf" width="100%"></center>
上述选项协商的时序图中已经去掉了TCP连接建立、中止的部分。<br>

**❮Telnet client同server端建立完TCP连接后，不一定需要进行选项协商❯** <br>
我们还可以使用Telnet连接一些unix系统中的标准服务进程：daytime、echo等。下图展示了chapter18中和discard服务建立TCP连接的过程。
<details>
    <summary>[点击展开] chapter18 在svr4主机上通过Telnet进程和bsdi的标准discard服务建立&中止TCP连接。<br>
    使用tcpdump打印分组交换情况如下所示：</summary>
    <center><img src="/img/in-post/tcp-ip_img/tcp_18_2.pdf" width="100%"></center>
</details>
从上述tcpdump打印的分组交换过程显示：client端svr4主机并没有发起选项协商。这是因为在unix系统中，除非使用标准Telnet端口23，否则不进行选项协商。这个特性使得Telnet client端可以和其他的非Telnet的server端借助standard NVT(network virtual terminal)技术的进行数据交换。

##### 2 真行模式：linemode(real line mode)
为了描述Telnet的linemode模式选项协商过程，在bsdi主机(BSD/386)上运行Telnet client进程，和位于vangogh.vs.berkelay.edu的系统为4.4BSD的主机建立连接；client端和server端的系统都支持这个选项。

对于linemode模式，本部分内容将不再关注在该模式下的选项协商中数据分组交换情况、及选项/子选项的协商步骤；这部分内容同上一种模式本质相同，omitted。取而代之，我们将关注在linemode模式下的选项协商过程中的一些不同之处：
- ➀ Telnet client端BSD/386系统的bsdi主机尝试协商的选项：window size、local flow control、status、accepting environment variables、terminal speed，在server端4.4BSD系统vangogh主机上都支持。
- ➁ Telnet server端4.4BSD主机尝试对一个BSD/386不支持的选项进行协商：authentication。该选项用于避免在远程登录时将密码以cleartext的形式发送。
- ➂ Telnet client发送选项协商命令WILL LINEMODE(请求server端启用LINEMODE模式)，server端主机支持该选项，因此返回DO LINEMODE应答。server的应答将导致client以suboption形式发送16个特定字符，即16个影响client进程的terminal character：the interrupt character、EOF character, etc。<br>
Telnet server发送suboption到client端，令其处理所有的input lines，执行所有编辑操作(e.g. 删除字符、删除行等)，client只发送完整的lines到server端。server端还要求client端将所有的interrupt keys或signal keys转换为对应的Telnet字符；例如：如果interrupt key是Control-C，user在client端键入Control-C以中断server端正在运行的进程，client端将Control-C转换为Telnet的IP 命令\<IAC, IP\>到server端。
- ➃ 当user在client端键入登录密码时的情况也会有所不同：在使用Rlogin应用or使用Telnet character-at-a-time模式时，都是server端负责回显，但它们都不会对client端键入&传输的密码进行回显。然而在Telnet linemode模式下，由client端负责回显任务。为了针对在linemode模式下回显机制的改变，需要进行如下调整：
    - (a) server发送WILL ECHO到client端。
    - (b) client端回应DO ECHO。
    - (c) server端向client端发送字符串"Password: "，client端将接受的"Password: "字符串回显到terminal。
    - (d) user在client端键入登录password，当user键入RETURN，密码将被发送到server端。这个password并不会被client回显，因为client端认为server负责将其回显。
    - (e) server端向client端发送2byte sequence CR+LF用来移动client端terminal中的光标，因为user之前键入的RETURN并没有被回显。
    - (f) server端发送WONT ECHO。
    - (g) client端返回DONT ECHO应答。后续将继续由client端负责回显。

当采用行模式的Telnet登录成功后，client端负责将需要发送的character处理成lines，并将其发送到server端，这也是linemode选项的基本目的。linemode方式减少了client和server之间的交换的数据分组数量，并且为user的输入提供更快的响应。

**❮在vangogh使用linemode模式的Telnet connection中键入"date"命令后的分组数据交换情况❯** <br>
<center><img src="/img/in-post/tcp-ip_img/telnet_rlogin_15.pdf" width="60%"></center>
上面的分组交换时序图中已经将服务类型字段去掉，并且不显示窗口通告信息。
<details>
    <summary>[点击展开] chapter19 在bsdi主机上发送"date\n"到svr4主机，用tcpdump打印分组交换情况</summary>
    <center><img src="/img/in-post/tcp-ip_img/tcp_19_2.pdf" width="100%"></center>
</details>
将上述Telnet connection键入date和Rlogin conenction中键入同样命令涉及的分组交换情况(如下图)进行比较，可以发现：Telnet只需要2个数据分组(segment#3 & segment#4)就能完成date命令回显；而完成相同任务，Rlogin需要至少15个报文分组(5个分组用于键入的"date\n"，5个分组用于回显的数据，以及5个确认ack应答)。因此使用linemode模式的Telnet connection带来的流量节省是非常可观的。

**❮当server端运行的应用进程(e.g. vi editor)需要使用single-character模式时，将发生如下交互过程❯** <br>
- ➀ 当server端应用进程启动后，改变其内部的pseudo-terminal运行模式，因此Telnet server进程将被通知需要启用single-character模式。server端发送WILL ECHO命令到client端，并发送linemode suboption以通知client端进入character-at-a-time模式，不要按linemode模式发送数据。
- ➁ Telnet client端回应DO ECHO，并返回linemode suboption确认应答。
- ➂ 应用进程在server端运行，user在client端terminal键入的每个character都会被单独发送到server端(当然在实际传输时会受Nagle算法的限制)，server端将处理必要的回显工作。 
- ➃ 当server端应用进程中止后，恢复其pseudo-terminal运行模式，并通知Telnet server。 随后Telnet server发送WONT ECHO到client端，以及linemode suboption以通知client端恢复到linemode处理待发送数据。
- ➄ client端返回DONT ECHO应答，并返回linemode suboption确认信号。

上述案例表明echo function和character-at-a-time/line-at-a-time模式是互相独立的功能特性，当我们键入password时，必须关闭echo功能并打开line-at-a-time模式；对于一个full-screen应用而言(e.g. vi editor)，echo功能将被关闭，并打开character-at-a-time模式。

**❮Rlogin和Telnet在默认模式以及linemode下特性对比图表❯** <br>
<center><img src="/img/in-post/tcp-ip_img/telnet_rlogin_16.pdf" width="80%"></center>

##### 3 准行模式：line-at-a-time mode(kludge line mode)
从下图可以看出：如果Telnet client端不支持linemode选项，则支持linemode选项的Telnet server将转而使用line-at-a-time模式(kludge line mode)。
<details>
    <summary>"不同系统下Telnet client进程和server进程之间默认操作方式图"</summary>
    <center><img src="/img/in-post/tcp-ip_img/telnet_rlogin_12.pdf" width="80%"></center>
</details>
需要注意的是：上表中支持kludge line mode的Telnet client & server的默认模式并不是准行模式，并且该模式必须由server端或者client端主动激活才能使用。本小节后续内容将介绍准行模式(kludge line mode)是如何通过Telnet选型协商过程被启用的。

首先描述当Telnet client端不支持real linemode时，BSD/386系统的Telnet server如何通过选项协商以使两端进入kludge line模式。
- ➀ 当Telnet client拒绝了server端启用linemode选项的请求后，server端将发送DO TIMING MARK选项；RFC-860 [Postel and Reynolds 1983f]定义了该选项。这个选项的作用是：让Telnet进程两端在模式选取上同步(将在本小节后续部分进行讨论)；在本案例中用于判断client端是否支持kludge line mode。
- ➁ Telnet 返回WILL TIMING MARK应答，表明其支持kludge line mode。
- ➂ Telnet server端将WONT SUPPRESS GO AHEAD选项和WONT ECHO选项一并发送，通知client端它想要禁用这2个选项。本文前面曾经提到character-at-a-time模式需要同时开启SUPPRESS GO AHEAD和ECHO选项，因此禁用这2个选项后Telnet将进入准行模式kludge line mode。
- ➃ Telnet端返回DONT SUPPRESS GO AHEAD和DONT ECHO应答。
- ➄ Telnet端发送login提示符，然后user在client端terminal进程中键入登录名(login name)。键入的login name以整行的方式发送到server端，并由client端本地进行回显。
- ➅ Telnet server端发送字符串"Password: "以及WILL ECHO选项。这将关闭Telnet client端user键入password后的回显，因为client端Telnet进程认为server端将负责password的回显。Telnet client将返回DO ECHO应答。
- ➆ user在client端键入password，它将以整行的方式被client端发送到server上。
- ➇ Telnet server端发送WONT ECHO以使client端重新激活回显功能，client端将返回DONT ECHO应答。

经过上面的选项协商过程后，user键入的normal commands将以linemode模式类似的方式进行处理。Telnet将负责数据的编辑、回显以及向server端发送完整的行。

**❮处于character-at-a-time模式的Telnet server进入kludge line mode模式时的选项协商过程❯** <br>
在本文前面"不同系统下Telnet client进程和server进程之间默认操作方式"图表中，提到表中所有表项都支持准行模式(kludge line mode)，但是大部分情况的默认启动模式都不是准行模式。当发生模式转换时，我们可以很容易看到选项协商的过程如下图所示：
<center><img src="/img/in-post/tcp-ip_img/telnet_rlogin_17.pdf" width="100%"></center>

**❮处于kludge line mode模式的Telnet server进入character-at-a-time模式时的选项协商过程❯** <br>
如果我们在server端运行如vi editor这样的进程，将涉及到从kludge line mode到character-at-a-time模式的转换，并且在vi editor进程中止时恢复到kludge line mode。这个过程将涉及如下的选项协商过程：
- ➀ server端应用进程(e.g. vi editor)改变其pseudo-terminal的模式，然后通知Telnet server进程，因此server端将进入character-at-a-time模式。server端进程发送WILL SUPPRESS GO AHEAD和WILL ECHO，这将导致client端进入character-at-a-time模式。
- ➁ Telnet client端回应DO SUPPRESS GO AHEAD和DO ECHO。
- ➂ server端应用进程(e.g. vi editor)开始在server端运行。
- ➃ 当server端应用进程(e.g. vi editor)中止后，改变其pseudo-terminal的模式，Telnet server端进程将发送WONT SUPPRESS GO AHEAD和WONT ECHO，使得client端恢复准行模式(kludge line mode)。
- ➄ Telnet client端进程返回应答DONT SUPPRESS GO AHEAD和DONT ECHO，表明它已经恢复到准行模式(kludge line mode)。

**❮Telnet三种工作模式下关于SUPPRESS GO AHEAD和ECHO选项的设置情况❯** <br>
<center><img src="/img/in-post/tcp-ip_img/telnet_rlogin_18.pdf" width="70%"></center>

##### 4 行模式(客户中断键)：linemode(client interrupt key)
本部分内容主要观察：当Telnet client端在terminal键入interrupt key后，Telnet将执行什么操作。本案例中，在client端bsdi主机上和server端vangogh.cs.berkeley.edu主机之间建立一个会话进程(serssion)。下面展示了当在bsdi主机和vangogh主机建立的session中键入interrupt key后的分组交换时间序列(已经去掉窗口通告和服务类型字段部分的信息)。

**❮Telnet三种工作模式下关于SUPPRESS GO AHEAD和ECHO选项的设置情况❯** <br>
<center><img src="/img/in-post/tcp-ip_img/telnet_rlogin_19.pdf" width="100%"></center>
- ➀ Host Requirements RFC规定了IP(中断进程，interrupt process)应该使用Telnet的synch signal形式发送：则\<IAC, IP\>后面紧跟着\<IAC, DM\>标记，**紧急指针(urgent pointer)应当指向DM字节**。大多数unix系统下的Telnet client端有一个选项用于设置"使用synch signal形式发送IP标识，但该选项默认处于关闭状态。<br>
本案例中，该选项处于默认关闭状态，即IP标识使用普通数据形式而不是紧急模式synch signal发送。
- ➁ 为什么时序图中的synch signal被分成两个数据分组(segment#3 & segment#4)发送？<br>
原因是：Host Requirements RFC表明urgent pointer应当指向urgent data的最后一个byte(即紧急指针应指向\<IAC, DM\>的DM)，然而大多数伯克利实现的衍生版本将urgent pointer指向urgent data的尾后byte(e.g. 本案例中的vagogh主机)。vangogh主机的Telnet server进程故意将synch signal标识的第一个byte写成urgent data，利用urgent pointer指向urgent data尾后byte的特性指向DM(data mark)标识；因此作为urgent data的IAC byte将被立即发送，随后才是被urgent pointer指向的第二个DM byte。

### 六：小节
本文主要介绍了2种远程登录应用程序：Rlogin和Telnet。两者存在不同之处，Rlogin假定连接的双方都是unix系统，而Telnet可以通过选项协商机制在不同系统类型的主机之间运行。

另外Telnet在传输数据中有三种方式：character-at-a-time、linemode、line-at-a-time(kludge line mode)，传输模式同样可以通过选项协商机制确定。

**❮Rlogin和Telnet不同特征下的特性对照表❯** <br>
<center><img src="/img/in-post/tcp-ip_img/telnet_rlogin_20.pdf" width="80%"></center>

## Reference
> \<tcp-ip: illustrated vol1\> chapter26 <br>

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
