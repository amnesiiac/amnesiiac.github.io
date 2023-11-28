---
layout: post
title: "cpp design pattern - composite"
subtitle: '[structual pattern] c++设计模式之组合模式'       
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-07-29 11:10
lang: ch 
catalog: true 
categories: cpp 
tags:
  - Time 2021
  - cpp design pattern
---
## 组合设计模式
**[1] 基本概念** <br> 
组合设计模式(composite pattern)：将一个复杂的类型系统组织成树形结构，并将其中的类型抽象成：根节点、有枝节点、叶子节点三种类型(extensible?)。<br>
被抽象的复杂类型系统通常和树形结构有较强的同源性；另外，被抽象成的树形类层次结构中的每个节点的相关操作具有很大的耦合性。

**[2] 要素组成** <br>
1) Component类(root)：组合设计模式中的根节点，被实现为整个类层次结构的抽象基类，以规范组合设计模式中所有类型节点的公共接口。

2) Composite类(node)：组合设计模式的有枝节点(非叶子结点)，继承自component类，用来存储类层次结构中有枝节点的信息，并实现具体相关操作。

3 )Leaf类(leafnode)：组合设计模式的叶子结点，同样继承自component类，用来存储类层次结构中叶子结点的信息，并实现相关操作。

**[3] 应用场景(simply borrowed from gof)** <br>
1) you want to represent part-whole hierarchies of objects. 如果希望将目标对象使用'部分-整体'的树形数据结构进行表达时，可能会考虑使用组合设计模式。

2) you want clients to be able to ignore the difference between compositions of objects and individual objects. Clients will treat all objects in the composite structure uniformly. 如果希望客户端代码忽略**有枝节点**和**叶子结点**之间的区别，并通过统一的api对树形结构中的两种节点进行操作时，应当使用组合设计模式。

3) further explain: 有枝节点包含叶子结点，叶子结点不包含叶子结点or有枝节点(这种包含关系通过list进行存储)，其中叶子结点称为部分，有枝节点称为整体。

我觉的严格意义上说，不是整体和部分的关系，尤其在代码实现的数据结构层面。它们分别属于不同的抽象层次，**组合设计模式就是为不同抽象层次的类型，提供一个基础(但是不必要)的通用api，以实现客户端代码的统一性、复用性**？这也是组合的含义，即将不同抽象层次(可能具有相互包含关系)的类型组合在一起，提供通用的api。

**[4] 组合设计模式的strength & weakness** <br>
**Strength** <br>
**1** It defines class hierarchies that contain primitive and complex objects. 将不同抽象层次的obj组合处理，使得客户端代码更加统一整洁。<br>
**2** It makes easier to you to add new kinds of components. 模型非常易于扩展新的节点(comp or leaf)。<br>
**3** It provides flexibility of structure with manageable class or interface. 模型结构有很强的灵活性，接口统一易使用。<br>
**Weakness** <br>
**1** Implementation of component interfaces is very challenging. 通用节点接口的实现时非常有挑战性的(需要应对不同抽象层次的对象的考验)。<br>
**2** Subsequent adjustment of composite features is difficult and cumbersome to realize. 对于一个构建完的组合组合设计模型的特性进行后续修改是困难的，如想要删除通用节点的一个api，则继承自它的所有的有枝节点、叶子结点代码都需要调整。

## composite设计模式案例
**[1] 案例目标** <br>
给定公司组织架构如下图所示，组织架构中的每个部分的结点需要提供4种功能api。使用恰当的设计模式完成组织架构实现，要求所实现的组织结构模型可以扩展。

<center><img src="/img/in-post/cpp_img/composite_1.pdf" width="100%"></center>

**[2] composite设计模式公司结构组织案例UML类图**

<center><img src="/img/in-post/cpp_img/composite_2.pdf" width="100%"></center>

**[3] composite设计模式代码示例** <br>
Part-1 组合设计模式中抽象基类代码示例(root)
```cpp
class AbstractDepartment{
    protected:
        std::string name__;
    public:
        AbstractDepartment(std::string name):name__(name){};
        // add node
        virtual void Add(AbstractDepartment *ptrnode) = 0;
        // remove node
        virtual void Remove(AbstractDepartment *ptrnode) = 0;
        // show node info
        virtual void Display(int depth) = 0;
        // show node duty
        virtual void LineOfDuty() = 0;
        // overload operator== 为了进行node类型的比较
        bool operator==(const AbstractDepartment &node) const{
            return this->name__ == node->name__;
        }
};
```
Part-2 组合设计模式中派生类有枝节点代码示例(root)
```cpp
class ConcreteDepartment: public AbstractDepartment{
  private:
    // 以当前ptr_node___为链表首节点创建部门关系链表 
    std::list<AbstractDepartment*> *ptr_node___;
  public:
    // init department name & init department list
    ConcreteDepartment(std::string name):AbstractDepartment(name){
      *ptr_node___ = new list<AbstractDepartment*>;
    }
    ~ConcreteDepartment(){
      for(auto iter=ptr_node___->begin(); iter!=ptr_node___->end(); ++iter){
          delete iter;
      }
      delete ptr_node___;
    }
    // 为树形类层次结构维护的内部数据结构list增加节点
    void Add(AbstractDepartment *ptrnode){
      ptr_node___->push_back(ptrnode);
    }
    // 为树形类层次结构维护的内部数据结构list删除节点
    void Remove(AbstractDepartment *ptrnode){
      for(auto iter=ptr_node___->begin(); iter!=ptr_node___->end(); ++iter){
        if(**iter == *ptr_node___){// 主要需要使用双重解引用 才能调用op==
            ptr_node___->erase(iter);
        }
      }
    }
    // 打印department名称信息
    void Display(int depth){
      // no.1 打印当前node名字信息
      for(int i=0; i<depth; ++i){
        std::cout<<"-";
      }
      std::cout<<name__;
      // no.2 打印node后继名字信息
      for(auto iter=ptr_node___->begin(); iter!=ptr_node___->end(); ++iter){
        (*ptr_node___)->Display(depth+4);// 每跨一级部门 增加4个indentation
      }
    }
    // nodeduty信息
    void LineOfDuty(){
      for(auto iter=ptr_node___->begin(); iter!=ptr_node___->end(); ++iter){
        (*iter)->LineOfDuty();// 注意一定要使用*iter
      }
    }
};
```
Part-3 组合设计模式中派生类叶子结点代码示例(leaf)
```cpp
class HRDepartment: public AbstractDepartment{
    public:
        HRDepartment(std::string name):AbstractDepartment(name){}// set name
        void Add(AbstractDepartment *ptrnode){}
        void Remove(AbstractDepartment *ptrnode){}
        // 打印department名称信息
        void Display(int depth){
            // 打印当前node名称信息
            for(int i=0; i<depth; ++i){
                std::cout<<"-";
            }
            std::cout<<name__<<std::endl;
            // leaf无后继名称信息
        }
        // 打印department duty信息
        void LineOfDuty(){
            std::cout<<name__<<": Staff recruitment, training, managment."
        }
};
class FinanceDepartment: public AbstractDepartment{
    public:
        FinanceDepartment(std::string name):AbstractDepartment(name){}// set
        void Add(AbstractDepartment *ptrnode){}
        void Remove(AbstractDepartment *ptrnode){}
        // 打印department名称信息
        void Display(ind depth){
            // 打印当前node信息
            for(int i=0; i<depth; ++i){
                std::cout<<"-";
            }
            std::cout<<name__<<std::endl;
            // leaf无后继信息
        }
        // 打印department duty信息
        void LineOfDuty(){
            std::cout<<name__<<": Finance earnings, spendings, managment."
        }
};
```
Part-4 组合设计模式中客户端调用代码示例
```cpp
int main(){
    // root beijing
    // create node
    AbstractDepartment* root=new ConcreteDepartment("北京总公司");
    // add leaf
    root->Add(new HRDepartment("总公司人力资源部"));
    // add leaf
    root->Add(new FinanceDepartment("总公司财务处"));
    // composite shanghai
    AbstractDepartment* comp_sh=new ConcreteDepartment("上海华东分公司");
    comp_sh->Add(new HRDepartment("华东分公司人力资源部"));
    comp_sh->Add(new FinanceDepartment("华东分公司财务处"));
    root->Add(comp_sh);// link the node list: root->comp_sh
    // composite nanjing 
    AbstractDepartment* comp_nj=new ConcreteDepartment("南京办事处");
    comp_nj->Add(new HRDepartment("南京办事处人力资源部"));
    comp_nj->Add(new FinanceDepartment("南京办事处财务处"));
    comp_sh->Add(comp_nj);// link the node list: comp_sh->comp_nj
    // composite hangzhou 
    AbstractDepartment* comp_hz=new ConcreteDepartment("杭州办事处");
    comp_hz->Add(new HRDepartment("杭州办事处人力资源部"));
    comp_hz->Add(new FinanceDepartment("杭州办事处财务处"));
    comp_sh->Add(comp_hz);// link the node list: comp_sh->comp_hz
    // display department structure 
    std::cout<<"Structure: "<<std::endl;
    root->Display(1);
    std::cout<<std::endl;
    // display department duty 
    std::cout<<std::endl<<"Duty: "<<std::endl;
    root->LineOfDuty();
    // recycle  
    delete root;
}
```
Part-5 组合设计模式客户端输出代码结果
```txt
Structure:
-北京总公司
-----总公司人力资源部
-----总公司财务处
---------上海华东分公司
---------华东分公司人力资源部
---------华东分公司财务处
-------------南京办事处
-------------南京办事处人力资源部
-------------南京办事处财务处
-------------杭州办事处
-------------杭州办事处人力资源部
-------------杭州办事处财务处

Duty: 
总公司人力资源部: Staff recruitment, training, managment.
总公司财务处: Finance earnings, spendings, managment.

总公司人力资源部: Staff recruitment, training, managment.
总公司财务处: Finance earnings, spendings, managment.

华东分公司人力资源部: Staff recruitment, training, managment.
华东分公司财务处: Finance earnings, spendings, managment.

南京办事处人力资源部: Staff recruitment, training, managment.
南京办事处财务处: Finance earnings, spendings, managment.

杭州办事处人力资源部: Staff recruitment, training, managment.
杭州办事处财务处: Finance earnings, spendings, managment.
```

**[4] composite设计模式代码结果分析思考** <br>
上述公司组织架构中，并不是所有的节点都需要实现继承而来的4个接口，对于有枝节点，需要重新定义4个api，对于叶子结点，虽然需要定义所有继承而得的api(语法限制)，但是对于无用的api可以不给出定义。另外，为了避免外部调用无用api而出错，可以为其增加private访问限制条件。

## Reference
> \<大话设计模式\> chapter19 p189 <br>
> https://blog.csdn.net/xiqingnian/article/details/42081097 <br>
> gof chapter4 p183 <br>

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
