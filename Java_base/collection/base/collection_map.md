## Java集合分析: Map接口

学习了前面的知识, 不管是基于数组的`ArrayList`, 还是基于链表的`LinkedList`也是各有优缺点。数组查询速度快但插入很慢, 链表插入速度快但是查询效率低。

哈希表整合了数组与链表的优点, 能在插入和查找等方面效率都很高。`HashMap`就是基于哈希表实现的, 在JDk1.8后, 还引入了**红黑树**, 进一步提升性能。

在正式学习Map接口前, 我们先要了解Map接口的一个内部接口`Map.Entry`。

### 1. Map.Entry接口

存储在Map中的数据需要实现此接口, 主要提供对key和value的操作。源码如下:

```java
public interface Map<K,V> {
    interface Entry<K,V> {
        // 获取对应的key
        K getKey();

        // 获取对应的value
        V getValue();

        // 替换原有的value
        V setValue(V value);

        // 希望我们实现equals和hashCode
        boolean equals(Object o);
        int hashCode();

        // 从1.8起，还提供了比较的方法，类似的方法共四个
        public static <K extends Comparable<? super K>, V> Comparator<Map.Entry<K,V>> comparingByKey() {
                return (Comparator<Map.Entry<K, V>> & Serializable)
                    (c1, c2) -> c1.getKey().compareTo(c2.getKey());
        }

         public static <K, V extends Comparable<? super V>> Comparator<Map.Entry<K,V>> comparingByValue() {
            return (Comparator<Map.Entry<K, V>> & Serializable)
                (c1, c2) -> c1.getValue().compareTo(c2.getValue());
        }

        public static <K, V> Comparator<Map.Entry<K, V>> comparingByKey(Comparator<? super K> cmp) {
            Objects.requireNonNull(cmp);
            return (Comparator<Map.Entry<K, V>> & Serializable)
                (c1, c2) -> cmp.compare(c1.getKey(), c2.getKey());
        }

        public static <K, V> Comparator<Map.Entry<K, V>> comparingByValue(Comparator<? super V> cmp) {
            Objects.requireNonNull(cmp);
            return (Comparator<Map.Entry<K, V>> & Serializable)
                (c1, c2) -> cmp.compare(c1.getValue(), c2.getValue());
        }
    }
}
```

### 2. Map中核心方法

```java
// 返回当前数据个数
int size();

// 是否为空
boolean isEmpty();

// 判断是否包含key，这里用到了key的equals方法，所以key必须实现它
boolean containsKey(Object key);

// 判断是否有key保存的值是value，这也基于equals方法
boolean containsValue(Object value);

// 通过key获取对应的value值
V get(Object key);

// 存入key-value
V put(K key, V value);

// 移除一个key-value对
V remove(Object key);

// 从其他Map添加
void putAll(Map<? extends K, ? extends V> m);

// 清空
void clear();

// 返回所有的key至Set集合中，因为key是不可重的，Set也是不可重的
Set<K> keySet();

// 返回所有的values
Collection<V> values();

// 返回key-value对到Set中
Set<Map.Entry<K, V>> entrySet();

// 希望我们实现equals和hashCode
boolean equals(Object o);
int hashCode();
```

通过上面的源码可以解决一些疑惑:

1. `entrySet()`, 就是`Map.Entry`的Set集合;
2. `containsKey()`, `containsValue()`方法都需要重写equals的方法才能实现

### 3. 实现类AbstractMap

与`AbstractCollection`类似, `AbstractMap`也是为了提供一些方法的实现, 可以方便继承。

```java
 // 返回大小，这里大小基于entrySet的大小
public int size() {
    return entrySet().size();
}

public boolean isEmpty() {
    return size() == 0;
}

//基于entrySet操作, entrySet就是Map.Entry的Set集合
public boolean containsKey(Object key) {
    Iterator<Map.Entry<K,V>> i = entrySet().iterator();
    if (key==null) {
        while (i.hasNext()) {
            Entry<K,V> e = i.next();
            if (e.getKey()==null)
                return true;
        }
    } else {
        while (i.hasNext()) {
            Entry<K,V> e = i.next();
            if (key.equals(e.getKey()))
                return true;
        }
    }
    return false;
}

public boolean containsValue(Object value) {
    //...
}

public V get(Object key) {
    //...
}

public V remove(Object key) {
    //...
}

public void clear() {
    entrySet().clear();
}
```

`AbstractMap`还定义了两个变量:

```java
transient Set<K>        keySet;
transient Collection<V> values;
```

并提供了初始化的默认方法:

```java
public Set<K> keySet() {
    Set<K> ks = keySet;
    if (ks == null) {
        ks = new AbstractSet<K>() {
            public Iterator<K> iterator() {
                return new Iterator<K>() {
                    private Iterator<Entry<K,V>> i = entrySet().iterator();

                    public boolean hasNext() {
                        return i.hasNext();
                    }

                    public K next() {
                        return i.next().getKey();
                    }

                    public void remove() {
                        i.remove();
                    }
                };
            }

            public int size() {
                return AbstractMap.this.size();
           }

            public boolean isEmpty() {
               return AbstractMap.this.isEmpty();
            }

            public void clear() {
                AbstractMap.this.clear();
            }

            public boolean contains(Object k) {
                return AbstractMap.this.containsKey(k);
            }
        };
        keySet = ks;
    }
    return ks;
}

public Collection<V> values() {
    if (values == null) {
        values = new AbstractCollection<V>() {
            public Iterator<V> iterator() {
                return new Iterator<V>() {
                    private Iterator<Entry<K,V>> i = entrySet().iterator();

                    public boolean hasNext() {
                        return i.hasNext();
                    }

                    public V next() {
                        return i.next().getValue();
                    }

                    public void remove() {
                        i.remove();
                    }
                };
            }

            public int size() {
                return AbstractMap.this.size();
            }

            public boolean isEmpty() {
                return AbstractMap.this.isEmpty();
            }

            public void clear() {
                AbstractMap.this.clear();
            }

            public boolean contains(Object v) {
                return AbstractMap.this.containsValue(v);
            }
        };
    }
    return values;
}
```

除了上面的方法, `AbstractMap`还实现了`equals`, `hashcode`, `toString`, `clone`等方法。