---
title: Calling Convention “stdcall” VS “cdecl”
author: Johnson
type: post
date: 2016-07-31T15:09:40+00:00
categories:
  - C/C++
tags:
  - C/C++

---
We know calling convention of all Windows API is stdcall, and the default calling convention of C/C++ is cdecl, but why, why Windows choose the stdcall as API’s calling convention? What&#8217;s the benefit of it?

I found below answer from internet.

> It was historically added. On x86 &#8211; stdcall is hardly cheaper, than cdecl:   
> stdcall: in function &#8211; ret N (2 bytes)   
> cdecl: in function &#8211; ret (1 byte) + everywhere where this function is caused &#8211; add sp, N (3 bytes).

Before today, I just know the different of stdcall and cdecl, but has never think about the question "What&#8217;s the benefit of stdcall?". There are many different calling conventions, consider the method of passing parameters, some calling convention do it through stack such as stdcall and cdecl when some others do it using registers such as fastcall, consider the responsibility for cleaning the arguments from the stack, in stdcall the callee should do that when in cdecl it is caller.

<div align="right">
  <!--more-->
</div>

I get a example from textbook.

![](/images/9.jpg)
![](/images/8.jpg)

As shown in above figure, the last instruction of subroutine is “RET 6”, so the calling convention of it is stdcall, in this example we can get the change of stack when a subroutine is&#160; called, by the way this example is a 16-bit assemble application.

One of cdecl&#8217;s benefit is significant, it allow funciton have variable number of parameter, because the caller must know there are how many parameters, but callee can&#8217;t.
