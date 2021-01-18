## JVM调优 GC日志分析

### 1. 打印日志命令

- `-XX:+PrintGC`输出GC日志。类似`-verbose:gc`;

- `-XX:+PrintGCDetails`输出GC的详细日志;

- `-XX:+PrintGCTimeStamps`输出GC的时间戳;

- `-XX:+PrintGCDateStamps`以日期形式输出GC时间戳;

- `-XX:+PrintHeapAtGC`在进行GC的前后打印堆信息;

- `-Xloggc: ../logs/gc.log` 日志文件的输出路径

### 2. 日志参数分析

利用`-XX:+PrintGC`命令打印一段GC日志:

```java
[GC （Allocation Failure） 80832K一>19298K（227840K），0.0084018 secs]
[GC （Metadata GC Threshold） 109499K一>21465K （228352K），0.0184066 secs]
[Full GC （Metadata GC Threshold） 21 465K一>16716K （201728K），0.0619261 secs ]
```

- GC, Full GC: GC的类型, GC只在年轻代进行, Full GC包括年轻代, 老年代, 永久代。

- Allocation Failue: GC发生的原因;

- 80832k->19298k: 堆在GC前的大小和GC后的大小;

- 227840k: 现在堆的总大小;

- 0.0084018 secs: GC持续的时间;

我们利用`-XX:+PrintGCDetails`可以查看更详细的GC信息:

```java
[GC （Allocation Failure） [ PSYoungGen： 70640K一> 10116K（141312K） ] 80541K一>20017K （227328K），0.0172573 secs] [Times： user=0.03 sys=0.00， real=0.02 secs ]
[GC （Metadata GC Threshold） [PSYoungGen：98859K一>8154K（142336K） ] 108760K一>21261K （228352K），
0.0151573 secs] [Times： user=0.00 sys=0.01， real=0.02 secs]
[Full GC （Metadata GC Threshold） [PSYoungGen： 8154K一>0K（142336K） ] [ParOldGen： 13107K一>16809K（62464K） ] 21261K一>16809K （204800K），[Metaspace： 20599K一>20599K （1067008K） ]，0.0639732 secs]
[Times： user=0.14 sys=0.00， real=0.06 secs]
```

- PSYoungGen： 70640K一> 10116K（141312K）: 使用了`Parallel Scavenge`并行垃圾收集器, 年轻代GC前后的大小, 以及堆内存的总大小;

- ParOldGen：使用了Parallel Old并行垃圾收集器的老年代Gc前后大小的变化;

- Metaspace: 元数据区GC前后的大小变化;

- xxxx secs: GC花费的时间;

- Times: user: 指的是垃圾收集器花费所有CPU的时间, sys: 花费在等待系统调用的时间, real: GC从开始到结束的时间。

### 3. 日志补充说明

- [GC], [Full GC]说明了此次垃圾回收的类型, 如果有"Full GC"则说明GC是发生了STW;

- 使用Serial收集器在年轻代的名字是`Defaule New Generation`, 所以显示`[DefNew]`;

- 使用G1收集器的话, 会显示为: `garbage-first heap`;

### 4. 堆空间占用情况

![jvm_tuning1](/image/jvm_tuning1.png)

#### 4.1 Minor GC

![jvm_tuning2](/image/jvm_tuning2.png)

#### 4.2 Full GC

![jvm_tuning3](/image/jvm_tuning3.png)