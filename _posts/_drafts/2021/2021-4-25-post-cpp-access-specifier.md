---
layout: post
title: "cpp - access specifier"
subtitle: 'c++中acess specifier访问限定符整理'
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: true
date: 2021-04-24 20:16
lang: ch 
catalog: true 
categories: cpp
tags:
  - Time 2021
  - cpp fundermentals
---
### access specifier简介
访问限定符(access specifier)包括public，private和protected，分别称为公有，私有和保护类型限定符。三种限定符定义下的类的成员分别是公有成员，私有成员，和保护成员。按照访问限定的级别，最低的是public，其次protected，最高的是private。

### in-class/inherit access specifier rules
三种访问限定符在类内限定成员时的区别如下：<br>
**[1 public]** 被public修饰的数据、函数成员可以在类域内、派生类域内访问，也可以被类的对象、派生类的对象访问。 <br>
**[2 protected]** 被protected修饰的数据、函数成员可以在类域内、派生类域内访问；但是不可以被类的对象、派生类的对象访问。<br>
**[3 private]** 被private修饰的数据、函数成员只能在类域内进行访问。类的对象、派生类域、派生类的对象都不能对其进行访问。 

#### in-class访问限定关系
```c++
#include<iostream>
#include<cassert>
class A{
    // 关于类区的说明：一个类可以包含多个public,protected,private区
    // 每个区都会一直有效 直到另外一个区出现 或者遇到类结束符};
    // 访问限定符缺省的状态下 默认为private限定
    public:
        int a; int a1;
    protected:
        int a2;
    private:
        int a3;
    public:
        A(){
            a1=1; a2=2; a3=3; a=4;
        }
        void func();
};
void A::func(){
    std::cout<<a<<std::endl;// 正确 类域内访问public成员
    std::cout<<a1<<std::endl;// 正确 类域内访问public成员
    std::cout<<a2<<std::endl;// 正确 类域内码访问protected成员
    std::cout<<a3<<std::endl;// 正确 类域内访问private成员
}
int main(){
    A item_a;
    item_a.a=10; // 正确 类的对象可以访问public成员
    item_a.a1=20; // 正确 类的对象可以访问public成员
    item_a.a2=30; // 错误 类的对象不能访问protected成员
    item_a.a3=40; // 错误 类的对象不能访问private成员
    return 0;
}
```
#### inherit访问限定关系
继承同样有public，protected和private三种方式，他们分别的相应改变了基类成员在继承类中的访问属性，改变的是继承类的成员在其他继承子类中的访问属性，但是不改变基类成员在继承类中的访问方式。文字表述有点绕，直接看下面两个例子即可。

**[基类成员在继承类中的访问方式和继承方式无关]** <br>
继承类中对基类的访问限定由基类中相应成员的访问限定设定，和继承方式限定无关，即无论何种继承方式，继承类中对基类成员的访问方式只由基类成员访问限定符决定。详见下面的例子。
```c++
// public继承机制
#include<iostream>
#include<cassert>
class A{
    public:
        int a; int a1; 
    protected:
        int a2; 
    private:
        int a3;
    public:
        void func(){
            // 正确 类内代码拥有所有访问权限
            std::cout<<a; std:cout<<a1; std::cout<<a2; std:cout<<a3;
        }
};
class B:public A{
    public:
        int a;
        B(int i){
            A();
            a=i;
        } 
        void func(){
            std::cout<<a;// 正确 a是public成员 
            std::cout<<a1;// 正确 继承基类的public成员 可以在继承类域内访问 
            std::cout<<a2;// 正确 继承基类的protected成员 可以在继承类域内访问
            std::cout<<a3;// 错误 继承基类的private成员 不能在继承类域内访问
        }
};
class C:protected A{
    public:
        int a;
        C(int i){
            A();
            a=i;
        }
        void func(){
            std::cout<<a;// 正确 a是继承类public成员
            std::cout<<a1;// 正确 继承基类的public成员 可以在继承类中访问
            std::cout<<a2;// 正确 继承基类的protected成员 可以在继承类中访问 
            std::cout<<a3;// 错误 继承基类的private成员 不能在继承类中访问
        }
};
class D:private A{
    public:
        int a;
        D(int i){
            A();
            a=i;
        }
        void func(){
            std::cout<<a;// 正确 继承类的public对象 可以在继承类内访问
            std::cout<<a1;// 错误 继承基类的public成员 可以在继承类内访问
            std::cout<<a2;// 错误 继承基类的protected成员 可以在继承类内访问
            std::cout<<a3;// 错误 继承基类的private成员 不能在继承类内访问
        }
};
int main(){
    B b(10);// public继承
    std::cout<<b.a;// 继承类的public对象 可以被继承类的对象访问
    std::cout<<b.a1;// 继承了基类的public对象 可以被继承类的对象访问
    std::cout<<b.a2;// 继承了基类的protected对象 不能被继承类的对象访问
    std::cout<<b.a3;// 继承了基类的private对象 不能被继承类的对象访问
    C c(10);// protected继承
    std::cout<<c.a;// 继承类的public对象 可以被继承类的对象访问
    std::cout<<c.a1;// 继承了基类的public对象 可以被继承类的对象访问
    std::cout<<c.a2;// 继承了基类的protected对象 不能被继承类的对象访问 
    std::cout<<c.a3;// 继承了基类的private对象 不能被继承类的对象访问
    D d(10);// private继承
    std::cout<<d.a;// 继承类的public对象 可以被继承类的对象访问
    std::cout<<d.a1;// 继承了基类的public对象 可以被继承类的对象访问
    std::cout<<d.a2;// 继承了基类的protected对象 不能被继承类的对象访问
    std::cout<<d.a3;// 继承类的private对象 不能在用户代码段访问
    return 0; 
}
```
**[继承类对象覆盖基类对象]** <br>
上面的例子中，派生类定义了一个和基类同名的成员a，由于基类和派生类中都有定义，在派生类的对象中，基类的同名成员仍然存在(内存中存在)，下面的代码结果可以证明这一点。
```c++
int main(){
    std::cout<<sizeof(A);// 输出16 
    std::cout<<sizeof(B);// 输出20 这其中包含了4个byte 由基类中的同名对象a占用
    // 默认情况下 通过继承类对象调用数据成员a 
    B b(10);
    std::cout<<b.a;// 继承类对象直接调用的是继承类的数据/函数成员
    std::cout<<b.A::a;// 只有通过域作用符才能够调用基类的数据/函数成员
    system("pause");// 用于在获得cout输出后 暂停窗口 不让输出窗口一闪而过 
    return 0;
}
```
**[三种继承方式对于继承类成员的访问的影响]** <br>
**public继承(双域对象都可访问)**：基类中的public成员，protected成员，private成员分别变成继承类的public成员，protected成员，private成员。<br>
**protected继承(双域访问，对象不能访问)**：基类中的public成员，protected成员，private成员分别变成继承类的protected成员，protected成员，private成员。<br>
**private继承(仅限单域访问)**：基类中的public成员，protected成员，private成员分别变成继承类的private成员，private成员，private成员。下面使用代码进行理解。
```c++
#include<iostream>
#include<cassert>
class A{
    public:
        int a; int a1;
    protected:
        int a2;
    private:
        int a3;
    public:
        A(){
            a1=1; a2=2; a3=3; a=4;
        }
};

class B: public A{// public-inherit
    void funcb();
};
void B::funcb(){
    // 在继承类中访问基类的成员 - 访问方式和成员在基类的属性有关 和B:public A无关
    std::cout<<a<<a1<<a2<<a3<<std::endl;
}

class C: private B{// private-inherit
    void funcc();
};
void C::funcc(){
    // 在继承子类访问继承类的成员 - 访问方式和成员在继承类的属性有关 和C:private B无关
    // 成员在继承类的属性 = (继承类B的继承方式 + 基类A的访问限定方式)的共同作用
    std::cout<<a<<std::endl;// a在B中是public - 可以访问
    std::cout<<a1<<std::endl;// a1在B中是public - 可以访问
    std::cout<<a2<<std::endl;// a2在B中是protected - 可以访问
    std::cout<<a3<<std::endl;// a3在B中是private - 不能访问
}
int main(){
    C c_obj;// 继承子类C的对象
    c_obj.a2;// a2在B中是protected - 不能被继承子类C的对象访问
}
```


## Reference
> 《c++ primer》- N.Gregory Mankiw <br>
> https://www.cnblogs.com/tsingke/p/10052445.html

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内
