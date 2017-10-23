---
title: 'How debuggers work: Part 1 – Basics'
author: Johnson
type: post
date: 2016-08-05T07:07:40+00:00
categories:
  - C/C++
  - 软件调试
tags:
  - C/C++
  - Debug
  - Linux

---
<p align="right">
  <font color="#ff0000"><strong>Reproduce from</strong></font> <a title="http://eli.thegreenplace.net/2011/01/23/how-debuggers-work-part-1" href="http://eli.thegreenplace.net/2011/01/23/how-debuggers-work-part-1" target="_blank"><strong>http://eli.thegreenplace.net/2011/01/23/how-debuggers-work-part-1</strong></a>
</p>

This is the first part in [a series of articles][1] on how debuggers work. I&#8217;m still not sure how many articles the series will contain and what topics it will cover, but I&#8217;m going to start with the basics.

##### In this part

I&#8217;m going to present the main building block of a debugger&#8217;s implementation on Linux &#8211; the <tt>ptrace</tt> system call. All the code in this article is developed on a 32-bit Ubuntu machine. Note that the code is very much platform specific, although porting it to other platforms shouldn&#8217;t be too difficult.

##### Motivation

To understand where we&#8217;re going, try to imagine what it takes for a debugger to do its work. A debugger can start some process and debug it, or attach itself to an existing process. It can single-step through the code, set breakpoints and run to them, examine variable values and stack traces. Many debuggers have advanced features such as executing expressions and calling functions in the debbugged process&#8217;s address space, and even changing the process&#8217;s code on-the-fly and watching the effects.

Although modern debuggers are complex beasts [[1]][2], it&#8217;s surprising how simple is the foundation on which they are built. Debuggers start with only a few basic services provided by the operating system and the compiler/linker, all the rest is just [a simple matter of programming][3].

<div align="right">
  <!--more-->
</div>

##### Linux debugging &#8211; <tt>ptrace</tt>

The Swiss army knife of Linux debuggers is the <tt>ptrace</tt> system call [[2]][4]. It&#8217;s a versatile and rather complex tool that allows one process to control the execution of another and to peek and poke at its innards [[3]][5]. <tt>ptrace</tt> can take a mid-sized book to explain fully, which is why I&#8217;m just going to focus on some of its practical uses in examples.

Let&#8217;s dive right in.

##### Stepping through the code of a process

I&#8217;m now going to develop an example of running a process in "traced" mode in which we&#8217;re going to single-step through its code &#8211; the machine code (assembly instructions) that&#8217;s executed by the CPU. I&#8217;ll show the example code in parts, explaining each, and in the end of the article you will find a link to download a complete C file that you can compile, execute and play with.

The high-level plan is to write code that splits into a child process that will execute a user-supplied command, and a parent process that traces the child. First, the main function:

> int main(int argc, char** argv)   
> {   
> &#160;&#160;&#160; pid\_t child\_pid;
> 
> &#160;&#160;&#160; if (argc < 2) {   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160; fprintf(stderr, "Expected a program name as argument\n");   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160; return -1;   
> &#160;&#160;&#160; }
> 
> &#160;&#160;&#160; child_pid = fork();   
> &#160;&#160;&#160; if (child_pid == 0)   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160; run_target(argv[1]);   
> &#160;&#160;&#160; else if (child_pid > 0)   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160; run\_debugger(child\_pid);   
> &#160;&#160;&#160; else {   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160; perror("fork");   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160; return -1;   
> &#160;&#160;&#160; }
> 
> &#160;&#160;&#160; return 0;   
> }

Pretty simple: we start a new child process with <tt>fork</tt> [[4]][6]. The <tt>if</tt> branch of the subsequent condition runs the child process (called "target" here), and the <tt>else if</tt> branch runs the parent process (called "debugger" here).

Here&#8217;s the target process:

> void run_target(const char* programname)   
> {   
> &#160;&#160;&#160; procmsg("target started. will run &#8216;%s&#8217;\n", programname);
> 
> &#160;&#160;&#160; /\* Allow tracing of this process \*/   
> &#160;&#160;&#160; if (ptrace(PTRACE_TRACEME, 0, 0, 0) < 0) {   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160; perror("ptrace");   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160; return;   
> &#160;&#160;&#160; }
> 
> &#160;&#160;&#160; /\* Replace this process&#8217;s image with the given program \*/   
> &#160;&#160;&#160; execl(programname, programname, 0);   
> }

The most interesting line here is the <tt>ptrace</tt> call. <tt>ptrace</tt> is declared thus (in <tt>sys/ptrace.h</tt>):

> long ptrace(enum _\_ptrace\_request request, pid_t pid,   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; void \*addr, void \*data);

The first argument is a _request_, which may be one of many predefined <tt>PTRACE_*</tt> constants. The second argument specifies a process ID for some requests. The third and fourth arguments are address and data pointers, for memory manipulation. The <tt>ptrace</tt> call in the code snippet above makes the <tt>PTRACE_TRACEME</tt>request, which means that this child process asks the OS kernel to let its parent trace it. The request description from the man-page is quite clear:

> Indicates that this process is to be traced by its parent. Any signal (except SIGKILL) delivered   
> to this process will cause it to stop and its parent to be notified via wait(). Also, all subsequent   
> calls to exec() by this process will cause a SIGTRAP to be sent to it, giving the parent a chance   
> to gain control before the new program begins execution. A process probably shouldn&#8217;t make   
> this request if its parent isn&#8217;t expecting to trace it. (pid, addr, and data are ignored.)

I&#8217;ve highlighted the part that interests us in this example. Note that the very next thing <tt>run_target</tt>does after <tt>ptrace</tt> is invoke the program given to it as an argument with <tt>execl</tt>. This, as the highlighted part explains, causes the OS kernel to stop the process just before it begins executing the program in<tt>execl</tt> and send a signal to the parent.

Thus, time is ripe to see what the parent does:

> void run\_debugger(pid\_t child_pid)   
> {   
> &#160;&#160;&#160; int wait_status;   
> &#160;&#160;&#160; unsigned icounter = 0;   
> &#160;&#160;&#160; procmsg("debugger started\n");
> 
> &#160;&#160;&#160; /\* Wait for child to stop on its first instruction \*/   
> &#160;&#160;&#160; wait(&wait_status);
> 
> &#160;&#160;&#160; while (WIFSTOPPED(wait_status)) {   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160; icounter++;   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160; /\* Make the child execute another instruction \*/   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160; if (ptrace(PTRACE\_SINGLESTEP, child\_pid, 0, 0) < 0) {   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; perror("ptrace");   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; return;   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160; }
> 
> &#160;&#160;&#160;&#160;&#160;&#160;&#160; /\* Wait for child to stop on its next instruction \*/   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160; wait(&wait_status);   
> &#160;&#160;&#160; }
> 
> &#160;&#160;&#160; procmsg("the child executed %u instructions\n", icounter);   
> }

Recall from above that once the child starts executing the <tt>exec</tt> call, it will stop and be sent the<tt>SIGTRAP</tt> signal. The parent here waits for this to happen with the first <tt>wait</tt> call. <tt>wait</tt> will return once something interesting happens, and the parent checks that it was because the child was stopped (<tt>WIFSTOPPED</tt> returns true if the child process was stopped by delivery of a signal).

What the parent does next is the most interesting part of this article. It invokes <tt>ptrace</tt> with the<tt>PTRACE_SINGLESTEP</tt> request giving it the child process ID. What this does is tell the OS &#8211; _please restart the child process, but stop it after it executes the next instruction_. Again, the parent waits for the child to stop and the loop continues. The loop will terminate when the signal that came out of the <tt>wait</tt>call wasn&#8217;t about the child stopping. During a normal run of the tracer, this will be the signal that tells the parent that the child process exited (<tt>WIFEXITED</tt> would return true on it).

Note that <tt>icounter</tt> counts the amount of instructions executed by the child process. So our simple example actually does something useful &#8211; given a program name on the command line, it executes the program and reports the amount of CPU instructions it took to run from start to finish. Let&#8217;s see it in action.

##### A test run

I compiled the following simple program and ran it under the tracer:

> #include <stdio.h>
> 
> int main()   
> {   
> &#160;&#160;&#160; printf("Hello, world!\n");   
> &#160;&#160;&#160; return 0;   
> }

To my surprise, the tracer took quite long to run and reported that there were more than 100,000 instructions executed. For a simple <tt>printf</tt> call? What gives? The answer is very interesting [[5]][7]. By default, <tt>gcc</tt> on Linux links programs to the C runtime libraries dynamically. What this means is that one of the first things that runs when any program is executed is the dynamic library loader that looks for the required shared libraries. This is quite a lot of code &#8211; and remember that our basic tracer here looks at each and every instruction, not of just the <tt>main</tt> function, but _of the whole process_.

So, when I linked the test program with the <tt>-static</tt> flag (and verified that the executable gained some 500KB in weight, as is logical for a static link of the C runtime), the tracing reported only 7,000 instructions or so. This is still a lot, but makes perfect sense if you recall that <tt>libc</tt> initialization still has to run before <tt>main</tt>, and cleanup has to run after <tt>main</tt>. Besides, <tt>printf</tt> is a complex function.

Still not satisfied, I wanted to see something _testable_ &#8211; i.e. a whole run in which I could account for every instruction executed. This, of course, can be done with assembly code. So I took this version of "Hello, world!" and assembled it:

> section&#160;&#160;&#160; .text   
> &#160;&#160;&#160; ; The _start symbol must be declared for the linker (ld)   
> &#160;&#160;&#160; global _start
> 
> _start:
> 
> &#160;&#160;&#160; ; Prepare arguments for the sys_write system call:   
> &#160;&#160;&#160; ;&#160;&#160; &#8211; eax: system call number (sys_write)   
> &#160;&#160;&#160; ;&#160;&#160; &#8211; ebx: file descriptor (stdout)   
> &#160;&#160;&#160; ;&#160;&#160; &#8211; ecx: pointer to string   
> &#160;&#160;&#160; ;&#160;&#160; &#8211; edx: string length   
> &#160;&#160;&#160; mov&#160;&#160;&#160; edx, len   
> &#160;&#160;&#160; mov&#160;&#160;&#160; ecx, msg   
> &#160;&#160;&#160; mov&#160;&#160;&#160; ebx, 1   
> &#160;&#160;&#160; mov&#160;&#160;&#160; eax, 4
> 
> &#160;&#160;&#160; ; Execute the sys_write system call   
> &#160;&#160;&#160; int&#160;&#160;&#160; 0x80
> 
> &#160;&#160;&#160; ; Execute sys_exit   
> &#160;&#160;&#160; mov&#160;&#160;&#160; eax, 1   
> &#160;&#160;&#160; int&#160;&#160;&#160; 0x80
> 
> section&#160;&#160; .data   
> msg db&#160;&#160;&#160; &#8216;Hello, world!&#8217;, 0xa   
> len equ&#160;&#160;&#160; $ &#8211; msg

Sure enough. Now the tracer reported that 7 instructions were executed, which is something I can easily verify.

##### Deep into the instruction stream

The assembly-written program allows me to introduce you to another powerful use of <tt>ptrace</tt> &#8211; closely examining the state of the traced process. Here&#8217;s another version of the <tt>run_debugger</tt> function:

> void run\_debugger(pid\_t child_pid)   
> {   
> &#160;&#160;&#160; int wait_status;   
> &#160;&#160;&#160; unsigned icounter = 0;   
> &#160;&#160;&#160; procmsg("debugger started\n");
> 
> &#160;&#160;&#160; /\* Wait for child to stop on its first instruction \*/   
> &#160;&#160;&#160; wait(&wait_status);
> 
> &#160;&#160;&#160; while (WIFSTOPPED(wait_status)) {   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160; icounter++;   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160; struct user\_regs\_struct regs;   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160; ptrace(PTRACE\_GETREGS, child\_pid, 0, &regs);   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160; unsigned instr = ptrace(PTRACE\_PEEKTEXT, child\_pid, regs.eip, 0);
> 
> &#160;&#160;&#160;&#160;&#160;&#160;&#160; procmsg("icounter = %u.&#160; EIP = 0x%08x.&#160; instr = 0x%08x\n",   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; icounter, regs.eip, instr);
> 
> &#160;&#160;&#160;&#160;&#160;&#160;&#160; /\* Make the child execute another instruction \*/   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160; if (ptrace(PTRACE\_SINGLESTEP, child\_pid, 0, 0) < 0) {   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; perror("ptrace");   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; return;   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160; }
> 
> &#160;&#160;&#160;&#160;&#160;&#160;&#160; /\* Wait for child to stop on its next instruction \*/   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160; wait(&wait_status);   
> &#160;&#160;&#160; }
> 
> &#160;&#160;&#160; procmsg("the child executed %u instructions\n", icounter);   
> }

The only difference is in the first few lines of the <tt>while</tt> loop. There are two new <tt>ptrace</tt> calls. The first one reads the value of the process&#8217;s registers into a structure. <tt>user_regs_struct</tt> is defined in<tt>sys/user.h</tt>. Now here&#8217;s the fun part &#8211; if you look at this header file, a comment close to the top says:

> /* The whole purpose of this file is for GDB and GDB only.   
> &#160;&#160; Don&#8217;t read too much into it. Don&#8217;t use it for   
> &#160;&#160; anything other than GDB unless know what you are   
> &#160;&#160; doing.&#160; */

Now, I don&#8217;t know about you, but it makes _me_ feel we&#8217;re on the right track 🙂 Anyway, back to the example. Once we have all the registers in <tt>regs</tt>, we can peek at the current instruction of the process by calling <tt>ptrace</tt> with <tt>PTRACE_PEEKTEXT</tt>, passing it <tt>regs.eip</tt> (the extended instruction pointer on x86) as the address. What we get back is the instruction [[6]][8]. Let&#8217;s see this new tracer run on our assembly-coded snippet:

> $ simple\_tracer traced\_helloworld   
> [5700] debugger started   
> [5701] target started. will run &#8216;traced_helloworld&#8217;   
> [5700] icounter = 1.&#160; EIP = 0x08048080.&#160; instr = 0x00000eba   
> [5700] icounter = 2.&#160; EIP = 0x08048085.&#160; instr = 0x0490a0b9   
> [5700] icounter = 3.&#160; EIP = 0x0804808a.&#160; instr = 0x000001bb   
> [5700] icounter = 4.&#160; EIP = 0x0804808f.&#160; instr = 0x000004b8   
> [5700] icounter = 5.&#160; EIP = 0x08048094.&#160; instr = 0x01b880cd   
> Hello, world!   
> [5700] icounter = 6.&#160; EIP = 0x08048096.&#160; instr = 0x000001b8   
> [5700] icounter = 7.&#160; EIP = 0x0804809b.&#160; instr = 0x000080cd   
> [5700] the child executed 7 instructions

OK, so now in addition to <tt>icounter</tt> we also see the instruction pointer and the instruction it points to at each step. How to verify this is correct? By using <tt>objdump -d</tt> on the executable:

> $ objdump -d traced_helloworld
> 
> traced_helloworld:&#160;&#160;&#160;&#160; file format elf32-i386
> 
> Disassembly of section .text:
> 
> 08048080 <.text>:   
> 8048080:&#160;&#160;&#160;&#160; ba 0e 00 00 00&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; mov&#160;&#160;&#160; $0xe,%edx   
> 8048085:&#160;&#160;&#160;&#160; b9 a0 90 04 08&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; mov&#160;&#160;&#160; $0x80490a0,%ecx   
> 804808a:&#160;&#160;&#160;&#160; bb 01 00 00 00&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; mov&#160;&#160;&#160; $0x1,%ebx   
> 804808f:&#160;&#160;&#160;&#160; b8 04 00 00 00&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; mov&#160;&#160;&#160; $0x4,%eax   
> 8048094:&#160;&#160;&#160;&#160; cd 80&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; int&#160;&#160;&#160; $0x80   
> 8048096:&#160;&#160;&#160;&#160; b8 01 00 00 00&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; mov&#160;&#160;&#160; $0x1,%eax   
> 804809b:&#160;&#160;&#160;&#160; cd 80&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; int&#160;&#160;&#160; $0x80

The correspondence between this and our tracing output is easily observed.

##### Attaching to a running process

As you know, debuggers can also attach to an already-running process. By now you won&#8217;t be surprised to find out that this is also done with <tt>ptrace</tt>, which can get the <tt>PTRACE_ATTACH</tt> request. I won&#8217;t show a code sample here since it should be very easy to implement given the code we&#8217;ve already gone through. For educational purposes, the approach taken here is more convenient (since we can stop the child process right at its start).

##### The code

The complete C source-code of the simple tracer presented in this article (the more advanced, instruction-printing version) is available [here][9]. It compiles cleanly with <tt>-Wall -pedantic --std=c99</tt> on version 4.4 of<tt>gcc</tt>.

##### Conclusion and next steps

Admittedly, this part didn&#8217;t cover much &#8211; we&#8217;re still far from having a real debugger in our hands. However, I hope it has already made the process of debugging at least a little less mysterious. <tt>ptrace</tt> is truly a versatile system call with many abilities, of which we&#8217;ve sampled only a few so far.

Single-stepping through the code is useful, but only to a certain degree. Take the C "Hello, world!" sample I demonstrated above. To get to <tt>main</tt> it would probably take a couple of thousands of instructions of C runtime initialization code to step through. This isn&#8217;t very convenient. What we&#8217;d ideally want to have is the ability to place a breakpoint at the entry to <tt>main</tt> and step from there. Fair enough, and in the next part of the series I intend to show how breakpoints are implemented.

##### References

I&#8217;ve found the following resources and articles useful in the preparation of this article:

  * [Playing with ptrace, Part I][10] 
  * [How debugger works][11]&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; 
&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; ![][12]</ul> 

[[1]][13] I didn&#8217;t check but I&#8217;m sure the LOC count of <tt>gdb</tt> is at least in the six-figures range.

[[2]][14] Run <tt>man 2 ptrace</tt> for complete enlightment.

[[3]][15] _Peek_ and _poke_ are well-known system programming [jargon][16] for directly reading and writing memory contents.

[[4]][17] This article assumes some basic level of Unix/Linux programming experience. I assume you know (at least conceptually) about <tt>fork</tt>, the <tt>exec</tt> family of functions and Unix signals.

[[5]][18] At least if you&#8217;re as obsessed with low-level details as I am 🙂

[[6]][19] A word of warning here: as I noted above, a lot of this is highly platform specific. I&#8217;m making some simplifying assumptions &#8211; for example, x86 instructions don&#8217;t have to fit into 4 bytes (the size of<tt>unsigned</tt> on my 32-bit Ubuntu machine). In fact, many won&#8217;t. Peeking at instructions meaningfully requires us to have a complete disassembler at hand. We don&#8217;t have one here, but real debuggers do.

 [1]: http://eli.thegreenplace.net/pages/code
 [2]: http://eli.thegreenplace.net/2011/01/23/how-debuggers-work-part-1#id7
 [3]: http://en.wikipedia.org/wiki/Small_matter_of_programming
 [4]: http://eli.thegreenplace.net/2011/01/23/how-debuggers-work-part-1#id8
 [5]: http://eli.thegreenplace.net/2011/01/23/how-debuggers-work-part-1#id9
 [6]: http://eli.thegreenplace.net/2011/01/23/how-debuggers-work-part-1#id10
 [7]: http://eli.thegreenplace.net/2011/01/23/how-debuggers-work-part-1#id11
 [8]: http://eli.thegreenplace.net/2011/01/23/how-debuggers-work-part-1#id12
 [9]: https://github.com/eliben/code-for-blog/blob/master/2011/simple_tracer.c
 [10]: http://www.linuxjournal.com/article/6100?page=0,1
 [11]: http://www.alexonlinux.com/how-debugger-works
 [12]: http://eli.thegreenplace.net/images/hline.jpg
 [13]: http://eli.thegreenplace.net/2011/01/23/how-debuggers-work-part-1#id1
 [14]: http://eli.thegreenplace.net/2011/01/23/how-debuggers-work-part-1#id2
 [15]: http://eli.thegreenplace.net/2011/01/23/how-debuggers-work-part-1#id3
 [16]: http://www.jargon.net/jargonfile/p/peek.html
 [17]: http://eli.thegreenplace.net/2011/01/23/how-debuggers-work-part-1#id4
 [18]: http://eli.thegreenplace.net/2011/01/23/how-debuggers-work-part-1#id5
 [19]: http://eli.thegreenplace.net/2011/01/23/how-debuggers-work-part-1#id6