## Java集合分析: HashMap

`HashMap`可能是我们使用最多的键值对类型的集合类了, 它的底层基于哈希表, 采用数组存储数据, 使用链表来解决哈希碰撞。在JDK1.8还引入红黑树来解决链表长度过长导致的查询速度下降问题。

> `HashMap`可能是我们在集合框架学习中, 最为困难的一个集合类了。它在面试过程中所占的比例很大, 也是最能深入提问的点之一。

`HashMap`的结构如下:

![collection_hashmap](/image/collection_hashmap.png)

### 1. HashMap构造与成员变量

`HashMap`的基本数据单元有两个, 一个是Node, 一个是TreeNode。因为`HashMap`有普通元素, 还有红黑树的元素, 所以定义两个:

```java
// 普通节点
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;
    // ...
}

// 树节点，继承自LinkedHashMap.Entry
// 这是因为LinkedHashMap是HashMap的子类，也需要支持树化
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
    TreeNode<K,V> parent;  // red-black tree links
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    TreeNode<K,V> prev;    // needed to unlink next upon deletion
    boolean red;
    // ...
}

// LinkedHashMap.Entry的实现
static class Entry<K,V> extends HashMap.Node<K,V> {
    Entry<K,V> before, after;
    Entry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next);
    }
}
```

> `TreeNode`是一个非常重要的数据单元, 后面会深入分析。

#### 1.1 成员变量

```java
// capacity初始值，为16，必须为2的次幂
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;

// capacity的最大值，为2^30
static final int MAXIMUM_CAPACITY = 1 << 30;

// load factor，是指当容量被占满0.75时就需要rehash扩容
static final float DEFAULT_LOAD_FACTOR = 0.75f;

// 链表长度到8，就转为红黑树
static final int TREEIFY_THRESHOLD = 8;

// 树大小为6，就转回链表
static final int UNTREEIFY_THRESHOLD = 6;

// 至少容量到64后，才可以转为树
static final int MIN_TREEIFY_CAPACITY = 64;

// 保存所有元素的table表
transient Node<K,V>[] table;

// 通过entrySet变量，提供遍历的功能
transient Set<Map.Entry<K,V>> entrySet;

// 下一次扩容值
int threshold;

// load factor
final float loadFactor;
```

对于上面的成员变量, 我们要思考下面几个问题:

1. 为什么`capacity`必须是2的n次幂？

2. `loadFacotr`的作用是什么？

3. 哈希表什么时候才会使用链表?

4. 链表什么时候才会转为红黑树?

#### 1.2 构造函数

`HashMap`有多个构造函数, 主要支持配置容量capacity和loadFactor, 以及从其他Map集合获取初始化数据。

```java
public HashMap(int initialCapacity, float loadFactor) {
    // ... 参数校验    
    this.loadFactor = loadFactor;
    this.threshold = tableSizeFor(initialCapacity);
}

public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}

public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
}

public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);
}
```

这里构造函数比较简单, `putMapEntries`也是依次插入元素。这里主要看下`tableSizeFor`方法:

```java
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

这个方法的作用就是**寻找距离传入的`initialCapacity`参数最近的2的n次幂。**

> 我们知道HashMap的扩容机制必须是2的n次幂, 但是用户传入的参数不一定符合规则, 所以就需要根据用户的输入, 找到比它大的最近的2次幂的值。比如用户输入13, 就把它调整为16, 输入31, 就调整为32。具体的方法实现就是`tableSizeFor`。

#### 1.3 寻找最近的2次幂

当看到`tableSizeFor`方法时, 是否感觉很迷。下面我们来解析这段代码, 首先我们先掌握几个概念:

- 在java中, int的长度是32位, 有符号int可以表示的值范围是`(-2)^31`到`2^31-1`, 其中最高位是符号位, 0表示正数, 1表示负数。
- `>>>`: 无符号右移, 也就是忽略符号位, 空位都以0补齐。
- `|`: 位或运算, 按位进行或操作, 0|1=1， 1|1=1， 0|0 = 0， 1|0=1; 逢1进1。

上面的都是基本的计算机知识, 我们知道计算机存储任何数据都是采用二进制形式, 所以假设一个int值为80的数在内存中二进制如下: (int是4个字节, 也就是32位)

> 0000 0000 0000 0000 0000 0000 0101 0000

比80大的最近的2次幂是128, 如下:

> 0000 0000 0000 0000 0000 0000 1000 0000

我们多找几组2次幂的二进制就会发现规律:

- 每个2次幂用二进制表示时, 只有一位是1, 其余位都是0(不包含符号位)
- 要找到比一个数大的2次幂(在正数范围内), 只需要将其最高位左移一位(**这里的最高位指的是从左往右第一个1出现的位置**), 其余位置0即可。

但是真实情况下, 并没有可行的方法能够进行上面的操作, 即使通过`&`操作符可以将某一位置0变成1, 但是也无法确认最高位出现的位置, **也就是说基于最高位进行操作的方式不可行。**

我们可以利用另一个数字, 那就是`2^n-1`。例如: 128-1=127的二进制形式:

> 0000 0000 0000 0000 0000 0000 0111 1111

把127与80对比一下:

>0000 0000 0000 0000 0000 0000 0101 0000 //80
>0000 0000 0000 0000 0000 0000 0111 1111 //127

可以发现, 我们只要把80从最高位起每一位全置为1, 就可以得到比80大的最近2次幂减一的数, 也就是2^n-1, 最后在执行一次+1操作即可。

具体的实现步骤如下:

假设原值就是80:

> 0000 0000 0000 0000 0000 0000 0101 0000

1. 无符号右移1位

    > 0000 0000 0000 0000 0000 0000 0010 1000

2. 第1步右移后的值与原值进行`|`或操作

    > 0000 0000 0000 0000 0000 0000 0111 1000

3. 第2步的值, 无符号右移2位

    > 0000 0000 0000 0000 0000 0000 0001 1110

4. 第3步的值与第2步的值进行`|` 操作

    > 0000 0000 0000 0000 0000 0000 0111 1110

5. 第4步的值右移4位

    > 0000 0000 0000 0000 0000 0000 0000 1110

6. 第5步的值与第4步的值进行`|` 操作, 注意此时已经是127了, 但是依然会进行下面的操作

    > 0000 0000 0000 0000 0000 0000 0111 1110

7. 第6步的值右移8位

    > 0000 0000 0000 0000 0000 0000 0000 0000

8. 第7步的值与第6步的值进行`|`操作

    > 0000 0000 0000 0000 0000 0000 0111 1110

9. 第8步的值右移16位

    > 0000 0000 0000 0000 0000 0000 0000 0000

10. 第9步的值与第8步的值进行`|`操作

    > 0000 0000 0000 0000 0000 0000 0111 1110

11. 进行+1 操作:

    > 0000 0000 0000 0000 0000 0000 0111 1111

经过上面11步操作之后, 最终找到了合适的2次幂, 那么在代码中表示就是:

```java
int n = cap - 1;
n |= n >>> 1;
n |= n >>> 2;
n |= n >>> 4;
n |= n >>> 8;
n |= n >>> 16;
return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
```

最后返回时, 为了防止溢出以及是否出现负值(也就是符号位第一位是1), 还需要进行校验。

以上就是寻找最近2次幂的方案, 也是`HashMap`中使用的方式。

> 还有人疑问, 为什么只右移到最大16位, 不右移32位? <br>
> **因为int就是32位的, 所以如果无符号右移32位, 最后肯定32位都是0, 根本就没必要在进行后面的或操作了, 对结果没有影响。**
