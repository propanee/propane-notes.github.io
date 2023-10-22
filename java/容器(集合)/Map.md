`HashMap`：JDK1.8 之前 `HashMap` 由数组+链表组成的，数组是 `HashMap` 的主体，链表则是主要为了解决哈希冲突而存在的（“拉链法”解决冲突）。JDK1.8 以后在解决哈希冲突时有了较大的变化，当链表长度大于阈值（默认为 8）（将链表转换成红黑树前会判断，如果当前数组的长度小于 64，那么会选择先进行数组扩容，而不是转换为红黑树）时，将链表转化为红黑树，以减少搜索时间。

`LinkedHashMap`：`LinkedHashMap` 继承自 `HashMap`，所以它的底层仍然是基于拉链式散列结构即由数组和链表或红黑树组成。另外，`LinkedHashMap` 在上面结构的基础上，增加了一条双向链表，使得上面的结构可以保持键值对的插入顺序。同时通过对链表进行相应的操作，实现了访问顺序相关逻辑。

`Hashtable`：数组+链表组成的，数组是 `Hashtable` 的主体，链表则是主要为了解决哈希冲突而存在的。

`TreeMap`：红黑树（自平衡的排序二叉树）

# HashMap

https://pdai.tech/md/java/collection/java-map-HashMap&HashSet.html

```java
Map<Integer, Integer> map = new HashMap<Integer, Integer>()
map.getOrDefault(key, 0)
```

## Java7

*HashSet*和*HashMap*在java中有相同的实现，前者仅仅是对后者做了一层包装，也就是说*HashSet*里面有一个*HashMap*(适配器模式)。

特点：

- 实现了Map接口：允许放入`key`为`null`的元素，也允许插入`value`为`null`的元素；
- 除该类未实现同步外，其余跟`Hashtable`大致相同；
- **无序：**跟*TreeMap*不同，该容器不保证元素顺序，根据需要该容器可能会对元素重新哈希，元素的顺序也会被重新打散，因此不同时间迭代同一个*HashMap*的顺序可能会不同。 

- **冲突处理**：开放地址方式(Open addressing)和冲突链表方式(Separate chaining with linked lists)，**Java7采用冲突链表方式**。
  - 选择合适的哈希函数，`put()`和`get()`方法可以在常数时间内完成；
  - 对*HashMap*进行迭代时，需要遍历整个table以及后面跟的冲突链表。因此对于迭代比较频繁的场景，不宜将*HashMap*的初始大小设的过大。

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/1014%20java%20HashMap_base.png" alt="HashMap_base" style="zoom:33%;" />

- **初始容量**(inital capacity)和**负载系数**(load factor)
  - **初始容量**指定了初始`table`的大小，**负载系数**用来指定自动扩容的临界值。
  - 当`entry`的数量超过`capacity*load_factor`时，容器将自动扩容并重新哈希。
  - 对于插入元素较多的场景，将初始容量设大可以减少重新哈希的次数。
- `hashCode()`和`equals()`
  - **`hashCode()`方法决定了对象会被放到哪个`bucket`里，当多个对象的哈希值冲突时，`equals()`方法决定了这些对象是否是“同一个对象”**。
  - 所以，如果要将自定义的对象放入到`HashMap`或`HashSet`中，需要**@Override** `hashCode()`和`equals()`方法。

### 重要函数

**get**

- `get(Object key)`方法根据指定的`key`值返回对应的`value`。
  - 该方法调用了`getEntry(Object key)`得到相应的`entry`，然后返回`entry.getValue()`。因此`getEntry()`是算法的核心。 
  - 算法思想是首先通过`hash()`函数得到对应`bucket`的下标，然后依次遍历冲突链表，通过`key.equals(k)`方法来判断是否是要找的那个`entry`。
  - **时间复杂度取决于链表的长度，为 O(n)。**

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/1014%20java%20HashMap_getEntry.png" alt="HashMap_getEntry" style="zoom:33%;" />

```java
//getEntry()方法
//hash(k)&(table.length-1)等价于hash(k)%table.length，原因是HashMap要求table.length必须是2的指数，因此table.length-1就是二进制低位全是1，跟hash(k)相与会将哈希值的高位全抹掉，剩下的就是余数了。
final Entry<K,V> getEntry(Object key) {
	......
	int hash = (key == null) ? 0 : hash(key);
    for (Entry<K,V> e = table[hash&(table.length-1)];//得到冲突链表
         e != null; e = e.next) {//依次遍历冲突链表中的每个entry
        Object k;
        //依据equals()方法判断是否相等
        if (e.hash == hash &&
            ((k = e.key) == key || (key != null && key.equals(k))))
            return e;
    }
    return null;
}
```

**put**

- `put(K key, V value)`方法是将指定的`key, value`对添加到`map`里。
  - 该方法首先会对`map`做一次查找，看是否包含该元组，如果已经包含则直接返回，查找过程类似于`getEntry()`方法；
  - 如果没有找到，则会通过`addEntry(int hash, K key, V value, int bucketIndex)`方法插入新的`entry`，插入方式为**头插法**。

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/1014%20java%20HashMap_addEntry.png" alt="HashMap_addEntry" style="zoom:33%;" />

**remove**

- `remove(Object key)`的作用是删除`key`值对应的`entry`。
  - 该方法的具体逻辑是在`removeEntryForKey(Object key)`里实现的。`removeEntryForKey()`方法会首先找到`key`值对应的`entry`，然后删除该`entry`(修改链表的相应引用)。查找过程跟`getEntry()`过程类似。

<img src="https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/1014%20java%20HashMap_removeEntryForKey.png" alt="HashMap_removeEntryForKey" style="zoom: 33%;" />

## Java8 HashMap

最大的不同就是利用了红黑树，所以其由 **数组+链表+红黑树** 组成。为了降低链表查找的开销，在 Java8 中，当链表中的元素达到了 8 个时，会将链表转换为红黑树，在这些位置进行查找的时候可以降低时间复杂度为 O(logN)。

![img](https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/1014%20java-collection-hashmap8.png)

Java7 中使用 Entry 来代表每个 HashMap 中的数据节点，Java8 中使用 Node，基本没有区别，都是 key，value，hash 和 next 这四个属性，不过，Node 只能用于链表的情况，红黑树的情况需要使用 TreeNode。

### 重要函数

**put**

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

// onlyIfAbsent 如果是 true，那么只有在不存在该 key 时才会进行 put 操作
// evict 我们这里不关心
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // 第一次 put 值的时候，会触发下面的 resize()，类似 java7 的第一次 put 也要初始化数组长度
    // 第一次 resize 和后续的扩容有些不一样，因为这次是数组从 null 初始化到默认的 16 或自定义的初始容量
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // 找到具体的数组下标，如果此位置没有值，那么直接初始化一下 Node 并放置在这个位置就可以了
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);

    else {// 数组该位置有数据
        Node<K,V> e; K k;
        // 首先，判断该位置的第一个数据和我们要插入的数据，key 是不是"相等"，如果是，取出这个节点
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        // 如果该节点是代表红黑树的节点，调用红黑树的插值方法，本文不展开说红黑树
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            // 到这里，说明数组该位置上是一个链表
            for (int binCount = 0; ; ++binCount) {
                // 插入到链表的最后面(Java7 是插入到链表的最前面)
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    // TREEIFY_THRESHOLD 为 8，所以，如果新插入的值是链表中的第 8 个
                    // 会触发下面的 treeifyBin，也就是将链表转换为红黑树
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                // 如果在该链表中找到了"相等"的 key(== 或 equals)
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    // 此时 break，那么 e 为链表中[与要插入的新值的 key "相等"]的 node
                    break;
                p = e;
            }
        }
        // e!=null 说明存在旧值的key与要插入的key"相等"
        // 对于我们分析的put操作，下面这个 if 其实就是进行 "值覆盖"，然后返回旧值
        if (e != null) {
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    // 如果 HashMap 由于新插入这个值导致 size 已经超过了阈值，需要进行扩容
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

**resize**

resize() 方法用于初始化数组或数组扩容，每次扩容后，容量为原来的 2 倍，并进行数据迁移。

**get** 

- 计算 key 的 hash 值，根据 hash 值找到对应数组下标: hash & (length-1)
- 判断数组该位置处的元素是否刚好就是我们要找的，如果不是，走第三步
- 判断该元素类型是否是 TreeNode，如果是，用红黑树的方法取数据，如果不是，走第四步
- 遍历链表，直到找到相等(==或equals)的 key

