---
layout: post
title: "cpp design pattern - factory pattern"
subtitle: '[creational pattern] c++设计模式之工厂模式'
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-07-20 15:33
lang: ch 
catalog: true 
categories: cpp 
tags:
  - Time 2021
  - cpp design pattern
---
## 设计模式引言
**[1] 为什么需要设计模式？**设计模式之于面向对象的系统的设计和开发的作用就如数据结构之于面向过程开发的作用一般。

**[2] 面向对象系统的设计和开发追求什么？** 高内聚(cohesion)，低耦合(coupling)，用较为通俗的语言描述：同一个程序模块内部关联性越高越好，不同程序模块之间的联系越低越好。另外，c++中较高级的语法：封装、继承、多态以及设计模式中的原则都在追求这个目标的基础上进行运用。

然而，在实际编写程序中，高内聚往往伴随着高耦合：极限情况下将每个函数封装成一个类模块，则拥有最高的内聚但是导致了高耦合，低耦合往往导致低内聚(同理)，因此，我们需要通过一些设计模式来更方便的达成两者的均衡，这就是cpp-design-pattern部分知识的核心目的。

**[3] strength & weakness** <br>
**strength** <br>
**1** 可以避免产品类型和产品创建着代码间的耦合。**2** 符合单一职责原则，将产品代码和产品创建代码分开，便于维护、拓展。**3** 符合开闭原则，无需修改现有客户端代码即可即可拓展新的"产品线"。<br>
**weakness** <br>
**1** 工厂模式需要创建很多创建者子类，可能会引起代码复杂，这种情况可以使用模版工厂类的方式进行改善。

## Factory(工厂模式)
工厂模式属于一种创建型模式(creational pattern)，它提供了一种创建对象的最佳形式。换句话说，**工厂模式主要用在复杂类层次结构下的对象创建场景中(通常每个类的对象创建本身非常简单，但是层次复杂依赖众多)**，使用一个专门用于创建对象的基类来管理复杂类层次结构中对象的创建，能够使得所构建的对象更加便于管理。

工厂模式可以分成：简单工厂模式(simple factory pattern)、工厂方法模式(factory method pattern)、抽象工厂模式(abstract factory pattern)。三种方法互有优劣，互相补充。

### 1 \<简单\>工厂模式 
**[基本要素]** 工厂类、抽象产品类、具体产品类。其中，工厂类是工厂模式的核心，开放一个创建实例对象的api；抽象产品类是具体产品类的父类；具体产品类是创建的对象的类型实例。<br>
**[模式特点]** 工厂类封装了具体产生对象的函数(下图的createshoes())。<br>
**[模式缺陷]** 简单工厂模式的扩展性比较差，当有新的产品类型加入到具体产品类中时，需要重写工厂类。<br>
**[简单工厂模式UML图表]**

<center><img src="/img/in-post/cpp_img/creational_pattern_1.pdf" width="80%"></center>

**[简单工厂模式代码示例]** <br>
Part-1 产品类的类层次结构定义
```cpp
class Shoes{// 抽象产品类
    public:
        virtual ~Shoes();
        virtual void Show() = 0;
};
class NikeShoes: public Shoes{// 具体产品类-1
    public:
        void show(){
            std::cout<<"Nike Shoes, Just do it."<<std::endl;
        }
};
class AdidasShoes: public Shoes{// 具体产品类-2
    public:
        void show(){
            std::cout<<"AdidasShoes, Imposible is nothing."<<std::endl;
        }
};
class PumaShoes: public Shoes{// 具体产品类-3
    public:
        void show(){
            std::cout<<"PumaShoes, Everything is possible."<<std::endl;
        }
};
```
Part-2 工厂类: 负责上述产品类中的所有类别对象的创建(生产)
```cpp
enum ShoesType{Nike, Adidas, Puma};
class ShoesFactory{
    public:
    // 使用基类指针Shoes*操纵派生类对象new NikeShoes() 来实现运行时多态
        Shoes* createshoes(ShoesType type){
            switch(type){
                case Nike:
                    return new NikeShoes();
                    break;
                case Adidas:
                    return new AdidasShoes(); 
                    break;
                case Puma:
                    return new PumaShoes();
                    break;
                default:
                    return nullptr;
                    break;
            }
        }
};
```
Part-3 简单工厂模式的实际应用代码 
```cpp
int main(){
    ShoesFactory factory_1;
    // 案例一 利用简单工厂模式创建具体产品Nike类型对象
    Shoes *ptr_nike_shoes = factory_1.createshoes(Nike);
    if(ptr_nike_shoes!=nullptr){
        ptr_nike_shoes->show();
        delete ptr_nike_shoes;// 释放存放Nike鞋对象占用的资源
        ptr_nike_shoes=nullptr;// 重置指针
    }
    // 案例二 创建Adidas类型对象 同上略
    Shoes *ptr_adidas_shoes = factory_1.createshoes(Adidas);
    ...
}
```
**[总结简单工厂类的使用]** 简单工厂类实际上解决了：**(1)** 当具体产品类的类层次结构非常复杂时，对象创建非常混乱(对象命名繁杂、且对象创建的位置散乱)导致的难以管理的问题。通过使用简单工厂模式，所有对象经由同一工厂进行生产创建，能够统一对象创建路径，使得创建的对象便于管理。**(2)** 工厂模式的核心思想实际是：**将产品类型层次结构的定义部分和产品对象的具体生产创建相互分离**，这样使得程序逻辑更加清晰，并有一定延迟产品对象创建的作用。

### 2 工厂\<方法\>模式
**[应用场景]** 当对于特定的产品对象的生产创建需要和其他产品对象相互分开的时候，可采用工厂方法模式。<br>
**[基本要素]** 抽象工厂类、具体工厂类、抽象产品类、具体产品类。其中，抽象工厂类是工厂模式中的核心(父类)，提供了创建具体产品的api；具体工厂类是继承类，实现了创建具体产品的方法(method)；抽象产品类是产品类层次结构的父类；具体产品类是工厂模式创建对象的具体类别。<br>
**[方法特点]** 工厂方法模式为每类产品都创建一个具体的工厂类(方法)，专门用于特定类别产品创建。相比简单工程模式，工厂方法模式稍微复杂一些，它不仅考虑了将产品对象的定义和生产创建相互分离以便于管理，还为特定的产品对象提供独立的子工厂进行生产创建。<br>
**[方法缺陷]** 相比简单工厂模式，工厂方法模式在工厂构建的过程上更加复杂，对于一个新增加的产品对象，需要单独创建一个子工厂应对。<br>
**[工厂方法模式UML图表]**

<center><img src="/img/in-post/cpp_img/creational_pattern_2.pdf" width="80%"></center>

**[工厂方法模式代码示例]** <br>
Part-1 产品类的类层次结构定义 - 同简单工厂模式一致(略) <br>
Part-2 工厂方法基类及其子工厂(不同产品生产链)定义 <br>
```cpp
class ShoesFactory{
    public:
        virtual Shoes* createshoes() = 0;
        virtual ~ShoesFactory();
};
class NikeProducer: public ShoesFactory{
    public:
        Shoes* createshoes(){// 基类指针指向派生类对象实现多态
            return new NikeShoes();
        }
};
class AdidasProducer: public ShoesFactory{...};// 同上 omitted
class PumaProducer: public ShoesFactory{...};// 同上 omitted
```
Part-3 工厂方法模式实际应用代码
```cpp
int main(){
    ShoesFactory *ptr_nike_producer = new NikeProducer();// 开设一条Nike生产线
    Shoes* ptr_nike_shoes = ptr_nike_producer->createshoes();// 生产Nike球鞋
    ptr_nike_shoes->show();// 调用球鞋method、property
    // 释放相应资源
    delete ptr_nike_shoes; ptr_nike_shoes=nullptr;
    delete ptr_nike_producer; ptr_nike_producer=nullptr;
    ... // 其他品牌同理 omitted
}
```

### 3 \<抽象\>工厂模式
**[应用场景]** 抽象工厂模式相当于在工厂方法模式的基础上，针对其他的类层次结构进行处理，同时为多个**逻辑相关代码独立**的类层次结构提供产品对象。<br>
**[基本要素]** 抽象工厂模式的基本要素和工厂方法模式的要素相同，包含抽象工厂类、具体工厂类、抽象产品类(可能有多个)、具体产品类(可能有多种)。<br>
**[方法特点]** 基本特性同工厂方法模式一致，两者的区别主要在于抽象工厂模式服务的多个产品层次(抽象)，而工厂方法模式只服务一个产品层次。<br>
**[方法缺陷]** 同工厂方法模式。<br>
**[抽象工厂模式UML图表]**

<center><img src="/img/in-post/cpp_img/creational_pattern_3.pdf" width="90%"></center>

**[抽象工厂模式代码示例]** <br>
Part-1 产品类的类层次结构定义
```cpp
// 同简单工厂模式一致的部分略
// 新添加的产品类Clothe层次定义
class Clothe{
    public:
        virtual show() = 0;
        virtual ~Clothe();
};
class SportsClothe{
    public:
        void show(){
            std::cout<<"SportsClothe, better nothing."std::endl;
        }
}:
```
Part-2 抽象工厂基类及其子工厂(不同产品生产链or不同类层次产品生产链)定义
```cpp
class Factory{// 抽象工厂基类
    public:
        virtual Shoes* createshoes() = 0;// Shoes类层次api
        virtual Clothe* createclothe() = 0;// Clothe类层次api
        virtual ~Factory();
};
class NikeProducer: public Factory{// 针对nike品牌的产品对象producer
    public:
        Shoes* createshoes(){// 基类指针指向派生类对象实现多态
            return new NikeShoes();// 调用下层对象类 构建产品对象
        }
        Clothe* createclothe(){
            return new SportsClothe();// 调用下层对象类 构建产品对象 
        }
};
class AdidasProducer: public ShoesFactory{...};// 同上 omitted
class PumaProducer: public ShoesFactory{...};// 同上 omitted
```
Part-3 抽象工厂类实际应用代码
```cpp
int main(){
    Factory *ptr_nike_producer = new NikeProducer(); 
    Shoes *ptr_nike_shoes = ptr_nike_producer->createshoes();// 类层次-1 api
    Clothe *ptr_nike_clothe = ptr_nike_producer->createclothe();// 类层次-2 api
    ptr_nike_shoes->show();
    ptr_nike_clothe->show();
}
```
**[总结]** 上述三种工厂模式，从某种角度讲是一个包含的关系，简单工厂模式是工厂方法模式的一个特例，工厂方法模式是抽象工厂模式的一个特例。<br>
它们三个有共同的缺点：每次在已有类层次结构中添加新的产品类别，或者为抽象工厂模式添加新的类层次时，都需要对于工厂模型抽象部分，对象生成部分进行改动；工厂代码的扩展性不好。<br>
为了能够解决代码扩展性问题，考虑使用模版工厂类。

### 4 抽象\<模版\>工厂模式 
**[为什么将抽象工厂模式定义成模版]** 前面提到的三种工厂模式都不能解决新添加产品时，对于工厂代码的改动，一定程度上使得工厂代码难以维护。这一部分介绍的抽象模版工厂模式在抽象工厂模式的基础上，将工厂代码用c++中的模版进行维护，从而大大减轻了维护的成本。<br>
**[模版工厂类设计思想]**：如果现有的代码构建思路不能解决问题，那么应从如下两个角度对于现有代码进行重构。**(1) 平行重构**：将增加的需求部分转化为另外的模块并行添加到原有代码中去，使得总体功能满足要求。**(2) 垂直重构**：考虑将增加的需求部分和原始需求部分中的共性提炼出来，用一个类层次结构进行描述。<br>
**[基本要素]** 同抽象工厂模式相同，包含抽象模版 工厂类、具体模版工厂类、抽象产品类(可能有多个)、具体产品类(可能有多种)。<br>
**[方法特点]** 
通过使用\<模版\>技术，将抽象工厂模式中的具体产品生产工厂取缔为模版参数，增加代码的扩展性和简洁性。相比抽象工厂模式，进一步将生产产品对象的工厂类层次结构定义成模版类，从而增加了代码扩展性，在添加新的产品时，无需重构工厂类书写。<br>
**[抽象模版工厂模式UML图表]**

<center><img src="/img/in-post/cpp_img/creational_pattern_4.pdf" width="90%"></center>

**[抽象模版工厂模式代码示例]** <br>
Part-1 产品类的类层次结构定义 - 同抽象工厂类相同 omitted <br>
Part-2 抽象模版工厂基类及其子工厂模版类定义 <br>
```cpp
// 抽象模版工厂类
template<typename AbstractProduct_t> class AbstractFactory{
    private:
        // 工厂类维护产品类型指针 用于动态的创建相应产品
        AbstractProduct_t *ptr_product;
    public:
        virtual AbstractProduct_t* createproduct() = 0;
        virtual ~AbstractFactory();
};
// 具体模版工厂类 
template<typename AbstractProduct_t, typename ConcreteProduct_t> 
    class ConcreteFactory: public AbstractFactory<AbstractProduct_t>{
        private:
            ConcreteFactory(){// ctor 申请资源 创建相应对象
                ptr_product = new ConcreteProduct_t(); 
            }
        public:
            AbstractProduct_t* createproduct(){// 基类指针指向派生类对象实现多态
                return new ConcreteProduct();// 调用ctor 创建产品对象
            } 
    }
```
Part-3 抽象模版工厂类实际应用代码
```cpp
int main(){
    // no.1 获取生产创建特定产品对象的具体工厂模版类nikefactory
    ConcreteFactory<Shoes, NikeShoes> nikefactory;
    // 调用具体工厂模版类中的方法创建对象
    Shoes *ptr_nike_shoes = nikefactory.createproduct()->ptr_product;
    // 对于创建的产品对象进行使用
    ptr_nike_shoes->show();
    // 其他产品对象的处理 - ditto omitted
    // no.2 获取生产创建特定产品对象的具体工厂模版类sportclothefactory
    ConcreteFactory<Clothe, SportsClothe> sportclothefactory;
    Clothe *ptr_sport_clothe=sportclothefactory.createproduct()->ptr_product;
    ptr_sport_clothe->show();
    // 释放资源
    delete ptr_nike_shoes; ptr_nike_shoes=nullptr;
    delete ptr_sport_clothe; ptr_sport_clothe=nullptr;
}
```

### 5 单例\<模版\>工厂模式 + \<产品注册\>模版类
**[为什么给抽象模版工厂模式增加产品注册功能]** 抽象模版工厂模式虽然能够解决三种基本工厂模式中添加新产品后工厂代码难以维护的问题，但是，产品在工厂中的注册(工厂类模版实例化)以及产品在工厂中的生产(产品层次类实例化)绑定在一起，这导致每次需要生产某产品时，都需要重新在工厂中进行注册(生产一类产品，就需要实例化一次工厂类模版)。<br>
**[设计思想]** 为工厂模式增加了产品注册机制，在工厂类中使用map\<key,value\>的方式将已经注册的产品名称(key)、产品对象(value)保存；可以实现一经注册，终身使用的良好效果。<br>
**[方法特点]** 将产品注册功能抽离，使用数据结构维护注册信息。如果新产品类型已经注册，无需重新实例化模版，直接创建相应产品对象并返回控制指针即可。这种设计方式相比抽象模版工厂模式，为工厂模式增加了产品注册机制，可实现一经注册，终身生产的优势。<br>
**[单例模版工厂模式+产品注册模版类UML图表]**

<center><img src="/img/in-post/cpp_img/creational_pattern_5.pdf" width="100%"></center>

**[单例模版工厂模式+产品注册模版类的代码示例]** <br>
Part-1 产品类的类层次结构定义 - 同抽象工厂类相同 omitted <br>
Part-2 单例模版工厂的定义 
```cpp
// 单例模式的模版工厂类实现 - 模版类的单例模式的书写
template<typename AbstractProduct_t> class ProductFactory{
    private:
        // singleton: private ctor 
        ProductFactory(){}
        ~ProductFactory(){}
        // forbid external copy contruct or assignment
        ProductFactory(const ProductFactory&);
        const ProductFactory& operator=(const ProductFactory&);
        // data structure holding register info: map<产品名称,产品注册类的实例>
        // 单例工厂类和产品注册类之间通过这个数据结构进行联系
        std::map<std::string, AbstractProductRegistor<AbstractProduct_t>*> 
            productregistmap;
    public:
        // 单例模式获取单例的api - <模版类的singleton的类型仍然是一个模版类>
        static ProductFactory<AbstractProduct_t>& instance(){
            static ProductFactory<AbstractProduct_t> obj;
            return obj;
        }
        // 注册产品api
        void registerproduct(
            AbstractProductRegistor<AbstractProduct_t> *registor, 
            std::string name){
            productregistmap[name] = registor;
        }
        // 获取产品api(根据产品name获取产品对象指针) - 不负责创建产品
        AbstractProduct_t* getproduct(std::string name){
            if(productregistmap.find(name) != productregistmap.end()){// 注册过
                // 动态获取模版指针实例 并多态地创建相应产品
                return productregistmap[name]->createproduct();
            }
            else{// 未注册
                std::cout<<"No product found for: "<<name<<std::endl;
                return nullptr;
            }
        }
};
```
Part-3 抽象产品注册模版类的定义
```cpp
template<typename AbstractProduct_t> class AbstractProductRegistor{
  protected:// 允许派生类进行访问
    AnstractProduct_t *ptr_product;// 添加一个指向抽象产品类的指针
  protected:
    // 禁止外部调用ctor dtor 内部&继承子类可以调用
    AbstractProductRegistor(){};
    virtual ~AbstractProductRegistor(){}
  private:
    // default copy ctor - 禁止外部调用
    AbstractProductRegistor(const AbstractProduct_t&);
    // default assignment operator - 禁止外部调用
    const AbstractProductRegistor& operator=(const AbstractProductRegistor&);
  public:
    virtual AbstractProduct_t* createproduct() = 0;// 基类api
};
```
Part-4 具体产品注册模版类的定义(本身是一个产品类)
```cpp
template<typename AbstractProduct_t, typename ConcreteProduct_t>
  class ConcreteProductRegistor: public AbstractProductRegistor{
  public:
    // 具体产品数据成员创建函数 ctor
    ConcreteProductRegistor(){
        ptr_product = new ConcreteProduct_t();
    }
    // 产品注册构造函数 ctor(string)
    explicit ConcreteProductRegistor(std::string name){
    // 通过单例模式的模版工厂类开放的singleton获取api获取单例 并调用相应方法
    ProductFactory<AbstractProduct_t>::instance().registerproduct(this,name);
    }
    // 创建产品 - 资源申请
    AbstractProduct_t* createproduct(){
        // 创建具体对象 并返回指向具体对象的抽象基类指针(用于多态性调用)
        return new ConcreteProductRegistor();
    }
};
```
Part-5 单例模式模版工厂+模版产品注册类的应用代码实例
```cpp
int main(){
    // 调用具体产品注册创建类的ctor(string) 将产品注册到工厂中
    ConcreteProductRegistor<Shoes, NikeShoes> nikeshoes("nike");
    // 通过单例工厂模版中的单例 根据产品名字获取注册过的产品对象
    Shoes *ptr_nike_shoes=
        ProductFactory<Shoes>::instance().getproduct("nike")->ptr_product;
    // 应用产品对象
    ptr_nike_shoes->show();
    // 销毁对象 释放资源
    if(ptr_nike_shoes){
        delete ptr_nike_shoes;
        ptr_nike_shoes=nullptr;
    }
    // 注册创建其他类层次结构中的产品的例子
    ConcreteProductRegistor<Clothe, SportsClothe> sportclothe("pants");
    ...// ditto, omitted
}
```
**[总结]** 将抽象模版工厂模式替换成单例模版工厂模式+产品注册模版类进行程序组织满足程序设计中的开闭法则(Open Closed Principle)：软件实体对于扩展开发对于修改关闭。即：当软件需求改变时，能够在不修改软件实体源码的情况下，通过扩展相应模块的方式，满足新的需求，使得软件总体发展呈现一定的稳定性和延续性。<br>
关于同为creational pattern的建造者模式同工厂模式的不同的应用场景，以及特性，可以参考[这篇Builder博客](http://localhost:4000/cpp/2021/07/27/post-cpp-design-pattern-builder/#关于建造者设计模式的思考)。

## Reference
> cpp design pattern(GOF) chapter 1<br>
> https://zhuanlan.zhihu.com/p/83535678 <br>
> https://refactoringguru.cn/design-patterns/factory-method

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
