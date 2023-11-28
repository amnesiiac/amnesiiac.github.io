---
layout: post
title: "makefile - #7"
subtitle: '[跟我一起写makefile - 陈皓]' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-09-15 16:33
lang: ch 
catalog: true 
categories: makefile
tags:
  - Time 2021
---
## 使用变量(第一篇)
在Makefile中，变量是一个名字(类似c语言中的"宏")，变量的值代表一个文本字符串。Makefile中在目标、依赖、命令中引用变量的地方，变量会被其值所代替。Makefile中的变量有如下的特征：
- ➀ Makefile中的变量和函数(不包括规则命令行中的变量和函数)的展开，是在make读取Makefile文件时进行的。所提到的变量包括使用"="定义的变量、使用"define"定义的变量。
- ➁ Makefile中的变量可以用来表示：文件列表、编译选项列表、程序运行的选项参数列表、搜索源文件的目录列表、编译输出的目录列表等。
- ➂ Makefile中的变量是不包括：":"、"#"、"="、前置空白、尾空白的任何字符串。需要注意的是：尽管在GNU make中没有对变量的命名有其他限制，但是尽量不要定义一个包含除字母、数字、下划线以外其他字符的变量；这是因为其他字符中的特殊字符可能会被后续版本的make赋予特殊含义，这样命名的变量对于一些shell来说不能作为环境变量使用。
- ➃ Makefile中的变量是大小写敏感的：变量"foo"、"Foo"、"FOO"是三个不同的变量。Makefile中变量的传统命名法是全采用大写格式。但是推荐的做法是：对于Makefile内部使用的变量使用小写方式，e.g. 目标文件列表使用"objects"表示；对于一些外部变量则使用大写形式，e.g. 编译选项用"CFLAGS"表示。但这不是必须的，对于一个项目工程，Makefile中变量的命名应当使用同一种风格。
- ➄ 另外有一些变量名只包含了一个or很少的特殊字符(符号)如："$<"、"$@"、"$?"、"$\*"，它们被称为自动化变量。


### 一：变量的引用
当我们定义了一个变量后，就可以在Makefile中的其他地方使用该变量。可以通过"$(VARIABLE_NAME)"、"${VARIABLE_NAME}"的方式来引用变量。注意，美元符号"$"在Makefile中有特殊的含义，因此在命名时需要使用"$"时，需要使用两个美元符号进行表示"$$"。变量的引用可以在Makefile中的任何上下文中、目标、依赖、命令、绝大多数指示符、新变量的赋值中。

**[1] 案例**：变量引用用于表示目标文件(.o)列表：
```txt
objects = program.o foo.o utils.o
program : $(objects) 
    cc -o program $(objects)

$(objects) : defs.h
```
**[2] 案例**：变量的展开过程是❮严格❯的文本替换的过程，变量值的字符串被精确的展开在变量被引用的地方。这种展开方式同c语言中的宏相同，是一个严格的文本替换过程。
```txt
foo = c
prog.c : prog.$(foo)
    $(foo)$(foo) -$(foo) prog.$(foo) 
```
展开后为：
```txt
prog.c : prog.c
    cc -c prog.c
```
**[3] 关于Makefile引用变量的补充**：对于Makefile中一些简单变量的引用，可以不使用"$(x)"或"${x}"，而是直接使用"$x"的方式来实现。但是此方法只适合单字符的情况。Makefile中的自动化变量也使用这种形式(e.g. "$<"、"$@"、"$?"、"$\*")。

对于多字符变量的引用，必须使用括号进行标记，否则make将把变量名的首字母而不是整个字符串当作变量；e.g. "$PATH"在Makefile中被解释成"$(P)ATH"。这一点和shell中变量的引用不同：shell中变量的引用可以是"${xx}"或者"$xx"的形式。

**[4] 关于Makefile书写变量的引用格式时的注意事项**：
- ➀ Makefile中定义的变量或者make命令的环境变量的引用使用"$(VAR)"格式，无论"VAR"表示单字符变量还是多字符变量名。
- ➁ 出现在规则命令行中的变量(一般为执行命令过程中的临时变量，它不是Makefile变量而是一个shell变量)，这种变量的引用使用shell的"$VAR"格式。
- ➂ 对出现在命令行中的make命令的变量，使用"$(VAR)"格式来引用。

结合下面案例进行理解：
```txt
# sample Makefile
......
SUBDIRS := src foo

.PHONY : subdir
Subdir : 
    @for dir in $(SUBDIRS); do; \
        $(MAKE) -C $$dir || exit 1; \
    done
......
```


### 二：两种变量的定义/赋值
GNU Makefile中有两种变量的定义方式(风格)，它们之间的区别在于：1) 定义它们的方式，2) 展开它们的时机。本小节的内容主要对这两种不同的风格进行详细的讨论。

##### 1 递归方式展开变量(recursively expanded variable)
这种变量是通过"="以及"define"定义的。这种变量在被引用的地方是严格的文本替换过程，即变量的字符原模原样的出现在引用它的地方。当这种变量中出现其他变量引用时(变量的嵌套定义)，这些被引用的变量会在"外层"变量被展开的同时被一并展开。参考下面的案例进行理解：
```txt
foo = $(bar)
bar = $(ugh)
ugh = Huh?

all:; echo $(foo)
```
上述3层嵌套的变量在执行"echo $(foo)"时被展开：首先"$(foo)"被替换成"$(bar)"，接下来"$(bar)"被替换成"$(ugh)"，最后"$(ugh)"被替换成"Huh?"。因此执行make命令将打印"Huh?"。

这种recursively expanded variable也被其他make版本所支持。它既有优点也有缺点：<br>
- ➀ [pros] recursively expanded variable在定义时，可以引用其他的之前未被定义的变量(可能在Makefile后续部分被定义or通过make命令行选项参数传递进行定义)。
```txt
CFLAGS = $(include_dirs) -O
include_dirs = -Ifoo -Ibar
```
虽然"include_dirs"在变量"CFLAGS"之后才被定义，但是却可以被"CFLAGS"所引用，"CFLAGS"将被展开为"-Ifoo -Ibar -O"。
- ➁ [cons] 使用这种风格的变量定义，可能由于出现变量的递归定义而导致make陷入无限的变量展开过程中，最终导致make命令执行失败。
```txt
# 第一种情况：变量定义不能引用自身，否则将会出现循环展开
CFLAGS = $(CFLAGS) - O
```
```txt
# 第二种情况：两个变量的定义不能出现交叉引用，否则将出现循环展开
x = $(y)
y = $(x)$(z)
```
- ➂ [cons] 这种风格变量的定义中如果使用了函数，那么包含在变量值中的函数总会在变量被引用的地方执行(变量被展开时执行)，而不是定义该类型变量时被展开。这将导致make的执行效率降低：每次在变量被引用的地方都会进行一次函数展开；另外，这种函数引用的"延迟"展开可能会导致一些变量、函数引用的展开出现非预期的结果，特别是当变量定义中出现了"shell"和"wildcard"函数引用时，可能出现不可控制or难以预料的错误(无法确定它们何时被展开)。

##### 2 直接方式展开变量(simply expanded variables)
为了避免递归方式展开变量在引用时存在的不便，GNU make还支持另外一种风格的变量，称为simply expanded varibles。这种类型定义的变量使用":="或"::="来定义。变量值中的其他变量、函数的引用在simply expanded variables在定义时被展开，即simply expanded variables在定义时即确定其值。

**[1] 直接方式变量定义案例一(简单)**：<br>
下面的Makefile：
```txt
x := foo
y := $(x) bar
x := later
```
等价于：
```txt
# y变量定义时即刻展开
y := foo bar
x := later
```
直接方式展开变量由于在定义时需要将所有引用展开，因此无法同递归展开变量一样对后续变量定义的引用：
```txt
# 直接方式展开变量无法引用后续定义的变量
CFLAGS := $(include_dirs) -O
include_dirs := -Ifoo -Ibar
```
由于"include_dirs"出现了"CFLAGS"之后，因此在"CFLAGS"变量定义时，"include_dirs"的值为空，因此上述直接展开变量"CFLAGS"的值为"-O"而不是"-Ifoo -Ibar"。

**[2] 直接方式变量定义案例二(较复杂)**：<br>
下面的例子中涉及了make命令的shell函数(在shell中运行响应命令得到结果)、变量MAKELEVEL(此变量在make执行递归调用时表示make调用的深度)：
```txt
# 判断是否为顶层Makefile 则定义下面的变量
ifeq(0, ${MAKELEVEL})                                   
# 直接展开变量定义
cur-dir := $(shell pwd)
whoami := $(shell whoami)
host-type := $(shell arch)
MAKE := ${MAKE} host-type=${host-type} whoami=${whoami}
endif
```
利用直接展开变量的特点，可以书写如下的规则：
```txt
# 使用递归展开变量
${subdirs} : 
    ${MAKE} cur-dir=${cur-dir}/$@ -C $@ all
```
可以实现在不同的子目录下，"cur-dir"使用不同的值(即当前工作目录)。

##### 3 关于两种变量定义风格的总结
在复杂的Makefile书写中，推荐使用直接展开的方式定义变量，这样可以使一个复杂的Makefile一定程度上具有可预测性。另外，使用直接展开方式定义变量允许我们在引用变量数值之前重新定义它的取值(e.g. 使用某一函数对其之前的取值进行处理并为其重新赋值)。在Makefile中应当尽量避免、减少递归变量的使用。

##### 4 如果定义一个包含尾空格的变量
一般情况下，变量值的前导空格在变量引用、函数调用时被自动丢弃。使用直接展开式变量的定义可以让我们定义一个包含前导空格的变量值(controlled leading whitespace into variable values)。其定义方式如下：
```txt
nullstring := 
space := $(nullstring) # end of the line
```
上述直接展开方式定义的变量space恰好就是一个"空格"。这个"空格"的前面使用变量"nullstring"来标识，"空格"的后面使用注释符号"#"来标识结尾。

make对变量进行处理时，变量值中的尾空格是不能被忽略的，因此定义包含一个or多个尾空格的变量时，推荐使用上述实现方式。如果变量中不包含尾空格时，不要使用上述方式书写注释，否则注释之前的空格会被当作变量值的一部分；例如，下面的做法是违背初衷的：
```txt
dir := /foo/bar    # directory to put the frobs in
```
上面定义的变量dir的值是"/foo/bar/    "，而不是"/foo/bar"，显然如果将带有尾空格的路径表示文件file："$(dir)/file"将会大错特错。

在书写Makefile时，推荐将注释书写在独立的行或者单独占用多行，以防止出现上面案例中的意外情况。另外，当出现包含一个or多个空格的变量时，应书写注释对于包含的空格的数量进行说明。

##### 5 "?=" 操作符
GNU make中有一个被称为条件赋值操作符"?="。被称为条件赋值是因为：仅当此变量之前未被赋值的情况下才会对该变量赋值。如下面例子所示：
```txt
FOO ?= bar
```
其等价于：
```txt
# 如果变量FOO在之前没有被定义过 就将它赋值为bar 否则不改变它的取值
ifeq($(origin FOO), undefined)
FOO = bar
endif
```


### 三：变量的高级用法
这一部分内容主要介绍变量的高级用法，它们可以让我们更加灵活的使用变量。

##### 1 变量的替换引用
对于一个已经定义的变量，可以使用"替换引用"将其取值中的后缀字符替换为指定的字符。其格式为："$(VAR:A=B)"或"${VAR:A=B}"，即将变量VAR中所有以A字符❮结尾❯的字符串替换成以B字符结尾的字符串。❮结尾❯的含义是：变量值的空格之前。对于变量中其他部分的A字符不予替换，如下所示：
```txt
# 变量foo一共包含3个字：a.o、b.o、c.o 
foo := a.o b.o c.o
bar := $(foo:.o=.c)
```
经过替换后的变量bar的值为"a.c b.c c.c"。再看一个对照的例子：
```txt
foo := a.o b.o c.o o.o
bar := $(foo:.o=.c)
```
经过替换后的变量bar的值为"a.c b.c c.c o.c"而不是"a.c b.c c.c c.c"。

变量的替换引用其实是函数"patsubst"的一个简化实现。其格式和变量的替换引用类似："$(VAR:%.A=%.B)"，即需要在替换字符对之前使用模式字符"%"。GNU make同时提供了这两种方式替换变量中的字的结尾字符，以兼容其他版本的make。下面提供使用"patsubst"函数的例子：
```txt
foo := a.o b.o c.o
bar := $(foo:%.o=%.c)
```
其等价于：
```txt
foo := a.o b.o c.o
bar = $(patsubst o,c $(foo))
```
它们使得变量bar的数值为"a.c b.c o.c"。这种格式的替换相比第一种更加通用。

##### 2 变量的嵌套引用(变量之中包含对其他变量的引用)
变量的嵌套引用(计算变量名)是一个比较复杂的概念，仅用在较复杂的Makefile中。

**[案例1]** <br>
一个双层直接展开式变量嵌套的例子：
```txt
x = y
y = z
a := $($(x))
```
上面变量a的取值为"z"。首先❮从内而外❯的对变量嵌套引用进行分析：最内层的引用"$(x)"被替换为其值"y"，此时变量a的定义变成"$(y)"，继续解引用得到变量a的取值为z。

**[案例2]** <br>
Makefile中允许多层嵌套，如下面的三层直接展开式变量嵌套也是允许的(解析方式同上例一样，不再赘述)：
```txt
x = y
y = z
z = u
a := $($($(x)))
```

**[案例3]** <br>
递归展开式变量的嵌套定义的变量也是按照相同的❮从内而外❯的方式进行展开的：
```txt
x = $(y)
y = z
z = Hello
a := $($(x))
```
上面变量a的取值为"Hello"。首先❮从内而外❯的对变量嵌套引用进行分析：最内层引用"$(x)"被替换为"$(y)"，递归展开变量在引用时才被展开，因此内层"$(x)"被替换为"z"；然后解除外层引用"$(z)"得到Hello。

**[案例4]** <br>
变量的多层嵌套还可以包含变量的修改引用、函数调用。如下面例子所示，使用了make文本处理函数：
```txt
x = variable1
variable2 := Hello
y = $(subst 1,2,$(x))
z = y
a := $($($(z)))
```
这个案例同样实现了变量a取值为"Hello"。首先❮从内而外❯的对变量嵌套引用进行分析：最内层引用"$(z)"被替换为y，中间层引用"$(y)"为subst函数的返回结果variable2，最外层引用变成"$(variable2)"，解引用得到Hello。

**[关于如何正确地使用多层变量嵌套#1]** <br>
在书写Makefile时，应当尽量避免使用嵌套的变量引用，即使在一些必须使用的地方也只多不要使用超过2级的嵌套引用。尤其是当多级嵌套引用需要和递归展开式变量定义一起使用时，要注意递归展开式变量的展开时机，一旦书写不当容易导致难以预料的结果。

**[嵌套变量名的拓展用法]** <br>
一个嵌套变量名可以包含多个变量的引用，也可以包含一些文本字符串，即嵌套变量名可以由多个变量的引用+字符串的混合构成。

**[案例1：嵌套变量中包含多个变量的引用]** <br>
```txt
a_dirs := dira dirb
1_dirs := dir1 dir2

a_files := filea fileb
1_files := file1 file2

# 如果使用a为前缀
ifeq "$(use_a)" "yes"
    a1 := a
# 否则使用1为前缀
else
    a1 := 1
endif

# 如果使用dirs为后缀
ifeq "$(use_dirs)" "yes"
    df := dirs
# 否则使用files为后缀
else
    df := files
endif

dirs := $($(a1)_$(df))
```
上述案例中的变量dirs是一个嵌套变量名，这个变量名由两个独立的变量引用构成。这个变量的取值为"a_dirs"、"1_dirs"、"a_files"、"1_files"四者之一，具体依赖于"use_a"、"use_dirs"的定义。

**[案例2：嵌套变量定义和变量替换引用结合使用]** <br>
```txt
a_objects := a.o b.o c.o
1_objects := 1.o 2.o 3.o

sources := $($(a1)_objects:.o=.c)
```
上述案例定义的嵌套变量sources的可能取值为"a.o b.o c.o"或"1.o 2.o 3.o"，具体情况取决于变量a1的定义。

**[嵌套变量引用时的限制]** <br>
嵌套变量引用的限制是：不能通过将需要调用的函数名设置为嵌套变量的引用，来调用函数返回相应处理结果。这是因为：在make对嵌套调用变量的引用展开前就已经完成了对函数名的识别和测试。参考下面的案例进行分析理解：
```txt
ifdef do_sort
    func := sort
else
    func := strip
endif

bar := a d b g q c

foo := $($(func) $(bar))
```
这个Makefile的本意是想：通过判断是否定义了"do_sort"变量从而决定使用sort函数或者strip函数处理字符串"a d b g q c"，并将处理的结果返回给变量foo。但是本例的结果是：foo变量的取值为"sort a d b g q c"或者"strip a d g q c"。这是当前make版本处理嵌套变量作为函数名的限制。

**[嵌套引用定义的变量应用范围]** <br>
➀ 一个使用赋值操作符定义的变量的左值部分 <br>
➁ 使用define定义的变量名中(关于define的用法可以参考GNU make定义命令包相关部分内容)<br>

参考下例进行理解：
```txt
dir = foo

# 使用"="定义foo_sources变量
$(dir)_sources := $(wildcard $(dir)/*.c)

# 使用"define"定义foo_print命令包 下面两行分别是定义的变量名 以及lpr(按行打印shell命令)
define $(dir)_print 
lpr $($(dir)_sources)
endef
```
在这个案例中定义了三个变量："dir"、"foo_sources"、"foo_print"。嵌套的变量在被引用处的展开方式是❮从内而外❯展开的，"lpr $($(dir)\_sources)"首先执行内层展开得到"lpr $(foo\_sources)"，再执行最外层展开得到"lpr $(wildcard $(dir)/\*.c)"；即foo_print变量被定义成了一个用于执行"按行打印(line print)"符合"$(dir)/\*.c"模式的文件名。

关于linux lpr(按行打印)命令可以参考[microsoft lpr doc](https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/lpr)；make中wildcard内置函数的用法可以参考GNU make 3rd zh 8.38部分。

**[关于如何正确地使用多层变量嵌套#2]** <br>
变量的嵌套引用在Makefile中应当尽量避免使用。在必须使用嵌套引用的场合使用的原则是：嵌套的层数越少越好，尽量使用多个两层嵌套变量引用代替一个多层的嵌套引用。<br>
在Makefile中出现的多层嵌套的变量引用会导致阅读困难，并且有可能会导致简单问题复杂化。

题外话：作为一个优秀的程序员，在面对一个复杂的问题时，应该寻求一种尽可能简单的、直接、高效的处理方式来解决，而不是将简单问题复杂化。试图在简单问题上凸显自己对于某种语言熟练度的做法是一种愚蠢、不成熟的表现。


### 四：Makefile变量值的获取
一个Makefile变量的值可以通过如下几种方式获得：
- ➀ 在运行make时通过❮命令行选项❯来取代一个已经定义的变量值。
- ➁ 在makefile文件中通过❮赋值的方式❯or使用❮define❯来为一个变量赋值。
- ➂ 将变量设置为❮系统变量❯。所有系统环境变量都可以被make使用。
- ➃ ❮Makefile中的自动化变量❯，在不同的规则下被自动地赋予不同的数值。每个自动化变量都有各自的习惯性用法。
- ➄ ❮具有固定值的特定的变量❯。


### 五：Makefile变量值的设定
Makefile变量值的设置是通过使用"="(递归方式)、":="(静态方式)来实现的。在定义一个变量时，需要明确如下几点：
- ➀ 变量名中可以包含函数or其他变量的引用，make在读入此行时根据变量定义的方式来解引用以得到实际的变量名。
- ➁ 变量的定义值在长度上没有限制，但是在实际使用时，还是需要根据实际情况选择一个合理的变量值。当变量定义较长时，较好的做法是：将其分成多个行进行书写，除最后一行外，行与行之间需要使用反斜杠("\\")连接。这种处理方式可以使得Makefile更加清晰。
- ➂ 当引用一个未定义的变量时，make默认该变量为空。
- ➃ 一些特殊的变量在make中有固定内嵌值，不过这些变量允许我们在Makefile中显式重新为其赋值。
- ➄ 一些由两个字符构成的变量称为自动化变量，它们的值不能在Makefile中显式地进行修改，它们的值取决于实际应用的规则。
- ➅ 如果希望对一个之前尚未定义的变量赋值，则可以使用条件赋值操作符"?="来完成。


### 六：追加变量值
Makefile中定义的变量可以在其定义之后的其他位置，对其取值进行追加。我们也可以在定义时直接使用追加操作符"+="来赋予它一个基本值，如下所示：

**[#1 追加操作符改变变量值]** <br>
```txt
objects += another.o
```
追加操作将字符串"another.o"追加到object变量原有值的末尾，追加值和原有值之间使用空格分开，即有：
```txt
objects = main.o foo.o bar.o utils.o
objects += another.o
```

**[#2 普通定义方式]** <br>
则经过追加后的变量objects的值为"main.o foo.o bar.o utils.o another.o"。使用追加操作符定义的objects变量等同于如下的操作产生的结果：
```txt
objects = main.o foo.o bar.o utils.o
objects := $(objects) + another.o
```
需要注意的是：上述普通定义方式和"+="定义变量方式在一些简单的Makefile中可能有相同的效果，但在复杂的Makefile中它们之间的差异可能会导致一些问题。为了避免差异引发的问题，方便调试，需要了解两种方式之间的差异。

**[追加操作符和普通定义方式之间的区别]** <br>
- ➀ 如果被追加值的变量之前处于未定义状态，那么，"+="会自动变成"="，则该变量的定义即变成递归展开式的变量。如果之前存在这个变量的定义，那么"+="就继承之前定义时的变量风格(直接展开or递归展开)。
- ➁ 直接展开式变量的追加过程如下：
```txt
variable := value
variable += more
```
实际上发生了如下操作：
```txt
variable := value
variable := $(variable) more
```
最终variable变量被定义成"variable more"。
- ➂ 递归展开式变量的追加过程如下：
```txt
variable = value
variable += more
```
❮相当于❯发生了如下操作，实际上并没有产生中间变量temp，使用它只是为了方便描述递归展开式变量追加值的机制：
```txt
temp = value
variable = $(temp) value
```
- ➃ 关于直接展开式和递归展开式两种方式定义的变量的追加过程的区别，使用如下例子进行解析：
➃-➀ 递归展开式变量的追加操作：
```txt
# 递归展开式定义的变量CFLAGS的追加过程
CFLAGS = $(includes) -O
CFLAGS += -pg
```
等价于：
```txt
temp = $(includes) -O
CFLAGS = $(temp) -pg
# =>
CFLAGS = $(includes) -O -pg
```
上述案例定义了递归式展开变量CFLAGS，$(includes)将只在CFLAGS变量被引用时才被展开，由于CFLAGS为递归展开式定义，因此includes变量只需定义在CFLAGS变量被引用之前就可以了。<br>
➃-➁ 直接展开式变量的追加操作：
```txt
# 直接展开式定义的变量CFLAGS的追加过程
CFLAGS := $(includes) -O
CFLAGS += -pg
```
等价于：
```txt
CFLAGS := $(includes) -O
CFLAGS := $(CFLAGS) -pg
```
注意，这种形式和直接展开式变量追加操作之间的区别：如果includes变量在CFLAGS追加操作之前未被定义，则默认展开为空字符，CFLAGS变量的数值为" -O -pg"，后续再出现的变量includes的定义不被对CFLAGS的数值起作用。


### 七：override指示符
对于一个在Makefile中使用常规方式("="、":="、"define")定义的变量，可以在执行make时通过命令行方式重新指定这个变量的值，命令行指定的值将会替代出现在Makefile中的变量的值。如果不希望命令行指定的变量值替代Makefile中相关变量的定义，则我们需要在Makefile中使用指示符"override"对这个变量进行声明，如下所示：
```txt
override VARIABLE = VALUE
```
或者：
```txt
override VARIABLE := VALUE
```
以及使用对变量的追加方式进行声明：
```txt
override VARIABLE += MORE TEXT
```
需要注意的是：如果在变量定义时使用了"override"，则在后续对它的值进行追加时，也需要使用带有"override"指示符的追加方式，否则对此变量的追加不生效。

但是，需要注意：指示符"override"并不是用来调整Makefile和执行时命令行参数之间的冲突，其存在的目的是为了使用户可以改变、追加哪些通过Makefile命令行指定的变量的定义；即"override"是用来在Makefile中增加、修改命令行参数的机制。<br>
因此，可以通过命令行来指定一些❮附加❯的、❮特殊❯的编译参数，对一些❮通用❯的编译参数、❮必须❯的编译参数则需要在Makefile中进行指定。

**[案例1]** <br>
无论命令行指定哪些编译参数，编译时必须打开"-g"选项，因此应该将"-g"选项在Makefile中通过"override"指示符进行指定。
```txt
override CFLAGS += -g
```
这样，在执行make时，无论在命令行中指定了哪些编译选项，编译时"-g"参数总是存在。

**[案例2]** <br>
在Makefile中使用define定义变量时同样可以使用"override"进行声明。例如：
```txt
override define foo
bar
endef
```

**[案例3]** <br>
给定如下的Makefile样例：
```txt
# Sample Makefile
# EXEF变量迭代展开式定义
EXEF = foo

# 通过override指示符为编译选项CFLAGS添加字
override CFLAGS += -Wall -g

# 使用伪目标的方式定义三个独立调用目标
.PHONY : all debug test
all : $(EXEF)

foo : foo.c

......

$(EXEF) : debug.h
    $(CC) $(CFLAGS) $(addsuffix .c,$@) -o $@

debug :
    @echo "CFLAGS = $(CFLAGS)"
```
在shell中执行"make CFLAGS=-O2"指令，实际上将执行的编译过程为"cc -O2 -Wall -g foo.c -o foo"。上述编译选项CFLAGS的值可以通过执行"make CFLAGS=-O2 debug"指令来进行验证。


## Reference
> https://blog.csdn.net/haoel/article/details/2892 <br>
> GNU make zh chapter 6.1 & 6.2

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
