---
layout: post
title: "cpp design pattern - chain of responsibility"
subtitle: '[behavioral pattern] c++设计模式之责任链模式' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-08-04 13:46
lang: ch 
catalog: true 
categories: cpp 
tags:
  - Time 2021
  - cpp design pattern
---
## 责任链模式
**[1] 基本概念** <br> 
职责链模式(chain of responsibility)：允许多个对象处理请求，从而避免请求发送者和请求接受者之间的耦合。将这些对象构成一个链，沿着对象链传递请求，直到有一个对象能够处理该请求为止。

**[2] 要素组成** <br>
**Request类**：请求类，在职责链中传递的信息载体类型。<br>
**AbstractHandler类**：请求处理抽象基类，定义一个请求处理接口。<br>
**ConcreteHandler类**：请求处理派生子类，实现具体请求处理接口。

**[3] 应用场景(simply borrowed from gof)** <br>
1) more than one object may handle a request, and the handler isn't known a priori. The handler should be ascertained automatically. 对于一个请求，有至少一个handler可能能够处理，需要根据条件自动匹配处理request的handler。<br>
2) you want to issue a request to one of several objects without specifying the receiver explicitly. <br>
3) the set of objects that can handle a request should be specified dynamically. 处理链构建顺序需要客户端程序动态指定。<br>

**[4] strength & weakness** <br>
**strength** <br>
**1** 职责链在客户端程序中，可以手动控制、改变请求处理的顺序。**2** 满足单一职责原则，对于发起操作和执行操作的类型进行解耦。**3** 满足开闭原则，可以在不改变客户端代码的情况下，在职责链内部新增request处理类。<br>
**weakness** <br>
**1** 对于请求依赖条件较为复杂的情况下，每个职责链中的具体ConcreteHandler类的RequestApplication函数可能较难实现。 **2** 当职责链非常复杂的场景下(e.g.多链、循环引用)，职责链结点的析构需要谨慎设计、执行。

## 责任链设计模式案例
**[1] 案例目标** <br>
使用职责链设计模式，模拟下属员工请求(请假、加薪)在经理、总监、总经理逐级传递，直到找到合理回应的场景。

**[2] 责任链设计模式案例UML类图**

<center><img src="/img/in-post/cpp_img/chain_of_responsibility_1.pdf" width="100%"></center>

**[3] 责任链设计模式代码示例** <br>
Part-1 请求类(用于在职责链中传递的对象类型定义)
```cpp
class Request{
    private:// 对象内容
        std::string request_type___;
        std::string request_content___;
        int number___;
    public:// 操纵对象的接口
        void SetType(std::string type):request_type___(type){}
        std::string GetType(){
            return request_type___;
        }
        void SetContent(std::string content):request_content___(content){}
        std::string GetContent(){
            return request_content___;
        }
        void SetNumber(int num):number___(num){}
        int GetNumber(){
            return number___;
        }
};
```
Part-2 AbstractHandler类(职责链中结点的抽象基类)
```cpp
class Manager{
    protected:
        std::string name__;
        Manager *ptr_superior__;
    public:
        Manager(){}// 为成员执行默认初始化
        Manager(std::string name):name__(name),ptr_superior__(nullptr){}
        ~Manager(){
            // 如果使用这种析构代码 则客户端程序只需要析构职责链的首元素即可
            // 推荐使用new-delete一一对应的方式
            // if(ptr_superior__!=nullptr){// 判断是否还有后继
            //     delete ptr_superior__;
            // }
        }
        // 为节点类设置前驱
        void SetSuperior(Manager *ptrsuperior){
            ptr_superior__ = ptrsuperior;
        }
        // 处理请求的抽象方法 - 处理请求or将请求向上级传递
        virtual void RequestApplication(Request *ptrrequest) = 0;
};
```
Part-3 ConcreteHandler1类(职责链中结点的具体派生类)
```cpp
class CommonManager: public Manager{
    public:
        CommonManager(std::string name):Manager(name){}
        void RequestApplication(Request *ptrrequest){
            // within authority
            if(ptrrequest->GetType()=="请假" && ptrrequest->GetNumber()<=2){
                std::cout<<name__<<":"<<ptrrequest->GetContent()
                    <<" Num:"<<ptrrequest->GetNumber()<<" approved."
            } 
            // beyond authority
            else{
                if(ptr_superior__!=nullptr){
                    ptr_superior__->RequestApplication(ptrrequest);
                }
            }
        }
};
```
Part-4 ConcreteHandler2类(职责链中的具体派生类)
```cpp
class Majordomo: public Manager{
    public:
        Majordomo(std::string name):Manager(name){} 
        void RequestApplication(Request *ptrrequest){
            if(ptrrequest->GetType()=="请假" && ptrrequest->GetNumber()<=5){
                std::cout<<name__<<":"<<ptrrequest->GetContent()
                    <<" Num:"<<ptrrequest->GetNumber()<<" approved."
            }
            else{
                if(ptr_superior__!=nullptr){
                    ptr_superior__->RequestApplication(ptrrequest);
                }
            }
        }
};
```
Part-5 ConcreteHandler3类(职责链中的具体派生类)
```cpp
class GeneralManager: public Manager{
    public:
        GeneralManager(std::string name):Manager(name){}
        void RequestApplication(Request *ptrrequest){
            if(ptrrequest->GetType()=="请假"){// 需处理请求的阈值情况
                std::cout<<name__<<":"<<ptrrequest->GetContent()
                    <<" Num:"<<ptrrequest->GetNumber()<<" approved."
            }
            if(ptrrequest->GetType()=="加薪" && ptrrequest->GetNumber()<=500){
                std::cout<<name__<<":"<<ptrrequest->GetContent()
                    <<" Num:"<<ptrrequest->GetNumber()<<" approved."
            }
            // 需处理请求的阈值情况
            if(ptrrequest->GetType()=="加薪" && ptrrequest->GetNumber()>500){
                std::cout<<name__<<":"<<ptrrequest->GetContent()
                    <<" Num:"<<ptrrequest->GetNumber()<<" hold on."
            }
        }
};
```
Part-6 责任链设计模式客户端代码示例
```cpp
int main(){
    // 构建职责链上的对象
    Manager *jingli = new CommonManager("jingli");
    Manager *zongjian = new Majordomo("zongjian");
    Manager *zongjingli = new GeneralManager("zongjingli");
    // 将职责链中的对象链接起来 构成chain of responsibility 
    jingli->ptr_superior__=zongjian;
    zongjian->ptr_superior__ = zongjingli;
    // 创建请求 设置请求内容属性
    Request quest1;
    quest1.SetType("请假");
    quest1.SetContent("melon asked for paid leave");
    quest1.SetNumber(1);
    // 每次请求都由经理发起 但是具体每个请求在职责链上那个层级被处理不确定
    jinli->RequestApplication(quest1);
    // ditto omitted
    Request quest2;
    quest2.SetType("请假");
    quest2.SetContent("melon asked for paid leave");
    quest2.SetNumber(4);
    jinli->RequestApplication(quest2);
    Request quest3;
    quest3.SetType("请假");
    quest3.SetContent("melon asked for paid leave");
    quest3.SetNumber(6);
    jinli->RequestApplication(quest3);
    jinli->RequestApplication(quest4);
    Request quest4;
    quest4.SetType("加薪");
    quest4.SetContent("melon asked for salary rise");
    quest4.SetNumber(50);
    jinli->RequestApplication(quest4);
    Request quest5;
    quest5.SetType("加薪");
    quest5.SetContent("melon asked for salary rise");
    quest5.SetNumber(2000);
    jinli->RequestApplication(quest5);
    delete jinli, zongjian, zongjingli;
}
```
Part-7 责任链设计模式程序输出示例
```txt
Commonmanager:melon asked for paid leave 1: approved.
Majordomo:melon asked for paid leave 4: approved.
GeneralManager:melon asked for paid leave 6: approved.
GeneralManager:melon asked for salary rise 50: approved.
GeneralManger:melon asked for salary rise 2000: hold on.
```

## Reference
> \<大话设计模式\> chapter24 p263<br>
> https://blog.csdn.net/xiqingnian/article/details/42149387 <br>
> gof chapter5 p249 <br>
> https://refactoringguru.cn/design-patterns/chain-of-responsibility

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
