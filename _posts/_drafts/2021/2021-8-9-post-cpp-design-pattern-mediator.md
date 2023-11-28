---
layout: post
title: "cpp design pattern - mediator"
subtitle: '[behavioral pattern] c++设计模式之中介者模式' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-08-09 16:48
lang: ch 
catalog: true 
categories: cpp 
tags:
  - Time 2021
  - cpp design pattern
---
## 中介者模式
**[1] 基本概念** <br> 
中介者模式：用一个中介对象来封装特定类型层次结构对象之间的交互。中介者使各对象不需要显示的相互引用，从而在特定类型对象之间实现松耦合，以实现独立地改变这些兑现之间的交互。

**[2] 要素组成** <br>
1) AbstractComponent：抽象组分对象类。用于规定组分对象通用接口，以及维护一个抽象中介类的指针以方便调用中介类成员方法。<br>
2) ConcreteComponent：具体组分对象类。用于具体实现组分对象通用接口。<br>
3) AbstractMediator：抽象中介者类。用于规定中介者类型的"职责api"，即中介者能够为组分对象之间的"互动"做什么；以及维护不定数量的指向抽象组分对象的指针(指针的数量取决于中介者类想要进行"互动"的组分对象的数量)。<br>
4) ConcreteMediator：具体中介者类。用于具体实现组分对象的通用接口。<br>

**[3] 应用场景(simply borrowed from gof)** <br>
**1** a set of objects communicate in well-defined but complex ways. The resulting interdependencies are unstructured and difficult to understand. 一系列对象一种明确的但是非常复杂的方式进行通信，导致它们内部的通信没有条例，难以理解，非常混乱。<br>
**2** reusing an object is difficult because it refers to and communicates with many other objects. 在第一条提到的场景下，对象的重用是非常困难的，因为它可能引用了多个对象、与很多其他对象进行通信。<br>
**3** a behavior that's distributed between several classes should be customizable without a lot of subclassing. (特指下面例子中Mediator::Declare函数，通过构建中介者类型可以实现：通过简单修改Mediator::Declare函数的实现，就可以修改多个派生类型对象(USA::obj & Iraq::obj)之间的交互行为)。这种做法能够避免为了修改多个派生类型对象公共的特性而创建新的、冗余的子类。

<center><img src="/img/in-post/cpp_img/mediator_1.pdf" width="80%"></center>

**[4] strength & weakness** <br>
**strength** <br>
**1** 满足单一职责原则。Mediator::Declare函数将多个组件的交互从具体组分类型对象中，并进行**集中实现**，满足单一职责原则的同时，便于对多个对象之间的交互逻辑进行修改。**2** 满足开闭原则。无需修改组件类型，就可以添加新的中介者(新的中介仍然需要使用和Meditor::Declare相同的函数接口)。**3** 构建中介者类型，可以减少多个组件之间的耦合(相当于将它们之间的耦合抽离出来抽象成Mediator类)。**4** 能够更加方便的复用各个组件对象???????。<br>
**weakness** <br>
**1** 确保在必要时使用中介者模型，如果过度的滥用，可能会形成一个"上帝对象"(一个承担了太多职责，拥有太多权限的对象)。

## 中介者设计模式案例
**[1] 案例目标** <br>
给定一个受联合国(中介类型)管理的美国、伊拉克两个国家(组分类型)的通信场景。<br>
美国宣称"Research on Nuclear Weapons is forbiden."，则伊拉克收到消息为"Derived message (Iraq): Research on Nuclear Weapons is forbidden."，伊拉克宣称"We have no Nuclear Weapons and no fear for invasion."，则美国收到的消息为"Derived message (USA): We have no Nuclear Weapons and no fear for invasion."。

**[2] 中介者设计模式案例UML类图**

<center><img src="/img/in-post/cpp_img/mediator_2.pdf" width="100%"></center>

**[3] 中介者设计模式代码示例** <br>
Part-1 抽象、具体中介类定义部分代码示例 \<Mediator.h\>
```cpp
#ifndef MEDIATOR_H 
#define MEDIATOR_H 

#include <string>
class Country;

class UnitedNations{// abstract mediator
    public:
        // stipulate general api 
        virtual void SetComponent1(Country *ptrcomponent) = 0;
        virtual void SetComponent2(Country *ptrcomponent) = 0;
        virtual void Declare(std::string message, Country *ptrcomponent) = 0;
};
class UnitedNationsSecurityCenter: public UnitedNations{// concrete mediator
    private:
        Country *ptr_component1___;// maintain component1 ptr for calls
        Country *ptr_component2___;// maintain component2 ptr for calls
    public:
        // 对应Part-5(2) 使用构造函数进行初始化
        UnitedNationsSecurityCenter(Country *ptrcomponenta, 
            Country *ptrcomponentb){
            this->ptr_component1___ = ptrcomponenta;
            this->ptr_component2___ = ptrcomponentb;
        }
        // 对应Part-5(1) 使用set函数进行初始化(推荐->可以使客户端代码更加清晰)
        void SetComponent1(Country *ptrcomponent);// init 
        void SetComponent2(Country *ptrcomponent);// init
        // [核心函数] polymorphically call 处理多个Component对象之间的信息交互
        void Declare(std::string message, Country *ptrcomponent);
};
#endif
```
Part-2 具体中介者类具体实现部分代码示例 \<Mediator.cpp\>
```cpp
#include <string>
#include "Mediator.h"
#include "Component.h"

void UnitedNationsSecurityCenter::SetComponent1(Country *ptrcomponent){
    this->ptr_component1___=ptrcomponent;
}
void UnitedNationsSecurityCenter::SetComponent2(Country *ptrcomponent){
    this->ptr_component2___=ptrcomponent;
}
// [核心函数] 处理多个Component对象之间的信息交互 - [关键是维护一个组分对象类的抽象指针]
void UnitedNationsSecurityCenter::Declare(std::string, Country *ptrcomponent){
    // use typeid to call the getmessage polymorphically
    if(typeid(*this->ptr_component1___) == typeid(ptrcomponent)){
        ptr_component2___->GetMessage();// return the message of another obj
    }
    else{
        ptr_component1___->GetMessage();// return the message of another obj
    }
}
```
Part-3 抽象、具体组成成分类型定义部分代码示例 \<Component.h\>
```cpp
#ifndef COMPONENT_H
#define COMPONENT_H

#include <string>
#include <iostream>
#include "Mediator.h"

class Country{// abstract component
    protected:
        UnitedNations *ptr_mediator__;// maintain mediator ptr for calls
    public:
        // stipulate general api
        virtual void Declare(std::string message) = 0;
        virtual void GetMessage(std::string message) = 0;
};
class USA: public Country{// concrete component 1
    public:
        USA(UnitedNations *ptrmediator){// init
            this->ptr_mediator__ = ptrmediator;
        }
        void Declare(std::string message);
        void GetMessage(std::string message);
};
class Iraq: public Country{// concrete component 2
    public:
        Iraq(UnitedNations *ptrmediator){// init
            this->ptr_mediator__ = ptrmediator;
        }
        void Declare(std::string message):
        void GetMessage(std::string message);
};
#endif
```
Part-4 具体组成成分类实现部分代码示例 \<Component.cpp\>
```cpp
#include "Component.h"
#include <string>
#include <iostream>

// concrete component 1
void USA::Declare(std::string message){
    ptr_mediator__->Declare(message, this);    
}
void USA::GetMessage(std::string message){
    std::cout<<"Derived message (USA): "<<message<<std::endl;
}
// concrete component 2
void Iraq::Declare(std::string message){
    ptr_mediator__->Declare(message, this);
}
void Iraq::GetMessage(std::string message){
    std::cout<<"Derived message (Iraq): "<<message<<std::endl;
}
```
Part-5(1) 客户端程序代码示例 - 针对Mediator使用Set函数进行指针初始化的情况 \<main.cpp\>
```cpp
#include "Mediator.h"
#include "Component.h"

int main(){
    // init mediator
    UnitedNations *ptr_mediator = new UnitedNationsSecurityCenter();
    // init component
    Country *ptr_usa = new usa_announcement(ptrmediator);
    Country *ptr_iraq = new iraq_announcement(ptrmediator);
    // set mediator (construct link between mediator and each component obj)
    ptrmediator->SetComponent1(ptr_usa);
    ptrmediator->SetComponent2(ptr_iraq);
    // component interaction
    USA.Declare("Research on Nuclear Weapons is forbiden.");
    Iraq.Declare("We have no Nuclear Weapons and no fear for invasion.");
    return 0;
}
```
Part-5(2) 客户端程序代码示例 - 针对Mediator使用构造函数进行指针初始化的情况 \<main.cpp\>
```cpp
#include "Mediator.h"
#include "Component.h"

int main(){
    // init 2 ptr
    Country *ptr_usa = nullptr; Country *ptr_iraq = nullptr;
    // init mediator
    UnitedNations *ptr_mediator = 
        new UnitedNationsSecurityCenter(ptr_usa, ptr_iraq);
    // init component
    ptr_usa = new usa_announcement(ptrmediator);
    ptr_iraq = new iraq_announcement(ptrmediator);
    // component interaction
    USA.Declare("Research on Nuclear Weapons is forbiden.");
    Iraq.Declare("We have no Nuclear Weapons and no fear for invasion.");
    return 0;
}
```
Part-6 客户端程序输出示例 \<out\>
```txt
Derived message (Iraq): Research on Nuclear Weapons is forbidden.
Derived message (USA): We have no Nuclear Weapons and no fear for invasion.
```

## Reference
> \<大话设计模式\> chapter25 p275 <br>
> https://blog.csdn.net/xiqingnian/article/details/41307937 <br>
> gof chapter5 p305 <br>
> https://refactoringguru.cn/design-patterns/mediator

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内 <br>
> 8 `<center><img src="/img/in-post/economics_4/xxx.png" width="60%"></center>` <br>
> 9 使用html设置图片文字环绕方式: <br>
    `<div>` <br>
        `<img src="/img_path" align="left" width="40%" hspace="" vspace=""/>` <br>
        `<p>paragraph1 around the picture</p>` <br>
        `<p>paragraph2 around the picture</p>` <br>
        `<p>paragraph3 around the picture</p>` <br>
    `</div>` <br>
> 10 `<font style="color:red; font-weight:bold">加粗蓝色</font>`用来设置字体颜色
