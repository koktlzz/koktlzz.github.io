---
title: "CSAPP 读书笔记：异常控制流"
date: 2022-01-16T16:41:42+01:00
draft: true
tags: ["CSAPP","OS"]
summary: " ..."
---

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
