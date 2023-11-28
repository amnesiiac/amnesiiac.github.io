---
layout: post
title: "cpp - uncertain parameters"
subtitle: 'c++接受不定数量类型的参数知识整理'
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: true
date: 2021-05-05 15:15
lang: ch 
catalog: true 
categories: cpp 
tags:
  - Time 2021
  - cpp fundermentals
  - todo
---

## Reference
> https://en.cppreference.com/w/cpp/language/parameter_pack <br>
> https://en.cppreference.com/w/cpp/utility/variadic/va_list <br>
> https://www.learncpp.com/cpp-tutorial/ellipsis-and-why-to-avoid-them/ <br>
> http://www.cppblog.com/woaidongmao/archive/2015/06/19/149036.html



> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内
