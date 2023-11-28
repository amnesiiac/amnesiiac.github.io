---
layout: post
title: "cpp - initialization list"
subtitle: 'c++中成员初始化表相关知识整理'
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-06-05 21:35
lang: ch 
catalog: true 
categories: cpp 
tags:
  - Time 2021
  - cpp fundermentals
---
## 1 成员初始化表和在构造函数体中赋值初始化的区别
### 1.1 构造函数的初始化&赋值阶段
**(1)** 隐式or显式的初始化阶段，**(2)** 一般计算阶段。**(1)** 初始化阶段可以是隐式的or显式的，具体取决于是否存在初始化表。隐式初始化时，首先按照继承声明的顺序调用当前类的base class的缺省构造函数，然后再按照成员声明顺序调用每个数据成员类型的缺省构造函数。**(2)** 一般计算阶段由构造函数体内所有语句构成，在这个阶段对类的数据成员的操作被看成是赋值而不是初始化。

下面的程序针对同一构造函数声明提供了两种实现方式：一种使用成员初始化列表进行，另一种将类成员在构造函数体中进行赋值。两种实现方式结果一致，但是过程原理不相同。
```c++
#include<string>
class Account{
    public:
        Account();
        Account(const char *, double=0.0);
        Account(const string &, double=0.0);
        Account(const Account &);
        ...
    private:
        unsigned int _acct_no;
        double _balance;
        string _name;
};
inline Account::Account(const char *name, double balance)// no.1 explicit init
// 显式初始化
    :_name(name),_balance(balance){
    _acct_no = get_unique_acct_no();// derive unique user id
}
inline Account::Account(const char*name, double balance){// no.2 implicit init
    _name = name;  _balance = balance;
    _acct_no = get_unique_acct_no();
}
```
#### 1.1.1 类类型成员的初始化&赋值
上述代码中，**(no.1)**执行显示初始化：显示地通过成员初始化列表按照成员声明顺序进行初始化，函数体中之进行申请用户id赋值操作。
**(no.2)**隐式初始化：首先隐式执行string类的缺省构造函数将\_name初始化，然后在函数体中\_name被再次赋值。通过代码理解第二种方式在某些情况下的冗余：
```c++
inline Account::Account(const char *name, double balance){// implicit init
    _name="";// 将空字符串赋值给_name是多此一举的
    _balance = balance;
    _acct_no = get_unique_acct_no();
}
```
上面代码中给\_name赋值是不必要的，因为在执行隐式初始化的时候，已经调用了string类缺省构造函数对\_name进行初始化，具体可以显式的将这个机制用cpp pseudo code展示出来：
```c++
inline Account::Account(const char *name, double balance):_name(string()){
    _balance = balance;// 显示地将隐式构造函数初始化的过程写出
    _acct_no = get_unique_acct_no();
}
```
上述代码可以简写成：
```c++
inline Account::Account(const char *name, double balance){// implicit init
    _balance = balance;
    _acct_no = get_unique_acct_no();
}
```
整理上述代码中的观察：对于类类型数据成员的初始化和赋值是不同的，类类型数据成员必须在成员初始化列表(member initialization list)中进行初始化，而不是在构造函数体中被赋值。

#### 1.1.2 内置类型的初始化&赋值
一般情况下，内置类型数据成员的成员列表初始化和在构造函数体中的赋值是等价的：
下面代码中展示了两种内置类型数据成员的初始化or赋值方式。其中no.1方式为内置数据类型的初始化，no.2方式为内置数据类型的赋值。
```c++
inline Account::Account():_balance(0.0),_acct_no(0){}// no.1 implicit
inline Account::Account(){// no.2 
    _balance = 0.0;
    _acct_no = 0;
}
```
上述规则存在特殊情况：当内置数据类型为const类型or引用类型时，不能在构造函数体中进行赋值。如果const类型or引用类型在构造函数体中进行赋值，则产生编译错误。产生错误的原因很直接：const常量只支持初始化不支持赋值操作；而引用类型必须在定义时进行初始化。相关代码如下：
```c++
class ConstAccount{// const版本Account类 含有两个const数据成员
    private:
        const double _balance;
        int _no;
        int &_ref_no;
};
inline ConstAccount::ConstAccount(){
    _balance = 0.0; // wrong! const数据成员不能在ctor中赋值
    _ref_no = _no;// wrong! 引用数据成员不能在ctor中赋值
}
inline ConstAccount::ConstAccount():_balance(0.0),_ref_no(_no){}// right!
```

#### 1.2 类成员初始化顺序
**[1 初始化列表顺序]** 类成员的初始化顺序由它们在类中声明的顺序决定，和初始化表中的顺序无关。
```c++
class Account{
    public:
        ...
    private:
        unsigned int _acct_no;// no.1
        double _balance;// no.2
        string _name;// no.3
};
// 初始化顺序：_acct_no -> _balance -> _name
inline Account::Account():_name(string()),_balance(0.0),_acct_no(0){}
```
**[2 实际初始化顺序]** 类成员初始化时(如果有赋值语句)，总是在构造函数体中赋值前进行初始化。
```c++
class Account{
    public:
        ...
    private:
        unsigned int _acct_no;// no.3 -> 
        double _balance;// no.1
        string _name;// no.2
};
// 初始化顺序：_balance -> _name -> _acct_no
inline Account::Account(const char *ch, double bal):_name(ch),_balance(bal){
    // _acct_no总是在被赋值之前初始化
    _acct_no = get_unique_acct_no();// _acct_no被赋值
}
```
**[初始化类标顺序和实际初始化顺序不一致的问题&解决]** 当类的一个数据成员被另一个数据成员时，可能导致隐蔽性的错误：
```c++
class Account{
    public:
        ...
    private:
        double _balance;
        double _wage;
};
// 初始化顺序：_balance(此时_wage尚未初始化) -> _wage 
inline Account::Account(double val):_wage(val),_balance(_wage){}
```
上述程序中，\_balance数据成员使用了未经初始化的成员\_wage进行了初始化，其结果是未定义的。<br>
**[建议的做法]**：将使用其它数据成员进行初始化的类的内置成员放在构造函数体中进行(内置成员在初始化列表初始化和在函数体中赋值的性能、结果等价)。修正后代码如下：
```c++
class Account{...};
// 初始化顺序：_wage -> _balance 
inline Account::Account(double val):_wage(val){
    _balance = _wage;// _balance的赋值(和初始化等价)
}
```

## Reference
> cpp primer 3rd chapter 14.5 p587 <br>

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内 <br>
> 8 `<center><img src="/img/in-post/economics_4/xxx.png" width="60%"></center>`
