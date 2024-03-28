# 分布式事务(Distributed Transaction)
- 两阶段锁(2-phase locking, 2PL)
- 两阶段提交(2-phase commit, 2PC)

## 背景-跨机器原子操作(cross-machine atomic ops)。

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/1205%20%E5%88%86%E5%B8%83%E5%BC%8F%E4%BA%8B%E5%8A%A1%20cross%20machine%20atomic%20ops.png" alt="image-20231205205225552" style="zoom: 33%;" />

client从X账户向Y账户转账，需要保证故障和并发方面的原子性：

- failure：两种操作要么都会发生要么都不会发生；

- concurrency：另一个client检查这些账户时，需要both puts show atomically，其他事务不能观察到中间结果，如钱从X中减去但没有加到Y上；

这在分布式系统中也很常见，如do operation across shards（跨分片执行操作）；

## primitives原语

对操作进行分组，如转账中两个put组成了一个事务，然后这个事务原子地执行。



<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/image-20231205210743011.png" alt="image-20231205210743011" style="zoom:50%;" />

假设有T1和T2两个正在进行的事务，只需要注解指定的原语，就能保证一组操作被原子地执行，而无需感知系统内部实现使用的locking、recovery等机制。

- begin_x：声明一个事务的开始，客户端想要启动事务
  - add(x,-1)
  - add(y,+1)

- commit：提交事务，即指明事务何时完成。
  - commit成功被执行后，begin～commit之间的操作对于并发和故障被原子地执行。

- abort：取消事务。
  - begin和abort之间的逻辑将被撤销，即使已经做了一些操作，语义(semantic)仍应该是这些操作没有发生，即在所有abort和commit的情况下，要么全部发生，要么一个都不发生，不会有部分结果。
  - 除了人为在逻辑里使用abort，事务系统本身也可以调用abort，如两个事务之间存在死锁，则事务系统可以abort其中一个事务，让其他事务可以继续，并稍后重试中止的事务。

## 关键属性(semantics 语义)--ACID

- A，atomicity：原子性。在崩溃恢复(crash recovery)的情况下，事务在执行中途发生崩溃，事务内的所有写入要么都是可见的(visible)，即都进入存储，要么都没有。
- C，consistency：一致性。一致性通常与数据库有关，数据库具有内部变量，比如参照完整性(referential integrity)，而事务应该保持这种(内部)一致性。
- I，Isolation：隔离性。两个运行的事务，不会观察到彼此的中间结果。即事务只有内部所有逻辑执行完后，产生的影响/结果才能被其他事务观测到。
- D，durability：持久性。表示事务提交后，结果写入稳定存储，如果系统崩溃后恢复，最新提交的事务会记录在稳定的存储上，结果/影响是可见的/存在的。

 下面主要讨论A(atomicity)和I(Isolation)。

## 隔离性(Isolation)

> [12.1 分布式事务初探（Distributed Transaction） - MIT6.824 (gitbook.io)](https://mit-public-courses-cn-translatio.gitbook.io/mit6-824/lecture-12-distributed-transaction/12.1) <= 摘抄部分内容，作为补充说明
>
> 我们说可串行化是指，并行的执行一些事物得到的结果，与按照某种串行的顺序来执行这些事务，可以得到相同的结果。实际的执行过程或许会有大量的并行处理，但是这里要求得到的结果与按照某种顺序一次一个事务的串行执行结果是一样的。所以，**如果你要检查一个并发事务执行是否是可串行化的，你查看结果，并看看是否可以找到对于同一些事务，存在一次只执行一个事务的顺序，按照这个顺序执行可以生成相同的结果**。

### serializability

正确执行多个事务或并发事务，在数据库文献中经典的定义或标准，称为**可串行化(serializability)**。

- 即T1和T2事务就算并发运行，那么结果肯定是某种串行执行的顺序（serial order），即要么T1之后T2，要么T2之后T1，串行顺序必须产生与并发执行相同的结果（same outcome）。

- 如T1是在X、Y之间转账，T2是打印两个账户的结果；假设初始x=10，y=10，如果T1先执行，打印语句会是9,11，如果T2先执行，打印结果是10,10；

**可串行化和线性一致性的区别。**

- 线性一致性要求**real-time**，**即如果T2在T1结束后开始，那么T2在整体顺序中必须出现在T1之后**；
- 可串行化中无此要求，**即使T2开始时间晚于T1的结束时间，系统仍然允许对T1和T2重排序使得T2在T1之前**。所以可串行化要弱于线性一致性。

尽管如此，可串行化仍是一种很方便的编程思想，从编程角度来看总可以考虑事务以某种顺序执行，实际上禁止了很多有问题的情况。

### Problem cases

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/image-20231205215731624.png" alt="image-20231205215731624" style="zoom:50%;" />

不符合可串行化，T1的修改操作要么在T2事务之前，要么在之后，但是不能让T1在T2执行时同时对T2产生可见的影响。

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/image-20231205215822642.png" alt="image-20231205215822642" style="zoom:50%;" />

这里同样是可串行化不允许的，应该要么先执行T1然后T2，要么先T2再T1。

## 并发控制(Concurrency Control)

**并发控制(Concurrency Control)**避免串行化的Problem cases，有两种方向的方法：

- 悲观(pessimistic)：使用锁lock。当事务运行/开始时，需要锁来维护可串行化，只有确保能够串行化执行才会释放锁；
- 乐观(optimistic)：无锁。当到达提交点(commit point)时，系统确认之前的一系列操作是否符合可串行化的要求，如果符合则正常执行。**如果结果不对应于单次执行，则被abort**。这里可能会引入一些重试机制。

这里暂时不会讨论乐观的实现，后续FaRM论文实现了乐观的分布式事务系统。这里重点讨论悲观的实现。

在文献中，两种方案可以这么理解：

- 悲观：需要先获取许可，然后才能执行操作
- 乐观：只管执行操作，如果最后结果是错的(不符合可串行化)，随后道歉即可(撤销操作)

## 两阶段锁(2PL)

可串行化是数据库的黄金标准，但实际数据库提供了多种程度的隔离性，程序员可能会选择较弱的隔离性以获取更高的并发度。作为课程介绍，我们按照可串行化的标准进行隔离性实现的讨论。

### Rule

实现可串行化的常见实现方案——**两阶段锁(two-phase locking, 2PL)。**

**在两阶段锁(2PL)中，每个记录都有锁(lock per record)，其有两条需要遵循的规则**：

1. **在使用(某个记录)之前获取lock。(T acquire lock before using)**（在读或写x或y之前）
2. **一旦事务获取锁，只能在commit或abort时释放锁(T holds until commit or abort)**

在T1事务获取X的锁后，T2阻塞直到T1 commit释放X的锁后，T2事务才能继续进行

| T1                              | T2                                         |
| ------------------------------- | ------------------------------------------ |
| Lock X record                   | Lock X record (wait until T1 release lock) |
| Lock Y record                   | .... (other things)                        |
| commit (release Lock X, Lock Y) | Lock X success and do other things         |

 **2PL是对简单锁(严格锁)的改进**。

- **简单锁(simple locking)或严格锁(strict locking)，在事务开始前，你获取整个事务所需的所有锁，并持有这些锁直到commit或者abort，然后释放它们。**
- **2PL的锁更细粒度一点，不需要在事务开始前直接获取所有锁，相反的，在事务运行时动态增量的获取锁，支持某些严格锁(简单锁)不允许的并发模式**。

**一般而言，使用2PL要比使用简单锁(严格锁)有更高的并发度**。*比如事务中读取的某个变量极小概率会是true，而变量为true时，才会执行后序事务逻辑，那么这里不需要在事务开始前就加锁，可以等读到变量时再加锁。*

### Until commit

2PL的第一条很好理解，第二条似乎有点模糊，这里举例进行说明。

- 场景1（在操作完直接释放锁）


| T1                             | T2                             |
| ------------------------------ | ------------------------------ |
| Lock X, put(X), release Lock X |                                |
|                                | Lock X, get(X), release Lock X |
|                                | Lock Y, get(Y), release Lock Y |
| Lock Y, put(Y), release Lock Y |                                |

很明显这里T1和T2不符合可串行化，这和没有加锁时并没有什么太大区别。在两个锁集之间有交集时，让两个事务能够以特定方式排序是很重要的，即确保一些整体顺序。这意味着必须持有锁直到commit point，确保不会有事务的中间结果被其他事务看见。如果在commit point 之前释放锁，会让结果在中间可见，即使它后面可能被abort了。

### Dead lock 死锁

 在提交点才释放锁，很容易联想死锁的场景，如下：

| T1                  | T2                  |
| ------------------- | ------------------- |
| Lock X, put(X)      |                     |
|                     | Lock Y, get(Y)      |
|                     | wait Lock X, get(X) |
| wait Lock Y, put(Y) |                     |

T1和T2互相等待对方的Lock，进入死锁。

不过好在事务系统提供了abort操作。如果事务系统检测到死锁(deadlock)，就可以abort其中一个事务，使另一个能正常获取lock到commit。而客户端client或应用程序可以自己决定如何处理事务被abort的情况，比如重试。

**可以看出来2PL也会进入死锁的场景，但至少提供了abort操作，能够解决死锁问题，而不是永远停在死锁场景中**。

**事务系统如何检测到死锁？**人们用两种方法来检测，尽管不够可靠。

- 一种是基于**超时(timeout)**的，比如几个事务执行了许久并且似乎没有取得任何进展，则中止其中一个。
- 另一种更系统的方法是随着事务系统的进行动态构造一个**等待图(wait-for graph)**，如T1wait for lock y，则画T1到T2的箭头表示T1正在等待T2。如果发现等待图中出现环，则说明有死锁（比如上面T1等待T2的Lock Y，T2等待T1的Lock X，T1和T2成环）。

-----

> 问题：检测到死锁后，中止其中一个事务会发生什么？
>
> 回答：假设中止了T2，事务系统会安排T2none result或者其result是可见的，此时abort强制释放T2占有的Lock Y，T1可获得Lock Y完成事务，而发起T2的客户端会知道T2被系统中止，一般情况下你可以选择重新发起T2。
>
> **问题：是否可以在提交点之前释放锁？**
>
> **回答：视情况而定，如果你只使用我们目前在描述的独占锁(exclusive locking)，那么释放锁的时间点和提交点(commit point)或中止点(abort point)一致；如果你使用读写锁，锁允许区分读锁和写锁，那在某些限制的前提下可以提前释放读锁。**

## 两阶段提交(2PC)

### Rule

事务系统(transactiion system)中接收事务的机器称为**协调器(coordinator)**，协调器负责通过事务系统运行事务。

通常协调者以试探(tentative)的方式协调整个分布式事务。下面所有的操作，都需要预写日志，即(WAL, write-ahead log)，先写日志，在某个时刻数据库自己再做实际操作，直到commit才会install数据库中所有的东西。确保崩溃恢复时能重现log完成已commit的操作，以及舍弃或恢复未commit的操作。

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/1209%202PC%20abort.png" alt="image-20231209161421060" style="zoom:50%;" />

| 时序 | coordinator(协调器)                                          | A服务器拥有X记录                                             | B服务器拥有Y记录                                             |
| ---- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 1    | 新建事务(包括transaction id)，告知A执行put(X), 告知B执行put(Y) |                                                              |                                                              |
| 2    |                                                              | Lock X, put(X), log X(日志记录)。此时放入log，没有实际操作数据库。 | Lock Y, put(Y), log Y(日志记录)。此时放入log， 没有实际操作数据库。 |
| 3    | prepare。对A、B发起prepare(tid)，询问A和B是否可以执行事务    |                                                              |                                                              |
| 4    |                                                              | A查看自己的状态，发现持有X的锁，并且log了需要执行的put操作，回应prepare请求YES。 | B查看自己的状态，发现持有Y的锁，并且log了需要执行的put操作，回应prepare请求YES。 |
| 5    | commit。收到A和B的prepare响应YES后，提交事务，向A和B发起commit(tid)请求 |                                                              |                                                              |
| 6    |                                                              | A看本地log，发现tid对应的事务可以提交了，于是install日志，即执行日志中记录的put(X)操作，然后释放X的锁。 对commit请求响应OK。 | B看本地log，发现tid对应的事务可以提交了，于是install日志，即执行日志中记录的put(Y)操作，然后释放Y的锁，对commit请求响应OK。 |
| 7    | 收到A和B的OK，知道A和B都成功执行完事务了                     |                                                              |                                                              |

**协调者只在参与式事务的服务器都同意执行事务后才会commit**。**(Coordinator commit only if A and B agree)**

1. coordinator 向 B发送prepare后，B可能由于Y deadlock / log空间不足 / 账户中没有足够的钱等原因，无法执行事务，那么会对prepare回应NO；
2. coordinator 发现有人不同意执行事务，则coordinator 不能commit the TX，于是向A和B发送abort消息，要求停止执行事务。
3. A和B接收到abort请求后，会根据log里的记录，对tid事务操作进行回滚。回滚完毕后，A和B回应OK。
4. 协调者收到A和B的OK后，得知A和B都回滚tid事务成功了。

------

> 问题：有没有可能B在收到来自协调者的prepare请求后，回应Yes，但是又决定中止事务执行？
>
> 回答：不可能。如果你promise承诺要提交，那就必须准备提交，即等到commit请求来之后进行提交。一旦prepare回应了Yes后就不能再单方面中止事务了。
>
> 问题：有没有可能prepare时回应Yes，结果后面发现死锁了，又中止事务？
>
> 回答：你会在回应prepare之前发现死锁，所以不可能发生回应了prepare后才发现死锁的问题。

### Crashes

***Crash1 事务参与者在回应prepare ok后崩溃***

B已经对prepare承诺OK了，不能够abort了。所以B恢复后必须继续执行事务。

需要**稳定存储**一些状态：**已经prepared的Transaction ID**; **is holding the lock on Y**；

B恢复后需要查看自己是否是某个分布式事务的参与者，以及prepared tid，持有Y的锁。等待coordinator重试该tid的commit，后续就和没发生崩溃一样了。

显然，两阶段提交2PC是比较昂贵的，我们不仅需要发送多个回合的信息，而且事务的参与者还必须将一些操作记录到稳定存储中，写入稳定存储是昂贵的，可能需要1ms，就意味着1s只能处理1000个事务。

***Crash2：协调者在发送commit(tid)请求后崩溃***

类似事务参与者，协调者在发送commit(pid)请求之前，需要稳定存储：**commit tid**。

假设协调者崩溃前，A回应了OK，但是B回应协调者之前，协调者崩溃了。**那么B必须等待协调者恢复并宣布该事务的结果，而不能私自中止事务。**

这种情况下，B很不幸必须一直等待，**导致锁一直被占用**，这里其他事务如果要占用Y数据的锁就会失败，因为B需要等待协调者恢复后，能响应协调者时再释放锁。

大多数协议下，B会ping coordinator，并询问这个事务的结果是什么。

***Crash3：A一直没有回应协调者prepare请求***

假设A因为某些原因一直没有回应prepare，那么coordinator会以超时等机制，单方面决定abort，告知B执行abort来中止事务。后续A又和coordinator联系上时，coordinator不知道这个事务的更多信息，会告诉A，tid这个事务已经中止了。这意味着B收到abort后可以释放Y数据的锁，然后后续尝试其他涉及Y数据的事务。

------

> 问题：假设B持有Y的锁，直到把Y操作放入日志，以及最后install日志，执行实际对Y数据的操作后，然后才会释放锁是吗？
>
> 回答：是的。
>
> 问题：这里的锁是分布式的，Y数据只存在于服务器B上，或许我们不需要对Y数据加锁？
>
> 回答：A维护它所有分片的锁，B同理。而不同用户/事务可能访问同一个数据，所以需要锁。
>
> 问题：为什么B在接受到prepare请求后，需要持久化？当它恢复后，如果收到来自协调者的commit请求，它就不能假设自己已经准备好了吗？
>
> 回答：也许B在崩溃前打算abort，所以B需要记住自己做了什么。这里有个变体，即你可以总是假设后续要commit，然后做一些优化方案，这里还没有谈到对应的协议。2PC有很多变体实现。

### 加强协调者容错性

Crash2中，如果B回应prepare ok后coordinator崩溃，B不能够单方面abort，因为A可能已经收到commit了，B就会一直等待，导致Lock Y一直被占用，其他涉及到Y的事务无法获取锁，**协议会被阻塞，直到coordinator恢复。**

**通过Raft共识算法使协调者具有容错性(use Raft to make C available)**

- 运用Raft/Paxos等共识算法部署多台协调者，实现协调者同步，这样容许其中几台故障后也能继续提供服务。同样也可以使用Raft复制participants，获得更高的可用性。

## Raft类似于2PC？

leader & coordinator？ follower & participants？ 

- leader可以切换，而2PC类似于单点故障。

- **Raft基于majority原则；而2PC要求所有机器都有一致的回应。**
- Raft的所**有服务器做相同的事情**（all servers do the same thing），来实现**RSM复制状态机**；而2PC中**所有参与者操作不同的数据**（all servers operate on different data）。
- Raft用于实现**高可用性(high availability)**，而2PC用于实现**跨机器的原子操作(atomic ops across servers)**。即使某些方面Raft和2PC的实现类似，但两个协议针对的问题完全不同。

------

> **问题：2PL，看着也是关于原子操作的，但是看着好像是单台服务器的，而不是跨服务器的？然后2PC则是跨服务器的。**
>
> **回答：是的，实际上如果你在一台多核机器上实现事务系统，你必须对事务中涉及的记录加锁，而2PL在这种场景下是非常好的协议。而2PC才是真正关于分布式系统的。**
>
> **问题：2PL可能是2PC流程中的一部分吗？**
>
> **回答：我不太理解你的问题，但我认为2PL和2PC解决的是两种不同问题。也许你想说的是前面示例中A和B服务器对X和Y数据加锁用的是不是2PL两阶段锁。实际上这里可以是2PL，也可以是简单锁(严格锁)，但是对于2PC流程来说不重要，因为2PC是为了让各方reach agreement。**
>
> 问题：2PC是专门用于分片数据的吗？
>
> 回答：不完全是。最初2PC应对的场景是，比如你有不同的机构，它们需要协商同意某一些事情，比如你在旅游订单上，下单了机票，然后又下单了酒店房间预定，最后想提交整个旅游订单，这需要旅游网站和酒店网站都同意后，这个订单才能生效，即事务才能提交。实际上，这里需要互不信任的机场服务、酒店服务互相感知，互相依赖，如果一个服务宕机了，另一个服务就无法处理这个订单，所以2PC有点负面名声。而2PC最初就是为了解决这类问题，但是人们对于这个问题并不希望采用2PC来解决。**然而，在拥有数据中心的环境中，该单一机构内数据库是分片的，2PC则作为典型应用而广泛流行。**