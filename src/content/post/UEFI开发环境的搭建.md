---
title: UEFI开发环境的搭建
author: Johnson
type: post
date: 2014-11-10T09:19:43+00:00
categories:
  - UEFI开发
tags:
  - UEFI
  - 工作

---
以前写教程式的记录，总觉得写的特别详细，每个步骤所需要的东西都要直接加链接，这样别人可以直接点击就可以下载到，可是后来发现这样并不是好的方式，因为网上的链接过一段时间很可能就变了，看文章的人想去下载的时候发现下载不了，所以以后都不直接加链接而是精确地指明所需要的东西，然后用户自己一搜索就可以下载到了。

UEFI开发中，新手会遇到三个容易混淆的概念，EDKII、UDK2010/UDK2014和Intel提供给特定开发板的UEFI开发源码包，简单的说EDKII是UEFI标准的一个开源实现，其中包括好多个包，当然由于UEFI标准不是为了Intel一家芯片设定的，所以EDKII中会包括多种体系结构的支持。UDK2010/UDK2014可以说是一个Intel版的EDKII，其中使用了大量的EDKII的包，但只包含对Intel体系结构的支持。第三种Intel提供给特定开发板的UEFI开发源码包又可以说是UDK2010/UDK2014针对一个特定开发板的定制版本。 

* * *

**系统要求：Microsoft Windows 7 SP1 Ultimate 64-bit**

下面的步骤基本是在UEFI官方给出的步骤之上完善的，可以通过搜索Tianocore进入UEFI开源社区官网，然后点击UDK2014，再点击 How to Build UDK2014 Release，来找到。

<div align="right">
  <!--more-->
</div>

  1. **Setup Build Environment** 
      * Install Microsoft Visual Studio 2008* SP1 in the build machine and **<font style="background-color: #ffff00" color="#ff0000">make sure that AMD64 complier was selected when installing</font>.** <font style="background-color: #ffff00" color="#ff0000"><strong>(默认是不选择的，在语言工具下的Visual C++下的X64编译器和工具) </strong></font>
  2. **Extract Common Source Code** 
      * Extract files in [UDK2014.MyWorkSpace.zip] to the working space directory (e.g C:). Note the Directory "MyWorkSpace" will be created as a result. In this case, it is C:\MyWorkspace. 
      * There are two BaseTools package one is for Windows system and another is for UNIX-Like system. Please make sure BaseTools(Windows).zip is used here. Expand the appropriate BaseTools to C:\MyWorkSpace <font style="background-color: #ffff00" color="#ff0000"><strong>（我这里放在了E盘，MyWorkSpace相应的改成了UDK2014，最终，一级目录结构如下）<font style="background-color: #ffffff"></font></strong></font><font color="#ff0000"><strong> </strong></font>
    <p style="padding-left: 120px" class="eng-par-style">
      <font size="1">E:. <br />├─UDK2014 <br />│&#160; ├─BaseTools <br />│&#160; ├─Conf <br />│&#160; ├─DuetPkg <br />│&#160; ├─EdkCompatibilityPkg <br />│&#160; ├─EdkShellBinPkg <br />│&#160; ├─FatBinPkg <br />│&#160; ├─FatPkg <br />│&#160; ├─IA32FamilyCpuPkg <br />│&#160; ├─IntelFrameworkModulePkg <br />│&#160; ├─IntelFrameworkPkg <br />│&#160; ├─MdeModulePkg <br />│&#160; ├─MdePkg <br />│&#160; ├─NetworkPkg <br />│&#160; ├─Nt32Pkg <br />│&#160; ├─PcAtChipsetPkg <br />│&#160; ├─PerformancePkg <br />│&#160; ├─SecurityPkg <br />│&#160; ├─ShellBinPkg <br />│&#160; ├─ShellPkg <br />│&#160; ├─SourceLevelDebugPkg <br />│&#160; └─UefiCpuPkg </font>
    </p>

  3. **Generate OpenSSL* Crypto Library Note: this does not need to be done for Nt32** 
      * Open file "C:\MyWorkspace\CryptoPkg\Library\OpensslLib\Patch-HOWTO.txt" and follow the instruction to install OpenSSL* for UEFI building. <font style="background-color: #ffff00" color="#ff0000"><strong>(使用UDK2014这个补丁已经打上了，不用做这步了) </strong></font>
  4. **Build Steps \*\\*\* NT32 \*\**** 
      * 1)Open a command prompt, type command "cd C:\MyWorkspace" to enter the workspace directory, and then type command **<font style="background-color: #ffff00" color="#ff0000">(我这里是E:\UDK2014)</font>** 
    <p class="eng-par-style">
      > edksetup ––nt32
    </p>
    
    <p class="eng-par-style">
      to initialize the working environment. See also:<a href="https://github.com/tianocore/tianocore.github.io/wiki/Windows-systems-ToolChain-Matrix">Windows_systems_ToolChain_Matrix</a> for how to change the TOOL_CHAIN_TAG for supported compiler combinations.
    </p>
    
      * 2)Type below commands to build platforms (below assumes Microsoft Visual Studio 2008) 
    <p class="eng-par-style">
      > build -t VS2008x86 <font style="background-color: #ffff00" color="#ff0000"><strong>(这里一定是这个，不要写成VS2008，原因下面讲)</strong></font>
    </p>
    
    <p class="eng-par-style">
      Note: There are two methods to select the tool chain (Use Microsoft Visual Studio 2008* as sample):
    </p>
    
      * 1.Update TOOL\_CHAIN\_TAG in file Conf/target.txt: TOOL\_CHAIN\_TAG = VS2008<font style="background-color: #ffff00" color="#ff0000"><strong>(这里应该是VS2008x86)</strong> </font>
      * 2.Add -t build option in command line: "build -t VS2008<font style="background-color: #ffff00" color="#ff0000"><strong>(这里也应该是VS2008x86)</strong></font> &#8230; " 
    <p class="eng-par-style">
      For 32-bit VS2008 on 64-bit WINDOWS OS, VS2008x86 should be selected instead of VS2008. Please refer to tools_def.txt for all supported tool chains and detailed descriptions. (tools_def.txt will be generated at Conf directory after running "edksetup".)
    </p>
    
      * 3.Note Microsoft Visual Studio* 2010 is supported with -t VS2010 or -t VS2010x86 

* * *

VS2008这个软件就只有32位版本的，没有64位版本的，不像VS2013。所以，选择这个工具链的名称是写VS2008还是VS2008x86，依赖于开发主机的操作系统是32位的还是64位的。因为VS2008这个程序是32位程序，所以在32位的操作系统里一般是安装在"Program Files"文件夹中的，在64位的操作系统中是安装在"Program Files (x86)"文件夹中的。所以，如果开发主机的操作系统是32位的，那这个工具链的名字就写VS2008，如果开发主机的操作系统是64位的，就写VS2008x86。我们这里是64位的Win7所以这里写的是VS2008x86。