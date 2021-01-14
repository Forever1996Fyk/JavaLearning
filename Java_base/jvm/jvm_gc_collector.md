## JVM垃圾回收器基本知识

> 垃圾收集器没有固定的规范, 可以有不同的厂商, 不同版本的JVM实现。从不同角度分析垃圾收集器, 可以将GC分为不同的类型。


### 1. GC逻辑分类

#### 1.1 按线程数分类

这里的线程数指的是垃圾回收线程数。

- 串行垃圾回收器: 在同一时间段内只允许一个CPU用于执行垃圾回收操作, 此时用户线程暂停, 直到垃圾回收工作结束。

- 并行垃圾回收器: 并行收集可以运用多个CPU同时执行垃圾回收。因此提高了应用的吞吐量, 但是与串行回收一样, 采用独占式, 用户线程依然会被暂停。

![jvm_gc_collector1](/image/jvm_gc_collector1.png)

#### 1.2 按照工作模式分类

- 独占式垃圾回收器: 一旦GC线程执行, 就停止应用程序中的所有用户线程, 直至垃圾回收完全结束。

- 并发誓垃圾回收器: GC线程与用户线程交替工作, 来尽可能减少应用程序的停顿时间。

![jvm_gc_collector2](/image/jvm_gc_collector2.png)


#### 1.3 按碎片处理方式分类

- 压缩式垃圾回收器: 在垃圾清理完成后, 对存活对象进行压缩整理, 不存在内存碎片。

    > 分配对象空间使用: 指针碰撞

- 非压缩式垃圾回收器: 不会对内存碎片进行整理

    > 分配对象空间使用: 空闲列表

#### 1.4 按工作内存区间分

- 年轻代垃圾回收器

- 老年代垃圾回收器

### 2. 评估GC的性能指标

- **吞吐量: 运行用户代码的时间占总运行时间的比例。**

- 垃圾收集开销: 吞吐量的补数, 垃圾收集所用时间占总运行时间的比例。

- **暂停时间/低延迟: 执行垃圾收集时, 程序的工作线程被暂停的时间。**

- 收集频率: 相对于应用程序的执行, 收集操作发生的频率。

- **内存占用: Java堆区所占的内存大小**


> 一般来说, 最重要的两点是: **<font color='red'>吞吐量, 暂停时间(低延迟)</font>**

#### 2.1 吞吐量

吞吐量是**用户代码的运行时间与总运行时间的比值。**

**<font color='red'>吞吐量 = 用户代码运行时间 / (用户代码运行时间 + 垃圾收集时间)</font>**

> 例如: JVM总共运行100分钟, 其中垃圾收集花费1分钟, 那么吞吐量就是 99%。

这种情况下, 应用程序能容忍较高的暂停时间, 并不是特别要求低延迟。**用户交互不频繁的场景, 例如: 长时间的后台计算**

#### 2.2 低延迟(暂停时间)

我们知道GC时, 会导致STW。

所以暂停时间是指, 一个时间段内应用程序线程暂停, 让GC线程执行的状态。

**所谓低延迟, 就是尽可能的让单次STW的时间最短。**


#### 2.3 高吞吐与低延迟对比

![jvm_gc_collector3](/image/jvm_gc_collector3.png)

- **高吞吐量**: 注重用户线程执行的时间;

- **低延迟**: 注重响应时间。

"高吞吐量"和"低延迟"是一对相互矛盾的目标。

> - 如果选择吞吐量优先, 那么必然需要降低内存回收的执行频率, 但是这样就会导致GC需要更长的暂停时间来执行垃圾回收(因为垃圾存储过多)。
> - 吐过选择低延迟优先, 那么为了降低每次回收的暂停时间, 也只能频繁的执行垃圾回收, 这就导致程序吞吐量下降。


目标在设计GC算法时, 只可能针对一个目标, 或者 **在满足一定的响应时间的情况下，要求达到多大的吞吐量。**

### 3. 垃圾回收器与分代的关系

- **年轻代收集器:** `Serial`, `ParNew`, `Parallel Scaenge`;

- **老年代收集器:** `Serial Old`, `Parallel Old`, `CMS`;

- **整堆收集器:** G1。

![jvm_gc_collector4](/image/jvm_gc_collector4.png)

### 4. 垃圾收集器的组合关系

![jvm_gc_collector5](/image/jvm_gc_collector5.png)


- Serial - Serial Old
- ParNew - CMS ~ Serial Old
- Parallel Sacvenge - Parallel Old

针对不同的场景, 提供不同的垃圾收集组合, 提高垃圾收集性能。

### 4. 查看默认的垃圾收集器

有两种方法:

1. 使用命令`-XX:+PrintCommandLineFlags`

2. 命令行参数: jinfo -flag Use垃圾收集器种类 进程id

例如下面代码:

```java
/**
 * -XX:+PrintCommandLineFlags
 *
 *
 */
public class GCUseDemo {
    public static void main(String[] args) {
        ArrayList<byte[]> list = new ArrayList<>();

        while(true){
            byte[] arr = new byte[100];
            list.add(arr);
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

结果为:

```txt
-XX:InitialHeapSize=266742208 -XX:MaxHeapSize=4267875328 -XX:+PrintCommandLineFlags -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:-UseLargePagesIndividualAllocation -XX:+UseParallelGC 
```

- -XX:+UseSerialGC:表明新生代使用Serial GC ，同时老年代使用Serial Old GC

- -XX:+UseParNewGC：标明新生代使用ParNew GC

- -XX:+UseParallelGC:表明新生代使用Parallel GC, 使用Parallel GC同时也会激活老年代使用Parallel Old GC

- -XX:+UseParallelOldGC : 表明老年代使用 Parallel Old GC

-XX:+UseConcMarkSweepGC：表明老年代使用CMS GC。同时，年轻代会触发对ParNew GC的使用

使用命令行查看垃圾收集器:

```java
jps

jinfo -flag UseParallelGC 进程Id

结果:
-XX:+UseParallelGC

```

