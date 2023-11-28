---
layout: post
title: "cpp - trivial or non-trivial & POD"
subtitle: 'c++中trivial or non-trivial & POD类型相关知识整理'
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-06-01 21:21
lang: ch 
catalog: true 
categories: cpp 
tags:
  - Time 2021
  - cpp fundermentals
---

## Plain Old Data
Plain Old Data(POD)是c++中的概念，用来说明class、struct、union的属性。Plain表明它是一个普通的类型(不包含虚函数、虚继承特性)，Old表明这种类型和c兼容。<br>
POD类型的两个特性：**(1)** POD类型支持静态初始化。**(2)** POD类型拥有和c语言一样的内存布局。<br>
这两个特性分别对应两个概念：**(1)** trivial classes。**(2)** standard-layout。<br>
具有上述两个特性的类称之为POD类，并且POD类中的非静态成员此时也是POD类型。下面分别对于这两个概念进行介绍。

### Trivial classes
一个trivial class需要具备基本条件。这一小结内容参考自：[cplusplus-doc](http://www.cplusplus.com/reference/type_traits/is_trivial/)。
> A trivial class is a class (defined with class, struct or union) which implies that: <br>
> **(1)** Uses the implicitly defined default, copy and move constructors, copy and move assignments, and destructor. 默认构造、析构函数、拷贝构造、移动构造、拷贝赋值、移动赋值函数都是implicit的。<br>
> **(2)** Has no virtual members. 没有虚成员。<br>
> **(3)** Has no non-static data members with brace- or equal- initializers. 没有带括号or等号初始化的非静态成员。<br>
> **(4)** Its base class and non-static data members (if any) are themselves also trivial types. 该类的基类、非静态成员本身都是trivial的。<br>

#### trivial & non-trivial类型实例
**[两种最简单形式的trivial class]**。
```c++
#include <iostream>
class Trivial_1{};// no.1 empty class are trivial
class Trivial_2{// no.2 class with all special member implicit are trivial
    int i;
};
```
**[trivial只能继承自trivial基类，否则为non-trivial的]**。
```c++
class Trvial_3: public Trivial_2{// base class are trivial
    Trivial_3()=default;// 这种形式的缺省构造不是用户提供的 是编译器提供trivial ctor
    int i; 
};
```
**[trivial类型和访问限定符无关]**。
```c++
class Trivial_4{// trivial property is nothing todo with access-specifier
    public:
        int i;
    private:
        int j;
};
```
**[trivial类型可以包含trivial类的数据成员、trivial类型的数组、以及非虚普通函数etc]**。
```c++
class Trivial_5{
    Trivial_1 i;
    Trivial_2 j;
    Trivial_3 k;
};
class Trivial_6{
    Trivial2 a[5];
}; 
class Trivial_7{
    Trivial_6 i;
    void func();// it's ok to have non-virtual functions
};
```
**[trivial类型可以包含static non-trivial成员 & static trivial成员]**。
```c++
class Trivial_9{
    int i;
    static NonTrivial_1 j;// no restrictions on static members 
};
```
**[使用自定义(explicit)的6种speicial func的类是non-trivial的]**。
```c++
class NonTrivial_2{
    NonTrivial_2():i(1024){};// user ctor forbid implicit ctor 
    int i;
};
class NonTrivial_3{
    NonTrivial_3();// user ctor forbid implicit ctor
    int i;
};
NonTrivial_3::NonTrivial_3() = default;// 虽然定义成了默认构造 但是是explicit
```
**[如下形式定义的default ctor是implicit的]**。
```c++
class Trivial_8{
    Trivial_8() = default;// implicit default ctor
    Trivial_8(int x):i(x){};// regular ctor is ok cause we have default ctor
    int i;
};
```
**[含有virtual func或者继承自含有virtual func的类是non-trivial的]**。
```c++
class NonTrivial_1: public Trivial_3{// no.1
    virtual void func();// virtual func members invoke non-trivial ctor 
};
class NonTrivial_4{// no.2
    virtual ~NonTrivial_4();// virtual things cannot be trivial
};
```

#### 使用c++11模版可以判断类型的trivial属性
这一小结内容参考自：[cplusplus-doc](http://www.cplusplus.com/reference/type_traits/is_trivial/)。
```c++
#include <iostream>
// 基本用法
std::is_trivial<TYPE>::value;
// no.1用法实例 判断类型Trivial_i是否是trivial的
std::cout<<std::is_trivial<Trivial_i>::value<<std::endl;
```

### Standard layout
一个standard layout class需要具备的基本条件。
这一小结内容参考自：[cplusplus-doc](https://www.cplusplus.com/reference/type_traits/is_standard_layout/)。
> A standard-layout class is a class (defined with class, struct or union) that: <br>
> **(1)** Has no virtual functions and no virtual base classes. 不能含有虚函数、虚基类。<br>
> **(2)** Has the same access control (private, protected, public) for all its non-static data members. 非静态成员的访问限定一致。<br>
> **(3)** Has no non-static data members in the most derived class. At most one base class with non-static data members, or has no base classes with non-static data members. 要么最底层派生类只有static数据成员，此时至多有一个基类有non-static成员；要么没有含有non-static成员的基类。<br>
> **(4)** Its base class (if any) is itself also a standard-layout class. 基类本身也是standarad layout。<br>
> **(5)** And, has no base classes of the same type as its first non-static data member. 该类的第一个non-static数据成员不是基类类型。<br>

上述(1)、(2)两条针对类型的standard layour成员构成；上述(3)、(4)、(5)针对其继承树的性质。

#### standard layout的实例
**[两种最简单形式的standard layout]**。
```c++
class StandardLayout_1{};// no.1 空类=standard layout
class StandardLayout_2{// no.2 简单类=sl
    int i;
};
```
**[关于(2)non-static成员的方式方式一致性]**。
```c++
// 满足(2)
class StandardLayout_3{
    private:
        int i; int j;// non-static成员的访问限定一致 - standard layout
};
// 不满足(2)
class Non_StandardLayout_4{// non-static成员限定访问方式不一致
    private:
        int i;
    public:
        int j;
};
```
**[关于(3)继承树中non-static成员的分配]**
```c++
// 满足(3) 基类不含non-static成员
class StandardLayout_5: public StandardLayout_1{// 满足(4)基类是<标准分布>.
    int i; int j;// 满足(5)第一个non-static不是基类类型
    void func();
};
// 不满足(3) 有一个基类含non-static成员 但是该类同样含有non-static成员
class Non_StandardLayout_5: public StandardLayout_2{// 满足(4)基类是<标准分布>.
    int i; int j;// 满足(5)第一个non-static不是基类类型
    void func();
};
```
**[关于(4)该类之多只能有一个基类含有non-static数据成员]**
```c++
// 满足(4) sl_1没有non-static成员 sl_5含有一个non-static成员
class standardlayout_6: standardlayout_1, standardlayout_5{};
// 不满足(4) sl_3含有non-static成员 sl_5含有non-static成员
class standardlayout_6: standardlayout_3, standardlayout_5{};
```
**[关于(5)该类的第一个成员不能是基类类型]**
```c++
class StandardLayout_7: Standardlayout_1{
    int i;// 满足(5)第一个数据成员不是基类类型
    StandardLayout_1 j;// 第二个成员是基类类型 但无所谓
};
class StandardLayout_8: StandardLayout_1{
    StandardLayout_1 i;// 不满足(5)第一个数据成员和基类类型相同
    int j;
};
```
**[含有virtual function的类不是standard layout]**
```c++
class Non_StandardLayout_9{
    virtual void func();// 
};
```
**[含有non-standard layout基类的类不是standard layout]**
```c++
// standard layout类的基类中不能有<非标准分布的类>
struct Non_StandardLayout_10: Non_StandardLayout_9{};
```

#### 使用c++11模版可以判断类型的standard layout属性
这一小结内容参考自：[cplusplus-doc](https://www.cplusplus.com/reference/type_traits/is_standard_layout/)。
```c++
#include <iostream>
// [基本用法]
std::is_standard_layout<TYPE>::value;
// [用法实例] 判断类型StandardLayout_i是否是<标准分布>
std::cout<<std::is_standard_layout<StandardLayout_i>::value<<std::endl;
```

## 使用c++11模版可以判断类型的POD属性
这一小结内容参考自：[cplusplus-doc](https://www.cplusplus.com/reference/type_traits/is_pod/)。
```c++
#include <iostream>
// [基本用法]
std::is_pod<TYPE>::value;
// [用法实例] 判断类型StandardLayout_i是否是<POD类型>
std::cout<<std::is_pod<class_type>::value<<std::endl;
```

## POD类型相比non-POD类型的优点
**(1)** 字节赋值。POD类型变量可以不使用构造函数、赋值操作符赋值，直接通过memset()、memcpy()初始化赋值。
**(2)** 兼容c内存布局。c++程序可以和c进行交互，或者可以和其他语言交互。
**(3)** 保证静态初始化安全有效。静态初始化很多时候可以提高程序性能，POD类型初始化更加简单。

## Reference
> https://blog.csdn.net/a627088424/article/details/48595525 <br>
> https://zhuanlan.zhihu.com/p/56161728#:~:text=POD类型是C%2B%2B中常见的概念，用来说明类%2F结构体的属性，具体来说它是指没有使用面相对象的思想来设计的类%2F结构体%E3%80%82%20POD的全称是Plain,Old%20Data，Plain表明它是一个普通的类型，没有虚函数虚继承等特性；Old表明它与C兼容%E3%80%82 <br>


> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内 <br>
> 8 `<center><img src="/img/in-post/economics_4/xxx.png" width="60%"></center>`
