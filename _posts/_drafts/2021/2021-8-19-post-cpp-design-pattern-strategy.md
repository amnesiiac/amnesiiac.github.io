---
layout: post
title: "cpp design pattern - strategy"
subtitle: '[behavioral pattern] c++设计模式之策略模式' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-08-19 10:52
lang: ch 
catalog: true 
categories: cpp 
tags:
  - Time 2021
  - cpp design pattern
---
## 策略模式
**[1] 基本概念** <br> 
策略模式：定义了一个算法类层次结构，其中具体每算法子类都可以进行相互替换。并且在此基础上，算法的不同变化(切换不同算法)不会影响到使用算法的客户端程序。

**[2] 要素组成** <br>
**1** AbstractStrategy抽象策略类型：策略类层次结构的基类，用于规范策略通用接口api。<br>
**2** ConcreteStrategy具体策略类型：具体策略类，具体实现基类规范的通用接口。<br>
**3** Context策略管理类型：维护一个指向抽象策略基类的指针，方便客户端程序进行策略选取、调用策略接口以返回应用策略后的结果。<br>

**[3] 应用场景(simply borrowed from gof)** <br>
**(i)** many related classes(Strategy类) differ only in their behavior. Strategies provide a way to configure a class(Context类) with one of many behaviors. 一些相互关联的接口相同但是接口内的行为不同，策略模式允许配置一种类型，该类型在实例化时只需要使用相互关联类型中的某种接口行为。<br>
**(ii)** you need different variants of an algorithm. For example, you might define algorithms reflecting different space/time trade-offs. Strategies can be used when these variants are implemented as a class hierarchy of algorithms [HO87]. 通过策略模式，可以将算法中空间换取时间、时间换取空间等相互关联的策略进行统筹，并在构建的Context类型中予以选择应用。<br>
**(iii)** an algorithm uses data that clients shouldn't know about. Use the Strategy pattern to avoid exposing complex, algorithm-specific data structures. 当所构建的算法类层次结构中的数据、数据结构不想暴露给客户端程序，可以使用策略模式，并使用Context类将算法、数据相关的内部api进行封装，客户端程序通过间接调用Context类封装的接口使用底层算法数据结构。<br>
**(iv)** a class defines many behaviors, and these appear as multiple conditional statements in its operations. Instead of many conditionals, move related conditional branches into their own Strategy class. 策略模式可以用来简化繁杂的条件类层次结构，通过将不同的条件结构抽象解耦为单独的策略类型可以使得程序简洁、可控、易修改(单一职责原则)。

**[4] strength & weakness** <br>
**strength** <br>
**i** 可以实现运行时动态切换类型对象内部策略(运行时多态)。**ii** 可以将策略(算法)内部具体实现和调用策略(算法)的代码相互隔离。**iii** 使用组合的设计思想代替复杂的类型继承。**iv** 满足开闭原则，无需修改Context类型代码就可以添加新的算法。<br>
**weakness** <br> 
**i** 对于极少变动的策略类型层次(策略设计模式提供的易扩展性无法体现)，使用策略设计模式可能会让程序变得复杂。**ii** 客户端虽然使用Context类型间接调用内部策略(算法)，但是客户端必须对于策略有一定了解才能选择合适的策略(???)。

**[5] 与其他设计模式的关系** <br>
**(i)** 策略模式同桥接模式、状态模式，甚至适配器模式的接口构建方式非常相似，它们都是基于组合设计模式的思想，即充分应用单一职责原则，将底层实现、和应用代码想分离。<br>
**(ii)** 命令模式同策略模式从形式上很相似，但是它们的内在目的不同。策略模式用于描述完成某件事情的方式方法上的不同，而命令模式侧重于将具体的操作抽象为对象、将操作所需的控制参数转化为对象的数据成员，并且可以控制对象的生命期、执行调用、撤销等操作。<br>
**(iii)** 装饰模式允许改变对象的外表(接口形式)，而策略模式允许改变对象内部实现。<br>
**(iv)** 模版方法模式基于构建类型层次结构来拓展某种对象内部实现，策略模式通过"组合"的思想来增加对象内部实现方式。前者是静态的，需要在运行前完成类型继承结构配置，后者是静态的，可以在运行时，动态的进行方法切换(模版方法模式的对象一经创建就绑定了特定的方法，而策略模式的对象可以运行时动态的绑定)。<br>
**(v)** 状态设计模式可以看成是策略模式的扩展。两者都基于组合设计模式的思想。策略模式使得每个不同的策略(算法)之间相互独立，但是状态设计模式无此限制，它允许不同的状态之间间接地通过Context类型进行"交互"。


## 策略设计模式案例
**[1] 案例目标** <br>
构建一个策略类层次结构，具体策略类型分别为原价销售(无优惠)、满减优惠、折扣优惠。其中满减、折扣相应优惠参数可人为设置。另外构建一个策略管理类型，用于根据客户端输入创建不同的策略，以及调用策略的相应接口计算应用策略之后的结果。

**[2] 策略设计模式案例UML类图**

<center><img src="/img/in-post/cpp_img/strategy_1.pdf" width="100%"></center>

**[3] 策略设计模式代码示例** <br>
##### Part-1 抽象&具体策略类型定义+实现代码示例 \<strategy.h\>
```cpp
#ifndef STRATEGY_H
#define STRATEGY_H

#include <string>
#include <math.h>

class AbstractCashStrategy{
    public:// 抽象接口 -> 计算策略产生的结果
        virtual double AcceptCash(double money) = 0;
};
class ConcreteNormalCashStrategy: public AbstractCashStrategy{
    public:// 没有任何优惠折扣
        // 没有自己的数据成员 -> 使用编译器提供的默认ctor即可
        double AcceptCash(double money){
            return money;
        }
};
class ConcreteReturnCashStrategy: public AbstractCashStrategy{
    private:// 消费每满足money_condition___, 减免money_return___
        double money_condition___;// 300
        double money_return___;// 100
    public:// 拥有自己的数据成员 因此需要自定义ctor进行成员初始化
        ConcreteReturnCashStrategy(double moneycondition, double moneyreturn)
            :money_condition___(moneycondition),money_return___(moneyreturn){}
        double AcceptCash(double money){
            double result=money;
            if(money > money_condition___){
                result=money-floor(money/money_condition___)*money_return___;
            }
            return result;
        }
};
class ConcreteRebateCashStrategy: public AbstractCashStrategy{
    private:
        double money_rebate___;// 折扣
    public:// 拥有自己的数据成员 因此需要自定义ctor进行成员初始化
        ConcreteRebateCashStrategy(double moneyrebate)
            :money_rebate___(moneyrebate){}
        double AcceptCash(double money){
            return money * money_rebate___;
        }
};
#endif
```
##### Part-2 Context策略管理类代码示例 \<context.h\>
```cpp
#ifndef CONTEXT_H
#define CONTEXT_H

#include "strategy.h"
 
class Context{// strategy manager
    private:// maintain a abstract strategy pointer
        AbstractCashStrategy *ptr_cash_strategy___;
    public:// 根据参数type 决定将数据成员指针初始化成决策类层次结构的那种具体类型
        Context(int type):ptr_cash_strategy___(nullptr){
            switch(type){// 根据客户端用户输入 创建特定类型策略对象 执行相应策略
                case 1:// call ctor#1 无任何优惠&折扣
                    ptr_cash_strategy___ = new ConcreteNormalCashStrategy();
                    break;
                case 2:// call ctor#2 满多少返现
                    ptr_cash_strategy___ = 
                        new ConcreteReturnCashStrategy(300, 100);
                    break;
                case 3:// call ctor#3 折扣
                    ptr_cash = new ConcreteRebateCashStrategy(0.8);
                    break;
                default:;
            } 
        }
        ~Context(){
            if(ptr_cash_strategy___!=nullptr){
                delete ptr_cash_strategy___;// 析构对象并释放资源
                ptr_cash_strategy___ = nullptr;// 置空
            }
        }
        double GetResult(double money){// 获取应用特定策略后的结果
            return ptr_cash_strategy___->AcceptCash(money);
        }
};
#endif
```
##### Part-3 客户端代码代码示例 \<main.h\>
```cpp
#include "context.h"
#include <iostream>
#include <stdlib.h>

int main(){
    double total=0;
    double total_prices=0;
    // normal charge
    Context *ptr_normal_context = nullptr;
    ptr_normal_context = new Context(1);// strategy#1
    total_prices = ptr_normal_context->GetResult(300);
    total+=total_prices;
    std::cout<<"Normal charge: "<<total_prices
        <<" Total: "<<total<<std::endl;
    total_prices = 0;// reset
    // return charge
    Context *ptr_return_context = nullptr;
    ptr_return_context = new Context(2);// case1
    total_prices = ptr_return_context->GetResult(700);
    total+=total_prices;
    std::cout<<"Return charge: "<<total_prices
        <<" Total: "<<total<<std::endl;
    total_prices = 0;// reset
    // rebate charge
    Context *ptr_rebate_context = nullptr;
    ptr_rebate_context = new Context(3);// case1
    total_prices = ptr_rebate_context->GetResult(300);
    total+=total_prices;
    std::cout<<"Rebate charge by 0.8: "<<total_prices
        <<" Total: "<<total<<std::endl;
    total_prices = 0;// reset
    // recycle
    if(ptr_normal_context!=nullptr){
        delete ptr_normal_context; ptr_normal_context=nullptr;
    }
    if(ptr_return_context!=nullptr){
        delete ptr_return_context; ptr_return_context=nullptr;
    }
    if(ptr_rebate_context!=nullptr){
        delete ptr_rebate_context; ptr_rebate_context=nullptr;
    }
    return 0;
}
```
##### Part-4 客户端代码输出示例 \<txt.h\>
```txt
Normal charge: 300 Total: 300 
Return charge: 500 Total: 800 
Rebate charge by 0.8: 240 Total: 1040
```

## Reference
> \<大话设计模式\> chapter2 p35 <br>
> https://blog.csdn.net/xiqingnian/article/details/41855391 <br>
> gof chapter5 p349 <br>
> https://refactoringguru.cn/design-patterns/strategy

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
