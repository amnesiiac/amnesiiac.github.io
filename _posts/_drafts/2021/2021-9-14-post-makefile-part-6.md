---
layout: post
title: "makefile - #6"
subtitle: '[跟我一起写makefile - 陈皓]' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-09-14 16:36
lang: ch 
catalog: true 
categories: makefile
tags:
  - Time 2021
---
## 书写命令(书写通过依赖文件生成目标文件的命令)
makefile中每条规则中的命令由shell命令构成，make会按照顺序逐条进行执行。
- 除了紧跟在规则中依赖文件后面的使用分号隔开的第一条命令外，其他的命令都需要以"tab"字符开始。
- 多个命令行之间可以存在空行or注释行：空行是不包含任何字符的行。
- 以"tab"字符开头的是空命令行，空命令行在命令被执行时自动忽略。
- 如果不明确指定用于makefile命令执行的shell解释器，则命令默认使用"/bin/sh"来完成
- 命令行中的"#"字符到行尾的部分会被认为是注释。

### 一：makefile命令回显
通常情况下，make会在执行具体命令之前，将准备执行的命令打印到标准输出设备，称之为命令回显(echo)。如果我们想要避免make将待执行的命令打印到输出设别上，可以在命令前面加上"@"字符。<br>
**[1] 不使用"@"字符** <br>
make将命令打印到标准输出上。
```txt
echo 正在编译xxx模块...
```
上述命令被执行时，将会得到如下输出，其中第一行是make程序执行命令前的输出，第二行是执行echo程序得到的输出。
```txt
echo 正在编译xxx模块...
正在编译xxx模块...
```
**[2] 使用"@"字符** <br>
make不会将命令打印到标准输出上。
```txt
@echo 正在编译xxx模块...
```
上述命令执行时，将会得到如下输出：
```txt
正在编译xxx模块...
```

### 二：makefile命令执行
makefile中的规则中，当特定规则的依赖文件版本比规则版本要新，则规则相应的命令就会被执行。<br>
一个规则对应的命令可能是单行命令，也可能是多行命令。如果命令为多行命令，那么每行命令将会在独立的shell进程中执行，因此多行命令之间也应当是独立的，互相不能存在依赖。<br>
下面通过两个案例来加深理解。

##### 1 单行命令
单行命令使用分号";"进行连接。 
```txt
foo : bar/lose
    cd bar; gobble lose > ../foo
```
cd命令和后续的命令位于同一行，因此使用同一个shell进程进行处理，cd改变当前目录后将会对后续命令中的"../foo"产生影响。

##### 2 多行命令
多行命令需要使用反斜杠"\"进行连接。
```txt
foo : bar/lose
    cd bar; \
    gobble lose > ../foo
```
cd命令和后续的命令位于不同行，即它们属于不同的进程。cd改变当前目录，但是不会影响到后续命令中的"../foo"产生影响。

##### 3 关于make命令执行使用的程序
unix make默认使用"SHELL"环境变量所指定的应用程序对于makefile中规则相对应的命令进行执行。可以在makefile中指定所使用的命令解释程序，此时需要使用解释程序在系统中的完整路径名(e.g. "/bin/sh")。

在寻找命令解释器的过程中，首先在"SHELL"环境变量中指定的路径中进行寻找；如果找不到，将会在当前盘符的当前目录中进行寻找；如果仍然不能找到，那么会在PATH变量中定义的路径中进行寻找。

MS-DOS系统环境下没有"SHELL"环境变量，关于该系统如何设置命令解释器路径，参考GNU info make。 

### 三：命令执行中的错误信息及处理方式
每条命令执行完毕，make都会检查每个命令的返回码。如果命令返回码等于0(表示命令执行成功)，则make继续执行下一条命令。如果当前规则中所有命令都执行成功，则该规则执行完成。如果特定规则下某一条命令执行失败，则make终止执行当前规则，并有可能导致make终止所有规则的执行。

**有些时候，即使命令执行出错并不会影响整个规则的完成** <br>
例如：执行mkdir命令时，如果目标路径or文件不存在，则mkdir命令创建相应路径or文件，如果目标路径or文件已经存在，则mkdir命令报错。<br>
应当注意到，此时虽然mkdir命令报错，但是不会影响后续命令的执行(目标存在恰恰就是mkdir想要达到结果)，此时应当设置make程序继续执行。<br>
有如下几种方式可以忽略命令执行的出错：

##### 1 在命令前加上"-"
下面的makefile通过给rm命令添加"-"字符比表明，无论"\*.o"文件是否删除成功，都将继续执行后续规则相关命令。
```txt
clean :
    -rm -f *.o
```
使用"-"前缀后，即使相应命令出现错误，make也会将其看作是"成功执行的"，但是会提示错误信息，并且说明该错误已经被忽略。

##### 2 在全局上为make命令加上"-i"或者"\-\-ignore-errors"选项
使用"make -i"或者"make \-\-ignore-errors"选项，make命令将会忽略所有的错误。<br>
在make命令执行时，加上"-i"或者"\-\-ignore-errors"选项，即使相应命令出现错误，make也会将其看作是"成功执行的"，但是会提示错误信息，并且说明该错误已经被忽略。

##### 3 无依赖文件的特殊目标：".IGNORE"
如果一个规则以".IGNORE"为目标，那么这个规则中的所有命令将会忽略执行中产生的错误。<br>
这种方式已经很少使用，因为它不如"-"的方式灵活。

##### 4 在全局上为make命令加上"-k"或者"\-\-keep-going"选项
当不使用上述三种方式，以普通的方式执行make命令时，如果执行过程中发生错误，将意味着该命令规则对应的目标无法被创建，进而和该目标相关的其他目标也不会被重建。发生这种情况时，通常make会立即退出并返回一个非0状态值表示命令执行失败。<br>
如果在make执行命令时，添加"-k"或者"\-\-keep-going"选项，那么在某规则对应的命令出现错误时，make不会立即退出，而是继续后续命令的执行。使用该选项后，直到最后发生错误导致命令无法执行时才退出。

**"make -k"或者"make \-\-keep-going"命令的一般用途**：<br>
当我们同时修改了工程中的多个文件后，"-k"或"\-\-keep-going"选项可以帮助我们确认哪些修改是正确的(可接受的)，哪些文件的修改时错误的(不可接受的)。<br>
例如，我们修改了工程中的20个源文件，修改后运行带有上述选项的make，则该命令可以一次性找出修改的20个文件中哪些不能够被正确编译。即：使用"make -k"或者"make \-\-keep-going"命令编译能够通过的源文件都是正确的，而不能通过的源文件不是自身有问题，就是编译它所需的依赖文件存在问题。

### 四：嵌套执行make命令
make的递归执行指的是：在makefile中使用make作为命令来执行自身or其他makefile的过程。递归调用在一个具有多级子项目的大型工程中非常有用：使用该技术，可以避免将工程中所有的编译工序都写在同一个makefile中(使用单个makefile进行编译管理是困难、复杂、难以维护的)；另外，该技术可以允许我们对于大型工程进行分模块编译(空间分隔)、分段编译(时间分隔)。

##### 1 一个简单的递归执行的例子
除了当前目录的makefile之外，有一个子目录subdir，该目录下有一个makefile文件，用于指明subdir目录下项目文件的 编译规则。则"总控makefile"可以书写如下：
```txt
subsystem:
        cd subdir && $(MAKE)
```
上述写法等价于：
```txt
subsystem:
        $(MAKE) -C subdir
```
其中，"$(MAKE)"表示引用了变量MAKE；"-C"选项表示设置变量"CURDIR"，该变量表示当前make工作目录，使用"-C"选项设置工作路径后，CURDIR被重新赋值。
上述两种写法的含义都是：进入subdir目录，然后执行。

##### 2 多层级makefile之间变量的传递
上一节中的makefile称之为"总控makefile"，被调用的位于子目录makefile可以称为"子目录makefile"。需要注意的是："总控"makefile中的变量可以通过显式声明的方式传递到"下级"makefile中，但是所传递的变量不会覆盖"下级"makefile中的变量；可以通过指定"-e"选项的方式将传递的变量覆盖下层变量。

**[1] 将上层makefile中的变量传递到下层makefile中去** <br>
可以在上层makefile文件变量定义时通过"export \<variable...\>"命令显式实现：<br>
**[案例一]**
```txt
#1 第一种形式
export variable = value

#2 第二种形式
variable = value
export variable

#3 第三种形式
export variable := value

#4 第四种形式
variable := value
export variable
```
上述4种形式在向下层makefile传递变量时是等价的。<br>
**[案例二]**
```txt
#1 第一种形式
export variable += value

#2 第二种形式 
variable += value
export variable
```
上述2种形式在向下层makefile传递变量时是等价的。<br>
**[案例三]** <br>
如果想要将上层makefile中的**所有变量**全部传递到下层makefile中去，只需"export"关键字即可。
```txt
export
```
**[2] 关于特殊变量"SHELL"和"MAKEFLAGS"** <br>
这两个特殊变量除非使用指示符"unexport"进行声明，否则它们在整个make的执行过程中始终被自动地传递给所有下层makefile。
```txt
unexport SHELL = ...
unexport MAKEFLAGS = ...
```

**[3] 关于特殊变量"MAKEFILES"** <br>
只要"MAKEFILES"变量有值(不为空)，则变量同样会被自动的传递给下层makefile。

**[4] 关于没有使用"export"关键字定义的变量** <br>
没有使用"export"关键字定义的变量，make执行时，它们不会自动传递给下层makefile，因此，下层makefile可以定义这些上层变量的同名变量，且并不会引起变量名冲突。

**[5] 关于上层makefile中的make选项向下层makefile中进行传递** <br>
**➤ 支持上级传递到下级的进程的make选项参数** <br>
➀ 在make命令的递归执行过程中，最上层的make命令行选项**"-k"、"-s"**会**自动地**通过"MAKEFLAGS"环境变量传递给makefile子文件(make命令的解析子进程管理的文件)。即，如果在执行主控make命令时指定了"-k"以及"-s"选项，那么"MAKEFLAGS"环境变量的数值将会被自动设置为"ks"，子make进程处理相应文件时，会带有"-k"和"-s"两个命令行进行执行。<br>
➁ 同样地，在执行make命令时给定某变量的定义，则此变量及其定义的值将会借助环境变量"MAKEFLAGS"传递给子进程(及其负责的makefile文件)。例如：主控make命令执行时采用"make CFLAGS+=-g"命令，则该变量"CFLAGS"及其数值"CFLAGS+=-g"都会传递到子make进程中去。

**➤ 并不是所有的选项都会自动传递给make子进程**：<br>
"-C"、"-f"、"-o"、"-W"这几个选项不会赋值给"MAKEFLAGS"变量，因此它们不会被子进程make命令使用。

**➤ 执行多级make命令调用时，不希望上级make命令中的MAKEFLAGS传递给下级make** <br>
只需要在跨级之间的make命令调用时，将"MAKEFLAGS"环境变量置空即可。
```txt
subsystem :
        cd subdir && $(MAKE) MAKEFLAGS=
```

##### 3 使用"-w"或"\-\-print-directory"选项打印编译目录相关的提示信息
在多级make命令的调用中，使用"-w"或"\-\-print-directory"选项，可以让make在开始编译一个目录时or完成特定目录编译时打印相应提示信息。这些信息可以方便工程项目开发人员跟踪编译具体执行流程。

**[1] 具体案例**
例如：在目录"/u/gnu/make"目录下执行"make -w"命令，则在命令开始执行后将会看到：
```txt
make: Entering directory `/u/gnu/make`.
```
当make命令执行完成后，将会看到：
```txt
make: Leaving directory `/u/gnu/make`.
```
**[2] 关于"-w"选项的补充说明** <br>
通常情况下，"-w"选项会被自动打开；另外在"主控"makefile调用下层makefile时，使用"-C"选项为make命令指定了一个目录，或者使用类似"cd subdir && $(MAKE)"的命令时，"-w"选项会被自动打开。<br>
可以在主控make命令使用"-s"或者"\-\-slient"选项来禁用"-w"选项。<br>
另外，make命令中的"\-\-no-print-directory"选项可以禁止**所有**关于目录信息的打印。

### 五：定义命令包
对于makefile中出现的一组命令序列，我们可以为这些命令序列定义一个变量。这样以后再需要使用这组命令序列时，可以通过该变量进行引用，这样减少了重复工作、提升了效率。上述定义的变量包含的命令序列称之为"命令包"。<br>

##### 1 定义一个命令包(案例)
命令包的定义以"define"关键字开始，以"endef"关键字结束。

**[1] 命令包定义示例**
```txt
define run-yacc
# 运行yacc命令
yacc $(firstword $^)
# 将yacc输出的目标文件改一个名字
mv y.tab.c $@
endef
```
上面定义了一个称为"run-yacc"的变量(命令包的名字)，需要注意，所定义的命令包名字不能和makefile中的变量同名；同时"define"后面两行定义的命令序列称之为"命令体"。<br>
另外需要注意的是，命令包的名字和c/c++中的宏一样：命令体中的所有变量、函数引用不会立即展开。即上述命令体中所有内容包括："$、()"等都是变量的定义。在变量被引用的位置，先执行类似"宏替换"的操作，替换完成后再将命令体中的其他变量、函数引用展开。

**[2] 命令包的使用** <br>
给定原始命令包的使用makefile如下：
```txt
foo.c : foo.y
        $(run-yacc)
```
上述命令包可以使用"宏替换"的方式进行展开：
```txt
foo.c : foo.y
        # 运行yacc命令
        yacc $(firstword $^)
        # 将yacc输出的目标文件改一个名字
        mv y.tab.c $@
```
对于展开的命令包中的引用、变量进行分析："$^"表示当前规则中的所有依赖文件"foo.y"；"$@"表示所有目标文件"foo.c"。将命令包中所有命令替换完成后，执行yacc和mv命令即可。

##### 2 在命令包中添加前缀可以控制用于修饰命令包中所有命令
**[1] 在命令包中的每条命令前添加"-/+/@"进行修饰** 
```txt
define frobnicate
    @echo "frobnicating target $@"
    frob-step-1 $< -o $@-step-1
    frob-step-2 $@-step-1 -o $@
endef
```
上述命令包定义文件中，"@echo"起始的命令行由于使用了"@"，从而不会将正在执行的命令打印到标准输出上。

**[2] 引用"frobnicate"命令包 & 添加"整体修饰"**
```txt
frob.out :
    frob.in @$(frobnicate)
```
上述引用"frobnicate"命令前，使用了"@$(...)"进行整体修饰，因此frobnicate命令包中的所有命令执行时都不会向标准输出上打印命令。


## Reference
> https://blog.csdn.net/haoel/article/details/2891 <br>
> GNU make zh chapter 4.10 & 4.12 & 5.6 & 5.8

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
