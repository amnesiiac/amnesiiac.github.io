---
layout: post
title: "cpp design pattern - observer"
subtitle: '[behavioral pattern] c++设计模式之观察者模式' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-08-14 20:21
lang: ch 
catalog: true 
categories: cpp 
tags:
  - Time 2021
  - cpp design pattern
---
## 观察者模式
**[1] 基本概念** <br> 
观察者模式：定义了一种**一对多**的依赖关系，让多个观察者对象同时监听某一通知者对象，这个通知者对象在状态发生变化时，会通知所有观察者对象，使它们更新通知者的状态、更新观察者自身状态。

**[2] 要素组成** <br>
**1** AbstractSubject抽象通知者类：用于规范用于多态实现的接口api；定义所有具体通知者类公用的接口api(get-state、set-state)。<br>
**2** ConcreteSubject具体通知者类：多态地实现抽象通知者类api；维护一个观察者类型指针列表，用于调用(notify)。<br>
**3** AbstractObserver抽象观察者类：维护一个指向抽象通知者的指针，并通过这个指针更新(update)通知者的内部状态。<br>
**4** ConcreteObserver具体观察者类：多态地实现抽象观察者类的更新函数(update)，采用不同方式更新通知者类状态。<br>

**[3] 应用场景(simply borrowed from gof)** <br>
**1** When an abstraction has two aspects, one dependent on the other. Encapsulating these aspects in separate objects lets you vary and reuse them independently. 当一种抽象包含两个主要概念，并且这两个概念相互依赖，相互调用，将它们解耦并分别实现成"通知者"和"观察者"可以方便对它们进行单独修改&单独重用。<br>
**2** When a change to one object requires changing others, and you don't know how many objects need to be changed. 改变一个对象时，需要通知其他对象进行同步改变，但是不确定到底有多少对象需要改变。此时，将两个对象分别实现成通知者和观察者，并在通知者中维护一个观察者列表。有新的观察者需要同步改变时，将它加入列表即可。<br>
**3** When an object should be able to notify other objects without making assumptions about who these objects are. In other words, you don't want these objects tightly coupled. 当我们希望实现一些列对象的同步更新(更新的内容可以不同)，但是又不想让这些对象耦合在一起，可以将这些对象设置成抽象观察者类的具体派生类型进行解耦。<br>

**[4] strength & weakness** <br>
**strength** <br>
**1** 满足开闭原则。无需修改通知者的代码就可以添加新的观察者。**2** 可以允许客户端建立特定的"通知者-观察者"联系，并且可以取消已经建立的联系。<br>
**weakness** undefined.

**[5] 与其他设计模式的关系** <br>
**1 四种模式(责任链模式、命令模式、中介者模式、观察者模式)之间的关系** <br>
责任链模式按照责任链构建顺序将请求动态地发送给一系列潜在接受者，直到有一名接受者对该请求进行处理。<br>
命令模式建立了从请求者到接受者的单向连接。<br>
中介者将请求者和接受者之间的直接连接切断，强制它们通过中介对象进行连接(适用于多个请求者、接受者构成的复杂网络)。<br>
观察者模式允许动态的建立、动态的取消请求者和接受者之间的连接。

**2 中介者模式和观察者模式有时只是用一种模式，但也可以同时使用** <br>
**中介者模式的目标** <br> 通过建立Mediator类型，从而消除一系列系统组件之间的复杂耦合关系，系统组件都依赖一个Mediator对象。观察者的目标是：在对象之间动态地建立、销毁链接，使得一部分类型的对象可以作为其他对象的辅助发挥作用。<br>
**将Mediator、Observer模式结合使用** <br> 将中介者类型实现为通知者，将其他类型实现为观察者。这样在中介者简化了系统组件之间的复杂耦合的同时，还可以对于它们之间的依赖进行动态的管理。进一步地，一个系统组件集合的关系构建中，可以设置多个中介者(通知者)，使得整个系统服务程序没有中心化的中介者对象，一定程度上避免了上帝对的出现带来的脆弱性问题。


## 观察者设计模式案例
**[1] 案例目标** <br>
建立"观察者-通知者"关系模型。其中观察者通过维护一个通知者类型指针来观测并维护通知者的状态(state)；通知者通过维护一个观察者类型指针列表以方便按照\<所需顺序\>调用观察者的api以更新它们观察的通知者对象状态。

**[2] 观察者设计模式案例UML类图**

<center><img src="/img/in-post/cpp_img/observer_1.pdf" width="100%"></center>

**[3] 观察者设计模式代码示例** <br>
##### Part-1 抽象通知者类(AbstractSubject)定义代码 \<subject.h\>
```cpp
#ifndef SUBJECT_H
#define SUBJECT_H

#include <string>
#include <list>

class AbstractObserver;
class AbstractSubject{// AbstractSubject
    protected:
        // shared data member: state
        std::string subject_state__;
    public:
        // virtual void funcs -> need polymorphic implementation
        virtual void Attach(AbstractObserver *ptrobserver) = 0;
        virtual void Detach(AbstractObserver *ptrobserver) = 0;
        virtual void Notify() = 0;
        // normal funcs -> use as it is
        std::string GetSubjectState();
        void SetSubjectState(std::string state);
};
class Boss: public AbstractSubject{// ConcreteSubject
    private:
        // hold the observers attached
        std::list<AbstractObserver*> observer_list___;
        std::string action___;
    public:
        void Attach(AbstractObserver *ptrobserver);// add an ob
        void Detach(AbstractObserver *ptrobserver);// delete an ob
        void Notify();// notify all obs to update
};
#endif
```
##### Part-2 抽象通知者类实现代码 \<subject.cpp\>
```cpp
// mind the head files
#include "subject.h"
#include "observer.h"

std::string GetSubjectState(){
    return this->subject_state__;
}
void SetSubjectState(std::string state){
    this->subject_state__ = state; 
}
void Boss::Attach(AbstractObserver *ptrobserver){// attach an ob
    this->observer_list___.push_back(ptrobserver);
}
void Boss::Detach(AbstractObserver *ptrobserver){// detach an ob
    std::list<AbstractObserver*>::iterator it;
    for(it=this->observer_list___.begin(); 
        it!=this->observer_list___.end(); ++it){
        if(*it == ptrobserver){// need overload op== 
            this->observer_list___.erase(it);
            break;
        }
    }
}
void Boss:Notify(){// only notify the observer in attached list
    std::list<AbstractObserver*>::iterator it;
    for(it=this->observer_list___.begin(); 
        it!=this->observer_list___.end(); ++it){
        (*it).Update();// update each observer for current subject
    }
}
```
##### Part-3 抽象观察者类定义代码 \<observer.h\>
```cpp
#ifndef OBSERVER_H
#define OBSERVER_H

#include <list>
#include <string>
#include <iostream>
#include "subject.h"

class AbstractObserver{
    protected:
        std::string name__;
        AbstractSubject *ptr_subject__;// maintain a subject ptr
    public:
        // for new expression 
        AbstractObserver(){};
        // init ctor: below block thd default one by OS
        AbstractObserver(std::string name, AbstractSubject *ptrsubject)
            :name__(name),ptr_subject__(ptrsubject){}
        bool operator==(const AbstractObserver&) const;// overload op==
        virtual void Update();// why not =0?
};
class StockObserver: public AbstractObserver{
    public:
        StockObserver(){};// for new expression
        StockObserver(std::string name, AbstractSubject *ptrsubject)
            :name__(name), ptr_subject__(ptrsubject){}
        void Update();
};
class NBAObserver: public AbstractObserver{
    public:
        NBAObserver(){};// for new expression
        NBAObserver(std::string name, AbstractSubject *ptrsubject)
            :name__(name), ptr_subject__(ptrsubject){}
        void Update();
};
#endif
```
##### Part-4 抽象观察者类实现代码 \<observer.cpp\>
```cpp
#include "observer.h"

void AbstractObserver::Update(){// why should implement this????
    std::cout<<"abstractobserver's update"<<std::endl;
}
bool AbstractObserver::operator==(const AbstractObserver& refobserver) const{
    return (this->name__ == refobserver->name__) &&
        (this->ptr_subject__ == refobserver->ptr_subject__);
}
void StockObserver::Update(){
    // we can call SetSubjectState(change) or GetSubjectState(touch) to update
    std::cout<<this->ptr_subject__->GetSubjectState()<<" "
        <<name__<<", shut down the stock client! Back to work!"<<std::endl;
} 
void NBAObserver::Update(){
    std::cout<<this->ptr_subject__->GetSubjectState()<<" "
        <<name__<<", shut down the NBA client! Back to work!"<<std::endl;
}
```
##### Part-5 客户端代码示例程序 \<main.cpp\>
```cpp
#include "observer.h"
#include <iostream>
#include <cstdlib>

int main(){
    AbstractSubject *ptr_boss = new Boss();// init subject
    // init observers for the subject -> regist subject in these obs
    AbstractObserver *ptr_ob_a = new StockObserver("Avatar", ptr_boss);
    AbstractObserver *ptr_ob_b = new StockObserver("Goblin", ptr_boss);
    AbstractObserver *ptr_ob_c = new NBAObserver("Orcs", ptr_boss);
    AbstractObserver *ptr_ob_d = new NBAObserver("Godzilla", ptr_boss);
    // attach above obs to the subject -> attach obs in the subject
    Attach(ptr_ob_a); Attach(ptr_ob_b); Attach(ptr_ob_c); Attach(ptr_ob_d);
    Detach(ptr_ob_c);// detach orcs for their poor english
    // call obs to update
    ptr_boss->SetSubjectState("Ur daddy is back!")
    ptr_boss->Notify();
    // recycle
    delete ptr_boss; 
    delete ptr_ob_a; delete ptr_ob_b; delete ptr_ob_c; delete ptr_ob_d;
}
```
##### Part-6 客户端代码程序输出示例 \<out\>
```txt
Ur daddy is back! Avatar, shut down the stock client! Back to work!
Ur daddy is back! Goblin, shut down the stock client! Back to work!
Ur daddy is back! Godzilla, shut down the NBA client! Back to work!
```

## Reference
> \<大话设计模式\> chapter14 p141 <br>
> https://blog.csdn.net/xiqingnian/article/details/42059321 <br>
> gof chapter5 p326 <br>
> https://refactoringguru.cn/design-patterns/observer

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
