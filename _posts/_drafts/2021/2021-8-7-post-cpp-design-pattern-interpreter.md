---
layout: post
title: "cpp design pattern - interpreter"
subtitle: '[behavioral pattern] c++设计模式之解释器模式' 
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-08-07 14:59
lang: ch 
catalog: true 
categories: cpp 
tags:
  - Time 2021
  - cpp design pattern
---
## 解释器模式
**[1] 基本概念** <br> 
解释器模式：给定一种语言(或表示的集合)，定义它的一种规则表示。并定义一个解释器，解释器内部实现了表示该语言的各种规则，并通过构建解释器类层次结构以建立用于解释该语言的语法树。这种通过类层次结构综合描述一个较为复杂的规则体系的方式成为解释器模式。

**[2] 要素组成** <br>
**1) AbstractInterpreter(解释器抽象基类)**：通用规则制定抽象基类，规范通用的抽象的解释方法接口。定义的接口api由抽象语法树实例中的所有节点共享。抽象语法树是由解释器抽象基类制定的通用规则集合，以及所有继承子类定义的特殊规则集合共同构成的。<br>
**2) ConcreteInterpreter(解释器具体派生类)**：特殊定制规则制定派生类，实现抽象解释器类所规定的通用语法翻译接口；实现派生类定制的语法翻译接口(用于补充基础语法树规则)。<br>
**3) Context(解释器外部辅助资源管理类)**：包含在解释器之外的一些全局、辅助信息。

**[3] 应用场景(simply borrowed from gof)** <br>
**1 场景：Use the Interpreter pattern when there is a language to interpret**, and you can represent statements in the language as abstract syntax trees. <br>
**2 解释器模式适用情况：The Interpreter pattern works best when the grammar is simple**. For complex grammars, the class hierarchy for the grammar becomes large and unmanageable. Tools such as parser generators are a better alternative in such cases. They can interpret expressions without building abstract syntax trees, which can save space and possibly time. <br>
**3 执行效率不是解释器设计模式在语言解析场景中的障碍：Efficiency is not a critical concern**. The most efficient interpreters are usually not implemented by interpreting parse trees directly but by first translating them into another form. For example, regular expressions are often transformed into state machines. But even then, the translator can be implemented by the Interpreter pattern, so the pattern is still applicable.

**[4] strength & weakness - todo** <br>
**strength** <br>
**weakness** <br>

## 解释器设计模式案例
**[1] 案例目标** <br>
给定一个乐谱，包含3种基本音阶("1"、"2"、"3")分别表示低音中音高音，包含7种音符("C"、"D"、"E"、"F"、"G"、"A"、"B")，现有一段"上海滩"乐谱，使用解释器模式构建"乐谱翻译语法树"实现乐谱的自动翻译。

**[2] 解释器设计模式案例UML类图**

<center><img src="/img/in-post/cpp_img/interpreter_1.pdf" width="100%"></center>

**[3] 解释器设计模式代码示例** <br>
Part-1 解释器外部变量类(解耦合设计) - 用于管理解释器制定的规则模型中"乐谱"资源
```cpp
class Context{// 乐谱存放、设置、获取功能类
    private:
        // 资源存储string对象(stl自动管理资源 无需自定义ctor、dtor) 
        std::string text___;
    public:
        // 资源管理函数
        void SetText(std::string text){
            this->text___ = text;
        }
        std::string GetText(){
            return this->text___;
        }
};
```
Part-2 抽象解释器类型(用于提供通用的解释翻译函数，制定通用的解释翻译规则)
```cpp
class AbstractInterpreter{// AbstractInterpreter
    public:
        // 将实际乐谱设置、存放、获取解耦合为外部变量 -> 高内聚、低耦合
        void Interpret(Context *ptrcontext){
            std::string str1 = ptrcontext->GetText();
            std::string buf;
            std::string str2;
            if(str1.length()==0){
                return;
            }
            else{
                // string -> vector 将string中存放的乐谱导入vector容器中
                std::vector<std::string> vec;
                std::stringstream cs(str1);
                while(cs>>buf){
                    vec.push_back(buf);
                }
                std::string playkey = vec[0];// 定义音符notes
                std::string playvalue = vec[1];// 定义音阶scale
                Execute(playkey, playvalue);// 多态地play note或者play scale
                // 将播放完毕的音符、音调删除 剩余的乐谱继续存入Context类中以备下次调用
                vec.erase(vec.begin(), vec.begin()+2);// 删除已经播放的部分
                // 按照Context乐谱通用存放格式进行整理
                std::vector<std::string>::iterator it;
                for(it=vec.begin(); it!=vec.end(); ++it){
                    str2+=(*it);
                    if(it!=vec.end()-1){
                        str2+=" ";
                    }
                }
                ptrcontext->SetText(str2);// 将整理好的剩余乐谱存入Context管理类
            }
        }
        // 定义了字符、音阶的抽象interpret接口函数
        virtual void Execute(std::string key, std::string value) = 0
};
```
Part-3 音符类(派生类) - 用于实现抽象基类中规定的解释翻译接口(制定解释翻译子规则)
```cpp
class NotesInterpreter: public AbstractInterpreter{// ConcreteInterpreter1
    public:
        // 具体实现抽象基类中的interpreter方法
        void Excute(std::string key, std::string value){
            std::string note;// 存放映射完成后的音符
            switch(key[0]){// string[0]=char
                case 'C':// char
                    note="1";
                    break;
                case 'D':// char
                    note="2";
                    break;
                case 'E':// char
                    note="3";
                    break;
                case 'F':// char
                    note="4";
                    break;
                case 'G':// char
                    note="5";
                    break;
                case 'A':// char
                    note="6";
                    break;
                case 'B':// char
                    note="7";
                    break;
            }
            std::cout<<note<<" ";
        }
};
```
Part-4 音阶类(派生类) - 用于实现抽象基类中规定的解释翻译接口(制定解释翻译子规则)
```cpp
class ScaleInterpreter: public AbstractInterpreter{// ConcreteInterpreter2
    public:
        // 具体实现抽象基类中的interpreter方法
        void Excute(std::string key, std::string value){
            std::string scale;
            switch(value[0]){// string[0]=char
                case '1':// char
                    scale="BASS";
                    break;
                case '2':
                    scale="ALTO";
                    break;
                case '3':
                    scale="TREBLE";
                    break;
            }
            std::cout<<scale<<" ";
        }
}
```
Part-5 客户端代码示例
```cpp
int main(){
    Context context;// 没有显示定义析构函数 content对象被销毁内部资源自动释放
    std::cout<<"Shanghai bund, ready? sing..."<<std::endl;
    // 0.5 表示"延续"上一个音阶类型(无输出)
    context.SetText("O 2 E 0.5 G 0.5 A 3 E 0.5 G 0.5 D 3 E 0.5 
        G 0.5 A 0.5 O 3 C 1 O 2 A 0.5 G 1 C 0.5 E 0.5 D 3");
    AbstractInterpreter *ptrinterpreter;
    while(context.GetText().length()>0){// 每个字符依次进行解析
        char ch = context.GetText()[0];// 获取当前需要被翻译的string -> char
        switch(ch){
            case '0':// 创建一个音符翻译器 
                ptrinterpreter = new NotesInterpreter();
                break;
            case 'C':// 下面各种情况都创建音阶翻译器
            case 'D':
            case 'E':
            case 'F':
            case 'G':
            case 'A':
            case 'B':
                ptrinterpreter = new ScaleInterpreter();
                break;
            ptrinterpreter->Interpret(&context);// 进行翻译并输出
            delete ptrinterpreter;// 销毁翻译器(准备下一个字符用于翻译)
        }
    }
    return 0;
}
```

## Reference
> \<大话设计模式\> chapter27 p297 <br>
> https://blog.csdn.net/xiqingnian/article/details/42222369 <br>
> gof chapter5 p274 <br>

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
