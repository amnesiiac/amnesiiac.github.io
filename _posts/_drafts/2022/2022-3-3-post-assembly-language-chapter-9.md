---
layout: post
title: "assembly language (转移指令的原理)"
subtitle: '[assembly language wangshuang 3ed] - chapter9' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: true
date: 2022-03-03 14:31
lang: ch 
catalog: true 
categories: assembly
tags:
  - Time 2022
---
### 本文概述 
可以修改IP或者同时修改CS:IP的指令称为❮转移指令❯，该指令主要用于控制CPU执行指定位置的代码。

▶︎ __8086CPU的转移方式分类__ <br>
➀ 段内转移(只修改IP寄存器)：e.g. "jmp ax"。根据转移的距离，段内转移又可分为：<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
➊ 短转移：IP寄存器的修改范围为-128~127 = $-2^7$~$2^7$-1，单位：byte。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
➋ 近转移：IP寄存器的修改范围为-32768~32767 = $-2^{15}$~$2^{15}$-1，单位：byte。<br>
➁ 段间转移(同时修改CS,IP寄存器)：e.g. "jmp 1000:0"。<br>

▶︎ __8086CPU的转移指令分类__ <br>
➀ 无条件转移指令，例如：jmp <br>
➁ 条件转移指令，例如：jz、jnz... <br>
➂ 循环指令，例如：loop <br>
➃ 过程 <br>
➄ 中断 <br>
上述5种转移指令发生转移的前提条件可能不同，但它们的转移原理是基本相同的。本文主要通过"jmp"无条件指令来理解CPU转移指令的基本原理。

### 一：操作符offset
操作符offset是有编译器处理的符号，它的功能是：取得和标号对应的偏移地址，参考下面程序理解：
```txt
assume cs:codesg
codesg segment
    start: mov ax,offset start  ;等价于"mov ax,0" 将start标号的段偏移地址存入ax
        s: mov ax,offset s      ;等价于"mov ax,3" 将s标号的段偏移地址存入ax (第一条指令长度为3word)
codesg ends
end start
```

### 二：jmp指令
jmp指令为无条件转移指令，既可以实现段内转移(IP)，也可以实现段间转移(CS:IP)。<br>
因此，使用jmp指令时，需要给出2方面信息：<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
➀ 转移的目的地址。 <br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
➁ 转移的距离：段内短转移、段内长转移、段间转移。<br>
不同的转移方式对应了不同的指令，在本文后续部分分别介绍。

### 三：jmp指令：利用标号"相对位置"用于段内转移 (短转移、近转移)
▶︎ __jmp short 标号: 无条件段内短转移指令__ <br>
结合下面案例对jmp short实现短转移的机制进行理解：
```txt
assume cs:codesg
codesg segment
start: mov ax,0
       mov bx,0
       jmp short s  ;段内短转移到标号s执行 (无条件短转移)
       add ax,1     ;被跳过
    s: inc ax       ;跳转到此处执行 此时ax=1
codesg ends
end start
```
在dos debug程序中，利用"-u"选贤将上述程序机器码反汇编成汇编程序
```txt
debug -u  ;使用-u选项将机器码反汇编至汇编程序
0BBD:0000 B80000   ---->  MOV AX,0000
0BBD:0003 BB0000   ---->  MOV BX,0000
0BBD:0006 EB03     ---->  JMP 000B     ;转移IP:000B EB为JMP对应的机器码 03为IP偏移地址增加值3byte
0BBD:0008 050100   ---->  ADD AX,0001
0BBD:000B 40       ---->  INC AX       ;跳转标号位置
```
注意，"JMP 0008"跳转指令对应的机器码为"EB03"，并不包含跳转的目的偏移地址"0008"，那么计算机执行机器码时是如何确定跳转的目的地址呢？

首先回忆chapter2中汇编指令的执行过程：<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
➀ 从cs:ip指向的内存单元中读取指令，并将指令存入指令缓冲器。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
➁ (ip)=(ip)+读取指令的长度，从而指向下一条待运行指令。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
➂ 执行指令，然后转到步骤➀重复流程。<br>

结合上述读取/执行流程易知：在执行JMP指令之前，IP中的偏移地址已经指向"物理上"的下一条待执行指令的地址："0BBD:0000"。
在编译JMP指令时，将"跳转目的地址"相对"当前地址的距离"编译进机器码中：
```txt
debug -u  
0BBD:0000 B80000   ---->  MOV AX,0000
0BBD:0003 BB0000   ---->  MOV BX,0000  ;cs:ip=0bbd:0006
0BBD:0006 EB03     ---->  JMP 0008     ;机器码EB03的由来: 在编译前ip=0008 编译后ip=ip+03=000b 
0BBD:0008 050100   ---->  ADD AX,0001  ;
0BBD:000B 40       ---->  INC AX       ;
```
因此，得到 __jmp short 标号__ 实际原理：
```txt
jmp short s  (ip)=(ip)+8bit_offset  实现了段内"-128~127"的范围转移(byte)
```

▶︎ __jmp near ptr 标号: 无条件段内近转移指令__
```txt
jmp near ptr s  ;(ip)=(ip)+16bit_offset  实现了段内"-(16^4)~(16^4-1)"的范围转移(byte)
```

### 四：jmp指令：利用标号的cs:ip进行段间转移 (远转移)
上一节介绍的jmp指令对应的机器指令中并没有转移的目的地址，而是当前IP的相对位移。

▶︎ __jmp far ptr 标号: 无条件段间远转移指令__ <br>
其中"(cs)"存放了标号的段地址；"(ip)"存放了标号的偏移地址；"far ptr"指明了使用标号的"cs:ip"地址更新当前cs、ip寄存器。结合下面例程进行理解：
```txt
assume cs:codesg
codesg segment
    start: mov ax,0
           mov bx,0
           jmp far ptr s   ;无条件段间远转移指令 使用标号s的cs:ip更新当前cs:ip
           db 256 dup (0)  ;被跳过
        s: add ax,1        ;标号s cs:ip
           inc ax
codesg ends
end start
```
在编译JMP指令时，利用"标号s对应的地址cs:ip"更新当前cs、ip寄存器中的数值：
```txt
debug -u  ;使用-u选项将机器码反汇编至汇编程序
0BBD:0000 B80000      ---->  MOV AX,0000
0BBD:0003 BB0000      ---->  MOV BX,0000
0BBD:0006 EA0B01BD0B  ---->  JMP 0BBD:010B   ;标号s的cs:ip=0BBD:010B 被用于更新当前cs、ip寄存器数值
0BBD:000B 0000        ---->  ADD [BX+SI],AL  ;db 256 dup (0)被debug程序解释成若干条汇编指令
0BBD:000D 0000        ---->  ADD [BX+SI],AL  ;
0BBD:000F 0000        ---->  ADD [BX+SI],AL  ;
0BBD:0011 0000        ---->  ADD [BX+SI],AL  ;
     ...                           ...
0BBD:010B xxxx        ---->  ADD AX,1        ;s标号处cs:ip数值
```
分析上述 __jmp far ptr s__ 编译成的机器码：
```txt
;EA表示JMP指令 而"0B 01 BD 0B"是标号s的地址的逆序 数据在内存中的存储顺序使用逆序 方便在读取时自动变为正序
0BBD:0006 EA0B01BD0B  ---->  JMP 0BBD:010B   
```

### 五：jmp指令：(ip)=16bit\_reg 实现跳转
```txt
jmp 16bit_reg  ;指令格式
jmp ax         ;使用ax中的数值修改ip寄存器中的数值 (ip)=(ax)
```

### 六：jmp指令：跳转的目的地址在[内存单元]中
▶︎ __jmp word ptr [内存单元]: 无条件❮段内❯转移指令__ <br>
功能：从指定内存单元地址起始处存放1word数据，为转移的目的地址。<br>
注意： __jmp word ptr [...]__ 指令中的内存单元地址可以用任意一种内存单元寻址方式给出:
```txt
;方式(1)
mov ax,0123H       
mov ds:[0],ax        ;借助ax设置ds段寄存器
jmp word ptr ds:[0]  ;内存地址单元 ds:[0] --> (ip)=0123H
```
```txt
;方式(2)
mov ax,0123H
mov [bx],ax          ;换一种方式实现相同的跳转
jmp word ptr ds:[0]  ; ds:[0] --> ip
```

▶︎ __jmp dword ptr [内存单元]: 无条件❮段间❯转移指令__ <br>
功能：根据指定[内存单元]地址处存放double word数据来定位跳转的目标地址。<br>
跳转目标段地址: (cs)=(内存单元地址+2byte) <br>
跳转目标偏移地址: (ip)=(内存单元地址) <br>

注意： __jmp dword ptr [...]__ 指令中的内存单元地址可以用任意一种内存单元寻址方式给出:
```txt
;方式(1)
mov ax,0123H
mov ds:[0],ax          ;借助ax设置ds:[0] 内存单元地址=(ip)=0123H
mov word ptr ds:[2],0  ;设置ds:[2] 内存单元地址+2byte=(cs)=0H
jmp dword ptr ds:[0]   ;ds:[0]-->ip; ds:[2]-->cs; 跳转目标地址=0000H:0123H
```
```txt
;方式(2)
mov ax,0123H
mov bx,ax          
mov word ptr [bx+2],0  ;换一种内存单元访问方式实现相同的跳转
jmp dword ptr ds:[0]   ;ds:[0]-->ip; ds:[2]-->cs; 跳转目标地址=0000H:0123H
```
### 七：jcxz指令：条件转移指令
jcxz指令为有条件转移指令，所有有条件转移指令都是短转移。因此，有条件转移指令对应的机器码中包含转移的位移，而不是转移的目的地址；(ip)的修改范围是-128~127。

➀ 指令格式为 __jcxz 标号__。<br>
➁ 指令含义：如果(cx)=0，则跳转到标号处执行。<br>
➂ 实现原理同 __jmp near 标号__ 类似：
```txt
;jcxz  = judge (cx) z
jcxz 标号  ;(ip)=(ip)+8bit_offset  实现了-128~127范围内的转移 (byte)
```
➃ 跳转的位移大小计算方式：
```txt
8bit_offset = 标号处地址 - jcxz指令后的第一个字节的地址
```
➄ 跳转的位移范围为-128~127，用补码表示。<br>
➅ 跳转的8bit_offset由编译jcxz指令时计算得出。<br>
➆ 当(cx)≠0时，不执行跳转，程序顺序执行。<br>
➇ __jcxz 标号__ 条件跳转指令逻辑上等价于：
```txt
if((cx)==0){
    jmp short 标号
}
```

### 八：loop指令：循环条件转移指令
loop指令是循环指令，所有循环指令都是短转移。另外，loop指令还是有条件转移指令，根据上一节描述：所有有条件转移都是短转移。因此，loop指令编译得到的机器码中包含转移的位移，而不是目的地址，(ip)的修改范围是-128~127。

➀ 指令格式为 __loop 标号__。<br>
➁ 指令含义：如果(cx)≠0，则跳转到标号处执行。<br>
➂ 实现原理：
```txt
(cx)=(cx)-1; 
如果(cx)≠0 (ip)=(ip)+8bit_offset;
```
➃ 跳转的位移大小计算方式：
```txt
8bit_offset = 标号处的地址 - loop指令后的第一个字节的地址
```
➄ 跳转的位移范围是-128~127，用补码表示。<br>
➅ 跳转的8bit_offset由编译loop指令时计算得出。<br>
➆ 当(cx)=0时，不执行跳转，程序顺序执行。<br>
➇ __loop 标号__ 循环条件跳转指令逻辑上等价于：
```txt
;标号所在的2种位置: loop前/loop后
标号1: (cx)=(cx)-1;
if((cx)≠0){
    jmp short 标号
}
标号2:
```

### 九：根据位移进行跳转的意义
本文中介绍的如下几种短跳转指令，对于(ip)的修改都是根据：目的地址和转移起始地址之间的 __位移__ 进行的。
```txt
jmp short 标号     ;无条件短转移
jmp near ptr 标号  ;无条件近转移
jcxz 标号          ;有条件短转移
loop 标号          ;循环有条件短转移
```
4种短转移指令经过编译得到的机器码中，不包含跳转的目的地址，而是包含到目的地址的 __位移__ 。

▶︎ __根据"相对地址"而不是"绝对地址"进行跳转的优点__ <br>
一言蔽之：根据"相对地址"进行跳转，有利于程序段在内存中的浮动装配。

下面的汇编程序，无论装载在内存中的那个位置，都能够正常工作；因为，无论代码位于何处，其跳转的相对位移是固定的。
```txt
汇编代码              机器代码
   mov cx,6    ---->  B9 06 00
   mov ax,10h  ---->  B8 10 00
s: add ax,ax   ---->  01 C0
   loop s      ---->  E2 FC
```
如果loop指令跳转是根据绝对地址进行的，则当该代码片段被复制到内存其它位置，将得到非期望的运行结果。

### 十：编译器对跳转目标位置的越界检测
根据位移进行转移的指令，其转移范围受转移位移的限制。如果在汇编源代码中转移目的位置越界，则在编译时期将报错。

下面的汇编源代码在编译期将会报错：
```txt
assume cs:codesg
codesg segment
    start: jmp short s     ;无条件段内短转移指令  跳转范围-128~127  无法跳转到目的地址
           db 128 dup (0)  ;在跳转指令后 定义了128byte数据0
        s: mov ax,0ffffh   ;目的地址 标号s
codesg ends
end start
```
另外，需要注意，在debug程序中使用的指令，在汇编源程序中可能会报错：
```txt
jmp 2000:0100  ;在debug程序中 用于跳转到指定目的地址 如果在汇编源程序中使用会报错
```


## Reference
> \<assembly language by wangshuang 3ed\> chapter9 <br>

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
> 问题脚注: ### ???problem😫problem???  大标题标识符：▶︎ 小标题标识符：➤ <br>
> 实现文章排版tab：`&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;` <br>
> ⓪➀➁➂➃➄➅➆➇➈➉ ➊➋➌➍➎➏➐➑➒➓
