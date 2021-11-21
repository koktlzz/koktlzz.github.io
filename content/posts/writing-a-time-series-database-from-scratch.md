---
title: "【译】从零开始写一个时序数据库"
date: 2021-11-21T00:25:25+08:00
draft: false
tags: ["Prometheus", "TSDB"]
summary: "Prometheus 是一个包含了自定义时间序列数据库的监控系统，其查询语言、操作模型以及一些概念性决策使得它易于与 Kubernetes 集成。然而，Kubernetes 集群中的工作负载是动态变化的，有可能给它带来一定的压力。"
---

> 原文链接：[Writing a Time Series Database from Scratch | Fabian Reinartz (fabxc.org)](https://fabxc.org/tsdb/)

[Prometheus](https://prometheus.io/) 是一个包含了自定义时间序列数据库的监控系统，其查询语言、操作模型以及一些概念性决策使得它易于与 [Kubernetes](https://kubernetes.io/) 集成。然而，Kubernetes 集群中的工作负载是动态变化的，有可能给它带来一定的压力。因此，我们致力于提高 Prometheus 在这些运行着高度动态或瞬态服务的环境中的性能。

过去，单台 Prometheus 服务器能够每秒拉取多达一百万个样本（Sample），并且只占用非常少的磁盘空间。虽然它的性能十分卓越，但仍有改进空间。我提出了一种全新的存储系统设计，它可以解决当前方案的痛点，让 Prometheus 具备处理更大规模数据的能力。

## 问题，问题和问题空间

首先，我们简要介绍 Prometheus 需要实现的目标及其引发的关键问题。对于每个方面，我们都会讨论当前方案做得好的地方，以及做得不好亟待新方案解决的地方。

### 时间序列数据

Prometheus 会随时间不断采集数据点：

```
identifier -> (t0, v0), (t1, v1), (t2, v2), (t3, v3), ....
```

每个数据点都是一个由时间戳和数值组成的元组。其中，时间戳是一个整型，而数值则是 64 位浮点数。一系列带有严格单调递增时间戳的数据点称为 Series，它可以由含有指标名称和标签字典的标识符（identifier）来寻址。一组典型的 Series 标识符如下：

```
requests_total{path="/status", method="GET", instance=”10.0.0.1:80”}
requests_total{path="/status", method="POST", instance=”10.0.0.3:80”}
requests_total{path="/", method="GET", instance=”10.0.0.2:80”}
```

指标名称也可以被视为一个标签，如`_name_`，因此我们可以对这种表达方式进行简化。在查询数据时它可能会被特殊处理，但这与 Prometheus 的存储系统无关。

```
{__name__="requests_total", path="/status", method="GET", instance=”10.0.0.1:80”}
{__name__="requests_total", path="/status", method="POST", instance=”10.0.0.3:80”}
{__name__="requests_total", path="/", method="GET", instance=”10.0.0.2:80”}
```

在查询时间序列数据时，我们通常根据标签选择所需的 Series。一个最简单的例子，`{__name__="requests_total"}`会查询属于`requests_total`指标的所有 Series，Prometheus 将拉取指定时间窗口内的数据点。

有时我们希望一次查询就能选取匹配多个标签的 Series，这需要使用比等式更加复杂的条件。例如，否定 `(method!="GET")` 或正则表达式匹配`(method="PUT|POST")`。

本节介绍的数据结构在很大程度上定义了 Prometheus 存储和调用数据的方式。

### 垂直和水平

在简化的视图中，所有数据点都分布在一个二维平面上。其中，水平维度代表时间，垂直维度代表 Series 的标识符：

```
series
  ^   
  │   . . . . . . . . . . . . . . . . .   . . . . .   {__name__="request_total", method="GET"}
  │     . . . . . . . . . . . . . . . . . . . . . .   {__name__="request_total", method="POST"}
  │         . . . . . . .
  │       . . .     . . . . . . . . . . . . . . . .                  ... 
  │     . . . . . . . . . . . . . . . . .   . . . .   
  │     . . . . . . . . . .   . . . . . . . . . . .   {__name__="errors_total", method="POST"}
  │           . . .   . . . . . . . . .   . . . . .   {__name__="errors_total", method="GET"}
  │         . . . . . . . . .       . . . . .
  │       . . .     . . . . . . . . . . . . . . . .                  ... 
  │     . . . . . . . . . . . . . . . .   . . . . 
  v
    <-------------------- time --------------------->
```

这些数据点是由 Prometheus 周期性地拉取一组时间序列的当前值而得到的。由于该操作对每个数据源实体（称为 Target）均独立完成，因此 Prometheus 的写入模式是完全垂直且高并发的。

假设我们的数据规模为：单个 Prometheus 实例从数万个 Target 中采集数据点，而每个 Target 又暴露成百上千个不同的时间序列。在这种数据量达到百万级别的情况下，批量写入是必需的。

对于旋转磁盘来说，随机写入数据是非常缓慢的，因为它需要不断移动磁头来寻址。而对于 SSD，尽管其随机写操作很快，但是它只能以 4KiB 或更大的 Page 为单位写入，意味着写入一个 16 字节的样本相当于写入一个完整的 4KiB Page。这种现象属于写放大（[write amplification](https://en.wikipedia.org/wiki/Write_amplification)）的一部分，将会导致 SSD 的磨损和性能下降，甚至在几天或几周内摧毁您的磁盘。有关该问题的详细信息，可以参考系列文章 [Coding for SSDs](http://codecapsule.com/2014/02/12/coding-for-ssds-part-1-introduction-and-table-of-contents/)。综上所述，无论对于旋转磁盘还是 SSD，顺序写入和批量写入均是最理想的模式，这是我们必须坚持的原则。

查询模式则与写入模式完全不同。我们可以查询一个 Series 中的单个数据点，一万个 Series 中的单个数据点，一个 Series 中一周内的数据点以及一万个 Series 中一周内的数据点等等。在二维数据平面中，查询的数据点既不是完全垂直或完全水平的，而是两者的矩形组合。

我们可以使用 [Recording rules](https://prometheus.io/docs/practices/rules/) 来缓解一些执行常用查询语句时遇到的性能问题，但它对临时性的查询语句并不起作用。

> 译者注：关于 Recording rules，除了原文给出的文档链接外，还可以参阅 [Today I Learned: Prometheus Recording Rules](https://deploy.live/blog/today-i-learned-prometheus-recording-rules/) 一文。

当 Prometheus 批量写入时，每个批次（batch）的数据点分布在垂直方向上的多个 Series 中。而如果我们查询某个时间窗口内的某个 Series 的数据点时，不仅很难找出每个点在磁盘上的位置，还必须从磁盘上随机读取数据。每次查询可能会涉及到数百万个样本，即使在最快的 SSD 上进行也很慢。虽然每次查询请求的样本大小可能只有 16 字节，但读取操作会从磁盘上检索更多的数据。对于 SSD 是一个 Page，而对于 HDD 则是整个扇区。无论如何，我们都在浪费宝贵的吞吐量。

因此在理想情况下，同一个 Series 中的样本按顺序存储。这样只要我们知道某个 Series 的起始位置，就可以快速访问它所有的数据点，从而减少读取操作的次数。

将收集到的数据写入磁盘的理想模式和能够显著提升查询效率的设计之间显然存在着强烈的矛盾，这是我们的时间序列数据库必须解决的根本问题。

> 译者注：原文为：“There’s obviously a strong **tension** between the ideal pattern for writing collected data to disk and the layout that would be significantly more efficient for serving queries. It is the fundamental problem our TSDB has to solve.” 译者不确定此处的 tension 该如何翻译，猜测可能是想表达一种类似 trade-off 的概念。因为上文提到，在理想的写入模式中，数据点是垂直分布的。而通常查询得到数据点却是水平，甚至是矩形的。

#### 当前解决方案

是时候看看 Prometheus 当前的存储系统（我们称之为 “V2”）是如何解决这一问题的。我们为每个 Series 创建一个文件，其中所有的样本均按时间顺序排列。由于每隔几秒就将样本追加写入到这些文件末尾的成本很高，我们先将样本缓存到内存中的 Chunk。每个 Series 都有一个对应的 Chunk，待 Chunk（大小为 1 KiB）被写满之后再添加到文件尾部。这种方案既实现了批量写入，又将样本按顺序存储，解决了很多问题。一般来说，同一个 Series 中相邻样本的数值变化较小，因此可以使用非常高效的数据压缩格式。一篇关于 Gorilla TSDB 的 [论文](http://www.vldb.org/pvldb/vol8/p1816-teller.pdf) 介绍了一种类似基于 Chunk 的方法和压缩格式，能够将 16 字节的样本数据压缩到平均 1.37 字节。V2 版本的存储系统使用多种压缩格式，其中就包括 Gorilla 的变体。

```
   ┌──────────┬─────────┬─────────┬─────────┬─────────┐           series A
   └──────────┴─────────┴─────────┴─────────┴─────────┘
          ┌──────────┬─────────┬─────────┬─────────┬─────────┐    series B
          └──────────┴─────────┴─────────┴─────────┴─────────┘ 
                              . . .
 ┌──────────┬─────────┬─────────┬─────────┬─────────┬─────────┐   series XYZ
 └──────────┴─────────┴─────────┴─────────┴─────────┴─────────┘ 
   chunk 1    chunk 2   chunk 3     ...
```

尽管基于 Chunk 的方法很棒，但为每个 Series 维护一个独立文件会给 V2 存储系统带来很多麻烦：

- 实际上我们所需的文件数量远比当前收集到的时间序列多得多，原因参见 [Series Churn](/posts/writing-a-time-series-database-from-scratch/#series-churn) 章节。上百万个文件迟早会耗尽我们文件系统上所有的 [inodes](https://en.wikipedia.org/wiki/Inode)，只能通过重新格式化磁盘来恢复。
- 即使我们引入了 Chunk，每秒也会有数千个 Chunk 被写满并准备持久化，这意味着每秒数千次独立的磁盘写入。虽然可以通过将一个 Series  中几个已完成的 Chunk 批量落盘来缓解这一问题，但这样做反而会增加等待持久化的 Chunk 数量，从而占用更多的内存。
- 为了读写而保持所有文件均处于打开状态是不可行的。虽然约 99%的数据在 24 小时后就不再被查询，但只要查询到已持久化的样本，就必须打开数千个文件然后将结果读入内存中，最后再关闭它们。为了解决查询的延时问题，Chunk 会被更加积极地缓存，这将导致 [资源消耗](/posts/writing-a-time-series-database-from-scratch/#资源消耗) 章节中提到的问题。
- 最后我们必须删除旧数据，它们存储在数百万个文件的头部中，这表明删除实际上是写入密集型操作。此外，遍历数百万个文件并对其进行分析通常需要数个小时，可能刚一完成就得重新开始。没错，删除旧文件还会进一步地导致 SSD 的写放大。
- 当前积累的 Chunk 均存储在内存中，如果 Prometheus 发生崩溃数据就会丢失。为了避免这一情况，内存状态会通过 Checkpoint 周期性地保存在磁盘中，其完成时间可能远比我们能够接受的数据丢失窗口还长。恢复 Checkpoint 也可能需要几分钟，导致漫长而又痛苦的重启周期。

现有设计的关键是 Chunk 的概念，我们当然希望保留它。最新的 Chunk 始终保持在内存中通常很好，毕竟最近的数据查询得最多。不过，我们希望能够有新的方案来替代现有每个 Series 一个文件的方案。

### Series Churn

#### 当前解决方案

### 资源消耗