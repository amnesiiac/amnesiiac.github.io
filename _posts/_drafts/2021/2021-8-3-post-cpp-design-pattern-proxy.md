---
layout: post
title: "cpp design pattern - proxy"
subtitle: '[structual pattern] c++设计模式之代理模式' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-08-03 15:47
lang: ch 
catalog: true 
categories: cpp 
tags:
  - Time 2021
  - cpp design pattern
---
## 代理模式
**[1] 基本概念** <br> 
代理模式：为其他对象提供一种代理以控制or代替对这个对象的访问，也可作为一种额外附加的对象访问。

**[2] 要素组成** <br>
**1) Guest类型**：代理设计模式服务的对象类型。<br>
**2) AbstractSubject类型**：代理设计模式中的抽象基类，规范ConcreteSubject和Proxy的公共接口api。<br>
**3) ConcreteSubject类型**：继承自AbstractSubject，并为其公共api提供具体实现。<br>
**4) Proxy类型**：继承自AbstractSubject，通过调用ConcreteSubject类的api接口为Guest类提供服务。<br>
代理模式中有3层服务关系：*ConcreteSubject-\>Guest, Proxy-\>ConceteSubject, Proxy-\>Guest*，其中前两层关系通过维护一个"被服务类型"的指针来完成；最后一层关系通过间接调用完成。(-\>表示服务)

**[3] 应用场景(simply borrowed from gof)** <br>
**1) A remote proxy** provides a local representative for an object in a different address space. NEXTSTEP [Add94] uses the class NXProxy for this purpose. Also called the "Ambassador." 远程代理 - 为一个对象提供local representative以掩盖这个对象来自多个地址空间的事实。<br>

**2) A virtual proxy** creates expensive objects on demand. The ImageProxy described in the Motivation is an example of such a proxy. 虚拟代理 - 为一些创建开销很大的对象提供虚拟代理，代理在生成虚拟对象的同时也存放真实对象相关信息。HTML中的图片lazyload行为类似。<br>

**3) A protection proxy** controls access to the original object. Protection proxies are useful when objects should have different access rights. For example, KernelProxies in the Choices operating system [CIRM93] provide protected access to operating system objects. 安全代理 - 为对象的访问不同级别构建不同的代理，通过安全代理的访问需受到相应限制。<br>

**4) A smart reference** is a replacement for a bare pointer that performs additional actions when an object is accessed. Typical uses include: 代理类型可以作为原始对象的一种"智能引用"，功能包括：<br>
4.1) counting the number of references to the real object so that it can be freed automatically when there are no more references (also called smart pointers [Ede92]) 代理可以负责计算实际对象被引用的次数实现对象的0引用自动释放。<br>
4.2) loading a persistent object into memory when it's first referenced. 在第一次引用一个持久对象时负责将它载入内存。<br>
4.3) checking that the real object is locked before it's accessed to ensure that no other object can change it. 可以用于进行访问预先测试(使用代理创建的对象进行"lock test")，以确保实际对象处于锁定状态(其他的对象不可以修改实际对象的状态)。

**[4] strength & weakness** <br>
**strength** <br>
**1** 可以在客户端毫无察觉的情况下控制对象。**2** 如果客户端对服务对象的声明周期没有要求，可以通过proxy对对象的声明周期进行管理。**3** 即使服务对象没有准备好or不存在，proxy类仍能正常工作。**4** 满足程序设计开闭原则，可以在不修改服务端、客户端进行修改的情况下创建新的代理。<br>

**weakness** <br>
**1** 使用代理设计模式的代码可能会变得复杂，由于新建了很多代理类。**2** 服务器响应可能由于使用了大量代理而产生延时。


## 代理设计模式案例
**[1] 案例目标** <br>
构建一个送礼物的模型，为收礼物的类型单独维护一个类型，并且实际送礼物的类型需要通过"代理类"将礼物信息传达。即使用代理设计模型构建一个"间接送礼物模型"。

**[2] 代理设计模式案例UML类图**

<center><img src="/img/in-post/cpp_img/proxy_1.pdf" width="100%"></center>

**[3] 代理设计模式代码示例** <br>
Part-1 代理设计模式中的接受代理服务的类型(Guest类型) - 外部状态参数类
```cpp
class Pursuing{// 代理设计模式服务的对象类型
    private:
        std::string name___;
    public:
        void SetName(std::string name):name___(name){}
        std::string GetName(){
            return name___;
        }
};
```
Part-2 代理设计模式中AbstractSubject抽象基类 
```cpp
class GiftSender{// 代理和被代理类型的公共基类
    public:
    // 规范了ConcreteSubject和Proxy的共用api - ConcreteSubject和Proxy的应用场景重合
        virtual void DollSender() = 0;
        virtual void FlowerSender() = 0;
        virtual void ChocolcateSender() = 0;
};
```
Part-3 代理设计模式中ConcreteSubject派生类
```cpp
class Pursuer: public GiftSender{
    private:
        Pursuing *ptr_school_girl___;// 维护代理模式服务的类型指针
    public:
        // 接受并注册外部状态参数
        Pursuer(Pursing *mm):ptr_school_girl___(mm){}
        // 被代理类实现3种公共api - 需要提供实质性操作
        void DollSender(){
            std::cout<<"Doll for you, dear "
                <<ptr_school_girl___.GetName()<<std::endl;
        }
        void FlowerSender(){
            std::cout<<"Flower for you, dear "
                <<ptr_school_girl___.GetName()<<std::endl;
        }
        void ChocolcateSender(){
            std::cout<<"Chocolate for you, dear "
                <<ptr_school_girl___.GetName()<<std::endl;
        }
};
```
Part-4 代理设计模式中Proxy代理类
```cpp
class Proxy: public GiftSender{
    private:
        Pursuer *ptr_school_boy___;// 维护被代理类型指针(供调用)
    public:
        Proxy(Pursuing *mm){// 接受"外部状态参数" 并调用实际工作类进行参数注册
            ptr_school_boy___ = new Pursuer(mm);
        }
        // 代理类实现的公共api - 调用被代理类型的api实现
        void DollSender(){// proxy模式中 调用被代理的api完成实质任务
            ptr_school_boy___->DollSender();
        }
        void FlowerSender(){
            ptr_school_boy___->FlowerSender();
        }
        void ChocolcateSender(){
            ptr_school_boy___->ChocolcateSender();
        }
};
```
Part-5 代理设计模式客户端代码示例
```cpp
int main(){
    Pursuing *ptrmm = new Pursuing();
    ptrmm->SetName("xxx");
    std::cout<<"1) gift sending directly without proxy:"<<std::endl;
    Pursuer *ptrgg = new Pursuer(ptrmm);
    ptrgg->DollSender();
    std::cout<<"2) gift sending through proxy:"<<std::endl;
    Proxy *ptrfakegg = new Proxy(ptrmm);
    ptrfakegg->DollSender();
    ptrfakegg->FlowerSender();
    ptrfakegg->ChocolcateSender();
    delete ptrmm; delete ptrgg; delete ptrfakegg;
}
```
Part-6 代理设计模式客户端代码输出示例
```txt
1) gift sending directly without proxy:
Flower for you, dear xxx 
2) gift sending through proxy:
Doll for you, dear xxx 
Flower for you, dear xxx 
Chocolate for you, dear xxx 
```

## Reference
> \<大话设计模式\> chapter7 p75<br>
> gof chapter4 p233 <br>
> https://blog.csdn.net/xiqingnian/article/details/41970165 <br>
> https://refactoringguru.cn/design-patterns/proxy 

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
