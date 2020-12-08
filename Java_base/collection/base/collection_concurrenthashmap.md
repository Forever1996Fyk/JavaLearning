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

2. `sizeCtl = -(1 + nThreads)`, 表示有n个线程正在一起扩容;

3. `sizeCtl = 0`, 默认值, 当开始初始化时使用的就是默认值;

4. `sizeCtl > 0`, 初始化或扩容完成后下一次的扩容门槛, 相当于`threshold`。