## Java集合分析: Queue接口

队列是编程中最常用的数据结构之一, 常用的Queue基本上分为两种:

- **FIFO(First in First out)**: 先进先出队列
- **LIFO(Last in First out)**: 后进先出队列。例如: `Stack`栈

### 接口Queue

队列在开发过程也是常用的数据结构之一, 在处理并发问题时, `BlockingQueue`很好的解决数据传输的问题。

`Queue`也继承`Collection`接口, 说明队列`Queue`也是集合的一种, 它的主要方法如下:

```java
//将元素插入队列
boolean add(E e);

//将元素插入队列，与add相比，在容量受限时应该使用这个
boolean offer(E e);

//将队首的元素删除，队列为空则抛出异常
E remove();

//将队首的元素删除，队列为空则返回null
E poll();

//获取队首元素，但不移除，队列为空则抛出异常
E element();

//获取队首元素，但不移除，队列为空则返回null
E peek();
```

### 实现类AbstractQueue

`Queue`的定义很简单, 它的实现类也很简单。`AbstractQueue`仅实现了`add`, `remove`, `element`三个方法。

```java
//这里我们就明白，对于有容量限制的，直接调用offer肯定会更快
public boolean add(E e) {
    if (offer(e))
        return true;
    else
        throw new IllegalStateException("Queue full");
}

public E element() {
    E x = peek();
    if (x != null)
        return x;
    else
        throw new NoSuchElementException();
}

public E remove() {
    E x = poll();
    if (x != null)
        return x;
    else
        throw new NoSuchElementException();
}
```

通过上面的源码可以发现, `add`内部还是调用的`offer`方法, 并且通过`offer`的返回值判断是否添加成功。 所以, 如果队列是有限的, 肯定直接使用`offer`方法会更快。