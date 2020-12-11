## Java集合分析: CopyOnWriteArrayList

`CopyOnWriteArrayList`是`ArrayList`的线程安全版本, 内部也是通过数组实现, 只是每次对数组的修改都完全拷贝一份新的数组来修改, 修改完了在替换老数组, 这样保证了只阻塞写操作, 不阻塞读操作, 实现读写分离。

### 1. CopyOnWriteArrayList结构

![CopyOnWriteArrayList](/image/CopyOnWriteArrayList.png)

CopyOnWriteArrayList实现了List, RandomAccess, Cloneable, java.io.Serializable等接口。

CopyOnWriteArrayList实现了List, 提供了基础的添加、删除、遍历等操作。

CopyOnWriteArrayList实现了RandomAccess, 提供了随机访问的能力。

CopyOnWriteArrayList实现了Cloneable, 可以被克隆。

CopyOnWriteArrayList实现了Serializable, 可以被序列化。

### 2. CopyOnWriteArrayList核心属性

```java
/** 用于修改时加锁 */
final transient ReentrantLock lock = new ReentrantLock();

/** 真正存储元素的地方，只能通过getArray()/setArray()访问 */
private transient volatile Object[] array;
```

- `CopyOnWriteArrayList`使用`ReentrantLock`, 对修改数组时加锁, 使用transient表示不自动序列化

- `array`是真正存储元素的地方, 使用`volatile`表示一个线程对数组的修改可以保证对其他线程立即可见。

> 要注意的是, `CopyOnWriteArrayList`并没有size字段。

### 3. CopyOnWriteArrayList初始化

```java
public CopyOnWriteArrayList() {
    // 所有对array的操作都是通过setArray()和getArray()进行
    setArray(new Object[0]);
}

final void setArray(Object[] a) {
    array = a;
}

public CopyOnWriteArrayList(Collection<? extends E> c) {
    Object[] elements;
    if (c.getClass() == CopyOnWriteArrayList.class)
        // 如果c也是CopyOnWriteArrayList类型
        // 那么直接把它的数组拿过来使用
        elements = ((CopyOnWriteArrayList<?>)c).getArray();
    else {
        // 否则调用其toArray()方法将集合元素转化为数组
        elements = c.toArray();
        // 这里c.toArray()返回的不一定是Object[]类型
        // 详细原因见ArrayList里面的分析
        if (elements.getClass() != Object[].class)
            elements = Arrays.copyOf(elements, elements.length, Object[].class);
    }
    setArray(elements);
}

public CopyOnWriteArrayList(E[] toCopyIn) {
    setArray(Arrays.copyOf(toCopyIn, toCopyIn.length, Object[].class));
}
```

### 4. 操作数组元素

#### 4.1 添加元素到末尾 add(E e)

```java
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    // 加锁
    lock.lock();
    try {
        // 获取旧数组
        Object[] elements = getArray();
        int len = elements.length;
        // 将旧数组元素拷贝到新数组中
        // 新数组大小是旧数组大小加1
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        // 将元素放在最后一位
        newElements[len] = e;
        setArray(newElements);
        return true;
    } finally {
        // 释放锁
        lock.unlock();
    }
}
```
这段代码并不难, 步骤如下:

1. 利用`ReentrantLock`, 加锁;

2. 获取旧元素数组;

3. 新建一个数组, 大小为原数组的长度加1, 并把原数组元素拷贝到新数组;

4. 把新添加的元素放到新数组的末尾;

5. 新数组赋值给当前`array`属性, 覆盖原数组;

6. 释放锁。

#### 4.2 获取元素 get(int index)

获取指定索引的元素, 支持随机访问, 时间复杂度为O(1)。

```java
public E get(int index) {
    // 获取元素不需要加锁
    // 直接返回index位置的元素
    // 这里是没有做越界检查的, 因为数组本身会做越界检查
    return get(getArray(), index);
}

final Object[] getArray() {
    return array;
}

private E get(Object[] a, int index) {
    return (E) a[index];
}
```

获取元素时要注意的是, 并不会有加锁操作, 所以`CopyOnWriteArrayList`只保证最终一致性。

#### 4.3 删除元素 remove(int index)

删除指定索引位置的元素

```java
public E remove(int index) {
    final ReentrantLock lock = this.lock;
    // 加锁
    lock.lock();
    try {
        // 获取旧数组
        Object[] elements = getArray();
        int len = elements.length;
        E oldValue = get(elements, index);
        int numMoved = len - index - 1;
        if (numMoved == 0)
            // 如果移除的是最后一位
            // 那么直接拷贝一份n-1的新数组, 最后一位就自动删除了
            setArray(Arrays.copyOf(elements, len - 1));
        else {
            // 如果移除的不是最后一位
            // 那么新建一个n-1的新数组
            Object[] newElements = new Object[len - 1];
            // 将前index的元素拷贝到新数组中
            System.arraycopy(elements, 0, newElements, 0, index);
            // 将index后面(不包含)的元素往前挪一位
            // 这样正好把index位置覆盖掉了, 相当于删除了
            System.arraycopy(elements, index + 1, newElements, index,
                             numMoved);
            setArray(newElements);
        }
        return oldValue;
    } finally {
        // 释放锁
        lock.unlock();
    }
}
```

1. 加锁;

2. 获取旧数组, 以及旧数组长度length, 指定索引位置元素的旧值;

3. 如果移除的是最后一位元素, 则把原数组的前length-1个元素, 拷贝到新数组中；

4. 如果移除的不是最后一位元素, 则新建一个length-1长度的数组, 并把原数组除了指定索引位置的元素全部拷贝到新数组中;

5. 释放锁, 并返回旧值。


### 5. 总结

1. `CopyOnWriteArrayList`使用`ReentrantLock`重入锁加锁, 保证线程安全;

2. `CopyOnWriteArrayList`的写操作都要先拷贝一份新数组, 在新数组中做修改, 修改完在用新数组替换旧数组, 所以空间复杂度为O(n), 性能比较低;

3. `CopyOnWriteArrayList`的读操作支持随机访问, 时间复杂度为O(1);

4. `CopyOnWriteArrayList`采用读写分离的思想, 读操作不加锁, 写操作加锁, 并且写操作占用较大内存空间, 所以适用于读多写少的场景;

5. `CopyOnWriteArrayList`只保证最终一致性, 不保证实时一致性。

### 6. CopyOnWriteArrayList为什么没有size属性?

**因为每次修改都是拷贝一份正好可以存储目标个数元素的新数组, 所以不需要size属性, 数组的长度就是集合大小, 而`ArrayList`数组的长度实际是大于集合元素的大小。**
