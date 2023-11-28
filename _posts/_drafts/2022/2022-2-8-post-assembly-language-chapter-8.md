---
layout: post
title: "assembly language (数据处理的基本问题)"
subtitle: '[assembly language wangshuang 3ed] - chapter8' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2022-02-08 15:50
lang: ch 
catalog: true 
categories: assembly
tags:
  - Time 2022
---
### 本文概述 
计算机是进行数据处理、运算的机器，与之相关的两个基本问题：<br>
➀ 待处理的数据在计算机的什么地方？<br>
➁ 待处理的数据占用多少内存空间？<br>
本文后续部分内容将针对8086CPU上的这两个问题进行讨论、研究。虽然本文以8086CPU为例，但是这两个基本问题在任何一个处理器上都普遍存在。

为了描述方便，现定义2个描述性符号：reg、sreg，分别表示寄存器、段寄存器。<br>
reg包含：ax、bx、cx、dx；ah、al、bh、bl、ch、cl、dh、dl；sp、bp；si、di。<br>
sreg包含：ds、ss、cs、es。

### 一：bx、si、di、bp四个特殊寄存器用法总结
- ➀ 在8086CPU中，只有bx、bp、si、di四个寄存器可以用在"[...]"中来进行内存单元的寻址。<br>
```txt
mov ax,[bx]  ;(√)
mov ax,[bp]  ;(√)
mov ax,[si]  ;(√)
mov ax,[di]  ;(√)
mov ax,[ax]  ;ax不属于上述4个寄存器 (×)
```
- ➁ 在"[...]"中，bx、bp、si、di可以单个出现，或者以4种组合出现，但只能选择如下4种组合之一：bx和si、bx和di、bp和si、bp和di。<br>
```txt
mov ax,[bx+si]        ;(√)
mov ax,[bx+di]        ;(√)
mov ax,[bp+si]        ;(√)
mov ax,[bp+di]        ;(√)
mov ax,[bx+si+idata]  ;(√)
mov ax,[bx+di+idata]  ;(√)
mov ax,[bp+si+idata]  ;(√)
mov ax,[bp+di+idata]  ;(√)
mov ax,[bx+bp]        ;bx/bp不是上述4种组合之一 (×)
mov ax,[si+di]        ;si/di不是上述4种组合之一 (×)
```
- ➂ 只要在"[...]"中使用了寄存器bp，而指令中没有显式给出段地址，则段地址默认在段寄存器ss中。
```txt
mov ax,[bp]           ;等价于"mov ax,ss:[bp]"
mov ax,[bp+idata]     ;等价于"mov ax,ss:[bp+idata]"
mov ax,[bp+si]        ;等价于"mov ax,ss:[bp+si]"
mov ax,[bp+si+idata]  ;等价于"mov ax,ss:[bp+si+idata]"
```

### 二：机器指令处理的数据在什么地方
绝大多数机器指令都用于数据处理，处理方式可分成3类：读、写、运算。从机器指令的角度讲，它只关心指令执行前，所处理的数据所在位置。<br>
指令执行前，待处理数据可以在3个位置：**CPU内部、内存、端口**。其中，待操作数据位于端口的情况在本系列chapter14中进行讨论。

▶︎ __机器指令执行前，待操作数据所在的位置__ <br>
<center><img src="/img/in-post/assembly_language_img/chapter_8_1.pdf" width="60%"></center>

### 三：汇编语言中数据位置的表达
总结汇编语言中数据位置表达方式有3种：
- ➀ __立即数(idata)__ 待处理数据在指令执行前位于CPU内部的指令缓冲器中。<br>
```txt
mov ax,1
add bx,2000h
or bx,00010000b
mov al,'a'
```
- ➁ __寄存器__ 待处理数据在CPU内部的寄存器中，在汇编指令中给出相应寄存器名。<br>
```txt
mov ax,bx
mov ds,ax
```
- ➂ __段地址(segment address, SA)+偏移地址(excursion address, EA)__ 待处理数据在内存中，内存单元的地址由SA:EA给出。<br>
```txt
;默认使用ds段寄存器
mov ax,[0]        
mov ax,[bx+8]
mov ax,[bx+si+9]
;默认使用ss段寄存器
mov ax,[bp]
mov ax,[bp+si+8]
;显式指定使用ds段寄存器
mov ax,ds:[bp]
```

### 四：8086CPU寻址方式总结
<center><img src="/img/in-post/assembly_language_img/chapter_8_2.pdf" width="100%"></center>

### 五：(机器指令) 待操作内存长度的指明方式
8086CPU的机器指令可以处理两种长度的数据：byte、word。因此，在8086CPU的指令需要指明进行的是"字操作"还是"字节操作"。指明待操作数据的尺寸的方式如下所示：
- ➀ 通过寄存器名，来指明待操作内存的长度：<br>
```txt
;下面指令通过寄存器名 指明执行word操作
mov ax,1
mov bx,ds:[0]
mov ds,ax
mov ds:[0],ax
inc ax
add ax,1000
;下面指令通过寄存器名 指明执行byte操作
mov al,1
mov al,bl
mov al,ds:[0]
mov ds:[0],al
inc al
add al,100
```
- ➁ 在机器指令中不存在寄存器名的情况下，可以显式通过"word/byte ptr"指明待操作内存长度为word/byte。
```txt
;下面指令通过"word ptr" 指明执行word操作
mov word ptr ds:[0],1
mov word ptr [bx]
inc word ptr ds:[0]
add word ptr [bx],2
;下面指令通过"byte ptr" 指明执行byte操作
mov byte ptr ds:[0],1
mov byte ptr [bx]
inc byte ptr ds:[0]
add byte ptr [bx],2
```
为"没有寄存器参与的内存单元的访问指令"显式指明所访问的内存单元的长度是十分必要的。结合下面的案例理解：<br>
假设用debug程序查看从2000H:1000H起始的内存空间序列状态如下所示：
```txt
2000: 1000 FF FF FF FF FF FF ...
```
则使用"word/byte ptr"对于内存中数据的修改将导致2种不同的结果：
```txt
mov ax,2000H  ;公共部分
mov ds,ax     ;公共部分
```
```txt
;byte ptr访问ds:[1000H]内存单元 对单个内存单元执行写入操作
mov byte ptr ds:[1000H],1  
```
```txt
;word ptr访问ds:[1000H]内存单元 对ds:[1000H]和ds:[1001H]两个单元执行写入操作
mov word ptr ds:[1000H],1  
```
- ➂ 其它方式：部分机器指令默认了操作的内存单元长度。参考下面例子理解：
```txt
mov ax,2000H
mov ds,ax
push [1000H]  ;无需指明对word/byte进行操作 push指令默认对2个内存单元(word)进行操作
```

### 六：多种寻址方式的综合应用 - 综合案例分析
关于DEC公司的一条1982年的信息记录如下：
```txt
公司名称: DEC
总裁姓名: Ken Olsen
排名: 137
收入: 40(亿美元)
知名产品: PDP(小型机)
```
这些数据在计算机内存空间中按照如下格式存放：
<center><img src="/img/in-post/assembly_language_img/chapter_8_3.pdf" width="90%"></center>

到1988年，DEC公司相关信息更新如下：
```txt
公司名称: DEC
总裁姓名: Ken Olsen
排名: 38 (-99)
收入: 110(亿美元) (+70)
知名产品: VAX(计算机)
```
根据DEC公司信息从1982年-1988年的变动，编写修正内存空间数据信息的汇编程序如下：
```txt
;(1) 借助ax,bx确定数据记录起始地址 ds:[bx]
mov ax,seg
mov ds,ax
mov bx,60H
;(2) 修改总裁财富排名、处理公司收入增加
mov word ptr [bx+0ch],38      ;修改位于ds:[bx+0ch]处的长度为1word的排名字段
add word ptr [bx+0eh],70      ;修改位于ds:[bx+0eh]处的长度为1word的公司收入字段
;(3) 逐byte修改公司知名产品
mov si,0                      ;设置偏移寄存器初值
mov byte ptr [bx+si+10h],'V'  ;修改位于ds:[bx+10h]处的长度为1byte的字符数据
inc si                        ;更新偏移值
mov byte ptr [bx+si+10h],'A'  ;修改位于ds:[bx+11h]处的长度为1byte的字符数据
inc si                        ;更新偏移值
mov byte ptr [bx+si+10h],'X'  ;修改位于ds:[bx+12h]处的长度为1byte的字符数据
```
使用c语言，再次编写修改上述数据结构的程序如下：
```cpp
struct company{// 逻辑内存空间中存放的公司数据被抽象成c语言的结构体(c++中的类型)
    char company_name[3];
    char president_name[9];
    int company_rank;
    int income;
    char well_known_product[3];
}
struct company dec={"DEC", "Ken Olsen", 137, 40, "PDP"}
int main(){
    // (2) 修改总裁财富排名、处理公司收入增加
    dec.company_rank = 38; 
    dec.income = dec.income + 70;
    // (3) 逐byte修改公司知名产品
    int i=0;
    dec.well_known_product[i] = 'V';
    i++;
    dec.well_known_product[i] = 'A';
    i++;
    dec.well_known_product[i] = 'X';
    return 0;
}
```
最后，为了更好地理解汇编语言和c语言之间的对应关系，在同一幅图中给出两者对应代码：
<center><img src="/img/in-post/assembly_language_img/chapter_8_4.pdf" width="65%"></center>

c语言中结构体的定义，及其实例化分配具体内存时的对应关系图：
<center><img src="/img/in-post/assembly_language_img/chapter_8_5.pdf" width="90%"></center>

### 七：div指令
<center><img src="/img/in-post/assembly_language_img/chapter_8_6.pdf" width="100%"></center>
<center><img src="/img/in-post/assembly_language_img/chapter_8_7.pdf" width="100%"></center>
<center><img src="/img/in-post/assembly_language_img/chapter_8_8.pdf" width="100%"></center>
<center><img src="/img/in-post/assembly_language_img/chapter_8_9.pdf" width="100%"></center>
<center><img src="/img/in-post/assembly_language_img/chapter_8_10.pdf" width="100%"></center>

### 八：伪指令dd
本系列之前的文章中，用"db"和"dw"来定义字节型数据、字型数据。本小节介绍的"dd"指令用来定义双字型数据。参考下面的例子进行理解：
```txt
data segment
    db 1  ;define byte 以data:0内存单元为起始定义了一个<字节型>数据 01H
    dw 1  ;define word 以data:1内存单元为起始定义了一个<字型>数据 0001H
    dd 1  ;define doubleword 以data:3内存单元为起始定义了一个<双字型>数据 0000 00001H
data ends
```
"db"、"dw"、"dd"都是伪指令，伪指令是由编译器识别并处理的符号。

### 九：dup操作符
"dup"是一个操作服，同上一节中介绍的"dd"、"dw"、"dd"一样，它们都是由编译器处理的符号。"dup"和"dd"、"dw"、"dd"指令配合使用，可以非常方便的定义大量重复的数据。

▶︎ __dup操作符的简单应用案例__ <br>
```txt
db 3 dup (0)            ;定义了3byte 等价于"db 0,0,0" 
db 3 dup (0,1,2)        ;定义了9byte 等价于"db 0,1,2,0,1,2,0,1,2"
db 3 dup ('abc','ABC')  ;定义了18byte 等价于"db 'abcABCabcABCabcABCabcABCabcABCabcABC'"
```

▶︎ __dup操作符用法总结__ <br>
```txt
db 重复次数 dup (重复的字节型数据)
dw 重复次数 dup (重复的字型数据)
dd 重复次数 dup (重复的双字型数据)
```

▶︎ __dup操作符在"大量重复数据定义上"的便利__ <br>
```txt
stack segment
    dw 0,0,0,0,0,0,0,0
    dw 0,0,0,0,0,0,0,0
    dw 0,0,0,0,0,0,0,0
    dw 0,0,0,0,0,0,0,0
stack ends
```
使用dup操作符进行简化定义如下：
```txt
stack segment
    dw 32 dup (0)
stack ends
```


## Reference
> \<assembly language by wangshuang 3ed\> chapter8 <br>

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
