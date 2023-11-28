---
layout: post
title: "cpp design pattern - facade"
subtitle: '[structual pattern] c++设计模式之外观模式' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-08-02 14:43
lang: ch 
catalog: true 
categories: cpp 
tags:
  - Time 2021
  - cpp design pattern
---
## 外观设计模式
**[1] 基本概念** <br> 
外观模式：为子系统中的一组接口提供一个一致的界面，此模式将子系统的这组接口封装成一个高层接口，通过这个高层接口能够更加方便的使用子系统。

**[2] 要素组成** <br>
1) 客户端程序(client)：通过调用facade外观设计模式封装类，来使用底层子系统api。<br>
2) 外观设计模式封装类(facade)：维护子系统中的类层次结构的指针，将子系统方法封装，并向client暴露易用接口。<br>
3) 底层子系统类型层次结构(subsystem)：底层功能具体实现的类型，内部应用了多种设计模式导致类型之间的耦合性非常低，包含很多"很小但是功能明确api各异"的类型代码，难以直接调用。

**[3] 应用场景** <br>
➤ **simply borrowed from 大话程序设计** <br>
1) 在 \<设计初期\> 阶段，将整体业务分层。如可以将应用程序分成数据访问层、业务逻辑层、表示层，其中每两层之间使用facade模式进行"粘合"，这种架构可以大大降低复杂的应用程序之间的耦合性。**在大型程序or类库的编写中，将应用程序分层，并在层之间使用facade模式进行链接，可以大大降低框架的耦合度，使之逻辑结构清晰易于维护(e.g. tensorflow架构)。**<br>
2) 在 \<开发阶段\>，子系统往往会因为不断重构演化而越来越繁杂，内部套用了很多设计模式也会产生很多相互耦合度很低，内部聚合度很高的很小的类型，但是通常也会是的这些"很小的类型"难以被使用。为它们增加facade类，可以更加方便的调用它们，并且不会改变它们固有的特性和优势。**use facade to simplify.**<br>
3) 在 \<维护一个遗留的大型的系统\> 时，系统本身非常难以维护和拓展，但是它包含了很多核心的功能，在针对新需求进行开发时，使用facade设计模式是非常合适的。**shit wrapped by candy skin.**<br>

➤ **simply borrowed from gof** <br>
1) **you want to provide a simple interface to a complex subsystem.** Subsystems often get more complex as they evolve. Most patterns, when applied, result in more and smaller classes. This makes the subsystem more reusable and easier to customize, but it also becomes harder to use for clients that don't need to customize it. A facade can provide a simple default view of the subsystem that is good enough for most clients. **Only clients needing more customizability will need to look beyond the facade.** <br>
<center><img src="/img/in-post/cpp_img/facade_1.pdf" width="60%"></center>
2) there are many dependencies between clients and the implementation classes of an abstraction. **Introduce a facade to decouple the subsystem from clients and other subsystems, thereby promoting subsystem independence and portability.** <br>
3) you want to layer your subsystems. **Use a facade to define an entry point to each subsystem level.** If subsystems are dependent, then you **can simplify the dependencies between them by making them communicate with each other solely through their facades**. 在分层的subsystem系统之间使用facade可以简化它们之间的依赖表示。 <br>
<center><img src="/img/in-post/cpp_img/facade_2.pdf" width="60%"></center>

**[4] strength & weakness** <br>
**strength** <br>
**1** 通过使用外观模式，可以让客户端代码更加整洁，避免调用复杂子系统api。<br>
**weakness** <br>
**1** facade类型对象可能成为程序中所有类型都耦合的一种类型。子系统中每进行一次修改，都需要将耦合的facade类型重新进行维护。


## facade设计模式案例
**[1] 案例目标** <br>
现有系统由客户端代码和子系统构成，子系统由很多类型层次结构组成，客户端需要调用子系统的api。现在需要设计一种外观类，能够将子系统中的api进行封装，并对客户端程序暴露合理接口，使得客户端调用更加方便整洁易用。

**[2] facade设计模式公司结构组织案例UML类图**

<center><img src="/img/in-post/cpp_img/facade_3.pdf" width="100%"></center>

**[3] facade设计模式代码示例** <br>
Part-1 facade设计模式子系统类代码示例
```cpp
class SubSystem1{
    public:
        void Method1(){// 子系统类中的方法 子系统api繁杂 直接调用较为混乱
            std::cout<<"subsystem method-1"<<std::endl;
        }
};
class SubSystem2{
    public:
        void Method2(){
            std::cout<<"subsystem method-2"<<std::endl;
        }
};
class SubSystem3{
    public:
        void Method3(){
            std::cout<<"subsystem method-3"<<std::endl;
        }
};
class SubSystem4{
    public:
        void Method4(){
            std::cout<<"subsystem method-4"<<std::endl;
        }
};
```
Part-2 facade设计模式facade外观类代码示例
```cpp
class Facade{
    private:
        SubSystem1 *ptr1___;
        SubSystem2 *ptr2___;
        SubSystem3 *ptr3___;
        SubSystem4 *ptr4___;
    public:
        Facade(){// 负责初始化子系统对象
            ptr1___ = new SubSystem1();
            ptr2___ = new SubSystem2();
            ptr3___ = new SubSystem3();
            ptr4___ = new SubSystem4();
        }
        ~Facade(){
            delete ptr1___;
            delete ptr2___;
            delete ptr3___;
            delete ptr4___;
        }
        void MethodA(){// facade外观类对于subsystem子系统中的方法进行封装
            std::cout<<"method group A"<<std::endl;
            ptr1___->Method1();
            ptr2___->Method2();
            ptr4___->Method4();
            std::cout<<std::endl;
        }
        void MethodB(){// facade外观类对于subsystem子系统中的方法进行封装
            std::cout<<"method group B"<<std::endl;
            ptr2___->Method2();
            ptr3___->Method3();
            std::cout<<std::endl;
        }
};
```
Part-3 facade设计模式客户端程序代码示例
```cpp
int main(){
    Facade *ptrfacade = new Facade();// 构建facade类
    ptrfacade->MethodA();// 客户端请求 调用facade类封装好的A方法
    ptrfacade->MethodB();
    delete ptrfacade;
}
```
Part-4 facade设计模式应用程序输出示例
```txt
method group A
subsystem method-1
subsystem method-2
subsystem method-4
method group B
subsystem method-2
subsystem method-3
```

## Reference
> \<大话设计模式\> chapter12 p103 <br>
> gof chapter4 p208 <br>
> https://blog.csdn.net/xiqingnian/article/details/41990789 <br>
> https://refactoringguru.cn/design-patterns/facade

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
