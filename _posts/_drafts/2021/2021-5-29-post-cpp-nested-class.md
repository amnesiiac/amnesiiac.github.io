---
layout: post
title: "cpp - nested class"
subtitle: 'c++中嵌套类相关知识整理'
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-05-29 21:38
lang: ch 
catalog: true 
categories: cpp 
tags:
  - Time 2021
  - cpp fundermentals
---
## 嵌套类
一个类可以定义在另外的类里面，这样的类被称为嵌套类(nested class)。嵌套类是外层类的一个成员，它的名字只在外层域中可见，在外层域外不可见，因此嵌套类名字不会污染外边名字空间。
```c++
class Node{...};// 类域外部的同名类
class Tree{
    public:
        class Node{...};// 嵌套类 在类域中屏蔽外部的同名变量
        Node *tree;// 被解析为Tree::Node
};
Node *pnode;// 被解析为::Node *pnode
```
### 定义在外层类内部的嵌套类
**[1] 外层类和嵌套类之间的访问关系** <br>
需要首先明确的是：外层类对于定义在其内部的嵌套类没有特殊访问权限，嵌套类对于外层类也没有特殊访问权限。一般情况下，两个类之间的访问必须通过类域、继承类域、类的对象、继承类的对象4种方式进行；由于嵌套类在外层类域中，此时外层类的定义不完全可见，因此无法使用外层域访问操作符直接访问外层域的非静态成员；但是此时嵌套类可以使用外层类的静态成员(一般用作函数默认初值)。嵌套类可以访问外层类的公有的静态成员、类型名(typedef名字、枚举类型名、类名)，因为这些元素在外层类定义还尚未完全可见的时候就能确定占内存大小。
外层类和嵌套类只是作用域包含与被包含的关系。
```c++
class List{
    public:
        static int listval1;
        int listval2;
        enum ListStatus {Good, Empty, Corrupted};// 类型名 枚举类型
        typedef int (*pfunc)();// 类型名 typdef函数指针类型
    private:
        class ListItem{// 嵌套类 - 被隐藏的类
            public:
                ListItem(int val=0);
                void mf(const List &);// 未能获得外层类全部定义 只能使用引用、指针
                void func1(int tmp=List::listval1);// 正确 可以访问static成员 
                void func2(int tmp=List::listval2);// 错误 不能访问non-static
                ListStatus status;// 正确 定义一个枚举类型
                pfunc ptr_func;// 正确 定义一个函数指针ptr_func
        }; 
};
```

使用public、protected、private三种方式对于嵌套类进行修饰可以控制嵌套类型的作用域。嵌套类的访问限定方式同访问限定符修饰其他成员一样。关于访问限定符(access-specifier)可以看这篇[博客](/cpp/2021/04/24/post-cpp-access-specifier/)进行了解。

**[1] 外层类和嵌套类互相声明为友元(一种不好的嵌套类使用方式)** <br>
下面代码通过将外层类声明为嵌套类的友元类的方式，允许外层类访问嵌套类的成员，这种方式不是最好的方法。
```c++
class List{
    friend class ListItem;// 将嵌套类声明为友元类 - 便于嵌套类访问List成员
    public:// 将嵌套类声明为public - 嵌套类型可以在类域、继承类域中、以及它们的对象中访问
        class ListItem{
            friend class List;// 声明为友元类 - 便于List访问ListItem成员
            private:
                ListItem(int val=0);// 单参数构造函数(隐式转化)
                ListItem *next;// 指向同类对象的指针
                int value;
        };
    private:
        ListItem *list;  ListItem *at_end;
};
List::ListItem *ptr_global;// ListItem在外层类中是public限定的 - 可以直接使用
```
**[2] 通过使用private访问限定符+public成员限定符使用嵌套类(一种较好的方式)** <br>
将嵌套类声明为public反问限定有时是不妥的，因为我们可以在外层类域的外部对其进行使用。下面的代码限制了嵌套类的使用域，并且可以舍弃掉友元声明。
```c++
class List{
    public:
        ...
    private:// 限制了只有List的成员&友元具有访问ListItem权限 - 防止内部类型外泄
        class ListItem{
            public:// 将嵌套类成员声明为public
                ListItem(int val=0);
                ListItem *next;
                int value;
        };
        ListItem *list; ListItem *at_end;
};
```
上面的代码通过将嵌套类访问限定符声明为private来控制其使用域，通过将嵌套类成员声明为public来允许外层类对其成员进行访问。<br>
**[3] 关于嵌套类的成员函数、static成员的定义方式**<br>
嵌套类的成员函数的定义和static成员的定义无关访问限定符：嵌套类的public、protected、private三种成员都可以在全局域中进行定义。
```c++
class List{
    public:
        ...
    private:// no.1
        class ListItem{
            public:// no.2
                ListItem(int val=0);
                int value;
                static double dval;
        };
        ...
};
double List::ListItem::dval=3.14;// 嵌套类static成员的初始化
List::ListItem::ListItem(int val){// 通过外层类域访问到嵌套类域 & 完成构造函数定义
    value=val;
}
```
**[4] 嵌套类本身也可以定义在外层类之外** <br>
```c++
class List{
    public:
        ...
    private:
        class ListItem;// 在外层类中给出嵌套类的声明
};
class List::ListItem{
    ...
};
```

### 嵌套类域中的名字解析
本部分在[name-resolution](/cpp/2021/05/29/post-cpp-name-resolution/)博客中也有介绍，内容基本相同。 <br>
**[1] 用于嵌套类定义中的名字(不包括inline函数名、函数缺省实参名)的解析过程** <br>
```c++
enum ListStatus{Good, Empty, Corrupted};// no.3 最后考虑声明在<外层类之前>的名字
class List{
    public:
        ...
    private:
        ...// no.2 再查找外层类中定义在<ListItem类名之前>的名字
        class ListItem{
            public:
                ...// no.1 考虑定义在nested-class内部的名字 => 
                // no.1a 考虑声明在<xxx名字之前>的嵌套类的成员声明 
                // no.1b 考虑在嵌套类内的 <在xxx之前>的出现的 外层类的成员声明
                auto xxx;// xxx的名字解析顺序 no.1a -> no.1b -> no.2 -> no.3
        };
        ...// no.2 声明在nested-class之后的名字不用于名字解析
};
```
**[2] 如果嵌套类只在外层类中声明，定义在外层类域之外，名字解析如下：** <br>
```c++
enum ListStatue{Good, Empty, Corrupted};// no.3 考虑定义在<外层类之前>的名字声明
class List{
    public:
        ...// no.2 考虑<所有>在外层类内的声明
    private:
        class ListItem;// nested-class声明
};
class List::ListItem{// 在外层类外的nested-class定义
    public:
        ...
        // no.1 考虑声明在<xxx名字之前>的嵌套类的成员声明
        auto xxx;// xxx的名字解析顺序名字解析顺序 no.1 -> no.2 -> no.3
};
```


## Reference

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内 <br>
> 8 `<center><img src="/img/in-post/economics_4/xxx.png" width="60%"></center>`
