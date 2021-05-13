## JVM堆--年轻代与老年代

之前我们学习了, 堆内存主要是存储Java对象实例的区域, 而堆又可以细分为两类:

- 一类是生命周期较短的瞬时对象, 这类对象的创建和消亡都非常迅速;

- 另一类是生命周期非常长, 在某些情况下还能与JVM的生命周期保持一致。

这两个区域分别是, *年轻代(`YoungGen`)*, *老年代(`OldGen`)* 。其中年轻代又可以分为 `Eden`空间, `Survior0`空间, `Survior1`空间。

> `Survior`区有时又被称为 **from区, to区**。`Survior0`和`Survior1`哪个是空的, 那个区就成为to区。

![jvm_heap_old&young](/image/jvm_heap_old&young.png)

### 1. 配置年轻代和老年代结构占比

- 默认 **`-XX:NewRatio=2`**, 表示年轻代占1, 老年代占2, 年轻代占整个堆的1/3;

- 可以修改 **`-XX:NewRatio=4`**, 表示年轻代占1, 老年代占4, 年轻代占整个堆的1/5。

- `-XX:-UseAdaptiveSizePolicy`: 关闭自适应内存分配策略, 在JVM命令中 **`-`表示关闭**, **`+`表示打开**

- `-Xmn`: 设置新生代的空间大小。(一般不设置)

    ![jvm_heap_old&young1](/image/jvm_heap_old&young1.png)

    在`HotSpot`中, Eden区和另外两个Survivor区缺省所占比例是8:1:1, 我们可以通过`-XX:SurvivorRatio`调整空间比例, 如:`-XX:SurvivorRatio=8`。

- 几乎所有的Java对象new出来都是放在Eden区

- 大部分的Java对象都销毁在新生代


### 2. 对象分配过程

> 为新对象分配内存是非常严谨和复杂的任务, 不仅要考虑内存如何分配, 在哪里分配的问题, 并且由于内存分配算法与内存回收算法密切相关。所以还需要考虑GC执行完内存回收后是否会在内存空间中产生内存碎片。

下图是创建新对象时的堆内存分配情况:

![jvm_heap_objectallocation1](/image/jvm_heap_objectallocation1.png)


我们来分析下上图的过程(其中红色表示垃圾, 绿色表示不是垃圾对象):

1. 当创建新对象时, 先放入Eden区, 

2. 当Eden区空间不足时, 会触发一次`YGC/Minor GC(Young GC)`, JVM垃圾回收器会把Eden区中的垃圾(**不再被其他对象所有引用的对象**)进行销毁, 再把新对象放入Eden区

3. 接着把YGC过后, 不是垃圾的对象(**也就是经过YGC后幸存下来的对象**)移动到Survivor区(可能是0区, 也可能是1区)。并且给幸存对象标记一个分代年龄, 初始值为1。

    > 上图是移到`S0`区, 此时`S1`区是空的, 所以`S1`区成为To区, `S0`区称为From区

4. 如果还在不断的创建新的对象, 直到再次触发YGC时, 那么还是进行上面的步骤, 并且会清理Survivor区, 接着将幸存下来的对象放入到`S1`区, 更新分代年龄加1;

5. 当不断的进行上面的步骤, **直到有存活下来的对象分代年龄达到15, 那么这些对象就会移到老年代**;

    > - 分代年龄的阈值默认是15。
    > - 可以设置参数: `-XX:MaxTenuringThreshold=20`来设置

6. 而当老年代内存不足时, 则会触发`Major GC`, 清理老年代;

7. 如果老年代执行了`Major GC`后, 依然无法保存对象, 就会产生OOM异常。

> 根据上面的步骤, 对于垃圾回收会频繁在年轻代发生, 很少在老年代发生, 几乎不再永久代/元空间发生。

**<font color='red'>注意: 只有Eden区内存不足时才会触发`YGC/Minor GC`, 而Survivor区内存不足时绝对不会触发YGC, 它是被动触发的。</font>**

### 3. 对象分配特殊情况

![jvm_heap_objectallocation2](/image/jvm_heap_objectallocation2.png)

从上图可以看出, 正常情况下, 对象内存的分配就是上面的步骤。但是还是存在一些特殊情况。

1. 创建新对象时, 如果Eden区放不下, 则触发一次YGC; 而在YGC之后, Eden区如果还是放不下, 那么就直接把对象放在老年代Old区;

2. 而在进行YGC时, 将幸存下的对象放到`Survivor`区, 如果`Survivor`区也放不下, 那就直接把对象放到老年代Old区;

3. 如果老年代Old区也放不下, 则触发一次`Major GC`; 而在`Major GC`之后, 老年代还是放不下, 就会出现OOM。


我们用下面一段代码配合jvisualvm来观察堆空间内存情况:

先设置堆内存初始大小和最大内存为600m

```java
-Xmx600m -Xms600m
```

```java
public class HeapInstance {
    byte[] buffer = new byte[new Random().nextInt(1024 * 200)];

    public static void main(String[] args) {
        ArrayList<HeapInstance> list = new ArrayList<HeapInstance>();
        while (true) {
            list.add(new HeapInstance());
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

启动jvisualvm:

![jvm_heap_objectallocation3](/image/jvm_heap_objectallocation3.gif)

我们通过这个动图可以很清楚的看到, 堆内存中每个区的变化, 也正如我们上面所描述的对象分配过程一样。

![jvm_heap_objectallocation4](/image/jvm_heap_objectallocation4.png)

其中:

- `Eden Space`就是伊甸园区, 图像上每一次的谷底就是一次YGC;

- `Survivor`的变化可以看到, 每一次YGC`S0`区就与`S1`区进行清理并交换, 如果`Survivor`区也存不下, 那就存到`Old Gen`老年代。

### 4. 常用调优工具

下面是一些在实际开发中或者学习中常用的工具:

- `JDK命令行`

- `Jconsole`

- `Eclipse: Memory Analyzer Tool`

- `VisualVm`

- `Jprofiler`: 后面我们也会使用`Jprofiler`进行内存分析。

    > 先要去官网下载软件[Jprofiler](https://www.ej-technologies.com/products/jprofiler/overview.html), 然后在IDEA上安装Jprofiler插件。

- `GCViewer`

- `GC Easy`

### 5. Minor GC, Major GC, Full GC

> JVM在进行GC时, 并不是每次都针对上面三个内存区域(年轻代, 老年代, 方法区),大部分情况下都是在年轻代进行回收。

针对HotSpot VM的实现, 其中的GC按照回收区域分为两大种类: 一种是部分收集(Partial GC), 一种是整堆收集(Full GC)

1. **部分收集:** 不是完整收集整个Java堆的垃圾收集。

    - 年轻代回收(Young GC/Minor GC): 只针对老年代的垃圾回收;

    - 老年代回收(Major GC/Old GC): 只针对老年代的垃圾回收;

        > 注意: 很多时候`Major GC`和`Full GC`会混淆使用, 需要具体分辨老边带回收和整堆回收。

    - 混合回收(Mixed GC): 收集整个年轻代以及部分老年代的垃圾回收

2. **整堆收集(Full GC):** 整个Java堆和方法区的垃圾回收。

### 6. GC触发机制

#### 6.1 年轻代GC(YGC)触发机制

- 当年轻代中的Eden区空间不足时, 就会触发YGC, Survivor不足不会触发GC;

- Java大部分对象都是朝生夕死的, 所以YGC比较频繁, 一般回收速度也比较快;

- YGC/Minjor GC会引发`STW(Stop the World)`, 也就是暂停其他用户线程, 只有等待GC线程结束后, 用户线程才恢复运行。

![jvm_heap_objectallocation5](/image/jvm_heap_objectallocation5.png)

#### 6.2 老年代GC(Major GC/Full GC)触发机制

- 当老年代空间不足时, 会先尝试触发YGC, 如果YGC结束后, 年轻代空间还是不足, 则触发Major GC

- 发生`Major GC`, 一般都伴随至少一次YGC(不是绝对的);

- Major GC速度一般比YGC慢10倍以上, STW时间更长

- 如果Major GC之后, 老年代空间还是不足, 就会出现OOM。

#### 6.3 Full GC触发机制

触发`Full GC`一般有五中情况:

1. 调用`System.gc()`方法时, 系统建议执行Full GC, 但是不一定会执行;

2. 老年代空间不足;

3. 方法区空间不足;

4. 通过`Minor GC/YGC`后进入老年代的平均大小 **大于** 老年代的可用内存;

5. 当进行YGC把幸存对象复制到To区时, 如果Suvivor区空间不足, 则会把对象存到老年代, 此时如果老年代的可用内存小于该幸存对象的大小, 会触发Full GC

> `Full GC`是开发或调优中尽量要避免的, 这样用户线程等待时间会短一些。



