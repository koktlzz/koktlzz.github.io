---
title: "CSAPP 读书笔记：系统级 I/O"
date: 2022-08-07T11:30:42+01:00
draft: false
series: ["CSAPP 读书笔记"]
tags: ["OS"]
summary: "输入/输出 (I/O) 是在主存储器和外部设备（如磁盘驱动、终端和网络等）之间复制数据的过程。输入操作将数据从 I/O 设备复制到主存，输出操作则将数据从主存复制到设备 ..."
---

输入/输出 (I/O) 是在主存储器和外部设备（如磁盘驱动、终端和网络等）之间复制数据的过程。输入操作将数据从 I/O 设备复制到主存，输出操作则将数据从主存复制到设备。

## Unix I/O

Linux 文件是由 m 个字节组成的序列：

$$B_0, B_1, ..., B_k, ..., B_{m-1}$$

所有的 I/O 设备均被建模为文件，输入和输出是通过读写对应的文件来完成的。Linux 内核基于这种设备与文件之间的优雅映射为我们提供了一个简单而低级的应用程序接口，即 Unix I/O，它使得所有的输入和输出都能以一致的方式执行：

- 打开文件：应用程序通过向内核发起打开文件请求以访问 I/O 设备。内核将返回一个小的非负整数，即文件描述符（File Descriptor），它将在对文件的后续操作中标识该文件。内核跟踪与打开文件相关的所有信息，而应用程序则只跟踪描述符。每个由 Linux Shell 创建的进程都会打开三个文件：标准输入（`STDIN_ FILENO`，描述符 0）、标准输出（`STDOUT_FILENO`，描述符 1）和标准错误（`STDERR_FILENO`，描述符 2）；
- 改变当前文件位置：内核为每个打开文件维护一个文件位置（[File Position](https://www.gnu.org/software/libc/manual/html_node/File-Position.html#:~:text=The%20file%20position%20is%20normally%20set%20to%20the,write%20operations%20at%20any%20position%20within%20the%20file.)）k，初始值为 0。文件位置是文件中下一个即将被读取或写入的字符到文件起始位置的字节偏移量，并非指该文件在文件系统中的位置。应用程序可以通过执行 Seek 操作来显式地设置当前文件位置 k；
- 读写文件：读取操作从当前文件位置 k 开始，复制 n 个字节到内存中，随后令 k 增加 n。当文件位置大于或等于文件大小时，读取操作会触发 EOF（End-of-File）。类似地，写入操作从当前文件位置 k 开始，复制 n 个字节到文件中并更新 k 的值；
- 关闭文件：当应用程序结束对文件的访问时，它会向内核发起关闭文件请求，内核释放打开文件时创建的数据结构并将描述符放回可用描述符池。若进程因某些原因终止，内核将关闭所有打开的文件并释放相应的内存资源。

## 文件

每个 Linux 文件都有一个表征其在系统中角色的类型：

- 常规文件（Regular File）：对于应用程序来说，常规文件分为仅包含 ASCII 或 Unicode 字符的文本文件（Text File）和二进制文件（Binary File）；但对于内核而言，两者没有区别。Linux 文本文件由一系列文本行（Text Line）组成，其中每一行都以换行符`\n`结尾；
- 目录（Directory）：目录是由链接（Link）数组构成的文件。链接将一个文件名映射到一个文件，该文件可能是另一个目录（如下图所示）。每个目录中至少包含两个链接：`.`指向目录本身，而`..`指向上级目录；
- Socket：用于通过网络与另一个进程通信的文件。

![20220807220609](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220807220609.png)

## 打开和关闭文件

进程可以调用函数`open`来打开一个已存在的文件或创建一个新文件：

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
int open(char *filename, int flags, mode_t mode);
// Returns: new file descriptor if OK, −1 on error
```

返回的文件描述符是进程当前未打开的最小描述符。参数`flags`指示进程如何访问文件：

- `O_RDONLY`：只读
- `O_WRONLY`：只写
- `O_RDWR`：读写

该参数还可以与一个或多个位掩码进行或（`OR`）运算，这些位掩码提供写入的附加说明：

- `O_CREAT`：如果文件不存在，则创建一个空文件；
- `O_TRUNC`：如果文件已经存在，则清空文件内容；
- `O_APPEND`：在每次写入操作之前，将文件位置设置为文件末尾。

若文件已存在，参数`mode`应设为 0；反之，则设为新文件的访问权限位，可选项如下图所示：

![20220807233921](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220807233921.png)

作为进程上下文的一部分，每个进程都有一个通过`umask`函数设置的`umask`掩码。当进程调用`open`函数创建新文件时，文件的访问权限位会被设置为`mode & ~umask`。如下示例程序将创建一个所有者拥有读写权限、其他用户都有读取权限的新文件：

```c
#define DEF_MODE   S_IRUSR|S_IWUSR|S_IRGRP|S_IWGRP|S_IROTH|S_IWOTH
#define DEF_UMASK  S_IWGRP|S_IWOTH

umask(DEF_UMASK);
fd = Open("foo.txt", O_CREAT|O_TRUNC|O_WRONLY, DEF_MODE);
```

进程调用`close`函数来关闭一个打开的文件，若文件描述符已关闭将引发错误：

```c
#include <unistd.h>
int close(int fd);
// Returns: 0 if OK, −1 on error
```

## 读写文件
