---
layout: post
title: "makefile - 依赖规则对应命令中的\"+/-\"符号"
subtitle: '[makefile补充内容]' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2022-01-24 22:44
lang: ch 
catalog: true 
categories: makefile
tags:
  - Time 2022
---
### 内容提要
make通常会在命令行运行结束后检查命令执行的返回状态，如果正确返回，则启动一个新的子shell进程来执行下一个待执行命令；如果命令执行中途出错(返回非0状态)，则make默认放弃对当前规则后续的规则命令的执行，甚至会终止Makefile中所有规则对应命令的执行。

在某些特定的情况下，规则中一个命令执行失败并不代表整个规则执行错误，所以有时可完全忽略这条可能执行失败的命令。

### Makefile中的"-"符号
如果想要忽略某规则下某条命令执行结果对于该规则其他命令、以及Makefile中后续规则对应命令的执行，则可以在想要忽略的命令前添加"-"符号。参考下面的Makefile代码案例进行理解：
```txt
-@ include x.make xx.make xxx.make
```
上面的案例表明：即使"include"命令想要包含的文件不存在，也不会造成Makefile的解析终止(影响其他命令的执行)。
```txt
-@ rm -f x.make
```
上面的案例表明：即使"rm"命令想要删除的文件不存在，也不会造成Makefile的解析终止(影响其他命令的执行)。


### Makefile中的"+"符号
Makefile规则对应命令中的"+"符号和"-"符号的含义不同，它表示当规则对应指令执行出错时，make将不会忽略该错误。例如，当使用带有"-n/-t"命令行参数的make时，对应命令将被执行。

补充："make -n(\-\-just-print)"表示只打印outdated目标所依赖的规则集合，而不会执行对应命令以更新outdated目标；"make -t(touch)"表示只改变命令对应的所有目标文件的时间戳，但不会实际执行命令。

参考下面案例进行理解，假设当前目录下存在3个文件(Makefile、test.txt、test2.txt)：
```txt
all:
    @ rm -f test.txt
    +@ rm -f test2.txt
```
执行"make -n"后，在当前目录下运行"ls"命令，得到上述make命令执行结果：
```txt
mac$ ls
Makefile test.txt
```
可以发现，即使在make中指定"-n(只打印)"选项，但是规则命令中带有"+"标记的命令仍然被执行了；反观未带有"+"标记的命令没有被执行。


## Reference
> https://blog.csdn.net/chuanzhilong/article/details/52461410 <br>

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
