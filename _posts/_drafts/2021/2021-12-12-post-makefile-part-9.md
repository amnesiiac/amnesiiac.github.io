---
layout: post
title: "makefile - #9"
subtitle: '[跟我一起写makefile - 陈皓]' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-12-12 09:07
lang: ch 
catalog: true 
categories: makefile
tags:
  - Time 2021
---
## 使用函数(第一篇)
GNU make中的函数提供了处理文件名、变量、文本、命令的方法。使用函数可以使得Makefile更加灵活、健壮。在需要的地方使用函数处理指定的文本(文本作为函数的参数)，函数在调用它的地方被替换为函数处理的结果。函数引用和变量引用的展开方式相同。

### 一：函数的调用语法
函数调用语法格式如下：
```txt
$(FUNCTION ARGUMENTS)
```
或者：
```txt
${FUNCTION ARGUMENTS}
```

**[关于函数调用格式有如下几点说明]** <br>
- ➀ 调用语法格式中的"FUNCTION"是需要调用的函数名(make命令内嵌的函数名)，对于用户自定义的函数需要通过make的call函数来执行间接调用。
- ➁ ARGUMENTS是函数的参数，参数和函数名之间使用若干空格(推荐使用1个空格)或使用"tab"字符分隔；如果存在多个参数时，参数之间使用逗号","分开。
- ➂ 函数调用需要以"$"开头，用"()"或者"{}"将FUNCTION和ARGUMENTS包括起来。当调用表达式中的ARGUMENTS包含变量or函数的引用时，它们的分隔符尽量和和函数调用风格保持一致(推荐函数调用、参数引用统一使用圆括号)，即推荐使用"$(sort $(x))"而不是"$(sort ${x})"等变体风格。
- ➃ 函数FUNCTION处理参数ARGUMENTS时，参数中如果存在对其他变量or函数的引用，首先对这些引用展开得到参数的实际内容，然后再使用函数对它们进行处理，参数的展开是按照排列的先后顺序完成的。
- ➄ 函数的参数不能出现","以及空格字符，因为","被用于参数之间的分隔符，参数的前导空格会被自动忽略。在实际书写Makefile时，当需要使用","或者空格字符作为函数的参数时，应当将它们赋值给一个变量，在参数ARGUMENTS中通过引用该变量来实现。结合下面例子进行理解：
```txt
comma := ,
empty =
# --- 定义一个空格 并将它赋值给一个变量 ---
space := $(empty) $(empty)
foo := a b c
# --- make的内嵌subst字符替换函数:将变量foo中所有空格替换成"," ---
bar := $(subst $(space),$(comma),$(foo))
```
通过上述函数处理，实现了变量bar的值变成"a,b,c"。


### 二：文本处理函数
这部分内容主要介绍GNU make内嵌的文本(字符串)处理函数。

##### 1 $(subst FROM,TO,TEXT)
- 函数名称：字符串替换函数subst。
- 函数功能：将字符串TEXT中的FROM替换成TO。
- 返回值：替换后的新的字符串。
- 示例，执行如下的函数调用：
```txt
$(subst ee,EE,feet on the street)
```
函数返回的结果为："fEEt on the strEEt"。

##### 2 $(patsubst PATTERN,REPLACEMENT,TEXT)
- 函数名称：模式替换函数。
- 函数功能：搜索字符串TEXT中的以空格分隔开的单词，将其中符合模式PATTERN的单词替换成REPLACEMENT。模式PATTERN中可以使用"%"来代表一个单词中的若干字符。如果参数REPLACEMENT中也包含"%"字符，则这个"%"代表了和PATTERN中相同的字符串。注意，在PATTERN和REPLACEMENT中，只有第一个"%"被当作通配字符来处理，后续出现的"%"字符不再被当成统配字符处理(作为普通字符处理)；当PATTERN和REPLACEMENT中第一个出现的"%"希望作为普通字符处理时，需要使用反斜杠"\"进行转义处理。
- 返回值：替换后的新的字符串。
- 函数说明：字符串TEXT中单词之间如果存在多个空格用于分隔，在处理时将被合并成1个空格，并忽略前导、后缀空格。
- 示例，执行如下的函数调用：
```txt
$(patsubst %.c,%.o,x.c.c bar.c)
```
将变量"x.c.c bar.c"中以".c"结尾的单词替换成以".o"结尾的单词，将得到结果："x.c.o bar.o"。<br>
另外，[makefile-part-7](http://localhost:4000/makefile/2021/09/15/post-makefile-part-7/#1-变量的替换引用)介绍了变量的替换引用相关内容，它可以看作是一种简化板的patsubst函数实现。

##### 3 $(strip STRINT)
- 函数名称：去空字符函数strip。
- 函数功能：去掉字符串(包含若干单词、使用若干空字符分隔)STRINT开头、结尾的空字符，并将中间连续多个空字符合并成1个空字符。
- 返回值：返回无前导、结尾空字符的，使用单一空字符分隔的多单词字符串。
- 函数说明：空字符包括：空格、"tab"等不可显示的字符。
- 示例，执行如下的函数调用：
```txt
STR =      a   b c
LOSTR = $(strip $(STR))
```
将变量"STR"中的空字符进行处理，将得到结果："a b c"。
- 函数应用场景：strip函数经常会用于Makefile条件判断语句中，以确保表达式的可靠性、健壮性。

##### 4 $(findstring FIND,IN)
- 函数名称：查找字符串函数findstring。
- 函数功能：搜索字符串IN中的FIND字符串。
- 返回值：如果在字符串IN中能够找到FIND字符串，则返回FIND，否则返回为空。
- 函数说明：字符串IN中可以包含空格、"tab"，且查找字符串和目标字符串之间必须严格匹配。
- 示例，执行如下的函数调用：
```txt
$(findstring a,a b c)
$(findstring a,b c)
```
第一个函数引用的结果为"a"，第二个返回的结果为空。

##### 5 $(filter PATTERN...,TEXT)
- 函数名称：按模式过滤函数filter。
- 函数功能：过滤掉字符串TEXT中不符合任意一个模式的单词，只要该单词符合PATTERN...中任意一个模式就得到保留。模式字符串PATTERN...通常使用"%"。
- 返回值：以空格分隔的TEXT中所有符合模式PATTERN...的字符串。
- 函数说明：filter函数可以用来去掉一个变量中的某些字符串。
- 示例，执行如下的函数调用：
```txt
sources := foo.c bar.c baz.s ugh.h 
foo: $(sources)
    cc $(filter %.c %.s,$(sources)) -o foo
```
函数的返回结果为："foo.c bar.c baz.s"，规则命令行用于使用cc编译工具来编译生成目标函数foo。

##### 6 $(filter-out PATTERN...,TEXT)
- 函数名称：按模式反向过滤函数filter-out。
- 函数功能：和filter实现的功能恰好相反。过滤掉字符串TEXT中所有符合PATTERN...中任意一个模式的单词，保留不符合所有PATTERN...的单词。存在多个模式时，模式表达式之间使用空格分隔。
- 返回值：以空格分隔的TEXT中所有不符合任意模式PATTERN...的字符串。
- 函数说明：filter-out函数可以用于去掉一个变量中的某些字符串。
- 示例，执行如下的函数调用：
```txt
objects = main1.o foo.o main2.o bar.o
mains = main1.o main2.o
$(filter-out $(mains),$(objects))
```
函数的返回结果为："foo.o bar.o"

##### 7 $(sort LIST)
- 函数名称：排序函数sort。
- 函数功能：将字符串LIST中每个单词按照首字母的升序进行排序，并去掉重复的单词。
- 返回值：以空格分隔的不含任何重复单词的字符串。
- 函数说明：共包含两个功能，将字符排序、去掉字符串中的重复单词。可以单独使用其中某个功能。
- 示例，执行如下的函数调用：
```txt
$(sort foo bar lose foo)
```
函数的返回结果为："bar foo lose"。

##### 8 $(word N,TEXT)
- 函数名称：取字符串中指定的单词函数word。
- 函数功能：返回TEXT中的第N个单词(N的值从1开始)。
- 返回值：返回TEXT的第N个单词。
- 函数说明：如果N大于TEXT中的单词数目，返回空字符串；如果N为0，则出现错误。
- 示例，执行如下的函数调用：
```txt
$(word 2, foo bar baz)
```
函数的返回结果为："bar"。

##### 9 $(wordlist S,E,TEXT)
- 函数名称：截取字符串函数wordlist。
- 函数功能：从字符串TEXT中取出从"S"到"E"的单词串。S和E为单词在字符串中的位置对应的数字。
- 返回值：返回字符串TEXT中从"S"到"E"的子字符串。
- 函数说明："S"和"E"都为从1起始的数字，当"S"大于TEXT中的单词数，返回空；当"E"大于TEXT中的单词数，则返回从"S"开始到TEXT结尾的字符串。
- 示例，执行如下的函数调用：
```txt
$(wordlist 2, 3, foo bar baz)
```
函数的返回结果为："bar baz"。

##### 10 $(words TEXT)
- 函数名称：统计单词数目函数words。
- 函数功能：计算TEXT中的单词数目。
- 返回值：返回TEXT中单词的数目。
- 示例，执行如下的函数调用：
```txt
$(words foo bar)
```
函数的返回结果为："2"。因此，可以使用"$(word $(words TEXT), TEXT)"来表示字符串TEXT的最后一个单词。

##### 11 $(firstword TEXT)
- 函数名称：取字符串TEXT中的第一个单词函数firstword。
- 函数功能：取出TEXT中的第一个单词。
- 返回值：返回TEXT中的第一个单词。
- 函数说明：TEXT是使用空格分隔的包含多个单词的单词序列。函数忽略TEXT中除了第一个单词外的所有单词。
- 示例，执行如下的函数调用：
```txt
$(firstword foo bar)
```
函数的返回结果为："foo"。因此，firstword函数的功能等同于"$(word 1, TEXT)"。


### 三：文件名处理函数
GNU make除了支持上一部分介绍的文本处理函数外，还支持一些用于文件名处理的函数。这些函数主要用于对于使用空格分隔的文件名构成的字符串进行处理(转换)，并返回空格分隔的文件名序列。

##### 1 $(dir NAMES...)
- 函数名称：取目录函数dir。
- 函数功能：从文件名序列NAMES中取出各个文件名的目录部分。文件名的目录部分就是文件名中包含最后一个斜杆("/")的前面的部分。
- 返回值：返回空格分隔的文件名序列NAMES中每个文件的目录部分。
- 函数说明：如果文件名没有斜线，认为此文件为当前目录("./")下的文件。
- 示例，执行如下的函数调用：
```txt
$(dir src/foo.c hacks)
```
函数的返回结果为："src/ ./"。

##### 2 $(notdir NAMES...)
- 函数名称：取文件名函数notdir。
- 函数功能：从文件名序列NAMES中取出各个文件名的非目录部分，即删除所有文件名中的目录部分，只保留非目录部分。
- 返回值：返回文件名序列NAMES中每个文件的非目录部分。
- 函数说明：如果文件名序列NAMES中的某个文件名不包含斜线部分，则不对其进行更改；如果文件名以反斜线结尾，则使用空串代替。注意，当文件名序列NAMES中包含连续多个以反斜线结尾的文件名时，其返回的结果中各个字符间的空格数将为不确定的数值，在使用该函数时需谨慎。
- 示例，执行如下的函数调用：
```txt
objs = $(notdir src/foo.c hacks src1/ src2/ test.txt)
.PHONY : debug
debug :
	echo $(objs)
    @echo $(objs)
```
函数的返回结果为：
```txt
# --- shell回显自身命令的结果 ---
echo foo.c hacks   test.txt
# --- echo函数的回显的结果 ---
foo.c hacks test.txt
# --- @echo命令输出 ---
foo.c hacks test.txt
```

##### 3 $(suffix NAMES...)
- 函数名称：取文件名后缀函数suffix。
- 函数功能：从文件名序列NAMES中取出每个文件名的后缀。文件名后缀包含文件名中最后以"."开始的部分，如果文件名中不包含"."则为空。
- 返回值：返回以单个空格分隔的文件名序列NAMES中的每个文件的后缀序列。
- 函数说明：当NAMES是多个文件名时，返回值为多个以单个空格分隔的单词序列。如果文件名中没有后缀部分，则返回空。当文件名序列NAMES中包含连续多个不包含后缀的文件名时，也只会使用单个空格相连。
- 示例，执行如下的函数调用：
```txt
objs = $(suffix src/foo.c src-1.0/bar.c hacks hello.c.a)
.PHONY : debug
debug :
	echo $(objs)
	@echo $(words $(objs))
```
函数的返回结果为：
```txt
# --- shell回显自身命令的结果(即使hacks为空，两个文件名之间也只有1个空格相连) ---
echo .c .c .a
# --- echo函数的回显的结果(多个.号后缀，只会打印最后一个.号的后缀) ---
.c .c .a
# --- @echo命令输出 ---
.c .c .a
```

##### 4 $(basename NAMES...)
- 函数名称：取前缀函数basename。
- 函数功能：从文件名序列NAMES中取出各个文件名中的前缀部分。文件名的前缀部分包括文件名中最后一个"."号之前的部分。
- 返回值：返回以单个空格分隔的的文件名序列NAMES中各个文件的前缀序列。如果文件名中没有前缀部分，则返回空。
- 函数说明：如果NAMES中的文件名不包含后缀部分，此文件名不会改变。如果文件名中包含多个"."号，则只返回文件名中最后一个"."号之前的部分。
- 示例，执行如下的函数调用：
```txt
objs = $(basename src/foo.c src-1.0/bar.c hacks /home/jack/.font.cache-1)
.PHONY : debug
debug :
	echo $(objs)
	@echo $(objs)
```
函数的返回结果为：
```txt
# --- shell回显自身命令的结果 ---
echo src/foo src-1.0/bar hacks /home/jack/.font
# --- echo函数的回显的结果 ---
src/foo src-1.0/bar hacks /home/jack/.font
# --- @echo命令输出 ---
src/foo src-1.0/bar hacks /home/jack/.font
```

##### 5 $(addsuffix SUFFIX,NAMES...)
- 函数名称：加后缀函数addsuffix。
- 函数功能：为文件名序列NAMES中的文件名添加SUFFIX后缀。文件名序列NAMES使用空格分隔，SUFFIX将别追加到此序列的每个文件名的末尾。
- 返回值：返回以单个空格分隔的添加了后缀SUFFIX的文件名序列。
- 示例，执行如下函数调用：
```txt
$(addsuffix .c,foo bar)
```
函数的返回结果为："foo.c bar.c"。

##### 6 $(addprefix PREFIX,NAMES...)
- 函数名称：加前缀函数addprefix。
- 函数功能：为文件名序列NAMES中每一个添加前缀PREFIX。文件名序列NAMES使用空格分隔，PREFIX将被追加到此序列的每个文件名之前。
- 返回值：返回以单个空格分隔的添加了前缀PREFIX的文件名序列。
- 示例，执行如下的函数调用：
```txt
$(addprefix src/ foo bar)
```
函数的返回结果为："src/foo src/bar"。

##### 7 $(join LIST1,LIST2)
- 函数名称：单词链接函数join。
- 函数功能：将字符串LIST1和字符串LIST2中的每个单词分别对应连接。
- 返回值：返回单空格分隔的合并后的文件名序列。
- 示例，执行如下的函数调用：
```txt
$(join a b,.c .o)
```
函数的返回结果为："a.c b.o"。当LIST1和LIST2之间名字数目不一致时，两者中的多余部分也作为返回序列的一部分，如下所示：
```txt
$(join a b c,.c .o)
```
函数的返回结果为："a.c b.o c"。

##### 8 $(wildcard PATTERN)
- 函数名称：获取符合特定模式的文件名函数wildcard。
- 函数功能：获取当前目录下所有符合模式PATTERN的文件名。
- 返回值：返回空格分隔的、位于当前目录下的所有符合模式PATTERN的文件名序列。
- 函数说明：PATTERN中可以使用所有shell能识别的通配符，如："?"单字符匹配、"\*"多字符匹配等。
- 示例，执行如下的函数调用：
```txt
$(wildcard *.c)
```
函数的返回结果为当前目录下所有".c"文件列表。


## Reference
> https://blog.csdn.net/haoel/article/details/2894 <br>
> GNU make zh/en chapter 8 

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
