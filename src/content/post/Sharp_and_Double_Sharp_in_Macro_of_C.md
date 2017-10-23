---
title: 'C语言宏中的#和##号'
date: 2015-11-19T13:52:27+00:00
slug: 'C语言宏中的sharp和sharpsharp号'
categories:
  - C/C++
tags:
  - C/C++
author: 'Johnson Zhang'
---
**一、一般用法**

我们使用#把宏参数变为一个字符串，用##把两个宏参数贴合在一起。用法:

``` c++
#include<cstdio>
#include<climits>

using namespace std;

#define STR(s)      #s
#define CONS(a,b)   int(a##e##b)

int main() {
  printf(STR(vck));             // 输出字符串"vck"
  printf("%d\n", CONS(2,3));    // 2e3 输出:2000
  return 0;
}
```
<!--more-->

**二、当宏的参数是另一个宏的时候**

需要注意的是凡宏定义里有用&#8217;#&#8217;或&#8217;##&#8217;的地方宏参数是不会再展开。

  * **1.非&#8217;#&#8217;和&#8217;##&#8217;的情况**

> #define TOW       (2)
  
> #define MUL(a,b) (a*b)
  
> printf(&#8220;%d*%d=%d\n&#8221;, TOW, TOW, MUL(TOW,TOW));
  
> 这行的宏会被展开为：
  
> printf(&#8220;%d\*%d=%d\n&#8221;, (2), (2), ((2)\*(2)));
  
> MUL里的参数TOW会被展开为(2)。

  * **2. 当有&#8217;#&#8217;或&#8217;##&#8217;的时候**

> #define A           (2)
  
> #define STR(s)      #s
  
> #define CONS(a,b)   int(a##e##b)
  
> printf(&#8220;int max: %s\n&#8221;,   STR(INT\_MAX));     // INT\_MAX ＃i nclude<climits>
  
> 这行会被展开为：
  
> printf(&#8220;int max: %s\n&#8221;, &#8220;INT_MAX&#8221;);
  
> printf(&#8220;%s\n&#8221;, CONS(A, A));                // compile error
  
> 这一行则是：
  
> printf(&#8220;%s\n&#8221;, int(AeA));

INT_MAX和A都不会再被展开，然而解决这个问题的方法很简单。加多一层中间转换宏。
  
加这层宏的用意是把所有宏的参数在这层里全部展开，那么在转换宏里的那一个宏(_STR)就能得到正确的宏参数。

> #define A            (2)
  
> #define _STR(s)      #s
  
> #define STR(s)       _STR(s)                 // 转换宏
  
> #define _CONS(a,b)   int(a##e##b)
  
> #define CONS(a,b)    _CONS(a,b)        // 转换宏
  
> printf(&#8220;int max: %s\n&#8221;, STR(INT\_MAX));           // INT\_MAX,int型的最大值，为一个变量 ＃include<climits>
  
> 输出为: int max: 0x7fffffff
  
> STR(INT\_MAX) &#8211;>   \_STR(0x7fffffff) 然后再转换成字符串；
  
> printf(&#8220;%d\n&#8221;, CONS(A, A));
  
> 输出为：200
  
> CONS(A, A)   &#8211;>   _CONS((2), (2))   &#8211;> int((2)e(2))
