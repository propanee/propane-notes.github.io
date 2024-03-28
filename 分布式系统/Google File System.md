# <mark>Google File System</mark> 

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

# 一致性目标和问题
- Ideal consistency -->Behave as if single system
- **并发性concurrency**:
>C1写入x1，C2写入x2，C3读取x，C4读取x。不关心谁先谁后时，希望C3和C4读取到的值是一致的。服务器在并发存在的情况下可以使用锁来强制一致性（分布式锁等）

- **故障Failure**
>对于故障一般使用复制解决。而不成熟的复制操作，会导致读者在不做修改的情况下读取到两次不同的数据。Bad replication plan: 当C1、C2并发向两个备份S1和S2分别写入数据1和2，在写操作都完成后，C3、C4从S1或S2读数的结果可能不同
>（因为没有明确的协议指出这里W1和W2的数据在S1、S2上以什么方式存储，可能1被2覆盖，反之亦然）

# 假设
- 系统建立在便宜且经常出错的组件上；
- 系统需要存储数量不多的大型文件；
- 主要分为两种读操作：大型流式读取（streaming read）（连续的）和小型随机读取；
- 写操作：许多大的、顺序的写操作吧数据加到文件末尾；
- 系统必须实现定义良好的语义，让多方用户并发地添加数据到同一个文件；
- 高稳定的带宽比低时延更重要。
# GFS
GFS的几个主要特征:
- Big：large data set，巨大的数据集
- Fast：automatic sharding，自动分片到多个磁盘
- Gloal：all apps see same files，所有应用程序从GFS读取数据时看到相同的文件（一致性）
- Fault tolerance：automic，尽可能自动地采取一些容错恢复操作

# interface接口
 - 文件按层次目录组织，用路径名标识；
 - create, delete, open, close, read, write... 与通常的文件系统接口类似；
 - **snapshot** creates a copy of a file or a directory tree at low cost. 
 - **record append** allows multiple clients to append data to the same file concurrently  - while guaranteeing the atomicity of each individual client’s append.(原子的，不需要额外的锁)

# 架构
- **Non-standard** 
   - 只有一个master而不是复制的，存在单点故障的问题
   - 有不一致性，in-consistencies

single master + multiple chunkservers + multiple clients（可与multiple chunkservers在同一机器）

## master
- **metadata:** 
   - file and chunk namespaces (**persistent**)
   - mapping from files to chunks(array of chunk handles) (**persistent**)
      - chunk handle=>版本号（version#）& list of chunk servers & **主服务器primary和次要服务器secondaries**，以及主服务器有释放时间lease time
   - locations of each chunk's replicas (**not persistent**）
      - 一般为3个。
      - not persistent，在启动时请求chunkservers的chunk信息，之后周期性地请求。
   - access control information
   - 这些信息大多存储在内存中，所以master可以快速响应client
   
- **operation log + checkpoint**
   - contains a historical record of critical metadata changes，所有变更操作（mutations）都会写入这个日志，并放在稳定的存储里（replicate it on multiple remote machines，both locally and remotely） ，然后才响应client。
   - 当master故障，可以通过重放（replay）日志重建内部状态；
   - 定期创建自己状态的检查点。当log超过一定大小就记录其checkpoint（最小化启动时长），以便从最新的checkpoint **recover**（compact B-tree)
   - master切换到新的log文件，在另一个线程创建checkpoint，所以不需要delay incoming mutations
   
------

> - **哪些状态要存放在稳定的存储（放在log）中？**
> - file name =>array of chunk handles? **需要**。否则master崩溃后会丢失文件。
> - chunk handle =>list of chunk servers?**不需要**。master重启时会要求其他存储chunk数据的服务器说明自己维护的chunk handles数据。这里master只需要内存中维护即可。同样的，主服务器(primary)、次要服务器(secondaries)、主服务器(primary)的租赁时间(lease time)也都只需要在内存中即可。
> chunk handle =>version number?**需要**。否则master崩溃重启时，master无法区分哪些chunk server存储的chunk最新的。比如可能有服务器存储的chunk version是14，由于网络问题，该服务器还没有拿到最新version 15的数据，master必须能够区分哪些server有最新version的chunk。为什么不能在重启后找最大的版本号？因为有最新版本号的server可能正好也故障了。

- **system-wide activities **:chunk lease management/ garbage collection of orphaned chunks/ chunk migration between chunkservers. 
   - 告诉client它应该与哪些chunkservers通信。
- **Heartbeat** message with chunkservers

## client
- 实现文件系统API
- 与chunkservers和master通信，来为应用读写数据

**client和chunkserver都不缓存文件数据**

- client因为需要巨大的流式处理。但是缓存一段时间matedata。
- chunkservers因为chunks存储在本地文件（？）。

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


# 流程（read）：
![GFS_Architecture](https://raw.githubusercontent.com/propanec/propane-img/main/PicGo-img/GFS_Architecture.png)
- client将应用指定的file name + byte offset转化为文件中的chunk index；
- 向master发送包含file name 和chunk index的请求；（可能请求多个chunk）
- master返回对应的chunk handle + locations of the replicas(list of chunk server) + version #；
- client将file name和chunk index作为key缓存这个信息；
- client向其中一个replica（closest one）发送指定chunk handle和byte range的请求
- chunk server检查version #, 正确则返回数据

-----
>**client为什么要缓存list of chunk server?**
>- master只有一台，需要减少其负载。直到缓存信息expires或file is reopened都不需要再和master交互了。
>**为什么从最近的chunk server读？**
>- 最大限度减少网络流量（mininize network traffic），最大限度提高吞吐量。同时减少网路拓扑带来的时延。
>**为什么要检查版本号？**
>- 避免client读取过时的数据


# 流程（write：append）
## lease
lease: maintain a consistent mutation order across replicas. 最小化master的管理开销
- master向一个副本（primary）授予chunk lease，定义全局的mutation order，在租期内，primary选取所有对chunk的mutation的顺序，其他副本接收mutation时跟随这个顺序。
- timeout 60 seconds，但是可以无限期（indefinitely）通过*heartbeat*向master请求继续租期（extension）。
- master 也可以在租约到期前撤销（revoke）租约（如master想要禁用被重命名的文件上的mutations时）。
- 当master无法和primary通信时也可以在旧的租约到期后授予其他副本租约。

## 流程
![Write control](https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/GFS-write.png)
1. client（C） 询问master（M）有当前租期的chunk server（primary）和其他副本（secondary）的位置。
   - **M查询 filename=>chunk handle=>chunk server**
   - 如果没有primary ：M增加并发送版本号给复制组（有新的primary，则进入了new epoch，所以需要新的版本号），这些server将version # 保存在磁盘（而不是本地内存，否则重启后内存数据丢失，需要让master信服自己有哪些version的数据，master同理）。同时选择一个server给予lease。 
2. **M响应PR的id和SR的位置，以及版本号。**
  - C只有在无法与PR通信或PR不再持有lease时才再次和M通信。
3. **C向所有副本推送数据**（
	- 基于拓扑，只需要最近的，不管哪个是primary。之后这个server会推送给其他server。=>提高了C吞吐量，避免消耗大量网络接口资源向所有server都发送数据。
	- 每个**server缓存数据**直到被使用或超时。
5. **C向primary发送写请求**，标记所有先前推送的数据。primary为所有的mutation（可能来自多个client）分配连续的序列号（serial number），并按顺序接受mutation。
	- primary需要检查版本号，且租期需有效，无效时外部可能另有新的primary。
7. **primary向所有secondary推送写请求**，SR按相同的序列号接受mutation。
8. **secondary向primary响应**完成了所有操作。
9. **primary响应C**，并报告错误。

------

>可能的错误：在primary上成功写入，但有secondary没有响应，此时修改的区域是inconsistent状态，C通常在3-7采取一些措施，重新发布相同的操作以重试。（**at-least-once**）
>- 如果append失败，C重试时，primary会指定一个新的offset。假设primary+2台secondaries，可能上一次p和s1都写成功，仅s2失败。此时retry需要用新的offset，或许p、s1、s2就都写入成功了。
>- **副本记录是可以重复的(replicates records can be duplicated)**，这和标准的文件系统不一样。
>- 应用程序不需要直接和这种特殊的文件系统交互，而是通过库操作，库的内部实现隐藏了这些细节，用户不会看到曾经失败的副本记录数据。
>- 如果append数据，库会为其绑定一个id，如果库读取到相同id的数据，会跳过前面的一个。
>- 同时库内通过checksums检测数据的变化，同时保证数据不会被篡改。

-----

>写入流程中提到需要校验version，是否存在场景就是Client要读取旧version的数据？比如Client自己存储的version就是旧版本，所以读取数据时，拥有旧version的chunk server反而能够响应数据。
>- 假设有P，S1，S2和C，S2和P、S1断开了联系，此时重新选出Primary(假设还是原本的P)，version变为11，而S2断联所以还是version10的数据。一个本地cache了version10和list of chunk sevrers的Client正好直接从S2请求读取数据（因为S2最近），而S2的version和Client一致，所以直接读取到S2的数据。
>上面举例中，为什么version11这个信息不会返回到Client呢？
>- 原因可能是Client设置的cache时间比较长，并且协议中没有要求更新version。
>上面举例中，version更新后，不会推送到S2吗？即通知S2应该记录最新的version为11
>- version版本递增是在master中维护的，version只会在选择新的primary的时候更新。论文中还提到serial number序列号的概念，不过这个和version无关，这个用于表示写入顺序。
>primary完成写入前，怎么知道要检查哪一些secondaries（是否完成了写入）？
>- master告知的。master告诉primary需要更新secondaries，这形成了new replica group for that chunk。


# 一致性模型
[GFS文件系统剖析（中）一致性模型及读写流程介绍](https://sq.sf.163.com/blog/article/172834757163061248)

这里一致性问题，可以简单地归结于，当你完成一个append操作后，进行read操作会读取到什么数据？

这里假设一个场景，我们讨论一下可能产生的一致性问题，这里有一个M（maseter），P（primary），S（Secondary）：

- **某时刻起，M得不到和P之间的ping-pong通信的响应，什么时候Master会指向一个新的P？**

  1. M比如等待P的lease到期，否则会出现两个P
  2. 此时可能有些Client还在和这个P交互

  **假设我们此时选出新P，有两个P同时存在，这种场景被称为脑裂(split-brain)，此时会出现很多意料外的情况，比如数据写入顺序混乱等问题，严重的脑裂问题可能会导致系统最后出现两个Master**。

  这里M知道P的lease期限，在P的lease期限结束之前，不会选出新P，即使M无法和P通信，但其他Client可能还能正常和P通信。

------

这里扩展一下问题，**如何达到更强的一致性(How to get stronger consistency)？**

比如这里GFS有时候写入会失败，导致一部分S有数据，一部分S没数据（虽然对外不可见这种失败的数据）。我们或许可以做到要么所有S都成功写入，要么所有S都不写入。

事实上Google还有很多其他的文件系统，有些具有更强的一致性，比如Spanner, 且支持事务，其应用场景和这里不同。我们知道这里GFS是为了运行mapreduce而设计的。