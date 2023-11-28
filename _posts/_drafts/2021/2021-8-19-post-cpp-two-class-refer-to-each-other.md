---
layout: post
title: "cpp - mutual reference of two classes"
subtitle: '两个类型之间相互引用时 头文件组织方案' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-08-19 11:01
lang: ch 
catalog: true 
categories: cpp 
tags:
  - Time 2021
  - cpp fundermentals
---
#### Problem
现在有两个类型A和B需要定义，A类型包含一个和B类型相关的数据成员，B类型包含一个和A类型相关的数据成员。

#### Solve#1 - 将A类型和B类型都放在同一个头文件中
**[wrongmethod-1] 类型A包含一个B类型的成员对象**
```cpp
#ifndef A_B_H
#define A_B_H
// 这种形式无法完成A类型、B类型的定义
class A{
    public:
        B b_obj_;// 编译失败 -> class B undefined
};
class B{
    public:
        A a_obj_;// 即使在class A定义前使用了前置类型声明 也会报错
};
#endif
```
**[rightmethod-2] 类型A包含一个B类型的成员指针**
```cpp
#ifndef A_B_H
#define A_B_H
class B;// 使用前置类型声明
class A{
    public:
        // 本质上只能定义成指针的原因是: 定义A类型时，编译器需要确定A类型占用的内存
        // 而A类型包含一个B类型对象，所以编译器需要确定B类型对象占用多少内存
        // 而B类型包含一个A类型对象，所以编译器陷入了循环往复的无间道...
        B *ptr_b_obj_;// 编译正确 -> 前置类型声明允许定义ptr & ref
        B b_obj_;// 编译错误 -> 前置类型声明不支持定义对象(需要内存)
};
class B{
    public:
        A a_obj_;// 编译正确 -> A类已经定义
};
#endif
```

#### Solve#2 - 将类型A和类型B分别放到两个头文件中都使用#include进行包含
**[wrongmethod-1] 使用"#ifndef ... #define ... #endif"循环保护解决循环包含导致无限循环的问题**
```cpp
// [head file#1: "a.h"]
#ifndef A_H
#define A_H
// 引入b.h头文件 - include相当于直接将B头文件中的内容复制粘贴到此处
#include "b.h"
class A{
    public:
        B b_obj_;// "a.h"中包含了"b.h" A类型中可以定义B类型的ptr & obj
};
#endif
```
```cpp
// [head file#2: "b.h"]
#ifndef B_H
#define B_H
// 引入a.h头文件 - include相当于直接将A头文件中的内容复制粘贴到此处
// 但是由于#ifndef宏的缘故 这里a.h头文件不能被正常引入 因此无法定义A类型的指针
#include "a.h"
class B{
    public:
        A *ptr_a_obj_;// 编译错误! 无法找到A类型相关定义
};
#endif
```
**[wrongmethod-2] 使用"#pragma once"循环保护解决循环包含导致无限循环的问题(分析同上)**
```cpp
// [head file#1: "a.h"]
#pragma once
#include "b.h"// 互相引用
class A{
    public:
        B b_obj_;
};
#endif
```
```cpp
// [head file#2: "b.h"]
#pragma once
#include "a.h"// 互相引用
class B{
    public:
        A *ptr_a_obj_;
};
#endif
```

#### Solve#3 - 将A、B类型分别定义在两个头文件中并使用前置类型定义
**[rightmethod-1] 使前置类型声明的方式解决Probelm问题**
```cpp
// [head file#1: "a.h"]
#ifndef A_H
#define A_H
class B;// B的前置类型定义 - 相当于在此处定义了类型B 具体内容在"b.h"中
class A{
    public:
        B *ptr_b_obj_;// 编译正确! 前置B类型定义只允许创建ptr & ref
        B b_obj_;// 编译错误! 前置类型声明不允许直接声明B类型对象
};
#endif
```
```cpp
// [head file#2: "b.h"]
#ifndef B_H
#define B_H
// 引入头文件a 下面class B的定义相当于对于"a.h"中的简短定义进行补充
#include "a.h"
class B{
    public:
        A a_obj_;
};
#endif
```
```cpp
// ["main.cpp"]
#include "b.h"// 只需要包含类型B的头文件即可
int main(){...}
```
**[rightmethod-2] 使用"extern class xxx"的形式指明类型定义在文件外部**
```cpp
// [head file#1: "a.h"]
#ifndef A_H
#define A_H
extern class B;// 使用extern声明B在头文件"a.h"外部定义
class A{
    public:
        B *ptr_b_obj_;// 编译正确!
        B b_obj_;// 编译错误!
};
#endif
```
```cpp
// [head file#2: "b.h"]
#ifndef B_H
#define B_H
#include "a.h"
class B{
    public:
        A a_obj_;
};
#endif
```
```cpp
// ["main.cpp"]
#include "b.h"
int main(){...}
```


## Reference
> https://blog.csdn.net/xiqingnian/article/details/41214539

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内 <br>
> 8 `<center><img src="/img/in-post/economics_4/xxx.png" width="60%"></center>` <br>
> 9 使用html设置图片文字环绕方式: <br>
    `<div>` <br>
        `<img src="/img_path" align="left" width="40%" hspace="" vspace=""/>` <br>
        `<p>paragraph1 around the picture</p>` <br>
        `<p>paragraph2 around the picture</p>` <br>
        `<p>paragraph3 around the picture</p>` <br>
    `</div>` <br>
> 10 `<font style="color:red; font-weight:bold">加粗蓝色</font>`用来设置字体颜色
