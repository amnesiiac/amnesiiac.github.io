---
layout: post
title: "cpp - static"
subtitle: 'static关键字知识整理'
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: true
date: 2021-05-06 16:35
lang: ch 
catalog: true 
categories: cpp 
tags:
  - Time 2021
  - cpp fundermentals
---

## 1 static类数据成员
自定义类的多有对象都共用一份数据成员的时候，应当将这个类的数据成员定义成静态成员。static成员的常用的场景：用于在类中维护程序为这个类定义了多少个对象，自定义智能指针类的引用计数器等。静态类成员时类的所有对象共有的，是该类的全局数据。非静态数据成员在类的每个对象中都有一份拷贝，而静态类成员在类的所有对象中只有一份。

根据上面定义描述，类的静态类成员大致和全局对象作用相近。同全局对象相比，类的的静态数据成员有两个优势：
**[1]** 类的静态数据成员没有污染全局名字空间，没有和全局名字空间中的变量相冲突的问题。
**[2]** 类的静态数据成员能够利用类的信息隐藏特性(information hiding)，而全局对象没有相关特性。类的静态数据成员能够被public、protected、privated所影响。

下面展示了static类数据成员的特性及声明方式。

```c++
class Account{
public:
    Account(double amount, const string &owner);
    inline string owner(){return _owner;}
private:
    static double _interestrate;// 所有的账户都共用相同的利率 共享的数据
    double _amount;// 每个账户都有不同的金钱数量
    string _owner;// 每个账户都有不同的所有者
};
```

### 1.1 static类数据成员的初始化
**static类数据成员初始化的一般情况：** static类数据成员应当在类外被初始化，并且static类数据成员初始化方式需要用类名字限定符修饰：
```c++
#include "account.h"
double Account::_interestrate = 0.0589;
```
**static类数据成员初始化特例：** 标准c++允许：static const integral(有序类型)数据成员可以在类内直接进行初始化，但仍然需要在类外进行定义。下面代码展示了这种特殊情况如何应用。
```c++
// integral:[unsigned int, int, unsigned long, long, unsigned char, char]
class Account{
public:
    ...
private:
    static const int namesize=16;// 常量值初始化integral类型 = const expression
    static const char name[namesize];// 常量表达式可以用来定义数组的长度
};
const int namesize;// static const int仍必须要在类外进行定义 但可以不指定初始值
const char name[namesize] = "savings account";// 必须在类外进行初始化
```
**需要注意的是：** 上面的代码中类内初始化的static const integral可以直接在类外使用，而不用使用类域限定符。这是因为，静态成员在类外的定义和类的成员函数一样，能够直接引用类内任何数据成员。
```c++
const char name[Account::namesize] = "savings account";
const char name[namesize] = "savings account";// 可以不通过类域修饰符直接访问类成员
```
**需要注意的是：** static成员变量的内存空间不是在声明类时分配，也不是在类创建对象时分配，而是在static初始化时分配。静态成员变量必须初始化，而且只能在类体外进行(初始化的位置参考非inline函数进行)。否则，编译能通过，链接不能通过。

**为什么static成员必须需要类外进行定义呢?** <br> 类的static成员和普通成员不一样，类的所有对象共用一份static成员。而static成员不是在类实例化的时候分配内存，而是在成员初始化的时候分配，因此如果static在类内进行初始化，则每个类对象创建都初始化一次并包含一份static成员，这是和static含义相矛盾。

### 1.2 static类数据成员的访问方式
static类的成员可以通过类的对象进行访问，也可以直接通过类名进行访问。
```c++
Account a; a._interestrate = 0.123;// 通过类的对象进行访问
Account &ref_a=a; ref_a._interestrate;// 通过类的对象的引用来访问
Account *ptr_a=&a; ptr_a->_interestrate;// 通过类对象的指针进行访问
double Account::_interestrate=0.589;// 通过类名进行访问(non-const)
```

### 1.3 static变量的生命期和作用域
**static变量具有全局变量的生命期，但是只能作用于属于自己的局部作用域。**

### 1.4 static变量的内存占用属性
静态数据成员在**全局数据区(静态存储区)**分配内存，由本类的所有对象共享，所以，它不属于特定的类对象，不占用对象的内存，而是在所有对象之外开辟内存。因此，对一个含有static数据成员的类的对象运用sizeof不会将static数据成员占用内存记入其中。因此，在没有类的实例对象存在时，静态成员变量就已经存在，我们就可以操作它。

静态数据成员和全局数据成员都存储在全局数据区(静态存储区)，全局数据区中的数据进行初始化时，可以赋初始值，也可以不赋初始值：全局数据区中的数据拥有默认的初始值；而动态数据区(heap、stack)中变量默认是垃圾值。垃圾值和野指针并称**垃圾二兄弟**，写程序时须多加谨慎。

### 1.5 static类成员的链接性
和全局对象一样，类的静态数据成员具有外部链接性，因此，在整个程序中只能拥有一个定义。因此，类的静态数据成员不能够在头文件中进行初始化，而是应当和非inline函数的定义放在相同文件中进行管理。关于链接性的相关知识，可以参考[博客](/cpp/2021/05/06/post-cpp-linkage/)进行回顾。

### 1.6 static类成员相比非static成员的特殊性
**[1]** static成员可以在类内声明成类类型，而非static成员只能被声明成类类型的指针或引用。
```c++
class Account{
private:
    Account *data_ptr;// 正确 非static成员可以被声明为类类型的指针
    Account &data_ref;// 正确 非static成员可以被声明为类类型的引用
    Account data_member1;// 错误! 
    static Account data_member2;// 正确 static成员可以使用类类型
};
```
**[2]** static成员可以在类内作为成员函数的缺省实参，非static成员不能作为类成员函数缺省实参。解释：非静态成员的值在编译期需要确定，但是非静态成员本身是this指针的一部分，在编译期不能确定其值，产生了矛盾。而static成员在编译期可以获取其值。但是定义在类域之前的非静态数据可作为类内函数的缺省实参。<br>
作为类成员函数缺省实参的条件：只要在编译期限类声明时，能够获取其值的东西都可以用作类的缺省实参。
```c++
int data_member1;// 定义在类声明之前非静态数据可以用作类内函数的缺省实参
class Account{
private:
    int data_member1;// 非static普通成员
    static int data_member2;// static数据成员
public:
    int func1(int=data_member1);// 错误 非static不能作为类成员函数缺省实参
    int func2(int=data_member2);// 正确 static成员可作为类成员函数缺省实参
};
```

## 2 static类成员函数
**应当定义成static类成员函数的两种情况** <br> 
(1) 如果类的成员函数除了静态成员之外，不访问任何其他的类数据成员，则可以被定义为static类成员函数。<br>
(2) 如果类的成员函数不访问任何类的数据成员，则应定义为static类成员函数(这类函数没什么实际意义)。这种函数的功能和调用它的类的对象无关(所有对象都同等对待)，调用它的结果也不会访问或者修改任何对象(non-static)。
```c++
// ----------- account.h ----------
class Account{
public:
    void raiseinterest(double incr);// no.1 non-static version
    static void raiseinterest(double incr);// no.2 static version
private:
    static double _interestrate;
};
// ----------- non-inline func file ----------
double _interestrate = 0.58;
// ----------- inline func file ----------
inline void Account::raiseinterest(double incr){// only use static member
    _interestrate+=incr;// 只对static成员进行操作的函数应定义为static函数
}
```
**static成员函数没有this指针** 在static成员函数中隐式、显式的使用this指针导致编译错误。<br>
**static成员函数的访问方式：**static成员函数的访问方式和static数据成员一样，可以通过引用、指针、类域作用符进行访问：
```c++
Account a; Account &ref_a=a; Account *ptr_a=a;
a.raiseinterest(0.02);// no.1 
ref_a.raiseinterest(0.02);// no.2 
ptr_a->raiseinterest(0.02);// no.3
Account::raiseinterest(0.02);// no.4 相当于static数据成员的专用修改接口 
```

**static成员函数使用注意事项** <br>
**(1)** 在类体外static成员函数定义中不能使用static关键字。static关键字只能在声明中使用。<br>
**(2)** 静态成员函数只能访问静态数据成员、静态成员函数。如果访问了非静态成员，则应当将其定义为non-static函数。<br>
**(3)** 非静态成员函数可以随意的访问静态成员，使用上述三种访问方式皆可。<br>
**(4)** 相比类的普通成员函数，类的静态成员函数不需要维护this指针，因此效率会略高。<br>
**(5)** static成员函数在类外定义时的域解析： 类外定义的变量将定义划分成两个域，位于定义变量之前的东西位于类域外，需要使用类名域访问修饰；位于定义变量后面的东西位于类域内，不需要使用类域访问修饰。
 ```c++
class Account{
    private:
        typedef double Money;
        static Money _interestrate;// 类外被定义变量
        static Money initinterest();
};
// 注意money必须加Account域限定 而initinterest不需要加域限定
// 被定义变量之前的东西不在类域内 而在被定义变量之后的东西在类域内
Account::Money Account::_interestrate = initinterest();
```

## Reference
> 《c++ primer》chapter 13.5 <br>
> https://zhuanlan.zhihu.com/p/37439983

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内
