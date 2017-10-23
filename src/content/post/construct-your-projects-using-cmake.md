---
title: Construct Your Projects Using CMake
author: Johnson
type: post
date: 2016-07-26T15:28:04+00:00
categories:
  - C/C++
  - 项目管理工具
tags:
  - C/C++
  - CMake

---
Firstly, I will say thanks to my colleague Jiangli, your suggestion give me more power to practise my English writing. This is my first English post, I will persist in practising my English by this way even though it is hard to me.

These days, I reorganize my little project using CMake, CMake is cross-platform free and open-source software for managing the build process of software using a compiler-independent method, it is used in conjunction with native build environments such as make, Apple&#8217;s Xcode and Microsoft Visual Studio. It is a good construct tool which can generate configuration file of native build environments such as Makefile, project files automatically.

In my little project, I use Goolge Test to do unit test, and comply Goolge C++ Style Guide when I&#8217;m writing any code, besides I make it possible to lint my code with cpplint.py. At first I haven&#8217;t find a good way to organize files of my project, then I found below directory structure is good for me.

<div align="right">
  <!--more-->
</div>

> .   
> │&#160; .gitignore   
> │&#160; CMakeLists.txt   
> │&#160; LICENSE   
> │&#160; README.md   
> ├─build&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;   
> ├─gtest&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;   
> ├─include   
> │&#160; ├─c   
> │&#160; │&#160;&#160;&#160;&#160;&#160; single_list.h   
> │&#160; └─cxx   
> │&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; single_list.h   
> ├─src   
> │&#160; │&#160; source.cmake   
> │&#160; ├─c   
> │&#160; │&#160;&#160;&#160;&#160;&#160; single_list.c   
> │&#160; │&#160;&#160;&#160;&#160;&#160; source.cmake   
> │&#160; └─cxx   
> │&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; single_list.cc   
> │&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; source.cmake   
> ├─test   
> │&#160; │&#160; source.cmake   
> │&#160; ├─c   
> │&#160; │&#160;&#160;&#160;&#160;&#160; single\_list\_test.cc   
> │&#160; │&#160;&#160;&#160;&#160;&#160; source.cmake   
> │&#160; └─cxx   
> │&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; single\_list\_test.cc   
> │&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; source.cmake   
> └─utils   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160; add\_style\_check.cmake   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160; cpplint.py   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160; internal_utils.cmake

All header files need be put in “include” directory, “test” directory is for the unit test code, “build” directory is not included in git, because it is only used when you need build your project. “CMakeLists.txt" under root directory of project is the entry of CMake, I will explain its content in details one step at a time.

> cmake\_minimum\_required(VERSION 2.6.2)   
> project(data\_structure\_algorithm CXX C)
> 
> include(utils/internal_utils.cmake)   
> fix\_default\_compiler\_settings()&#160; # Defined in internal\_utils.cmake.

using "include" command to import another CMake script, "internal_utils.cmake" define&#160; some helper functions and macros used by current project. For MSVC, CMake sets certain compile flags to defaults we want to override. Our project is preferable to use CRT as static libraries, as we don&#8217;t have to rely on CRT DLLs being available. CMake always defaults to using shared CRT libraries, so we override that default here.

> \# Add Google Test as a subproject of our project.   
> add_subdirectory(gtest)   
> include_directories(gtest/include)
> 
> \# Our project header files location.   
> include_directories(include)
> 
> include(${CMAKE\_CURRENT\_LIST_DIR}/src/source.cmake)   
> include(${CMAKE\_CURRENT\_LIST_DIR}/test/source.cmake)

In our project, we organize all source files of each target through CMake script files recursive including. Every directory with source files in it, must have a CMake script file "source.cmake", and each CMake script file(this file(CMakeLists.txt) no exception) should contain all its sub directory&#8217;s "source.cmake" by "include" command. 

> add\_executable(cxx\_data\_structure\_test ${CXX\_DATA\_STRUCTURE\_TEST\_SRCS})   
> target\_link\_libraries(cxx\_data\_structure\_test gtest\_main)   
> add\_style\_check(cxx\_data\_structure_test   
> &#160; EXCLUDE_DIRS gtest   
> )
> 
> add\_executable(c\_data\_structure\_test ${C\_DATA\_STRUCTURE\_TEST\_SRCS})   
> target\_link\_libraries(c\_data\_structure\_test gtest\_main)   
> add\_style\_check(c\_data\_structure_test   
> &#160; EXCLUDE_DIRS gtest   
> )

Because source file list of each target, such as "CXX\_DATA\_STRUCTURE\_TEST\_SRCS", is defined in above CMake script files(source.cmake), so we must lay all commands which need source file list, such as "add\_executable", bellow to any "include" command which aim to include "source.cmake". "add\_style_check" command is implemented by myself, it can get all header files used by source files of this target automatically, and together with all source files of this target are passed to style check tool, it works well but have a lot of limitations, so I will redesgin and reimplement it using Python later.

Let see the content of "source.cmake" in "src" directory.

> include(${CMAKE\_CURRENT\_LIST_DIR}/c/source.cmake)   
> include(${CMAKE\_CURRENT\_LIST_DIR}/cxx/source.cmake)

Macro "CMAKE\_CURRENT\_LIST_DIR" is very useful, you will get a absolute path of the directory which current CMake script file is in.

Let see the content of "source.cmake" in "src/c" directory.

> set(C\_DATA\_STRUCTURE\_TEST\_SRCS ${C\_DATA\_STRUCTURE\_TEST\_SRCS}   
> &#160;&#160;&#160; ${CMAKE\_CURRENT\_LIST\_DIR}/single\_list.c   
> )

If you are familiar of&#160; CMake script language, you must have understand how it works now.