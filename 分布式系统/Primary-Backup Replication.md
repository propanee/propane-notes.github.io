# <mark>The Design of a Practical System for Fault-Tolerant Virtual Machines</mark>
- 记录primary的操作并确保backup操作相同（Fault Tolerace）的base technology：**deterministic replay**
## Failures

- 只处理**fail-stop failures**： 故障出现时，计算机瞬间从工作变为不工作。（故障的server导致了外部可见的错误行为前能够被检测到 which are server failures that can be detected before the failing server causes an incorrect externally visible action）
- 不能很好地处理 logic bugs(主服务器上出现logic bug，副本上会同样出现)，configuration error, malicious（攻击者）
## Challenge
- 故障发生时，primary真的故障了吗？
  - 无法区分网络分区和机器故障（primary还在运行，但由于网络分区有的机器无法与primary 交互，使backup认为primary已经失效了，所以需要防止出现两个primary，
  -  **脑裂系统split-brain system**: 两个客户端子集（网络分区）访问不同的primary。最后导致整个系统内部状态产生严重的分歧（比如存储的数据、数据的版本等差异巨大）。此时如果重启整个系统，我们就不得不手动处理这些复杂的分歧状态问题（就好似手动处理git merge冲突似的）。
- 如何使主备保持同步 in sync？
  - **我们的目标是primary失败时，backup能接手primary的工作，并且从primary停止的地方继续工作。这要求backup总是能拿到primary最新写入的数据，保持最新版本**。而不会向客户端返回错误或者无法响应请求，因为从客户端角度来看，primary和backup无区别，backup就是为容错而生，理应也能正常为自己提供服务。
  - 需要**保证应用中的所有变更，按照正确顺序被处理(apply changes in order)**
  - 必须**避免/解决非决定论(avoid non-determinism)**。即相同的变更在主备上应该有完全相同的作用。

- Fail over（故障转移）
  - 主机出现故障切换到备机时，必须确保主机已经完成了所有正在执行的工作。
  - 如primary正在响应client时突然切换backup，需要弄清是否已经发到了（如果遇到网络分区等问题，会使得故障转移难上加难）


## 主备复制途径

1. **状态转移(state transfer)**：primary正常和client交互，每次响应client之前，先生成记录checkpoint检查点，将checkpoint同步到备份backup，待backup都同步完状态后，primary再响应client。	

   缺点：一个操作生成很多状态，状态转移就会很expansive。

2. **复制状态机(replicated state machine，RSM)**：primary和backup之间同步的不是状态，而是操作。即primary正常和client交互，每次响应client之前，先生成操作记录operations，将operations同步到备份backup，待backup都执行完相同的操作后，primary再响应client。

   事实上，GFS也是采用这种操作，主机发送append或write操作给备机，而不是执行追加操作然后将结果发送到备机。
   
   **关键在于，使所有的操作都是确定性的，不允许出现非确定性的操作。**

> **为什么client不需要发送数据到backup备机？**
>
> 因为这里client发送的请求是具有**确定性的操作**，只需向primary请求就够了。主备复制机制保证primary能够将具有确定性的操作正确同步到其他backup，即系统内部自动保证了primary和backup之间的一致性，不需要client额外干预。
>
> **混合的机制，即混用状态转移(state transfer)和复制状态机(replicated state machine，RSM)？**
>
> 是的。比如有的混合机制在默认情况下以复制状态机(replicated state machine，RSM)方案工作，而当集群内primary或backup故障，为此创建一个新的replica时则采用状态转移(state transfer)转移/复制现有副本的状态。

## 复制操作级别

- application-level operations（如GFS file append,write）
  - 如果你在应用程序级别的操作上使用RSM，那也实现RSM内部需要密切关注应用程序的操作细节，比如GFS的append、write操作发生时，RSM应该如何处理这些操作。一般而言需要修改应用程序本身，以执行或作为RSM的一部分。
- machine level operations（或processor level / computer level/instruction level）
  - 状态是x86寄存器、内存状态，操作是传统的计算机指令。
  - **transparent**：**这种级别下，复制状态机无感知应用程序和操作系统，只感知最底层的机器指令**，应用程序无需修改，可以透明的复制。
  - appear to C that S is a single machine。
  - 并不需要真正的硬件复制，可以使用虚拟机**(virtual machine, VM)**。

## 虚拟化实现复制（Overview）

存在的问题：论文中是一个单核心的解决方案，无法提供多核支持。

虚拟机监视器(virtual machine monitor)，或者有时也被称为hypervisor，本文中即VM-FT。

![img](https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/0929%20VMFT%E4%B8%BB%E5%A4%87.png)

### 中断

当发生中断（比如定时器中断）时，作为hypervisor的VM-FT会先捕获到中断信号，此时它会：

1. 在某个时刻传递给应用程序（上层Linux）
2. **通过日志通道将中断信号发送给备份计算机（sends it over a logging channel to a backup computer）**然后将中断信号传递到实际的虚拟机，比如运行在guest space的Linux。

![img](https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/IMG_0566.PNG)

### 网络

Client向主机发送的数据包，硬件接收数据包，传递给hypervisor，结果是中断：
1. hypervisor发送中断给本地虚拟机，正常的处理和产生回应（写入虚拟网卡，再写入真实的硬件，再由真实的硬件发送给客户端）
2. hypervisor同时也发送中断给backup，backup的linux系统也会做与主机相同的事情，但hypervisor知道这是backup，所以不会向网络发送数据包。

### 存储服务器

![img](https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/IMG_0567.PNG)

网络上有存储服务器（storage server 可以想象成是两个虚拟机的硬盘）

存储服务器实际上扮演了两个角色，挂载到主备上的磁盘与仲裁服务器

- APP需要写入文件，linux系统将数据包发送给hypervisor，通过网络发送给storage server，storage server在某个时刻发送响应。
  - 与client类似，区别在于此时linux启动通信，另一个是client启动通信。
- storage更新状态，通过flag仲裁失败后谁成为主机。
  - 假设主备之间网络分区，即logging channel断开，但主备都可以与存储通信。此时主备都想要"go live"。
  - 主备进行test-and-set操作（由hypervisor启动，读取test-and-set标志），即向flag写入1，如果flag已经被置为1，就说明有一方更早的写入。首先完成的一方被返回旧值0并继续成为主机，返回1的终止自己。

> **flag何时置为0？**
>
> - 当flag被设置为1时不需要仲裁了，因为此时只有一个机器（另一个终止了自己）
>
> - 此时会将第二个新的备机拉起存活（repair plan，人工进行，有人监控到这种情况然后创建一个新的备份，或根据原有的镜像确保同步）具体方法是用户接口上选择进行复制操作（VMware *vMotion* 虚拟机实时迁移），停止处理客户端请求，将虚拟机的状态复制到备机。
>
> - 完成后logging 重启，然后用户接口可以重置flag为0，此时系统重新开始工作，处理客户端请求。
>
> **logging断了，到服务器也断了？**此时系统停止，直到问题被修复，因为此时不知道是什么状态。

### 补充 vmotion：

[vsphere之vmotion精华 虚拟机迁移](https://blog.csdn.net/xdy762024688/article/details/132057324?spm=1001.2101.3001.6650.2&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EYuanLiJiHua%7EPosition-2-132057324-blog-130725289.235%5Ev38%5Epc_relevant_sort_base1&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EYuanLiJiHua%7EPosition-2-132057324-blog-130725289.235%5Ev38%5Epc_relevant_sort_base1&utm_relevant_index=5)

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/0930%20vmotion%E6%B5%81%E7%A8%8B%E5%9B%BE.jpeg" alt="vmotion流程图" style="zoom:67%;" />

> 1. EXSI-1拷贝VM的当前内存数据到EXSI-2中；
> 2. 由于此时VM仍在运行中，肯定会有新的数据写入，因此EXSI-1会记录内存改变（memory bitmap）。这里说的记录内存改变不是记录改变的具体内容，而已记录内存改变的内容存放的地址。
> 3. 当内存数据完全拷贝到EXSI-2后，EXSI-1中的VM会停止对外服务，保证内存不会再改变了。
> 4. EXSI-1拷贝memory bitmap到EXSI-2；
> 5. EXSI-2根据memory bitmap中的地址，去克隆对应地址中的内存数据。完成后，EXSI-2就具备和EXSI-1一模一样的内存数据了。
> 6. 由于两个EXSI是共享一个存储，因此此时VMDK可以直接移动给EXSI-2使用。相当于EXSI-2具有VM的硬盘内容了
> 7. 此时，VM就能直接在EXSI-2运行并对外提供服务了，EXSI-1中内存数据会删除以释放空间。整个过程不存在操作系统的开关机操作，是一种在线式的迁移。
> 8. VM会通过反向ARP协议告诉网络，VM的IP地址对应的MAC是在EXSI-2上了
>
> #### 实现VMOTION的前提条件
>
> 1. 各个EXSI必须共享同一个外置存储（否则无法共享VMDK硬盘文件）
> 2. 服务器必须具有相同的硬件配置，尤其是CPU必须是一样的品牌型号（CPU不一样，很多高级功能可能无法落实或速度很慢）(开启EVC)
> 3. CPU必须支持虚拟化命令，如INTEL-VT
> 4. 如没有采用分布式交换机的，所有EXSI中的vswitch必须具有一样的名称，port group
> 5. VM必须是连入物理网络的，不能在纯虚拟网络中。
> 6. VM不能对应到RAW格式磁盘机
> 7. 必须安装vmware tools
> 对于这些条件，可以人工检查，也可以在集群中启用EVC模式（其实重点是检查CPU兼容性）来自动检查。当新加入的EXSI不匹配EVC中配置时，将不会启用VMOTION



## 处理差异来源（Divergence sorces）
goal: behave like a single machine

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/1003primarybackup.png" alt="image-20231003104141073" style="zoom:33%;" />

- 理想：从同样的状态开始，指令是确定性的，最终的状态是相同的
- 挑战：可能会有差异来源（Divergence sorces）
  - non-dterministic instruction，指令返回的值不同；
  - 数据包的输入，执行处理或带来中断；
  - timer interrupts，需要中断同时进行。
    - 比如网络包输入时导致中断，primary和backup在原本CPU执行流中插入中断处理的位置可能不同。比如primary在第1～2条指令执行的位置插入网络包的中断处理，而backup在第2～3条指令执行的位置插入中断处理，这也有可能导致后续primary和backup的状态不一致。所以我们希望这里数据包产生的中断（或者时钟中断），在primary和backup相同的CPU指令流位置插入执行中断处理，确保不会产生不一致的状态。

> - 可能的差异来源**multicore** --> disallow，论文中只允许有单个处理器：如果有两个线程同时争夺锁，必须让主机的胜者也是备机的胜者，需要额外的复杂机制。线程切换实际上是单一的指令，相对来说更容易处理。
>
> - 实际上这些指令不会发送到主机或备机，它们都有自己的复制，只是在程序的统一时间点启动它们，所以它们的运行以相同的顺序执行这些命令。

### 中断处理 Interrupts

在第100个指令后执行中断I，主机查看指令号是多少，并向备机发送{100，I，data}消息，告诉备机到指令100时执行这个中断。

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/image-20231003151623121.png" alt="image-20231003151623121" style="zoom: 33%;" />

> **当接受到中断时，VM-FT能知道CPU已经执行了多少指令（比如执行了100条指令），并且计算一个位置（比如100），告知backup之后在指令执行到第100条的时候，执行中断处理程序。备机会缓冲这个消息，直到它知道下一步必须交付一些东西。大多数处理器（比如x86）支持在执行到第X条指令后停止，然后将控制权返还给操作系统（这里即虚拟机监视器）**。
>
>  通过上面的流程，VM-FT能保证primary和backup按照相同的指令流顺序执行。当然，**这里backup会落后一条message（注意这里不是落后一条instruction，而是 one message on the channel）（因为primary总是领先backup执行完需要在logging channel上传递的消息）**。

> 问题：确定性操作，就不需要通过日志通道(logging channel)同步吗？
>
> 回答：是的，不需要。因为它们都有一份所有指令的复制，只有非确定性的指令才需要通过logging channel同步。

### 非确定性指令(non-deterministic instruction)处理

>  **在启动Guest space中的Linux之前，先扫描Linux中所有的非确定性指令，确保把它们转为无效指令(invalid instruction)。当Guest space中的Linux执行这些非确定性的指令时，它将控制权通过trap交给hypervisor，此时hypervisor通过导致trap的原因能知道guest在执行非确定的指令，它会模拟这条指令的执行效果，然后记录指令模拟执行后的结果，比如记录到寄存器a0中，值为221。而backup备机上的Linux在后续某个时间点也会执行这条非确定性指令（不会真的执行），然后通过trap进入backup的hypervisor，通常backup的hypervisor会等待，直到primary将这条指令模拟执行后的结果同步给自己(backup)，然后backup就能和primary在这条非确定性指令执行上运行的效果完全相同**。

------

> 问题：扫描guest space的操作系统拥有的不确定性指令，发生在创建虚拟机时吗？
>
> 回答：是的，通常认为是在启动引导虚拟机时(when it boot to the VM)。
>
> 问题：如果这里有一条完全确定的指令，那么backup有可能执行比primary快吗？
>
> 回答：论文中有一个完整的结论，这里只需要primary和bakcup能有大致相同的执行速度（谁快一点谁慢一点影响不大），通常让primary和backup具有相同的硬件环境来实现这一点。

### VM-FT失败场景处理

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/image-20231003155455560.png" alt="image-20231003155455560" style="zoom:33%;" />

 这里举例primary故障的场景。

 比如primary本来维护一个计数器为10，client请求将其自增到11（由于是网络的请求，所以主机的VM-FT会将这个请求也发送给backup）但是primary内部自增了计数器到11，但是响应client前正好故障了。如果backup此时接手，其执行完logging channel里要求同步的指令，但是自增到11这个并没有反映到bakcup上。如果client再次请求自增计数器，其会获取到11而不是12。

 为了避免出现上诉这种场景，VM-FT指定了一套**输出规则(Output rule)：primary在output之前，必须确保前面所有通过logging channel发给备机的消息已经在备机接收到了。**
实际上上述场景并不会发生。**primary在响应client（output）之前，会通过logging channel发送消息给backup，当backup确定接受到消息且之后能执行相同的操作后，会发送ack确认消息给primary。primary收到ack后，确认backup之后能和自己拥有相同的状态（就算backup有延迟慢了一点也没事，反正backup可以通过稳定存储等手段记录这条操作，之后确保执行后未来能和primary达到相同的状态），然后才响应client**。

如果备机没有发送到ack信号，主机也没有响应client，备机此时接管。client没有收到响应，会重新发送TCP数据包给到备机，完成这次操作。

 在任何复制系统(replication system)，都能看到类似输出规则(output rule)的机制（比如在raft或者zookeeper论文中）。

## 性能

因为VM-FT的操作都基于机器指令或中断的级别上，所以需要牺牲一定的性能。

 论文中统计在primary/backup模式下运行时，性能和平时单机差异不会太大，保持在0.94~0.98的水平。而当网络输入输出流量很高时，性能下降很明显，下降将近30%。

- primary处理外部数据包时，数据包必须发送给备机，并需要等待backup也处理完毕，以向客户端发送响应。

 **正因为在指令层面(instruction level)使用复制状态机性能下降比较明显，所以现在人们更倾向于在应用层级(application level)使用复制状态机。**但是在应用层级使用复制状态机，通常需要像GFS一样修改应用程序。

> 问题：前面举例client发请求希望primary中维护的计数器自增的操作，看似确定性操作，为什么还需要通过logging channel通知bakcup？
>
> 回答：因为client请求是一个网络请求，所以需要通知backup。并且这里primary会确定backup接受到相同的指令操作回复自己ack后，才响应client。
>
> 问题：VM-FT系统的目标，是为了帮助提高服务器的性能吗？上诉这些机制看着并不简单且看着会有性能损耗。或者说它是为了帮助分发虚拟机本身？
>
> 回答：这一套方案，仅仅是为了让运行在服务器上的应用拥有更强的容错性。虽然在机器指令或中断级别上做复制比起应用级别上性能损耗高，但是这对于应用程序透明，不要求修改应用程序代码。
>
> 问题：关于输出规则(Output rule)，客户端是否可能看到相同的响应两次？
>
> 回答：可能，客户端完全有可能收到两次回复。但是论文中认为这种情况是可容忍的，因为网络本身就可能产生重复的消息，而底层TCP协议可以处理这些重复消息。
>
> 问题：如果primary宕机了几分钟，backup重新创建一个replica并通过test-and-set将storage的flag从0设置为1，自己成为新的primary。然后原primary又恢复了，这时候会怎么样？
>
> 回答：原primary会被clean，terminate自己。
>
> 问题：上面的storage除了flag以外，只需要存储其他东西吗？比如现在有多个backup的话。
>
> 回答：论文这里的方案只针对一个backup，如果你有多个backup，那么你还会遇到其他的问题，并且需要更复杂的协议和机制。
>
> 问题：处理大量网络数据包时，primary只会等待backup确认第一个数据包吗？
>
> 回答：不是，primary每处理一个数据包，都会通过logging channel同步到backup，并且等待backup返回ack。满足了输出规则(output rule)之后，primary才会发出响应。这里它们有一些方法让这个过程尽量快。
>
> 问题：关于logging channel，我看论文中提到用UDP。那如果出现故障，某个packet没有被确认，primary是不是直接认为backup失败，然后不会有任何重播？
>
> 回答：不是。因为有定时器中断（心跳是间接作用），定时器中断大概每10ms左右触发一次。如果有包接受失败，primary会在心跳中断处理时尝试重发几次给backup，如果等待了可能几秒了还是有问题，那么可能直接stop停止工作。
