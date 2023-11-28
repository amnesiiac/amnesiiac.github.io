---
layout: post
title: "cpp design pattern - iterator"
subtitle: '[behavioral pattern] c++设计模式之迭代器模式' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-08-08 20:50
lang: ch 
catalog: true 
categories: cpp 
tags:
  - Time 2021
  - cpp design pattern
---
## 迭代器模式
**[1] 基本概念** <br> 
迭代器模式：提供一种方法顺序地访问一个聚合对象中的各个元素，并且不暴露对象的内部表示。

**[2] 要素组成** <br>
**1) AbstractIterator类**：抽象迭代器类，用于规范迭代器的基本操作接口。<br>
**2) ConcreteIterator类**：具体迭代器类，用于实现规范的迭代器基本操作接口。<br>
**3) AbstractAggregate类**：抽象聚合类，用于规范资源管理的功能接口(接口供迭代器调用)。<br>
**4) ConcreteAggregate类**：具体聚合类，用于实现规范的资源管理功能接口(接口供迭代器调用)。<br>

**[3] 应用场景(simply borrowed from gof)** <br>
**1** to access an aggregate object's contents without exposing its internal representation. 迭代器的一种功能是作为指向"aggregate obj"的指针(access without exposing)。<br>
**2** to support multiple traversals of aggregate objects. 迭代器的另一种功能是用于支持"aggregate obj"的多种遍历。<br>
**3** to provide a uniform interface for traversing different aggregate structures (that is, to support polymorphic iteration). 通过构建抽象迭代器-具体迭代器类型层次结构，可以实现针对不同的"aggregate obj"进行多态地遍历。

**[4] strength & weakness** <br>
**strength** <br>
**1** 满足单一职责原则。Aggregate类型抽象用于管理底层数据结构资源。Iterator类型用于构建迭代器相关操作。**2** 满足开闭原则。可以添加新的Aggregate类型或者新增Iterator迭代器，无需修改现有的代码(类层次结构、多态特性的优势)。**3** 支持构建多个Iterator对象"并行"遍历相同的资源，每个迭代器对象在职能上为独立的。<br>
**weakness** <br>
**1** 如果你的程序只和简单的资源打交道，那么使用迭代器模式可能矫枉过正、大材小用(迭代器一般适用于较为复杂的资源遍历、或者需要自定义遍历方式的场景下)。**2** 对于一些特定的资源集合，使用迭代器的效率可能回严重低于直接遍历的方式。

## 迭代器设计模式案例
**[1] 案例目标** <br>
设计一个迭代器类型(包含正向迭代器和反向迭代器类型)，为迭代器类型定义如下几种操作：返回首元素、返回下一个元素、判断迭代器是否位于尾后、返回特定下标的元素数值。<br>
提示：迭代器类型只负责和指针相关的操作，而迭代器所指向的资源(采用数据结构进行管理)需要另外使用一个聚合类(aggregate)进行管理(iterator通过维护一个aggregate类型指针，以调用GetVector方法来实现在iterator类中操纵vector对象)。

**[2] 迭代器设计模式案例UML类图**

<center><img src="/img/in-post/cpp_img/iterator_1.pdf" width="100%"></center>

**[3] 迭代器设计模式代码示例** <br>
Part-1 抽象、具体迭代器类型的定义 \<Iterator.h\>
```cpp
#ifndef ITERATOR_H
#define ITERATOR_H

#include <iostream>
#include <vector>
#include "Aggregate.h"

// abstract iterator 
class AbstractIterator{
    public:
        // stipulate the general api of iterator
        virtual std::string First()=0;// return 1st ele
        virtual std::string Next()=0;// return next ele
        virtual bool IsDone()=0;// end?
        virtual std::string CurrentItem()=0;// return vec[idx] 
};
// concrete forward iter 
class ConcreteForwardIterator: public AbstractIterator{
    private:
        // maintain aggregate pointer for resources management
        Aggregate *ptr_aggregate___;
        int current___;// index
    public:
        ConcreteForwardIterator(Aggregate *ptraggregate);
        std::string First();
        std::string Next();
        bool IsDone();
        std::string CurrentItem();
};
// concrete backward iter
class ConcreteBackwardIterator: public AbstractIterator{
    private:
        Aggregate *ptr_aggregate___;
        int current___;
    public:
        ConcreteBackwardIterator(Aggregate *ptraggregate);
        std::string First();
        std::string Next();
        bool IsDone();
        std::string CurrentItem();
};
#endif
```
Part-2 具体迭代器类型定义的实现 \<Iterator.cpp\>
```cpp
#include "Iterator.h"
// Forward Iterator implementation
ConcreteForwardIterator::ConcreteForwardIterator(Aggregate *ptraggregate){
    this->ptr_aggregate___ = (ConcreteForwardIterator*)ptraggregate;
    current___ = 0;
}
std::string ConcreteForwardIterator::First(){
    return ptr_aggregate___->GetVector()->at(0);
}
std::string ConcreteForwardIterator::Next(){
    ++current___;// add index
    if(current___ < ptr_aggregate___->GetVector().size()){// check rationality
        return ptr_aggregate___->GetVector().at(current___);
    }
}
bool ConcreteForwardIterator::IsDone(){
    return current___ >= ptr_aggregate___->GetVector().size()? true:false;
}
std::string ConcreteForwardIterator::CurrentItem(){
    return ptr_aggregate___->GetVector().at(current___);
}
// Backward Iterator implementation
ConcreteBackward::ConcreteBackwardIterator(Aggregate *ptraggregate){
    this->ptr_aggregate___ = (ConcreteBackwardIterator*)ptraggregate;
    current___ = ptr_aggregate___->GetVector().size()-1;
}
std::string ConcreteBackwardIterator::First(){
    return ptr_aggregate___->GetVector().end();
}
std::string ConcreteBackwardIterator::Next(){
    --current___;
    if(current>=0){// check rationality
        return ptr_aggregate___->GetVector().at(current___);
    }
}
bool ConcreteBackwardIterator::IsDone(){
    return current___<0? true:false;
}
std::string ConcreteForwardIterator::CurrentItem(){
    return ptr_aggregate___->GetVector().at(current___);
}
```
Part-3 抽象、具体聚合类类型的定义 \<Aggregate.h\>
```cpp
#ifndef AGGREGATE_H
#define AGGREGATE_H

#include <iostream>
#include <vector>
#include <string>

// import 3 classes 
class AbstractIterator;
class ConcreteForwardIterator;
class ConcreteBackwardIterator;

class AbstractAggregate{// abstract - 实现资源管理底层api
    public:
        // stipulate aggregate class api 
        virtual AbstractIterator* CreateForwardIterator() = 0; 
        virtual AbstractIterator* CreateBackwardIterator() = 0; 
        virtual std::vector<std::string> GetVector() = 0;
};
class ConcreteAggregate: public AbstractAggregate{// concrete
    private:
        std::vector<std::string> *ptr_item___;// maintain ptr for resouces
    public:
        ConcreteAggregate();// apply for resource
        ~ConcreteAggregate();// recycle resource resource
        // implementation of the base::stipulation
        AbstractIterator* CreateForwardIterator(); 
        AbstractIterator* CreateBackwardIterator();
        std::vector<std::string>* GetVector();
        // implementation of the added methods
        int Count();
        std:string GetElement(int index);
        void SetElement(int index, std::string str);
};
#endif
```
Part-4 具体聚合类类型的实现 \<Aggregate.cpp\>
```cpp
ConcreteAggregate::ConcreteAggregate(){
    ptr_item___ = new std::vector<std::string>;
}
ConcreteAggregate::~ConcreteAggregate(){
    delete ptr_item___;
}
ConcreteAggregate::CreateForwardIterator(){
    AbstractIterator *it = new ConcreteForwardIterator();
    return it;
}
ConcreteAggregate::CreateBackwardIterator(){
    AbstractIterator *it = new ConcreteBackwardIterator();
    return it;
}
std::vector<std::string>* ConcreteAggregate::GetVector(){
    return ptr_item___; 
}
int ConcreteAggregate::Count(){
    ptr_item___->size();
}
std::string ConcreteAggregate::GetElement(int index){
    return ptr_item___->at(index);
}
void ConcreteAggregate::SetElement(int index, std::string str){
    ptr_item___->at(index) = str; 
}
```
Part-5 客户端应用程序 \<main.cpp\>
```cpp
#include <iostream>
#include <cstdlib>
#include "Iterator.h" // 包含迭代器类型头文件
#include "Aggregate.h" // 包含聚合工具类型头文件

int main(){
    // apply for resources
    ConcreteAggregate *ptraggregator = new ConcreteAggregate(); 
    // config the resources
    ptraggregator->GetVector().push_back("passenger1");
    ptraggregator->GetVector().push_back("passenger2");
    ptraggregator->GetVector().push_back("passenger3");
    ptraggregator->GetVector().push_back("passenger4");
    ptraggregator->GetVector().push_back("passenger5");
    ptraggregator->GetVector().push_back("passenger6");
    // forward iterator
    std::cout<<"Using forward iterator..."<<std::endl;
    AbstractIterator *ptrforwarditerator->CreateForwardIterator();
    while(!ptrforwarditerator->IsDone()){
        std::cout<<ptrforwarditerator->CurrentItem
            <<" tickets required"<<std::endl;
        ptrforwarditerator->Next();
    }
    std::cout<<std::endl;
    // backward iterator
    std::cout<"Using backward iterator..."<<std::endl;
    AbstractIterator *ptrbackwarditerator->CreateBackwardIterator();
    while(!ptrbackwarditerator->IsDone()){
        std:cout<<ptrbackwarditerator->CurrentItem
            <<" tickets required"<<std::endl;
        ptrbackwarditerator->Next()
    }
    std::cout<<std::endl;
    delete ptraggregate, ptrforwarditerator, ptrbackwarditerator;// recycle
}
```
Part-6 客户端应用程序输出示例 \<out\>
```txt
Using forward iterator...
passenger1 tickets required
passenger2 tickets required
passenger3 tickets required
passenger4 tickets required
passenger5 tickets required
passenger6 tickets required

Using backward iterator...
passenger6 tickets required
passenger5 tickets required
passenger4 tickets required
passenger3 tickets required
passenger2 tickets required
passenger1 tickets required
```


## Reference
> \<大话设计模式\> chapter20 p218 <br>
> https://blog.csdn.net/xiqingnian/article/details/42089611 <br>
> gof chapter5 p289 <br>

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
