---
layout: post
title: "cpp design pattern - memento"
subtitle: '[behavioral pattern] c++设计模式之备忘录模式' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-08-10 21:41
lang: ch 
catalog: true 
categories: cpp 
tags:
  - Time 2021
  - cpp design pattern
---
## 备忘录模式
**[1] 基本概念** <br> 
备忘录模式：在不破坏封装性的前提下，捕获一个对象的内部状态，并将对象的状态保存在对象之外，以备随时恢复对象到保存的状态。

**[2] 要素组成** <br>
**1** Originator(发起人)：被Memento类备份的类型，开放SaveState、StateRecovery接口。<br>
**2** Memento(备忘录)：维护和Originator相同的数据成员，用于存储Originator的备份。<br>
**3** Caretaker(管理者)：Memento类的管理类型。维护指向Memento类型的指针通过Caretaker类对Memento类进行操作。实现了对Memento类型数据成员的隐藏，间接隐藏了Originator类的对象的数据构成。<br>

**[3] 应用场景(simply borrowed from gof)** <br>
**1** a snapshot of (some portion of) an object's state must be saved so that it can be restored to that state later. 某些对象的状态(成员数据)需要被全部、部分备份，以备后续用该备份恢复对象的状态。<br>
**2** a direct interface to obtaining the state would expose implementation details and break the object's encapsulation. 我们可以直接在客户端代码中通过成员函数获取该对象状态，并保存，但是这样会暴露对象内部实现，打破对象类型的封装性。**(为了保护封装性，设计了memento类用于对象状态的保存，但是为了不能直接调用memento类的对象(同样会暴露originator的数据封装性)，因此设计了caretaker类，维护一个memento类的指针，用于间接调用)**。

**[4] strength & weakness** <br>
**strength** <br>
**1** 可以在不破坏对象封装性的前提下，保存对象的快照。**2** 可以将Originator中关于保存历史状态，从历史状态恢复的代码解耦出来，构成备忘录类型，满足单一职责原则。<br>
**weakness** <br>
**1** 客户端过于频繁的创建备忘录，可能导致程序内存大量消耗。**2** 一切关于Memento类对象的操作都需要使用Caretaker类进行控制，避免两者不同步产生难以追踪的错误。Caretaker必须完整跟踪Memento类对象的生命周期。**3** 大多数动态语言(python、PHP、javascript)不能保证备忘录中的状态不被修改。

**[5] 与其他设计模式的关系** <br>
**1 "命令的撤销"** [命令模式 + 备忘录模式] - 用命令模式修改对象的状态，用备忘录模式将对象状态恢复到应用命令之前。<br>
**2 "迭代器回滚"** [迭代器模式 + 备忘录模式] - 使用备忘录模式获取当前迭代器对象的状态，并在需要时将迭代状态从历史状态恢复。 <br>
**3 "一种简化的备忘录"** [原型模式可以充当简化的备忘录] - 应用条件是：需要备份的对象状态较为简单，不需要链接到其他外部资源，或链接的外部资源较容易重建。<br>

## 备忘录设计模式案例
**[1] 案例目标** <br>
给定一个游戏角色类型，角色属性包含生命值、攻击力、防御力三种属性。构建一个游戏角色备份类型，使得游戏角色可以将状态从备份类中予以保存、调取。

**[2] 备忘录设计模式案例UML类图**

<center><img src="/img/in-post/cpp_img/memento_1.pdf" width="100%"></center>

**[3] 备忘录设计模式代码示例** <br>
Part-1 Memento类 \<memento.h\>
```cpp
#ifndef MEMENTO_H // 避免memento.h被重复包含 产生重复定义的错误
#define MEMENTO_H
class RoleStateMemento{
    private:
        // maintain the same data as originator
        int vitality___;// 生命值
        int attack_force___;// 攻击力
        int defence___;// 防御力
    public:
        // ctor set all data in one call
        RoleStateMemento(int vit, int ack, int def)
            :vitality___(vit), attack_force___(ack), defence___(def){
        }
        // tool func: set & get (for set & get certain memento state)
        void SetVitality(int vit){
            this->vitality___ = vit;
        }
        int GetVitality(){
            return this->vitality___;
        }
        void SetAttackForce(int ack){
            this->attack_force___ = ack;
        }
        int GetAttackForce(){
            return this->attack_force___;
        }
        void SetDefence(int def){
            this->defence___ = def;
        }
        int GetDefence(){
            return this->defence___;
        }
};
#endif
```
Part-2 Originator类 \<originator.h\>
```cpp
#include "memento.h"
class GameRole{ 
    private:
        int vitality___;
        int attack_force___;
        int defence___;
    public:
        void SetInitState(){// init state
            this->vitality = 100;
            this->attack_force___ = 100;
            this->defence___ = 100;
        }
        void AfterDeadlyFight(){// state-change api
            this->vitality = 0;
            this->attack_force___ = 0;
            this->defence___ = 0;
        }
        void StateDisplay(){// state display
            std::cout<<"Current Role State:"<<std::endl;
            std::cout<<"Vitality: "<<this->vitality<<std::endl;
            std::cout<<"Attack Force: "<<this->attack_force___<<std::endl;
            std::cout<<"Defence: "<<this->defence___<<std::endl;
            std::cout<<std::endl;
        }
        // [core function] save state & return the memento
        RoleStateMemento* SaveState(){
            return new RoleStateMemento(
                this->vitality___, this->attack_force___, this->defence___);
        }
        // [core function] recover state from memento
        void StateRecovery(RoleStateMemento *ptrmemento){
            SetVitality(ptrmemento->vitality___);
            SetAttackForce(ptrmemento->attack_force___);
            SetDefence(ptrmemento->defence___);
        }
        // set & get
        void SetVitality(int vit){
            this->vitality___ = vit;
        }
        int GetVitality(){
            return this->vitality___;
        }
        void SetAttackForce(int ack){
            this->attack_force___ = ack;
        }
        int GetAttackForce(){
            return this->attack_force___;
        }
        void SetDefence(int def){
            this->defence___ = def;
        }
        int GetDefence(){
            return this->defence___;
        }
};
```
Part-3 Caretaker管理者类型 \<caretaker.h\>
```cpp
#include "memento.h"
class RoleStateCaretaker{
    private:
        // maintain memento ptr for memento handler
        RoleStateMemento *ptr_memento___;
    public:
        RoleStateCaretaker(){// init memento ptr (for new expression)
            this->ptr_memento___ = nullptr;
        }
        ~RoleStateCaretaker(){
            if(memento!=nullptr){
                delete ptr_memento___;
                ptr_memento___ = nullptr;// reset ptr state
            }
        }
        // tool func: set & get private memento
        RoleStateMemento* GetMemento(){
            return ptr_memento___;
        }
        void SetMemento(RoleStateMemento *ptrmemento){// set memento ptr
            this->ptr_memento___ = ptrmemento;
        }
};
```
Part-4 客户端代码示例 \<main.cpp\>
```cpp
#include "originator.h"
#include "caretaker.h"

int main(){
    // init role
    GameRole *ptr_tracer;
    ptr_tracer->SetInitState();
    ptr_tracer->StateDisplay();
    // init memento-handler 
    RoleStateCaretaker *ptr_role_state_handler = new RoleStateCaretaker();
    ptr_role_state_handler->SetMemento(ptr_tracer->SaveState());
    // role state change 
    ptr_tracer.AfterDeadlyFight();
    ptr_tracer.StateDisplay();
    // recover role state from memento controlled by memento-handler
    ptr_tracer.StateRecovery(ptr_role_state_handler->GetMemento()); 
    ptr_tracer.StateDisplay();
    // recycle
    delete ptr_role_state_handler;
    delete ptr_tracer;
}
```
Part-4 客户端代码输出示例 \<out\>
```txt
Current Role State:
Vitality: 100 
Attack Force: 100 
Defence: 100 

Current Role State:
Vitality: 0 
Attack Force: 0 
Defence: 0 

Current Role State:
Vitality: 100 
Attack Force: 100 
Defence: 100 
```

## Reference
> \<大话设计模式\> chapter18 p198 <br>
> https://blog.csdn.net/xiqingnian/article/details/42063847 <br>
> gof chapter5 p316 <br>
> https://refactoringguru.cn/design-patterns/memento

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
