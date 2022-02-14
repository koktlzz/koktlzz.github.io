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

异常处理涉及到软件和硬件的密切合作，因此很容易将不同组件执行的工作相混淆。系统中每种可能的异常都对应了一个唯一的非负整数，即异常数字（Exception Number）。当计算机系统启动时，操作系统会初始化一个跳转表，称为异常表（Exception Table）。其中的每个条目 k 均包含了异常 k 的处理程序地址：

![20220209224522](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220209224522.png)

### 异常的分类

### Linux/x86-64 系统中的异常

## 进程

```c
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <stdio.h>
#include <sys/errno.h>
#include <string.h>

void unix_error(char *msg)
{
    fprintf(stderr, "%s: %s\n", msg, strerror(errno));
    exit(0);
}

pid_t Fork(void){
    pid_t pid;
    if ((pid = fork()) < 0)
        unix_error("Fork error");
    return pid;
}

int main()
{

    pid_t pid;
    int x = 1;
    pid = Fork();
    if (pid == 0)
    { /*Child*/
        printf("child : x=%d\n", ++x);
        exit(0);
    }
    /* Parent */
    // printf("parent: x=%d\n", --x);
    exit(0);
}
```
