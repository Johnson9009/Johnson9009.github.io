---
title: 'How debuggers work: Part 2 – Breakpoints'
author: Johnson
type: post
date: 2016-08-05T07:54:45+00:00
categories:
  - C/C++
  - 软件调试
tags:
  - C/C++
  - Debug
  - Linux

---
<p align="right">
  <font color="#ff0000"><strong>Reproduce from</strong> <a title="http://eli.thegreenplace.net/2011/01/27/how-debuggers-work-part-2-breakpoints" href="http://eli.thegreenplace.net/2011/01/27/how-debuggers-work-part-2-breakpoints"><strong>http://eli.thegreenplace.net/2011/01/27/how-debuggers-work-part-2-breakpoints</strong></a></font>
</p>

This is the second part in a series of articles on how debuggers work. Make sure you read [the first part][1] before this one.

##### In this part

I&#8217;m going to demonstrate how breakpoints are implemented in a debugger. Breakpoints are one of the two main pillars of debugging &#8211; the other being able to inspect values in the debugged process&#8217;s memory. We&#8217;ve already seen a preview of the other pillar in part 1 of the series, but breakpoints still remain mysterious. By the end of this article, they won&#8217;t be.

##### Software interrupts

To implement breakpoints on the x86 architecture, software interrupts (also known as "traps") are used. Before we get deep into the details, I want to explain the concept of interrupts and traps in general.

A CPU has a single stream of execution, working through instructions one by one [[1]][2]. To handle asynchronous events like IO and hardware timers, CPUs use interrupts. A hardware interrupt is usually a dedicated electrical signal to which a special "response circuitry" is attached. This circuitry notices an activation of the interrupt and makes the CPU stop its current execution, save its state, and jump to a predefined address where a handler routine for the interrupt is located. When the handler finishes its work, the CPU resumes execution from where it stopped.

<div align="right">
  <!--more-->
</div>

Software interrupts are similar in principle but a bit different in practice. CPUs support special instructions that allow the software to simulate an interrupt. When such an instruction is executed, the CPU treats it like an interrupt &#8211; stops its normal flow of execution, saves its state and jumps to a handler routine. Such "traps" allow many of the wonders of modern OSes (task scheduling, virtual memory, memory protection, debugging) to be implemented efficiently.

Some programming errors (such as division by 0) are also treated by the CPU as traps, and are frequently referred to as "exceptions". Here the line between hardware and software blurs, since it&#8217;s hard to say whether such exceptions are really hardware interrupts or software interrupts. But I&#8217;ve digressed too far away from the main topic, so it&#8217;s time to get back to breakpoints.

##### int 3 in theory

Having written the previous section, I can now simply say that breakpoints are implemented on the CPU by a special trap called <tt>int 3</tt>. <tt>int</tt> is x86 jargon for "trap instruction" &#8211; a call to a predefined interrupt handler. x86 supports the <tt>int</tt> instruction with a 8-bit operand specifying the number of the interrupt that occurred, so in theory 256 traps are supported. The first 32 are reserved by the CPU for itself, and number 3 is the one we&#8217;re interested in here &#8211; it&#8217;s called "trap to debugger".

Without further ado, I&#8217;ll quote from the bible itself [[2]][3]:

> The INT 3 instruction generates a special one byte opcode (CC) that is intended for calling   
> the debug exception handler. (This one byte form is valuable because it can be used to replace   
> the first byte of any instruction with a breakpoint, including other one byte instructions, without   
> over-writing other code).

The part in parens is important, but it&#8217;s still too early to explain it. We&#8217;ll come back to it later in this article.

##### int 3 in practice

Yes, knowing the theory behind things is great, OK, but what does this really mean? How do we use <tt>int 3</tt>to implement breakpoints? Or to paraphrase common programming Q&A jargon &#8211; _Plz show me the codes!_

In practice, this is really very simple. Once your process executes the <tt>int 3</tt> instruction, the OS stops it [[3]][4]. On Linux (which is what we&#8217;re concerned with in this article) it then sends the process a signal &#8211;<tt>SIGTRAP</tt>.

That&#8217;s all there is to it &#8211; honest! Now recall from the first part of the series that a tracing (debugger) process gets notified of all the signals its child (or the process it attaches to for debugging) gets, and you can start getting a feel of where we&#8217;re going.

That&#8217;s it, no more computer architecture 101 jabber. It&#8217;s time for examples and code.

##### Setting breakpoints manually

I&#8217;m now going to show code that sets a breakpoint in a program. The target program I&#8217;m going to use for this demonstration is the following:

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
> &#160;&#160;&#160; mov&#160;&#160;&#160;&#160; edx, len1   
> &#160;&#160;&#160; mov&#160;&#160;&#160;&#160; ecx, msg1   
> &#160;&#160;&#160; mov&#160;&#160;&#160;&#160; ebx, 1   
> &#160;&#160;&#160; mov&#160;&#160;&#160;&#160; eax, 4
> 
> &#160;&#160;&#160; ; Execute the sys_write system call   
> &#160;&#160;&#160; int&#160;&#160;&#160;&#160; 0x80
> 
> &#160;&#160;&#160; ; Now print the other message   
> &#160;&#160;&#160; mov&#160;&#160;&#160;&#160; edx, len2   
> &#160;&#160;&#160; mov&#160;&#160;&#160;&#160; ecx, msg2   
> &#160;&#160;&#160; mov&#160;&#160;&#160;&#160; ebx, 1   
> &#160;&#160;&#160; mov&#160;&#160;&#160;&#160; eax, 4   
> &#160;&#160;&#160; int&#160;&#160;&#160;&#160; 0x80
> 
> &#160;&#160;&#160; ; Execute sys_exit   
> &#160;&#160;&#160; mov&#160;&#160;&#160;&#160; eax, 1   
> &#160;&#160;&#160; int&#160;&#160;&#160;&#160; 0x80
> 
> section&#160;&#160;&#160; .data
> 
> msg1&#160;&#160;&#160; db&#160;&#160;&#160;&#160;&#160; &#8216;Hello,&#8217;, 0xa   
> len1&#160;&#160;&#160; equ&#160;&#160;&#160;&#160; $ &#8211; msg1   
> msg2&#160;&#160;&#160; db&#160;&#160;&#160;&#160;&#160; &#8216;world!&#8217;, 0xa   
> len2&#160;&#160;&#160; equ&#160;&#160;&#160;&#160; $ &#8211; msg2

I&#8217;m using assembly language for now, in order to keep us clear of compilation issues and symbols that come up when we get into C code. What the program listed above does is simply print "Hello," on one line and then "world!" on the next line. It&#8217;s very similar to the program demonstrated in the previous article.

I want to set a breakpoint after the first printout, but before the second one. Let&#8217;s say right after the first <tt>int 0x80</tt> [[4]][5], on the <tt>mov edx, len2</tt> instruction. First, we need to know what address this instruction maps to. Running <tt>objdump -d</tt>:

> traced_printer2:&#160;&#160;&#160;&#160; file format elf32-i386
> 
> Sections:   
> Idx Name&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; Size&#160;&#160;&#160;&#160;&#160; VMA&#160;&#160;&#160;&#160;&#160;&#160; LMA&#160;&#160;&#160;&#160;&#160;&#160; File off&#160; Algn   
> &#160; 0 .text&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; 00000033&#160; 08048080&#160; 08048080&#160; 00000080&#160; 2**4   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; CONTENTS, ALLOC, LOAD, READONLY, CODE   
> &#160; 1 .data&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; 0000000e&#160; 080490b4&#160; 080490b4&#160; 000000b4&#160; 2**2   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; CONTENTS, ALLOC, LOAD, DATA
> 
> Disassembly of section .text:
> 
> 08048080 <.text>:   
> 8048080:&#160;&#160;&#160;&#160; ba 07 00 00 00&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; mov&#160;&#160;&#160; $0x7,%edx   
> 8048085:&#160;&#160;&#160;&#160; b9 b4 90 04 08&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; mov&#160;&#160;&#160; $0x80490b4,%ecx   
> 804808a:&#160;&#160;&#160;&#160; bb 01 00 00 00&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; mov&#160;&#160;&#160; $0x1,%ebx   
> 804808f:&#160;&#160;&#160;&#160; b8 04 00 00 00&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; mov&#160;&#160;&#160; $0x4,%eax   
> 8048094:&#160;&#160;&#160;&#160; cd 80&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; int&#160;&#160;&#160; $0x80   
> 8048096:&#160;&#160;&#160;&#160; ba 07 00 00 00&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; mov&#160;&#160;&#160; $0x7,%edx   
> 804809b:&#160;&#160;&#160;&#160; b9 bb 90 04 08&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; mov&#160;&#160;&#160; $0x80490bb,%ecx   
> 80480a0:&#160;&#160;&#160;&#160; bb 01 00 00 00&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; mov&#160;&#160;&#160; $0x1,%ebx   
> 80480a5:&#160;&#160;&#160;&#160; b8 04 00 00 00&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; mov&#160;&#160;&#160; $0x4,%eax   
> 80480aa:&#160;&#160;&#160;&#160; cd 80&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; int&#160;&#160;&#160; $0x80   
> 80480ac:&#160;&#160;&#160;&#160; b8 01 00 00 00&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; mov&#160;&#160;&#160; $0x1,%eax   
> 80480b1:&#160;&#160;&#160;&#160; cd 80&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; int&#160;&#160;&#160; $0x80

So, the address we&#8217;re going to set the breakpoint on is 0x8048096. Wait, this is not how real debuggers work, right? Real debuggers set breakpoints on lines of code and on functions, not on some bare memory addresses? Exactly right. But we&#8217;re still far from there &#8211; to set breakpoints like _real_ debuggers we still have to cover symbols and debugging information first, and it will take another part or two in the series to reach these topics. For now, we&#8217;ll have to do with bare memory addresses.

At this point I really want to digress again, so you have two choices. If it&#8217;s really interesting for you to know _why_ the address is 0x8048096 and what does it mean, read the next section. If not, and you just want to get on with the breakpoints, you can safely skip it.

##### Digression &#8211; process addresses and entry point

Frankly, 0x8048096 itself doesn&#8217;t mean much, it&#8217;s just a few bytes away from the beginning of the text section of the executable. If you look carefully at the dump listing above, you&#8217;ll see that the text section starts at 0x08048080. This tells the OS to map the text section starting at this address in the virtual address space given to the process. On Linux these addresses can be absolute (i.e. the executable isn&#8217;t being relocated when it&#8217;s loaded into memory), because with the virtual memory system each process gets its own chunk of memory and sees the whole 32-bit address space as its own (called "linear" address).

If we examine the ELF [[5]][6] header with <tt>readelf</tt>, we get:

> $ readelf -h traced_printer2   
> ELF Header:   
> &#160; Magic:&#160;&#160; 7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00   
> &#160; Class:&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; ELF32   
> &#160; Data:&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; 2&#8217;s complement, little endian   
> &#160; Version:&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; 1 (current)   
> &#160; OS/ABI:&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; UNIX &#8211; System V   
> &#160; ABI Version:&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; 0   
> &#160; Type:&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; EXEC (Executable file)   
> &#160; Machine:&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; Intel 80386   
> &#160; Version:&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; 0x1   
> &#160; Entry point address:&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; 0x8048080   
> &#160; Start of program headers:&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; 52 (bytes into file)   
> &#160; Start of section headers:&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; 220 (bytes into file)   
> &#160; Flags:&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; 0x0   
> &#160; Size of this header:&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; 52 (bytes)   
> &#160; Size of program headers:&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; 32 (bytes)   
> &#160; Number of program headers:&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; 2   
> &#160; Size of section headers:&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; 40 (bytes)   
> &#160; Number of section headers:&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; 4   
> &#160; Section header string table index: 3

Note the "entry point address" section of the header, which also points to 0x8048080. So if we interpret the directions encoded in the ELF file for the OS, it says:

  1. Map the text section (with given contents) to address 0x8048080 
  2. Start executing at the entry point &#8211; address 0x8048080 

But still, why 0x8048080? For historic reasons, it turns out. Some googling led me to a few sources that claim that the first 128MB of each process&#8217;s address space were reserved for the stack. 128MB happens to be 0x8000000, which is where other sections of the executable may start. 0x8048080, in particular, is the default entry point used by the Linux <tt>ld</tt> linker. This entry point can be modified by passing the <tt>-Ttext</tt>argument to <tt>ld</tt>.

To conclude, there&#8217;s nothing really special in this address and we can freely change it. As long as the ELF executable is properly structured and the entry point address in the header matches the real beginning of the program&#8217;s code (text section), we&#8217;re OK.

##### Setting breakpoints in the debugger with int 3

To set a breakpoint at some target address in the traced process, the debugger does the following:

  1. Remember the data stored at the target address 
  2. Replace the first byte at the target address with the int 3 instruction 

Then, when the debugger asks the OS to run the process (with <tt>PTRACE_CONT</tt> as we saw in the previous article), the process will run and eventually hit upon the <tt>int 3</tt>, where it will stop and the OS will send it a signal. This is where the debugger comes in again, receiving a signal that its child (or traced process) was stopped. It can then:

  1. Replace the int 3 instruction at the target address with the original instruction 
  2. Roll the instruction pointer of the traced process back by one. This is needed because the instruction pointer now points _after_ the <tt>int 3</tt>, having already executed it. 
  3. Allow the user to interact with the process in some way, since the process is still halted at the desired target address. This is the part where your debugger lets you peek at variable values, the call stack and so on. 
  4. When the user wants to keep running, the debugger will take care of placing the breakpoint back (since it was removed in step 1) at the target address, unless the user asked to cancel the breakpoint. 

Let&#8217;s see how some of these steps are translated into real code. We&#8217;ll use the debugger "template" presented in part 1 (forking a child process and tracing it). In any case, there&#8217;s a link to the full source code of this example at the end of the article.

> /\* Obtain and show child&#8217;s instruction pointer \*/   
> ptrace(PTRACE\_GETREGS, child\_pid, 0, &regs);   
> procmsg("Child started. EIP = 0x%08x\n", regs.eip);
> 
> /\* Look at the word at the address we&#8217;re interested in \*/   
> unsigned addr = 0x8048096;   
> unsigned data = ptrace(PTRACE\_PEEKTEXT, child\_pid, (void*)addr, 0);   
> procmsg("Original data at 0x%08x: 0x%08x\n", addr, data);

Here the debugger fetches the instruction pointer from the traced process, as well as examines the word currently present at 0x8048096. When run tracing the assembly program listed in the beginning of the article, this prints:

> [13028] Child started. EIP = 0x08048080   
> [13028] Original data at 0x08048096: 0x000007ba

So far, so good. Next:

> /\* Write the trap instruction &#8216;int 3&#8217; into the address \*/   
> unsigned data\_with\_trap = (data & 0xFFFFFF00) | 0xCC;   
> ptrace(PTRACE\_POKETEXT, child\_pid, (void\*)addr, (void\*)data\_with\_trap);
> 
> /\* See what&#8217;s there again&#8230; \*/   
> unsigned readback\_data = ptrace(PTRACE\_PEEKTEXT, child_pid, (void*)addr, 0);   
> procmsg("After trap, data at 0x%08x: 0x%08x\n", addr, readback_data);

Note how <tt>int 3</tt> is inserted at the target address. This prints:

> [13028] After trap, data at 0x08048096: 0x000007cc

Again, as expected &#8211; <tt>0xba</tt> was replaced with <tt>0xcc</tt>. The debugger now runs the child and waits for it to halt on the breakpoint:

> /* Let the child run to the breakpoint and wait for it to   
> ** reach it   
> */   
> ptrace(PTRACE\_CONT, child\_pid, 0, 0);
> 
> wait(&wait_status);   
> if (WIFSTOPPED(wait_status)) {   
> &#160;&#160;&#160; procmsg("Child got a signal: %s\n", strsignal(WSTOPSIG(wait_status)));   
> }   
> else {   
> &#160;&#160;&#160; perror("wait");   
> &#160;&#160;&#160; return;   
> }
> 
> /\* See where the child is now \*/   
> ptrace(PTRACE\_GETREGS, child\_pid, 0, &regs);   
> procmsg("Child stopped at EIP = 0x%08x\n", regs.eip);

This prints:

> Hello,   
> [13028] Child got a signal: Trace/breakpoint trap   
> [13028] Child stopped at EIP = 0x08048097

Note the "Hello," that was printed before the breakpoint &#8211; exactly as we planned. Also note where the child stopped &#8211; just after the single-byte trap instruction.

Finally, as was explained earlier, to keep the child running we must do some work. We replace the trap with the original instruction and let the process continue running from it.

> /* Remove the breakpoint by restoring the previous data   
> ** at the target address, and unwind the EIP back by 1 to   
> ** let the CPU execute the original instruction that was   
> ** there.   
> */   
> ptrace(PTRACE\_POKETEXT, child\_pid, (void\*)addr, (void\*)data);   
> regs.eip -= 1;   
> ptrace(PTRACE\_SETREGS, child\_pid, 0, &regs);
> 
> /\* The child can continue running now \*/   
> ptrace(PTRACE\_CONT, child\_pid, 0, 0);

This makes the child print "world!" and exit, just as planned.

Note that we don&#8217;t restore the breakpoint here. That can be done by executing the original instruction in single-step mode, then placing the trap back and only then do <tt>PTRACE_CONT</tt>. The debug library demonstrated later in the article implements this.

##### More on int 3

Now is a good time to come back and examine <tt>int 3</tt> and that curious note from Intel&#8217;s manual. Here it is again:

> This one byte form is valuable because it can be used to replace the first byte of any instruction   
> with a breakpoint, including other one byte instructions, without over-writing other code.

<tt>int</tt> instructions on x86 occupy two bytes &#8211; <tt>0xcd</tt> followed by the interrupt number [[6]][7]. <tt>int 3</tt> could&#8217;ve been encoded as <tt>cd 03</tt>, but there&#8217;s a special single-byte instruction reserved for it &#8211; 0xcc.

Why so? Because this allows us to insert a breakpoint without ever overwriting more than one instruction. And this is important. Consider this sample code:

> &#160;&#160;&#160; .. some code ..   
> &#160;&#160;&#160; jz&#160;&#160;&#160; foo   
> &#160;&#160;&#160; dec&#160;&#160; eax   
> foo:   
> &#160;&#160;&#160; call&#160; bar   
> &#160;&#160;&#160; .. some code ..

Suppose we want to place a breakpoint on <tt>dec eax</tt>. This happens to be a single-byte instruction (with the opcode <tt>0x48</tt>). Had the replacement breakpoint instruction been longer than 1 byte, we&#8217;d be forced to overwrite part of the next instruction (<tt>call</tt>), which would garble it and probably produce something completely invalid. But what is the branch <tt>jz foo</tt> was taken? Then, without stopping on <tt>dec eax</tt>, the CPU would go straight to execute the invalid instruction after it.

Having a special 1-byte encoding for <tt>int 3</tt> solves this problem. Since 1 byte is the shortest an instruction can get on x86, we guarantee than only the instruction we want to break on gets changed.

##### Encapsulating some gory details

Many of the low-level details shown in code samples of the previous section can be easily encapsulated behind a convenient API. I&#8217;ve done some encapsulation into a small utility library called <tt>debuglib</tt> &#8211; its code is available for download at the end of the article. Here I just want to demonstrate an example of its usage, but with a twist. We&#8217;re going to trace a program written in C.

##### Tracing a C program

So far, for the sake of simplicity, I focused on assembly language targets. It&#8217;s time to go one level up and see how we can trace a program written in C.

It turns out things aren&#8217;t very different &#8211; it&#8217;s just a bit harder to find where to place the breakpoints. Consider this simple program:

> #include <stdio.h>
> 
> void do_stuff()   
> {   
> &#160;&#160;&#160; printf("Hello, ");   
> }
> 
> int main()   
> {   
> &#160;&#160;&#160; for (int i = 0; i < 4; ++i)   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160; do_stuff();   
> &#160;&#160;&#160; printf("world!\n");   
> &#160;&#160;&#160; return 0;   
> }

Suppose I want to place a breakpoint at the entrance to <tt>do_stuff</tt>. I&#8217;ll use the old friend <tt>objdump</tt> to disassemble the executable, but there&#8217;s a lot in it. In particular, looking at the <tt>text</tt> section is a bit useless since it contains a lot of C runtime initialization code I&#8217;m currently not interested in. So let&#8217;s just look for <tt>do_stuff</tt> in the dump:

> 080483e4 <do_stuff>:   
> 80483e4:&#160;&#160;&#160;&#160; 55&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; push&#160;&#160; %ebp   
> 80483e5:&#160;&#160;&#160;&#160; 89 e5&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; mov&#160;&#160;&#160; %esp,%ebp   
> 80483e7:&#160;&#160;&#160;&#160; 83 ec 18&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; sub&#160;&#160;&#160; $0x18,%esp   
> 80483ea:&#160;&#160;&#160;&#160; c7 04 24 f0 84 04 08&#160;&#160;&#160; movl&#160;&#160; $0x80484f0,(%esp)   
> 80483f1:&#160;&#160;&#160;&#160; e8 22 ff ff ff&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; call&#160;&#160; 8048318 <puts@plt>   
> 80483f6:&#160;&#160;&#160;&#160; c9&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; leave   
> 80483f7:&#160;&#160;&#160;&#160; c3&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; ret

Alright, so we&#8217;ll place the breakpoint at 0x080483e4, which is the first instruction of <tt>do_stuff</tt>. Moreover, since this function is called in a loop, we want to keep stopping at the breakpoint until the loop ends. We&#8217;re going to use the <tt>debuglib</tt> library to make this simple. Here&#8217;s the complete debugger function:

> void run\_debugger(pid\_t child_pid)   
> {   
> &#160;&#160;&#160; procmsg("debugger started\n");
> 
> &#160;&#160;&#160; /\* Wait for child to stop on its first instruction \*/   
> &#160;&#160;&#160; wait(0);   
> &#160;&#160;&#160; procmsg("child now at EIP = 0x%08x\n", get\_child\_eip(child_pid));
> 
> &#160;&#160;&#160; /\* Create breakpoint and run to it\*/   
> &#160;&#160;&#160; debug\_breakpoint\* bp = create\_breakpoint(child_pid, (void\*)0x080483e4);   
> &#160;&#160;&#160; procmsg("breakpoint created\n");   
> &#160;&#160;&#160; ptrace(PTRACE\_CONT, child\_pid, 0, 0);   
> &#160;&#160;&#160; wait(0);
> 
> &#160;&#160;&#160; /\* Loop as long as the child didn&#8217;t exit \*/   
> &#160;&#160;&#160; while (1) {   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160; /* The child is stopped at a breakpoint here. Resume its   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160; ** execution until it either exits or hits the   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160; ** breakpoint again.   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160; */   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160; procmsg("child stopped at breakpoint. EIP = 0x%08X\n", get\_child\_eip(child_pid));   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160; procmsg("resuming\n");   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160; int rc = resume\_from\_breakpoint(child_pid, bp);
> 
> &#160;&#160;&#160;&#160;&#160;&#160;&#160; if (rc == 0) {   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; procmsg("child exited\n");   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; break;   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160; }   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160; else if (rc == 1) {   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; continue;   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160; }   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160; else {   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; procmsg("unexpected: %d\n", rc);   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; break;   
> &#160;&#160;&#160;&#160;&#160;&#160;&#160; }   
> &#160;&#160;&#160; }
> 
> &#160;&#160;&#160; cleanup_breakpoint(bp);   
> }

Instead of getting our hands dirty modifying EIP and the target process&#8217;s memory space, we just use<tt>create_breakpoint</tt>, <tt>resume_from_breakpoint</tt> and <tt>cleanup_breakpoint</tt>. Let&#8217;s see what this prints when tracing the simple C code displayed above:

> $ bp\_use\_lib traced\_c\_loop   
> [13363] debugger started   
> [13364] target started. will run &#8216;traced\_c\_loop&#8217;   
> [13363] child now at EIP = 0x00a37850   
> [13363] breakpoint created   
> [13363] child stopped at breakpoint. EIP = 0x080483E5   
> [13363] resuming   
> Hello,   
> [13363] child stopped at breakpoint. EIP = 0x080483E5   
> [13363] resuming   
> Hello,   
> [13363] child stopped at breakpoint. EIP = 0x080483E5   
> [13363] resuming   
> Hello,   
> [13363] child stopped at breakpoint. EIP = 0x080483E5   
> [13363] resuming   
> Hello,   
> world!   
> [13363] child exited

Just as expected!

##### The code

[Here are][8] the complete source code files for this part. In the archive you&#8217;ll find:

  * debuglib.h and debuglib.c &#8211; the simple library for encapsulating some of the inner workings of a debugger 
  * bp_manual.c &#8211; the "manual" way of setting breakpoints presented first in this article. Uses the<tt>debuglib</tt> library for some boilerplate code. 
  * bp\_use\_lib.c &#8211; uses <tt>debuglib</tt> for most of its code, as demonstrated in the second code sample for tracing the loop in a C program. 

##### Conclusion and next steps

We&#8217;ve covered how breakpoints are implemented in debuggers. While implementation details vary between OSes, when you&#8217;re on x86 it&#8217;s all basically variations on the same theme &#8211; substituting <tt>int 3</tt> for the instruction where we want the process to stop.

That said, I&#8217;m sure some readers, just like me, will be less than excited about specifying raw memory addresses to break on. We&#8217;d like to say "break on <tt>do_stuff</tt>", or even "break on _this_ line in <tt>do_stuff</tt>" and have the debugger do it. In the next article I&#8217;m going to show how it&#8217;s done.

##### References

I&#8217;ve found the following resources and articles useful in the preparation of this article:

  * [How debugger works][9] 
  * [Understanding ELF using readelf and objdump][10] 
  * [Implementing breakpoints on x86 Linux][11] 
  * [NASM manual][12] 
  * [SO discussion of the ELF entry point][13] 
  * [This Hacker News discussion][14] of the first part of the series 
  * [GDB Internals][15]&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160; 
&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;&#160;  ![][16]</ul> 

[[1]][17] On a high-level view this is true. Down in the gory details, many CPUs today execute multiple instructions in parallel, some of them [not in their original order][18].

[[2]][19] The bible in this case being, of course, Intel&#8217;s Architecture software developer&#8217;s manual, volume 2A.

[[3]][20] How can the OS stop a process just like that? The OS registered its own handler for <tt>int 3</tt> with the CPU, that&#8217;s how!

[[4]][21] Wait, <tt>int</tt> again? Yes! Linux uses <tt>int 0x80</tt> to implement system calls from user processes into the OS kernel. The user places the number of the system call and its arguments into registers and executes<tt>int 0x80</tt>. The CPU then jumps to the appropriate interrupt handler, where the OS registered a procedure that looks at the registers and decides which system call to execute.

[[5]][22] [ELF][23] (Executable and Linkable Format) is the file format used by Linux for object files, shared libraries and executables.

[[6]][24] An observant reader can spot the translation of <tt>int 0x80</tt> into <tt>cd 80</tt> in the dumps listed above.

 [1]: http://eli.thegreenplace.net/2011/01/23/how-debuggers-work-part-1/
 [2]: http://eli.thegreenplace.net/2011/01/27/how-debuggers-work-part-2-breakpoints#id7
 [3]: http://eli.thegreenplace.net/2011/01/27/how-debuggers-work-part-2-breakpoints#id8
 [4]: http://eli.thegreenplace.net/2011/01/27/how-debuggers-work-part-2-breakpoints#id9
 [5]: http://eli.thegreenplace.net/2011/01/27/how-debuggers-work-part-2-breakpoints#id10
 [6]: http://eli.thegreenplace.net/2011/01/27/how-debuggers-work-part-2-breakpoints#id11
 [7]: http://eli.thegreenplace.net/2011/01/27/how-debuggers-work-part-2-breakpoints#id12
 [8]: https://github.com/eliben/code-for-blog/tree/master/2011/debuggers_part2_code
 [9]: http://www.alexonlinux.com/how-debugger-works
 [10]: http://www.linuxforums.org/articles/understanding-elf-using-readelf-and-objdump_125.html
 [11]: http://mainisusuallyafunction.blogspot.com/2011/01/implementing-breakpoints-on-x86-linux.html
 [12]: http://www.nasm.us/xdoc/2.09.04/html/nasmdoc0.html
 [13]: http://stackoverflow.com/questions/2187484/elf-binary-entry-point
 [14]: http://news.ycombinator.net/item?id=2131894
 [15]: http://www.deansys.com/doc/gdbInternals/gdbint_toc.html
 [16]: http://eli.thegreenplace.net/images/hline.jpg
 [17]: http://eli.thegreenplace.net/2011/01/27/how-debuggers-work-part-2-breakpoints#id1
 [18]: http://en.wikipedia.org/wiki/Out-of-order_execution
 [19]: http://eli.thegreenplace.net/2011/01/27/how-debuggers-work-part-2-breakpoints#id2
 [20]: http://eli.thegreenplace.net/2011/01/27/how-debuggers-work-part-2-breakpoints#id3
 [21]: http://eli.thegreenplace.net/2011/01/27/how-debuggers-work-part-2-breakpoints#id4
 [22]: http://eli.thegreenplace.net/2011/01/27/how-debuggers-work-part-2-breakpoints#id5
 [23]: http://en.wikipedia.org/wiki/Executable_and_Linkable_Format
 [24]: http://eli.thegreenplace.net/2011/01/27/how-debuggers-work-part-2-breakpoints#id6