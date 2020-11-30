## Java集合分析: 接口Deque

`Deque`全称为`double ended queue`, 即双向队列, 它允许在两侧插入或删除元素, 除此之外, 其余特性与父类`Queue`类似。

`Deque`中定义的方法主要分为四部分:

1. 使用`Deque`的特性, 提供两侧插入或删除的方法;
2. 继承`Queue`的实现;
3. 实现基于`Stack`的方法;
4. 继承`Collection`的方法。

### 1. Deque两侧插入、删除

这里方法与`Queue`定义方法一直, 主要是针对两侧插入或删除的:

```java
//在队首添加元素
void addFirst(E e);
//在队首添加元素
boolean offerFirst(E e);

//在队尾添加元素
void addLast(E e);
boolean offerLast(E e);

//删除队首元素
E removeFirst();
E pollFirst();

//删除队尾元素
E removeLast();
E pollLast();

//获取队首元素
E getFirst();
E peekFirst();

//获取队尾元素
E getLast();
E peekLast();

//删除第一个事件，大多数指的是删除第一个和 o equals的元素
boolean removeFirstOccurrence(Object o);
//删除最后一个事件，大多数指的是删除最后一个和 o equals的元素
boolean removeLastOccurrence(Object o);
```

### 2. 与Queue对应的方法

因为`Queue`遵循`FIFO`, 所以它的方法与`Deque`中对应关系是不一样的, 结合上面`Deque`的方法定义, 它们的方法间对应关系如下:

```java
//与addLast(E e)等价
boolean add(E e);

//与offerLast(E e)等价
boolean offer(E e);

//与removeFirst()等价
E remove();

//与pollFirst()等价
E poll();

//与getFirst()等价
E element();

//与peekFirst()等价
E peek();
```

### 3. 实现Stack

我们知道`Stack`, 是后进先出`LIFO`的队列, 所以Stack仅支持一侧的插入删除操作, 实现下面两个方法即可:

```java
//与addFirst()等价
void push(E e);

//与removeFirst()等价
E pop();
```

### 4. 继承Collection的方法

这里主要关注两个方法:

```java
//顺序是从队首到队尾
Iterator<E> iterator();

//顺序是从队尾到队首
Iterator<E> descendingIterator();
```