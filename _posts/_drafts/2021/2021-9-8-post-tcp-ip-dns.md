---
layout: post
title: "tcp-ip: DNS (域名系统)"
subtitle: '[tcp-ip: illustrated vol1] - chapter14' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-09-08 08:07
lang: ch 
catalog: true 
categories: tcp-ip 
tags:
  - Time 2021
  - tcp-ip illustrated vol1 
---
### 一：引言
域名系统(domain name system, DNS)是一种基于TCP/IP应用程序的分布式数据库。DNS主要提供的服务包括：主机名(hostname)和IP地址之间的转换，以及为电子邮件提供路由信息。

**DNS是一个分布式系统** 每个internet上的单个站点都不能拥有全部的DNS映射信息；相反每个站点保存自己的信息数据库，并运行一个服务程序供internet上其他主机查询。

**访问DNS数据库的方式** 通过地址解析器访问DNS服务器，在unix系统中，可以通过2个API：gethostbyname()、getnamebyhost()来调用地址解析器。

**地址解析服务属于应用层** 
地址解析器应当属于应用程序层的操作，而不像TCP/IP协议一样属于操作系统内核。应用程序在尝试建立TCP连接之前或者或使用UDP发送数据报之前，需要先进行地址解析。操作系统中的TCP/IP协议簇对于DNS服务一无所知。

本文重点介绍关于使用TCP/IP协议(主要是UDP)和DNS名字服务器通信的内容，不对服务器本身接口参数等实现细节做过多描述。
RFC 1034说明了DNS的基本概念和功能。RFC-1035说明了DNS的规范和实现。

### 二：DNS基础
DNS的名字空间和unix的文件系统相似，都具备层次接口。

##### 1 域名层次图 - DNS的名字空间中的层次结构
<center><img src="/img/in-post/tcp-ip_img/dns_1.pdf" width="80%"></center>

**[1] 关于上述域名层次结构图中的注意事项** <br>
- 域名层次结构树中每个节点至多能够包含63字符(byte)长的标识。
- 域名层次结构树中的根结点是没有任何标识的根结点。
- 域名层次结构树中的任何节点均不区分大写、小写。
- 域名层次结构树中每个节点必须有唯一的域名(domain name)，但是多个节点可以有相同的标识(label)。
- 域名层次结构树中的任何节点的域就是将该节点自下而上地逐个书写(直到根节点)，不同节点之间使用"."连接起来。注意这不同于文件路径的书写方式，文件路径是从根目录依次向叶节点书写。
- 以"."结尾的域名称之为绝对域名、完全合格域名(full-qualified domain name，FQDN)，否则认为该域名是"有待补全的"。关于该域名是以何种方式被补全取决于当前运行的DNS服务软件。如果不完整的域名包含2个或更多的标识(label)，那么它可以被看作是完整的域名(full-qualified)。

**[2] 顶级域名可以分成如下3个部分** <br>
- arpa域名是一个用于完成"地址到名字"转换的特殊域。因此，当获得其他域名空间的授权时，同时也会获得in-addr.arpa的授权。
- 上面的域名层次图中从"com"到"org"的7个3字符长度的称之为普通域，每个节点的应用范围已经在图中标识出。
- 所有2字符的域都是基于ISO3166定义的国家代码，这些域被称为国家域or地理域。 

**[3] 关于域名层次结构中节点(域名)的管理** <br>
网络信息中心(network information center, NIC)负责分配顶级域名，以及将新域名申请分配权委托(delegate)给其他的特定区域的授权机构。<br>
一个被授权的独立管理的域名层次子树称之为区域(zone)，例如：大学根据不同的系进行将子树(zone)进一步划分，公司根据不同的部门对于区域进一步划分。

### 三：DNS的报文格式
##### 1 DNS查询&响应报文的通用格式
DNS服务程序定义了一种用于查询&响应通信的报文格式。下图显示了该报文的总体格式。
<center><img src="/img/in-post/tcp-ip_img/dns_2.pdf" width="60%"></center>

查询or应答报文由12byte的报文首部，以及4个长度可变的字段构成。<br>

##### 2 DNS报文的标识字段简介
报文中的标识字段由客户程序设置，并被复制到DNS服务程序返回报文中。标识字段的主要作用是在客户端DNS查询程序中，将本机发出的DNS请求和DNS应答相互匹配。

##### 3 DNS报文的标志字段详解
16bit标志字段的详细格式如下图，下面对于标志字段中的每个参数进行详细解析。
<center><img src="/img/in-post/tcp-ip_img/dns_3.pdf" width="50%"></center>

- **QR字段** 占用1bit，QR=0表示查询报文，QR=1表示响应报文。
- **opcode字段** 占用4bit，大多数情况下数值为0，表示"标准查询"；其他情况：当数值为1表示"反向查询"，当数值为2表示"请求DNS服务器状态"。
- **AA字段** 占用1bit，表示"DNS-server具备权威性的应答(authoritative answer)"。该字段可以用于表明该DNS-server在该域是权威的。
- **TC字段** 占用1bit，表示"被截断的"。用于DNS服务的UDP数据报最大支持512byteDNS报文长度，当DNS应答数据报长度超过512byte会被自动截断，设置TC字段可以表示此DNS应答已经被截断。(关于为什么提供DNS服务的UDP数据报仅支持512byte，详见本文后续部分。)
- **RD字段** 占用1bit，表示"期望收到递归应答(recursion desired)"。该标志位可以在DNS请求报文中设置，并在DNS响应报文中被返回。RD标志位告诉DNS-server必须处理这个查询报告。如果该标志位被设置为0，且被请求的DNS-server没有能力提供权威性应答，该DNS-server就返回一个用于对该请求进行应答的name-server列表。这种能够"不断寻找帮手"的请求方式称为"迭代查询"。
- **RA字段** 占用1bit，表示"支持递归查询(recursion available)"。如果DNS-server支持递归查询，那么在应答报文中将该bit为设置为1。除了某些root server以外，大多数DNS-server都支持递归查询。
- **zero字段** 占用3bit，必须设置为0。
- **rcode字段** 占用4bit，返回码字段，用于表示"domain查询的结果"。通常情况下rcode数值为0或者3，rcode=0表示no error，rcode=3表示name error。name error只会从authoritative name server上返回：表示请求报文中指定的domain不存在。

##### 4 DNS报文标志字段后面的4个16bit字段
这4个16bit"number of \*"字段分别对应后面的32bit的question、answer、authority、additional info中条目的数量。<br>
一般地，对于DNS查询报文，number of questions一般为1，而其他三项为0；对于DNS应答报文，number of answers至少为1，authority和additional info的条目数可以为0，也可以非0。

##### 5 DNS查询报文中请求部分
**[1] DNS请求部分的单个请求报文部分格式** 
<center><img src="/img/in-post/tcp-ip_img/dns_4.pdf" width="50%"></center>

通常情况下只有一项请求。其中，查询名是需要查询的名字，是由1个or多个标识符(label)构成的序列。

**[2] DNS请求中标识符(label)**  
<center><img src="/img/in-post/tcp-ip_img/dns_5.pdf" width="50%"></center>

以gemini.tuc.noao.edu为例对于标识符结构进行简析。**i** 每个标识符分为两个部分：计数部分、符号部分。1bit计数部分用于表示后面连续的符号个数。标识符使用1bit"0"字符用于表示结尾，称为根标识符。**ii** 计数的数值区间范围是0-63，因为整个标识符的最大长度为63byte。**iii** 不同于之前提到的其他报文可变字段，标识符字段不需要补0以填充至32bit的整数倍。

**[3] DNS请求中的查询类型(query type)** <br>
DNS查询报文中每个请求(question)都有一个**查询类型**字段，DNS应答报文中的每个响应(response)(也可以称为一个资源记录resource record)都有一个**类型**；注意，查询类型字段是类型字段的超集。<br>
<center><img src="/img/in-post/tcp-ip_img/dns_6.pdf" width="50%"></center>

上表中，大多数查询名既可以作为DNS请求中的**查询类型**，也可以作为DNS资源记录报文中的**类型**。表中只有最后两项查询名仅用于DNS请求，而DNS应答报文不支持此类型。<br>
最常用的查询类型是**A类型**，表示DNS请求报文希望根据标识符(label)获取对应IP地址。**PTR查询类型**表明，DNS请求报文希望根据IP地址获取对应域名。

##### 6 DNS响应报文中资源记录部分(回答、授权、额外记录)
DNS报文后面3个字段回答、授权、额外记录都采用如下称之为"资源记录(resource record)的格式"。
<center><img src="/img/in-post/tcp-ip_img/dns_7.pdf" width="50%"></center>

**[1] 3种类型的资源应答** 
- **应答类型资源记录(answer)** 包含DNS应答数据报中的关于DNS查询报文中所请求的域名对应的IP地址等信息。
- **授权信息类型资源记录(authority)** 表示DNS服务器返回的"能够提供权威性应答的"名字服务器列表。
- **额外信息类型资源记录(additional)** 表示DNS服务器返回的"和上述权威性应答的额外补充信息"，e.g.包括上述权威性应答DNS服务器的IP地址。

**[2] 资源应答通用格式中的部分核心字段含义简介**
- **域名(domain name)** 是整个资源记录中资源数据对应的名称。域名的数据格式和DNS请求报文中查询名的格式相同。
- **类型(type)** 详见上一节的DNS请求报文中"response type字段"表格内容中的相关说明。
- **类(class)** 一般来说，对于internet数据而言，类字段数值为1。
- **生存时间字段(time to live)** 表示客户端程序保留该资源记录的秒数。资源记录通常的生存时间为2天。
- **资源数据长度(RRs length)** 说明了资源数据部分数据的数量。
- **资源数据(RRs)** 数据部分的存储格式取决于类型(type)，对于"A类型"，资源数据为4byteIP地址。

### 四：案例分析(name resolver和name server之间通信过程)
##### 1 telnet远程连接命令行程序示例
在sun主机上运行telnet客户程序远程登录到gemini主机上，并连接该主机gemini服务程序。相应命令行程序如下。
<center><img src="/img/in-post/tcp-ip_img/dns_8.pdf" width="100%"></center>

注意，上述过程中使用telnet程序链接gemini主机时，使用的是主机名，因此在建立链接之前需要借助位于noao.edu的name server提供的DNS"A类型"服务获取目标主机gemini的IP地址。

##### 2 DNS name resolver和name server以及目标主机三者结构关系图
<center><img src="/img/in-post/tcp-ip_img/dns_9.pdf" width="60%"></center>

- DNS名字解析是客户端程序(应用层)程序的一部分，在sun主机telnet客户端程序和gemini主机的daytime服务程序建立TCP连接之前，就需要通过name resolver获取目的gemini主机的IP地址。
- sun主机达到以太网140.252.1.0是通过SLIP线路、路由器netb、以及两个调制解调器完成的。该线路部分并不影响关于DNS查询&应答服务的讨论，因此本案例可以忽略不计。

##### 3 系统特殊文件"/etc/resolv.conf"和名字解析
系统特殊文件"/etc/resolv.conf"主要包含两种涉及名字解析服务的信息：名字解析服务器(name server)的IP地址、默认域名(domain)。<br>
<center><img src="/img/in-post/tcp-ip_img/dns_10.pdf" width="100%"></center>

**(i)** 名字解析服务器地址最多在文件中提供3个，用于防止网络的DNS名字解析服务器发生错误or不可达。**(ii)** 当应用程序中查询的域名不是一个完整的域名(没有以"."作为结束)，则默认的域名(tuc.noao.edu.)将会追加到待查询的域名后。例如："telnet gemini"，变成"telnet gemini.tuc.noao.edu"；名字解析程序在待查询名字后加上句点以指明它是一个绝对字段名(完整性get)。

##### 4 使用tcpdump抓包工具分析名字解析&名字服务程序之间通信的分组交换
配置tcpdump程序，使其不再显示每个IP数据报(网络层)的源地址和目的地址，而是只显示名字解析器的IP地址和名字服务器的IP地址。客户端程序使用临时端口号1447，而提供名字解析服务的服务程序采用公认的标准DNS服务端口53。

<center><img src="/img/in-post/tcp-ip_img/dns_11.pdf" width="100%"></center>

##### 5 使用host程序向名字服务器发送查询信息并打印结果(A类型查询)
host打印查询得到的结果如下图所示。host在名字服务器查询得到两个结果，这是由于名字服务器为多端口主机导致的，即对于同一份查询报告，名字服务器通过所有支持DNS服务的端口各返回一份应答。
<center><img src="/img/in-post/tcp-ip_img/dns_12.pdf" width="100%"></center>

##### 6 关于tcpdump打印的DNS应答报文UDP数据报为什么是69byte进行解析
**[1] tcpdump程序捕获的IP报文格式分析图** 
<center><img src="/img/in-post/tcp-ip_img/dns_13.pdf" width="100%"></center>

在分析tcpdump所打印的DNS应答数据长为69byte的原因，需要注意如下2点：**(i)** 在DNS应答部分包含DNS请求中的查询查询名(question)部分。**(ii)** 在DNS应答报文结果中有很多重复的域名项，DNS应答使用一种"压缩"机制来简化表示这些重复的域名项。压缩方式中存储的域名标识符占用2byte而不是存储完整域名的21byte。

**[2] 压缩方式简单介绍** 
- **(1)** 域名需要使用21byte存储空间进行存储，如果遇到重复的域名，可以仅使用2byte进行存储。
- **(2)** 将2byte=16bit中的高2bit置1。这2bit仅代表一个"符号"(此2byte为重复域名指针而不是一个域名的计数字节)。
- **(3)** 将剩余14bit空间用于存放该"指针"。该"指针的数值是"指向的DNS标识符(label)的偏移值，域名区域第一个byte的偏移值=0。
- **(4)** 只要域名中一个标识符(label)被压缩，就可以使用这种指针压缩方式，不需要整个域名中的标识符都被压缩才应用。

### 五：指针查询(给定一个IP地址返回其域名)
##### 1 域名空间的申请&相应DNS名字的书写
- **域名空间的授权** 当一个组织加入internet，获得DNS域名空间"noao.edu"的授权，则它将同时获得in-addr.arpa域名的授权(用于完成地址到域名之间的转换)。
- **DNS名字的书写** "noao.edu"对应的IP地址网络号为"140.252"(一个B类网络)。需要注意的是，DNS名字的书写是从DNS树的底部逐步向上书写的。因此"noao.edu"的某台IP地址为"140.252.13.33"的主机的DNS名为"33.13.252.140.in-addr.arpa"。
- **DNS名字自底而上书写的原因** 如果DNS的FQDN名字自上而下进行书写，则该IP地址的名字将为"arpa.in=addr.140.252.13.33"，因此通过DNS名字解析服务得到的域名为"edu.noao.tuc.sun"。
- **arpa分支的特殊作用** 如果DNS名字解析树中没有用于从"IP地址-\>DNS域名"的独立分支，那么DNS反向域名解析将会变得异常困难。如果将独立的arpa分支删除，那么已知IP地址的情况下，为了获得相应域名，需要从DNS域名树的根结点开始，逐个进行尝试匹配。

##### 2 反向DNS查询 - PTR类型查询(从IP地址-\>域名)的基本流程
**#1 DNS解析首先将IP地址反向，将IP地址转变成DNS域名地址：** The DNS resolver reverses the IP, and adds it to ".in-addr.arpa" (or ".ip6.arpa" for IPv6 lookups), turning 192.0.2.25 into 25.2.0.192.in-addr.arpa. 

**#2 DNS解析器根据获得的域名查询PTR记录以获取其IP地址：** The DNS resolver then looks up the PTR record for 25.2.0.192.in-addr.arpa.
- **#2.1 DNS解析器向DNS根服务器请求PTR记录信息：** The DNS resolver asks the root servers for the PTR record for 25.2.0.192.in-addr.arpa.
- **#2.2 DNS根服务器将DNS解析器指引到负责192域名的DNS子服务器上：** The root servers refer the DNS resolver to the DNS servers in charge of the Class A range (192.in-addr.arpa, which covers all IPs that begin with 192).
- **#2.3 大部分情况下，根服务器将DNS解析器指引到负责分配IP地址的RIR上：** In almost all cases, the root servers will refer the DNS resolver to a "RIR" ("Regional Internet Registry"). These are the organizations that allocate IPs. E.g. ARIN handles North American IPs, APNIC handles Asian-Pacific IPs, and RIPE handles European IPs.
- **#2.4 DNS解析器将会向e.g. ARIN请求PTR记录信息：** The DNS resolver will ask the ARIN DNS servers for the PTR record for 25.2.0.192.in-addr.arpa.
- **#2.5 DNS解析器将会被RIR指引到网络服务供应商ISP的DNS服务器上：** The ARIN DNS servers will refer the DNS resolver to the DNS servers of the organization that was originally given the IP range. These are usually the DNS servers of your ISP, or their bandwidth provider.
- **#2.6 DNS解析器将会向ISP-DNS服务器请求PTR记录信息** The DNS resolver will ask the ISP's DNS servers for the PTR record for 25.2.0.192.in-addr.arpa.
- **#2.7 网络服务供应商DNS服务器将指引DNS解析器到其管理的DNS服务器上：** The ISP's DNS servers will refer the DNS resolver to the organization's DNS servers.
- **#2.8 DNS解析器将会向#2.7中导向的DNS服务器请求PTR相关记录：** The DNS resolver will ask the organization's DNS servers for the PTR record for 25.2.0.192.in-addr.arpa.
- **#2.9 该DNS服务器返回最终完整域名应答：** The organization's DNS servers will respond with "host.example.com".

##### 3 一个简单例子(PTR查询)
**[1] 在sun主机上使用host完成指针查询** 称之为指针查询是因为：在借助IP地址查询相应的域名时，需要查询DNS- server内部存储的"IP-域名指针表"。通过逐级向下查询的方式完成从IP地址到域名的转换。
<center><img src="/img/in-post/tcp-ip_img/dns_14.pdf" width="100%"></center>
**[2] 使用tcpdump程序分析上述PTR查询的数据包细节**
<center><img src="/img/in-post/tcp-ip_img/dns_15.pdf" width="100%"></center>

##### 4 主机名检查(待补充)

### 六：资源记录(resource records)
本文之前的部分已经介绍了几种常用的资源记录类型：IP地址查询(域名-\>IP)：A类型，指针查询(IP-\>域名)：PTR类型；以及名字服务器返回的资源记录：应答RR，授权RR，附加信息RR记录。一些典型的RR类型记录如下表所示。
<center><img src="/img/in-post/tcp-ip_img/dns_16.pdf" width="70%"></center>

### 七：DNS高速缓存
为了减少internet上的互联网通信量，所有名字服务器(name server)都使用DNS高速缓存。

##### 1 为什么DNS高速缓存放在DNS服务器上
在标准的Unix实现中，高速缓存是由名字服务器维护的，而不是由名字解析器维护。其原因是：名字解析器是每个应用程序的一部分，而应用程序又不可能总是处于工作状态；DNS服务程序是系统的一部分，将DNS高速缓存存放在只要系统处于工作状态就能起作用的程序中是很重要的。

##### 2 sun主机同时运行DNS解析器和DNS服务器以观察DNS高速缓存(案例)
之前的案例中，DNS名字解析器和DNS服务器运行在不同的主机上，而本案例，使用sun主机运行名字服务器，并在sun主机上通过应用程序发送DNS查询报文。<br>
使用tcpdump程序监测SLIP链路上的通信数据报情况，应当只能看到因为超出sun主机本地DNS高速缓存而处理不了的查询数据报。

**[1] 配置sun主机使DNS解析器优先使用本地名字服务器** 
<center><img src="/img/in-post/tcp-ip_img/dns_17.pdf" width="100%"></center>

**[2] 使用host命令查询"ftp.uu.net"域名对应的IP地址(查询DNS高速缓存)** 
<center><img src="/img/in-post/tcp-ip_img/dns_18.pdf" width="100%"></center>

**[3] 使用"tcpdump -w"收集进出53号端口的数据** <br>
通过如下步骤执行相关命令以获得如下图所示的"与sun主机相连的SLIP链路上的、关于DNS请求&应答的"的数据报抓包结果。
- 使用"tcpdump -w"命令，除了能够将tcpdump收集的数据报存在一个文件中，以供后续分析；还可以防止tcpdump程序自身调用name resolver以显示和待查询域名对应的IP信息。
- 执行完"tcpdump -w"命令后，tcpdump启动了一个后台程序以实时监测指定SLIP链路上的数据报收发情况。
- 执行host命令查询本地高速缓存。该查询&应答过程涉及到DNS数据包的收发。本地DNS高速缓存中存在的条目可以直接通过sun主机的名字服务程序提供；对于本地没有的条目，则需要借助SLIP链路向网络上其他DNS名字服务器进行查询。
- DNS查询host命令完成后，运行"tcpdump -r"命令以读取刚才保存的原始数据记录文件，并产生正式输出显示。<center><img src="/img/in-post/tcp-ip_img/dns_19.pdf" width="100%"></center>

- 在上述步骤完成后，再次调用host命令执行查询相同的域名("ftp.uu.net")的IP地址信息，并再次使用"tcpdump"命令打印SLIP链路上的输出，得到的结果为空。这是因为DNS名字解析器直接从本地DNS高速缓存中读取相应域名信息，而无需借助于以太网上其他名字服务器。

**[4] 使用host命令显示host命令请求的10个资源记录(5个授权资源记录+5个附加信息记录)** 
<center><img src="/img/in-post/tcp-ip_img/dns_20.pdf" width="100%"></center>

### 七：DNS报文使用TCP协议还是UDP协议作为载体?
DNS域名服务同时支持使用TCP、UDP进行通信，并且无论使用TCP还是UDP作为信息载体，都使用相同的端口号53。TCP和UDP具有不同的特点，下面讨论使用两种方式的各自特点。

##### 1 为什么提供DNS服务的UDP数据报仅支持512byteDNS报文长度 - 参考[serverfault](https://serverfault.com/questions/587625/why-dns-through-udp-has-a-512-bytes-limit)

**[1] 限制提供DNS服务的UDP数据报DNS报文部分不超过512byte的直接原因**
>  The 512 byte payload guarantees that DNS packets can be reassembled if fragmented in transit. Also, generally speaking there's less chance of smaller packets being randomly dropped. 

**[2] 为什么限制DNS报文部分为512byte就可以保证DNS数据报在分片传递中可以被reassemble**
>  The IPv4 standard specifies that every host must be able to reassemble packets of 576 bytes or less. With an IPv4 header (20 bytes, though it can be as high as 60 bytes options) and an 8 byte UDP header, a DNS packet with a 512 byte payload will be smaller than 576 bytes. IPv4标准指明每个主机必须能够重组最大为576byte长度的数据报。576byte \> 20byte(IP首部) + (8byteUDP首部) + (512byteDNS数据 - 需为16bit的整数倍)。

##### 2 使用TCP进行DNS服务通信
- 当名字解析器发送一个请求，得到的响应长度超过512byte时，如果返回响应报文中TC字段(删减标志)被设置为1，则将响应报文超出512byte的部分进行截断。此时，名字解析器可以使用TCP重发该请求，TCP可以对于长度超过512byte的报文进行分段，因此TCP将允许返回的应答报文长度超过512byte(理论上是任意长度)。 
- 当负责特定域的辅助名字服务器启动时，将从管理该域的主名字服务器进行"zone transfer"操作(process of copying the zone file from primary DNS server to secondary DNS server)。此时传送的文件一般较大，因此使用TCP进行传送。
- DNS主要使用UDP进行数据传送。其他依赖于UDP的大部分应用如TFTP、BOOTP、SNMP，大部分操作集中在局域网内，而DNS服务(查询&响应)经常借助广域网，因此采用UDP通信的DNS服务需要在相关应用程序层面(名字解析器)、内核相关程序上(名字服务器)提供"报文超时处理"、"报文重传"的服务。

### 八：两个应用程序(Rlogin)远程登录涉及的DNS通信完整流程(案例)
启动一个Rlogin客户端程序，然后连接到其他域中另一个Rlogin服务器程序，对于该过程中发生的分组交换流程进行分析。注意，本案例所有数据分组交换过程不考虑使用任何DNS高速缓存。

##### 1 案例涉及的分组交换关系图表
<center><img src="/img/in-post/tcp-ip_img/dns_21.pdf" width="70%"></center>

##### 2 案例涉及的分组交换关系流程简析
- **#1** Rlogin客户端程序启动后，向根名字服务器发送A类型查询请求报文。(主机域名-\>对应IP地址)。
- **#2** 根域名服务器返回NS类型DNS响应报文(不应该设置递归查询bit)。该报文包含**Rlogin程序服务器**所在的域的名字服务器的信息。
- **#3** Rlogin客户端程序向#2返回的名字服务器重新发送上述A类型查询请求报文(一般需要设置递归查询bit=1)。
- **#4** 名字服务器返回A类型应答。应答报文中包含Rlogin客户端程序所查询的域名对应的IP地址。
- **#5** Rlogin客户端程序和服务端程序使用TCP协议进行通信。首先需要建立TCP连接，需要发送3封数据报(3次握手)。
- **#6** Rlogin服务端程序收到Rlogin客户端程序发送的连接请求后，调用名字服务器上的名字解析器通过收到的数据报中的IP地址获取Rlogin客户端主机名。这是一个PTR指针查询请求，该请求需要被根域名服务器处理，且该服务器可不同于#2中的根域名服务器。
- **#7** #6中的根域名服务器返回NS类型应答报文。该报文包含特定的名字服务器信息：这些名字服务器负责管理**Rlogin程序客户端**"in-addr.arpa"域。
- **#8** Rlogin服务端程序向#7返回的名字服务器重新发送PTR类型查询报文。
- **#9** 名字服务器返回PTR类型应答报文。该报文包含关于Rlogin客户端的FQDN(full-qualified domain name)。
- **#10** Rlogin服务端程序向负责Rlogin客户端的名字服务器发送A类型查询请求报文，查询#9中返回的FQDN对应的IP地址。这个流程可能会被#9中应用的"gethostbyaddr()"函数自动完成。
- **#11** 负责Rlogin客户端的名字服务器向Rlogin服务端程序返回A类型应答报文。Rlogin服务端程序将返回的IP地址和PTR指针查询返回的地址进行比较。
- **补充** 使用DNS高速缓存将会减少部分上述#1-#11的DNS数据报分组交换流程。


## Reference
> \<tcp-ip: illustrated vol1\> chapter14 <br>
> https://serverfault.com/questions/587625/why-dns-through-udp-has-a-512-bytes-limit

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
