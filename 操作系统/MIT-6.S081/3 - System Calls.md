
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

### 分析

![[trace.c.png]]
分析用户程序可以发现, trace命令的流程是先校验参数是否符合要求, 然后执行trace命令, 用掩码标识需要追踪的系统调用, 然后正常执行之后的命令.
对于系统调用trace, 只有一个参数就是掩码, 一个返回值标识是否成功追踪

根据提示, 下一步要做的是在sys_trace()中将需要追踪的系统调用记录到进程结构中(需要增加新的变量), 由于trace是用mask判断是否需要追踪的, 所以只需要在proc.h中记录mask就行

![[proc-mask.png]]

再下一步就需要在sys_trace中记录mask

![[sys_trace.png]]

修改fork方法, 实现子进程能继承父进程的mask
![[fork-tarce.png]]

## Sysinfo(moderate)

>In this assignment you will add a system call, sysinfo, that collects information about the running system. The system call takes one argument: a pointer to a struct sysinfo (see kernel/sysinfo.h). The kernel should fill out the fields of this struct: the freemem field should be set to the number of bytes of free memory, and the nproc field should be set to the number of processes whose state is not UNUSED. We provide a test program sysinfotest; you pass this assignment if it prints "sysinfotest: OK".

hints:

- Add $U/\_sysinfotest to UPROGS in Makefile
- Run make qemu; user/sysinfotest.c will fail to compile. Add the system call sysinfo, following the same steps as in the previous assignment. To declare the prototype for sysinfo() in user/user.h you need predeclare the existence of struct sysinfo:
        struct sysinfo;
        int sysinfo(struct sysinfo \*);
	Once you fix the compilation issues, run sysinfotest; it will fail because you haven't implemented the system call in the kernel yet.
- sysinfo needs to copy a struct sysinfo back to user space; see sys_fstat() (kernel/sysfile.c) and filestat() (kernel/file.c) for examples of how to do that using copyout().
- To collect the amount of free memory, add a function to kernel/kalloc.c
- To collect the number of processes, add a function to kernel/proc.c

### 分析

首先依照hints把需要添加的相关配置都添加上, 包括用户空间中的usys.pl, makefile的系统调用接口, kernel中的syscalls数组以及函数定义

然后实现sysinfo系统调用, sysinfo需要将freemem和nproc正确设置后返回

根据提示先查看sys_fstat()函数和filestat()函数学习如何使用copyout()

![[copyout.png]]
看注释可以直到该函数的作用是从内核向用户空间复制, 在给定的页表中, 将长度为len的字节从src拷贝的目的虚拟地址dstva

很明显sysinfo系统调用要做的就是构造一个sysinfo结构体, 填充相关字段后拷贝到用户传入的sysinfo结构体指针所在位置.

获取用户传入的地址使用argaddr(0, &addr)即可


收集freemem需要在kalloc.c中添加一个新的函数
参照其他内存相关的函数, 这里很容易得到只需要遍历一下空闲列表就能算出来
![[c_freemem.png]]
同时, 看kalloc的代码也能发现, xv6组织内存的方式是将空闲页表作为一个链表串联, 每次释放一个页时, 就将其拼接到头部

收集nproc需要在proc.c中添加一个新的函数
![[c_nproc.png]]
