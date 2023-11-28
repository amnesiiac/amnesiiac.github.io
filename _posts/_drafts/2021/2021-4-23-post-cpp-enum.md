---
layout: post
title: "cpp - enum"
subtitle: 'c++中enum关键字知识整理'
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: true
date: 2021-04-23 17:44
lang: ch 
catalog: true 
categories: cpp
tags:
  - Time 2021
  - cpp fundermentals
---
### enum简介
`c++ primer 3nd p91`

枚举类型基本用法：
```c++
// enum关键字 + 自定义枚举类型名(注意这是一个类型) + {枚举成员列表}
enum forms{shape, sphere, cylinder, polygon};
forms form1=shape; // forms是一个类型，定义了一个类型对象form1
```
对于枚举成员列表的成员，可以显示的赋值(所赋的值不一定唯一)，也可以指定第一个成员的初始值，后面成员依次+1。缺省初始值的情况下，枚举成员数值从0开始。
```c++
// 在下面的情况下，point2w=3 point3w=4 不指定初始值，自动+1
enum Points{point2d=2, point2w, point3d=3, point3w};
```
对于上面定义的forms类型form1对象，他可以参与表达式运算，也可以作为参数传递给函数，但是只能被同类型枚举成员初始化或者赋值。
```c++
std::cout<<form1<<" "<<form1*form1<<std::endl; // 正确 可以参与表达式运算
form1=3; // 错误 枚举类型对象只能被同类型成员赋值
form1=point2d; // 正确的赋值操作
// 但是 必要时 枚举类型可以被提升成算数类型 - 可以参与算数运算
int array_size=4; int result=0;
form1=point2d;
result=array_size*form1;// 正确的操作 枚举类型form1被提升成算数类型
```

情景：一个文件可能按照如下三个方式进行打开：输入(input)，输出(output)和追加(append)。我们可能想到使用如下的方式对这个文件打开方式进行传递：
```c++
// 一个普通但是有缺陷的做法
const int input=1; const int output=2; const int append=3;// 设置三个变量存放状态
bool openfile(string filename, int openmode);
openfile("myfilename", input); // 打开文件函数调用方式
```
上面的方法不能保证openfile函数在openmode接受的参数只来自input，output或者append，即有可能存在其他变量能够为这个参数赋值。枚举类型解决了这个问题。
```c++
// 使用枚举类型的做法
enum open_modes{input=1, output, append};// 指定了input的初始值 后面依次+1
bool openfile(string filename, open_modes om); // 使用枚举类型限定传入打开文件参数
openfile("myfilename", append);// 则按照上述定义 第二个参数只能接受枚举类型内的变量
openfile("myfilename", 1); // 即便第二个参数传递了一个可用的数值，仍然会报错
```
从上面的代码可知，枚举类型可以限定传入参数范围，并且可以限定传入参数的来源。

另外，我们可以用用单个变量声明枚举类型变量，如：
```c++
// 枚举类型的其他用法 - 枚举类型对象和枚举成员可以互相替代
open_modes om=input; // 使用定义过的input变量 对open_modes枚举类型进行声明
openfile("myfilename", om) // 并且将枚举类型对象 代替单个枚举成员
std::cout<<input<<" "<<om<<std::endl; // 打印的结果为“1 1”
// 枚举类型不能做的事
for(open_modes iter=input; iter!=append; ++iter);// 错误的写法 枚举类型不能用于迭代
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
