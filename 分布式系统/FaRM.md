# 乐观并发控制-FaRM(Optimistic Concurrency Control)]

> [FaRM论文笔记 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/597272185) <= 网络大佬的笔记

> [非易失性存储器技术_百度百科 (baidu.com)](https://baike.baidu.com/item/非易失性存储器技术/22383908?fr=aladdin)
>
> 非易失性存储器技术是在关闭计算机或者突然性、意外性关闭计算机的时候数据不会丢失的技术。非易失性存储器技术得到了快速发展，非易失性存储器主要分为[块寻址](https://baike.baidu.com/item/块寻址/2291420?fromModule=lemma_inlink)和字节寻址两类。
>
> [如何实现内核旁路（Kernel bypass）？_夏天的技术博客的博客-CSDN博客_内核旁路](https://blog.csdn.net/wwh578867817/article/details/50139819) <= 原文列举出了很多流行的内核旁路技术
>
> 关于 Linux 内核网络性能的局限早已不是什么[新鲜事](https://lwn.net/Articles/629155/)了。在过去的几年中，人们多次尝试解决这个问题。最常用的技术包括创建特别的 API，来帮助高速环境下的硬件去接收数据包。不幸的是，这些技术总是在变动，至今没有出现一个被广泛采用的技术。
>
> [RDMA_百度百科 (baidu.com)](https://baike.baidu.com/item/RDMA/1453093?fr=aladdin)
>
> RDMA(Remote Direct Memory Access)技术全称远程直接数据存取，就是为了解决网络传输中服务器端数据处理的延迟而产生的。RDMA通过网络把资料直接传入计算机的存储区，将数据从一个系统快速移动到远程系统存储器中，而不对操作系统造成任何影响，这样就不需要用到多少计算机的处理功能。它消除了[外部存储器](https://baike.baidu.com/item/外部存储器/4843180?fromModule=lemma_inlink)复制和上下文切换的开销，因而能解放内存带宽和CPU周期用于改进应用系统性能。

- **高性能的事务(High performance txns)**
  - FaRM在TATP基准上，使用90台机器，获取1.4亿个事务/秒。作为对比，Spanner能处理10～100个事务/秒。当然，两者是完全不同的系统，Spanner是全球范围内进行同步复制的事务系统，而FaRM的一切都在同一个数据中心内运行；

- **严格的可串行化(strict serializability)**
  - 类似于Spanner提供的外部一致性；

 FaRM(Fast Remote Menmory)为了获取高性能，采取如下手段/策略：

- 单数据中心(One data center)：系统只部署在单个数据中心内
- **数据分片(shard)**：如果不同事务访问不同分片，若互不影响，则可完全并行。
- **非易失性DRAM(non-volatile DRAM)**：采用非易失性DRAM硬件，避免不得不写入稳定存储设备时的写入性能瓶颈。所以在FaRM中，不必写入关键路径到固态硬盘或磁盘，这消除了存储访问成本(storage access cost)。
- **内核旁路技术(kernel-by-pass)**：kernel-by-pass避免了操作系统与网卡交互；
- RDMA(Remote Direct Memory Access)：使用具有RDMA特殊功能的网卡，允许网卡从远程服务器读取内存，而不必中断远程服务器。这提供了对远程服务器或**远程内存(remote memory)**的低延迟网络访问。
- **乐观并发控制(OCC, Optimistic Concurrency Control)**：为了充分利用以上这些提速手段，他们采用乐观并发控制。对比前面经常谈论的**悲观**并发控制方案其需要**在事务访问对象时获得锁**，确保到达提交点时拥有所有访问对象的锁，然后再进行提交；而使用**乐观**并发控制不需要锁，尤其在FaRM中，**不需要在读事务中获取锁**，当进行commit时，需要验证读取是最近的对象，是则提交，不是则中止且可能伴随重试。FaRM使用乐观并发控制的原因，主要是被RDMA的使用所需动的。

 Spanner目前仍是活跃使用中的系统，而FaRM是微软的research prototype。

# Setup

> [不间断电源_百度百科 (baidu.com)](https://baike.baidu.com/item/不间断电源/271297?fromModule=lemma_search-box&fromtitle=UPS&fromid=2734349)
>
> UPS即不间断电源(Uninterruptible Power Supply)，是一种含有[储能装置](https://baike.baidu.com/item/储能装置/53290983?fromModule=lemma_inlink)的不间断电源。主要用于给部分对电源稳定性要求较高的设备，提供不间断的电源。

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/1214%20FaRM.png" alt="image-20231214211341735" style="zoom:50%;" />

- 高速网络：数据中心中，90台机器通过高速数据中心网络连接，内部是一个交换网络。
- 数据分片：分布在不同机器上。根据分片的级别，被称为区域(region)，大小为2GB。
- DRAM：**region分布在内存DRAM中，而不是在磁盘上**，减少存储设备性能的瓶颈。所以数据库的全部数据集必须和机器的joint DRAM相适应。如果数据集大于所有机器的内存总量，则必须配置更多的机器和DRAM。
- 主备复制：如果宕机DRAM数据会丢失，需要对region进行主备复制(P/B replication)。这里通过配置管理器(configuration manager, CM)配合zookeeper，跟踪region#到primary、backups机器的映射(mapping)。
- 不间断电源(uninterruptible power supply, UPS)：数据中心断电就会导致所有DRAM丢失数据，所以DRAM配置在有不间断电源的机器上。如果发生断电，FaRM通过UPS及时将数据存储到SSD上，或者是将内存中的内容(regions、transaction state、logs等)刷新(flush)到SSD上。之后恢复电源再从SSD加载内容。这里仅在故障恢复时使用SSD。

 在区域region中有一些对象object，可以把region想象成一个2GB的字节对象数组：

- **对象拥有唯一的标识符oid，由区域编号和区域内的地址偏移量构成**`<region#, offset>`

- 对象有**元数据信息，对象头部包含一个64bit的数字，底(bottom) 63bit版本号(`V#`)，顶1bit锁位(`lock bit`)。**这个头部元信息在乐观并发控制中起重要作用


# API

- txbegin：启动事务；
  - read(oid)：读取指定oid的对象

  - 然后应用可以修改该对象的一些field，如o.f+=1；

  - write(oid, o)：以指定的o对象更新对应oid的对象；

- txcommit：事务提交；

当然，说不定有些情况还需要事务中止，但乐观并发控制，这里会采取重试事务。

实际上会操作位于不同region的对象，FaRM会使用类似2PC的协议来执行原子操作。

------

> 问题：地址oid，是在机器本身的地址吗？
>
> 回答：是的，是对象在region内的偏移量(offset)。region可以通过修改配置管理器里的映射而被移动，所以实际上的object address可能会变动，所以oid是一个region#+偏移量offset。
>
> 问题：创建全局地址空间背后的设计选择或设计思想是什么？
>
> 回答：即为了把所有数据存储到内存中，目标是在内存数据库(in-memory database)上运行事务，不需要访问持久化存储(persistent storage)。
>
> 追问：所以它们共享一个全局地址空间？
>
> 回答：地址空间是按机器的，每个机器都有自己的地址空间，从0到某个数值。而对象的oid是全局编号或全局名称。
>

# 内核旁路(Kernel-bypass)
> [DPDK介绍_growing_up_的博客-CSDN博客](https://blog.csdn.net/growing_up_/article/details/124323725) <= 图文并茂，推荐阅读
>
> DPDK是INTEL公司开发的一款高性能的网络驱动组件，旨在为数据面应用程序提供一个简单方便的，完整的，快速的数据包处理解决方案，主要技术有用户态、轮询取代中断、零拷贝、网卡RSS、访存DirectIO等。

FaRM作为用户程序运行在Windows操作系统服务器上。通常操作系统OS内的内核驱动程序(driver)读写网卡(network interface card, NIC)的寄存器来完成网络数据交互。而应用程序与网卡交互时，则需要进行内核系统调用，涉及到操作系统、TCP stack、网络栈(network stack)，相关的开销十分昂贵，而FaRM想要避免这类开销。

- FaRM采用**内核旁路技术**，网卡的发送/接收队列被直接映射或进入应用程序的地址空间，应用程序可以直接请求操作系统读取网卡的队列。先不考虑内核旁路实现细节，这里可以认为用户应用程序FaRM可以直接读写网卡数据，而不需要涉及操作系统。

- 在FaRM的内核旁路实现中，不采用中断机制来感知网卡数据，而是用一个用户级别的线程主动**轮询网卡中的接收队列**，查看是否有包可用，FaRM在应用程序线程和轮询网卡线程之间来回切换。

DPDK一个利用内核旁路的开发工具包，是一个合理的标准，在诸多操作系统上可以使用。

# 远程直接数据存取(RDMA)
> [RDMA_百度百科 (baidu.com)](https://baike.baidu.com/item/RDMA/1453093?fr=aladdin)
>
> RDMA是Remote Direct Memory Access的缩写，意思是远程直接数据存取，就是为了解决网络传输中服务器端数据处理的延迟而产生的。

除了内核旁路直接处理网卡数据外，FaRM还使用远程直接数据存储(RDMA)。

RDMA的大致使用介绍，即两台FaRM机器通过电缆(cable)连接，机器1可以将RDMA包通过内核旁路直接放入发送队列，并将其发送到机器2网卡，机器2网卡解析与RDMA包一起的指令，这些指令可能是读取或写入特定内存位置，如读取region的某个object的地址，然后直接将数据返回source。

在RDMA中**网卡能够直接操作内存**，而不需要依赖中断或服务器的其他帮助，不产生中断，也不需要处理器上运行任何代码。相反，网卡具有固件执行这些指令，加载存储在内存中的值，请求内存地址直接进入响应包(response packet)，并发回数据包。而接受端会在接收队列中获取RDMA的结果。

这里描述的RDMA流程对应的版本，论文中称为**单边RDMA(ont-sided RDMA)**，通常指读取操作，只需要约5微秒。

FaRM也是用RDMA进行写入，真正实现RPC，论文中称之为**写RDMA (write RDMA)**。和读取流程基本一致，除了发送方说明这是一个写操作RDMA包，并将以下字节写入特定地址。

 **在论文中有两处使用到写RDMA**：

- **日志(log)**：包括事务的提交记录等，source可以通过写RDMA来append日志记录。每个sender receiver pair都有一个队列和一个日志，所以sender可以管理和感知日志的开头和结尾(???)。
- **RPC消息队列(message queue)**：消息队列也是每对一个(one per pair)，用于实现RPC。如果想进行远程过程调用，客户端发送方生成write RDMA包，将数据信息写入RPC消息队列。而目标侧/RPC接收方有一个线程用于轮询(所有)消息队列，看到消息则进行处理，然后可以再使用 write RDMA进行响应。使用RDMA实现RPC比标准的RPC性能好（标准的RPC没有使用RDMA，需要经过传统的网络协议栈等流程）。

整体而言，RDMA是一种很棒的用于提高网络数据处理性能的技术，在过去十年来被各方普遍使用。在FaRM中，单边RDMA耗时约为5微秒，这比写入本地内存慢不了多少，非常快。

------

> 问题：可以重复一下网卡是如何工作的吗？
>
> 回答：客户端只有一个专门读取特定内存位置的线程，表示数据包是否已经到达。当网卡接收到包，将其放入接收队列中，作为接收队列中设置的一侧，标志位变为1，应用程序知道那里有一个包。
>
> 追问：是不是一个特殊的线程轮询？
>
> 回答：是的，系统中有特定的线程专门轮询队列。
>
> **问题：网卡是否与系统合作，正常工作，就像普通网卡一样？**
>
> **回答：这不是普通网卡，这个网卡同时支持内核旁路(kernel-bypass)和远程直接内存访问(RDMA)。通常，网卡为了支持内核旁路，意味着它必须有多个接收和发送队列。它只项应用程序提供一对发送或接收队列。当然，你不能让计算机上每个进程都有一个发送和接收队列。这里通常是16个或32个队列，把其中一些放到特定的操作系统上，允许一些应用程序拥有发送接收队列。这也意味着有对RDMA的支持，并使它正常工作。所以它(FaRM)需要一个相当复杂的网卡，尽管如今这是一个合理的标准(即这类网卡现在很常见了)。**
>
> 问题：这里是否存在任何验证步骤，为了确保你只在允许RDMA访问的内存区域进行写入？
>
> 回答：有的。当你设置RDMA时，为了做单边RDMA(one-sided RDMA)或写入RDMA(write RDMA)，你首先必须进行连接设置(connection setup)，在发送者和接收者之间存在协商步骤来设置，就好像TCP channel，虽然RDMA不采用TCP。但是RDMA建立了面向连接的可靠、有序的通道，所以安全检查和访问控制检查，是在设置时进行的。
>
> 追问：需要对每个机器都进行RDMA配置吗？
>
> 回答：是的。
>
> 追问：如果每个机器都需要手动进行RDMA配置，那么加入更多机器称为大集群，配置操作是昂贵的，对吧？
>
> 回答：是的，你会有n\^2个RDMA连接需要配置，否则就要有n\^2个TCP连接。
>
> **问题：上面说的write RDMA的应用场景之一，日志log，也是存储在内存中，并且是和对象object存在不同位置？**
>
> **回答：是的。你可以认为FaRM机器上的内存布局，即一部分用于区域(Region)存储对象(object)数据，一部分用于存放log，一部分用于存放RPC消息队列。**
>
> 问题：为了让网卡支持从内存直接访问，这里不涉及网卡外的其他软件，甚至不需要通知应用或操作系统，不应该在硬件层面进行一些协调，或者至少从处理器支持这个特性？
>
> 回答：是的，网卡可以原子地读写缓存行。为了支持这点，有一个接口在内存系统和网卡之间，必须在操作系统仔细设置，当RDMA连接设置完成时。
>
> 问题：队列只用于读RDMA，而写入直接到接收者的内存？
>
> 回答：在 write RDMA中，可能会有确认响应，所以如果发送方发送 write RDMA，它可以等待来自接收方网卡的确认响应(acknowledgement)，表示执行了写入RDMA。
>

# Challenge: transaction using RDMA

前面提到的内核旁路(kernel-bypass)，远程直接数据存取(RDMA)都是很棒的技术，但都算是业界内已存在的技术标准。而FaRM解决的真正挑战在于使用RDMA实现事务(transaction using RDMA)，即通过write RDMA和one-sided RDMA进行事务。

之前介绍过的事务协议、2PC等，都需要服务端参与(server-side participation )，如获取锁或进行一些验证步骤查看是否可以commit... 这意味着必须在服务器上运行代码。而**RDMA本身没有提供在服务器上运行代码的能力**，所以需要论文的作者需要额外的协议实现2PC和事务，不使用或减少服务端参与。

# 乐观并发控制(Optimistic concurrency controlOCC)

> [FaRM论文笔记 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/597272185) <= 图片出处

FaRM使用**乐观并发控制(OCC, Optimistic concurrency control)**。

对读取操作的处理--**无锁读取对象(read objs w.o. locks)**，

- 需要依赖object的头部信息版本号判断数据版本；

- *如果读操作需要锁，就意味着需要中断服务器，服务器被中断后就需要做一些额外工作，这可能会阻塞客户端直到锁可用才返回对象。使用锁的方案不是很适合RDMA*；

 **FaRM只读事务基本流程：**

1. **read objs w.o. locking(version#)** 事务无锁地读对象object以及version版本号
2. **validation step for conflicts** 提交前进行冲突校验
   - **version# are different => abort** 如果coordinator在一开始读取的对象和当前的版本号不同(即该对象从读取到当前又被其他事务修改了)，则说明发生了冲突，中止abort事务，客户端随后可能会再次运行整个事务；
   - **version# are same => commit** 如果没有冲突，意味着没有其他事务修改对象；

这种OCC使读取操作不需要服务器上的任何状态变化，可以完全利用RDMA。

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/1216%20farm%20occ.jpg" alt="FaRM论文笔记" style="zoom: 67%;" />

C是coordinator应用程序（应用程序运行在相同的机器上，其运行在90台机器的系统中其中一台，这里忽略机器数量，后续都假设在同一台机器上运行，方便课程讲解(==这句话没看懂???==)）。P1～P3为3个分片的primary，对应back-up B1～B3。其中P1和P2的数据read and written，而P3 only read。虚线表示RDMA reads (one-sided RDMA)，实线表示RDMA writes ，点线表示网卡硬件acks。

**事务的流程分为执行阶段(execution phase)和提交阶段(commit phase)**：

- **执行阶段**从不同机器上读取obj和version#，并在本地做一些修改。
- 应用程序calls end of Tx则为提交点，在**提交阶段**应用更改，要么提交成功要么中止，中止因为有其他事务同时运行，并且修改了其中一个对象。
- 挑战是实现**严格的可串行化(strict serializability)**。在某些方面这里关于writes的协议类似于2PC。

**提交阶段(commit phase)分为5个步骤**：LOCK、VALIDATE、COMMIT BACKUP、COMMIT PRIMARY、TRUNCATE。

1. 加锁(LOCK)

   在这个case中，write RDMA依赖primary的lock entry。**primary的log日志记录锁的使用情况**，写object时，向log追加**lock entry，其组成包括：execute读取时object的原版本version#、对象object的oid、新值new value。**

   P1和P2的数据被写，所以lock entry通过write RDMA追加到P1和P2的日志。P1/P2机器上有线程轮询log，看到新的log记录，尝试通过**test-and-set获取object的lock**（lock bit = 0并成功置为1），然后发回write RDMA信息并附加到Coodinator的消息队列中，告知C成功获取P1/P2的这些数据对象的锁。

   如果P1和P2尝试获取锁时已经有其他事务加锁了， 则**test-and-set失败**，返回并附加RDMA消息到TC的消息队列，告知获取锁失败，**C会abort当前事务**。

   they ask for locks after really written the object(个人理解是在本地修改)，如果提交阶段加锁失败，意味着有其他事务修改了当前事务写的object，所以需要中止事务（如果继续执行，当前事务就违反了串行化）。**某种程度上LOCK是write txn的串行化点(the serialization point for write transaction)。**在这个点上，事务获取到所有执行阶段写的object的锁，其他事务都不能修改相关的object。

2. 验证(VALIDATE)

   **校验步骤只使用了单边RDMA(one-sided RDMA)**，不需要真正的服务器参与。读操作不需要LOCK，但是需要VALIDATE步骤，校验version版本是否冲突（即是否被其他事务修改了），如果没有冲突则继续，有冲突则中止事务。

   **对于每个已读取但未修改的对象(P3)，Coodinator请求从primary读取object元信息。如果lock bit为1或者v#较txn开始时发生变化，意味着其他并发事务尝试修改object，则中止事务，否则正常进行事务**。

   事务按照版本号的顺序提交，来获取严格的可串行化，因为在当前事务commit后start的任何事务都有更高的版本号。**这里VALIDATE之后，COMMIT BACKUP之前，可认为就是事务的提交点(commit point)**，因为已经对所有被写的object获取锁，也校验了只读object的version。

3. 提交备份节点(COMMIT BACKUP)

   为了容错性，TC通过write RDMA向backup的log写入**commit backup record**，记录与primary lock record相同的信息：object的version number、oid、new value。

   之后backup server不需要运行任何东西，NIC响应ack，告知TC已收到并执行write RDMA。这时候TC知道对象所在的所有primary和backup都已经拥有了变更log记录。

4. 提交主节点(COMMIT PRIMARY)

   TC通过write RDMA附加commit record到primary的日志log中，包含正在commit的transaction id。之后primary NIC对写入RDMA响应ack。（这里不需要任何中断，没有服务器参与，只有网卡参与了操作。）此时该事务really truly committed，**所以这是true commmit point**，协调者通知app事务已经提交并完成了。

5. 截断(TRUNCATE)

   runs almost lazily。tid事务执行完毕后，日志需要进行清理、缩短和截断等工作，这里TC通知被写数据object所在的primary和backup进行日志截断等清理工作。

截断(TRUNCATE)是耗时较长的步骤。而在我们看来提交主节点(COMMIT PRIMARY)后，截断(TRUNCATE)之前，就是事务结束的时间点。

------

> **问题：这里提交阶段(commit phase)的锁是怎么获取的，通过zookeeper吗？**
>
> **回答：不是，另一套锁才会使用到zookeeper，那是为了进行配置管理，比如region number到primary和backup之间的映射。这里用的是由primary自己维护的内存锁。region中的object头部信息64ibt中，最高位1bit用于表示锁lockbit，底部63bit表示版本version。**
>
> 追问：这里lock是内存锁，那primary停机怎么办，backup是否和prmary持有相同的锁？
>
> 回答：如果primary停机，这里事务会中止，恢复协议中有完整的reconfiguration protocol。关于容错性，后面在详细讨论。
>
> 问题：版本号是针对每个object的，对吧？
>
> 回答：是的，每个对象。
>
> 问题：为什么对象已经被锁了之后，当前想加锁的事务直接中止，而不是选择阻塞等待锁被释放？
>
> 回答：因为对象被锁说明对象有新值，而事务没有读取到最新的值（因为读取操作在执行阶段已经执行完了，而加锁在后续的提交阶段中执行），所以事务必须中止。
>
> **追问：懂了，因为锁意味着当前对象下一次会发生改变。**
>
> **回答：是的，但他们在写完对象后才会要求加锁。协调者基于某个版本号修改对象，提交了一些写入。在提交开始时，尝试获取锁，结果发现另一个事务已经获取锁了，这意味着此时存在别的事务已经修改我们刚操作的对象了。而这时候如果继续事务，会违反串行化，所以中止事务。所以获得lock时在某种程度上是write txn的串行化点(serialization point for the write part of the transaction)。**
>
> **问题：上图中，Primary或Backup机器的网卡是直接和TC的网卡交互？**
>
> **回答：是的。回到前面RDMA的介绍中，TC的write RDMA发送到Primary或Backup的网卡，然后接收方网卡发现是RDMA包，解析后按照RDMA包的要求将数据放到指定内存位置，这里会将数据放到内存的log中，然后Primary或Backup网卡操作完毕后，发送确认响应给TC的网卡，TC会在网卡的接收队列中看到确认响应消息。**
>
> 问题：所以写入RDMA只是写入日志？
>
> 回答：write RDMA用于两种情况，对应消息队列和日志追加。
>
> **问题：所以，当我们说已经执行了写入RDMA时，只是指它被附加到log中，并不一定由应用程序实际执行？**
>
> **回答：是的。例如bakcup在log中收到来自TC的write RDMA附加的object变更信息后，再读取log记录，应用对应object的更新。**
>
> 问题：每个对象的lock bit，由于所有数据都在内存中，object的64bit元信息，我想可以放入一个单一的内存地址，但假设处理器将内存地址提取到寄存器中，多核机器的不同内核读取相同地址，都将它们从0变成1，是否假设有一些来自硬件的支持？
>
> 回答：是的。LOCK阶段primary在log中收到来自TC的write RDMA，要求primary对object加锁，primary通过test-and-set加锁，然后回应。正因为object的元信息大小只有64bit，所以test-and-set指令是能够保证原子加锁的。就算是多核机器同时执行test-and-set，也只有一个core会成功，另一个会失败。
>
> 问题：VALIDATE之后，COMMIT BACKUP之前，当前事即务到达提交点(commit point)后，另外一个完全独立的并发事务，是否可能写入P3进行交错（get interleaved）？
>
> 回答：不能，因为如果写入P3，意味着在提交阶段会获取P3的锁，此时会检查P3的对象的锁bit和版本version。这里讲师说先保留这个问题，后面讲课再说类似场景。（我个人理解是可以交错的，因为当前事务对P3只是读取操作，并没有修改version，也没有对P3加锁）
>
> 问题：如果在执行阶段后，它试图获取一个锁，然后在那之后崩溃了，锁已经被获取，之后其他人无法获取。这之后怎么处理？
>
> 回答：这里锁只在内存中，所以Primary崩溃后，锁信息就丢失了。后续会讲的恢复协议中，协议最终会中止事务。
>
> 问题：这里事务协调者TC是客户端，就像应用程序一样，而客户端所有步骤，比如lock... （被打断）
>
> 回答：是的，你可以认为应用程序在论文中系统的90台机器上运行，运行这个事务，然后写P1、P2，读P3。
>
> 问题：所以Primary不直接和Backup通信吗？
>
> 回答：primary不直接和bakcup通信。除了在恢复协议期间，有各种各样的通信发生，但这里事务流程没有体现。
>
> 问题：这里事务协调者TC使用zookeeper的配置？
>
> 回答：是的。这里课程内没有讨论太多细节，但是由zookeeper和connection manager决定我们运行的当前配置，配置包括region的区域划分，region和主/备份之间的映射等等内容。在任何失败发生时，有一个完整的重新配置进程并恢复。
>

## 严格的串行化(strict serializability)

**严格的可串行化要求，如果当前事务在某个事务提交之后才开始，当前事务也要在那个事务之后提交，这通过协议的版本号保证。**

- **如果T2在事务T1提交之后才开始，那么T2必须能观察到T1的结果。**
- **如果T2在T1提交之前开始，那么T2和T1是并发事务，T1和T2可以观察到彼此，其顺序没有关系(T1和T2谁先执行都可接受)。**（如果是读相同的数据，没有写操作，则互不冲突；如果存在写数据，则会有锁冲突，其中一方会中止事务，另一个成功）

***case 1*** 

这里按照下面举例的事务进行串行化讨论，假设T1和T2并发事务执行如下逻辑：

```pseudocode
TxBegin
    o = Read(oid) // 读取指定oid的object，假设o初始值为0
    o += 1 // 修改 object
    Write(oid, o) // 写入指定oid的object
Commit(tid) // or fail
```

假设T1和T2并发执行，则oid对应的对象o的值，可能是0/1/2：0表示一个发生冲突所以中止，另一个失败/崩溃，或者两个都失败/崩溃；1表示一个成功，另一个冲突或失败；2表示两个都成功。

假设T1和T2都几乎在同一时间执行完Read操作，后续在commit phase T1先获取锁(o被修改，需要锁)，那此时T2可能获取到锁吗？

- 如果T1获取了锁但还没有提交，那此时T2会加锁失败，然后事务中止；
- 如果T1很快完成了提交并释放锁，使得T2这里**获取到锁，并检查o的version是否仍然正确，发现o的数据version变动**，也会中止事务。

所以，假设T1和T2都不发生崩溃，那么T1和T2一个成功后，另一个中止事务后可以尝试重新进行事务。

***case 2***

这里举一个经典用例，常用来测试协议是否提供可串行化。当然用例本身不能用于证明可串行化，但它是关键的例子之一。

 T1和T2两个事务的逻辑伪代码如下：

```pseudocode
// 假设 x 和 y 初始值都是0
// T1事务逻辑
if x = 0
    y = 1
    
// T2事务逻辑
if y = 0
    x = 1
```

这是一个很好的可串行化测试，要么T1在T2之后进行，要么T2在T1之后进行，这里要么x=1要么y=1，但永远不该出现x和y都为1（违背了可串行化）。

| 事务 | 时序1                  | 时序2                                                        | 时序3                                                        |
| ---- | ---------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| T1   | Readx(v#0), Ready(v#0) | T1先进入commit phase，Lock Y (Y写操作，设置Y的lock bit 和v#)，Valiadate X(v#0) |                                                              |
| T2   | Ready(v#0), Readx(v#0) |                                                              | T2后进入commit phase，Lock X (X写操作)，Validate Y(v#0)，v#可能还same，但Y的lock bit为1，已经有锁，**Y校验fail，事务abort** |

------

------

> 问题：他们使用这个整个硬件结构，是否适用于悲观并发控制？
>
> 回答：理论上可行，因为这里通过RDMA实现的RPC性能更高。而前面的事务流程图中，可以看到P3只读操作，只需要单边RDMA，没有什么写入，性能很好（执行阶段一次单边RDMA，提交阶段的校验步骤又一次单边RDMA，然后没有后续步骤了）。但是这里RDMA必须配合乐观并发控制才能让只读事务提速。
>
> 问题：网卡读取内存的部分，安全吗？
>
> 回答：实际操作系统和网卡之间有一系列设置，所以操作系统也不会允许网卡写入任意位置。
>
> 问题：关于性能的问题，得益于单边RDMA，读取操作很快，但如果有大量的写入，比如大量数据冲突？
>
> 回答：如果有冲突，那么只有一个事务能正常运行。如果事务操作的都是不同对象，那整体性能会高。
>
> 问题：FaRM这种设计（乐观并发控制）的主要应用场景是什么？
>
> 回答：有很多关于悲观和乐观并发控制的研究，在论文中使用的benchmark如TPC-C和TATP没有太多的冲突。可能是由不同的用户或不同的客户端提交的，它们接触/使用不同的表。
>
> 问题：如果有多个客户端在同一个对象上执行事务，它们想要执行write RDMA，写入到Primary或Backup的log，有没有可能发生冲突？比如其中一个会重写另一个日志之类的。
>
> 回答：不，写log说明有写操作，这里会获取锁。
>
> 问题：事务基于什么基准提供可串行化？是时间还是别的什么。
>
> 回答：版本号version。这里没有类似Spanner的TrueTime之类的机制，但版本号本身起到了类似时间的作用。
>
> 问题：如果两个事务获得了相同的版本号，那么只有先达到提交点的那个才会...（打断）
>
> 回答：是的。只有其中一个会成功，另一个会失败。
>
> 问题：如果在每一对机器之间都建立了消息队列，比如多个消息队列的消息都给primary，你怎么知道读取这些消息的顺序，而不会乱读它们的顺序？
>
> 回答：你以相同的顺序读取同一来源的所有消息。因为它们在接收方的同一个队列里。如果一个来源就对应一个队列，那多个机器同时写入不同的队列，你没法知道顺序是什么。
>
> 问题：论文中提到了无锁读取，也提到提供了本地线索(locality hints)，使程序员能够将同一组机器上的相关对象相互关联。我不理解后段句什么意思，能解释下吗？
>
> 回答：我想你指的是，如果你的对象在各种不同的区域(regions)，然后你必须和很多不同的primary交互。如果你总是接触同一个集群的对象，这是较好的情况，如果所有对象集群都在同一个primary上，那你只需要联系一个primary，而不是多个。
>
> 问题：所以FaRM不是很适合长事务？
>
> 回答：是的，你担心长事务会更容易导致冲突。
>
> 问题：假设有一个事务T1先写入分片1，分片2，然后读取分片3，此时T2在T1完成提交之前，读取分片3然后写分片3，并且比T1先执行了提交阶段的步骤，会发生什么？
>
> 回答：你说的这个场景，假设T1的version是0，T2的version是1，那么T2能成功提交，而T1会中止，因为T1晚进入校验阶段，会发现自己version落后了。
>
> 追问：我想知道的是，如果T1在执行完VALIDATE后，准备执行COMMIT BACKUP之前，T2完成了提交，那T1会怎么处理？
>
> 回答：这个等下节课举例的时候再进行说明。
>
> 问题：可串行化使我们能够对事务重新排序，或许前一个问题T1和T2，我们可以对其重新排序，避免问题描述的场景发生？
>
> **回答：这里FaRM采用的是严格的可串行化，不可对事务重排序。严格的可串行化要求，如果事务在某个提交之后开始，事务也要在那个事务之后提交，这通过协议的版本号保证。**如果T2在T1提交前开始，被认为是并发事务，T1和T2可以观察到彼此，排序前后都可以。

------

> **问题：对于事务来说，如果只是读操作，可以无锁操作？**
>
> **回答：是的，事务只读P3数据时，只有在执行阶段度数据时执行一次单边RDMA，以及在提交阶段的VALIDATE步骤执行一次单边RDMA，但没有涉及锁操作，没有任何写入，没有任何记录附加。也正是因为如此，FaRM在只读事务上能获得高性能。在只读事务中，不需要执行提交阶段的LOCK步骤。**
>
> **问题：为什么只读事务，还需要在提交阶段执行VALIDATE步骤，不是只读取值吗，为什么还需要验证版本？**
>
> **回答：因为可能另一个事务修改了当前只读事务读取的对象，而修改对象的事务提交后，当前只读事务需要能观察到最后的写入。**
>
> 追问：但是如果它们同时发生（写事务和只读事务），我们可以以任何一种方式对它们进行重排序。
>
> 回答：是的。
>
> **追问：所以在我看来，只读事务的第二次涉及RDMA的校验VALIDATE步骤，你第一次读取它，第二次看到版本应该是相同的，在我看来第二次验证似乎没有必要。（我感觉他疑问就是，按照前面说的严格串行化，如果T2在T1提交之前开始，那么T1和T2是并发事务，顺序不重要。所以只读事务和写事务如果并发运行，就没必要要求只读事务一定要读取并发的写事务的新version数据，反正读事务可以排在前面，进而不需要验证version）**
>
> **回答：你可能是对的，我之前没仔细想过这个问题。如果是只读事务（也就是说没有任何写操作），那么验证肯定是不必要的。但是当有混合事务（事务内既有写操作，也有读操作）时，你需要验证VALIDATE步骤。（我个人想了下，这里只读事务的VALIDATE是不能省略的，不然可能出现：只读事务开始前，X和Y版本都是0，但是只读事务开始时，读取到X的version0数据，而Y读取由于延迟，读取到version1的数据。假设另一个并发事务同时写X和Y，version最后都变成1，但是只读事务却读到两个不同版本的数据。不符合可串行化。按照可串行化规则，如果T1和T2并发，我们可以对T1和T2任意重排序，那么T1要么在T2之后，要么T1在T2之前，所以只读事务中X和Y要么都读取到version1，要么都读取到version0，才是正确的。）**
>
> 问题：按照上面说的，如果事务完全只有读操作，你在只读事务中希望原子地读取两个对象，在读取第一个对象之后，有另一个事务修改了另一个对象...（打断）（我猜想这个老哥疑问点，比如是 if X = 0 and Y = 0这类的只读事务逻辑，可能只读事务只做这两个读取，然后后续代码需要依照这个条件做其他事情）
>
> 回答：这里回答比较含糊，基本可以认为没有回答。（我个人认为，这个老哥的疑问是对的，这里只读事务不能删除VALIDATE步骤）
>

## 容错(Fault tolerance)]

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/v2-99cc1ea5228fa28e28396fcd3ad9b792_720w.jpg" alt="FaRM论文笔记" style="zoom:50%;" />

这里不深入讨论FaRM的容错机制，主要谈论FaRM容错实现中遇到的挑战。

**TC crashes after telling application(协调者通知应用程序后发生崩溃)**：

- **txn persist(事务必须被持久化)**，因为我们已经通知应用程序事务已提交，所以我们**不能丢失事务已完成的所有写入**。

假设在COMMIT PRIMARY之后崩溃，那么我们已经commit提交事务了，后续我们有足够的信息能恢复整个系统，确保已完成的写入能够持久化。

这里最坏的情况是B2崩溃了，即提交阶段的COMMIT BACKUP步骤执行时，B2丢失了被修改的object的记录，Primary还没有commit record，因为它在我们看到一个primary的确认后崩溃了，所以我们假设P1有提交记录，当然backup有B1的提交记录，所以这在恢复时有足够的信息。事务已经提交，有tid事务commit记录，我们有备份中的所有信息(锁、描述写事务的提交记录)，所以恢复阶段我们有足够的信息来确认事务是否已经实际提交。对于新的TC，恢复进程来决定这个事务已经提交，应该持久化。

 *（吐槽：这里容错，我个人觉得视频基本没讲清楚，草草就过了，估计是因为FaRM实在占用太多课程时间了。）*

## 小结
- Fast：高性能，尤其是只读事务无锁
- Assume few conflict：系统假设冲突少，所以采用乐观并发控制实现事务。也因为使用了单边RDMA，不需要任何服务器参与，使用乐观并发控制实现契合度更高。（论文中通过一些基准测试，表明系统本身冲突的场景很少）
- Data must fit memory：数据必须存储在内存中，这意味着如果你有一个非常大的数据库，你需要增加机器或内存。如果数据集就是太大，内存存储不下，那你应该使用传统的数据库系统，采用持久存储进行读写。
- Replication is only within data center：复制只在单个数据中心内进行。与前面讨论的Spanner截然相反，Spanner针对的是跨国家/地区/数据中心的同步事务，同步复制。
- Require fancy hardware：系统需要依赖特定类型的硬件，主要包括两个：UPS，使数据中心在安全故障中存活下来；RDMA网卡，获取高性能。

到这里为止，已经讨论完分布式系统中最具挑战性的部分，即构造容错存储系统。这节课往后的内容，会讨论与存储系统无关的其他知识。