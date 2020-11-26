## Java集合源码: Iterable

我们知道, 在实际开发中如果想要遍历集合, 通常有三种方式:

1. for循环

```java
for (int i = 0; i < length; i++) {
    ...
}
```

2. foreach循环

```java
for (String s : strings) {
    ...
}
```

3. Iterator迭代器

```java
Iterator iterator = strings.iterator();
while(iterator.hasNext()) {
    System.out.println(iterator.next());
}
```

上面的三种遍历方式有什么区别呢?

> for循环: 根据下标来获取元素;<br/>
> foreach主要对类似链表的结构提供遍历支持, 链表没有下标, 所以使用for循环遍历会大大降低性能;<br/>
> **Iterator实际上就是foreach。**

**那么, 为什么集合可以进行foreach遍历, 而我们自定义的Java对象有什么办法可以支持这种遍历方式呢? 答案就是`Iterator`**

### Iterator分析

`Iterator`是迭代器的意思, 作用是为集合类提供**for-each**循环的支持。

使用for循环需要通过位置来获取元素, 而这种获取方式仅有数组支持, 其他数据结构, 例如链表, 只能通过查询获取数据, 这样就会大大的降低效率。

`Iterator`是一个接口, 作用就是为Java对象提供foreach循环, 主要方法是返回一个`Iterator`对象

```java
Iterator<T> iterator();
```

也就是说, 如果想让一个Java对象支持foreach, 只要实现**Iterator**接口, 然后就可以像集合那样, 通过`Iterator iterator = strings.iterator()`方式, 或者使用foreach遍历了。

#### Iterator源码

`Iterator`是foreach遍历的主体, 代码主要如下:

```java
// 判断一个对象集合是否还有下一个元素
boolean hasNext();

// 获取下一个元素
E next();

// 删除最后一个元素。默认是不支持的，因为在很多情况下其结果不可预测，比如数据集合在此时被修改
default void remove(){...}

// 主要将每个元素作为参数发给action来执行特定操作
default void forEachRemaining(Consumer<? super E> action){...}
```

`Iterator`还有一个子接口, 是为需要双向遍历数据时准备的, 在后面的`ArrayList`和`LinkedList`都是有它的影子。它主要增加了下面几个方法:

```java
// 是否有前一个元素
boolean hasPrevious();

// 获取前一个元素
E previous();

// 获取下一个元素的位置
int nextIndex();

// 获取前一个元素的位置
int previousIndex();

// 添加一个元素
void add(E e);

// 替换当前元素值
void set(E e);
```

### 总结

Java有许多特性都是通过接口来实现的, foreach循环也是。

- for循环主要是依赖元素的下标来查询数据
- foreach主要解决的就是for循环依赖下标的问题, 为高效遍历更多的数据结构提供了支持。