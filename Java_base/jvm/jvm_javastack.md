## JVM运行时数据区--Java虚拟机栈

由于跨平台性的设计, java的指令都是根据栈来设计的。不同平台CPU架构不同, 所以不能基于寄存器设计。

> 根据栈设计的**优点是跨平台, 指令集小, 编译器容易实现; 缺点就是性能下降, 实现同样的功能需要更多的指令。**

在内存中堆与栈的作用:

- **栈是运行时单位, 堆是存储单位;**

- **一般来说, 对象主要是放在堆内存中, 是运行时数据区比较大的区域;**

- **栈空间存放的局部变量: 包括 8种基本数据类型, 引用数据类型(对象的引用地址, 包括数组, 类, 接口)。**


### 1. 虚拟机栈的本质

每一个线程在创建时都会创建一个虚拟机栈, 而每个线程所执行的方法都会以**栈帧(Stack Frame)**的方式存储。

> 虚拟机栈内部保存一个个 **栈帧(Stack Frame)** 。栈帧是一一对应java方法调用, 它是线程私有的。**所以虚拟机栈的生命周期和线程时一致的。**

**虚拟机栈的作用: 主要管理Java程序的运行, 它保存方法的局部变量, 并参与方法的调用和返回。**

栈帧(Stack Frame)的结构如下:

- **局部变量表(Local Variables)**: 存储局部变量, 包括 **8种基本数据类型**, **引用类型变量(类, 数组, 接口的引用地址)**

- **操作数栈(Operand Stack)**: 也是一个栈结构, 主要是临时的保存局部变量

- **动态链接(Dynamic Linking)**: 执行"运行时常量池"的方法引用, 这也是Java多态特性的原理。

- 方法返回地址(Return Address): 方法正常退出或者异常退出的定义

- 附加信息


JVM直接对虚拟机栈的操作只有两个:

    1. 每个方法执行, 栈帧入栈;
    2. 方法执行结束, 出栈。

所以对于栈空间来说, 不会存在垃圾回收问题, 但是会存在`StackOverFlowError`, `OOM`错误。

### 2. 虚拟机栈的存储结构和运行原理

- 栈帧本身是一个内存区块, 存储着方法执行过程中的各种数据信息。JVM对虚拟机栈的操作时遵循先进后出/后进先出的原则。 每一次方法的调用和结束, 就是一次栈帧的入栈和出栈操作。

> 要注意的是, 在一条活动线程上, 同一个时间点只会有一个活动的栈帧。即只有在栈顶的方法是有效的, 这个栈帧被称为**当前栈帧(Current Frame)**, 与当前栈帧对应的方法就是当前方法。

- 如果存在调用的方法中, 又调用了其他方法, 那么对应的新的栈帧就会创建出来并且入栈, 放在栈的顶端, 成为新的当前栈帧。

- 如果当前方法调用的其他方法存在返回值, 那么当前栈帧会传回此方法的执行结果给前一个栈帧, 然后JVM会丢弃当前栈帧, 是前一个栈帧成为当前栈帧。

- Java方法有两种返回函数的方式: **一种是正常函数返回, 使用return指令; 另外一种是抛出异常。但是不管是哪种方式, 都会导致栈帧出栈。**

- **不同线程中所包含的栈帧是不允许相互引用的, 也就是说不可能在栈帧中引用另外一个线程的栈帧。这也是为什么局部变量是线程安全的原理。**

![jvm_javastack1](/image/jvm_javastack1.png)

### 3. 虚拟机栈可能出现的异常

**Java虚拟机规范允许虚拟机栈的大小是动态的或者是固定不变的。**

1. 如果采用固定大小的Java虚拟机栈, 那每一个线程的虚拟机栈容量可以在线程创建的时候独立选定。如果线程请求分配的栈容量超过了Java虚拟机栈允许的最大容量, Java虚拟机栈会抛出一个 **`StackOverFlowError`异常**。例如下面代码:

    ```java
    /**
    * 演示栈中的异常
    */
    public class StackErrorTest {
        public static void main(String[] args) {
            main(args);
        }
    }
    ```

2. 如果对虚拟机栈进行动态扩展, 并且在尝试扩展时无法申请到足够的内存, 或者创建新的线程时没有足够的内存去创建对应的虚拟机栈, 那么Java虚拟机会抛出 **`OutOfMemoryError`异常**。

> 我们可以使用参数 **`-Xss`命令来设置线程的最大虚拟机栈空间**。栈的大小直接决定了函数调用的最大可达深入。
>```java
    -Xss256k
    -Xss1024M
    -Xss1G
>```


### 4. 虚拟机栈的相关面试题

1. 举例栈溢出的情况? (`StackOverFlowError`)

**方法的递归调用; 通过`-Xss`设置栈的大小不合适。**

2. 调整栈的大小, 就能保住不出现溢出吗?

答案是不能的。因为整个JVM的内存空间是有限的, 所以不管怎么调整栈的大小, 只能保证可执行的方法更多, 但是如果遇到无限递归操作, 最终还是会导致内存溢出(OOM)的异常。

3. 分配栈内存越大越好吗?

不是的。因为虚拟机栈是跟线程对应的, 整个JVM的内存是有限的, 所以栈分配的内存越大, 必然导致可创建的线程越少。因为会导致内存不足, 无法创建新的线程。

4. 垃圾回收是否会涉及到虚拟机栈?

垃圾回收, 并不会影响虚拟机栈。我个人理解是, 垃圾回收其实也是一个单独的线程, 那么既然是线程就会存在一个虚拟机栈, 而虚拟机栈是线程私有的, 相互是无法调用的, 所以垃圾回收不会影响虚拟机栈。

5. 方法中定义的局部变量是否线程安全?

> 线程安全问题的本质, 多个线程对共享数据进行修改, 可能会存在线程安全问题。

一般情况下, 因为局部变量是线程私有的, 所以不存在线程安全的问题。但是可能在一个方法中创建多个线程对同一个局部变量进行修改, 或者局部变量通过形参的方式传入方法。 也算是线程安全问题。

```java
public void m1() {
    int i = 0;

    new Thread(() -> {
        i++;
    }).start;

    new Thread(() -> {
        i++;
    }).start;

    System.out.println(i);

}

```

### 5. 扩展Java中的Error和Exception

`Exception`和`Error`都是继承于`Throwable`类, 在Java中只有`Throwable`类型的实例才可以被 **抛出(`throw`)** 或者 **捕获(`try catch`)** , 它是异常处理机制的基本类型;

- `Exception`是java程序运行中可预料的异常情况, 我们可以获取到这种异常, 并且对这种异常进行业务外的处理; 而`Exception`又分为检查性异常和非检查型异常。

    - `Exception`检查性异常: 必须在编写代码时使用`try catch`捕获(比如: IOException);

    - `Exception`非检查性异常: 在编写代码时可以忽略捕获异常, 这种异常是可以再代码编写或者使用过程中通过规范避免的。


- `Error`是Java程序中不可预料的异常, 这种异常的发生会直接导致JVM不可处理或者不可恢复的情况。所以这种异常不可能获取, 比如`OutOfMemoryError`, `NoClassDefFoundError`等。

