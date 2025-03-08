
>This chapter is corresponding to [Lab: Xv6 and Unix utilities](https://pdos.csail.mit.edu/6.828/2021/labs/util.html)
>[xv6: a simple, Unix-like teaching operating system](https://pdos.csail.mit.edu/6.828/2021/xv6/book-riscv-rev2.pdf)

___

## Boot xv6 (easy)

After the first part Overview & Environment Configuration, we can start the xv6 successfully with the command `make qemu`
And you can try the executable programs, which is included by mkfs in the initial file system, like ls.
Pay attention to that xv6 has no ps command, but if you type ctrl-p, the kernel  will print information about each process.
To quit qeum type `ctrl-a x`

## Grading

you can run `make grade` to test your solution with the grading

## Sleep (easy)

> Implement the UNIX program sleep for xv6; your sleep should pause for a user-specified number of ticks. A tick is a notion of time defined by the xv6 kernel, namely the time between two interrupts from the timer chip. Your solution should be in the file user/sleep.c.

The sleep program has following demands and hints:
- argument is padded as a string, and we need to convert it to integer using atoi(user/ulib,c)
- using the system call sleep
- seeing the implement of sleep system call in kernel/sysproc.c/sys_sleep, and user/user.h for the C definition of sleep callable from a user program, and user/usys.S for the assenmbler code that jumps from user code into the kernel for sleep
- make sure main calls exit() in order to exit
- if you want to test the single assignment, type: `./grade-lab-util sleep`, or you can type `make GRADEFLAGS=sleep grade`
- If the user forgets to pass an argument, sleep should print an error message.
### analysis

- atoi
```c
int
atoi(const char *s)
{
  int n;

  n = 0;
  while('0' <= *s && *s <= '9')
    n = n*10 + *s++ - '0';
  return n;
}
```
get the number presented by the continuous string like '123'

- the implement of sleep system call
```c
uint64
sys_sleep(void)
{
  int n;
  uint ticks0;

  // get the nth 32-bit system call argument
  // you can follow the function call to find the implement of it
  if(argint(0, &n) < 0)
    return -1;

  // acquire the ticks lock
  acquire(&tickslock);
  // ticks is a variable that records the times of clock interrupt
  ticks0 = ticks;
  while(ticks - ticks0 < n){
    // if the sleep proc has been killed
    if(myproc()->killed){
      release(&tickslock);
      return -1;
    }
    // let the current proc sleep, but the release of it is a bit confused now, may we can find it out later
    sleep(&ticks, &tickslock);
  }
  // release the ticks lock
  release(&tickslock);
  return 0;
}
```

- the C definitionof sleep `int sleep(int)`, get the number of sleep ticks and return wheter it sleep successfully
- Then see the usys.S to find the assembler code that jumps from user code into the kernel for sleep. If you not find the file under user, run `perl usys.pl > usys.S` firstly.
```shell
# marks the sleep as a global symbol, 
# making it accessible to other files or modules
.global sleep 
# defines the entry point of the sleep functino
sleep:
 # Loads the system call number SYS_sleep intp register a7
 # li stands for "load immediate", which loads a constants value into a register
 # a7 is the register used in RISC-V to pass the system call number
 li a7, SYS_sleep
 # executes an environment call, triggering the system call.
 ecall
 # returns from the function, after the system call completes, control returns to user mode.
 ret
```
- add the sleep into the UPROGS in the Makefile as following. The reason why there is a underline is that in xv6, user programs are compiled into executable files with an underscore (`_`) prefix
![[UPROGS-sleep.png]]
- finally implement the sleep.c in /user, the logic is simple, checking the args and call the system call sleep
```c
//
// Created by univero on 25-3-7 in ECNU.
//
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int
main(int argc, char* argv[])
{
    int n;

    if (argc < 2)
    {
        fprintf(2, "Usage: sleep second\n");
        exit(1);
    }
    # Here may have some error, because no function to ensure the argv[1] is actually a int, but it's enough to pass the test.
    n = atoi(argv[1]);
    sleep(n);
    exit(0);
}

```

## pingpong(easy)

>Write a program that uses UNIX system calls to ''ping-pong'' a byte between two processes over a pair of pipes, one for each direction. The parent should send a byte to the child; the child should print "`<pid>`: received ping", where `<pid>` is its process ID, write the byte on the pipe to the parent, and exit; the parent should read the byte from the child, print "`<pid>`: received pong", and exit. Your solution should be in the file user/pingpong.c.

here are some hints
- Use pipe to create a pipe
- Use fork to create a child
- Use read to read from the pipe, and write to write the pipe
- Use getpid to find the process ID of the calling process
- Add the program to UPROGS in Makefile
- You can only use the library functions in the user/ulib.c, user/printf.c and user/umalloc.c

```c
//
// Created by univero on 25-3-7 in ECNU.
//

#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int
main(int argc, char* argv[])
{
    int p[2];
    char buf[1];

    // Create a pipe
    if (pipe(p) < 0) {
        fprintf(2, "pipe failed\n");
        exit(1);
    }

    // Fork a child process
    if (fork() == 0) {
        // Child process
        // Read from the pipe
        if (read(p[0], buf, sizeof(buf)) != sizeof(buf)) {
            fprintf(2, "read failed\n");
            exit(1);
        }
        printf("%d: received ping\n", getpid());

        // Write to the pipe
        if (write(p[1], buf, sizeof(buf)) != sizeof(buf)) {
            fprintf(2, "write failed\n");
            exit(1);
        }
    } else {
        // Parent process
        // Write to the pipe
        buf[0] = 'a'; // Send a single byte
        if (write(p[1], buf, sizeof(buf)) != sizeof(buf)) {
            fprintf(2, "write failed\n");
            exit(1);
        }

        // Wait for the child to finish
        wait(0);

        // Read from the pipe
        if (read(p[0], buf, sizeof(buf)) != sizeof(buf)) {
            fprintf(2, "read failed\n");
            exit(1);
        }
        printf("%d: received pong\n", getpid());
    }

    exit(0);
}
```
**the most important thing is that each print needs a \n**, or you will not pass the test.
And it's necessary to deal the return value of functions like pipe, read and write which indicates the function succeed or not. At the first time I do the lab, I forget to handle it.

## primes (moderate/hard)

>Write a concurrent version of prime sieve using pipes. This idea is due to Doug McIlroy, inventor of Unix pipes. The picture halfway down [this page](http://swtch.com/~rsc/thread/) and the surrounding text explain how to do it. Your solution should be in the file user/primes.c.
>Your goal is to use pipe and fork to set up the pipeline. The first process feeds the numbers 2 through 35 into the pipeline. For each prime number, you will arrange to create one process that reads from its left neighbor over a pipe and writes to its right neighbor over another pipe. Since xv6 has limited number of file descriptors and processes, the first process can stop at 35.

Here are some hints
- Be careful to close file descriptors that a process doesn't need, because otherwise your program will run xv6 out of resources before the first process reaches 35.
- Once the first process reaches 35, it should wait until the entire pipeline terminates, including all children, grandchildren, &c. Thus the main primes process should only exit after all the output has been printed, and after all the other primes processes have exited.
- Hint: read returns zero when the write-side of a pipe is closed.
- It's simplest to directly write 32-bit (4-byte) ints to the pipes, rather than using formatted ASCII I/O.
- You should create the processes in the pipeline only as they are needed.
- Add the program to UPROGS in Makefile.
### Analysis
- Firstly, create the prime.c in /user and add the prime to UPROG in make file
- 感觉这里的分析有点难写就用中文了。不知道是我英语太差了，还是lab的文档写的太过简略，看完后压根不知道要做什么。搜了几篇博客后发现，要实现的应该是一个并发的素数筛，类似于厄拉托斯素数筛的并发版。大致的逻辑如下图
![[并行素数筛.png]]
- 知道了要做什么之后，也算是有了方向。先看看厄拉托斯素数筛是什么。简单来说，如果要找出n及以下的所有素数，那么从2到n都先假设为素数，然后从2开始依次将遍历到的素数的倍数标记为非素数，如遍历到2就把4,6,8都标记为非素数.这样就能找到所有n及以下的素数
- 那么对于prime这个程序来说,我们要做的就是将这么一个依次遍历的过程改成一个流水线的过程, 主线程向pipe中写入2到35, 第一个子线程输出2 然后 drop掉所有2的倍数, 写入所有不整除2的数
- 实现思路
- **这个素数筛的实现思路比较简单，最困难的是如何处理好fd的close**，重复关闭close会导致usertrap的错误，需要把何时关闭捋得很清楚。最终靠着deepseek-R1解决了fd关闭的问题😅
- 另外就是文件名应该是primes，不小心弄成了prime，直接跑可以过，但是测试过不了，还纳闷了半天。