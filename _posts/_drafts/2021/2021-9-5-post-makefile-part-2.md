---
layout: post
title: "makefile - #2"
subtitle: '[跟我一起写makefile - 陈皓]' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-09-05 21:36
lang: ch 
catalog: true 
categories: makefile
tags:
  - Time 2021
---
### 三：make的工作方式
在默认模式下，只输入make命令，则make执行如下的操作：
- make会在当前目录下找名字为"MAKEFILE"或者名字为"makefile"的文件
- 如果找到该文件，则寻找第一个目标文件(target=edit)，并把它作为最终目标文件。
- 如果在工程文件夹中找不到edit文件或者edit的依赖文件比生成当前edit文件要新，则执行edit依赖关系下一行的命令以更新edit文件。
- 如果生成edit文件所依赖的文件 "\*.o" 在工程目录中全部存在，那么make命令会在工程目录中寻找 "\*.o" 文件的依赖文件，如果找到则使用相应命令生成该 "\*.o" 文件。
- 最后如果最终目标文件所有依赖文件都正常生成or依赖文件已经存在，那么make命令执行生成最终目标文件。

这就是make命令根据makefile的依赖性规则生成最终目标文件的流程，从查询依赖到生成依赖有点类似LIFO的栈展开。

**make命令只关心依赖关系**： make命令只关心工程文件中依赖项是否存在or能够通过执行命令编译生成，而不关心具体命令生成依赖文件时的错误(如某个文件编译生成过程存在编译警告、错误)。make命令只关心依赖关系，不关心依赖项是如何被生成的。

**关于上述案例makefile文件中的clean**：clean没有被第一个目标文件(edit)直接or间接的关联，因此clean后面的命令不会自动执行。可以使用"make clean"显式执行该命令(该命令清除所有目标文件以准备重新编译)。

**整个工程编译完成后再修改某一文件**：比如在编译完成生成了edit最终目标文件后，修改了files.c，那么根据依赖性files.o文件会被重新编译。此时files.o文件相比生成edit文件的files.o要新，因此edit最终目标文件会被重新链接。如果修改了command.h，那么kbd.o、command.o、files.o都会被重新编译。edit被重新链接。

### 四：makefile中使用变量 (使makefile易于维护)
回顾前文的makefile文件，将不相关的部分省略掉，剩余部分如下所示。
```txt
edit : main.o kbd.o command.o display.o \
       insert.o search.o files.o utils.o
       cc -o edit main.o kbd.o command.o display.o \
                  insert.o search.o files.o util.o

... 

clean :
        rm edit main.o kbd.o command.o display.o \
           insert.o search.o files.o utils.o
```
很容易发现，edit最终目标文件的依赖文件序列("\*.o")，依赖项下一行的操作系统命令包含同样的文件序列，文件底部clean命令下面也有同样的清除文件序列。

当工程文件依赖结构变动(e.g. 添加一个新的文件)，那么至少需要在上述3个地方对于文件序列进行维护。当工程较小，makefile文件体系结构不复杂时，手动在3个地方分别进行修改很容易；当工程依赖层次结构复杂，每次改动工程文件结构，使用手动改写很容易漏该某处需要修改的部分，而这将导致目标文件编译失败。

通过变量的方式很容易解决上述问题(makefile中的变量和c/c++中的宏类似)：定义一个变量，使其存放上述3个重复的依赖文件序列。重新修改上述makefile文件如下：
```txt
objects = main.o kbd.o command.o display.o \
          insert.o search.o files.o utils.o

edit : $(objects) 
       cc -o edit $(objects) 

... 

clean :
        rm edit $(objects) 
```
使用上述makefile文件，如果需要添加新的依赖文件到文件序列，只需修改变量"object"即可。

### 五：让makefile自动推导生成目标文件的命令
GNU的make工具可以通过名称自动识别目标文件和依赖文件的关系，并自动生成相关命令。make看到一个 "anyname.o" 文件，它就会自动将 "anyname.c" 加入到生成依赖中，并生成"cc -c anyname.c"。有了make命令自动推导机制，可以将上述makefile进行简化成下面的形式：
```txt
objects = main.o kbd.o command.o display.o \
          insert.o search.o files.o utils.o

edit : $(objects) 
       cc -o edit $(objects) 

main.o : main.c defs.h
kbd.o : kbd.c defs.h command.h 
command.o : command.c defs.h command.h
display.o : display.c defs.h buffer.h
insert.o : insert.c defs.h buffer.h
search.o : search.c defs.h buffer.h
files.o : files.c defs.h buffer.h command.h
utils.o : utils.c defs.h

.PHONY : clean
clean :
        rm edit $(objects)  
```
注意：上述makefile的".PHONY : clean"表示clean是一个伪目标文件，即clean不是真正的目标文件，也不够成任何目标文件的依赖，它只用于作为make命令的可选选项进行执行。 

### 六：另类风格的makefile (根据头文件进行分类归纳)
很容易发现，上述makefile中很多的头文件都被目标文件(最终、中间)所共享，因此可以将原始根据目标文件进行分类归纳的makefile整理成按照头文件进行分类归纳的形式。得到的结果如下所示：
```txt
objects = main.o kbd.o command.o display.o \
          insert.o search.o files.o utils.o

edit : $(objects) 
       cc -o edit $(objects) 

$(objects) : defs.h
kbd.o command.o files.o : command.h
display.o insert.o search.o files.o : buffer.h

.PHONY : clean
clean :
        rm edit $(objects)  
```
这种按照工程项目中头文件进行归纳的makefile形式，虽然型制上比较简单，但是文件依赖关系很难从直观上看出。鱼和熊掌不可得兼。

### 七：清除目标文件的规则 (clean的写法)
推荐的做法是每个目标文件都书写目标文件清除规则，不仅利于整个工程的重编译，同时能够保持工程文件的清洁。

**两种makefile清除规则的比较** <br>
一种普通的写法：
```txt
clean :
        rm edit $(objects)  
```
较为稳健的写法：
```txt
.PHONY : clean
clean :
        -rm edit $(objects)  
```
需要注意的是：
**(i)** 第二种写法中的".PHONY"表明clean是一个"伪目标"，clean不属于中间目标文件，也不属于其他文件的依赖文件。<br>
**(ii)** rm命令前面的"-"表示rm命令删除某些文件时可能会出现问题，请忽略这些问题并继续执行删除后面的文件。<br>
**(iii)** 不成文的规定是clean伪目标应该放在makefile的最后。clean不能放在makefile的开头，这会使得clean成为makefile的默认目标(默认目标用于生成可执行文件)。

## Reference
> https://blog.csdn.net/haoel/article/details/2887 <br>

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
