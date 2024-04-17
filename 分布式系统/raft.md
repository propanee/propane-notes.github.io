# Fault tolerance-Raft

[Raft 协议原理详解，10 分钟带你掌握！ - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/488916891) <= 图片很多，推荐阅读

[Raft 协议 - 简书 (jianshu.com)](https://www.jianshu.com/p/c9024d05887f) <= 有动图，还不错

https://thesecretlivesofdata.com/raft/

## Raft历史发展

 在1980s～1990s，基本不存在诸如majority的协议，所以一直存在单点故障的问题。

而在1990s出现了两种被广泛讨论协议，但是由于当时的应用没有自动化容错的需求，所以基本没有被应用。但近15年来(2000s~2020s)大量商用产品使用到这些协议：

- Paxos
- View-Stamped replication (也被称为VR)

Raft，大概在2014左右有相关的论文，它应用广泛，可以用它来实现一个完全复制状态机(complete replicated state machine)。

## 单点故障(single point of failure)

 前面介绍过的复制系统，都存在单点故障问题(single point of failure)。

- mapreduce中的cordinator
- GFS的master
- VM-FT的test-and-set存储服务器storage

 而上诉的方案中，采用单机管理而不是采用多实例/多机器的原因，是为了避免**脑裂(split-brain)**问题。

 不过大多数情况下，单点故障是可以接受的，因为单机故障率显著比多机出现一台故障的概率低，并且重启单机以恢复工作的成本也相对较低，只需要容忍一小段时间的重启恢复工作。

## 脑裂(split-brain)

 **为什么单机管理可以避免严重的脑裂问题，尽管单机管理会有单点故障问题**。

 以VM-FT的test-and-set存储服务器storage举例。假设我们复制storage服务器，使其有两个实例机器S1、S2。（即打破单机管理的场景，看看简单的多机管理下有什么问题）

 此时C1想要争取称为Primary，于是向S1和S2都发起test-and-set请求。假设因为某种原因，S2没有响应，S1响应，成功将0改成1，此时C1可能直接认为自己成为Primary。

 这里S2没有响应可以先简单分析两种可能：

1. S2失败/宕机了，对所有请求方都无法提供服务

   如果是这种情况，那么C1成为Primary不会有任何问题，S2就如同从来不曾存在，任何其他C同样只能访问到S1，他们都会知道自己无法成为Primary，并且整个系统只会存在C1作为Primary。

2. S2和C1之间产生了网络分区(network partition)，仅C1无法访问到S2

   这时候如果存在C2也向S1和S2发出请求，此时S1虽然将0改成1会失败，但是S2会成功。如果我们的协议不够严谨，这里C2会认为自己成为了Primary，导致整个系统存在两个Primary。这也就是严重的脑裂问题。

## 大多数原则

诸如Raft一类的协议用于解决**单点故障**问题，同时也用于解决**网络分区（Network partition）**问题。这类解决方案的基本思想即：**大多数原则(majority rule)**，简单理解就是少数服从多数。

>  同样以VM-FT的test-and-set存储服务器storage举例，这次storage不是复制出2个实例，而是总共复制出3个实例，S1、S2、S3。
>
>  此时C1同时向S1、S2、S3请求test-and-set，其中S1和S2成功将0改成1，S3因为其他问题没有响应，但是我们不关系为什么。这里按照majority rule，只要3个S中有2个给出成功响应，我们就认为C1能够成为Primary。此时就算同时有C2向S1、S2、S3发起请求，就算S3成功了，C2根据majorty rule，3台只成功1台，不能成为Primary。

majority rule可以防止脑裂：因为能确保多个C之间的请求有**重叠部分(overlap)**，这样根据majority rule，多个C最后也能达成一致，因为只有1个C有可能成为多数的一方，其他C只能成为少数的一方。

- 在majority rule下，尽管发生网络分区，只会一个拥有多数的分区，不会有其他分区具有多数，只有**拥有多数的分区能继续工作**（比如这里3台被拆成1台、2台的分区，只有和后者成功通信的能继续工作）。
- 而如果极端情况下所有分区都不占多数（ 比如这里3台被拆成1台、1台、1台的分区），那么整个系统都不能运行。

### 可容忍宕机数

上述3台的场景，只能容忍1台宕机，如果宕机2台，那么任何人都无法达到majorty的情况。**这里通过2f+1拓展可容忍宕机的机器数，f表示可容忍宕机的机器数量**。**2f+1，即使宕机了f台，剩下的f+1>f，仍然可以组成majority完成服务**。

> 例如，当f=2时，表示系统内最多可容忍2台机器处于宕机状态，那么至少需要部署5台服务器（2x2+1=5）。

- 这里的majority指的是所有服务器中的大多数（不管状态如何，包括开机和停机的）。如果5台中宕机3台，那么没有人能获取到至少3台的赞成，即无法得到majority。需要超过一半，如6台需要4台赞成。

-----

> 问题：这里大多数是否包括参与者服务器本身，假设是raft，服务器会考虑自身吗？
>
> 回答：是的，通常我们看到的raft，candidate会直接vote自己。而leader处理工作时，也会vote记录自己。

## RSM with raft

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/1004%20rms.png" alt="image-20231004163922761" style="zoom:50%;" />

RSM(复制状态机replicated state machine) 。这里raft就像一个library应用包。

> Raft将状态的一致性转化为日志的一致性。

假设我们通过raft协议构造了一个由3台机器组成的K/V存储系统。

### 系统正常流程

- Client向作为leader的K/V服务器发put/get请求；
- leader K/V服务器将接收到的请求记录到底层raft的顺序log中；（上层service调用raft的start）
- 当前leader的raft将log尾部新增的记录通过网络同步到其他2台机器；
- 其他K/V的raft成功追加记录到自己的顺序log（保存在storage）中后，回应leaderACK；
- 当复制了足够多的机器，这个任务就被raft提交（commit）。leader的raft将提交的操作按顺序传递给自己的K/V应用（通过applyChannel）；
- K/V应用实际进行K/V查询，leader单独将结果响应给Client。

### 系统异常处理

- Client向leader请求；
- leader向其他2台机器同步log并且获得ACK；
- leader准备响应时突然宕机，无法响应Client；
- 其他2台机器重新选举出其中1台作为新的leader；
- Client请求超时或失败，重新发起请求，**系统内部failover故障转移**，所以这次Client请求到的是新leader；
- 新leader同样记录log并且同步log到另一台机器获取到ACK；
- 新leader响应Client。

 这里可以想到的是，剩下存活的两台机器的log中会有重复请求，而我们需要能够检测(detect)出这些重复请求。

------

> 问题：访问leader的client数量通常是多少？
>
> 回答：我想你的疑问是系统只有1个leader的话，那能承受多少请求量。实际上，具体的系统设计还会采用shard将数据分片到多个raft实例上，每个shard可以再有各自的leader，这样就可以平均请求的负载到其他机器上了。
>
> 问题：旧leader宕机后，client怎么知道要和新leader通信？
>
> 回答：client中有系统中所有服务器的访问列表。客户端随机尝试，如果不是leader，会将客户端重定向到实际的leader。

## Overview

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/image-20231003202125351.png" alt="image-20231003202125351" style="zoom:33%;" />

假设3台Raft机器，其中一台为leader，另外两台为follower。leader和follower都在storage维护log，所有K/V的put/get操作都会被记录到log中。

- Client请求leader put/get；
- leader将操作记录到自己的log尾部；
- leader将新的log条目发送给其他2个follower；
- 此时follower1成功将log条目追加到自己本地的log中，回应leader一个ack；
- 此时leader和follower1共2台机器成功追加log，达到majority，于是leader可以进行commit，将操作移交给上层的kv服务；
  - 这里即使宕机了一台，之后重新选举，包含最后操作的服务器将当选成为新的leader，比如原leader或follower1将当选，所以服务能继续正常提供；

- follower2在之后才回应leader一个ack。从follower角度其并不知道leader已经commit了操作，它只知道leader向自己同步了log并且需要回应ack；
- 如果有新的client请求发给leader，leader会追加另一个条目，并向follower发送一个新的appendEntry RPC，这会做两件事：
  - 这会为新操作提供新的日志条目，
  - 同时也确认之前的所有操作，告诉follower目前提交了哪些操作，这时候followers会传递给它们的K/V实例。


------

> 问题：如果log从leader同步到其他follower时，leader宕机了，会怎么样？
>
> 回答：会重新发生选举，而拥有最新操作log的机器成为新leader后会将追加的log条目传递给其他follower，这样就保证这些机器都拥有最新的log了。

![image-20231110105147628](https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/1110%20raft%20RPCs.png)

## Log

### Why logs？

k/v server也有一个数据库，表里有所有的信息，为什么要记录这个信息两次：log和kv table?

- 重传(retranmission)：leader向follower同步消息时，消息可能传递失败，所以要保留所有log记录，方便重传
- **顺序(order)：主要原因，我们需要被同步的操作，以相同的顺序出现在所有的replica上**
- 持久化(persistence)：持久化log数据，才能支持失败重传，重启后日志恢复等机制
- **试探性操作(space tentative)**：比如follower接收到来自leader的log后，并不知道哪些操作被commit了，可能等待一会直到明确操作被commit了才进行后续操作。我们需要一些空间来做这些试探性操作(tentative operations)，而log很合适。

**尽管中间有些时间点，可能有些机器的log是落后的。但是当一段时间没有新log产生时，最终这些机器的log会同步到完全相同的状态(logs identical on all servers)**。并且因为这些log是有顺序的，意味着上层的kv服务最终也会达到相同的状态。

### Log entry

 在log中有很多个log entry。每个log entry被log index、leader term唯一标识。

 每个log entry含有以下信息：

- command（接受到的指令、操作）
- leader term（当前系统中leader的任期期限）

 从上面log entry的格式，可以引出两点信息：

- elect leader：我们需要选举出特定任期的leader
- ensure logs become indentical：确保最后所有机器的log能达到一致的状态

------

> 问题：没有提交(uncommit)的日志条目(log entries)会被覆盖吗？
>
> 回答：是的，可能被覆盖，详情后续会谈。
>

## Leader Election

### 基本流程

**本质上是用随机的计时器进行leader选举。**

倒计时，变为condidate，term++，进行投票：

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/1004%20raft-leader%20election.gif" alt="img" style="zoom:43%;" />    <img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/raft-leader%20election2.gif" alt="img" style="zoom:50%;" />

获得多数投票，变为leader，周期性发送heartbeat；leader故障，新一轮投票：

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/1004%20raft%20leader%20election3.gif" alt="img" style="zoom:50%;" />    <img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/1004%20raft-leader%20election4.gif" alt="img" style="zoom:50%;" />

 假设这里有3台机器，1台leader，2台follower，它们都处于term10。

- 某一时刻，leader fail或和其他2follower产生了网络分区；

- 2个follower重新进行选举，原因是它们错过了leader的**心跳通信(heartbeats)**，而follower维护的election timer超时了，表明需要重新选举；
- 当leader没有收到来自client的新消息时，leader也仍会定期发送heartbeat到其他所有followers，告知自己仍是leader；
  - **heartbeat**是正常的appendEntry形式，但是没有新的日志条目。在heartbeat中，leader会携带一些自己log状态的信息，比如当前log应该多长，最后一条log entry应该是什么样的，帮助其他followers同步到最新状态；
  - Follower 节点在收到 Heartbeat 会重置自身的倒计时时间。这个 Term 会一直持续到某个节点收不到 Heartbeat 变成 Candidate 才会结束。
  
- 假设follower1的election timer超时更早，follwer1变为**Condidate**，其将term自增，从term10变成term11，并且发出选举，先投自己一票，然后向原leader和follower2请求选票；

- 由于在这个 Term 内其他节点都未投票，所以会投票给请求的 Candidate。follower2响应follwer1，投出赞成票，原leader由于网络分区问题没有响应；

- follower1成为新的leader；

- client将请求failover故障转移到新的leader上，即follower1，后续请求发送至follower1。

> **如果leader在follower1成为新leader时，仍然有client请求到leader上，并且leader又重新恢复了和原follower1、原follower2的网络连接了，会不会导致系统出现两个leader，即脑裂问题？**
>
>  **实际上不会出现脑裂(split-brain)问题**。如下分析：
>
> - 原leader接收到client的请求，尝试发log给原follwer1和原follower2；
> - 原follower1（新leader）收到log后，会拒绝追加log到本地log中，并回复原leader新的term11；
> - 原leader接收到term11后，对比自己的term10会意识到自己不再是leader，接下去要么直接成为follower，要么重新发起选举，但不会继续作为leader提供服务，不导致脑裂；
> - client此时可能得到失败或拒绝服务等响应，然后重新向新leader(原follower1)发起请求。

### 分裂选举问题(split vote)

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/1004raft-split%20vote.gif" alt="img" style="zoom:50%;" />

 在Raft中规定每个term只能投票一次，而前面的原leader网络分区后，follower1和follower2可能产生分裂选举(split vote)的问题：

- 由于巧合，follower1和follower2的election timer几乎同一时刻到期，它们都将term由10改成11
- follower1和follower2在term11都vote自己，然后向对方发起获取投票的请求
- follower1和follower2因为都在term11投票过自己了，所以都会拒绝对方的拉票请求，这导致双方都成为不了majority，没有人成为新leader
- 也许有个选举超时时间节点，follower1和follower又继续将term11增加到12，然后重复上诉流程。**如果此时没有人为介入操作，上面重复的term增加、vote自己、没人成为leader，超时后又进行新一轮election的过程可能一直重复下去**

**为了避免陷入上面的选举死循环，通常election超时时间是随机的(election timeout is randomized)**。

论文中提到，它们设置election timer时，选择150ms到300ms之间的随机值。每次这些follower重制自身的election timeout时，会选择150ms～300ms之间的一个随机数，只有当计时器到点时，才会发起选举eleciton。这样上诉流程中总有一刻follower1或者follower2因为timer领先到点，最终成功vote自己且拉票让对方vote自己，达到majority条件，优胜成为leader。

### 选举超时(election timeout)

 选举超时的时间，应该设置成大概多少才合适？

- **略大于心跳时间(>= few heartbeats)**

  如果选举超时比心跳还短，那么系统将频繁发起选举，而选举期间系统对外呈现的是阻塞请求，不能正常响应client。因为election时很可能丢失同步的log，一直频繁地更新term，不接受旧leader的log（旧leader的term低于新term，同步log消息会被拒绝）

- **加入一些随机数(random value)**

  加入适当范围的随机数，能够避免无限循环下去的分裂选举(split vote)问题。random value越大，越能够减少进行split vote的次数，但random value越大，也意味着对于client来说，整个系统停止提供对外服务的时间越长（对外表现和宕机差不多，反正选举期间无法正常响应client的请求）

- **尽量短(short enough that down time is short)**

  因为选举期间，对外client表现上如同宕机一般，无法正常响应请求，所以我们希望eleciton timeout能够尽量短

 Raft论文进行了大量实验，以得到250ms～300ms这个在它们系统中的合理值作为eleciton timeout。

### vote需记录到稳定storage

这里提一个选举中的细节问题。假设还是leader宕机，follower1和follower2中的follower1发起选举。follower1会先vote自己，然后发起拉票请求希望follower2投票自己。

**这里follower1应该用一个稳定的storage记录自己的vote历史记录，原因是避免重复vote**。假设follower1在vote自己后宕机一小段时间后恢复，我们需要避免follower1又vote自己一次，不然follower1由于vote过自己两次，直接就可以无视其他follower的投票认为自己成为了leader。

**所以，为了保证每个term，每个机器只会进行一次vote行为，以保证最后只会产生一个leader，每个参选者都需要用稳定的storage记录自己的vote行为**。

------

> 问题：这里需要记录vote之前当前机器自身是follower、leader或者candidate吗？
>
> 回答：假设机器宕机后恢复，且仍然在选举，它可以在选举结束后根据vote的结果知道自己是leader还是follower。（注意，vote情况记录在稳定的storage中，所以宕机恢复后也能继续选举流程。大不了就是选举作废，term增加又进行新一轮选举）
>

## 日志分歧(log diverge)

> [7.2 选举约束（Election Restriction） - MIT6.824 (gitbook.io)](https://mit-public-courses-cn-translatio.gitbook.io/mit6-824/lecture-07-raft2/7.2-xuan-ju-yue-shu-election-restriction)
>
> 如果你去看处理RequestVote的代码和Raft论文的图2，当某个节点为候选人投票时，节点应该将候选人的任期号记录在持久化存储中。（换言之，就算当前server的term记录落后于其他server，也可以通过通信知道下一次选举term值应该是多少，比如S1的term为5，但是S2的term为7，S1下次选举时也知道要从term8开始，而不是term6）

 这里讨论一下不同raft server中log出现分歧的问题。

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/1003%20log%20diverge2.png" alt="image-20231003223147562" style="zoom:50%;" />

 一个简单容易想到的场景即S1～S3都在log index 10～11有一致的log记录，而S1作为leader率先在log index12的位置写入log后宕机，之后S2、S3拿不到这个log记录。这个场景比较简单。

 一个复杂的场景如下：

- S1在log index10位置有term3的记录，log index11～12暂无记录
- S2在log index10、11位置有term3的记录，log index12有term4记录
- S3在log index10、11有term3记录，log index12有term5记录

**在raft协议下，有可能出现S2和S3在相同log index12下出现不同term记录的情况吗？**

- S2作为leader向S1和S3同步term3的log记录，于是S1～S3都在index10有term3记录；
- S1某一时刻宕机了，于是没有拿到后续log index11～12的记录；
- S2发起选举，term从3改成4，并且成功成为term4的leader；
- S2接收到新的请求，记录到log中，所以log index12出现term4记录；
- S2同步term4 log到S3之前，宕机；
- S3重新发起选举，此时也许S2恢复了，S3随后当选成为term5的leader（因为S2恢复，S3获取2票达到majority条件）；
- client向S3请求，S3记录term5的log，于是有log index12出现term5记录。

------

![img](https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/v2-473c0c978279aadd20c01654426aad8a_1440w.webp)

哪些index的log会被rejected/depends/accepted？

- term2 rejected。因为f不会当选，且由于没有人在index4上有term2的条目，所以f的index4会被term4覆盖；
- term7 depends。d当选时，会强制它的日志到所有人，7被接受；当d停机，c成为leader时，当d恢复时它的条目将被覆盖。（如果日志的term相同，会选择最长的日志）

------

> 问题：上面a～f争抢leader的场景中，为什么d有term7记录，它原本是leader吗？
>
> 回答：d至少是term7的leader，否则它不会有term7的log记录。
>
> 问题：前面小节提到leader同步log给其他follower后，会进行commit，这里会不会出现leader自己先commit了，结果崩溃了，其他follower还没有commit？
>
> 回答：不会。因为leader会等待确认了其他follower能commit了，再执行commit，然后才会告知上层的KV服务对应的client请求消息。
>
> **追问：这里有个问题，leader不发送其他消息的话，仅发送同步log消息给其他followers，那又怎么知道其他follower能够commit了？因为leader必须等大多数follower能够commit后才会执行自己的commit。**
>
> **回答：我们有说过可以执行一些试探性的操作，这里可以有tentative的log传输，来确认其他follower是否能够commit。（我个人猜测，log同步传输后，follower接收到log并写入storage回应ACK。之后leader准备commit之前，又发送tentative试探性消息，而大多数follower收到消息，先将tentative记录到log中表示自己随时可以commit，然后响应ACK。此时大多数follower回应leader可以commit了，尽管follower可能还没有完成commit操作，但是leader可以进行commit了。因为此时尽管followers中出现宕机，其恢复时通过log也会知道自己在commit状态中，需要把收到的log进行commit。）**
>
> 问题：如果leader追加了很多log，然后崩溃了，那么后续选举出来的新leader，会把旧leader同步过来的log进行commit吗？
>
> 回答：可能会，也可能不会。这里场景比较复杂，就算新leader没有做commit，即不响应client也没事。因为KV服务的client会认为服务失败（毕竟election阶段无法对外提供服务），client会重新发起请求，所以新leader会插入一条新的log entry，之后再执行log同步、commit的完整流程。
>
> 问题：raft和其他课前提到的算法，有什么可以优化的点吗？比如使用批处理之类的，看起来raft很适合使用批处理，因为leader可以一次再log中放入不止一个log entry。或者说raft在性能角度有什么劣势吗？
>
> 回答：首先，raft并没有采用批处理来一次插入积攒的多个log entry，也许是因为这会让协议变得更复杂。也许像你所说，在插入log entry的地方增加批处理能提高性能，但是Raft没这么做，仅此而已。可以认为Raft有很多可以提高性能的潜在手段，但是Raft没有实现它们。
>
> 问题：看上去如果出现log丢失，可能client就无法得到响应，所以Raft是不是只适用于能够允许log丢失的场景或服务？
>
> 回答：是的，一般来说需要client能够支持重试机制。并且如果有重复的请求，就如前面提到的，需要Raft的上层服务维护**重复检测表(duplicate detection table)**来保证重复请求不会导致错误的业务结果。
>

## Leader election rule

> [7.2 选举约束（Election Restriction） - MIT6.824 (gitbook.io)](https://mit-public-courses-cn-translatio.gitbook.io/mit6-824/lecture-07-raft2/7.2-xuan-ju-yue-shu-election-restriction)
>
> 在Raft论文的5.4.1，Raft有一个稍微复杂的选举限制（Election Restriction）。这个限制要求，在处理别节点发来的RequestVote RPC时，需要做一些检查才能投出赞成票。这里的限制是，节点只能向满足下面条件之一的候选人投出赞成票：
>
> 1. 候选人最后一条Log条目的任期号**大于**本地最后一条Log条目的任期号；
> 2. 或者，候选人最后一条Log条目的任期号**等于**本地最后一条Log条目的任期号，且候选人的Log记录长度**大于等于**本地Log记录的长度

- **majority：大多数原则**，即至少获取整个系统内大于全部机器数量一半的选票（包括自己，且每人只能投一次票，宕机的机器也算在系统机器总数内。如果剩余机器数压根凑不到刚好大于一半的机器数，则没有人能够成功获选）
- **at-least-up-to-date：能当选的机器一定是具有最新term的机器**。condidate至少要和follower的term是一样的，最后一个log的term相同时，有最长log的获胜。

![img](https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/v2-473c0c978279aadd20c01654426aad8a_1440w.webp)

**如果原本系统中第一行的leader宕机，剩余a-f的server中谁会成为leader？**

- b、e、f不会，因为a-f都不宕机的情况下，b、e、f拥有的term都很小，不可能获取到majority条件的选票数。因为term较大的server会拒绝其他term小的server的拉票请求。
- c+d down：a会得到b,e,f的选票，成为大多数，当选lleader；
  - c+d up时，a有没有可能变成leader?
  - **editional rule：** 如果 follower（如d）有更高的term，会回应condidate（如a）它的term，condidate会停止选举，变成follower。当d的选举定时器结束时，它就会开始选举。

## Log catch up(unoptimized)
- **nextIndex**：所有raft节点都维护nextIndex用于记录下一个需要填充log entry的log index。
  - **optimistic variable**：当leader当选时，假设自身的log一定是最新的，初始化nextIndex值为当前log index值+1）

- **matchIndex**：leader为所有raft节点(包括leader自己)维护一个matchIndex，表示leader和某个follower在matchIndex之前的所有log entry都是对齐的。
  - **pessimistic**：leader当选时，初始化matchIndex值为0，认为自身log中没有一条记录和其他follower能匹配上。随着leader和其他follower同步消息时，matchIndex会慢慢增加。**leader为每个自己的follower维护matchIndex，因为平时根据majority规则，需要保证log已经同步到足够多的followers上**。

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/1011%20raft%20log%20catchup2.png" alt="image-20231011205047736" style="zoom: 25%;" />

假设上一个term是5，S2当选了本term的leader，nextIndex设为13。通过heartbeat发起log catch up，即和其他followers同步log entry，以S2与S3的通信为例：

- S2向S3发送heartbeat，此heartbeat没有log entry（因为nextIndex指向空log[]），但是携带 preterm = 5，preindex = 12；
- S3检查自己的log发现log index12为term4，回复S2rejection no，表明自己还存活，但是不能同意S2要求的append操作，因为S3自己发现自己的log落后了；
- S2得知S3落后于自己，将nextIndex从13减到12；
- S2重新发送请求到S3，携带log entry  [5]， preterm = 3，preIndex = 11；
- S3检查自己log index11的位置为3，发现和S2说的一样，于是按照S2的log记录，在自己log index12的位置将term4改成term5，然后回复S2一条确定消息ok；
- S2收到来自S3的ok后，将自己维护的对应S3的matchIndex更新为13，表示S2和S3在log index13之前的log entry都是对齐的。
- 此时log entry5至少复制在两个节点（S2, S3）上了，此时按照majority原则，S2可以交付给应用程序。

**未优化的版本存在的问题，对于log远远落后的follower（如新加入或崩溃后重新回来的节点），leader需要进行很多次请求才能将其log与自己对齐**。如S1需要两次才能对齐。

## Erasing log entries

在上面的例子中，S3的log4直接被S2擦除了，但其实有一些微妙的边界条件。

> [聊聊RAFT的一个实现(4)–NOPCommand – 萌叔 (vearne.cc)](https://vearne.cc/archives/1851)

![image-20231011202629716](https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/1011raft%20fig8.png)



A time sequence showing why a leader cannot determine commitment using log entries from older terms. 

- In (a) S1 is leader and partially replicates the log entry at index 2. （此时log entry2没有被提交，因为还不是大多数）
- In (b) S1 crashes; S5 is elected leader for term 3 with votes from S3, S4, and itself, and accepts a different entry at log index 2. 

- In (c) S5 crashes; S1 restarts, is elected leader, and continues replication. At this point, the log entry from term 2 has been replicated on a majority of the servers, but it is not committed. 
  - **提交规则** **Commit after the leader has committed one entry in its own term**.因此此时不允许将2提交到服务器，因为这来自前一个term，而不是当前的term；

- If S1 crashes as in (d), S5 could be elected leader (with votes from S2, S3, and S4) and overwrite the entry with its own entry from term 3. 
  - 根据提交规则，尽管2是大多数，但此时还是被擦除了；

- However, if S1 replicates an entry from its current term on a majority of the servers before crashing, as in (e), then this entry is committed (S5 cannot win an election). At this point all preceding entries in the log are committed as well.  
  - 根据提交规则，在leader自己的term中有4为大多数，此时前面的log才可以被提交给应用；
- 同时，只能擦除未被提交的规则（日志可以回退，但是状态机不能够回退）

## Log catch up quickly

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/1011%20raft-catch%20up%20quickly.png" alt="image-20231011211431381" style="zoom:25%;" />

|      | 1    | 2    | 3    | 4    | 5    |
| ---- | ---- | ---- | ---- | ---- | ---- |
| S1   | 4    | 5    | 5    | 5    | 5    |
| S2   | 4    | 6    | 6    | 6    | 6    |

> 如果通过Log catch up(unoptimized)，S2要向S1同步历史log的记录，需要从log index5（nextIndex=6）开始请求，一直请求到log index1（nextIndex=2）的位置后，才能找到S2和S1对齐的第一个位置log index1，然后又以1log index为单位，一直同步到nextIndex=6为止。这显然很浪费网络资源。
>

 这里Log catch up quickly在论文中没有很详细的描述，但是大致流程如下：

- 假设S2在term7当选leader，于是nextIndex=6，如之前一样，向S1发送heartbeat时携带log同步信息，preTerm = 6，preIndex = 5；
- S1对比自己logIndex5位置为term5，因此返回rejection，并附带冲突信息：
  - 请求中preIndex位置的term值（conflicted term，此处为5）;
  - 本地log中该term最早出现的位置（conflicted index，此处为2）;

- S2收到回应后，直接将nextIndex改成2，并且下次appendEntry包括从logIndex2开始的所有内容：[6,6,6,6]，preTerm = 4, preIndex = 1；
- S1收到heartnbeat后，发现logIndex1是term4是对齐的，于是按照S2说的，将logIndex2开始往后的共4个位置更新成最新的[6,6,6,6]。

但可能发送的数据量很大，**快照**可以减少必须发送的日志条目。

## 持久化(Persistence)

 重启(Reboot)时发生/需要做的事情。

- 策略1：Raft节点故障重启后，重新加入Raft集群。即重启和新加入Raft节点没有太大区别
  - re-join(重新加入raft集群) => replay log(重放日志)；
  - 如果系统已经运行了很久，就需要重放大量的日志；
- 策略2：快速重启(start from your persistence state)，使用存储的持久化状态重建，后续可以通过log catch up等机制，赶上leader的log状态。

 策略2更好，这就要搞清楚，需要持久化哪些状态。

------

 Raft持久化以下状态state：

- vote for：投票情况，因为需要保证每轮term每个server只能投票一次
- log：故障前的log记录。如果丢失的话，可能有些条目就不构成大多数了，这些条目不会交付给其他的副本到服务器，**promise the leader to commit，保证已发生的commit不会被回退**。否则崩溃重启后，可能发生一些奇怪的事情，比如client先前的请求又重新生效一次，导致某个K/V被覆盖成旧值之类的。
- current term：故障时的当前term值。因为选举(election)需要用到，用于投票和拉票流程检测是否有过时的leader或follower的RPC，并且**需要保证单调递增**(monotonic increasing)

> [7.4 持久化（Persistence） - MIT6.824 (gitbook.io)](https://mit-public-courses-cn-translatio.gitbook.io/mit6-824/lecture-07-raft2/7.4-chi-jiu-hua-persistent)
>
> 持久化currentTerm的原因要更微妙一些，但是实际上还是为了实现一个任期内最多只有一个Leader，我们之前实际上介绍过这里的内容。如果（重启之后）我们不知道任期号是什么，很难确保一个任期内只有一个Leader。
>
> 在这里例子中，S1关机了，S2和S3会尝试选举一个新的Leader。它们需要证据证明，正确的任期号是8，而不是6。如果仅仅是S2和S3为彼此投票，它们不知道当前的任期号，它们只能查看自己的Log，它们或许会认为下一个任期是6（因为Log里的上一个任期是5）。如果它们这么做了，那么它们会从任期6开始添加Log。但是接下来，就会有问题了，因为我们有了两个不同的任期6（另一个在S1中）。这就是为什么currentTerm需要被持久化存储的原因，因为它需要用来保存已经被使用过的任期号。

------

> 问题：什么时候，server决定进行持久化的动作呢？
>
> 回答：每当上面提到的需要持久化的变量state发生变化时，都应该进行持久化，写入稳定存储(磁盘)，即使这可能是很昂贵的操作。你必须保证在回复client或者leader的请求之前，先将需要持久化的数据写入稳定存储，然后再回复。否则如果先回复，但是持久化之前崩溃了，你相当于丢失了一些无法找回的记录。
>

## 服务恢复(Service recovery)

 类似的，服务重启恢复时有两种策略：

1. 日志重放(replay log)：在复制状态机中，理论上将log中的记录全部重放一遍，能得到和之前一致的工作状态（recreate state）。这一般来说是很昂贵的策略，特别是工作数年的服务，从头开始执行一遍log，耗时难以估量。
2. **周期性快照(periodic snapshots)**：

   - 用过去的方式重建状态；压缩日志，cut off raft状态；=> 需要stable storage（**持久化状态**）；

   - 假设在i的位置创建了快照(**state contain all ops through i**) => 那么可以裁剪log，只保留i往后的log(**cut th log through i**)，这样就可以通过定期请求服务器快照，控制log的大小。
   - 重启时从持久化磁盘加载snapshot快照，reconstruct应用程序的状态，然后后续可以再通过log catch up或其他手段，将log同步到最新状态。（一般来说周期性的快照不会落后最新版本太多，所以恢复工作要少得多）
   - 快照是由service驱动的(driven by the service)，service隔一段时间告诉raft，它创建了一个包括到i的所有操作的快照，然后raft写入快照并将日志剪裁到i，并将所有这些信息写入磁盘。

这里可以扩展考虑一些场景，比如Raft集群中加入新的follower时，可以让leader将自己的snapshot传递给follower，帮助follower快速同步到近期的状态（这就需要新的installSnapshot RPC），尽管可能还是有些落后最新版本，但是根据后续log catch up等机制可以帮助follower随后快速跟进到最新版本log。

 使用快照时，需要注意几点：

- 需要拒绝旧版本的快照：有可能收到的snapshot比当前服务状态还老，这会导致隐式的回滚状态机；如果follower的log超出了快照的范围，必须将剩余的部分保留在日志中；
- 需要保持快照后的log数据：在加载快照时，如果有新log产生，需要保证加载快照后这些新产生的log能够能到保留。

------

> 问题：看上去好像这破坏了抽象的原则，现在上层应用需要感知Raft的动作？
>
> 回答：是的，这里需要上层应用辅助一些Raft的工作，比如告知Raft我已经拥有i之前的快照了，可以对应的删除Raft在i状态之前的log了之类的。因此raft与上层应用之间需要有更多的api。
>
> 问题：如果follower收到snapshot，这是它日志的前缀，由快照覆盖的log记录将被删除，而其余的会被保留。这种情况下，状态机是否会被覆盖？
>
> 回答：前面没有什么问题，这里主要需要关注快照如何和状态机通信。在课程实验中，状态机通过apply channel获取快照，然后做后续需要做的事情（比如根据快照，修改状态机中的变量之类的）。
>

## 使用Raft

 重新回顾一下服务使用Raft的大致流程

1. 应用程序中集成Raft相关的library包
2. 应用程序接收Client请求
3. 应用程序调用Raft的start函数/方法
4. 下层Raft进行log同步等流程
5. Raft通过apply channel向上层应用反应执行完成
6. 应用程序响应Client

- 并且前面提过，可能作为leader的Raft所在服务器宕机，所以Client必须**维护server列表**来切换请求的目标server为新的leader服务器。
- 同时，有时候请求会失败，或者Raft底层失败，导致重复请求，而需要做**重复检测**。通常可以在get、put请求上加上请求id或其他标识来区分每个请求。一般维护这些请求id的服务，被称为clerk。提供服务的应用程序通过clerk维护每个请求对应的id，以及一些集群信息。

## 线性一致性/强一致性(Linearizability/strong consistency)

> [线性一致性_百度百科 (baidu.com)](https://baike.baidu.com/item/线性一致性/22395305?fr=aladdin)
>
> **线性一致性**(Linearizability)，或称**原子一致性**或**严格一致性，**指的是程序在执行的历史中在存在可线性化点P的执行模型，这意味着一个操作将在程序的调用和返回之间的某个点P起作用。这里“起作用”的意思是被系统中并发运行的所有其他线程所感知。
>
> [线性一致性：什么是线性一致性？ - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/42239873)

> https://anishathalye.com/testing-distributed-systems-for-linearizability/
>
> In a linearizable system,**every operation appears to execute atomically and instantaneously at some point between the invocation and response**. Linearizability is a strong consistency model, so it’s relatively easy to build other systems on top of linearizable systems.

在论文中对整个系统提供的服务的正确性(correctness)称为**线性一致性(Linearizability)**，线性一致性需要保证满足一下三个条件：Like a single machine

1. **整体操作顺序一致(total order of operations)**

   即使操作并发进行，仍然可以按照total order对它们进行排序。（即可以根据读写操作的返回值，对所有读写操作整理出一个符合逻辑的整体执行顺序）

   同时，对于整个请求历史记录，必须**只存在一个序列**，不允许不同的客户端看见不同的序列，或者说不允许一个存储在系统中的数据有不同的演进过程。

2. **实时匹配(match real-time)**

   顺序和真实时间匹配，如果第一个操作在第二个操作开始前就完成，即使操作在不同的机器，那么在整体顺序中，第一个操作必须排在第二个操作之前。

   如果两个事务是真的同时发生的，两种顺序都是符合要求的；

3. **读操作总是返回最后一次写操作的结果(read return results of last write)**


证明系统具有线性一致性：查看历史或执行，是否能根据历史将它变成一个整体顺序，即使操作是并发执行的。

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/1012%20raft%20linearizability%20history1.png" alt="image-20231012203017691" style="zoom:25%;" />

将上图组合成total order: Wx1, Rx1, Wx2, Rx2，一台机器执行这个顺序就可以得到同样的结果，是线性一致的。

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/1012%20raft%20linearizability%20history2-2.png" alt="image-20231012203432451" style="zoom:25%;" />

Rx1和Rx2冲突，或者说Rx1返回了旧值（stale value），所以不是线性一致的（not linearizable execution）。

[线性一致（Linearizability）（3）请求故障](https://mit-public-courses-cn-translatio.gitbook.io/mit6-824/lecture-08-zookeeper/8.3-xian-xing-yi-zhi-linearizability3)

![assets_-MAkokVMtbC7djI1pgSw_-MCZOMvzhNQ9svfDymIJ_-MCZnR4O4w_F7Ad3rMwQ_image](https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/assets_-MAkokVMtbC7djI1pgSw_-MCZOMvzhNQ9svfDymIJ_-MCZnR4O4w_F7Ad3rMwQ_image.png)

> 服务器处理重复请求的合理方式是，服务器会根据请求的唯一号或者其他的客户端信息来保存一个表。这样服务器可以记住，哦，我之前看过这个请求，并且执行过它，我会发送一个相同的回复给它，因为我不想执行相同的请求两次。例如，假设这是一个写请求，你不会想要执行这个请求两次。所以，服务器必须要有能力能够过滤出重复的请求。第一个请求的回复可能已经被网络丢包了。所以，服务器也必须要有能力能够将之前发给第一个请求的回复，再次发给第二个重复的请求。所以，服务器记住了最初的回复，并且在客户端重发请求的时候将这个回复返回给客户端。如果服务器这么做了，那么因为服务器或者Leader之前执行第一个读请求的时候，可能看到的是X=3，那么它对于重传的请求，可能还是会返回X=3。所以，我们必须要决定，这是否是一个合法的行为。

Raft如何保证线性一致？

- 所有的请求都和leader进行交互；
- 通过log记录固定执行请求的顺序；
- 拥有最新的term的才能成为leader确保读到最新的操作；
- 请求重复检测？

------

> **问题：这里说的线性一致性，是不是就是人们说的强一致性？**
>
> **回答：是的。一般直觉就是表现上像单机，而技术文献中准确定义称为线性一致性。**
>
> 问题：人们为什么决定定义这个property？（指，线性一致性这个概念为啥会被定义出来）
>
> 回答：比如你希望多机系统对外表现如同单机一样，线性一致性就是非常直观的定义。数据库世界中有类似的术语，叫做**可串行化(serializability)\**。基本上线性一致性和可串行化的唯一区别是，**可串行化不需要实时匹配(match real-time)**。当然，人们对强一致性有不同定义，而我们这里认为线性一致性就是一种强一致性。
>

> 学生提问：**所以说线性一致不是用来描述系统的，而是用来描述系统的请求记录的？**
>
> Robert教授：这是个好问题。线性一致的定义是有关历史记录的定义，而不是系统的定义。所以我们不能说一个系统设计是线性一致的，我们只能说请求的历史记录是线性一致的。如果我们不知道系统内部是如何运作的，我们唯一能做的就是在系统运行的时候观察它，那在观察到任何输出之前，我们并不知道系统是不是线性一致的，我们可以假设它是线性一致的。之后我们看到了越来越多的请求，我们发现，哈，这些请求都满足线性一致的要求，那么我们认为，或许这个系统是线性的。如果我们发现一个请求不满足线性一致的要求，那么这个系统就不是线性一致的。所以是的，线性一致不是有关系统设计的定义，这是有关系统行为的定义。
>
> 所以，当你在设计某个东西时，它不那么适用。在设计系统的时候，没有一个方法能将系统设计成线性一致。除非在一个非常简单的系统中，你只有一个服务器，一份数据拷贝，并且没有运行多线程，没有使用多核，在这样一个非常简单的系统中，要想违反线性一致还有点难。但是在任何分布式系统中，又是非常容易违反线性一致性。

> 问题：可以稍微详细一点介绍clerk吗？
>
> 回答：clerk是一个RPC库，它可以帮助记录请求的RPC服务器列表。比如它认为server1是leader，于是Client发请求时，通过clerk会发送到server1，如果server1宕机了，也许clerk根据维护的server列表，会尝试将Client的请求发送到server2，猜测server2是leader。并且clerk会标记每次请求(get、put等)，生成请求id，可以帮助server服务检测重复的请求。
>
> 问题：论文12页中提到follower擦除旧log，但是不能回滚状态机，是吗？
>
> 回答：正如前面日志擦除所说，Raft**可以擦除未提交(uncommitted)的log**。
>
> **问题：server加载snapshot的时候，可能要加载很多，怎么保证后续还可以接收新的log**
>
> **回答：可以在加载snapshot之前，先通过COW（写时复制）的fork创建子进程，子进程加载snapshot，而父进程继续提供服务，例如获取新的log之类的。因为子进程和父进程共享同样的物理内存，所以后续总有办法使得加载完snapshot的子进程获取父进程这段时间内新增的log。**
>
> 问题：当生成snapshot且因此压缩/删除旧log后，sever维护的log index是从0开始，还是在原本的位置继续往后？
>
> 回答：从原本的位置继续往后，不会回退log index索引。
>

## 通信复杂度（与pbft对比）

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/1003raftvspbft.png" alt="img" style="zoom: 50%;" />

raft 是 $O(n)$，而 pbft 是 $O(n^2)$，这里主要考虑算法的共识过程。

对于 raft 算法，核心共识过程是日志复制这个过程，这个过程分两个阶段，一个是日志记录，一个是提交数据。两个过程都只需要领导者发送消息给跟随者节点，跟随者节点返回消息给领导者节点即可完成，跟随者节点之间是无需沟通的。所以如果集群总节点数为 n，对于日志记录阶段，通信次数为 $n-1$，对于提交数据阶段，通信次数也为 $n-1$，总通信次数为 $2n-2$，因此raft算法复杂度为$O(n)$。

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/1003pbft%E6%B5%81%E7%A8%8B.png" alt="img" style="zoom: 33%;" />

对于 pbft 算法，核心过程有三个阶段，分别是pre-prepare（预准备）阶段，prepare（准备）阶段和commit（提交）阶段。

- pre-prepare，主节点广播pre-prepare消息给其它节点即可，因此通信次数为 $n-1$；

- prepare，每个节点如果同意请求后，都需要向其它节点再parepare消息，所以总的通信次数为 $n*(n-1)$，即$ n^2-n$；

- commit 阶，每个节点如果达到prepared状态后，都需要向其它节点广播commit消息，所以总的通信次数也为 $n*(n-1)$，即 $n^2-n$。

所以总通信次数为 $(n-1)+(n^2-n)+(n^2-n)$，即$2n^2-n-1$，因此pbft算法复杂度为 $O(n^2)$ 。
