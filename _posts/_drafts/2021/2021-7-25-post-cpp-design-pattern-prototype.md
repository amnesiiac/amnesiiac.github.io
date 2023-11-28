---
layout: post
title: "cpp design pattern - prototype"
subtitle: '[creational pattern] c++设计模式之原型模式'
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-07-25 09:43
lang: ch 
catalog: true 
categories: cpp 
tags:
  - Time 2021
  - cpp design pattern
---
## 原型模式
**[1] 什么是原型模式** <br>
原型模式就是根据原型对象克隆创建普通对象，并且不需要关注原型对象提供的预设内部细节。普通对象的创建可以通过浅复制or深复制两种方式完成，方法的选择取决于类型拷贝构造函数实现的方式。

**[2] 什么情况下需要使用原型模式** <br> 
(a) 当我们不希望建立和原始类层次结构平行的(UML图角度)工厂类层次结构进行类对象实例生产时。<br>
(b) 当我们创建一个类的实例，但是又不关心该类型实例是\<如何被创建的\>、\<由什么构成的\>、\<如何对其进行描述\>时。**原型模式中的clone()函数，直接复制而不是重新创建类型实例**。<br>
(c) 当需要实例化的类的具体类型需要在运行时刻动态加载时(runtime dynamic loading)。<br>
**(d)** 当一个类的实例只能有几种不同的状态组合中的一种，那么有两种方式可供选择：(1) 对于每个特性不同的实例，依次进行手动实例化。(2) 抓住主要状态组合创建一定数量的原型并通过clone的方式创建这个类的实例。特定的场景下，后者的执行效率将远远高于前者。**原型模式进行类层次结构设计的原因：特定场景下，应用拷贝构造函数效率远高于调用构造函数进行对象创建时，使用原型模式有较大的优势；另外，原型模式构建的类层次结构中，需要有构建原型实例(preset)、在原型实例的基础上进行对象克隆(clone)创建的概念体现。**<br>

**[3] 原型模式实现方式的初步窥探** <br>
原型类首先需要创建带有preset的类型实例，需要通过原型类的default ctor来完成。然后，完成clone函数用于克隆原型生成新对象。clone函数主要调用copy ctor用于新对象创建。最后，类型还将提供新对象属性修改api，借此实现新对象的个性化定制。

**[4] strength & weakness** <br>
**strength** <br>
**1** 你可以克隆对象，而无需与这些对象所属的具体类型相耦合。**2** 可以更加方便直观的生成复杂对象。**3** 原型模式提供了一种处理复杂对象不同配置新方式(不同于类层次继承方式)。<br>
**weakness** <br>
**1** 在使用原型模式克隆较为复杂的对象时(e.g.循环引用的对象)，可能会非常麻烦。

## 以简历类为例子对于原型设计模式进行理解 
创建一个简历类用于简历的生成。要求：每份生成的简历必须有姓名属性，可以设置姓名属性、年龄属性、工作经历属性。最终需要生成三个独立的简历对象。

### 普通模式完成简历类(非原型模式)
**[1] 普通非原型模式UML类图** <br>
普通非原型模式，每次需要创建简历时，都需要调用构造函数。

<center><img src="/img/in-post/cpp_img/prototype_1.pdf" width="100%"></center>

**[2] 普通非原型模式代码示例**
```cpp
class Resume{
    private:
        // 基本成员 
        std::string name__;
        // 可设置成员
        std::string gender__;
        std::string age__;
        std::string stage__;
        std::string company__;
    public:
        Resume(std::string name):name__(name){}// ctor
        void SetPersonalInfo(string gender, string age){// set
            gender__ = gender;
            age__ = age;
        } 
        void SetWorkExperience(string stage, string company){// set
            stage__ = stage;
            company__ = company;
        }
        void display(){// show info
            std::cout<<"name, gender, age: "<<
                name__<<" "<<gender__<<" "<<age__<<std::endl;
            std::cout<<"stage, workexperience: "<<
                stage__<<" "<<company__<<std::endl;
        }
}:
```
**[3] 客户端调用代码创建了三份melon的简历：**
```cpp
int main(){
    Resume a = new Resume("melon");// no.1 call ctor construct resume obj 
    a.SetPersonalInfo("male", "26");
    a.SetWorkExperience("2017-09", "sjtu");
    Resume b = new Resume("melon");// no.2 call ctor construct resume obj 
    b.SetPersonalInfo("male", "26");
    b.SetWorkExperience("2017-09", "sjtu");
    Resume c = new Resume("melon");// no.3 call ctor construct resume obj 
    c.SetPersonalInfo("male", "26");
    c.SetWorkExperience("2017-09", "sjtu");
    a.display(); b.display(); c.display(); // show info
}
```
**[4] 程序运行结果分析** <br>
客户端代码想要创建3个melon简历，需要调用三次简历类构造函数。<br>
在某些应用场景下，大量重复执行构造函数开销很大。进一步地，如果所创建的对象大部分属性值都相同，只有几个属性数值不同，那么重复频繁的调用构造函数往往得不偿失(我们只想为一个新对象修改部分属性)。<br>
如果，我们能够从类型实例特征出发，将类型实例按照它们具有的属性特征进行聚类(cluster)，并将聚类中心作为模版，使用克隆(clone)的方式创建对象，避免了调用类型构造函数，则能够很大程度上提升类型的运行时性能。

**[5] 一种too young too simple的做法(浅拷贝，对于指针、引用类型数据成员执行浅拷贝可能不合乎逻辑)**
直接调用c++默认赋值运算符，完成3个简历对象的创建，基本客户端程序如下：
```cpp
int main(){
    Resume a = new Resume("melon");// no.1 call ctor construct resume obj 
    a.SetPersonalInfo("male", "26");
    a.SetWorkExperience("2017-09", "sjtu");
    Resume b = a;// default op=
    Resume c = a;// default op=
    a.display(); b.display(); c.display(); // show info
}
```
上述代码的弊端是：虽然能够很快的完成3份简历的创建，但是根据默认赋值构造函数的定义可知，参数的传递和参数的返回都是通过引用完成的，因此，这样创建的3份简历实际上完全共用同一块内存，没有创建新的对象而是引用换绑定。修改一个内容，其他两个也会改变。这种方法创建的3份简历不能满足我们的独立性要求。关于引用、指针之间的赋值详见[博客](http://localhost:4000/cpp/2021/04/20/post-cpp-reference/#引用和指针的区别)中的内容。

### 普通原型模式完成简历类
原型模式即：使用少量的原型实例指定创建对象的种类，并且通过克隆(clone)而不是构造(construct)的方式创建原型实例。

**[1] 普通原型模式UML类图** <br>
普通的原型模式为简历类添加了克隆函数，克隆函数调用简历类的拷贝构造函数构建目标对象。

<center><img src="/img/in-post/cpp_img/prototype_2.pdf" width="100%"></center>

**[2]** 普通原型模式设计简历类。
```cpp
class Resume{
    private:
        // 原型实例preset 
        std::string name__;
        // 用于克隆原型类进行个性化设置的成员
        std::string gender__;
        std::string age__;
        std::string stage__;
        std::string company__;
    public:
        // no.1 ctor
        // 构造函数：用来构建原型preset 这个原型类为name参数提供了preset
        Resume(std::string name):name__(name){}
        // no.2 copy ctor: can be deep-copy or shallow-copy
        // 拷贝构造函数专门用于被clone调用 用于创建带有preset的克隆对象 
        Resume(const Resume& resume):name__(resume.name__){}
        // no.3 clone function: 调用拷贝构造函数创建克隆实例(以原型preset为基础)
        Resume* clone(){
            return new Resume(*this);
        }
        void SetPersonalInfo(string gender, string age){// set
            gender__ = gender;
            age__ = age;
        } 
        void SetWorkExperience(string stage, string company){// set
            stage__ = stage;
            company__ = company;
        }
        void display(){// show info
            std::cout<<"name, gender, age: "<<
                name__<<" "<<gender__<<" "<<age__<<std::endl;
            std::cout<<"stage, workexperience: "<<
                stage__<<" "<<company__<<std::endl;
        }
};
```
**[3]** 客户端调用代码创建了三份melon的简历：
```cpp
int main(){
    Resume a = new Resume("melon");// no.1 call ctor construct resume obj 
    a.SetPersonalInfo("male", "26");
    a.SetWorkExperience("2017-09", "sjtu");
    Resume *ptr_b = a.clone();// b is constructed from a
    ptr_b->SetPersonalInfo("female", "23");
    ptr_b->SetWorkExperience("2020-05", "sjtu");
    Resume *ptr_c = a.clone();// c is constructed from a
    ptr_c->SetPersonalInfo("female", "26");
    ptr_c->SetWorkExperience("2020-03", "sjtu");
    a.display(); ptr_b->display(); ptr_c->display(); // show info
}
```
**[4] 关于原型模式代码的分析** 原型模式的关键是原型类中实现了一个用于克隆(clone)对象的接口，可以用于借助原型类实例创建其他克隆体。<br>
另外，在c++中，原型模式的克隆接口可以通过直接显式定义拷贝构造函数来实现。根据定义的拷贝构造函数实现方式不同，分为浅拷贝(shallow-copy)方式和深拷贝(deep-copy)方式。本部分内容实用的简历类由于数据结构中没有涉及指针or引用类型，因此深拷贝和浅拷贝的copy ctor的定义形式一致，如果数据成员中含有指针、引用则需要使用new expression申请资源创建对象再将指针or引用类型绑定到新对象上面。<br>

### 拓展原型模式完成简历类
**[基本框架]** <br>
原型拓展模式是在原型模式的基础上，将抽象原型类设计成虚基类，将具体原型类设计成派生类。每个具体原型类都代表了简历类层次结构中的一种较为通用的简历模版。通过创建原型类并返回虚基类类型的指针，通过基类的指针实现方法调用时的多态。在具体原型类的定义中实现clone()方法，clone()调用拷贝构造函数完成克隆对象的创建。<br>
**[功能剥离]** <br>
另外，将普通的原型模式中关于工作经验的部分从原型类实现中抽离出来构成WorkExperience类，并提供了信息注册set和get信息获取的api，方便简历类进行调用。为了能够方便的在简历类中操作WordExperience类的api，为简历类添加了一个指向WorkExperience类型的指针(控制媒介)。<br>
**[实现注意事项]** <br> 
(1) 原型模式在c++中的实现是以拷贝构造函数为基础实现，因此根据copy ctor的实现方式不同，分别以深复制和浅复制的方式完成。(2) 拓展的原型模式将工作经验信息从原型类中抽离出来形成WorkExperience辅助类，在调用拓展原型模式的clone()时，需要分别为原型类(Concreteresume1)以及辅助类(WorkExperience)构建拷贝构造函数，缺一不可。

**[1] 拓展的原型模式UML类图** <br>
拓展的原型模式相比普通的原型模式，主要的改动有2点：(1) 考虑了更加复杂的情况，即一个简历类中可能有多种原型，因此将原型类层次实现为抽象虚基类+派生具体实例类的层次结构。(2) 对于简历类中的工作经验相关数据，单独抽出来用WorkExperience类进行维护，简历类通过维护一个指向WorkExperience类的指针数据进行信息通信。

<center><img src="/img/in-post/cpp_img/prototype_3.pdf" width="100%"></center>

其中，抽象原型类作为基类，不进行具体原型实例的构造和克隆。具体原型类作为派生类，用于对不同的原型分别进行构建。客户端代码通过抽象原型类型指针动态绑定到具体原型类创建的原型上，并通过基类的指针调用具体原型实例的克隆方法克隆出新的具体实例类型对象。

**[2] 拓展的原型模式结构代码示例** <br>
Part-1 WorkExperience工作经验类(用于信息获取和注册)
```cpp
class WorkExperience{// 工作经验类 用于信息获取和注册
    private:
        std::string stage__;
        std::string company__;
    public:
        // no.1 default ctor: 供原型类ctor创建preset调用
        WorkExperience(){}
        // no.2 copy ctor: 用于供原型类copy ctor克隆preset调用
        WorkExperience(const WorkExperience& experience){
            stage__ = experience.stage__;// set *this member
            company__ = experience.company__;// set *this member
        }
        // 信息注册api
        void SetWorkStage(std::string stage){
            stage__ = stage; 
        }
        void SetWorkCompany(std::string company){
            company__ = company;
        }
        // 信息获取api
        std::string GetWorkStage(){
            return stage__;
        }
        std::string GetWorkCompany(){
            return company__;
        }
}:
```
Part-2 抽象简历基类(提供简历类层次结构的基本框架)
```cpp
class AbstractResume{// 
    protected:// 限定只有继承类可以访问
        // 基本成员 
        std::string name_;
        // 可设置成员
        std::string gender_;
        std::string age_;
        // 下面两个关于工作经历的变量交给WorkExperience类进行维护
        // 在必要时 通过WorkExperience类的api进行信息注册&信息获取
        // std::string stage_;
        // std::string company_;
    protected:// 限定只有继承类可以访问
        // pure virtual function: 不能在抽象基类中实例化 必须在继承类中实例化
        virtual SetPersonalInfo(std::string gender, std::string age) = 0;
        virtual SetWorkExperience(std::string stage, std::string company) = 0;
        virtual AbstractResume* clone() = 0
        virtual void display() = 0; 
};
```
Part-3 具体简历派生类(提供简历类层次结构的具体每种原型的实现，并完成原型->新对象的创建)
```cpp
class ConcreteResume1: public AbstractResume{
    private:
        // 用于和WorkExperience类进行信息传递: 注册信息 & 获取信息
        // 由于涉及到指针类型的数据成员 因此需要使用深复制 而不是浅复制
        WorkExperience *ptr_work_experience__;
    public:
        // no.1 ctor: 创建原型实例所需preset: (名字,工作经历)
        ConcreteResume1(std::string name):name_(name){
            ptr_work_experience__ = new WorkExperience();
        }
        // no.2 copy ctor: 按照原型实例preset为基础创建克隆实例
        ConcreteResume1(const ConcreteResume1 &resume):
            name_(resume.name_){
            // WorkExperience copy ctor: 原型preset中的指针成员 使用深复制
            ptr_work_experience__ = 
                new WorkExperience(*resume.ptr_work_experience__);
        }
        ~ConcreteResume1(){// dtor
            if(ptr_work_experience__!=nullptr){// 销毁指针管理对象 释放资源
                delete ptr_work_experience__;
                ptr_work_experience__ = nullptr;// 重置指针
            }
        };
        ConcreteResume1* clone(){
            // copy ctor: 调用拷贝构造函数创建原型的克隆对象(带有preset)
            return new ConcreteResume1(*this);
        }
        void SetPersonalInfo(std::string gender, std::string age){// set
            gender_ = gender;  
            age_ = age;
        }
        void SetWorkExperience(std::string stage, std::string company){// set
            ptr_work_experience__->SetWorkStage(stage);// 注册stage
            ptr_work_experience__->SetWorkCompany(company);// 注册company
        }
        void display(){// show info: 信息获取stage 信息获取company
            std::cout<<"name, gender, age: "<<
                name__<<" "<<gender__<<" "<<age__<<std::endl;
            std::cout<<"stage, workexperience: "<<
                ptr_work_experience__->GetWorkStage()<<" "<<
                ptr_work_experience__->GetWorkCompany()<<std::endl;
        }
};
class ConcreteResume2: public AbstractResume{...};// ditto omitted
class ConcreteResume3: public AbstractResume{...};// ditto omitted
```
Part-4 客户端(client)应用简历类层次结构代码示例(创建原型Concreteresume1，并根据原型克隆对象)
```cpp
int main(){
    // 直接创建原型a
    ConcreteResume1 *ptr_resume_a = new ConcreteResume1("melon");// ctor
    ptr_resume->SetPersonalInfo("male", "23");// set
    ptr_resume->SetWorkExperience("2020-05", "sjtu");// set
    // 克隆构造b
    ConcreteResume1 *ptr_resume_b = ptr_resume_a->clone();// clone a for b
    ptr_resume_b->SetPersonalInfo("female", "25");// set
    ptr_resume_b->SetWorkExperience("2017-09", "sjtu");// set
    // 克隆构造c
    ConcreteResume1 *ptr_resume_c = ptr_resume_a->clone();// clone a for b
    ptr_resume_c->SetPersonalInfo("male", "24");// set
    ptr_resume_c->SetWorkExperience("2015-09", "sjtu");// set
    // 销毁对象 释放资源
    delete ptr_resume_a, delete ptr_resume_b, delete ptr_resume_c;
}
```

## 关于原型设计模式的思考
**[1] 为什么克隆的方式能够实现更高的效率?是不是拷贝构造函数的执行效率要高于构造函数的执行效率?** <br>
这个问题目前没有找到答案，关于ctor和copy ctor的效率比较的问题答案没有一个固定套路。原型设计模式的意义应该不在于改变对象的创建方式带来的执行效率的提升，而是在于原型提供了一些模版化的preset，能够更加方便的让我们构建并组织对象。

**[2] 原型模式在c++中实际上相当于调用拷贝构造函数进行对象构造，原型设计模式在什么场景下有较高的应用价值?** <br>
参考[stackoverflow](https://stackoverflow.com/questions/13887704/whats-the-point-of-the-prototype-design-pattern)中Mark Pauley的回答：
> The Prototype pattern is a creation pattern based on cloning a pre-configured object(原型设计模式主要用于克隆一个**带有预设**的obj). The idea is that you pick an object that is configured for either the default or in the ballpark(可变通领域) of some specific use case and then you clone this object and configure to your exact needs. 

> The pattern is useful to remove a bunch of boilerplate code, when the configuration required would be onerous. I think of Prototypes as a preset object(原型就是一系列带有预设的obj), where you save a bunch of state as a new starting point(也可以看作是**一系列创建对象的起始配置**).


## Reference
> \<design patterns: gof\> chapter3 p134 <br>
> \<大话设计模式\> chapter9 p95 <br>
> https://refactoringguru.cn/design-patterns/prototype 

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
