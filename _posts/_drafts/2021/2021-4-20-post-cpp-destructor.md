---
layout: post
title: "cpp - destructor"
subtitle: 'c++中析构函数知识整理'
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: true
date: 2021-04-20 23:50
lang: ch 
catalog: true 
categories: cpp
tags:
  - Time 2021
  - cpp fundermentals
---
## 析构函数
构造函数用于为类的对象获取相应资源，和构造函数相反，析构函数用于在对象生命期即将结束时释放相应资源。具体地，当类的对象离开其作用域，或者使用delete表达式应用于指向该对象的指针时，自动调用析构函数销毁对象、释放相应资源。

## 一个典型的析构函数
```c++
class Account{
    public:
        Account();// default ctor
        explicit Account(const char*, double=0.0);// ctor: forbid type convert
        Account(const Account &);// copy ctor
        ~Account();// dtor: dtor不能有返回值、参数表 -> 它不能被重载
    private:
        char *_name;
        unsigned_int _acct_no;
        double _balance;
};
inline Account::~Account(){
    delete[] _name;
    return_acct_no(_acct_no);// 将销毁户号重新置为‘可用’
}
```

## 并不是总需要自定义的析构函数
准确的讲，析构函数用于释放哪些：在类对象构造函数中获得的资源(释放在构造函数中构建的Mutex)或在对象生命期中获得的资源(delete操作符new申请的内存)。

下面定义了一个不需要显式定义析构函数的类：Point3d对象不需要显式定义析构函数，因为它的数据成员在构造、生命期没有显式的申请资源，因此在对象生命期结束后被编译器执行自动释放。
```c++
class Point3d{
    public:
        ...
    private:
        float x, y, z;
};
```

## 析构函数的功能不仅局限在释放资源
析构函数除了释放构造函数、对象生命期中申请的资源，还可以用于执行对象即将被销毁之前的任何任务。<br>
例如：评估程序性能的常见技术是构建一个**Timer类**，Timer类的构造函数用于启动程序时钟，Timer类的构造函数用来停止时钟。利用Timer类我们可以在特定需要计时的程序段中，构建一个Timer类的对象：
```c++
{
    ... // 关键代码段start
    #ifdef PROFILE
        Timer t;// 构建一个Timer计时器对象 -> 用于记录关键代码段执行时间
    #endif
    ... // 关键代码段执行
}// 关键代码段end -> Timer t被销毁 -> 调用Timer类析构函数 -> 停止时钟记录
```

## 析构函数和placement-new成对使用
new expression包含了两层含义：new-operator以及placement-new。前者负责申请特定内存，后者用于在指定的内存上创建类型对象。析构函数和pure-delete操作符则分别用来销毁特定类型对象、释放内存区域。
```c++
char *ptr1 = new char[sizeof(Img)];// new-operator: 申请内存
Img *ptr2 = new (ptr1) Img("The Mona Lisa");// placement-new: 在内存上构建对象
delete ptr2;// Not Good! 释放对象的同时删除了申请的内存
ptr2->~Img();// Good! 销毁ptr2指向的对象但不释放申请内存 - 方便重用
delete ptr1;// Good! 只释放申请内存 不涉及重复销毁内存上的对象!
```

## inline的析构函数可能会导致代码膨胀
将析构函数声明为inline本质上是一种空间换取时间的做法：通过在每个析构函数调用点上进行展开的方式，来节约在调用点上调用析构函数的时间。如果希望优化掉析构函数被在各调用点展开，则可以采用删除inline声明、或改写代码，减少析构函数调用点的方式进行。参考下面的例子进行理解(注意inline析构函数需要根据实际情况进行使用)：
```c++
Accounc acct("Mona Lisa");
int flag;
... // set flag
switch(flag){
    case 0:
        ... // operate on acct;
        return;
    case 1:
        ... // operate on acct;
        return;
    case 2:
        ... // operate on acct;
        return;
    ... // 很多个case 在每个case return之需要将inline析构进行展开 - code explosion
}
```
下面展示了一种代码改写的方式，能够有效加少上述inline析构函数在调用点被展开导致的代码膨胀。
```c++
Account acct("Mona Lisa");
int flag;
... // set flag
switch(flag){
    case 0:
        ... // operate on acct;
        break;
    case 1:
        ... // operate on acct;
        break;
    case 2:
        ... // operate on acct;
        break;
    ... 
    return;// 通过改写程序结构 减少程序返回点的个数来减少代码膨胀
}
```
        
## Reference
> cpp primer 5th chapter 13.1.3 p444 <br>

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内
