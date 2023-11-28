---
layout: post
title: "cpp - mutable"
subtitle: 'mutable关键字知识整理'
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-05-22 21:45
lang: ch 
catalog: true 
categories: cpp 
tags:
  - Time 2021
  - cpp fundermentals
  - todo
---

## Mutable关键字
可变的，允许修改类成员声明可变即使包含对象被声明为常量。Mutable用于在局部打破const修饰的类对象的const数据成员的限制：即使类对象的所有数据成员被声明为const常量，通过在类声明中添加mutable关键字可以让其被修改。

```c++
class Screen{
public:
    inline void move(int r, int c);// member func
private:
    string _screen;
    string::size_type _cursor;// no.1 普通声明
    mutable string::size_type _cursor;// no.2 mutable声明
    short _height;
    short _width;
};
inline void Screen::move(int r, int c){// def _cursor move
    if(checkrang(r,c)){
        int row = (r-1) * _width;
        _cursor = row+c-1;// 修改了光标的位置
    }
}
const Screen s(5, 5);// 构造了一个Screen类的常量对象 对象中的成员不能被修改
s.move(3, 4); char c=s.get(); // 错误! 读取(3,4)处的内容 改变了const数据成员_cursor
```
**no.1 普通数据成员声明的限制** 将Screen类对象声明为const，目的是不想修改Screen对象光标处的内容(_screen)，但是允许对象内数据成员_cursor进行移动(修改_cursor位置)。如果按照no.1普通声明的方式，那么由于定义了一个常量Screen类对象，那么其所有数据成员都被声明为const类型，move函数将无法正常使用，const变量_cursor无法修改。<br>
**no.2 mutable数据成员声明的优势** 使用no.2mutable声明方式允许：即使将Screen对象(及其所有数据成员)声明为const类型，还是能够对mutable变量_cursor成进行修改。即使这个mutable数据成员被用在类的const常量对象中，或者在const修饰this的成员函数中，被声明为mutable的数据成员总是可变的。

## Mutable关键字应用的例子 - TODO
Mutable关键字在**线程安全队列**中，多线程加锁的时，需要将mutex成员声明为mutable的。

## Reference
> 《cpp primer》3rd chapter 13.36 
> https://www.zhihu.com/question/64969053 <br>
> https://en.cppreference.com/w/cpp/language/cv <br>


> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内 <br>
> 8 `<center><img src="/img/in-post/economics_4/xxx.png" width="60%"></center>`
