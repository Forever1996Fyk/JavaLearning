## Java集合基础: 跳表

> 什么是跳表?

跳表实质就是一种可以进行二分查找的有序链表, 它在原有的有序链表的上面增加了多级索引, 通过索引来实现快速查找, 不仅提高了搜索性能, 同时也提高插入和删除操作的性能。

### 1. 跳表的原理

下面是一个简单的有序链表

![skiplist1](/image/skiplist1.png)

因为链表的缺陷, 如果我们要查找3, 7, 17这几个元素, 只能从头开始遍历链表, 知道找到元素为止。

虽然是有序链表, 但是却无法使用二分查找。所以跳表的作用, 就是为了实现有序链表的二分查找功能。

我们把上面有序链表中的某些节点提取出来, 建立一个索引列表: 就变成了下面的结构:

![skiplist2](/image/skiplist2.png)

现在如果我们要查找17这个值, 就比之前快很多, 直接可以在一级索引层往后遍历即可, 只需要经过1, 6, 15就能得到17。

如果我们要查找11这个值, 如何操作?

遍历一级索引, 从1开始, 发现11 > 1, 向右到6, 发现11 > 6, 再向右遍历, 发现11 < 15, 于是再返回元素6节点, 从6往下走, 再从下面的节点6往右走到7, 发现 11 >7, 继续往右走, 找到11。

同样的, 一级索引还可以网上在提取一层, 组成二级索引, 如下:

![skiplist3](/image/skiplist3.png)

此时我们再次查找17, 只要经过6, 15就可以找打17了。

以上基本上就是跳表的核心思想了, 这是一个典型的"**空间换时间**"的算法, 通过向上提取索引增加了查找的效率。

### 2. 跳表的插入

如果我们要向上图的跳表中插入一个元素8, 如何操作?

我们根据抛硬币的方式, 来决定8这个元素要占据的层数, 
1. 节点8插入到第一层, 此时抛硬币决定, 是否加层; 如果抛硬币正面, 节点8增加到第一级索引层;
2. 再抛硬币决定, 是否继续加层, 如果是正面, 那就继续增加到第二级索引层, 如果是反面, 则结束加层。

![skiplist4](/image/skiplist4.png)

### 3. 跳表的删除

删除操作时, 首先找到各层包含要删除元素的节点, 然后直接移除每一层这个元素, 然后把它的前驱结点与后继节点双向指向即可。

比如要删除17这个值:

![skiplist5](/image/skiplist5.png)

### 4. 时间复杂度

有序单链表查询的事件复杂度为O(n), 而插入, 删除操作需要先找到对应的位置, 所以插入, 删除的事件复杂度也是O(n)。

根据上面跳表的定义, 每一级的索引都会减少k/2个元素, 其中k表示下面一级索引的元素个数, 那么整个跳表的高度就是(log n)。

所以, 跳表的时间复杂度也是O(log n)。

### 5. 空间复杂度

我们以标准的跳表来分析, 每两个元素向上提取一个元素, 那么最后额外需要的空间就是:

```txt
n/2 + (n/2)^2 + (n/2)^3 + ... + 8 + 4 + 2 = n-2
```

所以, 跳表的空间复杂度是O(n)。

### 6. 总结

1. 跳表是可以实现二分查找的有序链表;

2. 每个元素插入时随机生成它的level;

3. 最底层包含所有元素;

4. 如果一个元素出现在level(x), 那么它肯定出现在x以下的level中;

5. 每个索引节点包含2个指针, 向下和向右;

6. 调表查询, 插入, 删除的事件复杂度为O(log n) 与 平衡二叉树接近。

### 7. 为什么Redis选择使用跳表而不是红黑树来实现有序集合?

首先我们要知道, Redis的有序集合支持的操作:

1. 插入元素;

2. 删除元素;

3. 查找元素;

4. 有序输出所有元素;

5. 查找区间内所有元素。

其中前4个操作红黑树都可以完成, 而且时间复杂度与跳表一致。

但是第5个, 要查找区间的元素, 我们只要定位到两个区间的断电在最底层级的位置, 然后按顺序遍历元素就可以了, 效率很高。

而红黑树只能定位到端点后, 再从首位置开始每次都要查找后继节点, 相对来说比较耗时。

而且, 跳表实现起来很容易且易读, 红黑树实现起来比较困难, 所以Redis选择使用跳表来实现有序集合。


### 8. 跳表的应用

最直接的应用就是Java并发包下的`ConcurrentSkipListMap`, 这里就不在解析了, 可以看这位大佬写的文章, 分析的很透彻. [Java集合之ConcurrentSkipListMap源码分析](https://www.cnblogs.com/tong-yuan/p/ConcurrentSkipListMap.html)