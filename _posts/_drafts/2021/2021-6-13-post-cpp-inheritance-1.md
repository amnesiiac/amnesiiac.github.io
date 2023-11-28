---
layout: post
title: "cpp - inheritance-1"
subtitle: '多重继承、虚继承相关知识整理'
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-06-13 18:06
lang: ch 
catalog: true 
categories: cpp 
tags:
  - Time 2021
  - cpp fundermentals
---
## 继承关系中的名字解析
**[1] 简单直接继承关系中的名字解析**
<center><img src="/img/in-post/cpp_img/inheritance_2.pdf" width="40%"></center>

可以使用如下的简化代码对于上图中的继承关系进行描述：
```c++
class zooAnimal{
    public:
        ostream& print(ostream&) const;
        string is_a;
        int ival;
    private:
        double dval;
};
class Bear: public zooAnimal{
    public:
        ostream& print(ostream&) const;
        int mumble(int);
        string name;
        int ival;
};
```
**(1) 名字解析实例**
```c++
// 1 在调用者所在类域中查找 bear.is_a -> 首先在Bear类内进行查找 -> not found
// 2 在直接基类中进行查找 -> 找到bear.zooAnimal::is_a -> 并且访问合法 -> ok
Bear bear;
bear.is_a;
``` 
**(2) 名字解析实例**
```c++
// 派生类和基类同名成员的访问 -> 派生类成员可以屏蔽掉基类成员 
Bear bear;
bear.ival;// equivalent to bear.Bear::ival;
```
**(3) 名字解析实例**
```c++
// 通过派生类对象访问基类成员 -> 需要借助域访问作用符 
Bear bear;
bear.zooAnimal::ival;// equivalent to bear.(zooAnimal::ival)
```
**(4) 一个综合的例子**
```c++
int ival;
int Bear::mumble(int ival){
    // 依次调用 函数参数 全局变量 zooAnimal类成员 Bear类成员
    return ival + ::ival + zooAnimal::ival + Bear::ival;
}
```
**(5) 编译器名字解析步骤简析**
```c++
// 1 先解析一个成员   2 再判断访问该成员是否合法 
int dval;
int Bear::mumble(int ival){
    // 1 编译器将ival和dval分别解析成 函数参数&zooAnimal::dval
    // 2 编译器检查是否合法 -> zooAnimal::dval为私有成员 类域外不能访问 -> wrong!
    return ival+dval;
}
```
**[2] 多重继承下的名字解析** 多重继承下，一个派生类拥有多个直接基类，如果从多个直接基类中继承了同名的成员，则可能导致二义性调用。

<center><img src="/img/in-post/cpp_img/inheritance_1.pdf" width="40%"></center>

```c++
class Endangered{
    public:
        ostream& print(ostream&) const;// no.1 可能引起二义性
        void highlight();// no.2 可能引起二义性
};
class zooAnimal{
    public:
        bool onExihibit() const;
    private:
        bool highlight(int zoo_location);// no.2 可能引起二义性
};
class Bear: public zooAnimal{
    public:
        ostream& print(ostream&) const;// no.1 可能引起二义性
        void dance(dance_type) const;
};
class Panda: public Bear, public Endangered{
    public:
        void cuddle() const;
};
```
**(1) 名字解析实例** 多层继承中，名字解析的查找过程对继承列表中的每个继承子树同时进行检查。如果**只**在其中一个继承子树找到了声明，则名字解析完成。
```c++
int main(){
    Panda huanhuan;
    huanhuan.dance(Bear::macarena);// 在Bear/zooAnimal继承子树中找到声明
}
```
**(2)** 如果在多个继承子树中都找到了声明，则表示对这个名字的引用是二义的，会产生一个编译时刻错误。
```c++
int main(){
    Panda huanhuan;
    huanhuan.print(cout);// 二义性的print名字引用 Bear? or Endangered?
}
```
**(3)** 多个继承子树中同名成员产生的二义性的解决：
```c++
// no.1 使用域作用符对调用的名字进行修饰 -> 治标不治本 需要类的用户决定调用
huanhuan.Bear::print(cout);
// no.2 为Panda类定义它的调用路径 -> 治本 无须类的用户决定调用
inline void Panda::highlight(){// (a) 为Panda类指定调用函数
    Endangered::highlight();
}
inline ostream& Panda::print(ostream &os) const{// (b) 指定多个调用函数
    Bear::print(os);
    Endangered::print(os);
    return os;
}
```


## 多重继承
多重继承是派生类继承自多个直接基类，多重继承的派生类继承了所有父类的属性。构造一个派生类的对象将同时构造并初始化它的所有基类的子对象，每个派生类只能初始化其直接基类。

### 大熊猫在动物园中的多重继承关系

<center><img src="/img/in-post/cpp_img/inheritance_1.pdf" width="40%"></center>

上述关系图可以使用伪代码进行描述：
```c++
class Bear: public zooAnimal;
class Panda: public Bear, public Endangered;
```
上述多重继承关系中的对象、子对象包含关系：
```c++
// Bear类子对象包含了zooAnimal类子对象
// Panda对象由Bear类子对象+Endangered类子对象+自身的非静态成员构成
Panda huanhuan;
```
**[ctor调用顺序]** 在构建Panda类对象时，直接基类的构造函数的调用(用于初始化子对象)按照派生表中的顺序进行，不受基类在Panda类构造函数成员初始化表中是否存在以及被列出的顺序的影响：
```c++
class Panda: public Bear, public Endangered;// 派生表
// 虽然Panda构造函数中Bear构造函数不存在 但仍按照Bear() Endangered() 顺序调用
Panda:Panda():Endangered(Endangered environment, Endangered critical){...}
// 虽然初始化表中Endangered类构造函数位于Bear类之前 但仍按照派生表顺序调用
Panda:Panda():Endangered():Bear(){...}
```
**[dtor调用顺序]**  调用Panda类的析构函数用于销毁对象时，各个直接基类的析构函数调用顺序和构造函数调用顺序相反，销毁Panda对象时析构调用顺序为：
```c++
~Panda();// no.1
~Endangered();// no.2
~Bear();// no.3
~zooAnimal();// no.4
```
**[调用两个基类的同名成员导致二义性]** 如果Bear类和Endangered类同时定义了print函数，则下面的代码调用会产生二义性：
```c++
Panda huanhuan;
// wrong! Bear::print(ostream&) or Endangered::print(ostream&)
huanhuan.print(cout);
```

### 多继承对于虚函数的影响
```c++
class Bear: public zooAnimal{
    public:
        virtual ~Bear();
        virtual ostream& print(ostream&) const;
        virtual string is_a() const;// exclusive in Bear
};
class Endangered{
    public:
        virtual ~Endangered();
        virtual ostream& print(ostream&) const;
        virtual void highlight() const;// exclusive in Endangered 
};
class Panda: public Bear, public: Endangered{
    public:
        virtual ~Panda();
        virtual ostream& print(ostream&) const;
        virtual void cuddle();// exclusive in Panda
};
```
**(1)** Panda类对象用于初始化Bear类的指针或引用 -> **(Panda中特有的部分)**以及**(Endangered所有部分)**不能被访问。
```c++
Bear *ptr_b = new Panda;
ptr_b->print(cout);// √ -> Panda::print(ostream&)
ptr_b->is_a();// √ -> call Bear::is_a()
ptr_b->cuddle();// × -> cuddle not part of Bear
ptr_b->highlight();// × -> highlight not part of Bear
delete ptr_b;// √ -> call Panda::~Panda() 
```
**(2)** Panda类对象用于初始化Endangered类的指针或引用 -> **(Panda中特有的部分)**以及**(Bear所有部分)**不能被访问。
```c++
Endangered *ptr_e = new Panda;
ptr_e->print(cout);// √ -> Panda::print(ostream&)
ptr_e->is_a();// × -> 不能访问Bear特有的部分 
ptr_e->cuddle();// × -> 不能访问Panda特有的部分
ptr_e->highlight();// √ -> Endangered::hightlight()
delete ptr_e;// √ -> call Panda::~Panda()
```
**(3)** delete执行调用的函数和new expression调用的构造函数属于同类型，和new expression用来初始化的指针类型无关：
```c++
Bear *ptr = new Panda;
delete ptr_b;// -> Panda::~Panda()
Endangered *ptr = new Panda;
delete ptr_e;// -> Panda::~Panda()
```
析构函数按照声明继承列表的逆向顺序进行依次调用。


## 虚拟继承
### 虚继承的作用
**[1] 单继承模型(模型中每个类只有一个直接基类)** 下图所示的单继承模型提供了最有效、最紧凑的对象表示。在很多场景下，单继承模型能够一定程度上避免二义性。

<center><img src="/img/in-post/cpp_img/inheritance_3.pdf" width="40%"></center>

**[2] 多继承模型(模型中的类拥有多个直接基类)** 下图所示的多继承模型中，熊科和浣熊科同时继承自zooAnimal类，即多继承模型中不同的继承子树拥有交点：这种特殊的多继承模型会产生**存储效率问题**、**访问实例二义性问题**：Panda类包含Bear类以及Raccoon类的子对象，而Bear类和Raccoon又分别包含同一个基类zooAnimal。

<center><img src="/img/in-post/cpp_img/inheritance_4.pdf" width="40%"></center>

按照如下方式对上图中的继承关系进行描述，则：Panda类同时拥有zooAnimal类的两个相同的子对象，从效率上讲存储同一个基类的两个子对象副本浪费了存储区；未经限定的访问zooAnimal类的成员也容易引起错误：不知道访问哪个子对象成员。 
```c++
class Bear: public zooAnimal{...};
class Raccoon: public zooAnimal{...};
class Panda: public Bear, public Raccoon{...};
```
针对上述问题的解决方案是：采用虚拟继承。本质上，虚拟继承提供了一种\<按引用\>的继承机制。在这种机制下，无论该基类在派生层次中出现多少次，只有一个共享的基类子对象被继承。共享的基类称为虚基类(virtual base class)。
```c++
class Bear: public virtual zooAnimal{...};
class Raccoon: public virtual zooAnimal{...};
```
**[尽可能使用虚继承？还是避免使用虚继承？]** 在设计构造类时，由于不知道这个类在后续的继承派生中，是否被不同的继承子树包含多次，那么是否应该尽可能将它设计成虚基类的形式？答案是否定的，将每个可能的类定义成虚基类将会极大的影响程序性能。因此，虚拟继承一般用来保证**当下**的程序继承关系的稳定高效，而不用来考虑**未来**类继承树被后续程序使用的情况。除非万不得已使用虚继承方式解决问题，否则不要滥用它(能不用则不用)。

### 虚继承的声明方式
**[1]** virtual和public关键字的顺序不重要，下面的两条声明等价。
```c++
class Bear: public virtual zooAnimal{...};// 个人喜欢使用这种方式
class Bear: virtual public zooAnimal{...};
```
**[2]** 声明为虚继承，不影响派生类和基类之间的各种类型转换关系：
```c++
extern void dance(const Bear*);
extern void rummage(const Raccoon*);
extern ostream& operator<<(ostream&, const zooAnimal&);
int main(){
    Panda huanhuan;
    dance(&huanhuan);// no.1 √ Panda -> Bear
    rummage(&huanhuan);// no.2 √ Panda -> Raccoon
}
```

### 待完善的大熊猫虚继承类树定义
下面代码用于对大熊猫虚继承类关系模型进行建模：

<center><img src="/img/in-post/cpp_img/inheritance_4.pdf" width="40%"></center>
**[虚基类的定义]**
```c++
#include<iostream>
#include<string>
class zooAnimal;
extern ostream& operator<<(ostream&, const zooAnimal&);
class zooAnimal{
    public:
        // zooAnimal虚基类构造函数声明
        zooAnimal(string name, bool onExhibit, string family_name)
            :_name(name),_onExhibit(onExhibit),_family_name(family_name){}
        virtual ~zooAnimal();
        virtual ostream& print(ostream&) const;
        string name() const{return _name;}
        string family_name() const{return _family_name;}// 
    protected:
        bool _onExhibit;
        string _name;
        string _family_name;
};
```
**[Bear类的定义]**
```c++
class Bear: public virtual zooAnimal{
    public:
        enum DanceType{two_left_feet, macarena, fandango, waltz};
        // Bear类构造函数声明 -> '可能'会调用zooAnimal构造
        Bear(string name, bool onExhibit=true)
            :zooAnimal(name, onExhibit, "Bear"), _dance(two_left_face){}
        virtual ostream& print(ostream&) const;
        void dance(DanceType);// exclusive func memeber
    protected:
        DanceType _dance;// exclusive property
};
```
**[Raccoon类的定义]**
```c++
class Raccoon: public virtual zooAnimal{
    public:
        // Raccoon构造函数声明 -> '可能'会调用zooAnimal构造
        Raccoon(string name, bool onExhibit=true)
            :zooAnimal(name, onExhibit, "Raccoon"), _pettable(false){}
        virtual ostream& print(ostream&) const;
        bool pettable() const{return _pettable;}// exclusive func member
        void pettable(bool petval){_pettable = petval;}// exclusive member
    protected:
        bool _pettable;// exclusive property
};
```
**[Panda类的定义]**
```c++
class Panda: public Bear, public Raccoon, public Endangered{
    public:
        // Panda构造函数声明
        Panda(string name, bool onExhibit=true);
        virtual ostream& print(ostream&) const;
        bool sleeping() const{return _sleeping;}
        void sleeping(bool newval){_sleeping = newval;}
    protected:
        bool _sleeping;// exclusive property
};
```

### 特殊初始化语义 - 最终派生类负责虚基类初始化(改进方案)
**[1] 非虚继承关系中，派生类只能显示初始化其直接基类。采用初始化表的方式属于隐式初始化，因此不能用于非虚继承关系中的基类初始化：**
```c++
class Bear: public zooAnimal{...};// non-virtual inherit
class Raccoon: public zooAnimal{...};// non-virtual inherit
class Panda: public Bear, public Raccoon, public Endangered{
    public:
        // no.1 在Panda初始化表中构造基类中的成员 -> wrong!
        Panda(string name, bool onExhibit=true)
            :_name(name), _onExhibit(onExhibit), _family_name("Panda"){}
        // no.2 在Panda初始化表中调用基类的构造函数 -> wrong!
        Panda(string name, bool onExhibit=true)
            :zooAnimal(name, onExhibit, "Panda"){}
};
```
**[2] 虚继承关系中，只有最终派生类(most derived class)能够进行虚基类的初始化。**具体最终派生类的类型由类对象的声明决定：
```c++
class Bear: public virtual zooAnimal{...};
class Raccoon: public virtual zooAnimal{...};
class Panda: public Bear, public Raccoon, public Endangered{
    public:
        Panda(string name, bool onExhibit=true):
            zooAnimal(name, onExhibit, "Panda"), 
            Bear(name, onExhibit), Raccoon(name, onExhibit), 
            Endangered(Endangered::environment, Endangered::critical), 
            _sleeping(false){}
};
```
上述代码中，类作者默认Panda作为最终派生类，用于初始化继承子树中Bear类和Raccoon类的虚基类zooAnimal。并且此时Bear类和Raccoon类对zooAnimal的构造函数调用不再执行。
```c++
// no.1 Bear是winnie的最终派生类 -> 因此Bear负责虚基类构造函数的调用执行
Bear winnie("pooh");
cout<<winnie.family_name<<endl;// 输出"Bear"
// no.2 Panda是huanhuan的最终派生类 -> 因此Panda负责虚基类构造函数调用执行
Panda huanhuan("pooh");
cout<<huanhuan.family_name<<endl;// 输出"Panda"
```
**[3] 最终派生类是相对的**上面的例子中，Bear类和Raccoon类都被用作中间派生类而不是最终派生类，中间派生类相关的继承子树中虚基类的构造函数被抑制。如果Panda类被其他类继承，则它对zooAnimal的构造函数同样被抑制。

**[4] 在虚继承关系中，为合适的类指定多种类型的构造函数 - 最为灵活的方案**
```c++
class Bear: public virtual zooAnimal{
    public:
        Bear(string name, bool onExhibit=true)
            :zooAnimal(name, onExhibit, "Bear"),
            _dance(two_left_feet){}// no.1 当Bear类作为最终派生类调用的构造函数
    protected:// 限制对Bear类私有成员的访问只能通过Bear类域内or继承类进行
        Bear():dance(two_left_feet){}// no.2 当Bear类作为中间派生类调用的构造函数 
};
class Raccoon: public virtual zooAnimal{...};// similarly
class Panda: public Bear, public Raccoon, public Endagered{...};
// 重新定义Panda构造函数 -> 可以省略构建Bear和Raccoon部分
Panda::Panda(string name, bool onExbihit=true)
    :zooAnimal(name, onExhibit, "Panda"),
    Endagered(Endangered::environment, Endangered::critical),
    _sleeping(false){}
```

### 复杂继承关系中ctor和dtor调用顺序
**[1] 含有虚基类的继承树中构造函数调用顺序** 考虑如下图所示的继承关系树：

<center><img src="/img/in-post/cpp_img/inheritance_5.pdf" width="50%"></center>

描述继承关系树的代码如下：
```c++
class Character{...};
class BookCharacter: public Character{...};
class toyAnimal{...};
class TeddyBear:public BookCharacter,public Bear,public virtual toyAnimal{};
```
虚基类查找顺序：**(1)** 编译器按照直接基类在声明中的顺序，检查虚基类的出现情况：首先检查Character类，然后是Bear类，最后是toyAnimal类。**(2)** 在每个继承子树中，按照深度优先的逆序进行检查(根节点-\>叶子节点)：对于BookCharacter子树，先检查Character类，再检查BookCharacter类；对于Bear子树，先检查zooAnimal类，再检查Bear类。<br>
因此TeddyBear类的虚基类构造函数调用顺序是：zooAnimal，toyAnimal。一旦调用了虚基类构造函数，非虚基类构造函数按照声明的顺序被调用：BookCharacter，Bear。其中BookCharacter按照普通多重继承顺序调用：先调用基类Character再调用BookCharacter(多重继承参考本文后续部分进行了解)。

```c++
TeddyBear naonao;// 给定声明 调用的基类构造函数顺序如下 ->
zooAnimal();// no.1 Bear虚基类
toyAnimal();// no.2 直接虚基类
Character();// no.3 BookCharacter非虚基类 
BookCharacter();// no.4 直接非虚基类
Bear();// no.5 直接非虚基类
TeddyBear();// no.6 声明的最终派生类 
```
关于上述声明的析构函数调用顺序：只需分析构造函数的调用顺序，编译器保证析构函数调用的顺序和构造函数调用的顺序相反。

### 复杂继承关系中类成员的可见性(visibility) 
考虑如下的继承树关系：

<center><img src="/img/in-post/cpp_img/inheritance_4.pdf" width="50%"></center>

使用graphviz展示类继承模型图的细节如下。并为Bear类添加一个onExhibit函数，使其覆盖从虚基类zooAnimal继承的onExhibit函数:

<center><img src="/img/in-post/cpp_img/inheritance_6.pdf" width="100%"></center>

现在，当我们进行如下的调用时：
```c++
// no.1 Bear类自己定义了onExhibit() -> 优先使用自己的onExhibit函数
Bear winnie("a lover of honey");
winnie.onExhibit();// 调用的是: Bear::onExhibit()函数
// no.2 Raccoon类没有为自己定义onExhibit() -> 默认使用zooAnimal继承的onExhibit函数
Raccoon meeko("a lover of all foods");
meeko.onExhibit();// 调用的是: zooAminal::onExhibit()函数
```
事实上，Panda类从上述图中定义的继承树中继承的成员可以分成如下几类：**(1)** zooAnimal虚基类成员：name(),family_name()，它们没有被Bear和Raccoon改写。**(2)** 继承自中间基类Raccoon的zooAnimal虚基类定义的成员onExhibit；以及继承自经过Bear改写的成员onExhibit。**(3)** 继承自zooAnimal，但是分别被Bear和Raccoon特化的成员定义(print)。用图形表示三个种类关系如下：

<center><img src="/img/in-post/cpp_img/inheritance_7.pdf" width="60%"></center>

```c++
Panda huanhuan("a lover of bamboo");
// no.1 非虚继承下 产生二义性 Bear::onExhibit? or Raccoon::onExhibit?
// no.2 虚继承下 无二义性 调用Bear::onExhibit not Raccoon::onExhibit
//      原因: Raccoon直接继承虚基类成员 调用优先级小于 Bear类特化的虚基类成员
huanhuan.onExhibit();
// no.3 如果Raccoon也特化了onExhibit成员 则上面调用是二义的 需要用类域操作符进行限定
bool Panda::onExhibit(){// 为Panda类特化onExhibit函数 解决二义性调用问题
    return Bear::onExhibit() && Raccoon::onExhibit() && !_sleeping;
}
```

## Reference
> cpp primer 3rd chapter 18.2 18.4 18.5  <br>

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
