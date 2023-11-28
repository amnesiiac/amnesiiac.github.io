---
layout: post
title: "makefile - #4"
subtitle: '[跟我一起写makefile - 陈皓]' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-09-08 15:09
lang: ch 
catalog: true 
categories: makefile
tags:
  - Time 2021
---
## 书写规则(书写依赖关系) - 第一篇
makefile的书写规则包含两个部分：依赖关系、生成目标的方法。<br>
另外makefile文件中，各个依赖关系的顺序很重要，因为makefile只有一个终极目标，其他目标都可以被看作是该目标的附属产品。因此必须让make命令解析出该makefile的终极目标，而终极目标是由依赖关系的书写顺序决定的。<br>
一般而言，定义在一个makefile中的依赖关系有很多，但是第一个依赖关系中的目标为该makefile的终极目标。如果第一个依赖关系中存在多个目标，那么目标序列中的第一个目标为最终目标。

### 一：分析一个简单的依赖规则
书写一条简单的依赖关系如下：
```txt
foo.o : foo.c defs.h
        cc -c -g foo.c
```
上述规则中，foo.o是目标文件，foo.c和defs.h是依赖文件，命令cc -c用于借助依赖关系生成目标文件。该规则指明了2点：<br>
- **1 文件依赖关系**：foo.o目标文件依赖于foo.c和defs.h两个文件。如果foo.c和defs.h文件的日期比foo.o日期要新，或者foo.o目标文件不存在，则依赖关系发生作用。
- **2 如何生成or更新目标文件foo.o**：通过cc -c命令借助依赖关系生成目标文件。

### 二：分析依赖规则语法
共有两种方式组织、书写makefile中的依赖规则。<br>
##### 1 第一种书写方式
```txt
target : prerequisites
        command
        ... 
```
##### 2 第二种书写方式
```txt
target : prerequisites; command
        command
        ... 
```
上述两种方式都可以，但需要注意的是command行一定要使用"tab"进行缩进，而不能使用"space"。<br>
另外，一般情况下，make会使用unix标准默认shell指向相关命令，即"/bin/sh"；如果需要，可以通过命令行切换其他shell执行命令。

### 三：在依赖关系书写中使用通配符(usage of wildcard)
makefile支持三种通配符："\*"(多字符任意匹配通配符)，"?"(单字符任意匹配通配符)，"[]"(匹配指定范围的字符串的通配符)。

##### 1 三种通配符基本概念介绍
通配符"\*"可以指代一系列文件，例如："\*.c"指代了所有后缀为"c"的文件。如果文件名中包含字符"\*"可以使用反斜杠加以区分。<br>
单字符通配符"?"可以指代除了特定字符外其他字符都相同的一系列文件，例如："2to?"指代了所有"2to"开头的，包含4个字符的文件。<br>
范围字符通配符"[]"可以指代特定字符为一个限定字符范围的一些列文件，例如："melo[a-z]"指代了所有以"melo"开头的结尾可为任意字符的5字符的文件。

##### 2 "~"目录特殊字符基本概念介绍
除了支持三种通配符，makefile还支持书写路径常用的"~"字符。例如："~/test"表示当前用户的$HOME目录下的test目录；"~melon/test"表示melon用户的$HOME目录下的test目录。特别地，在windows和MS-DOS操作系统上的用户没有宿主目录，那么"~"所指的目录依据HOME环境变量而定。

##### 3 应用通配符的案例 
**在clean伪目标命令中应用"任意指代通配符"**
```txt
clean:
    rm -f *.o
```
**在makefile依赖规则中使用"单字符匹配"通配符** 目标print文件依赖于所有".c"文件，"$?"表示一个自动化变量，在后续的章节中进行介绍。
```txt
print: *.c
    lpr -p $?
    touch print
```
**在变量定义过程中使用通配符** 下面语句中的任意字符串通配符并不会展开，而是$(objects)为一个名称为"\*"的".o"类型文件；即objects变量并不表示所有".o"类型文件。
```txt
objects = *.o
```
**可以使用wildcard关键字显式地要求通配符在变量中展开**
```txt
objects := $(wildcard *.o)
```

### 四：文件搜寻 
在某些大的工程中，包含大量的源文件。这些源文件按照工程需要分别放在不同的路径下面，使得makefile书写变得复杂。有两种方式可以解决这类文件路径搜寻的问题：<br>
**(i)** make命令寻找文件之间的依赖关系时，可以在文件上加上相应路径。<br>
**(ii)** 另一种方式是将一系列特定路径告知make程序，make程序将会自动在该路径下寻找依赖文件和目标文件。具体地，makefile通过特殊变量"VPATH"完成该功能。不指明该变量，则make只会在当前目录中查找；定义了该变量，make在当前目录找不到的情况下会到"VPATH"目录处寻找文件。<br>
**(iii)** 最后一种方式是使用make的"vpath"关键字。需要注意的是它不是变量，它和"VPATH"很相似，但它更加灵活。它可以为每个变量指定不同的搜索路径。"vpath"关键字的使用模式主要有三种，在后续**使用"vpath"的3种使用模式**部分进行介绍。

##### 1 使用VPATH的案例
下面的makefile使用"VPATH"变量指定了两个路径。分别为"src"和"../headers"。
```txt
VPATH = src:../headers
```
注意到，特殊变量"VPATH"中的路径通过冒号":"进行分隔。实际上，即使指定了"VPATH"特殊变量，make命令仍然会按照："当前目录"，"src"，"../headers"的顺序进行目标文件、依赖文件搜索；当前目录的搜索优先级总是最高的。

##### 2 使用vpath关键字的3种模式
**[1] 第一种模式** 为符合\<pattern\>模式的文件指定搜索目录\<directories\>
```txt
vpath <pattern> <directories>
```
**[2] 第二种模式** 清除符合\<pattern\>模式的文件的搜索目录
```txt
vpath <pattern>
```
**[3] 第三种模式** 清除所有文件搜索目录
```txt
vpath
```
**[补充] vpath相关注意事项** <br>
**(i)** vpath关键字中，\<pattern\>的匹配需要包含"%"字符："%"表示匹配0个或者若干个字符，详见下例：
```txt
vpath <pattern> <directories>
vpath %.h ../headers
```
上述vpath语句表明：在"../headers"目录下寻找所有以".h"结尾的文件(前提是：在当前目录没有找到的话)。

**(ii)** 可以连续的使用vpath关键字，以指定不同的搜索策略；如果出现相同的\<pattern\>，或者出现呈"包含关系"的两个\<pattern\>，那么make按照vpath的先后顺序进行查找。详见下例：
```txt
vpath %.c foo
vpath %   blish
vpath %.c bar
```
上述vpath语句表明了一种文件路径搜索优先顺序：首先在foo目录下搜寻所有以".c"结尾的文件；然后在blish目录下搜索所有文件(和第一个搜索路径推荐信息成包含关系)；最后在bar目录下搜索所有以".c"结尾的文件。<br>
因此，给定一个新的vpath语句如下，其符合特定pattern的文件的搜索路径优先级推荐顺序为：现在foo目录搜索所有以".c"结尾的文件，然后在bar目录搜索，最后在blish目录搜索。
```txt
vpath %.c foo:bar
vpath %.c blish
```

### 五：伪目标的几种使用方法
##### 1 以clean伪目标为例简介伪目标特性
```txt
clean :
    rm *.o temp
```
clean伪目标可以用于清除目标文件，以方便我们对于工程进行重编译。需要注意的是，伪目标实际上不是目标，它只是一个标签，我们并不能通过为伪目标构建依赖关系来决定它是否执行；只能通过"make \<伪目标名\>"来显式让伪目标生效。

伪目标尽量不要和工程文件重名，如果不能确定是否存在重名，则可以使用".PHONY"标记显式指明目标是一个伪目标。
```txt
.PHONY : clean
clean :
    rm *.o temp
```
使用".PHONY"标记之后，即使工程中存在clean文件，"make clean"也只会执行伪目标相关命令。

##### 2 带有依赖关系的伪目标(伪目标之间可以相互依赖)
伪目标一般没有依赖文件，但根据实际工程编译需要可以为伪目标添加相应依赖。伪目标同样可以写在makefile的所有目标的第一位，以作为默认目标。<br>

**[1] 什么场景下为伪目标添加依赖关系** <br>
**situation#1** 一般情况下，makefile只需要生成一个最终目标文件，即可执行文件。但有些工程可能用一个makefile同时生成多个可执行文件；或者某些场景下需要将多个makefile进行合并，但仍然需要为每个待合并的makefile生成可执行文件时，需要使用"带有依赖关系的伪目标"。详见下面样例#1。<br>
**situation#2** 可以同时构建多个带有依赖关系的伪目标，可以为它们之间添加依赖关系，以实现"按需分别调用"主程序、子程序的功能。详见下面样例#2。

**[2] 带有依赖关系的伪目标书写样例#1** <br>
```txt
all : prog1 prog2 prog3
.PHONY : all

prog1 : prog1.o utils.o
        cc -c prog1 prog1.o utils.o
prog2 : prog2.o
        cc -c prog2 prog2.o
prog3 : prog3.o sort.o utils.o
        cc -c prog3 prog3.o sort.o utils.o
```
**[3] 对样例#1进行简单分析** <br>
makefile中的第一个目标作为默认目标，上述makefile中伪目标all是默认目标。**注意：带有依赖关系的伪目标，无论伪目标的依赖项是否更新，执行"make \<伪目标名\>"，后续命令总会被执行**；而普通目标则仅当依赖文件版本比目标新时才执行命令。换一种方式进行理解：all是一个伪目标，不是具体文件实体，可以看作是该伪目标文件并不存在，因此无论何时执行"make all"都会执行依赖文件对应的命令。<br>
上述案例实现了通过"make all"命令同时生成3个可执行文件的效果。

**[4] 带有依赖关系的伪目标书写样例#2** <br>
```txt
.PHONY : cleanall cleanobj cleandiff

cleanall : cleanobj cleandiff
        rm program
cleanobj : 
        rm *.o
cleandiff :
        rm *.diff
```

**[5] 对样例#2进行简单分析** <br>
执行"make cleanall"将依次执行3个命令："rm \*.o"、"rm \*.diff"、"rm program"，即首先执行依赖项cleanobj和cleandiff相关的命令，然后执行当前命令。执行"make cleanobj"将仅执行相应的命令："rm \*.o"。执行"make cleandiff"将仅执行相应的命令："rm \*.diff"。<br>
上述3个伪目标之间的依赖关系使得调用相应命令时，就像是在执行"主程序"、"子程序"一样。


## Reference
> https://blog.csdn.net/haoel/article/details/2889 <br>

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
