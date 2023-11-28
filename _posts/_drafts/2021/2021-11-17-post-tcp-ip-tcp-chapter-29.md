---
layout: post
title: "tcp-ip: NFS (网络文件系统)"
subtitle: '[tcp-ip: illustrated vol1] - chapter29' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: true 
date: 2021-11-17 20:08
lang: ch 
catalog: true 
categories: tcp-ip 
tags:
  - Time 2021
  - tcp-ip illustrated vol1 
---
### 一：引言
网络文件系统(netword file system, NFS)，一种较为通用的提供透明文件访问(transparent file access)的应用程序。Sun RPC远程过程调用(remote procedure call)是NFS的基本构成模块。

使用网络文件系统(NFS)的client端不需要做什么特别的工作，系统内核(kernel)将负责监测正在访问的文件是否位于NFS server上，并自动生成远程过程调用请求(RPC calls)以访问指定文件。

本文主要介绍网络文件系统(NFS)如何利用UDP完成相应工作，对于NFS如何使用internet协议不做介绍。

### 二：Sun RPC(remote procedure call)
一般而言，大多数网络程序的设计和编写都是调用系统提供的函数接口来完成特定网络操作。例如，一个函数负责TCP active open，一个负责passive open，一个负责在TCP连接上发送数据，一个用于设置特定协议选项(e.g. 打开TCP keepalive定时器)...有两种常用的用于网络编程的API：socket API和TLI API。

**❮决定client & server两端能否进行通信的判据❯** <br>
另外，client端使用的API和server端使用的API可以不同，在client端和server端使用的操作系统也可以不同。网络通信应用程序设计需要应对复杂的应用场景，决定一对client和server能够彼此进行通信的核心判定依据是：通信协议(communication protocol)、应用程序协议(application protocol)。<br>
e.g. 一个使用c语言编写的、使用套接字socket和TCP的unix client可以与一个使用COBOL编写的、使用其他API和TCP的大型机(mainframe)server进行通信，只要通信两端都通过网络进行连接，并且都有基于TCP/IP的实现。

**❮通用网络编程模式：client发送命令到server，server返回应答给client❯** <br>
目前为止，已经介绍的所有应用程序：ping、traceroute、选路守候程序(routing daemons)，以及DNS(域名服务程序)、TFTP(简单文件传输协议)、BOOTP(引导程序协议)、SNMP(简单网络管理协议)、Telnet、FTP(文件传输协议)、SMTP(简单邮件传输协议)的client端以及server端都是采用这种方式实现的。

**❮Sun remote procedure call(RPC)的网络编程模式：a difference way❯** <br>
Sun远程过程调用RPC的程序设计具体流程如下：
- ➀ 当client端调用remote procedure时，实际上它只是调用了一个本地主机(local host)上的由RPC package生成的函数。这个被client调用的函数被称为"client stub"。client stub负责将procedure参数打包进一个网络信息报(network message)中，并将该信息报发送到server端。
- ➁ server stub在serer端负责接收网络信息报(network message)。server stub从网络信息报中提取参数，然后调用应用程序作者编写的server function以执行server procedure。
- ➂ server function运行得到的返回值将返回给server stub；server stub将函数返回值打包成网络信息报(network message)，然后将信息报返回给client stub。
- ➃ client stub从收到的网络信息报中得到server function的返回值后，将其转发给client端应用进程。

**❮网络程序设计的框架结构各个组件之间的垂直调用、封装关系❯** <br>
基于Sun RPC的网络程序设计在利用socket API和TLI API的基础上完成，其中socket API和TLI封装了底层的stub functions以及RPC library routines。client program以及server procedures并不会调用socket API、TLI API，client program调用server procedures，而server procedures通过client stub、server stub、RPC package实现。

**❮使用RPC packages作为网络编程基础的好处❯** <br>
- ➀ 可以将程序设计工作从复杂的network programming中剥离出来，application programmer只需实现client端代码以及client端代码调用的server procedures。将业务逻辑实现和底层网络编程实现相互分离。
- ➁ 如果application program使用了不可靠协议(e.g. UDP)，超时(timeout)和重传(retransmission)等机制被RPC package来处理，降低了application program程序任务难度。
- ➂ RPC packages为传入的参数类型和返回的结果数据类型提供了任何需要的数据类型转化。例如，程序的传入参数包含整数、浮点数，RPC package能够处理在client和server之间数据类型存储方式的差异，简化了在client & server之间异构环境(heterogeneous)中数据的编码问题。

**❮Sun remote procedure call(RPC)❯** <br>
Sun RPC有两个版本，一个版本建立在socket API上，依赖于TCP和UDP；另一个版本，称为TI-RPC(transport independant RPC)，建立在TLI API上，和系统内核提供的任何传输层(transport layer)共同运作。本章中只讨论依赖于TCP&UDP的Sun RPC版本，从本文介绍的内容来看两种实现版本是一样的。

(a) Sun RPC call message以及(b) Sun RPC reply message的UDP数据报封装格式如下图所示。
<center><img src="/img/in-post/tcp-ip_img/nfs_1.pdf" width="100%"></center>
(a) Sun RPC call报文格式解析：
- IP首部&UDP首部都是标准格式，在chapter3和chapter11中已经显示过。
- transaction ID(XID)事务标识符由client进程设置，并由server端程序返回。当client端收到一个reply，它将reply的XID和之前发送的request相比较：如果不匹配，则client端放弃reply报文，等待从server返回的下一份reply。每次client端发送一个新的request时，都设置一个新的XID数值；如果client端重传之前发送的RPC报文(未能收到该RPC报文reply)，则重传报文的XID和原始报文的XID一致。
- call调用变量值在call报文中设置为0。
- program number、version number、procedure number三个字段指定调用(identify)server端特定的procedure。version版本号填写当前RPC的版本：2。
- credential证书字段用于标识client。在某些情况下，该字段设置为空；有些情况下将该字段设置为client端的numeric user ID和group IDs。server端可以查看证书字段以决定是否执行应答。
- verifier验证字段用于使用了DES encryption(加密)的Secure RPC。虽然credential字段和verifier字段属于长度可变字段，但它们的长度本身也作为字段的一部分被编码。
- procedure parameter过程参数字段中，参数的格式依赖于remote procedure的定义格式。该字段是一个变长字段，获取其长度大小的方式是：UDP数据报的总长度-除当前字段外的所有字段的长度和=过程参数字段长度；UDP数据报是❮面向报文❯的，在传输中有边界的概念，因此能够标识报文长度。当使用TCP进行传输时，需要注意TCP是❮面向字节流❯的协议，因此在传输过程中TCP本身不能对数据划分边界，因此一份TCP报文不能标识sender单次发送的报文长度；在TCP首部和XID字段之间添加一个4byte长度字段，以通知接收端RPC call报文由多少字节组成；该机制允许了RPC call在借助TCP进行传输时可以分成多个报文段进行(DNS和RPC call使用了类似的机制)。

(b) Sun RPC reply报文格式解析：
- 图中显示了RPC reply报文的格式，当远程过程返回时，server stub将这个报文发送给client stub。
- reply报文的XID字段是从call报文中的XID字段复制而来。
- call调用变量在reply报文中设置为1。
- 如果call报文被成功接受，则status字段被设置为0；当call报文的RPC version number不为2，或者server端不能鉴别client端的身份，call报文可能会被拒绝。
- Secure RPC使用verifier字段来标识server。
- 如果远程过程调用成功(RPC success)，在accept status字段设置为0；当call报文的包含无效的version number或是无效的procedure number时，accept status字段设置为非0值。
- 和RPC call message相同，如果传输中使用TCP而不是UDP，那么需要在TCP首部和XID字段之间添加4byte长度字段，用来标识sender单次发送的报文长度。


### 三：XDR(external data representation)
外部数据表示(XDR)是一种编码标准，用于对于RPC call报文以及RPC reply报文中所有字段的数值进行编码。这种针对RPC call & reply报文中字段编码的统一标准能够允许：运行在某个系统下的client端调用运行在不同的系统架构下的server端的procedure。

XDR定义了许多数据类型，以及这些数据类型在RPC报文中的传输方式(bit order、byte order)。sender需使用XDR格式构造RPC报文，receiver将收到的XDR格式的RPC报文转换成本机中的数据表示形式。


### 四：端口映射器(port mapper)
**❮为什么需要使用端口映射器❯** <br>
server端的涉及远程过程调用RPC的应用进程使用临时端口(ephemeral port)，而不是公知端口进行通信(well-known port)。为了便于对涉及RPC的server端应用进程进行管理，需要某种形式的注册程式来跟踪"哪个RPC进程占用了哪个临时端口"。在Sun RPC中，这个注册程式被称为端口映射器(port number)。

**❮关于端口映射器的补充说明❯** <br>
- ➀ 端口映射器本身必须占用一个公知端口：UDP-111、TCP-111。
- ➁ 端口映射器本身也是一个远程过程调用RPC server程序，其各个字段的数值为：program number(100000)，version number(2)。
- ➂ TI-RPC可以运行在任何传输层(transport layer)上，不仅仅是TCP、UDP，因此使用TI-RPC的系统中(e.g. SVR4、Solaris 2.2)，端口映射器被称为"rpcbind"。

**❮端口映射器提供的4种服务类型❯** <br>
server端应用进程通过RPC call向端口注册器程序来进行注册自身，client端应用进程通过RPC call向端口映射器程序进行查询。提供的4种服务类型如下所示：
- ➀ PMAPPROC_SET：被RPC server在启动时调用，RCP server启动时将注册**[**program number, version number, protocol, port number**]**。
- ➁ PMAPPROC_UNSET：被server端调用以删除一个已经注册的映射(应用程式\<---\>端口)。 
- ➂ PMAPPROC_GETPORT：被RPC client端启动时调用，根据给定的**[**program number, version number and protocol**]**来获取port number。
- ➃ PMAPPROC_DUMP：返回端口映射器数据库中所有的记录，每条记录包含**[**program number, version number, protocol, port number**]**。

**❮RPC server进程启动后，被RPC client进程调用的过程需要执行如下几个步骤❯** <br>
- ➀ 端口映射器必须第一个启动，启动通常发生在系统引导过程中。该过程创建一个TCP end point并在TCP-111端口执行passive open；同时也会创建一个UDP end point并在UDP-111端口等待UDP数据报的到达。
- ➁ **首先**，RPC server端进程启动时，会为每个RPC支持的program version分别创建TCP/UDP end point；PS：RPC server可同时支持多个program version，client端在执行RPC call时指明使用特定的version。**然后**，为TCP/UDP end points的两端分别分配临时端口号(ephemeral port)。**最后**，RPC server通过执行RPC call操作到端口映射器，以调用其PMAPPROC_SET服务来注册**[**program number, version number, protocol, port number**]**。
- ➂ RPC client端进程启动时，将调用端口映射器的PMAPPROC_GETPORT服务，来获取对于给定的**[**program number, version number, protocol**]** RPC server端进程设置的临时端口号(ephemeral port)。
- ➃ RPC client端发送一个RPC call报文到RPC server端的➂中返回的port。如果采用UDP传输方式，则client端发送一个包含RPC call报文的UDP数据报到RPC server:UDP-port(➂)。RPC server通过返回一个包含RPC reply报文的UDP数据报到RPC client。如果采用TCP传输方式，则RPC client执行active open到RPC server:TCP-port(➂)，然后在TCP连接上发送RPC call报文，RPC server通过TCP连接返回RPC-reply报文。

**❮使用rpcinfo命令能够打印端口映射器的映射表(需要调用端口映射器的PMAPPROC_DUMP服务)❯** <br>
<center><img src="/img/in-post/tcp-ip_img/nfs_2.pdf" width="100%"></center>
- 通过打印的sun主机的端口映射器的映射表可以看出：一些应用进程确实支持多个version number。
- 对于端口映射器而言，每个不同的**[**program number, version number, protocol**]**三元组，都在映射表中保存相应的唯一port number与之对应。
- version=1和version=2的NFS mount daemon使用相同的TCP/UDP port number(702/699)，但是不同版本的NFS lock manager使用了不同的port number。


### 五：NFS protocol (网络文件系统协议)
**❮网络文件系统NFS和FTP的区别❯** <br>
使用NFS，client进程可以透明的访问server端的文件以及文件系统。NFS不同与文件传输协议FTP，后者会在client端产生一个server端文件的完整副本，即client端进程实际操作的是FTP从server端复制&传输过来的文件副本；而NFS只会访问client端调用的server端进程所引用的服务器上的文件部分。

**❮如何理解NFS使得client对server上的文件系统的访问透明化❯** <br>
NFS使得文件的远程访问透明化，即client端上的任何应用进程能对本地文件执行的操作，在网络文件系统NFS上都能完成。

**❮关于NFS的实现方式的讨论❯** <br>
NFS是一个基于Sun RPC的client-server之间交互的应用程序，NFS client通过向NFS server发送RPC requests来实现对服务器文件的访问。

很容易想到：NFS中的这种文件访问方式可以借助普通用户进程来实现，即将NFS client实现为客户端上的一个user process，NFS server实现为一个运行在服务端的user process，两个user process通过explicit RFC call/reply进行通信。NFS通常不会按照这种方式进行实现的2个原因是：
- ➀ 对server端NFS文件的访问必须对client是透明的，因此NFS client calls必须是由client operating system代表client user process发出的(访问远端文件机制须和访问本地文件机制一致)。
- ➁ NFS servers借助server端的操作系统进行实现，这样可以提升效率。如果NFS server实现为一个user process，则每个client request、server reply(包含正在读、写的数据)都需要穿越kernel和user process的边界，因而代价很大(效率问题)。

**❮本小节内容概述❯** <br>
本小节内容主要考察version 2 of NFS(在RFC-1094中定义)。[X/Open 1991]给出了Sun RPC、XDR、NFS的一个更好的描述。[Stern 1991]给出了使用、管理NFS的细节。本文后面的内容将对version 3 of NFS进行简单描述。

**❮NFS client和NFS server的典型配置结构示意图❯** <br>
<center><img src="/img/in-post/tcp-ip_img/nfs_3.pdf" width="70%"></center>
- ➀ 对于client kernel来说，访问本地文件(将通过local file access)和访问NFS文件(将通过NFS client)是透明的：从呈现给user process的角度看两者一致(transparent)，内核内部执行方式有所不同。
- ➁ client kernel内部的NFS client发送RPC request到server kernel内部的NFS server中。NFS主要使用UDP进行通信，但是较新的实现方式也使用TCP。
- ➂ NFS server以UDP数据报的形式在2049端口接收client request。虽然NFS server可以使用临时端口(需要向端口映射器port mapper进行注册)，但是绝大多数实现方式都直接指定2049(hardcoded into implementations)。
- ➃ 当NFS server收到request后，该request被转发到server's local file access中。
- ➄ NFS server处理request需要花费一定时间，local file access处理request完成文件访问也需花费一定时间。与此同时，server kernel不会屏蔽其他client request，to handle this，大多数NFS server是多线程的，即在server kernel内部有很多NFS servers在运行。不同的操作系统对此的实现细节不同，由于大多数unix kernel并不是多线程的，一个常用的技巧是：对于单个用户进程启动多个实例(nfsd)，每个nfsd都执行一个系统调用，并作为一个kernel process保留在kernel中；即用多进程代替多线程，通过快速在进程之间进行切换来"模拟"多线程程序。
- ➅ NFS client处理user process的request需要花费一定时间。为了给client host上的user process提供更多的并发性(concurrency)，通常有多个NFS client运行在client kernel中。同样的，不同操作系统对此实现细节不同，同NFS server的实现技术相类似：每个user process实例(biod)，负责执行一个系统调用，并作为一个kernel process保留在kernel中。
- 补充：大多数unix主机可作为NFS client或者NFS server，或者同时作为NFS client & server。大多数PC(MS-DOS)只提供了NFS client的实现，大多数IBM 大型机(mainframe)只提供了NFS server的实现。

**❮远程过程调用程序的基本构成 - 除NFS协议还包含其他组成部分❯** <br>
<center><img src="/img/in-post/tcp-ip_img/nfs_4.pdf" width="60%"></center>
- 图表中的NFS应用程序是SunOS 4.1.3系统中提供的。其他较新的系统版本还支持更新的NFS版本。
- 挂载守护进程(mount daemon)需要先被NFS client host调用，然后client才能访问server上的文件系统。
- lock manager和status monitor允许client锁定一个NFS server上部分文件。这两个应用进程在实现上NFS协议相互独立，因为加锁(locking)时需要获取client和server两者的状态，但NFS在server端本身是stateless的。

##### 1 文件句柄(file handles)
**❮文件句柄的定义❯** <br>
文件句柄是NFS中的一个基本概念，它是一个非透明的对象(opaque object)用于引用服务器上的文件or目录。"非透明"是指：由服务端创建file handle，并将其传回client；当client端访问该文件时将使用该文件句柄。在这种文件访问过程中，client不会查看file handle中的内容，其内容只对server有意义。

**❮文件句柄的应用场景❯** <br>
- ➀ 每当client process打开一个实际上位于NFS server上的文件时，NFS client就会从NFS server上获取该文件的一个句柄。
- ➁ 每当NFS client收到user process的request对从NFS server端获取的文件执行读&写操作时，被操作文件的file handle被返回NFS server以标识出(identify)将被操作的文件。
- 补充：一般情况下，用户进程不会和file handles打交道，file handles只会在NFS client code和NFS serevr code之间传来传去。

**❮文件句柄大小&包含的信息❯** <br>
- ➀ 在version 2 of NFS中，file handle占用32bytes；在version 3 of NFS中，file handle占用64bytes。
- ➁ unix servers通常在file handles中存储如下信息：the filesystem identifier(major & minor device numbers)、the i-node number(a unique number within a filesystem)、the i-node generation number(each time an i-node is reused for a different file, it changes)。

##### 2 文件系统挂载协议(mount protocol)
在client能够访问server端的文件系统之前，client必须使用NFS mount protocol来挂载server的文件系统(filesystem)，这个过程需要使用网络文件系统挂载协议，且通常在client启动时完成。在该过程完成后，client将会得到一个引用了server端文件系统的file handle。

**❮unix client执行mount命令后，NFS mount过程示意图❯** <br>
<center><img src="/img/in-post/tcp-ip_img/nfs_5.pdf" width="100%"></center>
- ➀ server端的端口映射器(port mapper)通常在server系统引导(bootstrapped)时被启动。
- ➁ mountd(mount daemon)程序在port mapper后启动。mountd将创建TCP/UDP end point，并且为TCP/UDP分别分配ephemeral port，然后通过port mapper将创建的端口号注册到映射表中。
- ➂ client端mount command组件向server端的port mapper发出一个"RPC call"报文，用于获取server端mountd(mount daemon)程序占用的ephemeral port。和port mapper之间的通信可使用TCP/UDP，通常使用UDP来完成。
- ➃ server端port mapper返回"RPC reply"报文以应答server端mountd进程的端口号。
- ➄ client端mount command组件向server端的mountd(mount daemon)发出一个"RPC call"报文，用于在server端挂载(mount)一个文件系统。然后，server端可以验证client端：借助client端IP地址、端口号来检查server是否允许当前client通过file handle调用当前挂载的文件系统。和mountd程序的通信可以使用TCP/UDP，通常使用UDP来完成。
- ➅ server端mountd程序对于上述挂载的文件系统返回相应的file handle。
- ➆ client端mount command组件执行"mount system call"将➄中返回的file handle和client端本地挂载点(mount point)相关联起来。file handle被存储在NFS client code中，并且从现在开始user process中任何对server端file system中的文件的引用(ref.)都会以该file handle作为starting point。

**❮unix client sun主机执行mount命令将server bsdi主机文件目录挂载成本地文件目录❯**
<center><img src="/img/in-post/tcp-ip_img/nfs_6.pdf" width="60%"></center>

##### 3 NFS procedure(NFS过程)
本小节内容主要介绍NFS server提供的15个过程，(每个过程在本小节中的序号和NFS procedure numbers并不一致，因为本文按照不同procedure的功能将其进行了分组)。<br>
虽然NFS被设计成支持不同的操作系统之间的过程调用(不仅仅限于unix系统)，但是一些提供了unix fucntionality的procedures可能不会被其他操作系统所支持(e.g. hard links、symbolic links、group owner、execute permission, etc.)。
- ➀ ***NFSPROC_GETATTR***。返回文件属性：type of file(regular file, dir, etc.), permissions, size of file, owner of file, last-access time, and so on。
- ➁ ***NFSPROC_SETATTR***。设置文件属性(只有部分属性可以被设置)：permissions, owner, group owner, size, last-access time, last-modification time。
- ➂ ***NFSPROC_STATFS***。返回文件系统的状态：amount of available space, optimal size for transfer, and so on。unix系统中的"df"命令调用该过程。
- ➃ ***NFSPROC_LOOKUP***。搜寻一个指定文件。user process打开一个位于NFS server端的文件时，该过程将被client端调用。调用该过程将返回file handle以及文件属性。
- ➄ ***NFSPROC_READ***。从文件中读取数据。由client端指定读取数据的：file handle, starting byte offset, max number of bytes to read(≤8192).
- ➅ ***NFSPROC_WRITE***。向文件写入数据。由client端指定写入数据的：file handle, starting byte offset, number of bytes to write, the data to write。server端成功完成data write以及file info updated后，才能返回OK信号(即NFSPROC_WRITE过程是"synchronous"的)。
- ➆ ***NFSPROC_CREATE***。创建一个文件。
- ➇ ***NFSPROC_REMOVE***。删除一个文件。
- ➈ ***NFSPROC_RENAME***。重命名一个文件。
- ➀⓪ ***NFSPROC_LINK***。为一个文件构建"硬连接hard-link"。hard-link：将文件名称和文件目录之间的映射关系保存起来的directory entries。磁盘上的文件可以拥有任意数量的hard-link。
- ➀➀ ***NFSPROC_SYMLINK***。为一个文件创建"符号连接symbolic-link"。symbolic-link：一个文件包含有指向其他文件or目录的reference。大多数调用符号连接的操作(e.g. open)的实际操作对象是符号连接所指向的对象。
- ➀➁ ***NFSPROC_READLINK***。读取一个符号连接，即返回符号连接所指向的文件名。
- ➀➂ ***NFSPROC_MKDIR***。创建一个文件目录。
- ➀➃ ***NFSPROC_RMDIR***。删除一个文件目录。
- ➀➄ ***NFSPROC_READDIR***。读取一个文件目录。e.g. 被unix "ls"命令调用。

##### 4 UDP? or TCP?
NFS最初是用UDP来实现的，所有vendor都提供了这种实现方式。较新的NFS实现方式也支持TCP。为NFS增加对TCP支持的原因是：随着广域网速度越来越快，NFS不仅需要在LAN上运行，对于WAN应用场景也需要适配。

但是应用场景从LAN到WAN会带来一些问题：
- ➀ LAN和WAN下的[network dynamics](https://en.wikipedia.org/wiki/Network_dynamics)截然不同。
- ➁ WAN下的报文RTT时间在非常大的范围内波动，网络拥塞的情况将更加频繁的发生。

上述提到的网络特性的变化是考虑使用TCP的原因：TCP提供了slow-start算法以及拥塞避免算法，可以一定程度解决上述问题。但是UDP的实现中没有提供类似机制，因此需要考虑：在NFS client&server中加入类似优化算法，或者直接使用TCP进行通信。

##### 5 NFS over TCP(基于TCP实现的NFS)
伯克利实现的Net/2 NFS同时支持UDP/TCP，[Macklem 1991]对于该实现进行了描述。TCP和UDP的NFS实现有如下几点不同：
- ➀ 当server端系统引导时，启动一个NFS server进程，该进程在TCP端口2049执行passive open，等待client端连接请求。
- ➁ 当client端通过TCP挂载server端的文件系统，它对server端的TCP端口2049执行active open。因此在NFS client和NFS server之间为该文件系统建立起一个TCP连接。如果相同的NFS client上挂载了相同的NFS server端上的另一个文件系统，则应创建另一个TCP连接。
- ➂ NFS client端和NFS server端建立的TCP连接两端都需要设置TCP keepalive选项，以允许连接的任何一端能够检查对端是否出现crashes以及crashes and reboots的状况。
- ➃ client端所有使用了server端的指定的文件系统的应用进程共享同一个TCP连接。e.g. 下图中client端"/nfs/bsdi/usr/rstevens/"和"/nfs/bsdi/usr/smith"共享了相同的连接。<center><img src="/img/in-post/tcp-ip_img/nfs_7.pdf" width="100%"></center>
- ➄ 当client端检测到server端已经crashed或者crashed and rebooted(将会收到TCP连接错误："connection timed out"、"connection reset by peer")，client端将尝试和server端重新建立连接：它将执行active open来为同一server端的文件系统重新建立TCP连接。之前建立的连接上的所有超时的request报文在新连接上都会重新发送。
- ➅ 当client端crashes，client端崩溃时正在运行的应用进程也会随之crashes。当client重启时，将会通过TCP重新挂载崩溃前引用的server端文件系统，会建立一条新的TCP连接。client崩溃前建立的TCP连接处于"half-open"的状态，由于server端设置了keepalive选项，因此当server端下一个TCP keepalive probe发送时，server端的"half-open"连接将会被关闭。


### 六：NFS实例分析 
通过tcpdump程序来查看通过网络文件系统执行典型文件操作时，client端调用了哪些NFS过程。<br>
当tcpdump检测到一份包含RPC call、目的端口为UDP2049端口的UDP数据报时(call=0)，它将该数据报按照NFS request进行解码。类似地，如果tcpdump检测到一份包含RPC reply的UDP数据报(reply=1)，将该数据报按照NFS reply进行解码。

##### 1 简单案例：读取一个文件
**[1]** 使用cat(short for concatenate)命令将位于NFS server上的文件hello.c复制显示到client的终端上：
<center><img src="/img/in-post/tcp-ip_img/nfs_8.pdf" width="100%"></center>

**[2]** 使用tcpdump对于上述cat命令执行过程进行分析并打印NFS数据分组传输情况(tcpdump解析NFS request/reply时将打印XID数值而不是端口号)：
<center><img src="/img/in-post/tcp-ip_img/nfs_9.pdf" width="100%"></center>
tcpdump输出的NFS request/reply报文中的数据大小均为去掉TCP/UDP首部之后的"数据部分"的数值。

**[3]** 分析总结：案例中，NFS client(sun主机)对于sun主机和bsdi主机内核之间传输的这些RPC call/reply一点也不知道：应用进程只是调用了内核的open函数(该函数引起了3个RPC call和一个RPC reply)；随后应用进程调用了内核的read函数(该函数引起了2个RPC call和一个RPC reply)。整个远程过程调用的过程对于客户应用进程而言是透明的(如何操作本地文件一样)。

##### 2 简单案例：创建一个文件目录
**[1]** 使用cd命令更改当前文件目录，并使用mkdir命令在新的当前文件目录下创建一个名为"Mail"的目录：
<center><img src="/img/in-post/tcp-ip_img/nfs_10.pdf" width="100%"></center>

**[2]** 使用tcpdump对于上述命令执行过程进行分析并打印NFS数据分组传输情况：
<center><img src="/img/in-post/tcp-ip_img/nfs_11.pdf" width="100%"></center>
改变文件目录的cd命令需要调用2次GETATTR过程；创建新的文件目录的mkdir命令需要调用1次GETATTR和1次LOOKUP过程。tcpdump程序不能理解NFS过程返回的数据含义，它只能打印OK以及返回的报文中数据部分的字节数。

##### 3 无状态(statelessness)
网络文件系统(NFS)有一个特性：NFS server是无状态的，即server端不会keep track of which clients are accessing which files。

本文前面提到的[NFS procedure](http://localhost:4000/tcp-ip/2021/11/17/post-tcp-ip-tcp-chapter-29/#3-nfs-procedurenfs过程)中没有OPEN和CLOSE过程。LOOKUP过程有点类似OPEN操作：将特定文件打开后，会继续对当前file handle的文件进行操作；然而LOOKUP操作和OPEN又不完全相同：在执行LOOKUP操作后，server不能确定client端是否会继续引用LOOKUP操作查询的文件(handle)。

将server端设置为无状态的原因是：为了简化server端crashes & reboots后的crash recovery步骤。

##### 4 案例分析：服务端崩溃(server crash)
本小节内容将会介绍一个案例：从一个crash & reboot的NFS server上读取一个文件。该案例演示了：NFS client端并不能获悉statelessness的NFS server已经崩溃并重启：除了由于server端崩溃重启导致的一个time pause之外，client端上的应用进程不受影响。

**❮服务器崩溃案例模拟方式❯** <br> 
在NFS client端sun主机上，对NFS server端svr4主机上的一个"long file"(/usr/share/lib/termcap)执行cat命令。为了模拟服务端主机崩溃重启的过程：在termcap文件传输过程中，将svr4主机的以太网电缆拔掉，将svr4主机关闭后重新启动，在连接以太网电缆。

**❮使用tcpdump程序打印模拟服务端崩溃重启案例的NFS数据分组交换信息❯** <br>
<center><img src="/img/in-post/tcp-ip_img/nfs_12.pdf" width="100%"></center>
- ➀ tcpdump输出的#130和#131行可以看到两个请求超时报文，分别对应#132和#133行重传报文，为什么会存在两个读请求报文(一个从offset=65536开始读取，一个从offset=73728开始读)？原因：client端kernel检测到client端应用进程正在执行"sequence read"，因此kernel尝试预先读取(prefetch)数据块(许多unix内核实现都支持read-ahead)。client端kernel运行了多个NSF block I/O守护进程(biod进程)，每个biod进程都尝试产生RPC请求。在本案例中，一个NFS boid 守护进程在offset=65536byte请求读取8192byte数据，另一个进程在offset=73728byte处执行"read-ahead"读取操作。
- ➁ 从tcpdump输出的#129行服务端崩溃，到#171行服务端svr4主机重启完成的5min时间内，sun主机上的NFS client端进程并不知道服务端已经崩溃又重新启动了。NFS server是statelessness的，因此NFS client并不能获取serevr端的状态。
- ➂ 关于案例的超时时间、重传时间：根据➀，整个数据传输过程共有2个biod守护进程(一个从offset=65536开始读取，一个从offset=73728读取)，它们分别拥有各自的超时间隔。第一个biod守护进程超时时间(从#130到#144)分别为：0.6832、0.868(1)、1.7391(2)、3.4799(3)、6.9598(4)、13.9204(5)、19.9987(6)、19.9992(7)；第二个boid守护进程超时时间类似。可以发现：每次超时间隔为0.87s倍数，每次超时后的下一个超时重传间隔翻倍，直到达到上限20s。
- ➃ 当NFS server崩溃没有重启，那么NFS client执行超时重传持续多长时间呢？结果取决于client端的2个选项。如果NFS client挂载file system时设置"hard"(选项#1)，client的超时重传将不会停止，此时设置选项#2以决定是否允许用户使用interrupt key中断超时重传；如果NFS client挂载file system时采用"soft"方式，client在固定的重传后主动挂断。

##### 5 等幂过程(idempotent procedure)
如果一个远程文件调用过程(RFC procedure)执行多次仍然能够返回相同的结果，那么称这个RFC过程为等幂过程。

**❮NFSPROC_READ过程是等幂的❯** <br>
正如在上一节❮案例分析：服务端崩溃❯中展示的那样，由于server端崩溃，client端向server端多次超时重发READ调用直到它得到一个回应，而无论client端执行了多少次超时重发，期望的结果是固定不变的，因为READ调用在读取时指定了固定的offset。如果一个NFS过程在调用时读取服务器文件的next N byte，那么对于无状态的server，每次调用将会得到不同结果(非等幂过程)，除非server被实现成有状态的(能够记录上一次读取状态)。NFSPROC_READ和NFSPROC_WRITE过程都要求client端指出starting offset point，即client负责维护状态，而不是server。

**❮并不是所有文件系统操作都可以实现成等幂过程❯** <br>
例如，client端发来REMOVE请求来删除server端的文件，server端将文件删除，并回应了OK reply；在server端应答返回过程由于某种原因丢失；client端未能收到应答因而发生超时重传；重传请求到达server端，但server端找不到指定文件，将会返回文件不存在错误应答；这个应答是错误的，因为server端已经将该文件删除。这种情况表明该REMOVE过程不是等幂的(对于无状态的server多次重复执行返回结果不同)。

**❮等幂的NFS过程 & 不等幂的NFS过程(均省略NFSPROC_前缀)❯** <br>
等幂过程：GETATTR、STATES、LOOKUP、READ、WRITE、READLINK、READDIR。另外SETATTR过程如果不用来截断文件(truncate a file)一般是等幂的。<br>
不等幂过程：CREATE、REMOVE、RENAME、LINK、SYMLINK、MKDIR、RMDIR。<br>

**❮server端处理非等幂过程在传输丢包、超时时可能出现的问题❯** <br>
在NFS使用UDP传输时，总会发生server端返回的应答报文丢失的情况，此时对于非等幂操作，client端并不能获悉statelessness server端的正确状态。为了解决这一个问题，大多数server端实现了一个保存最近的非等幂过程的应答的高速缓存。每当server收到一个请求，它首先查询该高速缓存，如果找到一个匹配项，那么就返回之前的应答而不是执行该过程调用。[Juszczak 1989]对于这种NFS idempotent reply cache的实现进行了描述。

**❮等幂服务器过程 - idempotent server procedure❯** <br>
等幂服务器过程的概念可以应用于任何基于UDP的应用程序，而不仅仅是NFS。e.g. DNS也提供了等幂的服务：一个DNS server可以任意多次的执行解析端的请求不会产生不良后果(不考虑网络资源浪费)。

### 七：version 3 of NFS
本小节内容主要介绍NFS version 2和version 3之间的区别：
- ➀ NFS-v2的file handle都是固定32byte的数组，在NFS-v3中file handle变成了一个≤64byte的可变长度数组。在外部数据表示XDR中，可变长度的数组被编码为4byte数组元素个数+实际数据元素byte。NFS-v3的这种实现方式能够减少file handle的平均编码长度，并且允许一些"non-unix"实现利用剩余空间维护额外信息。
- ➁ NFS-v2限制每次READ、WRITE RPC操作的字节数≤8192 bytes；这项限制在NFC-v3中被移除，意味着NFS-v3使用UDP数据报传输时，操作的字节数限制为IP数据包长度的限制：65535 bytes。该特性允许在传输速率更快的网络上传输更大的read/write数据包。
- ➂ NFS READ/WRITE过程涉及的文件大小、起始点offset从32bit拓展到64bit，从而使得读写操作的动态范围更大。
- ➃ NFS-v3中每种能够改写文件属性的过程都会返回改写后的文件属性，从而减低了client端发送GETATTR调用的频次。
- ➄ 不同于NFS-v2中同步的WRITE，在NFS-v3中WRITE过程可以是异步的(asynchronous)，能够有效提升WRITE过程的效率。
- ➅ 从NFS-v2到v3，删除了1个过程：NFSPROC_STATFS，增加了七个过程：ACCESS(检查文件访问权限)、MKNOD(创建一个unix特殊文件)、READDIRPLUS(返回指定目录下的所有文件名及其属性)、FSINFO(返回指定文件系统的static info)、FSSTAT(返回指定文件系统的dynamic info)、PATHCONF(返回指定文件的POSIX.1信息)、COMMIT(将之前的异步WRITE过程内容写入到stable storage中)。

**❮Stable storage❯** <br>
> Stable storage is a classification of computer data storage technology that guarantees atomicity for any given write operation and allows software to be written that is robust against some hardware and power failures. To be considered atomic, upon reading back a just written-to portion of the disk, the storage subsystem must return either the write data or the data that was on that portion of the disk before the write operations. <br>
> Most computer disk drives are not considered stable storage because they do not guarantee atomic write; an error could be returned upon subsequent read of the disk where it was just written to in lieu of either the new or prior data.

## Reference
> \<tcp-ip: illustrated vol1\> chapter29 <br>

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
