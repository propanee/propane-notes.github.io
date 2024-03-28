# Approaches to RSMs

这里简单提一下两种构造复制状态机(RSM, replicated state machine)的方案

1. Run all options through Raft/Paxos
   - 通过Raft/Paxos或其他共识算法，完成所有的操作；
   - 事实证明，这种通过Raft/Paxos这类共识算法运行所有操作的方式，在实际应用中并不常见，并不完全是一种标准的方法（指用于实现复制状态机）。后续会介绍一些其他的设计来实现这一点，比如Spanner。
2. **Configuration service + P/B replication**：
   - **更常见的是通过一个配置服务器**（比如zookeeper），配置服务内部可能使用Paxos、Raft、ZAB或其他共识算法框架，配置服务扮演coordinator或master的角色。**除了采用共识算法实现配置服务外，还可以运行主备复制(primary backup replicaiton)**。
   - 比如GFS的master，决定哪组服务器保存了哪些chunk，以及the replica group of a chunk。然后the replica group of a chunk执行主备复制。
   - 而VM-FT中，配置服务器是test-and-set服务器，决定谁是primary或backup，P/B通过channel大致同步log等数据。

 相比第一个方案，第二种方案更常见和通用。

 **采用配置服务器+主备复制的特点**：

- **复制服务在状态state维护的成本较低，一般需要维护的数据量很少**
- **主备复制则主要负责大量的数据复制工作**

 而直接用共识算法实现复制状态机，往往意味着需要直接在提供服务的server之间来回进行大量的数据复制，检查点的state数据同步等工作，一般实现会更复杂。

------

> 问题：方案1比起方案2有什么好处吗？
>
> 回答：你不需要同时使用它们两个，只需要选择一种。方案1中，你每运行操作，Raft就为你进行同步，所有的东西都集成在单一的组件中（包括主备同步等操作）；而在方案2中，我们拆分出2个组件，一个配置服务中可以包括Raft，而同时我们有主备方案，这分工更加清晰。
>
> 问题：那方案2有什么优势吗？如果通过leader达成共识，leader永远不会失败，对吧。
>
> 回答：方案2的优势，在随后准备介绍的链式复制中会看到。有一个单独的进程负责配置部分的工作，你不必担心你的主备复制方案。对比的话，GFS的master指定某几个server组成一个特定的replica group。而这里主备协议不用考虑这个问题。
>

# 链式复制Overview

> [MIT 6.824 - Chain Replication for Supporting High Throughput and Availability - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/519927237)

 链式复制，就是上面提到的方案2的主备复制方案(P/B replication)。

 在论文中，**链式复制(Chian replicaiton)假设系统中存在一个配置服务(Configuration service)作为master，然后链式复制本身有一些属性(properties)**：

1. Reads ops involve 1 server 读/查询操作(read / query op)只涉及一个服务器；
2. simple recovery plan 具有简单的恢复方案；
3. linearizability 线性一致性()；

 整体来说，这是一个很成熟的设计，已经应用到诸多系统中。

![img](https://pic1.zhimg.com/80/v2-4f029c98e6d963ab6233557dc83fba2c_1440w.webp)

首先有一个配置服务(configuration service)记录链式连接的节点信息（它实际上是链的一部分？）。

1. client发起write请求，**write写请求总是发送到head节点**；
2. head节点生成log等，通过storage稳定存储更新相关state状态，然后传递操作到下一个节点；
3. 下一个节点同样更新自己storage中的state，然后向下一个节点传递操作；
4. 直到tail修改storage中的状态后，**tail向client回应确认信息**。

可通过增加链中的节点数量来获取更高的availability。

**这里tail节点就是提交点(commit point)**。

- **写请求总是发送到head节点**

  head更新storage中的state后，向下传递操作，直到tail。中间节点总是更新state，然后向下传递操作。最后由tail节点响应client。

- **读请求总是发送到tail节点**

  任何client发起读请求，都会请求tail节点，而tail节点会直接响应。

读写工作负载至少分布在两台服务器上。对于读操作，只需要一台服务器支持，且能够快速回应，这使得我们有很多优化空间（后续介绍）。

**线性一致性**：所有写操作按total order在head被应用，到tail收到更新后才响应客户端，实际应答读和写请求的只有tail节点。只要系统没有崩溃的情况下，client进行先写后读操作，一定会观察到最后一次或最近一次写入的结果，但不会读到旧数据，所以在单个client中所有操作都是完全有序的）。

如果写请求直接在head完成后直接由head回应client，那么若后续节点更新前，client向tail发送读请求，它甚至无法观测到自己刚刚的写入，明显违背了线性一致性。

# Crash

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/1121%20%E9%93%BE%E5%BC%8F%E5%A4%8D%E5%88%B6crash.png" alt="image-20231121151428247" style="zoom:50%;" />

- 头节点故障(head fail)

   - 配置服务发现S1 is gone，抛弃S1，提升S2成为新的head，更新链配置并通知所有服务器。
   - 1未向下同步的log记录可以丢弃，因为还没有真正commit。之后client请求失败后会重试，并请求新的head，S2节点。

   - 可以让S2自己决定S1宕机并成为head吗？不可以，这样会创造脑裂，因为S2和S1可能有网络分区，就会有两个head，也许都在处理命令，所以需要有一个configuration server。

- 中间节点故障(one of the intermediate server fail)

   - 配置服务器通知S1和S3需要组成新的链。
   - 由于S2还有些操作如U2未传给S3，所以需要额外流程将S3同步到和S1一致的状态。

- 尾节点故障(tail fail)

   - S3故障后，配置服务器通知S1、S2组成新链，其中S2成为新的tail。
   - client可以从配置服务器知道S2是新tail。其他的流程基本没变动。
   - 有一些操作可能会被S2自动提交如U2，此时client可能会重试，并需要重复检测，这里没有明确说明。

**链式复制的失败处理比起Raft要明显简单得多**：

- 整个系统以简单的链维护；
- 配置相关的工作交给了配置服务器；
- 对于主备复制部分，只有三种配置需要考虑。

# 新增副本(add replica into chain)

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/1121%20%E9%93%BE%E5%BC%8F%E5%A4%8D%E5%88%B6%20add%20replica.png" alt="image-20231121162648779" style="zoom:50%;" />

 正如论文所述，在tail添加新replica最简单，大致流程如下：

1. 假设原tail节点S2下新增S3，此时client依旧和原tail S2交互
2. S2会持续将同步信息传递给S3，并记录哪些操作是在S3开始复制之后进入的，需要保存已经更新的，但是未传播到S3的列表updates；
3. 某时刻S3和S2同步完成，S3通过配置服务告知S2，自己可以成为新的tail，S2回应updates;
4. S3处理完这些updates前会阻塞请求，结束后回应S2；
5. 配置服务设置S3为新tail；
6. 后续client改成请求tail节点获取读写响应（client可以通过配置服务知道谁是head、tail）

# 链式复制和Raft在主备复制的比较

光从主备复制的角度出发，比较一下链式复制CR(Chain Replication)和Raft。

CR比起Raft的优势：

- CR拆分请求RPC负载到head和tail；

  - CR的head负责接收write请求，tail负责响应wrtie请求和接收并响应read请求；
  - 不必像Raft通过leader完成write、read请求;

- head头节点仅发送update一次；

  - Raft的leader需要向其他所有followers都发送update；
  - CR的head节点只需要向链中的下一个节点发送update，涉及的messages更少；

- 读操作只涉及tail节点(read ops involve only tail)；

  - 即使在Raft中做read-only optimization，即read不经过log以及不附加到所有节点log上，还是需要leader与大多数节点联系来决定这是不是可服务的操作。

  - 而CR中，只需要tail负责read请求，并且tail一定能同步到最新的write数据（因为write的commit point就是tail节点）。

- 简单的崩溃恢复机制(simple crash recovery)

  与Raft相比，CR的崩溃处理更简单。

CR比起Raft的劣势：

- 一个节点故障就需要重新配置(one failure requires reconfiguration)

  - CR中write需要同步到整个链，write无法被确认，直到链中每个server都完成。所以一有fail，就需要重新配置，这意味着中间会有一小段时间的停机。
  
  - Raft中只需要majority接收了write并追加到日志中，系统就可以继续运作没有中断（如果剩下的server仍构成majority）。

# 并行读优化拓展(Extension for read parallelism)

链式复制中，只需要tail响应read请求，可以做一些优化，提高read吞吐量：**基本思路是进行拆分(split)对象，论文中称之为volume，将对象拆分到多个链中(splits object across many chain)**。

 例如有3个节点S1～S3，可以构造3个Chain链：

- Chain1：S1、S2、S3 (tail)
- Chain2：S2、S3、S1 (tail)
- Chian3：S3、S1、S2 (tail)

这里可以通过配置服务通过map，进行数据分片shard操作，如shard1中的objetc分配到chain1。如果数据被均匀write到Chain1～Chain3，那么读操作可以并行地命中不同的shards。均匀分布下，读吞吐量(read throughput)会线性增加，理想情况下这里能得到3倍的吞吐量。

1. **良好的read性能，并随服务器数量增加获得有效拓展 (shard越多，chain越多，read吞吐量理论上越高)；**
2. **保持线性一致性(linearizability)**；

------

问题：在拆分split的场景下，client怎么决定从哪个chain中读取数据？

回答：论文中没有详细说明，或许client可以通过配置服务器得知read请求对应哪个shard，再找出shard对应的chain，然后告知client该请求哪个chain的tail节点。

# 链式复制小结

> [9.7 链复制的配置管理器（Configuration Manager） - MIT6.824 (gitbook.io)](https://mit-public-courses-cn-translatio.gitbook.io/mit6-824/lecture-09-more-replication-craq/9.7-lian-fu-zhi-de-pei-zhi-guan-li-qi-configuration-manager) <= 摘抄我觉得有意义的问答内容：
>
> 学生提问：为什么存储具体数据的时候用Chain Replication，而不是Raft？
>
> Robert教授：这是一个非常合理的问题。其实数据用什么存并不重要。因为就算我们这里用了Raft，我们还是需要一个组件在产生冲突的时候来做决策。比如说数据如何在我们数百个复制系统中进行划分。如果我需要一个大的系统，我需要对数据进行分片，需要有个组件来决定数据是如何分配到不同的分区。随着时间推移，这里的划分可能会变化，因为硬件可能会有增减，数据可能会变多等等。Configuration Manager会决定以A或者B开头的key在第一个分区，以C或者D开头的key在第二个分区。至于在每一个分区，我们该使用什么样的复制方法，Chain Replication，Paxos，还是Raft，不同的人有不同的选择，有些人会使用Paxos，比如说Spanner，我们之后也会介绍。在这里，不使用Paxos或者Raft，是因为Chain Replication更加的高效，因为它减轻了Leader的负担，这或许是一个非常关键的问题。某些场合可能更适合用Raft或者Paxos，因为它们不用等待一个慢的副本。而**当有一个慢的副本时，Chain Replication会有性能的问题，因为每一个写请求需要经过每一个副本，只要有一个副本变慢了，就会使得所有的写请求处理变慢**。这个可能非常严重，比如说你有1000个服务器，因为某人正在安装软件或者其他的原因，任意时间都有几个服务器响应比较慢。每个写请求都受限于当前最慢的服务器，这个影响还是挺大的。**然而对于Raft，如果有一个副本响应速度较慢，Leader只需要等待过半服务器，而不用等待所有的副本。最终，所有的副本都能追上Leader的进度。所以，Raft在抵御短暂的慢响应方面表现的更好**。一些基于Paxos的系统，也比较擅长处理副本相距较远的情况。对于Raft和Paxos，你只需要过半服务器确认，所以不用等待一个远距离数据中心的副本确认你的操作。这些原因也使得人们倾向于使用类似于Raft和Paxos这样的选举系统，而不是Chain Replication。这里的选择取决于系统的负担和系统要实现的目标。不管怎样，配合一个外部的权威机构这种架构，我不确定是不是万能的，但的确是非常的通用。
>
> **学生提问：如果Configuration Manger认为两个服务器都活着，但是两个服务器之间的网络实际中断了会怎样？**
>
> Robert教授：对于没有网络故障的环境，总是可以假设计算机可以通过网络互通。对于出现网络故障的环境，可能是某人踢到了网线，一些路由器被错误配置了或者任何疯狂的事情都可能发生。所以，因为错误的配置你可能陷入到这样一个情况中，Chain Replication中的部分节点可以与Configuration Manager通信，并且Configuration Manager认为它们是活着的，但是它们彼此之间不能互相通信。**这是这种架构所不能处理的情况。如果你希望你的系统能抵御这样的故障。你的Configuration Manager需要更加小心的设计，它需要选出不仅是它能通信的服务器，同时这些服务器之间也能相互通信。在实际中，任意两个节点都有可能网络不通**。

 前面提到构造复制状态机，有两种方式：

1. 传统的只通过Raft/Paxos/ZAB等共识算法实现
2. 通过配置服务(Raft/Paxos/ZAB)+主备复制实现(链式复制chain replicaiton)

 可以看到这里方案2更吸引人：

- 能够获取可拓展的读性能（链式复制配合配置服务split几个chain，可以通过shard均匀读请求，提高读吞吐）；
- 由于配置服务和主备系统独立，所以在大数据量下可以有更优秀的同步机制/主备复制方案。

因此，人们更偏向于在实际应用中使用方案2。当然也不排除有使用方案1实现复制状态机的，比如Spanner就是采用Paxos完成（方案1）。

------

问题：你前面提到Raft使用时，所有的read需要通过majority的同意，我不懂为什么？因为leader不是有所有的committed entries吗？

回答：你问的是另一种策略实现。最原始的，如果你的read都只经过leader处理，并且保证leader总有最新数据，那么leader可以直接处理read。而另一种策略，你希望follower能处理read，那它必须通过majority来确认自己是否拥有最新的数据，否则应该等待同步到最新的数据后，才响应client的read请求，以确保线性一致性。

**问题：add replica增加tail时，S3什么时候能成为新的tail？**

**回答：论文中有提到一些切换tail的细节，大概可以这么说，S2可以用snapshot快照之类的手段先将目前所有的同步信息都传递给S3，也许S2同步snapshot给S3时，log到100，当S3同步完时，S2又接收了101～110。此时S3可以告诉S2，剩下的101～110你同步给我后，不要当tail了。之后S3通过配置服务告知client自己是新tail，同时告知client自己还有101～110要处理，会等处理完同步信息后再响应client（所以这里会有短暂延迟/阻塞）。**

问题：有没有可能，这里扩展可以采用别的数据结构，例如tree，而不是链表？

回答：读取tree叶子节点，听着可能破坏线性一致性。因为client请求的叶子节点，很可能还没有同步到最新write后的数据。也许你想的方案比我的想象要复杂，树的深度或者链的深度，是由失败时间决定的。如果你通常运行3～5台服务器就满足高可用性的要求了，那你可以比如只选用4台，可能崩溃恢复和日常write会引入一些延迟（这里我没怎么懂后面回答的啥东西，感觉大意就是如果构成系统的机器数本身很少，那链表的深度和树的深度差距不会太大，换言之就是tree结构的性能基于深度可能比链表小，但是这里机器数量本身不多的话，树和链表的读写性能差别也不大）。

问题：我好奇split拆分多个链后怎么保持强一致性？这里S1、S2、S3都可以作为tail被进行读取。

回答：你可以获取不同的分片shard或对象object以获取强一致性，这是分配给chain的。因为分片了，所以你写/读某个对象，它一定是对应具体某一个chain，所以能保证chain的强一致性。

追问：拆分chain后，在所有的对象上，我们都有强一致性吗？

回答：不，我认为这可能需要更多的机制。比如client读取不同shard的对象1和对象2，你需要额外的机制保证整体顺序和线性一致性。（虽然单个shard对应的chain读写能够保证线性一致性，但整体的需要额外机制。这里不同chain之间对于不同的object并不能严格保证执行的顺序。）

问题：当CR(Chian Replication)发生网络分区而不是崩溃，那会发生什么？比如S2可能还存活，但是和配置服务器或者其他什么的存在网络分区。

回答：假设所有配置都有编号，此时如果编号不匹配，那么S3不会接受来自S2的命令。

**问题：有没有可能S3崩溃后，存在Client以为S3还是tail，仍然读取S3？**

**回答：所以论文中提到Client要使用代理Proxy。（即Client可以先通过Proxy访问配置服务，得知最新的tail是什么，然后再请求具体的tial。就算tail崩溃了，下次重试时，Proxy也知道该让Client请求新的tail）**

问题：重复上面全局所有对象的线性一致性的问题，是不是需要诸如分布式事务一类的机制保证？因为我们希望的是数据a和数据b两个不同shard的数据读写具有线性一致性。（毕竟涉及多个不同对象的操作，我也感觉就是需要分布式事务机制）

回答：是的，就是需要分布式事务，这个大话题后面会说。

问题：前面zlock提到和go锁不一样，zlock在fail的时候会被看到中间状态是什么意思？

回答：因为fail后，zookeeper因为session断开了，会删除EPHEMERAL锁文件，这会导致临界区数据可能被处理/加工到一半就不再被保护（这些就是中间状态），因为zookeeper操作的都是被持久化的文件，所以这些中间状态是可见的。但是你可以通过leader选举后，增加清理这些中间状态的步骤（因为这里说zlock通常可以用于leader election）。而如果获取go锁的goroutine发生fail(崩溃)，表示整个go进程都崩溃了，所以当然没有办法读到中间状态。