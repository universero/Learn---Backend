> In this lab, we will add more system calls and be exposed to some of the internals of the xv6 kernel. Before we start coding, read Charpter 2 of the xv6 book and Sections 4.3 and 4.4 of Chapter 4, and related source files:
> - The user-space code for systems calls is in user/user.h and user/usys.pl.
> - The kernel-space code is kernel/syscall.h, kernel/syscall.c.
> - The process-related code is kernel/proc.h and kernel/proc.c.
> To start the lab, switch to the syscall branch by `git checkout syscall`

### Reading notes of xv6

- xv6 is written in "LP64"C, which means long and pointers in the C programming language are 64 bits, but int is 32-bit. And it's written for the support hardware simulated by qumus's "machine virt" option, incluing RAM, a ROM containing boot code, a serial connectin to the user's key-board/screen, and a disk for storage.

- xv6 has three modes in which the CPU can execute instructions: 
	- machine mode：has full privilege; a CPU starts in machine mode. Machine is intended for configuring a computer. xv6 executes a few lines in machine mode and change to supervisor mode.
	- supervisor mode:  allowed to execute privileged instructions. If an application in use mode attempts to execute a privileged instruction, then the CPU doesn't execute the instruction, but switches to supervisor mode so that supervisor-mode code can terminate the application, because it did something it shouldn't be doing
	- user mode: application can only execute only user-node instructions and is said to be running in user space; while the software in supervisor mode is saied to be running in kernel space, and is called the kernerl. 
#### xv6 organization

The xv6 kernel source is in the /kernel/. 
![[xv6 kernel source files.png]]
The inter-module interfaces are defined in defs.h

#### Process overview

The mechanisms used by the kernel to implement processes include the user/supervisor mode flag, address spaces, and time-slicing of threads.
Xv6 uses page tables to give each process its own address space.
![[Layout of a process’s virtual address space.png]]
- In the layout, instructions come first, followed by global variables, the the stack, and finally a heap area that the process can expand as needed.At the top of the address space xv6 reserves a page for a trampoline and a page mapping the process's trapframe which used to trasition into the kernel and back; the trampoline page contains the code to transition in and our of the kernel and mapping the trapframe is necessary to save/restore the state of the user process.
- Xv6 only 38 of 64 bits as virtual address, thus the maximum address is 2^38 -1. which is MAXVA(kernel/riscv.h).
- Xv6 kernel maintains many pieces of state for each process, which it gathers into a struct proc(kernel/proc.h)
- A process has a user stack and a kernel stack which respectively execute user mode code and kernel mode code. And they are seperated, code in kernel stack can be execute even if user stack is wrecked
- A process can make a system call by executing the RISC-V ecall instruction. This instruction raises the hardware privilege level and changes the program counter to a kernel-defined entry point. The code at the entry point switches to a kernel stack and executes the kernel instructions that implement the system call.When the system call completes, the kernel switches back to the user stack and returns to user space by calling the sret instruction, which lowers the hardware privilege level and resumes executing user instructions just after the system call instruction.

## System Call Tracing(moderate)

> In this assignment you will add a system call tracing feature that may help you when debugging later labs. You'll create a new trace system call that will control tracing. It should take one argument, an integer "mask", whose bits specify which system calls to trace. For example, to trace the fork system call, a program calls trace(1 << SYS_fork), where SYS_fork is a syscall number from kernel/syscall.h. You have to modify the xv6 kernel to print out a line when each system call is about to return, if the system call's number is set in the mask. The line should contain the process id, the name of the system call and the return value; you don't need to print the system call arguments. The trace system call should enable tracing for the process that calls it and any children that it subsequently forks, but should not affect other processes.

hints:
- Add `$U/_trace` to UPROGS in Makefile
- Run `make qemu` and you will see that the compiler cannot compile `user/trace.c`, because the user-space stubs for the system call don't exist yet;
- Add a prototype for the system call to `user/user.h`, a stub to `user/usys.pl`, and a syscall number to `kernel/syscall.h`. The Makefile invokes the perl script `user/usys.pl`, which produces user/usys.S, the acutal system call stubs, which use the RISC-V `ecall`instruction to transition to the kernel.
	 - Add a `sys_trace()` function in `kernel/sysproc.c` that implements the new system call by remembering its argument in a new variable in the proc structure (see kernel/proc.h). The functions to retrieve system call arguments from user space are in kernel/syscall.c, and you can see examples of their use in kernel/sysproc.c.
 - Modify `fork()` (see kernel/proc.c) to copy the trace mask from the parent to the child process.
 - Modify the `syscall()` function in kernel/syscall.c to print the trace output. You will need to add an array of syscall names to index into.