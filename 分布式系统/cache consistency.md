# 缓存一致性

 这篇论文发表于1997，研究的背景是网络文件系统，整体目标是在一组用户之间共享文件，但Frangipani本身并没有被大量应用。下面列举本章节的重点：
- 缓存一致性协议(cache coherence)
- 分布式锁(distributed locking)
- 分布式崩溃恢复(distributed crash recovery)

## Network File System

传统的网络文件系统(traditional or common Network File System)，file servers实现文件系统操作，对外提供open、create等文件操作函数，内部实现复杂，而client端除了函数调用和缓存以外，基本不做其他工作。

从安全的角度来说是个很好的设计，因为大多file servers是可信的，但client不一定是可信的。

## Frangipani-简述

![image-20231121221401064](https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/1121%20fragipani.png)

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/202311232111164.png" alt="image-20231123211129279" style="zoom: 33%;" />

> **Frangipani structure.** In one typical Frangipani configuration, some machines run user programs and the Frangipani file server module; others run Petal and the distributed lock service. In other configurations, the same machines may play both roles  

**Frangipani中没有真正的file server，client本身运行file server代码**。

所有的client共享一个虚拟磁盘(virtual disk)，就像共享a big SSD。这个虚拟磁盘内部使用Petal实现，由数个机器构成，机器复制磁盘块(disk blocks)，内部通过Paxos共识算法保证操作按正确顺序应用等。**虚拟磁盘对外的接口是read块(read block)或write块(write block)，看上去就像普通的磁盘**。

在Frangipani这种设计下，复杂性更多在client。你可以通过增加工作站(workstation)数量来拓展文件系统。通过增加client数量，**每个client都可以在自己的文件系统上运行，许多繁重的计算都可以在client机器上完成，相当于将文件系统拆分到不同的文件服务器中。而传统的网络文件系统的性能瓶颈往往出现在文件服务器**。

## 使用场景和设计目标

Frangipani的使用背景，一堆重度计算机使用者的研究者/学者，偶尔有需要分享的文件需要传递。因为所有参与者都是可信的，所以安全性对于这个文件系统不是很重要。

需求的文件分享形式：

- user-to-user sharing
- some users logs into >1 workstation

 Design choices：

1. caching：缓存，不是所有数据都保留在Petal(Frangipani文件服务器部分被称为Petal)中。**client采用回写式缓存(write-back cache)取代直写式(write-through)**。即操作发生在client本地的cache中，未来某时刻被传输到Petal。
2. strong consistency：强一致性。即希望client1写file1时，其他client2或者workstation能够看到file1的变动。
3. performance：高性能。

对比GFS，GFS是为mapreduce设计的，没有进行数据缓存，所以也没有提供缓存一致性。GFS也并不是一个真正的文件系统，不提供POSIX或Unix兼容性，而Frangipani可以运行Unix标准的应用程序，应用的行为方式和没有分布式系统一样，而是像单一文件系统。

------

问题：为什么说client上运行server code可以增强scalability可拓展性？而不是向传统网络文件系统一样，server code只在file server上存在，而client只负责调用file server的接口。

回答：传统的网络文件系统中，所有的计算都是针对文件系统本身，而文件系统只在文件服务器上。在Frangipani中，所有文件系统操作发生在workstation工作站上（client上运行的server code），我们可以运行多个client（工作站）来扩展工作负载。

## challenges

假设WS1工作站1执行read f的操作，之后通过本地cache对f通过vi进行更新或操作，在晚些时候将结果写回Petal。几个可能发生的场景：

1. WS2 cat f

   工作站2查看f文件，这时需要通过**缓存一致性(cache coherence / consistency)** 确保工作站2能看到正确的内容。

2. WS1创建d/f，WS2创建d/g

   Both WS1 and WS2 want to create a file in the share directory. 需要保证WS1创建d目录下的f文件，以及WS2创建d目录下的g文件时，双方不会因为创建目录导致对方的文件被覆盖或消失等问题。这里需要**原子性(atomicity)**保证操作之间不会互相影响。

3. WS1 crash during FS op

   工作站1进行复杂的文件操作时发生崩溃crash。需要**崩溃恢复(crash recovery)**机制。

## 缓存一致性(cache coherence/consistency)

> [11.4 缓存一致性（Cache Coherence） - MIT6.824 (gitbook.io)](https://mit-public-courses-cn-translatio.gitbook.io/mit6-824/lecture-11-cache-consistency-frangipani/1.4-huan-cun-yi-zhi-xie-yi-coherence-protocol) <= 摘抄一些内容，作为补充说明
>
> 每个工作站用完了锁之后，不是立即向锁服务器释放锁，而是将锁的状态标记为Idle就是一种优化。
>
> 另一个主要的优化是，Frangipani有共享的读锁（Shared Read Lock）和排他的写锁（Exclusive Write Lock）。如果有大量的工作站需要读取文件，但是没有人会修改这个文件，它们都可以同时持有对这个文件的读锁。如果某个工作站需要修改这个已经被大量工作站缓存的文件时，那么它首先需要Revoke所有工作站的读锁，这样所有的工作站都会放弃自己对于该文件的缓存，只有在那时，这个工作站才可以修改文件。因为没有人持有了这个文件的缓存，所以就算文件被修改了，也没有人会读到旧的数据。
>
> 这就是以锁为核心的缓存一致性。
>
> 学生提问：如果没有其他工作站读取文件，那缓存中的数据就永远不写入后端存储了吗？
>
> Robert教授：这是一个好问题。实际上，在我刚刚描述的机制中是有风险的，如果我在我的工作站修改了一个文件，但是没有人读取它，这时，这个文件修改后的版本的唯一拷贝只存在于我的工作站的缓存或者RAM上。这些文件里面可能有一些非常珍贵的信息，如果我的工作站崩溃了，并且我们不做任何特殊的操作，数据的唯一拷贝会丢失。所以为了阻止这种情况，不管怎么样，工作站每隔30秒会将所有修改了的缓存写回到Petal中。所以，如果我的工作站突然崩溃了，我或许会丢失过去30秒的数据，但是不会丢更多，这实际上是模仿Linux或者Unix文件系统的普通工作模式。在一个分布式文件系统中，很多操作都是在模仿Unix风格的文件系统，这样使用者才不会觉得Frangipani的行为异常，因为它基本上与用户在使用的文件系统一样。

Frangipani通过lock锁机制实现缓存一致性。锁服务器(lock server)维护一张表table，里面维护每个file对应的inode编号，以及锁的owner拥有者对应哪个工作站。这里lock server本身是一个分布式服务，可以想象成类似zookeeper，其提供加/解锁接口，且具有容错性，使用基于Paxos的实现，分布在多个机器上。

| file inode | owner |
| ---------- | ----- |
| f          | ws1   |
| g          | ws1   |
| h          | ws2   |

同时，工作站自身也需要维护一张table，列举它所持有的锁，table维护lock对应的状态，比如f文件锁对应的状态为busy，g文件锁对应的状态为idle，表示g这一时刻没有被修改。这里idle状态的锁被称为**粘性锁(sticky lock)**。

| file inode | State |
| ---------- | ----- |
| f          | busy  |
| g          | idle  |

由于ws1占有g对应的**sticky lock**，之后如果再次使用文件g，不必与Petal和锁服务器通信或重新加载cache之类的，因为拥有sticky lock说明这段时间内没有其他工作站获取锁。

通过上述的两种锁，配合一套rule规则，即可实现缓存一致性：

- Guiding Rule: **缓存文件之前，需要先获取锁(to cache file, first acquire lock)**

在论文中描述的lock锁是排他的(exclusive)或读写(read-write)锁，后续的课程为了优化，会假设不需要锁的排他性质，多个工作站可以在只读模式下拥有文件缓存。

## 协议(protocol)

假设有WS1(工作站/client1)、LS(锁服务器)、WS2(工作站/client2)。

在WS和LS之间通信，会使用4种消息：

- 请求锁，requesting a lock
- 授予锁，granting a lock
- 撤销锁，revoking a lock
- 释放锁，releasing a lock

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/202311222035682.png" alt="image-20231122203549057" style="zoom: 33%;" />

工作流程：

1. WS1向LS请求锁**(request f)**，要求访问文件f；

2. LS查询自己维护的file inode => owner表，发现f没有被任何WS上锁，记录f对应的owner为WS1，并响应WS1授予锁**(grant f)**；

3. WS1获得f的锁，可以从Petal读取或修改文件，这些修改保存在client本地(write-back cache)。

   本地table中修改f文件对应的状态为busy，写操作执行完后，将f文件的锁状态改成idle。这里lock绑定租约期限，WS1需要定期向LS续约，但是不必从Petal重新读取文件f；

4. WS2想问访问f，向LS发送请求锁**(request f)**；

5. LS查看table，发现f对应的owner为WS1，于是向WS1发送撤销锁消息**(revoke f)**；

6. 必须确保WS2观察到WS1完成的写入，因此WS1对f的修改同步到Petal；

7. Petal确认已经收到了所有数据后，WS1向LS发送释放锁**(release f)**，同时删除自己本地对f锁的记录；

8. LS发现WS1释放锁后，重新修改table中f的owner为WS2；

9. LS向WS2发送授予锁**(grant f)**；

**因为工作站访问Petal的文件需要获取锁，并且之前的owner释放锁前会将状态刷新(flush)到Petal，所以保证了缓存一致性(cache coherence)，即后续对相同文件进行访问的工作站一定能看到最新的修改内容**。

Frangipani的lock确保不同服务器对相同data的更新是有序的。即保证至多有1个log能够持有对某个特定block的未完成的更新。

> Frangipani’s locking protocol ensures that updates requested to the same data by different servers are serialized.  A write lock that covers dirty data can change owners only after the dirty data has been written to Petal, either by the original lock holder or by a recovery demon running on its behalf. This implies that at most one log can hold an uncompleted update for any given block  

------

问题：我们需要写入Petal，当释放读和写锁时，为什么我们需要在释放读锁时写入Petal？

回答：让我们忽略读写，只关注排他(即这里要求不管是读还是写，都要获取排他锁)。

问题：这看上去效率很低，如果两个工作站都一直操作同一个文件的话。

回答：是的，如果两个工作站一直在处理相同的文件，就会导致文件在工作站的缓存和Petal之间来回跳动。不过这里应用场景中，大多数研究者都在处理私人文件，所以它们同时处理一个共享文件的概率很低。就好像git，平时在本地操作，偶尔才会和远程进行同步。

问题：如果WS1占有f锁为busy时，WS2向LS请求f的lock会发生什么？

回答：WS2会等待，直到WS1完成修改并在本地释放锁，并开始将所有操作都刷新到Petal，然后释放锁。

## 原子性(atomicity using locks)

> [11.5 原子性（Atomicity） - MIT6.824 (gitbook.io)](https://mit-public-courses-cn-translatio.gitbook.io/mit6-824/lecture-11-cache-consistency-frangipani/11.5-yuan-zi-xing-atomicity) <= 摘抄部分内容，作为补充说明
>
> 为了让操作具备原子性，Frangipani持有了所有的锁。对于锁来说，这里有一件有意思的事情，Frangipani使用锁实现了两个几乎相反的目标。
>
> - 对于缓存一致性，Frangipani使用锁来确保写操作的结果对于任何读操作都是立即可见的，所以对于缓存一致性，这里使用锁来确保写操作可以被看见。
> - 但是对于原子性来说，锁确保了人们在操作完成之前看不到任何写操作，因为在所有的写操作完成之前，工作站持有所有的锁。

Frangipani使用同样的锁机制来实现原子文件系统操作，不让其他工作站看到中间结果。

假设WS1在目录d下创建f目录(create f)，那么根据论文所描述，会获取目录d的锁，然后获取文件f的inode锁，大致伪代码如下：

```pseudocode
acquire("d") // 获取目录d的锁
create("f" ,....) // 创建文件f
acquire("f") // 获取文件f的inode的锁
    allocate inode // 分配f文件对应的inode
    write inode // 写inode信息
    update directory ("f", ...) // 更新目录关联f对应inode到d目录下
release("f") // 释放f对应的inode锁(本地释放操作busy->idle)
```

**这里通过锁，保证对本地Unix文件系统内部涉及的inode等操作进行保护，达到原子操作文件的目的**。

当进行操作时，WS1将lock状态改成busy，操作完成后，改状态为idle。

此时期间LS要求撤销锁(revoking a lock)，请求不会被处理，直到WS1释放了本地锁（busy->idle），看到了有一个revoke在等待。此时WS1刷新缓存状态到到Petal，之后安全地释放锁(releasing a lock)。

------

问题：论文中提到目录和inode都需要锁，所以WS1在释放锁(releasing a lock)之前，需要先释放2种文件锁？

回答：是的。论文中有提到实现中需要用不同种类的锁，它们对每个inode都有一个锁，包括目录的inode、文件的inode。事实上目录和文件没什么不同，只不过有特定的format格式。所以创建f，我们需要先分配和获取目录d的锁，然后分配获取到inode f的锁，所以持有两把锁。一旦出现获取多把锁的场景，就可能陷入死锁。如果一个工作站以不同的顺序分配锁，你可能陷入死锁。**所以Frangipani遵循规则，所有的锁都以特定的方式排序，以固定的顺序获取锁，我想锁时按照inode编号排序的**。

## crash recovery -- write-ahead logging

> [11.6 Frangipani Log - MIT6.824 (gitbook.io)](https://mit-public-courses-cn-translatio.gitbook.io/mit6-824/lecture-11-cache-consistency-frangipani/11.6-gu-zhang-hui-fu-crash-recovery) <= 摘抄部分内容，作为补充说明
>
> 有关Frangipani的Log系统有意思的事情是，**工作站的Log存储在Petal，而不是本地磁盘中**。几乎在所有使用了Log的系统中，Log与运行了事务的计算机紧紧关联在一起，并且几乎总是保存在本地磁盘中。但是出于优化系统设计的目的，Frangipani的工作站将自己的Log保存在作为共享存储的Petal中。每个工作站都拥有自己的半私有的Log，但是却存在Petal存储服务器中。**这样的话，如果工作站崩溃了，它的Log可以被其他工作站从Petal中获取到。所以Log存在于Petal中**。
>
> **这里其实就是，每个工作站的独立的Log，存放在公共的共享存储中，这是一种非常有意思，并且反常的设计**。
>
> 这里有一件事情需要注意，**Log只包含了对于元数据的修改，比如说文件系统中的目录、inode、bitmap的分配。Log本身不会包含需要写入文件的数据，所以它并不包含用户的数据，它只包含了故障之后可以用来恢复文件系统结构的必要信息**。
>
> **为了能够让操作尽快的完成，最初的时候，Frangipani工作站的Log只会存在工作站的内存中，并尽可能晚的写到Petal中**。这是因为，向Petal写任何数据，包括Log，都需要花费较长的时间，所以我们要尽可能避免向Petal写入Log条目，就像我们要尽可能避免向Petal写入缓存数据一样。
>
> 所以，这里的完整的过程是。当工作站从锁服务器收到了一个Revoke消息，要自己释放某个锁，它需要执行好几个步骤。
>
> 1. 首先，工作站需要将内存中还没有写入到Petal的Log条目写入到Petal中。
> 2. 之后，再将被Revoke的Lock所保护的数据写入到Petal。
> 3. 最后，向锁服务器发送Release消息。

更新Petal中的state，需要遵循**预写式日志(write-ahead logging)**协议。可以说Petal就是为预写式日志(write-ahead logging)协议设计的。

![image-20231123213507984](https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/202311232135243.png)

在Petal中，virtual disk是一个块阵列，由两部分组成：

- **log** (Each Frangipani server obtains a portion of this space to hold its private log.)
- file system(其余块用于文件系统，包含inode，data block数据块)

当更新Petal中的state状态时（WS把锁还给lock server，将状态写入Petal），经过以下流程：

- 更新日志(first log to update)
   - 创建描述该更新的记录并定期附加到Petal的log中（需要发生在file system blocks上的修改），如create操作、分配的inode编号、目录变化；

   - 一旦记录了所有的变化，更新data blocks就安全了，因为它们总会更新文件系统并以一致的状态结束；

- 安装更新(install the update)
   - 工作站log记录完毕后，就可以实际安装更新，即modify th actual metadata in its permanent locations，这被Unix update demon定期执行，demon查看log并应用到文件系统；

**为什么不立即写入或更新文件系统？**因为在更新的过程中可能会崩溃。分配inode f和将inode f添加到指定目录是两个独立的磁盘写入，不是原子的，可能在某处发生崩溃，比如分配了inode却不再目录中，就会丢失inode。

------

问题：这里怎么保证第一步的更新log的操作是原子的？

回答：论文中有提到几种方式，每个log记录都有一个校验和(checksum)，在读取日志记录之前用校验和确保log的完整性。

### Log record

每台Frangipani服务器都有log日志，log中的log entry拥有序列号(sequence number)。**log entry中存放更新数组(array of updates)**，用于描述文件系统操作，包含以下内容：

- block# 需要更新的块号(block number that needs to be updated), 

  在我们的示例中，对应the block contains inode编号

- version# **版本号(version number), **

- 块编号对应的新字节数据(new bytes)


例如`create f`，在数组中会有两个条目，一个描述对inode块的更新，一个描述对目录数据块(the data block of the directory)的更新。

**复制过程**(即LS向WS发送revoke撤销锁后，WS需要将文件变更同步到Petal)，经过以下流程：

1. **log发送到Petal (force the log to Petal)**
2. **发送更新块到Petal (send the updated blocks to Petal)**
3. **释放锁(release lock)**

### metadata & file blocks

- **文件数据(file data)写入不会通过日志，这些数据块会直接写入到Petal。**

- **通过日志的唯一更新是元数据(meta data)更改**。元数据的含义是关于file文件的信息，比如inode、目录等，这些会通过log。应用级数据，实际构成文件的文件块直接写入到Petal，而不需要通过log。

- **因为文件数据没有记录到log中，对数据的更新可能会丢失。**

  - 这意味着如果应用程序需要原子地写某个文件，需要自己以某种方式保证，而大多数Unix文件都是这么设计的，在Frangipani角度不能破坏Unix原来的文件系统设计。

  - **对于需要原子写文件的场景，通常应用程序通过先将所有数据写入一个临时文件(temporary file)，然后做一个原子重命名（atomic rename），使得临时文件变成最终需要的文件**。同理，如果Frangipani需要原子写文件，也会采用这种方式。

- 只将文件系统的元数据通过log存储是因为，元数据比实际数据小很多。
  - 如果写GB级别的文件，首先需要存入log，之后写入磁盘会极大降低性能。
  - 但是文件系统内部结构保持一致非常重要，比如inode、目录等需要保证在Petal上有一致的表现，所以元数据需要记录到log中。

### crash recovery

> [11.7 故障恢复（Crash Recovery） - MIT6.824 (gitbook.io)](https://mit-public-courses-cn-translatio.gitbook.io/mit6-824/lecture-11-cache-consistency-frangipani/11.7-gu-zhang-hui-fu-crash-recovery) 
>
> Frangipani出于一些原因对锁使用了租约，当租约到期了，锁服务器会认定工作站已经崩溃了，之后它会初始化恢复过程。**实际上，锁服务器会通知另一个还活着的工作站说：看，工作站1看起来崩溃了，请读取它的Log，重新执行它最近的操作并确保这些操作完成了，在你完成之后通知我。在收到这里的通知之后，锁服务器才会释放锁**。这就是为什么日志存放在Petal是至关重要的，因为一个其他的工作站可能会要读取这个工作站在Petal中的日志。
>
> 但是幸运的是，执行恢复的工作站可以直接从Petal读取数据而不用关心锁。这里的原因是，执行恢复的工作站想要重新执行Log条目，并且有可能修改与目录d关联的数据，它就是需要读取Petal中目前存放的目录数据。接下来只有两种可能，要么故障了的工作站WS1释放了锁，要么没有。如果没有的话，那么没有其他人不可以拥有目录的锁，执行恢复的工作站可以放心的读取目录数据，没有问题。如果释放了锁，那么在它释放锁之前，它必然将有关目录的数据写回到了Petal。这意味着，Petal中存储的版本号，至少会和故障工作站的Log条目中的版本号一样大，因此，之后恢复软件对比Log条目的版本号和Petal中存储的版本号，它就可以发现Log条目中的版本号并没有大于存储数据的版本号，那么这条Log条目就会被忽略。所以这种情况下，**执行恢复的工作站可以不持有锁直接读取块数据，但是它最终不会更新数据。因为如果锁被释放了，那么Petal中存储的数据版本号会足够高，表明在工作站故障之前，Log条目已经应用到了Petal。所以这里不需要关心锁的问题**。

 接下来分析一下不同崩溃场景下，Frangipani如何处理：

- 写log前就崩溃(before writing log) --> 数据丢失

- log写到Petal后崩溃(crash after writing log to Petal)

  - 当log写到Petal后WS1崩溃，如果WS2想要获取同一个文件inode的lock，Lock Server会等待WS1的lock租约过期（防止网络分区）；
  - **要求剩余的WS的*recovery demon*读取WS1的log，并应用log中记录的操作**（隐式的基于recovery demon崩溃服务器的log和locks的所有权The recovery demon is implicitly given ownership of the failed server’s log and locks.  并且用exclusive lock确保任何时间只有一个recovery demon尝试replay特定server的log region）

  - demon工作完成后，Lock Server重新分配锁，即授予WS2锁；（这里术语demon，通常指一项服务or服务器or服务器进程，通常用于处理一些额外工作，而不是提供主要的服务。）
  - 这里写log后崩溃，不能保证用户写的文件数据：**log系统只用于保证内部文件系统数据结构一致**。如果内部文件系统数据结构混乱是很糟糕的一件事情，每个人都可能会丢失数据。

- 写log到Petal的过程中崩溃(crash during writing the log to Petal)

  prefix end up in the log，每个前缀可能包含多个操作，如果在其中一个记录更新期间崩溃，那么校验和（checksum）就不会通过，recovery demon会停止在checksum不通过的那个记录。
  
  > 除非在Petal中找到了完整的Log条目，否则执行恢复的工作站WS2是不会执行这条Log条目的，所以，这里的隐含意思是需要有类似校验和的机制，这样执行恢复的工作站就可以知道，这个Log条目是完整的，而不是只有操作的一部分数据。这一点很重要，因为在恢复时，必须要在Petal的Log存储区中找到完整的操作。所以，对于一个操作的所有步骤都需要打包在一个Log条目的数组里面，这样执行恢复的工作站就可以，要么全执行操作的所有步骤，要么不执行任何有关操作的步骤，但是永远不会只执行部分步骤。

## Frangipani-日志和版本

![assets_-MAkokVMtbC7djI1pgSw_-MFzXK0wOxMuYYWWxtF5_-MG16qe6dMtFMJ6UfCXi_image](https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/assets_-MAkokVMtbC7djI1pgSw_-MFzXK0wOxMuYYWWxtF5_-MG16qe6dMtFMJ6UfCXi_image.png)

 这里讨论下一个场景，假设有WS1～WS3，整体操作时序如下：

1. WS1在log中记录`delete ('d/f')`，删除d目录下的f文件
2. WS2在log中记录`create('d/f')`
3. WS1崩溃
4. WS3观察到WS1崩溃，为WS1启动一个recovery demon，执行WS1的log记录`delete('d/f')`

原本WS2能在Petal中创建d目录下的f文件，但是WS3却有可能因为恢复执行WS1的log的缘故，将f文件删除。**这里Frangipani通过日志版本号避免了这个问题**。

> 当工作站需要修改Petal中的元数据时，它会向从Petal中读取元数据，并查看当前的版本号，之后在创建Log条目来描述更新时，它会在Log条目中对应的版本号填入元数据已有的版本号加1。之后，如果工作站执行到了写数据到Petal的步骤，它也会将新的增加了的版本号写回到Petal。
>
> 所以，如果一个工作站没有故障，并且成功的将数据写回到了Petal。这样元数据的版本号会大于等于Log条目中的版本号。如果有其他的工作站之后修改了同一份元数据，版本号会更高。

 实际上，上面的操作，可以详细成如下：

1. WS1在log中记录`delete ('d/f')`，删除d目录下的f文件。log的版本号假设为10，表示log影响的metadata的版本号为10；
2. WS2在log中记录`create('d/f')`，必然WS1已经释放了相关数据的锁，WS2获得了锁，并在写入数据时会将版本号置为11（因为**锁保证了log的版本号是完全有序的**）
3. WS1崩溃
4. WS3观察到WS1崩溃，为WS1启动一个recovery demon，准备执行WS1的log记录`delete('d/f')`，它会首先检查版本号，log条目中的版本号以及petal中存储的版本号，**如果Petal中存储的版本号大于等于Log条目中的版本号，那么WS3会忽略Log条目中的修改**，因为很明显Petal中的数据已经被故障了的工作站所更新，甚至可能被后续的其他工作站修改了。如发现Petal中已应用的log对应的version为11，大于等于准备重放的log的version为10，所以demon会放弃重放这个log。

------

**问题：version版本号总是绑定到正在编辑的inode上吗？**

**回答：是的。比如文件有一个version。前面"12.8 Frangipani-崩溃恢复(crash recovery)"提到的更新列表中，每一项会记录更新的块编号、version、更新的数据**。

> 这里有个比较烦人的问题就是，WS3在执行恢复，但是其他的工作站还在频繁的读取文件系统，持有了一些锁并且在向Petal写数据。WS3在执行恢复的过程中，WS2是完全不知道的。WS2可能还持有目录 d的锁，而WS3在扫描故障工作站WS1的Log时，需要读写目录d，但是目录d的锁还被WS2所持有。我们该如何解决这里的问题？
>
> 一种不可行的方法是，让执行恢复的WS3先获取所有关联数据的锁，再重新执行Log。这种方法不可行的一个原因是，有可能故障恢复是在一个大范围电力故障之后，这样的话谁持有了什么锁的信息都丢失了，因此我们也就没有办法使用之前的缓存一致性协议，因为哪些数据加锁了，哪些数据没有加锁在断电的过程中丢失了。
>
> 但是幸运的是，执行恢复的工作站可以直接从Petal读取数据而不用关心锁。这里的原因是，执行恢复的工作站想要重新执行Log条目，并且有可能修改与目录d关联的数据，它就是需要读取Petal中目前存放的目录数据。接下来只有两种可能，要么故障了的工作站WS1释放了锁，要么没有。如果没有的话，那么其他人不可以拥有目录的锁，执行恢复的工作站可以放心的读取目录数据，没有问题。如果释放了锁，那么在它释放锁之前，它必然将有关目录的数据写回到了Petal。这意味着，Petal中存储的版本号，至少会和故障工作站的Log条目中的版本号一样大，因此，之后恢复软件对比Log条目的版本号和Petal中存储的版本号，它就可以发现Log条目中的版本号并没有大于存储数据的版本号，那么这条Log条目就会被忽略。所以这种情况下，执行恢复的工作站可以不持有锁直接读取块数据，但是它最终不会更新数据。因为如果锁被释放了，那么Petal中存储的数据版本号会足够高，表明在工作站故障之前，Log条目已经应用到了Petal。所以这里不需要关心锁的问题。

## Frangipani-小结

 Frangipani虽然实际应用不广泛，但是其应用到的一些思想/设计，直接学习。

- 缓存一致性协议(cache coherence)
- 分布式锁(distributed locking)
- 分布式恢复(distributed recovery)

 后续章节即将讨论的分布式事务，也会用到这些设计思想。

------

问题：这里的缓存一致性，不是在两个地方有一个文件缓存，对吧？

回答：是的。

问题：你说每一条记录都是原子操作的，但是每条记录也有一些更新，对吧？

回答：论文描述比较含糊，磁盘可以保证单个扇区(512字节)的操作是原子的，可以利用这一点实现。或者使用log的校验和，读扇区时重新计算校验和，对比存储中的校验和是否一致，如果没错，那就是完整的一个记录。
