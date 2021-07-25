---
layout: post
title: "Understanding qemu logs for debugging"
date: 2021-7-24
tags:
  - qemu
  - emulator
  - vm
  - x86
author: Tim Thompson
avatar: assets/profile.png
category: Debugging
---

Hello everyone, today I wanted to talk about the output of Qemu's `-d int` flag.

If you don't know what qemu is, it's a widespread emulator supporting a range of architectures, in this post we will focus on `qemu-system-x86_64`.

The flag `-d int` will dump registers whenever an interrupt or something internal to qemu occurs. (Fun fact: this command-line option was created to debug qemu itself however it has equal use to us give insight as to what the guest binary is doing without having to start a debugger session)

### Requirements

This post will assume that you have a basic understanding of x86, including but not limited to #GP registers, segments, interrupts, and privilege levels (rings)

### Qemu output simplification

Let's start off by looking at an example:
```c
8: v=20 e=0000 i=0 cpl=0 IP=0008:ffffffff80200d38 pc=ffffffff80200d38 SP=0010:ffffffff80426963 env->regs[R_EAX]=0000000000023c18
RAX=0000000000023c18 RBX=0000000068747541 RCX=0000000000001000 RDX=0000000000000070
RSI=0000000000000000 RDI=000000000000000a RBP=ffffffff804269c3 RSP=ffffffff80426963
R8 =0000000000000400 R9 =00000000fd000000 R10=ffffffff80326b97 R11=000000000000001e
R12=ffffffff8020607f R13=0000000000000000 R14=0000000000000000 R15=0000000000000000
RIP=ffffffff80200d38 RFL=00000293 [--S-A-C] CPL=0 II=0 A20=1 SMM=0 HLT=0
ES =0010 0000000000000000 00000000 00009700 DPL=0 DS   [EWA]
CS =0008 0000000000000000 00000000 00209a00 DPL=0 CS64 [-R-]
SS =0010 0000000000000000 00000000 00009700 DPL=0 DS   [EWA]
DS =0010 0000000000000000 00000000 00009700 DPL=0 DS   [EWA]
FS =0010 0000000000000000 00000000 00009700 DPL=0 DS   [EWA]
GS =0010 0000000000000000 00000000 00009700 DPL=0 DS   [EWA]
LDT=0000 0000000000000000 0000ffff 00008200 DPL=0 LDT
TR =0000 0000000000000000 0000ffff 00008b00 DPL=0 TSS64-busy
GDT=     ffffffff80426bc0 00000027
IDT=     ffffffff80427400 00000fff
CR0=80000011 CR2=0000000000000000 CR3=0000000007fce000 CR4=00000020
DR0=0000000000000000 DR1=0000000000000000 DR2=0000000000000000 DR3=0000000000000000 
DR6=00000000ffff0ff0 DR7=0000000000000400
CCS=0000000000000091 CCD=fffffffffffe6b89 CCO=EFLAGS  
EFER=0000000000000500
```

As you can see this output is very verbose, so let's narrow it down to the things that will matter the most to you when debugging your kernel

```c
IP=0008:ffffffff80200d38 SP=0010:ffffffff80426963
RAX=0000000000023c18 RBX=0000000068747541 RCX=0000000000001000 RDX=0000000000000070
RSI=0000000000000000 RDI=000000000000000a RBP=ffffffff804269c3 RSP=ffffffff80426963
R8 =0000000000000400 R9 =00000000fd000000 R10=ffffffff80326b97 R11=000000000000001e
R12=ffffffff8020607f R13=0000000000000000 R14=0000000000000000 R15=0000000000000000
RIP=ffffffff80200d38 CPL=0 A20=1 HLT=0
ES =0010 0000000000000000 00000000 00009700 DPL=0 DS   [EWA]
CS =0008 0000000000000000 00000000 00209a00 DPL=0 CS64 [-R-]
SS =0010 0000000000000000 00000000 00009700 DPL=0 DS   [EWA]
DS =0010 0000000000000000 00000000 00009700 DPL=0 DS   [EWA]
FS =0010 0000000000000000 00000000 00009700 DPL=0 DS   [EWA]
GS =0010 0000000000000000 00000000 00009700 DPL=0 DS   [EWA]
LDT=0000 0000000000000000 0000ffff 00008200 DPL=0 LDT
TR =0000 0000000000000000 0000ffff 00008b00 DPL=0 TSS64-busy
GDT=     ffffffff80426bc0 00000027
IDT=     ffffffff80427400 00000fff
CR0=80000011 CR2=0000000000000000 CR3=0000000007fce000 CR4=00000020
DR0=0000000000000000 DR1=0000000000000000 DR2=0000000000000000 DR3=0000000000000000 
DR6=00000000ffff0ff0 DR7=0000000000000400
EFER=0000000000000500
```

### Now let's go through this line by line:

`IP=0008:ffffffff80200d38 SP=0010:ffffffff80426963`

- IP  = IP is the instruction pointer, and it is displayed in a segment:offset addressing format which can be quite handy when working on a gdt. The format is CS:ADDRESS because IP is pointing to code that is intended to be executed.
- SP  = SP is the stack pointer, also shown in the segment:offset format. This format is DS:ADDRESS as the stack will be referencing data. You might be wondering: "Wouldn't I use SS:ADDRESS since I'm using the stack?", no not really. The registers SP and BP (base pointer) are merely an _index_ into the stack, so addressing with SS:ADDRESS would mess the entire stack up and cause you some **big** trouble.

`RAX...R15` ~ This is pretty simple as it's just a dump of #GP registers. No more, no less.

`CPL=0 A20=1 HLT=0`

- CPL = Current privilege level (Also known as the ring level, here it is `0` indicating that we are running in ring 0 (kernel mode))
- A20 = If this is set (1) then the A20 line has been enabled. This is done for you in bootloaders such as limine. Here's a short summary: The a20 line is an address line that had to be added for the IBM-AT which consisted of an i286 and 16MB of memory in order to allow backwards compatibility for the 8086. If it wasn't enabled it could only access 1MB of memory + 64KB due to segment registers. Enabling it would allow access to all memory, otherwise the address lines would wrap around to A0.
- HLT = If this is set, then the processor has been stopped which is generally not a good thing (Basically a visual representation of the state of the `hlt` instruction)

`ES =0010 0000000000000000 00000000 00009700 DPL=0`

- ES  = In this case the Extra Segment. The output is the same for all segment registers below, again it is assumed that you know about segment registers and their abbreviations.
- DPL = The `Descriptor Privilege level` of the segment register. You will learn what this is in more detail when you get around to user space. I won't go into detail here as to not confuse you too much since it's a long road to user space.

`TR =0000 0000000000000000 0000ffff 00008b00 DPL=0 TSS64-busy`

- I will only give a brief introduction to this as it is something you will only need when you get to userspace and or hardware multitasking, explaining these concepts early on may be confusing to some.
- TR = TSS register. This is also something that has to do with userspace, so it will most likely not affect you at this stage. (Again it's just to avoid confusion)
- TSS64-busy = The state of the TSS

`DR0=0000000000000000 DR1=0000000000000000 DR2=0000000000000000 DR3=0000000000000000 DR6=00000000ffff0ff0 DR7=0000000000000400`

- DR0-3 = Debug register 0-3. These hold the physical address of up to 4 breakpoints
- DR6   = The lower 4 bits in this register are set upon entering a debug handler when an debug exception has occurred, which provides information about the triggered breakpoint (bit0->DR0, bit1->DR1, etc).
- DR7	= This register is used to selectively enable the four address breakpoint conditions. For more information read the [wiki article][DR7-Explanation]

`EFER=0000000000000500`
- EFER = Extended Feature Enable Register. The EFER is a model specific register and it can be used to enable the syscall/sysret instruction. It is architectural to amd and has been adopted by intel. For more information consult the [AMD][AMD-SDM] or [Intel][INTEL-SDM] SDM.

### Almost there..
Wow, all in all that was pretty smooth sailing.
We aren't done yet, you may or may not have noticed the absense of the mysterious `check_exception` that occurs (seemingly) randomly.

Fortunately for us, this isn't random.

This is caused when an exception occurs, and it gives us some information about the type of exception, and if it came from a previous exception (i.e. a double or triple fault).

Let's take a look at an example:

`check_exception old: 0xffffffff new 0xe`

We now know that an exception has occurred, but which? And what does `old` mean?

Well `old` and `new` are the ISR number's (i.e. the type of interrupt represented as a number). `old` displaying the ISR number of the previous exception, which in this case is not present (`-1` or `0xffffffff`) and `new` is the ISR of the current interrupt.

For a list of ISR numbers refer to the [intel][INTEL-SDM] SDM or [this][OSDEV-WIKI-ISR-NUMBERS] osdev wiki article (Look for the "Vector nr.").

### Wrapping it up
That's it for this post!
You should now be able to understand and leverage the output of this once mysterious output being spammed to the console.

Pro-tip: If you are already printing to the console (stdio) via qemu's serial emulation or some other way you can use -D `filename` to redirect the output of the log into a file which you can then use for debugging

[OSDEV-WIKI-ISR-NUMBERS]: https://wiki.osdev.org/Exceptions
[DR7-Explanation]: https://en.wikipedia.org/wiki/X86_debug_register#DR7_-_Debug_control
[INTEL-SDM]: https://software.intel.com/content/www/us/en/develop/download/intel-64-and-ia-32-architectures-sdm-combined-volumes-1-2a-2b-2c-2d-3a-3b-3c-3d-and-4.html
[AMD-SDM]: https://developer.amd.com/resources/developer-guides-manuals/