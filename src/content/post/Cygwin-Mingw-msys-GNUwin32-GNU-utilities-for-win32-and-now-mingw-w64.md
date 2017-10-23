---
title: Cygwin, MinGW, MSYS, GnuWin32, GNU Utilities for Win32 and now MinGW-w64
author: Johnson
type: post
date: 2016-07-07T08:38:41+00:00
categories:
  - 专业笔记
tags:
  - C/C++
  - Linux

---
<p align="right">
  <font color="#ff0000"><strong>Reproduce from</strong> <a href="http://poquitopicante.blogspot.com/2012/08/cygwin-mingw-msys-gnuwin32-gnu.html" target="_blank"><strong>http://poquitopicante.blogspot.com/2012/08/cygwin-mingw-msys-gnuwin32-gnu.html</strong></a></font>
</p>

#### Introduction

GNU on Windows has a sordid past. Not really, but sordid sounds so much more interesting than complicated. So here&#8217;s how I understand it, without any emotion, this is really just a navigational exercise.
  


#### Disclaimer

What was just said and what I am about to say is entirely opinion and not based on fact. I hope that I do not offend anyone, but I am sure that I inevitably will, as much of my opinion is based on hearsay. In that case I apologize, and please remember that the purpose of this blog is purely selfish; it is a note to myself to remind me of what took me so long to finally understand. So instead of getting angry with me, perhaps post a comment and disabuse me of my still as yet incomplete knowledge.
  


#### Cygwin

In the beginning there was [Cygwin][1]. Their tagline is "Get that Linux feeling &#8211; on Windows!" This does a lot to explain exactly what Cygwin is, Linux emulated on Windows. What this means in practice is that Cygwin applications use the cygwin.dll, a common runtime that must be linked to any applications that run on Cygwin, and consequently any applicaton compiled with Cygwin gcc. So essentially, when you use Cygwin you are more or less stuck in Cygwin. True, you could distribute the cygwin.dll with your application, so it would appear to be _native_, but it wouldn&#8217;t be truly native. By native here I mean that it runs on one of the windows runtimes like the [win32api or msvcrt][2]. The [Windows binary of GNU nano][3] is an example of an application compiled using Cygwin that requires cygwin.dll to run. The probable downside of using this layer between the Windows API and your application may be potentially slower speed and some limits in its features. 

<div align="right">
  <!--more-->
</div>

#### MinGW

From Cygwin was born [MinGW][4]. I don&#8217;t know if this is actually true, but that is what I&#8217;ve heard. Either way it doesn&#8217;t really matter. The essential thing to realize here is this.
  


> MinGW creates native applications for Windows using native libraries.</p>
This means exactly what it sounds like, applications compiled using MinGW&#8217;s compilers (gcc, g++, gfortran, etc.) will run on any Windows machine natively without any runtime other than the Windows API or mscvrt. Now it might be fun to speculate about some internal strife or a passionate drive of a group of individuals with some bold ideal, but that&#8217;s irrelevant to this fundamental difference between the Cygwin and MinGW.
  


#### MSYS

[MSYS][5] may seem like a variation on Cygwin, except it is really meant to be used as an environment for running shell scripts, similar to [bash (the Bourne Again SHell)][6]. It&#8217;s true that if you compile an application using the MSYS version of gcc, then you will need to link it to the msys-1.0.dll, so in a sense it is the same as Cygwin. The difference is really in a state of mind. MSYS is presumably part of MinGW, which stands for Minimalist GNU for Windows. What that translates to is that MinGW and MSYS contain only the most essential tools required for developing native applications for Windows using the GNU open source suite of compilers, whereas Cygwin aims to provide Windows users every application available to Linux. In Cygwin you will find [Python][7], [GTK][8], [nano][9], [Ruby][10], [Git][11], even some games I think, whereas in MinGW you will only find libraries most often used in common code. However there is a huge chunk of open source code that is compiled using [autotools][12] and depending on shell scripting, which I think is where MSYS gets involved. MSYS makes it easier to do this. True autotools, make and shell script functions have been ported as native win32 applications, but since they are only used as temporary tools to get to the finished product (a natively compiled win32 application) and the results don&#8217;t depend on any of those temporary tools, why go through the effort of duplicating what Cygwin has already done so nicely?
  


#### GnuWin32 and GNU Utilities for Windows

If you were just reading the section above, then continuing on, [GnuWin32][13] and the now defunct [GNU Utilities for Windows][14]seem to do precisely that. Both of these generally utilize MinGW to port GNU applications that were native to Linux or Cygwin to run natively on Windows. And in fact many open source projects now try to configure options for some Windows compiler, be it MinGW, [MSVC][15] or [BCC][16] (really, I haven&#8217;t seen this too often although [apparently it is still out there][17], wow they even have Delphi!). Some projects are optionally opting for [CMake][18] and alternative to Autotools. OK, so the key distinction here is that these are both compiled using MinGW and are basically an alternate to MSYS, if you prefer to continue to use the native Windows CMD console or something like Powershell.
  


#### MinGW-w64

[MinGW-w64][19] is a fork of MinGW that allows you to cross compile code from one machine to another. For example, say you want to make 64-bit code, but you are on a 32-bit machine or a Linux box, or using Cygwin, then you would use MinGW-w64. The "w64" comes from it&#8217;s ability to compile 64-bit code, which is not currently available (AFAIK) with MinGW (sometimes called mingw32). There are alternate distributions of both MinGW and MinGW-w64 provided by[TDM][20]. Also there are other cross-compilers, such as [mxe.cc][21], but it is only for *nix (Linux/Unix). Coincidentally MinGW are MinGW-w64 both available as cross-compilers in most Linux distro repositories. Ubuntu, Fedora and Suse all have it. MacPorts, Fink and Homebrew may have them too. Same probably goes for mxe.cc. And who knows there are probably more cross-compilers out there as well.
  


#### Epilogue

Ah. So glad to get that all down, finally. So the moral of the story is this.
  


> **If you want to use native Windows apps compiled using GNU gcc use MinGW, but _do not polute your toolchain with any MSYS_ or Cygwin libraries _or you will be sorry_.**</p>

 [1]: http://cygwin.com/
 [2]: http://en.wikipedia.org/wiki/Microsoft_Windows_library_files
 [3]: http://www.nano-editor.org/download.php
 [4]: http://www.mingw.org/
 [5]: http://www.mingw.org/wiki/msys/
 [6]: http://www.gnu.org/software/bash/
 [7]: http://www.python.org/
 [8]: http://www.gtk.org/
 [9]: http://www.nano-editor.org/
 [10]: http://www.ruby-lang.org/en/
 [11]: http://git-scm.com/
 [12]: http://en.wikipedia.org/wiki/GNU_build_system
 [13]: http://gnuwin32.sourceforge.net/
 [14]: http://unxutils.sourceforge.net/
 [15]: http://www.microsoft.com/visualstudio/en-us
 [16]: http://www.embarcadero.com/products/cbuilder/free-compiler
 [17]: http://www.embarcadero.com/
 [18]: http://www.cmake.org/
 [19]: http://mingw-w64.sourceforge.net/
 [20]: http://tdm-gcc.tdragon.net/
 [21]: http://mxe.cc/