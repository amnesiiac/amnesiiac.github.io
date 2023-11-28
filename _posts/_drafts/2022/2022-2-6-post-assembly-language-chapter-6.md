---
layout: post
title: "assembly language (源代码的分段管理)"
subtitle: '[assembly language wangshuang 3ed] - chapter6' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2022-02-06 19:55
lang: ch 
catalog: true 
categories: assembly
tags:
  - Time 2022
---
### 本文概述 
汇编程序有时需要占用多个空间存放数据，即程序可能需要占用多个段来作为：程序段、存放程序所需数据段、作为特殊顺序访问空间的栈段。上一篇文章曾经提到：0:200到0:2ff这段256kb的空间是相对安全的，如果程序所需段空间超过256kb怎么办？<br>
在操作系统提供的环境中，合法通过操作系统取得的内存空间都是安全的，即在操作系统的允许下，汇编程序可以取得任意内存空间。

汇编程序获取所需内存空间的方法有两种：1) 在汇编程序在内存中加载时进行分配(需要在源程序中定义段的方式来进行获取)；2) 在汇编程序执行时向系统申请。<br>

在汇编程序的段空间安排中，建议将程序所需的数据、代码、栈空间放入不同段中。下面分三种情况进行对比介绍：1) 在代码段中混合使用数据段；2) 在代码段中混合使用栈空间段；3) 将代码、数据部分、栈空间分别在不同的段中组织。


### 一：在代码段中使用数据
▶︎ **[问题] 编程计算内存中存储的下面8个数据之和，将结果存在AX寄存器中** <br>
```txt
0123h、0456h、0789h、0abch、0defh、0fedh、0cbah、0987h
```
在[上一篇文章](http://localhost:4000/assembly/2022/02/03/post-assembly-language-chapter-5/#7-loop和bx的联合使用案例分析)的案例中，只是直接将内存中的数据取出，循环累加到DX寄存器中。本文中的案例则需要为这8个数据送入连续的内存空间中，再从内存中取出并执行累加。

▶︎ **[解决] 为了安全规范起见，不能自己随意决定哪段空间分配为数据段，因此我们在❮源程序中❯定义待处理数据，这些数据就会被编译、链接程序作为待执行程序的一部分自动加载到内存中(待处理数据因此获得了存储空间)** 

➤ 第一种解决方案：将8个word数据在程序中代码段起始位置予以定义：
```txt
assume cs:code
code segment                                            ;代码段起始
    ;  cs:0  cs:1  cs:2  cs:3  cs:4  cs:5  cs:6  cs:7   ;代码段起始地址ds:[偏移数值]
    dw 0123h,0456h,0789h,0abch,0defh,0fedh,0cbah,0987h  ;使用dw指令在源程序中定义8个待处理数据
    mov bx,0                                            ;初始化偏移变量
    mov ax,0                                            ;初始化累加器
    mov cx,8                                            ;循环8次
s:  add ax,cs:[bx]                                      ;执行累加
    add bx,2                                            ;bx=bx+2 更新偏移变量 +2byte(1word)
    loop s                                              ;循环
    mov ax,4c00h                                        ;
    int 21h                                             ;21号中断
code ends                                               ;代码段中止
end                                                     ;通知编译器程序结束
```
上述代码可能出现的问题：代码在执行时，由于在代码的起始位置定义了8个数据，实际源程序的执行应该从"mov bx,0"开始。为了让源程序在正确的地方开始执行，需要在debug模式下，手动设置程序执行相关段寄存器CS:IP的数值，指向"mov bx,0"代码。<br>
如此一来，为了正确执行上述代码，只能在debug模式下运行，而在系统中实际执行可能会出现问题(程序入口不是希望被首先执行的命令)。

➤ 第二种解决方案：直接在源程序中指定程序入口：
```txt
assume cs:code
code segment
    dw 0123h,0456h,0789h,0abch,0defh,0fedh,0cbah,0987h  ;使用dw指令在代码段起始处定义待处理数据
start:  mov bx,0                                        ;初始化偏移变量
        mov ax,0                                        ;初始化累加器
        mov cx,8                                        ;循环8次
    s:  add ax,cs:[bx]                                  ;执行累加
        add bx,2                                        ;bx=bx+2 更新偏移变量 +2byte(1word)
        loop s                                          ;循环
        mov ax,4c00h                                    ;
        int 21h                                         ;21号中断
code ends                                               ;code代码段结束
end start                                               ;通知编译器 程序结束 & 源程序入口标号位置
```
在源程序结尾处通过"end+标号"的方式可以显式通知编译器：源程序入口的具体位置。具体地：回忆[chapter-4](http://localhost:4000/assembly/2022/01/20/post-assembly-language-chapter-4/#二源程序举例分析)中介绍源程序结构可知，源程序经过编译器、链接器被翻译成相关描述信息、    2进制文件。其中，相关描述信息中包括：编译、链接器将源程序中伪指令进行处理以得到的信息。<br>
即，上面源程序中的"end start"指令在编译、链接过程中被处理成一个入口地址；形成的可执行文件在内存中加载后，加载者从可执行文件的描述信息读取入口地址，并设置CS:IP使得CPU按照指定的程序入口执行。

总结上面内容，整理得到"指定程序入口"的框架程序如下：
```txt
assume cs:code
code segment
    ...
    ...      ;数据部分
    ...
start: ...
       ...   ;代码部分
       ...
code ends
end start
```


### 二：在代码段中使用栈
▶︎ **[问题] 编程利用栈，将下面样例程序中定义的8个数据实现逆序存放** <br>
```txt
assume cs:code
codesg segment
    dw 0123h,0456h,0789h,0abch,0defh,0fedh,0cbah,0987h  ;待处理数据 编程实现其逆序存放
    ...
start:  ...
        ...
codesg ends
end start
```

▶︎ **[解决] 程序运行时将定义的8个word型数据存放在cs:0到cs:F内存单元中，再预留8个word大小的内存单元作为栈空间，使用push操作依次将8个数据入栈，然后再使用pop操作依次出栈到定义数据的8个内存单元中，从而实现数据逆序存放** <br>
```txt
assume cs:code
codesg segment
    ;  cs:0  cs:2  cs:4  cs:6  cs:8  cs:10 cs:12 cs:14  ;数据地址
    dw 0123h,0456h,0789h,0abch,0defh,0fedh,0cbah,0987h  ;待处理数据
    ;  cs:16 cs:18 cs:20 cs:22 cs:24 cs:26 cs:28 cs:30  ;栈空间地址
    dw 0,0,0,0,0,0,0,0                                  ;占用空间 在后面程序中作为栈使用
start:  mov ax,cs
        mov ss,ax                                       ;借助AX寄存器设置SS段寄存器
        mov sp,30h                                      ;设置ss:sp栈顶"指针"指向cs:30 (指向栈底)
        mov bx,0                                        ;初始化待处理数据地址偏移值
        mov cx,8                                        ;设置循环计数8
    s:  push cs:[bx]                                    ;将cs:[bx]待处理数据入栈 sp=sp-2
        add bx,2                                        ;更新待处理数据偏移值
        loop s
        mov bx,0                                        ;重置待处理数据地址偏移值
        mov cx,8                                        ;充值循环计数8
   s0:  pop cs:[bx]                                     ;将待处理数据出栈到cs:[bx]中 sp=sp+2
        add bx,2
        loop s0
        mov ax,4c00h                                    ;
        int 21h                                         ;21号中断
codesg ends                                             ;codesg代码段结束
end start                                               ;通知编译器程序结束 & 指明程序入口start
```


### 三：将数据、代码、栈放入不同的段
前面两个小节的代码中，都将程序需要使用的数据、栈结构占位、以及源代码本身放到一个段内部("codesg")。使用这种源码组织方式，则在后续编程时，需要特别注意"define word(dw)"占用、定义的数据、栈空间等数据结构的边界。另外，上面的源码组织方式有2个主要问题：
- ➀ 数据、栈空间、代码三个部分混淆在一起，使得源代码显得非常混乱。
- ➁ 一个代码段最多容纳64kb数据(8086CPU的限制)，因此当整个源码段(数据、栈空间、代码)大于64kb时，不能放在一个段中。

下面考虑将数据、栈空间、代码段分别放在不同的段来进行管理，改写上一节的代码如下：
```txt
assume cs:code,ds:data,ss:stack                         ;使用assume伪指令将段寄存器和标号相对应
data segment
    ;  cs:0  cs:2  cs:4  cs:6  cs:8  cs:10 cs:12 cs:14  ;数据地址
    dw 0123h,0456h,0789h,0abch,0defh,0fedh,0cbah,0987h  ;待处理数据
data ends
stack segment
    ;  cs:16 cs:18 cs:20 cs:22 cs:24 cs:26 cs:28 cs:30  ;栈空间地址
    dw 0,0,0,0,0,0,0,0                                  ;占用空间 在后面程序中作为栈使用
stack ends
code segment
start:  mov ax,stack                                    ;
        mov ss,ax                                       ;借助AX寄存器设置SS段寄存器为stack段起始地址
        mov sp,20h                                      ;设置ss:sp栈顶地址
        mov ax,data                                     ;
        mov ds,ax                                       ;设置ds寄存器指向data数据段起始地址
        mov bx,0                                        ;初始化data数据段偏移变量
        mov cx,8                                        ;初始化循环计数8
    s:  push ds:[bx]                                    ;将data段空间中的数据入栈 sp=sp-2
        add bx,2                                        ;
        loop s                                          ;循环
        mov bx,0                                        ;重置
        mov cx,8                                        ;重置
   s0:  pop ds:[bx]                                     ;将栈空间存储的数据出栈到data段空间 sp=sp+2
        add bx,2                                        ;
        loop s0                                         ;循环
        mov ax,4c00h                                    ;
        int 21h                                         ;21号中断
code ends                                        
end start                                               ;通知编译器源码结束 & 代码执行起始标号start
```
借助上面的程序，有2点需要特别注意：
- ➀ 在源代码中定义多个内存空间的方法：使用"xxx segment ... xxx ends"的结构，不同的段名不能重复。
- ➁ 对于定义的不同的段的引用方式：借助段地址:偏移地址的方式对定义的段空间内部的内存空间进行引用。需要注意：8086CPU不支持直接通过段名对段寄存器DS进行初始化("mov ds,data")。
- ➂ 源码中"代码段"、"数据段"、"栈段"是根据程序设计者功能需要指定的，CPU并不知道程序中"code"、"data"、"stack"的含义，CPU只能通过与之相关联的寄存器对于相应段空间进行访问。而源代码中定义的"具有一定功能的"段的标号和寄存器的关联是通过assume指令来完成的。



## Reference
> \<assembly language by wangshuang 3ed\> chapter6 <br>

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
> 问题脚注: ### ???problem😫problem???
