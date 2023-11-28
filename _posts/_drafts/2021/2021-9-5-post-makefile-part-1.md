---
layout: post
title: "makefile - #1"
subtitle: '[跟我一起写makefile - 陈皓]'
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-09-05 15:15
lang: ch 
catalog: true 
categories: makefile
tags:
  - Time 2021
---
### 一：概述
makefile是用于工程自动化编译的工具，它关系到整个工程的编译规则。<br>
一个大的工程涉及的源文件不计其数，并且按照类型、功能、模块存在在不同的目录中。makefile定义了一系列规则来指定：哪些文件先编译、哪些文件后编译、哪些文件需要重复编译(编译顺序)，甚至执行一些更加复杂的操作(makefile有点像shell脚本一样，能够执行操作系统提供的命令)。

makefile和make是不同的概念，make本身是一个用于解释makefile中指令的命令工具。一般而言，大多数IDE提供了该命令工具，例如Delphi(make)，Visual c++(nmake)，Linux GNU(make)。

后续章节的内容将以c/c++源码工程为基础(会涉及一些c/c++编译器相关的知识)，因此必要时可以参考相关编译器文档进行理解。

### 二：makefile简单介绍
首先通过一个GNU make手册来说明makefile的书写规则。

工程文件介绍：整个工程包括8个源文件(c)，3个头文件(h)。通过编写makefile指定工程中源文件、头文件的编译规则所指定的规则如下：<br>
- 如果整个工程文件都没有编译过，那么就将所有源文件(c)进行编译并链接。
- 如果工程中部分源文件被修改，那么只编译被修改的源文件，然后链接到目标程序即可。
- 如果工程中某个头文件被修改，则需对所有引用该头文件的源文件进行编译，并链接到目标程序。

上述编译链接规则看起来有点复杂，但只要将makefile脚本写好，实际编译时只需要通过"make"命令即完成自动化编译&链接。make命令可以自动地根据整个工程中文件的修改情况来决定哪些文件需要被编译，决定它们的编译顺序，以及连接到目标文件。

##### 1 makefile的基本规则
通过如下样例makefile文件来了解makefile的构成，以及基本的规则。
```txt
target ... : prerequisites ...
    command#1
    command#2
    command#3
...
```
**[1] makefile基本组成元素及其概念** <br>
**target**表示目标文件，可以是object file(中间目标文件)，可以是可执行文件，还可以是一个label(标签)。关于"target为label"的情况，在makefile后续"伪目标"章节中进行解析。<br>
**prerequisites** 表示生成特定target文件所需要的文件or目标。<br>
**command** 表示make命令需要执行命令(可以是任意的shell命令)。

**[2] makefile基本元素之间的相互作用、联系** <br>
makefile本质上定义了工程中各个文件之间的依赖关系(target文件依赖于prerequisites文件)，使用prerequisites生成target文件的生成规则定义在command中。
换一种角度进行解释：如果prerequisites文件的版本比合成target所用的prerequisites文件要新，那么command中相关命令就会执行：使用prerequisites文件按照既定规则合成新的target文件。

##### 2 完成makefile三种规则的完整示例
**[1] makefile文件示例**
```txt
edit : main.o kbd.o command.o display.o \
       insert.o search.o files.o utils.o
       cc -o edit main.o kbd.o command.o display.o \
                  insert.o search.o files.o util.o
main.o : main.c defs.h
        cc -c main.c
kbd.o : kbd.c defs.h command.h 
        cc -c kbd.c
command.o : command.c defs.h command.h
        cc -c command.c
display.o : display.c defs.h buffer.h
        cc -c display.c
insert.o : insert.c defs.h buffer.h
        cc -c insert.c
search.o : search.c defs.h buffer.h
        cc -c search.c
files.o : files.c defs.h buffer.h command.h
        cc -c files.c
utils.o : utils.c defs.h
        cc -c utils.c
clean :
        rm edit main.o kbd.o command.o display.o \
           insert.o search.o files.o utils.o
```
将上述文件保存成"makefile"，在终端cd到该makefile目录下，并执行"make"命令，就可以生成可执行文件edit。如果想要删除可执行文件edit以及所有中间目标文件，只需要简单地执行"make clean"命令即可。

**[2] 分析上述makefile的target-prerequisites-command模型成员** <br>
target文件包括：最终生成可执行文件"edit"，以及所有的中间文件 "\*.o"。prerequisites文件包括：所有冒号后面的 "\*.c" 文件以及 "\*.h" 文件。

**[3] 分析上述makefile的文件依赖关系** <br>
makefile的第一行说明：最终可执行文件"edit"的生成依赖项为所有 "中间\*.o文件" ；而每个 "中间\*.o文件" 的依赖项为相应冒号后面的 "\*.c文件以及\*.h文件"。

**[4] 关于依赖文件后面一行命令的解释** <br>
"cc -c \*.c" 定义了生成中间目标文件的操作系统命令。需要注意的是依赖文件下一行的缩进需要使用"tab"而不是"space"。
解释：makefile自身依赖语句和shell命令之间的区别是通过判断：行首是否以"tab"开头来判断的，非"tab"开头则为makefile语句，"tab"开头则为操作系统OSshell命令。因此为了区分解释两种"语言"，系统命令行严格使用"tab"缩进。

**[5] 关于makefile文件最后的clean** <br>
clean并不是一个中间文件，因此不是最终可执行文件的依赖项；它只是一个makefile给予特定操作的名字而已(类似c语言的goto语句中的label)。<br>
clean命令后面冒号后没有内容，因此make命令解释不会查找文件相关依赖项，进而不会执行下一行的系统命令。如果需要执行后续命令，则需在make命令后指出label名字 =\> "make clean"。<br>
使用make命令的label功能("make label")可以允许定义一些备用编译流程or编译无关的命令(e.g. 程序打包、程序备份)。


## Reference
> https://blog.csdn.net/haoel/article/details/2886 <br>
> https://www.cnblogs.com/RabbitHu/p/makefile_tab.html

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内 <br>
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
