---
layout: post
title: "makefile - #5"
subtitle: '[跟我一起写makefile - 陈皓]' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-09-09 10:21
lang: ch 
catalog: true 
categories: makefile
tags:
  - Time 2021
---
## 书写规则(书写依赖关系) - 第二篇
makefile的书写规则包含两个部分：依赖关系、生成目标的方法。<br>
另外makefile文件中，各个依赖关系的顺序很重要，因为makefile只有一个终极目标，其他目标都可以被看作是该目标的附属产品。因此必须让make命令解析出该makefile的终极目标，而终极目标是由依赖关系的书写顺序决定的。<br>
一般而言，定义在一个makefile中的依赖关系有很多，但是第一个依赖关系中的目标为该makefile的终极目标。如果第一个依赖关系中存在多个目标，那么目标序列中的第一个目标为最终目标。

### 六：多个目标文件
单个makefile规则中可以存在多个目标，规则定义的命令对于当前规则的所有目标都有效。具有多个目标的规则相当于同时平行定义多个规则。<br>
多个目标文件可以大体分成2种情况：单一规则中的多个目标对应完全相同的命令、单一规则中的多个目标分别对应根据目标文件而变化的命令。

##### 1 单一规则中多个目标对应完全相同的命令
下面的规则中，3个目标文件都依赖于同一个文件，则不需要显式定义命令，依赖make命令自动推导即可。
```txt
kbd.o command.o files.o : command.h
```
##### 2 单一规则中多个目标对应根据目标文件而变化的命令
首先，先了解2个关于make相关的基础命令用法："$@"表示：当前规则下所有的目标文件集合组成的字符串；"subst"表示：make命令的一个函数，定义为"$(subst, FROM, TO, TEXT)"，将TEXT中的FROM字符部分替换成TO字符。
```txt
bigoutput littleoutput : text.g
        generate text.g -$(subst, output, , $@) > $@
```
上述"-$(...)"部分表示，generate命令根据命令行的参数来决定生成的"$@"的结果，即对于bitoutput目标文件，采用"generate text.g -big"命令产生输出，对于littleoutput目标文件，采用"generate text.g -little"命令产生输出。

上述makefile规则等价于两个独立的目标文件生成规则：
```txt
bigoutput : text.g
        generate text.g -big > $@ 
littleoutput : text.g
        generate text.g -little > $@ 
```
注意，"$@"永远表示当前规则下的目标文件，因此可以将上面的makefile进一步进行改写为：
```txt
bigoutput : text.g
        generate text.g -big > bigoutput 
littleoutput : text.g
        generate text.g -little > littleoutput 
```
##### 3 关于makefile多目标的补充内容
虽然makefile多目标可以实现：对于同一规则下的多个目标文件，分别采用不同的命令版本以得到相应的目标，但是，这种makefile书写技巧并不能实现：对于同一规则下的多个目标文件，分别构建出不同的依赖文件项。<br>
如果希望实现针对同规则下，针对不同目标文件，构建不同的依赖项，需要借助\<make的静态模式\>。\<make的静态模式\>：规则存在多个目标，并且不同的目标可以根据目标文件的名字来自动构造出依赖文件。


### 七：make的静态模式
静态模式是这样一个规则：规则可以存在多个目标文件，并且不同的目标可以根据目标文件名字的不同自动构造出各自的依赖文件。静态模式相比前一部分介绍的多目标模式更加通用，因为它不需要完全相同的依赖文件。但是静态模式也有一定应用局限性：它要求依赖文件必须是在一定规则下具有相似性的。

##### 1 静态模式makefile规则书写语法
```txt
<targets> : <target-pattern> : <prerequisites-patterns...>
        <commands>
        ...
```
- \<targets\>定义了一系列目标文件(集合)，可以包含通配符。
- \<target-pattern\>指明了targets的模式，即目标文件集合中文件名的通用模式。
- \<prerequisites-patterns\>指明了prerequisites的模式，即依赖文件集合中的文件名通用模式。

##### 2 结合一个简单的例子对静态模式书写语法进行分析
下面makefile案例涉及的符号表示："$<"：表示该规则的第一个依赖文件；"$@"：表示该规则的所有目标文件。
```txt
objects = foo.o bar.o
all : $(objects)
$(objects) : %.o : %.c
        $(CC) -c $(CFLAGS) $< -o $@
```
上述makefile文件中，\<targets\>为"$(objects)"表示的所有文件；\<target-pattern\>为"%.o"：所有以".o"后缀结尾的文件，对于所有目标文件的模式进行了**二次规范**；\<prerequisites-pattern\>为"%.c"：所有以".c"后缀结尾的文件。<br>
额外需要注意的是：对于同一个规则的同一行命令中：通配符"%"所表示的文件集合总是相同的。

**[1] "%"含义推断** <br>
"$(objects)"=foo.o bar.o，目标文件集合规范为以".o"后缀结尾的文件，因此可以推断上述规则中的"%"表示文件名集合"foo"和"bar"。

**[2] \<reprequisites-pattern\>所包含的文件推断** <br>
基于"%"的推断结果，则依赖文件集合为"foo.c"和"bar.c"。

**[3] 根据推断结果将上述makefile规则展开** <br>
注意多个目标文件相当于每一个对应一项单独的生成依赖关系规则。
```txt
foo.o : foo.c
        $(CC) -c $(CFLAGS) foo.c -o foo.o
bar.o : bar.c
        $(CC) -c $(CFLAGS) bar.c -o bar.o
```
试想，如果这种依赖关系结构类似，而目标文件数量很多(e.g.几十or几百个)，那么使用makefile静态规则可以让我们以统一、简洁的方式完成大量具有特定pattern的依赖关系的书写。

##### 3 静态模式额外补充的例子 
下面的例子使用了makefile的filter函数，用于过滤以获得需要的目标文件集合。
```txt
files = foo.elc bar.o lose.o

$(filter %.o, $(files)) : %.o : %.c
    $(CC) -c $(CFLAGS) $< -o $@
$(filter %.elc, $(files)) : %.elc: %.el
    emacs -f batch-byte-compile $<
```
**[1] 简单介绍makefile的filter函数**
filter函数的基本模式为：$(filter PATTERN#1 PATTERN#2..., TEXT)，其作用是：过滤掉TEXT中所有不符合PATTERN#1、PATTERN#2...任意一个的字符串，剩余的部分作为filter的输出。
```txt
sources := foo.c bar.c baz.s ugh.h foo: $(sources)
        cc $(filter %.c %.s $(sources)) -o foo
```
上述makefile规则等价于：
```txt
sources := foo.c bar.c baz.s ugh.h foo: $(sources)
        cc foo.c bar.c baz.s -o foo
```
**[2] 简介makefile的filterout函数**
filterout函数的基本模式为：$(filterout PATTERN#1 PATTERN#2..., TEXT)，其作用和filter函数正好相反，将所有符合PATTERN#1、PATTERN#2的字符串都去掉，保留所有不符合的部分作为filter的输出。
```txt
sources := foo.c bar.c baz.s ugh.h foo: $(sources)
        cc $(filterout %.c $.s $(sources)) -o foo
```
上述makefile规则等价于：
```txt
sources := foo.c bar.c baz.s ugh.h foo: $(sources)
        cc ugh.h -o foo
```

### 八：自动生成依赖性
##### 1 大型工程手动维护makefile的依赖文件是困难的 
makefile中的依赖关系需要包含头文件，如：main.c中使用了#include "defs.h"语句，那么对应的makefile中的依赖关系应该为：
```txt
main.o : main.c defs.h
```
通过手动添加依赖关系对于小的工程项目是可以接受的，但是对于较大的工程项目，手动方式维护makefile可能是非常繁琐复杂的：必须清楚每个c文件中#include包含头文件的情况，在添加、删除头文件时，也需要非常谨慎地修改makefile(需要掌握该头文件被多少源文件引用)。

##### 2 使用c/c++编译器-M选项自动生成依赖关系
针对上述makefile依赖关系维护困难的问题，c/c++编译器提供了相应选项，可以自动生成根据文件的#include的动态依赖关系。<br>
可以在c/c++模块编译时使用"cc -M main.c"，带有该选项的编译命令的输出为：
```txt
main.o : main.c defs.h
```
使用编译器自动生成的依赖关系，在大型c/c++工程中，可以减轻手动书写每个源文件的依赖关系文件的烦恼。

##### 3 使用GNU c/c++编译器自动生成依赖关系的注意事项
GNU c/c++编译器中，"-M"选项和"-MM"选项的输出不同。<br>
"-M"选项的输出的依赖关系不仅包含工程源文件中的#include头文件，还会将c/c++标准库中的头文件包含进来。
使用"gcc -M main.c"命令编译的输出如下：
```txt
  main.o : main.c defs.h /usr/include/stdio.h /usr/include/features.h /
         /usr/include/sys/cdefs.h /usr/include/gnu/stubs.h /
         /usr/lib/gcc-lib/i486-suse-linux/2.95.3/include/stddef.h /
         /usr/include/bits/types.h /usr/include/bits/pthreadtypes.h /
         /usr/include/bits/sched.h /usr/include/libio.h /
         /usr/include/_G_config.h /usr/include/wchar.h /
         /usr/include/bits/wchar.h /usr/include/gconv.h /
         /usr/lib/gcc-lib/i486-suse-linux/2.95.3/include/stdarg.h /
         /usr/include/bits/stdio_lim.h
```
使用"gcc -MM main.c"命令编译的输出如下：
```txt
main.o : main.c defs.h
```

##### 4 如何利用编译器-M/-MM选项完成整个c/c++工程依赖关系构建
GNU组织建议：使用合适的依赖关系生成编译选项执行编译，并将自动生成的依赖关系放到特定文件中：为\<name\>.c源文件生成一个\<name.d\>的makefile文件，\<name\>.d文件中的依赖关系就对应\<name\>.c中的依赖关系。<br>

于是，可以尝试在工程的❮主makefile❯文件中写出"通用"的"%.c"文件自动生成"%.d"的依赖关系命令，并将所有编译选项自动生成的"sub-makefile"包含在主makefile中。这样，就可以近似实现自动化管理工程makefile编译依赖关系了。

**[1] 下面给出一个根据"%.c"生成"%.d"文件的makefile样例**：
```txt
%d : %c
    @set -e; rm -f $@; \
    $(CC) -M $(CPPFLAGS) $< > $@.$$$$; \
    sed 's, \($*\)\.o[ :]*, \1.o $@ : ,g' < $@.$$$$ > $@; \
    rm -f $@.$$$$
```
对于上述文件进行分析，并以注释的方式展示如下。<br>
其中，"<"和">"分别表示linux命令输入、输出重定向(详见shell命令重定向章节)，sed命令使用","代替了"/"作为分隔符，"$@.\$\$\$\$"表示当前进程生成的目标文件("\$\$\$\$"表示当前进程号)。
```txt
# 工程中每个源文件都作为相应的makefile文件的依赖 即<name>.d : <name>.c
%d : %c
    # 后续shell command的返回值非0则整个shell脚本立即退出; 强制删除所有已经生成的目标文件;
    @set -e; rm -f $@; \

    # -M选项：编译命令自动生成include头文件依赖关系; 编译选项设置为CPPFLAGS(for c preprocessor);  
    # "$< > $@"：使用第一个依赖文件 生成(">") 规则中所有目标文件;
    $(CC) -M $(CPPFLAGS) $< > $@; \

    # sed命令执行宏替换：sed 's/OLD/NEW/g'
    sed 's/\($*\)\.o[ :]*/\1.o $@ : /g' < $@. > $@; \

    # 强制删除中间生成的临时文件
    rm -f $@.$$$$
```
$(CC)编译命令生成的中间临时文件如下所示：
```txt
main.o : main.c defs.h
```
通过使用sed命令可以将上述中间生成的临时makefile文件替换成如下所示的"待调用"makefile文件：
```txt
main.o main.d : main.c defs.h
```
这样，"待调用"的makefile文件将和main.o一样，作为依赖文件main.c、defs.h的目标文件；即如果main.c或defs.h文件更新(比如include文件发生修改、替换...)，那么根据makefile依赖关系，main.d同时会被更新，更新的规则按照上述makefile脚本定义所示。

**[2] 将"待调用"makefile加入"核心"makefile** <br>
最后，整个工程应当包含一个用于管理所有"待调用"的"核心"makefile文件。使用include命令可以很容易完成这点：
```txt
sources = foo.c bar.c
include $(sources:.c=.d)
```
上述".c=.d"的含义是进行一种简单替换，即上述文件等效于：
```txt
sources = foo.c bar.c
# 以此方式将"待调用"makefile文件加入到"核心"makefile文件中
include $(foo.d bar.d)
```
**[3] 题外话：关于为makefile写注释** <br>
makefile的注释使用"#"来完成，如果需要多行注释，则在每一行注释的结尾使用"\\"即可。


## Reference
> https://blog.csdn.net/haoel/article/details/2890 <br>
> GNU make zh chapter 4.14

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
