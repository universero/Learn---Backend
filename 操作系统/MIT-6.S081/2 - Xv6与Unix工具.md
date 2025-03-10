
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
```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

#define FIRST_NUM 2
#define LAST_NUM  35

void sieve(const int* left);

int
main(int argc, char* argv[])
{
    // define and initialise the pipe
    int p[2];
    pipe(p);

    if (fork() > 0) // the parent process
    {
        // write 2 to 35 into the pipe
        for (int i = FIRST_NUM; i <= LAST_NUM; i++)
        {
            write(p[1], &i, sizeof(int));
        }
        close(p[1]);
        close(p[0]);
        wait(0);
        exit(0);
    }
    // the child process
    close(p[1]);
    sieve(p);
    close(p[0]);
    exit(0);
}
void
sieve(const int* left)
{
    int buf;
    int right[2];

    // 读取第一个数作为素数
    if (read(left[0], &buf, sizeof(int)) == 0)
    {
        close(left[0]);
        return;
    }

    int prime = buf;
    printf("prime %d\n", prime);

    pipe(right);

    if (fork() > 0)
    {
        close(right[0]);

        while (read(left[0], &buf, sizeof(int)) > 0)
        {
            if (buf % prime != 0)
            {
                write(right[1], &buf, sizeof(int));
            }
        }
        close(right[1]);
        close(left[0]);
        wait(0);
        exit(0);
    }
    close(right[1]);
    close(left[0]);
    sieve(right);
    close(right[0]);
    exit(0);
}
```
- main函数中父进程需要做的很简单，将2到35写入p即可
- 按照示例图的形式来理解，所有的子线程(包括子线程的子线程)需要做的事情是，从left管道获取数，然后将符合要求的写入右管道。这里需要注意的是，每个管道中第一个获取到的数一定是个素数，然后只需要写入不是这个数的倍数的数即可
- 主要的难点在于如何正确的关闭文件描述符。对应main中的父进程来说，写入2到35后直接关闭读写两端即可。对于子线程来说，在进入递归前，需要先关闭p的写，因为用不到。在递归结束后再关闭p的读，因为读完了。对于递归函数内来说，一个数都读取不到的话，直接关闭left的读即可，因为left的写也就是上一个进程的right写已经被关掉了。如果读到了一个数，那么再开一个管道，关闭读取，然后向其中写入，写完后关闭left的读与right的写。对于递归函数中的子线程来说，left的写调用前已经关掉right的写和left的读都用不到关掉，只需要递归结束后关闭right的读即可。
- 能力有限，这里很难用语言描述清楚，需要自己好好理解。主要是能做到关掉所有的写，避免读的无限阻塞。以及避免重复的关闭导致出现的页错误等奇怪问题。

## find (moderate)

>Write a simple version of the UNIX find program: find all the files in a directory tree with a specific name. Your solution should be in the file user/find.c.

- Look at user/ls.c to see how to read directories.
- Use recursion to allow find to descend into sub-directories.
- Don't recurse into "." and "..".
- Changes to the file system persist across runs of qemu; to get a clean file system run make clean and then make qemu.
- You'll need to use C strings. Have a look at K&R (the C book), for example Section 5.5.
- Note that == does not compare strings like in Python. Use strcmp() instead.
- Add the program to UPROGS in Makefile.
### Analysis

- The main logic of the find program is clearly, recusively traverse the directory, if it's a file, check the name whether equals to the target, or it's a directory, skip the . and .. and recusively repeat the logic.
- In xv6, a directory contains all it's entry which you can read them from the fd pointing to the directory. And you can get a stat of a file from the system call fstat

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"

void find(char* path, char* target);
char* fname(char* path);

int
main(int argc, char* argv[])
{
    if (argc < 3)
    {
        fprintf(2, "[usage]: find dirname filename");
        exit(1);
    }

    char* dir = argv[1];
    char* target = argv[2];

    find(dir, target);
    exit(0);
}

void
find(char* path, char* target)
{
    char buf[512], *p;
    int fd;
    struct dirent de;
    struct stat st;

    if ((fd = open(path, 0)) < 0)
    {
        fprintf(2, "find: cannot open %s\n", path);
        return;
    }

    if (fstat(fd, &st) < 0)
    {
        fprintf(2, "find: cannot stat %s\n", path);
        close(fd);
        return;
    }

    switch (st.type)
    {
    // if open a file, check whether it's the target
    case T_FILE:
        {
            char* filename = fname(path);
            if (strcmp(filename, target) == 0)
            {
                fprintf(0, "%s\n", path);
            }
        }
        break;
    case T_DIR:
        if (strlen(path) + 1 + DIRSIZ + 1 > sizeof buf)
        {
            printf("find: path too long\n");
            break;
        }
        strcpy(buf, path);
        p = buf + strlen(buf);
        *p++ = '/';
        while (read(fd, &de, sizeof(de)) == sizeof(de))
        {
            if (de.inum == 0)
                continue;
            if (strcmp(de.name, ".") == 0 || strcmp(de.name, "..") == 0)
            {
                continue;
            }
            memmove(p, de.name, DIRSIZ);
            p[DIRSIZ] = 0;
            find(buf, target);
        }
        break;
    }
    close(fd);
}

char*
fname(char* path)
{
    char* p;
    int n = 1;

    // Find first character after last slash.
    for (p = path + strlen(path); p >= path && *p != '/'; p--)
        n++;
    p++;

    char* buf = malloc(n);
    memmove(buf, p, n);
    return buf;
}
```

## xargs

> Write a simple version of the UNIX xargs program: read lines from the standard input and run a command for each line, supplying the line as arguments to the command. Your solution should be in the file user/xargs.c.

- Use fork and exec to invoke the command on each line of input. Use wait in the parent to wait for the child to complete the command.
- To read individual lines of input, read a character at a time until a newline ('\n') appears.
- kernel/param.h declares MAXARG, which may be useful if you need to declare an argv array.
- Add the program to UPROGS in Makefile.
- Changes to the file system persist across runs of qemu; to get a clean file system run make clean and then make qemu.
### Analysis

- 题目要求很简单，从标准输入中读取输出然后每行都拼接到当前的命令上，再用子进程执行
- 一开始采用下面的方式读取输入
```c
if ((n = read(0, buf, sizeof(char) * 512)) < 0)
{
	fprintf(2, "read failed");
	exit(1);
}
```
这里没有按照提示的来，希望一次性读到所有的然后再处理，结果就是一直只能读到一个字符，没法正确处理，需要换成gets来处理。原因应该是两个命令间的管道不是一次性写入所有的内容，而是异步的。
同时，为了解决一次性读入所有字符后的分割还实现了一个strtok函数，实际上应该不用，就加到了ulib.c中作为以后可能用到的库函数。
```c
char*
strtok(char* src, const char* delim)
{
    static char* last = 0;

    if (src != 0)
        last = src;

    if (last == 0)
        return 0;

    // 跳过开头的分隔符
    for (; *last != '\0' && strchr(delim, *last) != 0; last++);

    // 如果到达字符串末尾，返回 NULL
    if (*last == '\0')
    {
        return 0;
    }

    // 找到子字符串的起始位置
    char* token_start = last;
    // 找到下一个分隔符的位置
    for (; *last != '\0' && strchr(delim, *last) == 0; last++);

    // 如果找到分隔符，将其替换为 \0，并更新 last 的位置
    if (*last != '\0')
    {
        *last = '\0';
        last++;
    }
    else
    {
        // 如果没有找到分隔符，说明这是最后一个子字符串
        last = 0;
    }

    return token_start;
}
```

测试xargs时还发现一个错误，find中误把0当标准输出了，将文件输出写入了0导致后续读不到输入。

```c
#include "kernel/types.h"
#include "kernel/param.h"
#include "user/user.h"

void xargs(int argc, char* xargv[], char* buf);

int
main(int argc, char* argv[])
{
    char buf[512], *xargv[MAXARG];
    for (int i = 1; i < argc; i++)
        xargv[i - 1] = argv[i];

    while (gets(buf, 512) && buf[0] != '\0')
    {
        xargs(argc-1, xargv, buf);
    }
    while (wait(0) != -1);
    exit(0);
}

void
xargs(int argc, char* xargv[], char* buf)
{
    xargv[argc++] = buf;

    for (char* p = buf; *p; p++)
    {
        if (*p == ' ')
        {
            *p = '\0';
            xargv[argc++] = p + 1;
        }
        else if (*p == '\n')
            *p = '\0';
    }

    if (fork() == 0)
    {
        exec(xargv[0], xargv);
        exit(1);
    }
}
```

整体的思路如下
- 首先xargs xxx命令实际需要执行的是xxx + 额外读取到的参数，那么我们先用一个数组xargv存储初始的参数，然后用gets(buf,512)获取一行输出。
- 有了参数后，接下来我们需要组装命令，原有参数不变，并将buf拼接到最后面。如果每行只有一个参数的话，那么将buf结尾的\n去掉即可，但是如果一行有多个参数的话，这样是不够的。可以遍历buf，将其分割成多个字符串并依次拼接到xargv后面，最后再去除\n，这样就能得到完整的命令了，接下来只要子进程exec，父进程重复读取就行。
自己写(加偷瞄博客😂)花了大半天的时间，主要遇到了如下问题
- find.c中输出位置不对，不知道为什么测试还是过了，但是导致这里用find测试时读取不到输入
- 没有听提示的，逐个读取，而是想一行读取，然后用strtok分割，导致每次都只能读到一个字符
- 调用xargs时应该是argc-1而不是argc，因为xargv比argv少一个元素，导致一直多了一个null，跑不成功
- 对buf进行切割的代码段是从博客抄来的，感觉自己应该写不出这么简洁高效的代码
- 然后一开始很多地方进行了字符串的复制，后来发现根本没有必要，因为父子进程的变量是复制关系，不会相互影响。