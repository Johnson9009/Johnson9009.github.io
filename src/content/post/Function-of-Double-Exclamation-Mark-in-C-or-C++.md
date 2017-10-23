---
title: C/C++语言双感叹号作用!!
author: Johnson
type: post
date: 2015-11-18T16:38:32+00:00
categories:
  - C/C++
tags:
  - C/C++

---
两个!是为了把非0值转换成1,而0值还是0。在C语言中，所以非0值都表示真。

所以!非0值 = 0，而!0 = 1。所以!!非0值 = 1，而!!0 = 0。

Windows中对assert宏的定义就用到了这个知识：

#define assert(\_Expression) (void)( (!!(\_Expression)) || (\_wassert(\_CRT\_WIDE(#\_Expression), \_CRT\_WIDE(\_\_FILE\_\_), \_\_LINE\_\_), 0) )