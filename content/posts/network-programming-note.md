---
title: "CSAPP 读书笔记：网络编程"
date: 2022-08-11T09:48:42+01:00
draft: false
series: ["CSAPP 读书笔记"]
tags: ["Network"]
summary: "所有的网络应用程序都基于相同的基本编程模型，具有相似的整体逻辑结构，并且依赖于相同的编程接口。每个网络应用程序都基于客户端-服务器模型（Client-Server Model），并由一个服务器进程和多个客户端进程组成 ..."
---

所有的网络应用程序都基于相同的基本编程模型，具有相似的整体逻辑结构，并且依赖于相同的编程接口。

## 客户端-服务器编程模型

每个网络应用程序都基于客户端-服务器模型（Client-Server Model），并由一个服务器进程和多个客户端进程组成。服务器管理资源，操作它们以向客户端提供服务。

客户端-服务器模型中的基本操作是事务（Transaction），它由以下四个步骤组成：

![20220811213403](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220811213403.png)

请注意，这里的客户端和服务器指的是进程而非机器或主机（Host）。单台主机能够同时运行多个不同的客户端和服务器，客户端和服务器事务也可以在相同或不同的主机上执行。

## 网络

不过，客户端和服务器通常在不同的主机上运行，两者使用计算机网络的软硬件资源进行通信。对于主机来说，网络只是一种 I/O 设备。如下图所示，主机使用 DMA 技术将网络数据通过 I/O 总线和存储器总线从适配器复制到主存（反之亦然）：

![20220811214041](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220811214041.png)

网络是按地理邻近度组织的分层系统，其最低层是覆盖一个建筑物或校园的局域网（Local Area Network，LAN）。迄今为止最流行的局域网技术是以太网（Ethernet）：

![20220811215257](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220811215257.png)

在更高层，多个互不连通的局域网可以通过路由器连接为互联网（Interconnected Network，Internet），而多个路由器点对点连接便形成了广域网（Wide Area Network，WAN）：

![20220811220244](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220811220244.png)

互联网可以由使用完全不同且不兼容技术的局域网和广域网组成，这是它的一个关键特性。因此，我们必须在每台主机和路由器上运行协议软件（Protocol Software）来消除不同网络之间的差异。该软件实现的协议将管理主机和路由器如何协作以传输数据，它必须提供两个基本功能：

- 命名方案（Naming Scheme）：为主机地址定义统一的格式，并为每台主机分配至少一个唯一标识它的互联网地址；
- 交付机制（Delivery Mechanism）：定义一种统一的方式将数据位封装为若干个块，即数据包（Packet）。其大小和源/目的主机地址位于包的头部（Header），而源主机发送的数据位则在有效负载（Payload）之中。

## 全球 IP 互联网

![20220811222535](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220811222535.png)

几乎所有的现代计算机系统都支持 TCP/IP 协议（Transmission Control Protocol/Internet Protocol），因此每台互联网主机上均运行着实现了该协议的软件。客户端和服务器使用 Socket 接口函数和 Unix I/O 函数混合的方式进行通信。前者通常为系统调用，它们会请求内核（Trap into Kernel）调用 TCP/IP 中的各种内核态函数。

在程序员看来，互联网是具有以下属性的主机集合：

- 所有主机均映射到一组 32 位的 IP 地址；
- 所有 IP 地址均映射到一组标识符，即域名（Domain Name）；
- 一台主机上的进程可以通过连接（Connection）与其他任何主机上的进程通信。

### IP 地址

IP 地址是一个无符号的 32 位整数。由于历史原因，网络程序将其存储在如下结构体中：

```c
/* IP address structure */
struct in_addr {
    uint32_t  s_addr; /* Address in network byte order (big-endian) */
};
```

互联网中主机存储字节的顺序可能不同，因此 TCP/IP 必须为整型数据项（如数据包头部的 IP 地址）定义一个统一的网络字节顺序（大端）。即使主机的字节顺序是小端，`in_addr`结构体中的 IP 地址也会以网络字节顺序存储。Unix 提供了用于转换字节顺序的函数：

```c
#include <arpa/inet.h>
// host to network
uint32_t htonl(uint32_t hostlong);
uint16_t htons(uint16_t hostshort);
// Returns: value in network byte order

// network to host
uint32_t ntohl(uint32_t netlong);
uint16_t ntohs(unit16_t netshort);
// Returns: value in host byte order
```

为了便于人类阅读，IP 地址通常以点分十进制（Dotted-Decimal）的形式表示。应用程序可以使用函数`inet_pton`和`inet_ntop`对上述两种方式进行转换：

```c
#include <arpa/inet.h>
int inet_pton(AF_INET, const char *src, void *dst);
// Returns: 1 if OK, 0 if src is invalid dotted decimal, −1 on error

const char *inet_ntop(AF_INET, const void *src, char *dst,
                      socklen_t size);
// Returns: pointer to a dotted-decimal string if OK, NULL on error
```

### 域名

像 IP 地址这样较大的整数很难让人记住，因此互联网定义了一组更加人性化的域名集合并将其与 IP 地址映射。域名是由句点分隔的单词（字母、数字和破折号）序列，如`whaleshark.ics.cs.cmu.edu`。域名的层级结构如下图所示：

![20220814171721](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220814171721.png)

### 连接

客户端和服务器通过连接来收发字节流并进行通信。连接是点对点，全双工（数据可以同时在两个方向上传输）且可靠的——除非发生一些灾难性的故障。

Socket 是连接的端点，每个 Socket 都对应了一个Socket 地址。该地址由 IP 地址和 16 位整型的端口（Port）组成，表示为：`address:port`。

客户端 Socket 地址中的端口通常是其发起连接请求时由内核自动分配的，被称为临时端口（Ephemeral Port）；而服务器 Socket 地址中的端口则通常与服务永久关联，被称为知名端口（Well-known Port）。

连接由两个端点的 Socket 地址（Socket Pair）唯一标识，可以用元组表示为：`(cliaddr:cliport, servaddr:servport)`。

![20220814173954](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220814173954.png)

## Socket 接口

![20220814174316](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220814174316.png)
