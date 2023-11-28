---
layout: post
title: "cpp - local class"
subtitle: 'c++中局部类相关知识整理'
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-05-31 13:44
lang: ch 
catalog: true 
categories: cpp 
tags:
  - Time 2021
  - cpp fundermentals
---
## 局部类(local class)
定义在函数体中的类称之为局部类。局部类只在定义它的局部域中可见。和定义在外层类中的嵌套类不同，定义在局部域中的类在局部域外边没有语法能够引用local-class的成员，因此局部里的成员必须定义在类定义中。由于不能在局部域外引用局部类成员，而类的静态成员必须在类外进行初始化，所以局部类不能包含static类成员。关于static为什么必须在类外初始化以及更多的知识详见[博客](/cpp/2021/05/06/post-cpp-static/)中的内容。

### 局部类中定义嵌套类
下面展示了在func函数定义的局部类中声明嵌套类的方法。一般而言，将局部域中的嵌套类声明为public而不是private的，局部域已经很大程度的限制了局部类的可访问空间，把局部类中的东西声明为私有一般是画蛇添足。
相比定义在名字空间(namespace)中的类，以及定义在名字空间中的嵌套类而言，局部类在局部域外没有提供访问的机制，封装的更加紧密。
```c++
void func(int val){
    class Bar{
        public:// 将嵌套类声明为public 而不是 private
             int barval;
             class Nested;// 嵌套类的声明
    };
    class Bar::Nested{// 嵌套类的定义 - 必须在外层类相同的局部域中进行定义
        ...
    };
}
```
### 局部类中名字的访问
**[1] 外层局部域访问局部类内的名字** <br> 外层类不能够直接访问局部类中的私有成员，除非将外层类声明为局部类的友元(...多此一举)。

**[2] 局部类内访问外层局部域中的名字** <br>
局部类访问外部的名字受到局部域的限制，因此只能访问外层局部域中定义的**类型名(typedef)**、**静态变量**、**枚举类型及枚举值**。
```c++
int a; int val;
void func(int val){
    typedef int INT;
    static int s_int;
    enum Loc{a=1024, b};
    class Bar{
        public:
            Loc locval;// 定义了一个枚举类型对象
            int barval;
            // 局部类成员函数的定义 
            void funcBar(Loc l=a){// 正确 局部域中能够访问外层域定义的枚举值
                INT test=0;// 正确 访问外层局部域中的类型名
                barval=val;// 错误 局部域定义的局部变量不能在嵌套域中访问
                barval=::val;// 正确 访问全局对象
                barval=s_int;// 正确 访问外层局部域中的静态变量
                locval=b;// 正确 访问外层enum类型枚举值
            }
    };
}
```
上述代码中的`barval=val`不能访问到外层局部域中的局部变量，并且`barval=val`也不会继续向外找到全局变量`val`，使用全局变量`val`必须通过`::val`来进行。

### 在局部类中的名字的解析
**(1)** 局部类中(不包括其成员函数定义)的名字解析过程：在外围域中查找出现在局部类定义之前的声明。**(2)** 局部类成员函数定义的解析过程：查找外围域之前，首先查找该类的完整域。**(3)** 特殊情况：如果查找到的声明使该名字的用法无效，则不考虑其他声明，参考下面的例子进一步理解：
```c++
// ---------- situation-1 ----------
int val;
void func(int val){// 定义了局部变量val
    class Bar{
        public:
            void funcBar(){
                barval=val;// 错误 不能在局部类中访问外层局部域中的局部变量
                barval=::val;// 正确 显式通过::访问全局变量
            }
    };
}
// ---------- situation-2 ----------
int val;
void func(){// 没有定义局部变量val
    class Bar{
        public:
            void funcBar(){
                barval=val;// 正确 能够在局部类中访问全局域中的变量
                barval=::val;// 正确 一种等价的访问
            }
    };
}
```
根据名字解析规则，局部类成员函数名字首先查找类完整域、再查找外层局部域，如果找到的同名变量无法访问(见`situation-1`)，则会导致本来在成员函数域中能够正常访问的变量无法被访问到(如`situation-2`)，除非使用域访问作用符`::`进行访问。

## Reference
> cpp primer 3rd chapter 13.12 p562 <br>
> cpp primer 5th chapter 19.7 p754 

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内 <br>
> 8 `<center><img src="/img/in-post/economics_4/xxx.png" width="60%"></center>`
