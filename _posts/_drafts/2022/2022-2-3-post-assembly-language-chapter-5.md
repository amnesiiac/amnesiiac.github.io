---
layout: post
title: "assembly language ([BX]和loop指令)"
subtitle: '[assembly language wangshuang 3ed] - chapter5' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2022-02-03 22:29
lang: ch 
catalog: true 
categories: assembly
tags:
  - Time 2022
---
### 1 [bx]和内存单元
[bx]的使用方式和[0]有些类似，[0]表示地址为DS:0的内存单元，[bx]表示地址为DS:bx的内存单元。参考下面的案例进行理解：
```txt
mov ax,[0]  ;内存单元地址为DS:0 内存单元长度为1word=2byte
mov al,[0]  ;内存单元地址为DS:0 内存单元长度为1byte
```
因此，要完整的描述一个内存单元，需要两部分信息：
- ➀ 内存单元的地址；
- ➁ 内存单元的长度(类型：word/byte)。

于是，[bx]可支持如下两种长度的操作：
```txt
mov ax,[bx]  ;内存单元地址为DS:bx 内存单元长度为1word=2byte
mov al,[bx]  ;内存单元地址为DS:bx 内存单元长度为1byte
```
汇编指令操作的内存单元长度取决于指令中涉及的寄存器能容纳的长度。另外，[bx]还支持的操作有：
```txt
mov [bx],ax  ;其功能等价于((ds)*16+(bx))=(ax)
```

### 2 loop指令 
loop指令和控制汇编程序中的循环逻辑相关，其指令的格式为"loop 标号"。CPU执行loop指令时，需要进行2步操作：
- ➀ (cx)=(cx)-1
- ➁ 判断cx寄存器中的数值：如果"(cx)=0"则向loop指令下面方向继续执行，如果"(cx)!=0"则转到loop指令标号处循环执行程序。

因此，可以发现CX寄存器中的数值决定了loop指令循环体循环执行的次数。<br>
结合案例进行理解：计算2^12的数值。
```txt
assume cs:code
code segment
    mov ax,2
    mov cx,11     ;设置CX寄存器中数值为11, (cx)=0,1,2...11共需执行循环体12次
s:  add ax,ax     ;标号
    loop s        ;loop指令 (cx)=(cx)-1, (cx)!=0跳转到标号s处执行, (cx)=0则继续向下执行后续指令
    mov ax 4c00h
    int 21h
code ends
end
```

### 3 描述内容的符号"()"
为了文章行文描述上的便利，在本文后面部分将使用符号"()"来表示一个寄存器or内存单元中的内容，例如："(ax)"表示寄存器ax中的内容；"(al)"表示低8bit寄存器al中的内容；"(20000H)"表示地址为20000H的内存单元中的内容；假设DS寄存器中内容为ADR1，BX中的内容为ADR2，则"((ds)\*16+(bx))"表示地址为ADR1\*16+ADR2的内存单元中的内容，即ADR1:ADR2中的内容。

关于描述性符号的使用需要注意："()"中只可包含3种元素类型：1) 寄存器名，2) 段寄存器名，3) 内存单元物理地址(20bit)。

### 4 约定符号"idata"表示常量
"idata"的使用方法如："mov ax,[idata]"表示"mov ax,[1]"，"mov ax,[2]"，"mov ax,[3]"... 而"mov bx,idata"表示"mov bx,1"，"mov bx,2"，"mov bx,3"...

### 5 汇编源程序中数据表示的注意事项
在汇编源程序中，数据不能以字母开头，只能以数字开头。例如：数据9138H在汇编源程序中可以直接写为"9138H"，而数据A000H则不能直接写为"A000H"，而应该写为"0A000H"。

### 6 debug程序和汇编编译器对于形如"mov ax,[0]"汇编指令存在不同处理
```txt
mov ax,[0]  ;在汇编编译器中被当作指令"mov ax,0"来处理
mov ax,[0]  ;在汇编debug程序中被当作指令"mov ax,[0]"来处理
```
那么，如何在汇编源程序中书写程序，使得该程序被汇编编译器解释成"mov ax,[0]"？
- ➀ 方法一
```txt
mov ax,2000h
mov ds,ax     ;借助AX寄存器设置DS段寄存器
mov bx,0      ;将偏移地址0送入BX寄存器
mov al,[bx]   ;通过[bx]实现"mov al,[0]"的功能
```
- ➁ 方法二 使用段前缀的方式显式指定段地址：
```txt
mov ax,2000h
mov ds,ax      ;借助AX寄存器设置DS段寄存器
mov al,ds:[0]  ;通过段地址:偏移地址(ds:[0])的方式实现"mov al,[0]"功能
```

### 7 loop和[bx]的联合使用案例分析
计算内存单元从ffff:0到ffff:b中所有数据的和，并将结果存储在DX寄存器中。需要考虑如下几个问题：
- ➀ 内存单元中ffff:0到ffff:b中的数据之和是否会超出DX寄存器的范围？答：ffff:0到ffff:b中存储的数据为8-bit数据(0-255)，12个数据想家结果不会大于65535(16-bit容纳最大数据)。
- ➁ 能否将内存单元ffff:0到ffff:b中的数据直接累加到dx中？答：不可以，ffff:0到ffff:b中存储的数据为8-bit数据，不能直接累加到16-bit寄存器dx中。
- ➂ 能否将内存单元ffff:0到ffff:b中的数据累加到dl中，并设置dh=0？答：不可以，dl是8-bit寄存器，能够容纳(0-255)的数据，12个8-bit数据相加可能出现进位溢出的问题。

为解决上述问题，编写汇编源程序如下：
```txt
assume cs:code
code segment
    mov ax,0ffffh
    mov ds,ax      ;借助AX寄存器设置DS寄存器
    mov dx,0       ;初始化累加寄存器为0
    mov al,ds:[0]  ;将内存单元ffff:0处的数值传递给AX寄存器中低8-bit
    mov ah,0       ;将AX寄存器高8-bit设置成0 
    add dx,ax      ;使用add指令将AX寄存器中的数值累加到DX寄存器中
    mov al,ds:[1]  ;同理
    mov ah,0       ;
    add dx,ax      ;
    ...            ;12次
code ends
end
```
考虑到上面程序中存在重复部分，借助loop指令以及[bx]支持变长度寄存器间传值特性，实现对上述程序的简写如下：
```txt
assume cs:code
code segment
    mov ax,0ffffh  
    mov ds,ax      ;借助AX寄存器设置DS寄存器
    mov bx,0       ;BX寄存器用于保存每次循环偏移地址变量
    mov dx,0       ;初始化累加寄存器为0
    mov cx,12      ;设置循环12次
s:  mov al,[bx]    ;
    mov ah,0       ;
    add dx,ax      ;
    inc bx         ;更新循环偏移地址变量
    loop s         ;循环
    mov ax,4c00h   ;
    int 21h        ;
code ends
end
```

### 8 随意向一段未知内存写入数据是危险的
```txt
mov ax,1000h
mov ds,ax      ;设置DS段寄存器
mov al,0       ;设置al初始值
mov ds:[0],al  ;随意向ds:[0]未知内存处写入数据是十分危险的
```
因此，当我们向内存写入数据时，一种安全的做法是使用操作系统分配的空间；但是我们也应该尽量充分理解计算机底层工作原理，在不产生冲突的条件下，直接对硬件进行编程。

##### 8.1 CPU实模式和CPU保护模式下汇编语言的"权限"不同
在DOS系统下(CPU实模式)，可以忽略DOS，直接通过汇编语言去操作真实硬件，这是因为运行在CPU实模式下的DOS系统没有能力对计算机硬件进行全面、严格地管理。但在windows 2000、unix这些运行在CPU保护模式下的系统中，硬件已经被操作系统借助CPU保护模式予以全面、严格的控制起来了，因此无法借助汇编语言操作真实硬件。

##### 8.2 DOS系统下的一段合法空间"0:200 - 0:2ff"
一般的计算机中，在DOS模式下，DOS本身和其他合法程序都不会使用0:200～0:2ff这256kb的空间，因为：。因此后续需要向一段内存中写入数据时，就选取这段内存空间。


## Reference
> \<assembly language by wangshuang 3ed\> chapter5 <br>

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
