# 1.ZoneKV: A Space-Efficient Key-Value Store for ZNS SSDs
## 背景知识

[论文原文在此](https://ieeexplore.ieee.org/document/10247926)
本文主要与ZenFS进行对比<br>
关于SSTable的low或者High的意义<br>
**high**:level 0 ,level 1<br>
low :level 10...<br>	
ZNS SSDs (Zoned Namespace SSDs) , is a new kind of SSDs adopting the new NVMe storage interface. ZNS SSDs divide the logical block address (LBA) into multiple zones of the same size. Each zone can only be written sequentially and used again after resetting.<br>
ZNS SSD将GC和data placement的任务分配给了host端，让用户而非device来决定如何充分发挥ZNS SSD的优势。ZNS SSD的特性使之天生就与LSM树适应（log-structured merge tree）。LSM树将随机写转换成顺序写，提高了写性能。<br>
![img](https://cdn.nlark.com/yuque/0/2024/webp/42361192/1705396938501-270f9c25-0619-4959-a12c-4c3755b38414.webp?x-oss-process=image%2Fformat%2Cpng%2Fresize%2Cw_720%2Climit_0)

关于LSM树 具体可看[此文](https://zhuanlan.zhihu.com/p/181498475 "LSM树详解")。

LSM树 含有一个内存部件，其中**内存**中存有**Memtable**以及**immutable MemTable**。<br>
首先当写入键值对放入**WA**(write-ahead logging，预写式日志，方便之后进行数据恢复,因为写在MemTable中的内容并没有被写入到SSD中,only append mode)以及**Memtable**当中。
**Memtable(only append mode)**中存储的是key-value对，当Memtable中存储的key-value对 达到一个阈值后，Memtable will convert to **Immutable Memtable**(only for reads)，immutable memtable是为了不阻塞Memtable的写操作，因为Immutable memtable中的内容会 simultaneously 写到SSD中去。之后Immutable MemTable中的内容会被存储到ZNS SSD中以SSTable的形式。<br>
SSTable(Sorted String Table):亦是存储的键值对，但是其中的键值对是按照Key值排序了的。存储在磁盘中的结构呢。
![img](https://cdn.nlark.com/yuque/0/2024/png/42361192/1705396923330-2ae496ea-9edc-4e59-9962-ae9a13d0e800.png?x-oss-process=image%2Fresize%2Cw_720%2Climit_0)
&emsp;&emsp;布隆过滤器可以加速查找。可以类比理解为STL库中的set，然后调用find函数。
因此当MemTable达到一定大小flush到持久化存储变成SSTable后，**在不同的SSTable中，可能存在相同Key的记录**，当然**最新的那条记录才是准确的**。这样设计的虽然大大提高了写性能，但同时也会带来一些问题：
- 问题
1. **冗余存储**，对于某个key，实际上除了最新的那条记录外，其他的记录都是冗余无用的，但是仍然占用了存储空间。因此需要进行Compact操作(合并多个SSTable)来清除冗余的记录。
2. 读取时需要从最新的**倒着查询**，直到找到某个key的记录。最坏情况需要查询完所有的SSTable，这里可以通过前面提到的索引/布隆过滤器来优化查找速度。

&emsp;&emsp;
- 概念介绍
1. 读放大:读取数据时实际读取的数据量大于真正的数据量。例如在LSM树中需要先在MemTable查看当前key是否存在，不存在继续从SSTable中寻找。
2. 写放大:写入数据时实际写入的数据量大于真正的数据量。例如在LSM树中写入时可能触发Compact操作，导致实际写入的数据量远大于该key的数据量。
3. 空间放大:数据实际占用的磁盘空间比数据的真正大小更多。上面提到的冗余存储，对于一个key来说，只有最新的那条记录是有效的，而之前的记录都是可以被清理回收的。<br>
![img](https://cdn.nlark.com/yuque/0/2024/png/42361192/1705397979688-2687b250-2b22-4142-96b2-ba6af90446d0.png)  
&emsp;&emsp;LSM树逻辑结构图如上<br>
&emsp;&emsp;每个level亦有一个size limit ，当一个level的sstable达到一个limit后，通过压缩合并SSTable,将其new sstable push down to 下一个level。<br>

&emsp;&emsp;&emsp;目前ZenFS（作为一个middleware）支持LSM树结构。Zenfs支持data placement 以及 zone cleaning。<br>
&emsp;&emsp;ZenFS的缺陷是:ZenFS does not fully consider the lifetime of SSTables and does not have complete GC logic.进而导致，有着不同生命期的SSTables可能会被存储在同一个zone里，进而导致严重的空间放大问题(不管是raw rocksdb还是说 zenfs 都会存在这种问题)。<br>

-------

## 方法设计
&emsp;&emsp;通过直接修改RocksDB的源码中的Filesystem Warpper类来实现ZoneKV。<br>

&emsp;&emsp;该文章的主要思想感觉和LIZA(lifetime zone allocation algorithm)(Hot Storage)一般无二。<br>

ZoneKV adopts a lifetime-based zone storage model and a level-specific zone allocation algorithm to make the **SSTables in each zone have a similar lifetime**, yielding lower space amplification, better space efficiency, and higher time performance than ZenFS. They propose to store SSTables with a similar lifetime within the same zone to reduce space amplification of the LSM-tree.（换句话说 是 要将具有相似lifetime的SSTables存在同一个zone里，减少之后的amplification的问题。）

&emsp;&emsp;**definition:**<br>

&emsp;&emsp;Lifetime of SSTable :SSTable创建时间和无效时间(invalidated包括：因compaction而导致原SSTable无效 和 因为SSTable得到更新而导致现SSTable无效)之间的间隔<br>

**idea**:

&emsp;&emsp;Store all SSTables with similar lifetimes into the same zone.
<br>
&emsp;&emsp;然而，我们事前不可能知道SSTable的lifetime（与optimal算法有异曲同工之妙）。规律是：level越高，其lifetime越小、短，因此可以根据SSTable所处的level来推断其 lifetime 。
- policy
1. 对于L0 L1，采用tiering compaction策略。it invalidate all the SSTables in L0 and merge them with SSTables in L1 and produce new SSTables that are finally written to L1.（将L0的所有SSTable无效，然后将L0所有的SSTables与L1的SSTables合并。而且SSTable in L1也会因为L0的tiering compaction策略，最后和L0合并----------进而导致L0 和 L1的SSTable有相似的Lifetime） 
2. 对于L2 L3...... 采用leveling compaction策略。leveling compaction会用循环赛的方式选择 L i 的一个SSTable  A，然后将其与L i+1的所有与SSTable A关键字重叠的 SSTables 进行合并
<br>
  
- 英文总结  
1. Lifetime-Based Zone Storage. ZoneKV proposes to maintain SSTables with a similar lifetime in one zone to reduce space amplification and improve space efficiency. ZoneKV does not maintain the lifetime information for each SSTable explicitly but uses the level number of each SSTable to infer the lifetime implicitly. Such a design can avoid memory usage and maintain cost of the lifetime information. 
2. Level-Specific Zone Allocation. First, ZoneKV proposes to put L0 and L1 in one zone because the SSTables in these two levels have a similar lifetime. Second, ZoneKV horizontally partitions all the SSTables in L i (i ≥ 2) into slices,
and each slice is stored in one zone.<br>
![img](https://cdn.nlark.com/yuque/0/2024/png/42361192/1705471487148-62b5ae44-7bbf-4981-9458-aa3a61da494d.png)
&emsp;&emsp;从上图可以看出，尽可能把 lifetime 相近的文件（在同一个level的文件）放在同一个zone中。<br>
&emsp;&emsp;Log文件**单独存储**在Disk中的一个部分，Log文件append-only and never update therefore their lifetime can be regarded as infinity.<br>
&emsp;&emsp; **传统的磁盘**IO栈需要一系列IO子系统（例如Kernel file system、block layer、IO scheduling layer 和  block device driver layer），这些IO子系统形成的长链会降低数据存储的效率，较少磁盘吞吐和数据存储效率. <br>
&emsp;&emsp;ZoneKV则与ZNS SSD直接对接,跳过了中间的这些IO栈,进而提高了性能.<br>
&emsp;&emsp;data placement也有讲究:Level 0和 Level 1的SSTables被放入到同一个zone里,然后其余Level i (i >= 2 )的SSTables被放到多个不同的zone中(1.是因为一个zone放不下level 低的所有SSTables  2.是因为我们的同一个level的SSTables的Lifetime 亦有差异 由于是用rocksdb实现的, rocksdb 又偏爱 先对小keys的SSTables进行合并).<br>
&emsp;&emsp;So,对于较低level 的 SSTable 也要对其进行分组(group),根据key range分组(因为key range 相同或相近的 其生命周期也往往相同)来将同一个 group 放到同一个 zone里.<br>

```c++
//search for suitable zones

Function Find_zone(SSTable)
	lifetime_SSTable ← Get_lifetime(SSTable);
	zone ← Get_first_active_zone();
  //遍历所有的ACTIVE ZONE看还有没有  有多余空间的active_zone 用于分配
  while zone do
	//如果该SSTable的lifetime与某个zone的lifetime
   	//相同且该zone有多余空间,则将该SSTable insert
	//到该zone中
  	if lifetime_SSTable = zone.lifetime_zone and
    		Is_not_full(zone) then 
        return zone;
    zone ← Get_next_active_zone();
  end
	//该循环结束 表明没有合适的ACTIVE_ZONE来装下此SSTAble
	//active zone应当是活跃的zone,已经allocate的zone,不为空的zone
    //MAX_NUM_ACTIVE_ZONE:应该是一个SSD中可以有MAX_NUM_ACTIVE_ZONE个ZONE
	//可以被用来分配,还要留下ALL_ZONE 减去(minus) MAX_NUM_ACTIVE_ZONE个zone用于zone cleaning
 
 //当仍然有active_zone的空间时,分配一个新zone给该SSTable 
 if Get_num_empty_zone() > 0 then
      if  Get_num_active_zone() <
      		MAX_NUM_ACTIVE_ZONES then
          zone ← Assign_new_zone();
          return zone;
 //没有active_zone的空间时,关掉一个active_zone再开辟一个zone来分配SSTable
      else if Get_num_active_zone() =
           MAX_NUM_ACTIVE_ZONES then
           Close_active_zone();
           zone ← Assign_new_zone();
        	 return zone;
//没有empty zone了 那索性从第一个zone里挑一个 未满能够装下该SSTable的且Zone.lifetime_zone
//	大于该SSTable的zone分配咯
      else if Get_num_empty_zone() = 0 then
          zone ← Get_first_zone();
          while zone do
              if lifetime_SSTable<zone.lifetime_zone 
              and Is_not_full(zone) then
              return zone;
              zone← Get_next_zone();
          end
//上面的情况不满足,索性就找一个能够装下该SSTable的Zone得了
          zone ← Get_first_zone();
          while zone do
          	if Is_not_full(zone) then
                  return zone;
            zone ← Get_next_zone();
            end
  	return NULL;      
```

&emsp;&emsp;上面代码的主要思想是:Level specific zone allocation的思想.<br>
&emsp;&emsp;其思想可以总结为:
The main goal of the algorithm is to find the zone with the same lifetime as the current newly generated SSTable from all active zones. If two or more zones are found, we select the first-found zone as the candidate. If no suitable zone is found from all active zones, a new zone is assigned directly to the SSTable, and **the lifetime of this zone is set to the lifetime value of the SSTable.** If no suitable zone is found, and no empty zone exists in the ZNS SSD, we look for a zone with a lifetime value larger than the SSTable. If no such zone is found, then any zone with enough space can be allocated to current SSTable.

## 实验效果
db_bench 中的 fillseq 和 fillrandom . fillseq包含顺序写 而 fillrandom 包含 random writes .
step: 先顺序写50GB数据 to warm ZNS SSD . 之后分别用 fillrandom 随机写50GB 100GB 200GB 400GB的key-value数据.
![img](https://cdn.nlark.com/yuque/0/2024/png/42361192/1705479922492-ce1d4646-1cc3-453f-8580-1fc6f6d39382.png)
![img](https://cdn.nlark.com/yuque/0/2024/png/42361192/1705479934399-79908acd-258f-4516-ae8f-67d262c36c83.png)
&emsp;&emsp;显然这几个指标都对Zone KV非常有利. 这个StdDev标准差(standard deviation . the smaller this indicator is ,the more stable throughput is ).<br>
&emsp;&emsp;且对于used zones而言,ZenFS使用的zone的数量居然和RocksDB的不相上下,ZenFS使用的zone的数量甚至一度超过RocksDB.而Zone KV占用的zone的数量很少,还有很多empty zone,这要归功于find_suitable_zone算法.当Write的量达到2TB后,RocksDB和ZenFS甚至不能够使用,但ZoneKV还是轻松处理.<br>

**Update performance**
-------------------------------------

使用在db_bench中的 overwrite workload来测试ZoneKV随机更新的性能.
在复写之前,首先随机写50GB数据来warm ZNS SSD.设置复写数据的大小为50GB.
![img](https://cdn.nlark.com/yuque/0/2024/png/42361192/1705481047454-64a7d556-f4d2-4af0-b9f7-2d5a8bd399ba.png)

同样也是ZoneKV最好

**Read performance**
------------

&emsp;&emsp;We first write 50 GB data sequentially with fillseq, then write 50 GB data randomly with fillrandom. Next, we update 50 GB data with overwriting, then read 5 GB data sequentially and read 5 GB data randomly.
![img](https://cdn.nlark.com/yuque/0/2024/png/42361192/1705481708426-32851bf9-1f86-4199-8cfc-0669e54e968b.png)
&emsp;&emsp;Sequential read RocksDB is the best,followed by ZoneKV  ZenFS. Reason:This is because ZenFS has many compaction operations before performing random reads, which will lower the system’s throughput. In contrast, RocksDB’s compaction completes quickly, which will not highly impact random reads. In addition, ZoneKV causes fewer compactions than ZenFS, which helps maintain high random-read performance.

**Mixed-Read/Write Performance**
------
&emsp;&emsp;使用db_bench中的readrandomwriterandom workload in db_bench to test the mixed-read/write performance.<br>
&emsp;&emsp;The readrandomwriterandom workload allows us to **specify the read-write** ratio in the workload.In this experiment, we use** a 1:1 read-write ratio**, i.e., reading 5 GB data while writing 5 GB data simultaneously. To reflect the performance of ZoneKV in terms of other metrics, we report the latency of each system in this experiment.<br>
&emsp;&emsp;**ZoneKV is not only space efficient but also with high performance in read/write-mixed workloads.**
![img](https://cdn.nlark.com/yuque/0/2024/png/42361192/1705482140475-a6c2288c-7915-42fc-ab53-584ef30d02c8.png)


**Multi-Thread Performance**
------
&emsp;&emsp;实验方法:先顺序写50GB的数据,然后用多线程分别写50GB的数据。<br>
&emsp;&emsp;实验结果：各种性质都好，且在单线程中的性能与多线程的性能一致。<br>ZoneKV can scale with the threads. It can always maintain the highest throughput in all settings. Moreover, ZoneKV also consumes the least number of zones and achieves the lowest space amplification. Also, the StdDev of ZoneKV is smaller than that of RocksDB and ZenFS. In general, the performance of ZoneKV in multi-thread environments is consistent with that in single-thread running, which means that ZoneKV can maintain stable and scalable performance in multi-thread settings.
![img](https://cdn.nlark.com/yuque/0/2024/png/42361192/1705558630059-5da02b33-6099-4d48-8df7-42b5a54d1b23.png)

**Conclusion**
------
&emsp;&emsp;In this paper, we proposed a new key-value store for ZNS SSDs, which was called ZoneKV, to mitigate space amplification and maintain high performance. The novelty of ZoneKV lies in the lifetime-base zone storage and levelspecific zone allocation, which can lower the number of used zones and reduce space amplification of the LSM-tree on ZNS SSDs. We experimentally demonstrated that ZoneKV outperformed RocksDB and the state-of-the-art ZenFS under various workloads and settings. Especially, ZoneKV can reduce up to 60% space amplification while keeping higher throughput than its competitors. Also, the experiments under a multi-thread setting showed that ZoneKV could maintain stable and high performance under multi-thread environments.
## 改进思考

------------------------

# 论文 2. Compaction-aware zone allocation for LSM based key-value store on ZNS SSDs
## 背景知识
ZNS:zone namespace ssd。采用zone 接口，借助flash-based SSD实现的。zone由多个闪存(擦除)块构成，大小固定，仅允许顺序写和reset擦除指令。

![img](https://cdn.nlark.com/yuque/0/2024/webp/42361192/1705051884930-81282617-7e36-484d-b647-ec3325145733.webp?x-oss-process=image%2Fformat%2Cpng)
[ZNS:解决传统SSD问题的高性能存储栈设计](https://zhuanlan.zhihu.com/p/425123214)

&emsp;&emsp;**ZNS**的特点：
&emsp;&emsp;由于ZNS SSD引入了zone的概念，其中的zone只能顺序写而不允许重写。（由于接口的设计）因此ZNS SSD的FTL不会进行Garbage collection，进而消除了写放大(copy valid data)的问题。但是如果只能顺序写，数据量会越来越大，必须采取别的措施来释放存放invalid data的空间，该措施是zone cleaning，zone cleaning也会产生写放大问题。zone cleaning以zone为单位。<br>
&emsp;&emsp;一些应用为了缓解zone cleaning造成的写放大问题，会采用将lifetime相同或者相近的data放到同一个zone中(一种数据放置策略)<br>
&emsp;&emsp;ZNS进行了软件层的修改：应用代替FTL接管空闲空间回收（以及zone cleaning）、应用确定数据放置的位置(which zone to place into)。

&emsp;&emsp;**LSM树**(log-structured Merge-Tree):用于优化写性能的数据结构且被用于各种键值存储之中（如 RocksDB ,  LevelDB , MongoDB）。LSM树写吞吐 高（顺序写）。LSM树由MemTable和多个level组成。MemTable临时存储键值对（MemTable位于主存中），之后MemTable会转移到level0中。SSTable会将内部的key排好序，除了Level0的sstable外，其余level的sstable会对关键字range进行分割。每个level大小一定，大小指数型增长。一旦某个level超过了大小限制，便会引发压缩操作：在level i选择一个SSTable(a)->在level i和level i+1选择和(a)有重叠关键字的sstables，对这些tables进行压缩、归并产生新的SSTables(bs)，并将(bs)插入到level i+1。SSTables的改变（重写和删除）只可能发生在压缩过程中。
&emsp;&emsp;**ZenFS**:是应用级的中间件(用于zone 管理，例如：合并数据（或为新创建的SSTables分配zone以及回收zone）等操作。Zone cleaning也会导致写放大。当没有适合的zone时，就要开始zone cleaning。（本文就是想尽可能地减小下面过程造成的写放大的问题）。其主要步骤为：
- 步骤
1. 选择要擦除zone a
2. 将有效数据从该zone a中复制到free zone b中
3. 通过ZNS SSD的 reset指令，擦除zone a



&emsp;&emsp;RocksDB采用ZenFS(a user-level file system for ZNS-enabled RocksDB)来最小化zone cleaning的开销。而ZNS-enabled RocksDB是一个基于ZNS SSD的log-structured merge tree-based key-value store。ZenFS采用的zone分配算法是基于lifetime的zone分配算法(**LIZA**)(该算法采用第二段中的思想)。LIZA通过预测它的lifetime，将其分配到不同的SSTable中。根据SSTable所处的level可以预测它的lifetime/根据新创建的sstable所属的Level预测其的lifetime。[ LSM的特性：很快将被更新或者是删除的SSTables位于更高的level而相对久的数据会被放在更低的level ]。这也会导致 一旦对lifetime预测失败，便会导致sstable放置错误、使得不能最小化WA问题。此外LIZA的另外一个缺陷便是没有考虑SSTables是怎样失效的。因此提出了CAZA（Compaction-Aware Zone Allocation algorithm)。<br>
&emsp;&emsp;当前rocksDB中的Zenfs使用的算法是**LIZA**。该算法的主要思想是：LSM树的level对应一个lifetime hint value，每个SSTable会根据其destination level “获得”一个lifetime hint value（该值反应了SSTable中数据变无效需要的时间的长短，分为(1)short (2)medium (3)long (4)extreme），之后会根据其所处的level 得到其预测lifetime hint value，该值会指导LIZA给SSTable分配zone(会找zone 的Lifetime hint value与该table相等或者比table大的zone)，尽可能地将具有相同lifetime hint value地SSTables放到同一个zone里，从而减少了之后可能出现的在zone cleaning期间有效数据的复制问题。如下是lifetime hint value的分配原则。


| destination level |	lifetime hint	| Restriction |
|-------------------|---------------| ---------- |
|0 or 1|	Medium(2)|	for sstable|
|2	|long(3)|	for sstable|
|3	|extreme(4)	|for sstable|
|only for data such as Write-Ahead Log(WAL) and Manifest(summary information of th LSM-tree)	|short(1)|	for data (WAL and Manifest)|

有个东西没看懂：
![img](https://cdn.nlark.com/yuque/0/2024/png/42361192/1705303755308-d4f806d4-e5d0-4a60-8459-6a78f820383e.png)
![img](https://cdn.nlark.com/yuque/0/2024/png/42361192/1705316420981-fdba7a68-ff6a-4a03-8aa0-d8440e237eab.png)

- LIZA的缺点：
1. 根本性的缺点 LIZA未能减少被选中要reset的zone中有效数据的数量，导致zone cleaning压力大。
2. compaction过程会导致有着不同lifetime hint值会被压缩到一起（跨zone的），进而导致未来可能进行的reset操作仍有写放大存在。/有着重叠key ranges 的SSTables 会在不同的zones，这样会给之后的compaction产生的写放大埋下伏笔。
3. 缺乏对SSTable怎么变得无效这个设计的思考（没有考虑SSTable 为啥无效了 ——是因为compaction）
4. 由于在不同的zones里，因此当这些zone 不位于同一个level，则会导致lifetime hint level不同的zone无效，和2一样，会造成写放大。
因此便提出Compaction-awae zone allocation这一算法

## 方法设计
CAZA算法的提出是基于对LSM-tree原理（select、merge、invalidate）的观察：一组要删除或者变无效了的SSTables是由LSM树的压缩算法确定的。压缩算法会选择处于相邻level且有着重叠关键字的SSTables，将他们合并成1个或者产生多个新的SSTables，并使(invalidate)原SSTables无效；之后将新产生的SSTables放到与它关键字重叠最多的zone里。这样可以降低未来merge的开销。
- policy
       
1. CAZA 1 . Start with L , the target level of the new SSTable 𝑆. CAZA constructs a set of SSTables (𝑆𝑜𝑣𝑒𝑟𝑙𝑎𝑝 ) by searching level 𝐿 + 1 for SSTables that overlap the key range of the SSTable 𝑆.
2. CAZA 2: CAZA builds a set of all zones 𝑍 containing SSTables from 𝑆𝑜𝑣𝑒𝑟𝑙𝑎𝑝 .
3. CAZA 3: Then the set 𝑍 is sorted in a descending order by the number of SSTables belonging to 𝑆𝑜𝑣𝑒𝑟𝑙𝑎𝑝 .
4. CAZA 4: CAZA allocates zones from 𝑍 in order.

- 我的理解是
1. step1:建立Soverlap集合：Soverlap 就包含 下面两个蓝色的 SSTable

2. step 2:建立 Z(zone)集：寻找含有Soverlap的SSTable的zone 

3. step 3:排序：将Z集合根据这些zone中与上面三个SSTable重合的数量按照降序排列。在该例子中**Zone 1**属于Z集合。经过排序后，应为Zone1 。
4. step 4：分配空间
&emsp;&emsp;尝试 **Zone 1**可以分配吗？若不能则该情况无解，只能采用fallback策略。
&emsp;&emsp;但是，**CAZA**如果仅有上面的四步，也会有不能解决的情况。
- 
1. Some SSTables may have entirely new key ranges（也就是说 **Soverlap**为空集，进而 **Z** 集合为空集，上述算法无解）.In this case, CAZA allocates an empty zone so that the zone can contain the new key range.
2. CAZA may not have enough space to store new SSTables in all zones where SSTables with overlapping key ranges reside.（换句话说 Z 集合不为空集，但是 Z 集合中所有的zone 都没有额外的空间来存储该新的SSTables ） In this case, CAZA will allocate a new empty zone rather than zones including disjoint key-ranges.(重新开辟空间而不是随机分配一个有多余空间的 与新SSTables无交集的zone)Otherwise, SSTables with non-overlapping key ranges will be located in the same zone and are less likely to be compacted together.(如果采用后面的这个方法，那么这个空间就难以压缩。)
&emsp;&emsp;因此有两个fallback策略，用于解决上面没有考虑到的情况.<br>&emsp;&emsp;fallback 1(better):CAZA searches for an SSTable in the same level that has the closest key ranges possible, then allocates the zone containing the SSTable.结合上面例子理解：CAZA在level L寻找一个SSTable(该SSTable与new SSTable最接近)，然后将new SSTable分配给该SSTable所在的zone中。因为两个SSTable有交集，方便之后的压缩操作。<br>
&emsp;&emsp;fallback 2: CAZA退化成LIZA.<br>


----
## 实验效果

&emsp;&emsp;通过修改 RocksDB 6.14版本来实现CAZA。CAZA有效缓解了了写放大问题。<br>
&emsp;&emsp;实验环境：100GB ZNS SSD(由100个1GB的zone组成) Intel Xeon E5-2640 linux5.10.13 通过修改RocksDB中ZenFS的代码实现的，采用贪心zone cleaning策略：每次选择含有无效数据最多的zone执行reset命令,为了从100个zone中拨出10个zone作为预留空间，防止zone cleaning 阻塞。<br>
&emsp;&emsp;分析CAZA和LIZA，使用db_bench。MemTable和L0 SSTable的大小均为64MB。L0大小为256MB，L1为2560MB以此类推。首先装载40GB均匀分布的键值对（其中16B的key 128B的 value），装载好之后，通过随机覆写另外40GB的键值对（在load装载的键值对的范围内）。<br>

**写放大问题比较：**
----------

&emsp;&emsp;CAZA shows a lower WA than LIZA for all thresholds, and the gap becomes larger as the threshold increases. The rate in increase of WA of LIZA is larger than that of CAZA.
![img](https://cdn.nlark.com/yuque/0/2024/png/42361192/1705387749292-f5705f4f-0109-4813-8042-02f11c999398.png)
&emsp;&emsp;threshold represents the minimum amount of free space zone cleaning must achieve.
<font size=4 >**lifetime segregation effect**</font>
&emsp;&emsp;CAZA are located higher than the LIZA’s in all time spans. CAZA has 3.32x more zones with an invalid ratio of 1.0 and 4.2x more zones with an invalid ratio of 0.95 than LIZA. This shows that CAZA places data invalidated at similar timing into the same zone so that less valid data are co-located with invalid data.

<font size=4 >**Impact of Compaction-Awareness**</font>

综合图像以及一些描述，可以得出，compaction时，CAZA(与LIZA相比)选择的SSTables 的Average Number of Zones更小，而每个Zone中CAZA的无效数据更多。（这意味着在之后的reset时，WA的情况要好得多）


<font size=4 >**Performance**</font>

throughput and latency情况。
在THS较高的情况下，CAZA（相较LIZA）才能体现出它的优势。
Suspect that this insignificant performance difference is because (i) we use DRAM-emulated ZNS without considering the NAND latency of ZNS, and (ii) the implementation overhead for CAZA greatly affects DRAM-emulated ZNS. However, if an actual ZNS device is used, it is expected that CAZA will outperform LIZA.<br>
## 改进思考

-------------
# 论文 3.Seperating Data via Block Invalidation Time Inference for Write Amplification Reduction in Log-Structured Storage

[论文原文在此](https://www.usenix.org/system/files/fast22-wang.pdf)<br>

&emsp;&emsp;本文设计了Sepbit数据放置策略来最小化WA开销;并且设置了原型实验，以及采取了阿里云和腾讯云的数据进行了分析，并和最优数据放置策略进行对比，得出SepBIT效果卓越的结论；<br>
<br>

## 背景知识
<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="https://cdn.nlark.com/yuque/0/2024/png/42361192/1709797130947-f6a75417-297b-40c3-b540-7db9a69fcb1a.png?x-oss-process=image%2Fresize%2Cw_1500%2Climit_0">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;"></div>
</center>
&emsp;&emsp;数据放置流程图如上所示；写block的操作有两种:

1. user_written blocks
2. Gc-Rewritten blocks
<br>
&emsp;&emsp;GC操作大致可抽象出3步：triggering、selection、rewriting。<br>

&emsp;&emsp;本项目中的triggering由GP阈值触发:一个volume的无效块占比 $`\frac{invalid\space blocks}{invalid\space blocks+valid\space blocks}`$;当无效块占比超过该阈值会触发trigger机制<br>
&emsp;&emsp;Selection:选择一个或多个sealed segments for GC(garage collection)<br>
&emsp;&emsp;ReWriting:discard invalid blocks from sealed segments and writes back the remaining valid blocks into one or multiple open segments.

&emsp;&emsp;对于log-structured storage来说，其组成如下图：<br>
<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="https://cdn.nlark.com/yuque/0/2024/png/42361192/1709799456688-e74acadf-2a00-4135-9652-fce9d4aaac31.png?x-oss-process=image%2Fresize%2Cw_1500%2Climit_0">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">figure 2 log-structured system composition</div>
</center><br>
&emsp;&emsp;每一个block要么是由于客户的new write 请求 要么是一个现存block的更新操作，block会被加入到一个segment(open segment which still accepts blocks );当一个segment满后(reach maximum size)，segment 变成immutable segement(in this paper called sealed segment)<br>
&emsp;&emsp;更新block属于一个out-of-place的过程。直接更新在别的块中而不改变当前块。<br>

&emsp;&emsp;WA(写放大)定义为: $` \frac{user-written\space blocks+GC\space re-rewritten\space blocks}{user-written\space blocks}`$ <br>
&emsp;&emsp;WA=1时为最优，即当 **re-written blocks=0** 时,但实现它是**不切实际**的。原因是：
  1. 不可能预先知道每个block的BIT值
  2. open segment的值k=       $`\lceil\frac{m}{s} \rceil`$ 也不好设置

&emsp;&emsp;在设计的时候会考虑将BIT相近的优先放在同一个块中，之后在GC时的开销便也会相应减少。主要研究Alibaba cloud的traces<br>
&emsp;&emsp;通过分析，由3个观察结果
1. User-written blocks generally have short lifespan while GC-rewritten blocks generally have long lifespans.因此区分这两种写操作是非常有必要的。
2. Frequently updated blocks have highly varying lifespans.
3. Rarely updated blocks dominate and have highly varying lifespans.

&emsp;&emsp;实验结果2、3表明我们现在采用的基于温度的数据放置策略不能够很好地缓解WA的问题（因为rarely updated blocks通常被视为cold blocks被放在一起，而frequently updated blocks 通常被看作hot blocks被存放在一起）<br>


## 方法设计
&emsp;&emsp;SepBIT:依据BIT(block invalidation time)值来进行数据放置，将blocks分成user-written blocks和GC rewritten blocks.

<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="https://cdn.nlark.com/yuque/0/2024/png/42361192/1709870996658-65cb2c06-3824-492c-996e-edea4bf231f5.png?x-oss-process=image%2Fresize%2Cw_1374%2Climit_0">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">figure 3 SepBIT workflow</div>
</center><br>

&emsp;&emsp;当前的SEPBIT有six classes of segments，其中class 1-2对应了user-written blocks的segments，class 3-6对应了GC-written blocks。每一个class都有1个open segment和多个sealed
segments。一旦open segment达到了最大size，it is sealed 并且保存在相同的class当中。<br>
&emsp;&emsp;SepBIT推断blocks的生命期和其各自的BIT，其推断原则是:
- for user-written blocks,SepBIT将生命期短的Blocks存到Class 1，生命期长的存在Class 2.
- for GC-rewritten blocks SepBIT会将由于GC操作要被复写的Class 1的块添加到Class 3中,通过对BITS推断,将具有相似BIT。

&emsp;&emsp;SepBIT的主要思想如下：
- 对于 user-written block:若是来自一个new write,该block的Lifespan 设置为infinite;若是由于旧block的更新，**SepBIT用旧块的生命周期**(上一个用户写直到无效了这段时间之内，整个工作负载用户写的字节数)来**估计其生命周期。**
- 对于GC-rewritten block,SepBIT根据其年龄age(user-written 从**上一个用户写**直到它被GC重写变得无效的过程中、在整个工作负载中的字节数)来推断其剩余生命期(在**GC复写直到它无效/直到traces结束**整个过程中一共复写的字节数)，**SepBIT用其年龄age，将age相似的放在同一个class(segment)当中。**<br>
&emsp;&emsp;因此，GC复写块的生命期是年龄加上其剩余生命周期。

&emsp;&emsp;SepBIT的工作流程如上图 **figure 3**<br>
&emsp;&emsp;SepBIT推断blocks的生命期和其相应blocks的BIT。<br>

&emsp;&emsp;figure4则很能说明我们上面关于lifespan的论述<br>

<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="https://cdn.nlark.com/yuque/0/2024/png/42361192/1709891505596-684e3d7e-c2dd-4076-94de-be42e6a3e547.png?x-oss-process=image%2Fformat%2Cwebp">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">figure 4 infering of blocks</div>
</center><br>
&emsp;&emsp;该论文也对前面我们提到的关于infer BIT of user-write block和GC rewrite block的有关结论进行了数学上的计算和实验验证。<br>

&emsp;&emsp; **变量含义:** 对于**inferring BIT of User-Written Blocks**由于根据前面的设计，lifespan的单位是byte,令n为逻辑块地址的数量（1-n），pi是每一个写请求中(LBAi)被写的概率。一个write-only request sequence of blocks(块的请求序列)，这些块中的每一个块都与一个序列b(b确定块的信息)和逻辑地址Ab(Ab确定块b对应的地址)有关。其中b表示用户新写的块，而b'表示被b取代了的invadiated的块，又自然有 $A_{b}=A_{b'}$。


&emsp;&emsp;因此得到 $estimated\space BIT_b=current\space user\space write\space time+u$。其中u和v都是以块为单位的。条件概率 $P(u\leq u_{0}|v\leq v_{0})=\frac{P(u\leq u_{0}\space and v\leq v_{0})}{P(v\le v_{0})}$，根据Zipf分布来进行计算，该条件概率的含义是当被取代块b'的lifespan小于一个阈值$v_{0}$时，取代块b同样也小于一个阈值的概率。$u_{0}$ 和 $v_{0}$ 分别是u和v的可能的阈值<br>
&emsp;&emsp;
可能是将得到的block按照降序的顺序进行排序1~n，因此有 $p_{i}$ = $\frac{ \sum_{i=1}^{n} (1-(1-p_{i})^{u_0})\cdot (1-(1-p_{i})^{v_{0} } ) p_{i} }{ \sum_{i=1}^{n}(1-(1-p_{i}) ^ v0) \ cdot p_{i} }$ 应该是对所有情况的一个列举。<br>

<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="https://cdn.nlark.com/yuque/0/2024/png/42361192/1709985245397-c588dbff-1acb-4d3a-89f8-89241b491bc4.png">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">figure 5 条件概率推导</div>
</center><br>

&emsp;&emsp;根据此，并进行试验，得出了a user-written block is
highly likely to have a short lifespan if its invalidated block also has a short lifespan.（也就是lifespan小的更有可能取代lifespan小的块），因此用lifespan小的块u'(old block的lifespan)去infer v' 。除此之外还有一个发现
 $v_{0}$ 越小，对越小，对应的条件概率越大（也即被取代的块b'的v越小取代它的b的u也就越小）<br>

&emsp;&emsp;关于Inferring BITs of GC-Rewritten Blocks:同样也是从数学推导和实验数据两个方面来对之前提到的关于residual lifespan结论进行证明;SepBIT通过GC rewritten block的age来估计其residual lifespan，而其 $BIT_{GC\space rewritten\space block}=current\space GC-\space write\space time+estimated\space residual\space lifespan$。<br>
&emsp;&emsp;GC-rewritten块是由user-written块转换而来的，因此用user-written 块对应的number b来描述该GC-rewritten块，u,g,r分别表示其lifespan,age,residual lifespan,因此u=g+r(上面三个变量均以block为单位);通过数学描述和实验证明，得出了当g小的话其r值也会笑。之后便是和上面一样在对其条件概率进行数学分析，已经用实验进行验证。<br>
&emsp;&emsp;对user-written blocks,用lifespan threshold来分割短期blocks和长期blocks，对GC-rewritten blocks，我们需要多个age thresholds来分割。上面的这些lifespan都是通过从segment被创建之后(如从第一个block被加入到该segment中然后到由于GC被收回的这段时间)在工作负载中用户写的字节数来定义的。根据多个固定数量的最近被回收的segment的lifespan值的大小，算出其average segment lifespan $` l `$，对于每一个用户写块 $` userwritten\space block`$,如果它要取代一个lifespan 少于 $` l `$ 的块，我们就将其写入到class 1，否则将其写入到class 2（所对应的segment）。对于GC-rewritten blocks,关于age的threshold我们设置为l的倍数。<br>
&emsp;&emsp;关于算法的细节，算法展示的是SepBIT的伪代码，包括3个函数 :GarbageCollect,UserWrite和GCWrite，如下图所示：
<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="https://cdn.nlark.com/yuque/0/2024/png/42361192/1710040711454-df74b956-9bba-4c2a-94b1-580163a8a211.png?x-oss-process=image%2Fresize%2Cw_1326%2Climit_0">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">figure 6 SepBIT算法</div>
</center><br>

### 算法介绍

&emsp;&emsp;其中t是时间戳，用于确定当前要插入的block，初始的平均lifesp $l$ 设置为正无穷(之后会动态变化)<br>
&emsp;&emsp;GC由于GC操作被激活（基于2.1讲的GC策略，如当GP(garbage proportion)率很高时会触发），它执行GC操作。通过执行例如基于贪心的选择算法，计算出被收集的Class 1 的segement的lifespan的总和 $l_{tot}$ ，并且计算平均值 $l=l_{tot}\div n{c}$，其中 $n_{c}$ 是回收段的数量。<br>

&emsp;&emsp;UserWrite处理每一个user-written block。首先计算无效老块b'的lifespan(也即前面提到的v)，若$` v `$ < $` l `$ (即将b视为一个short-lived block)，UserWrite会将b加入到CLass1的open segment当中。否则将其(视为long-lived block)放入到Class 2中的一个open segment当中。<br>
&emsp;&emsp;GCWrite处理与user-written block b对应的GC-rewritten block。若b存储在Class 1中，GCWrite会将b产生的GC-rewritten block append 到Class 3对应的open segment当中。若b存储在Class 2当中,GCWrite会依据块b的 **age** 将b所产生的GC-rewritten块 append到Class 4、5、6中去，对应的范围分别是[0,4 $l$ ),[4 $l$ ,16 $l$ ),[16 $l$ ,+ $`\infin `$ ) <br>
&emsp;&emsp;至于内存使用情况，SepBIT仅仅存储每一个block上一次用户写的时间（作为元数据）在磁盘的每一个块上。
<br>
&emsp;&emsp;其中Prototype是模拟的，原型当中每个segment是一个一一映射(一个segment对应一个Zonefile:为ZenFS zoned storage backend的基本单元).有关ZenFS的介绍可以查找该论文的相关段落。<br>


## 实验结果
&emsp;&emsp;本论文一共做了9组实验，分别研究了impact of segment selection 、impact of segment sizes、impact of GP thresholds、BIT inference、Breakdown analysis、Results on tencent cloud traces、workload skewness、Memory overhead and Prototype evaluation。<br>
&emsp;&emsp;每个用户写将LBA提升到一个hotter segment，然而每个GC写将LBA降级到一个colder segment。对温度的刻画使用一个基于温度(用write count或者其他的方式来量化，基于策略的不同而相异)的计数器。<br>
&emsp;&emsp;实验考量了SepBIT以及基于温度的算法如 DAC、SFS、ML、ETI、MQ、Seqyetniality、Frequency、Recency、Fading Average Data Classifier and WARCIP。考虑了3大baseline strategies:NoSep(noseperate)、SepGC(seperate and gc)、FK（future knowledge）并且基于不同的算法做了适应性调整。<br>

&emsp;&emsp; **实验结果**如下:<br> 
1. SepBIT achieves the lowest WA among all data placement
schemes (except FK) for different segment selection algorithms (Exp#1), different segment sizes (Exp#2), and different GP thresholds
2. We show that SepBIT provides accurate BIT inference (Exp#4).
3. We provide a breakdown analysis on SepBIT, and show that
it achieves a low WA by separating each set of user-written blocks and GC-rewritten blocks independently (Exp#5).
4. SepBIT achieves the lowestWA in the Tencent Cloud traces
(Exp#6).
5. SepBIT shows high WA reduction for highly skewed work-
loads (Exp#7).
6. We provide a memory overhead analysis and show that
SepBIT achieves low memory overhead for a majority of
the volumes (Exp#8).
7. Our prototype evaluation shows that SepBIT achieves the highest throughput in a majority of the volumes (Exp#9).
   <br>&emsp;&emsp;
   默认的GC策略运用Cost-Benefit策略进行segment selection并且将segment size和GP threshold固定（该策略在前面已经讲过）

## 改进思考


# 论文4.ZNS+: Advanced Zoned Namespace Interface for Supporting In-Storage Zone Compaction

[论文原文在此](https://www.usenix.org/system/files/osdi21-han.pdf)