## volatile关键字

在学习`volatile`之前, 先看一段代码:

```java
public class VolatileTest {
    private static int INIT_VALUE = 0;

    private final static int MAX_LIMIT = 5;

    public static void main(String[] args) {
        new Thread(() -> {
            int localValue = INIT_VALUE;
            while (localValue < MAX_LIMIT) {
                if (localValue != INIT_VALUE) {
                    System.out.printf("The value update to [%d]\n", INIT_VALUE);
                    localValue = INIT_VALUE;
                }
            }

        }, "READER").start();

        new Thread(() -> {
            int localValue = INIT_VALUE;
            while (INIT_VALUE < MAX_LIMIT) {
                System.out.printf("Update the value to [%d]\n", ++localValue);
                INIT_VALUE = localValue;
                try {
                    Thread.sleep(500);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }, "UPDATE").start();
    }
}

```

上面的代码中, 运行会发现, 只有`UPDATE`的线程运行了, 但是`READER`线程却没有反应。

> 产生这种现象的原因是, `UPDATE`线程修改`INIT_VALUE`之后, `READER`线程并没有感知到`INIT_VALUE`的变化, 所以对于`READER`线程来说localVlaue仍然为0。

这部分内容涉及到Java内存模型, 以及CPU相关问题。

### 1. 内存模型的相关概念

对于CPU内存而言, 分为主内存和高速缓存。当程序第一次拿数据时, 会先将主内存中的数据放入高速缓存中, 然后再从高速缓存中取出。在多线程的环境下, 对数据修改时, 会先在缓存中修改数据后, 再刷新到主内存中去。

> 例如: i = 1; i= i + 1<br/>
> 在CPU中操作流程是: 在主内存中读取i的数据, 然后把i放入缓存中, CPU执行i+1的指令, 然后把结果更新到缓存, 然后缓存刷新到主内存。<br/>
> 这个操作在单线程中是没有问题的。但是多线程环境就会出现问题。当第二个线程执行上面的操作时, 线程从缓存中读出的i的值仍然为1, 这就导致了两个线程对i+1的操作结果是一致的, 所以最后两个线程得出的结果都是2, 最终i的值仍然是2.

解决上面的问题有两种方式解决:

- **总线锁机制**: 给数据总线加锁[总线(数据总线, 地址总线, 控制总线)], 在CPU中就是`LOCK#`指令。一旦对数据总线加锁, 那么每次操作都只能有一个线程, 但是这就导致了多核CPU串行化了, 效率大大降低。
- **CPU高速缓存一致性协议**: 多个高速缓存中的数据副本始终保持一致。它的核心思想是: 当CPU写入数据时, 如果该变量被共享(也就是说, 在其他CPU中也存在该变量的副本),就会发出一个信号, 通知其他CPU该变量的缓存无效。

而对于Java内存模型而言, 与CPU内存模式是很相似的。每一个线程都有自己的工作内存, 当线程获取共享变量时, 如果在自己的工作内存中获取不到, 那就从主内存中获取, 并且把这个共享变量的值缓存到自己的工作内存中, 否则就直接在工作内存中获取这个值。当线程修改这个共享变量时, 也会先把工作内存中的值修改, 然后在刷新到主内存中。


### 2. 并发编程中的三个重要性质

在并发编程中, 我们所遇到的问题基本上都是从**原子性**,**可见性**,**有序性**延伸而来, 只要有一个没有被保证, 那么就有可能导致程序运行不正确, 所以深入理解这三个性质非常重要。

##### 2.1 原子性

> 一个操作或者多个操作要么全部执行并且执行的过程不会被任何因素打断, 要么都不执行

**对于基本数据类型的变量读取和赋值是保证原子性的。**例如: i = 10;

```java
a = 10; // 原子性
b = a;  // 不满足 1. read a 2. assign(赋值) b
c++;    // 不满足 1. read c 2. add c 3. assign c
c=c+1;  // 不满足 1. read c 2. add c 3. assign c

Object obj = obj2; // 也是原子性的。这种引用类型的赋值, 实际上赋值的是对象的地址。
```

##### 2.2 可见性

> 可见性是指多个线程访问同一个变量时, 一个线程修改了这个变量的值, 其他线程能够立即看得到修改的值。

Java内存模型是可见性问题的主要原因, 因为每个线程都有自己的工作内存, 就会导致每个线程操作的共享变量可能都是自己缓存的值, 结果就是线程对共享变量的修改, 其他线程无法感知到。所以这也正是一开头那段代码错误的原因。

```java
private static int INIT_VALUE = 0;
```
没有保证`INIT_VALUE`的可见性。所以`UPDATE`修改时, `READER`线程并没有获取到已经修改的值, 导致没有输出。

当然在Java中, 保证可见性的方式有几种。可以通过`synchronized`和`Lock`也能够保证可见性。因为它们能够保证同一时刻只有一个线程获取锁然后执行代码, 并且在释放锁之前会将变量的修改刷新到主存中。

##### 2.3 有序性

这是并发编程中比较复杂的一个性质。简单来说:

> 程序的执行顺序按照代码的先后顺序执行!

例如下面这段代码
```java
int i = 0;
boolean flag = false;
i = 1;                  // 语句1
flag = true;            // 语句2
```
对于上面的代码, 从代码的顺序上看, 语句1是在语句2的前面, 但是JVM在真正执行这段代码的时候并不一定会保证语句1在语句2之前执行。因为这里有可能发生 **指令重排序(Instruction Reorder)** 。

> `指令重排序`: **处理器为了提高程序运行效率, 可能回输入的代码进行优化, 它不保证程序中各个语句的执行先后顺序与代码中的顺序一致, 但是它会保证程序最终执行结果和代码顺序执行结果一致即可。**

所以对于上面的代码, 语句1, 语句2谁先执行对最终的结果并没有影响, 所以就有可能发生指令重排序。

但是对于指令重排序, 要注意一个重要的原则: **happens-before原则**。例如下面的代码:

```java
int a = 10; // 语句1
int r = 2;  // 语句2
a = a + 3;  // 语句3
r = a * a;  // 语句4
```
对于这段代码, 不可能存在语句4执行在语句3之前执行的情况。因为如果存在数据之间有依赖关系的话, 那么就不会发生重排序。很明显,语句4中r的值是依赖于最终a的值, 所以不可能重排序。

所以看到这, 应该很容易理解, 为什么之前的double check 单例模式, 需要加上`volatile`关键字。

> 还有一个需要注意的问题: 指令重排序只会在多线程环境下并发执行会影响正确性, 但是在单线程环境中不存在问题。

### 3. 深入理解volatile关键字

对于`volatile`的作用:
1. 保证变量的可见性
2. 保证变量的有序性, 也就是禁止指令重排序

**而`volatile`并不保证原子性。所以使用`volaile`关键字依然会存在线程安全问题。**

所以对于文章开头的代码, `READER`线程为什么一直不会执行呢?

之前我们分析过每个线程都有自己的工作内存, 每次取数据都先将数据从主内存存入高速缓存中, 然后直接从缓存中拿数据,而Java内存有自己的优化机制。**Java内存在判断一个程序在线程中没有任何写操作时, JVM就会任务这段程序不会去主内存拿数据, 而去缓存中拿数据, 这就导致了之前的问题, `READER`线程在缓存中的数据没有任何变化, 所以从缓存中拿出的INIT_VALUE一直不会变化。**

但是如果用`volatile`修饰INIT_VALUE就不一样了:

1. 使用`volatile`关键字会强制将修改的值立即写入主存
2. 使用`volatile`关键字修饰变量时, 当`UPDATE`线程进行修改时, 会导致`READER`线程的缓存中的变量无效, 必须要从主内存中取数据。

```java
public class VolatileTest {
    private static volatile int INIT_VALUE = 0;

    private final static int MAX_LIMIT = 5;

    public static void main(String[] args) {
        new Thread(() -> {
            int localValue = INIT_VALUE;
            while (localValue < MAX_LIMIT) {
                if (localValue != INIT_VALUE) {
                    System.out.printf("The value update to [%d]\n", INIT_VALUE);
                    localValue = INIT_VALUE;
                }
            }

        }, "READER").start();

        new Thread(() -> {
            int localValue = INIT_VALUE;
            while (INIT_VALUE < MAX_LIMIT) {
                System.out.printf("Update the value to [%d]\n", ++localValue);
                INIT_VALUE = localValue;
                try {
                    Thread.sleep(500);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }, "UPDATE").start();
    }
}
```

##### 3.1 volatile不保证原子性

```java
private static volatile int i = 0;

// 多个线程进行 i++ 操作
```
根据上面的分析, i++ 并不是原子性的操作, 如果进行多个线程修改i的值, 并且输出。你会发现还是有大量重复的值。

> 如果某一时刻线程A和B都拿到了主内存中的i=0并且都完成了i++操作, 那么线程A和B的工作内存中i的值都是为1, 如果线程B获取到了CPU的执行权, 将自己内存中的i=1刷新到主内存中, 此时会发一个信号(**总线嗅探机制**)通知线程A, 让线程A中的缓存变量i=1失效了, 需要重新从主内存读取, 确实此时线程A会重新读取主内存中的i=1, 但是之后的指令是将i=1再写回主内存, 而不是进行i++操作, 因为i++操作已经执行过了, 只不过结果失效了。

**更深入的来讲, 在这种情况下, 这个失效机制(总线嗅探)会消耗掉线程A对i变量的`use操作`, 导致线程A会i修改的失效**

**变量i从主内存到某个线程的工作内存再回传的具体过程:**

1. **read: 读取主内存中值(即线程A读到主内存中i=0)**
2. **load: 将主内存中的i=0加载到自己的工作内存, 更深入的说是复制到栈中局部变量表里**
3. **use: 在工作内存中对i进行++操作, 得到结果i=1, 此时这个1被压入操作数栈中**
4. **assing: 将操作数栈中的1复制给局部变量表**
5. **store: 将i变量传回主内存(这个过程需要通过总线, 就会触发总线嗅探)**
6. **write: 将被传回的i=1写到主内存的变量i上, 即刷新主内存**

##### 3.2 volatile能够保证有序性

这里举个例子说明:

```java
int x = 2;  // 语句1
int y = 0;  // 语句2
volatile boolean flag = true; // 语句3
x = 4; // 语句4
y = -1; // 语句5
```

由于flag被`volatile`修饰, 那么在进行指令重排的过程中, 不会讲语句3放到语句1, 语句2的前面, 也不会放到语句4, 语句5的后面, 但是语句1和语句2的顺序, 语句4和语句5的顺序不能保证有序性。

> 所以, 对于之前学习的`double check 单例模式`, 如果对实例加上`volatile`, 就一定会保证返回实例前, 实例已经初始化完全。

##### 3.3 volatile的原理和实现机制

> 根据《深入理解Java虚拟机》中的描述: **"加入volatile关键字和没有加入关键字时所生成的汇编代码发现, 加入volatile关键字时, 会多出一个lock前缀的指令。"**

lock前缀的指令实际上相当于一个内存屏障, 内存屏障会提供3个功能:

1. 它确保指令重排序是不会包后面的指令排到内存屏障之前的位置, 也不会把前面的指令排到内存屏障的后面, 也就是说执行到内存屏障这个指令时, 它前面的操作已经全部完成了.
2. 它会强制对缓存的修改操作立即写入主存
3. 如果是写操作, 它会导致其他CPU中的缓存变量失效。

### 4. 总结

通过之前的`double check`以及本篇一开头的代码就可以知道, `volatile`关键在一般在什么情况下使用了。

1. 状态标记量
2. double check














