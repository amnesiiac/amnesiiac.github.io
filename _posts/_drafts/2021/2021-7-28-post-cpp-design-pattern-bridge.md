---
layout: post
title: "cpp design pattern - bridge"
subtitle: '[structual pattern] c++设计模式之桥接模式'       
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-07-28 15:17
lang: ch 
catalog: true 
categories: cpp 
tags:
  - Time 2021
  - cpp design pattern
---
## 桥接设计模式
**[1] 基本概念** <br> 
桥接模式：将抽象部分和抽象实现部分相互分离，使抽象部分和抽象实现部分都可以独立控制。<br>

**[2] 要素组成** <br>
1) 抽象实体abstract class: 参考下面示例理解。<br>
2) 抽象实现implementation: 参考下面示例理解。<br>

**[3] 应用场景(simply borrowed from gof)** <br>
1) you want to avoid a permanent binding between an abstraction and its implementation. This might be the case, for example, when the implementation must be selected or switched at run-time. 防止抽象和抽象实现的永久绑定，需要将它们解绑。

2) both the abstractions and their implementations should be extensible by subclassing. In this case, the Bridge pattern lets you combine the different abstractions and implementations and extend them independently. 抽象和抽象实现需要分别进行扩展，需要将它们解绑。

3) changes in the implementation of an abstraction should have no impact on clients; that is, their code should not have to be recompiled. 将抽象和抽象实现代码相互分离，改变抽象底层实现逻辑不会影响抽象接口，抽象接口不变所以抽象逻辑的实现对于客户端程序没有影响，因此client代码无需重编译。

4) (适用于C++) you want to hide the implementation of an abstraction completely from clients. In C++ the representation of a class is visible in the class interface. 将抽象和抽象实现相互分离，实现对客户端代码隐藏抽象的实现流程。

5) you have a proliferation(细胞增殖) of classes. Such a class hierarchy indicates the need for splitting an object into two parts. Rumbaugh uses the term "nested generalizations" [RBP+91] to refer to such class hierarchies. 如果抽象实体类层次结构是一种‘潜在’不断增殖的类型，为了避免在每个增殖(inherit)的子类别都重新进行抽象实现，可以将重复的抽象实现部分抽取出来，使用bridge方式进行独立管理，这样每增殖一个subclass，只需要调用抽象实现api即可。

6) you want to share an implementation among multiple objects (perhaps using reference counting), and this fact should be hidden from the client. A simple example is Coplien's String class [Cop92], in which multiple objects can share the same string representation (StringRep). 如果抽象的实现需要为多个抽象类层次接口服务，那么显然需要将抽象实现部分抽取出来，单独模块化进行管理。

**[4] strength & weakness** <br>
**strength** <br>
**1** 可以创建和平台无关的应用程序。**2** 客户端代码可仅和高层抽象部分进行交互，而不会频繁调用平台架构代码。**3** 开闭原则，可以新增抽象部分、抽象实现部分，无需改动原有设施。**4** 桥接模式遵循单一职责原则，将抽象部分和抽象实现相互分离。<br>
**weakness** <br>
**1** 如何对于内部属性内聚程度非常高的类型代码设施使用桥接模型可能会事得其反，越搞越复杂。


### 合成/聚合复用原则(CARP)
**[1] Basics** <br>
**聚合**(aggregation)表示一种弱的拥有关系，A聚合B：A对象可以包含B对象，但是B对象不是A对象的一部分。
**合成**(composition)表示一种强的拥有关系，B合成A：A对象包含B对象，且B对象是A对象的一部分。
**Example:** 大雁和雁群属于聚合的关系，大雁和自己的翅膀属于合成的关系。

**[2] Concepts** <br>
合成/聚合复用原则：尽量使用合成聚合，尽量不要使用类继承。<br>
解释1：如果A和B满足聚合/合成关系如：A是手机，B是手机软件，aggreation关系。不要将它们设计成class B: public A的形式，应当分别设计A/B类，并使用bridge pattern完成协作功能。<br>
解释2：优先使用合成聚合，有助于确保每个类的封装性，保证每个类能够集中在单个任务上，从而控制类和类继承层次保持在较小的规模上，从而避免形成一个庞然大物难以维护(指类层次结构)。

### bridge设计模式产生背景
**[1] 手机案例的两种紧耦合模式UML类图** <br>

<center><img src="/img/in-post/cpp_img/bridge_1.pdf" width="100%"></center>

**[2] 为什么需要使用bridge模式对于上述类图结构进行refactor？** <br>
在[factory pattern](http://localhost:4000/cpp/2021/07/20/post-cpp-design-pattern-factory-pattern/#设计模式引言)的总结中，软件的工程组织应当遵循内部高内聚(cohesion)外部低耦合(coupling)的思路。事物存在普遍的联系，内部和外部的划分具有相对性。不妨将PhoneBrand和PhoneSoftware组成一个整体概念(作为高内聚的内部)，那么在内部如何应用有效的组织方式，使得内聚部分代码在阅读和应用上更加高效？这就是bridge设计模式应用的目标。

本文后续关于PhoneBrand和PhoneSoftware的例子使用一种松耦合的方式组织类结构，使得内部的代码组织清晰，且功能上将PhoneBrand和PhoneSoftware分离，使得每个部分可以独立地进行控制。


## bridge设计模式 - 抽象&其抽象实现的解耦
### 桥接模式案例
**[1] 桥接设计模式UML类图**

<center><img src="/img/in-post/cpp_img/bridge_2.pdf" width="100%"></center>

**[2] 桥接设计模式代码示例** <br>
Part-1 抽象部分代码示例(需要维护抽象实现类型指针，实际编写需注意代码组织顺序)
```cpp
class PhoneBrand{// no.1 抽象基类 -> 主要负责抽象定义和维护桥接器
    protected:
        PhoneSoftware *ptr_software__;// 桥接器: 沟通抽象部分和抽象实现部分的桥梁
        virtual ~PhoneBrand();
    public:
        void SetPhoneSoftware(PhoneSoftware *ptr_software){// 设置桥接器
            this->ptr_software__ = ptr_software;
        }
        virtual void Run() = 0;// 需要在派生类中实现
};
class Nokia{// no.2 派生类1
    public:
        Nokia(){// ctor
            std::cout<<"Nokia, walnut breaker!"<<std::endl;
        }
        void Run(){// 通过桥接器 调用抽象实现部分
            ptr_software__->Run();
        }
        ~Nokia(){
            std::cout<<"Bye, Nokia!"<<std::endl;
        }
};
class BlackBerry{// no.3 派生类2
    public:
        BlackBerry(){// ctor
            std::cout<<"BlackBerry, faith top-up!"<<std::endl;
        }
        void Run(){// 通过桥接器 调用抽象实现部分 
            ptr_software__->Run();
        }
        ~BlackBerry(){
            std::cout<<"Bye, BlackBerry!"<<std::endl;
        }
}:
```
Part-2 抽象实现类层次结构部分代码示例
```cpp
class PhoneSoftware{// no.1 抽象实现基类 -> 主要负责抽象部分的实现过程
    public:
        virtual void Run() = 0;// 需要在派生类中实现
};
class Game: public PhoneSoftware{// no.2 派生类1
    public:
        void Run(){// 具体抽象实现
            std::cout<<"Super Mary, Go!"<<std::endl;
        }
};
class AddressBook: public PhoneSoftware{// no.3 派生类2
    public:
        void Run(){// 具体抽象实现
            std::cout<<"AddressBook, Open!"<<std::endl;
        }
};
```
Part-3 客户端代码示例
```cpp
int main(){
    PhoneBrand *ptr_nokia = new Nokia();// init
    ptr_nokia->SetPhoneSoftware(new AddressBook());// set bridge
    ptr_nokia->Run();// call through the bridge
    delete ptr_nokia; ptr_nokia=nullptr;// recycle 
    PhoneBrand *ptr_blackberry = new BlackBerry();// init
    ptr_blackberry->SetPhoneSoftware(new Game());// set bridge
    ptr_blackberry->Run();// call through the bridge
    ptr_blackberry->SetPhoneSoftware(new AddressBook());// reset bridge
    ptr_blackberry->Run();// re-call through the bridge
    delete ptr_blackberry; ptr_blackberry=nullptr;// recycle
    return 0;
}
```
Part-4 程序执行结果及其分析
```txt
Nokia, walnut breaker!
AddressBook, Open!
Bye, Nokia!
BlackBerry, faith top-up!
Super Mary, Go!
AddressBook, Open!
Bye, BlackBerry!
```

**[3] 桥接设计模式代码结果分析思考** <br>
simple, omitted.


## Reference
> \<大话设计模式\> chapter22 p238 <br>
> https://blog.csdn.net/xiqingnian/article/details/42100247 <br>
> https://refactoringguru.cn/design-patterns/bridge 

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
