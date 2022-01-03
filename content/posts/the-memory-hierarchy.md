---
title: "CSAPP 读书笔记：存储系统的层级结构"
date: 2021-12-28T21:41:42+01:00
draft: true
tags: ["CSAPP","OS"]
summary: "存储系统（Memory System）可以分为多个层级（Hierarchy），每个层级的存储设备（Storage Device）具有不同的容量、成本和访问时间。层级越低，存储设备的容量就越大，成本也越低，但访问速度却越慢。存储系统的层次结构对应用性能有着显著的影响 ..."
---

存储系统（Memory System）可以分为多个层级（Hierarchy），每个层级的存储设备（Storage Device）具有不同的容量、成本和访问时间。层级越低，存储设备的容量就越大，成本也越低，但访问速度却越慢。存储系统的层次结构对应用性能有着显著的影响，而 CPU 与主存储器之间的高速缓存则尤为重要，因为它对程序性能的影响最大。

## 存储技术

### 随机存取存储器

随机存取存储器（Random Access Memory，RAM）有两种，分别是静态（static）的 SRAM 和 动态（dynamic）的 DRAM。其中，SRAM 的访问速度比 DRAM 更快，不过其成本明显更高。SRAM 用于 CPU 芯片内外的缓存，而 DRAM 则用于主存储器和图形系统中的帧缓存（Frame Buffer）。

#### SRAM

SRAM 将每个位存储在一个双稳态（bistable）的存储单元中，每个单元是由一个六晶体管电路实现的，其结构类似于下图中的倒转摆锤：

![20211228222800](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20211228222800.png)

该电路将永远处于两种电压配置（称为状态）之一。任何其他的状态都是不稳定的，将会快速地朝向其中一个稳定的状态移动。因此只要 SRAM 通电，就会永久地保存其值而不受一些扰动（如电噪声）的影响。

#### DRAM

DRAM 将每一位存储为一个电容上的电荷，其存储单元对任何扰动都十分敏感。电流泄露会导致 DRAM 单元在大约 10 到 100 毫秒的时间内失去电荷，因此存储系统必须定期地通过读取并重写来刷新其中的每一位。

![20211228223039](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20211228223039.png)

#### 传统的 DRAM

DRAM 芯片上的位可以被划分为 $d$ 个超级单元（supercell），每个超级单元包含 $w$ 位。因此，一个 $d \times w$ 的 DRAM 能够存储 $dw$ 位的信息。下图展示了一个 16 $\times$ 8 的 DRAM 芯片结构：

![20211229220341](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20211229220341.png)

16 个超级单元被排列成一个 4 $\times$ 4 的矩阵，图中标为阴影的超级单元地址为 (2, 1)。信息通过外部连接器（称为 pin）流入或流出芯片，每个 pin 都可以携带一位信号。上图中包含了能够传输一字节信息的八个`data` pin，以及携带两位超级单元地址的`addr` pin。

每个 DRAM 芯片都连接了一个称为存储控制器（Memory Controller）的电路，它可以一次传输 $w$ 位的数据。为了读取超级单元 (i, j) 中的内容，存储控制器会向 DRAM 发送行地址 i 和 列地址 j 的寻址请求，分别称为 RAS（Row Access Strobe）请求和 CAS（Column Access Strobe）请求。详细的读取过程如下图所示：

![20211229222645](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20211229222645.png)

存储控制器首先发送行地址请求，DRAM 会将整个第 2 行拷贝到内部的行缓冲区（Internal Row Buffer）中。然后存储控制器再发送列地址请求，DRAM 将从行缓冲区中拷贝超级单元 (2, 1) 的值返回给存储控制器。

只要对存储控制器读取 DRAM 的过程有所了解，我们就能明白超级单元的排列方式为什么是矩阵而非一维线性数组。在我们的例子中，DRAM 包含了 16 个超级单元。如果按一维数组排列，则超级单元的地址范围为 0 到 15，因此就需要四个 pin（四位）来进行寻址。当然，矩阵排列方式也有一定缺点。存储控制器读取超级单元需要分为两个步骤，从而增加了访问时间。

#### 存储模块

DRAM 芯片被封装在存储模块（Memory Modules）中，如下图所示：

![20211230215759](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20211230215759.png)

示例的存储模块包含 8 个 8M $\times$ 8 DRAM 芯片，能够存储 64 MB 的数据。每个超级单元存储主存储器中的一字节（八位）信息，八个超级单元就可以表示一个地址为 A 的 64 位 word。存储控制器首先将地址 A 转换为超级单元地址 (i, j)，然后将其发送到存储模块中。存储模块会向所有的 DRAM 广播地址 i 和 j，从而得到每个 DRAM 中超级单元 (i, j) 的内容。存储模块中的电路会将结果整合成一个 64 位的 word，最终返回给存储控制器。

#### 改进的 DRAM

我们对传统的 DRAM 进行优化，从而设计出访问速度更快的 DRAM：

- FPM（Fast Page Mode）DRAM：传统 DRAM 将整行数据拷贝到缓冲区中，使用其中一个并丢弃剩余的超级单元。FPM DRAM 可以连续访问同一行的超级单元，因此速度更快。例如读取一行中的四个超级单元，传统 DRAM 需要发送四次 RAS 请求和四次 CAS 请求，而 FPM DRAM 则只需要一次 RAS 请求和四次 CAS 请求；
- EDO（Extended Data Out） DRAM：通过减少发送 CAS 信号的时间间隔来对 FPM DRAM 进行优化；
- SDRAM（Synchronous DRAM）：上文提及的所有 DRAM 都是异步（Asynchronous）地向存储控制器发送信号的，而同步的 SDRAM 输出超级单元的速率更快；
- DDDR（Double Data-Rate Synchronous）DRAM：使用时钟边缘（Clock Edge）作为控制信号来将 SDRAM 的速度提升一倍；
- VRAM（Video RAM）：常用于图形系统的帧缓存，其本质与 FPM DRAM 类似。区别在于 VRAM 通过依次对内部缓冲区的内容移位来输出数据，并且可以向内存 并行读写。

#### 非易失性存储器

非易失性存储器（Nonvolatile Memory）在断电后还会继续保留其存储的信息。即使有些非易失性存储器是可读写的，但由于历史原因，它们被统一称为 ROM（Read-only Memory）。不同种类 ROM 的区别在于其能够被重新编程（写入）的最大次数，以及写入的实现机制。

- PROM（Programmable ROM）：只能被编程一次；
- EPROM（Erasable programmable ROM）：可以被擦除和重新编程约 1000 次；
- EEPROM（Electrically Erasable PROM）：与 EPROM 相比，不需要单独的物理编程设备，但只可以重新编程 105 次；
- 闪存（Flash Memory）：基于 EEPROM 的一项重要存储技术，SSD 就是通过闪存实现的。

存储在 ROM 中的程序通常称为固件（Firmware）。一些系统会在固件中提供简单的输入/输出功能，比如 PC 的 BIOS（Basic Input/Output System）。

#### 访问主存储器

数据通过共享的电气管道在 CPU 和 DRAM 主存储器之间流动，这些管道称为总线（Bus）。而数据传输的过程可以分成一系列的总线事务（Bus Transaction），其中读事务将数据从主存加载到 CPU 中，写事务则将数据从 CPU 传输到主存中。

总线是一组可以传输地址、数据和控制信号的并行线的集合，其中控制线负责同步事务并识别当前正在执行事务的类型。下图展示了一个示例计算机系统中的总线结构：

