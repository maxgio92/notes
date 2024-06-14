---
Title: CPU profiling 101 pt.1 - The execution environment
---

CPU profiling allows us to analyze the program's performance, identify bottlenecks, and optimize its efficiency.

Have you ever wondered what happens behind the scenes when you run a program and how to account for CPU time for the actual program functions? And even more, how to write such a tool to profile the program?

Even though great open-source projects provide continuous profiling with vast support for compiled, JITed, and interpreted, languages, with or without debug info, with or without frame pointers, etc., don't be discouraged!

Writing your own can be a fantastic learning experience. Building your own profiler offers a unique challenge and the satisfaction of unlocking powerful performance analysis tools.

This blog series will embark on a journey to give you the basics for writing a program profiler.

In this first episode, we'll establish the foundation by exploring the program execution environment. We'll dig into how the CPU executes a program and keeps track of the execution flow. Finally, we'll discover how this tracking data is stored and becomes the key to unlocking the profiling primitives.

## Introduction

We know that the CPU executes the programs and that the program's binary instructions are stored in a volatile memory which is the random access memory.

As RAM locations are byte-addressable the CPU needs a way to keep track of the addresses in order to retrieve the data from it, which is in our case CPU instructions that are then executed.

The CPU uses small built-in memory areas called registers to hold data retrieved from the main memory. Registers come in two types: general-purpose and special-purpose. Special-purpose registers include pointer registers, which are designed specifically to store pointers, which means, they store the memory address's value.

There are other types of registers but they're out of scope for this walkthrough.

The first part will go through the main pointer registers, which are commonly implemented by the predominant architectures (x86, ARM, MIPS, PowerPC as far as I know).
So, please consider that these specifics may differ depending on the architecture.

## The program counter (PC), the stack pointer (SP), and the base pointer (BP) processor registers

The program counter (PC), often also called instruction pointer (IP) in x86 architectures, is a register that points to code, that is, the instruction that will be executed next. The instruction data will be fetched, will be stored in the instruction register (IR), and executed during the instruction cycle.
Depending on the instruction set, the IP will be increased instruction by instruction by the instruction size (e.g. 8 bytes on 64 but Instruction Set Architectures).

When compiling a program it will contain the instructions to be executed, that the CPU will fetch and execute, and how they're stored depends on the executable format. For example, considering the ELF format, the code is represented by the `.text` section.

![memory-program-counter-1](https://raw.githubusercontent.com/maxgio92/notes/14bdde325f646b53ee0b6501f0ba9d3ecbaded4f/content/notes/memory-cpu-program-counter.gif)
![memory-program-counter-2](https://raw.githubusercontent.com/maxgio92/notes/14bdde325f646b53ee0b6501f0ba9d3ecbaded4f/content/notes/memory-cpu-program-counter-1.gif)

On the other side, the stack pointer (SP) and base pointer (BP) point to the stack, which contains data about the program being executed.

While a detailed explanation of the stack is beyond the scope of this blog, here's a basic idea: it's a special area of memory that the CPU uses to manage data related to the program's functions (subroutines) as they are called and executed, pushing it to it in a LIFO method. We'll see later on in more detail.

Data and code are organized in the process's address space with areas. Explaining how the OS manages access to memory with virtual and physical addressing or processor rings, is out of the scope.

As the stack grows whenever the CPU adds new data while executing the program, the stack pointer is at the lowest position in the stack.
> Remember: the stack grows from the highest address to the lowest address
So, when a new variable of 4 bytes is declared, the stack pointer will be increased by 4 bytes too.

In the following image you can find an example considering a single process:

![memory-program-counter-2](https://raw.githubusercontent.com/maxgio92/notes/3db4d57bd2a84df56925e19ab24b03badfd649f1/content/notes/memory-process-data-code.png)

Specifically, a stack pointer (SP) points to the first free and unused address on the stack.
It can reserve more space on the stack by adjusting the stack pointer, and then `PUSH` instruction (valid in many architectures) pushes data at the address pointed to by the stack pointer (e.g. local variables).

Meanwhile, usually, the base pointer (BP) is a snapshot of the stack pointer (SP) at the start of the frame, so that function parameters and local variables are accessed by adding and subtracting, respectively, a constant offset from it.

You can find a diagram in the picture below:

![stack-frame](https://raw.githubusercontent.com/maxgio92/notes/14bdde325f646b53ee0b6501f0ba9d3ecbaded4f/content/notes/memory-stack-frames-simple.png)

In the previous image, the base pointer is referred to as the frame pointer (FP). What is fundamental here is the main stack structure is organized in sub-structures named frames. We'll go through it while explaining how the function call path works.

### The call path

When a new function is called a dedicated memory space dedicated to the new function is pushed to the stack. This memory space will contain function specific data like arguments, local variables, saved registers if needed.
Also, the previous base pointer is also pushed to the stack.

While this is usually true, it's not mandatory and it depends on how the binary has been compiled. This mainly depends on the compiler optimization techniques.

> If you're interested on the main impacts of libraries compiled with this optimization and distributed by common Linux distributions I recommend this Brendan Gregg's great article: https://www.brendangregg.com/blog/2024-03-17/the-return-of-the-frame-pointers.html. 

In particular, CALL instruction pushes also the value of the program counter at the moment of the new function call (next instruction address), and gives control to the target address. The program counter is set to the target address of the `CALL` instruction, which is, the first instruction of the called function.

In a nutshell: the just pushed return address is a snapshot of the program counter, and the pushed frame pointer is a snapshot of the base pointer, and they're both available in the stack.

As a result, control is passed to the called subroutine address and the return address, that is the address of the instruction next to `CALL`, is available.

### The return path

On the return path from the function, `RET` instruction `POP`s the return address from the stack and puts it in the program counter register. So, the next instruction is available from that return address.

Since the program counter register holds the address of the next instruction to be executed, loading the return address into PC effectively points the program execution to the instruction that follows the function call. This ensures the program resumes execution from the correct location after the function is completed.

![stack-frames](https://raw.githubusercontent.com/maxgio92/notes/14bdde325f646b53ee0b6501f0ba9d3ecbaded4f/content/notes/memory-stack-frames.png)

In the case of a function calling a function, the program counter returns to the return address in the previous stack frame and starts executing from there.

Because all of the above points need to be memorized on the stack, the stack size will naturally increase, and on return decrease. And of course, the same happens to the stack and base pointers.

As I'm a visual learner, the next section will show how the program's code and data are organized in its process address space. This should give you a clearer picture of their layout within the process's address space.

### The process address space regions

The process address space is a virtual memory region allocated by the operating system (OS) for a running program. It provides a logical view of memory for the program and hides the complexities of physical memory manage

While explaining how memory mapping implementations work in operating systems is out of scope here, it's important to say that user processes see one contigous memory space thanks to the memory mapping features provided by the OS.

The address space is typically divided in different regions, and the following names are mostly standard for different OSes:
* Text segment: this is the area where the (machine) code of the progam is stored
* Data segment: this region contains typically static variables which are initialized
* BSS (Block Started by Symbol) / Uninitialized data segment: it contains global and static variables that are not initialized when the program starts
* Heap: it's a region available for dynamic allocation available to the running process. Programs can request pages from it at runtime (e.g. `malloc` from the C stdlib).
* Stack: we already talked about it.

The operating system can enforce protection for each of them, like marking the text segment read-only to prevent modificatio of the running program's instructions.

When a program is loaded into memory, the operating system allocates a specific amount of memory for it and dedicate specific regions to static and dynamic allocation. The static allocation includes the allocation for the program's instructions and the stack.

Dynamic allocations can be handled by the stack or the heap. The heap usually acquires memory from the bottom of the same region and grows upwards towards the middle of the same memory region.

The next diagram will show the discussed memory regions:

![memory-regions-stack-instructions](https://raw.githubusercontent.com/maxgio92/notes/68c5220995702493845a3d96cc9d6dc7ce61ec8f/content/notes/memory-regions-allocations.jpg)
> Credits for the diagram to [yousha.blog.ir](https://yousha.blog.ir/).

Now let's get back to the pointer register. We mentioned that the base pointer is often called frame pointer.

## How the program is loaded into memory in Unix-like OSes?

On program execution (Unix-like `fork` and `exec` system call groups) OS allocates memory to later store the program's instructions (in the text segment) and data (in the stack).
The `exec` family of system calls replaces the program executed by a process.
When a process calls `exec`, all instructions - in ELF executable format it's the `.text` section - and the data in the process are replaced with the executable of the new program.
The OS then sets the PC to the memory address of the first instruction, which is fetched, decoded, and executed one by one.

![memory-map-exec](https://raw.githubusercontent.com/maxgio92/notes/d3bf6f231c330ba746354cc463469245fc9de7bc/content/notes/memory-map-exec.png)
> I didn't manage yet to find yet where the information about how to set up the stack at exec time from an ELF file is stored in the ELF structure. If you do, feel free to share it!

Moreover, as a detail, although all data is replaced, all open file descriptors remain open after calling exec unless explicitly set to close-on-exec.

In particular, on Linux, on execs, the `.text` and `.data` ELF sections are loaded by the kernel at the base address. The main stack is located just below and grows downwards. Each function call will have its own stack frame. Each stack is separated by a guard page to detect stack overflow.

![memory-map-elf](https://raw.githubusercontent.com/maxgio92/notes/d3bf6f231c330ba746354cc463469245fc9de7bc/content/notes/memory-map-elf.png)

If you want to go deeper on the Linux `exec` path, I recommend [this chapter](https://github.com/0xAX/linux-insides/blob/f7c6b82a5c02309f066686dde697f4985645b3de/SysCall/linux-syscall-4.md#execve-system-call) from the [Linux insides](https://0xax.gitbooks.io/linux-insides/content/index.html) book.

### References: the ELF structure

Digging into the ELF format you can find below the structure of this executable and linkable format:

![elf-structure](https://raw.githubusercontent.com/maxgio92/notes/20f4417f50afb71a79a8712decea1f76ffc16cc9/content/notes/elf-dissection.avif)

For more information please refer to the man of file formats and conventions for elf (`man 5 elf`).

## Frame pointer and the stack unwinding

Now let's go back to our pointer registers because we often read about frame pointer and less about the base pointer, but actually the frame pointer *is* the base pointer.

As already discussed, the name base pointer comes to the fact that is set up when a function is called to establish a fixed reference (base) to acces local variables and parameters within the function's stack frame.

Depending on the ABI, parameters are passed either on the stack or via registers. For instance:
* x86-64 System V ABI: in the general purpose registers `rdi`, `rsi`, `rdx`, `rcx`, `r8`, and `r9` for the first six parameters. On the stack from the seventh parameter onward.
* 386 System V ABI: in the general purpose registers `eax`, `ecx`, `edx`, and `ebx` for the first four parameters. On the stack from the fifth parameter onward.

In any case, the main data that is stored on the stack (frame) is:
- the return address
- the previous frame pointer
- saved register state
- the local variables of the function.

![stack-return-address-previous-frame-pointer](https://raw.githubusercontent.com/maxgio92/notes/5ab379b18942d782ac152cc81ad9029ae15d8dd1/content/notes/memory-stack-ip-bp.png)
> Remember: the return address is a snapshot of the program counter, so it points to instructions (code).
The previous frame pointer is a snapshot of the base pointer, so it points to the stack (data).

Below the local variables are other stack frames resulting from more recent function calls, as well as generic stack space used for computation and temporary storage. The most recent of these is pointed to by the stack pointer. This is the difference between the stack pointer and the frame/base pointer.

However, the frame pointer is not always required. Compiler optimization technique can generate code that just uses the stack pointer.

Frame pointer elimination (FPE) is an optimization that removes the need for a frame pointer under certain conditions, mainly to reduce the space allocated for the stack and to optimize performance because pushing and popping the frame pointer takes time during the function call. The compiler analyzes the function's code to see if it relies on the frame pointer for example to access local variables, or if the function does not call any other function. At any point in code generation, it can determine where the return address, parameters, and locals are relative to the stack pointer address (either by a constant offset or programmatically).

Frame pointer omission (FPO) is instead an optimization that simply instructs the compiler to not generate instructions to push and pop the frame pointer at all during function calls and returns.

Because the frame pointers are pushed on function call to the stack frame just created for the newly called function, and its value is the value of the stack pointer at the moment of the `CALL`, it points to the previous stack frame.

At the end, all the frame pointer saved to the stack can be used to build a stack trace, by walking the stack. This technique is leveraged particularly by debuggers and profilers and it's usually referred to as *stack unwinding*. You can see it in the following picture:

![stack-walking](https://raw.githubusercontent.com/maxgio92/notes/5eeff1703e85c00799e7af0117a3898918d7a438/content/notes/stack-walking.avif)

And this comes to the next episode of this series, which will dive into how to program stack unwinding and how to account for each function CPU time!

I hope this has been interesting to you. Any feedback is more than appreciated.

See you in the next episode!

### Clarification about the register names

On 16-bit ISA are usually called `sp`, `bp`, and `ip`.
Instead on 32-bit `esp`, `ebp`, and `eip`.
Finally, on 64-bit they're usually called `rsp`, `rbp`, and `rip`.

## References

* https://www2.it.uu.se/edu/course/homepage/os/vt19/module-2/process-management/
* https://stackoverflow.com/questions/18278803/how-does-elf-file-format-defines-the-stack
* https://stackoverflow.com/questions/21718397/what-are-the-esp-and-the-ebp-registers
* https://groups.google.com/g/golang-nuts/c/wtw0Swe0CAY
* https://www.polarsignals.com/blog/posts/2022/01/13/fantastic-symbols-and-where-to-find-them
* https://0xax.gitbooks.io/linux-insides/content/index.html
