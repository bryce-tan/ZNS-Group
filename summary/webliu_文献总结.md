# 阅读记录

## ZNS: Avoiding the Block Interface Tax for Flash-based SSDs

### 背景

传统接口的一些特性：

1. 传统的SSD和主机软件的功能划分:![img](https://img2023.cnblogs.com/blog/3067108/202306/3067108-20230611130607424-554407542.png)其中SSD的FTL层(Flash Translation Layer，闪存翻译层)需要有映射和垃圾回收机制且SSD要有一定的预留空间
2. 块接口和当前存储设备特性(闪存颗粒)之间存在明显的不匹配，这种不匹配主要体现在我们可以将块写入闪存(接口决定的)但在擦除时必须以更大的粒度擦除闪存(闪存颗粒的性质决定的)。传统SSD使用块接口，导致其要想减少垃圾回收相应的主机软件会较为复杂，用于页面映射的DRAM较大，预留空间也较大(为了减少垃圾回收)，且仍然可能导致吞吐量限制 [17]、写入放大 [3]、性能不可预测性和高尾延迟(后台的GC导致)

#### SSD

[SSD特性的来源](https://www.tonguebusy.com/a/peixun/xinxi/03-we-q-w-06.html)

SSD中：channels(并行闪存控制器) > flash chips > flash die > flash plane > erase block > page > typical logical block size(4KB)
![img](https://img2023.cnblogs.com/blog/3067108/202403/3067108-20240323143709800-1272721894.png)
![img](https://img2023.cnblogs.com/blog/3067108/202403/3067108-20240323143716508-892223598.png)

* 逻辑块的大小一般是一页，一个擦除块可能包含多个页![img](https://img2023.cnblogs.com/blog/3067108/202306/3067108-20230611134729590-1304353693.png)![img](https://img2023.cnblogs.com/blog/3067108/202306/3067108-20230611142220405-1672501370.png)

为了给基于闪存的设备一个块接口，我们需要FTL层来让设备支持随机"覆写"，映射，垃圾回收，均匀使用闪存

一些缺点：
> 闪存颗粒不支持"覆写"，它只能将数据擦除后重新写，所以逻辑块的写入实际上是写入到一个没有数据的地方，且需要旧的不会用到的数据所在的擦除块被擦除来腾出空间。当擦除块中有的数据用不到了，有的数据还需要使用，那么我们得将需要使用的数据转移到其他擦除块中，把其他擦除块中要擦除的数据转移到那个将要擦除的块统一擦除，为此我们需要预留一部分空间(7%或28%)来方便转移数据。![img](https://img2023.cnblogs.com/blog/3067108/202306/3067108-20230611142321342-1853616515.png)
> 剩余容量越小，垃圾回收就会越频繁，而垃圾回收需要转移数据和初始化擦除块，这会带来一定的写放大
> 其次SSD的使用寿命等同于Block的擦写次数，为了提升总体寿命，SSD的主控芯片会尽可能的让每个Block的擦写次数均匀。这就需要擦写特定的块，也会导致一定的写放大。
> 再次由于逻辑块的粒度较小，为了支持映射需要较大的DRAM来保存页表
> 最后是预留空间的代价，它不能用来存数据

现有的两种改进措施

1. Stream SSD
2. Open-Channel SSD

Stream SSD将传入数据分到不同的擦除块，从而提升整体的性能和寿命。做法是主机软件用流提示符标记其写入命令，SSD根据这些标记分配数据。这要求主机软件仔细区分数据的生命周期并给它们标记，从而真正减少垃圾回收。Stream SSD对OP和DRAM的代价没有什么帮助(因为依旧需要转移数据、且物理块粒度依旧很小)

> ![img](https://img2023.cnblogs.com/blog/3067108/202306/3067108-20230612192233965-792410754.png)
> ![img](https://img2023.cnblogs.com/blog/3067108/202306/3067108-20230612192513146-825310759.png)
> OCSSD向主机公开了物理块，消除了设备内垃圾收集开销，并降低了OP和 DRAM 的成本。主机需要决定把数据放到哪个物理块，如何回收块，遇到介质故障时如何解决（而这些问题都和硬件特性有关）。这就相当于没有接口，物理资源直接暴露出来，软件可能需要频繁更新(不再模块化，硬件改变软件就要改变)

#### ZNS

一些ZNS相关知识：

ZNS使用状态机来确定可写zone，使用一个写指针来确保zone内顺序写
![img](https://img2023.cnblogs.com/blog/3067108/202306/3067108-20230612193220417-333190586.png)

![img](https://img2023.cnblogs.com/blog/3067108/202306/3067108-20230612194247591-2103696286.png)
ZNS SSD的区域的可写容量可以小于区的实际容量(它规定一些LBA是不可写的)，来适应SMR HDD的行业标准。

ZNS可以按需要限制active zone的数量
![img](https://img2023.cnblogs.com/blog/3067108/202306/3067108-20230612195204581-294234931.png)
![img](https://img2023.cnblogs.com/blog/3067108/202306/3067108-20230612195322840-1023506186.png)
最大活动区域的限制是基于SSD的介质特性(program disturb)

### 方法设计

![img](https://img2023.cnblogs.com/blog/3067108/202306/3067108-20230614235935801-658670902.png)

ZNS接口的做法：

新的责任划分、新的粒度

* 暴露出闪存擦除块边界和写入顺序规则(flash erase block boundaries and write-ordering rules),要求主机软件以擦除块为粒度解决数据管理的问题(不支持随机写，同时主机软件来负责显式擦除)从而不再要求SSD提供垃圾回收机制和预留空间
* 同时降低DRAM需求(传统的FTL闪存转换层需要1GB的DRAM才能映射1TB的NAND闪存，这是因为粒度为4Kb，ZNS每个区域数百M,当然相应的映射所需的数据量也会多一些，ZNS不是逻辑块的一维数组，而是将多个逻辑块分为一个区域(zone)，zone和物理块是对齐的，区域中的逻辑块可以随机读但必须顺序写，且只允许擦除写)，但SSD仍负责管理SSD内的介质可靠性( media reliability)，顺序写要求是因为只维护了到逻辑zone到物理zone的映射。
* 在分区之后，降低了多用户的工作流之间的干扰（防止不同租户对channel或die的带宽进行竞争）
![img](https://img2023.cnblogs.com/blog/3067108/202403/3067108-20240323144528304-1176558776.png)

将 FTL 职责转移给主机的效率不如与存储软件的数据映射和放置逻辑集成（集成到f2fs或者ZenFS）
这一工作基于引入 Linux 内核、fio 基准测试工具、f2fs 文件系统和 RocksDB 键值存储对的 ZNS SSD 的支持 （这些工作已经开源）

#### Hardware

ZNS 的FTL层的一些改变

1. 区域大小调整。区域的写入容量与 SSD 实现的擦除块的大小之间存在直接关联。SSD以擦除块或条带(stripe)为单位提供奇偶校验来防止故障。通常我们主张以提供校验的最小单位来设置区域大小，这样既能在设备级提供数据保护，也能提高主机放置数据的自由度
2. 新的映射表。由于 ZNS SSD 区域写入要求是顺序的,我们完全可以以zone为粒度建立映射表(原本占据SSD上DRAM的大部分)
3. 规划资源。部分写入的擦除块(也就是活动区)会占用资源，ZNS SSD一般设置8~32个活动区

> 擦除块的粒度大小关系到读写性能(写放大)，I/O 队列长度
> I/O queue depths是指在计算机系统中，用于管理输入/输出(I/O)请求的队列中，可以同时排队等待处理的I/O请求的数量。通常，I/O请求会被放置在队列中，等待系统资源(如磁盘、网络等)处理。如果队列中的I/O请求数量过多，可能会导致系统资源繁忙或者饱和，从而影响系统的性能和响应时间。
>在操作系统和存储系统中，通常会设置I/O queue depths的最大值，以控制队列中I/O请求的数量。这个最大值通常是根据系统的硬件配置、工作负载和性能需求等因素进行调整的。如果I/O queue depths过高，可能会导致系统资源的过度使用，从而降低系统的性能和响应时间；如果I/O queue depths过低，可能会导致系统资源的浪费和I/O请求的延迟。
>因此，对于不同的系统和应用场景，需要根据实际情况进行I/O queue depths的设置和调整，以保证系统的性能和响应时间。

zone设置过大可能会导致擦除时出现写放大，如果设置得过小会导致I/O队列过深 (最小不能小于设备提供可靠性支持的最小单元)

#### Software

有三种方式可以使主机软件适配ZNS 接口

1. 主机引入Host-side FTL(HFTL)， HFTL 层类似于 SSD FTL 的职责，**向应用提供了随机写和覆写**，但 HFTL 层只需要管理转换映射和关联的垃圾回收。
2. 文件系统在存储堆栈的更高一级设置zone，来确保其写操作主要是顺序的。
3. 实现端到端的数据放置(应用程序直接控制数据写到ZNS SSD)，可以消除来自文件系统和转换层的间接开销，但也要求应用程序主要顺序写，集成zone支持且为用户提供执行检查、错误检查和备份/还原操作的工具。

> 虽然并非所有的写入都是顺序的（例如超级块和一些元数据写入），但像 f2fs [36], btrfs [53], 和 zfs这样的文件系统可以通过严格的日志结构写入 [43]、浮动超级块 [4] 和类似的功能来弥补差距。
> 主要呈现顺序写入的应用程序有：基于 LSM 树的存储（如 RocksDB）、基于缓存的存储（如 CacheLib [7]）和对象存储（如 Ceph SeaStore [55]。
> ![img](https://img2023.cnblogs.com/blog/3067108/202306/3067108-20230613223313909-366056414.png)

增加了对四个主要软件项目的支持，以评估 ZNS 的优势。

1. 对Linux内核进行了修改以支持ZNS SSD。
2. 修改了 f2fs 文件系统，以评估在更高级别的存储堆栈层进行区域集成的好处。
3. 修改了 fio [6] 基准测试以支持新添加的 ZNS 特定属性。
4. 开发了ZenFS [25]，这是RocksDB的新型存储后端，允许通过区域控制数据放置，以评估分区存储端到端集成的好处。

其中前三个项目已经支持ZAC/ZBC定义的区域模型，只需要相对较少的更改就可以支持ZNS
ZenFS的架构则是全新的

![img](https://img2023.cnblogs.com/blog/3067108/202306/3067108-20230613003640076-477714461.png)

##### linux

为Linux 内核设计了Zoned Block Device子系统，来为各种ZNS类设备提供新的接口

ZBD提供了内核内 API 和基于 ioctl 的用户空间 API，来支持设备枚举、区域报告和区域管理（例如区域重置），应用程序通过这些接口发出 I/O 请求

>![img](https://img2023.cnblogs.com/blog/3067108/202306/3067108-20230613222447548-564103900.png)

在内核内维护一组有关各个zone的信息，他们由主机维护，当出错时则从特定的磁盘中刷新。

##### fio 和 f2fs

fio和f2fs需要加以更改来识别zone描述表

更改包括

1. 区域容量的相关修改
   * fio需要避免发出超出区域容量的写入 I/O
   * f2fs 按段的粒度管理容量，通常为 2MiB 块，我们要将多个段合并为一个zone进行管理。f2fs的段的状态有:free, open和full，为了支持ZNS，我们又加了两种:unusable(对应zone中unwritable的部分),partial(对应跨越了zone中unwritable和writable部分的段)，这样就解决了zone的容量和段不能对齐的情况。
2. 活动区域限制的相关修改
    * fio不做修改，需要用户去遵守活动区域限制，否则就会出现 I/O 错误
    * f2fs限制可以同时打开的段数为6，但可以减少它来与活动区域限制保持一致。这需要修改 f2fs-tools 来检查区域活动限制。f2fs的元数据是要保存在块设备上的，作者没有直接解决这一问题，而是暴露一部分ZNS SSD的容量作为块设备

##### RocksDB 与 ZenFS

调整常用的键值数据库 RocksDB，以使用 ZenFS 存储后端在分区存储设备上实现端到端数据放置。

ZenFS利用RocksDB的日志结构合并（LSM）树[45]数据结构，用于存储和维护其数据，以及相关的不可变顺序压缩过程。

![img](https://img2023.cnblogs.com/blog/3067108/202306/3067108-20230614122955433-272117941.png)

LSM树的工作过程：

* $L_0$层在内存中，它保存新的或者更新了的键值对，并定期或者当满的时候将数据传递给下层。刷新之间更新通过预写日志来保存。
* $L_1,L_2, ... ,L_n$层在硬盘中，传递过来的数据将按键值排序，然后以 Sorted String Table (SST) 文件的形式写入到磁盘中。这样可以实现冷热数据分离和**顺序写**。

具体工作原理可以参考[深入浅出分析LSM](https://zhuanlan.zhihu.com/p/415799237)，总的来说就是插入删除和原地更新都直接在$L_0$进行，但查询需要一层一层的往下找。第$L_0$层用树来存，提高插入删除和更新效率，其余层用有序对来存，方便进行归并排序。其数据特征时越上层的数据为越新，当满了之后将其与下层数据归并之后更新下层同时清空本层。

设计原则：

* 先内存再磁盘
* 内存原地更新
* 磁盘追加更新
* 归并保留新值

RockDB需要通过文件系统的API来访问磁盘数据，这个API使用标识符来区分数据单元，基于标识符来实现一系列操作(例如，添加、删除、当前大小、利用率)。

RocksDB 避免了管理文件范围、缓冲和可用空间管理，但也失去了将数据直接放入区域的能力，这阻止了端到端数据放置，从而降低了整体性能。

ZenFS实现了一个最轻量的磁盘文件系统，并与RocksDB的文件系统API集成。

结构如图所示
![img](https://img2023.cnblogs.com/blog/3067108/202306/3067108-20230614150755671-1444946172.png)

ZenFS 定义了两种类型的区域：日志区域和数据区域。

* 日志区域用于恢复文件系统的状态，并维护超级块数据结构，以及 预写日志 和数据文件到zone的映射。日志存储在专门的日志区域中，一般是设备的前两个非offline的区域。在任何时间点，都会选择其中一个区域作为活动日志区域，保存日志状态更新。日志区域的开头保存一个header，它记录存储序列号（每次初始化新日志区域时递增）、超级块数据结构和当前日志状态的快照。区域的剩余可写容量用于记录日志的更新。

    恢复日志状态有三步

  1. 读取两个日志区域中每个区域的第一个 LBA 以确定每个日志区域的序列号，较大的那个是活动日志区域
  2. 读取活动日志区域的完整header，初始化superblock和日志状态
  3. 对header的日志快照按照记录进行更新

    当ZenFS初始化是就按照上面的步骤进行恢复，之后就可以接受来自RocksDB的数据了。

* 数据区域存储文件内容。数据区域包含多个Extent，它是大小可变、块对齐、顺序写入的一块连续空间，包含与特定标识符关联的数据。Extent的分配和释放会被写入日志，同时extent到zone的映射用内存中的一个数据结构记录下来，当zone中的所有extent被删除，该zone可以被reset。

* 超级块数据结构是从磁盘初始化和恢复 ZenFS 状态时的初始入口点。超级块包含当前实例的唯一标识符、魔术值和用户选项。超级块中的唯一标识符 （UUID） 允许用户识别文件系统，即使系统上块设备枚举的顺序发生变化。(不是很懂)

* 文件大小是数据区域容量的倍数时，能够充分利用所有容量。但我们实现不知道文件大小，这和压缩和压缩过程的结果有关。

    ZenFS 通过允许用户配置数据区域容量限制来解决这一问题。当然如果文件大小不符合设置好的限制时，ZenFS会使用其区域分配算法来利用所有容量。经过检验文件大小增加带来的性能下降并不显著。

* 数据区域选择。ZenFS采用best effort的算法来选择存储RocksDB数据文件的最佳区域。RocksDB 通过在写入文件之前为文件设置写入生命周期标识来分离 WAL 和 SST 文件。ZenFS 首先尝试根据文件的生存期和区域中存储的数据的最大生存期来查找区域。仅当文件的生存期小于区域中存储的最旧数据时，匹配才有效，以避免延长区域中数据的生存期。

* 活动区域限制。ZenFS 必须遵守分区块设备指定的活动区域限制。要运行 ZenFS，至少需要三个活动区域，这些区域分别分配给日志、WAL 和压缩过程。为了提高性能，用户可以提高活动区域的数目。

* 直接 I/O 和缓冲写入。ZenFS 利用对 SST 文件的写入是顺序且不可变的，对 SST 文件绕过内核页面缓存，执行直接 I/O 写入。对于其他文件（如 WAL），ZenFS 把写入暂存到内存，在刷新内存时才填充到磁盘中。

### 效果

实验效果建立在顺序写的限制上

写入吞吐量，提高了1.7-2.7倍，减少了7 − 28%OP

随机读延迟更优(write 依旧是顺序的)
![img](https://img2023.cnblogs.com/blog/3067108/202403/3067108-20240304223859447-1911331186.png)

overwrite测试ZenFS的优化效果明显(183%)，因为overwrite需要较多的垃圾回收

没有写入速度限制的readwhilewriting，ZenFS的写速度优化效果明显

可以发现，XFS和F2FS的读延迟在P99和P99.99差别较大。而F2FS(ZNS)和ZenFS只有randomread测试差别较大，且三组测试延迟均更优，同时F2FS(ZNS)的延迟比ZenFS低

相交7%OP的stream SSD，ZenFS的性能依旧更优

### 可能的改进

## ZNS+: Advanced Zoned Namespace Interface for Supporting In-Storage Zone Compaction

### 背景

尽管LFS具有顺序追加的特性，但因为日志文件系统需要不断地将日志文件的段合并，这意味着大量的拷贝操作，这是相较传统文件系统要多承担的开销。
而ZNS必须和LFS搭配使用(基于顺序写限制)，所以主机需要承担LFS的段压缩开销（本应由SSD的垃圾回收机制代为承担）。
需要研究一种面向ZNS的LFS系统。

srds 传统的SSD也是一种顺序写友好的硬件，传统SSD+F2FS的搭配并不uncommon
这是因为SSD的擦除粒度大于写的粒度，覆写操作实际上是out-of-place manner的，即先追加写新的数据，然后将旧的数据标记为invalid。所以说本质上是顺序写友好的

> log-on-log problem
> Our work investigates the impacts to performance and endurance in flash when multiple layers of log-structured applications and file systems are layered on top of a log-structured flash device. We show that multiple log layers affects sequentiality and increases write pressure to flash devices through randomization of workloads, unaligned segment sizes, and uncoordinated multi-log garbage collection.

在传统的SSD的应用LFS文件系统就是一个log-on-log问题。因为SSD也按照先写再垃圾回收的方式运行，所以也是一个log系统。上层LFS文件系统的段压缩（垃圾回收）对于SSD来说只是正常的写操作，进一步引发下层的垃圾回收，总的写放大倍率将会是上层的写放大倍率乘上下层的写放大倍率。
而ZNS SSD则没有垃圾回收机制，避免了写放大。

要想利用SSD的闪存芯片并行性需要增大zone的大小（因为每个zone是隔离的，如果zone只包含一个channels，那么对zone的写入就没有并行性了），而zone越大段越大，压缩代价就越大

映射到page上的一段连续的logical block称为chunk
zone中的并行闪存芯片的数目称为$D_{zone}$
分别位于不同闪存芯片上的逻辑连续的若干chunk称为条带$stripe$

![img](https://img2023.cnblogs.com/blog/3067108/202403/3067108-20240311165400258-709078669.png)
一个zone包含一个或多个FBG(flash block group),一个FBG要实现并行需要包含多个来自不同并行闪存芯片的擦除块（通常FBG包含不同闪存芯片上块偏移相同的块），而一个条带上包含多个chunk(通常条带包含来自不同擦除块的page偏移相同的chunk)

chunk是copyback操作的基本单位，而同一flash plane内的chunk的copyback操作开销会得到优化

F2FS有6种段(hot、warm、cold 的 node 和 data)每种段同时至多打开一个。
node块包含一个data块的索引，data块则包含目录或用户文件数据。
hot或warm段中的cold块在段压缩过程中会被转移到cold段。
F2FS同时支持append logging和thread logging

⚠️thread logging比直接的段压缩性能更好：虽然要拷贝的块数是一样的(internal plugging)，但internal plugging可以在闪存芯片闲置时在后台进行，且要修改的元数据更少。
但同时thread logging也有一些缺点：thread logging写只能在与要写的数据具有相同类型的脏段上写(否则就会出现不同类型的数据混合在某个段的情况)，但段压缩则可以从所有脏段中找到有效块最少的段。其次，thread logging没办法把cold data移动到cold segement中，这样段中cold data每次参与到internal plugging中。(段压缩的结果段可以是多个不同类型的段)
其次thread logging的回收不方便，当文件系统回收某一个chunk时，可能并不会写入checkpoint，这样在thread logging眼中该chunk仍是有效的，依旧会参与到internal plugging中。

### 方法设计

ZNS+ 接口支持 zone内部压缩(IZC) 和 稀疏顺序覆写

* IZC：
  * 压缩操作可以细分为四个subtask: 👉 待压缩段选择，分配块保存结果、数据拷贝、元数据更新。其中数据拷贝的工作可以交给SSD而不是主机
  ![img](https://img2023.cnblogs.com/blog/3067108/202403/3067108-20240311190218301-501952618.png)
  * 在发送读磁盘请求之前必须分配内存页，这可能会导致page frame回收。同时因为LBA在磁盘上大概率是不连续的，需要依次发送多个读请求（尽管它们可能在同一个闪存芯片上），没法综合所有的读请求来利用闪存芯片的并行性。（例如在图a中每次请求可以并行地读两个闪存芯片，但闪存芯片0在第一次读请求到第四次读请求之间依旧有idle）
  * 而写阶段需要所有读请求都完成才会开始，这样当一个读请求完成后，在内存中的数据不会立刻写入到磁盘中，这段时间SSD也是idle的
  * 而IZC不需要将数据从磁盘拷贝到内存中，同时会更有效率地安排读写操作(利用并行性和copyback操作)
  * zone_compaction请求是异步的(与传统SSD自动进行GC相比，ZNS的GC是主机端用zone_compaction指令显式指定的，那么该指令执行的时机就有了自由，这一设计可以减少尾延迟)
* 稀疏顺序覆写：
  * 有一种名为 threaded logging的回收机制，该机制通过随机产生覆写操作，向脏段的无效空间中写新数据，来代替传统的段回收。但ZNS要求顺序写，所以不能直接通过该机制减少压缩开销。ZNS+选择松弛顺序写的限制，引入稀疏顺序覆写，threaded logging的稀疏顺序覆写在ZNS+中是被允许的，我们可以在覆写请求之间插入有效区域的顺序写，将稀疏顺序写转化真正的顺序写。
  * thread logging的覆写可以很容易与正常的写指令区分开来，因为覆写的LBA将会位于WA之前。收到覆写指令后需要对要跳过的有效块进行判断，为了减少这部分延迟，引入tl_open指令来提前创建闪存芯片中的有效块的位图。
  * F2FS的threaded logging并不直接写入原来的FBG(因为SSD必须先擦除才能写入)，而是分配一个新的LogFBG，在LogFBG上进行threaded logging写，而且有效块可以用copyback操作来拷贝。
    ![img](https://img2023.cnblogs.com/blog/3067108/202403/3067108-20240311221120609-918088273.png)
    完成一次threaded logging写之后，如果后面还有有效块，可以顺次将其也拷贝到Log FBG
    中，这样可以提前将WP移动到合适的位置
    如果一个并行闪存芯片上的低地址有效块都被拷贝了，而高地址块有效的话，可以顺次将其拷贝到Log FBG中，因为这满足每个闪存芯片上的顺序写要求，同样可以为WP的移动做准备（例如当WP还在chunk1时，就可以将chunk3的内容拷贝到Log FBG中了）

文件系统引入两项技术来与ZNS配合

* 回写感知的块分配：
    回拷操作用于在同一闪存plane内的闪存页面之间复制数据，而无需进行片外数据传输，已经应用到了标准NAND接口中。新的文件系统在块分配时采用放置策略来利用回拷操作(将copy的源逻辑块和目的逻辑块映射到同一个闪存芯片上)
    ![img](https://img2023.cnblogs.com/blog/3067108/202403/3067108-20240311223728440-75688065.png)
    核心就是在目标段所在的FBG中分配恰好能填满的空间（一般情况下），之后先尽可能地对chunk进行copyback（chip ID可以在map机制中可以很容易地得到），之后用read&write指令将剩下的块填到分配的空间中去
* 混合段压缩方式：使文件系统能够分析开销判断该采用IZC还是稀疏顺序覆写来回收段
    为了减少pre-valid 块，引入周期性设置checkpoint的机制，当pre-valid块的数目大于与之 $θ_{PI}$ 时对元数据进行更新
    引入回收代价计算模型，对开销进行分析

### 效果


### 可能的改进

thread logging 具有一些缺点，例如cold data移动的问题和pre-valid block的问题，或许可以针对这些缺点进行改进

## eZNS: An Elastic Zoned Namespace for Commodity ZNS SSDs

### 背景



### 方法设计

zone管理器：管理zone分配和活动资源

分层I/O调度器：读拥塞控制、写权限控制

### 效果

### 可能的改进


## Separating Data via Block Invalidation Time Inference for Write Amplification Reduction in Log-Structured Storage

### 背景

为什么要坚持使用Log-structured 存储
![img](https://img2023.cnblogs.com/blog/3067108/202403/3067108-20240316151233544-916223759.png)

boxplot图（箱线图）是一种用于显示数据分布的常用统计图表。

* 箱体（Box）：箱体代表了数据的中间50%（第二四分位数（Q2）），上下两边分别是第一四分位数（Q1）和第三四分位数（Q3）。箱体的长度代表了数据的分布范围。
* 中位数（Median）：箱体内的中线代表数据的中位数，也就是第二四分位数（Q2）。
* 上下边缘（Whiskers）：通常是从箱体延伸出来的线段，它们表示数据的范围。Whiskers的长度通常是箱体长度的1.5倍IQR（四分位数间距，即Q3-Q1）。
* 异常值（Outliers）：箱线图中的个别点，它们超出了上下边缘的1.5倍IQR的范围。

BIT是block invalidation time的缩写，这里的时间是用写入的块数为单位来标识的
SepBIT是一种新算法，它根据存储负载推断BIT，并将具有相近BIT的块放到一组，进而减少写放大
这种推断基于实际存储负载的写入倾斜特征(基于对阿里云和腾讯云的分析)

现有的块放置算法基于块温度(写/修改 频率)来分组

当一个段被写满之后，称之为sealed segment，反之为open segment，open segment不需要垃圾回收，只需要接着追加写即可。

常见的垃圾回收目标段选择策略有：

* Greedy  👉 回收invalid块占比最高的若干sealed segment（可能会选择包含热数据的段，热数据本来不需要copy因为他们很快就会失效）
* Cost-Benefit 👉 选择  垃圾占比GP * 段满之后经过的时间/ (1 - GP) 值最高的若干段。该策略可能考虑到，段满的时间太短hot block还来不及失效需要再等等，尽管该算法较复杂，但一般而言CB的效果是最好的。

块的lifespan被定义为从该块开始被写到它被无效化的这段时间总负载的写入字节数
卷的write working set (WSS)即为对该卷的写操作的总字节数(更新操作不是unique LBA 不算在内)

有以下三个发现：
以下发现都是对每个卷单独进行分析得到的，且累加占比是满足条件的卷占所有卷的比例

* User-write 块通常有较短的lifespan，更准确地说其lifespan通常小于该卷的总写字节数
  ![img](https://img2023.cnblogs.com/blog/3067108/202403/3067108-20240316195240339-1929694785.png)
  例如，半数卷有超过 79.5% 的用户写入块的生命期小于其写入 WSS 的 80%，有超过 47.6% 的用户写入块的生命期仅小于写入 WSS 的 10%。（横轴是统计进去的所有卷中满足条件的用户写入块占比的最小值？？？？）
  而GC回收块的lifespan一般较长，因为GC回收块写入的数据是GC回收段中有效块的数据。
  根据我们的段选择策略，Greedy选择GP最高的段回收，说明该段有很多无效块，那么相应地，还剩下的有效块的寿命倾向于比较长(因为根据寿命的定义，如果有效块比这些无效块更早写入的话，那么有效块的寿命比这些无效块的寿命都长)。
  而Cost-Benefit选择GP较高且存在时间比较长的段，这样的段中的有效块寿命也比较长。

  ⚠️这里认为寿命长是该块从写入到最后无效化的lifespan长，而该块从拷贝到GC回收块到无效化的lifespan我觉得无从推断。除非能够证明块的lifespan长的非常长，短的非常短。
* 经常更新的块的lifespan变化很大
  使用CV(coeffieient of variation)变异系数来衡量变化度，CV = σ / μ，σ为标准差，μ为平均值，通常高于1的CV值表示变化度较大
  ![img](https://img2023.cnblogs.com/blog/3067108/202403/3067108-20240317114731507-1529433759.png)
  例如，在前 1%、前 1-5%、前 510%和前 10-20% 四个组别中，分别有 25%的卷的 CV 值超过 4.34、3.20、2.14 和 1.82。
  这意味着现有的根据块温度来放置数据的策略并不合适。
* 不经常更新的块占WSS比例很大且其lifespan也变化较大
  ![img](https://img2023.cnblogs.com/blog/3067108/202403/3067108-20240317155938933-1533765530.png)
  不经常更新的块为trace中更新次数少于4次的块
  在 25% 的数据卷中，71.5% 以上很少更新的数据块的寿命小于 0.5 倍写入 WSS。其余四组的百分比中值分别为 24.9%、8.1%、3.3% 和 2.2%。 ？？？
  这意味着不经常修改的块的lifespan既有长的也有短的
  这同样说明了根据块温度来放置数据的策略并不合适。

### 方法设计

![img](https://img2023.cnblogs.com/blog/3067108/202403/3067108-20240317191541607-147402803.png)
根据发现1，分为user-write block和GC block 
根据发现2，推断BIT将不同lifespan的数据放到不同的class里面

如何推断lifespan
![img](https://img2023.cnblogs.com/blog/3067108/202403/3067108-20240318154732055-120344455.png)

对于user-written 块：
如果写的数据是全新的，那么该user-written块的estimated lifespan为 $\infty$
如果是对原先数据的更新，那么写入的user-written块的estimated lifespan为原先数据所在块的lifespan
直觉是：某块的数据是对旧的short-lived数据的更新，那么该块也很可能是short-lived的
证明：基于访问特征倾斜的假设(即对块的访问是不均匀的)，用Zipf分布对块的访问概率进行建模。该分布的特征是，概率与它在排序中的位置成反比关系，换句话说，访问概率第二高的块大约是概率最高的块的一半，概率第三高的块大约是概率最高的单词的三分之一，以此类推。$$p_i=\left(1 / i^\alpha\right) / \sum_{j=1}^n\left(1 / j^\alpha\right) \qquad 1 \le i \le n \quad \alpha \ge 0, \quad $$ 
$\alpha$越大，说明分布越倾斜
则 $1-p_i$ 就是第i块不被访问的概率，要想计算lifespan小于 $v_0$ 的概率，那么就要计算第一次写到第 $i$ 块且之后 $v_0$ 次写入至少有1次访问到第 $i$ 块，即 1 - 连续 $v_0$ 次都没有写入第 $i$ 块的概率， $1 - (1 - p_i)^{v_0}$，因为可能写到1到n中任意一块，lifespan小于 $v_0$ 的概率为一个累加，$\sum_{i=1}^{n} {上式}$ ，进而得到
$$
\begin{aligned}
& \operatorname{Pr}\left(u \leq u_0 \mid v \leq v_0\right)=\frac{\operatorname{Pr}\left(u \leq u_0 \text { and } v \leq v_0\right)}{\operatorname{Pr}\left(v \leq v_0\right)} \\
& =\frac{\sum_{i=1}^n\left(1-\left(1-p_i\right)^{u_0}\right) \cdot\left(1-\left(1-p_i\right)^{v_0}\right) \cdot p_i}{\sum_{i=1}^n\left(1-\left(1-p_i\right)^{v_0}\right) \cdot p_i} .
\end{aligned}
$$
对该公式进行分析，得到v0越小，u < u0的条件概率越大。
这一点很好理解，因为分布是倾斜的，v0越小说明该块越有可能是访问概率高的块，进而下次访问的间隔也会小。
对Trace的统计也验证了理论分析。

对于GC-rewritten块：
定义age为数据从被写到被回收至GC块这中间的workload的写入量
定义residual lifespan为该数据从写到GC块到其最终无效的过程中的user-written的写入量
那么该数据中的lifespan应是age + residual lifespan
SepBIT不直接分析GC-written块的lifespan，而是分析其前身user-written块的lifespan。当一个user-written块的lifespan超过某个阈值，就认为该块已经被回收到GC-written块了，对应数据的age就是这一阈值。(???为什么要这样认为)
直觉是 数据的age短，residual lifespan也就短
证明：$g_0$是数据的age，也就是阈值，$r_0$是其residual lifespan，那么该数据的lifespan就是 $g_0 + r_0$
和前面计算lifespan小于 $v_0$ 的方式类似，lifespan大于 $g_0$ 的前提下，小于 $g_0 + r_0$ 的条件概率为
$$
\begin{aligned}
& \operatorname{Pr}\left(u \leq g_0+r_0 \mid u \geq g_0\right)=\frac{\operatorname{Pr}\left(g_0 \leq u \leq g_0+r_0\right)}{\operatorname{Pr}\left(u \geq g_0\right)} \\
& =\frac{\sum_{i=1}^n p_i \cdot\left(\left(1-p_i\right)^{g_0}-\left(1-p_i\right)^{g_0+r_0}\right)}{\sum_{i=1}^n p_i \cdot\left(1-p_i\right)^{g_0}}
\end{aligned}
$$
给定 $\alpha、r_0$， $g_0$ 越大，条件概率越小，也即 $g_0$ 越大，该数据的residual lifespan 小的概率越低。
这点也很好理解，因为分布是倾斜的，上面说明了数据的lifespan与user-written写入的是哪一块有关，所以同一块中无效数据和有效数据的lifespan应该是相差不大的。而其中的有效数据正是GC-written块的数据来源。age反映了无效数据的lifespan，假设差别不大是指相差20%左右，那么显然age越大，20%对应的residual lifespan也就越大。
对Trace的统计也验证了理论分析。

代码实现：
![img](https://img2023.cnblogs.com/blog/3067108/202403/3067108-20240318223303644-2146394193.png)
$\mathcal{l}$ 为一个阈值，用于给数据按BIT进行分类，它通过计算最新被回收的段的平均lifespan得到。
$\mathcal{t}$ 是user-write的时间戳，每次写操作增加1。

数据结构：实现SepBIT只需要维护每块的last user write time，该值作为元数据保存在disk中。

对于user-written块：为了正确放置，需要知道原块的lifespan，如果去读该块的disk显然是不划算的。但我们只要知道其lifespan是否小于 $\mathcal{l}$ 即可， 所以维护一个FIFO来记录最近的若干次写操作，该队列的长度稳定后应该要大于 $\mathcal{l}$ 。做法是对于要新写入的LBA，在FIFO中查找它，如果在队列中能找到它且其下标小于 $\mathcal{l}$，那么就说明原块的lifespan小于 $\mathcal{l}$ 。
该队列的长度应该根据 $\mathcal{l}$ 动态调整。
同时设置一个map，用来表示不同LBA在队列中的最新位置，这样可以加快判断，只需要在元素入队出队时更新map即可。

对于GC-written块：因为GC块的写本来就需要读user-written块的有效数据，所以一并读metadata并不会增加多余的开销。

### 效果

![img](https://img2023.cnblogs.com/blog/3067108/202403/3067108-20240318230310402-1329847947.png)
80 % 优化效果蛮好，相关系数大于0.75

对于GC很少的卷，SepBIT由于要维护FIFO，吞吐量相较别的放置策略损失了3.0% ~ 6.9%

### 可能的改进


## MIDAS: Minimizing Write Amplification in Log-Structured Systems through Adaptive Group Number and Size Configuration

### 背景

![img](https://img2023.cnblogs.com/blog/3067108/202403/3067108-20240323165258997-972712449.png)
不同的数据放置策略对不同lifespan的块的预测准确率不同(各有所长)，红框部分被作者解释为当实验结束时还有一些coldest block(C6)正准备被划分为coldest block(C6)。
![img](https://img2023.cnblogs.com/blog/3067108/202403/3067108-20240323170151797-1694524387.png)
即使是ORA，如果不精心选择组的数量和大小也会导致suboptimal

![img](https://img2023.cnblogs.com/blog/3067108/202403/3067108-20240323171411642-1706006996.png)
考虑不同victim segement选择策略对WAF的影响，可以发现CB是最好的，而用ORA进行数据放置时，即使是FIFO，其WAF也是一样小的

### 方法设计

lifespan预测和group configuration是减少写放大的两个关键点。

#### lifespan预测

通过对块分group来实现

![img](https://img2023.cnblogs.com/blog/3067108/202403/3067108-20240323171636488-1209402581.png)

有1个HOT group，和N个COLD group，每个group有一定大小，当满了之后进行GC。
先根据update interval判断是不是HOT块，如果是将其送入HOT group，如果不是送入$G_1$。
COLD group顺次连接，$G_i$垃圾回收的有效块写入$G_{i+1}$，这样同一COLD group的age是相同的，其estimated lifespan也相同。（HOT写入 $G_1$， $G_N$仍写入 $G_N$）

通过和SepBIT类似的方式根据update interval区分冷热块，即仅当lifespan为1/3 最近被回收段的平均lifespan时才从G1中提升块到HOT group中。

#### group configuration

通过UID和MCAM来估计group的调整对WAF的影响来实现

MCAM是一个基于马尔可夫链的分析模型

将block划分为valid和free态，valid又细分为 $V_H, V_{G_i}$，表示在HOT或 $G_i$ group中。给出这些状态的转移概率，当整个系统收敛时，计算其WAF用来估计真实的WAF。
![img](https://img2023.cnblogs.com/blog/3067108/202403/3067108-20240323201050828-325856446.png)
实验证明给出准确的转移概率后，WAF估计的错误率不会超过2.82%

当然没办法事先给出精确的转移概率，MiDAS通过UID来估计转移概率
update interval distribution中横轴是update interval其单位是写入的块数，纵轴是块还valid的概率。

COLD group的转移概率：

![img](https://img2023.cnblogs.com/blog/3067108/202403/3067108-20240323204721466-1304403288.png)
则从0~$G_1$size的UID的积分是 $G_1$满时，其中的块还valid的概率。则据图转移概率为0.6/1 = 0.6
而进入$G_2$的块，首先它们一定在$G_1$中经历过$G_1$size次write了，则从$G_1$size~$G_1$size + $G_2$size/($G_1$到$G_2$转移概率) 的UID的积分是 $G_2$满时，其中的块还valid的概率。则据图$G_2$到$G_3$转移概率为 0.14/0.6 = 0.35

此类分析对于 $G_N$不适用，其转移概率照搬了Desnoyers的工作

HOT group的转移概率：

根据HOT group的定义，lifespan要小于1/3 阈值才会从free转移到HOT group，假设算法可以正确分辨hot block，则0 ~ 1/3  阈值 的 UID的积分即为free到HOT group的转移概率，但作者说用0 ~ 阈值的积分预测错误率仍小于 5 %，所以采用了 0 ~ 阈值的积分。
![img](https://img2023.cnblogs.com/blog/3067108/202403/3067108-20240323205947020-1598724359.png)

在区分了HOT 和 COLD group后，UID分别为
![img](https://img2023.cnblogs.com/blog/3067108/202403/3067108-20240323211045428-1722000176.png)
根据分析，HOT group的update interval全部小于阈值，故超出阈值后其valid概率为0；COLD group的update interval全部大于阈值，故小于阈值的部分其valid概率为1。

有了UID就可以根据group configuration计算出转移概率，并用MCAM估计WAF了。
但遍历所有configuration找出最优配置并不现实，故采用GCS启发式贪心算法找出足够优的解。

![img](https://img2023.cnblogs.com/blog/3067108/202403/3067108-20240323211831101-1635330874.png)
其首先将UID小于1segment的块放到HOT group中，其余的块放到$G_1$，据此设计一开始的HOT和 $G_1$ 的大小。
之后其用MCAM开始计算WAF，然后将HOT段的UID范围加1segment，重新计算，直至WAF不再减少。
这样HOT group的大小就最终确定了。递归地对$G_1$做分割处理即可。
当递归至一定程度时，WAF没有明显的降低了就停止，此时组的数量和大小就初步确定了。

$G_N$ 的写放大会比较高，因为它得到的块少(???why)，所以通过将其他组的segmetn重新分配给 $G_N$ 来解决这一问题。在分配1个segment时会重新计算WAF，如果WAF降低就重复这一过程，否则取消。

仅在UID更新时重做GCS算法。

MiDAS维护两个UID，一个为当前使用的，一个为当前得到的。在一个任期结束时，用新的UID估计WAF，如果WAF减少的超过一个阈值，就使用新的UID，否则在下个任期接着用旧的。

MiDAS根据估计的WAF来调整组，这建立在I/O 特征的可估计性上。为了避免当I/O 特征难以估计时，使用MiDAS带来的劣化效果，引入了irregular I/O 处理机制。

MiDAS定期检查其估计的转移概率和真实概率的误差，当检测到 $G_k$ 至 $G_{k+1}$ 的转移概率误差很大时，就停止调整除 $G_k$ 以外的组，并将从 $G_k$ 一直到 $G_N$ 的所有组都合并为 $G_k$，之后退化为 age-based MiDA。

### 效果


### 可能的改进