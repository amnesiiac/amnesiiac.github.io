---
layout: post
title: "assembly language (第一个汇编程序)"
subtitle: '[assembly language wangshuang 3ed] - chapter4' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2022-01-20 14:19
lang: ch 
catalog: true 
categories: assembly
tags:
  - Time 2022
---
### 本文简介
本章内容将主要介绍关于汇编程序从书写到执行的整个过程中的一些细节知识。


### 一：一个源程序从书写到执行之间的过程
<center><img src="/img/in-post/assembly_language_img/chapter_4_1.pdf" width="80%"></center>
- ➀ 关于汇编源程序的编译链接：使用汇编程序编译器对于汇编语言编写的源程序进行编译，生成目标文件。再使用链接程序对于生成的目标文件进行链接，生成可在操作系统中执行的可执行文件。
- ➁ 经过编译链接生成的可执行文件可以分成2个部分：程序&数据、相关描述信息。程序为从汇编代码源程序中翻译得到的机器码，数据为源程序中定义的数据部分；相关描述信息如程序的大小，需要占用多少内存空间...
- ➂ 在操作系统中，执行可执行文件中的程序：OS依照可执行文件中的描述信息，将可执行文件中的机器码、数据加载到存储单元中，并进行相关初始化操作(设置CS:IP...)，然后由CPU执行程序。


### 二：源程序举例&分析
➤ 待分析汇编程序样例
```txt
assume cs:codesg  ;伪指令"assume"将codesg的程序段和CS段寄存器相互关联

codesg segment ;伪指令"xxx segment"表示段开始

    mov ax,0123H
    mov bx,0456H
    add ax,bx
    add ax,bx

    mov ax,4c00H
    int 21H

codesg ends  ;伪指令"xxx ends"表示段结束

end  ;伪指令"end"表示汇编程序结束
```

➤ 汇编程序中的伪指令 <br>
在汇编语言书写的程序中，包含两种指令，一种是汇编指令，一种是伪指令。汇编指令是有对应机器码的指令，可以被编译器编译为机器指令，最终被CPU执行；伪指令没有与之对应的机器码，最终不会被CPU执行，而是在编译过程中由编译器负责执行。<br>
因此易知：汇编指令是给CPU写的，用于控制CPU的执行过程，而伪指令是给编译器写的，用于控制汇编程序的编译过程。

➤ 结合上面的汇编源程序进行分析，程序中出现了3种伪指令：<br>
- ➀ "xxx segment ... xxx ends" 这对伪指令成对出现，是书写汇编程序中必须要用到的伪指令。"segment"和"ends"关键字共同定义了一个段，"xxx"为所定义的段的名称。<br>
另外，一个汇编程序是由多个段组成的， 这些段可用来存放代码、数据、当作栈空间来使用；因此一个源程序中所有将被计算机处理的信息：指令、数据、栈都应该被划分到不同的段中。
- ➁ "end" 这个伪指令为一个汇编程序结束标记，编译器在编译汇编程序时，如果遇到"end"则结束对程序的编译。在汇编程序书写结束时，一定要在结尾加上"end"，否则编译器在执行程序时，无法获悉程序在何处结束。<br>
需要注意的是：不要弄混"end"和"xxx ends"伪指令，它们的作用完全不同。
- ➂ "assume" 该伪指令含义为"假设"，它假设某寄存器和程序中的某个使用"xxx segment ... xxx ends"所定义的一个段相关联。在实际汇编编程过程中，使用"assume"将具有特定用途的段和相关的寄存器相互关联起来即可。

➤ 汇编源程序文件经过计算机编译、链接处理过程的简化图示 <br>
<center><img src="/img/in-post/assembly_language_img/chapter_4_2.pdf" width="60%"></center>
- [标号] 汇编源程序文件中的标号实际指代了一个地址。"codesg segment"中，"codesg"作为一个段的名称，会被编译链接器处理成标识该段的"段地址"。

➤ 理解汇编程序的结构 - 以编写计算2^3结果的汇编程序为例 <br>
- ➀ 首先定义一个标号为"abc"的程序段：
```txt
abc segment
    ...
abc ends
```
- ➁ 在上述定义好的程序段来实现计算任务：
```txt
abc segment 
        mov ax,2
        add ax,ax
        add ax,ax
abc ends
```
- ➂ 然后，在程序段的后面指出程序何时结束：
```txt
abc segment
        mov ax,2
        add ax,ax
        add ax,ax
abc ends
end
```
- ➃ 最后由于书写的"abc"作为代码段来使用，因此应当将段"abc"和cs寄存器关联起来：
```txt
assume cs:abc
abc segment
        mov ax,2
        add ax,ax
        add ax,ax
abc ends
end
```
最终，经过构建上述汇编程序的结构，就构成了一个完整的汇编源程序文件内容。

➤ 汇编程序在DOS操作系统中的调用执行 <br>
编写的汇编程序以汇编指令存储在源程序中，经过编译、链接转变成机器码，存储在可执行文件中；可执行文件是如何被执行的呢？<br>
假定当前程序program2已经被编译器处理成可执行文件，要想运行program2，则需要有一个正在运行的程序program1，它负责将program2加载进内存，将CPU的控制权交给program2；至此program1才能被执行，当program2开始运行后program1停止运行。

➤ 汇编程序的返回 - 当汇编程序执行结束后，将CPU控制权交还给调用&执行它的程序，这个过程被称为：程序返回。<br>
汇编程序使用如下的通用代码用于程序的返回：
```txt
mov ax,4c00H  ;设置寄存器AH=4CH
int 21H  ;调用了系统中断程序
```
上述汇编代码的作用是：调用操作系统中的4CH号中断程序，该中断表示程序安全退出。另外，上述程序中断退出程序等价于：
```txt
mov ah,4cH
int 21H
```

➤ 汇编程序中的逻辑错误、语法错误 <br>
一般而言，程序在编译时期被编译器发现的错误是语法错误。例如下面的汇编源程序在编译时产生语法错误：
```txt
aume cs:abc        ;aume无法被编译器识别 - 应为assume
abc segment
        mov ax,2
        add ax,ax
        add ax,ax
end                ;缺少必要的段结束 - abc ends
```
当源程序被编译完成后，在运行时发生的错误为逻辑错误。语法错误较容易发现，但逻辑错误不容易提前发现。例如，本文前面计算2^3的程序存在一处明显的逻辑错误，现将其修正为：
```txt
assume cs:abc
abc segment
        mov ax,2
        add ax,ax
        add ax,ax
        mov ax,4c00H  ; 增加了汇编程序返回的逻辑部分 源程序文件不包含这2行 不会发生编译错误 但是会引发逻辑错误
        int 21H
abc ends
end
```


### 三：编辑汇编源程序
使用编辑器编写汇编源程序，并将源程序保存为"xxx.asm"格式即可。


### 四：汇编程序的编译
采用微软masm5.0汇编编译器(masm.exe)编译汇编源程序"xxx.asm"，基本步骤大体如下：
- ➀ 设置生成的目标文件名"xxx.obj"。
- ➁ 选择是否保留中间列表文件"xxx.lst"。
- ➂ 选择是否保留交叉文件"xxx.crf"。
- ➃ 最后编译程序结束。

执行上述编译步骤后，将得到一个"xxx.obj"文件，它是对汇编源文件"xxx.asm"的编译结果。<br>
对于编译过程中产生的"xxx.lst"和"xxx.crf"，它们分别是中间列表文件、中间交叉文件，都属于编译过程的中间结果，在编译时可以手动指定忽略它们的生成。

当然，如果编译过程中出现如下2种错误，将不能得到期望的目标文件：
- ➀ 编译汇编源程序时出现"severe error"。
- ➁ 编译器不能找到指定的汇编源程序文件。

### 五：汇编程序的链接
汇编源程序文件经过编译得到目标文件"xxx.obj"后，需要对得到的目标文件进行链接，从而得到可执行文件"xxx.exe"。

使用微软Overlay Linker3.60链接器"link.exe"对于之前得到的目标文件进行链接，步骤大体如下：
- ➀ 运行"link.exe"程序。
- ➁ 找到需要进行链接的目标文件，执行链接操作。
- ➂ 链接器提示是否生成链接中间镜像文件"xxx.map"。
- ➃ 忽略中间镜像文件的生成后，链接程序提示用户输入链接需要的库文件名"xxx.lib"，如果程序中调用了某库文件中的程序，则链接时将它们链接在一起，否则可以直接忽略该提示。
- ➄ 最后得到可执行程序"xxx.exe"。

当然，如果链接过程中出现错误，将不能得到期望的可执行程序文件。

下面简单介绍汇编程序链接的作用：
- ➀ 当汇编源程序很大时，由于有链接这一步骤，因而可以将汇编源程序文件分成多个源程序文件来进行编译，在每个程序独立编译成目标文件后，再通过链接器将它们链接在一起，生成可执行文件。
- ➁ 当汇编源程序调用了某个库中的程序片段，则此时需要将这个库文件和汇编源程序编译生成的目标文件链接到一起，生成一个可执行文件。
- ➂ 在汇编源程序经过编译后，得到机器码格式的目标文件，目标文件中有部分内容不能直接用来生成可执行文件，必须依赖链接器将它们处理成可执行文件。因此，即使在存在一个汇编源程序，且源程序文件中没有引用其他库中的子程序，也必须使用链接器对目标文件进行处理，生成可执行文件。


### 六：以简化方式完成编译&链接 - omitted


### 七：可执行文件的执行 - omitted


### 八：谁负责将可执行文件加载到内存并使其运行?
在DOS系统中，可执行文件"xxx.exe"想要运行，必须由一个正在运行的程序"zzz.exe"将其加载到内存中，将CPU的控制权交给"xxx.exe"，它才得以运行；当"xxx.exe"程序执行完毕后，将CPU的控制权交还给调用它执行的程序"zzz.exe"。

即，在DOS系统中，正在运行的程序是"command.exe"，当"xxx.exe"想要执行时：
- ➀ 首先找到"xxx.exe"可执行程序。
- ➁ 然后将CS:IP段寄存器指向程序的入口。
- ➂ 然后"command.exe"程序暂停运行，将CPU的控制权交给CS:IP指向的可执行程序。
- ➃ "xxx.exe"程序执行完成后，将CPU控制权交还给"command.exe"程序。


### 九：可执行文件执行过程跟踪
➤ 为什么需要对可执行文件进行跟踪？<br>
对于书写的程序中简单的错误，可以通过人工仔细检查源程序的方式来实现；对于程序中隐藏较深的错误，则需要对于程序进行逐步分析跟踪才能实现。

➤ 仅通过程序执行过程不能实现对程序运行的逐步跟踪？<br>
回想本文上一小节中，在DOS系统下运行"xxx.exe"程序时，由"command.exe"将可执行程序加载到内存中，设置CS:IP指向程序入口，两个操作是连续完成的；一旦CS:IP被设置完成，"command.exe"立即放弃CPU的控制权，CPU则立即开始运行程序，直至程序结束。

➤ 如何实现可执行文件的逐步运行分析？<br>
可以使用debug程序完成。debug程序将程序加载到内存，设置CS:IP指向程序执行入口，但debug程序不放弃对CPU的控制，从而借助debug命令相关命令来单步执行程序。

➤ DOS系统下可执行文件在内存中的加载过程 <br>
<center><img src="/img/in-post/assembly_language_img/chapter_4_3.pdf" width="100%"></center>


## Reference
> \<assembly language by wangshuang 3ed\> chapter4 <br>

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
