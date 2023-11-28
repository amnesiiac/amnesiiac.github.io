---
layout: post
title: "cpp - typedef"
subtitle: 'c++中typedef关键字知识整理'
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: true
date: 2021-04-24 17:09
lang: ch 
catalog: true 
categories: cpp
tags:
  - Time 2021
  - cpp fundermentals
---
### typedef简介
`c++ primer 3nd p104`

typedef为我们提供了一种类型定义措施，可以用来为内置类型或者用户自定义类型引入一个助记符号。需要注意的是typedef定义的首先是一个类型，其次，它是一种类型的助记(简化)符号。
```c++
// 定义 typedef定义方式简单直接
typedef double wages;
typedef vector<int> vec_int;
typedef vec_int test_scores;
// 使用 typedef定义的名字可以完全充当一个类型名来使用
wages hourly, weekly; 
vec_int vec1;
test_scores vec2;
```
### typedef用法
1 可以被用于程序文档的辅助说明。2 可以用于降低声明的复杂度如345。3 增强复杂模版定义的可读性。4 增强指向函数指针定义的可读性。5 增强指向类的成员函数的指针的可读性。

### typedef用法中注意事项
```c++
typedef char *cstring;// typedef 定义了cstring类型
extern const cstring cptr;// 使用自定义类型名对于cptr进行定义
// 问题：cptr是什么类型？
const char * cptr; // 错误 - 相当于把typedef类型按照'宏'规则进行展开
char * const cptr; // 正确 - 相当于把typedef类型看成一个'类型名'进行展开
// 注意 typedef的定义不能够完全按照'宏'的方式展开 按照定义类型直接原位替换是危险的
// 分析cptr类型:
// 问题的核心在于const修饰的是什么 显然将cstring看成一个类型如 'xxx'带入
// 上面的声明变成 extern const xxx cptr -> const修饰的是cptr 于是const应该临近与cptr
// 因此，在使用typedef类型转化时，需要考虑带有const和*或者&的情况 
// 因为const的作用域不能无视*/&
```

## Reference
> 《c++ primer》- N.Gregory Mankiw <br>

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内
