## JVM运行时数据区--堆(heap)

**堆(`heap`)** 应该是JVM技术栈中最核心的部分了, 它涉及的知识点很多例如 GC, 堆内存布局, 对象分配, 性能调优等。

这篇文章主要是对**堆**的概述, 后面会针对每个知识点重点分析。

### 1. 堆的基本知识

在正式学习堆之前, 我们要先了解一些基本的知识, 也是对堆知识的扫盲。

> 一个进程对应一个JVM实例, 一个进程中有多个线程, 是共享运行时数据区的方法区和堆, 每个线程又包含私有的程序计数器, 本地方法栈和虚拟机栈。

1. 一个JVM实例只存在一个堆内存, 堆也是Java内存管理的核心区域。

2. JVM堆是在JVM启动的时候就已经被创建了, 空间大小都已经确定了。JVM管理最大的一块内存空间, 但是堆内存的大小是可以调节的。

3. **虽然堆分为多个区空间(年轻代, 老年代, 永久代/元空间), 但是它们在物理上是不连续的, 但是一般在逻辑上我们认为它们都是连续的空间都是属于堆空间。**

    > 所以只要一说到堆空间, 那就是代表 年轻代, 老年代, 永久代/元空间。

4. 这里还有明确一点, **<font color='red'>堆是线程共享的空间, 但是不一定堆中所有的空间都是线程共享的!!!</font>** 
因为堆空间中还存在线程私有的缓冲区`TLAB`(Thread Local Allocation Buffer)

5. 我们知道在创建对象时, 真正对象的实例是保存在堆中的, 栈中保存的是指向这个实例的地址; 当方法结束后, 栈中保存的这个地址就被清除了。

   **<font color='red'>当没有任何地方有指向这个堆中的对象的地址时, 堆中的对象也不会马上被移除, 而是只有发生垃圾回收的时候才会被移除。</font>**

#### 1.1 查看JVM进程

在jdk目录中, `jdk1.8/Contents/Home/bin`下找到jvisualvm运行, 或者直接终端通过`jvisualvm`命令, 查看进程。

我们执行下面代码:

```java
public class HeapJvisualVm {

    public static void main(String[] args) throws InterruptedException {
        System.out.println("start....");
        TimeUnit.SECONDS.sleep(10000);
        System.out.println("end....");
    }
}
```

在终端使用`jvisualvm`命令, 就可以打开visualvm工具。

![jvm_jvisualvm](/image/jvm_jvisualvm.png)

我们还可以安装插件查看更多的信息, 比如`Visual GC`

#### 1.2 简单分析JVM中的Heap

```java
public class SimpleHeap {
    private int id;//属性、成员变量

    public SimpleHeap(int id) {
        this.id = id;
    }

    public void show() {
        System.out.println("My ID is " + id);
    }
    public static void main(String[] args) {
        SimpleHeap sl = new SimpleHeap(1);
        SimpleHeap s2 = new SimpleHeap(2);

        int[] arr = new int[10];

        Object[] arr1 = new Object[10];
    }
}
```

上面代码在JVM中的局部如下图:

![jvm_heap1](/image/jvm_heap1.png)

正如我们之前分析的一样, **Java虚拟机栈存放指向实例的地址, 堆内存存放实例本身, 而具体类的信息存放在方法区。**

#### 1.3 堆内存细分内存结构

JDK7以前: **逻辑上**分为 *年轻代*、*老年代*、*永久代*。但是在物理上, 我们分配内存时不会涉及永久代。(例如使用Xmx/Xms命令分配内存)

- `Young Generation Space`: 年轻代, 又被分为Eden区和Survior区;

- `Tenure Generation Space`: 老年代Old/Tenure;

- `Permanent Space`: 永久代Prem。

![jvm_heap2](/image/jvm_heap2.png)

JDK8之后: 在逻辑上分为 *年轻代*、*老年代*、*元空间*。

- `Meta Space`

#### 1.4 设置堆内存大小与OOM

Java堆是用于村粗Java对象实例, 堆的大小在JVM启动时就已经设定好了, 我们可以用个`-Xmx`和`-Xms`来进行设置。

- **`-Xms`用来设置堆空间(老年代+年轻代)的初始内存大小, 等价于`-XX:InitialHeapSize`**

    - `-X` 是JVM运行参数;

    - `ms` 是memory start。


- **`-Xmx`用来设置堆的最大内存, 等价于`-XX:MaxHeapSize`**

    - 默认情况下, 初始内存大小为 **物理内存大小/64**; 最大内存大小为 **物理内存大小/4**。

    - 手动设置: `-Xmx600m`, `-Xms600m`; 在IDEA中配置`Run Edit Configuration`->`VM Options`添加参数即可

    > 一旦堆的内存大小超过了`-Xmx`所指定的最大内存时, 就会抛出OOM异常

#### 1.5查看堆内存大小

在Java程序运行时, 我们在终端通过`jps`命令, 可以看到运行时java程序的进程id; 再通过`jstat -gc 进程id`命令打印堆内存信息。

我们设置堆内存大小为10m, `-Xmx10m -Xms10m`:

```java
public class HeapSpace {

    public static void main(String[] args) {
        //返回Java虚拟机中的堆内存总量
        long initialMemory = Runtime.getRuntime().totalMemory() / 1024 / 1024;
        //返回Java虚拟机试图使用的最大堆内存量
        long maxMemory = Runtime.getRuntime().maxMemory() / 1024 / 1024;

        System.out.println("-Xms : " + initialMemory + "M");//-Xms : 245M
        System.out.println("-Xmx : " + maxMemory + "M");//-Xmx : 3641M

        System.out.println("系统内存大小为：" + initialMemory * 64.0 / 1024 + "G");//系统内存大小为：15.3125G
        System.out.println("系统内存大小为：" + maxMemory * 4.0 / 1024 + "G");//系统内存大小为：14.22265625G

        try {
            Thread.sleep(1000000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

打印结果为:

```java
-Xms : 9M
-Xmx : 9M
系统内存大小为：0.5625G
系统内存大小为：0.03515625G
```

我们使用**jps查看进程:**

```java
yukaifan@192 jvm % jps

//进程id  程序名称
  29203  Launcher
  29204  HeapSpace
  29214  Jps
```

再通过 **`jstat -gc 进程id`查看JVM堆内存信息:**

```java
yukaifan@192 jvm % jstat -gc 29204

 S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT     GCT   
512.0  512.0   0.0   496.0   2048.0   532.5     7168.0     228.0    4864.0 3436.0 512.0  379.8       1    0.001   0      0.000    0.001

```

我们先简单解释下, 上面JVM内存信息的意思:

- `S0C`, `S1C`分别表示`Survior0区`, `Survior1区`所占总内存;

- `S0U`, `S1U`分别表示`Survior0区`, `Survior1区`已用内存; 我们发现Survior0, Survior1始终会有一个是空的;

- `EC`, 表示新生代所占总内存;

- `EU`, 表示新生代已用内存;

- `OC`, 表示老年代所占总内存;

- `OU`, 表示老年代已用内存。


> 我们把`S0C`, `S1C`, `EC`, `OC`全部加起来, 正好等于10M, 也是我们设置的堆内存大小。但是运行程序时控制台打印的是9M, 是因为其中`Survior区`只有一个占用了内存, 另一个没有使用, 所以没有算上去。