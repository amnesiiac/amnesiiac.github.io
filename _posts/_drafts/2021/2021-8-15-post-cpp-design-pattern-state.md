---
layout: post
title: "cpp design pattern - state"
subtitle: '[behavioral pattern] c++设计模式之状态模式' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-08-15 14:43
lang: ch 
catalog: true 
categories: cpp 
tags:
  - Time 2021
  - cpp design pattern
---
## 状态模式
**[1] 基本概念** <br> 
状态模式：当一个类型的对象内在状态改变时允许改变该对象的行为，通过背景类型和几种状态类之间的相互调用完成：当前状态设置，更新状态，并根据状态进行相应处理。

**[2] 要素组成** <br>
**1** Context类型：在类型内部维护了一个ConcreteState类型的实例，该实例存储了当前对象的状态。并根据当前状态多态地调用具体状态类。<br>
**2** AbstractState类型：抽象状态类。用于规范具体状态类的接口方法。<br>
**3** ConcreteState类型：具体状态类。实现抽象状态类规范的接口方法，并进行相应处理or为Context类更新的当前状态，并调用Context类接口移交新的状态类进行处理。<br>

**[3] 应用场景(simply borrowed from gof)** <br>
**1** An object's behavior depends on its state, and it must change its behavior at run-time depending on that state. 状态设计模式允许在运行时改变一个对象的内部状态，并且运行时根据状态的不同而采取不同行为措施。<br>
**2** Operations have large, multipart conditional statements that depend on the object's state. This state is usually represented by one or more enumerated constants. Often, several operations will contain this same conditional structure. The State pattern puts each branch of the conditional in a separate class. This lets you treat the object's state as an object in its own right that can vary independently from other objects. 当控制一个对象的状态转换的条件表达式过于复杂的时候，将用于状态判断的逻辑状态转移到表示不同状态的一系列类型中去，可以将复杂的状态判断、行为执行进行简单化。

**[4] strength & weakness** <br>
**strength** <br>
**1** 满足单一职责原则：将对象中复杂的状态分割解耦，将划分成多子状态分别构建成为不同类型。**2** 满足开闭原则：无需修改已有的状态类型集合、已有的背景类就可以添加新的状态。**3** 消除臃肿、繁杂的状态条件语句简化上下文代码。<br>
**weakness** <br> 
如果状态机只有很少几个状态，以及影响对象状态的成员数据单一、简单，那么应用状态设计模式可能是小题大做。

**[5] 与其他设计模式的关系** <br>
状态设计模式与：桥接设计模式、策略设计模式、某种程度上包括适配器模式的构建方式，功能类型之间的配合方式非常相似，都是基于**组合模式**通过维护其他类型的指针、对象、接口从而可以允许将复杂的场景解耦。<br>
状态设计模式可以看作是策略模式的另一种变体：两种设计模式都是基于组合设计模式，它们都是将可以解耦的工作职责部分交给"helper"类型，以处理不同情景下的行为。策略使得这些对象之间完全独立(不涉及相互调用、每个策略负责不同逻辑)，而状态模式无此限制，它允许设置Context类型的状态、并根据状态之间的相互调用(借助Context类)。

## 状态设计模式案例
**[1] 案例目标** <br>
构建一个状态类层次结构，包括抽象状态类、具体状态类，每个状态类根据状态条件不同负责不同状态的处理实施；构建一个背景类，用于存储当前状态，并根据当前状态调用相应状态类方法进行具体方法实施。

**[2] 状态设计模式案例UML类图**

<center><img src="/img/in-post/cpp_img/state_1.pdf" width="100%"></center>

**[3] 状态设计模式代码示例** <br>
##### Part-1 抽象、具体状态类的定义 \<state.h\>
```cpp
#ifndef STATE_H
#define STATE_H

extern class Context;// 等价于#include "context.h"
class AbstractState{
    public:
        AbstractState(){};// why need this func???
        // stipulate the general api for func call
        virtual void WriteProgramme(Context *ptrwork) = 0;
};
class ForeNoonState: public AbstractState{// start state#0
    public:// default ctor
        void WriteProgramme(Context *ptrwork);// overload
};
class NoonState: public AbstractState{// state#1
    public:
        void WriteProgramme(Context *ptrwork);// overload
};
class AfterNoonState: public AbstractState{// state#2
    public:
        void WriteProgramme(Context *ptrwork);// overload
};
class EveningState: public AbstractState{// state#3
    public:
        void WriteProgramme(Context *ptrwork);// overload
};
class SleepingState: public AbstractState{// ending state#4
    public:
        void WriteProgramme(Context *ptrwork);// overload
};
class RestState: public AbstractState{// ending state#5
    public:
        void WriteProgramme(Context *ptrwork);// overload
};
#endif
```
##### Part-2 抽象、具体状态类的实现 \<state.cpp\>
```cpp
#include <iotream>
#include <state.h>
#include <context.h>

void ForeNoonState::WriteProgramme(Context *ptrworker){
    if(ptrworker->GetHour()<12){// if at this state
        std::cout<<"Current Time: "<<ptrworker->GetHour()
            <<". Working like a donkey."<<std::endl;
    } 
    else{// not in this state
        ptrworker->SetState(new NoonState());// switch to next state 
        ptrworker->WriteProgramme();// then call context api
    }
}
void NoonState::WriteProgramme(Context *ptrworker){
    if(ptrworker->GetHour()<13){// if at this state
        std::cout<<"Current Time: "<<ptrworker->GetHour()
            <<". Working like a sloth."<<std::endl;
    } 
    else{// not in this state
        ptrworker->SetState(new AfterNoonState());// switch to next state
        ptrworker->WriteProgramme();// then call context api
    }
}
void AfterNoonState::WriteProgramme(Context *ptrworker){
    if(ptrworker->GetHour()<17){// if at this state
        std::cout<<"Current Time: "<<ptrworker->GetHour()
            <<". Working like a donkey again."<<std::endl;
    } 
    else{// not in this state
        ptrworker->SetState(new EveningState());// switch to next state
        ptrworker->WriteProgramme();// then call context api
    }
}
void EveningState::WriteProgramme(Context *ptrworker){
    if(ptrworker->GetFinish()){// not in this state: work done 
        ptrworker->SetState(new RestState());// switch to next state
        ptrworker->WriteProgramme();// then call context api
    }
    else{
        if(ptrworker->GetHour()<21){// if at this state 
            std::cout<<"Current Time: "<<ptrworker->GetHour()
                <<". Overtime working like a snail."<<std::endl;
        } 
        else{// not in this state: too late
            ptrworker->SetState(new SleepingState());// switch to next state
            ptrworker->WriteProgramme();// then call context api
        }
    }
}
void SleepingState::WriteProgramme(Context *ptrworker){// ending state
    std::cout<<"Current Time: "<<ptrworker->GetHour()
        <<". Sleeping is my work."<<std::endl;
}
void RestState::WriteProgramme(Context *ptrworker){// ending state
    std::cout<<"Current Time: "<<ptrworker->GetHour()
        <<". Go home."<<std::endl;
}
```
##### Part-3 Context类的定义 \<context.h\>
```cpp
#ifndef CONTEXT_H
#define CONTEXT_H

#include "state.h"

class Context{
    private:
        // maintain state ptr for polymorphically call
        AbstractState *ptr_current___;
        double hour___;// state-1
        bool finish___;// state-2
    public:
        Context();
        ~Context();
        // set & get
        void SetState(AbstractState *ptrstate);
        void SetHour(double hour);
        double GetHour();
        void SetFinish(bool finish);
        bool GetFinish();
        // client function call interface
        void WriteProgramme();
};
#endif
```
##### Part-4 Context类的实现 \<context.cpp\>
```cpp
#include "context.h"

Context::Context(){
    // 每次调用都从forenoonstate状态开始
    this->ptr_current___ = new ForeNoonState();// init ptr
    this->hour___ = 9;// init hour
    this->finish___ = false;// inti finish
}
Context::~Context(){
    if(this->ptr_current___!=nullptr){
        delete ptr_current___;
    }
}
void Context::SetState(AbstractState *ptrstate){// switch to new state
    this->ptr_current___ = ptrstate;
}
void Context::SetHour(double hour){
    this->hour___ = hour;
}
double Context::GetHour(){
    return this->hour___;
}
void Context::SetFinish(bool finish){
    this->finish___ = finish;
}
bool Context::GetFinish(){
    return this->finish___;
}
void Context::WriteProgramme(){
    // 调用abstractstate类层次结构的api
    ptr_current___->WriteProgramme(this);
}
```
##### Part-5 客户端代码示例 \<main.cpp\>
```cpp
#include <iostream>
#include <cstdlib>
#include "context.h"

int main(){
    // execute default init (current_state,hour,finish)
    Context emergency_context;
    emergency_context.SetHour(9);// set state
    // call certain write programme by the state (state )
    emergency_context.WriteProgramme();
    emergency_context.SetHour(10);
    emergency_context.WriteProgramme();
    emergency_context.SetHour(12);
    emergency_context.WriteProgramme();
    emergency_context.SetHour(13);
    emergency_context.WriteProgramme();
    emergency_context.SetHour(14);
    emergency_context.WriteProgramme();
    emergency_context.SetHour(17);
    emergency_context.WriteProgramme();
    emergency_context.SetFinish(false);// set state
    emergency_context.SetHour(19);
    emergency_context.WriteProgramme();
    emergency_context.SetHour(22);
    emergency_context.WriteProgramme();
    return 0;
}
```
##### Part-6 客户端代码输出示例 \<main.cpp\>
```txt
Current Time: 9. Working like a donkey.
Current Time: 12. Working like a sloth.
Current Time: 13. Working like a donkey again.
Current Time: 14. Working like a donkey again.
Current Time: 17. Overtime Working like a snail.
Current Time: 19. Overtime Working like a snail.
Current Time: 22. Sleeping is my work.
```

## Reference
> \<大话设计模式\> chapter16 p176 <br>
> https://blog.csdn.net/xiqingnian/article/details/41242167 <br>
> gof chapter5 p338 <br>
> https://refactoringguru.cn/design-patterns/state

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
