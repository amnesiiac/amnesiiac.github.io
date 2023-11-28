---
layout: post
title: "cpp - class template"
subtitle: '类模版相关知识整理'
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: true
date: 2021-05-11 22:50
lang: ch 
catalog: true 
categories: cpp 
tags:
  - Time 2021
  - cpp fundermentals
---

## 类模版
类模版是用来生成类的蓝图的，它是一种类的创建规范，描述了类中有1个或多个类型、值被参数化。类模版可以被看作是类的类型。

### 为什么要定义类模版
本部分内容通过自定义一个行为类似queue机制的类来展示类模版的定义细节。一个普通行为类似queue的类的实现如下所示：
```c++
class Queue{// no.1 normal class def
    public:
        Queue();
        ~Queue();
        T& remove();// fake pop()
        void add(const T &);// fake push()
        bool isempty();
        bool isfull();
};
```
上述代码的问题在于：我们不能够在设计Queue类时确定容器内部数据类型T应该采用什么类型。如果T被设置成int，则针对float、double等其他数据类型的队列需要重新设计，由于类不支持重载机制，因此我们可能需要重新定义IntQueue类、FloatQueue类、DoubleQueue类。但是这种做法导致**直观上**的代码膨胀问题，即整体代码不够简洁而且不易于管理。采用类模版机制，可以避免**直观上的**代码膨胀，并且便于维护？(可能是个伪命题)。下面改写上面的代码：
```c++
template<typename T> class Queue{// no.1 template class def
    public:
        Queue();
        ~Queue();
        T& remote();
        void add(const T&);
        bool isempty();
        bool isfull();
};
// template class usage
Queue<int> int_que;
Queue<string> str_que;
Queue<complex<double>> complex_que;
```
### 类模版的定义方法(注意事项)
**[1] 模版类型参数** 类模版可以有一个或者多个类型参数，使用多个类型参数时需要注意每个类型参数都需要使用typename或class关键字：
```c++
template<typename T> class Queue{};// single type
template<typename T1, typename T2, typename T3> class Queue{};// multi type
```
**[2] 模版非类型参数** 非类型参数由普通参数声明构成，参数中的数值代表了模版定义中的某个常量(如Buffer类模版非类型参数表明了它的大小)：
```c++
template<typename T, int size> class Buffer{};
```
**[3] 模版参数类型可以隐藏外部参数**
```c++
typedef double T;// no.1
template<tyepname T> class Queue{// no.2
    public:
        ...
    private:
        T item;// T表示模版类型 而不是外部typedef定义的类型
};
```
**[4] 模版参数名不能在类内被typedef实例化**
```c++
template<typename T> class Queue{
    public:
        ...
    private:
        typedef double T;// wrong! 类模版参数名不能被typedef实例化为成员类型double
        T item;
};
```
**[5] 同一个类的模版参数名字不能重复，不同类的模版参数名字可以重复**
```c++
template<typename T, typename T> class Queue{};// no.1 forbid repitition
template<typename T> class Queue1{};// no.2 allow repitition
template<typename T> class Queue2{};// no.2 allow repitition
```
**[6] 模版前置声明和模版具体定义中的参数名字可以不同：下面no.1&no.2声明引用的类模版的定义都是同一个类(no.3)**
```c++
template<typename T> class Queue;// no.1 declaration -> ref to no.3
template<typename U> class Queue;// no.2 equivalent declaration -> ref to no.3
template<typename Type> class Queue{};// no.3 template class def
```
**[7] 类模版的类型参数or非类型参数都可以有缺省实参，在类模版实例化过程没有提供实参时备用**
```c++
template<typename T=double, int size=10> class Queue;
```
**[7.1] 类模版可以有多个声明，并且后续的声明可以附加缺省实参**
```c++
// 通过no.1+no.2 为Queue类模版两个参数提供缺省值
template<typename T, int size=10> class Queue;// no.1 从右侧开始添加缺省值or类型
template<typename T=double, int size> class Queue;// no.2 补充typename的缺省类型
```
**[7.2] 提供缺省实参的规则同函数提供缺省实参的规则一致(必须集中在参数表右侧放置)** 
```c++
// 想要为T提供缺省类型 则它右边所有的参数都需要预先有缺省值(同一个类的多个声明可以互相补充)
template<typename T=double, int size> class Queue;// wrong! size尚未提供缺省值
```
**[8] 类模版在定义&应用时的书写方式**
```c++
// no.1 类模版在定义时 可以省略<T>模版参数 
template<typename T> class Queue{
    public:
        Queue(const T &);// Queue<T>(...)
    private:
        T item;
        Queue *next;// right! 定义时可以省略模版参数列表
};
// no.2 将类模版用在函数模版中
template<typename T> void display(Queue<T> &q){
    Queue<T> *ptr_q = &q;// right!
    Queue *ptr_qq = &q;// wrong! 在使用时不能省略模版参数列表
}
// no.2 将类模版应用在其他类模版中
template<typename T> class OtherQueue{
    public:
        ...
    private:
        Queue<T> *ptr1;// right!
        Queue *ptr2;// wrong! 在使用时不能省略模版参数列表
};
```

### 类模版的实例化(使用方式)
**[一：类模版中类类型参数的使用方式]** <br>
**(1)**类模版使用时，参数列表不能省略，需要显示指定模版实参。不同于函数模版(参数列表可以推演)，在类模版实例化的过程中，编译器无法根据类模版的上下文环境推演出类模版的参数列表。
**(2)** 类模版使用范围：凡是能够使用非模版类型的地方都可以使用类模版。
```c++
typedef complex<double> d_cmp
extern Queue<d_cmp> func(Queue<d_cmp> &, Queue<d_cmp> &);// no.1 return&para 
bool (Queue<double>::*func_ptr)() = nullptr;// no.2 func_ptr 指向Q<d>的成员函数
Queue<char*> *qch_ptr = static_cast<Queue<char*>>(0);// no.3 用于强制类型转化
```
**(3)** 类模版用于声明指针和引用不会实例化：声明类型的指针&引用不需要提供类的定义。
```c++
void func(Queue<int> &i){
    Queue<int> *ptr = i;// 不涉及Queue<T>的实例化 不需要看到类的定义
}
```
**(4)** 类模版用于声明类的对象时进行实例化， 并且需要看到类的定义。
```c++
void func(){
    Queue<int> i;// 声明一个类对象 Queue<T>被实例化 需要看到类模版的完整定义
    Queue<int> *ptr = &i;
    ptr->add(24);// 当指针被解引用or调用相关对象的成员函数->实例化
}
```
**[二：类模版中的非类型参数的应用方式]** <br>
**[1]** 非类型参数必须使用常量表达式进行实例化，即用于实例化的式子必须在编译时被计算出结果。
```c++
template<typename h, typename w> class Screen{// no.1 非类型参数用法和函数参数类似
    public:
        Screen():_height(h),_width(w),_cursor(0),_screen(h*w, '#'){}
    private:
        string _screen;
        string::size_type _cursor;
        short _height;  short _width;
};
typedef Screen<24,80> huaweiscreen;// <24,80> 常量表达式用于非类型参数的绑定
huaweiscreen hw01;// 实例化
```
如上述代码所示：绑定给非类型参数的必须是常量表达式，下面的例子将会导致编译错误。
```c++
template<int *ptr> class BufPtr{...};// 非类型参数ptr指针
BufPtr<new int[24]> i;// wrong! new-expression在程序运行时才分配具体的指针值
```
**非const对象的值一般不是常量表达式，因此不能用于类模版的非类型参数的绑定**。但是存在例外：名字空间中任何对象的(const&non-const)地址是一个常量表达式，可以绑定非类型参数。名字空间中sizeof(对象)的值是一个常量表达式，可以绑定非类型参数。
```c++
template<int size> class Buf{...};
template<int *ptr> class BufPtr{...};
int size_val=1024;
const int c_size_val=1024;
Buf<size_val> b0;// wrong! non-const-expression bindings!
Buf<c_size_val> b1;// right! const expression bindings.
Buf<sizeof(size_val)> b2;// right! sizeof returns const-expression.
Buf<&size_val> b3;// right! & returns const-expression.
```
**(2)** 非类型模版只和常量表达式最终值相关，和绑定的形式无关。下面三个类型实例都绑定到同一个模版的实例Screen<24,80>中。
```c++
const int height = 24;  const int width = 80;
Screen<height, width> s0;// no.1
Screen<2*12, 40*2> s1;// no.2 
Screen<6+6+6+6, 20*2+40> s2;// no.3 
```
**(3)** 非类型模版在被赋值的过程中允许发生类型转换，这种转换是函数实参实例化的转化的一个子集。<br>
**(3.1)** 左值转化：左值->右值、数组转化成指针、函数到指针的转换。
```c++
// no.1 lvalue->rvalue conversion cpp primer 3rd p383:
// 当函数需要一个按值传递的实参 而该实参是一个lvalue时 发生lvalue->rvalue转化
template<int i>class Buf{...};// template with lvalue
int j;  Buf<j> obj;// instantiate by type conversion 
// no.2 arr->pointer conversion
template<int *ptr> class BufPtr{...};
int arr[10];  BufPtr<arr> b1;
// no.3 func->func_pointer conversion
template<void (*ptr)(int)> class FuncPtr{...};
void func(int);
FuncPtr<func> b2;
```
**(3.2)** 限定修饰的转化(const、volatile)。
```c++
template<const int *> class Ptr{...};
int i;  Ptr<&i> obj;// int* -> const int* conversion
```
**(3.3)** 类型提升(promotion)。
```c++
template<int height, int width> class Ptr{...};
short i; int j;
Ptr<i, j> obj;// short->int promotion
```
**(3.4)** 允许整形转换 & 禁止函数类型转化。
```c++
extern void func1(char*);
extern void func2(void*);
typedef void (*funcptr)(void*);
const unsigned int x=1024;// 1024是常量表达式=>x是常量表达式

template<class T, unsigned int size, funcptr f> class Arr{...};

// no.1 right! no conversion needed! ui->ui
Arr<int, 1024U, func2> obj;
// no.2 wrong! there's no converison between function & func_pointer
Arr<int, 1024U, func1> obj;
// no.3 right! int->unsigned int
Arr<int, 1024, func2> obj;
// no.4 right! no conversion needed! x=常量表达式 const ui->ui
Arr<int, x, func2>
```
**(3.5)** 在模版非参数类型匹配中，0只能被转化成int类型，不能转化成指针。
```c++
template<int *ptr> class Buf{...};
// no.6 wrong! 0 is implicitly convert to int but not nullptr!
Buf<0> obj;
```

### 类模版的成员函数(函数模版)
**[1] 在模版类内部定义成员函数：**
```c++
template<typename T> class Queue{
    public:
        Queue():_front(0),_back(0){}// ctor def as inline
    private:
        ...
};
```
**[2] 在模版类外部定义成员函数：**
```c++
template<typename T> class Queue{
    public:
        Queue();// ctor declare
    private:
        ...
};
// 在类外定义成员函数: [所属模版声明] + [inline] + [类模版::成员函数名]
template<typename T> inline Queue<T>:Queue(){// ctor def as inline
    _front = _back = 0;
}
```
**[3] 类模版的成员函数本身也是一个函数模版。**这可以从[2]中定义模版类的成员函数的行为中看出。模版类被实例化时，模版类的成员函数并不会实例化；只有在相应类的对象被调用或者取地址时才能被实例化，实例化的类型就是相应类模版被实例化的类型。
```c++
Queue<string> q;// 调用Queue<string>类的ctor 其构造函数被实例化为string类型
```

### 类模版的友元
**[1] 非模版友元类&非模版友元函数**
```c++
class Queue{
    void func1();
};
template<typename T> class QueueItem{
    friend class Queue;// no.1 非模版普通类
    friend void func2();// no.2 非模版普通函数 
    friend void Queue::func1();// no.3 非模版类的普通函数 - 要求Queue定义可见
};
```
**[2] 绑定的友元类模版&绑定的友元函数模版(友元的模版类型和当前定义的模版类的模版参数绑定)**
```c++
template<typename T> class TestClass{...};
template<typename T> void func1(QueueItem<T>);
template<typename T> class Queue{
    void func2();
};
template<typename T> class QueueItem{
    friend class TestClass<T>;// no.1 绑定的模版类
    friend void func1<T>(QueueItem<T>);// no.2 绑定的函数模版
    friend Queue<T>::func2();// no.3 绑定的模版类的成员函数 - 要求Queue定义可见 
};
```
**[3] 非绑定的友元类模版&绑定的友元函数模版(友元的模版类型和当前定义的模版类的模版参数无关)** 
```c++
template<typename T> class QueueItem{
    template<typename U> friend class TestClass;// no.1 非绑定的模版类
    template<typename U> friend void func1(QueueItem<U>);// no.2 非绑定的函数模版
    template<typename U> friend void Queue<U>::func2();//no.3非绑定模版类成员函数
};
```
**[Conclusion]** <br>
**[1]** 绑定的类模版的友元类声明无需通过*template\<typename U\>*的方式重新声明模版的具体类型，非绑定的类模版的友元类声明需要声明模版类型。<br>
**[2]** 将一个类模版声明为另一个类模版的友元时，采用绑定的方式还是非绑定的方式？解析：绑定方式和非绑定方式取决于模版类和友元类、友元函数之间的逻辑关系。当他们实例化的类型强制要求保持一致时，只能使用绑定方式声明友元。

### 类模版的静态数据成员
下面的程序描述了如何在模版类版中声明静态数据成员，并且为这个模版类定义了*operator-new*和*operator-delete*。不过，下面的程序的真正价值不在于模版类定义的形式，而在于其提供了一种*operator-new*和*operator-delete*的定义思路。<br>
**[1]** 类模版中静态数据成员的声明、定义方式。
```c++
template<typename T> class QueueItem{
    private:
        // new & delete op overload
        void * operator new(size_t);// 创建类型<T>大小的QueueItem实例 
        void operator delete(void *, size_t);
        // static members declare 
        static QueueItem *mem_list;// 指向mem_list的首地址 
        static const unsigned QueueItem_chunk;// 一次性分配的类实例的个数
};
// static member def 
template<typename T> QueueItem *QueueIterm<T>::mem_list = nullptr;
template<typename T> const unsigned QueueItem<T>::QueueItem_chunk = 24;
```
模版类中的静态成员和普通类中的静态成员不同，它本身是一个模版。模版类的静态成员在定义时不会引起内存分配。只有静态成员被实例化时才会分配内存：
```c++
int val0 = QueueItem::QueueItem_chunk;// wrong! 静态成员未实例化
int val1 = QueueItem<int>::QueueItem_chunk;// right! 实例化成static int
int val2 = QueueItem<short>::QueueItem_chunk;// right! 实例化成static short
```
**[2]** 使用list底层结构来管理类模版对象内存的*operator-new*实现思路。主要想法是：一次性申请*QueueItem_chunk\*sizeof(QueueItem)*数量的内存，并使用list链表数据结构进行管理。<br>
*operator-new*做的工作是：维护一个**预分配**的内存链表(如果没有则创建它)，这个链表用来存储可以被初始化的内存位置，链表中每个内存位置都存放一个QueueItem类实例。mem_list指针指向下一个分配内存链表中的位置，op-new返回内存链表中的首指针。
```c++
// overload operator-new def (只分配内存 不涉及对象构建)
template<typename T> void* QueueItem<T>::operator new(size_t size){
    QueueItem<T> *p;// 指向每个QueueItem的实例
    if(!mem_list){// 如果尚未创建mem_list -> QueueItem<T>没有实例
        size_t chunk = QueueItem_chunk * size;// set mem chunk size
        // reinterpret_cast to convert raw mem into certain type
        mem_list = p = reinterpret_cast<QueueItem<T>*>(new char[chunk]);
        // construct a mem_list
        for(; p!=&mem_list[QueueItem_chunk-1]; ++p){
            p->next = p+1;
        }
        p->next = nullptr;// set tail pointer
    }
    p = mem_list;// set p
    mem_list = mem_list->next;// 指向mem list中下一个QueueItem实例存放位置
    return p;// 返回用于构建对象的内存位置的指针
}
```
**[3]** 使用list底层结构来管理类模版对象内存的*operator-delete*实现思路。<br>
delete主要做的工作是：将一块孤立内存(曾经存放一个已经被析构的QueueItem实例)加入到mem_list中(可构造内存链表)。更新维护的[可构造内存链表]的首地址。
```c++
// overload operator-delete def (和op-new互补 用于内存的释放 不涉及对象析构)
template<typename T> void QueueItem<T>::operator delete(void *p, size_t){
    static_cast<QueueItem<T>*>(p)->next = mem_list;// 将析构过的mem加入mem_list 
    mem_list = static_cast<QueueItem<T>*>(p);// 更新mem_list首指针
}
```
### 类模版中的嵌套类
QueueItem类用来专门辅助Queue类的创建，无需为其他任何类提供服务，为了实现Queue和QueueItem的这种关系，有如下两种设计策略：<br>
**[设计策略1]**将QueueItem类的构造函数声明为private，将Queue声明为QueueItem的友元模版类，这样Queue和Queue Item成员可以创建QueueItem对象。<br>
**[设计策略2]**：将QueueItem定义在Queue的private嵌套类，此时QueueItem类只对Queue和其嵌套类可见；如果将嵌套的QueueItem类成员声明为public，则QueueItem类不必声明为Queue的友元。<br>
**[总结]**：使用嵌套类的方式相比将QueueItem声明为Queue的友元的方式能够更好的描述原始语义，并且在程序组织上更加优雅的表述类之间的关系模型。
```c++
template<typename T> class Queue{
    public:
        ...
    private:
        class QueueItem{// QueueItem是一个'绑定'的类模版 可直接使用参数类型T
        // 可以理解成:类模版template<>具有向内渗透性 嵌套类默认继承了外围类的<T>参数
            public:
                QueueItem{T val}:item(val),next(nullptr){...};
                T item;
                QueueItem *next;
        };
    // 在Queue类域内定义嵌套类型指针(两者绑定模版<T>)无需声明模版实参QueueItem<T>
    QueueItem *_front, *_back;
};
```
**[关于嵌套类的实例化]** QueueItem\<T\>实例化与否和外围类Queue\<T\>无关，只有当使用QueueItem\<T\>的对象或者通过指向对象的指针解引用间接操作对象时，才会被实例化。这一点嵌套类和友元模版两种方法行为类似。

### 类模版中的枚举类型和typedef类型名
枚举类型是一种特殊的类型，定义在类模版中的枚举类型也可以看成是一个嵌套类，对应的类模版是外围类。类模版中声明枚举类型以及typedef的使用方式如下：
```c++
template<typename T, int size> class Buffer{
    public:
        enum BufVal{last=size-1, buf_size};
        typedef T BufType;
        BufType arr[size];
};
```
在上述代码中，没有通过*buf_size=xxx*的方式设置缓存的大小，而是通过模版非类型参数设置数值：
```c++
Buffer<int, 512> smallbuf;// 设置了一个数据类型为int 含有512个元素的buffer
Buffer<int, 1024> mediumbuf;// 设置了一个数据类型为int 含有1024个元素的buffer
```
引用类模版中的枚举类型或者typedef类型需要声明模版参数类型：
```c++
Buffer::BufVal bfv0;// wrong! 不能确定类模版类型 
Buffer<int, 512>::BufVal bfv1;// right!
```

## 成员模版(函数模版or类模版作为一个类or模版类的成员)
**[1] 成员模版的定义**。函数模版or类模版可以是普通类的成员，也可以是类模版的成员。
```c++
template<typename T> class Queue{
    private:
        // 类模版 - 需要声明自己的模版参数(和类模版参数不绑定)
        template<typename U> class QueueMember{
            U member0;// 使用自己的模版参数
            T member1;// 使用外部类模版的模版参数
        };
    public:
        // 函数模版 - 需要声明自己的模版参数(和类模版参数不绑定)
        template<typename Iter> void assign(Iter first, Iter last){
            while(!empty){
                remove();// call Queue<T>::remove()
            }
            for(; first!=last; ++first){
                add(*first);// call Queue<T>::add(const T&)
            }
        }
};
```
上述代码中，在一个类模版Queue中声明了一个成员类模版QueueMember和一个函数模版assign，这意味着，对于任何一个类模版的实例如Queue\<int\>，有无数个成员类QueueMember\<U\>实例和assign\<Iter\>与之匹配。

**[2] 成员模版的实例化**
```c++
int main(){
    Queue<int> q;
    int arr[4]={0, 1, 2, 3};
    q.assign(arr, arr+4);// 成员函数模版被实例化<int*[4]>
    vector<int> vec(arr, arr+4);
    q.assign(vec.begin(), vec.end());// 成员函数模版实例化<vector<int>::iterator>
}
```
**[3] 在类模版成员实例化时允许发生类型转化** 但是转化后的类型在成员函数中是否能够正确运行是程序员的任务。
```c++
class SmallInt{
    public:
        SmallInt(int val=0):_value(val){}// no.1 int->SmallInt
        operator int(){return _value;}// no.2 SmallInt->int
    private:
        int value;
};
int main(){
    Queue<int> q;
    // no.1 right! SmallInt可以转化成int类型 assign函数中add调用不会出错
    vector<SmallInt> vec;
    // Queue<int>::assign(vector<SmallInt>::iterator, ...)
    q.assign(vec.begin(), vec.end());
    // no.2 wrong! int*类型不能转化成int类型 assign函数中的add调用会报错 
    list<int*> lst;
    // Queue<int>::assign(list<int*>::iterator, list<int*>::iterator)
    q.assign(lst.begin(), lst.end());
}
```
**[4] 任何成员函数都可以被定义成函数模版**
```c++
template<typename T> class Queue{
  public:
    // 构造函数定义为成员函数模版 - 可以使用非绑定参数
    template<typename Iter> Queue(Iter first, Iter last):_front(0),_back(0){ 
        for(; first!=last; ++first){
            add(*first);
        }
    }
};
```
**[5] 成员函数模版可以被定义在类内or类外or其外围类中**
```c++
template<typename T> class Queue{
    private:
        template<typename U> class TestClass;
    public:
        template<typename Iter> void assign(Iter first, Iter last);
};
template<typename T> template<typename U> /* 模版类的类模版成员 */
class Queue<T>::TestClass<U>{
    U member0;
    T member1;
}
template<typename T> template<typename Iter> /* 模版类的函数模版成员 */
void Queue<T>::assign<Iter>(Iter f, Iter l){
    while(!is_empty()){
        remove();
    }
    for(; f!=l; ++f){
        add(*f);
    }
}
```
注意，类模版的模版成员定义中的模版参数不需要和声明时的参数相同，只需保证形式上一致即可。下面定义同样指向上述声明：
```c++
// 一个等价的assign函数定义 - 替换了模版参数形式 不改变实质
template<typename NewT> template<typename NewIter>
void Queue<NewT>::assign<NewIter>(NewIter f, NewIter l){...}
```

## 类模版和编译模式
编译器在执行类模版实例化之前必须要看到类模版的定义。由于类模版可能会在程序中的各个文件中进行实例化，因此，类模版的定义应该放在头文件中。<br>
另外，类模版的成员函数、静态数据成员、嵌套类的行为和模版类类似，它们也都是模版。但是这些类模版的成员并不是在类模版实例化(指定类模版参数类型)的同时进行实例化，而是被类模版的对象调用、被类模版对象的指针or引用调用时才进行实例化。

### 包含编译模式
在包含编译模式下，类模版的成员函数、静态成员的定义必须包含在将它们实例化的所有文件中。<br>
因此：**(1)** 类模版应当定义在头文件中，被使用它的程序文件所包含。**(2.1)** 类模版的成员函数&静态数据成员如果定义在类内(inline)，则自然能够满足上述要求。**(2.2)** 类模版的成员函数&静态数据成员如果定义在类外，则这些定义应当包含在类模版定义同一个文件中。
```c++
// ---------- dog.h ----------
template<typename T> class Dog{
    template<typename U> void func1(){...};// no.1 定义在类模版内 - 包含编译模式
    template<typename V> void func2();// no.2 定义在类模版外 - 包含编译模式
};
// no.2 需要和Dog定义在同一头文件下
template<typename T> template<typename V> void func2(){...}
```
包含编译模式具有一些特性需要谨慎使用：**(1)** 部分类模版成员函数结构复杂，代码庞大。将其定义在类模版定义同一个头文件下，可能导致在多处包含头文件的地方进行多次编译从而增加编译时间。**(2)** 将类模版的成员函数接口实现在类模版定义中，在用户不想知道or程序员希望隐藏这些实现细节的情况下，不适用。

### 分离编译模式
分离编译模式下，类模版及其inline函数定义都放在头文件中，非inline函数和静态数据成员被放在程序文本文件中。<br>
**[1] 将类模版中所有非inline成员&静态成员定义为可导出。** 下面的代码简单展示了类模版的分离编译定义模式：
```c++
// ---------- Queue.h ---------- 头文件
export template<typename T> class Queue{// export关键字将Queue定义为'可导出的'
    public:
        Queue(){};// inline函数 在类模版头文件中定义
        T & remove();// 类模版的非inline成员函数
        void add(const T &);
};
// ---------- Queue.c ---------- 类模版文本文件 - 对user.c不可见
#include <Queue.h>
template<typename T> T& Queue<T>::remove(){...};
template<typename T> void Queue<T>::add(const T&){...};
// ---------- user.c ---------- 用户程序文本文件
#include <Queue.h>
int main(){
    Queue<int> *ptr_q = new Queue<int>;
    int val;  ptr_q->add(val);
}
```
上述代码中，虽然Queue.c成员实现文件在用户程序文件中不可见，但是用户程序仍然可以调用定义在Queue.c实现文件中的类模版成员函数add()。为了确保上述分离编译的特性能够实现，Queue类模版定义前需要使用export关键字将其声明为**可导出的**。<br>
**[2] 将类模版中部分非inline函数&静态成员定义为可导出的。**
上述例子将类模版中所有的函数成员、静态数据成员定义成可导出的。有些情况下，我们只希望类模版中个别声明的成员可导出，则可按如下方式使用export：
```c++
// ------- Queue.h ------- 类模版头文件(用于类模版&其inline部分&不可导出的部分的定义)
template<typename T> class Queue{
    public:
        T& remove();
        void add(const T&);
};
// remove未定义成可导出的 不能在Queue.c中定义!
template<typename T> T& remove(){...};
// ------- Queue.c ------- 类模版文本文件(用于管理exported成员函数&静态成员)
export template<typename T> void Queue<T>::add(const T&val){...};
```
**[3] export的特性及其使用的注意事项：**可导出的类模版具有如下特性：当它的成员函数实例或静态数据成员实例被使用时，编译器只要求类模版的定义，而类模版成员函数的定义or静态成员的定义可以对调用类模版的程序文本不可见。另外，关键字export虽然在类模版定义处声明，但它只对类模版中的成员函数or静态成员生效。export关键字是针对编译单元(module)之间的协同而提出的。<br>
如果一个类模版的成员函数or静态成员被定义成可导出的，则如果在不同的编译单元中多次定义这个可导出的成员可能导出不确定的编译器行为：**1** 可能产生链接错误：指出一个类模版的同一个成员提供了多个定义。**2** 编译器可能为同一个类模版的实例多次实例化该exported成员。这导致链接错误：重复的模版实例定义。**3** 编译器只使用其中一个定义，忽略其他的定义。<br>
更多关于export关键字相关知识整理，详见[博客](/cpp/2021/06/10/post-cpp-export/)中的内容。另外：不是所有的c++编译器都支持分离编译模式。

### 显式实例化声明(允许程序员控制模版实例化发生的时间)
**[Problem:]** <br> 在包含编译模式下，类模版成员的定义在使用其实例的所有程序文本处都可见，编译器具体何时何地对这些定义进行实例化无法精确知晓。甚至一些编译器对于同一个类模版实参(如Queue\<int\>)多次实例化其成员模版的定义，然后选择一个用于使用，其他的定义被忽略；虽然一个成员被实例化一次or多次**可能**不会影响程序结果，但是编译程序的时间会显著增加。<br>
**[Solve]** <br>
使用显式实例化声明，可以指定在当前声明位置处，对类模版进行特定类型的实例化：
```c++
// --------- user.c ---------- 用户程序文件
#include <Queue.h>
// 显式实例化声明类模版 - 其所有成员(也是模版)也被实例化 =>
// 因此显示实例化声明要求：声明前类模版&其所有成员定义可见!
template class Queue<int>;
```
**[Another Problem]** <br> 但是上述代码有个问题：如果其他程序文本同样使用了同一个类模版的实例(如Queue\<int\>)，该如何告诉编译器，这个类模版的类型(如Queue\<int\>)已经实例化过了？(确保整个类模版的特定类型(如int)在程序中只实例化一次) <br>
**[Solve]** <br>
这个解决办法在[函数模版](/cpp/2021/05/11/post-cpp-function-template/)中已经讨论过：使用**抑制隐式编译选项**来禁止编译器隐式的对类模版进行实例化，所有的实例化的位置、时间都交给c++程序员来完成😥。这样相当于使用template机制仅仅是为了**减少直观上的代码膨胀，并不能减轻程序员在泛型设计中思维层面的burden**。<br>
因此，在实际使用template过程中，我推荐先不借助template机制进行代码实现，然后将需要template机制的东西抽离出来，使用**显式实例化+抑制隐式实例化**来减少直观代码膨胀，which i think conforms to the template original design principle。

## 类模版的特化
实例化针对的是不同的模版参数类型，而显式特化针对的是不同的模版参数类型以及不同的模版函数体执行过程。当一个函数模版本身函数体执行不能完成某项工作时，采用特化的手段修改其模版函数体执行定义以达到目的。<br>
**[1] 什么时候需要使用显式特化** 下面给出本小节内容的代码基础，并基于下面的代码展示何时需要进行显式特化，以及显式特化如何定义：
```c++
template<typename T> class Queue{
    public:
        T min();// 返回Queue中的最小值
        T max();// 返回Queue中的最大值
};
template<typename T> Queue<T>::min(){
    assert(is_empty());// check
    T min_val = _front->item;// init
    for(QueueItem* ptr=_front->next; ptr!=nullptr; ptr=ptr->next){
        if(ptr->item<min_val){// 注意 模版类型T需要定义operator()< 否则编译错误
            min_val = ptr->item;
        }
    }
    return min_val;
}
template<typename T> Queue<T>::max(){...}// omit 
```
上述代码中涉及模版类型的比较，即模版类型必须定义operator()<操作符，否则编译出现错误。假设Queue类模版参数类型为自定义类型LongDouble，其定义如下：
```c++
class LongDouble{
    public:
        LongDouble(double val):_value(val){}
        bool isless(const LongDouble&);// 没有重载operator()< 用isless比较大小
    private:
        double _value;
};
```
对于上述情况，如果类模版类型为Queue\<LongDouble\>类型，由于LongDouble类型没有重载比较操作符，编译会报错，为了能够在不改变LongDouble类型代码还能应用Queue类模版，可以转而为类模版成员函数min、max定义显式模版特化。

**[2] 显式特化定义min函数的具体方式** 特化的代码如下：
```c++
template<> Queue<LongDouble>::min(){
    assert(is_empty());
    LongDouble min_val = _front->item; 
    for(QueueItem<LongDouble> *ptr=_front->next; ptr!=nullptr; ptr=ptr->next){
        if(ptr->item.isless(min_val)){// min函数模版显式特化 用isless代替op()<
            min_val = ptr->item;
        }
    }
    return min_val;
}
template<> QueueItem<LongDouble>::max(){...}// omit
```
**[3] 类模版成员函数&普通成员函数在特化形式上的不同** 两者代码形式对比如下： 
```c++
// no.1 普通函数模版定义
template<typename T> T max(T t1, T t2){
    return (t1>t2? t1:t2);
}
// no.1 普通函数模版的显式特化 - [函数名]<特化类型>
template<> const char* max<const char*>(const char *s1, const char*s2){
    return (strcmp(s1,s2)>0? s1:s2);// 通过函数模版特化进行重新定义
}
// no.2 类模版及其成员函数定义
template<typename T> class Queue{
    public:
        T min();
};
// no.2 类模版的成员函数(也是一个模版)的显式特化 - [类名]<类型名>::[函数名]
template<typename T> Queue<T>::min(){...}// 显式特化声明
```
上述no.1和no.2特化的形式中，\<类型名\>的位置有所不同：一个模版类型属于函数本身，一个模版类型属于类型。

**[4] 类模版成员函数特化在程序中的代码组织方式** 上述代码中类模版的成员函数min()&max()不是inline函数，因此它们的定义不能放在定义类模版的头文件中。处理方法是：将成员函数显式特化的声明放在头文件中，将显式特化的定义放在程序文本文件中：
```c++
// ---------- Queue.h ---------- 显式特化声明
template<typename T> class Queue{...};
template<> LongDouble Queue<LongDouble>::min();
template<> LongDouble Queue<LongDouble>::max();
// ---------- user.c ---------- 显式特化定义 
template<> LongDouble Queue<LongDouble>::min(){...}
template<> LongDouble Queue<LongDouble>::man(){...}
```
**[5] 整个类模版特化声明&定义** 某些情况下，整个类模版的定义对于模版参数的某种类型不适用，一种解决方案是：将其所有的成员分别进行显式特化，然而这种方式从代码上很难以理解；另一种方式是：为整个类显式实例化。
```c++
#include<Queue.h>// [注意] 显式特化之前必须要'看到'通用的类模版声明(无须看到定义)
template<> class Queue<LongDouble>{
    public:
        Queue<LongDouble>(); 
        ~Queue<LongDouble>();
        LongDouble & remove();
        void add(const LongDouble &);
        bool is_empty()const;
        LongDouble min();
        LongDouble max();
    private:
};
// 显式特化的类的成员函数特化定义: 
// 1: inline函数定义在头文件中 2: 非inline函数声明在头文件中 定义在程序文本文件中
LongDouble Queue<LongDouble>::Queue(){...};// def for inline
LongDouble Queue<LongDouble>::min();// declare for non-inline
```
注意，整个类模版被显式特化修饰符**template<>**修饰之后，其成员函数的定义不再能使用特化定义修饰符修饰。

**[6] 同一个模版参数类型，既然声明了特化，就不要使用实例化定义。一旦定义了模版特化(如Queue\<LongDouble\>)，则须在每个使用Queue\<LongDouble\>的程序文件中声明该特化。**<br>

**错误的做法**
```c++
// ---------- File1.c ----------
#include<Queue.h>
void readin(Queue<LongDouble> *ptr){
    ptr->add();// no.1 使用Queue<LongDouble>实例化定义 
}
// ---------- File2.c ----------
#include<QueueLongDouble.h>
void readin(Queue<LongDouble> *);// 显式特化的声明
int main(){
    Queue<LongDouble> *ptr = new Queue<LongDouble>;
    readin(ptr);// no.2 使用Queue<LongDouble>特化定义 
}
```
**正确的做法**
```c++
// ---------- File1.c ----------
#include<QueueLongDouble.h>// 特化的头文件中已经引入了通用模版定义
void readin(Queue<LongDouble> *);// 显式特化声明
void readin(Queue<LongDouble> *ptr){
    ptr->add();// no.1 使用Queue<LongDouble>特化定义
}
// ---------- File2.c ----------
#include<QueueLongDouble.h>// 特化的头文件中已经引入了通用模版定义
void readin(Queue<LongDouble> *);// 显式特化的声明
int main(){
    Queue<LongDouble> *ptr = new Queue<LongDouble>;
    readin(ptr);// no.2 使用Queue<LongDouble>特化定义 
}
```

## 类模版的部分特化
对于一个含有多个模版参数的类模版，有时希望只为一个特定的模版实参or一组特定的模版实参进行特化处理，这种操作称之为partial specialization.
```c++
template<int height, int width> class Screen{...};// 类模版通用定义
// 部分特化定义形式 =>
template<int height> class Screen<height, 80>{// 类模版的部分特化的定义
    public:
        Screen();
    private:
        string _screen;
        string::size_type _cursor;
        // short _width; => 由于特化的类模版指定了width 无须使用重复定义
        short _height;
        // 为宽度为80的屏幕特化相应的算法
        ...
};
```
**部分特化的类模版的隐式实例化方式**
```c++
Screen<24,80> AppleHDR;// 使用部分特化的类模版进行实例化
```
上述代码中，Screen\<24,80\>也可以被通用模版实例化，编译器为什么会选择使用部分特化的定义进行实例化呢？解释：当程序声明了通用类模版的多个特化版本时，编译器选择最为特化的类模版定义进行实例化：
```c++
Screen<24,128> HuaweiHDR;// 使用通用类模版进行实例化
```
**部分特化的类模版和通用的类模版从模版参数表&类内的成员集合完全无关** 特化本身就是对通用的类模版的重构，如果不涉及类模版的重构，直接使用实例化就完事了😅。下面代码展示了部分特化的类模版的构造函数：
```c++
template<int height> Screen<height,80>::Screen():
    _height(height),_cursor(0),_screen(height*80,'#'){}
```
特化的类模版所有的东西都需要自身来提供：例如，程序没有提供上述特化的构造函数模版定义，而此时需要特化的类模版(Screen\<height,80\>)实例化一个类型(如Screen\<24,80\>)，则不会从通用模版类(Screen\<height,width\>)中来实例化这样的类型。**简化成地球语言(cpp pseudo-code)**：
```c++
// 如果定义了通用类模版Screen<height, width> 以及特化类模版Screen<height, 80>
// 那么所有的形如Screen<height, 80>的实例化都由特化的类模版来解决 =>
// 即便特化的类模版不能够解决(如未定义合适的构造函数导致无法正确初始化Screen<2,80>对象)
// 也只能报错! 编译器可不迁就!(指调用Screen<height,width>中的构造函数) 
// 但也不一定...不知道什么版本的编译器具备了'退而求其次'的功能 WTF!
```

## 类模版中的名字解析
在类模版定义or类模版成员定义时，名字解析的步骤如下：**1** 在模版被定义时，解析出不依赖于模版参数的名字。**2** 在模版被实例化时，解析出依赖于模版参数的名字。可以参考下面的类模版的成员函数定义进行理解：
```c++
#include<iostream>// 为不依赖模版参数名字提供声明
#include<cstdlib>// 为不依赖模版参数名字提供声明
template<typename T> T Queue<T>::remove(){
    if(is_empty()){
        std::cerr<<"remove on empty queue!"<<endl;// no.1 不依赖模版参数的名字
        exit(-1);// no.1 不依赖模版参数的名字
    }
    QueueItem<T> *ptr = _front;
    _front = _front->next;
    delete ptr;
    T returnval = ptr->item;// no.2 依赖模版参数的名字
    std::cout<<"value removed: "<<returnval<<endl;// no.2 依赖模版参数的名字
    return returnval;
}
```
**1** 因此，在模版定义之前，需要为no.1不依赖模版参数的名字提供声明，如果定义时找不到这些名字，则引发编译错误。
**2** 对于no.2依赖模版参数的名字，需要考虑其实例化点(point of instantiation)。实例化点的位置很重要，因为实例化点的位置决定了为依赖模版参数的名字考虑声明的范围。

**[模版的实例化点的位置]** **1** 类模版的实例化点总是在名字空间域中，并且总是在使用类的实例的声明or定义之前。**2** 类模版的成员函数or静态成员实例化点总是在使用类的实例的声明or定义之后。
```c++
#include<Queue.h>
#include<QueueLongDouble.h>
// no.1 Queue<LongDouble>类模版实例化点
int main(){
    // no.1 引用Queue<LongDouble>类模版实例
    Queue<LongDouble> *ptr = new Queue<LongDouble>;
    // no.2 引用Queue<LongDouble>::remove()成员函数的实例
    ptr->remove();
}
// no.2 Queue<LongDouble>::remove()函数实例化点
```
上述代码说明了：在类模版被实例化时，其成员函数or静态数据成员不会被实例化，只有当程序使用这些成员时才会进行实例化。**[Solve the Puzzle]** 因此，一种避免出错(实例化时看不到相应声明)的方法是：在类模版定义中用到的名字、以及类模版成员函数中用到的名字应该放在头文件中，并且在**第一次**实例化类模版or类模版成员之前，包含该头文件。

## 类模版和名字空间
上面的章节讲述的内容都是基于全局空间作用域而言，这一部分重点介绍类模版及其成员在名字空间中的声明、定义及使用方式。<br>
**[1] 类模版及其成员在名字空间中的定义**
```c++
#include<iostream>
#include<cstdlib>
namespace cpp_primer{
    template<typename T> class Queue{...};
    template<typename T> Queue<T>::remove(){...}
}
```
**[2] 类模版及其成员在名字空间中的使用**
```c++
int main(){
    // no.1 不推荐! 名字污染! 代码膨胀! 
    using namespace cpp_primer;
    Queue<int> *ptr = new Queue<int>;
    ptr->remove();
    // no.2 使用using‘精确’引入 
    using cpp_primer::Queue;// 使用using声明引入Queue类模版及其成员的定义
    Queue<int> *ptr = new Queue<int>;
    ptr->remove();
    // no.3 无须using的方式 
    cpp_primer::Queue<int> *ptr = new cpp_primer::Queue<int>;
    ptr->cpp_primer::remove();
}
```
**[3] 类模版及其成员的定义or特化声明不必定义在名字空间中** 
**定义or特化声明在名字空间内**
```c++
#include<iostream>
#include<cstdlib>
namespace cpp_primer{
    template<> class Queue<char*>{...};// 类模版特化声明
    template<> double Queue<double>::remove(){...};// 类模版成员的特化声明
}
```
**定义or特化声明在名字空间外**
```c++
#include<iostream>
#include<cstdlib>
namespace cpp_primer{
    // Queue类模版的定义
    template<typename T> class Queue{...};
    // Queue的成员函数的定义
    ...
}
// 将类模版特化声明及其成员的特化声明移到namespace外
template<> class cpp_primer::Queue<char*>{...};
template<> double cpp_primer::Queue<double>::remove(){...}
```

## 模版数组类(一个综合的例子)
```c++
#ifndef ARRAY_H
#define ARRAY_H
#include<iostream>
// 模版类声明 & 模版友元重载op<<声明
template<typename EleType> class Array;
template<typename EleType>
    ostream& operator<<(ostream&, const Array<EleType>&);
// 模版类定义
template<typename EleType> class Array{
    public:
        // ctor
        explicit Array(int size=DefaultArraySize){init(nullptr, size)}
        Array(const EleType *ptrarr, int size){init(ptrarr, size)}
        Array(const Array &arr){init(arr._ptrArr, arr._size)}
        // dtor
        ~Array(){delete[] _ptrArr;}
        // overload op
        Array& operator=(const Array&);
        EleType& operator[](int idx)const{return _ptrArr[idx];}
        // tool func
        int size()const{return _size;}
        ostream& print(ostream &os=std::cout)const;
        void grow();
        void sort(int, int);
        int find(EleType);
        EleType min();
        EleType max();
    private:
        // private ctor tool func
        void init(const EleType*, int);
        void swap(int, int);
        // private data member
        static const int DefaultArraySize = 12;
        int _size;
        EleType *_ptrArr;
};
```
```c++
// init private ctor tool func
template<typename EleType> 
    void Array<EleType>::init(const EleType *ptr, int size){
        _size = size;
        _ptrArr = new EleType[size];// operator new 
        for(int i=0; i<_size; ++i){
            if(!ptr){
                _ptrArr[i]=0;
            }
            else{
                _ptrArr[i]=ptr[i];
            }
        }
}
```
```c++
// overload op=
template<typename EleType> 
    Array<EleType>& Array<EleType>::operator=(const Array<EleType> &arr){
        if(this!=&arr){
            delete[] _ptrArr;
            init(arr._ptrArr, arr._size);
        }
        return *this;
    }
// overload op<<
template<typename EleType> 
    ostream& operator<<(ostream &os, const Array<EleType> &arr){
        return arr.print(os);
    }
```
```c++
// [no.1 tool func] print 按照设定的格式打印数组元素
template<typename EleType> 
    ostream& Array<EleType>::print(ostream &os=std::cout)const{
        const int linelength = 12;// 设置一行输出的长度
        os<<"("<<_size<<")<";
        for(int i=0; i<_size; ++i){
            if(i%linelength==0 && i){
                os<<"\n\t";
            }
            os<<_ptrArr[i];
            // 最后一行or最后一列元素后不输出','
            if(i%linelength!=linelength-1 && i!=_size-1){
                os<<",";
            }
        }
        os<<">\n";
        return os;
    }
// [no.2 tool func] grow 重新分配内存 扩增数组容纳元素的数量
template<typename EleType>
    void Array<EleType>::grow(){
        EleType *old_ptrArr = _ptrArr;
        int old_size = _size;
        _size = old_size + old_size/2 + 1;
        _ptrArr = new EleType[_size];// operator new申请新内存
        // 分两部分构造新的数组元素
        int idx;
        for(idx=0; idx<old_size; ++idx){// 原始元素直接赋值
            _ptrArr[idx] = old_ptrArr[idx];
        }
        for(; idx<_size; ++idx){// 新扩增的部分调用构造函数
            _ptrArr[idx] = EleType();
        }
        delete[] old_ptrArr;// 析构旧对象、释放旧内存
    }
// [no.3 tool func] min() & max() 获得数组的最大最小值 
template<typename EleType>
    EleType Array<EleType>::min(){
        assert(_ptrArr!=0);
        EleType min_val = _ptrArr[0];
        for(int i=0; i<_size; ++i){
            if(_ptrArr[i]<min_val){
                min_val = _ptrArr[i];
            }
        }
        return min_val;
    }
template<typename EleType>
    EleType Array<EleType>::max(){...}// 同理
// [no.4 tool func] find 函数用于在数组中查找特定元素 并返回相应idx 
template<typename EleType>
    int Array<EleType>::find(EleType val){
        for(int i=0; i<_size; ++i){
            if(val==_ptrArr[i]){
                return i;
            }
        }
        return -1;
    }
```
```c++
// [no.1 tool func] sort & swap 函数采用快速排序算法对于数组元素进行排序
template<typename EleType> 
    void Array<EleType>::swap(int i, int j){// toolfunc for quicksort
        EleType tmp = _ptrArr[i];
        _ptrArr[i] = _ptrArr[j];
        _ptrArr[j] = tmp;
    }
template<typename EleType>
    void Array<EleType>::sort(int low, int high){// quicksort
        if(low>high){return;}
        int l=low;  int h=high+1;
        EleType ele = _ptrArr[low];
        for(;;){
            while(_ptrArr[++l]<ele && l<high);
            while(_ptrArr[--h]>ele && h>low);
            if(l<h){
                swap(l, h);
            }
            else{break;}
        }
        swap(low, hi);
        sort(low, h-1);
        sort(h+1, high);
    }
#endif
```

## Reference
> cpp primer 3rd chapter 16.1 p407 <br>
> cpp primer 5th chapter 16.1.2 p609 <br>
> cpp primer 3rd chapter 9.3.1 p382

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内
