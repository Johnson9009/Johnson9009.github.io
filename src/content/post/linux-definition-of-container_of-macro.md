---
title: Linux Definition of container_of Macro
author: Johnson
type: post
date: 2016-09-11T02:28:05+00:00
categories:
  - C/C++
tags:
  - C/C++

---
<p align="right">
  <font color="#ff0000"><strong>Reproduce from</strong> <a href="http://stackoverflow.com/questions/6083734/rationale-behind-the-container-of-macro-in-linux-list-h" target="_blank"><strong>http://stackoverflow.com/questions/6083734/rationale-behind-the-container-of-macro-in-linux-list-h</strong></a></font>
</p>

#### Question

In the implementation of linux kernel lists in `/include/linux/list.h`, what is the rationale behind the first line (pasted below) of the `container_of` macro?

> const typeof( ((type \*)0)->member ) \*__mptr = (ptr); 

In a sample code of mine, I removed this line and changed the definition to

> #define container_of(ptr, type, member) ({\   
> &#160;&#160;&#160;&#160; (type \*)( (char \*)ptr &#8211; offsetof(type,member) );})

and my code still showed expected results. Is the first line redundant then? Or does it have some hidden trap that I am not aware of?

The code I found at <a href="http://kernelnewbies.org/FAQ/LinkedLists" target="_blank">Faq/LinkedLists</a>

> /**   
> * container_of &#8211; cast a member of a structure out to the containing structure   
> * @ptr:&#160;&#160;&#160;&#160;&#160;&#160;&#160; the pointer to the member.   
> * @type:&#160;&#160;&#160;&#160;&#160;&#160; the type of the container struct this is embedded in.   
> * @member:&#160;&#160;&#160;&#160; the name of the member within the struct.   
> *   
> */   
> #define container_of(ptr, type, member) ({\   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160; const typeof( ((type \*)0)->member ) \*__mptr = (ptr);\   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160; (type \*)( (char \*)__mptr &#8211; offsetof(type,member) );})
> 
> #define offsetof(TYPE, MEMBER) ((size_t) &((TYPE *)0)->MEMBER)

<!--more-->

#### Answer

It adds some type checking. With your version, this compiles fine (without warning):

> struct foo { int bar; };   
> &#8230;.   
> float a;   
> struct foo *var = container_of(&a, foo, bar);

With the kernel version, the compiler reports:

> warning: initialization from incompatible pointer type

Good explanation of how the macro works: [container_of][1] by Greg Kroah-Hartman.

 [1]: http://www.kroah.com/log/linux/container_of.html