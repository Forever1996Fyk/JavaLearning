## Java集合分析: ConcurrentHashMap 相关概念与总结

### 1. ConcurrentHashMap分段锁机制

`ConcurrentHashMap`是线程安全的HashMap, 实现的思想就是使用分段加锁机制。这也是它比`HashTable`效率更高的原因。

> `HashTable`直接在添加元素或者删除元素等方法上直接加上`synchronized`关键字, 导致每次操作方法时, 只能有一个线程执行, 效率很低。

我们通过学习[ConcurrentHashMap源码分析](/Java_base/collection/base/collection_concurrenthashmap.md)可以知道, `ConcurrentHashMap`的put方法, 加锁分为两种:

1. 没有发生hash冲突时, 如果添加元素的位置是null的话, 那就使用CAS的方式来添加元素, 这里加锁的粒度是数组中的元素;

2. 如果出现了hash冲突, 添加的元素所在的位置已经存在值了, 那么有存在三种情况:

    - key相同, 那么直接用新的元素覆盖旧元素;

    - 如果数组中的元素是链表形式, 那么将新的元素插入到链表的尾部;

    - 如果数组中的元素是红黑树, 那么将新元素插入红黑树中, 并调整红黑树;

    而对于这种情况, 直接使用的是`synchronized`关键字, 锁住的对象就是当然位置的已存在的元素, 加锁的粒度与第一种情况相同。

所以我们可以得出结论, **`ConcurrentHashMap`的分段加锁机制, 其实锁住的是就是数组中的元素, 当操作数组中不同元素时, 是不会产生竞争的, 也就是会并行处理。**

> 至于为什么不用`ReentrantLock`, 之前也分析过了。一方面`synchronized`已经进行了很好的性能优化, 在特定情况下, 性能不比`ReentrantLock`差; 另一方面, 这里的操作不需要显示的操作锁, 例如获取锁, 释放锁, 所以也没必要使用`ReentrantLock`。

### 2. JDK1.8 与 JDK1.7 中 ConcurrentHashMap的分段锁区别

在JDK1.7中, `ConcurrentHashMap`使用的是`segment`锁, 继承`ReentrantLock`, 一旦初始化完成, 就不能在改变了, 虽然`segment`数组中的`HashEntry`数组是可以改变。

而JDK1.8中`ConcurrentHashMap`中废弃了`segment`锁, 直接使用了数组元素作为锁对象。

所以JDK1.8的`ConcurrentHashMap`加锁粒度比1.7更细, 并发性能更好。

### 3. 总结

1. `ConcurrentHashMap`采用 **数组 + 链表 + 红黑树**的结构存储元素;

2. `ConcurrentHashMap`相比于同样线程安全的`HashTable`, 效率更高;

3. `ConcurrentHashMap`采用的并发机制有: `synchronized`, `CAS`, `自旋锁`, `分段锁`, `volatile`;

4. `ConcurrentHashMap`中没有threshold和loadFactor这两个字段，而是采用sizeCtl来控制;

5. `sizeCtl = -1`, 表示正在进行初始化;

6. `sizeCtl = 0`, 默认值，表示后续在真正初始化的时候使用默认容量;

7. `sizeCtl > 0`, 在初始化之前存储的是传入的容量，在初始化或扩容后存储的是下一次的扩容门槛;

8. `sizeCtl = (resizeStamp << 16) + (1 + nThreads)`, 表示正在进行扩容，高位存储扩容邮戳，低位存储扩容线程数加1;

9. 更新操作时, 如果正在进行扩容, 当前线程协助扩容;

10. `ConcurrentHashMap`中不能存储key或value为null的元素;

11. 查询操作是不会加锁的, 所以`ConcurrentHashMap`不是强一致性的;

12. 元素的个数是把所有的段, 也就是`baseCount`和`CounterCell`相加起来得到的。

### 4. ConcurrentHashMap使用误区

在使用`ConcurrentHashMap`一定要注意, 否则稍有不慎, 还是会出现数据不一致问题。

```java
private static final Map<Integer, Integer> map = new ConcurrentHashMap<>();

public void unsafeUpdate(Integer key, Integer value) {
    Integer oldValue = map.get(key);
    if (oldValue == null) {
        map.put(key, value);
    }
}
```

对于上面的代码, 如果有多个线程同时调用`unsafeUpdate`方法, 那么即使使用`ConcurrentHashMap`也不能保证线程安全问题。

因为在get()获取元素之后, 在进入if判断之前, 可能有其他线程已经put这个元素了, 这个时候当前线程再调用put方法, 就把其他线程操作的元素给覆盖了, 导致了数据不一致的问题。

所以在这种情况下, 要使用`putIfAbsent()`方法, 它会保证元素不存在时才插入元素, 如下:

```java
public void safeUpdate(Integer key, Integer value) {
    map.putIfAbsent(key, value);
}
```

如果上面的oldValue不是跟null比较, 而是跟一个特定的值比较, 那么就不能使用`putIfAbsent`方法:

```java
public void unsafeUpdate(Integer key, Integer value) {
    Integer oldValue = map.get(key);
    if (oldValue == 1) {
        map.put(key, value);
    }
}
```

这是要使用`ConcurrentHashMap`提供了一个方法是`replace(key, oldValue, newValue)`, 但是要注意的是, 如果传入的newValue是null, 那么就会直接删除元素。

```java
public void safeUpdate(Integer key, Integer value) {
    map.replace(key, 1, value);
}
```