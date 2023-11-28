---
layout: post
title: "cpp - union"
subtitle: 'c++中union相关知识整理'
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: false 
date: 2021-05-24 10:40
lang: ch 
catalog: true 
categories: cpp 
tags:
  - Time 2021
  - cpp fundermentals
---

## union
联合(union)是一种特殊的类，联合可以拥有多个数据成员，但在任何时刻只有一个数据成员可以拥有数值。这是因为联合的所有数据成员在内存中的存储地址相互重叠：每个数据成员都在相同的地址开始储存，并且，分配给联合的内存数量是"包含它最大的数据成员所需的内存数"。<br>
联合(union)和类的根本区别是：联合同一时刻只有一个数据成员可用；而类同一时刻全部的数据成员对类对象都可用。联合声明了多个数据成员，但是每次实例化只用一个；类声明了多个数据成员，全都使用。联合所有的数据成员共享同一段内存；而类的数据成员分别拥有自己的内存存储位置。

### union的定义
这部分通过编译词法解析的过程，对于为什么要使用联合、联合的怎么使用初步了解。对于`int i=0;`这一语句，编译器的词法分析器将其分成token序列的方式传递给解析器。解析器首先分析传入的token，如果根据token序列判断出这是一个声明，则继续分析每个语法单元(token)对应的值(value)。
```c++
int i = 0; // 解析器分析得到token序列：Type ID Assign Constant Semicolon
// 解析器根据token序列判断出是一个声明之后 分析每个token对应的value
// token0: Type          value0: int
// token1: ID            value1: i 
// token2: Assign        value2: =
// token3: Constant      value3: 0
// token4: semicolon     value4: ;
```
关于token_Assign和token_semicolon不需要更多的信息支持，因为这两个token只能拥有一个value选择。而对于Type，可能有很多种选择，但同一时刻只能拥有一个数值，如int、char、double... 这种情况下，符合union的定义规则，因此可以定义成：
```c++
union Token{// 缺省情况下 union成员都是public成员
    char _cval;
    char *_cptr;
    int _ival;
    double _dval;// union中最大的数据成员 因此union大小总是和_dval一致
};
```

### union的使用
**[union和class的共性]** 联合union可以用在任何类class可以被使用的地方。union可以通过`.`或者`->`对联合的数据成员进行访问。union的数据成员同类一样可分为public、protected、private三种类型。union可以拥有自己的构造函数、析构函数及其他成员函数。

**[union和class的不同之处]** 分析：
**(1)** static数据成员存储在全局数据区，编译器为其执行默认初始化，因此无法在定义在union中和其他数据成员共享同一段内存。
**(2)** 引用成员本质是一个指针常量，而且必须被初始化(分配内存)，这样导致union其他数据成员无法使用，退化成了引用类型。
**(3)** ~~如果union数据成员是一个类类型，且该类定义了构造函数、析构函数、拷贝构造函数、赋值运算符的任意一个，则该类类型不能作为union的数据成员。~~ 
**(4)** 上一条规则在c++11中被取消，现在允许定义构造函数、析构函数，union对于内置数据类型和类类型的使用有所区别，详见**[补充内容]**。
**(5)** union执行默认初始化时，只会初始化第一个数据成员。`Token tk = {xxx};`只会初始化第一个char类型数据成员(无论提供的初值能够正确初始化)。

### 匿名的union
匿名union(annonymous union)是一个未命名的union。一但我们定义了一个annonymous union，编译器自动的为该union创建一个未命名对象。
```c++
union{// union默认的scope是public
    char cval;
    int ival;
    double dval;
};
```

**[补充内容]** <br>
**(1)定义限制** 当union包含的是内置类型成员时，编译器会按照顺序依次为他们合成默认构造函数或者拷贝控制成员。但如果union含有类类型成员，并且该类自定义了默认构造函数或者某个拷贝控制成员，则编译器将为union合成对应的成员函数，并将其定义为删除的(默认构造函数or拷贝控制成员=delete)，即编译器禁用了在union中定义默认构造以及拷贝控制成员(**但是允许类类型调用自身的构造函数or拷贝控制成员，这保证了类类型不会被重复的构造or拷贝**)。注意，union析构函数会调用而不是代替类类型析构函数，而且类类型的析构函数不能定义为=delete。<br>
**(2)使用限制** 当union包含的是内置类型成员时，可以通过普通赋值语句改变union中保存的值。当union包含的是类类型时，想要将内置类型改成类类型or想要将类类型改成内置类型，则应当分别调用该类的构造函数以及析构函数。<br>
**(3)总结** 上述两条限制本质上都是在union定义的类类型能够完成union的基本属性的条件下，保证内存管理不泄漏or内存重复释放等问题。

### 使用类来管理union成员
实际生产中，一般采用将匿名union包含在类内作为一个数据成员的方式来管理union。通过这个类可以管理union内内置类型、类类型相关的操作。这种做法的一个直观的出发点是：在union中管理类类型成员有很多限制，构造函数、拷贝控制成员必须定义成`=delete`等，所以这种将匿名union作为类的一个数据成员的方式能够方便对union进行管理。<br>
另外，在设计管理union的类的时候，通常会包含一个enum类型的数据成员(称为union判别式)，用于给union数据中的数据类型打标签。通过这种方式吗，能够即时获取union中保存的数据类型是什么，便于进行管理。
```c++
class Token{
    public:
        // 四个构造函数 需要分别维护一个enum标志位
        Token(char ch):tok(CHAR),_cval(ch){};// no.1 
        Token(char *ptr_ch):tok(P_CHAR),_cptr(ptr_ch){};// no.2 
        Token(int i):tok(INT),_ival(i){};// no.3 
        Token(double d):tok(DOUBLE),_dval(d){};// no.4 
        // 拷贝构造函数
        Token(const Token &t):tok(t.tok){copyunion(t);}
        // std::string类型定义1个默认构造+5个拷贝->必须在union中重复定义并置为=delete
        Token & operator=(char);
        Token & operator=(char*);
        Token & operator=(int);
        Token & operator=(double);

        ~Token(){// 由于Token含有类类型 则必须显示为其定义析构函数
            if(tok==STR){// 通过enum判别式 判断当前Token实例化的内容是否为string
                str.~string();// 调用相应析构函数
            }
        }
    private:
        enum {CHAR, P_CHAR, STR, INT, DOUBLE} tok;// enum定义判别式
        union{// 每个Token类的对象都含有一个未命名union的未命名对象成员
            // 内置类型
            char _cval;
            char *_cptr;
            int _ival;
            double _dval;
            // 类类型 -> 必须进行'特殊处理' 才能正确使用
            std::string str;
        };
        void copyunion(const Token&);// 私有类内tool func
};
int main(){
    Token tk;// 未初始化的Token对象
    Token tk1 = {'a'};// init Token的第一个数据成员_cval 此时其他成员为undefined
    tk._cval = '\n';// 正确 可以通过main访问Token
    tk._cptr = &tk._cval;// 错误 不能通过main访问Token保护成员
    tk._ival = 3;// 错误 不能通过main访问Token私有成员
}
```

### 在Token类中为union成员重载赋值运算符
下面的代码为Token类型重载了赋值运算符，注意针对union类类型，需要进行特殊处理。其他拷贝控制成员操作符的操作方式和下面的函数类似。注意，在使用Token类管理union数据成员时，如果union成员含有类类型，则需要调用placement-new在特定的地址上创建对象。
```c++
class Token{
    ...
}
// 将INT类型赋值给Token类类型
Token & Token::operator=(int i){
    if(tok==STR){str.~string();}// 如果Token存储的是STR 则须先调用string析构
    _ival = i;  tok = INT;// 执行赋值操作 并更新INT标志
    return *this;
}
// 将string类型赋值给Token类类型
Token & Token::operator(const std::string &s){
    if(tok==STR){str=s;}// 如果Token内部存储的是STR 则直接赋值
    else{
        new (&str) string(s);// 使用placement-new在当前地址创建string对象
    }
    tok=STR;// 更新tok标志
    return *this;
}
```

### 为Token类定义拷贝控制函数的tool func
```c++
void Token::copyunion(t.tok){// 将任意类型赋值给非string类型tool func
    switch(t.tok){
        case INT: _ival=t._ival; break;
        case CHAR: _cval=t._cval; break;
        case P_CHAR: _cptr=t._cptr; break;
        case STR: new (&str) string(t.str); break;// using placement-new!
    }
}
```

### 为Token类重载赋值运算符
```c++
Token & Token::operator=(Token &t){
    if(tok==STR && t.tok!=STR){// 如果非string类型赋值给string类型
        str.~string();
    }
    if(tok==STR && t.tok==STR){// 如果string之间的赋值
        str=t.str;
    }
    if(tok!=STR && t.tok==STR){// 如果将string类型赋值给非string类型
        copyunion(t);
    }
}
```

## Reference
> c++ primer 5th chapter 13.1.6 p475 <br>
> c++ primer 3rd chapter 13.7 p549 <br>

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内 <br>
> 8 `<center><img src="/img/in-post/economics_4/xxx.png" width="60%"></center>`
