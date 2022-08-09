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

应用程序调用`read`和`write`函数来执行输入和输出：

```c
#include <unistd.h>
ssize_t read(int fd, void *buf, size_t n);
// Returns: number of bytes read if OK, 0 on EOF, −1 on error
ssize_t write(int fd, const void *buf, size_t n);
// Returns: number of bytes written if OK, −1 on error
```

`read`函数从参数`fd`的当前文件位置复制最多`n`个字节到内存中`buf`指向的位置，`write`函数则从内存中`buf`指向的位置复制最多`n`个字节到参数`fd`的当前位置。示例程序使用上述两种函数将标准输入以字节为单位复制到标准输出：

```c
#include "csapp.h"

int main(void) 
{
    char c;
    while(Read(STDIN_FILENO, &c, 1) != 0) 
        Write(STDOUT_FILENO, &c, 1);
    exit(0);
}
```

在某些情况下，读写操作传输的字节数小于应用程序请求的字节数。不足数（Short Count）的产生并不代表发生了错误，它可能由多种原因导致：

- 读取时遇到 EOF：若我们对一个 20 字节的文件执行`read(fd, *buf, 50)`，那么第一次调用将返回一个 20 的不足数，第二次调用则返回 0（EOF）；
- 从终端读取文本行：若打开的文件是终端设备（即键盘和显示器），那么每次`read`调用都将传输一个文本行并返回一个与文本行大小相等的不足数；
- 读写 Socket：若打开的文件是 Socket，那么内部缓冲区限制和网络延迟将使读写操作返回不足数。

因此除遇到 EOF 外，读写磁盘文件不会导致不足数的产生。但如果我们想要构建健壮而可靠的网络应用程序，就必须重复调用`read`和`write`以保证所有请求的字节均已被传输。

## 使用 $R_{IO}$ 包实现健壮读写

$R_{IO}$ 包为应用程序提供了方便、健壮且高效的 I/O，可以解决编写网络程序时遇到的不足数问题：

- 无缓冲的输入/输入函数：直接在内存和文件之间传输数据，适用于从网络中读写二进制数据；
- 有缓冲的输入函数：从应用程序级别的缓冲区中读取文本行和二进制数据，与标准 I/O（如`printf`）函数类似。该函数是线程安全（Thread-safe）的，并且可以对同一描述符任意交错（Interleave）。例如，我们可以从描述符中读取一些文本行，然后读取一些二进制数据，最后再读取一些文本行。

### 无缓冲的输入/输入函数

```c
#include "csapp.h"
ssize_t rio_readn(int fd, void *usrbuf, size_t n);
ssize_t rio_writen(int fd, void *usrbuf, size_t n);
// Returns: number of bytes transferred if OK, 0 on EOF (rio_readn only), −1 on error
```

`rio_readn`函数从参数`fd`的当前文件位置复制最多`n`个字节到内存中`usrbuf`指向的位置，`rio_writen`函数则从内存中`usrbuf`指向的位置复制最多`n`个字节到参数`fd`的当前位置。前者只有在遇到 EOF 时返回不足数，后者则从不返回不足数。

若上述函数被应用程序的信号处理程序的返回中断，它们会重新调用`read`和`write`函数：

```c
ssize_t rio_readn(int fd, void *usrbuf, size_t n)
{
    size_t nleft = n;
    ssize_t nread;
    char *bufp = usrbuf;
    while (nleft > 0)
    {
        if ((nread = read(fd, bufp, nleft)) < 0)
        {
            if (errno == EINTR) /* Interrupted by sig handler return */
                nread = 0;      /* and call read() again */
            else
                return -1;      /* errno set by read() */
        }
        else if (nread == 0)
            break;              /* EOF */
        nleft -= nread;
        bufp += nread;
    }
    return (n - nleft);         /* Return >= 0 */
}

ssize_t rio_writen(int fd, void *usrbuf, size_t n)
{
    size_t nleft = n;
    ssize_t nwritten;
    char *bufp = usrbuf;
    while (nleft > 0)
    {
        if ((nwritten = read(fd, bufp, nleft)) < 0)
        {
            if (errno == EINTR) /* Interrupted by sig handler return */
                nwritten = 0;   /* and call write() again */
            else
                return -1;      /* errno set by write() */
        }
        nleft -= nwritten;
        bufp += nwritten;
    }
    return n;
}
```

### 有缓冲的输入函数

假设我们需要编写一个计算文本文件行数的程序，最简单的方法便是调用`read`函数每次读取一个字节并检查是否有换行符。但由于`read`是系统调用，频繁的上下文切换将导致程序效率低下。

更好的方法是调用包装函数`rio_readlineb`从内部读取缓冲区（Read Buffer）复制文本行，只有当缓冲区为空时才调用`read`以重新填充缓冲区。$R_{IO}$ 包还为同时包含文本行和二进制数据的文件（如 HTTP 响应）提供了`rio_readn`函数的有缓冲版本，即`rio_readnb`：

```c
#include "csapp.h"
ssize_t rio_readlineb(rio_t *rp, void *usrbuf, size_t maxlen);
ssize_t rio_readnb(rio_t *rp, void *usrbuf, size_t n);
// Returns: number of bytes read if OK, 0 on EOF, −1 on error
```

在调用上述两种有缓冲的输入函数前，我们需要为每个打开文件描述符调用一次`rio_readinitb`。该函数将描述符`fd`与地址`rp`处的读取缓冲区（类型为`rio_t`）相关联：

```c
#define RIO_BUFSIZE 8192
typedef struct {
    int rio_fd;                /* Descriptor for this internal buf */
    int rio_cnt;               /* Unread bytes in internal buf */
    char *rio_bufptr;          /* Next unread byte in internal buf */
    char rio_buf[RIO_BUFSIZE]; /* Internal buffer */
} rio_t;

void rio_readinitb(rio_t *rp, int fd) 
{
    rp->rio_fd = fd;  
    rp->rio_cnt = 0;  
    rp->rio_bufptr = rp->rio_buf;
}
```

$R_{IO}$ 包读取例程的核心是`rio_read`函数，它其实是`read`函数的有缓冲版本。若读取缓冲区中的未读字节数`rp->rio_cnt`为 0，则在循环内调用`read`函数对其填充；若读取缓冲区非空，则调用`memcpy`函数将`min(n, rp->rio_cnt)`字节从缓冲区复制到`usrbuf`指向的内存位置：

```c
static ssize_t rio_read(rio_t *rp, char *usrbuf, size_t n)
{
    int cnt;

    while (rp->rio_cnt <= 0)
    { /* Refill if buf is empty */
        rp->rio_cnt = read(rp->rio_fd, rp->rio_buf,
                           sizeof(rp->rio_buf));
        if (rp->rio_cnt < 0)
        {
            if (errno != EINTR)           /* Interrupted by sig handler return */
                return -1;
        }
        else if (rp->rio_cnt == 0)        /* EOF */
            return 0;
        else
            rp->rio_bufptr = rp->rio_buf; /* Reset buffer ptr */
    }

    /* Copy min(n, rp->rio_cnt) bytes from internal buf to user buf */
    cnt = n;
    if (rp->rio_cnt < n)
        cnt = rp->rio_cnt;
    memcpy(usrbuf, rp->rio_bufptr, cnt);
    rp->rio_bufptr += cnt;
    rp->rio_cnt -= cnt;
    return cnt;
}
```

在应用程序看来，`rio_read`函数与`read`函数具有相同的语义：执行发生错误时，它返回 -1 并设置 errno；执行遇到 EOF 时，它返回 0；当请求的字节数大于读取缓冲区中的未读字节数时，它返回一个不足数。因此我们可以通过将`read`替换为`rio_read`来构建不同类型的有缓冲读取函数。

实际上，`rio_readnb`与`rio_readn`具有完全相同的结构，只不过我们用`rio_read`替换了`read`：

```c
ssize_t rio_readnb(rio_t *rp, void *usrbuf, size_t n)
{
    size_t nleft = n;
    ssize_t nread;
    char *bufp = usrbuf;

    while (nleft > 0)
    {
        if ((nread = rio_read(rp, bufp, nleft)) < 0)
            return -1;  /* errno set by read() */
        else if (nread == 0)
            break; /* EOF */
        nleft -= nread;
        bufp += nread;
    }
    return (n - nleft); /* return >= 0 */
}
```

类似地，`rio_readlineb`函数从文件`rp`中读取一个文本行并将其复制到内存中`usrbuf`指向的位置。循环内每次对`rio_read`的调用都会把读取缓冲区中的一个字节复制到`&c`，然后检查它是否是换行符：

```c
ssize_t rio_readlineb(rio_t *rp, void *usrbuf, size_t maxlen)
{
    int n, rc;
    char c, *bufp = usrbuf;

    for (n = 1; n < maxlen; n++)
    {
        if ((rc = rio_read(rp, &c, 1)) == 1)
        {
            *bufp++ = c;  /* Copy rp to user buf */
            if (c == '\n')
            {
                n++;
                break;
            }
        }
        else if (rc == 0)
        {
            if (n == 1)
                return 0; /* EOF, no data read */
            else
                break;    /* EOF, some data was read */
        }
        else
            return -1;    /* Error */
    }
    *bufp = 0;
    return n - 1;
}
```

## 读取文件元数据

应用程序调用`stat`和`fstat`函数获取一个文件的元数据：

```c
#include <unistd.h>
#include <sys/stat.h>
int stat(const char *filename, struct stat *buf);
int fstat(int fd, struct stat *buf);
// Returns: 0 if OK, −1 on error
```

函数`stat`使用文件名`*filename`作为输入，将信息填写到`stat`结构体中。`fstat`与之类似，但它的参数是文件描述符`fd`。结构体`stat`如下图所示，我们只需关注字段`st_mode`和`st_size`：

![20220808223653](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220808223653.png)

`st_size`包含了文件的大小，而`st_mode`则包含了文件的访问权限和类型。

## 读取目录内容

应用程序调用`opendir`和`readdir`函数读取目录中的内容：

```c
#include <sys/types.h>
#include <dirent.h>
DIR *opendir(const char *name);
// Returns: pointer to handle if OK, NULL on error
#include <dirent.h>
struct dirent *readdir(DIR *dirp);
// Returns: pointer to next directory entry if OK, NULL if no more entries or error
```

函数`opendir`以目录的路径名为参数，返回一个指向目录流（Directory Stream）的指针。流是对有序项目列表的抽象，此处指的是目录中条目的列表。函数`readdir`返回指向目录流中下一个条目的指针，每个条目都是一个`dirent`结构体：

```c
struct dirent {
    ino_t d_ino;       /* inode number */
    char  d_name[256]; /* Filename */
};
```

`d_name`是文件名，`d_ino`是文件的 inode 数。当发生错误时，`readdir`返回`NULL`并设置`errno`。

函数`closedir`关闭目录流并释放所有相关资源：

```c
#include <dirent.h>
int closedir(DIR *dirp);
// Returns: 0 on success, −1 on error
```

## 共享文件

内核使用三种数据结构来表示打开的文件：

- 描述符表（Descriptor Table）：每个进程都有一个独立的描述符表，每个条目均指向打开文件表中的条目，其索引是进程打开的文件描述符；
- 打开文件表（Open File Table）：所有进程共享一个打开文件表，它表示了打开文件的集合。每个文件表条目由当前文件位置（下图中的“File pos”）、当前指向它的描述符表条目数量（下图中的“refcnt”）和一个指向 v-node 表条目的指针。只有当`refcnt`为 0 时，内核才会删除对应的文件表条目；
- v-node 表（V-node Table）：与打开文件表一样，v-node 表由所有进程共享。每个条目都包含了`stat`结构体中的大部分信息，如`st_mode`和`st_size`等。

![20220808234547](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220808234547.png)

如上图所示，描述符 1 和 4 通过不同的打开文件表条目引用不同的文件。这是最典型的情况，文件并未共享，每个描述符对应一个不同的文件。

多个描述符也可以通过不同的打开文件表条目引用相同的文件，例如对同一文件多次调用`open`函数：

![20220808235153](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220808235153.png)

描述符 1 和 4 指向不同的打开文件表条目，因此其文件位置不同。假设文件`foobar.txt`中包含 6 个 ASCII 字符`foobar`，那么如下示例程序的输出结果为`f`：

```c
#include "csapp.h"

int main()
{
    int fd1, fd2;
    char c;
    fd1 = Open("foobar.txt", O_RDONLY, 0);
    fd2 = Open("foobar.txt", O_RDONLY, 0);
    Read(fd1, &c, 1);
    Read(fd2, &c, 1);
    printf("c = %c\n", c);
    exit(0);
}
```

若父进程打开文件的数据结构如上图 [10.12](/posts/system-level-io-note/#共享文件) 所示，则父进程调用`fork`函数后情况变为：

![20220809000232](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220809000232.png)

子进程得到父进程的描述符表副本，两者共享相同的打开文件表和文件位置，因此如下示例程序的输出结果为`o`：

```c
#include "csapp.h"
int main()
{
    int fd;
    char c;
    fd = Open("foobar.txt", O_RDONLY, 0);
    if (Fork() == 0)
    {
        Read(fd, &c, 1);
        exit(0);
    }
    Wait(NULL);
    Read(fd, &c, 1);
    printf("c = %c\n", c);
    exit(0);
}
```

## I/O 重定向

`dup2`函数将描述符表条目`oldfd`复制到`newfd`并覆盖其原始内容。如果`newfd`已经打开，则该函数在复制`oldfd`之前会先关闭`newfd`：

```c
#include <unistd.h>
int dup2(int oldfd, int newfd);
// Returns: nonnegative descriptor if OK, −1 on error
```

假设某进程的打开文件数据结构如上图 [10.12](/posts/system-level-io-note/#共享文件) 所示。描述符 1（标准输出）指向文件 A（如终端），描述符 4 指向文件 B（如磁盘上的文件），文件 A 和 B 的`refcnt`均为 1。那么该进程调用函数`dup2(4, 1)`后情况变为：

![20220809160850](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220809160850.png)

文件 A 被关闭，内核会删除其打开文件表和 v-node 表条目。两个描述符均指向文件 B，其`refcnt`已增加为 2。从现在开始，任何写入到标准输出的数据都会被重定向到文件 B。

## 标准 I/O

C 定义了一组更高级别的输入和输出函数，即标准 I/O 库，它为程序员提供了比 Unix I/O 更高级别的替代方案。`libc`库提供了用于打开和关闭文件（`fopen`和`fclose`）、读取和写入字节（`fread`和`fwrite`）、读取和写入字符串（`fgets`和`fputs`）以及复杂的格式化 I/O（`scanf`和`printf`）函数。

标准 I/O 库将打开的文件建模为流。对于程序员来说，流是指向`FILE`类型结构体的指针。每个 [ANSI C](https://zh.wikipedia.org/wiki/ANSI_C) 程序都以三个打开的​​流`stdin`、`stdout`和`stderr`开头，分别与标准输入、标准输出和标准错误对应：

```c
#include <stdio.h>
extern FILE *stdin;  /* Standard input (descriptor 0) */
extern FILE *stdout; /* Standard output (descriptor 1) */
extern FILE *stderr; /* Standard error (descriptor 2) */
```

`FILE`类型的流是文件描述符和流缓冲区的抽象。 流缓冲区的目的与 Rio 读取缓冲区的目的相同：最大限度地减少昂贵的 Linux I/O 系统调用的数量。 例如，假设我们有一个程序重复调用标准 I/O getc 函数，其中每次调用都返回文件中的下一个字符。 第一次调用 getc 时，库通过一次调用 read 函数来填充流缓冲区，然后将缓冲区中的第一个字节返回给应用程序。 只要缓冲区中有未读取的字节，就可以直接从流缓冲区中提供对 getc 的后续调用。

## 我们应当使用哪种 I/O 函数？
