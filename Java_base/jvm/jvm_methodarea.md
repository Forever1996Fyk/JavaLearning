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

从上图可以很清晰的看到方法区存储的信息包括: **<font color='red'>类信息, 常量, 静态变量, 即时编译后的代码缓存</font>**

> 这里要注意的是, 方法区中的存储信息也是根据jdk版本的变化而变化的, 上面的只是方法区的经典结构, 但是**在JDK7之后, 静态变量和常量池(StringTable)都存储在堆区。**

#### 3.1 类型信息

对每个加载的类型(包括**类class, 接口interface, 枚举enum, 注解annotation**), JVM必须在方法区中存储一下信息:

- 类型的完整有效名称(全名 = 包名.类名)

- 这个类型的直接父类的完整有效名(对于interface或者java.lang.Object, 都没有父类)

- 这个类型的修饰符(public, abstract, final的某个子集)

- 这个类型直接接口的一个有序列表(因为一个类可以实现多个接口, 所以接口是一个列表)

#### 3.2 域信息(成员变量)

- JVM必须在方法区中保存类型的所有域的相关信息以及域的声明顺序;

- 域的相关信息包括: 域名称, 域类型, 域修饰符(public, private, protected, static, final, volatile, transient的某个子集)

#### 3.3 方法信息

JVM必须保存所有方法的一下信息, 并且包括声明顺序:

- 方法名称;

- 方法的返回类型(包括void);

- 方法参数的数量和类型(按顺序);

- 方法的修饰符(public, private, protected, static, final, synchronized, native, avstract的一个子集);

- 方法的字节码(bytecodes), 操作数栈, 局部变量表以及大小(abstract和native方法除外);

- 异常表(abstract和native方法除外), 每个异常处理的开始位置, 结束位置, 代码处理在程序计数器中的偏移地址, 被捕获的异常类的常量池索引。

#### 3.4 non-final的类变量(非final的static静态变量)

- 静态变量是与类关联在一起, 随着类的加载而加载, 类变量被类的所有实例共享, 即使没有类实例也可以访问。

我们看下面代码:

```java
public class MethodAreaTest {
    public static void main(String[] args) {
        Order order = null;
        order.hello();
        System.out.println(order.count);
    }
}

class Order {
    public static int count = 1;
    public static final int number = 2;


    public static void hello() {
        System.out.println("hello!");
    }
}
```

这段代码时不会报错的, 而且可以正常输出。

#### 3.5 全局常量(static final)

被声明为final的静态变量(全局常量)在编译的时候就被分配了。

我们编译上面的Order类, 得到Order.class字节码文件。可以通过`javap -v -p Order.class > test.txt`命令, 把字节码文件输出到txt文件中, 如下:

```txt
public static int count;
    descriptor: I
    flags: ACC_PUBLIC, ACC_STATIC

public static final int number;
    descriptor: I
    flags: ACC_PUBLIC, ACC_STATIC, ACC_FINAL
    ConstantValue: int 2    
```

**声明为`static final`的常量在number被编译的时候就赋值了, 而没有被final修饰的静态变量count, 是在类加载准备阶段才被赋值(也就是clinit方法的调用[类加载子系统](jvm_subsystem.md))**

### 4. 运行时常量池

还记得我们在Java栈中所提到了运行时常量池吗? 没错它就存储在方法区中。但是要理解运行时常量池, 就必须要知道常量池。

![jvm_methodarea9](/image/jvm_methodarea9.png)

**常量池(`Constant Pook Table`)主要是存储各种字面量, 类, 变量以及方法的符号引用。**
    > 字面量的意思就是一段字符串, 例如`String s = "hello"`。其中"hello"就是字面量。

一个Java源文件中类, 接口, 编译后产生的字节码文件需要数据支持, 一般情况下这些数据很大不能直接存储到字节码中。
**常量池, 中存储了各种符号引用。字节码文件用个指向常量池的的符号引用来找到对应信息的直接地址。**

这里要注意几点:

1. 运行时常量池是方法区的一部分, 而常量池在JDK7之后, 就被移到了堆区; JDK6之前还是存放在方法区中;

2. 常量池表示Class文件的一部分, 所以常量池是编译后直接存放在字节码文件中的。在类加载后会存放在运行时常量池中。

3. JVM为每个已加载的类或接口都维护一个常量池。通过索引找到对应的符号引用。

4. 常量池是在编译时就保存的符号引用表, 也就是每个符号仅仅表示的是引用; 而类加载之后, 运行时常量池保存的就不是符号引用了, 而是真正对应的真实地址。

5. 当创建类或接口的运行时常量池时, 如果构造所需的内存空间超过了方法区的最大值, 那么JVM就会抛出OOM异常。

### 5. 解决OOM: (内存泄露, 内存溢出)

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

#### 6. 方法区演进细节

首先要注意的是, 永久代是Hotspot才有的东西。其他虚拟机并不存在永久代的概念。

Hotspot中方法区的变化:

- jdk1.6之前: 有永久代(`permanent generation`), 静态变量存放在永久代上;

- jdk1.7: 有永久代, 但是已经开始"去永久代", 字符串常量池, 静态变量从永久代移除, 保存在堆中;

- jdk1.8之后: 无永久代, 类型信息, 字段, 方法, 常量保存在本地内存的元空间, **但是字符串常量池, 静态变量仍存储在堆空间中。**

    > JDK8中的元空间不再使用虚拟机内存, 而是直接使用本地内存。

![jvm_methodarea5](/image/jvm_methodarea5.png)

![jvm_methodarea6](/image/jvm_methodarea6.png)

![jvm_methodarea7](/image/jvm_methodarea7.png)

#### 6.1 永久代为什么要被元空间替换

- **首先元空间中的数据时分配在本地内存中, 所以元空间最大分配空间就是系统可用内存空间**;

- **永久代是属于虚拟机内存的, 而永久代设置空间大小很难确定。如果动态加载类过多, 容易产生Perm区(永久代)的OOM。而元空间的大小仅受本地内存限制, 这也是元空间与永久代最大的区别。**

- 永久代进行调优很困难。

#### 6.2 StringTable(常量池)为什么要调整到堆区

从JDK7开始, JVM将常量池存储到堆空间当中。

**因为永久代的回收效率很低, 只有在Full GC的时候才会涉及到永久代。而Full GC是老年代的空间不足, 永久代空间不足时才会触发的。这就导致了StringTable回收效率很低。而在开发中会有大量的字符串被创建, 回收效率低就会导致永久代OOM, 所以放到堆区中, 能及时回收内存。**

### 7. 静态变量, 成员变量, 局部变量存放位置分析

- 静态变量: 根据我们上面的分析, JDK7之后静态变量存在Java堆中。

- 成员变量: 也就是实例变量的生命周期是跟随对象的, 是属于对象的内容。而对象实例化之后存放在堆中, 所以成员变量也是存在堆中;

- 局部变量: 根据在虚拟机栈中学习的知识, 局部变量是存在虚拟机栈的栈帧中的局部变量表(注意我们这里说的是变量)。

> 对象实例本身存放的位置, 我们在逃逸分析中也说的很清楚, 也是一定在Java堆中分配。而具体类的信息则是存储在方法区中。

### 8. 方法区垃圾回收

一般来说这个区域的回收效果难度较大。尤其是类型的卸载, 条件很苛刻, 但是这部分区域的回收有时有确实有必要。所以方法区的垃圾回收主要回收两部分内容: **常量池的废弃常量和不再使用的类型。**

### 9. 总结

我们用一张图, 再次具体的看下整个运行时数据区的执行流程:

![jvm_methodarea8](/image/jvm_methodarea8.png)