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
- **CPU高速缓存一致性协议**: 多个高速缓存中的数据副本始终保持一致。它的核心思想是: 当CPU写入数据时, 如果大仙该变量被共享(也就是说, 在其他CPU中也存在该变量的副本),就会发出一个信号, 通知其他CPU该变量的缓存无效。

> 但是`READER`线程为什么一致不会执行呢?

这就是因为Java内存的优化导致的。**Java内存在判断一个程序在线程中没有任何写操作时, JVM就会任务这段程序不会去主内存拿数据, 而去缓存中拿数据, 这就导致了之前的问题, `READER`线程在缓存中的数据没有任何变化, 所以从缓存中拿出的INIT_VALUE一直不会变化。**

### 2. 并发编程中的三个重要性质

在并发编程中, 我们所遇到的问题基本上都是从**原子性**,**可见性**,**有序性**延伸而来, 所以深入理解这三个性质非常重要。

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

> 可见性是指多个线程访问同一个变量时, 一个线程修改了这个变量的值, 其他线程能够立即看得到修改的值，

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
对于上面的代码, 从代码的顺序上看, 语句1是在语句2的前面, 但是JVM在真正执行这段代码的时候并不一定会保证语句1在语句2之前执行。因为这里有可能发生**指令重排序(Instruction Reorder)**。

> `指令重排序`: **处理器为了提高程序运行效率, 可能回输入的代码进行优化, 它不保证程序中各个语句的执行先后顺序与代码中的顺序一致, 但是它会保证程序最终执行结果和代码顺序执行结果一致即可。**





