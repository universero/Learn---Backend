>In this lab you will explore page tables and modify them to to speed up certain system calls and to detect which pages have been accessed.
>Before you start coding, read Chapter 3 of the [xv6 book](https://pdos.csail.mit.edu/6.828/2021/xv6/book-riscv-rev2.pdf), and related files:
>- kern/memlayout.h, which captures the layout of memory.
>- kern/vm.c, which contains most virtual memory (VM) code.
>- kernel/kalloc.c, which contains code for allocating and freeing physical memory.
>It may also help to consult the [RISC-V privileged architecture manual](https://github.com/riscv/riscv-isa-manual/releases/download/Ratified-IMFDQC-and-Priv-v1.11/riscv-privileged-20190608.pdf).

## XV6 book notes

页表是操作系统为每个进程提供私有地址空间和内存的机制. 其决定了内存地址的含义以及哪些物理内存部分可用被访问.

### Paging hardware

xv6使用低39位作为虚拟地址, 物理页码PPN有44位, offset有12位(对应4096 bytes).

页表Page Table, 以三级树状结构存储在物理内存中(忽略了许多没有出现的虚拟内存和物理内存的映射, 可以减少非常多PTE占用的内存), 每9位bit对应一级的PPN(3\*9+12=39).

当查找的PTE不存在时, 硬件会触发 page-fault exception, 转到内核态处理这个exception

PTE中包含许多的flags, 用于告知硬件这个虚拟地址的状态
PTE_V说明了PTE是否存在或有效, PTE_R决定指令是否允许Read这个page, PTE_W决定了是否允许write这个page, PTE_X决定了这个page中的内容能否作为指令执行, PTE_U决定了在用户模式下的指令能否访问这个page, 

每块CPU都有独立的satp寄存器, 用于存放page table的root页的物理地址, 每个process也都有自己的页表和对应的root物理地址

指令操作的是虚拟地址, 而DRAM操作的是物理地址, 在指令发送到DRAM前paging hardware会翻译地址

### Kernel address space

xv6中每个process有一个单独的page table描述了用户地址空间, 同时还有一个page table描述了内核地址空间, 内核配置了自己的地址空间使得可以在可预测的虚拟地址中访问物理内存和多种硬件资源.memlayout.h描述了kernel memory 的layout

QEMU的RAM从0x80000000至少到0x86400000(PHYSTOP), QEMU在0x80000000以下的地址中, 将设备接口作为内存映射暴露给软件, 内核通过读写特定地址实现与硬件的交互.
内核空间中使用“直接映射”来获取RAM和内存映射的设备寄存器；也就是说，映射与物理地址相等的虚拟地址上的资源。

![[xv6内存映射.png]]
直接映射简化了读写物理内存的内核代码

不使用直接映射的部分
- trampoline page: 从虚拟地址的最大值映射到内核代码
- kernel stack page: 每个process都有独立的内核栈, 内核栈在虚拟地址比较高的位置, 从而可以在下方放置一个Guard page, Guard page的PTE_V被设为无效, 如果栈生长到这里就会触发异常, 防止了不同进程内核数据间的干扰

### Code: creating an address space
> 这里xv6book中的描述和代码不是完全的对应, 但是大体思路是一直的

vm.c 中包含了xv6操作内存的代码, 核心的数据结构是pagetable_t, 一个指向root page-table page的指针, 可能是内核页表也可能是某个进程的页表. 核心的函数是walk, 用于找到一个虚拟地址对应的PTE和mappages(为新的映射安装PTES). kvm开头的函数操作内核页表, uvm开头的函数操作用户页表, 其他则是两者皆可. coupyout和coupyin用于在用户虚拟地址和内核虚拟地址间复制值.

在启动阶段的前期, main函数会调用kvminit来创建内核页表, 这个函数调用时xv6还不具备页的映射能力, 所以地址是直接映射到物理空间. Kkvminit首先分配了一个物理页存储root page-table, 然后调用kvmmap来配置内核需要的翻译, 包括内核的指令和数据, 直到PHYSTOP的物理内存, 和硬件交互内存的区间

kvmmap调用了mappages, 实现了一段虚拟地址到物理地址的映射, 对这段范围中的每个虚拟地址都通过walk找到对应的PTE的地址, 然后初始化PTE项和相关的权限

walk函数模拟页映射硬件找到PTE的虚拟地址, 如果需要的页没有分配, 且alloc选项被设置了则分配一个新的页表页, 将物理地址作为PTE. walk返回最终的PTE.

上述描述的所有过程中都使用的是直接映射的地址, 将上一步取到的物理地址作为下一步的虚拟地址

main函数调用kvminithart来install内核页表, 它将一个跟页表的物理地址写到寄存器satp中, 然后CPU就可以使用这个寄存器来实现正确的内存映射

main在为每个process调用了proinit函数, 分配对应的内核栈

### Physical memory allocation
xv6 use the physical memory between the end of the kernel and PHYSTOP for run-time allocation. It allocates and frees whole 4096byte pages at a time, and keeps track of which pages are free by theading a linked list throug the pages themselves.

### Code: Physical memory allocator

kalloc.c 负责物理内存的分配, 核心的数据结构是free list, 组织了空闲可用的内存页. run中用链表的形式存储了所有的空闲页.

main中调用了kinit来初始化分配器, 记录所有在内核结束位置到PHYSTOP中的page. kinit调用freerange通过对每一页调用kfree来向free list中添加内存. PGROUNDUP来确保page的4096byte对齐. kfree会将这个页全设置为1, 从而使得free后使用这个页的指令获得脏数据, 然后将这个页拼接到kmem.freelist的最前面

### Process address space

每个进程有独立的页表, 当xv6在不同进程中切换时, 同时也会切换页表. 每个进程使用从0开始到MAXVA的虚拟地址.

当一个进程向内核申请用户内存时, 内核会用kalloc分配物理内存, 然后添加PTE到进程的页表中

![[用户内存layout.png]]

### Code: sbrk

sbrk是一个用于进程内存扩容或缩小的系统调用, 通过proc.c/growproc实现, 调用uvmalloc或uvmdealloc来分配内存, 然后添加PTE到用户页表中

进程的页表除了映射到物理地址外, 还作为唯一的实体记录了那一块物理地址被分配给了那个进程

### Code: exec

exec是创建一个用户地址空间的系统调用, 它从文件系统中的一个地址初始化用户地址空间, 首先打开二进制path, 然后readELF头(描述了程序构成和需要被读入内存的部分)

exec首先检测magic number是否正确, 若正确则分配一个新的用户页表, 然后给每个ELF segment分配内存并load到对应的内存中, 然后分配并初始化用户栈

## Lab: page tables

### speed up system call (easy)

> Some operating systems (e.g., Linux) speed up certain system calls by sharing data in a read-only region between userspace and the kernel. This eliminates the need for kernel crossings when performing these system calls. To help you learn how to insert mappings into a page table, your first task is to implement this optimization for the getpid() system call in xv6.
> 
> When each process is created, map one read-only page at USYSCALL (a VA defined in memlayout.h). At the start of this page, store a struct usyscall (also defined in memlayout.h), and initialize it to store the PID of the current process. For this lab, ugetpid() has been provided on the userspace side and will automatically use the USYSCALL mapping. You will receive full credit for this part of the lab if the ugetpid test case passes when running pgtbltest.

hints
- You can perform the mapping in proc_pagetable() in kernel/proc.c.
- Choose permission bits that allow userspace to only read the page.
- You may find that mappages() is a useful utility.
- Don't forget to allocate and initialize the page in allocproc().
- Make sure to free the page in freeproc().

#### 分析

根据题意, 在创建一个进程时, 我们需要在proc_pagetable()函数中, 给USYSCALL这个位置映射一个只读页, 在开头的位置存储usyscall这个结构体. 然后在freeproc()释放进程时释放这个页.
由上可以分析得到具体步骤如下
- 在proc.h中给进程添加一个字段存储共享页地址
	- 添加一个usyscall类型的指针字段即可
- 在proc_pagetable()函数中给USYSCALL这个虚拟地址映射到usyscall所在物理页, 且设置只读权限
	- 需要注意的是还需要一个PTE_U权限, 否则用户空间无法访问![[map共享页.png]]
	- uvmunmap的作用是取消第二个参数开始的n个页映射
- 在allocateproc()时初始化这个页, 并存入pid
	- kalloc会返回一个物理地址, 向其中填充满无意义的值,  使用前需要用memset初始化一下![[分配共享页.png]]
- 在freeproc()中增加共享页释放逻辑
	- 在freeproc中释放usyscall的物理内存![[释放共享页.png]]
	- 此外还需要在释放页表时移除掉usyscall的映射, 否则可能会导致freewall递归释放页表时出错![[移除共享页映射.png]]
### print a page table (easy)

> To help you visualize RISC-V page tables, and perhaps to aid future debugging, your second task is to write a function that prints the contents of a page table.
> 
> Define a function called vmprint(). It should take a pagetable_t argument, and print that pagetable in the format described below. Insert if(p->pid\==1) vmprint(p->pagetable) in exec.c just before the return argc, to print the first process's page table. You receive full credit for this part of the lab if you pass the pte printout test of make grade.

The first line displays the argument to vmprint. After that there is a line for each PTE, including PTEs that refer to page-table pages deeper in the tree. Each PTE line is indented by a number of " .." that indicates its depth in the tree. Each PTE line shows the PTE index in its page-table page, the pte bits, and the physical address extracted from the PTE. Don't print PTEs that are not valid. In the above example, the top-level page-table page has mappings for entries 0 and 255. The next level down for entry 0 has only index 0 mapped, and the bottom-level for that index 0 has entries 0, 1, and 2 mapped.

hints
- You can put vmprint() in kernel/vm.c.
- Use the macros at the end of the file kernel/riscv.h.
- The function freewalk may be inspirational.
- Define the prototype for vmprint in kernel/defs.h so that you can call it from exec.c.
- Use %p in your printf calls to print out full 64-bit hex PTEs and addresses as shown in the example.

#### 分析

题意非常的明确, 需要做的事情也非常的直接
- 在defs.h中声明函数vmprint()
- 在vm.c中定义函数vmprint(), 参考freewalk函数实现递归访问page table
	- 为了能控制深度, 同时避免代码太复杂, 就用了一个字符串来表示前缀(只是因为这个只有三层所以可以行得通, 如果多一层都不行, 算是取了个巧)![[vmprint.png]]
- 在exec.c返回前打印pid\==1的page table, 用%p打印64b的地址


### Detecting which pages have been accessed (hard)

> Some garbage collectors (a form of automatic memory management) can benefit from information about which pages have been accessed (read or write). In this part of the lab, you will add a new feature to xv6 that detects and reports this information to userspace by inspecting the access bits in the RISC-V page table. The RISC-V hardware page walker marks these bits in the PTE whenever it resolves a TLB miss.
> 
> Your job is to implement pgaccess(), a system call that reports which pages have been accessed. The system call takes three arguments. First, it takes the starting virtual address of the first user page to check. Second, it takes the number of pages to check. Finally, it takes a user address to a buffer to store the results into a bitmask (a datastructure that uses one bit per page and where the first page corresponds to the least significant bit). You will receive full credit for this part of the lab if the pgaccess test case passes when running pgtbltest.

Some hints:
- Start by implementing sys_pgaccess() in kernel/sysproc.c.
- You'll need to parse arguments using argaddr() and argint().
- For the output bitmask, it's easier to store a temporary buffer in the kernel and copy it to the user (via copyout()) after filling it with the right bits.
- It's okay to set an upper limit on the number of pages that can be scanned.
- walk() in kernel/vm.c is very useful for finding the right PTEs.
- You'll need to define PTE_A, the access bit, in kernel/riscv.h. Consult the RISC-V manual to determine its value.
- Be sure to clear PTE_A after checking if it is set. Otherwise, it won't be possible to determine if the page was accessed since the last time pgaccess() was called (i.e., the bit will be set forever).
- vmprint() may come in handy to debug page tables.

需要完成的要求如下
- 在sysproc.c中实现系统调用pgaccess(), 接受三个参数用户页开始的虚拟地址, 需要检查的页数, 用户空间缓冲区
- 用户空间的缓冲区以bitmask(第一页在最小位)的形式存储结果, 现在内核中完成然后copyout到用户空间中
	- 思路还是很直接的, 写起来也比较简单, 只在解析参数这里错了, argint拿第二个参数时第一个参数应该是1![[pageaccess.png]]
- 在riscv.h中定义PTE_A作为访问标志, 需要确保check后将其清除
	- 对应PTE_A![[PTE_A.png]]
