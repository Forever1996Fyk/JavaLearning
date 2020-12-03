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

>0000 0000 0000 0000 0000 0000 0101 0000 //80 <br>
>0000 0000 0000 0000 0000 0000 0111 1111 //127 <br>

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

> 还有人疑问, 为什么只右移到最大16位, 不右移32位?
**因为int就是32位的, 所以如果无符号右移32位, 最后肯定32位都是0, 根本就没必要在进行后面的或操作了, 对结果没有影响。**

> 为什么底层数组的长度总是要2的n次幂呢?

首先要知道JDK1.7之前的`HashMap`中的hash算法是:

```java
hash & (length - 1)
```

其中length就是数组长度, hash就是插入值的hash值, 如果length=2^n的话, 那么length-1 = 2^n-1。

我们通过上面的示例分析可以看到, 2^n-1用二进制表示, 所有位一定都是1, 此时跟hash值进行`&`与运算, 则可以完整保留原来的hash值的二进制数据。这样基本上很少会出现hash碰撞, 因为只要插入的key不相同, 基本上就不会出现hash碰撞。(除非hash值比较相近)

相反的, 如果数组长度不是2的n次幂, 则出现hash碰撞的可能性会大大提高。如下图示例:

![collection_hashmap_tablesize2n](/image/collection_hashmap_tablesize2n.png)

### 2. HashMap核心方法

无论是`List`还是`Map`, 最重要的操作都是增删改查部分, 对于HashMap来说, 添加元素是最重要也是非常复杂的功能之一。

#### 2.1 put添加元素

`put`的源码如下:

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

`HashMap`中使用的`hash`函数很简单, 就是把**key与其高16位进行异或操作**。这是为了尽量避免"低位不变, 高位变化"是造成的hash冲突。

> 因为没有完美的哈希算法可以彻底避免碰撞, 所以只能尽可能减少碰撞。

`putVal()`方法具体源码:

```java
// 参数onlyIfAbsent表示是否替换原值
// 参数evict我们可以忽略它，它主要用来区别通过put添加还是创建时初始化数据的
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // 空表，需要初始化
    if ((tab = table) == null || (n = tab.length) == 0)
        // resize()不仅用来调整大小，还用来进行初始化配置
        n = (tab = resize()).length;
    // (n - 1) & hash这种方式也熟悉了吧？都在分析ArrayDeque中有体现
    //这里就是看下在hash位置有没有元素，实际位置是hash % (length-1)
    if ((p = tab[i = (n - 1) & hash]) == null)
        // 将元素直接插进去
        tab[i] = newNode(hash, key, value, null);
    else {
        //这时就需要链表或红黑树了
        // e是用来查看是不是待插入的元素已经有了，有就替换
        Node<K,V> e; K k;
        // p是存储在当前位置的元素
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p; //要插入的元素就是p，这说明目的是修改值
        // p是一个树节点
        else if (p instanceof TreeNode)
            // 把节点添加到树中
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            // 这时候就是链表结构了，要把待插入元素挂在链尾
            for (int binCount = 0; ; ++binCount) {
                //向后循环
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    // 链表比较长，需要树化，
                    // 由于初始即为p.next，所以当插入第9个元素才会树化
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                // 找到了对应元素，就可以停止了
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                // 继续向后
                p = e;
            }
        }
        // e就是被替换出来的元素，这时候就是修改元素值
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            // 默认为空实现，允许我们修改完成后做一些操作
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    // size太大，达到了capacity的0.75，需要扩容
    if (++size > threshold)
        resize();
    // 默认也是空实现，允许我们插入完成后做一些操作
    afterNodeInsertion(evict);
    return null;
}
```

我们在整理一下`putVal`方法的整体流程:

1. 当插入第一个元素时, 此时哈希表table如果是空表就会调用`resize()`来初始化;(除非构造HashMap时, 传入一个Map集合);

2. 判断当前传入的hash位置有没有元素, 实际位置就是`hash&(length-1)`。

3. 如果当前位置没有元素, 那就将元素直接插入进去;

4. 如果当前位置存在元素, 再判断当前位置存储的元素key值与传入的key值相等, 那就直接覆盖原来的value;

5. 如果当前位置存储的元素key值是否与传入的key值不相等, 再判断当前位置存储的元素是否是一个树节点`TreeNode`, 如果是, 就说明此处存储结构是红黑树, 就把节点添加到红黑树中;

6. 如果当前存储元素不是`TreeNode`类型, 就表示存储结构是链表, 直接要新的元素插入到链尾;

7. 在第6步时, 遍历链表插入新的元素时 比较每一个元素, 判断新元素key是否存在, 如果存在就直接覆盖value, 否则就插入链尾; 如果此时链表长度较长达到了8, 那就将整个链表进行树化;

8. 插入元素成功后, `HashMap`的容量size+1, 如果此时的size大于**`capacity*loadFactor`**, 那么就需要扩容, 再次调用`resize()`方法。

![collection_hashmap_put](/image/collection_hashmap_put.png)

上面的就是`put`方法的基本流程原理, 我们发现在插入元素时, 如果此时存储结构是红黑树时就调用`TreeNode.putTreeVal`方法向红黑树中插入元素。

而其中当插入元素后, 如果链表的长度大于8时, 链表就会树化, 调用方法为`treeifyBin`。具体的方法如下:

```java
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    //如果表是空表，或者长度还不到树化的最小值，就需要重新调整表了
    // 这样做是为了防止最初就进行树化
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        TreeNode<K,V> hd = null, tl = null;
        // while循环的目的是把链表的每个节点转为TreeNode
        do {
            // 根据当前元素，生成一个对应的TreeNode节点
            TreeNode<K,V> p = replacementTreeNode(e, null);
            //挂在红黑树的尾部，顺序和链表一致
            if (tl == null)
                hd = p;
            else {
                p.prev = tl;
                tl.next = p;
            }
            tl = p;
        } while ((e = e.next) != null);
        if ((tab[index] = hd) != null)
            // 这里也用到了TreeNode的方法，我们在最后一起分析
            // 通过头节点调节TreeNode
            // 链表数据的顺序是不符合红黑树的，所以需要调整
            hd.treeify(tab);
    }
}
```

从上面的代码看出, 无论是`put`还是`treeify`, 都依赖于`resize`方法, 所以`resize`很重要。

> `resize`不仅可以调整大小, 还能调整树化和反树化(从树变为链表)所带来的的影响。

`resize`源码如下:

```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        // 大小超过了2^30
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 扩容，扩充为原来的2倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                    oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    // 原来的threshold设置了
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    else {               // zero initial threshold signifies using defaults
        // 全部设为默认值
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                    (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
     // 扩容完成，现在需要进行数据拷贝，从原表复制到新表
    @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)
                    // 这是只有一个值的情况
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    // 重新规划树，如果树的size很小，默认为6，就退化为链表
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    // 处理链表的数据
                    // loXXX指的是在原表中出现的位置
                    Node<K,V> loHead = null, loTail = null;
                    // hiXXX指的是在原表中不包含的位置
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        //这里把hash值与oldCap按位与。
                        //oldCap是2的次幂，所以除了最高位为1以外其他位都是0
                        // 和它按位与的结果为0，说明hash比它小，原表有这个位置
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    // 挂在原表相应位置
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    // 挂在后边
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

我们可以看到`resize`的代码逻辑比较复杂, 首先我们要知道扩容的时机:

**当`HashMap`中包含的`Entry`的数量大于等于`threshold = loadFactor * capacity`的时候, 此时触发扩容机制, 也就是调用`resize`方法, 将其容量扩大为原来的2倍(这里的原因跟之前说的数组长度必须是2的n次幂一样)。**

那么对`resize`的执行步骤如下:

1. 在`resize`中, 定义了`oldCap`参数, 记录原table的长度; 定义一个`newCap`参数, 记录新table的长度,将新table的长度扩展到原来的2倍(`newCap = oldCap << 1`), 同时扩展长度也变为原来的2倍(`newThr = oldThr << 1`);

2. 当扩容大小确定时, 遍历原来的table, 将原table中的每个元素放入到新table中;

3. 如果`e.next == null`, 也就是说不存在链表以及红黑树的情况下, 直接把原来的元素放入到新table中, 其中`e.hash & (newCap - 1)`就是计算原来的元素e在新table中的位置;

4. 如果`e instanceof TreeNode`, 就表示此时该位置上存在红黑树, 那就需要重新调整红黑树的结构, 如果树的size很小, 默认是6, 那就将红黑树退化成链表;

5. 如果该位置的数据结构是链表, 那就重新计算链表中元素的hash值并分配到对应位置。

    > 由于扩容数组的长度是原来的2倍, 所以假设初始oldCap=4, 要扩容到newCap=8时, 在二级制中就是 0100 到 1000的变化, 也就是左移一位就是2倍，**所以扩容时, 如果遇到链表结构时, 只需要判断原来的hash值与原来的table长度进行`&`与运算**

#### 2.2 remove删除元素

`remove`源码如下:

```java
public V remove(Object key) {
    Node<K,V> e;
    return (e = removeNode(hash(key), key, null, false, true)) == null ?
        null : e.value;
}
```

和`put`方法一样, 实际的操作在`removeNode`方法中完成。

```java
// matchValue是说只有value值相等时候才可以删除，我们是按照key删除的，所以可以忽略它。
// movable是指是否允许移动其他元素，这里是和TreeNode相关的
final Node<K,V> removeNode(int hash, Object key, Object value,
                           boolean matchValue, boolean movable) {
    Node<K,V>[] tab; Node<K,V> p; int n, index;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (p = tab[index = (n - 1) & hash]) != null) {
        Node<K,V> node = null, e; K k; V v;
        // 不同情况下获取待删除的node节点
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            node = p;
        else if ((e = p.next) != null) {
            if (p instanceof TreeNode)
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else {
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key ||
                            (key != null && key.equals(k)))) {
                        node = e;
                        break;
                    }
                    p = e;
                } while ((e = e.next) != null);
            }
        }
        if (node != null && (!matchValue || (v = node.value) == value ||
                                (value != null && value.equals(v)))) {
            if (node instanceof TreeNode)
                // TreeNode删除
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            else if (node == p)
                // 在队首，直接删除
                tab[index] = node.next;
            else
                // 链表中删除
                p.next = node.next;
            ++modCount;
            --size;
            // 默认空实现，允许我们删除节点后做些处理
            afterNodeRemoval(node);
            return node;
        }
    }
    return null;
}
```

#### 2.3 get获取元素

查询的`get`方法是通过`getNode`方法完成的, 源码如下:

```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}

final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        if ((e = first.next) != null) {
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

`get`方法的逻辑与`put`很像, 也是判断元素是否存在哈希表中, 还是链表, 还是红黑树。

### 3. TreeNode源码分析

我们通过上面分析`HashMap`的核心方法时, 会发现增删查中涉及到红黑树的操作都是通过`TreeNode`来实现的。所以我们看下`TreeNode`的具体实现:

`TreeNode`算上其继承的成员变量, 共有11个:

```java
// 继承的变量, 也就是HashMap.Node
final int hash;
final K key;
V value;
Node<K,V> next;

// TreeNode内部的变量
TreeNode<K,V> parent;  // red-black tree links
TreeNode<K,V> left;
TreeNode<K,V> right;
TreeNode<K,V> prev;    // needed to unlink next upon deletion
boolean red;
```

在`put`方法添加元素时, 将元素插入到红黑树用到了`TreeNode.putTreeVal`:

```java
final TreeNode<K,V> putTreeVal(HashMap<K,V> map, Node<K,V>[] tab,
                                int h, K k, V v) {
    Class<?> kc = null;
    boolean searched = false;
    // 获取到root节点
    TreeNode<K,V> root = (parent != null) ? root() : this;
    for (TreeNode<K,V> p = root;;) {
        // dir表示查询方向
        int dir, ph; K pk;
        // 要插入的位置在树的左侧
        // 树化会依据key的hash值
        if ((ph = p.hash) > h)
            dir = -1;
        // 要插入的位置在树的右侧
        else if (ph < h)
            dir = 1;
        else if ((pk = p.key) == k || (k != null && k.equals(pk)))
            return p; //找到了，替换即可

        // comparableClassFor是如果key实现了Comparable就返回具体类型，否则返回null
        // compareComparables是比较传入的key和当前遍历元素的key
        // 只有当前hash值与传入的hash值一致才会走到这里
        else if ((kc == null &&
                    (kc = comparableClassFor(k)) == null) ||
                    (dir = compareComparables(kc, k, pk)) == 0) {
            if (!searched) {
                TreeNode<K,V> q, ch;
                //左右都查过了
                searched = true;
                // 通过hash和Comparable都找不到，只能从根节点开始遍历
                if (((ch = p.left) != null &&
                        (q = ch.find(h, k, kc)) != null) ||
                    ((ch = p.right) != null &&
                        (q = ch.find(h, k, kc)) != null))
                    return q;
            }
            // 元素的hashCode一致，且没有实现Comparable，在树里也没有
            // tieBreakOrder则是调用System.identityHashCode(Object o)来进行比较，
            //它的意思是说不管有没有覆写hashCode，都强制使用Object类的hashCode
            // 这样做，是为了保持一致性的插入
            dir = tieBreakOrder(k, pk);
        }
        
         // 代码执行到这，说明没有找到元素，也就是需要新建并插入了
        TreeNode<K,V> xp = p;
        // 经历过上述for循环，p已经到某个叶节点了
        if ((p = (dir <= 0) ? p.left : p.right) == null) {
            Node<K,V> xpn = xp.next;
            TreeNode<K,V> x = map.newTreeNode(h, k, v, xpn);
            if (dir <= 0)
                xp.left = x;
            else
                xp.right = x;
            xp.next = x;
            x.parent = x.prev = xp;
            if (xpn != null)
                ((TreeNode<K,V>)xpn).prev = x;
            
            // moveRootToFront目的很明确也是必须的。
            // 因为这个红黑树需要挂在数组的某个位置，所以其首个元素必须是root
            // balanceInsertion是因为插入元素后可能不符合红黑树定义了
            // 这部分知识在分析TreeMap中有详细介绍
            // 需要了解的话可以查看文末链接
            moveRootToFront(tab, balanceInsertion(root, x));
            return null;
        }
    }
}
```

看到这, 估计很多人都蒙了。虽然上面的代码, 都有注释, 但是还是很难理解, 所以我们配合一些简单的示意图还理解源码。

### 4. 图解put方法过程

我们以put方法为例:

在初始化`HashMap`时, 内部会定义table表`Node<K, V>[] table`。假设插入的数据的key在table表相同位置的hash值都一致, 并且都实现了`Comparable`接口。那么`Comparable`按照key的自然顺序比较, 下图表中的数字都表示key。

![collection_hashmap_table](/image/collection_hashmap_table.png)

上图的左侧是hash算法(`hash&(length-1)`)完成后的hash值, 也就对应着哈希表的下标, 表中的数字是key值, 有的位置还没有数据, 有的位置已经插入一些数据, 并且由于hash冲突变成了链表, 假设`capacity`已经大于64(**64是可以树化的阈值**)

1. 此时我们向表中插入一个hash=6的新元素。由于6的位置是空的, 所以元素直接插入表中:

    ![collection_hashmap_table1](/image/collection_hashmap_table1.png)

2. 我们继续插入一个hash=6的值, 由于6的位置已经存在一个元素, 于是在比较key值是否相同, 如果相同那就是直接覆盖原值, 否则新的元素会通过链表的方式插入到18的后面, 如下:

    ![collection_hashmap_table2](/image/collection_hashmap_table2.png)

3. 我们在插入几个hash=6的元素, 直接让其达到链表变为红黑树的阈值(默认是8):

    ![collection_hashmap_table3](/image/collection_hashmap_table3.png)

4. 此时6的位置上已经存在8个元素了, 然后我们在插入一个hash=6的元素, 就需要进行树化, 用红黑树代替链表来提升查询性能。

    > 在进行树化时, 先获取第一个元素18, 将其转为`TreeNode`节点, 并设置为head。然后把后续节点遍历转为`TreeNode`节点, 并依次插入到head之后, 也就是它们的**prev域**指向前一个元素, **next域**指向后一个元素。

    ![collection_hashmap_table4](/image/collection_hashmap_table4.png)

5. 转为树节点之后, 需要通过head, 也就是上图的18, 来调整为红黑树结构。

    > 1. 首先, 18就是作为root节点, 所以红黑树根节点颜色设置为黑色;<br>

    > 2. 然后比较18与20, 它们的hash值都一样, 所以会采用`Comparable`来比较它们的key值, 此时20应该是18的右孩子, 而且20节点的颜色是红色, 符合红黑树的规则;<br>

    ![collection_hashmap_redblacktree1](/image/collection_hashmap_redblacktree1.png)

    > 3. 再调整31, 31在20的右侧, 如下:

    ![collection_hashmap_redblacktree2](/image/collection_hashmap_redblacktree2.png)

    > 4. 经过第3步时, 已经破坏了红黑树, 根据红黑树的规则, 进行调整, 这里不再展示过程了。

    ![collection_hashmap_redblacktree3](/image/collection_hashmap_redblacktree3.png)

    > 5. 如果这里仅仅是一个红黑树, 那么就已经调整完毕了, 但是**我们知道这个红黑树是挂在hash表中的, 所以其根节点必须要在首位。也就是说根节点必须要在table的位置上。**经过上面的操作, 此时根节点已经从18变成了20, 所以我们需要把新的root与原root在table表中的位置进行交换, 如下:

    ![collection_hashmap_redblacktree4](/image/collection_hashmap_redblacktree4.png)

    > 6. 我们按照上面的规则, 不断的调整元素, 最终的红黑树结果如下:

    ![collection_hashmap_redblacktree5](/image/collection_hashmap_redblacktree5.png)

### 5. 总结

`HashMap`应该是目前位置最复杂的一个集合类了。不仅要深入的理解红黑树的结构, 还需要了解`HashMap`每一个阶段发生的原因, 以及对应的操作。

对于使用`HashMap`注意下面几点, 可以尽量的保证`HashMap`的优势:

1. 设计key对象一定要实现`hashcode`方法, 并尽可能保证均匀少重复;

2. 由于树化过程会依次通过 比较 **`hash`值, key值** 进行排序, 所以key还可以实现`Comparable`, 以方便树化排序;

3. 如果可以预先估计数量级, 可以指定初始容量`iniyial capacity`, 来减少`resize`的过程;

4. 虽然`HashMap`引入了红黑树, 但它的使用是很少的, 如果大量出现红黑树, 说明数据本身设计的不合理, 我们应该从数据源寻找优化方案。