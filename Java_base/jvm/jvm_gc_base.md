## JVM--GC相关概念解释

### 1. System.gc()的理解

通过调用`System.gc()`或者`Runtime.getRuntime().gc()`, 会显式触发Full GC, 同时对老年代和年轻代进行回收, 尝试释放垃圾。

```java
public final class System {
    public static void gc() {
        Runtime.getRuntime().gc();
    }
}
```

> 但是`System.gc()`无法保证对垃圾收集器的调用(无法保证马上触发GC)。

`System.runFinalization()`这个方法的调用, 会让JVM强制调用垃圾对象的`finalize()`方法。

```java
public class SystemGCTest {
    public static void main(String[] args) {
        new SystemGCTest();
        System.gc();//提醒jvm的垃圾回收器执行gc,但是不确定是否马上执行gc
        //与Runtime.getRuntime().gc();的作用一样。
        System.runFinalization();//强制调用 失去引用的对象的finalize()方法
    }

    @Override
    protected void finalize() throws Throwable {
        super.finalize();
        System.out.println("SystemGCTest 重写了finalize()");
    }
}
```

如果注掉System.runFinalization(); 那么控制台不保证一定打印,

从而证明了System.gc()无法保证GC一定执行。


### 2. 内存溢出与内存泄露

这两种情况都会导致OOM的出现。

#### 2.1 内存溢出

内存溢出相对是比较容易理解的, 就是没有空闲内存, 而且在垃圾收集之后, 也无法提供空闲的内存。

原因一般有两点:

1. Java虚拟机的堆内存设置不够

    如果堆内存的大小不合理,比如我们要处理数据量较大的情况, 没有显示的指定JVM堆大小很有可能出现OOM。我们可以通过`-Xmx`, `-Xms`来调整最大和初始内存

2. 代码中创建了大量大对象, 并且长时间不能被GC(存在引用链)。

    - JDK1.7之前, 一般存活较长的大对象都是存储在永久代, 而永久代的大小是有限的, 并且JVM对永久代垃圾回收并不积极, 所以如果不断的添加对象, 永久代出现OOM也很常见:
    `java.lang.OutOfMemoryError:PermGen space`。

    - JDK1.8之后, 随着元空间的引用, 由原来的虚拟机内存永久代, 改变为本地内存的元空间, 以及常量池, 静态变量的移动到堆内存中。元空间的大小也取决于本地内存的大小, 所以OOM的情况已经有所改善。
    本地内存不足, 也会OOM:`java.lang.OutOfMemoryError:Metaspace`。

#### 2.2 内存泄露(Memory Leak)

当对象不再被程序使用到, 但是GC有无法将这些对象回收, 就称为内存泄露。

> 但是这种情况不太好发现, 所以一般情况下, 对象周期很长而导致的OOM, 也可以称为"内存泄露"

**尽管内存泄漏并不会立刻引起程序崩溃，但是一旦发生内存泄漏，程序中的可用内存就会被逐步蚕食，直至耗尽所有内存，最终出现OutOfMemory异常，导致程序崩溃。**

![jvm_memory_leak](/image/jvm_memory_leak.png)

### 3. STW

Stop The World简称STW。意思是发生GC时, 会产生用户线程(应用程序)的停顿, 没有任何响应, 有点像卡死的感觉。

例如: 垃圾标记阶段时, 会导致所有用户线程停顿。

> 所以我们开发中不能显示的调用`Systemm.gc()`方法, 因为它会触发GC, 导致STW。

**为什么会停顿?**

因为在GC阶段, 对垃圾的分析和清除都必须**确保在某个时间点上的一致性。**这样分析出来的结果来能保证准确。

> 任何垃圾回收器都会产生STW现象。

### 4. GC的并行与并发

之前我们提到过程序的并行与并发。

**并行(parallel)**: 多个CPU在**同一个时间点**, 同时执行任务; 

![jvm_parallel1](/image/jvm_parallel1.png)


**并发(concurrency)**: 一个CPU在**一个时间段**内, "同时执行任务"。这里的同时进行, 只是CPU把一个时间段分为几个时间片, 快速的切换任务。

![jvm_concurrency1](/image/jvm_concurrency1.png)

二者对比:

- 并发，指的是多个事情，在同一时间段内同时发生了。
- 并行，指的是多个事情，在同一时间点上同时发生了。
- 并发的多个任务之间是互相抢占资源的。
- 并行的多个任务之间是不互相抢占资源的。
- 只有在多CPU或者一个CPU多核的情况中，才会发生并行。否则，看似同时发生的事情，其实都是并发执行的。


**并行垃圾回收器**: **多条垃圾回收线程并行工作**, 但是用户线程还是处于STW状态。例如: `ParNew`、 `Parallel Scavenge`、 `Parallel 0ld`;

**串行垃圾回收器**: 单线程执行;`Serial GC`

![jvm_parallel2](/image/jvm_parallel2.png)

![jvm_parallel3](/image/jvm_parallel3.png)

**并发垃圾回收器**: 用户线程与垃圾收集线程同时执行(不是同一时间, 而是交替执行)。 例如: `CMS`, `G1`。
