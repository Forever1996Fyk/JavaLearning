## J.U.C: AQS

Java的内置锁一直都是饱受争议的, 在JDK1.6之前, `synchronized`的性能一直都比较低下, 虽然jdk不断的进行锁优化策略, 但是与`Lock`相比`Synchronized`还是存在一些缺陷。
> 虽然`synchronized`提供了隐式获取锁释放机制(基于JVM机制), 但是他缺少获取锁的释放锁的可操作性, 可中断, 超时获取锁, 而且它是一种互斥锁, 在高并发场景下性能大打折扣。

所以在后面学习`Lock`相关工具类之前, 我们必须要先熟悉了解非常重要的组件, 只要掌握该组件JUC中大部分问题也就比较清楚了。这个组件就是**AQS**。

**AQS, 是`AbstractQueuedSynchronizer`的缩写, 即队列同步器。它是构建锁或者其他同步组件的基础框架(`ReentrantLock`, `ReentrantReadWriteLock`, `Semaphore`等)**

### AQS简介

**`AQS`通过内置的FIFO(First In First Out)同步队列来完成资源获取线程的排队功能, 如果当前线程获取同步状态失败(锁/资源)时,  `AQS`则会将当前线程以及等待状态等信息构造成一个节点(Node)并将其加入同步队列, 同时阻塞当前线程, 当同步状态释放时, 则会把节点中的线程唤醒, 再次尝试获取同步状态。这个同步队列称为`CLH队CLH列`**

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

对于独占方式, 我们以`ReentrantLock`为例, state初始化为0, 表示未锁定状态。线程A调用lock()时, 会调用`tryAcquire()`独占该锁并将`state + 1`。所谓独占锁, 就是当其他线程再调用`tryAcquire()`时会通过CAS判断state是否为0, 如果不为0那么就会返回false, 直到线程A调用`unlock()`时`state=0`, 其他线程才有机会获取资源。 而且在释放锁之前, 线程A可以重复获取此锁(state会累加), 这就是可重入的概念。**但是要注意的是, 获取多少次就要释放多少次, 这样才能保证state最终等于0。**

对于共享方式, 我们以`CountDownLatch`为例, 任务分为N个子线程去执行, state也初始化为N(**其中N要与线程个数一致**)。这N个子线程是并行执行的, 每个子线程执行完后执行`countDown()`方法, state通过CAS减1。等到所有子线程都执行后(state=0), 那就会`unpark()`主调用线程, 那么主线程就会继续后面的任务。

> 一般来说, 自定义同步器要么是独占方法, 要么是共享方法, 他们也只需要实现`tryAcquire-tryRelease`, `tryAcquireShared-tryAcquireReleaseShared`中的一种既可以。但是AQS也支持自定义同步器同时实现独占和共享两种方式, 如`ReentrantReadWriteLock`。

### AQS: CLH同步队列

`CLH同步队列`是一个FIFO双向队列, AQS依赖它来完成同步状态的管理。在CLH同步队列中, 一个节点表示一个线程, 它保存着线程的引用(thread), 状态(waitStatus), 前驱节点(prev), 后继节点(next), 其定义如下:

```java
static final class Node {
    /** 共享 */
    static final Node SHARED = new Node();

    /** 独占 */
    static final Node EXCLUSIVE = null;

    /**
     * 因为超时或者中断，节点会被设置为取消状态，被取消的节点时不会参与到竞争中的，他会一直保持取消状态不会转变为其他状态；
     */
    static final int CANCELLED =  1;

    /**
     * 后继节点的线程处于等待状态，而当前节点的线程如果释放了同步状态或者被取消，将会通知后继节点，使后继节点的线程得以运行
     */
    static final int SIGNAL    = -1;

    /**
     * 节点在等待队列中，节点线程等待在Condition上，当其他线程对Condition调用了signal()后，改节点将会从等待队列中转移到同步队列中，加入到同步状态的获取中
     */
    static final int CONDITION = -2;

    /**
     * 表示下一次共享式同步状态获取将会无条件地传播下去
     */
    static final int PROPAGATE = -3;

    /** 等待状态 */
    volatile int waitStatus;

    /** 前驱节点 */
    volatile Node prev;

    /** 后继节点 */
    volatile Node next;

    /** 获取同步状态的线程 */
    volatile Thread thread;

    Node nextWaiter;

    final boolean isShared() {
        return nextWaiter == SHARED;
    }

    final Node predecessor() throws NullPointerException {
        Node p = prev;
        if (p == null)
            throw new NullPointerException();
        else
            return p;
    }

    Node() {
    }

    Node(Thread thread, Node mode) {
        this.nextWaiter = mode;
        this.thread = thread;
    }

    Node(Thread thread, int waitStatus) {
        this.waitStatus = waitStatus;
        this.thread = thread;
    }
}
```

##### 1.1 入列

CLH队列入列是比较简单的, tail指向新节点, 新节点的prev指向当前最后的节点(tail节点), 当前最后一个节点的next指向当前节点。具体代码是`addWaiter(Node node)`:

```java
private Node addWaiter(Node mode) {
    //新建Node
    Node node = new Node(Thread.currentThread(), mode);
    //快速尝试添加尾节点
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        //CAS设置尾节点
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    //多次尝试
    enq(node);
    return node;
}
```

##### 1.2 出列

由于CLH队列是FIFO, 所以首节点的线程释放同步状态后, 将会唤醒它的后继节点(也就是firstNode的next Node), 而后继节点在获取同步状态成功时将自己设置为首节点。

> head节点断开员首节点的next, 并连接 原首节点的 next节点 的 prev节点 即可。而且这个过程不需要使用CAS来保证, 因为只有一个线程能够成功获取到同步状态。

### 同步状态的获取与释放

##### 2.1 独占式

> 独占式, 同一时刻仅有一个线程持有同步状态。

`acquire(int arg)`方法是AQS提供的模板方法, 该方法是独占式获取同步状态, 但是这个方法对线程中断无效, 也就是说由于线程获取同步状态失败加入到CLH队列中, 即使对线程进行中断操作, 线程也不会从队列中移除。

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

- `tryAcquire`: 尝试获取锁, 获取资源成功则设置锁状态并返回true, 否则返回false。这个方法需要自定义同步组件自己实现,而且必须保证线程安全的获取同步状态。
- `addWaiter`: 如果`tryAcquire`返回false(获取同步状态失败), 则调用该方法将当前线程加入到CLH队列的尾部, 并标记独占模式
- `acquireQueued`: 当前线程通过自旋的方式来阻塞等待, 知道获取资源为止; 如果整个等待过程中被中断, 则返回true, 否则返回false
- `selfInterrupt`: 如果线程在等待过程中被中断过, 线程是不响应的。只是获取资源后才进行自我中断

> **`tryAcquire(int)`:**
>```java
> protected boolean tryAcquire(int arg) {
>    throw new UnsupportedOperationException();
> }
>```
> 我们发现这个方法直接抛出一个异常。因为AQS只是一个框架, 具体资源的获取/释放方式是由自定义同步器去实现(通过state的get/set/CAS)。**但是要注意的是: 自定义同步器在进行资源访问时要考虑线程安全的影响** <br/>
> 至于这个方法并没有定义成abstract, 是因为独占模式下只实现`tryAcquire-tryRelease`, 而共享模式下只实现`tryAcquireShared-tryReleaseShared`。如果都定义成abstract, 那么每个模式也要去实现另一模式的接口。减少不必要的工作量。


> **`acquireQueued(Node, int)`：**
> 如果tryAcquire()返回false, 那么调用`addWaiter()`方法, 该线程获取资源失败, 已经被放入等待队列尾部了。**那么线程就进入等待状态, 直到其他线程彻底释放资源后唤醒线程, 线程获取资源。**
> ```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true; // 标记是否成功拿到资源
    try {
        boolean interrupted = false; // 标记等待过程中是否被中断过
        // 自旋操作
        for (;;) {
            final Node p = node.predecessor(); // 拿到前置节点
            // 如果前置节点是head, 也就是说当前节点有资格尝试获取资源
            if (p == head && tryAcquire(arg)) {
                setHead(node); // 拿到资源后, 将head指向该节点。所以head节点就是可以获取资源的节点
                p.next = null; // setHead中node.prev=null, 此处再将head.next=null, 是为了方便GC回收以前的head节点。也就是之前的head节点拿完资源出队列了
                failed = false; // 成功获取资源
                return interrupted; 
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
> ```

> `shouldParkAfterFailedAcquire(Node, Node)`: 通过park()方法将线程变成waiting状态。在方法内部进行判断, 如果前置节点的状态不是`signal`, 那么就不断的往前找, 直到找到最近一个正常等待的状态的节点, 并排在后面。<br/>
> `parkAndCheckInterrupt()`: 此方法是真正的让线程进入等待状态。



