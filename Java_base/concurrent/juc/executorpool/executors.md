## Executors线程池工厂类

Java通过Executors工厂类提供了4中线程池, 分别为:

- `newFixedThreadPool`
- `newCachedThreadPool`
- `newSingleThreadExecutor` 
- `newScheduledThreadPool`

我们通过代码来详细的介绍这前三种线程池的用法。

### 用法解析

#### newFixedThreadPool

源码如下:

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>());
}
```

通过上面的源码可以看出, 传入的参数nThreads, 既是corePoolSize, 又是maximumPoolSize; 这就意味着无论如何该线程池最多只会有nThreads线程来执行任务, 而且没有执行策略, 因为使用的阻塞队列是LinkedBlockingQueue, 这是无边界的阻塞队列。keepAliveTime为0, 也就说明了空闲线程永远都不会回收。

所以newFixedThreadPool会创建一个固定线程大小的线程池, 能够很好的控制并发数。


#### newCachedThreadPool

源码如下:

```java
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                    60L, TimeUnit.SECONDS,
                                    new SynchronousQueue<Runnable>());
}
```

通过源码可以发现, `newCachedThreadPool`返回的是corePoolSize=0, maximumPoolSize=Integer.MAX_VALUE的线程池, 而且阻塞队列是`SynchronizedQueue`, 意思是队列每次只能插入一条数据。也就是说阻塞队列只能存储一个任务, 而且空闲线程的存活时间是60s。

所以, 当我们提交100任务时, 由于不会存储任务, 只要发现有空闲线程, 就启用100个worker线程去处理任务, 在等待60s后, 如果还有空闲线程, 就全部回收。

这也是`newCachedThreadPool`的特性, 执行任务的线程多, 但是很快。所以这个线程池适用于执行一些非常短的异步任务, 如果执行时间很长, 那么线程池的工作线程会越来越多, 因为它的最大线程数是Integer.MAX_VALUE。

#### newSingleThreadExecutor

源码如下:

```java
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
```

我们从这个线程池的名字就能推断出, 这是只能产生一个工作线程的线程池。

> 为什么还要有这样的一种线程池, 它与new Thread的区别是什么?

1. Thread执行完一个任务之后, 周期就结束了; 但是`newSingleThreadExecutor`中的线程可以复用继续执行队列中的任务。

2. Thread无法将任务提交到queue中; 但是`newSingleThreadExecutor`可以。

我们再看`newSingleThreadExecutor`构造的源码发现, newSingleThreadExecutor实际上是`FinalizableDelegatedExecutorService`对ThreadPoolExecutor进行的包装;

`FinalizableDelegatedExecutorService`, 和它的父类`DelegatedExecutorService` 都是Executors的内部类, 源码如下:

```java
static class DelegatedExecutorService extends AbstractExecutorService {
    private final ExecutorService e;
    DelegatedExecutorService(ExecutorService executor) { e = executor; }
    public void execute(Runnable command) { e.execute(command); }
    public void shutdown() { e.shutdown(); }
    public List<Runnable> shutdownNow() { return e.shutdownNow(); }
    public boolean isShutdown() { return e.isShutdown(); }
    public boolean isTerminated() { return e.isTerminated(); }
    public boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException {
        return e.awaitTermination(timeout, unit);
    }
    public Future<?> submit(Runnable task) {
        return e.submit(task);
    }
    public <T> Future<T> submit(Callable<T> task) {
        return e.submit(task);
    }
    public <T> Future<T> submit(Runnable task, T result) {
        return e.submit(task, result);
    }
    public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException {
        return e.invokeAll(tasks);
    }
    public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                            long timeout, TimeUnit unit)
        throws InterruptedException {
        return e.invokeAll(tasks, timeout, unit);
    }
    public <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException {
        return e.invokeAny(tasks);
    }
    public <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                            long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException {
        return e.invokeAny(tasks, timeout, unit);
    }
}

static class FinalizableDelegatedExecutorService
    extends DelegatedExecutorService {
    FinalizableDelegatedExecutorService(ExecutorService executor) {
        super(executor);
    }
    protected void finalize() {
        super.shutdown();
    }
}
```

我们发现,DelegatedExecutorService是一个内部包装类, 它仅仅只暴露了`ExecutorService`的实现方法, 所以被`DelegatedExecutorService`包装的线程池, 无法调用`ThreadPoolExecutor`中的所有方法。

所以`newSingleThreadExecutor`无法强制转换成ThreadPoolExecutor, 也无法使用ThreadPoolExecutor中的一些方法。