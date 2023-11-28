---
layout: post
title: "assembly language (CPU内部中断)"
subtitle: '[assembly language wangshuang 3ed] - chapter12' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics\_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2022-03-09 15:17
lang: ch 
catalog: true 
categories: assembly
tags:
  - Time 2022
---
### 本文概述 
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
➀ 用来存储相关指令的某些计算结果。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
➁ 用来为CPU执行相关指令提供依据。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
➂ 用来控制CPU相关工作方式。<br>
这些特殊的寄存器在8086CPU中，被称为标志寄存器(flag)。8086CPU的标志寄存器(flag)有16bit，其中存储的信息被称为程序状态字(PSW)。

▶︎ __8086CPU的flag寄存器结构__ <br>
<center><img src="/img/in-post/assembly_language_img/chapter_11_1.pdf" width="100%"></center>


## Reference
> \<assembly language by wangshuang 3ed\> chapter12 <br>

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
