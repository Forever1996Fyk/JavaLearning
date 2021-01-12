## JVM垃圾回收标记算法

垃圾回收算法是垃圾回收器的核心部分, 理解其中的算法原理, 才能学习后面的各种垃圾回收器。

### 1. 垃圾标记阶段

在堆中存放的几乎都是Java对象的实例, 在GC执行垃圾回收前, 首先需要区分出内存中哪些是垃圾, 并进行标记。这个过程称为垃圾标记阶段。

**我们知道判断是否是垃圾? 简单来说, 就是一个对象已经不再被任何存活对象继续引用。**

判断对象存活的两种方式:

- 引用计数法(Reference Count)

- 可达性分析算法/根搜索方法(Root Searching)

#### 1.1 引用计数法(Java没有采用)

**引用计数法**是比较简单的方法, 对每个对象保存一个整型的引用计数器, 用于记录对象被引用的情况。

对于一个对象A, 只要有任何一个对象引用了A, 则A的计数器就加1; 当引用失效时, 引用计数器就减1.只要对象A的引用计数器的值为0, 即表示对象A不可能再被使用, 可进行回收。

优点: 

- 实现方式简单, 垃圾对象容易辨识; 

缺点:

- 需要单独的字段存储计数器, 增加了存储空间的开销;

- **无法处理循环引用问题。这也是Java没有采用的原因!!!**

循环引用导致内存泄露问题:

![jvm_reference_count](/image/jvm_reference_count.png)

> 虽然引用计数法会引起循环引用导致的内存泄露问题, 但是在面试的时候如果遇到内存泄露的场景,最好不要引用计数法举例。因为Java没有用这个算法!!!

#### 1.2 可达性分析算法

也称为**根搜索算法**, **根可达算法**或追踪性垃圾收集。

相对于引用计数法而言, 可达性分析算法依然实现简单, 执行高效。最主要的是该算法可以很好的解决循环引用问题, 防止内存泄露。

- **可达性分析算法是以根对象(`GCRoots`)为起始点, 按照从上至下的方式搜索被根对象集合所引用的目标对象是否可达。**

- 经过可达性分析算法后, 内存中存活的对象都会被根对象集合直接或间接的连接, 搜索走过的路径称为引用链(Reference Chain)

- 如果目标对象没有任何引用链相连, 则是不可达的, 就意味着该对象已经死亡, 可以标记为垃圾对象。

![jvm_root_search](/image/jvm_root_search.png)


### 2. GC Roots

在Java中, GC Roots对象如下:

1. 虚拟机栈中引用的对象: 例如, 每个栈帧中使用的 参数, 局部变量等;

2. 本地方法栈内JNI引用的对象;

3. **方法区中类静态属性**引用的对象: 例如, Java类的引用类型静态变量;

```java
Class Dog {
    // 该对象也是根对象
    private static Object obj;
}
```

4. **方法区中常量**引用对象: 例如, 字符串常量池的引用;

```java
Class Dog {
    private final Object obj;
}
```

5. 所有被同步锁`synchronized`持有的对象;

6. Java虚拟机内部的引用: *基本数据类型对应的Class对象(Integer, Long等包装类)*, *常驻异常对象(NullPointerException, OutOfMemoryError)*, *系统类加载器*;

> 由于Root采用栈方式存放变量和指针， 所以如果一个指针指向堆内存中的对象, 但是自己不存放在堆内存中, 那么它就是一个Root。

![jvm_gc_roots](/image/jvm_gc_roots.png)

**要注意的是:**

- 如果要使用可达性分析算法来判断内存是否可回收, 那么分析工作必须在一个能保障一致性的快照中进行。如果这点不满足的话, 那么分析的准确性就无法保证。**这也是GC时, 必须STW的主要原因!!!**

    > 因为很有可能刚刚才通过算法分析出的垃圾对象可能会"复活"。

### 3. finalization机制

Java语言提供了对象终止(`finalization`)机制, 允许开发人员提供对象被回收之前的自定义处理逻辑, 也就是调用`finalize()`方法。

`finalize()`方法允许在子类中被重写, 用于在对象被回收时进行资源释放。通常在这个方法中进行一些资源释放和清理工作, 比如关闭文件, socket连接和数据库连接等。

**永远不要主动调用某个对象的`finalize()`方法, 应该由垃圾回收机制调用。** 原因有三点:

1. `finalize()`方法会在一个低优先级的线程中等待执行(前提是发生了GC), 不需要人为主动调用;`finalize()`编写不好会影响GC的性能。

2. 调用`finalize()`方法时可能会导致对象复活

3. `finalize()`方法的执行时间没有保障, 完全由GC线程决定, 如果不发生GC, 也不会执行finalize()方法。

### 4. 对象状态

由于`finalize()`方法的存在, JVM中的对象可能回处于三种状态:

1. **可触及:** 也就是通过可达性分析算法可以到达的对象;

2. **可复活:** 该对象已经没有引用指向它, 但是有可能在`finalize()`方法中再次被指向;

3. **不可触及:** <font color='red'>对象的`finalize()`被调用, 并且此时依然没有引用指向它, 那么就会进入不可触及状态。</font> 不可触及的对象不可能被复活, 因为`finalize()`只会被调用1次。

**判定是否可以回收, 至少要经历两次标记过程:**

过程1: 如果对象A到GC Roots没有引用链, 则进行第一次标记;

过程2: 进行筛选, 判断此对象是否有必要执行`finalize()`方法;

- 如果对象A没有重写finalize()方法, 或者finalize()方法已经被JVM调用过, 那么JVM就会判断对象A是不可触及的;

- 如果对象A重写了finalize()方法, 并且还未执行过, 那么对象A会被插入到一个`Finalization-Queue`(简写为`F-Queue`)队列中, 这是有虚拟机自动创建的, 低优先级的Finalizer线程触发finalize()方法执行。

- finalize()方法是对象逃脱死亡的最好机会, 之后GC会对`F-Queue`队列中的对象进行第二次标记。如果对象A的finalize()方法中与引用链上的任何对象建立了联系, 那么第二次标记时对象A会被移出"即将回收"的集合; 如果执行finalize()方法后, 对象A依然没有建立引用链, 那么就会变成不可触及状态, 被JVM回收。


### 4. 使用MAT与Jprofiler工具分析GCRoots

MAT是Memory Analyzer的简称, 用于查找内存泄露以及查看内存消耗情况。(Eclipse开发)。

#### 4.1 获取dump文件

dump文件其实就是在GC时的快照文件, 会记录当前GC时的具体信息。获取方式有两种:

1. 命令行使用`jmap`

    - `jps` 

    - `jmap -dump:format=b,live,file=test1.bin {进程id}`

2. 使用jvisualVM导出

捕获的heap dump文件是一个临时文件, 关闭jvisualVM后会自动删除, 如果想保留, 需要另存为本地文件。

在`jvisualVM` -> `Monitor(监视)`窗口 -> 点击`堆 Dump`按钮

![jvm_dump1](/image/jvm_dump1.png)

![jvm_dump2](/image/jvm_dump2.png)

#### 4.2 GC Roots分析

我们执行下面一段代码:

```java
public class GCDumpDemo {

    public static void main(String[] args) {
        List<Object> numList = new ArrayList<>();
        Date birth = new Date();

        for (int i = 0; i < 100; i++) {
            numList.add(String.valueOf(i));
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

        System.out.println("数据添加完毕，请操作：");
        new Scanner(System.in).next();
        numList = null;
        birth = null;

        System.out.println("numList、birth已置空，请操作：");
        new Scanner(System.in).next();

        System.out.println("结束");
    }
}
```

生成dump文件, 并在MAT或者Jprofiler中打开。我们这里使用Jprofiler打开:

![jvm_dump_jprofiler1](/image/jvm_dump_jprofiler1.png)

![jvm_dump_jprofiler2](/image/jvm_dump_jprofiler2.png)

![jvm_dump_jprofiler3](/image/jvm_dump_jprofiler3.png)

根据上图步骤即可查看dump文件的GCRoots对象。

#### 4.3 JProfiler分析OOM

在运行时我们使用`-XX:+HeapDumpOnOutOfMemoryError`命令, 当程序出现OOM时, 就会生成一个dump文件。

![jvm_oom_jprofiler1](/image/jvm_oom_jprofiler1.png)

![jvm_oom_jprofiler1](/image/jvm_oom_jprofiler2.png)





