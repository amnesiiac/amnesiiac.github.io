---
layout: post
title: "cpp design pattern - decorator"
subtitle: '[structual pattern] c++设计模式之装饰器模式' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-07-31 08:45
lang: ch 
catalog: true 
categories: cpp 
tags:
  - Time 2021
  - cpp design pattern
---
## 装饰器设计模式
**[1] 基本概念** <br> 
装饰器模式(decorator pattern)：动态地给一些类型添加一些额外的功能职责(api)。相比通过直接给类层次结构增加成员函数api，以及给类层次结构增加子类的方式，更加灵活。

**[2] 要素组成** <br>
1) AbstractComponent：定义了一个抽象基类用于指定一种对象类型。<br>
2) ConcreteComponent：定义了一个派生类用于指定一种具体的对象。<br>
3) AbstractDecorator：定义了一个抽象基类，继承自AbstractComponent类，用于拓展Component类的功能。信息单向流动，AbstractComponent类无需知道抽象装饰器类的存在，但装饰器类需维护一个AbstractComponent类的指针。<br>
4) ConcreteDecorator：具体的装饰器类，用于将抽象装饰器类中的方法进行实现。<br>

**[3] 应用场景(simply borrowed from gof)** <br>
1) to add responsibilities to individual objects dynamically and transparently, that is, without affecting other objects. 动态的获取特定的Component对象信息，并可对获得的信息进行操作处理(Decorator's mission)，整个过程不会影响到同类别其他对象。

2) for responsibilities that can be withdrawn. 装饰器模式可用于一些可以被撤销的Component类型的信息修改上。我的理解：装饰器本身不会修改Component类型对象的内部信息，而是通过在装饰器类中包含一个指向Component类型指针的方式将\<装饰的内容\>和\<被装饰的Component类对象\>结合在一起。使用被装饰后的对象，直接操纵Decorator类指针即可；使用未装饰的对象，只需操纵Component类原指针即可。not sure...

3) when extension by subclassing is impractical. Sometimes a large number of independent extensions are possible and would produce an explosion of subclasses to support every combination. Or a class definition may be hidden or otherwise unavailable for subclassing. 当目标Component对象进行大量的相互独立的拓展处理时，如果采用类继承的方式实现，可能需要创建大量的子类，使用装饰器模式可很好解决此类问题。或者，当Component类型是隐藏的或其他不可被继承的类型，则需使用Decorator完成此任务。

**[4] 为什么装饰器设计模式相比\<为类型添加api\>以及\<为类型定义派生子类\>的方法更灵活?** <br> 
1) 装饰器模式的操作对象是特定类型的某个对象，而给特定类型添加api或者定义继承子类的方法针对的是整个类型。<br>
2) 相比后两种方法，装饰器模式使用起来更加轻量化：如需添加新的装饰功能，只要定义新装饰器类型而无需修改被装饰类型代码(信息只需单向流动)。然而添加api、定义继承子类的方式较为笨重，每次修改需求都需要recompile被装饰类型。

**[4] strength & weakness** 
**strength** <br>
**1** 无需创建继承子类型即可扩展对象的行为。**2** 通过装饰器模式可实现在运行时添加或删除对象的功能。**3** 可以通过为单一类型添加多种装饰器类以组合几种功能。**4** 满足单一职责原则，可以将实现复杂功能的大型类，拆分成较小的类型。<br>
**weakness** <br>
**1** 在封装栈中删除特定的封装器有时比较困难。**2** 实现行为不受装饰栈顺序影响的装饰器类型比较困难。**3** 各层初始化配置代码可能会很糟糕。

## decorator设计模式案例
**[1] 案例目标** <br>
给定一个带有很多'装饰作用'的成员函数api的类结构UML图如下，要求将类结构中的装饰功能剥离出来，能够实现装饰api的自由组合灵活调用。

<center><img src="/img/in-post/cpp_img/decorator_1.pdf" width="100%"></center>

**[2] decorator设计模式公司结构组织案例UML类图**

<center><img src="/img/in-post/cpp_img/decorator_3.pdf" width="100%"></center>

**[3] decorator设计模式代码示例** <br>
Part-1 decorator设计模式中AbstractDecorator代码示例
```cpp
class Person{
    protected:
        virtual void show() = 0;
};
```
Part-2 decorator设计模式中ConcreteComponent类型代码示例
```cpp
class Plebian: public Person{
    private:
        std::string name___;
    public:
        Plebian(){}
        Plebian(std::string name):name___(name){}
        void show(){
            std::cout<<"Decorated Loser: "<<name___<<std::endl;
        } 
}
```
Part-3 decorator设计模式中AbstractDecorator类型代码示例
```cpp
// AbstractDecorator: 间接继承 同时拥有Plebian和Person的成员
class Finery: public Plebian{
    protected:
        Person *ptr_component__;// 装饰器的控制句柄 使用基类指针 用于多态
    public:
        void RegistDecoratorHandler(Person *ptrcomponent){// 注册装饰器控制句柄
            this->ptr_component__ = ptrcomponent;
        }
        virtual void show(){// 装饰器调用的虚函数 -> 多态调用show函数
            if(ptrcomponent__!=nullptr){
                ptrcomponent__->show();
            }
        }
};
```
Part-4 decorator设计模式中ConcreteDecorator类型代码示例
```cpp
class Tshirts: public Finery{// ConcreteDecortor1
    public:
        void show(){
            std::cout<<"Big Tshirts ";// 操作当前类型对象(显示信息) 
            Finery::show();// 操作当前类型对象的装饰器(显示信息)
        }
};
class BaggyPants: public Finery{// ConcerteDecorator2
    public:
        void show(){
            std::cout<<"Baggy Pants ";
            Finery::show();
        }
};
class Sneakers: public Finery{// ConcerteDecorator3
    public:
        void show(){
            std::cout<<"Sneakers ";
            Finery::show();
        }
};
class Suits: public Finery{// ConcerteDecorator4
    public:
        void show(){
            std::cout<<"Suits ";
            Finery::show();
        }
};
class Tie: public Finery{// ConcerteDecorator5
    public:
        void show(){
            std::cout<<"Tie ";
            Finery::show();
        }
};
class LeatherShoes: public Finery{// ConcerteDecorator6
    public:
        void show(){
            std::cout<<"Leather Shoes ";
            Finery::show();
        }
};
```
Part-5 decorator设计模式中客户端调用代码示例
```cpp
int main(){
    Person *ptrplebian = new Plebian("Melon");
    // no.1
    std::cout<<"The first Dress-up"<<std::endl;
    // create obj
    Sneakers *ptrsneakers = new Sneakers();
    BaggyPants *ptrbaggypants = new BaggyPants();
    Tshirts *ptrtshirts = new Tshirts(); 
    // 将ptrplebian作为装饰器 添加到ptrsneakers上 返回(装饰器+被装饰对象)的整体
    ptrsneakers->RegistDecoratorHandler(ptrplebian);
    // 将ptrsneakers作为装饰器 添加到ptrbaggypants上 返回(装饰器+被装饰对象)的整体
    ptrbaggypants->RegistDecoratorHandler(ptrsneakers);
    // 将ptrbaggypants作为装饰器 添加到ptrtshirts上 返回(装饰器+被装饰对象)的整体
    ptrtshirts->RegistDecoratorHandler(ptrbaggypants);
    // show
    ptrtshirts->show();
    // no.2 ditto omitted
    std::cout<<"The second Dress-up"<<std::endl;
    LeatherShoes *ptrleathershoes = new LeatherShoes();
    Tie *ptrtie = new Tie();
    Suit *ptrsuit = new Suit();
    ptrleathershoes->RegistDecoratorHandler(ptrplebian);
    ptrtie->RegistDecoratorHandler(ptrleathershoes);
    ptrsuit->RegistDecoratorHandler(ptrtie);
    ptrsuit->show();
    // recycle
    if(ptrplebian!=nullptr){delete ptrplebian;}
    // no.1 recycle
    if(ptrsneakers!=nullptr){delete ptrsneakers;}
    if(ptrbaggypants!=nullptr){delete ptrbaggypants;}
    if(ptrbaggypants!=nullptr){delete ptrbaggypants;}
    // no.2 recycle
    if(ptrleathershoes!=nullptr){delete ptrleathershoes;}
    if(ptrtie!=nullptr){delete ptrtie;}
    if(ptrsuit!=nullptr){delete ptrsuit;}
}
```
Part-6 decorator设计模式客户端输出代码结果
```txt
The first Dress-up
Tshirts BaggyPants Sneakers Decorated Loser: Melon
The second Dress-up
Suits Tie LeatherShoes Decorated Loser: Melon
```

**[4] decorator设计模式代码结果分析思考** <br>
1) 装饰器设计模式UML结构分析：<br>
\<Plebian:Person\>的继承关系构成了装饰器模式中Component部分，\<Finery:Person\>的继承关系构成了装饰器模式中Decorator部分。装饰器需要继承自Component类，并且需要维护一个指向Component类型的指针。这个指针的作用是，保存了被装饰的对象的信息(通过指针调用被装饰对象的相关方法)。

## Reference
> \<大话设计模式\> chapter6 p44 <br>
> https://blog.csdn.net/xiqingnian/article/details/41866685 <br>
> gof chapter4 p196 <br>
> https://refactoringguru.cn/design-patterns/decorator 

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
