# MapReduce

解决问题：TB级数据上的大量计算

特点：自动的并行计算，隐藏分布式细节，实现容错和负载均衡。Automatically parallelized and executed on a large cluster of commodity machines: hides the messy details of parallelization, fault-tolerance, data distribution and load balancing in a library 

具体做法：
The computation takes a set of input key/value pairs, and produces a set of output key/value pairs .

## Map

将input转化为中间级k-v对。并收集相同的中间级key，将它们传给reduce。

Map, written by the user, takes an input pair and produces a set of intermediate(中间的) key/value pairs. The MapReduce library groups together all intermediate values associated with the same intermediate key I and passes them to the Reduce function.（将input k/v pair（可能是files）转化为一个中间的k/v pair list）

## Reduce

收集中间级key和对应的一组value，将它们的值合并形成一个更小的value集合。

The Reduce function, also written by the user, accepts an intermediate key I and a set of values for that key. It merges together these values to form a possibly smaller set of values. Typically just zero or one output value is produced per Reduce invocation. The intermediate values are supplied to the user’s reduce function via an iterator. This allows us to handle lists of values that are too large to fit in memory.（收集所有的中间k/v pair，merge相同k）

## Example

统计文件中单词出现的次数：

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/map%20reduce%20example0.png" alt="image.png" style="zoom: 50%;" />

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/map%20reduce%20example%20.png" alt="image.png" style="zoom:67%;" />

```c++
map(String key, String value):
	// key: document name
	// value: document contents
	for each word w in value:
		EmitIntermediate(w, "1");
reduce(String key, Iterator values):
	// key: a word
	// values: a list of counts
	int result = 0;
	for each v in values:
		result += ParseInt(v);
	Emit(AsString(result));
```



## 系统架构

**master**: 

- map和reduce tasks的state (idle, in-progress, or completed)  和worker的identity；
- map产生的R个中间级文件的位置和大小。担任通道将这些信息从map接收并传给in-progress的reduce worker。

**map**: processes a key/value pair to generate a set of intermediate key/value pairs.

**reduce**: merges all intermediate values associated with the same intermediate key.

**shortcoming** : **map** must be completely independent and functional pure;

瓶颈：网络吞吐量（为了避免使用网络，在同一组机器上运行GFS servers 和 map workers，master调度时让本地的input data作为同一台机器map的输入文件，map的输出也在本地磁盘上，如果失败了，就近取replica（如同一网络中）。

![image.png](https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/map%20reduce%20arch.png)

1. 用户程序中MapReduce库将input files分成M个16-64MB的切片，并在集群中启动程序的多个副本；
2. master+workers，master向workers分发M个Map和R个Reduce任务;
3. map worker通过map函数将输入数据转化成中间级k-v对，并缓存在内存里；
4. 中间级k-v对由内存写入磁盘，并通过partitioning function (e.g., hash(key) mod R)  划分为R个区域；将其在磁盘中的位置发给master，master发给reduce;
5. reduce通过RPC读取map worker的中间级数据，排序使相同的key组合到一起；
6. 将key和中间级value传入reduce函数，并将输出append到最终的output file；
7. 所有map和reduce结束后，master唤醒user，并将结果返回。

其他重要的Refinements  ：

- combiner: map worker在通过网络将中间级结果传输给reduce时承担部分的merge工作（Combiner function that does partial merging of this data before it is sent over the network.）  基本和reduce一样，不同的是reduce输出到final output file，combiner输出到要传给reduce的中间级文件。

## Fault Tolerance

### worker failure

The master pings every worker periodically. (Heartbeat)

检测到worker failed：将原map或reduce任务置为idle并发给其他worker。

- 已完成的map遇到failure也要被重新执行，因为此时它的本地磁盘不可访问了；
- 已完成的reduce不需要重新执行，因为结果已经被存储到了global file system。

- A的map被B重新执行，告知所有reduce worker。没有从A读到数据的reduce将从B读取。

能够抵御大规模worker failure，因为只要将unreachable  worker的任务重新安排就好了。

### master failure

the master write periodic **checkpoints** of the master data structures。当master出现错误时，通过最近的checkpoint启动新副本。（不过文中说概率很小，直接丢掉所有计算重启）

### backup task

straggler: 在最后仅剩的map或reduce花费了太长的时间。（a machine that takes an unusually long time to complete one of the last few map or reduce tasks in the computation  ）

在mapReduce将要完成时，master会安排backup执行剩下的进行中任务，只要backup或primary完成这个task都会被标记为完成。（When a MapReduce operation is close to completion, the master schedules backup executions of the remaining in-progress tasks. The task is marked as completed whenever either the primary or the backup execution completes.   ）





