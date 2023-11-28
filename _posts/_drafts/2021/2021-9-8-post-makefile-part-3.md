---
layout: post
title: "makefile - #3"
subtitle: '[跟我一起写makefile - 陈皓]' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-09-08 08:08
lang: ch 
catalog: true 
categories: makefile
tags:
  - Time 2021
---
## makefile总述
### 一：makefile中有什么
makefile主要包含了5个东西：显式规则、隐式规则、变量定义、文件指示以及注释。
- **显式规则** 用于说明如何借助依赖文件生成1 or more目标文件：即由makefile作者明确写出makefile的文件依赖关系，以及借助依赖关系生成目标文件的命令。
- **隐式规则** make命令具备自动推导功能，允许我们书写较为简洁的makefile。
- **变量定义** makefile中可以定义变量，一般都是字符串。变量使用起来类似c中的宏定义，makefile执行时，所有的变量将会替换为定义值。
- **文件指示** 总共分为三个部分：(i) 在makefile中引用其他makefile，类似c中的include。(ii) 根据特定情况指定makefile中的有效部分，使用起来类似c中的条件编译#if。(iii) 定义一个多行命令，在后续章节中进行介绍。
- **注释** makefile中只有行注释，其符号为'#'，如果想要在makefile依赖关系、命令以及正文其他部分使用'#'符号，需要使用反斜杠进行转义：\\。

最后特别需要注意的是：makefile中的命令需要使用'tab'开头。

### 二：makefile中的文件名
##### 1 三个标准makefile文件名
默认情况下，make命令会在当前目录下依次寻找文件名为"GNUmakefile"、"makefile"、"Makefile"的文件。最好不要使用"GNUmakefile"文件名，因为该文件是供GNU的make命令识别的。大多数make命令都支持"makefile"和"Makefile"两种书写方式。推荐使用"Makefile"，因为它比全小写的"makefile"在文件目录中更加醒目。

##### 2 通过-f --file选项来指定make命令的文件名
当然，make命令支持自定义makefile文件名，如："Make.Linux"，"Make.Solaris"等。此时需要使用带选项的make命令："make -f Make.Linux" 或者 "make \-\-file Make.Solaris"。

### 三：引用其他makefile
在makefile中使用"include"关键字可以将其他makefile文件原封不动的引入当前makefile文件中"include"当前位置。
##### 1 include基本语法
```txt
include <filename>
```
filename可以是当前操作系统的shell文件模式(可以包含路径、可以使用通配符)。include前面可以存在"空字符"，但绝不能以"tab"键开头。

##### 2 稍微复杂的例子
给定当前目录下的几个makefile：a.mk b.mk c.mk 以及foo.make，以及一个定义的变量var包括e.mk f.mk。则下面的include语句：
```txt
var = e.mk f.mk
include foo.make *.mk $(var)
```
等价于 => 
```txt
include foo.make a.mk b.mk c.mk e.mk f.mk
```
##### 3 include文件搜索路径
makefile中使用了"include \<filename\>"命令后，make命令将按照如下的顺序在工程中对filename文件进行解析：
- **i** 如果filename中没有指定绝对路径or相对路径，则首先从当前目录中查找\<filename\>。
- **ii** 当前目录没有找到filename文件，则如果使用了"make -I"或者"make --include-dir"参数，那么make命令就会在I选项指定的目录下去寻找。 
- **iii** 如果目录"\<prefix\>/include"存在，一般为"/usr/local/bin/include"或者"/usr/include"，则make也会在该目录下寻找。

如果经过上述3步，仍然没有找到相应的makefile，则会生成一条"警告信息"，但不会马上生成致命错误。一旦完成整个makefile读取，make会再次尝试搜索尚未找到的filename文件，如果再次搜索失败，则生成致命错误信息。

可以使用"-"来让make命令忽略那些没有找到的filename文件，使用方式如下：
```txt
-include <filename>
```
上述命令表示，无论在include过程出现何种错误都不要报错，而是继续执行。<br>
其他版本的make可能使用"sinclude"作为"-include"命令的替代。

### 四：环境变量：MAKEFILES - 一种另类的include
可以通过设置环境变量的方式完成和"include"命令相似的功能。如果定义了工程全局环境变量MAKEFILES，那么makefile会将该环境变量中的makefile文件名数值做类似"include"操作。

MAKEFILES环境变量引入的makefile和include引入的makefile具体分存在如下几点差异：
- 使用MAKEFILES环境变量引入的makefile中，target不会起作用。
- 从MAKEFILES引入的makefile文件中发现错误，make不予理睬。

**不建议使用这个环境变量引入makefile文件**，环境变量会影响到所有它能够作用到的工程范围：一个IDE的环境变量将会影响到所有IDE下编写的文件，一个workspace的环境变量会影响到所有工作区产出的程序，一个工程的环境变量MAKEFILES会影响整个工程的编译模式。<br>
使用MAKEFILES后，所有环境变量作用范围内的makefile都会收到影响。<br>

**陈皓大大友情提示：** 有时Makefile出现奇怪的问题，可以检查当前环境下是否定义了MAKEFILES环境变量。

### 五：make命令的工作方式
##### 1 GNU make执行的具体步骤
- **1** 读入所有的makefile
- **2** 读入include命令引入的makefile =\> 得到完整的makefile文件
- **3** 初始化makefile文件中定义的变量 =\> 得到"无变量"makefile文件
- **4** 推导并分析所有隐式规则 =\> 得到依赖文件生成目标文件的命令
- **5** 为所有目标文件创建依赖关系链 =\> 命令执行准备阶段
- **6** 执行生成的命令

##### 2 makefile变量的拖延展开
1-5为第一阶段，6-7为第二阶段。第一阶段中，如果makefile定义的变量在依赖关系中or在目标文件序列中应用，make命令不会将其马上展开，而是采用一种"拖延"的方式进行展开：仅当变量应用的依赖关系or目标文件序列被make使用时，才会被展开。


## Reference
> https://blog.csdn.net/haoel/article/details/2888 <br>

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
