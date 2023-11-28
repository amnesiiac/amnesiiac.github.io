---
layout: post
title: "makefile - #11"
subtitle: '[跟我一起写makefile - 陈皓]' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-12-15 14:55
lang: ch 
catalog: true 
categories: makefile
tags:
  - Time 2021
---
## make的执行
一般而言，用于控制整个工程编译规则的Makefile可以通过不止一种方式来执行。其中最简单直接的一种情况是：使用不带任何参数的"make"命令来重新编译所有"outdated"的文件。但是某些情况下，需要使用更加复杂的编译流程：
- ➀ 需要使用make命令更新一部分"outdated"文件，而不是更新全部文件。
- ➁ 需要使用其他编译器，或需要使用新的编译选项来进行重新编译。
- ➂ 只需查看哪些文件被修改，而不需要重新编译。

为了实现上述特殊目的，需要使用make的命令行参数来实现。然而，Make的命令行能够实现的功能不止如此，许多特殊的功能都可以通过make命令行参数来实现。

**[make命令的退出状态]** <br>
- ➀ 退出状态为0：表示make执行成功。
- ➁ 退出状态为2：表示make执行出现错误，并提示错误信息。
- ➂ 退出状态为1：在执行make时使用了"-q"参数，且当前工程中存在"outdated"文件。

本文的内容主要介绍了：如何使用make的命令行选项来实现一些特殊的目的。在本章最后会对make的命令行参数选项进行比较详细的讨论。


### 一：为make指定编译特定的Makefile
当需要将一个普通命名的文件指定为Makefile时，需要使用"-f"、"\-\-file"、"\-\-makefile"选项来实现。例如：
```txt
make -f altmake
```
如果在make命令中通过多个"-f"选项指定了多个文件，则所有指定的文件将共同作为Makefile使用。
```txt
make -f altmake0 -f altmake1 -f altmake2
```
默认情况下，在没有使用"-f"、"\-\-file"、"\-\-makefile"选项指定Makefile文件时，make会在工作目录(./)下，依次搜索命名为"GNUmakefile"、"makefile"、"Makefile"的文件，最终解析执行最先搜索到的那个。


### 二：为make指定最终目标(goals)
> The ❮goals❯ are the targets that make should strive ultimately to update. Other targets are updated as well if they appear as prerequisites of goals, or prerequisites of prerequisites of goals, etc.

➤ **make的默认最终目标** <br>
默认情况下，goals出现在Makefile中，除去以" . "开始的第一个规则中的第一个目标。<br>
因此，Makefile的书写需要保证：该Makefile中，除去以" . "开始的第一个规则中的第一个目标能够描述整个工程、程序的编译过程和规则。在Makefile所在的目录下执行make时，将完成对默认最终目标goals的重编译。

➤ **指定Makefile中的特定目标作为最终目标** <br>
可以通过"make TARGET_NAME"的方式，将Makefile中特定目标指定为make的最终目标(替代默认最终目标)。这种命令行方式支持同时为make指定多个最终目标。

Makefile中任何目标都可以被指定为最终目标，除非该目标使用"-"开头or该目标包含"="(在这两种情况下，该规则将被解释成命令行参数(switch?)、变量定义)。甚至可以指定一个在Makefile不存在的目标作为最终目标，前提是make能够找到一条隐式规则能够完成该目标的构建。<br>
例如：在目录"src"下存在一个源文件"foo.c"，但Makefile中不存在对foo的规则描述。为了编译生成可执行的"foo.c"，只需要将foo作为参数执行"make foo"就可以实现。

➤ **make的特殊变量MAKECMDGOALS** <br>
在命令行指定的最终目标后，make将设置特殊变量MAKECMDGOALS的值。如果命令行中没有指定最终目标，MAKECMDGOALS为空。需要注意：这个变量仅用于特殊场景下(用在Makefile中作为判断条件)，在Makefile中不要对它进行重新定义！例如：
```txt
sources = foo.c bar.c
# ------ 判断命令行参数中指定的最终目标是否为clean ------
ifneq ($(MAKECMDGOALS),clean)
# ------ ------
include $(sources:.c=.d)
endif
```
上例的功能是：在执行"make clean"前，确保避免当前文件包含其他Makefile文件，从而避免创建这些文件后又立即删除它们(a waste)。

➤ **make的特殊变量MAKECMDGOALS** <br>
- ➀ 对整个工程的一部分进行编译，或是只对某几个程序进行编译，而不是完整编译整个工程。也可以使用这种机制完成整个工程的编译："make all(默认最终目标)"。通常的Makefile书写方式如下：
```txt
.PHONY : all
all : size nm ld ar as
```
当仅需要重编译"size"目标时，只要执行"make size"即可，其他目标文件不会被重编译。
- ➁ 指定编译、创建那些在正常编译过程中不能生成的文件，这些文件在Makefile中存在重编译规则，但是它们并没有出现在默认最终目标的所有中间目标依赖中。这个机制通常被用于如：重编译一个调试输出文件、编译一个调试版本的程序...
- ➂ 指定执行一个由伪目标定义的若干条命令、或是由伪目标定义的空目标文件。例如，绝大多数Makefile中都会包含一个"clean"伪目标，该伪目标定义了删除make执行生成的所有文件的命令；通过执行"make clean"就可以删除这些文件。下面列举了一些典型的伪目标、空目标的名字：
    - **all**
    - **clean** 这个伪目标定义了一组命令，这些命令的功能是删除所有由make创建的文件。
    - **mostly clean** 和clean伪目标功能相似，但区别在于它所定义的命令不会全部删除由make创建的文件。例如，不会删除某些编译无关的库文件。
    - **distclean**
    - **realclean**
    - **clobber** 和clean伪目标功能相似，但区别在于它所定义的命令所能够删除的文件返回更广：可以删除非make创建的文件，例如：编译之前系统的配置文件、链接文件等。
    - **install** 将make创建的可执行文件拷贝到shell环境变量PATH指定的目录下。典型的，应用程序编译生成的可执行文件被拷贝到"/usr/local/bin"目录下，相关库文件被拷贝到"/usr/local/bin"目录下。
    - **print** 打印出所有被更改的source files listings。
    - **tar** 创建一个source files对应的tar格式文件(归档文件包)。
    - **shar** 创建一个source files的shell archive(shar文件)。
    - **dist** 创建一个source files的distribution file(分发文件)。该文件可能是一个tar格式文件，或是一个shar格式文件，或是前两种文件的压缩版本。
    - **TAGS** 创建当前目录下所有source files的符号信息文件表，该文件可被vim调用。
    - **check**
    - **test** 对于make生成的应用程序进行self-check。

➂中提到的"伪目标⇔功能"对应关系并不是GNU make规定的，允许在Makefile中定义任意命名的伪目标。但上述对应关系都是约定俗成的，所有开源项目中这些特殊的目标的命名都遵循上述约定。因此，我们的项目中也应该遵循这些约定，使得开源项目看起来像是一个"正规"的项目，而不仅仅为一个样例。


### 三：替代命令的执行(instead of executing recipes)
**为什么需要设置此编译选项** 借助Makefile可以让make获知某个目标是否处于"up to date"的状态，以及更新每个目标的方式(依赖规则)。但是并不是每次执行make时都希望更新目标，例如：有时只想通过执行make检查更新目标的命令是否正确，或者只想查看哪些目标需要更新。

##### 1 替代命令执行的选项介绍
可以通过为make指定相关参数来控制make的执行动作，即通过指定参数来控制make默认动作的执行。相关参数包括：

➤ **"No-op"空操作选项** 
```txt
-n
--just-print
--dry-run
--recon
```
使用此类型选项，make将会打印出"outdated"目标更新所依赖的规则集合，但不会真正执行这些规则。注意，即使使用此类型选项，某些规则仍然会被执行。例如，当规则命令之前采用了"+"指示符，则无论是否使用"No-op"选项，该命令都会被执行。参考下面的示例进行理解：
```txt
all :
    @ rm -f test.txt
    +@ rm -f test2.txt
```
使用"make -n"执行上述Makefile，将会得到如下输出：
```txt
rm -f test.txt
rm -f test2.txt
```
最终，实际上只删除了test2.txt文件，而test.txt文件并没有被删除。也就是说：带有"+"指示符的命令即使在使用"make -n"参数时也会强制执行。

➤ **"Touch"操作选项** 
```txt
-t
--touch
```
更新所有目标的时间戳，但并不会执行实质编译以生成新的目标文件。make将它们的modification time时间戳标记为"up-to-date"，但并不改变文件内容。

➤ **"Question"操作选项** 
```txt
-q
--question
```
检查(sliently check)所有目标文件时间戳是否"up-to-date"，但不会执行相关的规则，也不会打印任何信息。可通过返回的exit code来判断该Makefile中的目标文件是否需要更新：exit code=0表示所检查的目标为"up-to-date"，否则返回exit code=1。

➤ **"What if"操作选项** 
```txt
-W FILE
--what-if=FILE
--assume-new=FILE
--new-file=FILE
```
每使用一个"What if"参数都需要为其指定一个文件名。make将当前系统时间作为"What if"选项指定的文件的modification time时间戳作为，此操作并不会改变该文件实际的modification time时间戳；因此，这个文件的时间戳被认为是"刚刚更新的"，在make执行时依赖于当前文件的所有目标将会被重编译。

**"What if"选项和"-n"选项配合使用** 就可以实现：查看哪些目标依赖于所指定的文件(修改当前文件后哪些目标将被更新)。<br>
**"What if"选项和"-t"选项配合使用** make将只针对"-W"选项指定的文件相关的目标执行"touch"命令，以更新特定目标文件的时间戳。当make没有指定"-s"选项时，可以看到对哪些文件执行了"touch"操作。<br>
需要说明的是：由于执行效率的原因，make并不会调用shell中的touch，而是make直接执行touch操作。<br>
**"What if"选项和"-q"选项配合使用** make不会执行任何依赖规则更新任何目标，不会打印任何信息；当且仅当"What if"选项指定的文件处于"up-to-date"状态时，返回的exit code=0；如果exit code=1，则表明"What if"指定的文件涉及的目标文件需要被更新；如果exit code=2，则表明make执行过程中出现错误。

##### 2 替代命令执行的选项在使用时的注意事项
- ➀ 在同一个make调用时同时指定≥3个上述选项时，会导致错误发生。
- ➁ "-n"、"-t"、"-q"三个选项并不会影响以"+"指示符开头、或者包含$(MAKE)/${MAKE}字符串的命令行的执行。当命令行以"+"开头、或者包含$(MAKE)/${MAKE}时，它的执行不受上述选项影响(强制执行)。在相同Makefile同一规则的其他命令行则因为使用了"-n"、"-t"、"-q"选项的原因而不会被实际执行。
- ➂ "-W"参数提供了两个特性：
    - ➂-➀ 该选项可以和"-n"和"-q"选项一起使用，以查看当前"-W"选项指定的文件上的修改会带来的影响(哪些目标将会被重新编译)。
    - ➂-➁ 不指定"-n"和"-q"选项只使用"-W"参数指定一个文件时，这个文件看起来好像被"刚刚更新过"，因此make执行时将重编译所有依赖该文件的目标；可以用来模拟指定文件被更改后，整个工程的动态变化过程。
- ➃ "-p"和"-v"参数允许输出Makefile相关的执行过程信息，这个功能在调试Makefile时特别有用。


### 四：阻止特定文件被重编译(recompilation)
**为什么需要设置此编译选项** 当修改了一个工程中的源文件后，有些情况下，并不希望重编译依赖于这个文件的目标文件。例如，当修改在一个头文件中新引入了一个宏定义，或增加了一个声明(未被其他文件引用的函数、变量)，该头文件被许多其他文件所包含(依赖)；这些修改不会对已经编译完的程序部分产生任何影响。但默认情况下执行make时，任何header文件中的改动将会导致所有包含它的源文件被重新编译。为了避免重新编译整个工程，可以参考如下的处理流程：

➤ **第一种方式** 
- ➀ 使用make命令对于分别对所有"确实"需要被重编译的源文件进行编译，在编译前需要保证所有目标文件都处于"up-to-date"状态。
- ➁ 编辑需要修改的头文件。注意，此处头文件的修改内容不能对之前已经编译的部分有影响，例如：更改了头文件中原有的宏定义，这将造成该头文件和以存在的源文件代码不匹配！
- ➂ 执行"make -t"命令来将所有目标文件的时间戳标记为"up-to-date"状态，这样再次修改头文件时，和修改内容无关的已编译文件将不会被重编译。

这种实现方式是基于一个重要前提：需要对修改头文件之前的整个工程对应的编译程序进行保存。绝大多数情况下，这种要求是很难实现的，因为，在修改这个头文件之前，已经对工程文件(header files/source files)进行了较多的修改，且未执行make。这种方式适用于"有计划"的修改，而不适用于正常构建工程的过程。因此我们需要引入第二种方式如下。

➤ **第二种方式** 
- ➀ 执行"make -o headerfile"命令，headerfile为需要忽略其修改内容的文件，指定-o选项以防止那些依赖于headerfile的其他文件被重编译。如果涉及到多个headerfile的修改，则为每个headerfile指定"-o"选项："make -o headerfile1 -o headerfile2..."。通过这种方式，使用"-o"指定的headerfile的修改就不会导致依赖它的目标文件被重编译。注意，这种方式仅使用于headerfile，不能将"-o"选项用于源文件。关于make命令的"-o"选项，可参考本文最后部分的总结。
- ➁ 使用"make -t"命令更新所有目标文件的时间戳。


### 五：替换变量的定义(overriding variables)
在执行make时，使用带有"="符号的命令行参数如"v=x"的含义是：定义变量v的值为x，并将变量v作为make的参数。<br>
默认情况下，这种方式定义的make参数将会覆盖Makefile中同名变量的定义；但是这种变量定义的方式不会覆盖使用"override"修饰的Makefile变量。

➤ **第一种方式** 在Makefile中使用CFLAGS参数来指定编译参数：
```txt
make -c $(CFLAGS) foo.c
```
使用上述形式的make命令行，可以通过在Makefile中定义变量CFLAGS的值来控制编译选项：
```txt
# Makefile Sample
CFLAGS=-g
```
上述案例中执行make命令时，实际的编译命令为"cc -c -g foo.c"。

➤ **第二种方式** 
```txt
# 在参数中如果包含空格、shell的特殊字符 则需要将参数放在引号中
# Use '' to enclose spaces and other special characters in the value of a commandline variable.
make CFLAGS=' -g -O' foo.c
```
使用上述形式的make命令行，实际编译命令为"cc -c -g -O foo.c"。

##### 替代变量定义方法的注意事项
➀ 通过命令行指定变量定义以替代变量定义时，也存在两种风格的变量的定义：递归展开式变量定义、直接展开式变量定义。除非在命令行定义的变量值中包含了对其他变量、函数的引用，否则两种方式在此是等价的。
➁ 有时为了防止命令行参数的变量定义覆盖Makefile中同名变量的定义，可以在Makefile中使用override指示符对变量进行声明。


### 六：使用make进行编译测试(testing the compilation of a program)
通常情况下，make执行Makefile时，如果命令执行发生错误，会立即放弃继续执行，并返回一个非0的状态。因此，当执行发生错误时将导致最终目标不能被重建。

在特定的场景下(我们在修改一些源文件后重新编译工程)，希望即使某个文件发生错误，仍然能够继续后续文件的编译，直到发生连接错误才退出。这种场景通常用于获悉：工程中被修改的源文件中那些文件没有修改正确；这样我们可以在下次编译时，对所有出现错误的文件进行修改，而不是完成一个文件的修改就执行一次重编译。

为了实现上述目的，需要使用带命令行选项的make命令："make -k"、"make \-\-keep-going"来完成。这个参数的功能是：make在遇到错误时继续执行编译(只是给出错误消息)，直到最后出现致命错误(无法重建最终目标)才返回非0值并退出。这是一种非常重要的调试Makefile或查找工程源文件错误的一种非常重要的手段。

### 七：make的命令行选型(summary of make options)
本部分内容主要介绍make支持的命令行选项(这个选项可以通过make man手册进行查询)。
- ➀ **"-b"、"-m"** 这两个选项被make自动忽略，以和其他版本的make相互兼容。
- ➁ **"-B"、"\-\-always-make"** 强制重建所有目标，不需要根据Makefile中的规则依赖关系决定是否重建目标。
- ➂ **"-C DIR"、"\-\-directory=DIR"** 先进入指定目录DIR，然后再读取目录下的Makefile并执行make。如果make命令行中存在多个"-C"选项，较后面的选项指定的DIR是前一个DIR下的相对目录。例如："make -C / -C etc"等价于"make -C /etc"。这属于make的递归调用的用法。
- ➃ **"-d"** make在执行过程中，打印出所有的调试信息。包括：哪些文件需要被重编译、哪些文件正在被比较以及和哪个修改时间的结果相比较、哪些文件实质上需要被重编译、哪些隐含规则需要被调用。"-d"选项的作用等价于"\-\-debug=a"。
- ➄ **"\-\-debug=[OPTIONS]"** 该命令行选项用于make执行时输出调试信息(通过设置OPTIONS来设置输出的调试信息的级别)。其中OPTIONS字段可以为如下几种情况：
    - ➄-➀ **a (all)** 输出所有类型的调试信息。等效于"-d"选项。
    - ➄-➁ **b (basic)** 输出基本的调试信息。包含：哪些目标处于outdated状态、以及当前重编译是否成功完成。
    - ➄-➂ **v (verbose)** 输出基本的调试信息，并含有一些补充信息。包含：哪些makefile被make解析、一些不需要被重编译的的依赖规则, etc.
    - ➄-➃ **i (implicit)** 输出描述所有用到的隐式规则的相关信息。此选项默认打开basic级别的调试信息。
    - ➄-➄ **j (jobs)** 输出所有调用specific sub-commands的子进程相关信息。
    - ➄-➅ **m (makefile)** 输出make指令读取、更新、执行Makefile的相关信息。all选项默认打开此选项；此选项也将打开basic选项信息。
    - ➄-➆ **n (none)** 禁用当前启用的所有调试选项。在这个选项后make遇到的的调试选项不会被禁用。
- ➅ **"-e"、"\-\-environment-overrides"** 使用系统环境变量的定义覆盖Makefile中同名变量的定义。
- ➆ **"-E string"、"\-\-eval=string"** 该选项将命令行中输入的string按照Makefile语法进行评估。该选项可以被看作是eval函数的命令行版本。对于输入的string的估计(eval)是在default rules和variables定义之后，在读取任何Makefile之前。
- ➇ **"-f=FILE"、"\-\-file=FILE"、"\-\-makefile=FILE"** 指定FILE作为make执行的makefile文件。
- ➈ **"-h"、"\-\-help"** 打印帮助信息。
- ➀⓪ **"-i"、"\-\-ignore-errors"** 忽略所有用于编译工程的规则执行过程中的错误。 
- ➀➀ **"-I DIR"、"\-\-include-dir=DIR"** 指定Makefile中包含其他Makefile时的搜索路径DIR。当使用多个-I选项指定多个路径时，目录搜索按照顺序执行。
- ➀➁ **"-j [JOBS]"、"\-\-jobs[=JOBS]"** 指定可以同时执行的命令数目。在没有指定"-j"参数的情况下，默认支持同时执行系统支持的最大并行命令数目。当存在多个-j参数时，只有最后一个-j选项执行的参数值有效。
- ➀➂ **"-k"、"\-\-keep-going"** 执行命令发生错误时不会中止make的执行，make尽可能完成所有命令规则的执行，直到出现fatal error导致无法继续执行。
- ➀➃ **"-l LOAD"、"\-\-load-average[=LOAD]"、"\-\-max-load[=LOAD]"** 该选项通知make：当存在其他任务执行时，如果系统负荷超过LOAD，则不再启动新任务。如果-I选项未指定LOAD数值，则被用于取消之前-I选项设置的LOAD限制。
- ➀➄ **"-n"、"\-\-just-print"、"\-\-dry-run"、"\-\-recon"** 该选项使得make只打印所要执行的命令，但不会实际执行命令。
- ➀➅ **"-o FILE"、"\-\-old-file=FILE"、"\-\-assume-old=FILE"** 该选项指定不重编译FILE文件，即使FILE相对它的依赖项已经处于outdated状态；另外也不会重编译依赖于FILE的任何目标文件。注意：此选项不会通过变量MAKEFLAGS传递给子make进程。
- ➀➆ **"-p"、"\-\-print-data-base"** 命令执行前，打印出make指令读取的Makefile中的data base(预定义的规则、变量)，同时打印出make版本信息。如果想打印这些信息但不实际执行命令，则可以使用"make -qp"。如果想查看执行makefile前的data base变量和规则，你可以使用"make –p –f /dev/null"；这个参数输出的data base信息包含Makefile文件的文件名和行号，因此该命令可作为复杂环境下的调试指令。关于unix中的/dev/null特殊文件详见[wikipedia](https://zh.wikipedia.org/wiki//dev/null)。
- ➀➇ **"-q"、"\-\-question"** 进入"询问模式"，不运行任何规则命令，不会打印任何输出。如果指定的目标已经处于up-to-date的状态，则返回退出状态0(表示执行成功)。如果任何目标需要被重编译，则返回退出状态1(表示存在outdated文件)。如果命令执行中出现错误，则返回退出状态2。
- ➀➈ **"-r"、"\-\-no-builtin-rules"** 取消所有内置的隐式规则，仍然可以在Makefile中通过编写模式规则的方式来定义自己的规则。-r选项会取消所有支持后缀规则(suffix rules)的默认后缀列表，仍然可以在Makefile中通过".SUFFIXES"定义自己的后缀规则。注意，-r选项只对规则产生影响，该选项不会取消make内置的dafault variables。
- ➁⓪ **"-R"、"\-\-no-builtin-variables"** 取消make内置的隐式变量，仍然可以在Makefile中明确定义这些变量。注意，启用-R选项将同时启用-r选项，因为隐式规则是以隐式变量为基础的，取消了隐式变量(-R)，隐式规则(-r)也失去了效用。
- ➁➀ **"-s"、"\-\-slient"、"\-\-quiet"** 命令执行时不会将命令本身打印到shell上。
- ➁➁ **"-S"、"\-\-no-keep-going"、"\-\-stop"** 取消-k选项的效用。-k选项可以通过递归执行make中从上层MAKEFLAGS变量中继承而来，也可以通过环境中的MAKEFLAGS中获得；使用-S选项可以取消从上层继承的-k选项，或者取消系统环境变量MAKEFLAGS中的-k选项。
- ➁➂ **"-t"、"\-\-touch"** 该选项同linux中的touch指令作用相同：更新所有目标文件的时间戳到当前系统时间。该选项可用于对所有outdated目标文件的重编译。
- ➁➃ **"-v"、"\-\-version"** 该选项用于查看make版本信息。
- ➁➄ **"-w"、"\-\-print-directory"** 在make进入一个目录读取Makefile前打印该工作目录。这个选项可以帮助我们定位make正在工作的目录，便于跟踪错误。使用"-C"选项时将默认打开这个选项。
- ➁➅ **"\-\-no-print-directory"** 取消"-w"选项。可以用于递归调用的make过程中，取消-C选项默认打开的打印工作目录功能。
- ➁➆ **"-W FILE"、"\-\-what-if=FILE"、"\-\-new-file=FILE"、"\-\-assume-file=FILE"** 该选项设置FILE文件的时间戳为当前时间，但不改变文件的实际最后修改时间。该选项主要用于对于所有依赖于FILE文件的目标的强制重编译。当该选项和-n选项一起使用时，可以看到当修改FILE文件的时间戳后，将执行什么操作。
- ➁➇ **\-\-warn-undefined-variables** 当发现Makefile中存在对未定义的变量进行使用时给出警告信息。该选项用于辅助我们调试一个存在多级嵌套变量引用的复杂Makefile；另外注意，在Makefile书写时尽量避免超过3级以上变量嵌套引用。


## Reference
> https://blog.csdn.net/haoel/article/details/2896 <br>
> GNU make zh/en chapter 9 

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
