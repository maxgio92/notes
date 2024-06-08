---
Title: The memory stack and the BP, SP, PC pointer registers
---

## The program counter, the stack pointer, and the base pointer

The program counter (PC)/instruction pointer (IP) is a register that points to code, that is, the instruction that will be executed next.
It always points to somewhere in the code.
The stack pointer (and base pointer) points to the data: the stack contains data. 
Considering that the stack grows whenever you add anything new to your stack, including new variables, the stack pointer is the lowest position in the stack (the stack grows from the highest address to the lowest address): so if you declare a new variable of 4 bytes, the stack pointer will be increased by 4 bytes too.
Specifically, a stack pointer points to the first free and unused address on the stack. You reserve more space on stack by adjusting the stack pointer.

CALL instruction pushes the current value of PC (next instruction address) and the function arguments into stack, and gives control to the target address (PC is set to the target address of CALL instruction).
So, the just pushed return address is a snapshot of the program counter, available in the stack.
As a result, control is passed to the called address (subroutine) and the return address (the address of the instruction next to CALL) is available.[
RET instruction POPs value from stack (the return address) and puts it in PC.
So, the next instruction is from return.
So, the CALL - RET pair is very useful in the reusability of code.

Because all of the above points need to be memorized on the stack, the stack size will naturally increase (and thus the stack and base pointers too).

When a function finishes normally and returns, if the compiler can tell it won’t need to call it again right away, then it is ‘popped’ off the stack and the stack pointer returns to the last position (towards the top) it was before. In the case of a function calling a function, the program counter returns to the next line in the previous stack frame and starts executing from there. The return address was stored in a register or in RAM automatically when the stack frame was created for the function, and in C is not alterable from the source code.

In a nutshell, the PC is used to memorize the current instruction address:
- After a semicolon, it should increase by 8 bytes (assuming a 64bit instruction set) to point to the next instruction
- If we are in an if() statement but the condition is false, or if we call a function, we'll jump on the 1st instruction of the desired block
- If we are exiting a function, we'll assign the return address to it.

The memory is already set for the program, it never grows or shrinks, unless more is needed then the OS steps in and gives it some new areas of memory to use.
The heap acquires memory from the bottom of the same region, and “grows up” towards the middle of the same memory region.
Virtual memory and paging, etc, are all kernel stuff. The program uses the one-big linear memory region model that the compiler works out.

![memory-regions-stack-instructions](https://raw.githubusercontent.com/maxgio92/notes/68c5220995702493845a3d96cc9d6dc7ce61ec8f/content/notes/memory-regions-allocations.jpg)

## What about the frame pointer? How does it relate to the base pointer?

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

On program execution (Unix-like fork and exec system call groups) OS allocates memory to later store the program's instructions (code) and data.
On Unix-like operating systems, the exec family of system calls replaces the program executed by a process.
When a process calls exec, all instructions and data - in ELF executable format, they're the text and the data sections - in the process is replaced with the executable of the new program.
The OS then sets the PC to the memory address of the first instruction, which is fetched, decoded, and executed one by one.
> As a detail, although all data is replaced, all open file descriptors remain open after calling exec unless explicitly set to close-on-exec.

![memory-map-exec](https://raw.githubusercontent.com/maxgio92/notes/d3bf6f231c330ba746354cc463469245fc9de7bc/content/notes/memory-map-exec.png)

In particular, on Linux, on execs, the .text and .data ELF sections are loaded by the kernel at the base address.
The main stack is located just below and grows downwards.
Each thread and function call will have its own-stack / stack frame.
This is located below the main stack.
Each stack is separated by a guard page to detect Stack-Overflow.

![memory-map-elf](https://raw.githubusercontent.com/maxgio92/notes/d3bf6f231c330ba746354cc463469245fc9de7bc/content/notes/memory-map-elf.png)

