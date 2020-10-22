对于一个Java程序员, **能否熟练掌握并发编程式判断他优秀与否的重要标准之一**。因为并发编程是Java语言中最为晦涩的知识点, 它涉及操作系统, 内存, CPU, 编程语言等多方面的基础能力, 更考验一个程序员的内功。

虽然在公司做的项目可能并不会涉及高并发, 大数据, 但是学习并发编程一定是对自己编程生涯非常好的提升, 同时也是进大厂的高频面试点。


**学习并发阶段, 我个人分为四个阶段:**

1. 多线程基础

    > - 多线程的基础包括, 线程的常用API以及原理, 锁的原理, 线程安全的原理以及如何解决。
    > - 多线程的使用场景

2. 多线程编程的提升

    > - 与多线程相关的设计模式, 例如: 多种单例模式, 生产者消费者模式, Active异步 
    > - 使用这些模式产生的问题如何解决
    > - volatile关键字源码解析(深入到CPU内存, 汇编级别)
    > - ThreadLocal源码解析
    > - 自己实现一个读写锁

3. JUC 常用API的使用以及原理

    JUC是java.util.concurrent的简称, 是java处理线程的工具包, 从JDK1.5开始出现, 是极其重要的工具包

    > - Atomic相关原子类的API使用以及原理, 例如: `AtomicInteger`, `AtomicLong`, `AtomicReference`等
    > - 深入理解CAS的原理, 以及ABA问题如何解决
    > - 常用工具类, 例如: `CountDownLatch`, `CyclicBarrier`, `Exchanger`, `Lock`, `Semaphore`, `ExecutorSerivce`
    > - unsafe的原理, 源码解析。

4. AQS原理以及各种安全框架集合原理

    > - JUC几乎所有的工具类都是在AQS的基础上进行开发, 所以理解AQS的原理十分重要。 AQS全称`AbstractQueuedSynchronizer`。
    > - 安全框架集合的原理以及数据结构, `ConcurrentHashMap`