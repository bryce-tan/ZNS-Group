# ZNS SSD Group

## 1 平台的使用方法

IDSM ZNS小组使用本平台作为知识库的搭建环境，小组的**任何成员**可以在此**访问、编辑**相关的知识文件。
本仓库包含以下主要DOC：

1. **任务规划**；
2. 每个成员的**文献阅读记录**；
3. **知识汇总**；
4. **工作日志**；

接下来将分别介绍以上文档的阅读与使用方法。

---

## 2 任务规划-一个清晰的研究进展脉络

任务规划DOC指定了每个周的具体任务情况，小组成员需及时完成每周的任务list，从任务规划可以清晰地窥见ZNS小组的研究进展，该文档也可作为日后查阅往日工作的一个主线目录。
以下表格是该文档中每周任务的一个示例：

| 姓名   | 文献任务 |
| ------ | -------- |
| Bryce |**Pathways: Asynchronous Distributed Dataflow for ML**<br />**Achieving us-scale Preemption for Concurrent GPU-accelerated DNN Inferences**<br />**TVM: An Automated End-to-End Optimizing Compiler for Deep Learning**<br />**BaM: A Case for Enabling Fine-grain High Throughput GPU-Orchestrated Access to Storage**          |


## 3 阅读记录-一个可供自己/他人快速查阅or复习的信息提炼

一篇paper除去核心的知识之外，往往还有一些知识外的元素，我们称其为**包装**。为了让论文具备高可读性，又或是为了凸显论文工作的重要性，更或是为了夸大自己工作的实际作用，**包装已经成为现代学术论文越来越必不可少的一部分**。

作为一个专业的领域内工作者，我们应学会**剥掉论文包装的外壳**，**总结出论文有价值的知识部分**。将论文进行总结，既方便日后的查阅，也方便其他成员对改论文进行快速的阅览。

论文有价值的**知识部分**一般包含以下：

1. **Domain Knowledge**。这是一些背景知识，往往是产业/学术界的一些基础共识，例如：ZNS的特点、ZNS的设计带来的SSD优化等等。这一部分往往存在在introduction、background、motivation等章节中。
2. **Proposed Method**。论文提出的方法当然是论文的知识核心。
3. **Evaluation**。论文提出的方法的实验结果是怎样的？了解实验结论有助于让我们加深对论文方法的理解，论文的方法优化/恶化了哪些指标？为什么会造成这样的恶化/优化？对实验结果的洞察往往能启发我们设计更好的方法。

因此，我们也建议每位成员按以上三个部分来整理、总结一篇论文中的核心部分，它包括**背景**（特别包括一些我们没有接触过的背景知识）、**方法设计**、**效果**三个部分。如果可以的话，还可以动用你智慧的头脑整理一下可以想到的**对该论文的改进方向**；

每个人拥有一个文献的整理记录DOC，以“XXX_文献总结”命名。以下是一些撰写的注意事项：

1. ⚠️ 对于domain knowledge中你无法理解的部分，**应做详尽的调查，再总结到背景当中**。例如，论文写到：“SSD的FTL层负责了垃圾回收”，如果你不明白“FTL层”究竟是什么，它是怎么对垃圾回收负责的，请不要简单地将这句话总结到背景知识当中，请详尽地调查何为FTL层，为何其能对垃圾回收负责，以及是怎么负责的，再将调查到的知识整理总结下来；
2. ⚠️ 不要在Evaluation部分简单地陈列所有的实验图表与实验结果，请挑选**重要的优化/恶化指标记录下来**，当然了，还有那些**能加深我们对实验方法理解的指标**。
3. ⚠️ 对该论文可改进方向的思考并非必要，如果你读完这篇paper（或者甚至是你在读这篇paper的时候）脑子里明显的都能想到更好的方法，或者进一步的优化方向，请把这样的好idea记录下来，灵光一现并不容易，请记录下你的每一个灵光一现。
4. ⚠️ 注意提高可读性。注意图文结合，并适当列举公式，引用文献等等。

可以参考以下文档的示例，该示例展示了背景知识（knowledge）部分的整理文档：
[文献总结示例](https://www.yuque.com/brycetan/sgt67r/zn5wnuzen6h04axy?view=doc_embed)

## 4 知识汇总-一个以知识点为索引的知识库

“3 阅读记录”中关于论文的记录与总结可以理解成一个**以论文标题为索引的知识库**。其实为了方便论文/总结/综述/报告的撰写，或者知识点的查阅与回顾，知识应该整理成**以知识点为索引**比较合理。

每篇paper的背景、技术中可能会包含各种各样通用的工业/理论知识，在各位成员阅读完论文，完成文献阅读记录的总结之后，应定期把其中的知识部分都统一copy到一个文档里面，方便查阅，也方便在之后的研究中以此形成论文/报告。

此外，我们把知识点分为两类，一种是**基础知识**，一种是**方法论**。举例而言：

1. “传统SSD的垃圾回收方式”，这是一个基础知识；
2. “ZNS的内部结构”，这是一个基础知识；
3. “SSD设备驱动的种类”，这是一个基础知识；
4. “多段线性回归拟合曲线”，这是一个方法论；
5. “基于统计的生命周期预测”，这是一个方法论。

请将知识按两种种类总结到两个不同的DOC中，它们分别是“**基础知识汇总**”和“**方法论汇总**”。在“基础知识汇总”文档中已经包含了示例。

### Tips：
一般来讲，**每个人的文献总结文档中的背景知识部分**都应整理到“**基础知识汇总**”中（部分可能是方法论的背景知识，应整理到“**方法论汇总**”中）；

文献总结中的方法设计部分中通用的方法应该总结到“**方法论汇总**”中。

**每位成员应该在每双周的组会前定期把论文中的知识点合并整理到知识汇总的文档中。**

## 5 工作日志-一个跟组员同步工作进展的日志

鼓励大家在日志中按日期记录自己的contribution。
以下是一个示例：

### 2024.1.9

谭頔凡 更新了README.md；

谭頔凡 对ZenFS的代码进行了阅读，修改了XXXX；

李晓晓 完成了实验图的绘制；

李佳炜 完成了SCDA的复现，发现存在实验结果XXX的问题；

覃梓晋 完成了修改仓库大小的后的实验；

## 6 IF YOU ARE NEW TO THIS

**如果你是刚加入team的rookie**，不知道该从何开始入手科研，请移步文档“IF YOU ARE NEW TO THIS”，这个文档将指引你简单上手并入门如何做research！
[IF YOU ARE NEW TO THIS](https://www.yuque.com/brycetan/sgt67r/yxkk0cy00km48g1z?view=doc_embed)

## 7 关于自由贡献者

如果**你对ZNS抱有浓厚兴趣，但却不是IDSMlab的一员**，我们也**同样欢迎你对此知识库/我们的研究做contributions**，让我们一起**do something different**！

自由贡献者请通过以下邮箱联系我获取编辑权限:

difan_tan@hust.edu.cn

获得编辑权限后，请在自由贡献者目录下新建自己的文献总结文档，如“自由贡献者x_文献总结”文档所示，并按上述格式/标准进行自由贡献。

同样欢迎你将阅读到的知识整合到知识汇总中！

_——README.md written by Difan_

