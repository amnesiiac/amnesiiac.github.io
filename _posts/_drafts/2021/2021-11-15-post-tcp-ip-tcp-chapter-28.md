---
layout: post
title: "tcp-ip: SMTP (简单邮件传送协议)"
subtitle: '[tcp-ip: illustrated vol1] - chapter28' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: true 
date: 2021-11-15 15:17
lang: ch 
catalog: true 
categories: tcp-ip 
tags:
  - Time 2021
  - tcp-ip illustrated vol1 
  - todo
---
### 一：引言
电子邮件是最流行的应用程序之一。平均每个邮件包含大约1500bytes数据，一部分邮件甚至包含megabytes of data，因为electronic mail有时也用来传输文件。简单邮件传输协议(simple mail transfer protocol, SMTP)用于在TCP/IP连接中完成两个邮件代理之间的数据传输。

**使用TCP/IP交换电子邮件的传输原理示意图**
<center><img src="/img/in-post/tcp-ip_img/smtp_1.pdf" width="80%"></center>
- 用户(sender和receiver)和用户代理打交道，对于unix系统有多个用户代理供选择(MH、Berkelay Mail、Elm、Mush)。
- 准备发送or准备接收的邮件在TCP连接上的传输是通过报文传输代理完成的(message transfer agent, MTA)。unix系统通常选择Sendmail作为MTA。设置local MTA是系统管理员的职责，并为用户提供一个选项供其选择。

**本文内容概述** <br>
本文主要介绍位于TCP/IP连接两端的邮件传输代理(MTA)是如何使用TCP传输邮件数据。用户代理的实现&运行不属于本文介绍范畴。<br>
RFC-821规范了SMTP：指出在一个简单TCP连接上，两个MTA如何进行通信。RFC-822指示了利用RFC-821在两个MTA之间进行电子邮件传输的数据格式。

### 二：SMTP协议
两个邮件传输代理使用NVT ASCII数据格式进行邮件数据传输。client端向server端发出指令，server端使用数字应答码(numeric reply codes)以及可选的字符串(optional human-readable strings)进行应答。这些部分和chapter-27 ftp文件传输协议有相同之处。

在SMTP协议中，client只能向server发送很少种类的指令(less than a dozen)，上一章中ftp有超过40个指令。本部分内容不打算一一介绍SMTP协议指令，而是通过几个案例来分析SMTP在邮件数据在MTA之间的传输过程。

##### 1 简单案例分析(simple example)
发送一个只有一行的简单邮件，并观察SMTP运行情况。<br>
使用\<-v\>选项调用用户代理(user agent)，该选项将会被用户代理传输给邮件传输代理Sendmail(MTA)。使用该选项的MTA进程将打印通过STMP连接发送&接收的数据。
<center><img src="/img/in-post/tcp-ip_img/smtp_2.pdf" width="100%"></center>
- 上述简单邮件的传输过程中，SMTP client端只使用了5个指令：HELO、MAIL、RCTP、DATA、QUIT。

虽然我们键入到user agent的数据只有一行："1, 2, 3"，但是实际上传输邮件的数据分组大小为393bytes。数据的组成如下图所示：
<center><img src="/img/in-post/tcp-ip_img/smtp_3.pdf" width="100%"></center>

简单邮件案例时序图(与用户交互式命令输出对应)
<center><img src="/img/in-post/tcp-ip_img/smtp_4.pdf" width="100%"></center>

##### 2 SMTP指令
最基本的SMTP实现支持8个指令。上一节中的简单案例已经介绍了5个基本SMTP指令：HELO、MAIL、RCPT、DATA、QUIT。这一部分补充介绍几个SMTP命令。
- **RSET** 放弃当前邮件传输(transaction)，并将SMTP连接的两端复位(reset)。在邮件传输中存储的关于sender、recipients的信息以及邮件数据本身都被丢弃(discarded)。
- **VRFY** 该指令被client用来要求邮件发送端(sender)验证接收端地址(recipient address)，without sending mail to the recipient。VRFY命令通常被系统管理员用来debug邮件传输问题。
- **NOOP** "no operation"指令不进行任何实质性操作，只是强制server端响应一个OK信号(reply code:200)。

额外可选的SMTP指令介绍(additional, optional)
- **EXPN** 通常被系统管理员用来扩增邮件列表(expanding a mail list)。
- **TURN** 该指令允许client和server交换角色，以在相反方向上传输邮件，并且不需要关闭当前TCP连接、不需要创建新的连接支持反向通信。Sendmail邮件传输代理不支持此命令。
- **SEND/SOML/SAML** 这些指令基本不会在SMTP中实现，他们功能上属于MAIL指令的替代指令：These three allow ❮combinations❯ of the mail being delivered directly to the user's terminal (if logged in), or sent to the recipient's mailbox.

##### 3 SMTP邮件的"信封、首部、正文"(envelope、headers、body)
**[1] envelope** SMTP邮件的信封被邮件传输代理MTA用于邮件投递(delivery)。在本文前面的简单案例中，邮件的信封如下所示：<br>
<center><img src="/img/in-post/tcp-ip_img/smtp_5.pdf" width="100%"></center>
RFC-821规定了SMTP邮件envelop的内容和解释方式(contents & interpretation)、以及用于TCP连接上进行邮件交换的协议。

**[2] headers** 首部字段被用户代理进程(user agent)使用。在本文前面的简单案例中可以看到邮件正文被添加了9个首部字段后发送给邮件传输代理MTA，9个首部字段分别是：Received、Message-Id、From、Date、Reply-To、X-Phone、X-Mailer、To、Subject。
<details>
  <summary>[点击展开] 简单案例邮件首部+正文</summary>
  <center><img src="/img/in-post/tcp-ip_img/smtp_3.pdf" width="100%"></center>
</details>
RFC-822规范了首部字段中部分field的格式和解释方式(format & interpretation)，其中首部字段中以"X"开头的field是用户自定义的字段。对于较长的首部字段如"Received"，将会以多行文本显示，除第一行外的其他行使用white space开头。

**[3] body** 正文即sender user发送给receiving user的邮件内容。RFC-822指出邮件正文的使用NVT ASCII文本格式。当使用DATA指令传输邮件时，首先发送user agent填写的代理部分，紧接是一行空白行(blank line)，最后是正文部分。需要注意，DATA指令传输的邮件正文每行都必须少于1000bytes。

**[4] body + header#1 + header#2 + envelope组装流程图**
<center><img src="/img/in-post/tcp-ip_img/smtp_6.pdf" width="100%"></center>

##### 4 中继代理(relay agent)
本文前面简单邮件案例中，邮件传输代理MTA的第一行输出如下所示：
<center><img src="/img/in-post/tcp-ip_img/smtp_7.pdf" width="100%"></center>
上诉输出信息第一行表明：sun主机的管理员已经将系统配置成：所有nolocal outgoing mail都需要经过中继代理主机(mailhost)进行转发。

**❮为什么使用中继代理主机对邮件进行转发❯** <br>
- **(1)** 使用中继代理主机对于特定organization的所有主机的邮件进行代理，可以简化除中继系统MTA的之外其他主机MTA的配置(配置一个MTA并不简单)。
- **(2)** 中继代理主机相当于一个邮件集线器(mail hub)，从而将organization中独立的子系统(individual system)隐藏起来。

关于**(1)**进行补充说明：<br>
简单邮件案例中，中继代理系统在本地域(.tuc.noao.edu)中的主机名为mailhost。使用"host"指令可以查看该主机名在individual system的主机上的DNS中的配置情况：
<center><img src="/img/in-post/tcp-ip_img/smtp_8.pdf" width="100%"></center>
如果用作中继代理的主机需要更换，只需改变所有individual system主机上DNS配置的代理主机名即可，而无需更改其他配置项。如果不采用中继代理主机，则为一个organization下所有的individual system修改邮件传输代理MTA的配置项是非常繁琐的。

**❮邮件收发两端都具有中继代理的邮件传输系统的组织结构图❯** <br>
<center><img src="/img/in-post/tcp-ip_img/smtp_9.pdf" width="70%"></center>
- 根据上述模式图：从sending host到receiving host之间一共有4个邮件传输代理。
- 发送端邮件传输代理(local MTA)只负责将邮件转交给自己的中继MTA，这种通信通过在本地(organization's local internet)网路上借助SMTP实现。接收端邮件传输方式和发送端相同。

##### 5 NVT ASCII
SMTP使用NVT ASCII格式表示所有相关的信息：邮件信封(envelope)、邮件首部(headers)、邮件正文(body)。NVT ASCII是一个7bit字符编码格式，但使用8bit数据单位进行传输，最高位bit置为0。

在本文后续部分将会介绍extended SMTP以及multimedia mail(MIME)，这些协议允许收发音频数据、视频数据。其中MIME协议格式报文在信封(envelope)、邮件首部(headers)、邮件正文(body)中同样使用NVT ASCII格式编码，只是在用户代理进程的实现上有所不同。

##### 6 重试间隔时间(retry interval)
当用户代理将新的邮件报文传给它的邮件传输代理MTA时，通常MTA会立即进行邮件投递(delivery)。如果投递失败，MTA必须将该邮件加入队列中，并在稍后进行重试。

Host Requirement RFC推荐初始的超时时间至少为30min。邮件发送方(sender)至少4～5天后可以放弃邮件发送。邮件的投递出现投递失败的情况往往时暂时的(transient)，所以当邮件报文在MTA构建的"重投"队列中等待的第一个1h时间内，尝试再次进行重连是有效用的(make sense)。

### 三：SMTP案例分析(SMTP examples)
本文前面介绍的内容说明了普通邮件的传输，本部分内容将会介绍MX记录(record)如何在邮件传输中被使用，并说明VRFY指令、EXPN指令的用法。

##### 1 Mx Record: 邮件目的主机非直接连接到internet
在chapter-14 DNS文章中，曾经提到一种在DNS中资源记录(resource record)类型：邮件交换记录(mail exchange record, MX record)。 RFC-974描述了邮件传输代理对于MX Record的处理方式。本小节案例将说明如何利用MX record向不直接连到internet上的主机发送邮件。

主机mlfarm.com并没有直接连接到internet上，但是该主机拥有一份Mx record指向位于internet上的一个邮件转发站(mail forwarder)：mercury.hsi.com。

**[1]** 在sun主机上使用host命令查询邮件目的主机mlfarm.com的MX Record(一种资源记录类型)。
<center><img src="/img/in-post/tcp-ip_img/smtp_10.pdf" width="100%"></center>
- mlfarm.com主机上有两个MX Record，即有两个邮件转发站点：mercury、hsi86，优先级从低到高。
- mlfarm.com主机上关于MX Record的附加信息包括两个邮件转发站点的ip地址。

**[2]** 在sun主机上使用mail用户交互指令向ron@mlfarm.com发送邮件，该主机不直接连接到internet上，打印相关信息观察MX Record在其中的作用。
<center><img src="/img/in-post/tcp-ip_img/smtp_11.pdf" width="100%"></center>

**[3]** 使用tcpdump程序对于sun主机相连的SLIP链路上的通信数据分组进行监测并打印相关信息。

<center><img src="/img/in-post/tcp-ip_img/smtp_12.pdf" width="100%"></center>

在本案例中，sun主机被配置成不使用其normal relay MTA，因此所有邮件的发送都由local MTA执行，因此可以在sun主机相连的SLIP链路上抓取到和目的主机的邮件传输记录；另外，sun主机还被配置使用noao.edu子网上的域名服务器(DNS)，该DNS和sun主机通过SLIP链路相连，因此可以在sun主机相连的SLIP链路上抓取到sun主机和DNS通信的分组情况。

**[4]** UUCP协议：上述案例中，从某种角度讲，mercury.hsi.com邮件转发代理主机必须将邮件投递到目的主机：mlfarm.com。UUCP(unix-to-unix copy)是一个普遍应用的协议用于不直接连接到internet上的主机和它的MX Record站点交换邮件.

**[5]** 邮件传输代理MTA和域名服务器DNS之间的交互方式取决于它们的具体实现。<br>
上面的案例中，MTA首先向DNS请求MX Record信息，然后再发送邮件到目的主机。RFC-974指出邮件传输代理MTA必须首先请求MX Record，如果得不到相应记录，则尝试直接投递邮件到目的主机(e.g. 直接向DNS查询目的主机ip地址)。<br>
与之对应的，邮件传输代理MTAs也必须能够处理DNS返回的记录中CNAME(canonical name)信息。

从BSD/386主机向rstevens@mailhost.tuc.noao.edu主机发送邮件，BSD/386主机上的MTA(Sendmail)将执行如下步骤：
- **(a)** Sendmail =\> DNS: 查询目的主机mailhost.tuc.noao.edu的CNAME记录，得到如下信息：<center><img src="/img/in-post/tcp-ip_img/smtp_13.pdf" width="100%"></center>
- **(b)** 发送一个DNS query: 查询CNAME=noao.edu的相关信息，得到的应答为："none exist"。
- **(c)** Sendmail =\> DNS: 查询目的主机CNAME=noao.edu的MX Record信息：<center><img src="/img/in-post/tcp-ip_img/smtp_14.pdf" width="100%"></center>
- **(d)** Sendmail请求DNS关于(c)中得到的MX record主机的A Record(ip地址信息)，得到应答为：140.252.1.54。PS: 这项内容一般被DNS以"additional info"的方式在请求MX Record中予以返回。
- **(e)** Sendmail启动一个到达140.252.1.54的SMTP连接，发送邮件。

补充说明：DNS返回的MX Record必须是带有A Record的主机名，因此对于返回的MX Record不必再次进行CNAME查询。

##### 2 MX Record: 邮件目的主机出现故障
MX Record的另一个用处是：当邮件目的主机出现故障时，能够提供一个"alternative mail receiver"。

**[1]** 在sun主机查询DNS表项关于sun主机自身的MX Record，将得到如下信息：
<center><img src="/img/in-post/tcp-ip_img/smtp_15.pdf" width="100%"></center>
- 优先级较低(0)的MX Record表明应当❮首先❯尝试发送邮件到sun主机本身(sun主机本身优先作为一个邮件传输代理MTA)；下一个优先的MTA是noao.edu主机。

**[2]** 将sun主机的SMTP服务进程关闭(模拟sun主机出现故障无法接受邮件)，从vangogh主机向sun主机发送邮件。交互程序如下所示：
<center><img src="/img/in-post/tcp-ip_img/smtp_16.pdf" width="100%"></center>
- 按照从MX Record优先级从低到高的顺序，依次发送连接请求。当TCP连接请求到达sun主机的两个ip地址的25端口，由于该端口未被SMTP进程监听(no process has a passive open pending for that port)，因此TCP将返回RST复位信号。

**[3]** 使用tcpdump程序打印上述邮件传输过程建立TCP连接时的数据分组交换情况。
<center><img src="/img/in-post/tcp-ip_img/smtp_17.pdf" width="100%"></center>
- SMTP client没有尝试区分当执行active open建立连接时返回的错误类型，因此它会再次换一个目的端口ip继续尝试建立连接。
- 如果第一次尝试建立连接时返回的错误信息为"host unreachable"(表明进程没有监听该端口)，则第二次尝试才是有意义的。
- 如果第一次尝试建立连接时由于"server host is down"而失败，我们将会在tcpdump输出中看到SMTP client将持续重传SYN信号到ip:140.252.1.29(for 75s)，紧接着SMTP client将持续重传SYN信号到ip:140.252.13.33(75s)。经历150s之后，才会寻找MX Record中更高优先级的目的地址进行传递。

##### 3 VRFY和EXPN命令
VRFY命令可以验证recepient地址是否合理(OK)，而无需向该地址发送邮件。EXPN命令无需向邮件列表(mail list)发送邮件就可以扩充该列表。

**[1]** 在sun主机上连接到一个较新版本的邮件转发代理MTA(Sendmail)，并使用VRFY和EXPN指令进行：地址验证、扩充特定域名主机到邮件列表两种操作。
<center><img src="/img/in-post/tcp-ip_img/smtp_18.pdf" width="100%"></center>
- 使用较新版本的Sendmail的原因是：较老版本的Sendmail对于VRFY和EXPN命令的功能不做区分。
- 关于forwarding ip：转发的ip地址，即非EXPN所推展的域名主机自身的ip地址，而是负责该域名主机邮件转发的MTA的ip地址。

**[2]** 许多网络站点(sites)禁用了VRFY和EXPN命令，部分情况由于privacy原因将其禁用，有些情况认为开启这两个命令会导致安全漏洞。

e.g. 我们可以向白宫的SMTP server使用VRFY和EXPN命令执行一些操作，并观察server返回信息：
<center><img src="/img/in-post/tcp-ip_img/smtp_19.pdf" width="100%"></center>


### 四：SMTP的未来发展(SMTP futures) - todo
##### 1 邮件信封变动：拓展SMTP(extended SMTP)
##### 2 首部变动：允许首部使用非ASCII字符(non-ASCII characters)
##### 3 正文变动：通用邮件扩充(multipurpose internet mail extensions, MIME)





## Reference
> \<tcp-ip: illustrated vol1\> chapter28 <br>

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
