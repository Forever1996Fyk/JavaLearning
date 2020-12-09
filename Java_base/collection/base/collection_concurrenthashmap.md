## Java集合分析: ConcurrentHashMap

`ConcurrentHashMap`是`HashMap`的线程安全版本, 内部也是使用(数组 + 链表 + 红黑树)的结构来存储元素。

但是相比于同样线程安全的`HashTable`来说, 它的性能更好。

### 1. 前提

要学习`ConcurrentHashMap`, 必须要有并发编程的知识作为支撑。例如: `CAS`, `voltaile`, `synchronized`等。

点击学习并发相关知识 ====> 

- [synchronized关键字](/Java_base/concurrent/base/thread_synchronized.md)
- [深入解析synchronized](/Java_base/concurrent/base/thread_synchronized_deep.md)
- [深入解析volatile](/Java_base/concurrent/promote/thread_volatile.md)
- [深入解析CAS](/Java_base/concurrent/juc/thread_cas.md)
- [深入解析AQS](/Java_base/concurrent/juc/thread_aqs.md)

### 2. 构造方法

`ConcurrentHashMap`的构造方法如下:

```java
public ConcurrentHashMap() {
}

public ConcurrentHashMap(int initialCapacity) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException();
    int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
            MAXIMUM_CAPACITY :
            tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
    this.sizeCtl = cap;
}

public ConcurrentHashMap(Map<? extends K, ? extends V> m) {
    this.sizeCtl = DEFAULT_CAPACITY;
    putAll(m);
}

public ConcurrentHashMap(int initialCapacity, float loadFactor) {
    this(initialCapacity, loadFactor, 1);
}

public ConcurrentHashMap(int initialCapacity,
                         float loadFactor, int concurrencyLevel) {
    if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    if (initialCapacity < concurrencyLevel)   // Use at least as many bins
        initialCapacity = concurrencyLevel;   // as estimated threads
    long size = (long)(1.0 + (long)initialCapacity / loadFactor);
    int cap = (size >= (long)MAXIMUM_CAPACITY) ?
            MAXIMUM_CAPACITY : tableSizeFor((int)size);
    this.sizeCtl = cap;
}
```

构造方法与`HashMap`对比可以发现少了两个变量, 没有了`threshold`和`loadFactor`, 而是改用`sizeCtl`来控制, 而且只存储了集合的容量。

`sizeCtl`参数的作用如下:

1. `sizeCtl = -1`, 表示有线程正在进行初始化操作;

2. `sizeCtl`高16位存储`resizeStamp`表示扩容标识戳, 低16位存储扩容线程数加1, 即(1 + nThreads);

3. `sizeCtl = 0`, 默认值, 当开始初始化时使用的就是默认值;

4. `sizeCtl > 0`, 初始化或扩容完成后下一次的扩容门槛, 相当于`threshold`; 也就是说, 当集合中的元素数量大小达到`sizeCtl`时, 也会发生扩容机制。

### 3. 初始化哈希表(桶)数组

```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        if ((sc = sizeCtl) < 0)
            // 如果sizeCtl<0说明正在初始化或者扩容，让出CPU
            Thread.yield(); // lost initialization race; just spin
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            // 如果把sizeCtl原子更新为-1成功，则当前线程进入初始化
            // 如果原子更新失败则说明有其它线程先一步进入初始化了，则进入下一次循环
            // 如果下一次循环时还没初始化完毕，则sizeCtl<0进入上面if的逻辑让出CPU
            // 如果下一次循环更新完毕了，则table.length!=0，退出循环
            try {
                // 再次检查table是否为空，防止ABA问题
                if ((tab = table) == null || tab.length == 0) {
                    // 如果sc为0则使用默认值16
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    // 新建数组
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    // 赋值给table桶数组
                    table = tab = nt;
                    // 设置sc为数组长度的0.75倍
                    // n - (n >>> 2) = n - n/4 = 0.75n
                    // 可见这里装载因子和扩容门槛都是写死了的
                    // 这也正是没有threshold和loadFactor属性的原因
                    sc = n - (n >>> 2);
                }
            } finally {
                // 把sc赋值给sizeCtl，这时存储的是扩容门槛
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```

初始化哈希表数组时, 根据`sizeCtl`参数值来判断是否正在初始化。

1. 判断数组是否为空, 如果为空, 则进入循环;

2. 定义一个`sc`变量, 并将`sizeCtl`的值赋给`sc`; 

3. 如果`sc < 0`, 则说明正在初始化或者扩容, 则调用`Thread.yield()`方法, 让当前线程让出CPU执行权;

3. 如果`sc >= 0`, 则通过`CAS`, 尝试把`sizeCtl`的值更新为-1, 

    - 如果更新成功, 则当前线程进入初始化方法; 
    - 如果更新失败, 则表示有其他线程已经在进行初始化过程, 则重新进入循环;
    - 重新循环时, 如果还没有初始化完成或者仍在扩容, 则`sc < 0` 进入第3步, 让出CPU执行权;
    - 重新循环时, 如果数组初始化或者扩容完成, 那么此时table不为null, 则退出循环》

4. 如果`CAS`成功, 进行双重检查, 再次检查table是否为空;

5. 如果`sc > 0`, 那么创建新数组的长度就是`sc`, 如果`sc = 0`, 那么数组大小默认是16;

6. 根据第5步的数组长度`sc`, 创建新数组, 并且对`sc`进行更新, 设置`sc`为数组长度的0.75倍, `sc = n - (n >>> 2)`; (这里的`n - (n >>> 2)` 就相当于 `n - n/4` = `0.75n`);

7. 最终将`sc`的值赋给`sizeCtl`, `sizeCtl = sc`, 然后跳出循环, 返回数组。

通过上面的步骤我们可以看到, 利用`sizeCtl`的值, 既可以判断是否正在初始化或者扩容, 也可以作为数组扩容的大小。

- 利用CAS控制只有一个线程来初始化数组;

- `sizeCtl`在初始化或者扩容结束之后, 是作为扩容的门槛;

- 扩容大小是固定的原数组大小的0.75倍, 默认是16.


### 4. 添加元素

```java
public V put(K key, V value) {
    return putVal(key, value, false);
}

final V putVal(K key, V value, boolean onlyIfAbsent) {
    // key和value都不能为null
    if (key == null || value == null) throw new NullPointerException();
    // 计算hash值
    int hash = spread(key.hashCode());
    // 要插入的元素所在下标的元素个数
    int binCount = 0;
    // 死循环，结合CAS使用（如果CAS失败，则会重新取整个桶进行下面的流程）
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
            // 如果桶未初始化或者桶个数为0，则初始化桶
            tab = initTable();
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            // 如果要插入的元素所在的桶还没有元素，则把这个元素插入到这个桶中
            if (casTabAt(tab, i, null,
                    new Node<K,V>(hash, key, value, null)))
                // 如果使用CAS插入元素时，发现已经有元素了，则进入下一次循环，重新操作
                // 如果使用CAS插入元素成功，则break跳出循环，流程结束
                break;                   // no lock when adding to empty bin
        }
        else if ((fh = f.hash) == MOVED)
            // 如果要插入的元素所在的桶的第一个元素的hash是MOVED，则当前线程帮忙一起迁移元素
            tab = helpTransfer(tab, f);
        else {
            // 如果这个桶不为空且不在迁移元素，则锁住这个桶（分段锁）
            // 并查找要插入的元素是否在这个桶中
            // 存在，则替换值（onlyIfAbsent=false）
            // 不存在，则插入到链表结尾或插入树中
            V oldVal = null;
            synchronized (f) {
                // 再次检测第一个元素是否有变化，如果有变化则进入下一次循环，从头来过
                if (tabAt(tab, i) == f) {
                    // 如果第一个元素的hash值大于等于0（说明不是在迁移，也不是树）
                    // 那就是桶中的元素使用的是链表方式存储
                    if (fh >= 0) {
                        // 桶中元素个数赋值为1
                        binCount = 1;
                        // 遍历整个桶，每次结束binCount加1
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                            (ek != null && key.equals(ek)))) {
                                // 如果找到了这个元素，则赋值了新值（onlyIfAbsent=false）
                                // 并退出循环
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                // 如果到链表尾部还没有找到元素
                                // 就把它插入到链表结尾并退出循环
                                pred.next = new Node<K,V>(hash, key,
                                        value, null);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) {
                        // 如果第一个元素是树节点
                        Node<K,V> p;
                        // 桶中元素个数赋值为2
                        binCount = 2;
                        // 调用红黑树的插入方法插入元素
                        // 如果成功插入则返回null
                        // 否则返回寻找到的节点
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                value)) != null) {
                            // 如果找到了这个元素，则赋值了新值（onlyIfAbsent=false）
                            // 并退出循环
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            // 如果binCount不为0，说明成功插入了元素或者寻找到了元素
            if (binCount != 0) {
                // 如果链表元素个数达到了8，则尝试树化
                // 因为上面把元素插入到树中时，binCount只赋值了2，并没有计算整个树中元素的个数
                // 所以不会重复树化
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                // 如果要插入的元素已经存在，则返回旧值
                if (oldVal != null)
                    return oldVal;
                // 退出外层大循环，流程结束
                break;
            }
        }
        }
        // 成功插入元素，元素个数加1（是否要扩容在这个里面）
        addCount(1L, binCount);
        // 成功插入元素返回null
        return null;
    }
```

其实`ConcurrentHashMap`的`put`方法与`HashMap`比较类似, 只不过加入了`CAS`以及`synchronized`的操作。大致步骤如下:

1. 如果哈希表数组未初始化, 则初始化;

2. 如果新插入的元素下标所在位置为空, 则尝试把新元素直接插入到该数组位置;

3. 如果正在扩容, 则当前线程一起加入到扩容的过程中; 因为数组的扩容涉及到数据的迁移, 所以可以利用多个线程同时进行迁移工作; 在源码可以看到 `(fh = f.hash) == MOVED` 根据每个节点的状态可以判断, 当前容器是否正在扩容。

4. 如果新插入的元素所在下标的位置不为空, 并且此时也不再扩容, 那么则给这个下标位置加锁操作(也就是加上`synchronized`); 这里可以看到加锁的是每个数组中的元素, 也就是说只要多个线程操作的不是同一个元素, 那就是真正的并行去处理每个元素;

5. 如果当前下标位置中的元素存储结构是链表, 则遍历该链表, 判断这个key是否在链表中; 如果是, 则替换原值; 否则就在链表尾部插入元素;

6. 如果当前位置中的元素存储结构是红黑树, 也遍历这个树, 判断key是否在树中; 如果是, 则替换原值; 否则插入到红黑树中; 在插入红黑树后, 还需要判断是否调整红黑树;

7. 如果元素已经存在, 则返回旧值;

8. 如果元素不存在, 整个Map的元素个数加1, 并且检查是否需要扩容。

> 从上面的源码中我们可以看到, 添加元素操作中主要使用了 **自旋锁 + CAS + synchronized + 分段锁。** 

至于为什么使用`synchronized`, 而不使用`ReentrantLock`?

因为这里并不需要显示的调用锁的相关操作, 只需要保证线程安全性即可; 在这种情况下, 再加上`synchronized`本身就已经得到了很多的优化, 所以性能上并不比`ReentrantLock`差。

### 5. 判断是否需要扩容

根据上面`put`方法的源码可以看到, 每次添加元素后, 元素数量加1, 并判断是否达到了扩容阈值, 如果达到则进行扩容或者协助扩容。

具体扩容方法就是`addCount(1L, binCount)`:

```java
private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;
    // 这里使用的思想跟LongAdder类是一模一样的（后面会讲）
    // 把数组的大小存储根据不同的线程存储到不同的段上（也是分段锁的思想）
    // 并且有一个baseCount，优先更新baseCount，如果失败了再更新不同线程对应的段
    // 这样可以保证尽量小的减少冲突

    // 先尝试把数量加到baseCount上，如果失败再加到分段的CounterCell上
    if ((as = counterCells) != null ||
            !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        CounterCell a; long v; int m;
        boolean uncontended = true;
        // 如果as为空
        // 或者长度为0
        // 或者当前线程所在的段为null
        // 或者在当前线程的段上加数量失败
        if (as == null || (m = as.length - 1) < 0 ||
                (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
                !(uncontended =
                        U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
            // 强制增加数量（无论如何数量是一定要加上的，并不是简单地自旋）
            // 不同线程对应不同的段都更新失败了
            // 说明已经发生冲突了，那么就对counterCells进行扩容
            // 以减少多个线程hash到同一个段的概率
            fullAddCount(x, uncontended);
            return;
        }
        if (check <= 1)
            return;
        // 计算元素个数
        s = sumCount();
    }
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        // 如果元素个数达到了扩容门槛，则进行扩容
        // 注意，正常情况下sizeCtl存储的是扩容门槛，即容量的0.75倍
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                (n = tab.length) < MAXIMUM_CAPACITY) {
            // rs是扩容时的一个邮戳标识
            int rs = resizeStamp(n);
            if (sc < 0) {
                // sc<0说明正在扩容中
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                    // 扩容已经完成了，退出循环
                    // 正常应该只会触发nextTable==null这个条件，其它条件没看出来何时触发
                    break;

                // 扩容未完成，则当前线程加入迁移元素中
                // 并把扩容线程数加1
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                    (rs << RESIZE_STAMP_SHIFT) + 2))
                // 这里是触发扩容的那个线程进入的地方
                // sizeCtl的高16位存储着rs这个扩容邮戳
                // sizeCtl的低16位存储着扩容线程数加1，即(1+nThreads)
                // 所以官方说的扩容时sizeCtl的值为 -(1+nThreads)是错误的

                // 进入迁移元素
                transfer(tab, null);
            // 重新计算元素个数
            s = sumCount();
        }
    }
}
```

简单分析一下, 上面判断是否需要扩容的原理:

1. 元素个数的存储方式类似于`LongAdder`类, 存储在不同的段上, 减少不同线程同时更新size时的冲突;

2. 计算元素个数时, 把这些段的值以及baseCount相加算出总的元素个数;

3. `sizeCtl`表示扩容门槛, 扩容门槛为容量的0.75倍;

4. 扩容时`sizeCtl`高位存储扩容戳, 低位存储扩容线程数加1( 1 + nThread);

5. 其他线程添加元素后, 如果发现存在扩容, 也会加入到扩容过程。

### 6. 迁移元素

`addCount`只是用于判断是否需要进行扩容, 而真正进入扩容以及数据迁移的方法是`transfer`方法。

这个方法很长, 我也没有看懂, 只是在网上看了一下流程, 头已晕, 但是大神已经把这个流程进行简单描述了, 所以这里我们就看下基本步骤:

1. 新创建数组, 大小为原数组的2倍;

2. 创建一个`ForwordingNode`类型的节点, 把新数组存储在里面;

3. 遍历旧数组, 从n-1开始从后向前遍历, 如果遍历一次就完成了数据的迁移, 那么就替换旧数组, 并且把下一次扩容门槛`sizeCtl`设置为新数组大小的0.75倍;

4. 如果当前线程扩容完成了, 那么就把扩容线程数-1;

5. 如果旧数组i位置上的元素为null, 那就直接`ForwordingNode`中的数组并标记已迁移;

6. 如果旧数组i位置上的元素的hash值为`MOVE(-1)`, 就表示它已经是一个`ForwordingNode`节点, 该位置已经迁移了;

7. 如果上面条件都不满足, 就说明旧数组i位置上存在链表或者红黑树; 那就锁定旧数组i位置, 进行数据重新定位操作, 而这个操作与`HashMap`的扩容操作几乎一致;

8. 先判断旧数组i位置的元素是否变化(有可能其他线程先一步迁移了元素), 如果没有变化, 则判断存储元素的数据结构是链表还是红黑树;

9. 如果元素的hash值大于0, 则表示存储元素的数据结构是链表, 那就跟`HashMap`处理迁移链表的方式相同, `hash & oldTable.length`将hash值与旧数组长度进行与运算`&`, 得到的值只有两种含义:

    - 如果为0, 则表示不需要前一位置;

    - 如果大于0, 那么这个元素在新数组的位置就是当前元素的位置加上旧数组的长度。

10. 如果当前元素的类型是`TreeNode`, 那就表示存储的数据结构是红黑树, 需要重新调整红黑树; 如果红黑树的大小size小于6, 那就需要将红黑叔退化为链表。

### 7. 删除元素

删除元素与添加元素一样, 也是先找到元素所在的位置, 然后采用分段锁的思想锁住这个位置上的元素, 再进行操作。

### 8. 获取元素

获取元素的关键在于, 根据目标key所在位置的数据结构的不同, 有不同获取元素的方式, 而且获取元素不需要加锁。

- 如果是该位置上的第一个元素, 就说明找到了, 直接返回;

- 如果是树或者正在迁移元素, 那就遍历树查找元素;

- 如果是链表, 则遍历整个链表寻找元素

