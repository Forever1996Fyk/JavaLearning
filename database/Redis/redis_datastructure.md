## Redis-5中基本数据结构

用过Redis应该都知道Redis的存储数据结构。下面主要是做一些总结, 不做详细的命令讲解。

### 1. 字符串String

Redis中的字符串是一种**动态字符串**, 底层类似于Java的`ArrayList`, 是一个字符数组。Redis底层对于字符串的定义为**SDS**,
即*Simple Dynamic String*结构。

注意: Redis规定了字符串的长度不得超过512M。

### 2. 列表 List

Redis的列表相当于Java中的`LinkedList`, 所以Redis中list的查询和删除操作非常快, 时间复杂度为O(1), 但是索引定位很慢, 时间复杂度为O(n)。

利用链表的特性, 我们可以利用Redis的list结构, 实现队列和栈操作。

(这也就是为什么Redis可以实现消息队列的原因)

### 3. k-v Hash

Redis中的hash结构, 相当于`HashMap`。也是通过 **"数组+链表"**的链地址法解决哈希冲突。

### 4. 集合 Set

Redis的集合Set相当于`HashSet`, 它内部的键值对是无序, 唯一的。它的内部相当于一个特殊的HashMap, HashMap的所有value都是null。

### 5. 有序列表 Zset

这可能是Redis最具特色的数据结构了, 它相当于`SortedSet`和`HashMap`的结合, 一方面保证了内部value 的唯一性, 另一方面保证顺序性, 而且还为每个value对应一个score值, 用来代表排序的权重。

它内部的数据结构是**跳表**。 跳表的具体解析, 可以看这篇文章 [Java集合基础: 跳表](/java_base/collection/collection_skiplist.md)

zset基础操作:

```shell
> ZADD books 9.0 "think in java"
> ZADD books 8.9 "java concurrency"
> ZADD books 8.6 "java cookbook"

> ZRANGE books 0 -1     # 按 score 排序列出，参数区间为排名范围
1) "java cookbook"
2) "java concurrency"
3) "think in java"

> ZREVRANGE books 0 -1  # 按 score 逆序列出，参数区间为排名范围
1) "think in java"
2) "java concurrency"
3) "java cookbook"

> ZCARD books           # 相当于 count()
(integer) 3

> ZSCORE books "java concurrency"   # 获取指定 value 的 score
"8.9000000000000004"                # 内部 score 使用 double 类型进行存储，所以存在小数点精度问题

> ZRANK books "java concurrency"    # 排名
(integer) 1

> ZRANGEBYSCORE books 0 8.91        # 根据分值区间遍历 zset
1) "java cookbook"
2) "java concurrency"

> ZRANGEBYSCORE books -inf 8.91 withscores  # 根据分值区间 (-∞, 8.91] 遍历 zset，同时返回分值。inf 代表 infinite，无穷大的意思。
1) "java cookbook"
2) "8.5999999999999996"
3) "java concurrency"
4) "8.9000000000000004"

> ZREM books "java concurrency"             # 删除 value
(integer) 1
> ZRANGE books 0 -1
1) "java cookbook"
2) "think in java"
```