---
layout: post
title: "tcp-ip: SNMP (简单网络管理协议)"
subtitle: '[tcp-ip: illustrated vol1] - chapter25' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: true 
date: 2021-10-29 10:57
lang: ch 
catalog: true 
categories: tcp-ip 
tags:
  - Time 2021
  - tcp-ip illustrated vol1 
---
### 一：引言
网络飞速发展，网络的数量越来越多。网络中的设备可能来自不同的厂家，如何管理这些设备显得十分重要。本文介绍的SNMP(simple network management protocol)就是一个管理这些设备的标准。

基于TCP/IP的网络管理包含两个部分：network managemnt station(网络管理站or管理进程)、network elements(被管理的网络单元or网络设备)。network elements包括：路由器、X终端、终端服务器、打印机等，它们都需要运行TCP/IP协议。

##### 1 SNMP协议管理下的网络管理进程和客户端进程通信模式图
<center><img src="/img/in-post/tcp-ip_img/snmp_1.pdf" width="100%"></center>

##### 2 基于TCP/IP的网络管理的基本组成部分(SNMP-v1)
- 管理信息库(management information base, MIB)。MIB指定了所有代理进程可以被管理进程设置or查询的参数。RFC-1213定义了第二版MIB(MIB-II)。
- 关于MIB中包含的参数的一系列通用structures和identification scheme，称为管理信息结构(structure of management information, SMI)，在RFC-1155中定义。e.g. SMI定义了一个计数器，它的计数范围是0~4,294,967,295，达到最大值后再从头开始计数。
- 管理进程(manager)和代理进程(element)之间的通信协议：简单网络管理协议SNMP，在RFC-1157中定义。SNMP协议对于两端进程之间通信的数据包格式进行了详细的定义。SNMP最常使用的是UDP协议(Although a wide variety of transport protocols can be used)。

关于SNMP-v2的内容将在本文后续部分进行介绍。

##### 3 本文梗概
首先介绍manager和agent之间通信的协议，然后讨论MIB中定义的由agent维护的参数的数据类型。

### 二：SNMP协议
##### 1 SNMP协议5种操作示意图
<center><img src="/img/in-post/tcp-ip_img/snmp_2.pdf" width="60%"></center>
- **1) get-request** [manager-\>agent]：从代理进程中提取1或多个参数的数值。
- **2) get-next-request** [manager-\>agent]：从代理进程处提取1或多个参数的**下一个**参数值。
- **3) set-request** [manager-\>agent]：设置代理进程中的1或多个参数的数值。
- **4) get-response** [agent-\>manager + response to (3)]：返回1或多个参数的数值。
- **5) trap** [agent-\>manager/active notify]：when something happens on the agent，通知管理进程。
- 如上图所示，管理进程完成1～3操作时采用UDP-161端口，代理进程主动发出trap操作时，采用UDP-162端口。由于管理进程和代理进程使用不同的端口号，因此允许同一系统同时运行管理进程、代理进程。

##### 2 五种SNMP报文的数据格式(encapsulated in UDP datagram)
<center><img src="/img/in-post/tcp-ip_img/snmp_3.pdf" width="100%"></center>
- 图中仅对IP、UDP首部长度进行标注，SNMP报文部分的编码使用了ASN.1和BER，因此报文的长度取决于参数的类型和数值。关于ASN.1和BER的内容在本文后续部分介绍。下面分别介绍SNMP字段的格式。
- **version(0)** 该字段数值由SNMP版本号-1得到，version(0)表示SNMP-v1。
- **community** 该字段是manager和agent之间的明文密钥(cleartext password)，通常取值为"public"。
- **request ID** 对于"get"、"get-next"、"set"三个操作，request ID数值被manager设定。agent在返回的"get-response"中设置相应request ID数值。该字段允许client中的管理进程将"发出的查询(request)"和"server中的代理进程响应的应答(response)"相互匹配。该字段的设置允许管理进程对于1或多个代理进程发送多个请求，并区分收到的应答的对应情况。
- **error index** 当错误发生后，error index表示"发生错误的参数的整数偏移值"。该字段由代理进程标注，并且只在发生"noSuchName"、"readOnly"、"badValue"差错时才进行标注。
- **补充说明** "get"、"get-next"、"set"三种PDU操作类型的SNMP数据报文中，包含参数名称(name)以及参数数值(value)的map。
- **trap类型SNMP报文(PDU=4)** trap类型对应的SMNP报文格式和上面4种相比有所不同，将在本文后续部分单独整理介绍。

##### 3 两种SNMP报文标识字段的"数值\<-\>类型"对应关系
<center><img src="/img/in-post/tcp-ip_img/snmp_4.pdf" width="100%"></center>
- **PDU(protocol data unit) type** PDU is fancy word of packet。上图左侧子图展示了PDU数值和相应SNMP packet的对应关系。
- **error status** 该字段表示代理进程返回的标识特定类型错误的整数。上图右侧子图展示了error status数值和相应错误类型的对应关系。


### 三：管理信息结构(structure of management information)
本部分内容主要讨论SNMP中的涉及的数据类型，但是不会讨论这些数据类型是如何进行编码的(bit pattern to encode the data)。

##### [data types]
- **INTEGER** 一部分变量定义为"integer with no restrictions"(e.g. 接口的最大传输单元MTU)；一些变量定义为"integer taking on special values"(e.g. IP的转发标志只有1:允许转发 & 2:不允许转发)；一些变量被定义为"integer with minimum & maximum"(e.g. UDP&TCP的端口号介于0～65535范围内)。
- **OCTER STRING** "a string of 0 or more 8-bit bytes"。其中每个byte的数值都介于0～255之间。在针对OCTER STRING数据类型的BER编码中，"a count of number of the bytes precedes the string"。这种string数据类型不是"空字符结尾的数据类型(null-terminated string)"。
- **DisplayString** "a string of 0 or more 8-bit bytes"。其中每个byte必须取自NVT ASCII字符集。在MIB-II(管理信息库)规范中，这种类型的参数的长度范围介于0~255byte之间。
- **OBJECT IDENTIFIER** 在本文后面部分详细介绍。
- **NULL** 具有这种数据类型的参数为"no value variable"。e.g. 这种数据类型被用于：所有get/get-next request操作中查询的所有参数的数值，因为这些参数正在被查询，尚未确定。
- **IpAddress** "an OCTER STRING of length 4"，按照网络字节序表示的IP地址。其中每个字节表示IP地址中的一个字段。
- **PhysAddress** "an OCTER STRING for physical address"。e.g. a 6-byte以太网地址。
- **Counter** "non-negative integer [$0,2^{32}-1$]"。其数值单调增加(monotonically increase)，并在达到极值后归0。
- **Gauge** "non-negative integer [$0,2^{32}-1$]"。其数值可以增加or减少，但是达到极值后锁定，即当数值达到$2^{32}-1$后锁定，直到收到reset信号。e.g. MIB参数tcpCurrEstab：表示当前处于"ESTABLISHED状态"or"CLOSE_WAIT状态"的TCP连接个数。
- **TimeTicks** 一个时间计数器，按照0.01s的计数单位计算从特定epoch开始的时间。不同的参数在不同的epoch开始启用此计数器，所以MIB在声明参数时，需要指定"the epoch used for each of these variables"。e.g. sysUpTime表示代理进程处于"UP状态"的0.01s时间的数量。
- **SEQUENCE** 一个类似c语言中"structure"关键字的数据类型。一个SEQUENCE包含0或多个元素，每个元素亦是ASN.1数据类型。e.g. MIB中的"UdpEntry"(表示代理进程中目前处于"active"的UDP数量，active表示"ports in use by an app")。"UdpEntry"中包含两个元素：1) udpLocalAddress(IpAddress类型)表示IP地址；2) udpLocalPort(INTEGER类型0～65535)表示端口号。
- **SEQUENCE OF** 表示一种向量的数据类型，向量中的元素都具有相同的类型。e.g. 1) 如果每个元素都是INTEGER类型，那么就得到一个简单向量(1D)；2) SNMP报文中的SEQUENCE OF数据类型中，每个元素都是SEQUENCE结构，因而可以被认为是一个2D数组or表。udpTable的UDP监听表就是这种数据类型结构(如下图所示)：

<center><img src="/img/in-post/tcp-ip_img/snmp_5.pdf" width="70%"></center>
- **关于SEQUENCE OF数据类型的补充说明** 在SNMP数据报中，并不会显式指明该数据类型包含的SEQUENCE结构数量。本文后续部分将会介绍"get-next"操作如何判断已经遍历到表中最后一行；另外将会介绍管理进程如何对该数据结构的特定行进行get/set操作。


### 四：对象标识符(object identifier)
对象标识符是一种数据类型，用于标识"authoritatively named object"，其中"authoritatively"表示这些标识符是由相应权威组织进行管理&分配。

对象标识是一个整数序列，以"."分隔。不同的层次的标识部分构成一种树形结构(类似DNS和unix文件系统)。

##### SNMP通信报文所使用的对象标识符树形结构
<center><img src="/img/in-post/tcp-ip_img/snmp_6.pdf" width="70%"></center>
- **(1)** 所有的MIB变量都从1.3.6.2.1这个对象标识符开始。
- **(2)** 树形结构的对象标识符的每一个节点对应了一个"textual name"：e.g. 1.3.6.2.1对应了iso.org.dod.internet.memt.mib。使用"textual name"的原因是方便人们进行阅读。
- **(3)** 注意：上图中除了给出MIB的对象标识之外，还给出了iso.org.dod.internet.private.enterprises(1.3.6.1.4.1)这个标识。这是"vendor-specific MIB"对象标识符，在Assigned Number RFC中给出了在该节点下约400个标识的信息。


### 五：管理信息库#1 - 介绍(introduction to management infomation base)
所谓管理信息库(MIB)：即被代理进程(agent)维护的、能够被管理进程查询(query)和设置(set)的数据库信息。

如上一节中介绍的"SNMP通信报文使用的对象标识符树形结构"图所示，MIB被分成若干个组，如：system、interfaces、at(address translation)、ip...

在本节中主要讨论udp(7)组中的参数；下一节将以udp(7)组为例，详细讲解什么是实例标识(instance identification)、什么是字典排序(lexicographic ordering)、以及相关的案例；后续部分将会介绍MIB中其他组的内容。

##### 1 udp组的对象标识符树形结构
<center><img src="/img/in-post/tcp-ip_img/snmp_7.pdf" width="80%"></center>

##### 2 udp组下的简单参数(simple variables) - 数据类型、参数描述
<center><img src="/img/in-post/tcp-ip_img/snmp_8.pdf" width="100%"></center>
- 对于本文中所有的MIB参数，都采用上表中的格式进行描述。
- **R/W** 如果特定参数的R/W数值为空，则表示该参数是"只读(read-only)"；如果R/W数值为"."，则表示该参数是"可读写(read-write)"。
- 上表中数据类型都是"Counter"，表明这4个参数都是计数器类型。

##### 3 udpTable组下的简单参数(simple variables) - 数据类型、参数描述
<center><img src="/img/in-post/tcp-ip_img/snmp_9.pdf" width="100%"></center>
- 如果变量类型为INTEGER，则数据类型列将标识其上下限。
- 当我们按照SNMP表格形式来描述MIB参数时，表格第一行表示索引值(value of the "index")，"used to reference each row of the table"。

##### 4 case diagrams(udp组的案例图)
<center><img src="/img/in-post/tcp-ip_img/snmp_10.pdf" width="100%"></center>
- 从上图参数关系易知udpInDatagrams不包括udpNoPorts和udpInErrors两部分。
- 另外，上图印证了数据分组的所有流通的路径都是被计数的。


### 六：实例标识(instance identification)
##### 1 为什么需要使用实例标识
在MIB对参数进行操作时(e.g. query & set)，必须先对MIB的每个参数进行标识。
- **(1)** 在MIB树形结构中，只有叶子结点是可操作的。
- **(2)** SNMP无法操作参数的数据结构表中的一整行or一整列，即SNMP无法同时操作MIB树形结构中的多个叶子结点、亦无法同时操作树形结构到达特定叶子结点路径上的所有结点。

##### 2 简单参数数据结构的实例标识(simple variables)
<center><img src="/img/in-post/tcp-ip_img/snmp_11.pdf" width="100%"></center>
- 以"udpInDatagrams"参数为例，展示其对象标识、实例标识及其缩写方式。

##### 3 表格数据结构(SEQUENCE OF)的实例标识(tables)
表格的实例标识相对简单数据结构要复杂，回顾udp组树形结构图。

MIB数据结构表中每个参数都至少指明≥1个索引。对于"udp listener table"而言，MIB定义了包含两个变量的索引，udp listener table的实例标识如下图所示：
<center><img src="/img/in-post/tcp-ip_img/snmp_12.pdf" width="40%"></center>
- 实例表中有3个数据成员，每个数据成员具有udpLocalAddress和udpLocalPort两个索引值。
- udp listener table的上述实例表明：该监听系统将在67(BOOTP服务器)、161(SNMP)、520(RIP)接收来自任何端口的数据报。

##### 4 MIB参数表格数据结构按照字典序排序(Lexicographic ordering)
<center><img src="/img/in-post/tcp-ip_img/snmp_13.pdf" width="100%"></center>
- MIB按照对象标识符进行排序时有个隐含的顺序，即MIB表格数据结构是根据其对象标识(object identifier)按照字典序进行排序的。
- 左侧子图是按照MIB中表格数据结构的行(row)进行排序的，右侧子图是按照MIB中表格数据结构的列(column)进行排序的(即字典序)。

根据上面"Lexicographic udp listener table"可以得出如下两点结论：
- **(1)** 字典排序(Lexicographic)的MIB数据结构表格中每一行包含：给定参数的(e.g. udpLocalAddress)所有实例。
- **(2)** 字典排序(Lexicographic)的MIB数据结构表格中，每一行中给定参数的所有实例的顺序和其索引取值有关：e.g. 右侧子图中第一行端口号的字典序为"67\<161\<520"。

从MIB参数对象实例表 -\> 得到字典序MIB数据结构表格的方式
<center><img src="/img/in-post/tcp-ip_img/snmp_14.pdf" width="80%"></center>


### 七：SNMP应用的简单案例(simple examples)
本小节将会展示对SNMP代理进程中的参数的数值进行查询的案例，用于参数查询的软件属于ISODE系统，称为snmpi。[Rose 1994]对ISODE & snmpi有详细的介绍。

##### 1 snmpi获取简单参数的数值(simple variables value)
在client端sun主机上使用snmpi管理进程向server端gateway路由器查询2个udp组参数的数值：
<center><img src="/img/in-post/tcp-ip_img/snmp_15.pdf" width="100%"></center>
- community(共同体)字段是一个client(snmpi:the manager process)提供的cleartext password，如果server(agent process)能够识别community，则代理进程能够响应管理进程发出的参数查询。
- server端的某个代理进程能够让具有特定community名的client端管理进程"read-only access"，具有另一community名的client端管理进程进行"read-write access"。
- snmpi程序可以接受"get [variable name]"格式的命令，并将其转化成SNMP中的"get-request"操作报文。

通过tcpdump程序打印上述案例分组交换信息：
<center><img src="/img/in-post/tcp-ip_img/snmp_16.pdf" width="100%"></center>

##### 2 应用"get-next"操作的案例
<center><img src="/img/in-post/tcp-ip_img/snmp_17.pdf" width="100%"></center>
- "get-next-request(get-next)"操作是基于MIB字典序的，即根据MIB字典序来获取第一个参数、下一个参数。
- "get-next"操作总是会同时返回参数的名称和取值。
- 采用这种查询方式，可以允许client端管理进程：只需运行一个简单的loop程序，就可以实现从server端的MIB树根节点开始，对代理进程的参数维护的所有参数逐一进行查询。
- 采用这种查询方式，还可以实现对于表格数据结构(SEQUENCE OF)中的实例进行遍历。表格数据结构中的实例是按照字典序排列的。

##### 3 表格的访问(table access)
<center><img src="/img/in-post/tcp-ip_img/snmp_18.pdf" width="100%"></center>
- 使用client端管理进程snmpi按照字典序遍历查询udp listener table。
- udp listener table的实例详见本文前一小节[udp listener table实例标识](http://localhost:4000/tcp-ip/2021/10/29/post-tcp-ip-tcp-chapter-25/#3-表格数据结构sequence-of的实例标识tables)。
- 字典序遍历即将上述表格数据结构实例按照"先列后行"的顺序进行遍历，可以参考本文前一小节[字典序遍历udp listener table](http://localhost:4000/tcp-ip/2021/10/29/post-tcp-ip-tcp-chapter-25/#4-mib参数表格数据结构按照字典序排序lexicographic-ordering)的图进行理解。
- 关于client管理进程如何判断是否到达表格实例中的最后一行，参考上图注释部分给出的解释。


### 八：管理信息库#2 - 续(MIB continued)
这部分内容主要介绍如下MIB组：system(系统标识system identification)、if(接口interfaces)、at(地址转换address translation)、ip、icmp、tcp。

##### 1 system组
(a) system组中的7个简单参数及其描述如下表所示：
<center><img src="/img/in-post/tcp-ip_img/snmp_19.pdf" width="100%"></center>
- system对象标识符在"internet.private.enterprises(1.3.6.1.4.1)"组中。

(b) 在client端sun主机上运行snmpi管理进程，向netb路由器查询system组的4个简单参数：
<center><img src="/img/in-post/tcp-ip_img/snmp_20.pdf" width="100%"></center>
- 从上述查询结构分析：netb路由器的system组信息是0x04(internet)和0x08(end-to-end transport)的并集。这表明该路由器代理进程支持网络层(e.g. IP选路)，支持运输层(e.g. 端到端传输)。

##### 2 interface(if组)
(a) interface组定义的简单参数(simple variables) =>
<center><img src="/img/in-post/tcp-ip_img/snmp_21.pdf" width="70%"></center>
- ifNumber：interface组的简单参数，表示系统的接口数量。后续表格数据结构参数中包含ifIndex项，ifIndex介于[1,ifNumber]之间。

(b) interface组定义的表格数据结构参数:"ifTable" =>
<center><img src="/img/in-post/tcp-ip_img/snmp_22.pdf" width="100%"></center>
- 对于系统中的每个interface组的实例(即每个组内每个物理接口)都维护一个"SEQUENCE OF"数据类型的参数：ifTable。

(c) 在client端sun主机上运行管理进程snmpi可以查询这些接口组中的ifTabel表格中特定参数的取值 =>
<center><img src="/img/in-post/tcp-ip_img/snmp_23.pdf" width="100%"></center>
- 从snmpi输出可知sun主机共包含3个interface：le0、s10、lo0。这一点可以在sun主机上执行"/usr/etc/ifconfig -a"命令来进行验证。
- 根据上述"get-next"命令，并结合本小节(b)部分中的ifTable表格结构能够对于"get-next"按照"字典序(先列后行：优先遍历同一列的元素，除非到达最后一列)"遍历表格的方式进行理解：使用"get-next"命令返回的不是同一行中下一个参数(Datatype)，而是返回同一列中下一行的参数(Name)。

##### 3 at组(address translation组)
地址转换组对于所有系统是强制性的(mandatory)，但是在MIB-II中被弃用；从MIB-II开始每个网络协议组包含自己的地址转换表(e.g. 对于IP组，网络地址转换表就是ipNetToMediaTable)。

(a) 在at组中只包含一个表格数据结构变量：网络地址转换表：
<center><img src="/img/in-post/tcp-ip_img/snmp_24.pdf" width="90%"></center>

(b) at地址转换表实例(ARP高速缓存表)，表中绘制了字典序查询顺序：
<center><img src="/img/in-post/tcp-ip_img/snmp_25.pdf" width="50%"></center>
- kinetics路由器同时连接了一个TCP/IP网络、一个AppleTalk网络。
- 地址转换表接口索引(atIfIndex)为1时，对应的物理地址为48bit以太网地址；当地址转换表接口索引(atIfIndex)为2时，对应的物理地址为32bit地址(AppleTalk协议下的物理地址)。
- kinetics路由器的地址转换表中有一条表项(entry)和netb路由器相关，netb路由器的地址为140.252.1.183。这是因为kinetics和netb在同一局域网中，并且kinetics需要借助ARP(获取netb MAC地址)来将SNMP报文响应返回netb主机。

(c) 在client端sun主机上运行sumpi，字典序打印server端kinetics路由器中的at地址转换表实例：
<center><img src="/img/in-post/tcp-ip_img/snmp_26.pdf" width="100%"></center>

##### 4 ip组
ip组定义了许多简单参数以及3个表格数据结构参数。

(a) ip组中定义的所有简单参数如下表所示：
<center><img src="/img/in-post/tcp-ip_img/snmp_27.pdf" width="90%"></center>

(b) ip组第一个表格数据结构，ipAddrTable中的定义的参数类型 & 相应描述：
<center><img src="/img/in-post/tcp-ip_img/snmp_28.pdf" width="80%"></center>

(c) snmpi打印表格数据参数信息，在client端sun主机上使用snmpi管理进程查询ipAddrTable，并按照字典序打印参数信息：
<center><img src="/img/in-post/tcp-ip_img/snmp_29.pdf" width="100%"></center>

(d) ip组第二个表格数据结构，ipRouteTable中定义的参数类型 & 相应描述：
<center><img src="/img/in-post/tcp-ip_img/snmp_30.pdf" width="80%"></center>

(e) snmpi打印表格数据参数信息，在client端sun主机上使用snmpi管理进程查询ipRouteTable，使用"dump ipRouteTable"命令得到sun主机上的路由表信息，将其还原成表格格式如下图：
<center><img src="/img/in-post/tcp-ip_img/snmp_31.pdf" width="70%"></center>

(f) netstat命令显示sun主机的路由表信息，方便和c.2snmpi打印的信息进行对照，打印netstat命令输出如下(非字典序排列)：
<center><img src="/img/in-post/tcp-ip_img/snmp_32.pdf" width="100%"></center>

(h) ip组第三个表格数据结构，ipNetToMediaTable中定义的参数类型 & 相应描述：
<center><img src="/img/in-post/tcp-ip_img/snmp_33.pdf" width="80%"></center>

(i) 使用arp命令打印sun主机上的arp cache缓存信息，并使用snmpi dump命令字典序打印ipNetToMediaTable表格数据参数的信息(4个参数+2个实例)，如下图所示：
<center><img src="/img/in-post/tcp-ip_img/snmp_34.pdf" width="100%"></center>

##### 5 icmp组
<center><img src="/img/in-post/tcp-ip_img/snmp_35.pdf" width="100%"></center>
- icmp组包含4个"general variables"：total number of input/output ICMP messages & number of input/output ICMP messages with errors。这4个通用参数在图中以加粗方式标识。
- 除了4个通用参数，icmp组中还包含针对不同的icmp报文类型的22个计数器(counters)：11个input counters，11个output counters。

##### 6 tcp组
ip组定义了许多简单参数以及1个表格数据结构参数。

(a) tcp组中的简单参数如下表所示：
<center><img src="/img/in-post/tcp-ip_img/snmp_36.pdf" width="100%"></center>

(b) 可以通过snmpi管理进程查询(a)中的4个简单参数实例的取值：
<center><img src="/img/in-post/tcp-ip_img/snmp_37.pdf" width="100%"></center>

(c) tcp组定义的唯一的表格数据结构参数，tcpConnTable包含的参数 & 相关信息如下所示：
<center><img src="/img/in-post/tcp-ip_img/snmp_38.pdf" width="100%"></center>

(d) 通过在sun主机上运行snmpi管理进程，显示sun主机上的tcpConnTable表格数据结构参数实例如下所示：<br>
(d.1) 在"dump the tcpConnTable"之前，首先在sun主机建立两条TCP连接：
<center><img src="/img/in-post/tcp-ip_img/snmp_39.pdf" width="100%"></center>
(d.2) 在sun主机上运行snmpi管理进程以"dump the tcpConnTable"，并打印输出如下。需要注意，实际上有很多server的代理进程在监听和sun主机的TCP连接，但是为了方便&简化&便于理解，下面只显示部分"有用的"tcpConnTable"实例信息：
<center><img src="/img/in-post/tcp-ip_img/snmp_40.pdf" width="100%"></center>


### 九：其他的一些案例分析(additional examples)

##### 1 可以使用SNMP来获得接口MTU(interface MTU) - 
在chapter11中，曾经借助"设置DF禁止分片标志位"以及"icmp不可达报文机制"来获取netb到sun之间的SLIP链路接口MTU数值，而本节借助snmp程序直接查询接口表ifTable的形式查询接口MTU，操作方式如下所示：
<center><img src="/img/in-post/tcp-ip_img/snmp_41.pdf" width="100%"></center>

##### 2 路由表(routing tables)
本章节主要给出sun主机和一个多接口主机通信的案例，并借助SNMP机制来查询"route table entry"，分析通信选路过程以及每个路由器、主机选路策略。

(a) 本节案例涉及的主机网络拓扑结构如下：
<center><img src="/img/in-post/tcp-ip_img/snmp_42.pdf" width="100%"></center>

(b) gemini是一个multihomed host(多接口主机)，从sun主机上分别经由gemini的两个接口连接daytime服务进程。从telnet打印的信息易知：从gemini的两个接口和daytime进程进行通信除了目的接口IP地址不同之外，其他部分"好像"没有什么不同：
<center><img src="/img/in-post/tcp-ip_img/snmp_43.pdf" width="100%"></center>

(c) 采用traceroute命令查看sun主机和gemini主机的两个端口通信的选路情况。从traceroute进程打印的信息易知：从sun主机到gemini的两个接口的ip选路情况不同，一个需要经过3hops，一个需要经过2hops：
<center><img src="/img/in-post/tcp-ip_img/snmp_44.pdf" width="100%"></center>

(d) 采用宽松的源站选路选项的traceroute -g程序，对从sun主机到达geminin主机.3.54端口后，再返回sun主机的路由选路情况进行查看。如本小节(a)部分中的主机拓扑结构所示，从sun主机到达gemini主机和返回的路由采用了不同的路由选路(返回路由不经过swnrt主机；路由选路用dashed line标识)。宽松路由选项得到了结果如下所示：
<center><img src="/img/in-post/tcp-ip_img/snmp_45.pdf" width="100%"></center>

(e) 使用snmpi进程验证在路由信息中，从netb路由器到达140.252.3网络的路由器是swnrt，而不是gemini。即使用snmpi进程打印出哪个ip地址(端口if)对应了网络140.252.3：
<center><img src="/img/in-post/tcp-ip_img/snmp_46.pdf" width="100%"></center>

(f) 同样的，可以很容易通过gemini主机上的路由表信息来解释：为什么从gemini返回140.252.1网络时，能够直接返回给netb主机，而不是借助swnrt路由器。<br>
原因是：gemini主机要返回信息给140.252.1.29，而140.252.1网络通过if#2和gemini直接连接。<br>
另外，通过这个案例可以对于"multihomed host"和"router"在网络中的不同职责进行更深的理解：140.252.1子网上的netb主机到140.252.3子网需要**优先**通过路由器，而不是多接口主机gemini；多接口主机到达与其某个if直接相连的子网140.252.1上的主机netb不需要通过路由。

### 十：trap
本章之前的例子都是从管理进程(manager)到代理进程(agent)的，即都通过"get/get-next"执行查询参数操作。实际上，代理进程也可以向管理进程主动发送trap，以通知管理进程在server端代理进程中有某些manager需要知道的情况发生。trap从agent进程发送到manager进程的162公知端口(well-known port)。

##### 1 trap types(trap报文的种类)
<center><img src="/img/in-post/tcp-ip_img/snmp_47.pdf" width="100%"></center>
- 上表中介绍了6种trap类型，第7种trap类型是供应商自定义特定类型(enterprise-specific defined by vendor)。

##### 2 使用tcpdump程序打印代理进程和管理进程之间trap通信的情况
在sun主机上运行SNMP代理进程，然后使其产生coldStart类型的trap并发送给bsdi主机。不需要在bsdi主机上运行处理trap管理进程，cause no acknowledgment need to be returned by the manager。
<center><img src="/img/in-post/tcp-ip_img/snmp_48.pdf" width="100%"></center>
- 图中两份SNMP数据报从SNMP代理进程(snmp:161端口)发送到SNMP管理进程(snmp-trap:162端口)上的。
- C=traps是trap报文的community名称，属于ISODE SNMP代理进程的配置选项。
- Trap(28)和Trap(29)表示trap报文的PDU类型(长度)。
- E:unix.1.2.5表示"enterprise:agent's sysObjectID"。根据本文前面的[SNMP通信报文所使用的对象标识符树形结构图](http://localhost:4000/tcp-ip/2021/10/29/post-tcp-ip-tcp-chapter-25/#snmp通信报文所使用的对象标识符树形结构)，它属于"iso.org.dod.internet.enterprises(1.3.6.1.4.1)"下的某个节点，其代理进程的对象标识为1.3.6.1.4.1.4.1.2.5，简称为1.3.6.1.4.1.unix.agents.fourBSD-isode.5；其中5表示ISODE代理进程软件版本号，"unix.agents.fourBSD-isode"标识了产生trap的代理进程软件信息。
- [140.252.13.33]表示代理进程的IP地址。
- 两份SNMP报文的trap类型分别是"coldStart(0)"和"authenticationFailure(4)"，由于它们不是厂商特定(enterprise-specific)的trap类型，因此"specific code"字段为0，且无需打印。
- 两份SNMP报文的时间戳(timestamp)数值分别为20、1907，表示从代理进程初始化后经过多少个0.01s后发送trap报文。因此第一个coldStart trap报文在agent initialized后200ms产生，第二个authenticationFailure在agent initialized后19.07-18.86=200ms后发出。


### 十一：SNMP中的ASN.1语法和BER编码规范
正式的SNMP规范都使用ASN.1(Abstract Syntax Rule)语法，并且在SNMP报文中采用BER(Basic Encoding Rule)编码规范。<br>
ASN.1是一个描述数据及数据属性的的正式语言(formal language)，它和数据存储以及数据编码无关。<br>
fields in MIB以及SNMP messages都是使用ASN.1语法格式进行描述的。

通过使用ASN.1语法，可以使用多种编码方式将数据编码成方便传输的比特流(stream of bits)，SNMP报文采用BER编码规范进行编码。<br>
ASN.1和BER和理解SNMP网络管理的概念和基本流程没有很大的关系，在SNMP实现时才需要重视。


### 十二：SNMP-v2
RFC-1441系统的介绍了SNMP-v2网络管理协议。本小节主要介绍SNMP和SNMP-v2之间的区别。
- **(1)** SNMP-v2中定义了一个新的分组类型(PDU type)"get-bulk-request"，使得管理进程可以高效的从代理进程中读取数据。
- **(2)** SNMP-v2中定义了另一个新的分组类型(PDU type)"inform-request"，使得一个管理进程可以向另一个管理进程发送消息。
- **(3)** SNMP-v2中定义了两个MIB：SNMPv2 MIB(管理信息库)和SNMPv2-M2M MIB(管理进程到管理进程的管理信息库)
- **(4)** SNMP-v2相比SNMP的安全性有很大提高，SNMP-v1中manager到agent的community是以cleartext方式传递的，而SNMP-v2对community提供了加密&鉴别机制。

## Reference
> \<tcp-ip: illustrated vol1\> chapter25 <br>

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
