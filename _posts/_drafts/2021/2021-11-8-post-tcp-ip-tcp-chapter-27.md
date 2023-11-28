---
layout: post
title: "tcp-ip: FTP (文件传输协议)"
subtitle: '[tcp-ip: illustrated vol1] - chapter27' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: true 
date: 2021-11-08 10:56
lang: ch 
catalog: true 
categories: tcp-ip 
tags:
  - Time 2021
  - tcp-ip illustrated vol1 
---
### 一：引言
FTP(file transfer protocol)是一个较常用的应用程序，用于internet中文件传输的标准。

##### 1 文件传输和文件访问
在了解FTP之前，需要分清文件传输(file transfer)和文件访问(file access)的区别，前者是由FTP提供的，而后者是由类似NFS(网络文件系统)提供的。<br>
FTP提供的文件传输机制是：将一个完整的文件从一个系统复制到另一个系统中。另外使用FTP，就需要一个FTP server with account for it，或者一个FTP server that accept annonymous FTP。

##### 2 FTP兼容性(heterogeneity)
和Telnet类似，FTP最初也被设计用于两台不同的主机。两台主机可能运行在不同的操作系统下，使用不同的文件结构，可能使用了不同的字符集。Telnet的兼容异构性(heterogeneity)是通过"强制两端使用相同的标准：the NVT using 7-bit ASCII"来实现的。然而FTP使用了不同的方式以处理通信两端所有系统差异性：FTP仅支持传输特定类型的文件(ASCII、binary、etc.)以及特定类型的文件结构组织方式(byte stream、record-oriented)。

RFC-959是FTP的正式规范文件，该规范同时包含了近年来(tcp-ip:illustrated 1989)文件传输方法的演变。

### 二：FTP协议
和已经介绍的其他应用程序不同，FTP使用2个TCP连接来完成一个文件的传输。
- **control-connection** 控制连接以normal client-server模式建立。server端在FTP服务的公知端口(21)执行被动打开(passive open)，并等待client端执行主动打开(active open)以和端口21建立连接。client和server进行通信时，控制连接一直保持UP状态。控制连接用于从client发送到server命令、server的回复信息。另外，由于连接用于传输命令，为了快速响应键入的命令，所以该服务的IP type-of-service应当采用"最小时延(minimum delay)"策略。
- **data-connection** 每当一个文件从client发送到server时，即建立一条数据连接(数据连接可以在其他情况下被建立，在稍后部分进行介绍)。另外，由于该数据连接专门用于传输，所以该服务的IP type-of-service应当采用"最大吞吐量(maximum throughput)"策略。

##### 1 FTP传输中的client和server连接情况
<center><img src="/img/in-post/tcp-ip_img/ftp_1.pdf" width="80%"></center>
- **(a)** 控制连接中的命令的解释(interpret)和应答(reply)由位于client和server两端的解释器来完成，FTP交互式程序的用户并不处理这些内容。
- **(b)** 图中"user interface"的功能是：提供用户所需的交互界面(菜单选择、逐行输入命令，etc.)，并将用户的输入转化成控制连接中传输的FTP命令，类似地从服务端返回的FTP命令也被转换成呈现给用户的交互式显示内容。
- **(c)** client端和server端的协议解释器负责根据具体需要启用文件传输功能(连接)。

##### 2 数据表示(data representation)
- **1 文件类型(File type)**
    - **(a) ASCII file type** <center><img src="/img/in-post/tcp-ip_img/ftp_2.pdf" width="100%"></center>
    - **(b) EBCDIC file type** An alternative way of transferring text files when both ends are EBCDIC systems(用于传输文本文件).
    - **(c) Image file type** The data is sent as a contiguous stream of bits(连续的比特流). Normally used to transfer binary files.
    - **(d) Local file type** A way of transferring binary files between hosts with different byte sizes(使用byte传输二进制文件，1byte所包含的bit数量不定). The number of bits per byte is specified by the sender. For systems using 8-bit bytes, a local file type with a byte size of 8 is equivalent to the image file type.

- **2 格式控制(format control) - 只对ASCII和EBCDIC文件类型有效**
    - **(a) Nonprint(非打印格式)(Default)** The file contains no vertical format information(文件中不包含针对打印设备的行与行之间垂直距离信息).
    - **(b) Telnet format control**. The file contains Telnet vertical format controls for a printer to interpret(包含垂直信息).
    - **(c) Fortran carriage control**. The first character of each line is the Fortran format control character.

- **3 结构(structure)**
    - **(a) File structure(文件结构)(Default)** The file is considered as a contiguous stream of bytes. There is no internal file structure.
    - **(b) Record structure(记录结构)** This structure is only used with text files (ASCII or EBCDIC).
    - **(c) Page structure(页面结构)** Each page is transmitted with a page number to let the receiver store the pages in a random order(每个发送的页面自带页码，以允许页面的随机发送). Provided by the TOPS-20 operating system. Host Requirements RFC不推荐实现这种结构。

- **4 传输模式(transmission mode) - 文件通过数据连接进行传输的方式**
    - **(a) Stream mode(流模式)(Default)** The file is transferred as a stream of bytes(文件通过一系列字节流传输). For a file structure, the end-of-file is indicated by the sender closing the data connection(发送端关闭数据连接以指明文件传输结束). For a record structure, a special 2-byte sequence indicates the end-of-record and end-of-file(对于记录结构组织的数据，使用2byte特殊序列指示记录结束).
    - **(b) Block mode(块模式)** The file is transferred as a series of blocks, each preceded by one or more header bytes.
    - **(c) Compressed mode(压缩模式)** A simple run-length encoding(变长度编码,RLE) compresses consecutive appearances of the same byte(将连续出现的相同的byte数据通过RLE的方式进行压缩). In a text file this would commonly compress strings of blanks, and in a binary file this would commonly compress strings of O bytes. (这种压缩方式通常不被使用，针对FTP文件传输有更好的替代压缩方式可用)

- **5 unix系统数据表示(传输&存储)方案** <br> 采用上述1～4进行数据表示，考虑全部可能的情况，则有4×3×3×3=36种不同方式进行传输or存储。实际中可以忽略掉其中多数选项：because they are either antiquated(陈旧过时的,obsolete) or not supported by most implementations. Common Unix implementations of the FTP client and server is as follows: <br>
    - **Type** ASCII or image.
    - **Format control** nonprint only.
    - **Structure** file structure only.
    - **Transmission mode** stream mode only.

- **6 关于上面unix系统的基本FTP实现的评价** <br>
This implementation meets the minimum requirements of the Host Requirements RFC(上述FTP实现满足了Host Requirements RFC的最基本要求). The Host Requirements RFC states "The FTP protocol includes many features, some of which are not commonly implemented. However, for every feature In FTP, there exists at least one implementation(FTP中的每个特性都需要至少实现1个选项)."

##### 3 FTP命令(commands)
The commands and replies sent across the control connection between the client and server are in NVT ASCII. This requires a CR, LF pair at the end of each line (i.e., each command or each reply).

The commands are 3 or 4 bytes of uppercase ASCII characters, some with optional arguments. More than 30 different FTP commands can be sent by the client to the server. 

<center><img src="/img/in-post/tcp-ip_img/ftp_3.pdf" width="80%"></center>

一般情况下，用户通过user interface输入的交互命令和control connection中传输的FTP控制命令通常是一一对应的，但是对于一些特定操作而言，一条用户交互命令对应多个FTP控制命令。

##### 4 FTP应答(replies)
The replies are 3-digit numbers in ASCII, with an optional message following the number. <br>
The intent is that the software needs to look only at the number to determine how to process the reply(软件识别FTP应答包含的数字代码以决定如何处理该应答), and the optional string is for human consumption(选项字符串供人查看，无需记忆所有数字代码的含义).

**[1] 关于FTP应答中的3bit数字代码**
Each of the three digits in the reply code has a different meaning. We'll see in Chapter 28 that the SMTP(Simple Mail Transfer Protocol), uses the same ❮conventions❯ for commands and replies.

**(i) [FTP-Reply-Code] First digit & Second digit convections**
<center><img src="/img/in-post/tcp-ip_img/ftp_4.pdf" width="100%"></center>

**(ii) [FTP-Reply-Code] The third digit** <br>
The third digit gives additional meaning to the error message(3rd digits主要用于赋予错误信息额外的含义). Here are some typical replies, along with a possible message string:
- **125** Data connection already open; transfer starting.
- **200** Command OK.
- **214** Help message (for human user).
- **331** Username OK, password required.
- **425** Can't open data connection.
- **452** Error writing file.
- **500** Syntax error (unrecognized command).
- **501** Syntax error (invalid arguments).
- **502** Unimplemented MODE type.

**[2] [FTP-Reply-Content]** <br>
通常情况下，每个FTP command产生单行FTP reply，例如："QUIT"命令将产生如下应答：
<center><img src="/img/in-post/tcp-ip_img/ftp_5.pdf" width="12%"></center>

当FTP command产生多行FTP reply时，例如："HELP"命令将会产生如下多行应答：
<center><img src="/img/in-post/tcp-ip_img/ftp_6.pdf" width="60%"></center>
- the first line contains a hyphen(连字符) instead of a space after the 3-digit reply code
- the final line contains the same 3-digit reply code, followed by a space. 

##### 5 FTP连接管理(connection management)
**[1] 数据连接(data connection)的3种用途**：
- **(1)** 将一个文件从client发送到server。
- **(2)** 将一个文件从server发送到client。
- **(3)** 将文件列表or目录列表从server发送到client。

文件列表在数据连接上返回而不通过控制连接的多行应答(multi-line replies)返回，this avoids any line limits that restrict the size of a directory listing and makes it easier for the client to save the output of a directory listing into a file, instead of printing the listing to the terminal.

**[2] 数据连接(data connection)的建立方式**(连接端口号的选择、哪一端执行主动打开、哪端执行被动打开)。以unix系统上的FTP实现为例(参考本文中"unix系统数据表示")，数据连接的建立方式如下：
- **(1)** client端发送FTP command请求建立FTP连接，因此FTP连接是由client端的控制下建立的。
- **(2)** client端通常选择一个临时端口号作为其数据连接端口。client端以此临时端口执行被动打开(passive open)。
- **(3)** client端使用"PORT"命令将(2)中的临时端口通过控制连接发送给server端。server端固定使用端口21来建立控制连接。
<center><img src="/img/in-post/tcp-ip_img/ftp_7.pdf" width="100%"></center>
- **(4)** server端通过控制连接接受client端发送的端口号，并执行主动打开(active open)。server端数据连接固定使用端口20。
<center><img src="/img/in-post/tcp-ip_img/ftp_8.pdf" width="100%"></center>

**[3] 关于建立数据连接的补充说明**：<br>
**(1)** server端总是负责执行数据连接的active open，通常server端也负责执行数据连接的active close；然而当client端向server端通过流模式(stream mode)传输文件时，需要client来执行active close：向server端发送一个文件结束(EOF)以中断数据连接。<br>
**(2)** 当需要建立连接时，client可以不通过控制连接发送"PORT"命令，而是直接由server端从公知端口21向已经建立了控制连接的client端临时端口(1173)发送active open(如下图所示)。<br>
虽然client端没有通知用于建立数据连接的临时端口，但这种方式是可行的，因为可以通过server端公知端口使用21还是20来区分数据连接&控制连接。Nevertheless，在本文后续部分将介绍为什么现有的FTP实现不采用这种方式。
<center><img src="/img/in-post/tcp-ip_img/ftp_9.pdf" width="100%"></center>


### 三：FTP的案例分析
这部分内容主要介绍一些FTP案例，主要介绍：FTP对于数据连接的管理方式、NVT ASCII码格式的文本文件如何发送、FTP通过Telnet的"同步信号"来中止正在进行的文件传输、匿名FTP。

##### 1 FTP连接管理：临时数据端口(ephemeral data port)
**(a)** 在svr4主机上和bsdi主机建立ftp连接(client-\>server)，通过建立的数据连接列出一个指定文件相关信息。
<center><img src="/img/in-post/tcp-ip_img/ftp_10.pdf" width="100%"></center>
- "\-\-\-\>"表示client端发送给server端的ftp command。
- 3 digit数字开头的为server端ftp进程返回的ftp replies。 
- 用户交互命令和ftp指令大多数情况都是一一对应的关系，但在少数情况下会出现一对多的关系：dir命令被分成2个ftp命令"PORT+LIST"。

**(b)** svr4和bsdi之间建立的控制连接上的数据分组交换的时序图(移除了控制连接的建立&中断，以及所有窗口大小通知)
<center><img src="/img/in-post/tcp-ip_img/ftp_11.pdf" width="100%"></center>

**(c)** svr4和bsdi之间建立的数据连接上的数据分组交换的时序图(移除了所有窗口大小通知，保留了服务类型字段以说明数据连接采用"最大吞吐量"策略)
<center><img src="/img/in-post/tcp-ip_img/ftp_12.pdf" width="100%"></center>
- 对于FTP数据传输推荐TOS字段为"0x8"，即最大吞吐量策略。
- 关于不同应用程序的推荐的服务类型字段(TOS)的数值表可以参考[chapter3 internet-protocol](http://localhost:4000/tcp-ip/2021/08/05/post-tcp-ip-internet-protocol/#不同应用程序的建议toc数值表)

##### 2 FTP连接管理：默认数据端口(default data port)
如果client端没有向server发送PORT命令指定用于建立数据连接的端口信息，那么server端就使用client端控制连接的临时端口号建立数据连接。这会给使用流模式(stream mode)的client端带来一些问题(unix ftp server & client总是使用的传输模式)。

> The Host Requirements RFC recommends that an FTP client using the stream mode send a PORT command to use a nondefault port number before each use of the data connection.

**❮#A: Question#1❯** <br>
现在如果我们要求在列出第一个文件相关信息后，紧接着再列出另一个文件的相关信息，会出现什么情况？unix在使用流模式传输文件时，1个文件对应1个数据连接，那么显然需要重用服务端的公知数据端口20，但是此端口处于2MSL状态，应该怎么设置使其能够马上建立新的tcp连接以传输新的数据呢？<br>
**❮#A: Recall❯** <br>
回想在chapter18 TCP连接的建立和中止文章中[2MSL等待状态](http://localhost:4000/tcp-ip/2021/09/25/post-tcp-ip-tcp-chapter-18/#3-2msl等待状态)介绍的使用"SOCK -A"设置SO_REUSEADDR选项可以允许使用server端处于2MSL状态的端口建立新的连接，只需新的连接序号\>之前连接的序号即可。<br>
**❮#A: Solution❯** <br>
server端设置SO_REUSEADDR选项，以允许处于2MSL状态的公知数据连接端口20建立新的连接；由于1173临时端口仍然处于TIME_WAIT状态，所以client端通过新的临时端口号(e.g. 1174)和20端口建立新的ftp-data连接。

**❮#B: Question#2❯** <br>
当client端不发送PORT指令，即client端不通知对端用于数据连接的临时端口号(ephemeral port number)，server端执行主动打开时，默认在公知20端口和对端控制连接临时端口之间建立数据连接(如"关于建立数据连接的补充说明"示意图所示)。当ftp采用上述方式建立数据连接时，需要重新分析数据分组传递时序图。<br>
**❮#B: Configuration❯** <br>
unix FTP client通过"sendport"指令来取消每次建立数据连接前发送PORT指令到server端的操作。<br>
**❮#B: Timeline❯** <br>
&emsp;&emsp;(a) client端1176端口到server端21端口建立控制连接。<br>
&emsp;&emsp;(b) client端在为1176端口的数据连接执行passive open时，由于该端口已经在控制连接中使用，所以必须设置SO_REUSEADDR选项。<br>
&emsp;&emsp;(c) server端为从自身20端口到client端1176端口的数据连接执行active open。即使client端1176端口已经被使用，client仍然会接受该连接(segemnt#2)，这是因为控制连接socket\<svr4, 1176, bsdi, 21\>和数据连接socket\<svr4, 1176, bsdi, 20\>属于不同的连接。TCP demultiplexes(多路复用)中4元组只要有一个不同即属于不同的连接。<br>
&emsp;&emsp;(d) server端对数据连接执行active close(segment#5)，将socket\<svr4, 1176, bsdi, 20\>置为2MSL等待状态。<br>
&emsp;&emsp;(e) client端在控制连接上发送LIST命令。此前由于1176在端口为上一个数据连接执行了passive open，因此client端需要为当前新的数据连接设置SO_REUSEADDR选项，因为1176端口正被使用。<br>
&emsp;&emsp;(f) server端从端口20到client端1176的新的数据连接执行active open。server端必须设置SO_REUSEADDR选项，这是因为此时server端20端口处于2MSL等待状态。当前插口为socket\<svr4, 1176, bsdi, 20\>，很容易发现：当前插口和(d)中的socket是相同的，因此这个数据连接将不会建立成功。BSD server将会每隔5s重试建立数据连接请求(共18次≈90s)。在下面的时序图中，segment#9在初次发送连接请求1min后，成功建立新的数据连接(svr4主机的MSL=30s，2MSL=1min)。在时序图中看不到重试连接请求所发送的SYN报文，因为server端active open不能正常完成，不能发送SYN报文。<br>
**❮#B: Timeline Analysis❯** <br>
<center><img src="/img/in-post/tcp-ip_img/ftp_13.pdf" width="100%"></center>

**❮Conclusion❯** <br>
最后分析Host Requirements RFC建议使用PORT指令确定ip、数据连接端口后再建立数据连接的原因。当需要建立两个相继使用的数据连接时，使用PORT指令能够避免由于利用了相同的socket而需要至少等待2MSL的时间后才能成功建立数据连接传输数据。

##### 3 文本文件传输：使用NVT ASCII方式传输or使用Image方式传输
**[1] ftp在传输文本文件时，默认采用NVT ASCII方式进行传输** <br>
为了验证这一点，在sun主机上使用ftp进程连接到bsdi主机。本案例没有使用\<-d\>选项，因此client端ftp进程不打印client端发送到server端的command。
<center><img src="/img/in-post/tcp-ip_img/ftp_14.pdf" width="100%"></center>
解释为什么数据连接上传输的数据为42bytes而不是38bytes？<br>
获取的文本文件总共有4行，每一行在传输的过程中都将1byte换行符"\n"替换成NVT ASCII方式的2byte end-of-line序列"\r\n"。当文本文件传输到client端后，在恢复成原本格式进行存储。因此38bytes的文本文件传输时占用的数据流量为42bytes。

**[2] 较新的ftp在传输文本文件时，可采用2进制(image)方式进行传输** <br>
较新的ftp实现在传输文本文件时，会检查两端是否为相同的系统，如果系统相同，则采用2进制(image file type)传输而不是默认NVT ASCII。这种机制主要有两种优势：
- The sender and receiver don't have to look at every byte (a big savings). 
- Fewer bytes are transferred if the host operating system uses fewer bytes for the end-of-line than the 2-byte NVT ASCII sequence (a smaller savings).

可以通过一个BSD/386系统的client和server端进行ftp通信案例进行分析：
<center><img src="/img/in-post/tcp-ip_img/ftp_15.pdf" width="100%"></center>

Host Requirements RFC指出FTP server必须支持"SYST"指令。tcp-ip: illustrated vol1中，只有BSD/386和AIX 3.2.2支持这个指令，SunOS 4.1.3以及Solaris 2.x对于"SYST"指令将会返回"500 command not understood"作为应答，SVR4将会返回"500 command not understood"并关闭控制连接。

##### 4 中止一个文件的传输：使用Telnet Synch Signal
这一部分主要分析client端怎样中止一个发送自server的文件的传输。<br>
想要中止从client端向server端发送的文件传输很容易：client中止在数据连接上发送数据，并在控制连接上发送"ABOR"指令到server端即可。但是在client端中止接收文件要复杂很多，需要向server端通知立即停止发送数据，需要使用Telnet synch signal来完成。

**[1]** 在bsdi主机上使用ftp进程连接bsdi，并获取指定文件，并在传输中键入中断。打印交互程序输出如下：
<center><img src="/img/in-post/tcp-ip_img/ftp_16.pdf" width="100%"></center>
- 在键入中断按键"^?"后，client端立即执行异常中断，并进入等待对端响应的状态。
- 在收到"urgent data with ABOR"命令后，server端将返回两个应答426、226，分别表示server端数据传输[预]中断，数据连接[预]关闭、[正式]完成中断文件传输(reset)。

**[2]** 上述案例从建立数据连接，到完成第一个数据分组传输部分流程的时序图：
<center><img src="/img/in-post/tcp-ip_img/ftp_17.pdf" width="100%"></center>

**[3]** 上述案例从client端键入中断，到关闭数据连接，再到继续传输执行中断前网络设备驱动中排队的数据的过程时序图：
<center><img src="/img/in-post/tcp-ip_img/ftp_18.pdf" width="100%"></center>
上述时序图中，非常值得关注的问题是：尽管server已经返回[永久]确认数据传输中断成功信息，但是后续仍然有部分数据(segment#25、segment#27)仍然通过数据连接在传输。
> **Explanation** These segments were probably queued in the ❮network device driver❯ on the server when the abort was received, but the client prints "1536 bytes received" meaning it(the client) ignores all the segments of data that it receives (segments 17 and later) after sending the abort (segments 14 and 15).

**[4]** Telnet不会将中断进程指令作为紧急数据urg发送，ftp则将其作为紧急数据发送。为什么两种应用采用了不同的方式呢？首先需要明确，紧急指针、紧急数据的机制需要在ftp控制连接上进行传输，而普通数据将在ftp数据连接中传输。
> **Explanation** The answer is that FTP uses two connections, whereas Telnet uses one, and on some operating systems it may be hard for a process to monitor two connections for input at the same time(单一进程可能难以同时监测两个连接(数据连接&控制连接)的输入). <br>
> FTP assumes that these marginal operating systems at least provide notification that urgent data has arrived on the control connection(ftp假定了操作系统在紧急数据通过控制连接到达时给予通知), allowing the server to then switch from handling the data connection to the control connection(从而允许server紧急数据接收端进程能够将监测的"焦点"从数据连接转换到控制连接上).

##### 5 匿名FTP(annonymous ftp)
匿名ftp是一种被普遍应用ftp形式，当匿名ftp进程在server端运行时，将允许任何client使用匿名ftp传输文件。Vast amounts of free information are available using this technique.

**❮Case analysis❯** <br>
使用匿名ftp连接到ftp.uu.net站点(一个常用的提供匿名ftp服务的网站)，并获取tcp-ip:illustrated vol1的勘误表(errata)。
要使用匿名ftp，需要使用"anonymous"作为用户名在server上进行注册，可以自由设置登陆密码(本案例使用"rstevens@noao.edu")。
<center><img src="/img/in-post/tcp-ip_img/ftp_19.pdf" width="100%"></center>

##### 6 来自一个未知地址的匿名FTP(anonymous ftp from an unknown ip)
之前在chapter14 DNS曾经介绍过pointer queries(指针查询机制)：接收ip地址并返回hostname主机名。Unfortunately，并不是所有的系统管理员都能为pointer queries服务正确地设置name servers(名字服务器)：他们经常将新主机相关信息添加到name-to-address mapping文件中，但是忘记将相关信息添加到address-to-name mapping文件中。e.g. 当我们使用traceroute程序时，打印ip地址名而不是hostname时，表明了traceroute server主机存在上述缺省的配置。

一些匿名ftp server要求client端具有有效的域名，以允许匿名ftp server为文件传输对端记录日志。在进行匿名ftp文件传输时，server端唯一能够获取的关于client的信息是其ip地址，server端通过DNS服务执行指针查询以获取client端的domain name。如果负责client端的name server(名字服务器)没有被管理员正确的设置，则server端pointer queries将失败。

下面通过如下步骤来"模拟"出现这种错误的场景：
- **(1)** 将slip主机的ip地址更换成140.252.13.67。这个ip地址是slip主机所在子网的有效的ip地址，但是该ip地址并没有被登记到noao.edu子网的域名服务器上。
- **(2)** 将bsdi主机相连的SLIP连接的目的ip地址修正为140.252.13.67。
- **(3)** 在sun主机上增加一个路由表项，用于将来自140.252.13.67转发给bsdi主机。

经过上述配置后，从internet可以访问到slip主机：连接外网的路由器将数据分组转发给netb，netb作为140.252.13代理路由器将该数据分组转发给sun主机，sun主机将根据(3)中添加的路由表项来转发该数据分组到slip主机。<br>
上述1~3步骤设置的slip主机具有"complete internet connectivity"，但是不具有"valid domain name"，将会导致DNS pointer queries失败。

**❮Validation case❯**
<center><img src="/img/in-post/tcp-ip_img/ftp_20.pdf" width="100%"></center>


## Reference
> \<tcp-ip: illustrated vol1\> chapter27 <br>

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
