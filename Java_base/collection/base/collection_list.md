## Java集合源码: 超级接口List

`List`是`Collection`三大直接子接口之一, 其中的数据可以通过位置检索, 用户可以在指定位置插入数据。`List`的数据可以为空, 可以重复。

### 1. List特有方法

下面是`List`不同于`Collection`的方法:

```java
//在指定位置，将指定的集合插入到当前的集合中
boolean addAll(int index, Collection<? extends E> c);

//这是一个默认实现的方法，会通过Iterator的方式对每个元素进行指定的操作
default void replaceAll(UnaryOperator<E> operator) {
    Objects.requireNonNull(operator);
    final ListIterator<E> li = this.listIterator();
    while (li.hasNext()) {
        li.set(operator.apply(li.next()));
    }
}

//排序，依据指定的规则对当前集合进行排序，可以看到，排序是通过Arrays这个工具类完成的。
default void sort(Comparator<? super E> c) {
    Object[] a = this.toArray();
    Arrays.sort(a, (Comparator) c);
    ListIterator<E> i = this.listIterator();
    for (Object e : a) {
        i.next();
        i.set((E) e);
    }
}

//获取指定位置的元素
E get(int index);

//修改指定位置元素的值
E set(int index, E element);

//将指定元素添加到指定的位置
void add(int index, E element);

//将指定位置的元素移除
E remove(int index);

//返回一个元素在集合中首次出现的位置
int indexOf(Object o);

//返回一个元素在集合中最后一次出现的位置
int lastIndexOf(Object o);

//ListIterator继承于Iterator，主要增加了向前遍历的功能
ListIterator<E> listIterator();

//从指定位置开始，返回一个ListIterator
ListIterator<E> listIterator(int index);

//返回一个子集合[fromIndex, toIndex)，非结构性的修改返回值会反映到原表，反之亦然。
//如果原表进行了结构修改，则返回的子列表可能发生不可预料的事情
List<E> subList(int fromIndex, int toIndex);
```

### 2. 抽像实现类: AbstractList

- 要实现一个不可修改的集合, 只需要重写`get`和`size`即可;
- 要实现一个可以修改的集合, 还需要重写`set`方法;
- 如果要动态调整大小, 就必须再实现`add`和`remove`方法。

#### 2.1 Itr与ListItr

在`AbstractList`实现了`iterator()`方法, 这个方法直接返回了一个`Itr`对象

```java
public Iterator<E> iterator() {
    return new iterator();
}

private class Itr implements Iterator<E> {
    ...
}
```
`Itr`是`AbstractList`的内部类, 这个内部类实现了`Iterator`接口, 合理的处理`hasNext`, `next`, `remove`方法。

`listIterator()`方法:

```java
public ListIterator<E> listIterator() {
    return listIterator(0);
}

public ListIterator<E> listIterator(final int index) {
    rangeCheckForAdd(index);

    return new ListItr(index);
}
```

`ListItr`继承了`Itr`并且也实现了`ListIterator`接口, 而`ListIterator`继承了`Iterator`, 添加属于子类的独有方法，

利用`listIterator()`方法可以构造出`ListIterator`对象, 从而完成下面的方法:

```java
//寻找一个元素首次出现的位置，只需要从前往后遍历，找到那个元素并返回其位置即可。
public int indexOf(Object o) {
    ListIterator<E> it = listIterator();
    if (o==null) {
        while (it.hasNext())
            if (it.next()==null)
                return it.previousIndex();
    } else {
        while (it.hasNext())
            if (o.equals(it.next()))
                return it.previousIndex();
    }
    return -1;
}

//同理，寻找一个元素最后一次出现的位置，只需要从列表最后一位向前遍历即可。
//看到listIterator(int index)方法是可以传递参数的，这个我想我们都可以照着写出来了。
public int lastIndexOf(Object o) {
    //...
}

//这个方法是把从fromIndex到toIndex之间的元素从集合中删除。
//clear()方法也是调用这个实现的（我认为clear实现意义并不大，因为在其上级AbstractCollection中已经有了具体实现）。
protected void removeRange(int fromIndex, int toIndex) {
    ListIterator<E> it = listIterator(fromIndex);
    for (int i=0, n=toIndex-fromIndex; i<n; i++) {
        it.next();
        it.remove();
    }
}
```

#### 2.2 SubList, equals, hashcode

对于`SubList`并不是新建的集合, 只是持有当前集合的引用, 然后控制用户可以操作的范围。`SubList`定义在`AbstractList`内部, 并且是`AbstractList`的子类。**它的作用就是返回一个List集合的部分子集合。**

> 我们要注意的是, `SubList`是持有当前集合的引用, 所以只要我们对子集合进行修改操作, 原有集合也会随之变化。(这里涉及到引用传递, 传入SubList中的是原集合的引用, 也就是指向集合的地址。) 

`SubList`几乎重写了父类的大部分方法, 利用`subList()`方法即可返回原集合的部分子集合。

```java
public static void main(final String[] args) {  
    List<Object> lists = new ArrayList<Object>();  
  
    lists.add("1");  
    lists.add("2");  
    lists.add("3");  
    lists.add("4");  
  
    //注意这里是和本文顶部的代码不同的....  
    List<Object> tempList = new ArrayList<Object>(lists.subList(2, lists.size()));  
  
    tempList.add("6");  
  
    System.out.println(tempList); // 1  
  
    System.out.println(lists); // 2  
}  
```

如上面代码所示, 我们会发现, 即使修改tempList集合, 也会同时修改原有的集合, 因为SubList构造时就会把当前的集合引用传入。

`SubList`的构造方法:

```java
SubList(AbstractList<E> list, int fromIndex, int toIndex) {  
    if (fromIndex < 0)  
        throw new IndexOutOfBoundsException("fromIndex = " + fromIndex);  
    if (toIndex > list.size())  
        throw new IndexOutOfBoundsException("toIndex = " + toIndex);  
    if (fromIndex > toIndex)  
        throw new IllegalArgumentException("fromIndex(" + fromIndex +  
                                           ") > toIndex(" + toIndex + ")");  
    l = list;  
    offset = fromIndex;  
    size = toIndex - fromIndex;  
    expectedModCount = l.modCount;  
}  
```

对于`equals`和`hashcode`的实现, 也关系到我们的使用, 利用我们在项目中利用`contains`方法判断一个集合是否包含一个元素, 如果这个元素时一个Java对象, 那我们必然需要重写这个对象的equals和hashcode方法, 否则很有可能判断失败。

```java
public boolean equals(Object o) {
    if (o == this)
        return true;
    if (!(o instanceof List))
        return false;

    ListIterator<E> e1 = listIterator();
    ListIterator<?> e2 = ((List<?>) o).listIterator();
    while (e1.hasNext() && e2.hasNext()) {
        E o1 = e1.next();
        Object o2 = e2.next();
        //这里用到了数据元素的equals方法
        if (!(o1==null ? o2==null : o1.equals(o2)))
            return false;
    }
    return !(e1.hasNext() || e2.hasNext());
}

public int hashCode() {
    int hashCode = 1;
    for (E e : this)
        //这里用到了数据元素的hashCode方法
        hashCode = 31*hashCode + (e==null ? 0 : e.hashCode());
    return hashCode;
}
```