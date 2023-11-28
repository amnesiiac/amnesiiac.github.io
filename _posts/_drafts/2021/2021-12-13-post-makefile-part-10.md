---
layout: post
title: "makefile - #10"
subtitle: '[跟我一起写makefile - 陈皓]' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-12-13 16:12
lang: ch 
catalog: true 
categories: makefile
tags:
  - Time 2021
---
## 使用函数(第二篇)
GNU make中的函数提供了处理文件名、变量、文本、命令的方法。使用函数可以使得Makefile更加灵活、健壮。在需要的地方使用函数处理指定的文本(文本作为函数的参数)，函数在调用它的地方被替换为函数处理的结果。函数引用和变量引用的展开方式相同。


### 一：foreach函数
make中的foreach函数是一个循环函数，它类似于shell中的for语句。

➤ **foreach函数语法**：<br>
```txt
$(foreach VAR,LIST,TEXT)
```

➤ **foreach函数功能**：<br>
➀ 如果VAR和LIST中存在变量、函数的引用首先将其展开；表达式TEXT中的变量不用展开。<br>
➁ 函数执行时，将LIST中使用空格分隔的单词依次取出赋值给变量VAR，每赋值一次就执行一次TEXT表达式，重复操作直到LIST最后一个单词。<br>
➂ TEXT中的变量、函数的引用在其被执行时才展开；如果TEXT中包含对VAR的引用，则$(VAR)在每一次展开时都不同。<br>

➤ **foreach函数返回值**：<br>
将返回以空格分隔的多次TEXT表达式计算的结果序列。

➤ **foreach函数案例**：<br>
```txt
dirs := a b c d
files := $(foreach dir,$(dirs),$(wildcard $(dir)/*))
```
TEXT表达式为$(wildcard $(dir)/\*)，第一次展开为$(wildcard a/\*)，第二次展开为$(wildcard b/\*)，第三次展开为$(wildcard c/\*)，第四次展开为$(wildcard d/\*)。因此，上述Makefile片段等价于：
```txt
files := $(wildcard a/* b/* c/* d/*)
```
当foreach函数中的TEXT表达式过于复杂时，为了使Makefile更加清晰，可以将TEXT表达式定义为一个中间变量，并在foreach函数中引用该变量。上述Makefile片段等价于：
```txt
# --- 将TEXT表达式定义为递归展开变量是因为：为了防止$(dir)在定义find_files被展开 ---
find_files = $(wildcard $(dir)/*)
dirs := a b c d
files := $(foreach dir,$(find_files))
```

➤ **foreach函数说明**：<br>
foreach中的变量VAR的作用相当于一个局部临时变量，它只在foreach函数的上下文中有效，它的定义不会影响Makefile中其他部分中同名VAR变量的定义。另外，foreach中的变量VAR是一个直接展开式变量。<br>
尽量不要为VAR变量起奇怪的命名，尽管下面的例子可以正常工作，但不推荐：
```txt
files := $(foreach Esta escrito en espanol!,b c ch,$(find_files))
```


### 二：if函数
函数if在Makefile中提供一种在函数上下文中实现条件判断的功能，类似make支持的条件语句ifeq、ifdef一样。

➤ **if函数语法**：<br>
```txt
$(if CONDITION,THEN-PART)
```
```txt
$(if CONDITION,THEN-PART,ELSE-PART)
```

➤ **if函数功能**：<br>
➀ 条件表达式参数CONDITION，在if函数执行时将忽略其前导、后缀空格，如果CONDITION中包含其他变量引用、函数引用则需展开。如果CONDITION展开结果非空，则条件判断为真，执行THEN-PART表达式；否则执行ELSE-PART表达式。<br>
➁ if函数的返回结果为➀中表达式的计算结果。

➤ **if函数返回值**：<br>
根据CONDITION判断选择那个表达式进行执行，并将执行的结果作为if函数返回值。特别的，当CONDITION为空，但又不存在ELSE-PART表达式时，if函数返回为空。

➤ **if函数说明**：<br>
if函数中的条件表达式CONDITION决定了函数的返回值只能是THEN-PART或ELSE-PART两个之一的执行结果。

➤ **if函数示例**：<br>
```txt
obj = $(if $(SRC_DIR),$(SRC_DIR),/home/src)
.PHONY : debug
debug :
	echo $(obj)
	@echo $(obj)
```
if函数的返回结果为：
```txt
echo /home/src
/home/src
/home/src
```


### 三：call函数
call函数是唯一一个能够将自定义函数的引用作为参数的函数。

➤ **call函数语法**：<br>
```txt
$(call VARIABLE,PARAM1,PARAM2,...)
```

➤ **call函数功能**：<br>
在执行call函数时，将其参数PARAM1, PARAM2...依次赋值给make临时变量$(1), $(2)...，这些临时变量在VARIABLE参数的定义中被使用。<br>
call函数对于参数的数量没有限制，因此也可以没有参数，但没有参数的call函数没有任何实际存在的意义。

➤ **call函数返回值**：<br>
call函数的参数PARAM1, PARAM2...依次赋值给临时变量$(1), $(2)...之后，变量VARIABLE表达式的计算值。

➤ **call函数说明**：<br>
➀ call函数中VARIABLE是一个变量名，而不是一个变量引用，因此通常情况下VAIRABLE中不包含"$"字符，除非这个变量名是一个嵌套变量名。<br>
➁ 当VARIABLE是一个make命令内嵌函数名时(e.g. if、foreach、strip...)，对于PARAM1, PARAM2...的设置需要十分谨慎，否则可能会导致返回出人意料的结果。<br>
➂ call函数中多个参数PARAM之间需要使用逗号分隔。<br>
➃ VAIRABLE在定义是不能定义为直接展开式，只能定义为递归展开式。

➤ **call函数示例**：<br>
➀ 示例一：
```txt
# --- 自定义的reverse函数体 ---
reverse = $(1) $(2)
# --- call函数中调用自定义reverse函数 并将a,b依次赋值给$(1)、$(2) ---
foo = $(call reverse,a,b)
```
call函数调用reverse函数后得到的返回结果为"ab"。
```txt
# --- 自定义的reverse函数体 ---
reverse = $(2) $(1)
# --- call函数中调用自定义reverse函数 并将a,b依次赋值给$(1)、$(2) ---
foo = $(call reverse,a,b)
```
由于reverse函数定义和第一个相反，所以call函数调用reverse函数后得到的返回结果为"ba"。即reverse变量中的参数顺序可以根据需要来进行调整，不一定必须是从$(1)到$(2)。<br>
➁ 示例二：
```txt
pathsearch = $(firstword $(wildcard $(addsuffix /$(1),$(subst :, ,$(PATH)))))
LS := $(call pathsearch,ls)
.PHONY : debug
debug :
	echo $(LS)
	@echo $(LS)
```
call函数返回的结果为：
```txt
echo /bin/ls
/bin/ls
/bin/ls
```
执行过程为：首先使用内置函数subst将PATH变量从":"转换成空格分隔的路径字符串；使用内置函数addsuffix为指定路径添加格式后缀，后缀为"$(1)"="ls"；然后使用内置wildcard函数进行匹配，找出当前目录下所有符合模式的文件名构成序列；最后使用内置函数firstword找到符合模式的文件名序列的第一个文件名作为返回值。<br>

➂ 示例三：
```txt
map = $(foreach a,$(2),$(call $(1),$(a)))
obj = $(call map,origin,o map MAKE)
.PHONY : debug
debug :
	echo $(obj)
	@echo $(obj)
```
call函数返回的结果为：
```txt
echo file file default
file file default
file file default
```
执行过程为：首先将根据map变量的定义，将obj中的对应PARAM1, PARAM2分别带入，得到展开形式的obj表达式如下：
```txt
obj = $(call $(foreach a,$(2),$(call $(1),$(a))),origin,o map MAKE)
obj = $(foreach a,o map MAKE,$(call origin,$(a)))
```
origin函数将会返回每个输入变量($(a))的来源，foreach函数用于将输入的三个参数(o map MAKE)分别带入origin函数，并返回输出结果序列：
```txt
obj = $(call origin,o) $(call origin, map) $(call orgin MAKE)
```

➤ **call函数应用注意事项**：<br>
和其他函数一样，call函数会保留出现在其参数值列表中的空字符，在使用参数时对空格的处理需要特别小心：如果参数PARAM中存在多余空格，call函数可能会返回一个不符合预期的结果。因此，在变量PARAM作为call函数的参数之前，应调用strip函数去掉其中❮多余❯的空格。


### 四：value函数
value函数提供了一种：不对变量进行展开的情况下获取变量值的方法。value函数不会取消操作的变量在定义时已经发生了的替换展开。<br>
例如：定义了一个直接展开式变量，此变量在定义式中包含了对其他变量的引用；在对这个直接展开式变量进行取值时，得到的是不包含任何引用的值，而不是取消解引用操作后的原始形式表达式。

➤ **value函数语法**：<br>
```txt
$(value VARIABLE)
```

➤ **value函数功能**：<br>
不对VARIABLE变量进行任何展开操作(解引用)，直接返回VARIABLE的值。VARIABLE参数是一个变量名，变量名中一般不包含"$"，除非VAIRABLE中使用了嵌套变量机制。

➤ **value函数返回值**：<br>
返回VAIRABLE变量表达式中所定义的文本值。如果VAIRABLE被定义为递归展开式变量，其中包含其他变量、函数的引用，value函数不对这些引用进行展开，函数的返回值包含引用值。

➤ **value函数示例**：<br>
```txt
# Sample Makefile
FOO = $PATH
all :
    @echo $(FOO)
    @echo $(value FOO)
```
上述案例使用make执行得到的结果为(mbp)：
```txt
ATH
# --- 省略了部分输出内容 ---
/Users/mac/.rvm/gems/ruby-2.6.0/bin:/Users/mac/.rvm/gems/ruby-2.6.0@global/bin: ...
```
第一行输出为"ATH"，这是因为变量FOO被定义为"$PATH"，而"$P"被展开为空，所以得到"ATH"的输出结果。第二行输出为"..."，是mbp中的系统环境变量"$PATH"的值；我们可以在mbp系统shell中运行如下指令验证上述Makefile得到的输出：
```txt
# --- 此shell命令将得到和value函数示例相同的结果 --- 
echo $PATH
```


### 五：eval函数
eval是一个比较特殊的函数，使用这个函数可以在Makefile中构建一个可变的规则依赖关系链，在构建的关系链中可以使用其他变量和函数的引用。

➤ **eval函数语法**：<br>
```txt
$(eval VARIABLE)
```

➤ **eval函数功能**：<br>
函数eval对它的VARIABLE进行展开，展开的结果表达式被作为当前Makefile的一部分，因此make可以对展开的内容做语法解析。函数eval展开的结果表达式可以包含一个新变量、新目标、隐式规则、显式规则等。即eval函数的主要功能是：根据参数的关系、结构，对它们进行替换展开。eval常常和define定义的多行变量结合使用。

➤ **eval函数返回值**：<br>
eval函数对于变量进行展开操作，因此eval函数的返回值为空，即没有返回值。

➤ **eval函数示例**：<br>
下面的函数示例看起来有些复杂，但是实际上实现了一个较简单的"自动化"为Makefile添加规则的功能。
```txt
# Sample Makefile
# ------ 定义program变量#1------
PROGRAMS = server client
# ------ 定义server变量#2 ------
server_OBJS = server.o server_priv.o server_access.o
server_LIBS = priv protocol
# ------ 定义client变量#3 ------
client_OBJS = client.o client_api.o client_mem.o
client_LIBS = protocol
# ------ 构建伪目标all的依赖关系 ------
.PHONY : all
all : $(PROGRMAS)
# ------ 定义program_template多行变量 ------
# ------ 第一行：server/client的依赖关系 ------
# ------ 第二行：all_objs总目标变量定义 ------
define PROGRAMS_template
$(1): $$($(1)_OBJS) $$($(1)_LIBS:%=-1%)
ALL_OBJS += $$($(1)_OBJS)
endef
# ------ 调用call eval foreach函数 ------
# ------ 使用eval函数将foreach生成的每条规则在当前Makefile中进行展开 ------
# ------ 如果不使用eval函数 则此Makefile中将不存在target:prerequisites规则 ------
$(foreach prog,$(PROGRAMS),$(eval $(call PROGRAM_template,$(prog))))
# ------ 使用无依赖项的make label构建一些编译相关的指令make server/make client ------
$(PROGRAMS):
    $(LINK.o) $^ $(LDLIBS) -o $@
# ------ 使用无依赖项的make label执行一些编译收尾的工作 ------
clean :
    rm -f $(ALL_OBJS) $(PROGRAMS)
```
eval函数的例子启发我们：将一系列有相似之处的Makefile集合进行归纳总结，通过实现一个较复杂的通用自动化生成规则的Makefile模版来取代这些Makefile中的相似部分；然后在所有Makefile中包含该模版Makefile即可。这样做的好处是：1) 可以一定程度上的Makefile自动化，省去为复杂的包含大量相似规则的Makefile的工程手写规则的痛苦；2) 能够将子工程对应的Makefile书写的更加清晰简单。

### 六：origin函数
函数origin同其他函数不同，它并不会操作参数VAIRABLE，只是获取VARIABLE相关信息，并返回变量的出处。

➤ **origin函数语法**：<br>
```txt
$(origin VAIRABLE)
```

➤ **origin函数功能**：<br>
origin函数用于查询参数VARIABLE的出处。

➤ **origin函数返回值**：<br>
origin函数的参数VARIABLE是一个变量名，而不是变量的引用，因此通常VARIABLE不包含"$"，除非它是一个嵌套引用变量名。

➤ **origin函数说明**：<br>
origin函数返回VARIABLE的定义方式，用字符串进行表示。该函数的返回情况有如下几种情况：
- ➀ **undefined**：变量VARIABLE并未被定义。
- ➁ **default**：变量VARIABLE是一个默认定义(是一个make内嵌变量名)。如果这些内嵌变量在Makefile中被重定义，origin函数的相应返回值将会随之变化。e.g. "CC"、"MAKE"、"RM"...
- ➂ **environment**：VARIABLE是一个inherited的环境变量，且该变量未通过"Make -e"的方式被Makefile中的同名变量覆盖。
- ➃ **environment override**：变量VARIABLE是一个inherited的环境变量，且使用"make -e"执行Makefile。此时Makefile中存在一个和VARIABLE同名的变量，此时系统环境变量将会覆盖文件中的定义。
- ➄ **file**：变量VARIABLE在某个Makefile文件中被定义。
- ➅ **command line**：变量VARIABLE在命令行中被定义。
- ➆ **override**：变量VARIABLE在Makefile中被定义，且使用override指示符声明。
- ➇ **automatic**：变量VARIABLE是一个make自动化变量。

➤ **origin函数示例**：<br>
origin函数返回的信息对于书写Makefile是非常有帮助的，可以让我们在使用一个变量之前，对其应用的合法性进行判断，从而在一定程度上控制变量表达的优先级，实现更复杂、更灵活的变量使用。

例如：现有一个Makefile文件foo，它引用了另一个Makefile文件bar。Makefile foo中定义了变量bletch，bar中也定义了变量bletch。<br>
❮要求#1❯ 运行"make -f bar"命令，希望使用定义在bar文件中的变量bletch，而不是环境中(foo)事先定义的变量bletch，这可以通过在bar中使用override directive来定义bletch。但是这种做法存在弊端：当在bar使用了override后，将同时会override任何来自命令行的重定义。<br>
❮要求#2❯ bar中的bletch变量定义能够屏蔽环境中的定义但不屏蔽来自命令行对它的重定义。

一种能够同时满足上述两个要求的定义方式如下所示：
```txt
# Makefile bar-1
# ------ 如果已经定义了bletch变量 ------
ifdef bletch
# ------ 如果变量bletch的定义来自environment ------
ifeq "$(origin bletch)" "environment"
bletch = barf, gag, etc.
endif
endef
```
另一种能够满足上述两个要求的定义方式如下所示，在$(origin bletch)返回"environment"或"environment override"都将对变量bletch重新定义：
```txt
# Makefile bar-2
# ------- 如果变量bletch的定义来自environment -------
ifneq "$(findstring environment,$(origin bletch))" ""
bletch = barf, gag, etc.
endif
```


### 七：shell函数
shell函数不同于除wildcard之外的所有函数，它可以和make命令之外进行通信。

➤ **shell函数语法**：<br>
```txt
$(shell VARIABLE)
```

➤ **shell函数功能**：<br>
➀ 函数shell所实现的功能和shell中的反引号(backquotes)相同，用于实现命令拓展。<br>
➁ 该函数接受shell命令作为参数，并计算(evaluate)该命令的输出结果。针对shell函数中VARIABLE的执行结果的唯一的处理是：将每个"newline"或"carriage-return/newline"转化成一个单独的空格符；命令输出结果中结尾处的"carriage-return/newline"将被直接移除。<br>
➂ shell函数中参数VARIABLE中的命令当且仅当$(shell)所在的规则命令行被执行时才被运行。由于shell函数VARIABLE中的shell命令需要占用一个新的shell进程，因此需要谨慎考虑使用递归展开式变量or直接展开式变量对于shell函数执行效率的影响。

➤ **shell函数返回值**：<br>
shell函数的参数VARIABLE在shell环境下的命令执行结果。


➤ **shell函数说明**：<br>
➀ shell函数的返回值是其参数VARIABLE的执行结果，没有进行任何处理。对于shell函数返回结果的后续处理是由make来完成的。<br>
➁ 当对shell函数的引用出现在Makefile的规则命令行中，则当命令被执行时shell函数才被展开，并且shell函数VARIABLE命令的执行是在另外一个shell子进程完成的。<br>
➂ 在引用shell函数时，需要注意：对出现在Makefile规则命令行中的多级shell函数引用应当谨慎处理，否则会影响效率(每一级shell函数的VARIABLE都会占用各自的shell进程)。

➤ **shell函数示例**：<br>
➀ 示例一：
```txt
contents := $(shell cat foo)
```
将文件foo中的内容赋值给Makefile中的变量contents，其中需要将换行用空格代替。

➁ 示例二：
```txt
files := $(shell echo *.c)
```
将当前目录下所有后缀名为".c"的文件名(文件名之间使用空格分隔)赋值给Makefile中的files变量。这个示例和wildcard命令执行的结果相同：
```txt
$(wildcard *.c)
```

➤ **shell函数的思考**：<br>
通过上述示例一、示例二可以看出：shell函数中VARIABLE中的shell命令在make读入相应Makefile文件时就已经完成展开，这样在后续的对contents、files变量的引用不在进行展开操作。


### 九：make的控制函数
make提供了两个控制make运行方式的函数。make控制函数通常用于：当make执行过程中为用户提供一些必要信息，并且当检测到environmental error发生时，可以中止make的执行。

##### 1 $(error TEXT...)
➤ **函数功能**：<br>
产生"fatal error"，并提示相关信息TEXT给make用户。注意：当且仅当$(error TEXT...)函数被调用时，才会产生错误信息提示。因此，如果该函数在Makefile规则(recipe)中被引用，或出现在一个递归展开式变量的表达式中，在读取Makefile时不会立即产生错误信息。

➤ **函数返回值**：<br>
该函数返回值为空。

➤ **函数说明**：<br>
$(error TEXT...)函数一般不出现在直接展开式变量的定义中，否则每次make读取Makefile时就会出现fatal error；显然在一般情况下是不合理的。

➤ **函数示例**：<br>
➀ 示例一：
```txt
ifdef ERROR1
$(error error is $(ERROR1))
endef
```
该示例实现了：当make读取Makefile时，如果之前已经定义了变量ERROR1，则make将返回错误信息："error is $(ERROR1)"并退出。

➁ 示例二：
```txt
ERR = $(error found an error!)
.PHONY : err
err : ; $(ERR)
```
该示例实现了：在make读取Makefile时，不会返回错误信息：found an error!；仅当伪目标err被当作目标执行时才会退出make执行&显示错误信息$(ERR)。

##### 2 $(warning TEXT...)
➤ **函数功能**：<br>
函数$(warning TEXT...)从功能上来讲类似函数$(error TEXT...)，它们的区别在于，前者不会导致fatal error，make不会退出只是提示警告信息"TEXT..."，make的执行进程继续进行。

➤ **函数返回值**：<br>
该函数返回值为空。

➤ **函数说明**：<br>
函数的用法和$(error TEXT...)相似，展开过程相同。


## Reference
> https://blog.csdn.net/haoel/article/details/2895 <br>
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
