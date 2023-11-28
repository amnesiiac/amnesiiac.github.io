---
layout: post
title: "cpp design pattern - visitor"
subtitle: '[behavioral pattern] c++设计模式之访问者模式' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-08-27 15:08
lang: ch 
catalog: true 
categories: cpp 
tags:
  - Time 2021
  - cpp design pattern
---
## 访问者模式
**[1] 基本概念** <br> 
访问者模式：包含访问者类型层次结构，被访问元素类型层次结构，被访问者对象管理类。访问者通过被访问者基类指针动态调用被访问者类相关接口方法，本访问者通过访问者类型指针对于上述调用进行一层封装以供外部调用。被访问者管理类维护一个被访问者对象指针列表，并按照特定逻辑顺序通过上述这层封装进行调用&操纵被访问者对象。

**[2] 要素组成** <br>
**1** AbstractVisitor类：访问者基类，为类型结构规范通用操作(visit操作)，通过操作element类型的指针来动态调用被访问者的属性。<br>
**2** ConcreteVisitor类：具体访问者类，实现抽象访问者类规定的操作。<br>
**3** AbstractElement类：抽象被访问元素类，通过操作visitor类的指针来动态调用不同的访问者方法。<br>
**3** ConcreteElement类：具体被访问元素类，实现抽象元素类型规定的操作。<br>
**4** ObjectStructure类：被访问者元素组织类型，通过维护一个有序容器按照一定顺序对于具体元素类型对象进行管理，并调用元素对象的相关方法。<br>

**[3] 应用场景(simply borrowed from gof)** <br>
**(i)** an object structure contains many classes of objects with differing interfaces, and you want to perform operations on these objects that depend on their concrete classes. 如果希望通过typeid运算符实现了调用的运行时多态，根据具体派生子类类型调用内部不同的实现。<br>
**(ii)** many distinct and unrelated operations need to be performed on objects in an object structure, and you want to avoid "polluting" their classes with these operations. Visitor lets you keep related operations together by defining them in one class. When the object structure is shared by many applications, use Visitor to put operations in just those applications that need them. 访问者模式实现了被访问的元素对象和对象相关的一系列操作相互分离(解耦合)。<br>
**(iii)** the classes defining the object structure rarely change(访问者-被访问者类型接口不经常改变), but you often want to define new operations over the structure(对于被访问者元素对象组织类型的方法需求总是频繁变动-Display函数的实现方式). Changing the objects structure classes requires redefining the interface to all visitors, which is potentially costly. If the object structure classes change often, then it's probably better to define the operations in those classes. 将被访问者对象元素组织相关操作从类型中单独抽离解耦，从而应对不需要改动"访问者-被访问者"内部实现的前提下，修改被访问者接口外层封装组织函数的具体实现。<br>

**[4] strength & weakness** <br>
**strength** <br>
**i** 满足开闭原则。可以在被访问者对象封装的接口上定义新的操作，无需改动"访问者-被访问者"内部类型实现。**ii** 满足单一职责原则。访问者类型层次结构相当于软件发布的不同版本，对于每个版本的修改、添加、删除操作不会影响到其他派生类型，只需在被访问者封装处进行微调即可。<br>
**weakness** <br> 
**i** 被访问者类型层次结构和访问者类型层次结构之间的实现&接口有一定程度的耦合，改动其中任意一个需要同时修改相关联的方法封装。

**[5] 与其他设计模式的关系** <br>
**(i)** 访问者可以看成是命令模式的加强模式，命令模式的具体命令实施者只由一个类来完成(相当于访问者类型)，而访问者模式具体命令的实施以动态的在访问者类层次结构中选择相应类进行调用实现。<br>
**(ii)** 将同时使用访问者模式和迭代器模式，以遍历复杂的数据结构(数据组织方式复杂 & 数据类型多种多样)，以完成遍历+针对特定对象调用不同接口操作的复杂任务。<br>
**(iii)** 可以将访问者模式和组合模式(构建一种类似树的类层次结构)一起使用，组合设计模式中的树类型层次结构作为访问者模式的"被访问者"，从而在访问者模式中的obj_strcture类型中定义遍历方式，并且能够按照遍历顺序为不同的类型层次中的对象执行不同的操作。


## 访问者设计模式案例
**[1] 案例目标** <br>
构建"访问者类层次结构 - 被访问元素类层次结构 - 被访问元素对象管理层次结构"模型。其中访问者类型用于根据被访问的元素属性进行处理以产生相应输出；被访问者类型用访问者类型的方法并封装后对外开放访问接口；被访问者元素对象管理类维护列表，以按顺序调用相应被访问元素对象方法。

**[2] 访问者设计模式案例UML类图**

<center><img src="/img/in-post/cpp_img/visitor_1.pdf" width="100%"></center>

**[3] 访问者设计模式代码示例** <br>
##### Part-1 抽象&具体访问者类层次结构定义 \<visitor.h\>
```cpp
#ifndef VISITOR_H
#define VISITOR_H

#include "Element.h"
class AbstractAction{
    public:// stipulate general api
        virtual void GetManConclusion(AbstractPerson *ptrelement)=0;
        virtual void GetWomenConclusion(AbstreactPerson *ptrelement)=0;
};
class ConcreteSuccess: public AbstractAction{
    public:
        void GetManConclusion(AbstractPerson *ptrelement)
        void GetWomenConclusion(AbstractPerson *ptrelement);
};
class ConcreteFailure: public AbstractAction{
    public:
        void GetManConclusion(AbstractPerson *ptrelement);
        void GetWomenConclusion(AbstractPerson *ptrelement);
};
class ConcreteAmativeness: public AbstractAction{
    public:
        void GetManConclusion(AbstractPerson *ptrelement)
        void GetWomenConclusion(AbstractPerson *ptrelement);
};
#endif
```
##### Part-2 抽象&具体访问者类层次结构定义 \<visitor.cpp\>
```cpp
#include "visitor.h"
#include <iostream>
#include <string>
#include <typeinfo> // for typeid operator

void ConcreteSuccess::GetManConclusion(AbstractPerson *ptrelement){
    std::cout<<typeid(*ptrelement).name()<<"'s "<<typeid(this).name()
        <<" is due to a sexy women."
}
void ConcreteSuccess::GetWomenConclusion(AbstractPerson *ptrelement){
    std::cout<<typeid(*ptrelement).name()<<"'s "<<typeid(this).name()
        <<" is due to a strong man."
}
void ConcreteFailure::GetManConclusion(AbstractPerson *ptrelement){
    std::cout<<typeid(*ptrelement).name()<<"'s "<<typeid(this).name()
        <<" is due to an ugly-looking women."
} 
void ConcreteFailure::GetWomenConclusion(AbstractPerson *ptrelement){
    std::cout<<typeid(*ptrelement).name()<<"'s "<<typeid(this).name()
        <<" is due to an impotence man."
}
void ConcreteAmativeness::GetManConclusion(AbstractPerson *ptrelement){
    std::cout<<typeid(*ptrelement).name()<<"'s "<<typeid(this).name()
        <<" is due to a melting women."
}
void ConcreteAmativeness:GetWomenConclusion(AbstractPerson *ptrelement){
    std::cout<<typeid(*ptrelement).name()<<"'s "<<typeid(this).name()
        <<" is due to a brave man."
}
```
##### Part-3 抽象&具体被访问的元素类层次结构定义 \<element.h\>
```cpp
#ifndef ELEMENT_H
#define ELEMENT_H

class AbstractAction;// 前置类声明
class AbstractPerson{
    public:
        virtual void Accept(AbstractAction *ptraction)=0;
};
class ConcreteMan: public AbstractPerson{
    public:
        void Accept(AbstractAction *ptraction);
};
class ConcreteWomen: public AbstractPerson{
    public:
        void Accept(AbstractAction *ptraction);
};
#endif
```
##### Part-4 抽象&具体被访问类层次结构实现 \<element.cpp\>
```cpp
#include "element.h"
#include "action.h"

void ConcreteMan::Accept(AbstractAction *ptrvisitor){
    ptrvisitor->GetManConclusion(this);
}
void ConcreteWomen::Accept(AbstractAction *ptraction){
    ptrvisitor->GetWomenConclusion(this);
}
```
##### Part-5 具体被访问者对象管理数据结构类型 \<objectstructure.h\>
```cpp
#include "element.h"
#include "visitor.h"
#include <vector>
#include <string>
#include <iostream>

class ObjectStructure{
    private:// maintain a vec hold handles of elements
        std::vector<AbstractPerson*> objvec___;
    public:
        ObjectStructure();// ctor for dtor
        ~ObjectStructure():// dtor for its own vector member
        void Attach(AbstractPerson*);// add new element for operate
        void Detach(AbstractPerson*):// delete element
        void Display(AbstractPerson*);// call element method
};
```
##### Part-6 \<objectstructure.cpp\>
```cpp
#include "objectstructure.h"
#include <iterator>

ObjectStructure::~ObjectStructure(){// dtor: delte one by one
    std::vector<AbstractPerson*>::iterator it;
    for(it=objvec___.begin(); it!=objvec___.end(); ++it){
        delete *it;
    }
}
void ObjectStructure::Attach(AbstractPerson* ptrelement){// add ele
    objvec___.push_back(ptrelement);
}
void ObjectStructure::Detach(AbstractPerson* ptrelement){// delete ele
    // std::vector<AbstractPerson*>::iterator it;
    // for(it=objvec___.begin(); it!=objvec___.end(): ++it){
    //     if(it==ptrelement){// need overload = for abstractperson
    //         objvec___.erase(it);
    //     }
    // }
}
void ObjectStructure::Display(AbstractAction *ptrvisitor){
    std::vector<AbstractPerson*>::iterator it;
    for(it=objvec___.begin(); it!=objvec___.end(); ++it){
        it->Accept(ptrvisitor);// call method od element & handle the visitor
    }
}
```
##### Part-7 \<client.cpp\>
```cpp
#include "objectstructure.h"
#include <iostream>
#include <cstdlib>

int main(){
    ObjectStructure obj_structure;
    obj_structure.Attach(new ConcreteMan());
    obj_structure.Attach(new ConcreteWomen());
    ConcreteSuccess victory;
    obj_structure.Display(&victory);
    ConcreteFailure lose;
    obj_structure.Display(&lose);
    ConcreteAmativeness love;
    obj_structure.Display(&love);
    return 0;
}
```

##### Part-8 \<out\>
```txt
ConcreteMan's ConcreteSuccess is due to a sexy women.
ConcreteWomen's ConcreteSuccess is due to a strong man.
ConcreteMan's ConcreteFailure is due to an ugly-looking women.
ConcreteMan's ConcreteFailure is due to an impotence man.
ConcreteMan's ConcreteAmativeness is due to a melting women.
ConcreteWomen's ConcreteAmativeness is due to a brave man.
```



## Reference
> \<大话设计模式\> chapter28 p309 <br>
> https://blog.csdn.net/xiqingnian/article/details/42295987 <br>
> gof chapter5 p366 <br>
> https://refactoringguru.cn/design-patterns/visitor
 
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
