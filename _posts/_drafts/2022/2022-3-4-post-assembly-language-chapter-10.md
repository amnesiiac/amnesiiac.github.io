---
layout: post
title: "assembly language (CALL指令和RET指令)"
subtitle: '[assembly language wangshuang 3ed] - chapter10' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2022-03-04 11:24
lang: ch 
catalog: true 
categories: assembly
tags:
  - Time 2022
---
### 本文概述 
CALL和RET都是转移指令，它们都可以修改ip寄存器，或者同时修改cs:ip。CALL和RET经常一同使用来完成 __汇编子程序__ 的设计。本文主要介绍CALL、RET的使用原理。

### 一：ret和retf
▶︎ __ret指令__ <br>
ret指令利用 __栈__ 中的数据，修改ip寄存器中的内容，实现 __短转移__ 。

➀ 指令格式为 __ret__。<br>
➁ 指令含义：跳转到cs:(ss:sp)的位置处继续执行。<br>
➂ 实现原理：
```txt
(ip) = ((ss)×16+(sp))  ;pop ip: 将ss:sp中的内容放入ip 向栈底移动sp
(sp) = (sp)+2          ;
```
➃ 案例分析：
```txt
assume cs:code
stack segment
    db 16 dup (0)    ;16byte 0 - 用于初始化栈空间(占位)
stack ends
code segment
    mov ax,4c00h
    int 21h
start: mov ax,stack  ;设置ss:sp指向栈顶内存空间位置
       mov ss,ax     ;
       mov sp,16     ;
       mov ax,0      ;将数据0压栈 作为(ip)
       push ax       ;
       mov bx,0      ;
       ret           ;跳转到cs:(ss:sp)=cs:0=code:0处 继续执行
code ends
end start
```

▶︎ __retf指令__ <br>
retf指令利用 __栈__ 中的数据，修改cs:ip寄存器中的内容，实现 __远转移__ 。

➀ 指令格式为 __retf__。<br>
➁ 指令含义：跳转到(ss2:sp2):(ss1:sp1)的位置处继续执行。<br>
➂ 实现原理：
```txt
(ip) = ((ss)×16+(sp))  ;pop ip: 将ss:sp中的内容放入ip 向栈底移动sp
(sp) = (sp)+2          ;
(cs) = ((ss)×16+(sp))  ;pop cs: 将ss:sp中的内容放入ip 向栈底移动sp
(sp) = (sp)+2          ;
```
➃ 案例分析：
```txt
assume cs:code
stack segment
    db 16 dup (0)    ;16byte 0 - 用于初始化栈空间(占位)
stack ends
code segment
    mov ax,4c00h
    int 21h
start: mov ax,stack  ;设置ss:sp指向栈顶内存空间位置
       mov ss,ax     ;
       mov sp,16     ;
       mov ax,0      ;"手动"执行call操作
       push cs       ;先将当前代码段 段地址压栈 设置cs
       push ax       ;再将0压栈 设置ip
       mov bx,0      ;
       ret           ;跳转到(ss2:sp2):(ss1:sp1)=cs:0=code:0处 继续执行
code ends
end start
```

### 二：call指令
CPU执行call指令时，进行2步操作：<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
➀ 保存现场：读取call指令送入指令缓冲器，更新ip指向下一条待执行的指令，将更新后的ip或cs:ip压栈。<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
➁ 跳转：执行call指令：跳转到❮目的地址❯。<br>
注意：call指令不能实现短转移，另外，call指令实现转移的方式同chapter9中介绍的jmp指令相同。本文后续部分，将按照跳转的目的地址的来源不同为主线，介绍call的不同形式。

### 三：call指令：根据"相对位置"进行跳转
➀ 指令格式为 __call 标号__。<br>
➁ 指令含义：保存现场，根据相对距离，跳转到标号处继续执行。<br>
➂ 实现原理：
```txt
(i)  (sp)=(sp)-2; ((ss)×16+(sp))=(ip); ⟺ push ip;
(ii) (ip)=(ip)+16bit_offset;           ⟺ jmp near ptr 标号;
```
➃ 跳转的位移大小计算方式：
```txt
16bit_offset = 标号处地址 - call指令后的第一个字节的地址
```
➄ 跳转的位移范围为-32768~32767，用补码表示。<br>
➅ 跳转的16bit_offset由编译call指令时计算得出。<br>
➆ __call 标号__ 指令逻辑上等价于：
```txt
push ip
jmp near ptr 标号
```

### 四：call指令：跳转目的地址在标号的cs:ip中
➀ 指令格式为 __call far ptr 标号__。<br>
➁ 指令含义：保存现场，根据cs:ip数值，跳转到标号处继续执行。<br>
➂ 实现原理：
```txt
(i)  (sp)=(sp)-2; ((ss)×16+(sp))=(cs); ⟺ push cs;  保存现场1
     (sp)=(sp)-2; ((ss)×16+(sp))=(ip); ⟺ push ip;  保存现场2
(ii) (cs)=标号所在段的段地址; (ip)=标号在段中的偏移地址; ⟺ jmp far ptr 标号;  跳转
```
➃ __call far ptr 标号__ 指令逻辑上等价于：
```txt
push cs           ;保存现场1
push ip           ;保存现场2
jmp far ptr 标号  ;根据标号设置cs:ip 并跳转到cs:ip处继续执行
```

### 五：call指令：跳转目的地址在16bit-reg中
➀ 指令格式为 __call 16bit-reg__。<br>
➁ 指令含义：保存现场，根据16bit寄存器数值设置(ip)，跳转到cs:ip处继续执行。<br>
➂ 实现原理：
```txt
(i)  (sp)=(sp)-2; ((ss)×16+(sp))=(ip); ⟺ push ip;  保存现场
(ii) jmp 16bit-reg;  跳转
```
➃ __call 16bit-reg__ 指令逻辑上等价于：
```txt
push ip           ;保存现场
jmp 16bit-reg     ;跳转到cs:ip
```

### 六：call指令：跳转目的地址在[内存单元]中
▶︎ __call word ptr [内存单元]__ <br>
➀ 指令含义：保存现场，根据[...]数值设置(ip)，跳转到cs:ip处继续执行。<br>
➁ 实现原理：
```txt
(i)  (sp)=(sp)-2; ((ss)×16+(sp))=(ip); ⟺ push ip;  保存现场
(ii) jmp word ptr [...];  跳转
```
➂ __call 16bit-reg__ 指令逻辑上等价于：
```txt
push ip             ;保存现场
jmp word ptr [...]  ;跳转到cs:ip=cs:[...]处执行
```
➃ 结合下面的案例进行理解：
```txt
mov sp,10h            ;设置sp 
mov ax,0123h          ;借助ax设置ds:[0]的数值
mov ds:[0],ax
call word ptr ds:[0]  ;保存当前(ip): push ip(sp=sp-2); (ip)=ds:[0] 跳转到cs:ip处
                      ;(ip)=0123h (sp)=0eh
```

▶︎ __call dword ptr [...]__ <br>
➀ 指令含义：保存现场，根据[...]数值先后设置(cs)和(ip)，跳转到cs:ip处继续执行。<br>
➁ 实现原理：
```txt
(i)   (sp)=(sp)-2; ((ss)×16+(sp))=(cs); ⟺ push cs;  保存现场1
(ii)  (sp)=(sp)-2; ((ss)×16+(sp))=(ip); ⟺ push ip;  保存现场2
(iii) jmp dword ptr [...];  跳转
```
➂ __call dword ptr [...]__ 指令逻辑上等价于：
```txt
push cs              ;保存现场1
push ip              ;保存现场2
jmp dword ptr [...]  ;跳转到cs:ip
```
➃ 结合下面的案例进行理解：
```txt
mov sp,10h             ;设置sp 
mov ax,0123h           ;借助ax设置ds:[0]的数值 --> ip
mov ds:[0],ax
mov word ptr ds:[2],0  ;直接设置ds:[2]的数值 --> cs
call dword ptr ds:[0]  ;保存当前(cs): push cs(sp=sp-2); (cs)=ds:[2]
                       ;保存当前(ip): push ip(sp=sp-2); (ip)=ds:[0]
                       ;(cs)=0000h (ip)=0123h (sp)=0ch
```

### 七：call和ret配合使用 (构建汇编子程序)
通过如下几个案例来对：call和ret指令配合使用方式进行理解。

▶︎ __call 标号/ret 配合使用__ <br>
```txt
assume cs:code
code segment
    start: mov ax,1      ;
           mov cx,3      ;设置循环次数
           call s        ;update ip->"mov bx,ax" push ip; jmp near ptr s;
           mov bx,ax     ;call后第一个字节地址
           mov ax,4c00h  ;
           int 21h       ;
        s: add ax,ax     ;标号s
           loop s        ;
           ret           ;返回调用 pop ip 此时cs:ip->"mov bx,ax"
code ends
end start
```
▶︎ __结合栈、内存来理解 call 标号/ret__ <br>
```txt
assume cs:code 
stack segment
    db 8 dup (0)    ---- 1000:0000 00 00 00 00 00 00 00 00
    db 8 dup (0)    ---- 1000:0008 00 00 00 00 00 00 00 00
stack ends
code segment
    start: mov ax,stack ---- 1001:0000  B8 00 10  ;手动设置ss:sp指向stack段尾部
           mov ss,ax    ---- 1001:0003  8E D0     ;set ss=stack
           mov sp,16    ---- 1001:0005  BC 10 10  ;set sp=16
           mov ax,1000  ---- 1001:0008  8E E8 03
           call s       ---- 1001:000B  E8 05 00  ;ip=000EH; push ip; ip=0013H jmp near ptr s;
           mov ax,4c00h ---- 1001:000E  B8 00 4C  ;
           int 21h      ---- 1001:0011  CD 21     ;
        s: add ax,ax    ---- 1001:0013  03 C0     ;
           ret          ---- 1001:0015  C3        ;pop ip; ip=000EH 返回cs:ip处执行
code ends
end start
```
▶︎ __总结：具有子程序的汇编源代码框架__ <br>
```txt
assume cs:code
code segment
main: ...
      call function1  ;调用子程序function1
      ...
      mov ax,4c00h
      int 21h
function1: ...
           call function2  ;调用子程序function2
           ...
           ret             ;子程序返回
function2: ...
           ret             ;子程序返回
code ends
end main 
```

### 八：mul指令
▶︎ __mul指令使用的注意事项__ <br>
➀ 两个相同位数的数据才能相乘：8bit数据只能和8bit数据相乘，16bit数据只能和16bit数据相乘，8bit和16bit之间不能相乘。<br>

▶︎ __mul指令的基本用法__ <br>
➀ 指令格式为 __mul reg__ 或者 __mul [内存单元]__ 。<br>
➁ 指令计算方式：<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
➊ 如果执行8bit乘法，则一个乘数默认放在al中，另一个乘数在8bit寄存器或byte ptr [...]中；<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
➋ 如果执行16bit乘法，则一个乘数默认放在ax中，另一个乘数在16bit寄存器或word ptr [...]中。<br>
➂ 指令计算结果存放位置：<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
➊ 如果执行8bit乘法，则结果存放在ax寄存器中；<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
➋ 如果执行16bit乘法，则结果高16bit存放在dx中，低16bit存放在ax中。<br>
➃ __mul [...]__ 的内存单元可以用不同寻址方式给出：
```txt
mul byte ptr ds:[0]     ;(ax) = (al) × ((ds)×16+0)
mul word ptr [bx+si+8]  ;(ax) = low16bit [(ax) × ((ds)×16+(bx)+(si)+8)]
                        ;(dx) = high16bit [(ax) × ((ds)×16+(bx)+(si)+8)]
```
➃ 案例分析：<br>
```txt
;(i) 计算100×10的结果
mov al,100  ;100<255=8bit
mov bl,10   ;10<255=8bit
mul bl      ;(ax)=(al)×(bl) = 1000 = 03E8H
```
```txt
;(ii) 计算100×10000的结果
mov ax,100     ;100<255=8bit
mov bx,10000   ;10000>255=16bit
mul bx         ;(ax)=low16bit[(ax)×(bx)] = 100 0000 = F4240H = 4240H
               ;(dx)=high16bit[(ax)×(bx)] = 100 0000 = F4240H = 000FH
```

### 九：借助call/ret实现模块化程序设计
对于较复杂的问题，可以首先将问题简化成相互联系的不同层次的子问题。
借助call/ret指令，可以将解决问题的过程封装成函数：通过call指令调用该函数，再通过ret指令进行函数返回。

本文后续部分将介绍程序设计中的问题及其解决方法。

### 十：汇编子程序单个参数/返回值的传递
将问题简化成子问题后，子程序根据参数完成一定的事务，将结果提供给调用者。结合下面案例分析。

▶︎ __案例分析：编写子程序实现根据提供的参数N，计算N^3__ <br>
➀ 将参数存在什么位置？可以将参数N存储在bx中，供子程序调用。<br>
➁ 计算得到的数值，存储在什么位置？可以将计算结果存储在dx和ax中，供外部程序调用。
```txt
cube: mov ax,bx  ;将参数N存入bx中 再将bx传入ax(mul指令乘数默认放在ax)
      mul bx     ;(ax) = low16bit[(ax)×(bx)]; (dx) = high16bit[(ax)×(bx)]
      mul bx     ;(ax) = low16bit[(ax)×(bx)]; 当上一次mul指令计算结果<16bit 则(dx)=0
      ret
```

▶︎ __案例分析：计算data段中的一组word数据的3次方，结果保存在后面dword单元中__ <br>
```txt
assume cs:code
data segment
    dw 1,2,3,4,5,6,7,8
    dd 0,0,0,0,0,0,0,0  
data ends
code segment
start: mov ax,data    ;借助ax设置ds=data
       mov ds,ax      ;
       mov si,0       ;dw取数据位置
       mov di,16      ;dd存数据位置
       mov cx,8       ;设置循环8次
    s: mov bx,[si]    ;标号s 将ds:[si]处的dw数据存入bx 作为cube子程序的参数
       call cube      ;调用cube子程序 计算dw第一组数据的三次方
       mov [di],ax    ;将low16bit存入ds:[di]
       mov [di].2,dx  ;将high16bit存入ds:[di+2]
       add si,2       ;更新dw下一个读数据位置
       add di,4       ;更新dd下一个存数据位置
       loop s         ;返回s
       mov ax,4c00h   ;
       int 21h        ;
 cube: mov ax,bx      ;cube子程序 计算给定参数(bx)的3次方 通过(ax)返回
       mul bx
       mul bx
       ret
code ends
end start
```

### 十一：汇编子程序中多个参数/返回值的传递
▶︎ __问题提出__ <br>
上一小节中计算data的3次方程序中，cube子程序只接受1个参数(bx)，如果cube需要2个参数，则可以通过两个寄存器来存放；如果子程序需要传递更多参数时，而寄存器资源有限，则应该如何处理？

▶︎ __解决方案__ <br>
可以将待传递的批量参数数据存放在内存空间中，然后将批量参数数据占用内存空间的❮首地址❯赋值给寄存器，并借助寄存器传递给子程序。同理，对于批量结果数据的返回，可以使用同样方式进行。

▶︎ __案例分析：编写子程序将一个全是字母的字符串转化为大写__ <br>
```txt
capital: and byte ptr [si],1101111b  ;将ds:si内存单元中的字母转化成大写
         inc si                      ;处理下一字母
         loop capital                ;循环子程序(cx)次
         ret                         ;返回至调用处
```
▶︎ __案例分析：将data段中的字符全部转化成大写__ <br>
```txt
assume cs:code
data segment
    db 'conversation'    ;"批量"的参数在内存空间中依次存放
data ends
code segment
    start:   mov ax,data   ;ds=data
             mov ds,ax
             mov si,0      ;设置偏移初值 借助ds:[si]的方式逐个"传递"批量参数
             mov cx,12     ;设置循环次数 隐含了子程序所需的"字符串长度"参数
             call capital  ;调用子程序
             mov ax,4c00h
             int 21h
    capital: and byte ptr [si],1101111b  ;通过访问ds:[si]来使用参数
             inc si
             loop capital
             ret
code ends
end start
```

### 十二：寄存器冲突的问题
▶︎ __案例分析：将一个由字母组成，以0结尾的字符串转换成大写__ <br>
字符串一般以0结尾标识字符串的结束，可以逐个取字符进行检测：如果是0，则进行大写的转化，否则结束处理。本案例同上一小节中的案例不同之处在于：本案例通过0字符表示了字符串的结束，而上一小节中的案例通过传递字符串长度来实现。

▶︎ __汇编程序(1)：将data段中的字符串转换成大写(存在寄存器冲突问题)__ <br>
```txt
assume cs:code
data segment
    db 'word',0
    db 'unix',0
    db 'wind',0
    db 'good',0
data ends
code segment
start:   mov ax,data                  ;ds=data
         mov ds,ax                    ;
         mov bx,0                     ;(bx)=0 初始化内存单元偏移
         mov cx,4                     ;(cx)=4 循环4次 每次处理一行数据
    s:   mov si,bx                    ;初始化内存单元偏移值si
         call capital                 ;对每行字符串 调用转化大写子程序
         add bx,5                     ;更新偏移值
         loop s                       ;循环
         mov ax,4c00h
         int 21h
capital: mov cl,[si]                  ;从内存空间中取出待处理字符 赋值给cx 
         mov ch,0                     ;===== 同外部循环程序使用的cx寄存器冲突 =====
         jcxz ok                      ;(cx)=0 判断当前字符是否为0 是则调用ok
         and byte ptr [si],11011111b  ;大小写转换
         inc si                       ;更新内存空间偏移 取下一个字符
         jmp short capital            ;
     ok: ret                          ;子程序返回
code ends
```
▶︎ __如何避免主程序、子程序寄存器冲突__ <br>
➤ 首先能够想到2个简单的方案：<br>
➀ 在编写调用子程序的程序时，需要注意不要使用子程序"占用"的寄存器。<br>
➁ 在编写子程序时，不要使用产生冲突的寄存器。

➤ 分析上述2个简单的方案的可行性：<br>
➀ 不使用和子程序产生冲突的寄存器，将给调用子程序的编写造成很大麻烦，必须要小心检查子程序可能产生冲突的所有寄存器。<br>
➁ 在编写子程序时，很难预知子程序将来被调用的情况，因此也很难实现。

➤ 寄存器冲突问题的解决思路导向：<br>
➀ 编写子程序调用程序时，不必关心子程序到底使用了哪些寄存器。<br>
➁ 编写子程序时不必关心调用者可能使用的寄存器。<br>
➂ 不产生寄存器冲突。

➤ 寄存器冲突问题的解决：<br>
在子程序开始前，将子程序需要用到的所有寄存器中的内容保存起来，并在子程序调用返回前恢复。一般可以用 __栈__ 来保存寄存器中的内容。

编写子程序的标准框架：
```txt
子程序开始：子程序中<使用的寄存器>入栈
           子程序内容
           子程序中<使用的寄存器>出栈
           返回(ret/retf)
```

▶︎ __汇编程序(2)：将data段中的字符串转换成大写(解决了寄存器冲突问题)__ <br>
```txt
assume cs:code
data segment
    db 'word',0
    db 'unix',0
    db 'wind',0
    db 'good',0
data ends
code segment
start:   mov ax,data                  ;ds=data
         mov ds,ax                    ;
         mov bx,0                     ;(bx)=0 初始化内存单元偏移
         mov cx,4                     ;(cx)=4 循环4次 每次处理一行数据
    s:   mov si,bx                    ;初始化内存单元偏移值si
         call capital                 ;对每行字符串 调用转化大写子程序
         add bx,5                     ;更新偏移值
         loop s                       ;循环
         mov ax,4c00h
         int 21h
capital: push cx                      ;子程序执行前 将需要使用的寄存器入栈 - 保存现场
         push si
 change: mov cl,[si]                  ;从内存空间中取出待处理字符 赋值给cx 
         mov ch,0                     ;===== cx可以放心使用 外部程序cx状态已经入栈 =====
         jcxz ok                      ;(cx)=0 判断当前字符是否为0 是则调用ok
         and byte ptr [si],11011111b  ;大小写转换
         inc si                       ;更新内存空间偏移 取下一个字符
         jmp short change;
     ok: pop si                       ;子程序保存的寄存器出栈 - 顺序和入栈相反 - 恢复现场
         pop cx
         ret                          ;子程序返回
code ends
```


## Reference
> \<assembly language by wangshuang 3ed\> chapter10 <br>

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
