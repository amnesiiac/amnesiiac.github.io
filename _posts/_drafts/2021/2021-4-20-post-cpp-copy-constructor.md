---
layout: post
title: "cpp - copy constructor"
subtitle: 'c++拷贝构造函数知识整理'
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
## 为什么需要拷贝构造函数
用一个类的对象初始化该类的另一个对象称为按成员初始化，这种初始化是通过依次拷贝每个非静态成员来实现的。类的设计者可以通过自定义拷贝构造函数来代替缺省拷贝构造函数的行为。下面是一个需要自定义拷贝构造的例子：
```c++
class Account{
    public:
        Account(int i):_account_no(i){};// 屏蔽了implicit default ctor
        Account(const Account &);
    private:
        double _balance;
        char *_name;
        int _account_no;
        get_unique_account_no();// tool func to generate account_no
};
inline Account::Account(const Account &cpy):_balance(cpy._balance){
    _name = new char[strlen(cpy._name)+1];// 账户昵称可以直接拷贝 无关紧要
    strcpy(_name, cpy._name);
    _account_no = get_unique_account_no();// 每个账户拥有独立户号 直接拷贝不符合实际
}
int main(){
    Account a1(1);
    Account a2(a1);// 没有default ctor -> 调用拷贝构造创建a2 
}
```

## 拷贝构造函数的基本形式
如果一个构造函数的第一个参数是引用类型，且任何额外的参数都有缺省值，则这个构造函数是拷贝构造函数。
```c++
class None{
    public:
        None(None &);// no.1 copy ctor
        None(const None &);// no.2 copy ctor overload: 一般情况下 使用const版本
        explicit None(const None &);// no.3 通常情况下 拷贝构造不被定义成强制显式调用
};
```

## 合成的拷贝构造函数
和缺省构造函数不同，无论我们是否定义拷贝构造函数，编译器还是会为我们提供一个合成拷贝构造函数。**一般情况下**，合成拷贝构造函数会从给定对象中将所有non-static成员逐一拷贝到正在创建的对象中：对于内置类型成员直接拷贝；对于类类型成员调用其拷贝构造函数进行拷贝；对于内置类型数组成员采用逐元素拷贝，对于类类型数组成员采用逐元素调用拷贝构造来完成。**但某些情况下**，合成拷贝构造函数会用来阻止我们拷贝该类型对象。

## 直接初始化和拷贝初始化
**(1) 直接初始化**：编译器按照初始化表进行函数匹配来选择和参数最佳匹配的构造函数。**(2) 拷贝构造初始化**：编译器将`=`右侧作为初始化表创建一个对象(必要时会发生类型转化)，再将右侧对象拷贝给左侧对象。
```c++
#include <string>
// ---------- 直接初始化 ----------
string dots(10, '.');
string s1(dots);
// ---------- 拷贝初始化 ----------
string s2 = dots;
string null_book = "9-9999-99999-9";
string nines = string(100, '9');
"将一个类对象作为实参传递给非引用类型形参"
"从一个返回类型为非引用类型的函数返回对象"
"用{}列表初始化一个数组的元素or一个<聚合类>的成员"
```
关于\<聚合类\>的相关知识详见[博客](/cpp/2021/06/03/post-cpp-aggregate-class/)中的整理。另外某些类型会对所分配对象执行拷贝初始化：标准库中容器在使用`insert`和`push`成员时，容器对元素使用拷贝初始化；当使用`emplace`时，对容器中的元素使用直接初始化。

## 函数参数&返回值中的copy ctor
关于这一部分的内容，详见[博客](/cpp/2021/04/20/post-cpp-reference/)中\<函数返回值和返回引用的区别探讨\>小节中的内容。

## 对于拷贝构造初始化中的类型转化的限制
```c++
vector<int> vec1(10);// 正确 使用直接初始化 <=> 调用最匹配的构造函数
vector<int> vec2 = 10;// 错误 隐式地使用了一个explicit的拷贝构造函数进行类型转化
void func(vector<int>);  
func(10);// 错误 原因同上 
func(vector<int>(10));// 正确 使用直接初始化 <=> 调用最匹配构造函数
```

## 编译器堆对copy ctor做出的优化
```c++
// 默认情况下下面的code被执行拷贝初始化 =>
string book = "9-999-99999-9";
// 实际上 编译器可能略过copy ctor 将上述代码转化成直接初始化的形式 =>
string book("9-999-99999-9");// 编译器优化后的code 仍然要求copy-ctor是<可见>的
```

## Reference
> cpp primer 3rd chapter 14.2.3 p574 <br>
> cpp primer 5th chapter 13.1.1 p440 <br>

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内
