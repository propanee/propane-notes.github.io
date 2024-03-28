# **写在前面**

无论是什么系统，在研发的过程中不可避免的会使用到缓存，而缓存一般来说我们不会永久存储，但是缓存的内容是有限的，**那么我们如何在有限的内存空间中，尽可能的保留有效的缓存信息呢？** 那么我们就可以使用 **`LRU/LFU算法`** ，来维持缓存中的信息的时效性。

# **LRU 详解**

## **原理**

> LRU （Least Recently Used：最近最少使用）算法在缓存写满的时候，会根据所有数据的访问记录，淘汰掉未来被访问几率最低的数据。也就是说该算法认为，**最近被访问过的数据，在将来被访问的几率最大。**

流程如下：

![图片](https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/202403211644778.png)LRU流程

假设我们有这么一块内存，一共有26个数据存储块。

1. 当我们连续插入A、B、C、...Z的时候，此时内存已经`插满`了
2. 那么当我们再插入一个6，那么此时会**将内存存放时间最久的数据A淘汰掉。**
3. 当我们**从外部读取数据C的时候，此时C就会`提到头部`**，这时候C就是最晚淘汰的了。

其实流程来说很简单。我们来拆分一下的话，不难发现这就是在**维护一个双向链表**。

## **代码实现**

定义一个存放的数据块结构

```go
type item struct {
    key   string
    value any

    // the frequency of key
    freq int
}
```

定义LRU算法的结构体

```go
type LRU struct {
    dl       *list.List // 维护的双端队列
    size     int // 当前的容量
    capacity int // 限定的容量

    storage map[string]*list.Element // 存储的key
}
```

获取某个key的value的函数，如果存在这个key，那么我们就把这个值移动到最前面`MoveToFront`，否则返回一个nil。

```go
func (c *LRU) Get(key string) any {
    v, ok := c.storage[key]
    if ok {
        c.dl.MoveToFront(v)
        return v.Value.(item).value
    }

    return nil
}
```

当我们需要put进去一些东西的时候。会分以下几个步骤

1. 是否已经存在，如果已经存在则，直接返回，并且将`key`移动到最前面。
2. 如果没有存在，但是已经是到极限容量了，就把最后一个`Back()`，淘汰掉，然后在塞入。
3. 塞入的话，是塞入到最前面`PushFront`

```go
func (c *LRU) Put(key string, value any) {
    e, ok := c.storage[key]
    if ok {
        n := e.Value.(item)
        n.value = value
        e.Value = n
        c.dl.MoveToFront(e)
        return
    }

    if c.size >= c.capacity {
        e = c.dl.Back()
        dk := e.Value.(item).key
        c.dl.Remove(e)
        delete(c.storage, dk)
        c.size--
    }

    n := item{key: key, value: value}
    c.dl.PushFront(n)
    ne := c.dl.Front()
    c.storage[key] = ne
    c.size++
}
```

以上就是LRU算法的所有内容了，那我们看一下LFU算法。

# **LFU**

## **原理**

> LFU全称是最不经常使用算法（Least Frequently Used），LFU算法的基本思想和所有的缓存算法一样，一定时期内被访问次数最少的页，在将来被访问到的几率也是最小的。

相比于LRU（Least Recently Use）算法，LFU更加注重于使用的**频率** 。**`LRU是其实可以看作是频率为1的LFU的。`**

![图片](https://mmbiz.qpic.cn/mmbiz_png/UBiaA4ibmWk2ZrYdKLUMeeTR2s7K4E1LiaMRKg9DsjTLXfkFicgiciaexVmzBibVypxxRoyEjy0qF8eILB3bibwV6btWLw/640?wx_fmt=png&from=appmsg&wxfrom=5&wx_lazy=1&wx_co=1)LFU流程1

和LRU不同的是，LFU是根据频率排序的，当我们插入的时候，一般会把新插入的放到链表的尾部，**因为新插入的一定是没有出现过的，所以频率都会是1** ， 所以会放在最后。

所以LFU的插入顺序如下：

1. 如果A没有出现过，那么就会放在双向链表的最后，依次类推，就会是Z、Y。。C、B、A的顺序放到频率为1的链表中。
2. 当我们新插入 A，B，C 那么A，B，C就会到频率为2的链表中
3. 如果再次插入A，B那么A，B会在频率为3中。C依旧在2中
4. **如果此时已经满了** ，新插入一个的话，我们会把最后一个D移除，并插入 6

![图片](https://mmbiz.qpic.cn/mmbiz_png/UBiaA4ibmWk2ZrYdKLUMeeTR2s7K4E1LiaM9cOoiceeEQ5cbLrBtUbslhoTaUPoDREQOpy4WwnianOE2lcMO2jYMbWg/640?wx_fmt=png&from=appmsg&wxfrom=5&wx_lazy=1&wx_co=1)LFU流程2

## **代码**

定义一个`LFU`的结构体：

```go
// LFU the Least Frequently Used (LFU) page-replacement algorithm
type LFU struct {
    len     int // length
    cap     int // capacity
    minFreq int // The element that operates least frequently in LFU

    // key: key of element, value: value of element
    itemMap map[string]*list.Element

    // key: frequency of possible occurrences of all elements in the itemMap
    // value: elements with the same frequency
    freqMap map[int]*list.List // 维护一个频率和list的集合
}
```

我们使用LFU算法的话，我们插入的元素就需要带上频率了

```go
func initItem(k string, v any, f int) item {
    return item{
        key:   k,
        value: v,
        freq:  f,
    }
}
```

如果我们获取某个元素，那么这个元素如果存在，就会对这个元素的频率进行加1

```go
func (c *LFU) Get(key string) any {
    // if existed, will return value
    if e, ok := c.itemMap[key]; ok {
        // the frequency of e +1 and change freqMap
        c.increaseFreq(e)
        obj := e.Value.(item)
        return obj.value
    }

    // if not existed, return nil
    return nil
}
```

增加频率

```go
func (c *LFU) increaseFreq(e *list.Element) {
    obj := e.Value.(item)
    // remove from low frequency first
    oldLost := c.freqMap[obj.freq]
    oldLost.Remove(e)
    // change the value of minFreq
    if c.minFreq == obj.freq && oldLost.Len() == 0 {
        // if it is the last node of the minimum frequency that is removed
        c.minFreq++
    }
    // add to high frequency list
    c.insertMap(obj)
}
```

插入key到LFU缓存中

1. 如果存在就对频率加1
2. 如果不存在就准备插入
3. 如果溢出了，就把最少频率的删除
4. 如果没有溢出，那么就放到最后

```go
// Put the key in LFU cache
func (c *LFU) Put(key string, value any) {
    if e, ok := c.itemMap[key]; ok {
        // if key existed, update the value
        obj := e.Value.(item)
        obj.value = value
        c.increaseFreq(e)
    } else {
        // if key not existed
        obj := initItem(key, value, 1)
        // if the length of item gets to the top line
        // remove the least frequently operated element
        if c.len == c.cap {
            c.eliminate()
            c.len--
        }
        // insert in freqMap and itemMap
        c.insertMap(obj)
        // change minFreq to 1 because insert the newest one
        c.minFreq = 1
        // length++
        c.len++
    }
}
```

插入一个新的

```go
func (c *LFU) insertMap(obj item) {
    // add in freqMap
    l, ok := c.freqMap[obj.freq]
    if !ok {
        l = list.New()
        c.freqMap[obj.freq] = l
    }
    e := l.PushFront(obj)
    // update or add the value of itemMap key to e
    c.itemMap[obj.key] = e
}
```

找到频率最少的链表，并且删除

```go
func (c *LFU) eliminate() {
    l := c.freqMap[c.minFreq]
    e := l.Back()
    obj := e.Value.(item)
    l.Remove(e)

    delete(c.itemMap, obj.key)
}
```

以上就是所有LFU的算法实现了。

总的来说，LRU更偏向时间、LFU更偏向频率。我个人觉得 LRU 也是频率为1的特殊的LFU，这也是另一种思考了，不过这两者还是有很大区别的。