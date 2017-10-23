---
title: C++中对象的初始化
author: Johnson
type: post
date: 2016-07-02T00:50:00+00:00
categories:
  - C/C++
tags:
  - C/C++

---
<h3 align="center">
  Most Vexing Parse问题
</h3>

  * int a\_int\_object(5); 这样定义一个int对象对么？ 
  * int b\_int\_object(); 我只想让b\_int\_object使用默认构造函数进行初始化即可，这样行么？ 
  * 若Tool是一个类，那么Tool hammer(); 可以么？ Tool *hammer = new Tool(); 这样呢？ 
  * 若Tool是一个结构体呢？ 

这是一类问题先看这类问题，待会儿再看下一个问题。Tool hammer(); 或 int b\_int\_object(); 是通不过编译的，但是int a\_int\_object(5); 是可以的，这是C++中的Most Vexing Parse问题。

Tool hammer(); 之所以不行，是因为编译器无法知道这是在定义一个Tool类的对象，还是在声明一个函数，这个函数返回值为Tool类的对象且没有入参。

所以对于类对象应该直接 Tool hammer; 就好了，这样就会调用默认的构造函数的。对于基本类型 int b\_int\_object; 这样定义的话就不一定会被初始化了，如果此变量是局部变量那肯定是不会被初始化的，所以我们只能这么写 int b\_int\_object(0); 或 int b\_int\_object = 0; 

C++11对这个问题的解决办法叫Uniform Initialization即需要写成 int b\_int\_object{};也就是说用{}代替()。

<h3 align="center">
  Template类或结构体默认构造函数初始化列表
</h3>

<p align="left">
  &#160;
</p>

> template <class T>   
> struct Node {   
> &#160; Node() : value(), next(NULL) {}
> 
> &#160; T value;   
> &#160; Node *next;   
> }

我们知道C++中类和结构体之间没有本质的区别，只是默认的访问权限不同而已，所以上面的问题同时适用于类和结构体。Template的好处就是一套逻辑能应用于各种类型，不用为每种数据类型写一套代码，看上面这个Template的声明，我们想下其中的Node() : value(), next(NULL) {}有没有存在的必要？

尤其对value数据成员，其中默认构造函数只是调用了一下value数据成员的默认构造函数进行的初始化，我们知道如果我们定义一个Node对象，那么它在被创建的时候，一定会去调用默认构造函数，然后去构造它的每一个数据成员，构造数据成员的时候也一定会自动去调用数据成员自己的默认构造函数，那为什么还需要在Node的默认构造函数的初始化列表中来这么一下呢？应该是多余的吧？删掉变成这样Node() : next(NULL) {}应该也没有问题吧？

显然答案是否定的，因为如果T是一个ADT(Abstract Data Type)，那么确实可以删掉，但是如果T是基本数据类型，那就不行了，删掉变成这样Node() : next(NULL) {}的话，这个value就不会做任何初始化，那么就很不好。