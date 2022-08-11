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
