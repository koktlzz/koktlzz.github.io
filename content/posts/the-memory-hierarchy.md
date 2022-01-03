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

总线是一组可以传输地址、数据和控制信号的并行线的集合，其中控制线负责同步事务并识别当前正在执行事务的类型。下图展示了计算机系统中的总线结构：

![20220103173737](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220103173737.png)

图中的 I/O Bridge 是一组芯片集合，包含了上文中提到的存储控制器。它会将系统总线的电信号转换为存储器总线的电信号，反之亦然。当 CPU 执行数据读取指令时，其芯片上的总线接口（Bus Interface）电路会在总线上初始化一个读事务，过程如下：

![20220103175152](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220103175152.png)

写事务的过程与之类似：

![20220103175511](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220103175511.png)

### 磁盘存储

磁盘（Disk）是主力存储设备，可以容纳成百上千 GB 的数据，但其读写速度却远远低于 RAM。本节中的磁盘主要指的是旋转磁盘，并不涉及 SSD。

#### 磁盘构造

磁盘是由盘片（Platter）组成的，每个盘片都有两面（Surface）。下图 6.9(a) 展示了一个典型磁盘 Surface 的构造：

![20220103180307](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220103180307.png)

每个 Surface 都由一组被称为 Track 的同心环构成，每条 Track 又可以被划分为多个扇区（Section）。每个扇区中都存储了数量相同的数据位（通常为 512 字节），它们被只保存了扇区标识的 Gap 所分隔。

如上图6.9(b) 所示，Cylinder 是所有 Surface 上与主轴等距的 Track 集合。

#### 磁盘容量

磁盘的容量由以下三个参数决定：

- 记录密度（$bits/in$）：一英寸长度的 Track 中存储的位数；
- 轨道密度（$tracks/in$）：从主轴中心处延伸一英寸半径内的 Track 数量；
- 面密度（$bits/in^2$）：记录密度和轨道密度的乘积。

在以前的磁盘中，每条 Track 上的扇区数量是相同的。这样当面密度上升之后，Gap 就会越来越大，从而造成浪费。因此现代大容量磁盘使用多区记录（Multiple Zone Recording）技术，将 Cylinder 分成多个不相交且连续的 Zone。一个 Zone 中每条 Track 所包含的扇区数量均等于该 Zone 中最内侧 Track 的扇区数量。

#### 磁盘操作

磁盘使用连接在传动臂（Actuator Arm）末端的读/写头来进行读写操作：

![20220103210533](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220103210533.png)

传动臂沿自身旋转轴不断移动，可以将读/写头定位到任意 Track 上，这一操作称为寻道（Seek）。如图 6.10(b) 所示，拥有多个盘片的磁盘的每一个 Surface 都有其对应的读/写头。不过在任何时刻，所有的读/写头均处在相同的 Cylinder 上。传动臂的移动速度非常快，即使是微小的灰尘也会干扰到读/写头的寻道，因此磁盘需要密封包装。

磁盘以扇区大小的块为单元进行读写，其访问时间受三个主要因素的影响：

- 寻道时间：即移动传动臂所需的时间;
- 旋转延时：当读/写头定位到目标 Track 上时，磁盘还需要将目标扇区的第一个位旋转到读/写头下，所需时间称为旋转延时；
- 传输时间：磁盘读写目标扇区内容所需的时间，由磁盘旋转速度和每条 Track 中的扇区数量所决定。

相比其他两个因素，传输时间小到可以忽略。而寻道时间和旋转延时大致相等，因此可以用两倍的寻道时间来估算磁盘的访问时间。

#### 磁盘的逻辑块

为了向操作系统隐藏磁盘构造的复杂性，现代磁盘将自身简化为一个由 B 个逻辑块组成的序列。每个逻辑块的尺寸与扇区大小相同，编号为 0，1，... ，B - 1。磁盘控制器（Disk Controller）负责维护逻辑块编号与实际物理磁盘扇区之间的映射关系，它会将操作系统读取的目标逻辑块编号转换为唯一标识物理扇区的三元组（Surface，Track 和扇区）。读取到的信息将首先保存在磁盘控制器的缓冲区中，然后再拷贝到主存。

#### 连接的 I/O 设备

显卡、显示器、鼠标、键盘和磁盘等输入/输出 (I/O) 设备使用 I/O 总线连接到 CPU 和主存储器上：

![20220103214715](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220103214715.png)

#### 访问磁盘

CPU 从磁盘中读取数据的过程如下：

![20220103214944](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220103214944.png)

CPU 使用 Memory-mapped I/O 技术向 I/O 设备发送指令，如上图中 (a) 所示。系统会在地址空间中保留一块地址区用于与 I/O 设备的通信，其中的地址称为 I/O Port。每个连接到总线上的设备都会映射到一个或多个 I/O Port。

CPU 会发送三个指令到目标 I/O Port 来初始化读取操作：

- 告知磁盘启动读取的命令
- 目标数据所在逻辑块的编号
- 数据存储的主存地址

在发出请求之后，CPU 通常会在磁盘读取时进行其他工作。如上图中 (b) 所示，设备自行执行读写总线事务，无需 CPU 的参与，这一过程称为 DMA（Direct Memory Access ）。当数据存储在主存中之后，磁盘控制器便会向 CPU 发送中断信号以通知磁盘读取完成，如上图中 (c) 所示。

### SSD

固态硬盘（Solid State Disk，SSD）

## 局部性
