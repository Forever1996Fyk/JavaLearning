## JVM垃圾回收器

### 1. Serial收集器


`Serial`收集器是HotSpot中Client模式下的默认年轻代垃圾收集器。

`Serial`采用**复制算法**, 是一个串行回收和STW机制的方式执行内存回收。

> 这个收集器是单线程的, 这说明它只会使用一个GC线程执行垃圾回收工作, 而且GC执行时必须要暂停用户线程。


与`Serial`相对应的就是`Serial Old`收集器。

`Serial Old`是提供用于执行老年代垃圾收集的, 也是采用串行回收和STW机制。只不过内存回收算法使用的是**标记压缩算法**。

Serial Old不仅是与Serial配合使用的老年代收集器, 而且也是作为CMS收集器的后背方案。

![jvm_gc_collector_detail1](/image/jvm_gc_collector_detail1.png)

### 2. ParNew收集器

`Serial GC`是年轻代中的单线程垃圾收集器, 而`ParNew GC`则是多线程垃圾收集器; 同样也是年轻代的收集器。

> Par是Parallel的缩写, New: 只能处理年轻代

`ParNew`收集器除了采用并行回收方式之外, 与`Serial`没有区别, 也是采用**复制算法**和STW机制。

![jvm_gc_collector_detail2](/image/jvm_gc_collector_detail2.png)

对于年轻代而言, 回收次数很频繁, 使用并行的方式会更加高效。而老年代, 回收次数少, 使用串行方式可以节省资源。

虽然`ParNew`收集器是基于并行回收, 但是效率并不是任何场景都很高效。

- 在多CPU环境下, 由于可以充分利用多CPU等物理硬件的优势, 可以更快速的完成垃圾收集, 提高程序的吞吐量。

- 但是在单CPU环境下, `ParNew`不比`Serial`更高效。虽然`Serial`是串行回收, 但是不需要频繁的进行任务切换。

- `ParNew GC`也可以配合`CMS`收集器。

- 在程序中，开发人员可以通过选项"`-XX： +UseParNewGC`"手动指定使用ParNew收集器执行内存回收任务。它表示年轻代使用并行收集器，不影响老年代。

- `-XX：ParallelGCThreads`限制线程数量，默认开启和CPU数据相同的线程数。

### 3. Parallel收集器(吞吐量优先)

`Parallel Scavenge GC`与`ParNew`收集器都是基于并行回收的, 并且也是采用了复制算法和STW机制。那么为什么有了`ParNew`还要有`Parallel Sacvenge`呢?

> **`Parallel Scavenge`收集器的目标是达到一个可控的吞吐量, 也就是说是以吞吐量优先的垃圾收集器。** `Parallel Scavenge`收集器的自适应调节策略也是与`ParNew`的一个重要区别。

高吞吐量是高效率的利用CPU时间, 尽快完成程序的任务。**主要适合后台运算而不需要太多的交互任务。例如: 批量处理, 订单处理, 支付等应用程序。**

与`Parallel Scavenge`相对应的就是`Parallel Old`垃圾回收器。简称`PS` - `PO`。

`Parallel Old`收集器采用**标记压缩算法**, 也是基于并行回收和STW的机制。

> Java8, 默认PS - PO垃圾回收器

![jvm_gc_collector_detail3](/image/jvm_gc_collector_detail3.png)

#### 3.1 Parallel垃圾回收器参数配置

- `-XX:+UseParallelGC`手动指定年轻代使用PS收集器;

- `-XX:+UseParallelOldGC`手动指定老年代使用PO收集器;

    - Java8 默认PS, PO;

    - PS, PO是相互激活的。

- `-XX:ParallelGCThreads`,设置年轻代PS收集器的线程数。**一般最好与CPU数量相等, 避免线程切换带来的性能问题。**

    - 在默认情况下, 当CPU数量小于8个, PS Thread的值等于CPU数量;

    - 当CPU数量大于8个, PS Threads的值等于**3 + ((5 * CPU_Count)/8)**


- `-XX:MaxGCPauseMills`设置垃圾收集器醉倒停顿时间(即STW的事件)。单位是毫秒。

    > 对于用户来说, 停顿时间越短体验越好, 但是在服务端, 我们注重高并发, 整体的吞吐量, 所以服务器端适合`Parallel`进行控制。

- `-XX:GCTimeRatio`垃圾收集时间占总时间的比例, 用于衡量吞吐量的大小;

    - 取值范围(0, 100)。默认是99。

    - 该参数与`MaxGCPauseMills`是相矛盾的, 所以当我们设定`MaxGCPauseMills`的值时, 可能会影响这个比例。

- `-XX+UseAdaptiveSizePolicy`设置PS收集器的自适应调节策略。

    - 在这种模式下, 年轻代的大小, Eden和Survivor的比例, 晋升老年代的分代年龄等参数会被自动调整, 来达到堆大小, 吞吐量和停顿时间的平衡点。

