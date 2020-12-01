## Java集合分析: ArrayList

`ArrayList`可能是我们在平时开发中使用最为频繁的集合类之一, 它的重要性, 应该不需要多说。文档的介绍如下:

> `ArrayList`是`Vector`的翻版, 只是去除了线程安全。`ArrayList`是一个可以动态调整大小的`List`实现, 它的数据顺序与插入顺序始终一致。

### 1. ArrayList结构

![arraylist](/image/arraylist.png)

`ArrayList`是`AbstractList`的子类, 同时实现了`List`接口。

> 除此之外, 它还实现了三个标识型接口, 这三个接口没有任何方法, 仅仅作为标识表示实现类具备某项功能。<br/>
> `RandomAccess`表示实现类支持快速随机访问; `Cloneable`表示实现类支持克隆, 具体表现为重写`clone`方法, `Seriablizable`则表示支持序列化。

### 2. ArrayList初始化

> 一般情况下, 面试到相关`ArrayList`相关问题时, 可能会问`ArrayList`的初始大小是多少?

大部分情况下, 我们初始化`ArrayList`时, 可能都是直接调用无参构造函数。

```java
// 利用java的向上转型, 可能让程序更加灵活, 这样传递相关参数时, 可以传递任何List接口的实现类
List<String> list = new ArrayList();
```

我们知道, `ArrayList`是基于数组的, 而数组是固定长度的。为什么`ArrayList`不需要指定长度, 就可能插入多条数据。

既然`ArrayList`可以动态调整大小, 这也说明必然有一个默认的大小。而想扩充数组的大小, 只能通过数组的复制。这样默认大小以及如何动态调整大小对使用性能产生非常大的影响。

> 例如默认大小为5, 我们向`ArrayList`中插入5条数据, 并不会涉及到扩容。如果想插入100条数据, 就需要将`ArrayList`大小调整到100再进行插入, 这就涉及一次数组的复制。如果此时, 再插入50条数据, 那就得把大小再调整到150, 把原有的100条数据复制过来, 在插入新的50条数据。自此之后, 我们每插入一条数据, 都要涉及一次数据拷贝, 且拷贝的数据量越来越大, 导致性能的下降。

#### 2.1 ArrayList构造方法

`ArrayList`有三个构造方法:

```java

public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}

public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+
                                            initialCapacity);
    }
}

public ArrayList(Collection<? extends E> c) {
    elementData = c.toArray();
    if ((size = elementData.length) != 0) {
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        if (elementData.getClass() != Object[].class)
            elementData = Arrays.copyOf(elementData, size, Object[].class);
    } else {
        // replace with empty array.
        this.elementData = EMPTY_ELEMENTDATA;
    }
}
```

上面的构造方法, 用到了两个成员变量。

```java
transient Object[] elementData;

//这里就是实际存储的数据个数了
private int size;

protected transient int modCount = 0;
```

- **`elementData`**: 这是用来标记存储容量的数组, 也是存放实际数据的数组。当`ArrayList`扩容时, 其capacity就是这个数组的长度, 默认时为空, 添加第一个元素后, 就会直接扩展到DEFAULT_CAPACITY, 也就是10。 **这里和size区别在于, ArrayList扩容并不是需要多少就扩展多少。**

- **`size`**: 实际`ArrayList`中存储的数据个数

- **`modCount`**: 这个变量的主要作用是防止在进行一些操作时, 改变了`ArrayList`的大小, 那将使得结果不可预测。

> 默认的构造方法。默认大小为10, 但是只在插入一条数据时, 才会扩展为10, 而实际上默认是空的。
>
```java
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
```

> 带初始大小的构造方法, 一旦指定了大小, `elementData`就不再是原来的机制了。

```java
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+
                                            initialCapacity);
    }
}
```

> 从一个其他的Collection中构造一个具有初始化数据的`ArrayList`。这里可以看到size是标识存储数据的数量, 这也展示了Collection这种抽象的魅力, 可以在不同的结构间切换

```java
public ArrayList(Collection<? extends E> c) {
    elementData = c.toArray();
    if ((size = elementData.length) != 0) {
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        if (elementData.getClass() != Object[].class)
            elementData = Arrays.copyOf(elementData, size, Object[].class);
    } else {
        // replace with empty array.
        this.elementData = EMPTY_ELEMENTDATA;
    }
}
```

### 3. ArrayList重要方法

下面是`ArrayList`中一些比较简单的方法: 例如`indexOf`, 直接通过正向遍历elementData数组来获取元素的下标; `lastIndexOf`, 则通过反向遍历数组来获取元素最后的下标。

```java
//还记得在AbstractList中的实现吗？那是基于Iterator完成的。
//在这里完全没必要先转成Iterator再进行操作
public int indexOf(Object o) {
    if (o == null) {
        for (int i = 0; i < size; i++)
            if (elementData[i]==null)
                return i;
    } else {
        for (int i = 0; i < size; i++)
            if (o.equals(elementData[i]))
                return i;
    }
    return -1;
}

//和indexOf是相同的道理
 public int lastIndexOf(Object o) {
    if (o == null) {
        for (int i = size-1; i >= 0; i--)
            if (elementData[i]==null)
                return i;
    } else {
        for (int i = size-1; i >= 0; i--)
            if (o.equals(elementData[i]))
                return i;
    }
    return -1;
}

//一样的道理，已经有了所有元素，不需要再利用Iterator来获取元素了
//注意这里返回时把elementData截断为size大小
public Object[] toArray() {
    return Arrays.copyOf(elementData, size);
}

//带类型的转换，看到这里a[size] = null;这个用处真不大，除非你确定所有元素都不为空，
//才可以通过null来判断获取了多少有用数据。
public <T> T[] toArray(T[] a) {
    if (a.length < size)
        // 给定的数据长度不够，复制出一个新的并返回
        return (T[]) Arrays.copyOf(elementData, size, a.getClass());
    System.arraycopy(elementData, 0, a, 0, size);
    if (a.length > size)
        a[size] = null;
    return a;
}
```

数据的操作最重要的就是增删改查, 改查都不涉及长度的变化, 而增删就涉及动态调整大小的问题。

#### 3.1 ArrayList的改和查

```java
private void rangeCheck(int index) {
    if (index >= size)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}

//只要获取的数据位置在0-size之间即可
public E get(int index) {
    rangeCheck(index);

    return elementData(index);
}

//改变下对应位置的值
public E set(int index, E element) {
    rangeCheck(index);

    E oldValue = elementData(index);
    elementData[index] = element;
    return oldValue;
}
```

#### 3.2 ArrayList增和删

增和删是`ArrayList`最重要的部分。源码如下:

```java
//在最后添加一个元素
public boolean add(E e) {
    //先确保elementData数组的长度足够
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}

public void add(int index, E element) {
    rangeCheckForAdd(index);

    //先确保elementData数组的长度足够
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    //将数据向后移动一位，空出位置之后再插入
    System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);
    elementData[index] = element;
    size++;
}
```

以上两种添加数据的方式都调用了`ensureCapacityInternal`方法, 这个方法的源码如下:

```java
//在定义elementData时就提过，插入第一个数据就直接将其扩充至10
private void ensureCapacityInternal(int minCapacity) {
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    
    //这里把工作又交了出去
    ensureExplicitCapacity(minCapacity);
}

//如果elementData的长度不能满足需求，就需要扩充了
private void ensureExplicitCapacity(int minCapacity) {
    modCount++;

    // overflow-conscious code
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}

//扩充
private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    //可以看到这里是1.5倍扩充的
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    
    //扩充完之后，还是没满足，这时候就直接扩充到minCapacity
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    //防止溢出
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

从上面的源码就可以明白`ArrayList`的扩容机制了。

**首先创建一个空数组`elementData`, 第一次插入数据时直接扩充到10, 然后如果`elementData`的长度不足, 就扩充1.5倍, 如果扩充完还是不够, 就使用需要的长度作为`elementData`的长度。**

这种方式虽然比上面我们举的例子要好一点, 但是在遇到大量数据时还是会频繁的拷贝数据。

> 如何缓解`ArrayList`频繁拷贝数据的问题?

- 使用`ArrayList(int initialCapacity)`这个有参构造, 在创建时声明一个较大的大小。**但是这种方式缺点也很明显, 在初始化时, 就会占用较大的内存。**
- 使用`ensureCapacity(int minCapacity)`, 在插入前先扩容。
- 利用`LinkedList`, 非常适合于增删的集合类

#### 3.3 其他实现方法

`ArrayList`不仅实现了`List`中定义的所有功能, 还实现了`equals`, `hashcode`, `clone`, `writeObject`, `readObject`等方法。这些方法都需要与存储的数据配合, 否则结果将是错误的或者克隆得到的数据只是浅拷贝, 或者数据本身不支持序列化。

```java
private void writeObject(java.io.ObjectOutputStream s)
    throws java.io.IOException{
    // Write out element count, and any hidden stuff
    int expectedModCount = modCount;
    s.defaultWriteObject();

    // Write out size as capacity for behavioural compatibility with clone()
    s.writeInt(size);

        // Write out all elements in the proper order.
    for (int i=0; i<size; i++) {
        s.writeObject(elementData[i]);
    }

    if (modCount != expectedModCount) {
        throw new ConcurrentModificationException();
    }
}
```

上面的是`ArrayList`进行序列化的源码, 通过这段代码我们知道了

- `elementData`被`transient`修饰, 也就是不会参与序列化。存储数据的数组本身不会序列化, 而是遍历数组的中的数据进行序列化;
- `size`属性也被写入序列化;
- `modCount`的作用也在此体现, 如果序列化进行修改操作时, 就会抛出异常。

