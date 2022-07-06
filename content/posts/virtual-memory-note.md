---
title: "CSAPP 读书笔记：虚拟内存"
date: 2022-05-10T09:19:42+01:00
draft: false
series: ["CSAPP 读书笔记"]
tags: ["OS"]
summary: "为了更加有效地管理内存并减少错误的发生，现代系统提供了一种对主存储器的抽象，即虚拟内存（Virtual Memory，VM）。虚拟内存是硬件异常、硬件地址转换、主存储器、磁盘文件和内核软件之间的优雅交互，它为每个进程提供了一个大的、统一的和私有的地址空间 ..."
---

为了更加有效地管理内存并减少错误的发生，现代系统提供了一种对主存储器的抽象，即虚拟内存（Virtual Memory，VM）。虚拟内存是硬件异常、硬件地址转换、主存储器、磁盘文件和内核软件之间的优雅交互，它为每个进程提供了一个大的、统一的和私有的地址空间。

虚拟内存有以下三个重要功能：

- 将主存储器作为磁盘的缓存，只保留主存中的活跃区域并根据需要不断地在两者之间传输数据；
- 为每个进程提供统一的地址空间，从而简化内存管理；
- 保护每个进程的地址空间不被其他进程所破坏。

## 物理寻址与虚拟寻址

主存中的每个字节都有一个唯一的物理地址（Physical Address，PA），CPU 使用物理地址访问内存的方式被称为物理寻址（Physical Addressing）：

![20220608145110](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220608145110.png)

如上图所示，CPU 生成了一个有效的物理地址并通过存储器总线传递给主存储器，详见 [访问主存储器](/posts/the-memory-hierarchy-note/#访问主存储器)。

CPU 也可以通过虚拟地址（Virtual Address，VA）访问主存，只不过该地址在发送到主存之前需要被转换为适当的物理地址。这种寻址方式被称为虚拟寻址（Virtual Addressing）：

![20220608150546](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220608150546.png)

地址转换（Address Translation ）需要 CPU 硬件和操作系统之间密切合作，位于 CPU 芯片上的内存管理单元（Memory Management Unit，MMU）根据 [页表](/posts/virtual-memory-note/#页表)（Page Table) 动态地将虚拟地址转换为物理地址。页表存储在主存中，其内容由操作系统维护。

## 地址空间

地址空间（Address Space）是一组有序的非负整数地址：

$$\lbrace0, 1, 2, . . .\rbrace$$

如果这些整数是连续的，我们就称其为线性地址空间（Linear Address Space）。在拥有虚拟内存的系统中，CPU 从 n 位线性地址空间中生成虚拟地址，该虚拟地址空间共有 N = $2^n$ 个地址：

$$\lbrace0,1,2,...,N -1\rbrace$$

现代系统通常支持 32 位或 64 位虚拟地址空间。同样地，系统也有一个物理地址空间：

$$\lbrace0,1,2,...,M -1\rbrace$$

与虚拟地址空间不同，M 不一定为 2 的幂。但为了简化讨论，我们假设 M = $2^m$。

地址空间明确地将数据对象（字节）和其属性（地址）区分开来，因此每个数据对象都可以有多个独立的地址，这便是虚拟内存的基本思想。主存中的每个字节都有一个从物理地址空间中选择的物理地址，以及一个从虚拟地址空间中选择的虚拟地址。

## 虚拟内存作为缓存的工具

与存储层级结构中的其他缓存一样，磁盘与主存之间以 [Block](/posts/the-memory-hierarchy-note/#存储系统层级) 为单位传输数据。而在虚拟内存系统中，Block 被称为虚拟页面（Virtual Page，VP），其大小为 P = $2^p$。类似地，物理内存也可以被划分为多个大小为 P 的物理页面（Physical Page，PP)。

虚拟页面有以下三种状态：

- 未分配（Unallocated）：没有被进程申请使用的页面，不占用任何磁盘空间；
- 未缓存（Uncached）：仅加载到磁盘而未缓存到主存中的页面；
- 已缓存（Cached）：已缓存在主存中的页面。

### DRAM 缓存

我们将 CPU 和主存之间的 L1、L2 和 L3 级缓存称为 SRAM 缓存，而将主存中用来缓存虚拟页面的缓存称为 DRAM 缓存。

与 SRAM 缓存相比，DRAM 缓存发生缓存缺失的成本很高（需要从磁盘中加载数据），因此虚拟页面往往比较大——通常为 4 KB 到 2 MB。DRAM 缓存是 [全关联型](/posts/the-memory-hierarchy-note/#全关联型缓存) 的，这样任何一个虚拟页面都可以放在任何一个物理页面中。

> Due to the large miss penalty, DRAM caches are fully associative; that is, any virtual page can be placed in any physical page.

### 页表

页表是由页表条目（Page Table Entries，PTEs）组成的数组。每个虚拟页面在页表都有一条 PTE，它在页表中的偏移量是固定的。每条 PTE 中都包含了一个表示该虚拟页面是否已缓存的有效位，以及一个 n 位的地址字段。若有效位为 1，则地址字段是缓存该页面的物理页面的起始地址；若有效位为 0 且地址字段为空，则代表该虚拟页面未分配；若有效位为 0 且地址字段非空，则地址字段是该页面在磁盘上的起始地址。

![20220608215707](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220608215707.png)

如上图所示，系统中存在 8 个虚拟页面和 4 个物理页面：页面 1，2，4 和 7 缓存在 DRAM 中；页面 3 和 6 已分配但未缓存；页面 0 和 5 未分配。

### 缺页故障

我们将访问 DRAM 缓存时发生的缓存缺失称为缺页故障（Page Fault）。若 CPU 引用上图页面 3 中的某个字，地址转换硬件会从主存中读取 PTE 3 ，然后根据其有效位判断出该页面未缓存。缺页故障异常将调用内核中的异常处理程序，选择受害者页面（Victim Page）逐出主存。

在本例中，受害者页面为 VP 4，因此内核会先将 PTE 4 中的有效位重置为 0。如果该页面已发生更改，内核还需要将其复制回磁盘。接下来，内核把 VP 3 从磁盘复制到主存中的 PP 3，并更新 PTE 3 中的有效位。下图展示了异常处理程序返回后示例页表的状态：

![20220609113920](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220609113920.png)

上述在磁盘和主存之间传输页面的活动被称为交换（Swapping）或者分页（Paging）。页面从磁盘传输到主存被称为换入（Swapping In 或 Paging In），反之则为换出（Swapping Out 或 Paging Out）。尽管我们可以试图预测缺页故障并在引用未缓存的页面前分页，但现代系统均在异常发生后分页，即按需分页（Demand Paging）。

### 分配页面

![20220609152723](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220609152723.png)

上图展示了分配一个新的虚拟页面（如进程调用`malloc`）后页表的变化。操作系统先在磁盘上开辟空间，然后更新 PTE 5 使 VP 5 指向磁盘上新创建的页面。

## 虚拟内存作为内存管理的工具

操作系统为每个进程都维护了一个单独的页表，因此所有进程都拥有自己的虚拟地址空间：

![20220609155208](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220609155208.png)

如上图所示，进程 i 的页表将 VP 1 映射到 PP 2，将 VP 2 映射到 PP 7；进程 j 的页表将 VP 1 映射到 PP 7，将 VP 2 映射到 PP 10。多个虚拟页面可以映射到同一个共享物理页面。虚拟内存系统简化了链接、加载、代码和数据的共享以及应用程序的内存分配：

- 简化链接：独立的虚拟地址空间允许每个进程使用相同的 [内存格式](/posts/linking-note/#加载可执行目标文件)，因此链接器无需考虑可执行文件的代码和数据在物理内存中的实际位置。这种统一性极大地简化了链接器的设计和实现；
- 简化加载：若要将目标文件的 .text 和 .data 段加载到一个新进程的地址空间中，加载器只需为它们分配虚拟页面，然后将其标记为未缓存，最后再将页表条目指向目标文件中对应的位置。实际上加载器从未将任何数据从磁盘复制到主存中，代码和数据只有在被第一次引用时才会按需分页；
- 简化共享：操作系统可以将不同进程中的不同虚拟页面映射到相同的物理页面，从而实现进程之间代码和数据的共享；
- 简化内存分配：当应用程序申请额外的内存时，操作系统会为其分配一定数量的连续虚拟页面，然后将它们映射到任意位置的物理页面。这些物理页面无需连续，并且可以在物理内存中随机分布。

## 虚拟内存作为内存保护的工具

我们可以在 PTE 中添加一些权限位来管理进程对页面的访问：

![20220609164306](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220609164306.png)

如上图所示，SUP 表示是否只有在内核态运行的进程才能访问该页面，READ 和 WRITE 则分别表示页面是否可读写。例如，进程 i 在用户态中运行，那么它可以读取 VP 0，读取和写入 VP 1，但无法访问 VP 2。

如果某条指令违反了上述权限，CPU 就会触发通用保护故障，并将控制权转移到内核中的异常处理程序。该处理程序会向问题进程发送一个 SIGSEGV 信号，Linux Shell 通常将此异常报告为分段故障（Segmentation Fault）。

## 地址转换

![20220609223526](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220609223526.png)

CPU 中的页表基址寄存器（Page Table Base Register ，PTBR）指向当前页表，n 位的虚拟地址由 p 位的虚拟页面偏移量（Virtual Page Offset，VPO）和 n-p 位的虚拟页面编号（Virtual Page Number，VPN）组成。MMU 根据 VPN 的值选择对应的 PTE，如 VPN 0 选择 PTE 0，VPN 1 选择 PTE 1。由于物理页面和虚拟页面的大小相同，VPO 与物理页面偏移量（Physical Page Offset，PPO）也就相同，因此页表条目中的物理页面编号（Physical Page Number，PPN）与 VPO 共同组成了转换后的物理地址。

![20220609230216](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220609230216.png)

上图展示了页面命中时的 CPU 硬件操作步骤：

1. 处理器生成一个虚拟地址 VA 并发送到 MMU；
2. MMU 生成 PTE 地址 PTEA 并向高速缓存或主存发起请求；
3. 高速缓存或主存将 PTE 返回给 MMU；
4. MMU 构造物理地址 PA 并将其发送到高速缓存或主存；
5. 高速缓存或主存将请求的数据返回给处理器。

![20220609230237](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220609230237.png)

上图展示了缺页故障时的 CPU 硬件操作步骤：

1. 处理器生成一个虚拟地址 VA 并发送到 MMU；
2. MMU 生成 PTE 地址 PTEA 并向高速缓存或主存发起请求；
3. 高速缓存或主存将 PTE 返回给 MMU；
4. PTE 中的有效位为 0，因此 MMU 触发异常并将控制权转移给内核中的异常处理程序；
5. 处理程序从物理内存中选取受害者页面换出。若该页面已被修改，则还要将其复制到磁盘中；
6. 处理程序将新页面换入并更新 PTE；
7. 处理程序返回到原来的进程，之前引发缺页故障的指令重新执行。此时进程请求的页面已缓存，因此 CPU 随后的操作与页面命中时相同。

### 高速缓存与虚拟内存

大部分同时使用虚拟内存和高速缓存（SRAM 缓存）的系统均采用物理寻址的方式访问高速缓存。下图展示了两者的集成方式：

![20220610102342](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220610102342.png)

### 使用 TLB 加速地址转换

CPU 每次生成虚拟地址时，MMU 都必须引用 PTE 才能完成地址转换。如果 PTE 位于主存而非高速缓存中，那么地址转换的速度将大大下降。大多数系统的 MMU 中包含了一个被称为转换后备缓冲区（Translation Lookaside Buffer，TLB）的小型 PTE 缓存，其每个缓存行中都有一个由单条 PTE 组成的 Block。用于 [集合选择和行匹配](/posts/the-memory-hierarchy-note/#集关联型缓存) 的 Set Index 和 Tag 是从虚拟地址的 VPN 中提取的：

![20220610105500](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220610105500.png)

如果 TLB 有 T = $2^t$ 个集合，则 Set Index（TLBI）由 VPN 中 t 个最低位组成，Tag（TLBT）由 VPN 中的剩余高位组成。

![20220610111122](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220610111122.png)

上图展示了 TLB 命中时的 CPU 硬件操作步骤。由于地址转换均在 CPU 芯片上的 MMU 中执行，因此速度很快：

1. CPU 生成一个虚拟地址 VA；
2. MMU 向 TLB 发送 VPN 以请求 PTE；
3. TLB 将 PTE 返回给 MMU；
4. MMU 将虚拟地址转换为物理地址 PA 并发送到高速缓存或主存；
5. 高速缓存或主存将请求的数据返回给处理器。

![20220610111709](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220610111709.png)

上图展示了 TLB 未命中时的 CPU 硬件操作步骤。MMU 必须从高速缓存或主存中获取 PTE 并将其存储在 TLB 中，这可能会覆盖现有条目。

### 多级页表

在之前的讨论中，我们假设系统只使用单级页表进行地址转换。但如果地址空间有 32 位，一个页面 4 KB 并且一条 PTE 4 字节。那么即使应用程序只引用一小部分虚拟内存，我们也需要一个 4 MB 的页表常驻在内存中：

$$n=32, P=4K=2^{12}, n(PTE)=2^{n-p}=2^{20}=1M$$

我们可以通过对页表分级来压缩页表的大小：

![20220610115405](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220610115405.png)

如上图所示，一级页表中有 1024 条 PTE，每条 PTE 都映射到一个包含 1024 个连续虚拟页面的地址空间块。每个地址空间块的大小为 1024 \* 4 KB = 4 MB，因此 1024 条 PTE 就可以覆盖 32 位（4 MB \* 1024 = 4 GB = $2^{32}$ B）地址空间。

如果地址空间块中的所有页面均未分配，则一级页表中对应的 PTE 为空（如上图中的 PTE 2～7）；如果地址空间块中至少有一个页面已分配，那么一级页表中对应的 PTE 就指向二级页表中该块的起始位置（如上图中的 PTE 0～1）。二级页表中的每条 PTE 都映射到一个 4 KB 的物理内存页，这与我们之前查看的单级页表相同。

若一级页表中的 PTE 为空，那么二级页表内对应的条目就无需存在。多数应用程序的虚拟地址空间中大部分页面是未分配的，因此这将显著地降低页表的内存占用。另外，我们只需在主存中维护一级页表和被调用最为频繁的二级页表，其它的二级页表可以由操作系统按需创建和分页。

![20220610152657](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220610152657.png)

在 k 级页表中，虚拟地址被划分为 k 个 VPN 和一个 VPO，VPN i（1 ≤ i ≤ k）是第 i 级页表的索引。除第 k 级页表外，每个页表中的 PTE 均指向下一级页表的起始位置，而第 k 级表内的每条 PTE 则保存了对应物理页面的 PPN。与单级页表一样，PPO 与 VPO 相同。MMU 必须先请求 k 个 PTE，然后才能确定 PPN 以生成完整的物理地址。TLB 可以缓存多级页表中的 PTE，这使得多级页表的地址转换速度并不比单级页表慢很多。

## Intel Core i7/Linux 内存系统

尽管底层的 Haswell 微架构能够支持完整的 64 位虚拟和物理地址空间，但目前 Core i7 仅提供了 48 位 (256 TB) 的虚拟地址空间和 52 位 (4 PB ) 的物理地址空间，以及一个支持 32 位 (4 GB) 虚拟和物理地址空间的兼容模式。

### Core i7 地址转换

![20220610161613](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220610161613.png)

如上图所示，Core i7 使用四级页表结构。CR 3 控制寄存器中保存了一级 (L1) 页表的起始物理地址，其值是每个进程上下文的一部分并在上下文切换时恢复。48 位的虚拟地址包含了 36 位的 VPN 和 12 位（4 K = $2^{12}$）的 VPO，其中 VPN 又被划分为四个 9 位的地址空间块。

### Linux 虚拟内存系统

![20220610173011](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220610173011.png)

Linux 将虚拟内存划分为多个区域或段（Area 或 Segment），每个区域都是一些已分配且在某些方面相关的连续页面。例如，代码段、数据段、堆、共享库段和用户栈分别是不同的区域。每个已分配的页面都属于某个区域，因此不属于任何区域的页面不存在也无法被进程引用。区域概念的引入使得 Linux 允许虚拟地址空间中存在间隙。

![20220610175658](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220610175658.png)

如上图所示，Linux 内核为每个进程维护了一个独特的数据结构`task_struct`，其字段包含或指向内核运行该进程所需的全部信息（如 PID、用户栈指针、可执行目标文件名称和程序寄存器等）。其中，字段`mm`指向`mm_struct`，该结构体描述了虚拟内存的当前状态。`mm_struct`中的`pgd`字段指向一级页表的起始位置，它在进程运行时被内核存储在 CR 3 控制寄存器中。而`mmap`字段则指向一个由`vm_area_structs`组成的链表。描述区域信息的结构体`vm_area_structs`包含以下字段：

- `vm_start`：指向区域的起点；
- `vm_end`：指向区域的末端；
- `vm_prot`：描述该区域中所有页面的读/写权限；
- `vm_flags`：描述该区域中的页面是否与其他进程共享；
- `vm_next`：指向链表中下一个`vm_area_structs`。

![20220613110654](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220613110654.png)

异常处理程序可以将虚拟地址与所有`vm_area_structs`结构体中的`vm_start`和`vm_end`字段比较，从而判断它是否属于某个区域，如上图 ①。还可以根据`vm_prot`字段判断进程是否有权限访问目标区域中的页面，如上图 ②。

## 内存映射

Linux 使用内存映射（Memory Mapping）技术初始化虚拟内存区域并将其与磁盘上的“对象”相关联。该“对象”有两种类型：

- 文件系统中的常规文件（Regular File）：文件被分成多个与页面大小相同的片段，而每个片段都包含了一个虚拟页面的初始内容。由于操作系统采用按需分页的策略，因此页面在第一次被 CPU 引用前不会被换入到物理内存中。如果虚拟内存区域比文件大，则多余部分用零填充；
- 匿名文件（Anonymous File）：由内核创建，其内容全部为二进制零。CPU 第一次引用该区域内的虚拟页面时，内核会在物理内存中选择一个合适的受害者页面换出并用二进制零将其覆盖，然后更新页表使虚拟页面指向被覆盖后的物理页面。整个过程没有数据被换入到物理内存中，因此该区域内的页面又被称为零需求页面（Demand-zero Page）。

一旦我们使用内存映射初始化一个虚拟页面，它就会在由内核维护的交换文件（Swap File）与物理内存之间来回交换。交换文件又称交换区（Swap Area）或交换空间（Swap Space），它的大小限制了当前运行进程所能申请的虚拟页面总量。

### 回看共享目标文件

![20220613163833](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220613163833.png)

如上图所示，两进程将相同的 [共享目标文件](/posts/linking-note/#使用共享库动态链接) 映射到各自虚拟地址空间中的不同区域，而物理内存中只需存在单个文件副本。进程 1 对该共享区域的任何写入操作都对进程 2 可见，并且这些更改还会同步到磁盘上的原始文件。

![20220613165253](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220613165253.png)

如上图 (a) 所示，两进程将私有目标文件映射到各自虚拟地址空间的不同区域。进程 1 对该私有区域的任何写入操作都对进程 2 不可见，并且这些更改也不会同步到磁盘上的原始文件。如上图 (b) 所示，当进程 2 试图修改该区域中的内容时，内核会在物理内存中为页面创建一个新副本并更新页表条目使其指向它。由于页面复制发生在写入操作前，这种技术被称为写时复制（Copy-on-Write），这些区域则是私有写时复制的（Private Copy-on-Write）。

### 回看 fork 函数

进程调用 [`fork`](/posts/exception-control-flow-note/#创建和中止进程) 函数后，内核会为子进程分配一个唯一的 PID 并为其创建与父进程相同的 `mm_struct`、`vm_area_structs`以及页表。当任一进程后续执行写入操作时，内核将使用写时复制技术创建新页面，这便保证了进程虚拟地址空间的私有性。

### 回看 execve 函数

如果进程调用 [`execve`](/posts/exception-control-flow-note/#加载并运行程序) 函数，如 `execve("a.out", NULL, NULL)`，则加载并运行`a.out`的步骤如下：

1. 删除当前进程虚拟地址空间中用户区域的`vm_area_structs`；
2. 为新程序的代码、数据、.bss 和堆栈区域创建`vm_area_structs`。这些区域都是私有写时复制的，代码和数据区域被映射到`a.out`文件中的 [.text 和 .data](/posts/linking-note/#可重定位目标文件)，.bss 区域则被映射到匿名文件。堆栈的初始长度均为 0，其页面是零需求的；
3. 如果`a.out`文件链接了共享库，如 C 标准库 `libc.so`，那么还需要将这些对象动态链接到程序中，然后将其映射到虚拟地址空间中的共享区域内；
4. 使当前进程上下文中的程序计数器指向新程序代码区域的入口点。

![20220613213646](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220613213646.png)

### mmap 函数

```c
#include <unistd.h>
#include <sys/mman.h>
void  *mmap(void *start, size_t length, int prot, int flags,
            int fd, off_t offset);
// Returns: pointer to mapped area if OK, MAP_FAILED (−1) on error
```

`mmap`函数请求内核创建一个起始地址为参数`start`的虚拟内存区域，该区域映射到文件描述符`fd`所指定的对象。连续对象的长度为参数`length`，其首部在文件中的偏移量为参数`offset`：

![20220613222541](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220613222541.png)

参数`prot`包含了描述虚拟内存区域访问权限的位，即`vm_area_structs`中的`vm_prot`：

- PROT_EXEC：该区域中的页面包含可执行指令；
- PROT_READ：可以阅读该区域中的页面；
- PROT_WRITE：可以写入该区域中的页面；
- PROT_NONE：无法访问该区域中的页面。

参数`flag`包含了描述了映射对象类型的位：

- MAP_SHARED：共享对象；
- MAP_PRIVATE：私有写时复制对象；
- MAP_ANON：匿名对象，对应的虚拟页面是零需求页面。

`munmap`函数删除起始于虚拟地址`start`、长度为`length`的区域，后续对已删除区域的引用会引发分段故障。

```c
#include <unistd.h>
#include <sys/mman.h>
int munmap(void *start, size_t length);
// Returns: 0 if OK, −1 on error
```

## 动态内存分配

相比于低级别的`mmap`函数，C 程序员更倾向于在运行时调用动态内存分配器（Dynamic Memory Allocator）来创建虚拟内存区域。动态内存分配器为进程维护的虚拟内存区域被称为堆（Heap），其一般结构为：

![20220614113920](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220614113920.png)

堆向上增长，内核为每个进程都维护了一个指向堆顶的变量`brk`。分配器将堆看作一个包含不同尺寸 Block 的集合，每个 Block 都是一个连续的虚拟内存块。Block 有两种状态，已分配（Allocated）和空闲（Free）。所有分配器均显式地为应用程序分配 Block，但负责释放已分配 Block 的实体可能有所不同：

- 显式分配器：应用程序显式地释放已分配的 Block。C 和 C++ 程序分别调用`malloc`和`new`函数请求 Block，调用`free`和`delete`函数释放 Block；
- 隐式分配器：分配器自行释放程序不再使用的已分配 Block，该过程被称为垃圾回收（Garbage Collection）。Lisp、ML 和 Java 等高级语言均采用这种方法。

### malloc 和 free 函数

```c
#include <stdlib.h>
void *malloc(size_t size);
// Returns: pointer to allocated block if OK, NULL on error
```

`malloc`函数请求堆中的一块 Block 并返回指向该 Block 的指针。Block 的大小至少为`size`，并可能根据其中的数据对象类型进行适当的对齐。在 32 位编译模式下，Block 的地址始终为 8 的倍数，而在 64 位中则为 16 的倍数。如果执行`malloc`遇到问题，如程序请求的 Block 大小超过了可用的虚拟内存，则函数返回`NULL`并设置 [`errno`](https://man7.org/linux/man-pages/man3/errno.3.html)。我们还可以使用`malloc`的包装函数`calloc`，它会将分配的内存初始化为零。类似地，`realloc`函数可以更改已分配 Block 的大小。

```c
#include <unistd.h>
void *sbrk(intptr_t incr);
// Returns: old brk pointer on success, −1 on error
```

`sbrk`函数将参数`incr`与内核中的`brk`指针相加以增大或缩小堆。若执行成功，则返回`brk`的旧值，否则将返回 -1 并将`errno`设置为 ENOMEM。

```c
#include <stdlib.h>
void free(void *ptr);
// Returns: nothing
```

`free`函数将参数`ptr`指向的 Block 释放，而这些 Block 必须是由`malloc`、`calloc` 或 `realloc`分配的。该函数没有返回值，因此很容易产生一些令人费解的运行时错误。

![20220614154301](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220614154301.png)

上图展示了 C 程序如何使用`malloc`和`free`管理一个大小为 16 字（字长为 4 字节）的堆。图中的每个方框代表一个字，每个被粗线分隔的矩形代表一个 Block。有阴影的 Block 代表已分配，无阴影的 Block 则代表空闲。

如上图 (a) 所示，程序请求一个 4 字的 Block，`malloc`从空闲块中切出一个 4 字的 Block 并返回指向该 Block 中第一个字的指针`p1`；如上图 (b) 所示，程序请求一个 5 字的 Block，`malloc`从空闲块中切出一个 6 字的 Block 以实现双字对齐；如上图 (c) 所示，程序请求一个 6 字的 Block，`malloc`从空闲块中切出一个 6 字的 Block；如上图 (d) 所示，程序释放图 (b) 中分配的 Block。`free`返回后，指针`p2`依然指向已释放的 Block，因此程序不应在重新初始化`p2`前继续使用它；如上图 (e) 所示，程序请求一个 2 字的 Block。`malloc`从上一步释放的 Block 中切出一部分并返回指向新 Block 的指针`p4`。

### 动态内存分配的原因

在程序实际运行之前，我们可能并不知道某些数据结构的大小。示例 C 程序将`n`个 ASCII 整型从标准输入读取到数组`array[MAXN]`中：

```c
#include "csapp.h"
#define MAXN 15213

int array[MAXN];

int main()
{
    int i, n;
    scanf("%d", &n);
    if (n > MAXN)
        app_error("Input file too big");
    for (i = 0; i < n; i++)
        scanf("%d", &array[i]);
    exit(0);
}
```

由于我们无法预测`n`的值，因此只能将数组大小写死为`MAXN`。`MAXN`的值是任意的，可能超出系统可用的虚拟内存量。另外，一旦程序想要读取一个比`MAXN`还大的文件，唯一的办法就是增大`MAXN`的值并重新编译程序。而如果我们在运行时根据`n`的大小动态分配内存，以上问题便迎刃而解：

```c
#include "csapp.h"

int main()
{
    int *array, i, n;
    scanf("%d", &n);
    array = (int *)Malloc(n * sizeof(int));
    for (i = 0; i < n; i++)
        scanf("%d", &array[i]);
    free(array);
    exit(0);
}
```

### 对分配器的要求和目标

显式分配器必须在若干限制条件下运行：

- 处理任意顺序的请求：一个应用程序发出的`malloc`和`free`请求的顺序是任意的，因此分配器不能对其作出假设。例如，分配器不能假设所有的`malloc`都紧跟一个与之匹配的`free`；
- 立即响应请求：分配器不可以对请求重新排序或缓冲（Buffer）以提高性能；
- 仅使用堆：分配器使用的数据结构必须存储在堆中；
- 对齐 Block：分配器必须对齐 Block 以使其能够容纳任何类型的数据对象；
- 不修改已分配的 Block：分配器无法修改、移动或压缩 Block。

衡量分配器性能的指标有：

- 吞吐量（Throughput）：单位时间内完成的请求数；
- 内存利用率（Memory Utilization）：即堆内存的使用率。

最大化吞吐量和最大化内存利用率之间存在矛盾，因此我们在设计分配器时需要找到二者的平衡。

### 碎片

我们将未使用的堆内存无法满足分配请求的现象称为碎片（Fragmentation），它是内存利用率低的主要原因。碎片有两种形式：

- 内部碎片（Internal Fragmentation）：已分配的 Block 比进程请求的 Block（即 Payload）大，通常因分配器为满足对齐要求而产生；
- 外部碎片（External Fragmentation）：空闲内存充足但却没有空闲的 Block 能够满足分配请求。例如堆中有 4 个空闲的字且分布在两个不相邻的 Block 上，此时若进程申请一个 4 字的 Block 就会出现外部碎片。

内部碎片很容易量化，它只是已分配 Block 与 Payload 之间大小差异的总和，其数量仅取决于先前的请求模式和分配器的实现方式。外部碎片则难以量化，因为它还受未来请求模式的影响。为了减少外部碎片的产生，分配器力求维护少量较大的空闲 Block 而非大量较小的空闲 Block。

### 分配器的实现难点

我们可以想象一个简单的分配器，它将堆看作一个大型的字节数组，指针`p`指向该数组的第一个字节。当进程请求`size`大小的 Block 时，`malloc`先把`p`的当前值保存在栈中，然后将其加上`size`，最后返回`p`的旧值。当进程想要释放 Block 时，`free`则只是简单地返回给调用者而不做任何事情。

由于`malloc`和`free`仅由少量指令组成，这种分配器的吞吐量很大。然而`malloc`不会将已释放的 Block 分配给进程 ，因此堆内存的利用率非常低。能够在吞吐量和内存利用率之间取得良好平衡的分配器必须考虑以下问题：

- 组织（Organization）：如何跟踪空闲的 Block？
- 放置（Placement）：如何从空闲的 Block 中选择合适的来放置新分配的 Block？
- 分割（Splitting）：完成放置后，如何处理剩余的空闲 Block？
- 合并（Coalescing）：如何处理刚刚被释放的 Block？

### 隐式空闲链表

大多数分配器通过将一些数据结构嵌入到 Block 中以分辨其边界和状态，例如：

![20220615154704](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220615154704.png)

如上图所示，Block 由一个单字（四字节）的头部（Header）、有效负载（Payload）和一些额外填充（Padding）组成，头部中包含了 Block 的大小（Block Size）和状态信息（Allocated or Free）。如果系统采用双字对齐策略，那么每个 Block 的大小始终为 8 的倍数，其后 3 位始终为 0。因此我们可以仅在头部中存储该字段的前 29 位，剩余 3 位用来存储其他信息。上图中的位“a”便指示了此 Block 是已分配的还是空闲的。填充的大小是任意的，它可能是分配器为了避免外部碎片产生而设置的，也可能是为了满足对齐要求而存在的。

基于这种 Block 格式，我们可以将堆组织成一系列连续的已分配 Block 和空闲 Block：

![20220615162852](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220615162852.png)

空闲 Block 通过其头部中的大小字段隐式地链接起来（见上图中的箭头），因此我们将这种堆组织方式称为隐式空闲链表（Implicit Free List），分配器必须遍历堆中所有的 Block 才能得到全部空闲的 Block。我们还需要一个特殊的 Block 以标记堆的结尾，如上图中的 “0/1”。隐式空闲链表的优点是简单，但任何搜索空闲 Block 的操作（如放置新分配的 Block）的成本都与堆中 Block 的总数成正比。

### 放置新分配的 Block

当应用程序请求一个 Block 时，分配器需要在空闲链表中选取一个足够大的 Block 以响应。分配器搜索空闲 Block 的方式由放置策略（Placement Policy）所决定：

- 第一次拟合（First Fit）：从头开始遍历空闲链表并选择第一个满足条件的 Block；
- 下一次拟合（Next Fit）：从上一次搜索停止的地方开始遍历空闲链表并选择第一个满足条件的 Block；
- 最佳拟合（Best Fit）：遍历所有 Block 并选择满足条件且最小的 Block。

第一次拟合的优点是较大的 Block 通常存留在链表末尾，但一些较小的 Block 也会散落在链表开头，这将增加搜索较大 Block 的时间。如果链表开头存在大量较小的 Block，下一次拟合就比第一次拟合快很多。然而研究表明，下一次拟合的内存利用率比第一次拟合低。最佳拟合的内存利用率通常比其他两种策略高，但对隐式空闲链表来说，其搜索时间显然要比它们慢很多。

### 分割空闲的 Block

如果分配器找到了合适的 Block 并将整个 Block 分配给程序，就有可能产生内部碎片。为了避免这一问题，分配器可以将选取的 Block 分成一个已分配的 Block 和一个新的空闲 Block：

![20220615181024](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220615181024.png)

如图 9.37 所示，程序请求一个 3 字的 Block，于是分配器将图 9.36 中 8 字的空闲 Block 拆分为两个 4 字的 Block 并将其中之一分配给它。

### 获取额外的堆内存

如果分配器无法找到合适的 Block，它可以尝试将相邻的空闲 Block 合并以获取更大的 Block。但如果仍然无法满足请求，分配器便会调用`sbrk`函数向内核请求额外的堆内存并将其转换为一个新的空闲 Block。

### 合并空闲的 Block

分配器释放 Block 后，可能会有其他空闲的 Block 与之相邻。此时内存中将产生一种特殊的外部碎片现象，我们称其为虚假碎片（False Fagmentation）。

![20220615220820](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220615220820.png)

如图 9.38 所示，分配器将图 9.37 中第二个 4 字 Block 释放。尽管两个连续空闲 Block 中均有 3 字的有效负载，但它们也无法满足一个 4 字的分配请求。因此，分配器必须将相邻空闲的 Block 合并。

分配器可以在每次释放 Block 后立即合并 Block，也可以等到某个时刻，比如分配请求失败时才合并堆中所有的空闲 Block。立即合并很简单，但它也可能造成“颠簸”，即某个 Block 在短时间内被多次合并和分割。

### 使用边界标记合并 Block

我们把即将释放的 Block 称为当前（Current）Block，其头部指向下一个 Block 的头部。因此我们便很容易判断下一个 Block 是否空闲，并且只需将当前 Block 头部中的大小字段与之相加即可完成合并。

而若要合并上一个 Block，我们只能遍历整个链表，在到达当前 Block 前不断记下上一个 Block 的位置。因此对于隐式空闲链表，合并上一个 Block 的时间与堆内存的大小成正比。

我们可以在每个 Block 末尾都添加一个头部的副本以使合并 Block 的时间变为常数，这种技术被称为边界标记（Boundary Tags）：

![20220615231949](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220615231949.png)

上一个 Block 的尾部始终与当前 Block 的头部相距一个字长，因此分配器可以通过检查上一个 Block 的尾部来确定其位置和状态。下图展示了分配器是如何使用边界标记合并 Block 的：

![20220615232751](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220615232751.png)

由于每个 Block 都包含头部和尾部，因此当 Block 数量较多时，边界标记显著地增加了内存的开销。考虑到分配器只有在上一个 Block 空闲时才需要获取其尾部内的 Block 大小，我们可以将上一个 Block 的状态存储在当前 Block 头部的多余低位中，这样已分配的 Block 便不需要尾部了。

### 显式空闲链表

由于分配 Block 的时间与 Block 的总数成正比，隐式空闲链表不适用于通用分配器。我们可以在每个空闲 Block 中加入一个前驱（Predecessor）指针和一个后继（Successor）指针，这样堆的组织结构就变成了一个双向链表，我们称其为显式空闲链表（Explicit Free List）。

![20220616115552](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220616115552.png)

如果采用第一次拟合策略，显式空闲链表分配 Block 的时间与空闲 Block 的数量成正比，而释放 Block 的时间则取决于空闲 Block 的排序方式：

- 后进先出（Last-in First-out，LIFO）：将刚被释放的 Block 插入到链表开头。若采用第一次拟合策略，分配器将首先检查最近被使用的 Block。此时释放 Block 的时间为常数，并且可以通过边界标记使合并 Block 的时间也为常数；
- 地址顺序：使链表中每个 Block 的地址均小于其后继 Block 的地址。在这种情况下，释放 Block 需要一定的时间来寻找合适的位置，但堆内存利用率比后进先出高。

显式空闲链表的缺点在于指针的引入增加了空闲 Block 的大小，这将增大内部碎片发生的可能性。

### 分离式空闲链表

我们可以将堆组织成多个空闲链表，每个链表中 Block 的大小都大致相同，这种结构被称为分离式空闲链表（Segregated Free List）。为了实现这一结构，我们需要把 Block 的大小划分为多个大小类（Size Class），如：

$$\lbrace1\rbrace, \lbrace2\rbrace, \lbrace3, 4\rbrace, \lbrace5–8\rbrace,..., \lbrace1025–2048\rbrace, \lbrace2049–4096\rbrace, \lbrace4097–∞\rbrace$$

也可以让每个较小的 Block 独自成为一个大小类，较大的 Block 依然按 2 的幂划分：

$$\lbrace1\rbrace, \lbrace2\rbrace, \lbrace3\rbrace,..., \lbrace1024\rbrace, \lbrace1025–2048\rbrace, \lbrace2049–4096\rbrace, \lbrace4097–∞\rbrace$$

每个空闲链表都对应于一个大小类，因此我们可以将堆看成一个按大小类递增的空闲链表数组。当进程请求一个 Block 时，分配器会根据其大小在适当的空闲链表中搜索。如果找不到满足要求的 Block，它会继续搜索下一个链表。

不同的分离式空闲链表在定义大小类的方式、合并 Block 的时机以及是否允许分割 Block 等方面有所不同，其中最基本的两种类型为简单分离存储（Simple Segregated Storage）和分离拟合（Segregated Fits）。

在简单分离存储中，空闲链表内每个 Block 的大小均完全相同，等于其大小类中最大元素的大小。如某个大小类为 {17-32}，其对应的空闲链表中 Block 的大小都是 32。

当进程请求一个 Block 时，分配器选取满足请求的空闲链表并分配其中第一个 Block。当某个 Block 被释放后，分配器将其插入到合适的空闲链表前面。因此，简单分离存储分配和释放 Block 的时间均为常量。

## 垃圾回收

垃圾回收器（Garbage Collector）是一种动态存储分配器，它会自动释放程序不再需要的 Block（垃圾）。

### 基本思想

垃圾回收器将内存看作一个有向可达性图：

![20220616172206](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220616172206.png)

图中的节点被分为一组根节点（Root Nodes）和一组堆节点（Heap Nodes），每个堆节点都对应于堆中的一个已分配的 Block。有向边 $p \rarr q$ 表示 Block $p$ 中的某个位置指向 Block $q$ 中的某个位置。~~根节点对应于不在堆中却指向了堆的位置，这些位置可以是寄存器、栈中的变量或可读写数据区域中的全局变量~~。

> Root nodes correspond to locations not in the heap that contain pointers into the heap. These locations can be registers, variables on the stack, or global variables in the read/write data area of virtual memory.

如果根节点与堆节点之间存在一条有向路径，我们就称该堆节点是可达的（Reachable）。在任何时刻，不可达的节点都对应于程序不再使用的 Block。垃圾回收器定期释放不可达节点并将其返回到空闲链表。

ML 和 Java 等语言的垃圾回收器对应用程序使用指针的方式进行了严格的限制，因此它可以维护一个精确的可达性图，从而回收所有的垃圾。而 C 和 C++ 等语言的垃圾回收器则无法保证可达性图的精确性，一些不可达的节点可能被错误地识别为可达的，我们称其为保守垃圾回收器（Conservative Garbage Collector）。

垃圾回收器可以按需提供服务，也可以作为单独的进程与应用程序并行运行，不断更新可达性图并回收垃圾：

![20220616203627](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220616203627.png)

### Mark&Sweep 算法

Mark&Sweep 是常用的垃圾回收算法之一，它分为两个阶段：

- 标记（Mark）阶段：标记所有可达的根节点后代。通常我们将 Block 头部的多余低位之一用于指示该 Block 是否被标记；
- 清除（Sweep）阶段：释放所有未标记且已分配的 Block。

为了更好地理解 Mark&Sweep 算法，我们作出以下假设：

- `ptr`：由`typedef void *ptr`定义的类型；
- `ptr isPtr(ptr p)`：若`p`指向已分配 Block 中的某个字，则返回指向该 Block 起始位置的指针`b`，否则返回`NULL`；
- `int blockMarked(ptr b)`：如果该 Block 已被标记则返回`true`；
- `int blockAllocated(ptr b)`：如果该 Block 已分配则返回`true`；
- `void markBlock(ptr b)`：标记 Block；
- `int length(ptr b)`：返回 Block 除头部外的字长；
- `void unmarkBlock(ptr b)`：将 Block 的状态从已标记转换为未标记；
- `ptr nextBlock(ptr b)`：返回指向下一个 Block 的指针。

那么此算法就可以用下图中的伪码表示：

![20220616212514](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220616212514.png)

在标记阶段，垃圾回收器为每个根节点调用一次`mark`函数。如果`p`未指向已分配且未标记的 Block，则该函数直接返回。否则，它标记该 Block 并将其中的每个字作为参数递归地调用自身（`mark(b[i])`）。此阶段结束时，任何未标记且已分配的 Block 都是不可达的。在扫描阶段，垃圾回收器只调用一次`sweep`函数。该函数遍历堆中的每一个 Block，释放所有已分配且未标记的 Block。

![20220616220603](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20220616220603.png)

上图中的每个方框代表一个字，每个被粗线分隔的矩形代表一个 Block，而每个 Block 都有一个单字的头部。最初，堆中有 6 个已分配且未标记的 Block。Block 3 中包含指向 Block 1 的指针，Block 4 中包含指向 Block 3 和 6 的指针。根节点指向 Block 4，因此 Block 1、3、4 和 6 从根节点可达，它们会被垃圾回收器标记。在扫描阶段完成后，剩余不可达的 Block 2 和 5 将被释放。
