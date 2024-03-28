# Spanner

> [spanner_百度百科 (baidu.com)](https://baike.baidu.com/item/spanner/6763355?fr=aladdin)
>
> Spanner是谷歌公司研发的、可扩展的、多版本、全球分布式、同步复制数据库。它支持外部一致性的分布式事务。本文描述了Spanner的架构、特性、不同设计决策的背后机理和一个新的时间API，这个API可以暴露时钟的不确定性。这个API及其实现，对于支持外部一致性和许多强大特性而言，是非常重要的，这些强大特性包括：非阻塞的读、不采用锁机制的只读事务、原子模式变更。

Wide-area transation：支持广域的事务，即跨越多个国家/地区的分布式事务

- read-write trasaction using 2-phase commit, 2-phase locking and Paxos：

  - 读写事务使用两阶段提交实现，协议的参与者都是Paxos组(2PL+2PC+Paxos)

- Read-only transaction：

  - 只读事务可以在任意数据中心运行，大约比读写事务快10倍（Spanner针对只读事务做优化）

  - **snapshot isolation**：使用快照隔离技术，使得只读事务能更快
  - **synchronized clocks**：实现依赖同步时钟，这些时钟基本是完全同步的，事务方案必须处理不同机器的一些时间漂移或错误容限（具体的实现称为TrueTime API）


- Wide-used：Spanner是一项云服务，你可以作为Google的客户使用。比如Gmail的部分功能可能有依赖Spanner。

# Organization

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/1209%20spanner%20organization.png" alt="image-20231209222304962" style="zoom: 33%;" />

假设有A~C这3台服务器，其中有a～m数据的shard分片在A～C都有一份复制，这些shard组成一个Paxos组。A～C还有一份n～z的数据shard分片，又组成了另一个Paxos组。

- multiple shards => parallelism：
  - Spanner对数据分片，获取更高的并行度。如果事务涉及不同的分片，且事务之间互不相交，可并行处理。

- Paxos per shard (majority)：
  - 每个shard分片都有对应的Paxos组，用于replication复制。根据majority原则，只需要大多数服务器响应，服务就能正常运行，所以速度较慢的机器可能不会对性能产生太大影响；同时也增加容错性，容许部分服务器宕机。
  - data center fault tolerance (数据中心容错能力)
  - through slowness (绕过慢机器)

- Replica close to clients：
  - Replica通常部署在靠近client用户群体（这里client主要指公司内部的后端服务器，因为Spanner整体而言是对内的基础架构服务）的地理位置。目标是能让client智能地访问最近的Replica。**通常只读事务可以由本地Replica执行，而不需要与其他数据中心进行任何通信**。


# Challenges

- Read of local replica yield lastet write：

  - **从本地Relica读取，但要求看到最新的写入**
  - 事实上，**Spanner追求的是比线性强一致性更强的性质**。（zookeeper也允许读任何节点，但是并不保证能读到其他client的最新写入，只保证读取当前session下最新的写入，即弱一致性）；

- Support transactions across shards：

  - 支持跨分片的事务，具有ACID语义(semantics)；

- read-only txns and read-write txns must be serializable：

  - **只读事务和读写事务必须串行化**；

  - 实际上，这比可串行化要强一些；

  - 对于读写事务(read-write tx)，Spanner使用2PL和2PC。


# 读写事务(read-write txns)

这里在不考虑时间戳的情况下讨论读写事务。因为时间戳在读写事务中不是很重要，时间戳主要用于只读事务（因为需要保证只读事务读到最新写数据），它们需要对读写事务进行一些调整，以支持只读事务。时间戳在某种程度上也能漂移到读写事务(drifting to read-write transaction)，但是本质上，读写事务是直接的2PL和2PC。

- 执行**read**操作时，直接访问shard所在Paxos组的**leader**；
- 执行**write**，需要经过TX **Coordinator**，利用2PC协调各leader之间的写操作；
  - 2PC的TX **Coordinator也是Paxos组**，实现高可用(highly avaliable)，增强容错性：避免Coordinator宕机，导致参与者持有锁阻塞，需等待Coordinator恢复后才能commit事务；
- shards keep **lock table**：
  - 分片维护锁表，实际上owned by the client，记录如Lock X被客户端C持有，Lock Y被客户端C持有；
  - **read**下lock table**仅存储在leader中**，而不在Paxos组内复制，加快了读操作；若leader在事务过程中宕机，则事务将被abort，必须重新开始，因为锁信息丢失了；
  - 对于**write**，会将2PC必要的锁状态在**prepare阶段**写入Paxos组中的不同节点;
- **Participants Prepared State**：
  - 当参与者处于Prepared State，Paxos根据WAL(Write-ahead logging)将操作记录到log，并记录**事务状态、2PC状态、以及参与者持有的锁lock**等，leader将其写入到组中的不同节点，确保状态被复制，具有容错能力；
  - prepare阶段前崩溃，锁信息会丢失（因为还只在分片Paxos组的leader的锁表记录中）；prepare阶段后崩溃，某些记录被加锁的状态会通过log同步到Paxos组其他成员，锁记录不会丢失。

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/image-20231213150825573.png" alt="image-20231213150825573" style="zoom: 50%;" />

| Client                                                       | 协调者(Paxos组)                                       | $S_A$ w. lock table                                          | $S_B$ w. lock table                                          |
| ------------------------------------------------------------ | ----------------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 分别向$S_A$、$S_B$所在Paxos组的leader发起读请求 `read X`和 `read Y`(cross shard transaction with *Tid*); |                                                       |                                                              |                                                              |
|                                                              |                                                       | 根据2PL，$S_A$leader对X加锁，Owner是Client；                 | 根据2PL，$S_B$leader对Y加锁，Owner是Client；                 |
| 对X=X-1，Y=Y+1，并向协调者submit该更新事务;                  |                                                       |                                                              |                                                              |
|                                                              | 运行2PC，分别发送X和Y的更新到$S_A$和$S_B$的leader;    | *WAL*                                                        | *Write-ahead logging*                                        |
|                                                              |                                                       | 发现已持有锁(升级为写锁)；**prepared state** 记录操作到log(WAL)，状态同步paxos，准备好后返回ok； | 发现已持有锁(升级为写锁)；**prepared state** 记录操作到log(WAL)，状态同步paxos，准备好后返回ok； |
|                                                              | commit(tid) 通知提交事务。同样要将2PC状态写入paxos组; |                                                              |                                                              |
|                                                              |                                                       | Paxos组install log，实际执行事务操作，释放locks，回应OK;     | Paxos组install log，实际执行事务操作，释放locks，回应OK;     |
|                                                              | clean up its state;                                   | 之后某个时刻分片也清理它们的状态;                            |                                                              |

**整体上看和分布式事务中的2PL+2PC流程相似，区别在于coordinator和participant都是Paxos组，通过replicate实现highly available**。

------

> **问题：每个分片是否都复制了锁表？**
>
> **回答：不是直接复制lock table，它复制的是prepare阶段分片正在持有的锁(it's replicating the lock that's holding when it does the prepare.)，即执行2PC时所需要的state。**
>
> **问题：如果当前一些事务的锁，还没有到达prepare阶段，它们会丢失吗？**
>
> **回答：它们会丢失，然后事务会中止。然后participant(paxos组leader)告知协调者，我的锁丢失了，不能继续执行事务**

# 只读事务(read-only tnxs)

***Fast***(论文提到只读事务基本上比读写事务快10倍，读写事务在数百毫秒的量级，而只读事务在5～10毫秒的量级)：

- **Read from local shards** 通过只从本地分片读取数据，无需额外和其他分片通信，实现一致性(consistency)/串行化（serializability）是个难点；
- **no locks** 读写事务不会阻塞读写事务，同样只读事务不会阻塞读写事务；

- **no 2PC** 不需要两阶段提交，同时意味着不需要广域wide-area通信；

## 正确性定义(Correctness)

只读事务的正确性(Correctness)意味着两点：

1. **事务是可串行化的(serializable)**

   - 即几个读写事务和只读事务在一起运行时，能够按照结果得到一个排序。只读事务要么读取到某个读写事务的所有写入结果，要么其写入结果都看不到，但必须保证只读事务不能看到读写事务的中间结果。

2. **外部一致性(external consistency)**

   - **如果事务T2在T1提交之后开始，那么T2必须看到T1的结果**；即如果R/O(read-only)事务在某个R/W(read-write)提交之后开始，那么这个R/O必须看到该写入；

   - 这里外部一致性可以理解为，**在可串行化基础上再加上实时性(real-time)要求**。实际上外部一致性与**线性一致性(linearizability)**非常相似。
   - 区别在于external consistency是事务级别的属性 (transaction level property)，而linearizability则是单独的读写操作；但两者都提供了强一致性(very strong consistency property);
   - 如果两个事务真的同时发生，两种顺序都是符合要求的；将取决的读取的replica中有没有包含到；

> 线性一致性(linearizability)
>
> 1. **整体操作顺序一致(total order of operations)**
>
>    即使操作并发进行，仍然可以按照**total order**对它们进行排序。（即可以根据读写操作的返回值，对所有读写操作整理出一个符合逻辑的整体执行顺序）
>
>    同时，对于整个请求历史记录，必须**只存在一个序列**，不允许不同的客户端看见不同的序列，或者说不允许一个存储在系统中的数据有不同的演进过程。
>
> 2. **实时匹配(match real-time)**
>
>    顺序和真实时间匹配，如果第一个操作在第二个操作开始前就完成，即使操作在不同的机器，那么在整体顺序中，第一个操作必须排在第二个操作之前。
>
> 3. **读操作总是返回最后一次写操作的结果(read return results of last write)**

## Bad Plan

 bad实现：always read lastet committed value，总是读最后提交的数据。

| TX   | time-->               |                  |                      |                          |
| ---- | --------------------- | ---------------- | -------------------- | ------------------------ |
| T1   | set X=1, Y =1, Commit |                  |                      |                          |
| T2   |                       |                  | Set X=2, Y=2, Commit |                          |
| T3   |                       | Read X (now X=1) | a little bit delay   | Read Y (now Y=2), Commit |

显然，如果总是读最后提交的数据，是不正确的，T3观察到了来自不同事务的写入，而不是得到一致的结果。（不符合可串行化，理论上T3要么X和Y都读取到1，要么都读取到2，要么都读取到X和Y的原始值）

## 快照隔离(Snapshot Isolation)

**Spanner采用快照隔离(Snapshot Isolation)方案实现只读事务的正确性(Correctness)**。

**快照隔离是一个标准的数据库概念，主要是针对本地数据库，而非广域(wide-area)的**，后续会对广域方面的内容再做介绍。

快照隔离具备以下特征：

- **Assign a timestamp to a transaction** 为事务分配时间戳；
  - 读写事务：在**开始提交**时分配时间戳（R/W：commit）
  - 只读事务：在**事务开始**时分配时间戳（R/O：start）
  
- **Execute all in timestamp order** 按照时间戳顺序执行所有的事务；

- **Each replica stores data with a timestamp** 每个副本保存键的多个值和对应时间戳；
- 例如在一个replica中，可以说请给我时间戳10的x的值，或者给我时间戳20的x的值。**有时这被称为多版本数据库(multi-version database)或多版本存储(multi-version storage)**。对于每次更新，保存数据项的一个版本，这样你可以回滚(so you can go back in time)。

| 事务(@时间戳) | 时间戳10                 | 时间戳15              | 时间戳20                |                      |
| ------------- | ------------------------ | --------------------- | ----------------------- | -------------------- |
| T1 (@10)      | set X=1, Y =1, Commit@10 |                       |                         |                      |
| T2 (@20)      |                          |                       | Set X=2, Y=2, Commit@20 |                      |
| T3 (@15)      |                          | Read X (X=1) start@15 |                         | Read Y (Y=1), Commit |

T3需要读取时间戳15之前的最新提交值，即T1提交的写结果，所以T3读取的X和Y都是1。因为所有的事务按照全局时间戳执行，所以保证了线性一致性或串行化。

这里需要知道的是，每个replica给数据维护了version表，可以猜想结构大致如下：

| 数据（假设X和Y在同一个replica中） | 版本（假设用时间戳表示） | 值   |
| --------------------------------- | ------------------------ | ---- |
| X                                 | @10                      | 1    |
|                                   | @20                      | 2    |
| Y                                 | @10                      | 1    |
|                                   | @20                      | 2    |

所以当读请求到达replica时，replica可以根据事务请求的时间戳，查询维护的数据记录表，决定该返回什么版本的数据。

Replica hasn't X@10：**Spanner还依赖"safe time"机制，保证读操作从本地replica读数据时，能读取到事务时间戳之前的最新commit数据。**

- Paxos也按时间戳顺序发送所有写入操作(**Paxos sends writes in timestamp order**)；
- 在读取X的时间戳@T的数据前，replica必须等待时间戳@T之后的任意数据写入(**Before Read X @15, wait for Write > @15**)，这样就知道在@T之前不会再有写入;
  - 这意味这读取必须稍微延迟一些，直到下次写入，对于忙的机器，会一直有写入，等待可能几乎不存在；
  - **同时还需要等待在2PC中已经prepare但是还没有commit的事务(Also wait for TX that have prepared but not committed)**，所以需要保证任何在读取时间戳前准备好(prepared)的事务，必须在返回读取值之前提交(commit)。


------

> 问题：X本身处于一些shard分片上，被复制到Paxos组中，当读取X时，希望只读事务特别快，而只从本地replica读取而不一定是leader，如何保证不会读到旧数据？
>
> 回答：正如你所说，可能读取的local replica正好没有比如@10版本的数据。
>
> Spanner通过**"Safe time"**方案解决这个问题。即**Paxos或Raft也按照时间戳顺序发送所有写入(Paxos sends writes in timestamp order)**，你可以考虑总顺序是一个计数，但它实际是一个时间戳，由于时间戳形成了全局顺序，时间戳的全局顺序足够对所有写入进行排序。
>
> 并且，对于读取有一个额外的规则，**在读取时间戳15的X之前，replica必须等待时间戳大于15的写入。**一旦看到时间戳15之后的写入，它就知道在时间戳15之前不会再有别的写入，所以安全地在时间戳15执行读取，并且知道该返回什么值。这对服务而言，相当于读取可能必须稍微延迟一点，直到下一次写入。当然对于写入频繁的服务器而言，这里读操作需要的等待时间很短。 真正的情况会更复杂一些，需要等待，并且要等待已准备好提交但未提交的事务。**例如某个事务可能在时间戳14已经prepare了，但是还没有commit写入到键值存储。所以我们需要保证任何在我们读取时间戳前准备好(prepared)的事务，必须在我们返回读取值之前提交(commit)。
>
> 追问：不同分片是不是也需要这样考虑，我们是否单独考虑不同的分片shard？
>
> 回答：读取只命中本地的分片shard，本地的replica。
>
> **追问：这里的正确性保证，适用于跨分片的场景吗？**
>
> **回答：是的，适用于事务级别。如果只读取本地replica，我们仍然需要确保事务的一致性。通过遵循这些规则实现该目标。**

# Clocks must be perfect

Spanner的实现和时间戳强相关，要求不同机器的时钟必须准确。不同的参与者必须就在时间戳顺序上达成一致。事务在系统中的任何地方，时间戳都必须是相同的。

**实际上，时钟/时间戳只对只读事务非常重要(matters only for R/O txns)**。因为R/W事务会抓取日志(grab logs)，并使用2PL来确保total order，所以已经在实现serializable, externel consistent order；

若某个replica或服务器时间戳不准确：

1. 时间戳偏大(TS too large) => wait longer, nothing goes wrong

   比如 read-only T3在实际TS@15执行，机器却认为在TS@18执行，而只读事务需要等待TS@18之后的write出现后才能读取数据，因此需要等待更长时间。所以时间戳偏大，只是**单纯影响性能**。

2. 时间戳偏小(TS too small) => violates external consistency

   比如只读事务T3在实际TS@15执行，机器认为T3在TS@9执行，那么T3就看不到T1@10 commit的写入，这**打破了外部一致性(external consistency)**。

------

> 问题：假设T3机器的时钟有问题，在时间戳15时，给只读事务T3分配时间戳9。但是现实时间戳为15，这时候T3只读事务，读取本地replica，本地replica是不是可能从其他机器同步到了现实时间戳10的数据？
>
> 回答：是的，因为现实时间戳是15。但是由于T3机器时钟认为时间戳是9，所以不会给T3事务返回时间戳10的数据，而是会寻找时间戳9之前的最新已提交的数据。
>

## 时钟同步(Clock Synchronization)

> [网络时间协议_百度百科 (baidu.com)](https://baike.baidu.com/item/网络时间协议/5927826?fromtitle=NTP&fromid=1100433&fr=aladdin)
>
> 网络时间协议，英文名称：Network Time Protocol（NTP）是用来使计算机[时间同步](https://baike.baidu.com/item/时间同步?fromModule=lemma_inlink)化的一种协议，它可以使[计算机](https://baike.baidu.com/item/计算机/140338?fromModule=lemma_inlink)对其[服务器](https://baike.baidu.com/item/服务器/100571?fromModule=lemma_inlink)或[时钟源](https://baike.baidu.com/item/时钟源/3219811?fromModule=lemma_inlink)（如石英钟，GPS等等)做同步化，它可以提供高精准度的时间校正（LAN上与标准间差小于1毫秒，WAN上几十毫秒），且可介由加密确认的方式来防止恶毒的[协议](https://baike.baidu.com/item/协议/670528?fromModule=lemma_inlink)攻击。NTP的目的是在无序的Internet环境中提供精确和健壮的时间服务。
>
> 术语网络时间协议（NTP）适用于运行在计算机上的和客户端/服务器程序和协议，程序由作为NTP客户端、服务器端或两者的用户编写，在基本条件下，NTP客户端发出时间请求，与时间服务器交换时间，这个交换的结果是，客户端能计算出时间的延迟，它的弥补值，并调整与服务器时间同步。通常情况下，在设置的初始，在5至10分钟有内6次交换。 一旦同步后，每10分钟与服务器时间进行一次同步，通常要求单一信息交换。冗余服务器和不同的网络路径用于保证可靠性的精确度，除了客户端/服务器商的同步以外，NTP还支持同等计算机的广播同步。NTP在设计上是高度容错和可升级的。

difficulty -> clocks naturally drift 时钟会自然漂移

1. 使用**原子钟(atomic clocks)**

   Spanner在机器上使用原子钟，以获取更精准的时间

2. **与全球时间同步(synchronize with global time)**

   Spanner让时钟和全球时间同步，确保所有机器的时钟在全球时间上一致。使用GPS全球定位系统广播时间，作为同步不同原子钟的一种方式，然后保持它们同步运行。

论文中没有太多谈论真实时间系统如何工作，但看上去他们每个数据中心可能有少量或至少一个原子钟。比如时间服务器常规地与不同数据中心的不同time master通过GPS系统进行时间同步。

但就结论而言，Spanner依靠这些手段使得位于不同服务器上的时钟非常接近，论文中提到不同机器上时钟时间误差只有几微秒到几毫秒之间的量级。所以当访问操作系统获取当前时间时，可能与实际时间只相差几微秒/毫秒。

------

> 问题：论文中没有深入讲解同步时钟，通过GPS同步不同机器的时钟时，不应该考虑信息传递的时间吗？
>
> 回答：是的，前面没有特地说明，但他们同步时间时，可能通过时间库跟踪开始时间以进行估计平均延迟或正常延迟是多少，用来纠正时间以获取尽量精确的时间。他们协议也支持离散值，比如突然网络问题导致时间戳延迟许多，则你不该接收本次的时间同步值。同时时钟的时间振荡器要是出现问题，那么后续时间也不准确了。再次声明，论文并没有详细说明怎么进行时钟同步，但他们貌似是用了类似NTP的技术来处理这里的问题。
>

## 时钟偏移解决方案TrueTime

> [6.824：Spanner详解 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/598621805) <= 推荐阅读。这里摘抄部分内容，作为补充说明，因为21版的视频这部分讲得有点模糊
>
> Start Rule：为事务分配时间戳就是返回的TT区间中的latest time来赋值，用`TT.now().latest`赋值，**这确保事务被赋值的时间戳是比真实的时间要大一些**。对于只读型事务而言，时间戳应该在开始的时候就赋予；对于读写型事务而言，时间戳应该在提交的时候再赋予。
>
> Commit Wait Rule：这个规则只针对于读写型事务，由于事务被分配的时间戳为TT区间中的latest，实际是要大于真实时间的，后续则需要等待真实时间大于这个时间戳后才能提交该读写型事务。这确保了读写型事务被提交的那个时间的数值是比被分配的时间戳要大。
>
> **服务器仅需循环调用TT.now()获取真实时间的情况，当获取的TT区间的earliest time都大于这个时间戳了，表明真实时间必然已经大于这个时间戳了。**
>
> **我们需要解决的情况是，只读型事务要是被分配了相对于真实时间较小的时间戳，会导致只读型事务读取不到最新提交的数据。而只读型事务被分配相对于真实时间较大的时间戳的情况，只是需要多等待一会后续的write，但能读取新版本的数据。因此，我们需要避免只读型事务被分配较小的时间戳。而Start Time Rule保证只读事务的版本时间戳永远比实际时间戳大一点。**
>
> [关于Spanner中的TrueTime和Linearizability - 墨天轮 (modb.pro)](https://www.modb.pro/db/138174) <= 推荐阅读，也是对TrueTime机制做补充。文章里面有具体的推算逻辑
>
> Spanner通过TrueTime机制+Commit Wait机制，确保External consistency。
>
> Spanner的TureTime API设计非常巧妙， 保证了绝对时间一定是落在TT。now()返回的区间之中。 基于这个保证， 使得分布在全球的spanserver分配的timestamp都是基于同一参考系， 具有可比性。 进而让Spanner能够感知分布式系统中的事件的先后顺序， 保证了External Consistency。
>
> 但是TureTime API要求对误差的测量具有非常高的要求， 如果实际误差 > 预计的误差， 那么绝对时间很可能就飘到了区间之外， 后续的操作都无法保证宣称的External Consistency。另外， Spanener的大部分响应时间与误差的大小是正相关的。
>
> 自Spanner在OSDI12发表论文后， Spanner一直在努力减少误差， 并提高测量误差的技术，但是并没有透露更多细节。

尽管进行了时钟同步，但仍存在一个误差范围。True Time会返回最佳估计值，即当前的绝对时间或真实时间是什么，加上机器的误差范围。

解决时钟偏移(Clock drift)的方案，即不使用真实时间的时间戳，或者不仅仅是纯粹的时间戳，而是**使用时间间隔(timestamps are intervals)**。即**每个当前时间now的返回值，不是真正时间的单独值，而是一个时间区间`[earliest, lastest]`**，是对误差范围的估计，并且可以保证真实时间在这个时间间隔内。

因此有些关于时间的规则，需要修正：

- **当前时间规则 (Start rule)：now.lastest**

  - 当前时间取时间区间的lastest值，**确保时间戳肯定在真实时间之后**（timestamp is guranteed to be after true time）；
  - 读写R/W：commit，在**开始提交**(the commit starts)时分配当前时间`now.lastest`作为时间戳；
  - 只读R/O：start，在**事务开始**(start of the transaction)时分配当前时间`now.lastest`作为时间戳；

- **提交等待规则(Commit wait rule)：delay C until TS < now.earliest**

  我们会在读写事务中推迟提交(delay the commit)。事务在commit time获得时间戳(lastest)，直到分配的时间戳早于当前的`now.earliest`，确保提交一定是在当前的true time之前，即分配的时间戳已经exactly in the past。

  这样能够确保true time排在后面的R/O txn根据Start rule不会选择在T2commit之前的时间。

付出一些delay为代价，但是如果时间十分精确，这个delay是相当小的。

| 事务(@TS) | true time 1     | true time [1...10]                                           | true time [10...12]                                          |
| --------- | --------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| T1 (@1)   | set X=1, Commit |                                                              |                                                              |
| T2 (@10)  |                 | Set X=2, 请求时间戳[1...10]并选择@10作为时间戳，并等待并确保@10 in the past(协调者keep reading本地时钟，直到earliest 超过10，[11...20]) |                                                              |
| T3 (@12)  |                 |                                                              | Read X (X=2), Commit，获取时间间隔[10...12]，选择@12作为TS，观察到T2 |

# 小结

> [为什么很多数据库自称受spanner启发，难道他们也有TrueTime API? - 知乎 (zhihu.com)](https://www.zhihu.com/question/487134503) <= 不错的讨论文章，推荐阅读

- 读写事务(R/W txn)是全局有序的(可串行化+外部一致性)，因为使用2PL+2PC;
- 只读事务(R/O txn)只读本地replica，通过快照隔离(snapshot isolation)确保读正确性;
  - 快照隔离，数据项都是版本化的(data item is versioned)，并使用modified时间戳，保证可串行化；
  - real-time component，按时间戳顺序(timestamp order)执行只读操作，保证外部一致性；
    - 时间依赖完全同步的时钟，当前时间now采用时间间隔/区间表示`[earliest, lastest]`

通过以上技术/手段，只读事务非常快，而读写事务平均100ms。读写事务慢的原因是跨国家/地区进行分布式事务。

------

> 问题：前面"14.3 Spanner-读写事务(read-write txns)"介绍中，一开始读写事务只执行读操作的时候，不需要和TC(事务协调者)通信是吗？
>
> 回答：是的，一开始的只读操作，只需要和Paxos的leader进行交互。后续的写操作执行时，才需要和事务协调者交互。
>
> 问题：论文中4.2.3部分，事务模式章节的部分，他们讨论的预测提交的时间。
>
> 回答：这部分这个课没有讲述。模式更改(schema-change)意味着向表中添加列或者删除表中的列等，改变了数据库的布局。模式更改通常代价高昂，他们确保做到原子，在未来某个时间点运行。所以这里模式更改时使用的时间戳超越了当前时间。而这时候系统中的事务还能继续，我想是因为这些事务都是用版本内存(version memory)，他们创建版本内存直到遥远的未来。所以这里模式更改不会影响任何当前在运行的事务。正式提交模式迁移部分(commit the schema migration part)时，事务还是按时间前进的，规则是任何正在运行的事务都必须停止，直到迁移事务完成(migration transaction was completed)。因为迁移事务时间戳是新值。因为有版本内存，所以可以把迁移事务推迟到后面再进行。