---
layout: post
title: "makefile - #8"
subtitle: '[跟我一起写makefile - 陈皓]' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-12-11 10:45
lang: ch 
catalog: true 
categories: makefile
tags:
  - Time 2021
---
## 使用变量(第二篇)
在Makefile中，变量是一个名字(类似c语言中的"宏")，变量的值代表一个文本字符串。Makefile中在目标、依赖、命令中引用变量的地方，变量会被其值所代替。Makefile中的变量有如下的特征：
- ➀ Makefile中的变量和函数(不包括规则命令行中的变量和函数)的展开，是在make读取Makefile文件时进行的。所提到的变量包括使用"="定义的变量、使用"define"定义的变量。
- ➁ Makefile中的变量可以用来表示：文件列表、编译选项列表、程序运行的选项参数列表、搜索源文件的目录列表、编译输出的目录列表等。
- ➂ Makefile中的变量是不包括：":"、"#"、"="、前置空白、尾空白的任何字符串。需要注意的是：尽管在GNU make中没有对变量的命名有其他限制，但是尽量不要定义一个包含除字母、数字、下划线以外其他字符的变量；这是因为其他字符中的特殊字符可能会被后续版本的make赋予特殊含义，这样命名的变量对于一些shell来说不能作为环境变量使用。
- ➃ Makefile中的变量是大小写敏感的：变量"foo"、"Foo"、"FOO"是三个不同的变量。Makefile中变量的传统命名法是全采用大写格式。但是推荐的做法是：对于Makefile内部使用的变量使用小写方式，e.g. 目标文件列表使用"objects"表示；对于一些外部变量则使用大写形式，e.g. 编译选项用"CFLAGS"表示。但这不是必须的，对于一个项目工程，Makefile中变量的命名应当使用同一种风格。
- ➄ 另外有一些变量名只包含了一个or很少的特殊字符(符号)如："$<"、"$@"、"$?"、"$\*"，它们被称为自动化变量。


### 八：变量的多行定义
定义变量的另一种方式就是使用"define"指示符，它能够定义一个包含多行字符串的变量。也正是define的这种特性允许我们实现了一个完整的命令包的定义。

##### 1 define定义变量的基本语法格式
```txt
# 以define开始 所定义的变量和define位于同一行 使用空格分隔
define two-lines

# 从define的下一行开始，到endef之间的若干行是变量值
echo foo
echo $(bar)

# 标识了define定义的结束
endef
```
当将上面定义的two-lines变量作为一个命令包进行执行时，等价于下面的形式：
```txt
two-lines = echo foo; echo $(bar)
```
即相当于将变量two-lines的值作为一个完整的shell命令来处理，两个命令使用";"来分隔，并按照类似shell中的"独立"两行命令的方式来处理(make命令将会调用shell两次，为每个命令运行一个独立的sub-shell)。

##### 2 define定义的变量的基本风格
使用"define"定义的变量和使用"="定义的变量一样，都属于"递归展开式"变量，他们两者的区别只是在于语法格式不同。因此，define定义的变量值中，对于其他变量、函数的引用不会在定义变量时进行替换展开，而是在define定义的变量解引用时才被展开。

##### 3 define定义的变量支持嵌套引用
define定义的变量支持嵌套引用。但是由于define定义的变量是递归展开式变量，因此在出现嵌套引用时，内部的变量引用不能展开。

##### 4 define定义的变量格式
define定义的变量值中可以包含：换行符、空格等特殊符号。注意，如果define定义的变量中某一行以"Tab"字符开始，当引用变量时这一行将会被当作命令行来处理。

##### 5 define定义变量时支持override指示符
在Makefile中使用define定义的变量前声明"override"指示符，这样可以防止define定义的变量的值被make命令行指定的数值替代。参考下面的案例进行理解：
```txt
# 带有override修饰的define定义变量的方式
override define two-lines
foo
$(bar)
endef
```


### 九：系统环境变量(variables from the environment)
make在运行时，系统中❮所有的❯环境变量对它都是可见的。即在Makefile中，可以引用任何已定义的系统环境变量。

例如，我们可以设置名为CFLAGS的系统环境变量，用它来指定一个默认的编译选项；于是就可以在所有的Makefile中直接指定使用CFLAGS环境变量对c源码进行编译。使用如CFLAGS这种系统环境变量的前提是：系统内所有用户充分了解该环境变量的含义；当然，用户也可以在特定Makefile中对其进行重新定义。

**[关于系统环境变量以及make环境变量之间的区别]** <br>
系统环境变量是这个系统中所有用户所拥有的，而make的环境变量❮只对make的一次执行过程有效❯。

**[使用系统环境变量时需要注意的事项]** <br>
- ➀ 在Makefile中对于一个变量的定义、以make命令行形式对一个变量的定义，都将覆盖同名的系统环境变量。注意，这种方式并不会改变"全局"系统变量的定义，Makefile中的变量只对make执行该文件时有效。当以"make -e"执行该文件时，❮Makefile中定义的变量不会覆盖同名的环境变量，make将使用系统环境变量的定义值，但是make命令中定义的变量仍然会覆盖系统环境变量值❯。
- ➁ 在make命令执行递归调用时，所有系统环境变量都会被传递给下一级make。默认情况下，只有系统环境变量和通过命令行方式定义的变量才会被传递给子make进程；在makefile中定义的普通变量则需要通过显式使用"export"指示符来声明将它传递给子make进程。
- ➂ 一个较为特殊的系统环境变量是SHELL。在系统中这个环境变量被用于用户对系统的交互接口的选择，显然将其用于make命令的选项是不合适的，原因如下：<br>
不建议make使用其他的系统环境变量。对于Makefile而言，将其功能依赖于❮在其控制之外❯设置的环境变量是不明智的，因为这将可能导致不同的用户从同一个Makefile中得到不同的结果，这违背了大多数的Makefile的目的。另外，尽量不要污染系统环境变量，系统环境变量是非常重要的。<br>
因此，make在执行时没有使用外部系统环境变量SHELL的定义，而是默认将"/bin/sh"作为它的命令行解释程序。

**[关于系统环境变量、Makefile定义的变量、make命令行定义的变量调用关系案例]** <br>
给定如下样例Makefile，假如我们的机器名为"server-cc"：
```txt
# Test Makefile
HOSTNAME = server-http
......
.PHONY : debug
debug:
    @echo "hostname is : $(HOSTNAME)"
    @echo "shell is $(SHELL)"
```
- ➀ 执行"make debug"，将显示：
```txt
# 默认Makfile将覆盖系统环境变量
hostname is : server-http
shell is /bin/sh
```
- ➁ 执行"make -e debug"，将显示：
```txt
# 使用make -e，Makefile中定义的参数变量不会覆盖同名系统环境变量
# make将使用系统环境变量定义值执行Makefile
hostname is : server-cc
shell is /bin/sh
```
- ➂ 执行"make -e HOSTNAME=server-ftp debug"，将显示：
```txt
# 使用make -e，Makefile中定义的参数变量不会覆盖同名系统环境变量
# 但是make命令行中定义的变量仍然会覆盖系统环境变量
hostname is : server-ftp
shell is /bin/sh
```


### 十：目标指定变量值(target-specific variable value)
在Makefile定义一个变量，则此变量对于整个Makefile都是有效的，它对于Makefile相当于"全局"变量。如果需要对其他Makefile中的规则有效，就需要使用"export"指示符进行声明。被export指示符声明的变量相当于c语言中的全局静态变量。当然，export指示符不能用于make中的自动化变量。

除export之外，make中的另一个特殊变量定义就是本节将介绍的"target-specific variable"。目标指定变量允许根据不同的目标指定不同的值，功能上类似于自动化变量。target-specific variable只在特定的目标的上下文有效，对其他目标没有影响。

**[设置target-specific variable的语法]** <br>
target-specific variable定义：
```txt
TARGET ... : VARIABLE-ASSIGNMENT
```
以及使用override指示符(避免变量被覆盖)的定义版本：
```txt
TARGET ... : override VARIABLE-ASSIGNMENT
```

**[target-specific variable的特点]** <br>
- ➀ "VARIABLE-ASSIGNMENT"可以是任何有效的赋值方式："="(递归展开方式)、":="(直接展开方式)、"+="(追加赋值)、"?="(条件赋值)。
- ➁ 使用target-specific variable value时，target-specific variable不会影响与之同名的全局变量的值。如果Makefile之前已经存在此变量的非目标指定的变量的定义，即通过上述方式定义target-specific variable时不会影响同名Makefile全局目标的值，只对定义时的目标可见。
- ➂ target-specific variable和普通变量具有相同的优先级，即当我们通过make命令行定义同名变量时，target-specific variable和普通变量一样会被命令行定义的变量覆盖；同理，当使用"make -e"执行Makefile时，同名的系统环境变量将会覆盖target-specific variable。因此，为了避免定义的target-specific variable被覆盖，可以使用override修饰的定义。
- ➃ target-specific variable和Makefile中的全局普通变量属于两个不同的变量，因此他们定义的风格(递归展开、直接展开)上可以不同。
- ➄ target-specific variable会作用到由定义它的目标涉及的所有规则上去。例如：
```txt
# 目标指定变量的定义
prog : CFLAGS = -g
# prog目标涉及的规则
prog : prog.o foo.o bar.o
# prog目标涉及的以及包含prog.o或foo.o或bar.o的规则
```
这个例子中，无论Makefile中的全局普通变量CFLAGS的定义是什么，对于目标prog，以及其引发的所有规则(包含目标为prog.o、foo.o、bar.o的规则)，变量CFLAGS的值都是-g。

**[使用target-specific variable可实现针对不同的目标文件使用不同的参数]** <br>
```txt
# Sample Makefile

CUR_DIR = $(shell pwd)
INCS := $(CUR_DIR)/include
CFLAGS := -Wall - I$(INCS)

EXEF := foo bar

.PHONY : all clean
all : $(EXEF)

foo : foo.c
# --- #1 foo目标指定变量CFLAGS的定义 ---
foo : CFLAGS+=-O2
bar : bar.c
# --- #2 bar目标指定变量CFLAGS的定义 ----
bar : CFLAGS+=g

......

$(EXEF) : debug.h
    $(CC) $(CFLAGS) $(addsuffix .c,$@) -o $@

clean :
    $(RM) *.o *.d $(EXEF)
```
上述Makefile文件借助target-specific variable机制实现了：在编译目标foo时，使用优化选项"-O2"而不使用调试选项"-g"；在编译目标bar时，采用调试选项"-g"而不使用优化选项"-O2"。#1和#2定义的CFLAGS互不干扰相互独立。上述案例体现了target-specific variable在定义变量的灵活之处。


### 十一：模式指定变量值(pattern-specific variable value)
除了上一节中提到的target-specific variable之外，GNU make还支持另一种方式定义的变量：pattern-specific variable。<br>
使用target-specific variable定义变量时，该变量被定义在某个具体目标及其引发的规则目标上；而本节将介绍的pattern-specific variable则是将变量指定到所有符合当前规则模式的目标上。

**[设置pattern-specific variable的语法]** <br>
pattern-specific variable定义：
```txt
PATTERN ... : VARIABLE-ASSIGNMENT
```
以及使用override指示符的定义版本(避免变量被覆盖)：
```txt
PATTERN ... : override VARIABLE-ASSIGNMENT
```
注意，pattern-specific variable和target-specific variable之间唯一的区别是：这里的目标是一个or多个"模式"目标(包含模式字符"%")。参考如下例子进行理解：
```txt
%.o : CFLAGS += -O
```
该模式指定变量定义为所有的".o"类型文件的编译选项CFLAGS指定了"-O"，但是不会改变其他类型文件的编译选项。

**[pattern-specific variable在模式匹配时优先匹配"更加具体"的变量]** <br>
> If a target matches more than one pattern, the matching pattern-specific variables with longer stems(模版更长) are interpreted first. This results in more specific variables taking precedence over the more generic ones. <br>

参考下面的案例进行理解：
```txt
%.o : %.c
    $(CC) -c $(CFLAGS) $(CPPFLAGS) $< -o $@

# --- #1 pattern-specific variable的定义: 较为具体的pattern ---
lib/%.o: CFLAGS := -fPIC -g
# --- #2 pattern-specific variable的定义: 较为通用的pattern ---
%.o: CFLAGS := -g

all: foo.o lib/bar.o
```
上述例子中，#1中CFLAGS的变量值"-fPIC -g"将被用于更新符合#1中pattern的目标"lib/bar.o"；尽管#2中的pattern也符合目标"lib/bar.o"，但是还是优先匹配更加具体的模式对应的变量定义。<br>
foo.o目标只能够和#2中的pattern进行匹配，因此对于#2中CFLAGS变量的定义无二义性。


## 条件判断
条件语句可以根据一个变量的值来控制make执行or忽略Makefile的特殊部分。条件语句可以是两个变量值之间的比较、也可以是变量值和常量之间的比较。需要注意的是：条件语句只能用于控制make所执行的Makefile文件部分，但是它不能控制每条规则对应的shell命令执行过程细节。在Makefile中使用条件控制可以增加Makefile的灵活性、高效性。


### 一：一个例子
下面的Makefile部分通过判断变量CC的值是否为gcc，如果是则在程序连接时使用库"libgnu.so"或"libgnu.a"，否则不链接任何库。
```txt
......
# --- 两个递归展开变量定义 ---
libs_for_gcc = -lgnu
normal_libs = 

......

foo : $(objects)
# --- 使用ifeq/else/endif条件判断 ---
ifeq ($(CC),gcc)
    $(CC) -o foo $(objects) $(libs_for_gcc)
else
    $(CC) -o foo $(objects) $(normal_libs)
endif
```
从上述案例可以看出，Makefile的条件表达式是一个工作于文本级别的处理过程，条件的解析由make来完成(make读取&解析Makefile，解析完成后只会保留满足条件的文本行)。因此，上面的Makefile被make解析为：<br>
当变量CC的值为gcc时，整个条件表达式等效于：
```txt
foo : $(objects)
    $(CC) -o foo $(objects) $(libs_for_gcc)
```
当变量CC的值不为gcc时，整个条件表达式相当于：
```txt
foo : $(objects)
    $(CC) -o foo $(objects) $(normal_libs)
```
上述案例可以使用一种更加简洁的条件表达式进行实现：
```txt
libs_for_gcc = -lgnu
normal_libs = 
# --- 条件判断语句 注意变量不用[tab]开头 ---
ifeq ($(CC),gcc)
libs = $(libs_for_gcc)
else
libs = $(nomal_libs)
endif

foo : $(objects)
    $(CC) -o foo $(objects) $(libs)
```


### 二：条件判断的基本语法格式
一个不包含else分支的条件判断语句的语法格式为：
```txt
CONDITIONAL-DIRECTIVE
TEXT-IF-TRUE
else
TEXT-IF-FALSE
endif
```
一个包含else分支的条件判断语句的语法格式为：
```txt
CONDITIONAL-DIRECTIVE
TEXT-IF-TURE
else
TEXT-IF-FALSE
endif
```

##### 1 "ifeq"关键字
这个关键字用来判断两个变量的值是否相等，其应用格式如下：
```txt
ifeq (<arg1>, <arg2>)
ifeq '<arg1>' '<arg2>'
ifeq "<arg1>" "<arg2>"
ifeq "<arg1>" '<arg2>'
ifeq '<arg1>' "<arg2>"
```
ifeq常用于判断一个变量值是否为空：
```txt
# --- strip函数用于去掉foo变量中的前导、尾后空格，得到结果再和空字符比较 ---
ifeq ($(strip $(foo)),)
TEXT-IF-EMPTY
endif
```

##### 2 "ifneq"关键字
这个关键字用来判断两个变量的值是否不相等，其应用格式如下：
```txt
ifneq (<arg1>, <arg2>)
ifneq '<arg1>' '<arg2>'
ifneq "<arg1>" "<arg2>"
ifneq "<arg1>" '<arg2>'
ifneq '<arg1>' "<arg2>"
```

##### 3 "ifdef"关键字
这个关键字用来判断一个变量是否已经被定义过，其应用格式如下：
```txt
ifdef VARIABLE-NAME
```
Makefile中未定义的变量为空，如果VARIABLE-NAME的值为非空，则表达式为真，否则为假。

注意：ifdef关键字只会判断VARIABLE-NAME是否有值，而不会对于变量中的引用进行替换展开以判断变量的值是否为空。即对于变量VARIABLE-NAME而言，除了
```txt
VARIABLE-NAME=
```
这种格式外，其他任何方式对其进行定义都会使得ifdef表达式为真，参考下面两个例子进行理解：<br>
案例#1：
```txt
bar =
foo = $(bar)
ifdef foo
frobozz = yes
else
frobozz = no
endif
```
案例#2：
```txt
foo = 
ifdef foo
frobozz = yes
else
forbozz = no
endif
```
案例#1的结果是"frobozz = yes"；案例#2的结果是"frobozz = no"。

##### 4 "ifndef"关键字
关键字ifndef实现的功能和ifdef恰好相反，其格式为：
```txt
ifndef VARIABLE-NAME
```

##### 5 应用条件表达式时的注意事项
➀ make读取Makefile文件时计算表达式的值，并且根据表达式的值决定Makefile中哪部分作为执行内容。因此条件表达式中不能使用自动化变量，因为自动化变量只有在规则❮命令执行时❯才有效。
➁ 不能将完整的条件判断语句分别写在两个不同的makefile文件中，即不能使用include指示符链接两个Makefile文件，以构成一个完整的条件判断语句。


### 三：标记需要被测试的条件语句
可以通过使用条件判断语句、变量MAKEFLAGS、函数findstring来实现对make命令行选项的测试，结合如下例子进行理解：
```txt
archive.a: ...
# --- 条件判断语句 MAKEFLAGS中包含-t参数 ---
ifneq (,$(findstring t,$(MAKEFLAGS)))
    +touch archive.a
    +ranlib -t archive.a
# --- 条件判断语句 MAKEFLAGS中不包含-t参数 ---
else
    ranlib archive.a
endif
```
这个条件判断语句用来判断make的命令行参数中是否包含"-t"参数(该参数用于更新目标文件的时间戳)。"+"作为命令前缀将相应规则命令行标记成"recursive"，使用"+"作为命令前缀表明：即使make已经包含了-t参数(目标文件已经处于最新时间戳状态)，"+"后面的命令仍需要被执行(强制执行)。



## Reference
> https://blog.csdn.net/haoel/article/details/2893 <br>
> GNU make zh/en chapter 6.8 & 6.9 & 6.10 & 6.11 & 6.12 & 7

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
