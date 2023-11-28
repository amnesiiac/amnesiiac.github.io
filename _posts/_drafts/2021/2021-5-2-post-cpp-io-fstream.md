---
layout: post
title: "cpp - fstream"
subtitle: 'c++文件相关操作知识整理'
author: "twistfatezz"
header-style: text
# header-img: "img/in-post/economics_1/headimg.png"
# header-mask: 0.8
comments: false 
mathjax: true
date: 2021-05-02 17:30
lang: ch 
catalog: true 
categories: cpp
tags:
  - Time 2021
  - cpp fundermentals
---
### fstream继承关系图表
<center><img src="/img/in-post/cpp_img/fstream_1.png" width="100%"></center>

### 打开文件流三种基本方式
```c++
#include <iostream>// iostream = istream + ostream
#include <fstream>// fstream = ifstream + ofstream
using namespace std;

int main(){
    ofstream myfile;
    // no.1 open(filename)
    myfile.open("example.txt");
    // no.2 open(filename, mode)
    myfile.open("exmple.txt", ios::out);
    // no.3 all the flags canbe combined using '|' 
    myfile.open("example.txt", ios::out | ios:ate | ios::binary);
    // 用法和cout一致 cout针对显示 myfile针对文件
    myfile<<"Writing this to a file.\n";
    // 关闭
    myfile.close();
    return 0;
}
```
### open() mode flags

| :---: | :---: |
| ios::in | open for input operations |
| ios::out | open for output operations |
| ios::binary | open in binary mode |
| ios::ate | set the init position at the end of the file(default is at the beginning) |
| ios::app | all output operations are performed at the end of the file, appending the content to the end of file |
| ios::trunc | If the file is opened for output operations and it already existed, its previous content is deleted and replaced by the new one |

### usage of mode flag
使用`|`操作符可以同时使用mode_flag多种特性进行文件流操作。
```c++
// 第一种写法 创建文件流
ofstream myfile;
myfile.open("example.bin", ios::out | ios::app | ios::binary);
// 第二种写法 创建文件流
ofstream myfile("example.bin", ios::out | ios::app | ios::binary);
```

### default mode flag 
**fstream -> ios::out | ios::in** <br> 
```c++
int main(){
    fstream iofile;
    iofile.open("example.txt");// iofile此时同时具有infile和outfile属性
    iofile.open("example.txt", ios::in);// iofile被overriden成infile
    iofile.open("example.txt", ios::out);// iofile被overriden成outfile
    return 0;
}
```
**ofstream -> ios::out** <br>
```c++
int main(){
    ofstream outfile;
    outfile.open("example.txt");// outfile具有默认的ios::out属性
    outfile.open("example.txt", ios::out);// outfile具有ios::out属性
    outfile.open("example.txt", ios::in);// outfile可以使用ios::in属性 
    // 注意 虽然outfile可以使用ios::in属性 只能通过outfile将内容读入filebuf
    //     但是最终iostream不能支持其进行输入 filebuf中的数据可以使用 
    return 0;
}
```
**ifstream -> ios::in** <br>
```c++
int main(){
    ifstream infile;
    infile.open("example.txt");// infile具有默认的ios::in属性
    infile.open("example.txt", ios::in);// infile具有ios::in属性
    infile.open("example.txt", ios::out);// infile可以使用ios::out属性
    return 0;
}
```
**[为什么infile可以具有ios::out属性?为什么outfile可以具有ios::in属性?]** 
> 首先看文件读入、输出操作过程：<br>
> Default constructor: constructs a stream that is not associated with a file: default-constructs the std::basic_filebuf and constructs the base with the pointer to this default-constructed std::basic_filebuf member. <br>
> 首先调用default构造创建一个std::basic_filebuf对象和指向这个对象的指针rdbuf()；然后使用这个指针构建基类(stream类)的base；然后通过指针调用rdbuf()->open(filename, mode | std::ios_base::in)转为对标准输入输出流进行操作。

于是infile虽然可以指定ios::out属性，只能将构建一个std::basic_filebuf，但是在调用指针对输出流操作的时候会被istream拒绝。<br>
同理，outfile虽然可以指定ios::in属性，但它只能构建好std::basic_filebuf，但是在使用输入流的是后悔会被ostream拒绝。
虽然上述操作会被拒绝，但是构建好的std::basic_filebuf可以进行其他骚操作。总之，还是没啥大用。

```c++
// std::basic_filebuf是一个std::basic_streambuf的派生类
template<typename CharT, typename Traits=std::char_traits<CharT>> 
class basic_filebuf : public std::basic_streambuf<CharT, Traits>
```

### 文件读写程序
通过下面的程序可以检查file stream是否成功的打开了文件：

```c++
if(myfile.is_open());// true->open false->open failed
```
在完成文件读写之后，显式关闭文件输入输出流：

```c++
myfile.close();// flush the file buf, close the file
```
一个完整的读写txt文件的程序，只要文件的打开方式没有使用ios::binary，则默认以txt file stream的方式进行文件打开。
```c++
#include <iostream>
#include <fstream>
#include <string>
using namespace std;

void writefunc(){// txt文件写入程序
    ofstream myfile("example.txt");// 不使用ios::binary模式 则按txt读取
    if(myfile.is_open()){
        myfile << "This is a line.\n";
        myfile << "This is another line.\n";
        myfile.close();
    }
    else{cout << "Unable to open file";}
}
void readfunc(){// txt文件读取 并显示到屏幕上
    string str;
    ifstream myfile("example.txt");
    if(myfile.is_open){
        // getline(istream &, string &, char delimiter)
        while(getline(myfile, str)){
            cout << str;
        }
        myfile.close();
    }
    else{cout << "Unable to open file";}
}
```
### 读写状态检测函数
```c++
string str;
fstream iofile("example.txt");
getline(iofile, str);
iofile << "xxx blablabla";
iofile.bad();// true if 读写的文件打开失败 or 写操作时没有空间剩余
iofile.fail();// true if bad()==true or 读取操作类型不对
iofile.eof();// true if 用于读取的文件已经到达了end
iofile.good();// false if 上述所有检测为true | 注意good和bad不是完全对称的
// clear()可以用来重置相关的标识位
```
### 流指针(stream pointers)的位置确定
ifstream，有一个称为get pointer的指针，指向下一个将被读取的元素。<br>
ofstream，有一个称为put pointer的指针，指向写入下一个元素的位置。<br>
fstream，同时继承了get和put。

```c++
// obtaining file size - 利用tellg() seekg()获得文件大小
#include <iostream>
#include <fstream>
using namespace std;

int main(){
    streampos begin, end;
    ifstream myfile("example.bin", ios::binary);// stream obj
    begin = myfile.tellg();// get pointer position in stream
    myfile.seekg(0, ios::end);// set get & put pointer position
    end = myfile.tellg();// get pointer position in stream
    myfile.close();
    cout << "size is: " << (end-begin) << " bytes.\n";
    // 同理 可以对ofstream对象使用tellp() & seekp()进行相应streampos控制
    return 0;
}
```
上面的tellg()和tellp()函数没有参数，返回streampos类型，用于定位get pointer和put pointer在stream中的位置。seekg()和seekp()函数重载了如下的两种方式：
```c++
// 接受streampos位置信息 移动put()和get()到指定的位置
seekp(position); seekg(position);
// 将put&get在指定的位置为起点 移动offset
seekp(offset, direction); seekp(offset, direction);
// direction有如下enum类型几种选择 - 决定了offset开始计算的起点
ios::beg; ios::end; ios::cur;// 分别从stream的起始位置 结束位置 和当前位置开始
// streampos: stream position type
streampos;
// streamoff: type to represent position offsets in a stream, 
// large enough to represent the maximum possible file size  
streamoff; 
```
### 二进制文件的读写write & read
之前提到过使用使用流插入运算符`<<`和`>>`以及getline进行二进制文件读写，这种方式效率较低，因为二进制文件不需要进行format转换。使用write和read函数能够更好进行二进制读写。
```c++
write(memory_block, size);// member func of ofstream(inherited from ostream)
read(memory_block, size);// member func of ifstream(inherited from istream)
// memory_block: type(char *): the address of the read data stored or 
//              the address of the data tobe written are taken from
//             就是一个data传入传出stream buffer
// size: int value, the number of the characters to read & write
#include <iostream>
#include <fstream>
using namespace std;

int main(){
    streampos size;
    char *memblock;
    ifstream file("example.bin", ios::in|ios::binary|ios::ate);
    if(file.is_open()){
        size = file.tellg();// get stream pos
        memblock = new char [size];// open 2's buf
        file.seekg(0, ios::beg);// change get stream pos
        file.read(memblock, size);// read func to get size chars into buf
        file.close();// close file
        cout << "the entire file content is in memory";
        delete[] memblock;// release buf
    }
    else cout << "Unable to open file";
    return 0;
}
```

### buffer & flush
在文件流读写过程中，这个缓存buffer实际是一块内存空间，作为流(stream)和物理文件的媒介。在调用put指针写字符的时候，字符不是直接写入对应的物理文件中，而是先插入到该stream类对象的开辟的buffer中。<br>
当对缓存执行flush操作的时候，缓存中的内容(同一个stream)被一次性的写入物理文件中(对于输出流而言)，或者是被简单抹除(对于输入流而言)，这个操作称为同步(synchronize)。下面列举了几种涉及同步的情况：<br>
`1` 文件被关闭时，尚未被写入or已经读入的缓存被flush。<br>
`2` 当缓存buffer存储满，被自动flush。<br>
`3` 控制符明确调用：例如使用了flush or std::endl等。<br>
`4` 明确地调用sync()函数，引发立即同步，返回一个int值，-1表示流没有与之关联的缓存or操作失败。<br>
flush使用方法如下所示：
```c++
#include <ostream>// std::flush
#include <fstream>// std::ofstream

int main(){
    std::ofstream outfile("test.txt");
    for(int n=0; n<100; n++){
        outfile << n << std::flush;// explicitly flush the streambuf
    }
    outfile.close();
    return 0;
}
```
## Reference
> https://www.cplusplus.com/doc/tutorial/files/ <br>
> https://en.cppreference.com/w/cpp/io/basic_ifstream/basic_ifstream <br>
> https://en.cppreference.com/w/cpp/io/basic_ofstream  

> 1 当使用inline数学公式且公式经过GFM排版之后都在同一行 使用`$...$`符号<br>
> 2 当希望数学公式单独成行或者经过GFM排版之后占用多行 应当使用`$$...$$`符号<br>
> 3 对于表示条件概率 需要表示竖线的时候`|` 应当使用`\mid` 而不是直接在键盘上打出`|` => 容易被编辑器认为是一个md制表符<br>
> 4 在md引入图片的时候 不要使用`<center>`和`</center>` 在这篇文档的编辑过程中vscode的preview插件在使用了上述符号之后 导致下一段的数学公式预览显示不正常<br>
> 5 使用md的时候 单独的两段文字上下需要空出一行<br>
> 6 想要强制换行的时候 需要使用`<br>`而不是`<enter>`<br>
> 7 特殊字符如果想要避免和md解析关键字冲突 应当使用``将关键字包含在内
