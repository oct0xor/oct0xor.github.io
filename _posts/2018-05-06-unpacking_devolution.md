---
layout: page
title: Unpacking Devolution (PowerPC instrumentation on PC)
---

Recently I had seen a [tweet](https://twitter.com/marcan42/status/954263027917799425) of Marcan (Team Twiizers) about obfuscations that they are implemented in Homebrew Channel. I did not try to unpack Homebrew Channel before but it reminded me about how I unpacked another Wii homebrew called Devolution.

I would like this post to be a tutorial about how I unpacked it and show the easiest way of how to deal with similar cases.

Topics of this post:

 - How to debug code compiled for other architectures with the ability to instrument it.
 - How to fight obscurities and unpack exotic packers.

Often I hear a common opinion is that if you have some non-x86 code that you want to debug the easiest way to do this is to get platform that is based on the same architecture, somehow load this code there and debug with available tools. 

I even knew a person who used to have Power Mac G5 for solely purpose to debug PowerPC code.

This is wrong attitude. These things are much easier to do with QEMU. 

QEMU supports big variety of architectures and CPU's, and even if your code needs some specific features that is not implemented in QEMU, quite often you can avoid or implement it quite easy, and I will show you how.

Before I began writing this post, I wanted to learn something new and try [LuaQEMU](https://github.com/Comsecuris/luaqemu) but then I realized that it works only with ARM, therefore I will show you my original way that will work in almost all cases.

We downloaded this [homebrew](https://gbatemp.net/threads/devolution-public-release.330554/), unpacked zip archive and looked at executable file **boot.dol**.

Code in .text1 section is simply a loader that will jump to payload located in .data1 section.

<div align="center">
    <img src ="/assets/devolution_1.png"/>
</div>

The first two functions of payload copy the rest of payload at 0x80004000 and jumps to it with **Return from Interrupt** (rfi) instruction.

<div align="center">
    <img src ="/assets/devolution_2.png"/>
</div>

After setting of special purpose registers, it flushes cache for something that takes most of space in payload - some high entropy data and drama IRC logs from the days long gone.

<div align="center">
    <img src ="/assets/devolution_3.png"/>
</div>

Reverse engineering of what looks like the main function gives us two things: this payload contains some kind of **Real-Time Operating System** and function sub_80007D34 **generates vector tables**.

<div align="center">
    <img src ="/assets/devolution_4.png"/>
</div>

All hardware interrupts are passed to exception handler located at address 80007ED4. The most interesting part of it is that it has their own **syscall handler**.

<div align="center">
    <img src ="/assets/devolution_5.png"/>
</div>

Going back and observing main_thread makes it clear that RTOS is a part of obfuscation: it creates much more threads and events, they are interact with each other and to follow execution is very hard.

<div align="center">
    <img src ="/assets/devolution_6.png"/>
</div>

Thread1, thread2, thread4, thread5 do data ping-pong and thread3 executes decryption functions. It makes it clear that decrypting this statically is impractical.

Finally, when all threads execution is completed the next decrypted stage is executed with **mtctr** / **bctrl** instructions.

Our goal is to get next decrypted stage and we do not want to do this statically. What should we do?

There are two options:
- Patch payload at place of mtctr / bctrl instructions, put a code that will dump what at address pointed by r0 and launch patched payload on real hardware.
This is still hard because there might be additional code integrity checks, and you will need to spend time looking for them. In addition, you should know Wii architecture very well and should have a way and code to achieve this dumping process.
- 2nd option is to emulate and debug this payload. Luckily for us, it is possible with QEMU.

I have made a simple machine definition file for [Broadway](https://github.com/oct0xor/unpacking_devolution/blob/master/broadway.c) (codename for Wii CPU), with a very abstract memory layout. All what it does is sets memory and loads file payload.bin at address 0x80004000.

We save this machine definition as hw\ppc\broadway.c, change makefile and recompile QEMU. Then we start QEMU with a command "./qemu-system-ppc -S -gdb tcp::20000,ipv4 -M broadway" and attach to it with GDB from IDA Pro.

<div align="center">
    <img src ="/assets/devolution_7.png"/>
</div>

First thing we realize is that QEMU does not support **dcbz_l** instruction, so we will set pc = pc + 4 (PowerPC instructions are always 4 bytes long).

<div align="center">
    <img src ="/assets/devolution_8.png"/>
</div>

Then we realize that in IDA Pro 6.5+ changing the value of PC register is broken.

<div align="center">
    <img src ="/assets/devolution_9.png"/>
</div>

This particular issue seems to be fixed in IDA Pro 7.0 but there it does not seems to be able to walk over "rfi" instructions :( .

I would recommend using IDA Pro 6.1, there _almost_ everything works as intended.

<div align="center">
    <img src ="/assets/devolution_a.png"/>
</div>

If we will try to set breakpoint in IDA Pro 6.1 it will not work because it tries to set breakpoint with type **GDB_WATCHPOINT_ACCESS**.

<div align="center">
    <img src ="/assets/devolution_b.png"/>
</div>

As it's not possible to set valid breakpoint with GUI we will need to set it implicitly with a command **AddBptEx(ScreenEA(), 1, BPT_EXEC)**;

So we set PC register to point at next basic block after dcbz_l instruction, set breakpoint at bctrl when payload should be decrypted and hit go... and it's never hit.

After some more reverse engineering and debugging, we find out that we fail a check in one function executed in thread3.

<div align="center">
    <img src ="/assets/devolution_c.png"/>
</div>

Data that is compared here initially was stored at address 0xF0000000, and before that, it was xor'ed in thread2.

<div align="center">
    <img src ="/assets/devolution_d.png"/>
</div>

In our case, this buffer is empty and this is a reason why this check is failed. 

We can guess that it should have been populated in a previously executed thread5. It looks like an address 0xF0000000 is passed here to **special purpose register 0x39b**.

<div align="center">
    <img src ="/assets/devolution_e.png"/>
</div>

Reading Broadway specification reveals us that special purpose registers 922 and 923 are used for **DMA** (direct memory access). 
Of course this special purpose registers are not supported by QEMU as it is Broadway specific functionality. 

And this is where IDA Pro's scripting engine comes handy.

Of course, we can learn QEMU internals and implement this missing functionality in CPU, in this case, it would be enough to implement this MSR's in translate_init.c, but this post is not about that. **IDA Pro's scripting gives us a general way to instrument debugged code**. Also QEMU does not stand still and things that worked today might not work on next revision and you will need to spend more time to relearn how to use it again. For example this was a case when I was porting my machine definitions from 1.*.* to 2.11.1.

I created a python [script](https://github.com/oct0xor/unpacking_devolution/blob/master/devo_unpack.py), implemented DMAU/DMAL algorithm, and with IDA Pro's debug API the rest was easy. Unpacked payload is saved to devo_dump.bin.

<div align="center">
    <img src ="/assets/devolution_f.png"/>
</div>

I hope that you will find this technique useful. The complications that I faced with IDA Pro's GDB client and QEMU were only present because PPC is not a common architecture and IDA Pro's GDB plugin probably wasn't well tested to work with it. For example, everything works like a charm with ARM.

Speaking about Devolution it was very fun to reverse this packer. It is clear that author knows a system on a very low level. However, interesting to notice about a flaw that would let to get unpacked payload [without any reverse engineering](https://gbatemp.net/threads/nintendont.349258/page-127#post-4937545). 

{% include twitter_plug.html %}