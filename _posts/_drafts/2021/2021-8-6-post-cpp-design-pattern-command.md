---
layout: post
title: "cpp design pattern - command"
subtitle: '[behavioral pattern] c++设计模式之命令模式' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-08-06 09:23
lang: ch 
catalog: true 
categories: cpp 
tags:
  - Time 2021
  - cpp design pattern
---
## 命令模式
**[1] 基本概念** <br> 
命令模式：将命令请求封装成一个对象，从而使你可用不同的请求对于客户进行参数化。并且支持对命令请求进行排队、记录请求日志，且支持命令请求撤销的操作。

**[2] 要素组成** <br>
1) AbstractCommand类：抽象命令基类，用来规范执行命令api，维护receiver指针以供调用。<br>
2) ConcreteCommand类：具体命令派生类，用来具体实现命令api。<br>
3) Invoker类：客户端程序调用具体命令的辅助类，维护一个命令vector，用于命令管理。<br>
4) Receiver类：命令执行的主体类型，命令的具体实施者。<br>

**[3] 应用场景(simply borrowed from gof)** <br>
**1) parameterize objects by an action to perform**. You can express such parameterization in a procedural language with a callback function, that is, a function that's registered somewhere to be called at a later point. Commands are an object-oriented replacement for callbacks. 根据需要执行的动作来参数化命令对象。<br>
**2) specify, queue, and execute requests at different times**. A Command object can have a lifetime independent of the original request. If the receiver of a request can be represented in an address space-independent way, then you can transfer a command object for the request to a different process and fulfill the request there. 通过构建Invoker类型，并维护一个命令列表的方式，可以"自由的"控制命令的执行。<br>
**3) support undo. The Command's Execute operation can store state for reversing its effects in the command itself**. The Command interface must have an added Unexecute operation that reverses the effects of a previous call to Execute. Executed commands are stored in a history list. Unlimited-level undo and redo is achieved by traversing this list backwards and forwards calling Unexecute and Execute, respectively. 不是很清楚怎么undo???? 没有例子不能理解???? 应该也是通过Invoker类的命令存储机制实现的功能。<br>
**4) support logging changes so that they can be reapplied in case of a system crash**. By augmenting the Command interface with load and store operations, you can keep a persistent log of changes. Recovering from a crash involves reloading logged commands from disk and reexecuting them with the Execute operation. 通过Invoker类型维护一个命令执行列表，来支持针对"历史"命令的更多操作。<br>
**5) structure a system around high-level operations built on primitives operations**. Such a structure is common in information systems that support transactions. A transaction encapsulates a set of changes to data. The Command pattern offers a way to model transactions. Commands have a common interface, letting you invoke all transactions the same way. The pattern also makes it easy to extend the system with new transactions. 通过Invoker类型，能够将一些初级的命令进行组合、结构化操作，实现更高级别的功能任务。

**[4] strength & weakness** <br>
**strength** <br>
**1** 满足单一职责原则，将命令的构建(Command)、命令的触发(Client)、命令的组合结构化(Invoker)、命令的实际执行()相互解耦。**2** 满足开闭原则，可以在不修改客户端的情况下，在程序中创建新的命令(Abstrct-Concrete-Command多态机制就是为开闭原则而生的)。**3** 可以实现命令撤销和恢复的功能。**4** 可以实现操作的延时执行(not sure)。**5** 可以通过一组简单命令(由具体Command类完成)构成复杂命令(构建方式通过Invoker类完成)。<br>
**weakness** <br>
代码变得更加复杂，在命令类(command)和命令执行者(receiver)之外，增加了一个命令管理类(invoker)。

## 命令设计模式案例
**[1] 案例目标** <br>
一个烧烤摊场景，使用命令设计模式，完成顾客下单，服务员进行下单管理，烧烤师傅根据下单执行相应烧烤步骤。

**[2] 命令设计模式案例UML类图**

<center><img src="/img/in-post/cpp_img/command_1.pdf" width="80%"></center>

**[3] 命令设计模式代码示例** <br>
Part-1 Reveiver类代码
```cpp
class Barbecuer{// receiver - 执行具体order的对象
    public:
        void BakeMutton(){// 定义能够执行的功能api
            std::cout<<"bake mutton..."<<std::endl;
        }
        void BakeChickenWings(){
            std::cout<<"bake chicken wings..."<<std::endl;
        }
};
```
Part-2 AbstractCommand类代码
```cpp
class AbstractCommand{// AbstractCommand类 - 规范接口 调用receiver执行order
    protected:
        Barbecuer *ptr_receiver__;// 维护一个命令执行者类 以备调用
    public:
        // reigst receiver in abstract command
        AbstractCommand(Barbecuer *ptrreceiver)
            :ptr_receiver__(ptrreceiver){}
        // 规范具体命令类api
        virtual void ExecuteCommand() = 0;
};
class BakeMuttonCommand: public AbstractCommand{// ConcreteCommand类
    public:
        BakeMuttonCommand(Barbecuer *ptrreceiver)
            :ptr_receiver__(ptrreceiver){}
        // 具体命令api实现
        void ExecuteCommand(){
            ptr_receiver__->BakeMutton();
        }
};
class BakeChickenWings: public AbstractCommand{// ConcreteCommand类
    public:
        BakeChickenWings(Barbecuer *ptrreceiver)
            :ptr_receiver__(ptrreceiver){}
        // 具体命令api实现
        void ExecuteCommand(){
            ptr_receiver__->BakeChickenWings();
        }
};
```
Part-3 Ivoker类型代码
```cpp
class Waiter{
    private:
        // 维护一个命令列表的vector 用于命令增删 以及按照命令列表依次调用功能
        std::vector<AbstractCommand*> orders___;
    public:
        Waiter(){
            orders = new std::vector<AbstractCommand*>; 
        } 
        ~Waiter(){
            delete orders;
        }
        void SetOrders(AbstractCommand *ptrcommand){// order的新增、删除(未实现)
            // 使用typeid来实现运行时多态
            if(typeid(*ptrcommand) == typeid(BakeChickenWings)){
                std::cout<<"Log: chicken wings sold out!"<<std::endl;
            }
            elif(typeid(*ptrcommand) == typeid(BakeMutton)){
                orders__.push_back(ptrcommand);
                currenttime = time(0);// need include <ctime>
                std::cout<<"Log: add order:BakeMutton "
                    <<"Time:"<<currenttime<<std::endl;
            }
            else{
                std::cout<<"Log: No corresponding ingredients."<<std::endl;
            }
        }
        void Notify(){// 按照命令列表 依次调用AbstractCommand命令执行接口
            std::vector<AbstractCommand*>::iterator it; 
            for(it=orders__.begin(); it!=orders__.end(): ++it){
                it->ExecuteCommand();
            }
        }
};
```
Part-4 客户端运行代码示例
```cpp
int main(){
    Barbecuer *ptrcook = new Barbecuer();// 创建receiver
    // 创建concretecommand
    AbstractCommand *ptrbakemutton1 = new BakeMuttonCommand(ptrcook);
    AbstractCommand *ptrbakemutton2 = new BakeMuttonCommand(ptrcook);
    AbstractCommand *ptrbakechickenwings = new BakeChickenWings(ptrcook);
    // 创建invoker
    Waiter *ptrwaiter = new waiter();
    // 创建orderlist
    ptrwaiter->SetOrders(ptrbakecommand1);
    ptrwaiter->SetOrders(ptrbakecommand2);
    ptrwaiter->SetOrders(ptrbakechickenwings);
    // 执行orderlist
    ptrwaiter->Notify()
    // recycle
    delete ptrbakemutton1, ptrbakemutton2, ptrbakechickenwings; 
    return 0;
}
```
Part-5 客户端程序代码输出示例
```txt
Log: add order:BakeMutton Time:Sat Aug 7 13:33:10 2021
Log: add order:BakeMutton Time:Sat Aug 7 13:33:10 2021
Log: chicken wings sold out!"
bake mutton...
bake mutton...
```

## Reference
> \<大话设计模式\> chapter23 p252 <br>
> https://blog.csdn.net/xiqingnian/article/details/42105313 <br> 
> gof chapter5 p263 <br>
> https://refactoringguru.cn/design-patterns/command  

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
