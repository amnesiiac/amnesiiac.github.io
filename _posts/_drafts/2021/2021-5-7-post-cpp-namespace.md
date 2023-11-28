---
layout: post
title: "cpp - namespace"
subtitle: 'namespace命名空间知识整理'
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: true
date: 2021-05-07 10:58
lang: ch 
catalog: true 
categories: cpp 
tags:
  - Time 2021
  - cpp fundermentals
---
### 全局域
全局域中隐式的声明了一个全局名字空间，即缺省的情况下，在全局域中声明的每个对象、函数、类型或者模版都引入了一个全局实体(global entity)。由于全局实体具有外部链接性，在整个工程中，只能有一份实体，有唯一的名字。关于外部链接性，详见[博客](/cpp/2021/05/06/post-cpp-linkage/)。

在实际生产过程中，我们通常在程序中引入其他的库，在全局域中，需要保证已有的声明定义和引入的库中的名字不能出现冲突，即需要考虑避免全局名字空间污染的问题(global namespace pollution)。

一种直接的做法是：通过使用较长的不容易产生冲突的名字对于自定义的东西进行命名。
```c++
class cplusplus_primer_matrix { ... };// 使用下划线的方式将名字'独占'
```
这种方式能够解决问题，但是不是最佳方式。如果整个工程中所有定义的东西都使用这么长的名字，用起来很麻烦。一种替代的方式是使用自定的名字空间对空间中定义的名字加上一个修饰。

### 名字空间
**[名字空间的定义方式]**
```c++
namespace cpp_primer{
    class matrix{};
    void inverse(matrix &);
}
```
cpp_primer是用户声明的名字空间，和全局名字空间不同，用户声明的名字空间需要显示的被声明。在名字空间中声明的实体被称为名字空间成员，如matrix、inverse。自定名字空间中的每个名字需指向空间中的唯一实体。名字空间和空间内实体之间的修饰关系如下：
```c++
// 名字空间中的实体需要加上名字空间名进行修饰
matrix -> cpp_primer::matrix;
inverse -> cpp_primer::inverse;
// 使用多个名字空间中的matrix函数
cpp_primer::inverse(cpp_primer::matrix);
other_namespace::inverse(other_namespace::matrix);
```

**[名字空间的定义不一定是连续的]**
```c++
namespace cpp_primer{
    class matrix{};
    void inverse(matrix &);
}
// 其他程序片段
namespace cpp_primer{// 在程序的其他部分 可以向已定义的名字空间补充实体
    class new_matrix{};
    void new_inverse(matrix &);
}
```
名字空间定义的这种非连续特性，可以让我们将名字空间中的实体的声明和定义分开构建，如我们将自定义库的接口放在一起，在程序其他部分将库的具体实现放在一起。这种做法便于程序结构管理。
```c++
namespace cpp_primer{/* 库函数的声明部分 */}
namespace cpp_primer{/* 库函数的实现 */}
```
名字空间这种非连续性甚至可以跨文件应用：
```c++
// header file
namespace cpp_primer{/* 库函数的声明部分 */}
// source file
namespace cpp_primer{/* 库函数的定义 */}
```

**[使用名字空间中定义的实体]** <br>
定义在名字空间中的实体在使用时，必须加上限定修饰符，否则只能从全局名字空间中查找定义：
```c++
#include "header.h"
cpp_primer::inverse(matrix &m);// 错误! 不能找到matrix的定义
cpp_primer::inverse(cpp_primer::matrix &m);// 正确的名字空间实体访问
```
程序实体前不加限定修饰符，则默认从全局名字空间中寻找定义。但是全局名字空间没有具体的名字，使用如下方法显式的访问：
```c++
::inverse(cpp_primer::matrix &m);// 访问全局名字空间中的inverse 以及自定的matrix
```
下面给出一个例子，用来展示全局名字空间访问的具体应用。
```c++
#include<iostream>
const int max=10000;
void show(){
    int max=100;
    std::cout<<max<<std::endl;// 访问局部中的max
    std::cout<<::max<<std::endl;// 访问全局名字空间的max
}
```
运行show函数，我们将得到100 \n 10000的输出结果。

**[多重名字空间实体的声明定义及其使用]** <br>
名字空间的定义可以使用嵌套的机制，通过这种机制，可以应对更加复杂的函数命名划分情况。在访问最内层名字空间的实体时，需要带上嵌套的所有的名字空间的限定访问符。
```c++
int page_num;
namespace cpp_primer{
    int page_num;// 屏蔽掉全局同名对象::page_num
    namespace part{
        int page_num;// 屏蔽掉cpp以及全局名字空间的同名对象 
        namespace chapter{
            int page_num;// 屏蔽掉part、cpp以及全局名字空间的同名对象 
            void func(){
                int page_num;// 函数局部域对象屏蔽掉chapter以及外部所有的page_num
            }
        }
    }
}
::page_num; page_num;// 默认访问全局对象
cpp_primer::page_num;// 访问cpp中定义的对象
cpp_primer::part::page_num;// 访问cpp::part中的对象
cpp_primer::part::chapter::page_num;// 访问cpp::part::chapter中的对象
```
在使用多重嵌套的名字空间中的东西时，注意其访问的优先级顺序。内层的同名对象会将外层的屏蔽掉。

名字空间中实体的定义可以实现在相应的名字空间内部(同级)，也可以实现在相应的名字空间外部。在名字空间外部定义，应当根据情况添加相应的限定修饰符。
```c++
namespace cpp_primer {
    namespace matrixlib{
        class matrix{ /* ... */ };
        matrix operator+(const matrix &m1, const matrix &m2);
    } 
}
// 命名空间外部的定义
cpp_primer::matrixlib::matrix 
    cpp_primer::matrixlib::operator+(const matrix &m1, const matrix &m2){
    matrix result;// 定义一个cpp_primer::matrixlib::matrix局部变量
}
// [1] op+函数和参数在相同的域中 op+函数参数表中的<位于相同域下>的参数域访问符可省略
// [2] op+函数的返回值需要加域访问符 即使返回值和op+函数域、参数域一致也不能省略
// [3] op+函数参数域、定义域内 所有的东西都继承了op+函数域 它们公共的域访问符可省略
```
在声明的名字空间外部进行定义时，需要注意：只能在包含该对象声明的命名空间中进行定义，不包含其声明的外部命名空间无法定义。
```c++
// -- header.h --
namespace cpp_primer{
    namespace matrixlib{
        matrix operator+(const matrix &m1, const matrix &m2);
    } 
}
// -- source file --
#include "header.h"
namespace wrong_space{
    // <典型错误的定义> wrong_space命名空间下 无法完成operator+的定义 ->
    // <原因> wrong_space空间中 访问不到operator+的声明
    cpp_primer::matrixlib::matrix 
        cpp_primer::matrixlib::operator+(const matrix &m1, const matrix &m2){
        matrix result;
    }
}
```
上述规则可以简化成：**只能在通过限定修饰符访问到其声明的scope下，进行相应对象的定义，否则定义产生错误**。

**[命名空间定义的实体的链接性以及声明方式]** <br>
命名空间定义的函数隐形具有外部链接性，即在整个程序文件中(跨越多个编译单元)，只能有一份定义实体，可以有多个声明。因此，对于命名空间定义的对象的声明应当放在头文件中，所有文件如果想使用命名空间中定义的实体，应当包含这个头文件；而命名空间中的定义应当定义在某一个源文件中。

**[无名命名空间]** <br>
c++中可以使用未命名的名字空间声明一个作用域局限于某个文件的实体。
```c++
// source file 1
namespace{// 无名命名空间
    void swap(double *d1, double *d2);// swap函数只在source file1可见
}
// source file 2
namespace{// 无名命名空间
    void swap(double *d1, double *d2);// swap函数不同于上述swap 仅在file2可见
}
```
和c++中无名命名空间(unnamed namespace)作用类似的一种实现是c语言中的static关键字。无名命名空间和被声明为static的全局实体具有类似的特性。下面的swap函数声明同在source file1全局域无名命名空间声明swap函数一致，都使得swap函数具有单文件作用域。
```c++
// source file1
static void swap(double *d1, double *d2);// 在全局域声明static函数swap
```
尽量使用无名命名空间代替c语言中static关键字。

### 名字空间的使用
对于复杂的嵌套的名字空间声明的变量，使用namespace::member_name进行书写往往是非常麻烦的，名字空间别名、using声明、using指示符是帮助我们简化命名空间使用的三大法宝。

**[名字空间别名]** <br>
```c++
namespace shortname1 = cpp_primer::matrixlib
// 原始复杂的定义 返回类型通过两层命名空间嵌套进行修饰 -> 冗杂
cpp_primer::matrixlib::matrix operator+(const matrix &m1, const matrix &m2);
// 经过namespace alias简化过后的定义 -> 简介明了
shortname1 matrix operator+(const matrix &m1, const matrix &m2);
// 相同的命名空间可以拥有多个别名 不同的别名可以同时交替使用
namespace shortname2 = shortname1;// 增加一个别名 用法同shortname1
shortname2 matrix operator+(const matrix &m1, const matrix &m2);
```

**[using声明]** <br>
using声明通过另外一种方式，简化了多层嵌套的命名空间声明的变量。
```c++
// 多重嵌套定义的类对象matrix 
namespace cpp_primer{
    namespace matrixlib{
        class matrix{...};
    }
}
// 原始复杂的嵌套方式使用多层命名空间中的matrix
cpp_primer::matrixlib::matrix operator+(const matrix &m1, const matrix &m2);
// using声明 - 在全局域中声明的using - 在全局域默认情况使用命名空间嵌套定义的matrix
using cpp_primer::matrixlib::matrix;
// 通过using声明 可以直接在全局域中使用matrix
matrix operator+(const matrix &m1, const matrix &m2);
```
下面的程序展示了using声明在使用的过程中的注意事项。`1` using是一种特殊的声明方式，它在该域中必须惟一(在using声明同一个域中不能再次声明相同对象)；`2` using声明能够屏蔽域外的同名对象；`3` using声明可以被更加底层的嵌套域中的同名对象隐藏。具体特性见下面的程序。
```c++
namespace int_num{
    int i=16, j=15, k=23;
}
int j=0;
void change_num(){
    using int_num::i;// 在chang_num函数域中定义i
    ++i;// 17
    using int_num::j;// 在change_num函数域中隐藏外部定义的j
    ++j;// 16
    int k;// 声明一个局部变量k
    using int_num::k;// 错误! 在局部域中重复定义了k
}
int kk=k;// 错误! 此处对象k不可见
```

**[using指示符]** <br>
通过使用using别名的方式可以实现：将所需要东西的声明一一进行引入。在namespace机制引入之前，已经有数量众多的工程使用了未经限定修饰的名字，如果c++将底层库使用namespace进行改写、修饰，那么导致很多依赖底层库的工程将无法正常工作。 <br>
针对上述情况，使用using声明将所有新定义的库中的对象逐一引入是非常麻烦的。因此，使用using指示符一次性将整个名字空间中声明的对象进行引入是必要的。
```c++
// 一种非常繁琐的引用方式 - 将所需的名字空间中的东西逐一引入
using cpp_primer::matrixlib::matrix;
using cpp_primer::matrixlib::matrix1;
using cpp_primer::matrixlib::matrix2;
// 一种等价的替代方式 - 一次性将命名空间的所有变量全部引入 
using namespace cpp_primer;
```
在使用using指示符的时，应当注意避免和全局定义的名字空间中的变量冲突引发二义性的错误。
```c++
namespace int_num{
    int i=16, j=15, k=23;
}
int j=0;
void change_num(){
    using namespace int_num;// 将名字空间int_num全部引入
    ++i;// 使用int_num中的声明 i=17
    ++j;// 产生二义性的错误 编译器不知道使用全局中的j还是int_num中的j
    ++::j;// 使用全局中的j 
    ++int_num::j;// 使用int_num中的j
    int k=99;// 定义局部变量 隐藏int_num::k
    ++k;// k=100
}
```
上述程序`++j`产生二义性的根本原因是：编译器看到`j`时，无法判断是使用全局中的声明还是int_num中的声明。全局声明的变量在局部的作用域中可以省略::，而通过using指示符引入的声明集合也默许了这一点。因此产生了二义性。<br>
另外，需要注意的是：编译器只有看到`++j`表达式，才会寻找何其相关的命名空间，而不是看到**using namespace int_num**就自动的为域中每一个变量设置其限定修饰符。简而言之，**编译器确定一个变量的名字空间的方式是从下而上的，而不是从上而下的逐层隐藏覆盖的方式。**

上述程序`int k=99`可以隐藏int_num名字空间中的变量的原因是：当前域中的对象声明定义的优先级高于名字空间中的声明。或者可理解成：名字空间(int_num)仿佛实在using namespace int_num所在的域外声明的一样(优先级低于域内的声明定义)。

关于using指示符的补充说明(using指示符污染问题)：尽量使用using单一声明，代替using指示符。using指示符虽然方便，但是，如果程序依赖很多名字空间，那么如果同时引入多个名字空间(且名字空间包含了同名声明)，可能在同名对象调用的时候，产生二义性错误。这种错误只在定义时才能被检测出来。

**[using声明引入namespace重载函数]**
```c++
namespace overload{
    int max(int, int);  int max(double, double);// 重载max
    extern void print(int); extern void print(double);// 重载print
}
using overload::max;// 正确
using overload::print(double);// 错误 using声明引用名字空间的重载函数不能带参数表
int max(float, float);// 正确 using声明引入的max和这个max构成重载关系
int max(int, int);// 错误 using声明和引入的max重复
```
重载函数使用using声明进行引入的时候，不能通过参数表区分从而选择特定的函数进行使用，否则重载的机制失去了意义：和直接在名字空间分别取名无二。

### 标准名字空间std
标准名字空间中的声明的引入方式有三种：
```c++
#include <iostream>
#include <string>
#include <iterator>
int main(){
    istream_iterator<string> infile(cin);// 编译失败 使用未定义的对象
    // 方法一 - 在使用的地方显式的声明 - 较方法三麻烦一些
    std::istraem_iterator<string> infile(std::cin);// 显式使用std进行限定修饰
    // 方法二 - 可能导致std名字污染
    using namespace std;// 使用using指示符将std中声明的名字一次性引入
    istream_iterator<string> infile(cin);
    // 方法三 - 有效避免将std全部引入可能导致的名字污染
    using std::istream_iterator; using std::cin;// 使用using声明单独引入
    istream_iterator<string> infile(cin);

}
```
## Reference
> 《c++ primer》 3rd p359


> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内
