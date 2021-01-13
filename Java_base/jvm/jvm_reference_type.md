## JVm 引用的类型

引用类型的出现, 主要是为了满足这样的一个需求: **当内存空间还足够时, 这可以保存在内存中; 如果内存空间在垃圾回收之后还是很紧张, 则可以抛弃这些对象。**

根据上面的需求, JVM出现了4中引用: **强引用(`Strong Reference`), 软引用(`Soft Reference`), 弱引用(`Weak Reference`), 虚引用(`Phantom Reference`)。**

虽然有4种引用类型, 但是99%的情况下, 我们用的都是强引用, 间接使用弱引用(例如: ThreadLocal)。

### 1. 强引用: 不回收

这是最常见的一种引用类型, 使用new创建一个新的对象,并将其赋值给一个变量, 这个变量就是指向该对象的强引用。

垃圾回收器永远不会回收强引用对象。

> 强引用对象是可触及的, 无法被回收。所以**强引用是造成Java内存泄露的主要原因之一。**

### 2. 软引用: 内存不足即回收

软引用用来描述一些还有用, 但是非必须的对象。只被软引用关联的对象, 在系统将要发生内存溢出异常前，会把这些对象列进回收范围之中进行第二次回收，如果这次回收还没有足够的内存，才会抛出内存溢出异常。

**软引用通常用来实现内存敏感的缓存。** 比如：高速缓存就有用到软引用。如果还有空闲内存，就可以暂时保留缓存，当内存不足时清理掉，这样就保证了使用缓存的同时，不会耗尽内存。

java中使用`SoftReference`表示软引用。

### 3. 弱引用: 发现即回收

弱引用也是用来藐视非必须的对象, 被弱引用关联的对象只能生存到垃圾回收。

**在GC时, 只要发现弱引用, 不管内存是否足够, 都会直接回收。**

java中使用`WeakReference`表示弱引用。

> 在4中引用类型中, 弱引用应该算是比较重要的。因为它的使用场景出现在多线程中, 也就是`ThreadLocal`的源码中, 所以面试经常会问。

可以在[ThreadLocal源码解析](/Java_base/concurrent/promote/thread_threadlocal.md)这篇文章中了解更多。

```java
public class ThreadLocal<T> {
    ...

    static class ThreadLocalMap {

        /**
         * The entries in this hash map extend WeakReference, using
         * its main ref field as the key (which is always a
         * ThreadLocal object).  Note that null keys (i.e. entry.get()
         * == null) mean that the key is no longer referenced, so the
         * entry can be expunged from table.  Such entries are referred to
         * as "stale entries" in the code that follows.
         */
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }

        ...
    }

    ...
}
```

### 4. 虚引用: 对象回收跟踪

虚引用(Phantom Reference),也称为“幽灵引用”或者“幻影引用”，是所有引用类型中最弱的一个。

一个对象是否有虚引用的存在，完全不会决定对象的生命周期。如果一个对象仅持有虚引用，那么它和没有引用几乎是一样的，随时都可能被垃圾回收器回收。