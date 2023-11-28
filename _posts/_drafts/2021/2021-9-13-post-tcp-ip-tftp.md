---
layout: post
title: "tcp-ip: TFTP (简单文件传输协议)"
subtitle: '[tcp-ip: illustrated vol1] - chapter15' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-09-13 17:02
lang: ch 
catalog: true 
categories: tcp-ip 
tags:
  - Time 2021
  - tcp-ip illustrated vol1 
---
### 一：引言
简单文件传输协议(trivial file transfer protocol)最初被用于无盘系统(无盘工作站、X终端)进行系统引导时通信的协议。和chapter27将要介绍的依赖于TCP通信的FTP文件通信协议不同，为了保持通信数据报的"simple & small"，TFTP使用UDP作为载体进行通信。<br>
RFC-1350是描述TFTP-v2的正式规范。

### 二：TFTP协议
TFTP客户端于服务端交换信息，首先客户端发送一个读请求(RRQ)or写请求(WRQ)给TFTP服务器。

##### 1 TFTP协议相关数据报基本格式
<center><img src="/img/in-post/tcp-ip_img/tftp_1.pdf" width="100%"></center>

##### 2 TFTP读请求(RRQ)和写请求(WRQ)
- 在一个无盘系统进行正常系统引导场景下，第一个请求是读请求。
- 读请求和写请求的数据报文格式相同。
- **WRQ写请求准备阶段的通信** 客户端向服务端发送WRQ操作码数据报，同时指明需要写的文件名和写入模式。如果该文件可以被客户端执行写入操作，TFTP服务端就返回块编号为0的ACK数据包。
- **WRQ通信完成，具体执行写入操作** TFTP客户端将文件"first 512 bytes"以块编号为1的TFTP数据报文格式发出，当TFTP服务端收到数据后，返回块编号为1的ACK应答，以此类推。
- **RRQ读请求准备阶段的通信** 客户端向服务端发送RRQ操作码数据报，同时指明需要读取的文件名和读取模式。如果该文件可以被客户端读取，TFTP服务端就返回一个块编号为1的相应文件数据报，在收到该数据报后客户端返回块编号为1的TFTP-ACK应答；TFTP服务端返回块编号为2的相应文件数据报，客户端返回块编号为2的TFTP-ACK应答，以此类推直到整个文件被传送完成。
- **当服务端不能处理读请求or写请求时返回TFTP差错报文** 当在使用TFTP协议传输过程中发生读写差错也会传送TFTP差错报文，并终止传输。
- **TFTP差错报文信息格式** 包含一个差错码，以及一个用ASCII码表示的差错报文字段，字段中可能包含额外的系统环境说明信息。

这种类型的读/写传送通信方式称为"stop-and-wait protocol"，因为每份数据报发送之前都需要"**停止**立即发送的动作，**等待**上一份数据报成功接受的ACK应答信号"。这种类型的协议只能用于一些非常简单的传送协议(e.g. TFTP)中，因为其降低了整个TFTP传输通信的瞬时数据吞吐量。TFTP传输协议的优势在于简单。<br>
后续将要介绍的TCP通信采用了不同形式的应答确认方式，能够增加系统的吞吐量。

### 三：TFTP传输案例&分析
##### 1 使用TFTP程序读取文本文件
在bsdi主机上运行TFTP客户程序，并从svr4主机上读取一个文本文件，TFTP程序打印输出如下所示。
<center><img src="/img/in-post/tcp-ip_img/tftp_2.pdf" width="100%"></center>
从上面TFTP打印输出可以看出，TFTP程序传输的数据长度为962byte。

##### 2 在bsdi主机上查看读取的文件的信息(验证)
<center><img src="/img/in-post/tcp-ip_img/tftp_3.pdf" width="100%"></center>

从"ls -l"命令打印文件长度信息可知：文件长度为914byte。从"wc -l"命令打印文件长度信息可知：TFTP 程序传输的文件数据长度为96byte。<br>
TFTP程序传输的文件共有48行，如果文件按照"netascii"模式进行传输，则文件中每出现一个换行符号(1byte)，传输的数据中添加一个回车"\<CR\>"+换行"\<LF\>"(共2byte)，因此，TFTP传输的字节相比原始文件字节数多48byte。<br>
因此，bsdi主机输出结果解释了为什么TFTP程序传输的文本文件为962byte = 914byte + 48byte。

##### 3 使用tcpdump命令显示TFTP传输通信的数据分组交换过程
下面对于tcpdump程序的输出进行逐行解析：

**[1] tftp客户端通过1106端口向服务端69端口(tftp服务公认端口)发送"RRQ"读请求** <br>
19字节TFTP传输长度 = 2字节操作码 + 7字节文件名 + 1字节0(结束字符) + 8字节"netascii" + 1字节0(结束字符)。<br>
tcpdump程序只对tftp程序中第一份RRQ和WRQ信息进行解释，此时客户端进程和服务端公知tftp服务进程进行通信。<br>
<center><img src="/img/in-post/tcp-ip_img/tftp_4.pdf" width="100%"></center>

**[2] tftp服务端返回第二份tftp数据报(用于传输正式文件内容)** <br>
tftp返回的tftp数据报的长度为516byte = 2byte操作码 + 2byte数据块号 + 512byte数据部分。

**(i)** 为什么tftp使用传输端口号为1077而不是tftp公认端口69返回数据报？<br>
tftp协议规定客户端程序必须向公知端口发送"RRQ"/"WRQ"数据报，之后服务器进程向服务器主机申请尚未使用的端口(1077)，用于客户端进程和服务器进程之间的数据交换。

**(ii)** 为什么tcpdump程序没有对tftp后续文件数据的传输进行解析？<br>
是因为在后续正式文件数据部分传输过程中更换了端口号(1077)，tcpdump不知道该端口号属于tftp进程。
<center><img src="/img/in-post/tcp-ip_img/tftp_5.pdf" width="100%"></center>

**(iii)** 为什么tftp服务端程序使用公知端口69接受"RRQ"/"WRQ"数据报，却使用为占用端口1077返回or接受tftp客户端的通信数据报？<br>
tftp服务切换端口的原因是：tftp服务器进程不想长时间占用这个公知69号tftp端口用于tftp文件通信；在使用tftp服务端程序传送当前数据时，该公知tftp服务69号端口需要留出供以接受其他应用程序发送的"RRQ"/"WRQ"请求。<br>
对比chapter10 dynamic routing protocols章节中，RIP服务程序向客户端发送数据如果超过512byte，则需要分成两个UDP数据报进行发送，并且两份数据报都使用RIP服务的公知端口进行发送。<br>
tftp不同于RIP协议，tftp客户端需要和服务端在较长时间内进行通信，如果通信请求和通信数据报都占用公知端口，那么在此期间，tftp服务端只能拒绝其他客户端的请求，而这将导致tftp的服务利用率非常低。因此，在确认进行通信后，采用闲置端口传输正式数据将大大增加tftp服务利用率。

**[3] tftp服务端返回第三份tftp数据报(用于传输正式文件内容)** <br>
tftp返回的tftp数据报的长度为454byte(不足512byte表明为用于传输该文件的最后一份数据报)。<br>
454byte = 450byte + 2byte操作码 + 2byte块编号 =\> 450byte + 512byte = 962byte tftp传输数据总量。
<center><img src="/img/in-post/tcp-ip_img/tftp_6.pdf" width="100%"></center>

### 四：tftp传输文件的安全性 
使用tftp传输文件时，并不需要验证用户名和口令，这也是tftp的区别与其他传输协议的特性。tftp被最初设计用来引导系统启动进程(bootstrap)，因此不可能提供在系统启动完成后才需要的特性(指验证用户名&密码)。

tftp传输协议的这种特性被许多黑客用来传输获取unix系统上存储passwd的文件，并猜测用户对应的口令。为了限制该类型的访问，目前大多数tftp服务器提供一个选项：专门用于限制通过tftp只能访问特定目录下的文件(通常是"/tftpboot")，该目录只包含无盘系统引导时所需的必要文件。

针对tftp安全性问题，还可以将unix系统中tftp服务程序的所属用户(user)和组(group)进行独立管理，任何真正用户都不会加入该分组，并且限制该分组用户只能访问具有读/写权限的文件。

## Reference
> \<tcp-ip: illustrated vol1\> chapter15 <br>

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
