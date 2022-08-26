---
title: "【译】从零开始编写一个时序数据库"
date: 2021-11-21T00:25:25+08:00
draft: false
tags: ["Prometheus", "TSDB"]
summary: "Prometheus 是一个包含了自定义时间序列数据库的监控系统，其查询语言、操作模型以及一些概念性决策使得它易于与 Kubernetes 集成。然而，Kubernetes 集群中的工作负载是动态变化的，有可能给它带来一定的压力 ..."
---

> 原文链接：[Fabian Reinartz. Writing a Time Series Database from Scratch. fabxc.org, 2017.](https://fabxc.org/tsdb/)

[Prometheus](https://prometheus.io/) 是一个包含了自定义时间序列数据库的监控系统，其查询语言、操作模型以及一些概念性决策使得它易于与 [Kubernetes](https://kubernetes.io/) 集成。然而，Kubernetes 集群中的工作负载是动态变化的，有可能给它带来一定的压力。因此，我们致力于提高 Prometheus 在这些运行着高度动态或瞬态服务的环境中的性能。

过去，单台 Prometheus 服务器每秒能够拉取多达一百万个样本（Sample），并且只占用非常少的磁盘空间。虽然它的性能十分卓越，但仍有改进空间。因此我提出了一种全新的存储系统设计，它可以解决当前方案的痛点，让 Prometheus 具备处理更大规模数据的能力。

## 问题，问题和问题空间

首先，我们简要介绍 Prometheus 需要完成的任务及其引发的关键问题。对于每个方面，我们都会讨论当前方案做得好的地方，以及做得不好亟待新方案解决的地方。

### 时间序列数据

Prometheus 随时间不断采集数据点：

```shell
identifier -> (t0, v0), (t1, v1), (t2, v2), (t3, v3), ....
```

每个数据点都是一个由时间戳和数值组成的元组。其中，时间戳是一个整型，而数值则是 64 位浮点数。一系列带有严格单调递增时间戳的数据点被称为 Series，它可以由含有指标名称和标签字典的标识符（Identifier）来寻址。一组典型的 Series 标识符如下：

```shell
requests_total{path="/status", method="GET", instance=”10.0.0.1:80”}
requests_total{path="/status", method="POST", instance=”10.0.0.3:80”}
requests_total{path="/", method="GET", instance=”10.0.0.2:80”}
```

指标名称也可以被视为一个标签，如`_name_`，因此我们可以对这种表达方式进行简化。在查询时它可能会被特殊处理，但存储时则与其他标签无异。

```shell
{__name__="requests_total", path="/status", method="GET", instance=”10.0.0.1:80”}
{__name__="requests_total", path="/status", method="POST", instance=”10.0.0.3:80”}
{__name__="requests_total", path="/", method="GET", instance=”10.0.0.2:80”}
```

当查询时间序列数据时，我们通常会根据标签选择所需的 Series。一个最简单的例子，`{__name__="requests_total"}`会查询属于`requests_total`指标的所有 Series，Prometheus 将拉取指定时间窗口内的数据点。

有时候我们还希望一次查询能够选取满足多个标签选择器的 Series，或者在标签匹配中使用比等式更加复杂的条件。例如，否定`(method!="GET")`或正则表达式匹配`(method="PUT|POST")`。

本节介绍的内容在很大程度上定义了 Prometheus 存储和调用数据的方式。

### 垂直和水平

在简化的视图中，所有数据点都分布在一个二维平面上。其中，水平维度代表时间，垂直维度代表 Series 的标识符：

```shell
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

这些数据点是由 Prometheus 周期性地拉取一组 Series 的当前值而得到的。由于该操作对每个数据源实体（即 Target）均独立完成，因此 Prometheus 的写入模式是完全垂直且高度并发的。

假设我们的数据规模为：单个 Prometheus 实例从数万个 Target 中采集数据点，而每个 Target 又暴露成百上千个不同的 Series。在这种数据量达到百万级别的情况下，批量写入是必需的。

对于旋转磁盘来说，随机写入数据非常缓慢，因为它需要不断移动磁头来寻址。而对于 SSD，尽管其随机写操作很快，但是它只能以 4KiB 或更大的 Page 为单位写入。也就是说，写入一个 16 字节的样本其实相当于写入一个完整的 4KiB Page。这种现象属于写放大（[Write Amplification](https://en.wikipedia.org/wiki/Write_amplification)）的一部分，将会导致 SSD 的磨损和性能下降，甚至在几天或几周内摧毁你的磁盘。有关该问题的详细信息，可以参考系列文章 [Coding for SSDs](http://codecapsule.com/2014/02/12/coding-for-ssds-part-1-introduction-and-table-of-contents/)。综上所述，无论对于旋转磁盘还是 SSD，顺序写入和批量写入均是最理想的模式，这是我们必须坚持的原则。

查询模式则与写入模式完全不同。我们可以查询一个 Series 中的单个数据点，一万个 Series 中的单个数据点，一个 Series 中一周内的数据点以及一万个 Series 中一周内的数据点等等。在二维数据平面中，查询的数据点既不是完全垂直或完全水平的，而是两者的矩形组合。

我们可以使用 [Recording rules](https://prometheus.io/docs/practices/rules/) 来改善执行常用查询语句时遇到的性能问题，但它对临时性的查询并不起作用。

> 译者注：关于 Recording rules，除了原文给出的文档链接外，还可以参阅 [Today I Learned: Prometheus Recording Rules](https://deploy.live/blog/today-i-learned-prometheus-recording-rules/) 一文。

当 Prometheus 批量写入时，每个批次（Batch）的数据点分布在垂直方向上的多个 Series 中。而如果我们查询某个时间窗口内的某个 Series 的数据点时，不仅很难找出每个点在磁盘上的位置，还必须从磁盘上随机读取数据。每次查询可能涉及到数百万个样本，即使在最快的 SSD 上进行也很慢。虽然每次查询请求的样本大小可能只有 16 字节，但读取操作会从磁盘上检索更多的数据。对于 SSD 是一个 Page，而对于 HDD 则是整个扇区。无论如何，我们都在浪费宝贵的吞吐量。

因此在理想情况下，同一个 Series 中的样本按顺序存储。这样只要我们知道某个 Series 的起始位置，就可以快速访问它所有的数据点，从而减少读取操作的次数。

将数据写入磁盘的理想模式和能够显著提升查询效率的设计之间显然存在着强烈的矛盾，这是我们的时序数据库必须解决的根本问题。

> 译者注：原文为：“There’s obviously a strong **tension** between the ideal pattern for writing collected data to disk and the layout that would be significantly more efficient for serving queries. It is the fundamental problem our TSDB has to solve.” 译者不确定此处的 tension 该如何翻译，猜测原文作者可能是想表达一种类似 trade-off 的概念。因为上文提到，在理想的写入模式中，数据点是垂直分布的。而通常查询得到数据点却是水平，甚至是矩形的。

#### 当前解决方案

是时候看看 Prometheus 当前的存储系统（我们称之为 “V2”）是如何解决这一问题的。我们为每个 Series 创建一个文件，其中所有的样本均按时间顺序排列。由于每隔几秒就将样本追加写入到这些文件末尾的成本很高，我们先将样本缓存到内存中的 Chunk。每个 Series 都有一个对应的 Chunk，待 Chunk 被写满（即大小达到 1 KiB）之后再添加到文件尾部。这种方案既实现了批量写入，又将样本按顺序存储，解决了很多问题。一般来说，同一个 Series 中相邻样本的数值变化较小，因此可以使用非常高效的数据压缩格式。一篇关于 Gorilla TSDB 的 [论文](http://www.vldb.org/pvldb/vol8/p1816-teller.pdf) 介绍了一种类似基于 Chunk 的方法和压缩格式，能够将 16 字节的样本数据压缩到平均 1.37 字节。V2 版本的存储系统使用多种压缩格式，其中就包括 Gorilla 的变体。

![20211130203848](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20211130203848.png)

尽管基于 Chunk 的方法很棒，但为每个 Series 维护一个独立文件将给 V2 存储系统带来很多麻烦：

- 实际上我们所需的文件数量远比当前收集到的 Series 多得多，原因参见 [Series Churn](/posts/writing-a-time-series-database-from-scratch/#series-churn) 章节。上百万个文件迟早会耗尽我们文件系统上所有的 [inodes](https://en.wikipedia.org/wiki/Inode)，只能通过重新格式化磁盘来恢复。
- 即使我们引入了 Chunk，每秒也会有数千个 Chunk 被写满并准备持久化，这意味着每秒发生数千次独立的磁盘写入。虽然可以通过将一个 Series  中几个已完成的 Chunk 批量落盘来改善这一问题，但这样做反而增加了等待持久化的 Chunk 数量，因此会占用更多的内存。
- 为了读写而保持所有文件均处于打开状态是不可行的。尽管约 99%的数据在 24 小时后就不再被查询，但只要查询到已持久化的样本，就必须打开数千个文件然后将结果读入内存中，最后再关闭它们。因为这种操作极大地提高了查询的延时，Prometheus 将缓存更多的 Chunk ，从而导致 [资源消耗](/posts/writing-a-time-series-database-from-scratch/#资源消耗) 章节中提到的问题。
- 最后我们必须删除旧数据。它们存储在数百万个文件的头部中，因此删除其实是一种写入密集型操作。此外，遍历数百万个文件并对其进行分析通常需要数个小时，可能刚一完成就不得不重新开始。没错，删除旧文件还将进一步地导致 SSD 的写放大。
- 当前累积的 Chunk 均存储在内存中，如果 Prometheus 发生崩溃数据就会丢失。为了避免这一情况，内存状态将周期性地保存（Checkpoint）到磁盘中。然而，完成 Checkpoint 的所需时间可能远比我们能够接受的数据丢失时间窗口还长。同时恢复 Checkpoint 一般长达几分钟，使 Prometheus 的重启周期变得非常漫长。

现有设计的关键概念是 Chunk，我们当然希望保留它。将最新的 Chunk 始终保持在内存中也是一个很好的设计，毕竟近期的数据查询频率最高。不过，为每个 Series 都创建一个文件的方案看起来并不合理，我们希望能够找到新的方案代替它。

### Series Churn

在 Prometheus 中，Series Churn 表示一组 Series 变得不活跃。即新的数据点不再由它们接收，而是关联到一组新出现的 Series。例如，一个微服务暴露的所有 Series 都有一个对应的“实例”标签。如果我们对该微服务进行滚动更新并将每个实例替换为新版本，Series Churn 就会发生。在更加动态的环境中，这种现象甚至可能每小时就出现一次。集群编排系统（如 Kubernetes）允许应用连续地自动扩展和频繁地滚动更新，因此每天可能将有上万个实例以及相关的 Series 被创建。

```shell
series
  ^
  │   . . . . . .
  │   . . . . . .
  │   . . . . . .
  │               . . . . . . .
  │               . . . . . . .
  │               . . . . . . .
  │                             . . . . . .
  │                             . . . . . .
  │                                         . . . . .
  │                                         . . . . .
  │                                         . . . . .
  v
    <-------------------- time --------------------->
```

由于 Series Churn 的存在，即使整个基础架构的规模保持不变，Series 数量也会随时间线性增长。虽然 Prometheus 能够收集多达 1000 万个 Series 的数据，但要让它从十亿个 Series 中查找数据还是十分困难的。

#### 当前解决方案

V2 存储系统为当前存储的所有 Series 分配了一个基于 LevelDB 的索引。它允许查询包含给定标签对的 Series，但缺少一种可扩展的方式来对不同标签选择的结果进行组合。

例如，通过标签`__name__="requests_total"`选取所有的 Series 十分高效，但使用`instance="A" AND __name__="requests_total"`就会遇到扩展能力的问题。稍后我们将再次讨论导致这种现象的原因，以及想要改善查询性能所必须做出的调整。

这个问题实际上是我们寻找更好的存储系统的最初原因，Prometheus 需要一种更加完善的索引方法以从数亿个 Series 中快速搜索数据。

### 资源消耗

当尝试扩展 Prometheus（或其他任何东西）时，资源消耗是贯穿始终的主题之一。但是，真正困扰用户的并非是绝对的资源不足。实际上，Prometheus 通常能够满足用户所需的吞吐量，问题在于其面对变化时的不稳定性和不可预知性。V2 存储系统为样本数据缓慢地创建 Chunk，内存使用量随时间的推移而持续上升。待 Chunk 写满后，它们会被写入磁盘并从内存中驱逐。最终，Prometheus 的内存使用量将达到一个稳定的状态。但是一旦监控的环境发生变化——扩展应用或滚动更新时，Series Churn 就会增加内存、CPU 和磁盘的使用。

如果变化持续进行，资源消耗将再次达到一个稳定的状态，不过明显要比静态环境中的高。过渡周期通常长达数个小时，因此我们很难确定最大的资源使用量。

为每个 Series 维护一个文件的方案也会使单个查询操作很容易终止 Prometheus 的进程。当查询未被缓存的数据时，与之关联的 Series 文件需要被打开并将包含相关数据点的 Chunk 读入内存。如果数据量超过了内存配额，Prometheus 就会因 OOM 而相当不雅地退出。加载的数据可以在查询结束后释放，但为了后续能够更快地查询相同数据，通常要缓存更长的时间。

我们在上文中提到了 SSD 的写放大，以及 Prometheus 是如何通过批量写入来解决这一问题的。尽管如此，在一些场景中——如写入的批次太小或者数据没有与 Page 的边界精确对齐，写放大还是会产生。我们已经在一些规模较大的 Prometheus 服务器上观测到了硬件寿命缩短的现象，虽然这对写入吞吐量较高的数据库应用来说是正常的，但还是应当思考如何缓解它。

## 重新开始

到目前为止，我们已经对 Prometheus 需要解决的问题、V2 存储系统的设计方案及其缺点有了充分的了解。许多 V2 存储系统存在的不足可以通过一些优化和部分重新设计来改善，但为了让事情变得更有趣（当然也经过了仔细评估），我决定从零开始编写一个完整的时序数据库。

存储模式将直接影响到 Prometheus 的性能和资源消耗等关键问题，因此我们必须为数据找到正确的算法集和磁盘设计方案。这就是我能够免走弯路而直接找到解决方案的原因。

### V3——宏观设计

当我们在 Prometheus 的数据目录下使用`tree`命令时，就可以看到 V3 存储系统的宏观层级结构：

```shell
$ tree ./data
./data
├── b-000001
│   ├── chunks
│   │   ├── 000001
│   │   ├── 000002
│   │   └── 000003
│   ├── index
│   └── meta.json
├── b-000004
│   ├── chunks
│   │   └── 000001
│   ├── index
│   └── meta.json
├── b-000005
│   ├── chunks
│   │   └── 000001
│   ├── index
│   └── meta.json
└── b-000006
    ├── meta.json
    └── wal
        ├── 000001
        ├── 000002
        └── 000003
```

最上层目录是一系列经过编号的 Block，其前缀均为`b-`。每个 Block 中都有一个索引文件`index`和一个包含若干编号文件的`chunks`目录，该目录中保存了多个 Series 数据点的原始 Chunk。和 V2 存储系统一样，这种设计可以减少读取一个时间窗口内的 Series 数据的性能开销，并支持使用相同的高效压缩算法。基于 Chunk 的理念已经被证明是行之有效的，因此我们将沿用下去。不过，现在的存储系统不再是每个文件对应一个 Series，而是由几个文件保存多个 Series 的 Chunk。

索引文件`index`的存在不足为奇。让我们假设它拥有诸多黑魔法，可以用于查找标签及其可能的值、整个时间序列以及存储数据点的 Chunk。

但为什么要设计成若干个包含索引和多个 Chunk 文件的目录？为什么最后一个 Block 中还存在一个`wal`目录？只要理解了这两点，就能解决我们 90%的问题。

#### 多个小型数据库

我们将二维数据平面在水平维度（即时间）上划分为多个互不重叠的 Block，每个 Block 均表现为一个包含其时间窗口内所有 Series 数据的独立数据库。因此，Block 中存在自身的索引和多个 Chunk 文件。

![20211130204102](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20211130204102.png)

每个已持久化的 Block 都是不可改变的，但我们还需要一个可变的 Block（即上图中的“Mutable Block”）来接收新的样本数据。该 Block 是一个能够高效更新数据结构的内存数据库，并有着与已持久化的 Block 相同的查询特性。为了防止数据丢失，所有刚采集到的数据都会另外写入临时的预写日志（Write Ahead Log）中。它实际上就是`wal`目录中的一组文件，可以在 Prometheus 重启时恢复原有的内存数据库。

现在我们可以根据每个 Block 对应的时间范围将查询请求分发到各个 Block 中，最终的查询结果是由每个 Block 的返回值合并而成的。

这种设计的优势在于：

- 当查询某一时间范围内的数据时，我们可以轻易地忽略该时间范围外的所有 Block。随着查询开始时检索的数据集数量的减少， Series Churn 带来的问题迎刃而解；
- 当一个 Block 写满后，我们可以通过顺序写多个大文件的方式从内存数据库中持久化数据。这样 Prometheus 就避免了任何的写放大问题，并且能在 SSD 和 HDD 上表现得同样出色；
- 我们保留了 V2 存储系统中的优点，即查询最为频繁的 Chunk 始终保存在内存中；
- 我们不再需要为了数据对齐而限制 Chunk 的大小为固定的 1KiB，现在可以选择对于单个数据点和所选压缩格式最合适的任意大小；
- 删除旧数据的开销变得很小并且能够快速完成，因为我们只需要简单地删除一个目录。要知道在 V2 存储系统中，我们必须分析和重写多达数亿个文件，可能需要花费数个小时。

每个 Block 中还存有一个`meta.json`文件，它只保存一些与 Block 相关的可读信息，因此我们便可以轻松了解存储状态以及 Block 中包含的数据。

> 译者注：从 Prometheus v2.19.0 开始，Mutable Block 便不再全部存储在内存中，详见：[Prometheus TSDB (Part 1): The Head Block](https://ganeshvernekar.com/blog/prometheus-tsdb-the-head-block/)。

#### mmap

既然已持久化的数据从数百万个小文件变成了若干个大文件，那么我们就能够以很低的开销保持所有文件均处于打开状态。在这种情况下，我们可以引入系统调用 [mmap](https://en.wikipedia.org/wiki/Mmap)，将文件内容透明地映射到虚拟内存区域中。mmap 有些类似于交换分区（Swap Space），只不过我们所有的数据都已经在磁盘上了，在数据换出内存时不会发生写入。

这意味着我们可以把数据库中的所有内容都视为在内存中，而实际上却没有占用任何物理内存（RAM）。只有我们访问数据库文件中指定的字节范围时，操作系统才会从磁盘中懒加载 Page。我们让操作系统来负责所有已持久化数据的内存管理，因为它对整个机器和其中的进程有着全面的了解。为了处理查询请求，更多的数据将被缓存到内存中，但它们会在内存压力较大时被操作系统驱逐。如果机器中还有内存未被使用，Prometheus 甚至可以把整个数据库缓存到内存中。而一旦另一个应用需要使用这部分内存，Prometheus 便会立刻返还给它。

因此，查询比 RAM 容量更多的已持久化数据很容易导致进程的 OOM。内存中缓存的大小变得完全自适应，只有在需要响应查询请求时才会加载数据。

据我所知，如今大多数的数据库均采用这种设计。如果磁盘格式允许，这是一种理想的工作模式——除非有人有信心在管理进程方面胜过操作系统。

#### 压缩

存储系统必须定期“切割”出一个新的 Block，然后将前一个 Block 写入到磁盘中。只有当它成功持久化后，对应的预写日志文件才能被删除。

每个 Block 覆盖的时间范围不能太长（默认设置为两小时），否则将占用过多内存。但查询多个 Block 时，我们必须把每个返回结果合并到一起。这种操作显然会消耗性能，因此 Block 覆盖的时间范围也不能太短。一般来说，查询一周内数据所需要合并的 Block 数量不应超过 80 个。

为了同时实现这两点，我们对 Block 进行压缩，即将一个或多个 Block 中的数据写入到一个有可能更大的 Block 中。Prometheus 在压缩过程中还会修改现有数据，比如删除已被标记为即将删除的数据，或者为了提高查询性能而重新构建样本的 Chunk。

![20211130204202](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20211130204202.png)

在上图中，几个 Block 被顺序编号为 [1, 2, 3, 4]。Block 1、2 和 3 可以被压缩到一起，变成 [1, 4]。也可以将它们两两压缩，变成 [1, 3]。所有时间序列数据仍然存在，但现在 Block 的总数减少了。这样查询时需要被合并的返回结果就更少，从而显着地降低了合并操作的开销。

#### 保留

在 V2 存储系统中，删除旧数据是一个非常缓慢的过程，并且会花费大量的 CPU、内存和磁盘资源。而现在，我们只需要删除那些在指定保留时间窗口内没有数据的 Block 目录即可。下方示例中，Block 1 可以被安全地删除，Block 2 则必须等到它完全超出保留边界（Retention Boundary）后才能被删除：

![20211202203935](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20211202203935.png)

随着旧数据的不断积累，压缩后的 Block 会变得越来越大。我们必须为其最大值设置上限，否则它将增长到包含整个数据库，从而很难被删除。对于像上图中 Block 2 这样的跨越保留边界的 Block，这种做法也限制了维护它们所需的磁盘开销。当 Block 的最大尺寸被设为保留时间窗口的 10%时，维护 Block 2 的成本也会受到同样的限制。

总之，删除旧数据的开销从非常昂贵变成了几乎免费。

>如果你已经读到这里并掌握一些数据库知识，可能会有这样的疑问：“这些设计是全新的吗？”——实际上并非如此，甚至可能做得更好。
>
>在内存中批量处理数据、在预写日志中记录操作以及周期性地将数据落盘的设计模式在如今无处不在，使用这种方案的知名开源项目有 LevelDB、Cassandra、InfluxDB 以及 HBase 等。关键在于不要发明劣质的轮子，而是对已被证明有效的方法深入研究，并以正确的方式应用它们。
>
>当然，我们还有机会在 Prometheus 中加入自己的“魔法”。

### 索引

我们改进存储系统的初衷是希望解决 Series Churn 带来的问题，而基于 Block 的设计则减少了查询时涉及的 Series 数量（设为 n）。然而对于时间复杂度为 $O(n^2)$ 的查询操作，减少 n 的数量没有什么意义。如果以前查询性能很差，那么现在依然会很糟糕。

实际上，大多数查询操作的响应都很快。可一旦时间跨度较大，即使只查询几个 Series 也会很慢。在开展所有工作前，我最初的想法便是为这个问题找到一个解决方案。我们需要一个更加强大的 [倒排索引](https://en.wikipedia.org/wiki/Inverted_index)。

倒排索引可以基于数据内容的子集来为我们提供快速查找的能力。简而言之，我们可以在不遍历全部 Series 的情况下，找到所有包含标签`app="nginx"`的 Series。

为此，我们需要为每个 Series 分配一个独有的 ID。它可以在常量时间，即 $O(1)$ 内被检索出来。在这种情况下，ID 是我们的正排索引（Forward Index）。

>**示例**：如果包含标签`app="nginx"`的 Series ID 为 10、29 和 10，那么标签“nginx”的倒排索引就是一个简单的数组 [10, 29, 9]，我们可以用它来快速检索所有包含该标签的 Series。即使还有其他 200 亿个 Series，也不会影响这次查询的速度。

简单来说，如果 n 是 Series 总数，m 是给定查询结果的大小，那么查询的时间复杂度就从 $O(n)$ 变成了 $O(m)$。通常 m 要比 n 小很多，因此查询性能将显著提升。

实际上，这一设计与 V2 存储系统中所采用的倒排索引几乎相同，也是在数百万个 Series 中实现高性能查询的最低要求。敏锐的读者可能已经发现，在最坏的情况下，一个标签存在于所有的 Series 中，那么复杂度又变成了 $O(n)$。其实这是合理的，因为如果查询涉及到全部数据，自然需要更长的时间。不过，一旦我们使用更加复杂的查询语句，就会面临一些新的问题。

#### 标签的组合

一个标签与数百万个 Series 关联是很常见的。假设一个微服务"foo"水平扩展为数百个实例，其中每个实例又有数千个包含标签`app="foo"`的 Series。通常我们不会查询所有的 Series，而是使用多个标签的组合来对返回结果进行一定的限制。例如，我们可以通过`__name__="requests_total" AND app="foo"`来获取服务实例接收的请求数。

为了找到满足两个标签选择器的所有 Series，我们需要计算两个标签对应的倒排索引数组的交集，其结果通常比单独的倒排索引数组小几个数量级。在最坏的情况下，每个倒排索引数组的长度均为 n，那么在两个数组中通过嵌套迭代取交集的时间复杂度就为 $O(n^2)$。其他类似的查询操作，如`app="foo" OR app="bar"`，也会花费同样的开销。当我们向查询语句中添加更多的标签选择器时，时间复杂度会呈指数增长：$O(n^3)$、$O(n^4)$、$O(n^5)$ ... $O(n^k)$。

幸运的是，只需要一个小小的改动就可以解决我们的问题。如果倒排索引数组中的 ID 都已被排序，那么会发生什么呢？

```shell
__name__="requests_total"   ->   [ 999, 1000, 1001, 2000000, 2000001, 2000002, 2000003 ]
     app="foo"              ->   [ 1, 3, 10, 11, 12, 100, 311, 320, 1000, 1001, 10002 ]

             intersection   =>   [ 1000, 1001 ]
```

我们在每个数组的起始元素处放置一个游标，其中数字较小的会不断前进。当两个游标对应的数字相等时，我们就把它添加到结果中并同时推进两个游标。由于游标只会在其所在的数组中移动，因此其遍历全部数组的时间复杂度为 $O(2n) = O(n)$。

对两个以上的倒排索引数组取交集的过程与之类似。当标签增长到 k 个时，时间复杂度只会变为 $O(n * k)$，而不是 $O(n^k)$。这是一个非常大的改进。

本文介绍的是典型搜索索引的一个简化版本，几乎所有的 [全文搜索引擎](https://en.wikipedia.org/wiki/Search_engine_indexing#Inverted_indices) 都在使用它。每个 Series 的标识符都被视为一个简短的“document”，而每个标签（名称加上固定的值）则是其中的一个“word”。我们可以忽略一些搜索引擎中常用的的索引附加数据，比如 word 的位置和频率。

实际上还有很多技术可以对倒排索引进行压缩，但它们各有优缺点。考虑到我们的 document 很小，并且 word 在各个 Series 中的重复率很高，因此压缩其实无关紧要。例如，一个真实世界的数据集中大约有 440 万个 Series，每个 Series 大约有 12 个标签，但其中唯一的标签却不超过 5000 个。最初版本的存储系统没有采用压缩，只是做了一些简单优化以跳过大范围没有交集的 ID。

让 ID 始终按顺序排列并非看起来那么简单。例如，V2 存储系统将哈希值作为 ID 分配给新的 Series，这样我们就无法有效地对倒排索引进行排序。

另一个艰巨的任务是在数据被删除或更新时修改索引。通常最简单的方法就是在保证数据库可查询且一致的同时，重新计算并重写它们。V3 存储系统为每个 Block 都分配了一个独立的索引。对于已持久化的 Block，其索引只有在压缩时才会被重写。而对于内存中可变的 Block，其索引则需要被持续更新。

## 基准测试

我们使用 [测试工具](https://github.com/prometheus/test-infra) 将 Prometheus 部署在 AWS 上的 Kubernetes 集群中，其中包含两个 1.5.2 版本（V2 存储系统）和两个 2.0 版本（V3 存储系统）的实例。为了模拟 Series Churn，微服务会定期移除旧的 Pod 并创建新的 Pod 以生成更多新的 Series。服务扩展频率和查询负载远远超过了如今生产环境中的真实情况，因此可以确保 V3 存储系统能够应对未来的数据规模。例如，在我们的测试环境中微服务每隔 15 分钟就要更换自身 60% 的实例。而在实际的生产环境中，这种情况每天只会发生一到五次。Prometheus 每秒从 850 个 Target 中采集约 110000 个样本，每次涉及多达 50 万个 Series。基准测试的结果如下：

![20211205200953](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20211205200953.png)

$$Heap \space memory \space usage \space in \space GB$$

我们可以发现被查询的 Prometheus 实例消耗更多的内存，并且 2.0 版本的堆内存使用量比 1.5 版本减少了三到四倍之多。在测试开始后 6 小时左右，1.5 版本的 Prometheus 实例达到了峰值。这是因为我们将数据的保留期限设置为了 6 小时，而 V2 存储系统中删除数据的巨大开销导致了资源消耗的上升。

![20211205203043](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20211205203043.png)

$$CPU \space usage \space in \space cores/second$$

CPU 使用率的状况与内存类似，只不过查询操作对其的影响更大。总体来看，新的存储系统中 CPU 的使用率比原来减少了三到十倍。

![20211205203453](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20211205203453.png)

$$Disk \space writes \space in \space MB/second$$

我们可以通过这张图清楚地看到 Prometheus 1.5 是如何导致 SSD 磨损的。每当一个 Chunk 被持久化到 Series 对应的文件中，或删除旧数据并重写文件时，磁盘的写入速率就会大幅上升。而 Prometheus 2.0 每秒只会向预写日志中写入约 1MB 字节，写入速率只有在 Block 被持久化时才会出现峰值。新的设计方案成功地减少了约 97%～99%的磁盘写入。

![20211205205254](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20211205205254.png)

$$Disk \space size \space in \space GB$$

虽然两个版本的 Prometheus 使用的压缩算法近乎相同，但 Series Churn 的存在导致了两者占用磁盘空间的巨大差异。

![20211205210102](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20211205210102.png)

$$99th \space percentile \space query \space latency \space in \space seconds$$

在 Prometheus 1.5 中，查询延迟随着 Series 数量的不断上升越来越高。当数据达到保留期限并开始删除旧的 Series 时，查询延迟便又会趋于平稳。相比之下，Prometheus 2.0 的查询延迟从一开始就保持不变。

![20211205210730](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20211205210730.png)

$$Ingested \space samples/second$$

两个 Prometheus 2.0 实例的样本采集率完全吻合，并在数小时后开始变得不稳定。这并非是 Prometheus 自身的问题，而是集群中节点的负载过高而导致的。对于 Prometheus 1.5，即使节点仍有可用的 CPU 和内存资源，它的样本采集率也会因 Series Churn 而大大降低。

基准测试的结果表明，Prometheus 2.0 在云服务器上的表现远远超出了最初设计时的预期。不过其成功与否还是要取决于用户的反馈，而非基准测试中的数字。

## 总结

对于 Prometheus 来说，处理高基数（High Cardinality）的 Series 和独立样本的吞吐量是一项颇为艰巨的任务。不过，新的存储系统似乎已经准备好应对未来的挑战。

使用 V3 存储系统的 Prometheus 2.0 的第一个 [Alpha 版本](https://prometheus.io/blog/2017/04/10/promehteus-20-sneak-peak/) 已经可供测试，预计会出现一些崩溃、死锁和其他 Bug。

存储系统自身的代码可以在一个独立的 [项目](https://github.com/prometheus-junkyard/tsdb) 中找到。它其实与 Prometheus 无关，因此可以广泛用于其他需要高效本地存储的时序数据库应用中。

> 译者注：上述项目已于 2019 年合并到 Prometheus 主仓库中，原因详见：[https://github.com/prometheus/prometheus/pull/5805](https://github.com/prometheus/prometheus/pull/5805)

> 译者注：本文从宏观的角度介绍了 Prometheus 需要解决的问题，以及 1.X 版本（V2 存储系统）和 2.X 版本（V3 存储系统）的设计理念。想要了解其实现细节，除了阅读源码外还可以参考以下内容：
>
> - 关于 Promtheus 中的内存数据库：[Prometheus TSDB (Part 1): The Head Block](https://ganeshvernekar.com/blog/prometheus-tsdb-the-head-block/)；
>
> - 关于预写日志和 Checkpoint：[Prometheus TSDB (Part 2): WAL and Checkpoint](https://ganeshvernekar.com/blog/prometheus-tsdb-wal-and-checkpoint/)；[Write-Ahead Log](https://martinfowler.com/articles/patterns-of-distributed-systems/wal.html)；
>
> - 关于 mmap：[Prometheus TSDB (Part 3): Memory Mapping of Head Chunks from Disk](https://ganeshvernekar.com/blog/prometheus-tsdb-mmapping-head-chunks-from-disk/)；[Why mmap is faster than system calls](https://sasha-f.medium.com/why-mmap-is-faster-than-system-calls-24718e75ab37)；
>
> - 关于索引：[Prometheus TSDB (Part 4): Persistent Block and its Index](https://ganeshvernekar.com/blog/prometheus-tsdb-persistent-block-and-its-index/)；
>
> - 关于查询：[Prometheus TSDB (Part 5): Queries](https://ganeshvernekar.com/blog/prometheus-tsdb-queries/)；
>
> - 关于压缩和保留：[Prometheus TSDB (Part 6): Compaction and Retention](https://ganeshvernekar.com/blog/prometheus-tsdb-compaction-and-retention/)；[Time-series compression algorithms, explained](https://blog.timescale.com/blog/time-series-compression-algorithms-explained/)；
>
> - 一些视频：[PromCon 2017: Storing 16 Bytes at Scale - Fabian Reinartz](https://www.youtube.com/watch?v=b_pEevMAC3I)；[技术分享：Prometheus 是怎么存储数据的（陈皓）](https://www.youtube.com/watch?v=qB40kqhTyYM&t=2455s)；
