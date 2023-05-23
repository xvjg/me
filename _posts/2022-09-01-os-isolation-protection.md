---
layout: post
title: "(Xinu) OS: Isolation/ Protection"
date: 2022-09-01 17:00:00 -0400
category: classnotes
tags: os xinu
---

> _**These are lecture notes from Purdue CS503, Fall 2022, that I have made public to help others. The information here can be incomplete/ incorrect and definitely do not cover all points discussed in lectures. This is not a substitute for attending lectures. I will come back and modify it if I get time in the future.**_

# Motivation
Generally, sufficient protection is needed to isolate processes from each other for security purposes. Also, isolation/ protection is a key concept in virtualization software like VMWare, Docker... 

# Hardware and Software Support
1. Instruction Set Architecture: Only allow certain processes to execute certain instructions
  * Privileged mode and operations. For example, mov CRO is privileged
  * Non-privileged mode and operations
2. Processor Modes
  * Kernel mode
  * User mode
3. "Trapping" (Switching) between modes: User mode ↔ Kernel mode. Trapping is done using the following:
    * Synchronous interrupts
      - Exception/ Fault
      - Software Interrupt (int instruction)
    * Asynchronous interrupts
      - Hardware interrupts (IDT and pins)
4. Memory Protection (Paging)
5. Stack Switch Support: Hardware support for performance
    * User stack
    * Kernel stack (per-process)

# Isolate Process Memory
**How are process memory isolated so that they cannot peek into each other's data?**

So segmentation can be used to provide isolation, but it's also pretty hard to use. For example, it requires that the operating system allocate contiguous physical memory for each process (or, at best, a handful of contiguous ranges). This can lead to fragmentation problems and generally makes the OS's job harder. How can we do better?

Current architectures and OSes generally impose isolation, and manage their memory spaces, using paging. In paging, memory is divided into equally sized chunks called pages that are 2x bytes long.  On an x86: x=12 and each page is 4KB.

The hardware implements a page table function PT that enforces protection on the level of individual pages. This is much more flexible than segmentation, since the OS can, for example, provide access to every other page of memory! (However, only segmentation can grant access to units smaller than a page.) When a process accesses an address, the processor looks up that address in the page table. If the page table indicates that the process shouldn't have access to the address, a page fault occurs, which generally kills the program. (In Unix, page faults generally show up as segmentation violation signals.) [^1]

## Intel Galileo
In CR0 register, set PG = 1 to enable paging.
In Xinu, paging is disabled, so PG is set to 0.
Paging is used for memory protection between 2 processes.

## Note
![](/static/images/posts/os/os-reserved-memory.png)

OS reserves some part of memory for itself no matter the process.

# Organization of kernel as a reactive system
![](/static/images/posts/os/os-kernel-halves.png)

Two modes: User mode (applications) and kernel mode (system calls).

Kernel mode is divided into upper half (high level language) and lower half (low level language like assembly). Scheduler lies at the border of upper and lower mode. Try to avoid disabling pre-emption; more on this later when I post about device management.

Interrupts start from bottom while system calls start from top.

# GDT in Xinu
- Initialized in meminit.c
- Contains only Kernel mode segments
- Because everything in Xinu runs in Kernel mode
- Read Example of GDT (8 KB) in Main Memory for isolation protection of User and Kernel mode
- In Lab 2, the GDT is changed to the one presented above
- Table and Privilege level initialization is in start.S 
    - GDTR loaded using lgdtr instruction
    - CPL in CS register set to 00 (Kernel mode ring 0)

![](/static/images/posts/os/os-xinu-gdt.png)

# Traps

**How can user process modify kernel data-structures or call system services?**

User process must "trap" from user mode into kernel mode. The process of requesting a service from the operating system is called trapping.

**How does trapping work?**
A user mode process would trap into lower half of kernel mode, written in assembly, through software interrupts. In Xinu, the particular interrupt used in Lab 2 was int $46. 

On software interrupt, the processor checks Interrupt Descriptor Table (IDT) pointed to by IDTR to index the label for interrupt handler, also called the interrupt vector.

## Interrupt Descriptor Table Entry
The interrupt vectors are defined in IDT have the following data:
1. Function pointer (label in Assembly)
2. Descriptor Privilege level
3. Index to GDT to know whether it is kernel code or user code

## Stack Switching
Every process has two stacks.
1. Runtime stack
2. Kernel stack
Both of these stacks are assigned in create() syscall.

Lower half of kernel has one stack shared by all processes, but that stack is used for interrupt handling.

When trapping from user mode to kernel mode, stack needs to be switched to the appropriate data stack. Stack switching is performed with the help of Task State Segment (TSS). TSS is a segment with a descriptor in GDT that is pointed to by register TR.

Each entry in TSS contains per-task SS0 and ESP0 (for ring 0). For Xinu, since every process runs in Kernel mode, TSS use is not required but important to know.

![](/static/images/posts/os/os-idt-gdt.png)

## State Preservation in Stack Switching
There are two cases to consider: User mode to Kernel mode and Kernel mode to Kernel mode stack switching for process isolation. In Xinu, only 2nd case is seen.

When switching modes, the previous mode's stack state, that is, SS, ESP and the register state of EFLAGS, CS, and EIP need to be preserved.

![](/static/images/posts/os/os-stack-switch-rings.png)

# References

[^1]: https://www.read.seas.harvard.edu/~kohler/class/05s-osp/notes/notes9.html
