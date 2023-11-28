---
layout: post
title: "assembly language (标志寄存器)"
subtitle: '[assembly language wangshuang 3ed] - chapter11' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2022-03-06 12:26
lang: ch 
catalog: true 
categories: assembly
tags:
  - Time 2022
---
### 本文概述 
CPU内部寄存器中，有一种具有如下3种作用的特殊寄存器：<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
➀ 用来存储相关指令的某些计算结果。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
➁ 用来为CPU执行相关指令提供依据。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
➂ 用来控制CPU相关工作方式。<br>
这些特殊的寄存器在8086CPU中，被称为标志寄存器(flag)。8086CPU的标志寄存器(flag)有16bit，其中存储的信息被称为程序状态字(PSW)。

▶︎ __8086CPU的flag寄存器结构__ <br>
<center><img src="/img/in-post/assembly_language_img/chapter_11_1.pdf" width="100%"></center>

本文后续部分，我们学习标志寄存器中的CF、PF、ZF、SF、OF、DF标志位，以及一些相关典型指令。

### 一：ZF标志位: zero flag
➀ ZF标志位用法：记录相关指令执行后，结果是否为0：如果结果为0，则ZF=1，否则ZF=0。<br>
➁ ZF标志位应用案例：
```txt
mov ax,1
sub ax,1  ;sub指令执行后 结果=0 ZF=1
```
```txt
mov ax,2
sub ax,1  ;sub指令执行后 结果≠0 ZF=0
```
```txt
mov ax,1
and ax,0  ;and指令执行后 结果=0 ZF=1 (有0则0)
```
```txt
mov ax,1
or  ax,0  ;or指令执行后  结果≠0 ZF=0 (有1则1)
```
▶︎ __影响标志寄存器的指令__ <br>
8086CPU的指令集中，有的指令影响标志寄存器，比如：
```txt
add(+), sub(-), mul(×), div(÷), inc(++), or(||), and(&) ...
```
它们大多属于逻辑、算术运算指令；有的指令对标志寄存器没有影响，比如：
```txt
mov, push, pop ...
```
在使用指令时，需要注意这条指令的全部功能，其中包括：执行结果对标志寄存器的哪些标志位造成影响。

### 二：PF标志位：parity flag
➀ PF标志位用法：记录相关指令执行后，结果所有bit中1的个数是否为偶数：如果为偶数，则PF=1，否则PF=0。<br>
➁ PF标志位应用案例：
```txt
mov al,1
add al,10  ;结果为00001011B 有3个1 则PF=0
```
```txt
mov al,1
or al,2    ;结果为00000011B 有2个1 则PF=1
```
```txt
mov al,1
sub al,al  ;结果为00000000B 有0个1 则PF=1
```

### 三：SF标志位：sign(+/-) flag
➀ SF标志位用法：记录相关指令执行后，结果(忽略计算溢出)是否为负数：如果为负数，则SF=1，否则SF=0。<br>
➁ SF标志位应用案例：
```txt
mov al,10000001B  ;
add al,1          ;(al) = 10000010B < 0, SF=1, 表明如果进行的为有符号数运算 则结果为负
```
```txt
mov al,10000001B  ;
add al,01111111B  ;(al) = 0, SF=0, 表明如果进行有符号数运算 则结果(忽略计算溢出)为正
```

▶︎ __知识穿插1：计算机中的数值表示方法(计算机基础)__ <br>
计算机通常使用补码表示有符号数据，2点原因：<br>
➀ 使用补码表示数据，在执行数值计算(+/-)中可以将补码中的符号位看成是数值位统一处理；在计算机电路中只需实现加法器即可完成+/-运算。<br>
➁ 补码能够解决：计算机中0编码唯一性问题，即使用反码对0进行编码时，将0看作是正/负数所得到的反码表示形式不同：
```txt
(正数0)原 = 0000 0000B 
(+0)反 = 0000 0000B  ;当0作为正数 其反码=原码
(负数0)原 = 1000 0000B
(-0)反 = 1111 1111B  ;当0作为负数 其反码=符号位不变 数值位取反
```
➤ 原码、反码、补码求解：
```txt
(+127)原 = 0111 1111B  ;正数的原码 = 符号位 + 真值的绝对值
(+127)反 = 0111 1111B  ;正数的反码 = 原码
(+127)补 = 0111 1111B  ;正数的补码 = 原码
(-127)原 = 1111 1111B  ;符号位1表示正数, 数值位111 1111B表示127
(-127)反 = 1000 0000B  ;反码 = 原码(符号位不变1, 数值位取反: 000 0000B) = 1000 0000B 
(-127)补 = 1000 0001B  ;补码 = 原码(符号位不变1, 数值位取反+1: 000 0001B) = 1000 0001B
```
```txt
(+128)原 = 0000 0000 1000 0000B  ;8bit二进制只能表示[-128,127] 128越界 因此需要16bit才能表示
(+128)反 = 0000 0000 1000 0000B  ;正数的反码 = 原码
(+128)补 = 0000 0000 1000 0000B  ;正数的补码 = 原码
(-128)原 = 1000 0000 1000 0000B  ;同(-127)
(-128)反 = 1111 1111 0111 1111B  ;同(-127)
(-128)补 = 1111 1111 1000 0000B  ;同(-127)
```
➤ 为什么计算机中8bit数据表示[-128,127]而不是[-127,127]或者[-127,128]？<br>
由于底层电路实现+/-计算简便性的需要，计算机使用补码形式存储数据。
8bit有符号数据表示如下所示，结合下图分析，易知令其表示[-128,127]规律性最强。
<center><img src="/img/in-post/assembly_language_img/chapter_11_2.pdf" width="90%"></center>

▶︎ __知识穿插2：计算机中的有符号数、无符号数之间的界定__ <br>
计算机中的一个数据既可以看成是有符号数，也可以看成是无符号数：
```txt
0000 0001B  ;可以看成是无符号数1 或着看成是有符号数+1
1000 0001B  ;可以看成是无符号数129 也可以看作有符号数-127
```
执行如下指令，无论将al看成有符号还是无符号数，其按照补码形式执行add指令的结果都相同：
```txt
mov al,10000001B  ;(al)=1000 0001B 计算机可以将al作为有符号数 也可以看成无符号数
add al,1          ;(al)=1000 0010B 无论计算机将al如何看待 CPU在执行add指令时 已经包含了2种含义:
                  ;如果将al看成<无符号数> 则add指令结果可看成 129+1=130 (1000 0010B)
                  ;如果将al看成<有符号数> 则add指令结果可看成 -127+1=-126 (1000 0010B)
```

### 四：CF标志位：carry flag
➀ CF标志位用法：记录❮无符号数❯运算时，结果最高有效位向更高位的进位/借位值：CF=1表示进/借位为1，CF=0表示进/借位为0。<br>
<center><img src="/img/in-post/assembly_language_img/chapter_11_3.pdf" width="70%"></center>
➁ CF标志位应用案例：
```txt
mov al,98h  ;加法进位:
add al,al   ;执行后 (al)=30h, CF=1, 表明16进制计算时: 向"假想"更高位进位值=1 有进位
add al,al   ;执行后 (al)=60h, CF=0, 表明16进制计算时: 向"假想"更高位进位值=0 无进位
```
```txt
mov al,97h  ;减法借位:
sub al,98h  ;执行后 (al)=FFh, CF=1, 表明16进制计算时: 向"假想"更高位借位值=1 有借位
sub al,al   ;执行后 (al)=0h , CF=0, 表明16进制计算时: 向"假想"更高位借位值=0 无借位
0    9     7h
0    9     8h
---------------
1,16+8-9,16+7-8 = CF=1,F,F
```
```txt
mov ax,2   ;无符号数: (ax)=2
mov bx,1   ;无符号数: (bx)=1
sub bx,ax  ;(bx)= 2-1 = -1 => CF=1 10进制运算涉及了借位
```
```txt
mov ax,1   ;无符号数: (ax)=1
add ax,ax  ;无符号数: (ax)=2 => CF=0 10进制运算不涉及进位
```

### 五：OF标志位：overflow flag
➀ OF标志位用法：记录❮有符号数❯运算时，如果运算中发生溢出，则OF=1，否则OF=0。<br>
➁ 溢出：进行有符号数计算时，如果计算结果超过了❮机器所能表示的范围❯则称为溢出，此时将得到❮错误的运算结果❯。无符号数进行计算时，不存在溢出问题，因为计算结果自动"n-bit截断"，超范围的进/借位由CF表示。<br>
➂ 机器能够表示的范围：指令运算结果用寄存器、内存单元来存放；因此，寄存器、内存单元能够表示的有符号数范围，即为机器能够表示的范围。<br>
➃ OF标志位应用案例：
```txt
mov al,98  ;10进制算数运算: 数字表示 = 本身
add al,99  ;理论执行结果: 98+99=197
           ;实际执行结果: 01100010B + 01100011B = 11000101B = 0C5H = (-59)补 
           ;8bit寄存器al能够表示的有符号数范围:[-128,127], 因此add执行结果溢出
```
```txt
mov al,0F0H  ;16进制有符号数运算: 16进制数据表示 = 实际数据的补码 
add al,088H  ;理论执行结果: 0F0H+088H = 11110000B+10001000B => 10010000B+11111000B = (-16)+(-120)
             ;实际执行结果: 11110000B + 10001000B = 01111000B = 078H = (120)补 
             ;由于-136超过8bit寄存器al能够表示的有符号数范围:[-128,127], 因此add执行结果溢出
```
▶︎ __知识穿插3：已知有符号数的补码，如何求其原码？__ <br>
➀ 有符号数补码以0开头，则有符号数为正数，正数的原码=补码。<br>
➁ 有符号数补码以1开头，则有符号数为负数，负数的补码的补码=原码。

### 六：adc指令：带进位加法指令(add+carry)
➀ adc指令格式：
```txt
adc 操作对象1,操作对象2
```
➁ adc指令计算流程：
```txt
操作对象1 = 操作对象1 + 操作对象2 + CF, 设置CF标志位
```
➂ adc指令计算流程案例分析：
```txt
mov ax,2   ;无符号数: (ax)=2
mov bx,1   ;无符号数: (bx)=1
sub bx,ax  ;(bx)= 2-1 = -1 => CF=1 10进制运算涉及了借位
adc ax,1   ;(ax)+1+CF = 2+1+1 = 4
```
```txt
mov ax,1   ;无符号数: (ax)=1
add ax,ax  ;无符号数: (ax)=2 => CF=0 10进制运算不涉及进位
adc ax,3   ;(ax)+3+CF = 2+3+1=6
```
```txt
mov al,98h  ;无符号数: (al)=98h
add al,al   ;无符号数: (al)=98h+98h = 130h => CF=1 16进制运算涉及了进位
adc al,3    ;(al)+3+CF = 30h+3+1 = 34h
```
➃ adc指令可以做什么，首先结合一个普通加法案例进行分析：
<center><img src="/img/in-post/assembly_language_img/chapter_11_4.pdf" width="60%"></center>
结合上图，可以得知：adc指令可以模拟加法中：除最低位之外，其它bit位的"带进位加法计算"。另外，借助adc指令，8086CPU可以完成bit位数超过寄存器能够容纳的位数的数字的加法运算，参考➄。

➄ 借助adc指令"带进位加法计算"的功能，实现bit数很大的数据的加法计算。<br>
* 案例一：计算 1EF000H+201000H 的结果: <br>
分析：2个操作数位数超过16bit，8086CPU寄存器存储能力<16bit，无法通过下面的方式完成计算：
```txt
mov reg1,1EF000H
mov reg2,201000H
add reg1,reg2
```
解答：先将2个操作数分解：
```txt
1EF000H = 001EH(高位) | F000H(低位)  ;将数据分解
201000H = 0020H(高位) | 1000H(低位)  ;将数据分解
F000H + 1000H = 0000H, CF=1  ;低位数据执行普通加法(add) 设置CF标志位
001EH + 0020H + CF = 003FH  ;高位数据执行"带进位加法(adc) 使用"adc"完成高位数据的加法计算 
```
编写汇编程序如下：
```txt
mov ax,001EH
mov bx,0F000H
add bx,1000H
adc ax,0020H
```
* 案例二：计算 1EF0001000H+2010001EF0H 的结果：<br>
分析：2个操作数位数超过32bit，8086CPU寄存器存储能力<16bit，无法通过普通方式完成计算。由于2个操作数的位数超过32bit，考虑分成低16bit、次高16bit、高16bit分别进行处理，数据划分方式如下：<br>
```txt
1000H + 1EF0H: 低16bit相加 使用add指令 结果存放在cx中
F000H + 1000H: 次高16bit相加 使用adc指令 结果存放在bx中
001EH + 0020H: 高16bit相加 使用adc指令 结果存放在ax中
```
编写汇编程序如下：
```txt
mov ax,001EH
mov bx,F000H
mov cx,0001H
add cx,1EF0H  ;低16bit 使用add指令 
adc bx,1000H  ;次高16bit 使用adc指令 (bx)=(bx)+1000H+CF
adc ax,0020H  ;高16bit 使用adc指令 (ax)=(ax)+0020H+CF
```

### 七：sbb指令：带借位减法指令(sub+borrow)
➀ sbb指令格式：
```txt
sbb 操作对象1,操作对象2
```
➁ sbb指令计算流程：
```txt
操作对象1 = 操作对象1 - 操作对象2 - CF, 设置CF标志位
```
➂ sbb指令计算流程案例分析：<br>
计算 003E1000H-00202000H 的结果:<br>
分析：两个操作数位数>16bit，无法通过普通方式完成计算。由于2个操作数的位数位16bit，因此，考虑分成低16bit、高16bit分别进行处理，数据划分方式如下：
```txt
1000H + 2000H: 低16bit相减 使用sub指令
003EH + 0020H: 高16bit相减 使用sbb指令
```
编写汇编程序如下：
```txt
mov bx,1000H
mov ax,003EH
sub bx,2000H  ;通过sub指令 完成低16bit普通减法 设置CF
sbb ax,1000H  ;通过sbb指令 完成高16bit带进位减法
```

### 八：cmp指令：有/无符号数比较指令
➀ cmp指令格式：
```txt
cmp 操作对象1,操作对象2
```
➁ cmp指令原理：
```txt
计算<操作对象1-操作对象2>，不保存计算结果，根据计算结果对标志寄存器进行设置，例如：
cmp ax,ax  ;cmp计算(ax)-(ax)=0 但计算结果不在ax中保存 执行后：ZF=1 PF=1 SF=0 CF=0 OF=0
```
➂ cmp指令的应用场景：通过执行cmp指令，无需通过计算结果，而是间接根据寄存器的数值，就可以判断比较的结果。cmp指令配合一些根据寄存器状态跳转的指令，可以实现对程序流程逻辑的控制。

➃ 通过cmp指令执行后的相关标志位数值，判断比较结果：
* cmp ax,bx (无符号数的比较)：通过ZF、CF判断比较结果状态。
```txt
; ---- ZF ----
ZF(zero)=1 -> (ax)-(bx)=0 -> (ax)=(bx)
ZF(zero)=0 -> (ax)-(bx)≠0 -> (ax)≠(bx)
; ---- CF ----
CF(carry)=1 -> (ax)-(bx)借位 -> (ax)<(bx)
CF(carry)=0 -> (ax)-(bx)不借位 -> (ax)≥(bx)
; ---- CF & ZF ----
CF(carry)=0 && ZF(zero)=0 -> (ax)>(bx)
CF(carry)=1 || ZF(zero)=1 -> (ax)≤(bx)
```
* cmp ah,bh (有符号数的比较)：通过SF、OF判断比较结果状态。
```txt
; ---- SF & OF----
(1) SF(sign)=1 && OF(overflow)=0 -> (ah)-(bh)<0 -> (ah)<(bh)  ;没有溢出 实际结果和SF相同
(2) SF(sign)=1 && OF(overflow)=1 -> (ah)-(bh)<0 -> (ah)>(bh)  ;出现溢出 实际结果和SF相反
(3) SF(sign)=0 && OF(overflow)=1 -> (ah)-(bh)≥0 -> (ah)<(bh)  ;出现溢出 实际结果和SF相反
(4) SF(sign)=0 && OF(overflow)=0 -> (ah)-(bh)≥0 -> (ah)≥(bh)  ;没有溢出 实际结果和SF相同(*)
```

### 九：检测比较结果的条件转移指令
所有条件跳转指令都是短转移指令，通过修改IP寄存器实现转移，转移范围在[-128,127]之间。

▶︎ __常用的根据❮无符号数❯比较结果进行跳转的条件转移指令__
<center><img src="/img/in-post/assembly_language_img/chapter_11_5.pdf" width="60%"></center>

▶︎ __常用的根据❮有符号数❯比较结果进行跳转的条件转移指令__
<center><img src="/img/in-post/assembly_language_img/chapter_11_6.pdf" width="60%"></center>

<details>
    <summary><b>Intel x86 jump指令速查表 [toggle] </b></summary>
    <center><img src="/img/in-post/assembly_language_img/chapter_11_7.pdf" width="100%"></center>
</details>

### 十：DF标志位和串传送指令
▶︎ __DF标志位的基本概念__ <br>
DF(direction flag)表示方向标志位，在串处理指令中用于控制si、di的增减方向：<br>
➀ DF=0, 则每次操作后，si/di递增。<br>
➁ DF=1, 则每次操作后，si/di递减。

▶︎ __串处理指令：movsb(mov string bytes)__ <br>
指令功能参考下面的伪代码，注意其中的"mov byte ptr..."汇编指令实际中并不能这样使用，这里只是为了描述串处理指令的功能。
```txt
(i)  ((es)×16+(di)) = ((ds)×16+(si)) ---- mov byte ptr es:[di], byte ptr ds:[si]
     if(DF=0){
         (si)=(si)+1
         (di)=(di)+1
     }
(ii) if(DF=1){
        (si)=(si)-1
        (di)=(di)-1
     }
```

▶︎ __串处理指令：movsw(mov string words)__ <br>
指令功能参考下面的伪代码，注意其中的"mov word ptr..."汇编指令实际中并不能这样使用，这里只是为了描述串处理指令的功能。
```txt
(i)  ((es)×16+(di)) = ((ds)×16+(si)) ---- mov word ptr es:[di], word ptr ds:[si]
     if(DF=0){
         (si)=(si)+2
         (di)=(di)+2
     }
(ii) if(DF=1){
        (si)=(si)-2
        (di)=(di)-2
     }
```

▶︎ __串处理指令：movsb和movsw的使用__ <br>
一般而言，movsb和movsw都是串传送操作的一个步骤，通常和rep指令配合使用。
```txt
(i)  rep movsb
等价于 => s: movsb
             loop s
(ii) rep movsw
等价于 => s: movsw
             loop s
```

▶︎ __DF标志位设置方式__ <br>
DF标志位决定着串处理指令movsb、movsw的处理方向，因此x86提供了设置DF的指令：<br>
➀ cld指令：将标志寄存器的DF置0。<br>
➁ std指令：将标志寄存器的DF置1。

▶︎ __DF标志位及相关串处理指令的应用案例__ <br>
➀ 设置DF=0，实现"正向"串处理，将data段中前16byte复制到后16byte空间中去：
```txt
assume cs:code
data segment
    db 'welcome to masm'
    db 16 dup (0)
data ends
code segment
start: mov ax,data
       mov ds,ax    ;ds:[si]=data:0 -> 传送原始位置
       mov si,0     ;
       mov es,ax    ;es:[di]=data:16 -> 传送目的地址
       mov di,16    ;
       mov cx,16    ;循环16次
       cld          ;设置df=0 正向处理
       rep movsb    ;串处理...
code ends
end start
```
➁ 设置DF=1，实现"逆向"串处理，将内存空间中F000H段中最后16word复制到data段中：
```txt
assume cs:code
data segment
    db 16 dup (0)
data ends
code segment
start: mov ax,0f000h  ;设置段首地址
       mov ds,ax      ;ds:[si]=0f000h:0ffffh -> 传送原始位置
       mov si,0ffffh  ;
       mov ax,data    ;设置段首地址
       mov es,ax      ;es:[di]=data:000fh -> 传送目的地址
       mov di,15      ;0-15共16个slot !!!
       mov cx,16      ;循环16次
       std            ;设置df=1 逆向处理
       rep movsb      ;串处理...
code ends
end start
```

### 十一：pushf指令和popf指令
pushf的功能是将标志寄存器内容"压栈"，称为 __保存现场__，popf的功能是从栈中弹出数据，并送入标志寄存器中，称为 __恢复现场__。这两个操作本质上，是通过 __栈空间__ 来控制 __标志寄存器__ 的状态。

pushf/popf指令为直接访问标志寄存器提供了一种方法。在chapter12中断处理内容中，需要将标志寄存器状态保留，将再次介绍pushf/popf指令的应用。

### 十二：标志寄存器在debug程序中的表示
▶︎ __在debug程序中，标志寄存器按照每个标志位分别显示__ <br>
<center><img src="/img/in-post/assembly_language_img/chapter_11_8.pdf" width="70%"></center>

▶︎ __debug程序对于标志寄存器中不同的标志位取值的标记方式各有不同__ <br>
<center><img src="/img/in-post/assembly_language_img/chapter_11_9.pdf" width="50%"></center>



## Reference
> \<assembly language by wangshuang 3ed\> chapter11 <br>
> https://zhuanlan.zhihu.com/p/105917577 <br>
> https://www.cnblogs.com/zhangziqiu/archive/2011/03/30/computercode.html <br>
> __http://unixwiz.net/techtips/x86-jumps.html__

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
