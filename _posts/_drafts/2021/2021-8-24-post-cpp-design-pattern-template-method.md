---
layout: post
title: "cpp design pattern - template method"
subtitle: '[behavioral pattern] c++设计模式之模版方法模式' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-08-24 22:44
lang: ch 
catalog: true 
categories: cpp 
tags:
  - Time 2021
  - cpp design pattern
---
## 模版方法模式
**[1] 基本概念** <br> 
模版方法模式：定义一个操作的算法骨架，将其他算法细节放到子类中实现。模版方法设计模式可以在不改变算法的框架的情况下，改变算法的特定步骤。

**[2] 要素组成** <br>
**1** AbstractTemplate抽象模版类。定义并实现模版方法框架，该框架一般是通用顶层框架，而框架内部具体操作步骤推迟到具体子类中实现。<br>
**2** ConcreteImplementation具体实现类。实现抽象模版类规定的抽象方法，用于定制顶层框架内部操作。<br>

**[3] 应用场景(simply borrowed from gof)** <br>
**(i)** to implement the invariant parts of an algorithm once and leave it up to subclasses to implement the behavior that can vary. 在抽象基类中实现"框架中的不变部分"，在派生子类中实现"框架中的定制部分"。<br>
**(ii)** when common behavior among subclasses should be factored and localized in a common class to avoid code duplication. This is a good example of "refactoring to generalize" as described by Opdyke and ohnson [OJ93]. You first identify the differences in the existing code and then separate the differences into new operations. Finally, you replace the differing code with a template method that calls one of these new operations. 这中经典设计模式可以广泛的用于代码重构。将公共部分的代码抽取放到抽象基类中实现，将不同的需要定制的部分放到子类中，这样可以有效减少"code duplication"。<br>
**(iii)** to control subclasses extensions. You can define a template method that calls "hook" operations (see Consequences) at specific points, thereby permitting extensions only at those points. 没太弄懂啥意思? 子类不是可以随意定义属于自己的接口么?基类可以限制子类实现一些基本接口，但是如何阻止子类实现自定义接口??????

**[4] strength & weakness** <br>
**strength** <br>
**(i)** 模版设计模式符合单一职责原则、开闭原则，在抽象基类中添加新的子类无需修改基类代码。
**weakness** <br> 
**(i)** 模版方法可能导致维护工作变得困难。

**[5] 与其他设计模式的关系** <br>
**(i)** 工厂方法模式时模版方法模式的一个特例。工厂方法模式可作为大型模版方法模式中的一个步骤。<br>
**(ii)** 模版方法模式是基于继承机制：通过拓展子类来定制&改变部分算法，而策略设计模式基于组合机制：为相应行为传递不同参数以调用不同的策略来改变内部实现方式。模版方式在类层次结构上进行方法的增删，是静态的；而策略模式在运行时允许调用不同的策略，是动态的。<br>


## 模版方法设计模式案例
**[1] 案例目标** <br>
构建问卷类，并将其实现为抽象模版基类，提供&实现通用框架部分接口代码，并规范&实现派生类可定制接口。派生类构建为问卷子类，根据不同的需要定制抽象基类规范的可定制接口。

**[2] 模版方法设计模式案例UML类图**

<center><img src="/img/in-post/cpp_img/template_1.pdf" width="80%"></center>

**[3] 模版方法设计模式代码示例** <br>
##### Part-1 抽象模版类定义部分代码示例 \<testpaper.h\>
```cpp
#ifndef TESTPAPER_H
#define TESTPAPER_H

#include <iostream>
#include <string>

class TestPaper{// abstractbase class
    public:// 抽象模版类定义的通用算法、程序框架
        void FirstQuestion(){
            std::cout<<"question#1? a:_ b:__ c:___ d:____"<<std::endl;
            std::cout<<"answer#1: "<<FirstAnswer()<<std::endl;
        }
        void SecondQuestion(){
            std::cout<<"question#2? a:_ b:__ c:___ d:____"<<std::endl;
            std::cout<<"answer#2: "<<SecondAnswer()<<std::endl;
        }
        void ThirdQuestion(){
            std::cout<<"question#3? a:_ b:__ c:___ d:____"<<std::endl;
            std::cout<<"answer#3: "<<ThirdAnswer()<<std::endl;
        }
    protected:// 抽象模版类规定的虚接口 - 用于供继承子类多态实现
        virtual std::string FirstAnswer(){return "";}
        virtual std::string SecondAnswer(){return "";}
        virtual std::string ThirdAnswer(){return "";}
};
class TestPaperA: public TestPaper{// subclass#1
    protected:// 多态的实现抽象基类中的框架接口
        virtual std::string FirstAnswer(){return "b"}
        virtual std::string SecondAnswer(){return "c"}
        virtual std::string ThirdAnswer(){return "a"}
};
class TestPaperB: public TestPaper{// subclass#2
    protected:// 多态的实现抽象基类中的框架接口
        virtual std::string FirstAnswer(){return "c"}
        virtual std::string SecondAnswer(){return "a"}
        virtual std::string ThirdAnswer(){return "a"}
};
#endif
```
##### Part-2 客户端程序代码示例 \<main.cpp\>
```cpp
#include "testpaper.h"
#include <iostream>
#include <cstdlib>

int main(){
    std::cout<<"The first student's paper:"<<std::endl;
    // 基类指针指向派生类对象以供多态的调用
    TestPaper *ptr_paper_a = new TestPaperA();
    ptr_paper_a->FirstQuestion();
    ptr_paper_a->SecondQuestion();
    ptr_paper_a->ThirdQuestion();
    std::cout<<"The second student's paper:"<<std::endl;
    // 基类指针指向派生类对象以供多态的调用
    TestPaper *ptr_paper_b = new TestPaperB();
    ptr_paper_b->FirstQuestion();
    ptr_paper_b->SecondQuestion();
    ptr_paper_b->ThirdQuestion();
    delete ptr_paper_a; delete ptr_paper_b;// recycle
    return 0;
}
```
##### Part-3 客户端程序代码输出示例 \<out\>
```txt
The first student's paper:
question#1? a:_ b:__ c:___ d:____
answer#1: b
question#2? a:_ b:__ c:___ d:____
answer#2: c
question#3? a:_ b:__ c:___ d:____
answer#3: a
The second student's paper:
question#1? a:_ b:__ c:___ d:____
answer#1: c
question#2? a:_ b:__ c:___ d:____
answer#2: a
question#3? a:_ b:__ c:___ d:____
answer#3: a
```

## Reference
> \<大话设计模式\> chapter10 p108 <br>
> https://blog.csdn.net/xiqingnian/article/details/41989447 <br>
> gof chapter5 p360 <br>
> https://refactoringguru.cn/design-patterns/template-method
 
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
