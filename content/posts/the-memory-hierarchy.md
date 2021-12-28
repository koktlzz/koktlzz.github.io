---
title: "CSAPP 读书笔记：存储系统的层级结构"
date: 2021-12-28T21:41:42+01:00
draft: false
tags: ["CSAPP","OS"]
summary: "存储系统（Memory System）可以分为多个层级（Hierarchy），每个层级的存储设备（Storage Device）具有不同的容量、成本和访问时间。层级越低，存储设备的容量就越大，成本也越低，但访问速度却越慢。存储系统的层次结构对应用性能有着显著的影响 ..."
---

存储系统（Memory System）可以分为多个层级（Hierarchy），每个层级的存储设备（Storage Device）具有不同的容量、成本和访问时间。层级越低，存储设备的容量就越大，成本也越低，但访问速度却越慢。存储系统的层次结构对应用性能有着显著的影响，而 CPU 与主存储器之间的高速缓存则尤为重要，因为它对程序性能的影响最大。

## 存储技术

### 随机存取存储器

随机存取存储器（Random Access Memory，RAM）有两种，分别是静态（static）的 SRAM 和 动态（dynamic）的 DRAM。其中，SRAM 的访问速度比 DRAM 更快，不过其成本明显更高。SRAM 用于 CPU 芯片内外的缓存，而 DRAM 则用于主存储器和图形系统中的帧缓存（Frame Buffer）。
