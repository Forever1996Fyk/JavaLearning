## Java集合源码: LinkedList

之前学习了`List`与`Queue`, 这篇终于到了`LinkdedList`。

`LinkedList`弥补了`ArrayList`增删较慢的问题, 但是在查找方面比`ArrayList`效率低, 所以在使用时需要根据应用场景灵活选择。

> `LinkedList`既实现了`List`, 又实现了`Deque`。这就说明了, `LinkedList`能够像`ArrayList`一样使用, 又能够实现队列的功能。

`LinkedList`内部结构是一个双向链表, 就是扩充单链表的指针域, 增加一个指向前一个元素的指针**previous**。

### 1. AbstractSequentialList

`AbstractSquentialList`是`LinkedList`的父类, 它继承`AbstractList`, 并且是一个抽象类, 它主要为顺序表的链式实现提供一个骨架。

这里用的**模板方法模式**, 提供一个实现`List`接口, 并且实现了大部分的方法, 来减少子类基于链式存储的代码量。

最主要是提供一个方法的默认实现, 并且提供一个抽象的方法交给子类去实现。

```java
public void add(int index, E element) {
    try {
        listIterator(index).add(element);
    } catch (NoSuchElementException exc) {
        throw new IndexOutOfBoundsException("Index: "+index);
    }
}

public abstract ListIterator<E> listIterator(int index);
```

其他的方法都利用这个抽象方法, 让子类去实现类不同的逻辑, 但是大致的算法结果在父类中已经成形了。

### 2. LinkedList结构

![collection_linkedlist](/image/collection_linkedlist.png)

可以看到, `LinkedList`也实现了`Cloneable`, `Serializable`接口, 跟`ArrayList`采用同样的实现方式。

### 3. LinkedList源码分析

#### 3.1 数据单元Node

在链表数据结构中, 最基础的数据元素就是节点Node, 这个数据单元Node分为数据域和指针域, 分别存储数据和指向下一个元素,上一个元素。

```java
private static class Node<E> {
    E item; //数据
    Node<E> next; //下一个元素
    Node<E> prev; //上一个元素

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

#### 3.2 LinkedList成员变量

`LinkedList`的成员变量主要有三个:

```java
// 记录当前链表的长度
transient int size = 0;

// 第一个节点
transient Node<E> first;

// 最后一个节点
transient Node<E> last;
```

#### 3.3 构造函数

因为链表没有长度的限制, 所以不会涉及到扩容等问题。

```java
public LinkedList() {
}

public LinkedList(Collection<? extends E> c) {
    this();
    addAll(c);
}
```

`LinkedList`的构造函数比较简单, 一个默认的构造函数, 什么都没做; 一个使用其他集合初始化, 调用`addAll()`方法, 也就是将元素添加到链表中。

#### 3.4 核心方法

`LinkedList`既实现了`List`, 又实现了`Deque`, 所以它必然有一堆的`add`, `remove`, `addFirst`, `addLast`等方法。这些方法的基本语义都是差不多的, 实现也类似, 所以`LinkedList`将通用的方法提取出来, 简化代码。所以我们只需要了解那些内部私有的方法即可。

```java
//将一个元素链接到首位
private void linkFirst(E e) {
    //先将原链表存起来
    final Node<E> f = first;
    //定义一个新节点，其next指向原来的first
    final Node<E> newNode = new Node<>(null, e, f);
    //将first指向新建的节点
    first = newNode;
    //原链表为空表
    if (f == null)
        //把last也指向新建的节点，现在first与last都指向了它
        last = newNode;
    else
        //把原链表挂载在新建节点，也就是现在的first之后
        f.prev = newNode;
    size++;
    modCount++;
}

//与linkFirst类似
void linkLast(E e) {
    //...
}

 //在某个非空节点之前添加元素
void linkBefore(E e, Node<E> succ) {
    // assert succ != null;
    //先把succ节点的前置节点存起来
    final Node<E> pred = succ.prev;
    //新节点插在pred与succ之间
    final Node<E> newNode = new Node<>(pred, e, succ);
    //succ的prev指针移到新节点
    succ.prev = newNode;
    //前置节点为空
    if (pred == null)
        //说明插入到了首位
        first = newNode;
    else
        //把前置节点的next指针也指向新建的节点
        pred.next = newNode;
    size++;
    modCount++;
}

//删除首位的元素，元素必须非空
private E unlinkFirst(Node<E> f) {
    // assert f == first && f != null;
    final E element = f.item;
    final Node<E> next = f.next;
    f.item = null;
    f.next = null; // help GC
    first = next;
    if (next == null)
        last = null;
    else
        next.prev = null;
    size--;
    modCount++;
    return element;
}

private E unlinkLast(Node<E> l) {
    //...
}

//删除一个指定的节点
E unlink(Node<E> x) {
    //...
}
```

上面的源码其实是比较简单的, `LinkedList`提供了一系列方法用来插入和删除。通过移动节点Node元素的位置, 来实现。

`LinkedList`单独对数据查询提供另外的方法:

```java
public E get(int index) {
    checkElementIndex(index);
    return node(index).item;
}

//可以说尽力了
Node<E> node(int index) {
    // assert isElementIndex(index);
    
    //size>>1就是取一半的意思
    //折半，将遍历次数减少一半
    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
```

`LinkedList`利用折半的方式, 判断传入的index是否大于整个链表大小的一半, 如果大于, 那就后尾结点开始向前遍历; 如果小于, 那就从首节点开始向后遍历。

`LinkedList`对`Deque`实现的方法还有很多, 大部分都是调用上面的几种通用方法来实现的:

```java
//引用了node方法，需要遍历
public E set(int index, E element) {
    checkElementIndex(index);
    Node<E> x = node(index);
    E oldVal = x.item;
    x.item = element;
    return oldVal;
}

//也可能需要遍历
public void add(int index, E element) {
    checkPositionIndex(index);

    if (index == size)
            linkLast(element);
    else
        linkBefore(element, node(index));
}

//也要遍历
public E remove(int index) {
    checkElementIndex(index);
    return unlink(node(index));
}

public E peek() {
    final Node<E> f = first;
    return (f == null) ? null : f.item;
}

public E element() {
    return getFirst();
}

public E poll() {
    final Node<E> f = first;
    return (f == null) ? null : unlinkFirst(f);
}

public E remove() {
    return removeFirst();
}

public boolean offer(E e) {
    return add(e);
}

public boolean offerFirst(E e) {
    addFirst(e);
    return true;
}
//...
```

### 4. 总结

`LinkedList`非常适合大量数据的插入与删除, 但对于中间位置的元素, 无论是增删还是改查都需要折半遍历, 在数据量大的情况下十分影响性能。在使用时, 尽量不要涉及查询以及在中间插入数据, 如果要遍历, 也最好使用`foreach`, 也就是`Iterator`提供的方式。



