## JVM堆--年轻代与老年代

之前我们学习了, 堆内存主要是存储Java对象实例的区域, 而堆又可以细分为两类:

- 一类是生命周期较短的瞬时对象, 这类对象的创建和消亡都非常迅速;

- 另一类是生命周期非常长, 在某些情况下还能与JVM的生命周期保持一致。

这两个区域分别是, *年轻代(`YoungGen`)*, *老年代(`OldGen`)*。其中年轻代又可以分为 `Eden`空间, `Survior0`空间(或者From区), `Survior1`空间(或者To区)。

![jvm_heap_old&young](/image/jvm_heap_old&young.png)

### 1. 配置年轻代和老年代结构占比

- 默认 **`-XX:NewRatio=2`**, 表示年轻代占1, 老年代占2, 年轻代占整个堆的1/3;

- 可以修改 **`-XX:NewRatio=4`**, 表示年轻代占1, 老年代占4, 年轻代占整个堆的1/5。

![jvm_heap_old&young1](/image/jvm_heap_old&young1.png)