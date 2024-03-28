# 缓存一致性-Memcache

> [MIT6.824 Facebook的Memcache - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/471267649) <= 网络博主的笔记
>
> [redis探索之缓存一致性_青铜大神的博客-CSDN博客](https://blog.csdn.net/qq_22156459/article/details/125496995)
>
> 一致性是指同一时刻的请求，在缓存中的数据是否与数据库中的数据相同的。
>
> - **强一致性**：数据库更新操作与缓存更新操作是原子性的，缓存与数据库的数据在任何时刻都是一致的，这是最难实现的一致性。
> - **弱一致性**：当数据更新后，缓存中的数据可能是更新前的值，也可能是更新后的值，因为这种更新是异步的。
> - **最终一致性**：一种特殊的弱一致性，在一定时间后，数据会达到一致的状态。最终一致性是弱一致性的理想状态，也是分布式系统的数据一致性解决方案上比较推崇的。

这是一篇2013年来自Facebook的论文，memcached在网站开发中很常用。这篇论文不目的不是介绍构建系统的新思想/新概念/新方式，更多的是构建系统的实践经验的总结。

在这种情况下，可以支持10亿/s的请求量。从论文中可以学到3个lessons：

- 高性能：只使用现成的组件(`MySQL`, `memcached`)构建系统，但是整体性能很高。
- 性能和一致性之间的取舍(Tension between pref+consistency)：设计主要受性能驱动但同时想要得到某种程度的一致性。这里Facebook的应用不需要真正的线性一致性，比如新闻的消息延迟了一会后才更新并不影响业务。
- 经验法则(cautionaru tales)：，论文中列举了一些警示项。

## 网站演变

1. 少量用户，单DB

2. 多frontend，单DB
   - app code的计算周期(computation of cycles)瓶颈，cpu超载：
   - machine for DB + machines for frontend连接DB获取数据，FEs are stateless，所有数据都在DB，所以很容易容错，挂掉一台服务器只需要另起一个FE接管负载。

3. 多frontend，多DB

   - 分片或其他手段，使不同的分片在不同的DB机器上，获取DB的并行性。这里可以对keys分组，或采用cross-shard tnx分布式事务，可能需要2PL+2PC。
   - 如果从3中采用更多DB更多分片，每台服务器存储的keys更少，就会面临cross-shard tnx，增加了风险；

4. 多服务端，多DB，添加缓存 **add caching**

  - 可以从DB卸载read，只让DB进行write。FE首先从缓存读取，如果命中会得到一个快速的响应，否则从DB中读取，然后把数据安装(install)在缓存中；而写数据会直接发送给存储服务器；这非常适合read heavy workloads；

  - 系统新增缓存层(cache layer)，这里每个单独的缓存服务器被称为memcached daemon，整个缓存集群称为memcache。

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/image-20231220211354915.png" alt="image-20231220211354915" style="zoom: 33%;" />

 这里的**主要挑战**包括：

1. **DB+cache consistent** 如何保持数据库和缓存之间的一致性；
2. **Avoid DB overload** 如何保证数据库不会过载；

## 最终一致性(evantual consistency)

 Facebook这里不追求强一致性/线性一致性，他们追求的是evantual consistency：

- **顺序写入(write ordering)**：写入按照某种一致的整体顺序应用，而不会在时间问题上感到奇怪。这些由数据库完成，对于memcache层不是大问题；
- **可接受读旧数据(reads are behind is ok)**：读取可以短暂落后。
  
  例外就是要求client写后读到自身的写入(**clients read their own writes**)

  *zookeeper也是不保证不同session读到其他session的最新写入，但是要求同一个session内写后读能读到最新数据。*

## 缓存失效方案(cache invalidation)

 Facebook的基本方案是**缓存失效方案(cache invalidation plan)**。

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/202312202146484.png" alt="image-20231220214603274" style="zoom:50%;" />

**write**：在DB旁运行squeal deamon程序asynchronously，当FE将数据写入MySQL时，squeal监听MySQL的事务日志，发现有一个key被修改，就发送失效消息，删除缓存指定key的value缓存。删除的原因是确保能读到自己的写入(read our own writes)；

> <img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/202312202158541.png" alt="image-20231220215849774" style="zoom: 33%;" />
>
> SQL statements that modify authoritative state are amended to include memcache keys that need to be invalidated once the transaction commits [7]. **We deploy invalidation daemons (named `mcsqueal`) on every database.** **Each daemon inspects the SQL statements that its database commits, extracts any deletes, and broadcasts these deletes to the memcache deployment in every frontend cluster in that region.** Figure 6 illustrates this approach. We recognize that most invalidations do not delete data; indeed, only 4% of all deletes issued result in the actual invalidation of cached data.  

**read**：当另一个FE(或同一个FE)发起read，不会命中缓存，然后会从MySQL读数据，之后FE执行put，将数据安装到缓存中。

- 为什么这里FE(即应用程序)是自己将数据安装到缓存中呢？
- 这里缓存被称为**look-aside cache(旁路缓存)**，因为这里通常应用程序需要从数据库中读取的数据做一些计算，如获取页面的文本将其转换为HTML或HTML5页面，之后将HTML版本的结果存储到缓存中；或读取多个记录(请求可能有很多很多key)，聚合数据，将结果放在缓存中；
- 这里应用程序控制将什么放入缓存中，**是应用程序在控制缓存**，这给FE或应用程序增加了负担，而且DB不知道应用程序在缓存中写了什么；但这可以在放入缓存前做一些预处理，mencache并不是直接位于webserver和DB之间，而是在aside；

------

> **问题：为什么不在删除缓存后，立即更新缓存呢？**
>
> **回答：那就是所谓的更新方案(update scheme)，原则上也是可行的，但实现会更麻烦点，这里缓存、数据库、客户端之间会需要一些cooperation。比如client1和client2发请求，原本client1发送的x=1应该更早生效，client2发送的x=2应该后生效作为新值，但由于网络问题，导致client2请求先被处理，如果用更新方案，缓存x=2，后续client1请求被处理后，缓存x=1，覆盖x=2，这里缓存中的x会一直都是旧值(stale value)。当然，你可以通过修改MySQL内部一些流程，比如要求按时间戳排序处理client1和client2的请求等等。但是这里系统希望能直接用现成的组件搭建，而不是再进行复杂的修改，所以论文中选择的是失效方案。**

## 系统优化

 Facebook为了容灾，在两个地区部署了数据中心region：

- 只有primary region的storage layer接收写入，backup region只读；

- **就算是backup的FE发布了写入请求，也会请求primary的database**；之后backup的client读取自己的写入，会到backup的memcache中读取；

- primary通过squeal进程监听invalidaton message，并将日志同步到backup，backup可能将失效消息发送给它的memcache；

  可以获得好的读取性能和低延迟，但是primary和backup的缓存会有一些不同步，因为update和incalidation是异步的，不过系统不追求强一致性，所以可以接受。

 系统获取高性能的手段：

1. 分区或分片 (partition or shard data)
   - 大容量 (capacity)：一般对数据分区后，每个分区可以容纳大量数据
   - 并行度 (parallism)：分区后，不同分区数据读取可以并行
2. **复制数据 (replicate data)**
   - 热键友好 (hot key)：可以使相同键扩展到不同的memcached服务器，所有命中这个键的客户端可以分布在不同的memcahched服务器上，缓解单点热键问题；
   - 没有扩容 (capacity)：复制数据本身需要另外的硬件资源如两个数据中心，但并没有增加memcached存储的总容量，因为都存储了相同数量的数据；
3. **构造集群(cluster)**(单数据中心内replication)
   - 热键友好 (hot key)：同一个数据中心内，将mencache层复制多次，每个集群都有自己的frontend和memcache，用户负载均衡到这些集群；将单缓存服务扩展成缓存集群服务，热键由单实例存储复制到多个集群中；
   - 减少连接 (reduce connection)：避免了incast congestion问题，集群内更多的实例均摊了前端的缓存数据请求TCP连接；如果仅仅只是增加memcache分片，一个client就可能会建立很多的tcp链接；
   - 网络压力 (reduce network pressure)：很难建立具有双向带宽的网络来承担巨大的负载，这里通过复制将流量均摊到集群内的实例，整体网络压力承载量大；

如果系统中有冷键，其将被存储在多个regions，基本上什么都不做。所以Facebook还有一个**region pool，应用程序可以决定保存不是很受欢迎的键到regional pool，它们不会在时间上跨所有集群复制，你可以考虑region pool在多个集群之间共享，用于不太受欢迎的键存储。**

------

> 问题：这里如果backup发起写请求，会写到primary，但是这里backup缓存失效后，因为同步需要时间，backup可能还是会缓存到旧数据？
>
> 回答：是的，后面会说怎么处理这个问题。
>

## 保护数据库(protecting db)

> [什么是惊群，如何有效避免惊群? - 知乎 (zhihu.com)](https://www.zhihu.com/question/22756773)
>
> 惊群效应（thundering herd）是指多进程（多线程）在同时阻塞等待同一个事件的时候（休眠状态），如果等待的这个事件发生，那么他就会唤醒等待的所有进程（或者线程），但是最终却只能有一个进程（线程）获得这个时间的“控制权”，对该事件进行处理，而其他进程（线程）获取“控制权”失败，只能重新进入休眠状态，这种现象和性能浪费就叫做惊群效应。

 假设所有的memcache都失效了，db就算分片了也不能够扛下10亿/s的查询请求。

- 新建集群 (new cluster)

  假设新建了一个cluster，预想分担系统50%流量，但是new cluster的memcache还没有数据，如果50%读请求都进入new cluster，会导致流量都直接击中底层db，导致db崩溃。所以这里创建new cluster时，会填充new cluster的memcache，待预热完缓存数据后，再实际切50%流量到new cluster。

- **惊群问题 (thundering herd problem)**

  某个热键被更新后，缓存失效，多个前端请求同一个热键，得到null后都请求db查询数据，增大了数据库瞬时压力。**这里Facebook采用租约(lease)解决这个问题，即热键失效时，某个前端获得lease，负责更新该缓存，而其他前端则retry请求缓存，而不是直接查询db。**（这里lease用于解决性能问题，后续会提到同样也解决一致性问题）

- 缓存崩溃 (memcache server failed)

  如果不做处理，缓存崩溃后，会有大量请求直接发送到db，增加db压力。这里系统设置了gutter pool，用于临时处理memcache崩溃的场景。当memcache崩溃后，改成查询gutter pool，若还是没有，就查询数据库然后将数据放到gutter pool中（同样起到缓存作用，但gutter pool只是容错应急用的，非常态使用）。

## 缓存读写竞争(race)

1. 避免缓存旧值(avoid stale set problem)

   Client1执行`get(key)`，Client2执行`delete(key)`，Client1执行`put(key)`。由于有lease机制，这里C1执行get时获取lease，而C2执行delete时会使得C1的lease失效（因为更新了db，所以C2需要让key缓存失效），后面C1的put会失效(被拒绝)。**利用lease机制，避免了"17.5 保护数据库(protecting db)"提到的惊群问题，也避免了这里旧值填充的问题**。

2. 冷集群预热时缓存其他集群旧值(cold cluster)

   C1更新db值，执行`delete(key)`，C2从cold cluster读数据未命中，从另一个cluster缓存取数据，并执行`put(key)`。这里C2设置的缓存还是旧值。**这里Facebook通过two-second hold-off，推迟两秒机制解决该问题，其规定从cold cluser删除key值后，2s内不能对那个key做任何put操作**。所以这里举例的C2执行的put操作会被拒绝。这种问题只会出现在集群预热阶段，即集群新建后暂时没有数据，会从其他运行中的集群取数据（可能由于primary和backup之间同步延迟问题，导致取到旧缓存值）。因为这问题只出现在预热阶段，所以他们认为2s够用了，这也足够让其他写入传播/同步到cold database。

3. backup写后读不到值(primary/backup)

   C1在backup执行写操作，数据会写到primary的DB，随后backup的缓存失效(执行`delete(key)`)，此时C1直接`get(key)`取到空值，因为primary的数据修改通过squeal同步到backup的缓存需要时间。**这里Facebook通过remote marker解决该问题。当C1执行`delete(key)`后，标记key为"remote"，后续马上执行`get(key)`时，发现key的"remote"标记，于是从远程的primary的缓存读取值，而不是从本地读取。完成`get(key)`后，再移除key的"remote"标记。**

------

问题：即使没有lease invalidation机制，我们仍然会遵从弱一致性，你会获得在过去某个时间发生的顺序写入。但至少这能确保观察到自己的写入，对吗？

回答：lease还能确保你不会回到过去（读旧数据）。

## 小结

- 缓存的重要性(caching is vital)：正确的使用缓存，使得论文中的系统能够承载10亿/s的请求
- 分区和分片(partition and shard)：分区和分片，提高整体存储量和并行度
- 复制(replicaiton)：均摊热键访问成本
- 缓存和DB的一致性处理：利于lease、two-second hold off、remote marker等特殊技术手段，保证系统的弱一致性，主要是防止出现长期缓存旧值的问题。

------

问题：远程标记(remote marker)，他们怎么知道这是一个相关的data race。或者他们怎么决定什么时候需要使用远程标记？

回答：论文中没有明确说明，但猜测是为了保证最终一致性(eventual consistency)中的写后读能读取到刚才的最新写入。

问题：他们对get请求使用UDP，对其他请求使用TCP，这是一种业界普遍的做法吗？

回答：一般人们更喜欢用TCP，但TCP开销更大。有些人喜欢在UDP上使用他们自己的类似可靠传输协议的技术，比如QUIC等。