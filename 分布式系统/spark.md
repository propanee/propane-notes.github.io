# BigData-Spark

> [MIT 6.824：spark - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/516534599) <= 网络博主的笔记

## Spark-简述

> [Hadoop学习（二）——HDFS](https://zhuanlan.zhihu.com/p/659552665)

某种程度上，Spark是Hadoop(open source mapreduce)的继任者（毕竟都是批处理框架），通常用于有大量数据需要大量机器的科学计算。

Spark很适合需要执行多轮mapreduce的场景，因为Spark将中间结果保存在内存中。

- **In-memory computaion**：Spark针对内存进行计算，FaRM是关于内存中的数据库；

这篇论文中定义的RDD(Resilient Distributed Datasets, 弹性分布式数据库)已经过时了，RDD被**数据帧(dataframes)**取代。但是数据帧(dataframes)可认为是显式列的RDD(RDD with explicit columns)，RDD的所有好的设计思想同样适用于数据帧(dataframes)。这节课只讨论RDD，并将其等同于数据帧。

## RDD执行模型(execution model)

> [Spark中的Transformations和Actions介绍_sisi.li8的博客-CSDN博客_spark actions transformations](https://blog.csdn.net/qq_35885488/article/details/102745211) <= 有详细的算子清单描述
>
> 1. 所有的`transformation`都是采用的懒策略，如果只是将`transformation`提交是不会执行计算的，计算只有在`action`被提交的时候才被触发。
> 2. `action`操作：action是得到一个值，或者一个结果（直接将RDD cache到内存中）
>
> [Spark：Stage介绍_简单随风的博客-CSDN博客_spark stage](https://blog.csdn.net/lt326030434/article/details/120073802) <= stage划分讲解，图文并茂
>
> [spark——宽/窄依赖 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/350746998) <= 图文并茂，推荐阅读
>
> spark中， 某些操作会导致RDD中的数据在分区之间进行重新分区。新的分区被创建出来，同时有些分区被分解或合并。所有基于重分区操作而需要移动的数据被成为shuffle。在编写spark作业时，由于此时的计算并不是在同一个executor的内存中完成，而是通过网络在多个executor之间交换数据，因此shuffle操作可能导致严重的性能滞后。shuffle越多，作业执行过程中的stage也越多，这样会影响性能。之后的计划中将会学习spark的调优，所以shuffle过程将会在后面的学习中进行补充。
>
> spark driver会基于两方面的因素来确定stage。这是通过定义RDD的两种依赖类型来完成的，**窄依赖和宽依赖。**
>
> [spark持久化操作 persist(),cache()_donger__chen的博客-CSDN博客_spark persist](https://blog.csdn.net/donger__chen/article/details/86366339)
>
> spark对同一个RDD执行多次算法的默认原理：每次对一个RDD执行一个算子操作时，都会重新从源头处计算一遍。如果某一部分的数据在程序中需要反复使用，这样会增加时间的消耗。
>
> 为了改善这个问题，spark提供了一个数据持久化的操作，我们可通过persist()或cache()将需要反复使用的数据加载在内存或硬盘当中，以备后用。

### Using cases

 了解RDD编程模型的最好方式就是先了解一些简单用例：

```scala
1.lines = sparks.textFile("hdfs://...") 
2.errors = lines.filter(_.startsWith("ERROR"))
3.errors.persist()

// Count errors mentioning MySQL:
4.errors.filter(_.contains("MySQL")).count()

// Return the time fields of errors mentioning
// HDFS as an array (assuming time is field
// number 3 in a tab-separated format):
5.erorrs.filter(_.contains("HDFS")) // HDFS RDD
6.            .map(_.split('\t')(3)) // time RDD
7.            .collect()
```

1. 创建RDD对象`lines`；

   - read-only object，不能修改，只能从现有的生成新的；

   - 该RDD存储在HDFS中，HDFS可能有对这个文件有很多分区，例如第100万条记录在HDFS分区1上，第200万条数据在分区2。这个RDD `lines`代表了这组分区；

   - 运行这行时，什么都不会发生，即论文所说的**惰性计算(lazy computations)**；

2. 通过在`lines`上运行的filter创建新RDD `errors`，过滤出所有以`"ERROR"`开头的数据行。

3. 在内存中保存`errors`的copy，以便后续计算仍可以reuse，而不需要从HDFS中重建；

   - 可以看出来这里和Mapreduce有很大不同。在mapreduce中，计算结束后，如果想重新操作数据，则必须从文件系统中重新读取数据；而这里Spark使用persist方法避免从磁盘重新读取数据，节省了很多时间。
   - 到目前为止没有进行计算，只是预先创一个**数据流(dataflow)**，或者论文中所说的**计算迭代图(iterative graph of the computation)**。在后续运行action操作时，这些操作**流水线式**执行，即先创建lines，后过滤lines的数据获取errors，再进行后续的其他RDD操作。

4. 过滤出`errors`中带有"MySQL"字眼的记录，使用`count()`这个**action操作**统计数量，实际导致计算；

5. 过滤出`errors`中带有"HDFS"字眼的记录；

6. 用`\t`切割每一行的字符串后取出第3个部分(即time)；

7. 最后`collect()`，该action操作汇总所有的time结果（实际导致计算）

在4 `count()`和7 `collect()`的Action操作执行的时候，用户交互的命令行界面才会实际触发Spark进行计算，Spark会collect许多worker，向它们发送jobs，或者通知调度器(scheduler)需要执行job。各个worker在HDFS的不同分区上读取数据，执行这个**谱系图(lineage graph)**中的各stage：**lines(filter) -> errors(filter) -> HDFS(map) -> time(collect**)。前面的一直到获得time RDD都类似于map，然后shuffle，到最后collect()类似于reduce phase。

> 当ERROR文件从P1(HDFS分区1)和P2中提取出来时是并行发生的吗？是的，就像mapreduce中有很多worker，worker在每个分区上工作，调度者scheduler会发送job给每个worker，而job是和数据分区关联的task。worker获取一个task后运行。所以**可以在分区之间获得并行性。也可以获得流水线中各个阶段之间的并行性。**

- **宽依赖**：如对time RDD做collect()，操作依赖于多个父分区；

- **窄依赖**：如过滤得到HDFS RDD仅依赖于一个父分区，只需一个父分区就能计算；一般来说更希望计算窄依赖，因为不需要进行通信，在本地就可以进行；

- 这里说的依赖是针对partition的，而不是RDD；而谱系图(lineage graph)中的stage之间的箭头是对于RDD的；


> We found it both sufficient and useful to classify dependencies into two types: *narrow* dependencies, where each partition of the parent RDD is used by at most one partition of the child RDD, wide dependencies, where multiple child partitions may depend on it. For example, map leads to a narrow dependency, while join leads to to wide dependencies (unless the parents are hash-partitioned).  

![image-20231220115952683](C:\Users\cbw\AppData\Roaming\Typora\typora-user-images\image-20231220115952683.png)

### RDD APIs

RDD的API支持两类方法：**Transformations和Actions**。

> We give the signature of each operation, showing type parameters in square brackets. Recall that *transformations* are lazy operations that define a new RDD, while *actions* launch a computation to returna value to the program or write data to external storage.

- Transformations：负责将RDD转换成新的的RDD，**每个RDD都是只读(read-only)或者不可变的(immutable)**。所以不能修改RDD，只能从现有的RDD生成新的RDD。
- Actions：实际执行计算。只有执行了Actions后，前面的Transformations涉及的文件操作等逻辑才会实际被执行。**Actions才是实际导致计算发生的操作，所有构建的惰性计算都发生在运行Action的节点**。

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/1219%20sparker%20RDD%20ops.png" alt="image-20231219203959839" style="zoom:50%;" />

------

> **问题：当错误文件从P1（HDFS分区1）中提取出来时，然后另一个错误文件从P2中提取出来，我理解上这是并行发生的？**
>
> **回答：是的，就像mapreduce中有很多worker，worker在每个分区上工作，调度者scheduler会发送job给每个worker，而job是和数据分区关联的task。worker获取一个task后运行。所以你可以在分区之间获得并行性。你也会得到流水线中各个阶段之间的并行性。**
>
> 问题：lineage和事务日志(the log of transactions)有什么不同，是否只是操作粒度不同(the granularity of the operations)？
>
> 回答：目前我们看到的日志是严格线性的(strictly linear)，lineage也是线性的，但稍后会看到的使用fork的例子，其中一个阶段依赖于多个不同的RDD，这在日志中不具有代表性。它们有一些相似之处，比如你从开始状态开始，所有的操作都是具有确定性的，最后你会得到某种确定性的结束状态，就这点上它们是相似的。
>
> 问题：第三行persist是不是我们开始进行计算的时候？
>
> 回答：不，这里还没有任何东西会被计算出来，前三行仍然只是在描述Spark程序需要执行的操作。
>
> **问题：前面的代码中，如果不调用`errors.persist`，会发生什么？**
>
> **回答：如果不调用`errors.persist`，那么后续发生的第二次计算，即（序号7. action，collect），Spark需要先重新计算errors。**
>
> 问题：对于不调用persist的分区，在mapreduce的情况中，我们把它们存储在中间文件中，但我们仍然将它们存储在本地文件系统中。Spark这里处理数据时，是会产生中间文件存储在磁盘中吗，还是将整个数据流处理保存在内存中？
>
> 回答：**默认情况下，整个流都在内存中**，但有一个例外，稍后会详细讨论，即在`persist`使用reliable flag。然而RELIABLE flag将存储在HDFS，这个称为checkpoint。
>
> 问题：对于分区，这里代码序号2.对应的HDFS，说有多个分区，这个是Spark临时为每个worker进行的分区划分HDFS数据？
>
> 回答：代码序号1.对应的lines这个RDD产生于HDFS。这里说的分区，是HDFS文件直接直接定义的（也就是说原始的HDFS中的文件本来就分区好了，不是后面SPark临时分区的）。但是你也可以执行repetition重新分区的transformation，一般是stage这样做，后续会提到。比如使用hash partition或定义自己的分区程序，提供一个分区程序对象或抽象。
>
> 追问：所以这里代码里HDFS文件分区，之前就已经由HDFS处理了，但如果你后续想Spark再自定义patition流程，也可以再来一次。
>
> 回答：是的。
>

## 窄依赖容错

和mapreduce中处理容错基本一致，Spark中的某个worker崩溃时，需要重新执行stage。

Spark中worker崩溃(crash of worker)

​	=>意味着lose memory(丢失内存中的数据)；

​	=>意味着lose RDD partition(丢失RDD的partition数据)；

而后续的RDD计算部分可能依赖于丢失的partition。所以，这里需要重新读取(reread)或重新计算(re-compute)这个partition分区。

Solution：**scheduler re-runs the stage for that partition=>产生same partition**，调度器某时刻发现没有得到worker的应答(answer)，就会重新运行对应partition的stage。

和mapreduce函数式编程一样，视频中列举的transformations API，也都是**函数式的**。它们将RDD作为输入，产生另一个RDD作为输出，是完全确定性的(completely deterministic)。即如果重启一个stage，对相同input的一系列transformation操作将产生相同的output。所以如果发现一个worker崩溃了，对于窄依赖，scheduler只需要重新安排某个worker生成同样的partition即可。

------

> 问题：所以就是为什么RDD都是不可变的(immutable)吗？
>
> 回答：是的。
>

## 宽依赖容错

Spark的窄依赖容错，和mapreduce的容错处理基本一致。更为棘手的是宽依赖的容错处理。

<img src="C:\Users\cbw\AppData\Roaming\Typora\typora-user-images\image-20231219220534044.png" alt="image-20231219220534044" style="zoom:50%;" />

假设有个Spark在指令流中，某RDD的数据来源自多个parent-RDD，如join等操作，而该RDD有往下执行产生更多的child-RDD，如果此时worker崩溃，为了重建这个RDD，就需要往源头追溯，重新计算，最后会追溯到需要对多个其他worker上的进行重新计算parent-RDD(可以并行的计算)，然后再次产生最终的RDD。显然，如果还是按照窄依赖的方式进行容错恢复，一个worker fail可能导致许多partition的重新计算，性能损耗严重。

Solution：**对于宽依赖容错恢复，解决方案是设置检查点或持久化RDD**。

比如在parent-RDD取checkpoint检查点存储的partition快照RDD，当需要重新计算子RDD时，只需要从checkpoint读取父分区的结果就好，而不是重新进行并发的计算。

------

> **问题：如果使用了一般的persist函数，但是论文也提到了一个RELIABLE flag。想知道，使用persist函数和使用RELIABLE flag有什么区别？**
>
> **回答：persist函数，意味着你将RDD保存到内存，而不会把它丢掉，后续计算可以从内存中重复使用它（默认整个流都在内存中，后续需要使用到同样的RDD时，需要重新计算一遍）；checkpoint或者RELIABLE flag意味着，你将整个RDD的副本写入HDFS，而HDFS是持久或稳定的文件存储系统。**
>
> 问题：有没有一种方法可以告诉Spark不再持久化某些东西，因为如果你持久化了RDD，而且你做了大量的计算，但是后面的计算不在使用那个RDD，你可能会把它永远留在内存中。
>
> 回答：Spark使用了一个通用的策略，如果真的没有空间了，它们可能会将一些RDD放到HDFS或者删除它们，论文对具体的计划有些含糊。当然，当计算结束，用户退出或停止driver，那我想那些RDD肯定从内存中消失了。
>

## 迭代计算PageRank

> [google pagerank_百度百科 (baidu.com)](https://baike.baidu.com/item/google pagerank/2465380?fromtitle=pagerank&fromid=111004&fr=aladdin)
>
> PageRank，网页排名，又称网页级别、Google左侧排名或佩奇排名，是一种由根据[网页](https://baike.baidu.com/item/网页?fromModule=lemma_inlink)之间相互的[超链接](https://baike.baidu.com/item/超链接?fromModule=lemma_inlink)计算的技术，而作为网页排名的要素之一，以[Google](https://baike.baidu.com/item/Google?fromModule=lemma_inlink)公司创办人[拉里·佩奇](https://baike.baidu.com/item/拉里·佩奇?fromModule=lemma_inlink)（Larry Page）之姓来命名。
>
> PageRank通过网络浩瀚的超链接关系来确定一个页面的等级。Google把从A页面到B页面的链接解释为A页面给B页面投票，Google根据投票来源（甚至来源的来源，即链接到A页面的页面）和投票目标的等级来决定新的等级。简单的说，一个高等级的页面可以使其他低等级页面的等级提升。
>
> [PageRank算法详解 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/137561088) <= 推荐阅读
>
> 在实际应用中许多数据都以图（graph）的形式存在，比如，互联网、社交网络都可以看作是一个图。图数据上的机器学习具有理论与应用上的重要意义。 PageRank 算法是图的链接分析（link analysis）的代表性算法，属于图数据上的无监督学习方法。
>
> PageRank算法最初作为互联网网页重要度的计算方法，1996 年由Page和Brin提出，并用于谷歌搜索引擎的网页排序。事实上，PageRank 可以定义在任意有向图上，后来被应用到社会影响力分析、文本摘要等多个问题。

PageRank是一个对网页赋予权重或重要性的算法，（权重）取决于指向网页的超链接的数量。这里代码用例通过Spark实现PageRank算法，如果最后补充`ranks.collect()`，那么计算就会实际在机器集群上进行：

```scala
val links = spark.textFile(...).map(...).persist()
var ranks = // RDD of (URL, rank) pairs
for (i <- 1 to ITERATIONS) {
    // Build an RDD of (targetURL, float) pairs
    // with the contributions sent by each page
    val contribs = links.join(ranks).flatMap {
        (url, (links, rank)) =>
        links.map(dest => (dest, rank/links.size))
    }
    // Sum contributions by URL and get new ranks
    ranks = contribs.reduceByKey((x,y) => x+y)
    .mapValues(sum => a/N + (1-a)*sum)
}
ranks.collect() // 补充ranks.collect(),计算会在机器集群上进行
```

这里有2个RDD，其中

- `links`表示图的连接/边(the connection of the graphs)，即URL之间的指向关系，如[(U1, [U1,U3])(*表示U1->U1, U1->U3*), (U2, [U2,U3]), (U3, [U1])]；

- 而`ranks`表示每个url的当前排名，如初始化[(U1, 1.0), (U2, 1.0), (U3, 1.0)]；

- 这里`links`通过`persist()`常驻内存，所以下面`ranks`在for循环迭代中重复使用`links`，不需要重新计算`links`，直接从内存中读同一份RDD对象`links`即可；

之后进行迭代，在迭代中

- `links.join(ranks)`生成新的RDD `contribs`如[(U1, [U1,U3], Rank1),..., (U3, [U1], Rank3)]；

- 之后通过`flatMap`映射到一个`map`映射，这个`map`将Rank值划分给outgoing URL，即U1指向的链接U1和U3，如(U1, [U1,U3], Rank1) 最终映射到 [(U1, R1/2), (U3, R1/2)]；

> for example, *map* is a one-to-one mapping, while *flatMap* maps each input value to one or more outputs **(similar to the map in MapReduce)**

- 之后`reduceByKey` 将所有的U1收到的权重fractional weights放在一起并`mapValue`将它们相加，结果是 R1/2+R3/1，将这个总和列表相加，计算得到最终的Rank值。最终`ranks`形如之前的`ranks`[(U1, 1.0), (U2, 1.0), (U3, 1.0)]；

![image-20231220111137287](https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/image-20231220111137287.png)

循环的每次迭代都是一个stage，当调度器计算新的stage时会将transformation的一部分附加到lineage graph中。代码中，links被保存到内存，下面ranks的每次迭代都可以重复使用links，而mapreduce无法重复利用这个巨大的文件。

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/image-20231220114711668.png" alt="image-20231220114711668" style="zoom: 50%;" />

links和ranks在每次循环中做join，join could be a wide dependency，因为生成contrib中间结果需要links和ranks的partition。这里涉及网络分区通信，论文中提到一些优化手段，可以指定分区RDD使用hash分区，意味着links和ranks这两个RDD将以相同的方式分区，它们是按key或hash key进行partition的。**论文中提到这里可以通过hash key分区，使得具有相同hash key的数据partition落到同一个机器上，这样即使join直观上是宽依赖，但是可以像窄依赖一样执行**。比如对于U1，对应分区P1，你只需要查看links的P1和ranks的P1，而hash key以相同的方式散列到同一台机器上。scheduler或程序员可以指定这些散列分区，调度器看到join使用相同的散列分区，因此就不需要做宽依赖，不需要像mapreduce那样做一个完整的barrier，而是优化成窄依赖执行。这样最终的collect是唯一的宽依赖(???)。

这里如果一个worker崩溃，你可能需要重新执行许多循环或一次迭代，所以这里可以设置比如每10次迭代生成一个checkpoint，避免重新整个完整的计算。

------

> 问题：所以我们不是每次都需要重新计算links或其他东西，比如我们不持久化的话？
>
> 回答：这里唯一持久化的只有links，这里常驻内存，而ranks1、ranks2等都是新的RDD，但你偶尔可能像持久化它们，保存到HDFS中。这样如果你遇到故障，就不需要循环迭代从0开始计算所有东西。
>
> 问题：不同的contribs，可以并行计算吗？
>
> 回答：在不同的分区上，所以是可以并行计算的。
>

## 小结

- RDD by functional transformation：RDD是函数式转换创建的
- group togethr in sort of lineage graph：他们以谱系图的形式聚集在一起
  - reuse：允许重复使用
  - clever optimization：允许调度器进行一些优化组织
- more expressive than MR：表现力比mapreduce更强
- in memory：大多数数据在内存中，读写性能好

------

> 问题：我想论文中提到了自动检查点(automatic checkpoints)，使用关于每次计算所需时间的数据，我不太理解是什么意思。但是，他们将针对什么进行优化？
>
> 回答：可以创建一个完整的检查点，这是一种优化。创建检查点是昂贵的，需要时间。但在重新执行时，如果机器出现故障，也要花很多时间。比如，如果你从来不做检查点，那么你要从头开始计算。但如果你定期创建检查点，你不需要重新计算。但如果你频繁创建检查点，你可能把所有时间都花费在了创建检查点上。所以这里有一个优化问题。
>
> 问题：所以可能计算检查点，需要非常大的计算？
>
> 回答：是的，或者在PageRank的情况中，也许每10次迭代做一次checkpoint。当然如果检查点很小，你可以创建checkpoint稍微频繁点。但在PageRank中checkpoint很大， 每个网页都有一行或一条记录。
>
> 问题：关于driver的问题，driver是不是在客户端？
>
> 回答：是的。
>
> 问题：如果driver崩溃，我们会丢弃整个图，因为这是个应用程序？
>
> 回答：是的，但我也不知道具体会发生什么，因为调度器有容错。也许你可以重新连接，但我不太清楚。
>
> 问题：关于为什么dependency optimization的问题。你提到他们会进行hash patitioning，这是怎么回事？
>
> 回答：这个和Spark无关，是数据库数据常见的分区概念。即假设对dataset1和dataset2使用hash partion，之后相同hash key的links和ranks数据会分布到同一个机器上，而此时对links和ranks进行join，就不需要机器进行网络通信，直接在机器本地进行处理即可。即多个机器处理多个分区，但是每个分区内links和ranks的join只需要本地进行，不需要和其他机器网络通信。
>
> **问题：worker和stage？**
>
> **回答：每个worker在partition上运行一个stage。所有stage在不同的worker上并行运行。pipeline中每个stage是批处理(batch)。**