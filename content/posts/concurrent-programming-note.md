---
title: "CSAPP 读书笔记：并发编程"
date: 2022-08-25T15:31:42+01:00
draft: false
series: ["CSAPP 读书笔记"]
tags: ["OS", " Concurrent"]
summary: "根据第八章介绍的内容，两个在时间上重叠的逻辑控制流是并发的。硬件异常处理程序、进程和 Linux 信号处理程序等都是计算机系统在不同层级上对并发的应用。现代操作系统为构建并发程序提供了三种基本方法 ..."
---

根据 [第八章](/posts/exception-control-flow-note/#并发流) 介绍的内容，两个在时间上重叠的逻辑控制流是并发的。硬件异常处理程序、进程和 Linux 信号处理程序等都是计算机系统在不同层级上对并发的应用。现代操作系统为构建并发程序提供了三种基本方法：

- 进程
- I/O 多路复用（Multiplexing）
- 线程（Thread）

## 使用进程实现并发

构建并发程序最简单的方法是使用进程，如在父进程中接受客户端请求，然后创建子进程为客户端提供服务。

假设服务器在监听描述符`listenfd(3)`上接受来自客户端 1 的连接请求，并返回一个连接描述符`connfd(4)`：

![20220829163846](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220829163846.png)

服务器将调用`fork`创建一个子进程（下图中的“Child 1”），它获得服务器 [描述符表](/posts/system-level-io-note/#共享文件) 的完整副本。由于子进程不再需要监听描述符，而父进程不再需要连接描述符，我们应当将它们关闭：

![20220829164820](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220829164820.png)

随后服务器接受来自客户端 2 的连接请求并返回一个新的连接描述符`connfd(5)`：

![20220829222214](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220829222214.png)

服务器再次调用`fork`创建另一个子进程（下图中的“Child 2”）。此时，父进程正在等待下一个连接请求，两个子进程则并发地为各自的客户端提供服务：

![20220829222709](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220829222709.png)

### 基于进程的并发服务器

一个基于进程的并发服务器代码如下，其中第 32 行调用的`echo`函数来自于上一章介绍的 [echo.c](http://csapp.cs.cmu.edu/2e/ics2/code/netp/echo.c)：

```c
#include "csapp.h"
void echo(int connfd);

void sigchld_handler(int sig)
{
    while (waitpid(-1, 0, WNOHANG) > 0)
        ;
    return;
}

int main(int argc, char **argv)
{
    int listenfd, connfd;
    socklen_t clientlen;
    struct sockaddr_storage clientaddr;

    if (argc != 2)
    {
        fprintf(stderr, "usage: %s <port>\n", argv[0]);
        exit(0);
    }

    Signal(SIGCHLD, sigchld_handler);
    listenfd = Open_listenfd(argv[1]);
    while (1)
    {
        clientlen = sizeof(struct sockaddr_storage);
        connfd = Accept(listenfd, (SA *)&clientaddr, &clientlen);
        if (Fork() == 0)
        {
            Close(listenfd); /* Child closes its listening socket */
            echo(connfd);    /* Child services client */
            Close(connfd);   /* Child closes connection with client */
            exit(0);         /* Child exits */
        }
        Close(connfd); /* Parent closes connected socket (important!) */
    }
}
```

考虑到服务器将运行很长时间，我们需要安装一个 SIGCHLD 信号处理程序来回收子进程（第 4～9 行），详见 [正确的信号处理](/posts/exception-control-flow-note/#正确的信号处理)。

父进程必须关闭`connfd`（第 36 行），否则连接描述符指向的 [打开文件表条目](/posts/system-level-io-note/#共享文件) 永远不会被释放，从而导致内存泄漏。子进程则不需要关闭`connfd`（第 33 行可以省略），因为它会在子进程退出时由内核自动关闭。

### 进程的优缺点

父子进程共享打开文件表，但并不共享用户地址空间（虚拟内存），因此进程之间必须显式地使用 IPC（Interprocess Communications）机制来共享状态信息。由于进程控制和 IPC 的开销很高，基于进程的并发程序往往很慢。

## 使用 I/O 多路复用实现并发

I/O 多路复用技术的基本思想是应用程序调用`select`函数监视多个文件描述符，等待一个或多个描述符准备好用于某种 I/O 操作。该函数十分复杂并有多种使用场景，这里我们只讨论 I/O 操作为读取的情况：

```c
#include <sys/select.h>
int select(int n, fd_set *fdset, NULL, NULL, NULL);
// Returns: nonzero count of ready descriptors, −1 on error

// Macros for manipulating descriptor sets
FD_ZERO(fd_set *fdset);          /* Clear all bits in fdset */
FD_CLR(int fd, fd_set *fdset);   /* Clear bit fd in fdset */
FD_SET(int fd, fd_set *fdset);   /* Turn on bit fd in fdset */
FD_ISSET(int fd, fd_set *fdset); /* Is bit fd in fdset on? */
```

参数`fd_set`是一个描述符集，它在逻辑上是一个位向量（固定长度的 0，1 序列）：

$$b_{n - 1},..., b_1, b_0$$

其中的每个位 $b_k$ 都对应了一个描述符 $k$。当且仅当 $b_k$ 等于 1 时，描述符 $k$ 属于该描述符集。

在我们的应用场景中，参数`fd_set`是读取描述符集（Read Set），参数`n`是读取集的基数（Cardinality）。程序调用`select`函数后会一直阻塞，直到读取集中至少有一个描述符准备好被读取（即从该描述符中读取一个字节的请求不会阻塞）。

我们将读取集中已准备好被读取的描述符集合称为就绪集（Ready Set）。`select`函数会将`fdset`修改为就绪集，并返回就绪集的基数。因此，我们在每次调用该函数前都应当先更新读取集。

### 基于 I/O 多路复用的并发服务器

一个基于 I/O 多路复用的并发服务器代码如下：

```c
#include "csapp.h"

typedef struct
{                                /* represents a pool of connected descriptors */
    int maxfd;                   /* largest descriptor in read_set */
    fd_set read_set;             /* set of all active descriptors */
    fd_set ready_set;            /* subset of descriptors ready for reading  */
    int nready;                  /* number of ready descriptors from select */
    int maxi;                    /* highwater index into client array */
    int clientfd[FD_SETSIZE];    /* set of active descriptors */
    rio_t clientrio[FD_SETSIZE]; /* set of active read buffers */
} pool;

int byte_cnt = 0; /* counts total bytes received by server */

int main(int argc, char **argv)
{
    int listenfd, connfd;
    socklen_t clientlen;
    struct sockaddr_storage clientaddr;
    static pool pool;

    if (argc != 2)
    {
        fprintf(stderr, "usage: %s <port>\n", argv[0]);
        exit(0);
    }
    listenfd = Open_listenfd(argv[1]);
    init_pool(listenfd, &pool);

    while (1)
    {
        /* Wait for listening/connected descriptor(s) to become ready */
        pool.ready_set = pool.read_set;
        pool.nready = Select(pool.maxfd + 1, &pool.ready_set, NULL, NULL, NULL);

        /* If listening descriptor ready, add new client to pool */
        if (FD_ISSET(listenfd, &pool.ready_set))
        {
            clientlen = sizeof(struct sockaddr_storage);
            connfd = Accept(listenfd, (SA *)&clientaddr, &clientlen);
            add_client(connfd, &pool);
        }

        /* Echo a text line from each ready connected descriptor */
        check_clients(&pool);
    }
}
```

该程序使用`pool`类型的结构体（第 3～12 行）保存活跃的客户端，并使用`init_pool`函数初始化客户端池（第 29 行）。在无限循环的每次迭代中，服务器调用`select`函数检测两种不同类型的输入事件：来自新客户端的连接请求；为现有客户端提供服务的连接描述符已准备好被读取。当连接请求到达时（第 38 行），服务器打开连接（第 41 行）并将新客户端加入到客户端池中（第 42 行）。最后，服务器调用`check_clients`函数向每个已连接的描述符写入文本行（第 46 行）。

```c
void init_pool(int listenfd, pool *p)
{
    /* Initially, there are no connected descriptors */
    int i;
    p->maxi = -1;
    for (i = 0; i < FD_SETSIZE; i++)
        p->clientfd[i] = -1;

    /* Initially, listenfd is only member of select read set */
    p->maxfd = listenfd;
    FD_ZERO(&p->read_set);
    FD_SET(listenfd, &p->read_set);
}
```

`init_pool`函数初始化客户端池。`p->clientfd`数组包含了所有已连接的描述符，一开始我们使用 -1 填充它（第 5～7 行）。此时`listenfd`是读取集`p->read_set`中唯一的描述符（第 10～12 行）。

```c
void add_client(int connfd, pool *p)
{
    int i;
    p->nready--;
    for (i = 0; i < FD_SETSIZE; i++) /* Find an available slot */
        if (p->clientfd[i] < 0)
        {
            /* Add connected descriptor to the pool */
            p->clientfd[i] = connfd;
            Rio_readinitb(&p->clientrio[i], connfd);

            /* Add the descriptor to descriptor set */
            FD_SET(connfd, &p->read_set);

            /* Update max descriptor and pool highwater mark */
            if (connfd > p->maxfd)
                p->maxfd = connfd;
            if (i > p->maxi)
                p->maxi = i;
            break;
        }
    if (i == FD_SETSIZE) /* Couldn't find an empty slot */
        app_error("add_client error: Too many clients");
}
```

`add_client`函数将一个新客户端添加到活跃客户端池中。如果`p->clientfd`数组中还有空位（第 6 行），该函数就将连接描述符`connfd`添加到该数组并初始化一个读取缓冲区以调用 [`rio_readlineb`](/posts/system-level-io-note/#有缓冲的输入函数)（第 9～10 行）。随后函数将连接描述符`connfd`添加到读取集（第 13 行），并更新客户端池的一些属性：变量`maxfd`代表`select`函数监视的最大文件描述符；变量`maxi`代表`p->clientfd`数组的最大索引（这样`check_clients`函数就不需要遍历整个数组）。

```c
void check_clients(pool *p)
{
    int i, connfd, n;
    char buf[MAXLINE];
    rio_t rio;

    for (i = 0; (i <= p->maxi) && (p->nready > 0); i++)
    {
        connfd = p->clientfd[i];
        rio = p->clientrio[i];

        /* If the descriptor is ready, echo a text line from it */
        if ((connfd > 0) && (FD_ISSET(connfd, &p->ready_set)))
        {
            p->nready--;
            if ((n = Rio_readlineb(&rio, buf, MAXLINE)) != 0)
            {
                byte_cnt += n;
                printf("Server received %d (%d total) bytes on fd %d\n",
                       n, byte_cnt, connfd);
                Rio_writen(connfd, buf, n);
            }

            /* EOF detected, remove descriptor from pool */
            else
            {
                Close(connfd);
                FD_CLR(connfd, &p->read_set);
                p->clientfd[i] = -1;
            }
        }
    }
}
```

`check_clients`函数遍历客户端池中所有已就绪的连接描述符，如果从描述符中读取文本行成功（第 16 行），就将该行返回给客户端（第 18-21 行）。一旦客户端关闭连接且服务器检测到 EOF，服务器便关闭连接描述符（第 27 行）并将该描述符从读取集和客户端池中删除（第 28-29 行）。

I/O 多路复用的本质是将逻辑控制流建模为状态机（State Machines）：

![20220831234213](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220831234213.png)

如上图所示，基于 I/O 多路复用的并发服务器为每个新客户端 $k$ 创建一个状态机 $s_k$ 并将其与连接描述符 $d_k$ 关联。每个状态机都有一个状态（等待描述符 $d_k$ 准备好被读取），一个输入事件（描述符 $d_k$ 已准备好被读取）和一个状态转换（从描述符 $d_k$ 中读取文本行）。

在示例的并发服务器中，`select`函数检测输入事件，`add_client`函数创建新的状态机（逻辑控制流）。`check_clients`函数通过读写文本行来执行状态转换，并在客户端发送完文本行后删除状态机。

### I/O 多路复用的优缺点

基于 I/O 多路复用的应用程序运行在单个进程的上下文中，因此每个逻辑控制流都可以访问整个进程的地址空间，这使得控制流之间共享数据变得非常容易。由于它不需要通过上下文切换管理新进程，程序的运行效率较高。像 Node.js、Nginx 和 Tornado 等现代高性能服务器均使用 I/O 多路复用实现。

I/O 多路复用的缺点是编码十分复杂，并且无法充分利用多核处理器。

## 使用线程实现并发

线程是在进程上下文中运行的逻辑控制流，它由内核自动调度。每个线程都有自己的线程上下文，包括一个唯一的线程 ID（TID）、栈、栈指针、程序计数器、通用寄存器和条件码寄存器。在同一个进程内运行的所有线程共享该进程全部的虚拟地址空间。

### 线程执行模型

线程的执行模型与进程类似：

![20220907221941](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220907221941.png)

如上图所示，主线程（Main Thread）是进程生命周期的开始，它在某一时刻创建了一个对等线程（Peer Thread）。两线程同时运行，控制权通过上下文切换传递。

线程执行与进程的区别在于：

- 线程上下文比进程上下文小得多，因此线程上下文切换比进程快；
- 属于同一进程的线程构成了一个对等池，它们没有父子层级结构。线程可以杀死任何对等线程或等待任何对等线程终止；
- 每个对等线程都可以读写相同的共享数据。

### Posix 线程

Posix 线程（Pthread）是 C 程序操作线程的标准接口。它定义了大约六十个函数，允许程序创建线程、终止线程、回收线程、与对等线程安全地共享数据以及通知对等线程系统状态的变化等。

```c
#include "csapp/csapp.h"
void *thread(void *vargp) /* thread routine */
{
  printf("Hello, world!\n");
  return NULL;
}

int main() {
  pthread_t tid;
  Pthread_create(&tid, NULL, thread, NULL);
  Pthread_join(tid, NULL);
  exit(0);
}
```

示例程序中，主线程创建了一个对等线程并等待它终止，对等线程在打印`Hello, world!\n`后返回。

线程的代码和局部数据封装在线程例程（Thread Routine）中，它将通用指针`void *`作为输入并返回另一个通用指针（第 2 行）。如果需要向线程例程传递多个参数，则应将它们放入一个结构体中并传递指向该结构体的指针。同样，如果想让线程例程返回多个参数，则应返回一个指向包含多个参数的结构体指针。

### 创建线程

线程调用函数`pthread_create`创建新线程：

```c
#include <pthread.h>
typedef void *(func)(void *);
int pthread_create(pthread_t *tid, pthread_attr_t *attr,
                   func *f, void *arg);
// Returns: 0 if OK, nonzero on error
```

参数`f`是新线程在其上下文中运行的例程，参数`arg`是该例程的输入参数。参数`attr`可用于更改新线程的默认属性，一般设为`NULL`。该函数返回时，参数`tid`将变为新线程的线程 ID。线程还可以调用`pthread_self`函数确认自己的线程 ID：

```c
#include <pthread.h>
pthread_t pthread_self(void);
//Returns: thread ID of caller
```

### 终止线程

线程终止的方式包括：

- 在其例程返回时隐式终止；
- 调用函数`pthread_exit`显式终止，参数`thread_return`用于函数 [`pthread_join`](/posts/concurrent-programming-note/#回收线程)：

```c
#include <pthread.h>
void pthread_exit(void *thread_return);
// Never returns
```

- 调用 Linux 函数`exit`终止进程以及与该进程关联的所有线程；
- 调用函数`pthread_cancel`终止另一个线程 ID 为`tid`的对等线程：

```c
#include <pthread.h>
int pthread_cancel(pthread_t tid);
// Returns: 0 if OK, nonzero on error
```

### 回收线程

线程调用`pthread_join`函数等待另一个线程`tid`终止：

```c
#include <pthread.h>
int pthread_join(pthread_t tid, void **thread_return);
// Returns: 0 if OK, nonzero on error
```

线程`tid`终止后，该函数将线程例程返回的通用指针分配到`thread_return`指向的位置，然后回收终止线程持有的所有内存资源。与  [`waitpid`](/posts/exception-control-flow-note/#回收子进程) 不同，`pthread_join`只能等待某个特定线程终止。

### 分离线程

在任意时刻，线程都是可连接的（Joinable）或分离的（Detached）。一个可连接的线程可以被其他线程回收或杀死，其内存资源（如栈）在它被另一个线程回收之前不会释放；相反，一个分离的线程无法被其他线程回收或杀死，其内存资源将在它终止时由系统自动释放。

默认情况下，线程是可连接的。为了避免内存泄漏，每个可连接的线程都应当被另一个线程显式地回收，或者调用`pthread_detach`函数成为一个分离的线程：

```c
#include <pthread.h>
int pthread_detach(pthread_t tid);
// Returns: 0 if OK, nonzero on error
```

参数`tid`是被分离的线程 ID，线程可以将其设为`pthread_self()`来分离自己。

### 初始化线程

线程调用`pthread_once`函数初始化与线程例程关联的状态：

```c
#include <pthread.h>
pthread_once_t once_control = PTHREAD_ONCE_INIT;
int pthread_once(pthread_once_t *once_control,
                 void (*init_routine)(void));
// Always returns 0
```

参数`once_control`是一个全局变量或静态变量，它始终被初始化为`PTHREAD_ONCE_INIT`。该函数第一次被调用时会直接调用`init_routine`，随后使用相同`once_control`参数的调用不会起任何作用。如下示例程序将输出 1：

```c
#include <pthread.h>
#include <stdio.h>
pthread_once_t once_control = PTHREAD_ONCE_INIT;
int cnt;

void init_routine(void)
{
    cnt++;
}

int main()
{
    pthread_once(&once_control, *init_routine);
    pthread_once(&once_control, *init_routine);
    printf("%d", cnt);
}
```

当我们需要动态初始化多个线程共享的全局变量时，该函数十分有用。

### 基于线程的并发服务器

一个基于线程的并发服务器代码如下：

```c
#include "csapp.h"

void echo(int connfd);
/* thread routine */
void *thread(void *vargp)
{
    int connfd = *((int *)vargp);
    Pthread_detach(pthread_self());
    Free(vargp);
    echo(connfd);
    Close(connfd);
    return NULL;
}

int main(int argc, char **argv)
{
    int listenfd, *connfdp;
    socklen_t clientlen;
    struct sockaddr_storage clientaddr;
    pthread_t tid;

    if (argc != 2)
    {
        fprintf(stderr, "usage: %s <port>\n", argv[0]);
        exit(0);
    }
    listenfd = Open_listenfd(argv[1]);

    while (1)
    {
        clientlen = sizeof(struct sockaddr_storage);
        connfdp = Malloc(sizeof(int));
        *connfdp = Accept(listenfd, (SA *)&clientaddr, &clientlen);
        Pthread_create(&tid, NULL, thread, connfdp);
    }
}
```

程序的整体结构与基于进程的并发服务器类似，主线程反复等待客户端连接，然后创建对等线程处理请求。值得注意的是，该程序调用`Malloc`函数生成指向连接描述符的指针`connfdp`并将其传递给对等线程（第 32～34 行）。这是因为如果我们直接传递指针，如：

```c
connfd = Accept(listenfd, (SA *) &clientaddr, &clientlen);
Pthread_create(&tid, NULL, thread, &connfd);

void *thread(void *vargp) {
    int connfd = *((int *)vargp);
    . . .
}
```

就会导致对等线程中的赋值语句与主线程中的`Accpet`调用产生竞争：若新客户端在赋值语句执行完毕前与服务器建立连接，则对等线程中的局部变量`connfd`会变成新客户端的连接描述符。由于`Malloc`可以将`connfd`动态分配到不同的堆内存 Block 中，这一问题便得到了解决。

为了避免内存泄漏，我们必须分离每个线程（第 8 行）并释放主线程分配的堆内存（第 9 行）。进程中的所有线程共享描述符表，连接描述符的`rfcnt`始终为 1。因此我们只需在对等线程中关闭连接描述符，而不必像基于进程的并发服务器那样在主线程进行同样的操作。

## 线程程序中的共享变量

在程序员看来，线程的最大优势在于多个线程可以轻松地共享相同的程序变量。然而，这种便利可能会带来一些问题。为了正确地编写线程程序，我们以如下程序为例说明共享变量的含义及工作原理：

```c
#include "csapp.h"
#define N 2
void *thread(void *vargp);

char **ptr; /* global variable */

int main()
{
    int i;
    pthread_t tid;
    char *msgs[N] = {
        "Hello from foo",
        "Hello from bar"};

    ptr = msgs;
    for (i = 0; i < N; i++)
        Pthread_create(&tid, NULL, thread, (void *)i);
    Pthread_exit(NULL);
}

void *thread(void *vargp)
{
    int myid = (int)vargp;
    static int cnt = 0;                                    
    printf("[%d]: %s (cnt=%d)\n", myid, ptr[myid], ++cnt); 
    return NULL;
}
```

### 线程内存模型

每个线程都有自己独立的线程上下文，因此多个线程之间共享同一进程上下文中的其余部分。它们包括：只读文本（代码）段、可读写数据段、堆、共享库和打开文件描述符等。

线程无法读取或写入另一个线程的寄存器，但任何线程都能够访问共享虚拟内存中的任何位置。尽管栈属于线程上下文的一部分，但线程可以使用指针对另一个线程的栈内容读取和写入。示例程序第 25 行，对等线程通过全局变量`ptr`间接引用了主线程栈中的数组`msgs[N]`。

### 将变量映射到内存

C 线程程序根据变量的存储类型将其映射到虚拟内存：

- 全局变量：全局变量的唯一实例在运行时位于 [可读写段](/posts/linking-note/#加载可执行目标文件)，它可以被任意线程引用。示例程序第 5 行声明的全局变量`ptr`便是如此；
- 局部自动变量：局部自动变量在运行时位于每个线程的栈中，如示例程序中的变量`tid`和`myid`。为了区分不同线程中的相同变量，我们将它们分别表示为`tid.m`、`myid.p0`和`myid.p1`；
- 局部静态变量：与全局变量一样，局部静态变量的唯一实例在运行时位于可读写段。即使示例程序中的每个对等线程都声明了局部静态变量`cnt`（第 24 行），在运行时虚拟内存中也只有一个`cnt`实例。

### 共享变量

当一个变量的实例被多个线程引用时，我们就称它为共享的。在示例程序中，变量`cnt`是共享的，而`myid`则不是。对等线程均通过`ptr`间接引用了局部变量`msgs`，因此`msgs`也是共享的。

## 使用信号量同步线程

以下程序`badcnt.c`创建了两个对等线程，每个线程都会将全局共享变量`cnt`递增`niters`次：

```c
#include "csapp.h"

void *thread(void *vargp); /* Thread routine prototype */

/* Global shared variable */
volatile long cnt = 0; /* Counter */

int main(int argc, char **argv)
{
    long niters;
    pthread_t tid1, tid2;

    /* Check input argument */
    if (argc != 2)
    {
        printf("usage: %s <niters>\n", argv[0]);
        exit(0);
    }
    niters = atoi(argv[1]);

    /* Create threads and wait for them to finish */
    Pthread_create(&tid1, NULL, thread, &niters);
    Pthread_create(&tid2, NULL, thread, &niters);
    Pthread_join(tid1, NULL);
    Pthread_join(tid2, NULL);

    /* Check result */
    if (cnt != (2 * niters))
        printf("BOOM! cnt=%d\n", cnt);
    else
        printf("OK cnt=%d\n", cnt);
    exit(0);
}

/* Thread routine */
void *thread(void *vargp)
{
    long i, niters = *((long *)vargp);

    for (i = 0; i < niters; i++)
        cnt++;
    return NULL;
}
```

理论上，该程序的输出结果应为`2 * niters`。然而当它在 Linux 系统上运行时，我们不但会得到错误的答案，并且每次的结果还不同：

```shell
linux> ./badcnt 1000000 BOOM! cnt=1445085
linux> ./badcnt 1000000 BOOM! cnt=1915220
linux> ./badcnt 1000000 BOOM! cnt=1404746
```

为了清楚地理解这一问题，我们需要研究一下计数器循环（第 40～41 行）的汇编代码：

```nasm
; i in %rax, niters in %rcx, cnt in %rdx
    movq  (%rdi), %rcx
    testq %rcx, %rcx
    jle   .L2
    movl  $0, %eax
.L3:
    movq  cnt(%rip), %rdx
    addq  $1, %rdx
    movq  movq %rdx, cnt(%rip)
    addq  $1, %rax
    cmpq  %rcx, %rax
    jne   .L3
.L2:
```

这段代码可以分为以下五个部分：

- $H_i$：循环头部的指令块（第 2～5 行）；
- $L_i$：将变量`cnt`加载到寄存器 %$rdx_i$（第 7 行）；
- $U_i$：将 %$rdx_i$ 加一（第 8 行）；
- $S_i$：将 %$rdx_i$ 更新后的值存回变量`cnt`（第 9 行）；
- $T_i$：循环尾部的指令块（第 10～13 行）。

上述指令在单核处理器上以某种顺序依次执行，不同的执行顺序将导致不同的结果。我们以第一次循环为例：

![20220912235213](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220912235213.png)

如图（b）所示，线程 2 在第五步将变量`cnt`加载到 %$rdx_2$。此时线程 1 已经在第三步更新了 %$rdx_1$ 的值，但还未把它存回`cnt`。因此 %$rdx_2$ 的初始值为 0，线程 2 无法像图（a）那样将`cnt`从 1 递增到 2。

### 进度图

进度图（Progress Graph）将 n 个并发线程建模为 n 维笛卡尔空间中的轨迹（Trajectory）。其中，每个坐标轴对应线程 $k$ 的进度，每个点代表线程 $k$ 已完成指令 $I_k$ 的状态。程序`badcnt.c`第一次循环的进度图如下，点 $(L_1, S_2)$ 代表线程 1 已完成 $L_1$，线程 2 则已完成 $S_2$：

![20220913221845](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220913221845.png)

程序的执行历史可以用进度图中的轨迹表示。假设该程序第一次循环的指令执行顺序为：

$$H_1, L_1, U_1, H_2, L_2, S_1, T_1, U_2, S_2, T_2$$

则进度图轨迹为：

![20220913224057](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220913224057.png)

对于线程 $i$，操作共享变量`cnt`的指令 $(L_i, U_i, S_i)$ 构成了一个临界区（Critical Section），它不应当与其他线程的临界区相交。在进度图上，两线程临界区的交集构成了不安全区（Unsafe Region）：

![20220913224916](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220913224916.png)

不安全区不包括其边缘，例如状态 $(H_1, H_2)$ 和 $(S_1, U_2)$ 均不属于该区域。绕过不安全区的轨迹被称为安全轨迹，而触及了不安全区中任何部分的轨迹都是不安全的。

### 信号量

信号量（Semaphore）`s`是一个具有非负整数值的全局变量，我们只能对它进行两种操作：

- `P(s)`：若`s`非零，则将其减一并立即返回；若`s`为零，则将线程暂停。当`s`变为非零且线程由`V`操作重启后，`P`再将`s`减一并把控制权返回给调用者；
- `V(s)`：将`s`加一。如果存在任何被`P`操作阻塞的线程，就随机重启它们中的一个。

`P(s)`和`V(s)`操作不可分割（具有原子性），因此它们不会被中断。其定义保证了正确初始化的信号量永远不会变为负值，我们将这种属性称为信号量的不变性（Semaphore Invariant）。

Posix 标准定义了多种操作信号量的函数：

```c
#include <semaphore.h>
int sem_init(sem_t *sem, int pshared, unsigned int value);
int sem_wait(sem_t *s);   /* P(s) */
int sem_post(sem_t *s);   /* V(s) */
// Returns: 0 if OK, −1 on error
```

每个信号量都必须在使用前初始化，`sem_init`函数则将信号量`sem`初始化为参数`value`。在我们的应用场景中，参数`pshared`始终为 0。线程分别调用`sem_wait`和`sem_post`函数来执行`P(s)`和`V(s)`操作。为了简洁起见，我们使用以下等效的包装函数代替：

```c
#include "csapp.h"
void P(sem_t *s);   /* Wrapper function for sem_wait */
void V(sem_t *s);   /* Wrapper function for sem_post */
// Returns: nothing
```

### 使用信号量实现互斥

信号量提供了一种便捷的方法来确保线程对共享变量的访问互斥（Mutually Exclusive ）：将一个初始值为 1 的信号量与每个共享变量相关联，然后使用`P`和`V`操作包围临界区。

在这种情况下，信号量的值始终为 0 或 1，因此我们将它称为二进制信号量。用于实现互斥的二进制信号量通常被称为互斥锁（Mutex），对其进行`P`和`V`操作则分别被称为加锁和解锁。一个已被锁定但互斥锁还未被解锁的线程被称为持有互斥锁。

下图展示了我们如何使用二进制信号量正确同步程序`badcnt.c`，其中的每个状态都标注了该状态下信号量的值：

![20220914000230](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220914000230.png)

图中信号量为 -1 的状态共同构成了禁止区（Forbidden Region）。由于信号量的不变性，任何可行的轨迹都无法进入该区域。禁止区完全包围了不安全区，因此轨迹也不会触及不安全区的任何部分。无论运行时指令执行的顺序如何，程序都会正确地递增`cnt`。

综上，为了使用信号量实现互斥，我们需要首先声明一个信号量`mutex`：

```c
volatile long cnt = 0; /* Counter */
sem_t mutex;           /* Semaphore that protects counter */
```

然后在主线程中将其初始化：

```c
Sem_init(&mutex, 0, 1); /* mutex = 1 */
```

最终在对等线程中调用`P`和`V`包围对共享变量`cnt`的更新操作：

```c
for (i = 0; i < niters; i++) {
    P(&mutex);
    cnt++;
    V(&mutex); 
}
```

### 使用信号量调度共享资源

除了实现互斥之外，信号量的另一个重要作用是调度对共享资源的访问。在这种情况下，线程使用信号量与其他线程通信。让我们来看看两个经典案例：生产者-消费者（Producer-Consumer）问题和读取者-写入者（Readers-Writers）问题。

#### 生产者-消费者问题

![20220914223735](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220914223735.png)

如上图所示，生产者和消费者线程共享一个有界缓冲区。生产者线程重复生成新项目并将它们插入缓冲区，消费者线程则不断从缓冲区中删除项目，然后消费（使用）它们。

由于插入和删除项目涉及到更新共享变量，我们必须保证线程对缓冲区的访问是互斥的。但仅仅保证互斥是不够的，我们还需要调度线程对缓冲区的访问：若缓冲区已满（没有空位），则生产者必须等待空位；若缓冲区为空（没有可用的项目），则消费者必须等待项目可用。

我们开发了一个名为 $S_{buf}$ 的简单包，它可以操作`sbuf_t`类型的缓冲区：

```c
typedef struct {
    int *buf;          /* Buffer array */         
    int n;             /* Maximum number of slots */
    int front;         /* buf[(front+1)%n] is first item */
    int rear;          /* buf[rear%n] is last item */
    sem_t mutex;       /* Protects accesses to buf */
    sem_t slots;       /* Counts available slots */
    sem_t items;       /* Counts available items */
} sbuf_t;
```

所有项目均存储在一个动态分配且包含 `n` 个空位的整型数组`buf`中，`front`和`rear`用于记录数组中的第一个项目和最后一个项目。信号量`mutex`实现了对缓冲区访问的互斥，`slots`和`items`则分别计算缓冲区中的空位和可用项目的数量。

$S_{buf}$ 包的实现如下：

```c
#include "csapp.h"
#include "sbuf.h"

/* Create an empty, bounded, shared FIFO buffer with n slots */
void sbuf_init(sbuf_t *sp, int n)
{
    sp->buf = Calloc(n, sizeof(int)); 
    sp->n = n;                       /* Buffer holds max of n items */
    sp->front = sp->rear = 0;        /* Empty buffer iff front == rear */
    Sem_init(&sp->mutex, 0, 1);      /* Binary semaphore for locking */
    Sem_init(&sp->slots, 0, n);      /* Initially, buf has n empty slots */
    Sem_init(&sp->items, 0, 0);      /* Initially, buf has zero data items */
}

/* Clean up buffer sp */
void sbuf_deinit(sbuf_t *sp)
{
    Free(sp->buf);
}

/* Insert item onto the rear of shared buffer sp */
void sbuf_insert(sbuf_t *sp, int item)
{
    P(&sp->slots);                          /* Wait for available slot */
    P(&sp->mutex);                          /* Lock the buffer */
    sp->buf[(++sp->rear)%(sp->n)] = item;   /* Insert the item */
    V(&sp->mutex);                          /* Unlock the buffer */
    V(&sp->items);                          /* Announce available item */
}

/* Remove and return the first item from buffer sp */
int sbuf_remove(sbuf_t *sp)
{
    int item;
    P(&sp->items);                          /* Wait for available item */
    P(&sp->mutex);                          /* Lock the buffer */
    item = sp->buf[(++sp->front)%(sp->n)];  /* Remove the item */
    V(&sp->mutex);                          /* Unlock the buffer */
    V(&sp->slots);                          /* Announce available slot */
    return item;
}
```

`sbuf_init`函数为缓冲区分配堆内存，将`front`和`rear`设为 0，并为三个信号量分配初始值；`sbuf_deinit`函数在程序使用完缓冲区后释放它；`sbuf_insert`函数等待一个可用的空位，然后锁定`mutex`，将项目添加到缓冲区尾部并解锁`mutex`，最后通知消费者线程新项目可用；`sbuf_remove`函数与之对称：当缓冲区中有项目可用时，它锁定`mutex`，然后移除缓冲区头部的项目并解锁`mutex`，最后通知生产者有新的空位。

#### 读取者-写入者问题

读取者-写入者问题是互斥问题的一般化。假设并发线程集合正在访问一个共享对象，如主存中的数据结构或磁盘上的数据库。写入者必须拥有对该对象的独占访问权，但读取者却可以与其他读取者共享。

我们可以根据读取者和写入者的优先级将这一问题分为两种情况：

- 读取者优先：读取者不会因写入者在等待而等待；
- 写入者优先：一旦写入者准备好写入就尽快执行写入。

示例程序展示了一个读取者优先的解决方案：

```c
/* Global variables */
int readcnt;    /* Initially = 0 */
sem_t mutex, w; /* Both initially = 1 */

void reader(void)
{
    while (1)
    {
        P(&mutex);
        readcnt++;
        if (readcnt == 1) /* First in */
            P(&w);
        V(&mutex);

        /* Critical section */
        /* Reading happens  */
        
        P(&mutex);
        readcnt--;
        if (readcnt == 0) /* Last out */
            V(&w);
        V(&mutex);
    }
}

void writer(void)
{
    while (1)
    {
        P(&w);

        /* Critical section */
        /* Writing happens  */

        V(&w);
    }
}
```

信号量`w`实现了对共享对象访问的互斥，`mutex`则保护了对共享变量`readcnt`（表示当前处于临界区的读取者数量）的访问。写入者每次进入临界区时都会锁定`w`并在离开时对其解锁。但只有第一个进入临界区的读取者才需要锁定`w`，只有最后一个离开临界区的读取者才需要解锁它。因此只要有一个读取者持有`w`（位于临界区），其他读取者就都可以畅通无阻地访问共享对象。

所有此类问题的解决方案都会导致饥饿（Starvation），即线程被阻塞并无法取得任何进展。例如在上述程序中，当读取者线程批量到达时，写入者只能无限期等待。

### 基于预线程的并发服务器

前文介绍的 [基于线程的并发服务器](/posts/concurrent-programming-note/#基于线程的并发服务器) 需要为每个客户端创建一个新线程，因此其成本较高。基于预线程（Prethreading）的并发服务器可以通过生产者-消费者模型减少这一开销：

![20220915001107](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220915001107.png)

如上图所示，主线程不断地接受来自客户端连接请求并在有界缓冲区中插入连接描述符。每个工作线程重复地从缓冲区中移除一个描述符并为客户端提供服务，然后等待下一个描述符。

我们使用 $S_{buf}$ 包实现这一模型：

```c
#include "csapp.h"
#include "sbuf.h"
#define NTHREADS 4
#define SBUFSIZE 16

void echo_cnt(int connfd);
void *thread(void *vargp);

sbuf_t sbuf; /* shared buffer of connected descriptors */

int main(int argc, char **argv)
{
    int i, listenfd, connfd;
    socklen_t clientlen;
    struct sockaddr_storage clientaddr;
    pthread_t tid;

    if (argc != 2)
    {
        fprintf(stderr, "usage: %s <port>\n", argv[0]);
        exit(0);
    }
    listenfd = Open_listenfd(argv[1]);

    sbuf_init(&sbuf, SBUFSIZE);
    for (i = 0; i < NTHREADS; i++) /* Create worker threads */
        Pthread_create(&tid, NULL, thread, NULL);

    while (1)
    {
        clientlen = sizeof(struct sockaddr_storage);
        connfd = Accept(listenfd, (SA *)&clientaddr, &clientlen);
        sbuf_insert(&sbuf, connfd); /* Insert connfd in buffer */
    }
}

void *thread(void *vargp)
{
    Pthread_detach(pthread_self());
    while (1)
    {
        int connfd = sbuf_remove(&sbuf);/* Remove connfd from buffer */ 
        echo_cnt(connfd);               /* Service client */
        Close(connfd);
    }
}
```

在初始化缓冲区`sbuf`（第 25 行）之后，主线程创建了一组工作线程（第 26～27 行）。它进入无限循环，接受连接请求并将连接描述符插入到`sbuf`中。工作线程等待连接描述符可用后便将其从缓冲区中移除（第 42 行），然后调用`echo_cnt`函数为客户端提供服务。

`echo_cnt`是上文提到的`echo`函数的变体，它将服务器接收到的字节数记录在全局变量`byte_cnt`中：

```c
#include "csapp.h"

static int byte_cnt; /* byte counter */
static sem_t mutex;  /* and the mutex that protects it */

static void init_echo_cnt(void)
{
    Sem_init(&mutex, 0, 1);
    byte_cnt = 0;
}

void echo_cnt(int connfd)
{
    int n;
    char buf[MAXLINE];
    rio_t rio;
    static pthread_once_t once = PTHREAD_ONCE_INIT;

    Pthread_once(&once, init_echo_cnt); 
    Rio_readinitb(&rio, connfd);        
    while ((n = Rio_readlineb(&rio, buf, MAXLINE)) != 0)
    {
        P(&mutex);
        byte_cnt += n; 
        printf("server received %d (%d total) bytes on fd %d\n",
                n, byte_cnt, connfd);
        V(&mutex);
        Rio_writen(connfd, buf, n);
    }
}
```

该函数使用`Pthread_once`（第 19 行）初始化信号量`mutex`和`byte_cnt`，于是我们便不必在主线程进行同样的操作了。这种方法使包更加易于使用，不过同时也增加了许多无用的工作（只有第一次调用`Pthread_once`是有意义的）。

## 使用线程实现并行

到目前为止，我们对并发的研究仅局限于单核处理器。实际上并发程序往往在拥有多核处理器的机器上运行得更快，这是因为操作系统内核可以在多个 CPU 核心上并行调度线程。

![20220915230021](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220915230021.png)

编写并行程序十分棘手，看似很小的代码更改却会对程序性能产生重大的影响。并行程序同步线程的开销非常高，因此我们需要尽量避免它，否则就可能出现线程数增加程序性能反而降低的问题。如果同步操作不可避免，则应当尽可能地增加有用计算以分摊其开销。

由于在同一个核心上切换多个线程的上下文会产生额外的开销，并行程序的线程数通常与机器的 CPU 核心数相同。

### 并行程序的性能指标

加速比（Speedup）被定义为：

$$S_p = \cfrac{T_1}{T_p}$$

其中，$p$ 是处理器核心的数量，$T_p$ 是程序在 $p$ 个核心上运行的时间。这个公式也被称为强缩放（Strong Scaling）。当 $T_1$ 是并行程序的顺序执行版本的运行时间时，$S_p$ 被称为绝对加速比；当 $T_1$ 是并行程序在一个核心上的运行时间时，$S_p$ 被称为相对加速比。绝对加速比比相对加速比更能真实地反映程序的性能，然而它也更难测量。尤其是一些复杂的并行程序，为其创建一个顺序执行的版本非常困难。

效率（Efficiency）被定义为：

$$E_p = \cfrac{S_p}{p} = \cfrac{T_1}{pT_p}$$

该指标能够衡量程序并行化的开销，效率高的程序用于线程同步和通信的时间较少。

## 其他并发问题

### 线程安全

若一个函数被多个并发线程重复调用时总能生成正确的结果，我们就称它为线程安全的（Thread-safe）。反之，则为线程不安全函数。线程不安全函数可以分为以下四类：

- 不保护共享变量的函数：上文提到的 [`badcnt`](/posts/concurrent-programming-note/#使用信号量同步线程) 函数便属于此类。我们只需使用`P`和`V`等同步操作保护共享变量便可使函数线程安全，但同时也会增加程序的运行时间；
- 在多次调用中共享状态信息的函数：如伪随机数生成函数`rand`，其当前调用的结果取决于上次迭代的中间结果。因此如果多个线程调用该函数，我们就无法确定返回的随机数序列。修改此类函数唯一的方法便是重写它们，使其不依赖任何`static`数据并让调用者通过参数来传递状态信息；

```c
unsigned int next_seed = 1;

/* rand - return pseudo-random integer on 0..32767 */
int rand(void)
{
    next_seed = next_seed*1103515245 + 12345;
    return (unsigned int)(next_seed/65536) % 32768;
}

/* srand - set seed for rand() */
void srand(unsigned int new_seed)
{
    next_seed = new_seed;
} 
```

- 返回指向`static`变量指针的函数：一些函数，如`ctime`和`gethostbyname`，在`static`变量中计算结果并返回指向该变量的指针。如果并发线程调用此类函数，一个线程正在使用的结果就有可能被另一个线程覆盖。我们可以直接重写它们，但也可以在源码不可用或难以修改时使用锁定和复制（Lock-and-Copy）技术来解决线程不安全问题；

```c
char *ctime_ts(const time_t *timep, char *privatep)
{
    char *sharedp; 

    P(&mutex);
    sharedp = ctime(timep);
    strcpy(privatep, sharedp); /* Copy string from shared to private */
    V(&mutex);
    return privatep;
}
```

- 调用线程不安全函数的函数：如果函数`f`调用了第二类线程不安全函数`g`，那么`f`也是线程不安全的并且只能重写`g`；如果`g`是第一类或第三类函数，我们就可以使用互斥锁保护调用点和所有生成的共享数据以使`f`线程安全。在上面的例子中，虽然函数`ctime_ts`调用了线程不安全函数`ctime`，但它却是线程安全的。

### 可重入

可重入（Reentrant）函数是一种特殊的线程安全函数，它在被多个线程调用时不会引用任何共享数据。

![20220918212928](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220918212928.png)

可重入函数不需要进行同步操作，因此通常比不可重入函数更高效。将第二类线程不安全函数重写为可重入函数是使其线程安全的唯一方法。我们可以将上节提到的函数`rand`修改为：

```c
/* rand_r - a reentrant pseudo-random integer on 0..32767 */
int rand_r(unsigned int *nextp)
{
    *nextp = *nextp * 1103515245 + 12345;
    return (unsigned int)(*nextp / 65536) % 32768;
}
```

其关键思想在于，`rand_r`使用被调用者传入的指针`nextp`代替了静态变量`next_seed`。

如果函数的所有参数都按值传递且所有的数据引用都指向局部自动变量，那么我们就称该函数是显式可重入的。无论该函数被调用的方式如何，其可重入性都不变；如果函数的某些参数通过引用（指针）传递且调用者线程小心地将非共享数据传递给指针，那么我们就称该函数是隐式可重入的，函数`rand_r`便是如此。

### 竞争

当多个线程的执行顺序会影响程序的正确性时就会发生竞争，如：

```c
#include "csapp.h"
#define N 4

void *thread(void *vargp);

int main() 
{
    pthread_t tid[N];
    int i;

    for (i = 0; i < N; i++) 
        Pthread_create(&tid[i], NULL, thread, &i); 
    for (i = 0; i < N; i++) 
        Pthread_join(tid[i], NULL);
    exit(0);
}

/* thread routine */
void *thread(void *vargp) 
{
    int myid = *((int *)vargp);
    printf("Hello from thread %d\n", myid);
    return NULL;
}
```

示例程序中，主线程创建了四个对等线程并将指向了唯一整数 ID 的指针`&i`传递给它们。对等线程将参数传递的 ID 复制到局部变量（第 21 行），然后打印包含 ID 的消息。该程序看似简单，然而却输出了错误的结果：

```c
linux> ./race
Hello from thread 1 
Hello from thread 3 
Hello from thread 2 
Hello from thread 3
```

出现这一问题的原因在于，主线程循环中变量`i`的自增（第 12 行）与对等线程中对参数的解引用和赋值（第 21 行）之间产生了竞争。如果对等线程在主线程变量`i`自增之后才执行第 22 行的代码，那么变量`myid`就变成了其他线程的 ID。

为了消除竞争，我们需要为每个 ID 动态分配一个堆内存 Block，并向线程传递指向该 Block 的指针。实际上，前文介绍的 [基于线程的并发服务器](/posts/concurrent-programming-note/#基于线程的并发服务器) 便使用了这一方法。

另一种方法是主线程直接向对等线程传递`i`而非其指针：

```c
for (i = 0; i < N; i++) 
    Pthread_create(&tid[i], NULL, thread, (void *) i); 
```

对等线程则将参数转换回`int`类型并赋给变量`myid`：

```c
int myid = (int)vargp;
```

相比于第一种方法，这种方法的好处是减少了调用`malloc`和`free`带来的开销。但在类型转换中，它假设指针至少与整型一样大，可能不适用于某些操作系统。

### 死锁

信号量的引入可能会导致线程被永远阻塞，我们将这种运行时错误称为死锁（Deadlock）。进度图是理解死锁的重要工具：

![20220918230933](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220918230933.png)

上图中的两个线程使用信号量`s`和`t`实现互斥，但程序员错误地对`P`和`V`操作排序。一旦某个轨迹进入了死锁状态`d`，两信号量重叠的禁止区域便阻止了它所有的可行路线。换言之，由于每个阻塞线程等待的`V`操作永远不会被执行，程序发生死锁。

死锁是一个非常棘手的问题，因为它难以预测。程序可能正确地运行了上千次，但下一次便会出现死锁。更糟糕的是，并发程序每次执行的轨迹都有所不同，因此死锁还难以复现。

对于二进制信号量，我们可以使用互斥锁排序规则来防止死锁：若每个线程以相同的顺序加锁（如上图中两线程均先执行`P(s)`，再执行`P(t)`），则程序无死锁。解锁的顺序并不重要，因为`V`操作不会阻塞线程。
