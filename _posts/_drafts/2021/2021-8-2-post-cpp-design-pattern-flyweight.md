---
layout: post
title: "cpp design pattern - flyweight"
subtitle: '[structual pattern] c++设计模式之享元模式' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-08-02 21:03
lang: ch 
catalog: true 
categories: cpp 
tags:
  - Time 2021
  - cpp design pattern
---
## 享元设计模式
**[1] 基本概念** <br> 
享元模式(flyweight)：运用共享技术有效地支持大量细粒度对象。

**[2] 要素组成(核心在于享元的服务类的数据结构以及提供服务的方式)** <br>
**1) AbstractFlyweight类(抽象享元类)**：ConcreteFlyweight类的超类，为具体享元类提供基础外部接口。<br>
**2) ConcreteFlyweight类(具体享元类)**：继承自AbstractFlyweight，并实现AbstractFlyweight的接口。<br>
**3) UnsharedConcreteFlyweight类(具体非享元类)**：继承自AbstractFlyweight，但是不需要共享；享元设计模式不强制共享。<br>
**4) FlyweightFactory类(享元类型的"服务类")**：享元工厂，用来申请享元占用的资源、创建并管理享元对象。它的作用主要是保证合理地共享、管理享元对象(资源配置、外部状态的传入)。当客户端请求一个享元对象时，FlyweightFactory用于提供一个全新的实例、或者提供一个已经创建过的实例。

**[3] 应用场景(simply borrowed from gof)** <br>
**Apply flyweight pattern when [all] the followings are true:** <br>
**1)** An application uses a large number of objects. 程序本身创建、使用了大量的类型实例。<br>
**2)** Storage costs are high because of the sheer quantity of objects. 类型实例很多，"冗余"的内部状态占用了大量的storage。<br>
**3)** Most object state can be made extrinsic. 大多数状态都是外部状态。<br>
**4)** Many groups of objects may be replaced by relatively few shared objects once extrinsic state is removed. 所创建、使用的大量的实例只有非常少量的外部状态的差别。<br>
**5)** The application doesn't depend on object identity. Since flyweight objects may be shared, identity tests will return true for conceptually distinct objects. 使用享元设计模式的类型不能依赖于物体identity，否则由于享元的共享机制，导致概念上不同的两个对象a、b在运行*a.id==b.id*时返回*true*。<br>

**[4] Zen of flyweight pattern** <br>
**1) 享元模式可以减少大量类对象中维护相同的成员的内存开销(为相同的部分只存储一份实例)**在程序设计中，有时需要生成大量的细粒度的类型实例来表示数据，如果这些实例除了几个参数以外其他部分都是相同的，则可通过将不同的参数部分划归到外边，其余的相同的参数进行共享，可以大幅度减少维护类型实例所需要的资源。<br>
**2) 享元模式中类型实例的状态(component)分成两个部分：内部状态(成员)和外部状态(成员)**。<br>
内部状态：不随其他状态变化而变化and被很多其他状态所共享，由享元类本身进行维护。<br>
外部状态：随着其他状态变化而变化and几乎没有其他状态共享该种状态，由客户端代码or其他独立类型进行维护。<br>
结合案例进行分析：网站的ID属于内部状态，网站用户的Name属于外部状态。同一个网站ID可以对应多个网站的用户Name，但是一个Name只对应一种类型网站。<br> 
**3) 享元模式中的外部状态相当于"情景"，内部状态在享元类内实现，外部状态在享元内部用作api的参数，具体数值由客户端or其他类型进行维护**。内部状态在类的所有实例中不随着外部状态(场景)变化而变化，因此使用map数据结构进行管理，能够实现针对不同的外部状态所共享的内部状态，只维护一份具体实例，从而节省内存资源。

**[5] strength & weakness**
**strength** <br>
**1** 程序中涉及大量的相似对象，这些对象拥有大量相同的"内部状态"，此时使用享元模型进行对象创建、管理将节省大量的内存空间资源。<br>
**weakness** <br>
**1** 由于享元模式特殊添加的享元对象创建管理方式(额外调用享元工厂类进行享元管理、将外部状态剔除变为参数进行传递)，极有可能需要时间换空间，牺牲运行速度来换取内存资源。**2** 代码变得复杂，并且不够直接。

## 享元设计模式案例
**[1] 案例目标** <br>
构建一个"用户-网站"类型架构，实现一个用户只能拥有一种网站类型，但一个网站类型可以用于多个用户(但是不强制所有用户都共享一个网站)。这种网站共享的方式称之为"享元模式"。

**[2] 享元设计模式共享型网站结构设计案例UML类图**

<center><img src="/img/in-post/cpp_img/flyweight_1.pdf" width="100%"></center>

**[3] 享元设计模式代码示例** <br>
Part-1 用户类，享元类的服务对象，定义用户对象内容，以及提供的api(顶层)。
```cpp
class User{
    private:
        std::string name___;
        static int user_count___;
    public:
        User(std::string name):name___(name){
            user_count___++;// 更新static计数变量
        }
        std::string GetName(){
            return name___;
        }
        int GetUserCount(){
            return user_count___;
        }
};
int user_count___ = 0;// 初始化静态计数变量
```
Part-2 Flyweight享元类，定义享元对象内容，以及享元提供的api(中间层)。
```cpp
class AbstractWebsite{// no.1 AbstractFlyweight
    protected:
        virtual void ShowUser(User usr) = 0;
};
class ConcreteWebsite: public AbstractWebsite{// no.2 ConcreteFlyweight
    private:
        std::string id___;// 网站id 每个网站的id唯一
    public:
        ConcreteWebsite(std::string id):id___(id){}
        void ShowUser(User usr){
            std::cout<<"Website category: "id___
                <<"  User: "<<usr.GetName()<std::endl;
        }
};
```
Part-3 FlyweightFactory享元工厂类，用于享元的生产创建or直接取用(底层类)。
```cpp
class WebsiteFactory{
    private:
        // 维护一个id-website列表 用于网站的注册查询 
        std::map<std::string, AbstractWebsite*> flyweights___;
    public:
        WebsiteFactory(){}// ctor
        ~WebsiteFactory(){// dtor
            std::map<std::string, AbstractWebsite*>::iterator it;
            for(it=flyweights___.begin(); it!=flyweights___.end(); ++it){
                delete it->second;// 依次释放指针资源
            }
        }
        AbstractWebsite* GetWebsiteCategory(std::string id){// 网站注册查询函数
            std::map<std::string, AbstractWebsite*>::iterator it; 
            for(it=flyweights___.begin(); it!=flyweights___.end(); ++it){
                if(it->first==id){
                    return it->second;
                }
            }
            AbstractWebsite *ptr_website = new ConcreteWebsite(id);
            flyweights___.insert(
                std::pair<std::string, AbstractWebsite*>(id, ptr_website));
            return ptr_website;
        }
        int GetWebsiteScale(){// 返回网站规模大小
            return flyweights___.size(); 
        }
};
```
Part-4 客户端应用程序示例
```cpp
int main(){
    WebsiteFactory webfactory;// 创建享元服务类对象
    // 创建享元对象 并在享元服务类中进行注册
    AbstractWebsite *ptr_web1 = webfactory.GetWebsiteCategory("9527");
    ptr_web1.ShowUser(User("Melon"));// 调用享元类型的方法 进行一些操作
    AbstractWebsite *ptr_web2 = webfactory.GetWebsiteCategory("9789");
    ptr_web2.ShowUser(User("Twistfatezz"));
    AbstractWebsite *ptr_web3 = webfactory.GetWebsiteCategory("23");
    ptr_web3.ShowUser(User("Melongogogo"));
    AbstractWebsite *ptr_web4 = webfactory.GetWebsiteCategory("077");
    ptr_web4.ShowUser(User("Twistfatezzz"));
    AbstractWebsite *ptr_web5 = webfactory.GetWebsiteCategory("9527");
    User rookie("Newregister");
    ptr_web5.ShowUser(rookie);
    // 调用享元服务类的方法进行操作
    std::cout<<"Num of website created: "
        <<webfactory.GetWebsiteScale()<<std::endl;
    std::cout<<"Num of user registered: "<<rookie.GetUserCount()<<std::endl;
    return 0;
}
```
Part-5 客户端应用程序输出示例
```txt
Website category: 9527  User: Melon 
Website category: 9789  User: Twistfatezz
Website category: 23  User: Melongogogo 
Website category: 077  User: Twistfatezzz
Website category: 9527  User: NewRegister 
Num of website created: 4
Num of user registered: 5
```



## Reference
> \<大话设计模式\> chapter26 p285 <br>
> gof chapter4 p218 <br>
> https://blog.csdn.net/xiqingnian/article/details/42213851 <br>
> https://refactoringguru.cn/design-patterns/flyweight [c++ flyweight instance] <br>

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
