---
layout: post
title: "makefile - #12"
subtitle: '[跟我一起写makefile - 陈皓]' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-12-29 15:31
lang: ch 
catalog: true 
categories: makefile
tags:
  - Time 2021
---
## make的隐式规则(第一篇)
**隐式规则的应用场景** 在Makefile中重编译目标文件所需要的标准流程化的规则在很多场合都会用到。例如：根据.c源文件创建对应的.o文件；一种较为传统的方式是借助GNU c compiler根据源文件(.c)创建目标文件(.o)。

**隐式规则的作用** "隐式规则"机制为make提供了一种重编译某类型目标文件的❮隐式❯方法，从而不需要在Makefile中❮显式❯给出重编译特定目标文件所需的细节规则描述。例如：隐式规则可以自动地将后缀名为.c的源文件编译成后缀为.o的目标文件。

**make根据实际需要可支持同时应用多个隐式规则** 例如：make可以先将一个.y后缀的文件生成对应的.c文件，然后再次应用隐式规则将.c后缀文件生成最终的.o文件。<br>
即只要目标文件名中除了后缀名不同，其他部分都相同，则make可以使用若干隐式规则来最终生成这个文件(只要原始.y文件存在)。

**隐式规则执行的控制** 内嵌的隐式规则在在相关的规则定义中会用到一些变量(通常为内嵌变量)，因此可以通过修改这些变量的定义来控制隐式规则命令执行的情况。例如：内嵌变量CFLAGS表示了gcc编译器编译源文件的编译选项，我们可以通过重定义CFLAGS以改变编译源文件所使用的选项参数。

**使用模式规则、后缀规则来定义隐式规则** 尽管不能直接改变make内嵌的隐式规则，但是可以使用模式规则(pattern rules)来重新定义自己的隐式规则。后缀规则(suffix rules)是定义隐式规则的另一种more limited的方式。模式规则更加通用、清晰，但是为了兼容性保留了后缀规则。


### 一：使用隐式规则
##### 1 使用隐式规则的简单案例
使用make的隐式规则后，在Makefile中就不需要显式给出重编译某一目标的命令，甚至可以省略生成目标的规则；make会自动根据可被找到的源文件、可被创建的源文件的类型来启动相应隐式规则，例如：
```txt
foo : foo.o bar.o
    cc -o foo foo.o bar.o $(CFLAGS) $(LDFLAGS)
```
注意，上述Makefile中并未给出重建生成foo.o的规则。当make执行上述规则对应命令时，❮无论foo.o是否存在❯，都会试图通过隐式规则来重建该目标文件(即重编译隐式规则推测的对应源文件foo.c)。

##### 2 隐式规则相关概念、用法
具体地，make在执行过程中找到的隐式规则将提供重建目标文件的依赖关系；这个依赖关系确定了目标文件的依赖文件(通常是源文件，不包括对应头文件的依赖文件)和重建目标文件需要使用的具体指令。<br>
隐式规则默认提供最基本的依赖文件形式：EXENAME⟺EXENAME.o，EXENAME.o⟺EXENAME.c。当需要为特定目标文件增加依赖文件时，需要在Makefile中使用不含命令行的规则指定。

**隐式规则的目标模式和依赖模式** 每个内嵌的隐式规则都具有一个目标模式(target pattern)以及依赖模式(prerequisite patterns)。相同的目标模式可以由多种依赖模式来构建。例如：一个.o文件的目标可由c编译器编译对应的.c文件得到，或由Pascal编译器编译对应的.p文件得到, etc. make会自动根据不同的源文件使用不同的编译器(make兼容各种依赖模式的构建)。

**make创建隐式规则的依据** make会自动根据已经存在、或可以被创建的源文件类型来启动相应的隐式规则。可被创建的含义是：这个源文件在Makefile中被作为目标or依赖被明确的提及，或者可以根据已存在的文件借助其他的隐式规则来创建它。当一个隐式规则的目标文件是另一个隐式规则的依赖时，称之为一个隐式规则链。

**什么情况下make会创建隐式规则** 通常情况下，make会对不带有命令行的规则、双冒号规则寻找一个隐式规则来执行。即作为一个规则的依赖文件，在没有一个规则明确描述它的依赖关系时，make会将其作为一个目标文件并为其找到一个对应的隐式规则，并尝试重建该文件。

**隐式规则搜索具有一定优先级，给目标文件指定明确的依赖文件不会影响隐式规则的搜索** <br>
参考下面的例子进行理解：
```txt
# ------ 该规则指定了foo的依赖文件是foo.p ------
foo.o : foo.p
```
如果在make的工作目录下存在同名.c源文件foo.c，则执行make的结果就不是用pc编译foo.p来生成foo，而是用cc编译foo.c来生成目标文件。产生这种结果的根本原因是：隐式规则优先级列表中.c文件的隐式规则位于.p文件之前。<br>
此时，如果希望在存在.c依赖文件的情况下，编译foo.p生成foo，则规则描述中的命令行不能省略(不能通过隐式规则来构建)：
```txt
# ------ 该规则通过显示规则的方式"强制"使用pc编译foo.p------
foo.o : foo.p
    pc $< -o $@
```
其中，"$@"表示目标文件，"$<"表示第一个依赖文件。在多语言实现的工程编译时需要特别注意，否则可能编译出的可能不是你想要的程序格式。

**一个应用隐式规则的例子** 
```txt
# ------ 变量定义 ------
CUR_DIR = $(shell pwd)
INCS := $(CUR_DIR)/include
CFLAGS := -Wall - I$(INCS)

EXEF := foo bar

# ------ 伪目标定义 ------
.PHONY : all clean
all : $(EXEF)

# ------ 不带命令行的规则定义 ------
foo : CFLAGS+=O2
bar : CFLAGS+=-g

clean :
    $(RM) *.o *.d $(EXEF)
```
这个例子中没有出现任何关于源文件的描述，使用哪个源文件、哪种类型的源文件，根据什么构建规则生成哪种目标文件都交给make去处理，make将自动找到相应规则进行执行，并最终完成目标文件的构建。

**make的隐式规则总结** <br>
隐式规则为我们提供了一个编译整个工程非常高效的手段，一个大的工程毫无例外的会用到隐式规则。实际工作中，灵活运用GNU make提供的隐式规则功能，可以大大提高工程构建效率。


### 二：make的隐式规则一览(catalogue of build-in rules)
这部分内容将列举出GNU make中常见的内置隐式则。除非在Makefile中对于目标的生成规则由明确定义，或者使用命令行参数-r、-R来禁用内置隐式规则，否则这些隐式规则默认有效。

##### 1 make隐式规则和后缀规则之间的关系
即使make在执行时没有指定-r、-R选项禁用隐式规则，也并不是所有可能的隐式规则都被定义。预定义的隐式规则在make中是通过后缀规则来实现的，因此，make中哪些隐式规则被定义取决于后缀列表(the list of prerequisites of the special target .SUFFIX)。下面将介绍的隐式规则如果其依赖文件中的某项满足列表中列出的后缀，则实际上为后缀规则。

##### 2 默认后缀列表
.out、.a、.ln、.o、.c、.cc、.C、.cpp、.p、.f、.F、.m、.r、.y、.l、.ym、.lm、.s、.S、.mod、.sym、
.def、.h、.info、.dvi、.tex、.texinfo、.texi、.txinfo、.w、.ch、.web、.sh、.elc、.el。

##### 3 修改默认后缀列表
如果修改了默认后缀列表，则唯一有效(enabled)的预定义后缀规则为你指定的列表中的后缀规则，所有其他默认后缀规则将被禁用(disabled)。

##### 4 常用的隐式规则
对于不常用的隐式规则没有描述，可以通过查阅文档或通过"make -q"命令来查看。
- ➀ **编译c程序** "xxx.o"自动由"xxx.c"生成，执行命令为：
```txt 
$(CC) -c $(CPPFLAGS) $(CFLAGS)
```
- ➁ **编译c++程序** "xxx.o"自动由"xxx.cc"或"xxx.C"生成，执行命令为：
```txt
$(CXX) -c $(CPPFLAGS) $(CXXFLAGS)
```
建议使用".cc"作为c++文件的后缀，而不是".C"。
- ➂ **编译Pascal程序** "xxx.o"自动由"xxx.p"生成，执行命令为：
```txt
$(PC) -c $(PFLAGS)
```
- ➃ **编译Fortran/Ratfor程序** "xxx.o"自动由"xxx.r"或"xxx.F"或"xxx.f"生成，根据后缀执行对应命令：
```txt
# --- .f ---
$(FC) -c $(FFLAGS)
# --- .F ---
$(FC) -c $(FFLAGS) $(CPPFLAGS)
# --- .r ---
$(FC) -c $(FFLAGS) $(RFLAGS)
```
- ➄ **预处理Fortran/Ratfor程序** "xxx.f"自动由"xxx.r"或"xxx.F"生成，根据后缀执行对应的命令：
```txt
# --- .F ---
$(FC) -F $(CPPFLAGS) $(FFLAGS)
# --- .r ---
$(FC) -F $(FFLAGS) $(RFLAGS)
```
注意，此规则只用于转换Ratfor或有预处理的Fortran程序到一个标准的Fortran程序。
- ➅ **编译Modula-2程序** "xxx.sym"自动由"xxx.def"生成，执行命令为：
```txt
$(M2C) $(M2FLAGS) $(DEFFLAGS)
```
"xxx.o"自动由"xxx.mod"生成，执行命令为：
```txt
$(M2C) $(M2FLAGS) $(MODFLAGS)
```
- ➆ **汇编和需要预处理的汇编程序** "xxx.s"是不需要预处理的汇编源文件，"xxx.S"是需要预处理的汇编源文件，汇编器为as。"xxx.o"自动由"xxx.s"生成，执行命令为：
```txt
$(AS) $(ASFLAGS)
```
"xxx.s"自动由"xxx.S"生成，使用c预编译器cpp，执行命令为：
```txt
$(CPP) $(CPPFLAGS)
```
- ➇ **链接单一的object文件** "xxx"自动由"xxx.o"生成，通过c编译器来运行链接器(通常称为GNU ld)，执行命令为：
```txt
$(CC) $(LDFLAGS) xxx.o $(LOADLIBES) $(LDLIBS)
```
注意，此规则只用于：只需一个源文件来生成可执行文件的情况。当需要多个源文件来生成可执行文件时，则需要在Makefile中增加隐式规则的依赖文件：
```txt
# --- 表明目标文件x不止需要x.o来合成 还需要y.o z.o来合成 ---
x : y.o z.o
```
当工作目录中x.c、y.c、z.c都存在时，规则执行如下命令：
```txt
# --- 从源文件xxx.c生成对应的中间文件xxx.o ---
cc -c x.c -o x.o
cc -c y.c -o y.o
cc -c z.c -o z.o
# --- 从中间文件xxx.o生成目标文件x ---
cc x.o y.o z.o -o x
# --- 清理中间文件 ---
rm -f x.o
rm -f y.o
rm -f z.o
```
某些复杂场景下，目标文件和源文件之间可能不存在上例所示的名称对应关系("xxx"和"xxx.c"对应，隐式规则自动将"xxx.c"作为"xxx"的依赖文件)，此时需要在Makefile中明确给出描述目标依赖关系的规则。<br>
通常，在gcc编译源文件时，根据源文件的后缀名来启动对应的编译器，如果编译命令未指定-c选项，则gcc在编译完成后默认调用ld链接器将文件链接成可执行文件。
- ➈ **Yacc C程序** "xxx.c"自动由"xxx.y"生成，执行命令为：
```txt
# --- yacc是一个语法分析工具 ---
$(YACC) $(YFLAGS)
```
- ➀⓪ **Lex for C programs的隐式规则** "xxx.c"自动由"xxx.l"运行Lex程序生成，执行命令为：
```txt
# --- 关于lex细节见相关资料 ---
$(LEX) $(LFLAGS)
```
- ➀➀ **Lex for Ratfor programs** "xxx.r"自动由"xxx.l"运行Lex程序生成，执行命令为：
```txt
# --- 关于lex细节见相关资料 --- 
$(LEX) $(LFLAGS)
```
需要注意的是：无论这些Lex文件将生成C代码还是Ratfor代码，对于所有的Lex文件统一使用".l"后缀的惯例(convention)，此时在任何场景下让make来确定使用哪种语言的编译器来完成编译是不太可能的(make无法根据"统一的".l文件后缀来猜测使用哪个编译器)。通常情况下，make默认采用c编译器，因为这能够满足大多数情况。如果实际工程中使用Ratfor，则应该在Makefile的依赖关系中明确提到"xxx.r"，或者如果工程不包含c，则可以从后缀规则列表中将.c删除，如下：
```txt
# --- 清空后缀列表 ---
.SUFFIXES :
# --- 将后缀列表中按顺序手动添加所需项 ---
.SUFFIXES : .o .r .f .l ...
```
- ➀➁ **Making Lint Libraries from C, Yacc, Lex programs**
- ➀➂ **TEX and Web** see GNU make en p118.
- ➀➃ **Texinfo and info** see GNU make en p119.
- ➀➄ **RCS** see GNU make en p119.
- ➀➅ **SCCS** see GNU make en p119.

##### 5 关于make隐式规则生成的补充说明
➤ 实际上，在隐式规则中，命令行中的实际命令是根据相关的变量的值得到(e.g. COMPILE.c、LINK.o、PREPROCESS.S等)，这些变量被展开后再加上命令行选项(e.g. CFLAGS)就是对应的命令。例如，变量COMPILE.c的定义为"cc -c"，如果Makefile中包含了CFLAGS变量的定义，则变量COMPILE.c的值为"cc -c $(CFLAGS)"。

➤ 另外，make会根据默认的约定，使用"compiler_name.xxx"a来编译生成".xxx"后缀的文件；类似地，使用"linker_name.xxx"来链接".xxx"后缀文件；使用"preprocess_name.xxx"来对".xxx"文件进行预处理。

➤ 需要补充的是，make中每个隐式规则的创建都调用了变量"OUTPUT_OPTION"，make执行隐式规则命令时根据命令行参数来决定其数值。当命令行不包含-o选项时，OUTPUT_OPTION的=-o $@，否则OUTPUT_OPTION值为空。<br>
建议在规则命令行中显式使用-o选项指定输出文件路径。当需要源文件和输出文件组织在不同目录下时，使用-o选项可以保证输出文件能出现在正确的路径下。<br>
然而，一些特定的系统上的编译器不支持-o选项，如果在此类系统中使用VPATH变量，编译的结果可能出现在错误的路径下。解决该问题的方法是：设置OUTPUT_OPTION=; mv $\*.o $@，即将编译生成的文件(名&路径)修正到当前目标文件(名&路径)。


### 三：隐式变量(variables used by implicit rules)
内嵌隐式规则的命令中，使用的变量都是预定义变量，称为隐式变量。

##### 1 对make的隐式变量进行修改
在Makefile中，通过命令行参数or通过设置系统环境变量的方式对其进行重定义。只要make运行时这些变量定义有效，隐式规则将会自动使用这些变量。当然，可以使用-R、\-\-no-builtin-variables选项来取消所有隐式变量。

例如，编译.c后缀源文件的隐含规则为：$(CC) -c $(CFLAGS) $(CPPFLAGS)，默认的编译命令为cc，则执行的命令为cc -c。我们可以通过命令行参数、设置系统环境变量的方式将变量CC定义为ncc，则此时执行的命令为ncc -c。同理我们可以对变量CFLAGS进行重定义。如果希望对于这些变量的重定义对整个工程各个子目录有效，则需要使用关键字export，否则不同目录下的编译命令可能不一致。

##### 2 隐式变量的分类
用于隐式规则的变量可以分成2类：1) 代表了一类程序的名字，如$(CC)表示编译器的可执行程序；2) 代表了一类程序的执行参数，如$(CFLAGS)表示编译选项，注意多个参数之间使用空格分隔。允许程序名中包含参数，但不推荐使用。

##### 3 表示命令的变量(variables used as naems of programs)
- ➀ **AR** 函数库打包程序，可创建静态库.a格式文档文件。默认值为"ar"。
- ➁ **AS** 汇编程序，默认值为"as"。
- ➂ **CC** C编译程序，默认值为"cc"。
- ➃ **CXX** C++编译程序，默认值为"g++"。
- ➄ **CO** 从RCS中提取文件的程序，默认值为"co"。
- ➅ **CPP** 运行C程序的预处理器的程序，结果将被输出到标准输入输出上。默认值为"$(CC) -E"。
- ➆ **FC** 编译、预处理Fortran和Ratfor的程序，默认值为"f77"。
- ➇ **GET** 从SCCS中提取文件程序。默认值为"get"。
- ➈ **LEX** 将Lex语言转换成C或Ratfor程序，默认值为"lex"。
- ➀⓪ **PC** Pascal语言编译器，默认值为"pc"。
- ➀➀ **YACC** Yacc语法分析程序(针对C程序)，默认值为"yacc"。
- ➀➁ **YACCR** Yacc语法分析程序(针对Ratfor程序)，默认值为"yacc -r"。
- ➀➂ **MAKEINFO** 将Textinfo源文件(.texi)转化成Info文件的程序，默认值为"makeinfo"。
- ➀➃ **TEX** 根据Tex源文件创建Tex DVI文件的程序，默认值为"tex"。
- ➀➄ **TEX2DVI** 根据Texinfo源文件创建Tex DVI文件的程序，默认值为"texi2dvi"。
- ➀➅ **WEAVE** 将Web转化成Tex的程序，默认值为"weave"。
- ➀➆ **CWEAVE** 将CWeb转化成Tex的程序，默认值为"cweave"。
- ➀➇ **TANGLE** 将Web转化成Pascal的程序，默认值为"tangle"。
- ➀➈ **CTANGLE** 将CWeb转化成C的程序，默认值为"ctangle"。
- ➁⓪ **RM** 删除命令，默认值为"rm -f"。


##### 4 表示命令参数的变量(variables used as additional arguments of the programs)
- ➀ **ARFLAGS** 执行AR命令时对应的命令行选项，默认值为"rv"。
- ➁ **ASFLAGS** 执行汇编程序时对应的命令行选项(命令明确指定.s、.S后缀的文件)，默认值为空。
- ➂ **CFLAGS** 执行CC编译器程序时对应的命令行选项(编译.c文件的选项)，默认值为空。
- ➃ **CXXFLAGS** 执行g++编译器时对应的命令行选项(编译.cc文件的选项)，默认值为空。
- ➄ **COFLAGS** 执行co时的命令行选项(用于在RCS中提取文件的选项)，默认值为空。
- ➅ **CPPFLAGS** 执行c预处理器"cc -E"的命令行参数(C、Fortran编译时会用到)，默认值为空。
- ➆ **FFLAGS** 编译Fortran语言时编译器f77执行的命令行参数(编译Fortran源文件的选项)，默认值为空。
- ➇ **GFLAGS** SCCS "get"程序的参数，默认值为空。
- ➈ **LDFLAGS** 链接器(e.g. ld)的参数，默认值为空。
- ➀⓪ **LFLAGS** Lex语法分析器参数，默认值为空。
- ➀➀ **PFLAGS** Pascal语言编译器的参数，默认值为空。
- ➀➁ **RFLAGS** Ratfor语言的Fortran编译器的参数，默认值为空。
- ➀➂ **YFLAGS** Yacc文法分析器的参数。


### 四：make的隐式规则链
有些情况下，一个目标文件需要多个隐式规则来创建。

##### 1 案例分析
例如，创建文件"xxx.o"的过程可能是：首先执行yacc指令将文件"xxx.y"生成"xxx.c"，然后再由编译器将"xxx.c"编译成"xxx.o"。当一个目标文件需要多个隐式规则来创建，则称为隐式规则链。上述案例的执行过程可以分成2种情况：
- ➀ 如果"xxx.c"存在，或者它在Makefile中被提及(它不能被其他依赖关系创建)，则不需要进行其他搜索。make的处理过程为：首先make确定"xxx.o"文件可以由"xxx.c"创建，然后make尝试通过隐式规则来创建"xxx.c"，将导致make需要寻找"xxx.y"文件：假如"xxx.y"文件存在，则执行隐式规则来重建"xxx.c"并重建"xxx.o"；当"xxx.y"文件不存在时，直接编译"xxx.c"生成"xxx.o"。
- ➁ 如果"xxx.c"不存在，也没有在Makefile中被提及(它不能被其他依赖关系创建)，当存在"xxx.y"文件时，make将通过"xxx.y"=\>"xxx.c"=\>"xxx.o"的过程来创建"xxx.o"。这种情况下，"xxx.c"文件相当于一个中间文件，在make执行规则时，当需要中间文件才能完成目标的创建时，则该中间文件会自动被加入依赖关系链中(和Makefile中显式指出的依赖文件作相同处理)。

##### 2 生成目标的中间文件和Makefile中明确指定依赖文件之间的区别联系
make构建目标的中间过程文件和Makefile在合成目标过程中明确指定的文件具有相同的"地位"，但make针对两者的处理有一定的差异性：
- ➀ 当中间文件不存在时，make处理两者的方式不同。对于Makefile中明确提到的普通依赖文件而言，此文件由于作为某个规则的依赖，因此make在执行该文件所在的规则时，会尝试重建该文件。但是，对于make的中间文件，由于没有显式提及，make不会主动重建该文件，当且仅当该中间文件所依赖的文件("xxx.y")被更新时，该中间文件("xxx.c")才被更新。
- ➁ 如果make在执行时需要用到中间文件，则在make执行结束后，默认将该中间文件删除(make会在删除中间文件时打印执行的命令以显示哪些文件被删除)。默认情况下，Makefile中所有明确提及的文件都不会作为中间文件来处理，但是，我们可以在Makefile中特殊目标".INTERMEDIATE"来指出哪些文件作为中间文件来处理(即使这些文件在Makefile规则中被显式指出)；因此这些被显式标记的中间文件在make结束后同样被自动删除。<br>
另外，我们同样可以将中间文件进行保留(即使它们没有在Makefile中明确被提及)，在make执行结束时它们不会被删除，2种方式分别如下：<br>
    - ➁-➀ 可以在Makefile中使用特殊目标".SECONDARY"来指出需要被保留的文件(需要保留的目标将作为".SECONDARY"文件的依赖文件罗列)；这些保留的文件称为secondary文件。secondary文件都被作为中间文件来对待。<br>
    - ➁-➁ 以需要保留所有.o文件的依赖中间文件，我们可以将所有.o文件的模式"%.o"作为make特殊目标".PRECIOUS"的依赖。

##### 3 构建隐式规则链的注意事项
一个隐式规则链需要至少包含2个隐式规则调用，另外，需要注意：同一个隐式规则在一个链中只能被调用一次。

采用反证法：假设一个隐式规则链中调用2次同一规则，将会出现文件foo依赖于foo.o，文件foo.o依赖于foo.o.o的不合逻辑的情况产生。

在make查询隐式规则链时，此限制将能够避免搜寻陷入无限循环中(this constraint has the added benefit of preventing any infinite loop in the search for an implicit rule chain)。

##### 4 make的隐式规则链可以被优化
隐式规则链中的某些隐含规则，在一定情况下会被优化处理。

例如，从文件foo.c创建可执行文件foo，该过程可以按照如下隐式规则链的方式完成：首先根据隐式规则将文件foo.c编译成foo.o，然后再应用另一个隐式规则将foo.o文件链接成可执行文件foo。<br>
实际上，make对于foo文件的生成是通过一条被优化的规则来完成的，该规则使用"cc foo.c foo"命令完成了foo文件的生成。

注意，make的隐式规则表中，在应用隐式规则时，所有可用的优化规则处于首选位置。


## Reference
> https://blog.csdn.net/haoel/article/details/2897 <br>
> GNU make zh/en chapter 10

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
