---
title: "CSAPP 读书笔记：并发编程"
date: 2022-08-25T15:31:42+01:00
draft: true
series: ["CSAPP 读书笔记"]
tags: ["OS"]
summary: "正如我们在第八章介绍的，两个在时间上重叠的逻辑控制流是并发的。硬件异常处理程序、进程和 Linux 信号处理程序等都是计算机系统在不同层级上对并发的应用 ..."
---

正如我们在 [第八章](/posts/exception-control-flow-note/#并发流) 介绍的，两个在时间上重叠的逻辑控制流是并发的。硬件异常处理程序、进程和 Linux 信号处理程序等都是计算机系统在不同层级上对并发的应用。现代操作系统为构建并发程序提供了三种基本方法：

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

### 一个基于进程的并发服务器

一个基于进程的并发服务器代码如下，其中第 32 行调用的`echo`函数来自 [echo.c](http://csapp.cs.cmu.edu/2e/ics2/code/netp/echo.c)：

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

由于服务器将运行很长时间，我们必须安装一个 SIGCHLD 信号处理程序来回收子进程（第 4～9 行），详见 [正确的信号处理](/posts/exception-control-flow-note/#正确的信号处理)。

父进程必须关闭`connfd`（第 36 行），否则连接描述符指向的 [打开文件表条目](/posts/system-level-io-note/#共享文件) 永远不会被释放，从而导致内存泄漏。子进程则不需要关闭`connfd`（第 33 行可以省略），因为内核会在它退出时自动关闭其所有的打开描述符。

### 进程的优缺点

父子进程共享打开文件表，但并不共享用户地址空间（虚拟内存），进程之间必须显式地使用 IPC（Interprocess Communications）机制来共享状态信息。由于进程控制和 IPC 的开销很高，基于进程的并发程序往往很慢。

## 使用 I/O 多路复用实现并发

## 使用线程实现并发
