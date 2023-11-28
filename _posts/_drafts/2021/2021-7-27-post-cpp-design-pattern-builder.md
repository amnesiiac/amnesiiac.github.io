---
layout: post
title: "cpp design pattern - builder"
subtitle: '[creational pattern] c++设计模式之建造者模式'
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-07-27 14:05
lang: ch 
catalog: true 
categories: cpp 
tags:
  - Time 2021
  - cpp design pattern
---
## 建造者设计模式
**[1] 基本概念** <br> 建造者设计模式(builder)：将一个复杂的对象的构建过程和该对象的构建细节相互分离，使得相同的构建过程可以创建出不同的细节表示。<br>
在下面的代码中，构建过程由director类控制，每个模块的构建细节由builder类控制。对于相同的构建步骤，细节不同的产品，只需要修改director类内api的函数体即可实现。

**[2] 要素组成** <br>
1 AbstractBuilder: 建造者类型的虚基类，为创建Product类型对象提供并规范抽象建造接口。<br>
2 ConcreteBuilder: 建造者类型的派生类，实现虚基类中规范的建造接口。<br>
3 Product: 建造者建造对象的目标类型。<br>
4 Director: 建造指挥者类，每个指挥者对象定义了一套调用建造者类的完整工序流程。<br>

**[3] 应用场景** <br>
建造者模式主要用于创建一些复杂的、模块化的对象，这些对象内部模块构造的顺序通常遵循一定的组建规则，但是每个模块内部的细节表示又有复杂的变化。

**[4] strength & weakness** <br>
**strength** <br>
**1** 在对象构建时，将对象模块化的构建和每个模块内部的细节描绘相互分离，可以对于模块组件方式和模块内部细节表示分别进行清晰的实现。**2** 在特定场景下，还可以隐藏建造对象的模块组件流程，实现一定程度的封装性。**3** 可以方便的定义目标对象特征：只需要为新的目标对象创建一个新的建造者模型对象即可。<br>
**weakness** <br>
**1** 该模式会新增很多类，代码复杂度会明显增加。


## 结合代码案例对建造者设计模式进行理解
### 建造者模式(较简单的例子)
**[1] 建造者设计模式UML类图**

<center><img src="/img/in-post/cpp_img/builder_1.pdf" width="100%"></center>

**[2] 建造者设计模式代码示例** <br>
Part-1 Product建造者目标产品类
```cpp
class Product{
    private:
        std::vector<std::string> pencil;
    public:
        // 向Product类注册产品
        void RegistModule(std::string parts){
            pencil.push_back(parts);
        }
        // 打印所有目标产品中的组件 
        void ListProductParts(){
            std::cout<<"Pencil is made up of: ";
            std::vector<str::string>::iterator it;
            for(it=pencil.begin(); it!==pencil.end(); ++it){
                std::cout<<*it<<" ";
            }
        }
};
```
Part-2 抽象建造者类(用于具体建造者类提供接口规范)
```cpp
class AbstractBuilder{
    protected:
        // build tool function 
        virtual void BuildPart1()=0;
        virtual void BuildPart2()=0;
        // return constructed product
        virtual Product* GetResults()=0;  
};
```
Part-3 具体建造者类(对于抽象建造者类中提供的api的具体实现)
```cpp
class ConcreteBuilder1: public AbstractBuilder{
    private:
        Product *ptr_product__;
    public:
        ConcreteBuilder1(){// ctor
            ptr_product__ = new Product();
        }
        ~ConcreteBuilder1(){// dtor
            delete ptr_product__;
        }
        // builder1产品构建api 
        void BuildPart1(){
            ptr_product__->RegistModule("part1");
        }
        void BuildPart2(){
            ptr_product__->RegistModule("part2");
        }
        // 返回建造的产品
        Product* GetResults(){
            return ptr_product__;
        }
};
class ConcreteBuilder2: public AbstractBuilder{
    private:
        Product *ptr_product__;
    public:
        ConcreteBuilder2(){// ctor
            ptr_product__ = new Product();
        }
        ~ConcreteBuilder2(){// dtor
            delete ptr_product__;
        }
        // builder2产品构建api (针对不同的产品类型 产品构建内容不一致)
        void BuildPart1(){
            ptr_product__->RegistModule("component1");
        }
        void BuildPart2(){
            ptr_product__->RegistModule("component2");
        }
        // 返回建造的产品
        Product* GetResults(){
            return ptr_product__;
        }
};
```
Part-4 生产指导类(调用通用的产品创建流程对于同一类产品的细节进行定制)
```cpp
class Director{
    public:
        void Build(AbstractBuilder *ptr_build){
            // 调用Product类构建的'通用'流程
            ptr_build->BuildPart1(); 
            ptr_build->BuildPart2();
        }
};
```
Part-5 客户端程序代码示例
```cpp
int main(){
    // 创建指挥者
    Director *ptr_director = new Director();
    // 创建建造者
    AbstractBuilder *ptr_builder1 = new ConcreteBuilder1();
    AbstractBuilder *ptr_builder2 = new ConcreteBuilder2();
    // no.1 一号产品的生产 & 展示 
    ptr_director->Build(ptr_builder1);
    Product* ptr_product1 = ptr_builder1->GetResults();
    ptr_product->ListProductParts();
    // no.2 二号产品的生产 & 展示
    ptr_director->Build(ptr_builder2);
    Product* ptr_product2 = ptr_builder2->GetResults();
    ptr_product->ListProductParts();
    // 释放对象 销毁资源
    delete ptr_director, delete ptr_builder1, delete ptr_builder2;
}
```
**[3] 建造者设计模式结果分析** <br>
上面构建的建造者模式是一个非常基础的实现，只适用于固定的产品(Product)，以及所有产品按照相同规定的生产步骤进行(BuildPart1 + BuildPart2)。<br>
考虑针对上述代码的一种改进方案：将Product产品类用更加复杂的类层次结构进行表示，并且将生产指导类(Director)替换为带有继承层次关系的生产指导类层次结构进行构建，使之能处理同一类产品生产创建步骤不同的情况。

### 建造者模式(较复杂的例子)
**[1] 建造者设计模式UML类图** <br>
**1** 将Director类使用抽象类层次结构进行表示，是其能够构建出多种创建产品的流程。**2** 拓展产品类(Product)的类层次结构，增加了建造者模式目标产品线的种类。即Director不仅仅能在产品模块内容上进行定制督导，而且可以在产品模块数量上进行定制督导(需要修改Builder类、Director类)。

<center><img src="/img/in-post/cpp_img/builder_2.pdf" width="100%"></center>

**[2] 建造者设计模式代码示例** <br>
Part-1 拓展的Product建造者目标产品类
```cpp
class AbstractProduct{
    public:
        virtual RegistModule(std::string) = 0;
        virtual ListProductParts() = 0;
};
class ConcreteProduct1: public AbstractProduct{
    private:
        std::vector<std::string> pencil;
    public:
        void RegistModule(std::string parts){
            pencil.push_back(parts);
        }
        // 打印所有目标产品中的组件 
        void ListProductParts(){
            std::cout<<"Pencil is made up of: ";
            std::vector<str::string>::iterator it;
            for(it=pencil.begin(); it!==pencil.end(); ++it){
                std::cout<<*it<<" ";
            }
        }
};
class ConcreteProduct2{
    private:
        std::vector<std::string> rubber;
    ... // ditto, omitted
};
```
Part-2 拓展的抽象建造者类(实现方式同非拓展一致)
```cpp
class AbstractBuilder{
    protected:
        // build tool function 
        virtual void BuildPart1()=0;
        virtual void BuildPart2()=0;
        // return constructed product
        virtual Product* GetResults()=0;  
};
```
Part-3 拓展的具体建造者类(2点变化) <br>
(a:实现产品对象的模块组装上的多态性; b:实现产品对象的每个构成模块内部细节的多态性)
```cpp
class ConcreteBuilder1: public AbstractBuilder{...};// 同非拓展的一致 omitted
class ConcreteBuilder2: public AbstractBuilder{
    private:
        Product *ptr_product__;
    public:
        ConcreteBuilder2(){// ctor
            ptr_product__ = new Product();
        }
        ~ConcreteBuilder2(){// dtor
            delete ptr_product__;
        }
        // [注意] 重构的Builder2类的构建模块多了一个api
        // 这个多出的api将由ConcreteDirectory2提供支持
        // [另外] Builder2相比Builder1 不仅仅在模块内部细节上有所不同
        //        在模块组件的个数方面也可以定制
        void BuildPart1(){// no.1 继承自AbstractBuilder基类
            ptr_product__->RegistModule("component1");
        }
        void BuildPart2(){// no.2 继承自AbstractBuilder基类
            ptr_product__->RegistModule("component2");
        }
        void BuildPart3(){// no.3 派生类增补成员函数 
            ptr_product__->RegistModule("component3");
        }
        // 返回建造的产品
        Product* GetResults(){
            return ptr_product__;
        }
};
```
Part-4 拓展的生产指导类(拓展为抽象类层次结构 - 实现模块组装上的多态)
```cpp
class AbstractDirector{
    public:
        virtual void Build(AbstractBuilder *) = 0;
};
// 2个步骤建造完成的产品对应的concretebuilder都可以使用这个director进行督导
class ConcreteDirector1: public AbstractDirector{
    public:
        void Build(AbstractBuilder *ptr_build){
            // 调用concretebuilder1类构建的'通用'流程
            ptr_build->BuildPart1(); 
            ptr_build->BuildPart2();
        }
};
// 3个步骤建造完成的产品对应的concretebuilder都可以使用这个director进行督导
class ConcreteDirector2: public AbstractDirector{
    public:
        void Build(AbstractBuilder *ptr_build){
            // 调用concretebuilder2类构建的'通用'流程
            ptr_build->BuildPart1(); 
            ptr_build->BuildPart2();
            ptr_build->BuildPart3();
        }
};
```
Part-5 拓展的客户端程序代码示例()
```cpp
int main(){
    // 2-step director for pencil
    AbstractDirector *ptr_director1 = new ConcreteDirector1();
    // 3-step director for rubber
    AbstractDirector *ptr_director2 = new ConcreteDirector2();
    // 2-step builder for pencil
    AbstractBuilder *ptr_builder1 = new ConcreteBuilder1();
    // 3-step builder for rubber
    AbstractBuilder *ptr_builder2 = new ConcreteBuilder2();
    // no.1 一号产品的生产 & 展示 
    ptr_director1->Build(ptr_builder1);
    Product* ptr_product1 = ptr_builder1->GetResults();
    ptr_product->ListProductParts();
    // no.2 二号产品的生产 & 展示
    ptr_director2->Build(ptr_builder2);
    Product* ptr_product2 = ptr_builder2->GetResults();
    ptr_product->ListProductParts();
    // 释放对象 销毁资源
    delete ptr_director1, delete ptr_director2, 
        delete ptr_builder1, delete ptr_builder2;
}
```

## 关于建造者设计模式的思考
**[1] 相比之前介绍的工厂模式，两种模式有什么区别?分别在什么场景下应用两者?** <br>
根据[stackoverflow中Josh Berke的说法](https://stackoverflow.com/questions/328496/when-would-you-use-the-builder-pattern)：
*The key difference between a builder and factory IMHO, **is that a builder is useful when you need to do lots of things to build an object(并且创建该obj的流程具有一定规范性)**. For example imagine a DOM(html Dom元素的构建是应用建造者模式的典型例子). You have to create plenty of nodes and attributes to get your final object. **A factory is used when the factory can easily create the entire object within one method call(当目标obj非常易于建造完成，并且obj的产品依赖层次较复杂时)**.*



## Reference
> \<大话设计模式\> chapter13 p112 <br>
> https://blog.csdn.net/xiqingnian/article/details/42005627 <br>
> https://refactoringguru.cn/design-patterns/builder 

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
