---
title: "CSAPP 读书笔记：并发编程"
date: 2022-08-25T15:31:42+01:00
draft: true
series: ["CSAPP 读书笔记"]
tags: ["OS"]
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

在我们的应用场景中，参数`fd_set`是读取描述符集（Read Set），参数`n`是读取集的基数（Cardinality）。程序调用`select`函数后会一直阻塞，直到读取集中至少有一个描述符准备好被读取（即从该描述符读取一字节的请求不会阻塞）。

该函数还会将参数`fdset`修改为由读取集中已准备好被读取的描述符组成的就绪集（Ready Set），并返回就绪集的基数。因此，我们在每次调用`select`函数前都应当先更新读取集。

示例代码展示了一个使用`select`函数实现的迭代服务器：

```c
#include "csapp.h"
void echo(int connfd);
void command(void);

int main(int argc, char **argv)
{
    int listenfd, connfd;
    socklen_t clientlen;
    struct sockaddr_storage clientaddr;
    fd_set read_set, ready_set;

    if (argc != 2)
    {
        fprintf(stderr, "usage: %s <port>\n", argv[0]);
        exit(0);
    }
    listenfd = Open_listenfd(argv[1]);

    FD_ZERO(&read_set);              /* Clear read set */
    FD_SET(STDIN_FILENO, &read_set); /* Add stdin to read set */
    FD_SET(listenfd, &read_set);     /* Add listenfd to read set */

    while (1)
    {
        ready_set = read_set;
        Select(listenfd + 1, &ready_set, NULL, NULL, NULL);
        if (FD_ISSET(STDIN_FILENO, &ready_set))
            command(); /* Read command line from stdin */
        if (FD_ISSET(listenfd, &ready_set))
        {
            clientlen = sizeof(struct sockaddr_storage);
            connfd = Accept(listenfd, (SA *)&clientaddr, &clientlen);
            echo(connfd); /* Echo client input until EOF */
            Close(connfd);
        }
    }
}

void command(void)
{
    char buf[MAXLINE];
    if (!Fgets(buf, MAXLINE, stdin))
        exit(0);       /* EOF */
    printf("%s", buf); /* Process the input command */
}
```

该程序首先打开一个监听描述符（第 16 行），然后使用宏`FD_ZERO`创建一个空的读取集（第 19 行）：

![20220830233911](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220830233911.png)

然后代码第 20 行和第 21 行分别将描述符 0（标准输入）和描述符 3（监听描述符）加入读取集：

![20220830234609](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220830234609.png)

此时，典型的服务器循环开始。不过我们调用`select`而非`accept`函数来等待监听描述符或标准输入准备好被读取（第 26 行）。例如，如果用户按下回车键，则`select`函数将修改`ready_set`的值：

![20220830235034](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220830235034.png)

一旦 select 返回，我们使用 FD_ISSET 宏来确定哪些描述符可以读取。 如果标准输入准备好（第 25 行），我们调用命令函数，它在返回主程序之前读取、解析和响应命令。 如果监听描述符准备好了（第 27 行），我们调用 accept 来获取一个连接的描述符，然后调用图 11.22 中的 echo 函数，它会回显来自客户端的每一行，直到客户端关闭其连接端。

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

该程序使用一个`pool`类型的结构体（第 3～12 行）保存活跃的客户端，并调用`init_pool`函数初始化客户端池（第 29 行）。在无限循环的每次迭代中，服务器调用`select`函数检测两种不同类型的输入事件：来自新客户端的连接请求；为现有客户端提供服务的连接描述符已准备好被读取。当连接请求到达时（第 38 行），服务器打开连接（第 41 行）并将新客户端加入到客户端池中（第 42 行）。最后，服务器调用`check_clients`函数从每个已连接的描述符中写入单个文本行（第 46 行）。

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

`init_pool`函数初始化客户端池，`clientfd`数组包含所有已连接的描述符。最初我们使用 -1 填充该数组（第 5～7 行），参数`listenfd`是读取集中唯一的描述符（第 10～12 行）。

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

`add_client`函数将一个新客户端添加到活跃客户端池中。如果`clientfd`数组中有空位（第 6 行），函数就将连接描述符`connfd`添加到该数组中并初始化一个 $R_{io}$ 读取缓冲区以调用`Rio_readinitb`（第 9～10 行）。随后函数将连接描述符添加到读取集（第 13 行），并更新客户端池中的一些属性：`maxfd`变量跟踪选择的最大文件描述符；`maxi`变量跟踪`clientfd`数组的最大索引。这样`check_clients`函数就不需要遍历整个数组。

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

`check_clients`函数从每个连接描述符中回显一个文本行。如果我们成功地从描述符中读取了文本行，那么我们将该行回显给客户端（第 18-21 行）。 请注意，在第 15 行，我们正在维护从所有客户端接收到的总字节数的累积计数。如果我们检测到 EOF 是因为客户端已经关闭了它的连接端，那么我们关闭连接端（第 27 行）并从池中删除描述符（第 28-29 行）。

### I/O 多路复用的优缺点

## 使用线程实现并发
