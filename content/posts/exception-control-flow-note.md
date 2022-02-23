---
title: "CSAPP 读书笔记：异常控制流"
date: 2022-01-16T16:41:42+01:00
draft: true
tags: ["CSAPP","OS"]
summary: "在计算机运行过程中，程序计数器将依次指向一系列的值：$a_0, a_1, ..., a_n$。其中，$a_k$ 是其对应指令 $I_k$ 的地址。每个从 $a_k$ 到 $a_{k+1}$ 的转换都称为控制转移（Control Transfer），一系列的控制转移则称为处理器的控制流（Control Flow）..."
---

在计算机运行过程中，程序计数器将依次指向一系列的值：$a_0, a_1, ..., a_n$。其中，$a_k$ 是其对应指令 $I_k$ 的地址。每个从 $a_k$ 到 $a_{k+1}$ 的转换都称为控制转移（Control Transfer），一系列的控制转移则称为处理器的控制流（Control Flow）。

最简单的控制流便是程序中指令的顺序执行、跳转、调用以及返回，不过系统还必须对其状态的变化做出一定反应。这些变化无法被内部程序变量捕获，也不一定与程序的执行有关。比如，数据包到达网络适配器后需要被存储到内存中，程序访问磁盘中的数据需要得知其何时准备就绪，父进程必须在其子进程终止时收到通知。

现代系统通过异常控制流（Exceptional Control Flow，ECF）来处理上述情况，它应用于计算机系统的所有级别中：

- 硬件检测到的事件通过 ECF 将控制转移到异常处理程序（Exception Handler）；
- ECF 是操作系统实现 I/O、进程和虚拟内存的基本机制；
- 程序可以通过 ECF 请求一些操作系统服务，如向磁盘写入数据、读取网络中收到的数据以及创建新进程等；
- 一些编程语言通过 ECF 使程序进行非本地跳转（即违背通常的调用、返回栈规则的跳转）以对错误进行响应。

## 异常

异常（Exception）是为了响应处理器状态变化而在控制流中发生的突然变化，下图展示了其基本思想：

![20220208223330](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220208223330.png)

处理器状态的变化称为事件（Event），它可能与当前指令（$I_{curr}$）的执行直接相关，比如算术溢出或除数为零。 事件也可能与当前指令的执行无关，比如系统计时器关闭或 I/O 请求完成。

### 异常处理

异常处理涉及到软件和硬件的密切合作，因此很容易将不同组件执行的工作相混淆。系统中每种可能的异常都对应了一个唯一的非负整数，即异常数字（Exception Number）。当计算机系统启动时，操作系统会初始化一个跳转表，称为异常表（Exception Table）。异常数字是异常表的索引，而异常表中的每个条目 k 均包含了异常 k 的处理程序地址：

![20220209224522](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220209224522.png)

当处理器检测到事件的发生时，首先将确定异常数字，然后根据异常表调用对应的异常处理程序。

![20220210212750](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220210212750.png)

异常与过程调用类似，但也有一些重要区别：

- 异常的返回地址要么是当前指令（$I_{curr}$），要么是下一条指令（$I_{next}$）；
- 处理器还会将一些额外的处理器状态压入栈中；
- 当控制从用户程序转移到内核时，一切都将被压入到内核的栈中；
- 异常处理程序在内核态运行，因此可以访问所有系统资源。

### 异常的分类

![20220210214222](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220210214222.png)

#### 中断

中断（Interrupt）异步发生，因为它是由处理器外部的 I/O 设备发出的信号产生的。

![20220210215042](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220210215042.png)

当前指令执行完毕后，处理器注意到中断引脚变高，于是从系统总线读取异常数字，然后调用对应的中断处理程序。当处理程序返回时，它将控制权返回给下一条指令。随后程序继续执行，就好像中断从未发生过一样。
其余种类的异常作为当前指令的执行结果同步发生，我们将这类指令称为故障指令（Faulting Instruction）。

#### 陷阱和系统调用

与中断处理程序一样，陷阱（Trap）处理程序也将控制返回给下一条指令。其最重要的用途是在用户程序和内核之间提供接口，即系统调用（System Call）。

用户程序通过系统调用向内核请求服务，如读取文件（`read`）、创建新进程（`fork`）、加载新程序（`execve`）和终止当前进程（`exit`）等。

![20220210221251](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220210221251.png)

在程序员看来，系统调用和常规函数没有什么区别。但常规函数运行在用户态，因此其可执行的指令类型受限，也只能访问用户栈。而系统调用运行在内核态，能够执行特权指令并访问内核栈。

#### 故障

故障（Faulting）是由一些错误状况引起的异常，而这些错误情况有可能被处理程序修正，否则将返回到内核中的中止例程（图中的`abort`）：

![20220210222537](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220210222537.png)

#### 中止

与故障相比，导致中止（Abort）发生的错误状况无法挽救，通常是硬件出现问题，如 RAM 位损坏引起的奇偶校验错误。中止处理程序永远不会将控制权返回给应用程序：

![20220210223641](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220210223641.png)

### Linux/x86-64 系统中的异常

| Exception Number | Description |   Exception Class   |
| ---------------- | ----------- | ---- |
|        0          | Divide error | Fault |
| 13 | General protection fault | Fault |
| 14 | Page fault | Fault |
| 18 | Machine check | Abort |
| 32-255 | OS-defined exception | Interrupt or Trap |

#### 故障和中止

- 除法故障：当应用程序尝试除以 0 或除法指令的结果对目标操作数来说太大时，就会发生除法故障。Unix 不会试图纠正除法故障，而是直接中止程序；
- 一般保护性故障：若程序引用了未定义的虚拟内存区域，或试图写入只读文本段等，可能引起一般保护性故障。Linux 不会试图纠正该故障，Shell 一般称其为分段故障（Segmentation Faults）；
- 页面故障：当程序引用的虚拟地址对应的页面不在内存而在磁盘上时，将导致页面故障。处理程序将磁盘上合适的虚拟内存页面映射到物理内存页面，然后重新执行故障指令；
- 机器检查：在执行故障指令期间检测到致命的硬件错误时，会发生机器检查，处理程序永远不会将控制权返回给应用程序。

#### 系统调用

![20220213214429](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220213214429.png)

上图中的每个系统调用都有一个唯一的数字，对应了内核中跳转表的偏移量。该跳转表与上文提到的异常表不同。

C 标准库为大多数系统调用提供了一组包装函数（Wrapper Function），它们比直接使用系统调用更加方便。系统调用及其相关的包装函数统称为系统级函数。举例来说，我们可以使用系统级函数`write`代替`printf`：

```c
int main() {
write(1, "hello, world\n", 13);
_exit(0);
}
```

X86-64 系统通过`syscall`指令使用系统调用，其所有参数均通过寄存器传递。按照惯例，寄存器 %rax 保存系统调用编号，寄存器 %rdi、%rsi、%rdx、%r10、%r8 和 %r9 依次保存各参数的值。系统调用的返回值将写入到寄存器 %rax 中，若为负则表示发生了与负`errno`相关的错误。因此，上面的程序可以直接用汇编语言表示为：

![20220213221637](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220213221637.png)

## 进程

进程是正在执行的程序实例，而系统中的每个程序都在进程的上下文中运行。上下文包含了程序正确运行所需的状态，如程序代码和存储在内存中的数据、栈、寄存器、程序计数器、环境变量和打开的文件描述符集合。

进程为应用提供了两个关键抽象：

- 一个独立的逻辑控制流，让我们产生程序独占处理器的错觉；
- 一个私有的地址空间，让我们产生程序独占内存的错觉。

### 逻辑控制流

![20220213223239](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220213223239.png)

进程轮流使用处理器。每个进程执行其流程的一部分，然后在其他进程执行时被抢占（即暂时挂起）。

### 并发流

执行时间重叠的两个逻辑控制流称为并发流（Concurrent Flow），它们并发运行。如上图 8.12 所示，进程 A 和 进程 B 并发运行，但进程 B 和进程 C 则不是。

并发流的概念与处理器的核数以及计算机的数量无关，只要两个逻辑控制流在时间上重叠，那么它们便是并发的。如果两个逻辑控制流在不同的处理器内核或计算机上同时运行，我们就称它们为并行流（Parallel Flow）。显然，并行流是并发流的子集。

### 私有地址空间

进程为程序提供了私有地址空间。它是程序独享的，与空间内特定地址相关的内存字节通常不能被其他任何进程读取或写入。尽管私有地址空间的内容不同，但其具有相同的组织结构：

![20220214215948](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220214215948.png)

地址空间底部是为用户程序保留的，代码段总是从地址 0x400000 开始。地址空间顶部是为内核保留的，包含了内核为进程执行指令（如程序执行系统调用）时使用的代码、数据和栈。

### 用户态和内核态

处理器通过保存在控制寄存器中的模式位（Mode Bit）来识别进程当前享有的特权。当模式位被设置时，进程运行在内核态（Kernel Mode），反之则运行在用户态（User Mode）。在内核态中运行的程序可以执行指令集中的任意指令，并且可以访问系统中的任意位置。而在用户态中运行的程序则受到限制，只能使用系统调用间接地访问内核代码和数据。

应用程序的进程只能通过异常来从用户态切换到内核态。当异常发生且控制转移到异常处理程序时，处理器切换到内核态。随后异常处理程序在内核态中运行，处理器将在它返回时切换回用户态。

### 上下文切换

在进程执行期间，内核可以暂时挂起当前进程并重启先前被抢占的进程，这称为调度（Scheduling）。内核调度新进程是通过上下文切换（Context Switch）机制来实现的，该机制：

- 保存当前进程的上下文；
- 恢复之前被抢占进程的上下文；
- 将控制权转移给新进程。

程序使用系统调用时可能会发生上下文切换。比如系统调用`read`需要访问磁盘中的数据，内核可以通过上下文切换来调度另一个进程，这样就无需等待数据从磁盘加载到内存。

![20220215211459](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220215211459.png)

## 系统调用错误处理

当执行 Unix 系统级函数遇到错误时，它们会返回 -1 并设置全局整型变量`errno`的值。因此我们可以在程序中检查调用是否发生错误，如：

```c
if ((pid = fork()) < 0) {
  fprintf(stderr, "fork error: %s\n", strerror(errno));
  exit(0);
  }
```

其中，`strerror`函数会根据`errno`的值返回相关的文本字符串。我们可以定义一个错误报告（Error-reporting）函数，从而将上述代码进行简化：

```c
void unix_error(char *msg) /* Unix-style error */
  {
  fprintf(stderr, "%s: %s\n", msg, strerror(errno));
  exit(0);
  }

if ((pid = fork()) < 0)
  unix_error("fork error");
```

我们再定义一个错误处理（Error-handling）函数，将代码进一步地简化：

```c
pid_t Fork(void)
  {
  pid_t pid;
  if ((pid = fork()) < 0)
  unix_error("Fork error");
  return pid;
  }

pid = Fork();
```

这样我们就可以使用包装函数`Fork`代替`fork`及其错误检查代码。本书使用的包装函数均定义在头文件 [<csapp.h.>](http://csapp.cs.cmu.edu/2e/ics2/code/include/csapp.h) 中。

## 进程控制

### 获取进程 ID

每个进程都有一个唯一且大于 0 的进程 ID（PID）。函数`getpid`返回调用进程的 PID，而函数`getppid则返回创建调用进程的进程（父进程） 的 PID。

```c
#include <sys/types.h>
#include <unistd.h>
pid_t getpid(void);
pid_t getppid(void);
```

二者返回值的类型为`pid_t`，它在 Linux 系统的 sys/types.h 中定义为 int。

### 创建和中止进程

在程序员看来，进程有三种状态：

- 运行（Running）：该进程要么在 CPU 中执行，要么在等待内核调度；
- 停止（Stopped）：进程执行暂停，并且不会被调度；
- 终止（Terminated）：进程永久地停止。

函数`exit`会以参数`status`作为退出状态终止进程：

```c
#include <stdlib.h>
void exit(int status);
```

父进程通过调用`fork`函数创建一个新的子进程：

```c
#include <sys/types.h>
#include <unistd.h>
pid_t fork(void);
```

子进程将获得一个与父进程相同但独立的用户级虚拟内存空间副本，包括代码、数据、堆、共享库和用户栈等。它还会得到与父进程相同的文件描述符副本，因此可以在调用`fork`时读写任意父进程打开的文件。父进程和子进程之间最显著的区别便是 PID 不同。

函数`fork`执行一次却返回两次：在父进程中返回子进程的 PID，在子进程中返回 0。由于子进程的 PID 始终大于 0 ，因此我们可以通过返回值判断程序在哪个进程中执行。

```c
#include "csapp.h"

int main() 
{
    pid_t pid;
    int x = 1;

    pid = Fork(); 
    if (pid == 0) {  /* Child */
        printf("child : x=%d\n", ++x); 
        exit(0);
    }

    /* Parent */
    printf("parent: x=%d\n", --x); 
    exit(0);
}
```

该程序编译后运行的可能结果为：

```shell
linux> ./fork
parent: x=0
child : x=2
```

从结果我们可以看出：父进程和子进程并发执行，我们永远无法预测其执行顺序；子进程的地址空间是父进程的副本，因此当第六行的函数`Fork`返回时，两进程中的局部变量`x`均为1；两进程对变量的更改互不影响，所以最终输出的值不同。

绘制进程图（Process Graph）对理解`fork`函数很有帮助，如：

![20220216221102](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220216221102.png)

### 回收子进程

当进程终止时，内核不会立即将其移除。它需要被其父进程回收（Reap），否则将变成僵尸（Zombie）进程。当父进程回收其终止的子进程时，内核会将子进程的退出状态传递给父进程，然后再丢弃它。

如果父进程终止，内核需要安排`init`进程（PID 为 1）“收养”孤儿进程。如果父进程在终止前没有回收僵尸子进程，那么`init`进程会回收它们。

进程通过调用函数`waitpid`等待其子进程终止或停止：

```c
#include <sys/types.h>
#include <sys/wait.h>
pid_t waitpid(pid_t pid, int *statusp, int options);
```

默认情况下（参数`options`为 0 时），函数`waitpid`会暂停调用进程，直至其等待集（Wait Set）中的某个子进程终止。该函数始终返回导致其返回的子进程 PID。此时，终止的子进程已被回收，内核从系统中删除了它的所有痕迹。

若参数`pid_t`大于 0 ，则等待集中只有一个 PID 等于该参数的子进程。若参数`pid_t`等于 -1，则等待集包含调用进程的所有子进程。

我们可以通过修改参数`options`的值来改变函数`waitpid`的行为：

- WNOHANG：如果等待集中的子进程还未终止，则立即返回 0；
- WUNTRACED：暂停调用进程执行，直到等待集中的进程终止或停止（默认情况下仅会对终止的子进程返回）；
- WCONTINUED：暂停调用进程执行，直到等待集中的进程终止或等待集中停止的进程收到 SIGCONT 信号恢复。

若参数`statusp`不为 NULL，那么`waitpid`还会将导致其返回的子进程状态信息编码到`status`中（`*statusp = status`）。wait.h 文件定义了几个用于解释参数`status`的宏：

- WIFEXITED(status)：如果子进程正常终止（比如调用`exit`或返回），则返回 True；
- WEXITSTATUS(status)：如果 WIFEXITED() 返回 True，则返回终止子进程的退出状态；
- WIFSIGNALED(status)：如果子进程由于未捕获的信号而终止，则返回 True；
- WTERMSIG(status)：如果 WIFSIGNALED() 返回 True，则返回导致子进程终止的信号编号；
- WIFSTOPPED(status)：如果导致返回的子进程当前已停止，则返回 True；
- WSTOPSIG(status)：如果 WIFSTOPPED() 返回 True，则返回导致子进程停止的信号编号；
- WIFCONTINUED(status)：如果子进程收到 SIGCONT 信号后恢复，则返回 True。

如果调用进程没有子进程，`waitpid`将返回 -1 并将全局变量`errno`设为 ECHILD。而如果`waitpid`被信号中断，则返回 -1 并将全局变量`errno`设为 EINTR。

函数`wait`是`waitpid`的简化版本， `wait(&status)`等效于`waitpid(-1, &status, 0)`。

```c
#include "csapp.h"
#define N 2

int main()
{
    int status, i;
    pid_t pid;

    /* Parent creates N children */
    for (i = 0; i < N; i++)
        if ((pid = Fork()) == 0) /* child */
            exit(100 + i);

    /* Parent reaps N children in no particular order */
    while ((pid = waitpid(-1, &status, 0)) > 0)
    {
        if (WIFEXITED(status))
            printf("child %d terminated normally with exit status=%d\n", pid, WEXITSTATUS(status));
        else
            printf("child %d terminated abnormally\n", pid);
    }

    /* The only normal termination is if there are no more children */
    if (errno != ECHILD)
        unix_error("waitpid error");

    exit(0);
}
```

如示例程序所示，父进程首先调用`Fork`创建了 N 个退出状态唯一的子进程（`exit(100+i)`）。 随后在 While 循环的测试条件中通过`waitpid`等待其所有的子进程终止，并打印子进程的退出状态。最终所有的子进程均被回收，`waitpid`返回 -1 且将全局变量`errno`设为 ECHILD，函数执行完毕。

在 Linux 系统上运行该程序时，它会产生以下输出：

```c
linux> ./waitpid1
child 22966 terminated normally with exit status=100 
child 22967 terminated normally with exit status=101
```

值得注意的是，父进程回收子进程的顺序是随机的。我们可以对上述程序进行一定 [修改](http://csapp.cs.cmu.edu/2e/ics2/code/ecf/waitpid2.c)，从而使其按子进程的 PID 顺序输出。

### 让进程休眠

函数`sleep`可以让进程暂停执行一段时间：

```c
#include <unistd.h>
unsigned int sleep(unsigned int secs);
```

如果请求的暂停时间已经过去，则函数返回 0，否则将返回剩余的暂停时间。当该进程被信号中断时，后一种情况便会发生。

函数`pause`会使调用进程进入休眠状态，直至收到信号。该函数始终返回 -1:

```c
#include <unistd.h>
int pause(void);
```

### 加载并运行程序

函数`execve`在当前进程的上下文中加载并运行一个新程序：

```c
#include <unistd.h>
int execve(const char *filename, const char *argv[], const char *envp[]);
```

参数`filename`是加载并运行的可执行文件名称，`argv`和`envp`则分别是参数和环境变量列表。函数`execve`通常没有返回值，仅在出现错误时返回 -1。

变量`argv`指向一个以 Null 结尾的指针数组，其中的每个指针都指向一个参数字符串。一般来说，`argv[0]`是可执行目标文件名称。变量`envp`也指向一个以 Null 结尾的指针数组，其中的每个指针都指向一个环境变量字符串，而每个字符串都是一个 name=value 形式的键值对。两者的数据结构如下：

![20220221214923](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220221214923.png)

`execve`加载文件名后，会调用启动代码。 启动代码设置栈并将控制权传递给新程序的`main`函数，其原型为：

```c
int main(int argc, char *argv[], char *envp[]);
```

`main`函数执行时的用户栈结构如下图所示。其三个参数分别保存在不同的寄存器中：参数`argc`给出数组`argv[]`中的非空指针数量，参数`argv`指向数组`argv[]`中第一个元素，而参数`envp`则指向数组`envp[]`中的第一个元素。

![20220221221725](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220221221725.png)

Linux 提供了几个用于操作环境变量数组的函数：

```c
#include <stdlib.h>
char *getenv(const char *name);
int setenv(const char *name, const char *newvalue, int overwrite);
void unsetenv(const char *name);
```

如果数组包含格式为 name=value 的字符串，则函数`getenv`会返回其对应的 value， 函数`unsetenv`会将其删除，而函数`setenv`会将 value 替换为参数`newvalue`（`overwrite`非零时）。如果 name 不存在，则函数`setenv`会将 name=`newvalue` 添加到数组中。

### 使用 fork 和 execve 运行程序

Unix shell 和 Web 服务器等程序大量使用了`fork`和`execve`函数。本书提供了一个简单的 [shell 程序](http://csapp.cs.cmu.edu/2e/ics2/code/ecf/shellex.c)，其缺陷在于没有回收任何后台运行的子进程。我们需要使用下一节介绍的信号来解决这一问题。

## 信号

信号（Signal）是一种高级的异常控制流，它允许进程和内核通知其他进程系统中发生了某些类型的事件。Linux 支持的信号类型多达三十种：

![20220222213150](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220222213150.png)

低级别的硬件异常由内核的异常处理程序处理，通常不会对用户进程可见，而信号则可以将此类异常暴露给用户进程。例如一个进程试图除以 0，内核就会向它发送一个 SIGFPE（编号 8）信号。

### 信号术语

发送信号到目标进程需要完成两个步骤：

1. 发送（传递）信号：内核通过更新目标进程上下文中的某些状态来向目标进程发送信号。发送信号的原因有两种：（1）内核检测到系统事件的发生，如被 0 除错误或子进程终止；（2）进程调用了`kill`函数（将在下一节介绍）。进程可以向自己发送信号；
2. 接收信号：当内核强制目标进程以某种方式对信号做出响应时，它便会接收到信号。该进程可以通过执行用户级别的信号处理程序（Signal Handler）来忽略、终止或捕获信号。

![20220222214754](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220222214754.png)

已发送但还未接收的信号称为待处理信号（Pending Signal）。在任意时间点，相同类型的待处理信号最多只能有一个。这意味着如果一个进程已经有一个类型为 k 的待处理信号，那么后续所有发送给该进程的 k 类型信号都将被丢弃。进程还可以选择性地阻塞（Block）某些信号的接收。

### 发送信号

#### 进程组

每个进程都属于一个进程组（Process Group），它由一个正整数的进程组 ID 所标识。`getpgrp`函数返回当前进程的进程组 ID：

```c
#include <unistd.h>
pid_t getpgrp(void);
```

默认情况下，子进程与其父进程属于同一个进程组。一个进程可以通过`setpgid`函数改变自己或另一个进程的进程组：

```c
#include <unistd.h>
int setpgid(pid_t pid, pid_t pgid);
```

该函数会将进程`pid`的进程组更改为`pgid`。若将参数`pid`或`pgid`设为 0，则相当于使用调用进程的 PID 作为参数。举例来说，如果进程 15213 调用函数 `setpgid(0, 0)`，那么将会创建一个进程组 ID 为 15213 的新进程组，并使该进程加入此组。

#### 使用 /bin/kill 程序发送信号

命令`/bin/kill -9 15213`会将编号为 9 的 SIGKILL 信号发送到进程 15213。而 PID 为负则代表信号将发送到进程组 ID 中的所有进程，因此命令`/bin/kill -9 -15213`会该信号发送到进程组 15213 中的每一个进程。

#### 从键盘发送信号

Unix Shell 使用任务（Job）表示单个命令行（如`ls | sort`）创建的进程，同一时间内只能有一个前台任务和多个后台任务。

![20220222223432](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220222223432.png)

在键盘上键入 Ctrl+C 会使内核向前台进程组中的每个进程发送一个 SIGINT 信号，这将终止前台任务。同样，键入 Ctrl+Z 会使内核向前台进程组中的每个进程发送一个 SIGTSTP 信号，这将停止（挂起）前台任务。

#### 使用 kill 函数发送信号

进程可以通过调用`kill`函数向其他进程（包括其自身）发送信号：

```c
#include <sys/types.h>
#include <signal.h>
int kill(pid_t pid, int sig);
```

如果参数`pid`大于 0，则该函数将编号为`sig`的信号发送给进程`pid`；如果参数`pid`等于 0，则该函数将信号发送给调用进程所在进程组中的所有进程；如果参数`pid`小于 0，则该函数将信号发送给进程`|pid|`所在进程组中的所有进程。

#### 使用 alarm 函数发送信号

进程可以通过调用`alarm`函数向自己发送 SIGALRM 信号：

```c
#include <unistd.h>
unsigned int alarm(unsigned int secs);
```

内核将在`secs`秒内向调用进程发送 SIGALRM 信号。该函数会丢弃任何未处理的警告信号，并返回其本应剩余的秒数。

### 接收信号

当内核将进程 p 从内核态切换到用户态时，它会检查 p 未阻塞且未处理（Pending & ~Blocked）的信号集。通常该集合为空，内核将控制权转移给 p 逻辑控制流中的下一条指令。而如果该集合非空，则内核会选择集合中的某个信号 k 并强制 p 接收它。信号将触发进程完成一些动作（Action），预定义的默认动作有：

- 进程终止；
- 进程终止并转储核心（Dump Core，即将代码和数据内存段的镜像写入磁盘）；
- 进程停止（暂停），直到接收 SIGCONT 信号重新启动；
- 进程忽略该信号。

每种信号的默认动作见图。除 SIGSTOP 和 SIGKILL 信号外，进程可以通过函数`signal`修改信号的默认动作：

```c
#include <signal.h>
typedef void (*sighandler_t)(int);
sighandler_t signal(int signum, sighandler_t handler);
// Returns: pointer to previous handler if OK, SIG_ERR on error (does not set errno)
```

如果参数`handler`为 SIG_IGN，则 `signum`类型的信号将会被忽略；如果参数`handler`为 SIG_DFL，则`signum`类型的信号的动作将恢复为默认；如果参数`handler`为用户定义的信号处理程序地址，则进程接收到`signum`类型的信号后会调用该程序，这种方法称为安装处理程序（Installing Handler）。调用处理程序称为捕获信号（Catching Signal），执行处理程序称为处理信号（Handling Signal）。

如果我们在示例程序运行时按下 Ctrl+C，该进程不会直接终止而是输出一段信息后才终止：

```c
#include "csapp.h"

void handler(int sig) /* SIGINT handler */
{
    printf("Caught SIGINT\n");               
    exit(0);                                 
}                                            

int main() 
{
    /* Install the SIGINT handler */         
    if (signal(SIGINT, handler) == SIG_ERR)  
      unix_error("signal error");          
    
    pause(); /* Wait for the receipt of a signal */ 
    exit(0);
}
```

信号处理程序还可以被其他处理程序中断（$s \ne t$）：

![20220223154049](https://cdn.jsdelivr.net/gh/koktlzz/NoteImg@main/20220223154049.png)

### 阻塞信号

Linux 为阻塞信号提供了显式和隐式的实现机制：

- 隐式：默认情况下，内核会阻塞任何与处理程序当前正在处理的信号类型相同的未处理信号。比如上图 8.31 中，如果信号 t 的类型与 s 相同，则 t 将在处理程序 S 返回前持续挂起；
- 显式：应用程序可以通过调用`sigprocmask`等函数阻塞信号或解除信号的阻塞。

```c
#include <signal.h>
int sigprocmask(int how, const sigset_t *set, sigset_t *oldset);
int sigemptyset(sigset_t *set);
int sigfillset(sigset_t *set);
int sigaddset(sigset_t *set, int signum);
int sigdelset(sigset_t *set, int signum);
// Returns: 0 if OK, −1 on error

int sigismember(const sigset_t *set, int signum);
// Returns: 1 if member, 0 if not, −1 on error
```

`sigprocmask`函数可以改变当前阻塞信号的集合（设为`blocked`），具体行为取决于参数`how`的值：

- SIG_BLOCK：将参数`set`中的信号阻塞（`blocked = blocked | set`）；
- SIG_UNBLOCK：为`set`中的信号解除阻塞（`blocked = blocked & ~set`）；
- SIG_SETMASK：将阻塞信号集合设为`set`（`blocked = set`）。

如果参数`oldset`非空，则先前`blocked`的值会存储在`oldset`中。

函数`sigemptyset`将`set`初始化为空集；`sigfillset`将所有信号加入到`set`中；`sigaddset`将编号为`signum`的信号加入到`set`中；`sigdelset`将编号为`signum`的信号从`set`中删除；如果`signum`信号在`set`中，则函数`sigismember`返回 1，否则返回 0。

示例程序暂时阻塞了 SIGINT 信号的接收：

```c
sigset_t mask, prev_mask;
Sigemptyset(&mask);
Sigaddset(&mask, SIGINT);
/* Block SIGINT and save previous blocked set */
Sigprocmask(SIG_BLOCK, &mask, &prev_mask);

// Code region that will not be interrupted by SIGINT

/* Restore previous blocked set, unblocking SIGINT */
Sigprocmask(SIG_SETMASK, &prev_mask, NULL);
```

### 编写信号处理程序
