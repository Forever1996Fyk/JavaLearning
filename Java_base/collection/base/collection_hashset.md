## Java集合分析: HashSet

`HashSet`是`Set`的一种实现方式, 底层主要使用`HashMap`来确保元素不重复。

> 因为`HashMap`的key是不能重复的, 所以`HashSet`利用了这个特性。

我们先看`HashSet`中的关键属性:

```java
// 内部使用HashMap
private transient HashMap<E,Object> map;

// 虚拟对象，用来作为value放到map中
private static final Object PRESENT = new Object();
```

### 1. 初始化构造方法

```java
public HashSet() {
    map = new HashMap<>();
}

public HashSet(Collection<? extends E> c) {
    map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
    addAll(c);
}

public HashSet(int initialCapacity, float loadFactor) {
    map = new HashMap<>(initialCapacity, loadFactor);
}

public HashSet(int initialCapacity) {
    map = new HashMap<>(initialCapacity);
}

// 非public，主要是给LinkedHashSet使用的
HashSet(int initialCapacity, float loadFactor, boolean dummy) {
    map = new LinkedHashMap<>(initialCapacity, loadFactor);
}
```

从上面的构造方法源码可以看出, 每一个构造方法都是调用`HashMap`对应的构造方法, 只有最后一个构造方法有点特殊, 它不是public的, 也就是说它只能被同一个包或者子类调用, 这是`LinkedHashSet`专属方法。

### 2. 添加元素

```java
public boolean add(E e) {
    return map.put(e, PRESENT)==null;
}
```

直接调用`HashMap`的put方法, 把元素本身作为key, 把属性`PRESENT`作为value, 也就是说这个`HashMap`除了key不同, 所有的value都是一样的。

### 3. 删除元素

```java
public boolean remove(Object o) {
    return map.remove(o)==PRESENT;
}
```

直接调用`HashMap`的remove方法, 注意`HashMap`的remove有返回值, 返回的就是删除元素的value, 而`HashSet`的remove返回的boolean类型。

根据上面add方法, 可以知道, `HashMap`中的value都是一样的`PRESENT`。

### 4. 查询元素

```java
public boolean contains(Object o) {
    return map.containsKey(o);
}
```

`HashSet`是没有get方法的, 因为get方法没有意义, 它不向List可以通过index下标获取元素, 所以这里只需要检查元素是否存在的方法`contains()`, 也是直接调用`HashMap`的`containKey()`方法。


### 5. 遍历元素

```java
public Iterator<E> iterator() {
    return map.keySet().iterator();
}
```

直接调用`HashMap`的keySet的迭代器。

### 6. 总结

1. `HashSet`内部利用`HashMap`的key存储元素, 以此保证元素不重复;

2. `HashSet`是无序的, 因为`HashMap`的key是无序的;

3. `HashSet`中允许有一个null元素, 因为`HashMap`允许key为null;

4. `HashSet`是非线程安全的;

5. `HashSet`是没有get方法的。

### 7. fail-fast机制

`fail-fast`机制是java集合中的一种错误机制。

当使用迭代器迭代时, 如果发现集合有修改, 则快速失败做出响应, 抛出异常。

这种修改有可能是其它线程的修改, 也有可能是当前线程自己修改导致的, 比如迭代的过程中直接调用remove()删除元素等。

但是, 并不是java中所有集合都有`fail-fast`机制。比如, 像最终一致性的`COncurrentHashMap`, `CopyOnWriterArrayList`等都没有`fail-fast`。

那么如何实现`fail-fast`?

通过之前的学习, 会发现`HashMap`, `ArrayList`都有一个属性是`modCount`, 每次对集合的修改, 这个值都会加1, 在遍历前会记录这个值也就是赋值给`expectedModCount`变量, 在遍历中检查两者是否一致, 如果不一致就说明有修改, 则抛出异常。