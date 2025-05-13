> 本章没有lab
___

## xv6 book

driver(驱动), 是操作系统中用于管理特定设备的代码, 通常负责配置硬件, 告知硬件如何操作, 处理结果中断以及与需要I/O的进程交互. 驱动代码可能会比较棘手, 因为驱动和被管理的硬件是并行的, 此外驱动还必须理解复杂的硬件接口.
设备通常通过interrupt来被操作系统的trap机制处理.
驱动通常在两个context中执行, 一个是top half 运行在进程的内核线程中, 一个是bottom half 运行在中断时间内.top half被期望执行IO的system call(如read,write)调用, 通常请求硬件开始执行一个操作, 然后等待操作结束. 当硬件执行完操作后会产生一个中断信号, 唤醒等待的进程.

### Code: Console input

console.c是一个简单的驱动, 接受字符串键入, 通过UART串行端口硬件连接到RISC-V. console驱动一次收集一行输入, 处理特殊的输入字符, 如空格和contol+u. 用户进程如shell通过read系统调用来获取console的输入, 当我们键入字符时, 输入会被QEMU模拟的UART硬件传输到cv6.
UART硬件和QEMU模拟的16550芯片交互, 在真实的电脑中, 16550会管理一个RS232 serial link 连接到终端或者其他计算机, QEMU中则是连接到键盘和显示.

UART硬件对软件来说是一种内存映射的控制寄存器, 有一部分物理地址映射到UART设备, 执行loads和stores时直接和硬件交互而不是RAM. UART的映射地址是从0x10000000开始或者UART0.
Their offsets from UART0 are defined in (kernel/uart.c:22). For example, the LSR register contain bits that indicate whether input characters are waiting to be read by the software. These characters (if any) are available for reading from the RHR register. Each time one is read, the UART hardware deletes it from an internal FIFO of waiting characters, and clears the “ready” bit in LSR when the FIFO is empty. The UART transmit hardware is largely independent of the receive hardware; if software writes a byte to the THR, the UART transmit that byte.

main调用了consoleinit来初始化UART硬件, 并配置UART在每次收到一个字节的输入时触发一个输入中断, 以及在发送完一个字节输出后触发传输完成中断 

The xv6 shell reads from the console by way of a file descriptor opened by init.c (user/init.c:19). Calls to the read system call make their way through the kernel to consoleread (kernel/con- sole.c:82). consoleread waits for input to arrive (via interrupts) and be buffered in cons.buf, copies the input to user space, and (after a whole line has arrived) returns to the user process. If the user hasn’t typed a full line yet, any reading processes will wait in the sleep call (kernel/con- sole.c:98) (Chapter 7 explains the details of sleep).

When the user types a character, the UART hardware asks the RISC-V to raise an interrupt, which activates xv6’s trap handler. The trap handler calls devintr (kernel/trap.c:177), which looks at the RISC-V scause register to discover that the interrupt is from an external device. Then it asks a hardware unit called the PLIC  to tell it which device interrupted (kernel/trap.c:186). If it was the UART, devintr calls uartintr.

uartintr (kernel/uart.c:180) reads any waiting input characters from the UART hardware and hands them to consoleintr (kernel/console.c:138); it doesn’t wait for characters, since future input will raise a new interrupt. The job of consoleintr is to accumulate input characters in cons.buf until a whole line arrives. consoleintr treats backspace and a few other characters specially. When a newline arrives, consoleintr wakes up a waiting consoleread (if there is one).
Once woken, consoleread will observe a full line in cons.buf, copy it to user space, and return (via the system call machinery) to user space

### Code: Console output

A write system call on a file descriptor connected to the console eventually arrives at uartputc (kernel/uart.c:87). The device driver maintains an output buffer (uart_tx_buf) so that writing processes do not have to wait for the UART to finish sending; instead, uartputc appends each character to the buffer, calls uartstart to start the device transmitting (if it isn’t already), and returns. The only situation in which uartputc waits is if the buffer is already full. Each time the UART finishes sending a byte, it generates an interrupt. uartintr calls uartstart,
which checks that the device really has finished sending, and hands the device the next buffered output character. Thus if a process writes multiple bytes to the console, typically the first byte will be sent by uartputc’s call to uartstart, and the remaining buffered bytes will be sent by uartstart calls from uartintr as transmit complete interrupts arrive.

A general pattern to note is the decoupling of device activity from process activity via buffering and interrupts. The console driver can process input even when no process is waiting to read it; a subsequent read will see the input. Similarly, processes can send output without having to wait for the device. This decoupling can increase performance by allowing processes to execute concur- rently with device I/O, and is particularly important when the device is slow (as with the UART) or needs immediate attention (as with echoing typed characters). This idea is sometimes called I/O concurrency.

### Timer interrupt

Xv6 uses timer interrupts to maintain its clock and to enable it to switch among compute-bound processes; the yield calls in usertrap and kerneltrap cause this switching. Timer inter- rupts come from clock hardware attached to each RISC-V CPU. Xv6 programs this clock hardware to interrupt each CPU periodically.

RISC-V requires that timer interrupts be taken in machine mode, not supervisor mode. RISC- V machine mode executes without paging, and with a separate set of control registers, so it’s not practical to run ordinary xv6 kernel code in machine mode. As a result, xv6 handles timer interrupts completely separately from the trap mechanism laid out above

Code executed in machine mode in start.c, before main, sets up to receive timer interrupts (kernel/start.c:57). Part of the job is to program the CLINT hardware (core-local interruptor) to generate an interrupt after a certain delay. Another part is to set up a scratch area, analogous to the 51 trapframe, to help the timer interrupt handler save registers and the address of the CLINT registers. Finally, start sets mtvec to timervec and enables timer interrupts. A timer interrupt can occur at any point when user or kernel code is executing; there’s no way for the kernel to disable timer interrupts during critical operations. Thus the timer interrupt handler must do its job in a way guaranteed not to disturb interrupted kernel code. The basic strategy is for the handler to ask the RISC-V to raise a “software interrupt” and immediately return. The RISC-V delivers software interrupts to the kernel with the ordinary trap mechanism, and allows the kernel to disable them. The code to handle the software interrupt generated by a timer interrupt can be seen in devintr (kernel/trap.c:204).

The machine-mode timer interrupt vector is timervec (kernel/kernelvec.S:93). It saves a few registers in the scratch area prepared by start, tells the CLINT when to generate the next timer interrupt, asks the RISC-V to raise a software interrupt, restores registers, and returns. There’s no C code in the timer interrupt handler