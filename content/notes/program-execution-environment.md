---
Title: The program execution environment
---

We know that the programs are executed by the CPU and that the program's binary instructions are stored in a volatile memory that is the random access memory.

The scope of this blog is to dig into how the CPU keeps track of and executes instructions as expected by the program.
As RAM locations are byte-addressable the CPU needs a way to keep track of the addresses in order to retrieve the data from it, which is in our case CPU instructions that are then executed.

The CPU uses small built-in memory areas called register to hold data retrieved from main memory. Registers come in two types: general-purpose and special-purpose. Special-purpose registers include pointer registers, which are designed specifically to store pointers, which means, they store the memory address's value.

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

> **The saved base pointers and the stack unwinding**
> 
> When the base pointer is pushed to the stack, it points to the previous frame's base pointer, enabling debuggers or profilers to walk the stack, also called stack unwinding. But FPO or frame pointer omission optimization will actually eliminate this and use the base pointer as another register and access locals directly off of the stack pointer. In this case, the stack unwinding is a bit more difficult since it can no longer directly access the stack frames of earlier function calls.

In particular, CALL instruction pushes also the value of the program counter at the moment of the new function call (next instruction address), and gives control to the target address. The program counter is set to the target address of the `CALL` instruction, which is, the first instruction of the called function.

In a nutshell: the just pushed return address is a snapshot of the program counter, and the pushed frame pointer is a snapshot of the base pointer, and they're both available in the stack.

As a result, control is passed to the called subroutine address and the return address, that is the address of the instruction next to `CALL`, is available.

### The return path

On the return path from the function, `RET` instruction `POP`s the return address from the stack and puts it in the program counter register. So, the next instruction is available from that return address.

Since the program counter register holds the address of the next instruction to be executed, loading the return address into PC effectively points the program execution to the instruction that follows the function call. This ensures the program resumes execution from the correct location after the function is completed.

![stack-frames](https://raw.githubusercontent.com/maxgio92/notes/14bdde325f646b53ee0b6501f0ba9d3ecbaded4f/content/notes/memory-stack-frames.png)

In the case of a function calling a function, the program counter returns to the return address in the previous stack frame and starts executing from there.

Because all of the above points need to be memorized on the stack, the stack size will naturally increase, and on return decrease. And of course, the same happens to the stack and base pointers.

### The data, the code and the heap areas

When a program is loaded into memory, the operating system statically allocates a specific amount of memory for it. This includes space for the program's instructions and the stack.

I know this is too simplified, but we'll talk about how the program is loaded and organized in memory later on. 

On dynamic allocations requested by the program, the heap acquires memory from the bottom of the same region and grows upwards towards the middle of the same memory region. While explaining how dynamic memory mapping implementations work in operating systems is out of scope here, but it's important to say that user processes see one contigous memory space thanks to the virtual memory mapping managed by the OS.

Below is an example of the memory regions and mapping of data and code, particularly with the ELF executable format:

![memory-regions-stack-instructions](https://raw.githubusercontent.com/maxgio92/notes/68c5220995702493845a3d96cc9d6dc7ce61ec8f/content/notes/memory-regions-allocations.jpg)

## What about the frame pointer?

The frame pointer is the base pointer because is set up when a function is called to establish a fixed reference (base) point for accessing local variables and parameters within the function's stack frame. Depending on what ABI you use, parameters are passed either on the stack or via registers. For instance, on i386 System V ABI, parameter 0 is at the base pointer + 8 (i.e. "8(%ebp)"). In the x86-64 ABI, parameter 0 is stored in the %rdi register.

Below the parameters on the stack (remember, the stack grows down) are:
- the return address
- the previous frame pointer
- saved register state
- the local variables of the function.

Remember: the return address is a snapshot of the program counter, so it points to instructions (code).
The previous frame pointer is a snapshot of the base pointer, so it points to the stack (data).

![stack-return-address-previous-frame-pointer](https://raw.githubusercontent.com/maxgio92/notes/5ab379b18942d782ac152cc81ad9029ae15d8dd1/content/notes/memory-stack-ip-bp.png)

Below the local variables are other stack frames resulting from more recent function calls, as well as generic stack space used for computation and temporary storage. The most recent of these is pointed to by the real stack pointer. This is the difference between the stack pointer and the frame/base pointer.

However, the frame pointer is not always required. Modern compilers can generate code that just uses the stack pointer. At any point in code generation, it can determine where the return address, parameters, and locals are relative to the stack pointer address (either by a constant offset or programmatically).

Without a frame pointer, though, it is much more difficult for debuggers to determine the stack backtrace. In particular, hardware debuggers that do not have symbolic program information are almost always unable to show the stack.

Why would you not use a frame pointer? In most architectures it frees up a register, essentially increasing the size of the CPU's fastest and best memory -- its register file.

### Clarification about the register names

On 16-bit ISA are usually called SP, BP, and IP.
Instead on 32-bit ESP, EBP, and EIP.
Finally, on 64-bit they're usually called RSP, RBP, and RIP.

## How the code and the data are loaded and organized in the allocated memory?

> Note: the memory allocation during the fork is only mentioned here.

On program execution (Unix-like fork and exec system call groups) OS allocates memory to later store the program's instructions (code) and data (in the stack).
On Unix-like operating systems, the exec family of system calls replaces the program executed by a process.
When a process calls exec, all instructions - in ELF executable format it's the text section - and the data in the process is replaced with the executable of the new program.
The OS then sets the PC to the memory address of the first instruction, which is fetched, decoded, and executed one by one.
> I couldn't find yet where the information about how to set up the stack at exec time from an ELF file are stored in the ELF structure.
> Also, as a detail, although all data is replaced, all open file descriptors remain open after calling exec unless explicitly set to close-on-exec.

![memory-map-exec](https://raw.githubusercontent.com/maxgio92/notes/d3bf6f231c330ba746354cc463469245fc9de7bc/content/notes/memory-map-exec.png)

In particular, on Linux, on execs, the .text and .data ELF sections are loaded by the kernel at the base address.
The main stack is located just below and grows downwards.
Each thread and function call will have its own-stack / stack frame.
This is located below the main stack.
Each stack is separated by a guard page to detect Stack-Overflow.

![memory-map-elf](https://raw.githubusercontent.com/maxgio92/notes/d3bf6f231c330ba746354cc463469245fc9de7bc/content/notes/memory-map-elf.png)

If you want to go deeper on the Linux `exec` path, I recommend [this chapter](https://github.com/0xAX/linux-insides/blob/f7c6b82a5c02309f066686dde697f4985645b3de/SysCall/linux-syscall-4.md#execve-system-call) from the [Linux insides](https://0xax.gitbooks.io/linux-insides/content/index.html) book.

### ELF structure

Digging into the ELF format you can find below the structure of this executable and linkable format:

![elf-structure](https://raw.githubusercontent.com/maxgio92/notes/20f4417f50afb71a79a8712decea1f76ffc16cc9/content/notes/elf-dissection.avif)

For more information please refer to the man of file formats and conventions for elf (`man 5 elf`).

## References

* https://www2.it.uu.se/edu/course/homepage/os/vt19/module-2/process-management/
* https://stackoverflow.com/questions/18278803/how-does-elf-file-format-defines-the-stack
* https://stackoverflow.com/questions/21718397/what-are-the-esp-and-the-ebp-registers
* https://groups.google.com/g/golang-nuts/c/wtw0Swe0CAY
* https://www.polarsignals.com/blog/posts/2022/01/13/fantastic-symbols-and-where-to-find-them
* https://0xax.gitbooks.io/linux-insides/content/index.html
