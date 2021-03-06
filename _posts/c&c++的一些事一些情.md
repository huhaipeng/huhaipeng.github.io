---
layout:     post
title:      C&C++的一些事一些情
subtitle:   
date:       2019-2-1
author:     HHP
header-img: img/post-bg-unix-linux.jpg
catalog: true
tags:
    - linux  
    - c/c++
---

### .hpp是什么

.hpp，本质就是将.cpp的实现代码混入.h头文件当中，定义与实现都包含在同一文件，则该类的调用者只需要include该.hpp文件即可，无需再将cpp加入到project中进行编译。而实现代码将直接编译到调用者的obj文件中，不再生成单独的obj，采用hpp将大幅度减少调用project中的cpp文件数与编译次数，也不用再发布lib与dll文件，因此非常适合用来编写公用的开源库。

hpp的优点不少，但是编写中有以下几点要注意： 
1、是Header Plus Plus的简写。（.h和.hpp就如同.c和.cpp似的） 
2、与.h类似，.hpp是C++程序头文件格式。 
3、是VCL专用的头文件,已预编译。 
4、是一般模板类的头文件。 
5、一般来说，.h里面只有声明，没有实现，而.hpp里声明实现都有，后者可以减少.cpp的数量。 
6、.h里面可以有using namespace std，而.hpp里则无。 
7、不可包含全局对象和全局函数。

由于.hpp本质上是作为.h被调用者include的，所以当hpp文件中存在全局对象或者全局函数，而该hpp被多个调用者include时，将在链接时导致符号重定义错误。要避免这种情况，需要去除全局对象，将全局函数封装为类的静态方法。

实质上，在linux环境下，不管你头文件的后缀名是啥(不管你是.ipp 还是 .gpp),gcc都能编译通过，但是最好还是按照规范来，养成良好的编程习惯。





### c和c++的一些不同的编程风格

大家都知道，c++是兼容c的，但是在一些可以实现同样效果的地方，c和c++的编程风格还是有很多不同的。

说一些我常遇到的：

* c语言里，强制转换我们一般使用`(type)var`的形式，比如`int *b = (int*)malloc(sizeof(int))`

  但在c++里，我们一般用stl的`static_cast<type>`(编译时)或者`dynamic_cast<type>`(运行时)代替c的这种形式。比如`int *b = static_cast<int*>(char* a)`。 

* 在c++里，我们最好用new/delete或者智能指针（c++11）动态分配内存，而不是用malloc/free。

* C语言是面向过程的，但是通过指针的方式可以实现回调，从而实现面向对象和接口的思想；而在c++里，我们一般使用抽象基类的方法实现多态。

* 在C语言里，我们要想在函数里改变一个外部变量，一般会使用指针来进行参数传递，而在c++中，可以使用`&`引用符号。

* 在c++11中，新增了auto关键字，这样在定义变量时无需显式说明该变量类型，而是由编译器自行推断。这样可以有利于跨平台。

* c++的stl中实现了很多容器类，大大提高了我们的生产效率，在用c++编程时应善加利用。



### 关于const

主要是讨论const和指针的关系。

底层const：指针的值可以改变，指针指向的变量的值不能改变，如：const int *p 就是一个底层const；即指针常量；

顶层const：指针的值不能改变，指针指向的变量的值能改变，如：int *const p = &a 就是一个顶层const。顶层const一定要初始化，即常量指针

cosnt int*  const p:指向常量的指针常量。

```c++
const int *p;
int *p1;
int a = 1;
const int aa = 1;
int *pp = &a;//不合法；const类型变量只能赋给const指针；
p = &a;
p1  = p;//不合法，cosnt不能赋给non-const，但实际上可以调整编译参数使其编译通过。但很不安全。
```





```c++
#include <string>
#include <iostream>
using namespace std;
class text{
public:
    text() = default;
    text(string ss):s(ss){}
    text& operator=(const text& t)  {
        this->s = t.s;
        cout << "succ" << endl;
        return *this;
    } 
    string s;
};


int main(){
    text t("hello");
    text t1;
    text t2;
    t2 = t1 = t;
}
```







![](https://cdn.sinaimg.cn.52ecy.cn/large/005BYqpgly1g41wdfbddgj30j60sitdb.jpg)

