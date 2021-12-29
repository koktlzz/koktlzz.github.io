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

![20211228223039](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20211228223039.png)

#### SRAM

SRAM 将每个位存储在一个双稳态（bistable）的存储单元中，每个单元是由一个六晶体管电路实现的，其结构类似于下图中的倒转摆锤：

![20211228222800](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20211228222800.png)

该电路将永远处于两种电压配置（称为状态）之一。任何其他的状态都是不稳定的，将会快速地朝向其中一个稳定的状态移动。因此只要 SRAM 通电，就会永久地保存其值而不受一些扰动（如电噪声）的影响。

#### DRAM

DRAM 将每一位存储为一个电容上的电荷，其存储单元对任何扰动都十分敏感。电流泄露会导致 DRAM 单元在大约 10 到 100 毫秒的时间内失去电荷，因此存储系统必须定期地通过读取并重写来刷新其中的每一位。

#### 传统的 DRAM

DRAM 芯片上的位可以被划分为 $d$ 个超级单元（supercell），每个超级单元包含 $w$ 位。因此，一个 $d$ X $w$ 的 DRAM 能够存储 $dw$ 位的信息。下图展示了一个 16 X 8 的 DRAM 芯片结构：

![20211229220341](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20211229220341.png)

16 个超级单元被排列成一个 4 X 4 的矩阵，图中标为阴影的超级单元地址为 (2, 1)。信息通过外部连接器（称为 pin）流入或流出芯片，每个 pin 都可以携带一位信号。上图中包含了能够传输一字节信息的八个`data` pin，以及携带两位超级单元地址的`addr` pin。

每个 DRAM 芯片都连接了一个称为存储控制器（Memory Controller）的电路，它可以一次传输 $w$ 位的数据。为了读取超级单元 (i, j) 中的内容，存储控制器会向 DRAM 发送行地址 i 和 列地址 j 的寻址请求，分别称为 RAS（Row Access Strobe）和 CAS（Column Access Strobe）。详细的读取过程如下图所示：

![20211229222645](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20211229222645.png)
