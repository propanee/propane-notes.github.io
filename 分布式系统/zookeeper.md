### znode

- 数据节点
- hierarchical name space 层级命名空间

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/1109%20Illustration%20of%20ZooKeeper%20hierarchical%20name%20space.png" alt="image-20231109204517682" style="zoom:50%;" />

- 两种类型
  - **Regular**: Clients manipulate regular znodes by **creating and deleting them explicitly**;  必须显式创建和删除。
  - **Ephemeral**: Clients create such znodes, and they either delete them explicitly, or **let the system remove them automatically when the session that creates them terminates** (deliberately or due to a failure)  创建它们的session终止时，系统可自动移除。
- sequential flag
  - 带有sequential flag的node的名字拥有单增计数器。
  - If n is the new znode and p is the parent znode, then the sequence value of n is never smaller than the value in the name of any other sequential znode ever created under p.
- version
  - Znodes also have associated meta-data with time stamps and version counters, which allow clients to track changes to znodes and execute conditional updates based on the version of the znode  

### watch flag

- `getData(‘‘/foo’’, true)`  
- Watches are **one-time triggers** associated with a session
- ZooKeeper implements watches to allow clients to receive timely notifications of changes without requiring polling  

### Data model

- file system  with a simplified API and only full data reads and writes ；or a key/value table with hierarchical keys  

- znodes map to abstractions of the client application, typically corresponding to meta-data used for coordination purposes. znode. 不是用来进行一般的数据存储，而是映射到客户端应用程序的抽象，通常是用于协调的元数据。可以存储分布式计算中的一些meta-data或configuration信息。
  - To illustrate, in Figure 1 we have two subtrees, one for Application 1 (/app1) and another for Application 2 (/app2). The subtree for Application 1 implements a simple group membership protocol: each client process `p_i` creates a znode `p_i` under /app1, which persists as long as the process is running.

### Sessions 

- A client connects to ZooKeeper and initiates a session；
- session结束：client显式关闭或认为该client faulty (在超时前zookeeper没有从该session收到任何东西)

# Zookeeper

- 高性能(high perference)：

  - 异步(asynchronous)

  - 提供一致性，但非强一致性(consistency)


- 通用的协调服务(Coordination service)

# Replicated State Machine

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/1009%20zookeeper%E5%A4%8D%E5%88%B6%E7%8A%B6%E6%80%81%E6%9C%BA.png" alt="image-20231109220036387" style="zoom:50%;" />

 首先，Zookeeper是一个复制状态机(zookeeper is replicated state machine)。

 类似Raft流程，Zookeeper对外服务时：

1. Client访问Zookeeper，create一个Znode
2. Zookeeper调用ZAB库(类似Raft库)，leader将操作放入ZAB，ZAB生成log，并负责和其他节点进行交互，完成类似Raft的工作，日志同步、心跳保活等(整体思想和Raft类似，但实现方式不同)
   - ZAB保证日志顺序
   - 能避免网络分区、脑裂问题
   - 同样要求操作都是具有确定性的(换言之不会有非确定性的操作在ZAB之间传递)
3. ZAB完成日志同步等工作后，回应Zookeeper
4. Zookeeper处理逻辑，响应Client的create请求

 在Zookeeper中，维护着由znode组成的tree树结构。

 由于ZAB整体思路和Raft类似，这章节不会再重点讨论ZAB，我们主要围绕Zookeeper展开话题

# 吞吐量

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/image-20231109221704833.png" alt="image-20231109221704833" style="zoom: 67%;" />

- 写请求占比高时，增加服务器会降低吞吐量（3台21K/s，13台9K/s）
- 读请求占比高时，Zookeeper实例机器越多，吞吐量越高；读吞吐量与服务器数量成正比

 高吞吐的原因，Zookeeper有以下几种策略/设计：

- **异步(asynchronous)**

  操作都是异步的，Client可以一次向Zookeeper提交多个操作。比如Client一次提交多个put(批量操作，不等待put的响应)，但是实际上leader只向持久存储写入一次(而不是每次操作都写入)，即Zookeeper批量整合多次写操作，只进行一次磁盘写入操作。

- **读请求可以由任意server处理(read by any server)**

  - 这里不需要通过leader完成read操作，任何zookeeper实例节点都可以完成read操作
  - **并且其他zookeeper完成read操作时，不需要和leader有任何网络通信**

-----

读请求由任意server处理 (read by any server)

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/image-20231109224225509.png" alt="image-20231109224225509" style="zoom: 33%;" />

*L: leader; F: follower; C: client*

1. C1向L进行put写请求（将X从0写成1）
2. 根据majority原则，只需要L和F1将log写入稳定存储，L响应C1
3. C2再通过get读取X，即使put过去了很久，读取结果可能是0或1：
   - 读取X为0，C2正好读取到F2，而F2没有被写入log，所以X还是0
   - 读取X为1，C2正好读取到L或F1，所以X是1
4. C2如果上一次读取到了1，还可能读取到0吗？
   - 可以，不一定访问到和上次一样的节点，尽管大多数节点收到put，也可能访问到还是旧值的节点。（读到一些过去的信息back in time）

该场景不是线性一致性(Linearizability)的：

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/1112%20zookeeper%E7%BA%BF%E6%80%A7%E4%B8%80%E8%87%B4%E6%80%A7.png" alt="image-20231112110814296" style="zoom: 33%;" />


- a total order of ops
- order match real-time(1在2开始之前完成，那么total order上也需满足)
  - C在put后的很久后再进行get，即put在get开始前就完成，那么get需要返回新值X=1，才能满足"实时匹配(match real-time)"
- read op returns value of last write
  - 最后一次put导致X=1，那么很长时间后进行的get，应该读取X=1而不是X=0。

 **这里如果想保证read能维持线性一致性，简单的做法就是每次read都请求leader**。当然你可以用其他方式来保证。

# 一致性规则

> [8.5 一致保证（Consistency Guarantees） - MIT6.824 (gitbook.io)](https://mit-public-courses-cn-translatio.gitbook.io/mit6-824/lecture-08-zookeeper/8.5) 
>
> Zookeeper的一致性规则：
>
> 1. **写请求是线性一致性的(linearizable writes)**
>
>    对于所有的客户端发起的写请求，整体是线性一致性的。
>
> 2. **同一个客户端的请求按照FIFO顺序被执行(FIFO client order)**
>
>    这里的意思是，如果一个特定的客户端发送了一个写请求之后是一个读请求或者任意请求，那么首先，所有的写请求会以这个客户端发送的相对顺序，加入到所有客户端的写请求中（满足保证1）。所以，如果一个客户端说，先完成这个写操作，再完成另一个写操作，之后是第三个写操作，那么在最终整体的写请求的序列中，可以看到这个客户端的写请求以相同顺序出现（虽然可能不是相邻的）。所以，对于写请求，最终会以客户端确定的顺序执行。
>
>    对于读请求，这里会更加复杂一些。我之前说过，在Zookeeper中读请求不需要经过Leader，只有写请求经过Leader，读请求只会到达某个副本。所以，读请求只能看到那个副本的Log对应的状态。对于读请求，我们应该这么考虑FIFO客户端序列，客户端会以某种顺序读某个数据，之后读第二个数据，之后是第三个数据，对于那个副本上的Log来说，每一个读请求必然要在Log的某个特定的点执行，或者说每个读请求都可以在Log一个特定的点观察到对应的状态。然后，后续的读请求，必须要在不早于当前读请求对应的Log点执行。也就是一个客户端发起了两个读请求，如果第一个读请求在Log中的一个位置执行，那么第二个读请求只允许在第一个读请求对应的位置或者更后的位置执行。**第二个读请求不允许看到之前的状态，第二个读请求至少要看到第一个读请求的状态。这是一个极其重要的事实，我们会用它来实现正确的Zookeeper应用程序**。
>
>    这里特别有意思的是，如果一个客户端正在与一个副本交互，客户端发送了一些读请求给这个副本，之后这个副本故障了，客户端需要将读请求发送给另一个副本。这时，尽管客户端切换到了一个新的副本，FIFO客户端序列仍然有效。所以这意味着，如果你知道在故障前，客户端在一个副本执行了一个读请求并看到了对应于Log中这个点的状态，当客户端切换到了一个新的副本并且发起了另一个读请求，假设之前的读请求在这里执行，那么尽管客户端切换到了一个新的副本，客户端的在新的副本的读请求，必须在Log这个点或者之后的点执行。
>
>    **这里工作的原理是，每个Log条目都会被Leader打上zxid的标签，这些标签就是Log对应的条目号**。任何时候一个副本回复一个客户端的读请求，首先这个读请求是在Log的某个特定点执行的，其次回复里面会带上zxid，对应的就是Log中执行点的前一条Log条目号。客户端会记住最高的zxid，当客户端发出一个请求到一个相同或者不同的副本时，**它会在它的请求中带上这个最高的zxid**。这样，其他的副本就知道，应该至少在Log中这个点或者之后执行这个读请求。这里有个有趣的场景，**如果第二个副本并没有最新的Log，当它从客户端收到一个请求，客户端说，上一次我的读请求在其他副本Log的这个位置执行，那么在获取到对应这个位置的Log之前，这个副本不能响应客户端请求**。
>
>    **更进一步，FIFO客户端请求序列是对同一个客户端的所有读请求，写请求生效**。所以，如果我发送一个写请求给Leader，在Leader commit这个请求之前需要消耗一些时间，所以我现在给Leader发了一个写请求，而Leader还没有处理完它，或者commit它。之后，我发送了一个读请求给某个副本。这个读请求需要暂缓一下，以确保FIFO客户端请求序列。读请求需要暂缓，直到这个副本发现之前的写请求已经执行了。这是FIFO客户端请求序列的必然结果，（对于某个特定的客户端）读写请求是线性一致的。最明显的理解这种行为的方式是，如果一个客户端写了一份数据，例如向Leader发送了一个写请求，之后立即读同一份数据，并将读请求发送给了某一个副本，那么客户端需要看到自己刚刚写入的值。如果我写了某个变量为17，那么我之后读这个变量，返回的不是17，这会很奇怪，这表明系统并没有执行我的请求。因为如果执行了的话，写请求应该在读请求之前执行。所以，副本必然有一些有意思的行为来暂缓客户端，比如当客户端发送一个读请求说，我上一次发送给Leader的写请求对应了zxid是多少，这个副本必须等到自己看到对应zxid的写请求再执行读请求。

 **再次声明，Zookeeper并不提供读的线性一致性**("9.2 Zookeeper吞吐量"有进行过场景分析了)。

 **Zookeeper并不是遵循线性一致性实现的，它有自己的一致性定义和实现**：

- **线性化写入(Linearize write)**

  所有的写入操作，必须经过Zookeeper的leader。底层会将生成的log以相同的顺序同步到所有zookeeper节点的log记录中。(即对于写入操作，还是应用复制状态机的方案进行majority的写同步)

- **先进先出队列维护Client请求(FIFO client order)**

  尽管请求是异步的，所有的client请求，在zookeeper实例中以FIFO的形式维护。即如果client1先于client2请求，那么client2的处理一定在client1之后，client2的请求能观测到client1的操作结果。

  - **按client请求进行写操作(write in client order)**

    这是FIFO client order的附带产物，很好理解。因为client请求以FIFO顺序被zookeeper处理，所以写操作也是按照client请求顺序生效。

  - **读操作能观测到同一个client的最后一次写操作(read obverse last write from same client)**

    **如果相同的client执行完写操作后，立即进行读操作，那么至少可以看到自己的写入结果。如果是其他client在前一个client写后进行读，zookeeper并不保证能读取到前一个client的写结果**。

  - **读操作会观察到日志的某些前缀(read will observe some prefix of the log)**

    这意味着你可以看到旧数据 (stale data)，可能某个follower有指定前缀的log，但是log里面没有最新的值记录，但是follower仍然可以返回读取的结果。因为zookeeper只保证读取可以观察到log的前缀。按照这个特定，读取可以不按照顺序。

  - **不能读取旧前缀的数据(no read from from the past)**

    这意味着如果你第一次看到了前缀prefix 1，然后你发出读取前缀prefix 1。然后你执行第二次读取，第二次读取必须至少看到prefix 1或者更大的前缀数据(比如prefix 2)，但是绝不能看到比precix 1更小的前缀。
  
  **因此zookeeper允许C1put后C2读取到旧值，但不允许C2先读取到新值又读取到旧值。**

# Zookeeper的一致性实现

> [8.6 同步操作（sync） - MIT6.824 (gitbook.io)](https://mit-public-courses-cn-translatio.gitbook.io/mit6-824/lecture-08-zookeeper/8.6-tong-bu-cao-zuo-sync)
>
> Zookeeper有一个操作类型是sync，它本质上就是一个写请求。假设我知道你最近写了一些数据，并且我想读出你写入的数据，所以现在的场景是，我想读出Zookeeper中最新的数据。这个时候，我可以发送**一个sync请求，它的效果相当于一个写请求，所以它最终会出现在所有副本的Log中**，尽管我只关心与我交互的副本，因为我需要从那个副本读出数据。接下来，在发送读请求时，我（客户端）告诉副本，在看到我上一次sync请求之前，不要返回我的读请求。
>
> **如果这里把sync看成是一个写请求，这里实际上符合了FIFO客户端请求序列，因为读请求必须至少要看到同一个客户端前一个写请求对应的状态。所以，如果我发送了一个sync请求之后，又发送了一个读请求。Zookeeper必须要向我返回至少是我发送的sync请求对应的状态**。
>
> 不管怎么样，如果我需要读最新的数据，我需要发送一个sync请求，之后再发送读请求。这个读请求可以保证看到sync对应的状态，所以可以合理的认为是最新的。但是同时也要认识到，这是一个代价很高的操作，因为我们现在将一个廉价的读操作转换成了一个耗费Leader时间的sync操作。所以，如果不是必须的，那还是不要这么做。

 论文中没有特别详细的说明Zookeeper怎么实现"9.3 Zookeeper的一致性规则"中制定的规则，但是我们可以大致猜测其实现。

1. **当客户端想要连接zookeeper服务时，会创建一个session**，它使用session信息连接到zookeeper集群，并在整个session期间维护状态state。
2. Client创建session后和Zookeeper的leader建立连接，随后发起一个write写请求
3. 类似Raft流程，leader生成一个log插入到log记录中，其索引称为zxid。
4. leader类似Raft进行log日志同步，将log同步到majority数量的节点(包括自己)
5. **leader响应Client，同时返回zxid给客户端，客户端会在session中存储这次写入的zxid信息**
6. **Client向zookeeper发起read请求，这个read使用上次写入返回的zxid标记**
7. **假设被请求的follower正好没有这个zxid对应的log记录，于是它会等待直到收到leader的这条zxid同步log后，才会响应Client**
8. 此时假设Client又发起write请求，并且最后在leader和follower1产生zxid=1
9. 随后Client发起read请求，但携带的zxid=0(假设是最早一次写返回的)，并且读取的是follower2
10. **follower2正好有zxid=0的log记录，尽管它还没有zxid=1的记录，但是它仍会响应client的读zxid=0的请求**。**这意味着Zookeeper存在读取旧值的情况(但不允许主动读取旧值)**，这里说不定zxid=1的记录是某个变量的新值，而zxid=0则是旧值（注意，这里如果follower2已经有了zxid=1的记录后，会返回zxid=1的值，而不是返回zxid=0的值，因为不允许返回旧log前缀的数据）。

------

问题：据我所知，好像一般session具有粘性，即上面读写应该会尽可能发生在同一个zookeeper节点，所以上面流程是可能发生的吗？

回答：这里可以假设曾经发生过一个短暂的网络分区或类似的事情，所以上面的流程是可能发生的。另外zookeeper做了一些负载均衡，但是无论如何，上面流程是可能发生的。

**问题：上面写入时说zookeeper会返回zxid，是不是需要已经commit了才会返回zxid？**

**回答：是的，类似Raft流程，这里需要log被commit后，才会返回zxid。**

**问题：这里后续的read携带zxid=0，但是系统中已经有节点存在zxid=1的数据了，看上去似乎违背了zookeeper一致性原则中的"不能读取旧前缀的数据(no read from from the past)"的规则？因为看上去client好像在故意请求日志中特定的前缀数据，而且还是读取旧的数据。**

**回答：实际上，这里zxid含义上表示阻止时间倒流的计数器，作为follower必须至少含有zxid值截止的日志前缀，才能够响应client，而如果你有更高的zxid并不会有问题。这阻止了故意读取旧数据的可能性。**

学生提问：也就是说，从Zookeeper读到的数据不能保证是最新的？

Robert教授：完全正确。我认为你说的是，从一个副本读取的或许不是最新的数据，所以Leader或许已经向过半服务器发送了C，并commit了，过半服务器也执行了这个请求。但是这个副本并不在Leader的过半服务器中，所以或许这个副本没有最新的数据。这就是Zookeeper的工作方式，它并不保证我们可以看到最新的数据。Zookeeper可以保证读写有序，但是只针对一个客户端来说。所以，如果我发送了一个写请求，之后我读取相同的数据，Zookeeper系统可以保证读请求可以读到我之前写入的数据。但是，如果你发送了一个写请求，之后我读取相同的数据，并没有保证说我可以看到你写入的数据（这里同一个人表示同一个client）。这就是Zookeeper可以根据副本的数量加速读请求的基础。

学生提问：那么Zookeeper究竟是不是线性一致呢？

Robert教授：我认为Zookeeper不是线性一致的，但是又不是完全的非线性一致。首先，所有客户端发送的请求以一个特定的序列执行，所以，某种意义上来说，**所有的写请求是线性一致的**。同时，每一个客户端的所有请求或许也可以认为是线性一致的。尽管我不是很确定，Zookeeper的一致性保证的第二条可以理解为，**单个客户端的请求是线性一致的**。

学生提问：zxid必须要等到写请求执行完成才返回吗？

Robert教授：实际上，我不知道它具体怎么工作，但是这是个合理的假设。当我发送了异步的写请求，系统并没有执行这些请求，但是系统会回复我说，好的，我收到了你的写请求，如果它最后commit了，这将会是对应的zxid。所以这里是一个合理的假设，我实际上不知道这里怎么工作。之后如果客户端执行读请求，就可以告诉一个副本说，这个zxid是我之前发送的一个写请求。

# Zookeeper中watch通知

> [Zookeeper——Watch机制原理_庄小焱的博客-CSDN博客_zookeeper watch](https://blog.csdn.net/weixin_41605937/article/details/122095904) <= 推荐阅读
>
> 大体上讲 ZooKeeper 实现的方式是通过客服端和服务端分别创建有观察者的信息列表。客户端调用 getData、exist 等接口时，首先将对应的 Watch 事件放到本地的 ZKWatchManager 中进行管理。服务端在接收到客户端的请求后根据请求类型判断是否含有 Watch 事件，并将对应事件放到 WatchManager 中进行管理。
>
> 在事件触发的时候服务端通过节点的路径信息查询相应的 Watch 事件通知给客户端，客户端在接收到通知后，首先查询本地的 ZKWatchManager 获得对应的 Watch 信息处理回调操作。这种设计不但实现了一个分布式环境下的观察者模式，而且通过将客户端和服务端各自处理 Watch 事件所需要的额外信息分别保存在两端，减少彼此通信的内容。大大提升了服务的处理性能。
>
> **需要注意的是客户端的 Watcher 机制是一次性的，触发后就会被删除。**
>
> [zookeeper的watch机制原理解析_java_脚本之家 (jb51.net)](https://www.jb51.net/article/252975.htm) <= 推荐阅读，具体的指令操作示例，很好理解
>
> ------
>
> [8.7 就绪文件（Ready file/znode） - MIT6.824 (gitbook.io)](https://mit-public-courses-cn-translatio.gitbook.io/mit6-824/lecture-08-zookeeper/8.7-jiu-xu-wen-jian-ready-fileznode) <= 2021版这里讲得不太清楚，可以看看网络上2020版对于watch的讲解

 这里举了一些例子，大概就是在讨论怎么让read读取到最后的write。然后提到zookeeper提供的watch监听指定写事件的变化，**zookeeper保证被监听的写事件发生后，会在下一次写操作发生前完成事件通知**。

# ZNode API

> [9.1 Zookeeper API - MIT6.824 (gitbook.io)](https://mit-public-courses-cn-translatio.gitbook.io/mit6-824/lecture-09-more-replication-craq/9.1-zookeeper-api) <= 可以参考这个笔记，里面更详细
>
> 回忆一下Zookeeper的特点：
>
> - Zookeeper基于（类似于）Raft框架，所以我们可以认为它是，当然它的确是容错的，它在发生网络分区的时候，也能有正确的行为。
> - 当我们在分析各种Zookeeper的应用时，我们也需要记住Zookeeper有一些性能增强，使得读请求可以在任何副本被处理，因此，可能会返回旧数据。
> - 另一方面，Zookeeper可以确保一次只处理一个写请求，并且所有的副本都能看到一致的写请求顺序。这样，所有副本的状态才能保证是一致的（写请求会改变状态，一致的写请求顺序可以保证状态一致）。
> - 由一个客户端发出的所有读写请求会按照客户端发出的顺序执行。
> - 一个特定客户端的连续请求，后来的请求总是能看到相比较于前一个请求相同或者更晚的状态（详见8.5 FIFO客户端序列）。

 通常来说会给应用app1创建一个znode节点，然后znode下再挂在一些子节点，上面记录app1的一些属性，比如IP等。

 ZNode有三种类型：

- regular：常规节点，它的容错复制了所有的东西
- ephemeral：临时节点，节点会自动消失。比如session消失或Znode有段时间没有传递heartbeat，则Zookeeper认为这个Znode到期，随后自动删除这个Znode节点
- sequential：顺序节点，它的名字和它的version有关，是在特定的znode下创建的，这些子节点在名字中带有序列号，且节点们按照序列号排序（序号递增）。

------

- create接口，参数为path、data、flags。`create(path, data, flags)`，这里flags对应上面ZNode的3种类型
- delete接口，参数为path、version。`delete(path, version)`
- exists接口，参数为path、watch。`exists(path, watch)`
- getData原语，参数为path、version。`getData(path, version)`
- setData原语，参数为path、data、version。`setData(path, data, version)`
- getChildren接口，参数为path、watch。`getChildren(path, watch)`，可以获取特定znode的子节点

# Zookeeper-小结

- 较弱的一致性(weaker consistency)
- **适合用于当作配置服务(configuration)**
- 高性能(high perference)