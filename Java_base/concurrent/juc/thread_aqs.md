## J.U.C: AQS

Java的内置锁一直都是饱受争议的, 在JDK1.6之前, `synchronized`的性能一直都比较低下, 虽然jdk不断的进行锁优化策略, 但是与`Lock`相比`Synchronized`还是存在一些缺陷。
> 虽然`synchronized`提供了隐式获取锁释放机制(基于JVM机制), 但是他缺少获取锁的释放锁的可操作性, 可中断, 超时获取锁, 而且它是一种互斥锁, 在高并发场景下性能大打折扣。

所以在后面学习`Lock`相关工具类之前, 我们必须要先熟悉了解非常重要的组件, 只要掌握该组件JUC中大部分问题也就比较清楚了。这个组件就是**AQS**。

**AQS, 是`AbstractQueuedSynchronizer`的缩写, 即队列同步器。它是构建锁或者其他同步组件的基础框架(`ReentrantLock`, `ReentrantReadWriteLock`, `Semaphore`等)**

### AQS简介

`AQS`定义两种资源共享的方式: 
- **Exclusive (独占):** 只有一个线程能执行, 例如`ReentrantLock`;
- **Share (共享):** 多个线程可同时执行, 例如`Semaphore/CountDownLatch`

AQS的设计模式采用的模板方法模式, 子类通过继承的方式, 实现自己的方法来管理同步状态。不同的自定义同步器在争用共享资源的方式也不同。

**但是AQS已经在底层基本上已经实现了具体线程等待队列的维护, 例如获取资源失败入队, 唤醒线程出队**。所以对于子类来说主要实现的方法如下:

- `boolean isHeldExclusively()`: 判断该线程是否正在独占资源。(只有用到`condition`才需要去实现它)。
- `boolean tryAcquire(int)`: 独占方式。尝试获取资源, 成功则返回true, 失败则返回false。
- `boolean tryRelease(int)`: 独占方式。尝试释放资源, 成功则返回true, 失败则返回false。
- `int tryAcquireShared(int)`: 共享方式。尝试获取资源。负数表示失败; 0表示成功, 但是没有剩余可用资源; 正数表示成功, 而且有剩余资源。
- `boolean tryReleaseShared(int)`: 共享方式。尝试释放资源, 如果释放后允许唤醒等待节点则返回true, 否则返回false