## J.U.C线程池: ThreadPoolExecutor

`ThreadPoolExecutor`是`Executor`框架中最核心的类, 所以深入了解它是非常有必要的。

### 内部状态

线程有五种状态: 新建, 就绪, 运行, 阻塞, 死亡。线程池同样也有五种状态: **RUNNING, SHUTDOWN, STOP, TIDYING, TERMINATED。**

```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
private static final int COUNT_BITS = Integer.SIZE - 3;
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

// runState is stored in the high-order bits
private static final int RUNNING    = -1 << COUNT_BITS;
private static final int SHUTDOWN   =  0 << COUNT_BITS;
private static final int STOP       =  1 << COUNT_BITS;
private static final int TIDYING    =  2 << COUNT_BITS;
private static final int TERMINATED =  3 << COUNT_BITS;

// Packing and unpacking ctl
private static int runStateOf(int c)     { return c & ~CAPACITY; }
private static int workerCountOf(int c)  { return c & CAPACITY; }
private static int ctlOf(int rs, int wc) { return rs | wc; }
```

**变量ctl定义为AtomicInteger, 它的功能非常强大, 共32位。其中高3位表示"线程池状态", 低29位表示"线程池中的任务数量"。**

```java
RUNNING          -- 对应的高3位值是111。
SHUTDOWN         -- 对应的高3位值是000。
STOP             -- 对应的高3位值是001。
TIDYING          -- 对应的高3位值是010。
TERMINATED       -- 对应的高3位值是011。
```

- **Running**: 处于running状态的线程池能够接受新任务, 以及对新添加的任务进行处理。
- **shutdown**: 处于shutdown状态的线程池不可以接受新任务, 但是可以对已添加的任务进行处理
- **stop**: 处于stop状态的线程池不接受新任务, 不处理已添加的任务, 并且会中断正在处理的任务
- **tidying**: 当所有的任务已终止，ctl记录的"任务数量"为0，线程池会变为TIDYING状态。当线程池变为TIDYING状态时，会执行钩子函数terminated()。terminated()在ThreadPoolExecutor类中是空的，若用户想在线程池变为TIDYING时，进行相应的处理；可以通过重载terminated()函数来实现。
- **terminated**: 线程池彻底终止的状态。

各个状态的转换如下:

![executor_pool_state](../image/executor_pool_state.png)

### 线程池核心参数

我们可以通过`ThreadPoolExecutor`构造函数来创建一个线程池:

```java
 /**
 * Creates a new {@code ThreadPoolExecutor} with the given initial
 * parameters and default thread factory.
 *
 * @param corePoolSize the number of threads to keep in the pool, even
 *        if they are idle, unless {@code allowCoreThreadTimeOut} is set
 * @param maximumPoolSize the maximum number of threads to allow in the
 *        pool
 * @param keepAliveTime when the number of threads is greater than
 *        the core, this is the maximum time that excess idle threads
 *        will wait for new tasks before terminating.
 * @param unit the time unit for the {@code keepAliveTime} argument
 * @param workQueue the queue to use for holding tasks before they are
 *        executed.  This queue will hold only the {@code Runnable}
 *        tasks submitted by the {@code execute} method.
 * @param handler the handler to use when execution is blocked
 *        because the thread bounds and queue capacities are reached
 * @throws IllegalArgumentException if one of the following holds:<br>
 *         {@code corePoolSize < 0}<br>
 *         {@code keepAliveTime < 0}<br>
 *         {@code maximumPoolSize <= 0}<br>
 *         {@code maximumPoolSize < corePoolSize}
 * @throws NullPointerException if {@code workQueue}
 *         or {@code handler} is null
 */
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
        throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.acc = System.getSecurityManager() == null ?
            null :
            AccessController.getContext();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
```

`ThreadPoolExecutor`构造函数总共有7个参数, 这7个参数的意思一定要非常熟悉且理解, 极其重要的知识点。而对于这些参数的解释其实在源码中, 已经写得非常详细了。

> **corePoolSize:** 核心线程数。线程池中固定保持多少个线程, 即使线程是空闲的。当提交一个任务时, 线程池会新建一个线程来执行任务, 知道当前线程数等于corePoolSize。
>
> **maximumPoolSize:** 最大线程数。线程池中允许存在线程的最大数量。
>
> **keepAliveTime:** 线程空闲时间。线程的创建和销毁是需要代价的。线程执行完任务后不会立即销毁, 而是继续存活一段时间: keepAliveTime。默认情况下, 该参数只有线程数大于corePoolSize时才会生效。
>
> **unit:** keepAliveTime存活时间的单位。
>
> **workQueue:** 用来保存等待执行的任务的阻塞队列。等待的任务必须实现Runnable接口。
>
> **threadFactory:** 用于设置创建线程的工厂。ThreadFactory
>
> **RejectedExecutionHandler:** 线程池的拒绝策略。所谓拒绝策略, 是指将任务添加到线程池中, 线程池拒绝该任务所采取的的相应策略。

**这里的参数关系一定要搞清楚:**

**`corePoolSize`是核心线程大小, 线程池中固定保持多少个线程;**<br/>
**`workQueue`任务队列的作用是当线程池中正在执行任务的线程数量已达到核心线程数时, 再次提交任务就会先存到队列中进行等待, 直到达到队列的大小;**<br/>
**`maximumPoolSize`最大线程数, 是线程池允许存在的最大的线程数量。当`workQueue`等待任务已满时, 线程池就会从核心线程数扩充到最大线程数;**

> 这三个参数的关系: 当执行任务的线程数量达到核心线程数`corePoolSize`时, 提交的任务会存在`wokrQueue`等待执行; 但是如果提交任务过多, 达到了`workQueue`的大小, 那么线程池数量就会扩充到最大线程数`maximumPoolSize`; 而如果此时任务仍然不断的提交, 达到了最大线程数, 那么就会执行相应的拒绝策略。

按照上面的解释, 如果corePoolSize = 1, maximumPoolSize = 2, workQueue.size = 1; 如果任务执行时间过长< 导致线程还没来得及回收, 那么最多只能提交3个任务。当提交第4个任务是就会执行拒绝策略。
