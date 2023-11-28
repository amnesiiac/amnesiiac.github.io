---
layout: post
title: "cpp - name resolution"
subtitle: 'c++中名字解析相关知识整理'
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-05-29 09:42
lang: ch 
catalog: true 
categories: cpp 
tags:
  - Time 2021
  - cpp fundermentals
---

## 类域中的名字解析
**[1] 在类定义中的名字(不包括inline函数定义中的名字以及缺省实参名字)** <br>
```c++
typedef double Money;// no.2 在类定义之前的名字空间域中出现的名字被考虑
class Account{
    ...// no.1 在类内Money名字之前的名字被考虑
    private:
        static Money _interestrate;// Money名字解析
        static Money interest();
        ...// no.3 在类内Money名字之后的名字不被考虑
};
... // no.4 类所在的域中 声明在类定义后的名字不被考虑
```
**[2] 在类的成员函数定义中的名字的解析** <br>
```c++
class Account{
    ...// no.2 <所有>类成员的声明都被考虑
    private:
        static Money _interestrate;
        static Money increase_interest(double);
};
...// no.3 在成员函数定义之前的名字空间域中的名字被考虑
Money Account::interest(double incr){// no.1 在成员函数定义的局部域中的名字被考虑
    _interestrate += incr; // incr名字解析
```
**[3] 在类的inline函数中的名字解析(分为类内定义、类外定义两种情况)** <br>
```c++
int _height;// no.3 inline函数定义在类内 则在类域声明之前的名字被考虑
class Screen{
    public:
        Screen(int _height){// no.1 在inline函数局部域中的名字被考虑
            _height=0;
        }
    private:
        short _height;//no.2 在类域中声明的<所有>成员名字都被考虑
};
int _height;// no.3 定义在类外的inline函数定义之前的名字也被考虑
Screen::Screen(int _height){
    _height=0;
}
```
**[4] 类内函数成员中提供的缺省实参名字解析** <br>
类类型的缺省实参必须是static成员，如果缺省实参采用了一个non-static成员，则编译期无法确定缺省的值而报错。非静态成员是this指针的一部分，this指针只有在实例化时才确定其值。<br>
作为类成员函数缺省实参的条件：只要在编译器类声明时，能够获取其值的东西都可以用作类的缺省实参。例如：定义在类域之前的名字可用于类函数成员的缺省实参。
```c++
class Screen{
    public:
        Screen & clear(char=background);// 缺省实参只能是static成员
    private:
        static const char background = '#';// no.1 所有声明在类域内的名字都被考虑
};
```
**[5] 类的静态成员定义中的名字解析** <br>
```c++
...// no.2 类声明之前名字空间域中的名字被考虑
class Account{
    private:
        static double _interestrate;// no.1 <所有>类成员的名字都被考虑
        static double interest();
};
...// no.2 类的静态成员定义之前的名字被考虑
double Account::_interestrate = 0.12;
```
**[6] 包含在外层类内部的嵌套类中的名字解析(不含inline函数名、缺省实参名)** <br>
```c++
enum ListStatus{Good, Empty, Corrupted};// no.3 最后考虑声明在<外层类之前>的名字
class List{
    public:
        ...
    private:
        ...// no.2 再查找外层类中定义在<ListItem类名之前>的名字
        class ListItem{
            public:
                ...// no.1 考虑定义在nested-class内部的名字 => 
                // no.1a 考虑声明在<xxx名字之前>的嵌套类的成员声明 
                // no.1b 考虑在嵌套类内的 <在xxx之前>的出现的 外层类的成员声明
                auto xxx;// xxx的名字解析顺序 no.1a -> no.1b -> no.2 -> no.3
        };
        ...// no.2 声明在nested-class之后的名字不用于名字解析
};
```
**[2] 如果嵌套类在外层类中声明，定义在外层类域之外，名字解析如下：** <br>
```c++
enum ListStatue{Good, Empty, Corrupted};// no.3 考虑定义在<外层类之前>的名字声明
class List{
    public:
        ...// no.2 考虑<所有>在外层类内的声明
    private:
        class ListItem;// nested-class声明
};
class List::ListItem{// 在外层类外的nested-class定义
    public:
        ...
        // no.1 考虑声明在<xxx名字之前>的嵌套类的成员声明
        auto xxx;// xxx的名字解析顺序名字解析顺序 no.1 -> no.2 -> no.3
};
```

## 命名空间中的名字解析 
类域中讨论的内容都会建立在全局名字空间上的。这一部分考虑嵌套类及其外部类定义在namespace中，名字解析的流程。关于嵌套类相关知识，可见[这篇博客](/cpp/2021/05/29/post-cpp-nested-class/)进行理解；关于命名空间，可参考[这篇博客](/cpp/2021/05/07/post-cpp-namespace/)进行理解。
```c++
// ---------- primer.h ----------
// no.5 最后考虑全局名字空间域中cpp_primer名字空间域中action定义之前声明的名字
namespace cpp_primer{// no.4 接着考虑cpp_primer名字空间域中action定义之前声明的名字
    class List{// no.3 接着查找List类的<完整域>
        private:
            class ListItem{// no.2 然后再检查ListItem的<完整域>
                public:
                    int action();
            };
    };
    const int someval = 365;
}
// ---------- primer.c ----------
#include "priemr.h"
namespace cpp_primer{
    int List::ListItem::action(){// no.1 首先考虑成员函数的局部域 
        int local = someval;// 正确 someval的名字解析流程
        double result = calc(int){}// 错误 calc尚未被声明
    }
    double calc(int){...}// 在action函数定义之后被声明
}
```
关于多层命名空间以及含有多层嵌套类、外层类的名字解析，只需要按照域访问顺序的反方向进行查询即可，如上述action函数在全局域中显示通过域访问作用符进行访问方式如下：
```c++
::cpp_primer::List:
```

## Reference
> c++ primer 3rd chapter 13.9 p545 <br>

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内 <br>
> 8 `<center><img src="/img/in-post/economics_4/xxx.png" width="60%"></center>`
