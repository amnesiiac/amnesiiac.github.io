---
layout: post
title: "Linux Kernel Architecture"
subtitle: '[深入Linux内核架构] - chapter 1 简介和概述'
author: "twistfatezz"
header-style: text
# header-img: "img/post-bg-alitrip.jpg"
# header-mask: 0.1
mathjax: true
date: 2022-06-27 18:08
lang: ch 
catalog: true
categories: linux 
tags:
  - Time 2022
---

### 内核的作用
从纯技术层面上，内核是硬件与软件之间的中间层，其租用是将应用程序的请求传递给硬件，并充当底层驱动程序，对系统中的各种设备进行寻址。

➀ 从应用程序角度看：内核可以看成一台"增强"的抽象计算机。<br>
➁ 当若干程序在同一系统中并发运行时，可以将内核视为资源管理程序：内核负责将底层可用公共资源(cpu时间、磁盘空间、网络连接等)分配给各个系统进程，同时保证系统完整性。<br>
➂ 另一种视角是：将内核看作一个"提供一组面向系统命令"的库，借助库函数，程序员执行系统调用如同普通函数一样。<br>

➃ 
➄ 
➅ 
➆ 
➇ 

### 内核实现策略
在操作系统实现中，主要有两种范型：<br>
➀ 微内核：微内核只负责最基本的功能，所有其他功能都委托给一些独立进程，这些独立进程通过指定的通信接口同中央微内核通信；例如，独立进程可能负责实现各种文件系统、进行内存管理等。<br>
➁ 宏内核：内核全部代码，包含所有子系统(e.g. 内存管理、文件系统、设备驱动系统)都打包在一个文件中；内核中每个函数都可以其他所有部分。目前宏内核的性能仍然强于微内核，因此Linux采用宏内核范型实现；但在系统运行时，可以通过模块技术将代码嵌入内核、或从内核移除，从而实现宏内核动态添加、删除部分功能。


待整理。



## Reference
> 深入Linux内核架构 <br>

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用\`\`将关键字包含在内 <br>
> 8 `<center><img src="/img/in-post/economics_4/xxx.png" width="60%"></center>` <br>
> 9 使用html设置图片文字环绕方式: <br>
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
> 12 问题脚注: ### ???problem😫problem???  大标题标识符：▶︎ 小标题标识符：➤ <br>
> 13 实现文章排版tab：`&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;` <br>
> 14 项目序号 ⓪➀➁➂➃➄➅➆➇➈➉ ➊➋➌➍➎➏➐➑➒➓
