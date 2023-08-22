# Storage systems
- 构建(fault tolerance)可容错的系统，需要可靠的storage存储实现。
- 通常应用开发保持无状态stateless，而底层存储负责永久保存数据。（应用只有软状态没有硬状态）

# Why Hard
- 高性能(high perference)
   - shard：需要分片（并发读取）
   - data across server：跨多个服务机器存储数据（单机的网卡、CPU等硬件吞吐量限制）
- 多实例/多机器(many servers)
   - constant faults：单机故障率低，但是足够多的机器则出现部分故障的概率高。
   - 需要容错设计Fault tolerance-->replication-->potential inconsistence
   - strong consistency（强一致性）--> lower performance

# 一致性
- Ideal consistency -->Behave as if single system
   - **并发性concurrency**:
   - >C1写入x1，C2写入x2，C3读取x，C4读取x。不关心谁先谁后时，希望C3和C4读取到的值是一致的。服务器在并发存在的情况下可以使用锁来强制一致性（分布式锁等）
   - **故障Failure**
   - >对于故障一般使用复制解决。而不成熟的复制操作，会导致读者在不做修改的情况下读取到两次不同的数据。Bad replication plan: 当C1、C2并发向两个备份S1和S2分别写入数据1和2，在写操作都完成后，C3、C4从S1或S2读数的结果可能不同
   - >（因为没有明确的协议指出这里W1和W2的数据在S1、S2上以什么方式存储，可能1被2覆盖，反之亦然）

# 假设：
- 系统建立在便宜且经常出错的组件上；
- 系统需要存储数量不多的大型文件；
- 主要分为两种读操作：大型流式读取（streaming read）（连续的）和小型随机读取；
- 写操作：许多大的、顺序的写操作吧数据加到文件末尾；
- 系统必须实现定义良好的语义，让多方用户并发地添加数据到同一个文件；
- 高稳定的带宽比低时延更重要。
# GFS
GFS的几个主要特征：

- Big：large data set，巨大的数据集
- Fast：automatic sharding，自动分片到多个磁盘
- Gloal：all apps see same files，所有应用程序从GFS读取数据时看到相同的文件（一致性）
- Fault tolerance：automic，尽可能自动地采取一些容错恢复操作

# interface接口：
 - 文件按层次目录组织，用路径名标识；
 - create, delete, open, close, read, write... 与通常的文件系统接口类似；
 - **snapshot** creates a copy of a file or a directory tree at low cost. 
 - **record append** allows multiple clients to append data to the same file concurrently  - while guaranteeing the atomicity of each individual client’s append.(原子的，不需要额外的锁)

# 架构：
- **Non-standard** 
   - 只有一个master而不是复制的，存在单点故障的问题
   - 有不一致性，in-consistencies

single master + multiple chunkservers + multiple clients（可与multiple chunkservers在同一机器）

## master
- **metadata:** 
   - file and chunk namespaces (persistent)
   - mapping from files to chunks(array of chunk handles) (persistent)
   - locations of each chunk's replicas (not persistent）
      - 因为chunkserver可能出问题。keep it up-to-date，控制chunk placement和检测server状态
   - access control information
- **system-wide activities **:chunk lease management/ garbage collection of orphaned chunks/ chunk migration between chunkservers. 
   - 告诉client它应该与哪些chunkservers通信。
- **Heartbeat** message with chunkservers
- **operation log**
   - contains a historical record of critical metadata changes
   - 当log超过一定大小就记录其checkpoint，以便从最新的checkpoint **recover**（compact B-tree)

## client
- 实现文件系统API
- 与chunkservers和master通信，来为应用读写数据

**client和chunkserver都不缓存文件数据**
- client因为需要巨大的流式处理。但是缓存一段时间matedata。
- chunkservers因为chunks存储在本地文件（？）。

## 流程（read）：
- client将应用指定的file name和byte offset转化为文件中的chunk index；
- 向master发送包含file name 和chunk index的请求；（可能请求多个chunk）
- master返回对应的chunk handle和locations of the replicas；
- client将file name和chunk index作为key缓存这个信息；
- client向其中一个replica（closest one）发送指定chunk handle和byte range的请求
- 直到缓存信息expires或file is reopened都不需要再和master交互了。

![GFS_Architecture](https://raw.githubusercontent.com/propanec/propane-img/main/PicGo-img/GFS_Architecture.png)

## chunk size：
-  files  fixed-size chunks 
   - 全局且不可变的64bit id (chunk handle)标识，由master在chunk创建时分配 
   - For reliability, each chunk is replicated on multiple chunkservers
- 64MB, much larger than typical file system block sizes
- large chunk size advantages:
   - 减少客户端和master的交互（缓存位置信息）
   - 减少网络开销(network overhead)（持续的TCP连接）
   - 减少master存储的metadata大小
- disadvantages：
   - 一些chunkserver变成hot spot，收到并发请求时负载过重
