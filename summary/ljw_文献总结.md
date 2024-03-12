

### 1.1 (23' CLUSTER) Performance Characterization of NVMe Flash Devices with Zoned Namespaces (ZNS)

ZNS SSD的write命令指定目标块地址，因此SSD在IO调度时无法对多个写请求重新排序，否则造成地址错误，因此写并发度为1。

与之对应的，ZNS SSD的append命令只要求返回写入后的地址，因此可以进行重排序。

ZNS zone的状态转换图如下所示。相对应的ZNS提供的区域管理命令有open(打开一个区域，一个区域只有被打开才能进行读写，同时打开的区域有数量限制)、close(关闭一个区域)、finish(把一个活动区域标记为full，进行垃圾回收等)。

![ZONE 状态转换][cl23-1]

![ZONE 状态转换](https://github.com/ljwhust/pictures/blob/main/ZNS/CLUSTER%2023'/ZNS%20zone%E7%8A%B6%E6%80%81%E8%BD%AC%E6%8D%A2.jpg)

使用linux kernel block layer(with io_uring) 和 用户端的SPDK 共同测试ZNS的性能。因为fio和linux block layer不支持append和区域管理相关操作；SPDK只是用户端框架，不支持IO调度，因此一次只能发出一个write请求(linux block layer的IO调度采用mq-deadline调度，将application对同一区域的多个连续的LBA write合并为一个较大的写)。

评价变量：throughput (the number of operations or bytes per second) 和 operation latency (the time each operation takes)

1  append 和 write 命令的表现，见下图。(不考虑并发)

![fig2][cl23-2]

![fig2](https://github.com/ljwhust/pictures/blob/main/ZNS/CLUSTER%2023'/fig2.jpg)

![fig3][cl23-3]

![fig3](https://github.com/ljwhust/pictures/blob/main/ZNS/CLUSTER%2023'/fig3.jpg)

观察 #1：LBA format(512B or 4KB)会对write和append延迟产生重大影响。

观察 #2：使用 SPDK 存储堆栈可实现最低的延迟。此外，mq-deadline 调度器增加了不可忽略的延迟开销。

观察 #3：写入和追加吞吐量取决于请求大小。请求大小对性能的影响在追加和写入操作之间有所不同。追加吞吐量低于写入吞吐量不是 ZNS 设计所固有的，而是取决于固件。预计较新的 ZNS 设备的追加吞吐量将增加。maximal throughput(4/8) vs maximum throughput in bytes

观察 #4：写入操作的 I/O 延迟低于追加操作。

建议 #1：使用写入而不是追加操作来实现低 I/O 延迟（它们之间的差异可能高达 23%），并使用 SPDK 存储堆栈，因为它提供最低的 I/O 延迟。

2  可扩展性：区域内与区域间，见下图。

![fig4][cl23-4]

![fig4](https://github.com/ljwhust/pictures/blob/main/ZNS/CLUSTER%2023'/fig4.jpg)

通过 SPDK 测量(一组)random read 和 sequential append 和 区域间的write命令性能；使用mq-deadline 和 io_uring 的 Linux kernel layer测量区域内的write命令性能。

观察 #5：区域内并行性比区域间并行性实现更高的整体 IOPS。区域间可扩展性进一步受到最大开放区域限制。

观察 #6：追加吞吐量与区域间请求还是区域内请求无关。

观察 #7：在单个区域中，读取操作的可扩展性最好，其次是写入和追加操作。在队列深度为 4 之前，在单个区域中，追加操作的性能优于写入操作，但在更高的队列深度下，写入操作的性能优于追加操作。

观察 #8：对于大型（即 >=8 KiB）I/O 请求，区域内追加和区域间写入操作的性能达到设备限制。

建议 #2：首选区域内而不是区域间并行;前者非常适合追加和读取操作，而后者最适合写入操作。在较大的请求大小（即 >=8 KiB，接近内部块大小）下发出 I/O，因为较大的请求在更高的并发级别下扩展得更好。（？）

3 zone状态转换开销，见下图。

![fig5][cl23-5]

![fig5](https://github.com/ljwhust/pictures/blob/main/ZNS/CLUSTER%2023'/fig5.jpg)

观察 #9：显式和隐式区域开放转换之间没有性能差异，打开/关闭的成本是微不足道的。

观察结果 #10：区域占用率（或利用率）对重置和完成操作的性能都有重大影响。finish操作都是一项非常昂贵的操作。

建议 #3：避免finish操作，尤其是对于部分写入的区域。通过利用区域内并行性减少活动区域的数量，最大限度地减少需要finiah的区域数量。

4 read write append操作的相互干扰，见下图。

![fig6][cl23-6]

![fig6](https://github.com/ljwhust/pictures/blob/main/ZNS/CLUSTER%2023'/fig6.jpg)

观察结果 #11：与普通 NVMe 设备相比，ZNS 设备在存在并发写触发垃圾回收的情况下，读写性能更加稳定。

建议 4:无需考虑ZNS GC导致的性能波动。

5 并发IO对reset操作性能的干扰，见下图。

![fig7][cl23-7]

![fig7](https://github.com/ljwhust/pictures/blob/main/ZNS/CLUSTER%2023'/fig7.jpg)

观察结果 #12：重置操作不会干扰读取、写入或追加延迟。

观察结果 13：读取、写入和追加操作对重置延迟的干扰很大。

建议 5：reset操作可与read write append 同时进行。


### 1.2 (Hotstorage'22) Lifetime-Leveling LSM-Tree Compaction for ZNS SSD

背景知识

ZNS

Zoned NameSpace Interface提供了划分为固定大小区域(zone)的接口。每个zone由若干个逻辑块组成；zone内允许随机读取，但必须按顺序写入、按zone擦除。
(个人理解：对于传统的块接口SSD，按页写入、按块擦除，地址映射、垃圾回收等由FTL层负责。此时如果想要修改一个数据，大概需要把旧地址数据标记为无效、在新地址写入新数据、更改地址映射表。如果每个块里被标记为无效的数据超过一定比例，那就要执行垃圾回收(不管这个比例设置为多少，垃圾回收早晚都是要进行的)，那就要把剩余的有效数据写入新地方并更改它们的地址映射、然后擦除这个块，所以造成了写放大，还会导致性能不可预测等问题。但是对于ZNS SSD，Host-Side FTL负责地址映射等功能，ZNS内部只维护zone级别的粗粒度地址映射(WA)，所以在ZNS端只需要把旧数据标记为无效、顺序写入新数据、更改WA，从而避免了ZNS内的垃圾回收。但是还是可能会存在空间放大问题(因为可能会有一些无效数据)，而且最终也会导致写放大(写满到一定程度了，还是需要写到其他zone、然后擦除整个zone)。)

LSM Tree

LSM Tree由Memtable(存储在内存中)和多层SSTable(存储在磁盘中)组成。当Memtable达到一定大小，会下刷成为L$_0$-SSTable。SSTable是一个有序且不可变的key-value存储结构，当L$_k$的SSTs总大小达到阈值，会选中L$_k$的一个SST与下层的某些SSTs进行合并(compaction)，合并后的新SSTs位于L$_{k+1}$中。
除L0-SSTable外，其余level的SSTable无重叠key。

LSM Tree的顺序写特性适合ZNS，但是传统的LSM-Compaction方法可能会导致ZNS的空间放大与写放大。

传统的LSM-Compaction方法：

(1) 确定level L$_i$的一个SST T$_i^j$

(2) 选择level L$_{i+1}$ 中key范围与SST T$_i^j$ 有交集的所有SSTs

(3) 合并(1)(2)中的所有SSTs，在level L$_{i+1}$ 创建新的SSTs

(4) 删除(1)(2)中的所有SSTs

(5) 更改L$_i$中的CP(Compaction Pointer)，指向下一个要合并的SST的start key location

导致空间放大和写放大的两种情况：

第一个问题：不同level的SSTs存储在同一个zone内

解决方案：为每一个level的SST分配不同的专用zone(不在本论文研究范围)

第二个问题：long-lifetime SST && short-lifetime SST

![hs22-1][hs22-1]

![hs22-1](https://github.com/ljwhust/pictures/blob/main/ZNS/HotStrorge%2022%E2%80%98/1-2.jpg)

解决方案：

Lifetime-Leveling LSM-Tree Compaction(LL-Compaction，见下文)

方法设计

Compaction Window Expansion

(2)' 除了(2)中选取的SSTs，还要选取这样的level L$_{i+1}$ SST：{start key of SST > end key of compaction window} 并且 {key范围 of SST不与level L$_{i}$层的任何SSTs有交集}(这是为了避免long-lifetime SST，上图可以解释)。

SST Split

![hs22-2][hs22-2]

![hs22-2](https://github.com/ljwhust/pictures/blob/main/ZNS/HotStrorge%2022%E2%80%98/3.jpg)

(3)' (3)中新建SSTs的时候：跨越Next CP_i的，以Next CP_i分割成两个SST，后一个SST专门存储在T-Zone中(因为这个SST是短寿命的，马上就要继续被合并)，并且T-Zone可以使用内存空间(进一步提高性能)；跨越CP_{i+1}的，以CP_{i+1}分割成两个SST(这是为了保证CP始终指向SST的start key，进而保证选择合并SST的有序性)。

实验效果

背景：LL-Compaction，Host-side GC(Greedy：free zone个数为1时触发GC，GC选择复制开销最小的zone)，level separation technique

对比方法：BL(the GC-disabled original LevelDB) 、GC(GCenabled LevelDB)、LS(level separation technique in addition to GC)、Gear(Gear Compaction)

没有ZNS端的GC开销、Compaction开销小于Gear；SST Split导致的SST文件数量增加成本 & Window Expansion导致的SST复制开销 小于 性能提高；压缩中涉及的SST数量不同，可能会导致性能波动。



https://zhuanlan.zhihu.com/p/669385346






