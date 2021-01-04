## JVM运行时数据区————方法区

我们先从是否线程共享区域的角度来看, 方法区的位置:

![jvm_methodarea1](/image/jvm_methodarea1.png)

> 要注意的是**方法区**是抽象的运行时数据区的概念, 相当于java中的接口, 而具体真正实现方法区就是元空间(jdk1.7之前是永久代)

### 1. 堆, 栈, 方法区的交互关系

这张图非常直白的显示了, 三者之间的关系。

![jvm_methodarea2](/image/jvm_methodarea2.png)

> 方法区主要存放的是类的相关信息, 堆区存放类的实例信息, 栈区存放指向这个实例的引用地址。

- 方法区(Method Area) 与 Java堆一样, 是各个线程共享的内存区域;

- 方法区在JVM启动时就会被创建, 并且它的实际的物理内存空间和Java堆区一样都可以是不连续的;

- 方法区的大小和堆空间一样, 可以选择固定大小或者可扩展;

- 方法区的大小决定了系统可以保存多少个类, 如果系统定义了太多的类, 导致方法区溢出, JVM也会抛出OOM异常;

    - 加载大量的第三方Jar包;

    - Tomcat部署的工程过多;

    - 大量动态生成反射类。

- 关闭JVM就会释放这个区域的内存

很多时候看起来很简单的代码, 但是其中加载了很多的类, 例如下面的代码可以通过jvisualvm查看加载类的个数:

```java
public class MethodAreaDemo {
    public static void main(String[] args) {
        System.out.println("start...");
        try {
            TimeUnit.SECONDS.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println("end...");
    }
}
```

![jvm_methodarea3](/image/jvm_methodarea3.png)

> 虽然只是简单的demo, 但是确加载了1800多个类信息, 例如, System类, Thread类, Object类, 包括printlin输出流相关的类等等。

### 2. 设置方法区大小的参数

方法区的大小不必是固定的, JVM可以根据应用的需求动态调整。

**JDK7及以前(永久代):**

- `-XX:PermSize`来设置永久代初始分配空间。默认值是20.75M;

- `-XX:MaxPermSize`来设置永久代最大可分配空间。32位机器默认是64M, 64位机器模式是82M;

- 当JVM记载的类信息容量超过了这个值, 会报异常OOM: `OutOfMemoryError:PermGen space`。

查看永久代的大小信息命令:

```java
jinfo -flag PermSize [进程id]

-XX:PermSize=100m -XX:MaxPermSize=100m
```


**JDK8及以后(元空间):**

- 元数据区大小可以使用参数`-XX:MetaspaceSize`和`-XX:MaxMetaspaceSize`指定, 代替上面的原有的两个参数;

- 其默认值依赖于平台。windows下, `-XX:MetaspaceSize`是21M, `-XX:MaxspaceSize`的值是-1, 表示没有限制;

- **这里与永久代不同的是, 如果不指定元空间的大小, 默认情况下, 虚拟机会耗尽所有可用的系统内存。如果元数据区发生溢出, 还是会抛出OOM:`OutOfMemeryError:Metaspace`。**

> 这里要注意的是, 使用`-XX:MetaspaceSize`设置初始元空间大小, 这个初始值就是触发`Full GC`的底线。一旦达到这个值, Full GC就会触发并卸载没用的类(即这些类对应的类加载器不再存活), 然后这个初始值将会被重置。**新的初始值取决于Full GC之后释放了多少元空间。**如果释放的空间不足, 那么在不超过MaxMetaspaceSize时, 适当提高这个值; 如果释放空间过多, 则适当降低该值。

如果初始化的值设置过低, 可能会导致Full GC多次触发。为了避免频繁的GC, 建议将`-XX:MetaspaceSize`设置为一个相对较高的值。

查看元空间的大小信息命令:

```java
jinfo -flag MetaspaceSize  [进程id]

-XX:MetaspaceSize=100m -XX:MaxMetaspaceSize=100m
```

### 3. 方法区的内部结构

![jvm_methodarea4](/image/jvm_methodarea4.png)

从上图可以很清晰的看到方法区锁存储的信息包括: **<font color='red'>类信息, 常量, 静态变量, 即时编译后的代码缓存</font>**

#### 3.1 类型信息

#### 3.2 域信息

#### 3.3 方法信息

#### 3.4 non-final的类变量(非final的static静态变量)

#### 3.5 全局常量(static final)




### 3. 解决OOM: (内存泄露, 内存溢出)

我们 先看下面的一段代码:

```java
public class OOMDemo extends ClassLoader {
    public static void main(String[] args) {
        int j = 0;

        try {
            OOMDemo oomDemo = new OOMDemo();
            for (int i = 0; i < 10000; i++) {
                // 创建ClassWriter对象, 用于生成类的二进制字节码
                ClassWriter classWriter = new ClassWriter(0);
                // 指明版本号, 修饰符, 类名, 包名, 父类, 接口
                classWriter.visit(Opcodes.V1_8, Opcodes.ACC_PUBLIC, "Class" + i, null, "java/lang/Object", null);
                // 返回byte[]
                byte[] code = classWriter.toByteArray();
                // 类加载 Class对象
                oomDemo.defineClass("Class" + i, code, 0, code.length);
                j++;
            }
        } finally {
            System.out.println(j);
        }
    }
}
```

在启动前设置元空间大小, JDK8: `-XX:MetaspaceSize=10m -XX:MaxMetaspaceSize=10m`; JDK6/7: `-XX:PermSize=10m -XX:MaxPermSize=10m`

最后打印的结果为:

```txt
8531
Exception in thread "main" java.lang.OutOfMemoryError: Metaspace
	at java.lang.ClassLoader.defineClass1(Native Method)
	at java.lang.ClassLoader.defineClass(ClassLoader.java:760)
	at java.lang.ClassLoader.defineClass(ClassLoader.java:642)
	at com.java.jvm.methodarea.OOMDemo.main(OOMDemo.java:24)

Process finished with exit code 1
```

这里可以看到, 当我们设置元空间大小为10m时, 不断创建8000多个类, 最终导致OOM, 内存溢出。

那么如何解决这个错误?

1. 首先要解决OOM异常或者Heap Space异常, 最主要的是确认到底是出现了 **`内存泄露(Memory Leak)`** 还是 **`内存溢出(Memory Overflow)`**。

    > 一般是首先通过内存映像分析工具对dump的堆转储快照进行分析。

2. 如果是内存泄露, 可以通过工具查看泄露对象到GC Roots的引用链。就可以找到泄露对象是通过怎样的路径与GC Roots相关联并导致垃圾回收器无法自动回收它们。

    > 所谓内存泄露, 就是存在栈中的引用指向堆中的对象实例, 但是这个实例并没有用途是被闲置的, 导致GC无法回收。也就是说**堆当中的闲置对象由于引用链的引用关系无法被回收，虽然它已经属于闲置的资源**

3. 如果不存在内存泄露, 那基本上就是内存溢出的问题了。 

    - 应当检查虚拟机的堆参数(-Xms与-Xmx), 与机器物理内存对比是否可以调大。

    - 从代码上检查是否存在某些对象生命周期过长, 持有状态时间过长的情况, 尝试检查程序运行期的内存消耗。

