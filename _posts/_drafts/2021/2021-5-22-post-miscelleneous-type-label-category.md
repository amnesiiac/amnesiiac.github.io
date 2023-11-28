---
layout: post
title: "Category & Label & Tag"
subtitle: '关于category &label & tag三种概念的定义'
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-05-22 13:10
lang: ch 
catalog: true 
categories: miscelleneous 
tags:
  - Time 2021
---
## Category & Label & Tag 三者比较
**[1]** Category强调相同类别内的东西的共性，强调不同类别东西的差异性。Tag强调的是物体的个性，稍微弱化了不同Tag之间的差异性。

**[2]** Category对于博客而言是一种内源性的分类方法，属于客观的对博客文章进行分类，且每篇博客文章尽量只有一个Category标签。Tag对于博客而言是一种补充Category分类的机制，更常用于主观上的博客内容分类，且每篇博客文章可以拥有多个Tag标签。

**[3]** Category对于博客而言，更多的是给其他人看的。Tag标签更多的是给自己看的。

**[4]** Category的分类方式是由内而外的看待事物并进行分类的方式。Tag的分类方式是由外而内的看待事物并进行分类的方式。

**[5]** Label可以看成是一种Category下的更加细分的类别属性，另外Label同样具有Category的客观性，并强调共性和差异性。

## 生动形象的例子
**[1] 超市分类场景** <br>
Category是不同货架，Label是贴在商品上的描述信息标签，Tag是超市顾客能够自行添加的标记，比如划算、性价比低，太大等。

**[2] 对于人的分类场景** <br>
人的Category是高级动物，人的Label是姓名、身高、体重etc，人的Tag是长不大、nb程序员、坏老公等。

## 应用到博客中
本博客只使用了Category标签和Tag标签对于博客文章进行分类，其中Doc文件夹下按照Category进行分类；Archive文件夹下按照Tag进行分类。Doc文件夹下的分类按照文章内容进行客观的分类。Archive文件夹下的文章尚未总结出清晰、有效的主观博客文章分类方式。<br>
关于可能实用的Tag标签类别的思路：同一篇文章可以具有多个Tag，必须定义的Tag标签有：[`Year Tag`]...，可选的标签有：[`Like`]，[`Unlike`]，[`TODO`]，[`Revise Needed`]，[`Review Needed`]...(待补充)。

## Reference

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内 <br>
> 8 `<center><img src="/img/in-post/economics_4/xxx.png" width="60%"></center>`
