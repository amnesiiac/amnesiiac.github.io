---
layout: post
title: "cpp - default constructor"
subtitle: 'c++默认构造函数知识整理'
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
### 类的初始化
为了能够正确、安全的使用定义的类类型，我们必须确保类被正确的初始化。一些特定的场景下，我们需要初始化一个对象以表明这个对象没有得到一个有意义的初始值，或者不知道为它提供什么样的初始值；此时使用缺省构造函数(default constructor)对其进行初始化。

### 使用显式初始化表进行初始化
如果一个类的所有数据成员都是公有的，则可以使用显示初始化列表进行初始化(封装类型：protected、private不能如此方式初始化、抽象数据类型也不能如此初始化)。初始化列表根据数据成员在类中声明的顺序进行按位解析。<br>
```c++
class Data{
    public:
        int ival;
        char *ptr;
};
int main(){
    Data d1={0, 0};// 按照数据成员在类中<声明顺序>进行解析
    Data d2={1024, "Anna Livia Plurabelle"};// ival=1024  char*[]="..."
}
```
显式初始化列表的应用场景：在特定应用中，用常量值初始化大型数据结构比较有效。例如：使用初始化列表初始化一个调色板类、向一个程序文本中注入大量的常数值(复杂地理模型中的控制点和节点值)，在此类场景下，显示初始化可以在装载时刻完成，节省了构造函数的启动开销。


### 使用缺省构造函数进行初始化
通常来说，使用构造函数进行初始化较显式初始化列表较好。缺省构造函数是指：不需要用户指定实参就能够被调用的构造函数，但这并不意味它不能接受实参，只是限制了它接受的实参必须有缺省值。
```c++
class Account{
    public:
        Account();// no.1 缺省(默认)构造函数声明 - 和类同名 不能指定返回类型
        Account(int num=0);// no.2 缺省构造函数 - 实参必须有缺省值 提供了实参名
        Account(string="Melon", int=3);// no.3 缺省构造函数 - 未提供实参名
};
```
### 调用缺省构造函数进行初始化的场景
```c++
#include <iostream>
#include <vector>
using namespace std;
class Myclass{// 定义一个myclass类 - no.1/no.2 构造函数同时只能定义一个
    public:
        Myclass(){// no.1 无参数缺省构造函数
            val=1;  cout<<"no-param default constructor is called"<<endl;
        }
        Myclass(int i=10){// no.2 带缺省参数构造函数
            val=i;  cout<<"param default constructor is called"<<endl;
        }
    private:
        int val;
};
class Inheritclass: public Myclass{// 定义一个继承自myclass的类
    public:
        Inheritclass(){
            cout<<"derived class default constructor is called"<<endl;
        }
};
class Otherclass{// 定义一个包含myclass的数据成员的类
    public:
        Otherclass(){
            cout<<"class which has a <Myclass> member is called"<<endl;
        };
    private:
        Myclass member;
};
```
下面test_default_constructor_call函数中，从no.1-no.7总共7种场景下，会调用到myclass类的缺省构造函数。
```c++
void test_default_constructor_call(){
    // no.1 调用缺省构造函数构造myclass类对象 
    Myclass mc;
    // no.2 new-expression 调用缺省构造函数构造对象
    Myclass *ptr = new Myclass;
    Myclass *ptr = new Myclass();
    // no.3 静态分配myclass类型数组 调用5次缺省构造
    Myclass mc_arr[5];
    // no.4 动态分配myclass类型的数组 调用5次缺省构造
    Myclass *ptr_arr = new Myclass[5];
    // no.5 作为标准库容器类型 调用5次缺省构造
    vector<Myclass> vec(5);
    // no.6 构造派生类对象 先后调用Myclass & Inheritclass类缺省构造
    Inheritclass inherit_mc;
    // no.7 myclass类对象作为其他类的成员 先后调用Myclass & Otherclass 缺省构造
    Otherclass other_mc;
}
```

### 声明、定义、使用缺省构造函数的注意事项
**[1]** 如果没有为定义的类声明**\<任何\>**构造函数，编译器会自动生成一个默认无参数构造函数，该函数体为空，且什么都不执行。 <br>
```c++
class Myclass{
    public:
        ...// 没声明or定义任何构造函数
        Myclass(){}// 编译器自动生成一个无参数默认构造
    private:
        int val;
};
```
**[2]** 在上述需要调用缺省构造函数的场景下，如果没有找到(或不能访问)缺省构造函数(自定义的or编译器合成的)，编译错误。<br>
**[3]** 一定要注意，不是所有的类都具有缺省构造函数(没有自定义缺省构造函数or显式的禁用了缺省构造函数)，如果使用上一节中的7种方法调用缺省构造函数可能引发错误。<br>
**[4]** 如果派生类的基类没有定义缺省构造函数，则派生类也不会定义缺省构造函数，因为即使定义了派生类的缺省构造函数，也不能初始化基类的成员。<br>
**[5]** 如果派生类的基类定义了一个private的缺省构造函数，那么继承类不能定义**(需要利用基类缺省构造函数)**的继承类构造函数；此时，继承类只能通过：**(a)**显式调用基类开放的构造函数api、**(b)**或者声明为基类的友元类、**(c)**或者通过基类开放的static数据成员访问接口**(称为单例模式)**三种方式来正常运转。
```c++
class inherit;// 继承类声明
class base{
    friend class inherit;// [Method-2] 将继承类声明为友元(多次一举)
    public:
        base(int);// [Method-1] 开放的单参数构造函数api
        static base* getbase(){// [Method-3] singleton global access api
            static base tmp;
            return &tmp;
        }
    private:
        int val;
        base(){}// 将基类默认构造函数设置为私有
};
base::base(int i){
    val=i;
}
class inherit: public base{
    public:
        int val;
        // 一组编译期错误的构造函数定义方法 - 继承类构造函数无法访问基类缺省认构造
        // base::base() is private within this context: inherit(...){} 
        inherit(){}
        inherit(int a){}
        inherit(inherit *ptr){}
        // [Method-1] 继承类构造函数的定义方法 - 显式调用基类开放的单参数构造函数接口
        inherit():base(0){}// right
        inherit(int a):base(a){}// right 
        inherit(inherit *pc):base(0){}// right 
    // [Method-3] singleton - 所有构建的inherit类对象都继承自同一个base类对象
    private:
        base *ptr_base;
    public:
        inherit():ptr_base(base::getbase()){}
};
int main(){
    // inherit inh;
    return 0;
}
```
**[6] [提供non-trivial缺省构造函数的4种场景]** <br> 在下面展示的(a)(b)(c)(d)4种场景中，如果用户没有显式地提供缺省构造函数，则编译器将自动生成一个non-trivial的构造函数。其他场景下编译器将提供一个都不做的trivial构造函数。换一种角度理解：trivial缺省构造函数仅在类中含有trivial成员的时候才调用，并且仅仅会设定数据成员中trivial成员的默认值；当类内所有成员都是non-trivial的，则合成non-trivial缺省构造函数。 

**[trivial & non-trivial成员]**：non-trivial成员的初始化可以交给编译器来做(implicit non-trivial)，而trivial成员的初始化属于程序员的职责(即使编译器合成了trivial缺省构造，也不会进行初始化)。关于trivial和non-trivial以及POD(plain old data)概念的介绍可以参考[博客](/cpp/2021/06/01/post-cpp-trivial-nontrivial-pod/)。<br>
```c++
class One{
    public:
        One();
        One(int);
};
class Two{// two类包含one类型(含有缺省构造函数)的数据成员 two类没有自定义构造函数
    public:
        One o;// 编译器将为two类提供一个implicit non-trivial构造 
        char *str0;// two类数据成员的初始化是程序员的职责 (trivial成员初始化)
};
```

**(a)** B类型没有任何user-defined构造函数，并且B中含有一个A类型成员a，并且A类型中含有缺省构造函数，此时需要为B类型生成隐式缺省构造函数，并且为了初始化a，缺省构造函数定义成implicit non-trivial的。
```c++
class A{};// A类型含有缺省构造函数
class B{// 编译器会为B类型创建implicit non-trivial的构造函数 cause:需要调用A类构造函数
    private:
        A a;// B类含有一个A类型成员
};
```
**(b)** 类型B继承自类型A，如果A类含有or能访问到缺省构造函数，且继承类B没有定义**\<任何\>**构造函数，则编译器会为B提供基类A的构造函数。

**(c)** 类中声明了一个虚函数or类继承了基类的虚函数。 在这两种情况下，编译器需要为该类维护一个虚函数表(virtual function table)用于存放virtual functions的地址，编译器也需要为该类的每一个对象实例合成一个虚指针(virtual pointer)用于存放其专属的虚函数表。因此，声明了虚函数的类的缺省构造函数是non-trivial的：需要缺省构造函数为虚指针、虚函数表进行初始化操作。

**(d)** 继承自虚基类的类。由于继承自virtual base class，需要维护一个指针or索引来指向派生类共享的virtual base class区域。
```c++
class X{
    public:
        int x;
};
class A: public virtual X{
    public:
        int a;
};
class B: public virtual X{
    public:
        int b;
};
class C: public A, public B{
    public:
        int c;
};
// [Problem] 编译期不能确定ptr指针类型 进而不能确定ptr指向哪个virtual base class对象 
// [Method] 执行时为每个virtual base class维护一个指针以确定在虚基类表中的偏移
// [Details] 编译器提供特殊的缺省构造函数 - 实现了运行时对象<动态分配管理指针>初始化
void func(const A *ptr){
    ptr->i=1024;
}
void func(const A *ptr){
    ptr->_virtual_base_class_X->i=1024;// 维护一个虚基类指针来进行<类型动态管理>
}
```

### 空类、含有虚函数的类、派生类占用内存情况
空类可以被实例化，并且实例化后，在内存中拥有独一无二的地址(以区别不同的实例)，编译器给空类加一个字节用于占位。但是当空类作为基类时，该空类的占位符被优化掉，因此占用内存空间为0；这种内存优化机制称为空白基类最优化(empty base optimization-EBO 或 empty base classopimization-EBCO)。
```c++
#include <iostream>
using namespace std;
class one{};// no.1 空类
class two{// no.2 含有虚函数的类 - 编译器维护了一个指向虚函数的指针 x64系统中占用8个字节
    virtual void func()=0;
};
class three: public one, public two{};// no.3 派生类 - 空白基类优化不占用内存
int main(){
    // 输出各个类占用内存结果: 1  8  8(not 9!)
    cout<<sizeof(one)<<" "<<sizeof(two)<<" "<<sizeof(three)<<endl;
    return 0;
}
```

### 空类定义时生成的6个成员函数
定义一个空类None如下：
```c++
class None{};// 定义一个空类
```
上述定义的空类等价于显式提供了6个构造函数类：
```c++
class None{
    public:
        None();// no.1 缺省构造函数
        ~None();// no.2 析构函数
        None(const None &);// no.3 拷贝构造函数
        None & operator=(const None &);// no.4 赋值操作符
        None * operator&();// no.5 取地址操作符
        None * operator&()const;// no.6 取地址操作符const版本

};
```
上述编译器自动生成的成员函数的默认实现：
```c++
inline None::None(){}
inline None::~None(){}
inline None::None(const None&){
    ... // 对于类的non-static成员进行<逐成员>拷贝
    ... // 固定类型的对象拷贝构造从'源对象'到'目标对象'的<按位>拷贝
}
inline None & None::operator=(const None &){
    ... // 对于类的non-static成员进行<逐成员>赋值
    ... // 固定类型的对象拷贝构造从'源对象'到'目标对象'的<按位>赋值
}
inline None * None::operator&(){
    return this;
}
inline const None * None::operator&(){
    return this;
}
```


## Reference
> inside the cpp object model chapter 2 p37 <br>
> cpp primer 3rd chapter 14.2 p567 <br>
> cpp primer 5th chapter <br>
> https://blog.csdn.net/livecoldsun/article/details/25478313 <br>
> https://bbs.csdn.net/topics/300231407?page=2 <br>
> https://www.cnblogs.com/lxy-xf/p/11038207.html <br>
> https://blog.csdn.net/taiyang1987912/article/details/43485569 

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内
