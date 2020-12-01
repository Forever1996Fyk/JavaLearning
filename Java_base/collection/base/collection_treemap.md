## Java集合分析: TreeMap

`TreeMap`是**红黑树**的Java实现, 所以了解**红黑树**的原理很重要。后面的`HashMap`也是如此。

**红黑树**能保证增, 删, 查等基本操作的时间复杂度**O(lgN)**

我们先看下`TreeMap`的结构:

![collection_treemap](/image/collection_treemap.png)

### 1. TreeMap内部构造

#### 1.1 Entry定义

```java
static final class Entry<K,V> implements Map.Entry<K,V> {
    K key;
    V value;
    Entry<K,V> left;
    Entry<K,V> right;
    Entry<K,V> parent;
    boolean color = BLACK;

    Entry(K key, V value, Entry<K,V> parent) {
        this.key = key;
        this.value = value;
        this.parent = parent;
    }
    // ... 省略其他方法
}
```

#### 1.2 构造函数与成员变量

```java
// 比较器
private final Comparator<? super K> comparator;

// 根节点
private transient Entry<K,V> root;

// 大小
private transient int size = 0;

// 默认构造，比较器采用key的自然比较顺序
public TreeMap() {
    comparator = null;
}

// 指定比较器
public TreeMap(Comparator<? super K> comparator) {
    this.comparator = comparator;
}

// 从Map集合导入初始数据
public TreeMap(Map<? extends K, ? extends V> m) {
    comparator = null;
    putAll(m);
}

// 从SortedMap导入初始数据
public TreeMap(SortedMap<K, ? extends V> m) {
    comparator = m.comparator();
    try {
        buildFromSorted(m.size(), m.entrySet().iterator(), null, null);
    } catch (java.io.IOException cannotHappen) {
    } catch (ClassNotFoundException cannotHappen) {

    }
}
```

### 2. 核心方法

红黑树最复杂的地方就在于增删了

#### 2.1 添加元素

我们先从put方法开始:

```java
public V put(K key, V value) {
    // 暂存根节点
    Entry<K,V> t = root;

    // 根节点空，就是还没有元素
    if (t == null) {
        compare(key, key); // type (and possibly null) check
        // 新建一个元素，默认颜色黑色
        root = new Entry<>(key, value, null);
        size = 1;
        modCount++;
        return null;
    }

    // 根节点不为空，有元素时的情况
    int cmp;
    Entry<K,V> parent;
    // split comparator and comparable paths
    Comparator<? super K> cpr = comparator;
    // 初始化时指定了comparator比较器
    if (cpr != null) {
        do {
            // 把t暂存到parent中
            parent = t;
            cmp = cpr.compare(key, t.key);
            if (cmp < 0)
                // 比较小，往左侧插入
                t = t.left;
            else if (cmp > 0)
                // 比较大，往右侧插入
                t = t.right;
            else
                // 一样大，所以就是更新当前值
                return t.setValue(value);
        } while (t != null);
    }
    else {
    // 使用key的比较器，while循环原理和上述一致
        if (key == null)
            throw new NullPointerException();
        @SuppressWarnings("unchecked")
            Comparable<? super K> k = (Comparable<? super K>) key;
        do {
            parent = t;
            cmp = k.compareTo(t.key);
            if (cmp < 0)
                t = t.left;
            else if (cmp > 0)
                t = t.right;
            else
                return t.setValue(value);
        } while (t != null);
    }

    // 不断的比较，找到了没有相应儿子的节点
    //（cmp<0就是没有左儿子，cmp>0就是没有右儿子）
    Entry<K,V> e = new Entry<>(key, value, parent);
    // 把数据插入
    if (cmp < 0)
        parent.left = e;
    else
        parent.right = e;

    // 新插入的元素破坏了红黑树规则，需要调整
    fixAfterInsertion(e);
    size++;
    modCount++;
    return null;
}
```

`fixAfterInsertion`是实现的重难点, 它是调整红黑树的核心方法:

```java
private void fixAfterInsertion(Entry<K,V> x) {
    // 先把x节点染成红色，这样可以不增加黑高，简化调整问题
    x.color = RED;
    
    // 条件是父节点是红色的，且x不是root节点，
    // 因为到root节点后就走到另外的分支了，而那个分支是正确的
    while (x != null && x != root && x.parent.color == RED) {
        //x的父节点是其祖父节点的左儿子
        if (parentOf(x) == leftOf(parentOf(parentOf(x)))) {
            // y是x的叔叔，也就是祖父节点的右儿子
            Entry<K,V> y = rightOf(parentOf(parentOf(x)));
            //叔叔是红色的
            if (colorOf(y) == RED) {
                setColor(parentOf(x), BLACK);
                setColor(y, BLACK);
                setColor(parentOf(parentOf(x)), RED);
                // 调整完毕，继续向上循环
                x = parentOf(parentOf(x));
            } else {
            // 叔叔是黑色的
                if (x == rightOf(parentOf(x))) {
                    // x是右节点，以其父节点左旋
                    x = parentOf(x);
                    rotateLeft(x);
                }
                // 右旋
                setColor(parentOf(x), BLACK);
                setColor(parentOf(parentOf(x)), RED);
                rotateRight(parentOf(parentOf(x)));
            }
        } else {
            //x的父节点是其祖父节点的右儿子
            // y是其叔叔
            Entry<K,V> y = leftOf(parentOf(parentOf(x)));
            if (colorOf(y) == RED) {
                //叔叔是红色的
                setColor(parentOf(x), BLACK);
                setColor(y, BLACK);
                setColor(parentOf(parentOf(x)), RED);
                // 调整完毕，继续向上循环
                x = parentOf(parentOf(x));
            } else {
                if (x == leftOf(parentOf(x))) {
                    // x是左节点，以其父节点右旋
                    x = parentOf(x);
                    rotateRight(x);
                }
                //左旋
                setColor(parentOf(x), BLACK);
                setColor(parentOf(parentOf(x)), RED);
                rotateLeft(parentOf(parentOf(x)));
            }
        }
    }
    
    //root节点颜色为黑色
    root.color = BLACK;
}
```

左旋和右旋代码如下:

```java
// 右旋与左旋思路一致，只分析其一
// 结果相当于把p和p的儿子调换了
private void rotateLeft(Entry<K,V> p) {
    if (p != null) {
        // 取出p的右儿子
        Entry<K,V> r = p.right;
        // 然后将p的右儿子的左儿子，也就是p的左孙子变成p的右儿子
        p.right = r.left;
        if (r.left != null)
            // p的左孙子的父亲现在是p
            r.left.parent = p;

        // 然后把p的父亲，设置为p右儿子的父亲
        r.parent = p.parent;
        // 这说明p原来是root节点
        if (p.parent == null)
            root = r;
        else if (p.parent.left == p)
            p.parent.left = r;
        else
            p.parent.right = r;
        r.left = p;
        p.parent = r;
    }
}

//和左旋类似
private void rotateRight(Entry<K,V> p) {
    if (p != null) {
        Entry<K,V> l = p.left;
        p.left = l.right;
        if (l.right != null) l.right.parent = p;
        l.parent = p.parent;
        if (p.parent == null)
            root = l;
        else if (p.parent.right == p)
            p.parent.right = l;
        else p.parent.left = l;
        l.right = p;
        p.parent = l;
    }
}
```

##### 2.1.1 添加元素图示

这里结合代码以及红黑树的原理, 在深入了解红黑树添加元素的原理。

首先必须要牢记红黑树的规则:

1. 每个节点要么是红色, 要么是黑色;
2. 根节点是黑色;
3. 每个叶子节点(NULL)是黑色;(注意这里的叶子节点是为空的叶子节点, 一般会省略)
4. 如果一个节点是红色, 那么它的两个儿子都是黑色的;
5. 从一个节点到该节点的子孙节点的所有路径包含相同数目的黑节点。

例如, 有一颗简单的红黑树:

![collection_treemap_redblacktree1](/image/collection_treemap_redblacktree1.png)

当我们插入一个值为7的元素时, 根据`put`方法的步骤:

1. 先将新加入的元素与根节点比较, 发现7<14, 就向左遍历。到6时, 发现7>6; 于是再跟8比较, 发现8是一个叶节点(注意这里不是叶子节点), 所以把7插入8的左儿子处;
2. 为了尽量保证第5条规则, 所以默认插入的元素7设为红色。(如果是黑色, 那一定为导致规则5的破坏, 而红色,则不一定。)

    ![collection_treemap_redblacktree2](/image/collection_treemap_redblacktree2.png)

3. 根据上图, 此时这棵树已经不是红黑树了, 因为违反了规则4(`如果一个节点是红色, 那么它的两个儿子都是黑色的`), 所以我们在根据`fixAfterInsertion`方法的步骤, 对这个树进行调整, 这里元素7就是`fixAfterInsertion`的参数x。

调用 `fixAfterInsertion` 方法:

1. 判断元素7的不是根节点, 并且它的父节点是红色的; 所以进入while循环; 
2. 元素7的父节点是其祖父节点的右节点, 如果不是进入else判断, 发现7的祖父节点的左节点(也就是叔叔节点)4是红色的, 那么就把**x的父节点8**染成黑色, **叔叔节点4**也染成黑色, **祖父节点6**染成红色, 并且x指向祖父节点;
    ```java
    else {
        Entry<K,V> y = leftOf(parentOf(parentOf(x)));
        if (colorOf(y) == RED) {
            setColor(parentOf(x), BLACK);
            setColor(y, BLACK);
            setColor(parentOf(parentOf(x)), RED);
            x = parentOf(parentOf(x));
        } else {
            if (x == leftOf(parentOf(x))) {
                x = parentOf(x);
                rotateRight(x);
            }
            setColor(parentOf(x), BLACK);
            setColor(parentOf(parentOf(x)), RED);
            rotateLeft(parentOf(parentOf(x)));
        }
    }
    ```
    ![collection_treemap_redblacktree3](/image/collection_treemap_redblacktree3.png)

3. 此时x指向元素6, 再次进入while循环, 此时x的父节点是10, 它是祖父节点14的左儿子, 所以进入if判断, 并且x的祖父节点的右节点(也就是叔叔节点)18是黑色的, 所以代码就会进一步走向下面的else。也就是把**x的父节点10**染成黑色, **x的祖父节点14**染成红色; 然后以x的祖父节点14进行右旋;

    ```java
    if (parentOf(x) == leftOf(parentOf(parentOf(x)))) {
        Entry<K,V> y = rightOf(parentOf(parentOf(x)));
        if (colorOf(y) == RED) {
            setColor(parentOf(x), BLACK);
            setColor(y, BLACK);
            setColor(parentOf(parentOf(x)), RED);
            x = parentOf(parentOf(x));
        } else {
            if (x == rightOf(parentOf(x))) {
                x = parentOf(x);
                rotateLeft(x);
            }
            setColor(parentOf(x), BLACK);
            setColor(parentOf(parentOf(x)), RED);
            rotateRight(parentOf(parentOf(x)));
        }
    } 
    ```

    将10和14颜色互换:

    ![collection_treemap_redblacktree4](/image/collection_treemap_redblacktree4.png)

    以14为基础进行右旋, 具体操作为, 把10的右儿子12, 变为14的左儿子, 然后把14变为10的右儿子, 结果如下:

    ![collection_treemap_redblacktree5](/image/collection_treemap_redblacktree5.png)

3. 此时while循环条件不满足了, 也就是调整完毕, 此时又是一颗正确的红黑树。

**当然, 上面只是调整的一种情况, 还有更复杂的情况。(其实看到这大家都应该明白, 所谓更复杂的情况就是, 整棵树在不断的左右旋, 以及红黑转换。只要能满足红黑树的五条基本规则即可。)**

#### 2.2 获取元素

`TreeMap`的元素是有序的, 当使用中序遍历时就可以得到一个有序的Set集合, 所以获取元素可以采用二分法, 也就是折半查找:

```java
final Entry<K,V> getEntry(Object key) {
    // Offload comparator-based version for sake of performance
    if (comparator != null)
        return getEntryUsingComparator(key);
    if (key == null)
        throw new NullPointerException();
    @SuppressWarnings("unchecked")
        Comparable<? super K> k = (Comparable<? super K>) key;
    Entry<K,V> p = root;
    while (p != null) {
        int cmp = k.compareTo(p.key);
        if (cmp < 0)
            p = p.left;
        else if (cmp > 0)
            p = p.right;
        else
            return p;
    }
    return null;
}
```

除了获取某一个元素外, 还可以获取它的前一个元素和后一个元素:

```java
// 获取前一个元素
static <K,V> Entry<K,V> predecessor(Entry<K,V> t) {
    if (t == null)
        return null;
    else if (t.left != null) {
        // t有左孩子，所以t的前一个元素是它左孩子所在的子树的最右侧叶子结点
        Entry<K,V> p = t.left;
        while (p.right != null)
            p = p.right;
        return p;
    } else {
        // t没有左孩子，所以t的前一个元素有两种情况
        // 1. t是右孩子，那它的前一个元素就是它的父结点
        // 2. t是左孩子，它的前一个元素需要向上递归，直到递归到下一个是右孩子的节点，转为情况1
        Entry<K,V> p = t.parent;
        Entry<K,V> ch = t;
        while (p != null && ch == p.left) {
            ch = p;
            p = p.parent;
        }
        return p;
    }
}

// 获取后一个元素
static <K,V> TreeMap.Entry<K,V> successor(Entry<K,V> t) {
    //...
}
```

#### 2.3 删除元素

从红黑树中删除一个元素, 和增加一个元素一样复杂。它的源码如下:

```java
public V remove(Object key) {
    // 先用二分法获取这个元素，如果为null，不需要继续了
    Entry<K,V> p = getEntry(key);
    if (p == null)
        return null;

    V oldValue = p.value;
    deleteEntry(p);
    return oldValue;
}

private void deleteEntry(Entry<K,V> p) {
    modCount++;
    size--;

    // If strictly internal, copy successor's element to p and then make p
    // point to successor.
    //如果p有两个儿子，就把p指向它的后继者，也就是它后边的元素
    if (p.left != null && p.right != null) {
        Entry<K,V> s = successor(p);
        p.key = s.key;
        p.value = s.value;
        p = s;
    } // p has 2 children

    // Start fixup at replacement node, if it exists.
    // p有一个儿子，或者没有儿子，获取到之后放在replacement中
    Entry<K,V> replacement = (p.left != null ? p.left : p.right);

    // p有儿子
    if (replacement != null) {
        // Link replacement to parent
        // 把p的子孙接在p的父级
        replacement.parent = p.parent;
        
        //p是根节点 
        if (p.parent == null)
            root = replacement;
        //p是左儿子
        else if (p == p.parent.left)
            p.parent.left  = replacement;
        // p是右儿子
        else
            p.parent.right = replacement;

        //把p的链接都删掉
        // Null out links so they are OK to use by fixAfterDeletion.
        p.left = p.right = p.parent = null;

        // Fix replacement
        if (p.color == BLACK)
            //修正
            fixAfterDeletion(replacement);
    } else if (p.parent == null) { // return if we are the only node.
        root = null;
    } else {
        //p没有儿子
        //  No children. Use self as phantom replacement and unlink.
        if (p.color == BLACK)
            fixAfterDeletion(p);
        // 把其父节点链接到p的都去掉
        if (p.parent != null) {
            if (p == p.parent.left)
                p.parent.left = null;
            else if (p == p.parent.right)
                p.parent.right = null;
            p.parent = null;
        }
    }
}

private void fixAfterDeletion(Entry<K,V> x) {
    while (x != root && colorOf(x) == BLACK) {
        // x是左儿子
        if (x == leftOf(parentOf(x))) {
            // sib是x的兄弟
            Entry<K,V> sib = rightOf(parentOf(x));
            
            // 兄弟是红色的
            if (colorOf(sib) == RED) {
                setColor(sib, BLACK);
                setColor(parentOf(x), RED);
                rotateLeft(parentOf(x));
                sib = rightOf(parentOf(x));
            }
            
            // 兄弟没有孩子或者孩子是黑色的
            if (colorOf(leftOf(sib))  == BLACK &&
                colorOf(rightOf(sib)) == BLACK) {
                setColor(sib, RED);
                x = parentOf(x);
            } else {
                // 兄弟的右孩子是黑色的
                if (colorOf(rightOf(sib)) == BLACK) {
                    setColor(leftOf(sib), BLACK);
                    setColor(sib, RED);
                    rotateRight(sib);
                    sib = rightOf(parentOf(x));
                }
                setColor(sib, colorOf(parentOf(x)));
                setColor(parentOf(x), BLACK);
                setColor(rightOf(sib), BLACK);
                rotateLeft(parentOf(x));
                x = root;
            }
        } else { // symmetric
            Entry<K,V> sib = leftOf(parentOf(x));

            if (colorOf(sib) == RED) {
                setColor(sib, BLACK);
                setColor(parentOf(x), RED);
                rotateRight(parentOf(x));
                sib = leftOf(parentOf(x));
            }

            if (colorOf(rightOf(sib)) == BLACK &&
                colorOf(leftOf(sib)) == BLACK) {
                setColor(sib, RED);
                x = parentOf(x);
            } else {
                if (colorOf(leftOf(sib)) == BLACK) {
                    setColor(rightOf(sib), BLACK);
                    setColor(sib, RED);
                    rotateLeft(sib);
                    sib = leftOf(parentOf(x));
                }
                setColor(sib, colorOf(parentOf(x)));
                setColor(parentOf(x), BLACK);
                setColor(leftOf(sib), BLACK);
                rotateRight(parentOf(x));
                x = root;
            }
        }
    }

    setColor(x, BLACK);
}
```

删除的过程:
1. 先根据二分法, 获取元素的节点信息, 如果是null, 直接返回; 否则就删除节点;
2. 删除节点要判断节点是否存在孩子节点;
3. 删除后的树, 是否满足红黑树, 如果不满足依然要根据红黑树的规则进行调整。
