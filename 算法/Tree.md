# 二叉查找树（Binary Search Tree）（二叉搜索数）
- 根节点，左子树，右子树（也都是二叉查找树）
- 若左子树不空，左子树上所有节点的值**小于或等于**根节点的值；<br>若右子树不空，右子树上所有节点的值都**大于**根节点的值。
![img](https://cdn.staticaly.com/gh/propanec/propane-img@main/PicGo-img/2277972ac5db4e06a2e34ed27a35aaf4.png)

## 搜索
从根结点开始，如果查询的值与结点的值相等，那么就命中；否则，如果查询值比结点值小，就进入左儿子；如果比结点值大，就进入右儿子；如果左儿子或右儿子的指针为空，则报告找不到相应的值；
### 优点
如果B树的所有非叶子结点的左右子树的结点数目均保持差不多（平衡），那么B树的搜索性能逼近二分查找；但它比连续内存空间的二分查找的优点是，改变B树结构（插入与删除结点）不需要移动大段的内存数据，甚至通常是常数开销；

![img](https://cdn.staticaly.com/gh/propanec/propane-img@main/PicGo-img/v2-5f49cafe52a4cf9f27126a09fc8a3261_r.jpg)

### 存在的问题
B树：不同的插入删除可能导致同样的值的集合有不同的树结构，造成搜索性能下降

![img](https://cdn.staticaly.com/gh/propanec/propane-img@main/PicGo-img/v2-bbead032369bbfaa46b1e0df72691580_r.jpg)
右边也是一个B树，但它的搜索性能已经是线性的了；所以，使用B树还要考虑尽可能让B树保持左图的结构，和避免右图的结构（“平衡问题”）

# B-Tree
B（Balance）自平衡。二叉查找树的一般形式。磁盘管理系统中的目录管理，以及数据库系统中的索引组织多数都采用B-树这种数据结构。
m阶B-树：

- 树中每个结点至多有m棵子树；

- 若根结点不是叶子结点，则至少有两棵子树；

- 除根之外的所有非终端结点至少有「m/2]棵子树；

- 所有的叶子结点都出现在同一层次上，并且不带信息，通常称为失败结点（失败结点并不存在，指向这些结点的指针为空。引入失败结点是为了便千分析B-树的查找性能）；

- 所有的非终端结点最多有m- 1个关键字，结点的结构如图所示。

  ![B-树结点结构](https://cdn.staticaly.com/gh/propanec/propane-img@main/PicGo-img/63011fc1949e442fa82a78b56cb5147b.png)

其中，K为关键字，且K<sub>i</sub><K<sub>i+1</sub>；P为指向子树根结点的指针，且指针P<sub>i-1</sub>所指子树中所有结点的关键字均小于K<sub>i</sub>，P<sub>n</sub>所指子树中所有结点的关键字均大于K<sub>n</sub>。

对任一关键字 K; 而言，P<sub>i-1</sub> 相当于指向其 “ 左子树", P<sub>i+1</sub>相当于指向其 “右子树”。

![4阶B树](https://cdn.staticaly.com/gh/propanec/propane-img@main/PicGo-img/4d61235fb6b044f3842e8823bb6e7ab7.png)
