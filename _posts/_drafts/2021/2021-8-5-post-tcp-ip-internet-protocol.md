---
layout: post
title: "tcp-ip: internet protocol (IP互联网协议)"
subtitle: '[tcp-ip: illustrated vol1] - chapter3' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-08-05 19:38
lang: ch 
catalog: true 
categories: tcp-ip 
tags:
  - Time 2021
  - tcp-ip illustrated vol1 
---
## Introduction
IP协议是TCP/IP协议簇中最核心的协议。所有的TCP、UDP、ICMP、IGMP数据都以IP数据的形式进行传输。

**IP协议的不可靠性** <br>
IP协议不保证IP数据报能够成功到达目的地，它只提供最好的传输服务。例如，当传输过程发生错误(e.g.某个路由器暂时用完缓冲区)，IP协议将丢弃该数据报，并发送一个ICMP消息返回信源端。IP传输中的可靠性都必须由上层协议来提供(e.g.TCP)。

**IP协议的无链接性** <br>
IP协议不维护任何关于数据报的前驱、后继相关信息，每个IP数据包的处理是独立的。例如，同一信源向相同的信宿先后发送两份IP数据报A、B，每份数据报都将自由选择传输路线，因此B数据报可能在A数据报之前被送达。

### IP首部格式
<center><img src="/img/in-post/tcp-ip_img/internet_protocol_1.pdf" width="100%"></center>

### 不同应用程序的建议TOC数值表 
<center><img src="/img/in-post/tcp-ip_img/internet_protocol_2.pdf" width="100%"></center>

### IP路由选择
**基本概念** <br>
1) 如果目的主机和源主机直接相连(如点对点线路)，或都在一个共享网络上(以太网、令牌环网)，则IP数据直接送到目的主机上。2) 否则，主机将数据报发往一个默认路由器上，由路由器转发该数据报。

**IP路由对于数据报的转发** <br>
1) IP路由可以从TCP、UDP、ICMP、IGMP中接受数据报(本地产生的数据报)，也可以从一个网络接口中接受数据报并进行发送(转发网络数据报)。2) 当IP路由接收到一份数据报时，首先检查数据报的目的地址是否为IP广播地址or本机IP地址：是的话数据报被送到由8bit首部协议字段指定的协议模块进行处理；否则IP层被设置成路由器功能or将数据报丢弃。

**IP路由表中每一项包含的基本信息** <br>
1) 目的IP地址。可以是一个完整的主机地址，也可以是一个网络地址。<br>
2) 下一跳路由的IP地址(next-hop router ip)，或有直接连接的网络IP地址。<br>
3) 标志位。可指明目的IP地址是网络地址还是主机地址；可指明下一跳是一个真正的router还是一个直接相连的接口。<br>
4) 为数据报的传输指定一个网络接口。

**IP路由选择中查询路由表的基本步骤** <br>
1) 搜索路由表，寻找与目的主机IP地址匹配的条目(网络号和主机号都匹配)。找到则按标志位进行转发。<br>
2) 搜索路由表，寻找与目的主机网络号匹配的条目，找到则按照标志位进行转发(需要考虑子网掩码)。<br>
3) 搜索路由表，寻找默认的表目，找到则将数据报转发给表目中的next-hop router。<br>
4) 上述三个步骤没有成功，则该数据报不能被转发。如果数据报来自本机，则返回"主机不可达"or"网络不可达"的错误。<br>
5) 补充：为一个网络指定一个router而不是为每个电脑指定一个router是IP路由选择机制的重要特性。这样路由器只需维护几千个条目而不是数以百万计的条目。

**IP路由选择 - 本地网络转发(直接路由)** <br>
发往直接路由的数据报分组不但具有目的端(final destination)的IP地址，还具有其链路层地址。
<center><img src="/img/in-post/tcp-ip_img/internet_protocol_3.pdf" width="80%"></center>

**IP路由选择 - 远程网络转发(间接路由)** <br>
当数据报分组被发往简介路由时，IP地址指明的是最终目的地址，但是链路层地址指明的是网关(next hop router)，下一跳硬件地址可以通过ARP请求&应答机制获取。
<center><img src="/img/in-post/tcp-ip_img/internet_protocol_4.pdf" width="100%"></center>

### IP地址的分类
<center><img src="/img/in-post/tcp-ip_img/internet_protocol_5.pdf" width="80%"></center>

### 子网掩码
<center><img src="/img/in-post/tcp-ip_img/internet_protocol_6.pdf" width="80%"></center>

### 特殊的IP地址(undone)
<center><img src="/img/in-post/tcp-ip_img/internet_protocol_7.pdf" width="80%"></center>

### ifconfig命令简介
<center><img src="/img/in-post/tcp-ip_img/internet_protocol_8.pdf" width="70%"></center>

### ifconfig命令使用 
**启动关闭指定网卡** <br>
```c++
ifconfig eth0 up // 启动以太网卡eth0
ifconfig eth0 down // 关闭以太网卡eth0
```
**配置ipv4地址** <br>
```c++
ifconfig eth0 192.168.120.56 // 为网卡eth0配置IP地址: 192.168.120.56
ifconfig eth0 192.168.120.56 netmask 255.255.255.0 // 配置地址并设置子网掩码 
// 设置地址 设置掩码 设置广播地址(将送往指定地址的数据报当作广播来处理)
ifconfig eth0 192.168.120.56 netmask 255.255.255.0 broadcast 192.168.120.255
```
**配置ipv6地址** <br>
```c++
ifconfig eth0 add 33ffe:3240:800:1005::2/64 // 为网卡配置ipv6地址
ifconfig eth0 del 33ffe:3240:800:1005::2/64 // 为网卡配置ipv6地址
ifconfig eth0 hw ether 00:AA:BB:CC:DD:EE // 为网卡修改MAC地址
```
**启用关闭ARP协议** <br>
```c++
ifconfig eth0 arp // 开启网卡的eht0的ARP协议
ifconfig eth0 -arp // 关闭网卡的eht0的ARP协议
```
**设置最大传输单元** <br>
```c++
ifconfig eth0 mtu 1500 // 设置最大传输单元(1500byte)
```
**ifconfig配置命令备注** <br>
1 使用ifconfig命令配置的网卡信息，在网卡重启、机器重启后，配置不复存在。<br>
2 service network [start|stop|restart|status] 临时配置，重启network会恢复到原IP。<br>
3 通过修改下列配置文件才能实现永久修改IP: "/etc/sysconfig/network-scripts/ifcfg-eth[0-9]"

### netstat命令
<center><img src="/img/in-post/tcp-ip_img/internet_protocol_9.pdf" width="80%"></center>

### IP地址的未来
ditto, omitted.

## Reference
> \<tcp-ip: illustrated vol1\> chapter3

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
> 10 `<font style="color:red; font-weight:bold">加粗蓝色</font>`用来设置字体颜色
