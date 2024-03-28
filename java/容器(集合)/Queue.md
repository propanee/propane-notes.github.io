# Stack & Queue概述

Java里有一个叫做*Stack*的类，却没有叫做*Queue*的类(它是个接口名字)。当需要使用栈时，Java已不推荐使用*Stack*，而是推荐使用更高效的*ArrayDeque*；既然*Queue*只是一个接口，当需要使用队列时也就首选*ArrayDeque*了(次选是*LinkedList*)。

##  Queue

*Queue*接口继承自Collection接口，除了最基本的Collection的方法之外，它还支持额外的*insertion*, *extraction*和*inspection*操作。这里有两组格式，共6个方法，根据**因为容量问题而导致操作失败后处理方式的不同**可以分为两类方法: 一种在操作失败后会抛出异常，另一种则会返回特殊值。

|         | Throws exception | Returns special value |
| ------- | ---------------- | --------------------- |
| Insert  | add(e)           | offer(e)              |
| Remove  | remove()         | poll()                |
| Examine | element()        | peek()                |

##  Deque

`Deque`是"double ended queue", 表示双向的队列，英文读作"deck". Deque 继承自 Queue接口，除了支持Queue的方法之外，还支持`insert`, `remove`和`examine`操作，由于Deque是双向的，所以可以对队列的头和尾都进行操作，它同时也支持两组格式，一组是抛出异常；另外一组是返回特殊值。共12个方法如下:

|         | First Element - Head |               | Last Element - Tail |               |
| ------- | -------------------- | ------------- | ------------------- | ------------- |
|         | Throws exception     | Special value | Throws exception    | Special value |
| Insert  | addFirst(e)          | offerFirst(e) | addLast(e)          | offerLast(e)  |
| Remove  | removeFirst()        | pollFirst()   | removeLast()        | pollLast()    |
| Examine | getFirst()           | peekFirst()   | getLast()           | peekLast()    |

***Deque*的含义是“double ended queue”，即双端队列，它既可以当作栈使用，也可以当作队列使用。**当把`Deque`当做FIFO的`queue`来使用时，元素是从`deque`的尾部添加，从头部进行删除的； 所以`deque`的部分方法是和`queue`是等同的。下表列出了*Deque*与*Queue*相对应的接口:

| Queue Method | Equivalent Deque Method | 说明                                   |
| ------------ | ----------------------- | -------------------------------------- |
| `add(e)`     | `addLast(e)`            | 向队尾插入元素，失败则抛出异常         |
| `offer(e)`   | `offerLast(e)`          | 向队尾插入元素，失败则返回`false`      |
| `remove()`   | `removeFirst()`         | 获取并删除队首元素，失败则抛出异常     |
| `poll()`     | `pollFirst()`           | 获取并删除队首元素，失败则返回`null`   |
| `element()`  | `getFirst()`            | 获取但不删除队首元素，失败则抛出异常   |
| `peek()`     | `peekFirst()`           | 获取但不删除队首元素，失败则返回`null` |

下表列出了*Deque*与*Stack*对应的接口:

| Stack Method | Equivalent Deque Method | 说明                                   |
| ------------ | ----------------------- | -------------------------------------- |
| `push(e)`    | `addFirst(e)`           | 向栈顶插入元素，失败则抛出异常         |
| 无           | `offerFirst(e)`         | 向栈顶插入元素，失败则返回`false`      |
| `pop()`      | `removeFirst()`         | 获取并删除栈顶元素，失败则抛出异常     |
| 无           | `pollFirst()`           | 获取并删除栈顶元素，失败则返回`null`   |
| `peek()`     | `getFirst()`            | 获取但不删除栈顶元素，失败则抛出异常   |
| 无           | `peekFirst()`           | 获取但不删除栈顶元素，失败则返回`null` |

上面两个表共定义了*Deque*的12个接口。添加，删除，取值都有两套接口，它们功能相同，区别是对失败情况的处理不同。**一套接口遇到失败就会抛出异常，另一套遇到失败会返回特殊值(`false`或`null`)**。除非某种实现对容量有限制，大多数情况下，添加操作是不会失败的。**虽然*Deque*的接口有12个之多，但无非就是对容器的两端进行操作，或添加，或删除，或查看**。明白了这一点讲解起来就会非常简单。

事实上，`Deque` 还提供有 `push()` 和 `pop()` 等其他方法，可用于模拟栈。

## ArrayDeque

*ArrayDeque*和*LinkedList*是*Deque*的两个通用实现，由于官方更推荐使用*AarryDeque*用作栈和队列，加之上一篇已经讲解过*LinkedList*，本文将着重讲解*ArrayDeque*的具体实现。

从名字可以看出*ArrayDeque*底层通过数组实现，为了满足可以同时在数组两端插入或删除元素的需求，该数组还必须是循环的，即**循环数组(circular array)**，也就是说数组的任何一点都可能被看作起点或者终点。*ArrayDeque*是非线程安全的(not thread-safe)，当多个线程同时使用的时候，需要程序员手动同步；另外，该容器不允许放入`null`元素。

![ArrayDeque_addLast.png](https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/202312311900278.png)

上图中我们看到，**`head`指向首端第一个有效元素，`tail`指向尾端第一个可以插入元素的空位**。因为是循环数组，所以`head`不一定总等于0，`tail`也不一定总是比`head`大。

## ArrayDeque 与 LinkedList 的区别

`ArrayDeque` 和 `LinkedList` 都实现了 `Deque` 接口，两者都具有队列的功能，但两者有什么区别呢？

- `ArrayDeque` 是基于可变长的数组和双指针来实现，而 `LinkedList` 则通过链表来实现。
- `ArrayDeque` 不支持存储 `NULL` 数据，但 `LinkedList` 支持。
- `ArrayDeque` 是在 JDK1.6 才被引入的，而`LinkedList` 早在 JDK1.2 时就已经存在。
- `ArrayDeque` 插入时可能存在扩容过程, 不过均摊后的插入操作依然为 O(1)。虽然 `LinkedList` 不需要扩容，但是每次插入数据时均需要申请新的堆空间，均摊性能相比更慢。

从性能的角度上，选用 `ArrayDeque` 来实现队列要比 `LinkedList` 更好。此外，`ArrayDeque` 也可以用于实现栈。

##  方法剖析

###  addFirst()

`addFirst(E e)`的作用是在*Deque*的首端插入元素，也就是在`head`的前面插入元素，在空间足够且下标没有越界的情况下，只需要将`elements[--head] = e`即可。

![ArrayDeque_doubleCapacity.png](https://propane.oss-cn-nanjing.aliyuncs.com/typora_pic/202312311901863.png)

实际需要考虑: 1.空间是否够用，以及2.下标是否越界的问题。上图中，如果`head`为`0`之后接着调用`addFirst()`，虽然空余空间还够用，但`head`为`-1`，下标越界了。下列代码很好的解决了这两个问题。

```Java
//addFirst(E e)
public void addFirst(E e) {
    if (e == null)//不允许放入null
        throw new NullPointerException();
    elements[head = (head - 1) & (elements.length - 1)] = e;//2.下标是否越界
    if (head == tail)//1.空间是否够用
        doubleCapacity();//扩容
}
```

上述代码我们看到，**空间问题是在插入之后解决的**，因为`tail`总是指向下一个可插入的空位，也就意味着`elements`数组至少有一个空位，所以插入元素的时候不用考虑空间问题。

下标越界的处理解决起来非常简单，`head = (head - 1) & (elements.length - 1)`就可以了，**这段代码相当于取余，同时解决了`head`为负值的情况**。因为`elements.length`必需是`2`的指数倍，`elements - 1`就是二进制低位全`1`，跟`head - 1`相与之后就起到了取模的作用，如果`head - 1`为负数(其实只可能是-1)，则相当于对其取相对于`elements.length`的补码。

下面再说说扩容函数`doubleCapacity()`，其逻辑是申请一个更大的数组(原数组的两倍)，然后将原数组复制过去。过程如下图所示:

![ArrayDeque_doubleCapacity.png](/images/collection/ArrayDeque_doubleCapacity.png)

图中我们看到，复制分两次进行，第一次复制`head`右边的元素，第二次复制`head`左边的元素。

```Java
//doubleCapacity()
private void doubleCapacity() {
    assert head == tail;
    int p = head;
    int n = elements.length;
    int r = n - p; // head右边元素的个数
    int newCapacity = n << 1;//原空间的2倍
    if (newCapacity < 0)
        throw new IllegalStateException("Sorry, deque too big");
    Object[] a = new Object[newCapacity];
    System.arraycopy(elements, p, a, 0, r);//复制右半部分，对应上图中绿色部分
    System.arraycopy(elements, 0, a, r, p);//复制左半部分，对应上图中灰色部分
    elements = (E[])a;
    head = 0;
    tail = n;
}
```

### addLast()

`addLast(E e)`的作用是在*Deque*的尾端插入元素，也就是在`tail`的位置插入元素，由于`tail`总是指向下一个可以插入的空位，因此只需要`elements[tail] = e;`即可。插入完成后再检查空间，如果空间已经用光，则调用`doubleCapacity()`进行扩容。

![ArrayDeque_addLast.png](/images/collection/ArrayDeque_addLast.png)

```Java
public void addLast(E e) {
    if (e == null)//不允许放入null
        throw new NullPointerException();
    elements[tail] = e;//赋值
    if ( (tail = (tail + 1) & (elements.length - 1)) == head)//下标越界处理
        doubleCapacity();//扩容
}
```

下标越界处理方式`addFirt()`中已经讲过，不再赘述。

### pollFirst()

`pollFirst()`的作用是删除并返回*Deque*首端元素，也即是`head`位置处的元素。如果容器不空，只需要直接返回`elements[head]`即可，当然还需要处理下标的问题。由于`ArrayDeque`中不允许放入`null`，当`elements[head] == null`时，意味着容器为空。

```Java
public E pollFirst() {
    int h = head;
    E result = elements[head];
    if (result == null)//null值意味着deque为空
        return null;
    elements[h] = null;//let GC work
    head = (head + 1) & (elements.length - 1);//下标越界处理
    return result;
}
```

###  pollLast()

`pollLast()`的作用是删除并返回*Deque*尾端元素，也即是`tail`位置前面的那个元素。

```Java
public E pollLast() {
    int t = (tail - 1) & (elements.length - 1);//tail的上一个位置是最后一个元素
    E result = elements[t];
    if (result == null)//null值意味着deque为空
        return null;
    elements[t] = null;//let GC work
    tail = t;
    return result;
}
```

###  peekFirst()

`peekFirst()`的作用是返回但不删除*Deque*首端元素，也即是`head`位置处的元素，直接返回`elements[head]`即可。

```Java
public E peekFirst() {
    return elements[head]; // elements[head] is null if deque empty
}
```

### peekLast()

`peekLast()`的作用是返回但不删除*Deque*尾端元素，也即是`tail`位置前面的那个元素。

```Java
public E peekLast() {
    return elements[(tail - 1) & (elements.length - 1)];
}
```

## 说一说 PriorityQueue

`PriorityQueue` 是在 JDK1.5 中被引入的, 其与 `Queue` 的区别在于元素出队顺序是与优先级相关的，即总是优先级最高的元素先出队。

这里列举其相关的一些要点：

- `PriorityQueue` 利用了二叉堆的数据结构来实现的，底层使用可变长的数组来存储数据
- `PriorityQueue` 通过堆元素的上浮和下沉，实现了在 O(logn) 的时间复杂度内插入元素和删除堆顶元素。
- `PriorityQueue` 是非线程安全的，且不支持存储 `NULL` 和 `non-comparable` 的对象。
- `PriorityQueue` 默认是小顶堆，但可以接收一个 `Comparator` 作为构造参数，从而来自定义元素优先级的先后。

`PriorityQueue` 在面试中可能更多的会出现在手撕算法的时候，典型例题包括堆排序、求第 K 大的数、带权图的遍历等，所以需要会熟练使用才行。

## 什么是 BlockingQueue？

`BlockingQueue` （阻塞队列）是一个接口，继承自 `Queue`。`BlockingQueue`阻塞的原因是其支持当队列没有元素时一直阻塞，直到有元素；还支持如果队列已满，一直等到队列可以放入新元素时再放入。

```
public interface BlockingQueue<E> extends Queue<E> {
  // ...
}
```



`BlockingQueue` 常用于生产者-消费者模型中，生产者线程会向队列中添加数据，而消费者线程会从队列中取出数据进行处理。

[![BlockingQueue](https://camo.githubusercontent.com/563584a593285c3aeb0bc0cbd7c029e25ad002ac01fe4d5b33d1d6759859a079/68747470733a2f2f6f73732e6a61766167756964652e636e2f6769746875622f6a61766167756964652f6a6176612f636f6c6c656374696f6e2f626c6f636b696e672d71756575652e706e67)](https://camo.githubusercontent.com/563584a593285c3aeb0bc0cbd7c029e25ad002ac01fe4d5b33d1d6759859a079/68747470733a2f2f6f73732e6a61766167756964652e636e2f6769746875622f6a61766167756964652f6a6176612f636f6c6c656374696f6e2f626c6f636b696e672d71756575652e706e67)

## BlockingQueue 的实现类有哪些？

[![BlockingQueue 的实现类](https://camo.githubusercontent.com/6366b452db351c67f8c4f5a0175d4ffbd94032234f213418a119bf8a766ce8c9/68747470733a2f2f6f73732e6a61766167756964652e636e2f6769746875622f6a61766167756964652f6a6176612f636f6c6c656374696f6e2f626c6f636b696e672d71756575652d6869657261726368792e706e67)](https://camo.githubusercontent.com/6366b452db351c67f8c4f5a0175d4ffbd94032234f213418a119bf8a766ce8c9/68747470733a2f2f6f73732e6a61766167756964652e636e2f6769746875622f6a61766167756964652f6a6176612f636f6c6c656374696f6e2f626c6f636b696e672d71756575652d6869657261726368792e706e67)

Java 中常用的阻塞队列实现类有以下几种：

1. `ArrayBlockingQueue`：使用数组实现的有界阻塞队列。在创建时需要指定容量大小，并支持公平和非公平两种方式的锁访问机制。
2. `LinkedBlockingQueue`：使用单向链表实现的可选有界阻塞队列。在创建时可以指定容量大小，如果不指定则默认为`Integer.MAX_VALUE`。和`ArrayBlockingQueue`类似， 它也支持公平和非公平的锁访问机制。
3. `PriorityBlockingQueue`：支持优先级排序的无界阻塞队列。元素必须实现`Comparable`接口或者在构造函数中传入`Comparator`对象，并且不能插入 null 元素。
4. `SynchronousQueue`：同步队列，是一种不存储元素的阻塞队列。每个插入操作都必须等待对应的删除操作，反之删除操作也必须等待插入操作。因此，`SynchronousQueue`通常用于线程之间的直接传递数据。
5. `DelayQueue`：延迟队列，其中的元素只有到了其指定的延迟时间，才能够从队列中出队。
6. ……

日常开发中，这些队列使用的其实都不多，了解即可。

## ArrayBlockingQueue 和 LinkedBlockingQueue 有什么区别？

`ArrayBlockingQueue` 和 `LinkedBlockingQueue` 是 Java 并发包中常用的两种阻塞队列实现，它们都是线程安全的。不过，不过它们之间也存在下面这些区别：

- 底层实现：`ArrayBlockingQueue` 基于数组实现，而 `LinkedBlockingQueue` 基于链表实现。
- 是否有界：`ArrayBlockingQueue` 是有界队列，必须在创建时指定容量大小。`LinkedBlockingQueue` 创建时可以不指定容量大小，默认是`Integer.MAX_VALUE`，也就是无界的。但也可以指定队列大小，从而成为有界的。
- 锁是否分离： `ArrayBlockingQueue`中的锁是没有分离的，即生产和消费用的是同一个锁；`LinkedBlockingQueue`中的锁是分离的，即生产用的是`putLock`，消费是`takeLock`，这样可以防止生产者和消费者线程之间的锁争夺。
- 内存占用：`ArrayBlockingQueue` 需要提前分配数组内存，而 `LinkedBlockingQueue` 则是动态分配链表节点内存。这意味着，`ArrayBlockingQueue` 在创建时就会占用一定的内存空间，且往往申请的内存比实际所用的内存更大，而`LinkedBlockingQueue` 则是根据元素的增加而逐渐占用内存空间。