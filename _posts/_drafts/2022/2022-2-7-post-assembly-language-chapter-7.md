---
layout: post
title: "assembly language (内存空间的灵活定位)"
subtitle: '[assembly language wangshuang 3ed] - chapter7' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2022-02-07 17:21
lang: ch 
catalog: true 
categories: assembly
tags:
  - Time 2022
---
### 本文概述 
在之前的内容中，使用如"[0]"和"[bx]"的方式来定位内存空间单元的物理地址。本文主要通过结合几个具体问题来介绍一些更加灵活的内存空间定位方式。

本文将首先介绍一些背景知识：and指令、or指令，ASCII编码    相关知识，以作为后续内存空间灵活定位内容的基础。


### 一：and指令、or指令
▶︎ **and：逻辑与指令，执行按位与运算(有0则0)** 
```txt
mov al,01100011B  ;设置AX低8-bit寄存器数值
and al,00111011B  ;执行按位于运算 将运算结果保存在al中
                  ;上面代码执行结果为"al=00100011B"。
```
and指令常用的功能为：将待操作寄存器(保存的值)相应位设置为0，参考下面的案例进行理解：
```txt
and al,11111110B  ;将al中存储的数据第0位设置为0
and al,10111111B  ;将al中存储的数据第6位设置为0
and al,01111111B  ;将al中存储的数据第7位设置为0
```

▶︎ **or：逻辑或指令，执行按位或运算(有1则1)** 
```txt
mov al,01100011B  ;设置AX低8-bit寄存器数值
or al,00111011B   ;执行按位或运算 将运算结果保存在al中
                  ;上面代码执行结果为"al=01111011B"。
```
or指令常用的功能为：将待操作寄存器(保存的值)的相应位设置为1，参考下面的案例进行理解：
```txt
or al,00000001B  ;将al中存储的数据第0位设置为0
or al,01000000B  ;将al中存储的数据第6位设置为0
or al,10000000B  ;将al中存储的数据第7位设置为0
```


### 二：ASCII码相关知识介绍
ASCII码是一种编码方案，在计算机领域中被广泛应用。

在计算机中，当人们键入"a"按键，按键信息被送入计算机，计算机将利用ASCII编码规范予以编码，并将对应的ASCII码"61H"存储在相应的存储空间中，计算机程序(e.g.文本编辑软件)将其从内存中取出，送到显卡对应的显存中，显卡将其按照ASCII解码规则予以解析，驱动显示器将"a"字符显示在屏幕上。

简而言之，为了能够在显示器上看到键入的字符"a"，就需要将字符"a"对应的ASCII编码送入显存中。


### 三：在汇编源代码中以"字符"形式给出数据
在汇编源码中，可以使用单引号的方式('xxx')，来表明xxx是一个字符数据，编译器将它们转化成对应的ASCII码。结合下面的程序示例进行理解：
```txt
assume cs:code
data segment
    db 'uniX'  ;以字符形式给出的数据 编译器将其解码成 db 75h 6eh 49h 58h 
    db 'foRK'  ;以字符形式给出的数据 编译器将其解码成 db 66h 6fh 52h 4bh
data ends
code segment
    start:  mov al,'a'    ;编译器将其解码成 mov al,61h
            mov bl,'b'    ;编译器将其解码成 mov bl,62h
            mov ax 4c00h
            int 21h       ;中断
code ends
end start
```


### 四：ASCII编码规则下的字符大小写转换问题
▶︎ **[问题] 给定下面的汇编源码，将数据段中定义的第一个字符串转化成大写，第二个字符串转化成小写** 
```txt
assume cs:codesg,ds:datasg
datasg segment
    db 'BaSiC'
    db 'iNfOrMaTiOn'
datasg ends
codesg segment
    start:
codesg ends
end start
```

▶︎ **[解析] 将字符串进行大小写转换，实质上是借助字符串中每个字符大小写对应的ASCII码字符的规律进行处理** <br>
➤ Solve No.1 利用每个字符大写字符和小写字符之间的关系：
```txt
;ASCII('A') + 20H = ASCII('a')  ;26个字符大写、小写之间的关系
```
因此，完成上述问题的算法中需要借助判断指令(jz、jnz)，以决定当前处理字符是大写还是小写字符。
```txt
jz L1   ;判断ZF ZF=1则跳转标号L1
jnz L1  ;判断ZF ZF=0则跳转标号L2
;小写字符判定条件：ASCII('xxx') > 61H = true
;大写字符判定条件：ASCII('xxx') > 61H = false 
```

➤ Solve No.2 不使用判断语句，借助大写字符和小写字符的2进制表示中的规律进行转化：
```txt
;大写字符的ASCII码的2进制表示中第5位都为0  大写字符ASCII码的2进制表示第5位置为1  则变为小写字符
;小写字符的ASCII码的2进制表示中第5位都为1  小写字符ASCII码的2进制表示第5位置为0  则变为大写字符
;因此 无论给定字符是大写、小写 将其ASCII码2进制表示的第5位置0  则为大写  否则为小写
```
整理上述思路，并借助and指令(有0则0)用于置0，以及or指令(有1则1)用于置1，程序如下所示：
```txt
assume cs:codesg,ds:datasg
datasg segment
    db 'BaSiC'                ;ds:0 ds:1 ds:2 ds:3 ds:4
    db 'iNfOrMaTiOn'          ;ds:5 ...
datasg ends
codesg segment
    start:  mov ax,datasg     ;
            mov ds,ax         ;借助AX寄存器设置DS段寄存器
            mov bx,0          ;bx设置为第一个字符串初始偏移
            mov cx,5          ;更新偏移
        s:  mov al,ds:[bx]    ;使用AX低8bit寄存器暂存
            and al,11011111B  ;使用and操作将第5bit置0 -> 变大写
            mov ds:[bx],al    ;修改原字符串数值
            inc bx            ;更新偏移
            loop s            ;循环
            mov bx,5          ;设置第二个字符串初始偏移
            mov cx,11         ;设置循环计数11次
       s0:  mov al,ds:[bx]    ;
            or al,00100000B   ;使用or操作将第5bit置1 -> 变小写
            mov ds:[bx],al    ;
            inc bx            ;更新偏移
            loop s0           ;循环
            mov ax,4c00h 
            int 21h           ;21号中断
codesg ends
end start
```


### 五：[bx+idata] - 灵活的寻址方式(1)
在之前介绍的内容中，都使用[bx]的方式来用于指定内存单元，本小节将介绍一种更加灵活的方式指定内存单元：[bx+idata]，它的偏移值为"BX寄存器中的数值+idata"。

```txt
mov ax,[bx+200]    ;常用形式  相当于"mov ax,ds:(bx)+200"  (bx)表示bx寄存器内的值
mov ax,[200+bx]    ;等价形式
mov ax,200[bx]     ;等价形式
mov ax,[bx].200    ;等价形式
```


### 六：借助[bx+idata]来处理数中的数据结构中的数组结构
考虑本文[第4部分](http://localhost:4000/assembly/2022/02/07/post-assembly-language-chapter-7/#四ascii编码规则下的字符大小写转换问题)中介绍的大小写转换问题，之前的解法是借助2个循环分别处理数据段给出的2个字符串；现借助[bx+idata]机制，可以在1个循环同时定位2个字符串中的字符，从而将程序简化成1个循环来完成。

简化的程序如下：
```txt
assume cs:codesg,ds:datasg
datasg segment
    db 'BaSiC'                ;ds:0 ds:1 ds:2 ds:3 ds:4
    db 'iNfOrMaTiOn'          ;ds:5 ...
datasg ends
codesg segment
    start:  mov ax,datasg
            mov ds,ax         ;借助AX设置DS段寄存器
            mov bx,0          ;设置偏移变量
            mov cx,5          ;设置循环计数5
        s:  mov al,ds:[bx+0]  ;将指定内存单元存入al -> [bx+0]用于定位第一个字符串
            and al,11011111B  ;and指令将第5bit置0 -> 变大写
            mov ds:[bx+0],al  ;将转化完成的字符送回指定内存单元
            mov al,ds:[bx+5]  ;将制定内存单元存入al -> [bx+5]用于定位第二个字符串
            or al,00100000B   ;or指令
            mov ds:[bx+5],al  ;
            inc bx
            loop s
            mov ax,4c00h
            int 21h           ;21号中断
codesg ends
end start                     ;通知编译器程序终止 & 代码入口
```
上述程序中，将两个字符串偏移定位部分使用[bx+idata]的等价形式进行表示：
```txt
assume cs:codesg,ds:datasg
datasg segment
    ...
datasg ends
codesg segment
    start: ...
        s: mov al,ds:0[bx]   
           and al,11011111B  ;用c语言进行等价描述: a[i] = a[i] & 0xDF
           ...
           mov al,ds:5[bx]   
           or al,00100000B   ;用c语言进行等价描述: b[i] = b[i] | 0x20
           ...
codesg ends
end start
```
可以发现，"0[bx]"与"a[i]"相对应，"5[bx]"与"b[i]"相对应，底层语言为高级语言提供了便利机制。


### 七：SI和DI - 8086中功能和BX相近的寄存器
si和di是8086中功能和bx相近的寄存器，不同的是si和di不能分成两个8bit寄存器来使用。

下面的三组指令等价：
```txt
mov bx,0
mov ax,ds:[bx]  ;简写形式 mov ax,[bx]
mov si,0
mov ax,ds:[si]  ;简写形式 mov ax,[si]
mov di,0
mov ax,ds:[di]  ;简写形式 mov ax,[di]
```
下面的三组灵活定位指令等价：
```txt
mov bx,0
mov ax,ds:[bx+123]  ;简写形式 mov ax,[bx+123]
mov bx,0
mov ax,ds:[si+123]  ;简写形式 mov ax,[si+123]
mov bx,0
mov ax,ds:[di+123]  ;简写形式 mov ax,[di+123]
```


### 八：[bx+si]和[bx+di] - 灵活的寻址方式(2)
[bx+si]和[bx+di]的机制相同，本节以[bx+si]为例对其含义、用法进行简介。

[bx+si]表示段地址为"(ds)"，偏移地址为"(bx)+(si)"，它的三种等价形式如下所示：
```txt
mov ax,ds:[bx+si]  ;完整形式
mov ax,[bx+si]     ;简写形式
mov ax,[bx][si]    ;等价形式
```


### 九：[bx+si+idata]和[bx+di+idata] - 灵活的寻址方式(3)
[bx+si+idata]和[bx+di+idata]的机制相同，本节以[bx+si+idata]为例对其含义、用法进行简介。

[bx+si+idata]表示段地址为"(ds)"，偏移地址为"(bx)+(si)+idata"，它的三种等价形式如下所示：
```txt
mov ax,ds:[bx+si+idata]  ;完整形式
mov ax,[bx+si+idata]     ;简写形式
mov ax,[bx][si].idata    ;等价形式
mov ax,idata[bx][si]     ;等价形式
```


### 十：不同寻址方式的灵活应用 - 待整理
一般情况下，对于想要❮暂存❯的数据，都应该使用❮栈❯数据结构来进行实现。


## Reference
> \<assembly language by wangshuang 3ed\> chapter7 <br>

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
