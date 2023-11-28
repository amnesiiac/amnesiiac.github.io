---
layout: post
title: "cpp design pattern - adapter"
subtitle: '[structual pattern] c++设计模式之适配器模式'
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-07-28 10:13
lang: ch 
catalog: true 
categories: cpp 
tags:
  - Time 2021
  - cpp design pattern
---
## 适配器设计模式
**[1] 基本概念** <br> 
适配器(adapter)将一个类的接口转化成客户需要的另一种接口，本质上是一种接口转化。适配器模式使得原来不能互相兼容从而不能协同工作的类型能够一起工作。

**[2] 要素组成** <br>
1 Target: 客户所需要的目标接口，可以是具体类类型、抽象类类型、接口类型。<br>
2 Adaptee: 不符合客户需求而需要被改造的类、需要被适配的类。<br>
3 Adapter: 通过在内部包装一个Adaptee对象，将源接口改造成目标接口。<br>

**[3] 适配器设计模式子类别** <br>
适配器设计模式根据实现适配器的方式，有两种类型：对象适配器模式(object adapter)、类适配器模式(class adapter)。其中，对象适配器设计模式比较常用。

**[4] 应用场景** <br>
1) 希望使用的类型的接口和已有代码中产生的需求不符合。<br>
2) 已有代码中创建的接口风格不统一，需要将少部分不统一接口进行修正。保证接口对外的整洁性。<br>
**3) 不要在代码中滥用adapter**，只有当adatpee和target都不容易修改时才使用，否则导致混乱。<br>

**[5] strength & weakness**
**strength** <br>
**1** 满足单一职责原则，适配器模式支持将适配器类型相关代码从程序主要业务逻辑中分离。**2** 客户端代码通过适配器提供的接口和适配器程进行交互，可以添加新的适配器(接口一致)，而不影响客户端代码部分。<br>
**weakness** <br>
**1** 由于使用了适配器类型，导致代码整体复杂度增加。在很多情况下，直接修改Adaptee类型是更好的方案(不要滥用adapter模式)。

## 结合🚀季后赛案例适配器设计模式进行理解
### 适配器模式(对象适配器模式案例 - object adapter example)
**[1] 适配器设计模式UML类图**

<center><img src="/img/in-post/cpp_img/adapter_1.pdf" width="100%"></center>

**[2] 适配器设计模式代码示例** <br>
Part-1 target适配器目标类型代码示例
```cpp
class Player{
    protected:
        std::string name_;
    public:
        Player(std::string name):name_(name){}
        // 目标类型接口: 定义成纯虚函数 必须在派生类中进行实现
        virtual void Attack() = 0;
        virtual void Defence() = 0;
};
```
Part-2 Adaptee需要被适配类型代码示例
```cpp
class ForeignCenter{
    private:
        std::string name__;
    public:
        void SetName(std::string name):name__(name){}
        std::string GetName(){
            return name__;
        }
        // Adaptee中的待改造接口
        void ForeignAttack(){
            std::cout<<name__<<" Attack"<<std::endl;
        }
        void ForeignDefence(){
            std::cout<<name__<<" Defence"<<std::endl;
        }
};
```
Part-3 Adapter适配器代码示例(派生自target目标类型)
```cpp
class Translator: public Player{
    private:
    // 在adapter内部封装一个需要被适配的类型的对象
        ForeignCenter *ptr_foreign_center__; 
    public:
        // 初始化target 创建并初始化adaptee对象
        Translator(std::string name):Player(name){
            ptr_foreign_center__ = new ForeignCenter();
            ptr_foreign_center__->SetName(name);
        }
        ~Translator(){
            delete ptr_foreign_center__;
        }
        // no.1 适配器接口转化 添加一层封装
        void Attack(){
            ptr_foreign_center__->ForeignAttack();
        }
        // no.2 适配器接口转化 添加一层封装
        void Defence(){
            ptr_foreign_center__->ForeignDefence();
        }
};
```
Part-4 无需应用适配器的类型(其接口同适配的目标类型一致)
```cpp
class Guards{
    private:
        std::string name__;
    public:
        void SetName(std::string name):name__(name){}
        std::string GetName(){
            return name__;
        }
        // 无需改造的接口 
        void Attack(){
            std::cout<<name__<<" Attack"<<std::endl;
        }
        void Defence(){
            std::cout<<name__<<" Defence"<<std::endl;
        }
} 
```
Part-5 客户端程序代码示例
```cpp
int main(){
    // 使用无需进行接口改造的类型
    Guards guard1; 
    guard1.SetName("Tracy McGrady");
    // no.1 接口风格和目标类型一致
    guard1.Attack(); guard1.Defence();
    // 使用需要进行接口改造的类型
    Translator foreign_center1("Yao Ming");
    // no.2 经适配器调整 和目标接口风格一致 
    foreign_center1.Attack(); foreign_center1.Defence();
}
```
Part-6 客户端程序运行结果
```txt
Tracy McGrady Attack
Tracy McGrady Defence
YaoMing Attack
YaoMing Defence
```
**[3] 适配器设计模式代码结果分析思考** <br>
(1) 上述代码中，Guards类和ForeignCenter类都用来生成篮球队员对象的类型，Guards类对象接口和目标接口一致，ForeignCenter类型创建的对象中提供的操作接口和target目标接口形式不一致，因此，使用一个继承自目标接口类型的适配器类封装adaptee类的接口，方便统一风格进行调用。<br>
(2) 适配器需要维护一个adaptee类型的指针，其作用相当于一个控制句柄，用于调用adaptee类的方法接口。

### 适配器模式(类型适配器模式案例 - class adapter example)
**[1] 类型适配器设计模式代码示例** <br>
```cpp
#include<iostream>
using namespace std;

// target: 适配的目标接口
class Target{
    public:
        virtual void Request(){};
};
// adaptee: 需要被适配的对象
class Adaptee{
    public:
        void SpecificRequest(){
            cout<<"Called SpecificRequest()"<<endl;
        }
};
// adapter: 类型适配器 需多重继承自(目标类 + 适配器对象类)
class Adapter: public Adaptee, public Target{
    public:
        void Request(){
            this->SpecificRequest();
        }
};
int main(){
    Target *t = new Adapter();
    t->Request();
    return 0;
}
```
**[2] 类型适配器代码示例分析思考** <br>
上述代码展示了一种类型适配器进行适配的示例，类型适配器和对象适配器的核心区别是：类型适配器通过多重继承实现接口的封装适配；而对象适配器通过维护一个adaptee类型的对象or指针进行方法适配。

## 关于适配器设计模式的思考
**[1] 关于适配器设计模式应用场景**

➤ 根据[wiki](https://en.wikipedia.org/wiki/Adapter_pattern)，适配器设计模式(又被称为decorator pattern)可被用在如下几种场景中：<br>
*(1) How can a class be reused that does not have an interface that a client requires?* <br>
*(2) How can classes that have incompatible interfaces work together?* <br>
*(3) How can an alternative interface be provided for a class?* <br>

➤ 根据[gof chapter4 p159]，适配器设计模式(又被成为wrapper)可以被用于如下几种场景中：<br> 
*(1) you want to use an existing class, and its interface does not match the one you need.* <br>
*(2) you want to create a reusable class that cooperates with unrelated or unforeseen classes, that is, classes that don't necessarily have compatible interfaces.* <br>
*(3) **(object adapter only)** you need to use several existing subclasses, but it's impractical to adapt their interface by subclassing every one. An object adapter can adapt the interface of its parent class.* 

关于**(3) gof chapter4 p159**：特殊场景下，当需要适配的子类型太多时，使用class adapter利用多重继承的方式依次为每个子类实现适配器是得不偿失的(多重继承导致最终类型使用起来overhead很大)，因此，这种场景下，应当使用object adapter进行进行适配，只需维护一个抽象基类对象or指针。


## Reference
> \<大话设计模式\> chapter17 p189 <br>
> https://blog.csdn.net/xiqingnian/article/details/42061705 <br> 
> https://refactoringguru.cn/design-patterns/adapter

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
