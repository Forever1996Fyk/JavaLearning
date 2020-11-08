## J.U.C读写锁: ReentrantReadWriteLock

重入锁`ReentrantLock`是互斥排它锁。在同一时刻只能有一个线程可以访问, 与`synchronized`的语义基本一样。

> 如果有某个场景: 执行某个操作大部分时间都是提供的读操作, 而写操作占用的时间较少。然而读操作是不存在数据竞争问题, 如果一个线程在读操作的时候禁止其他线程进行读操作, 那么对这个操作加上互斥锁, 必定会导致性能的降低。所以就提供了读写锁。

**读写锁维护着一对锁, 一个读锁和一个写锁。通过分离读锁和写锁, 使得并发性比互斥锁要更好。在同一时间可以允许多个读线程同时访问, 但是在写线程访问时, 所有读写线程都会被阻塞。**

在之前的文章中, 我们利用wait(), notify()方法实现过自定义的读写锁, 在那篇文章中, 说明了读写锁的使用情况。[自定义读写分离锁](Java_base/concurrent/promote/thread_readwritelock.md)

读写锁的主要特性:

1. 公平性: 支持公平性和非公平性
2. 重入性: 支持重入。
3. 锁降级: 遵循 **获取写锁->获取读锁->释放写锁**的次序, 写锁能够降级为读锁。

`ReentrantReadWriteLock`定义如下:

```java
 /** 内部类  读锁 */
private final ReentrantReadWriteLock.ReadLock readerLock;
/** 内部类  写锁 */
private final ReentrantReadWriteLock.WriteLock writerLock;

final Sync sync;

/** 使用默认（非公平）的排序属性创建一个新的 ReentrantReadWriteLock */
public ReentrantReadWriteLock() {
    this(false);
}

/** 使用给定的公平策略创建一个新的 ReentrantReadWriteLock */
public ReentrantReadWriteLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
    readerLock = new ReadLock(this);
    writerLock = new WriteLock(this);
}

/** 返回用于写入操作的锁 */
public ReentrantReadWriteLock.WriteLock writeLock() { return writerLock; }
/** 返回用于读取操作的锁 */
public ReentrantReadWriteLock.ReadLock  readLock()  { return readerLock; }

abstract static class Sync extends AbstractQueuedSynchronizer {
    /**
     * 省略其余源代码
     */
}
public static class WriteLock implements Lock, java.io.Serializable{
    /**
     * 省略其余源代码
     */
}

public static class ReadLock implements Lock, java.io.Serializable {
    /**
     * 省略其余源代码
     */
}
```

`ReentrantReadWriteLock`与`ReentrantLock`一样, 锁的主体依然是定义内部类Sync, 它的读锁, 写锁都是依靠Sync来实现的,只不过获取锁的方式不同。

- 在`ReentrantLock`中使用一个int 类型的state来表示同步状态, 该值表示锁被线程重复获取的次数
- **`ReentrantReadWriteLock`内部维护两个锁, 需要用一个变量维护多种状态。**
    > 所以读写锁采用 "按位切割使用" 的方式来维护变量, 将变量切分为两部分。<br/>
    > 高16位表示读, 低16位表示写。分割之后, 读写锁是如何迅速确定读锁和写锁的状态呢? **通过位运算**<br/>
    > 假如当前同步状态为S, 那么写状态等于 `S & 0x0000FFFF`将高16位全部抹去; 读状态等于 `S >>> 16`无符号补0右移16位。

    ```java
    static final int SHARED_SHIFT   = 16;
    static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
    static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
    static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

    static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
    static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
    ```

用一个案例来简单的看下, 读写锁的用法:

```java
public class ReadWriteLockExample {
    private final static ReentrantReadWriteLock readWriteLock = new ReentrantReadWriteLock(true);

    private final static Lock readLock = readWriteLock.readLock();
    private final static Lock writeLock = readWriteLock.writeLock();

    private static final List<Long> data = Lists.newArrayList();

    /**
     * 实现读写锁的基本要求为:
     *
     * 可以同时读,
     * 不可同时写,
     * 当有线程写入时, 不可读,
     * 当有线程读时, 不可写入,
     *
     * 所以如果使用一般的锁, 对于有很多读线程的操作时, 效率是不高的。这时候就需要读写锁
     *
     * @param args
     * @throws InterruptedException
     */
    public static void main(String[] args) throws InterruptedException {
        Thread thread1 = new Thread(ReadWriteLockExample::write);
        thread1.start();

        TimeUnit.SECONDS.sleep(1);

        Thread thread2 = new Thread(ReadWriteLockExample::write);
        thread2.start();

    }

    public static void write() {
        try {
            writeLock.lock();
            data.add(System.currentTimeMillis());
            TimeUnit.SECONDS.sleep(5);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            writeLock.unlock();
        }
    }

    public static void read() {
        try {
            readLock.lock();
            data.forEach(System.out::println);
            TimeUnit.SECONDS.sleep(5);
            System.out.println(Thread.currentThread().getName() + " =======================");
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            readLock.unlock();
        }
    }
}
```

### 写锁

写锁是一个可重入的互斥锁。

之前学过AQS知道, 写锁是互斥锁, 也就是独占模式。所以它一定重写了AQS的`tryAcquire()`方法:

```java
protected final boolean tryAcquire(int acquires) {
    Thread current = Thread.currentThread();
    //当前锁个数
    int c = getState();
    //写锁
    int w = exclusiveCount(c);
    if (c != 0) {
        //c != 0 && w == 0 表示存在读锁
        //当前线程不是已经获取写锁的线程
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;
        //超出最大范围
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        setState(c + acquires);
        return true;
    }
    //是否需要阻塞
    if (writerShouldBlock() ||
            !compareAndSetState(c, c + acquires))
        return false;
    //设置获取锁的线程为当前线程
    setExclusiveOwnerThread(current);
    return true;
}
```

该方法与`ReentrantLock`的`tryAcquire()`大致相同。只是在判断重入时, 增加了一项条件: 读锁是否存在。**因为要确保写锁的操作对读锁是可见的, 如果在存在读锁的情况下允许获取写锁, 那么那些已经获取读锁的其他线程可能无法感知当前写线程的操作。所以只有等读锁完全释放后, 写锁才能够被当前线程所获取, 一旦写锁获取了, 所以其他读,写线程都会被阻塞。**

也就是说, 一定要符合读写分离锁的4种情况。
- 当都是读操作时, 不会阻塞。
- 存在读写操作时, 一定会等待读锁释放之后才会进行写操作。

### 读锁

读锁是可重入的共享锁。它能够被多个线程同时持有, 在没有其他写线程访问时, 读锁总是可以获取成功。

读锁是通过重写AQS的`tryAcquireShared()`方法实现的:

```java
protected final int tryAcquireShared(int unused) {
    //当前线程
    Thread current = Thread.currentThread();
    int c = getState();
    //exclusiveCount(c)计算写锁
    //如果存在写锁，且锁的持有者不是当前线程，直接返回-1
    //存在锁降级问题，后续阐述
    if (exclusiveCount(c) != 0 &&
            getExclusiveOwnerThread() != current)
        return -1;
    //读锁
    int r = sharedCount(c);

    /*
        * readerShouldBlock():读锁是否需要等待（公平锁原则）
        * r < MAX_COUNT：持有线程小于最大数（65535）
        * compareAndSetState(c, c + SHARED_UNIT)：设置读取锁状态
        */
    if (!readerShouldBlock() &&
            r < MAX_COUNT &&
            compareAndSetState(c, c + SHARED_UNIT)) {
        /*
            * holdCount部分后面讲解
            */
        if (r == 0) {
            firstReader = current;
            firstReaderHoldCount = 1;
        } else if (firstReader == current) {
            firstReaderHoldCount++;
        } else {
            HoldCounter rh = cachedHoldCounter;
            if (rh == null || rh.tid != getThreadId(current))
                cachedHoldCounter = rh = readHolds.get();
            else if (rh.count == 0)
                readHolds.set(rh);
            rh.count++;
        }
        return 1;
    }
    return fullTryAcquireShared(current);
}
```

读锁的获取比较复杂, 整个过程如下:

1. 如果存在写锁而且写锁的持有者不是当前线程则直接返回失败, 否则继续
2. 判断读锁是否需要阻塞, 读锁持有的线程数小于最大值(65535), 则设置锁状态成功, 并返回1.如果不满足条件, 执行`fullTryAcquireShared()`。

    - `fullTryAcquireShared()`会根据"是否需要阻塞等待", "读取锁的共享次数是否超过限制"等进行处理。如果不需要等待, 并且锁的共享次数没有超过限制, 则通过CAS尝试获取锁, 并返回1.
    - `HoldCounter`: 我认为这是一个计数器, 用于技术对每次共享锁的操作。获取共享锁, 计数器+1; 释放共享锁, 计数器-1.只有当线程获取共享锁后才能对共享锁进行释放, 重入操作。

### 锁降级

锁降级是读写锁的一个特性, 它意味着写锁可以降级为读锁, 但是需要遵循 **获取写锁->获取读锁->释放写锁** 的顺序。在获取读锁的方法`tryAcquireShared(int unused)`中, 有一段代码是来判断读锁降级的:

```java
int c = getState();
//exclusiveCount(c)计算写锁
//如果存在写锁，且锁的持有者不是当前线程，直接返回-1
//存在锁降级问题
if (exclusiveCount(c) != 0 &&
        getExclusiveOwnerThread() != current)
    return -1;
//读锁
int r = sharedCount(c);
```

> 所谓锁降级就是当前线程, 先拿到写锁之后, 又拿到了读锁, 当此线程先释放写锁时, 此时该线程拿到的锁只有读锁了, 也就是写锁降级为了读锁。**这里要注意的是, 一个线程时既可以获取写锁, 又可以获取读锁的。这就是锁的可重入性**


但是我在网上找了很多锁降级的资料, 基本上有锁降级的了解都是下面的两个关键:

1. 在锁降级过程中, 中间一步获取读锁是非常有必要的。假如, 当前线程A在没有获取读锁的情况下, 直接释放了写锁, 那么此时线程B获取了写锁, 这个线程B对数据的修改是不会当前线程A可见的。而如果获取了读锁, 则线程B在获取写锁的过程中判断如果有读锁还没释放, 就会被阻塞, 只有当前线程A释放读锁后,线程B才会获取写锁成功。
2. 为了性能，因为读锁的抢占必将引起资源分配和判断等操作，降级锁减少了线程的阻塞唤醒，实时连续性更强。

还有人用下面的代码举例:

```java
class CachedData {
    Object data;
    //volatile修饰，保持内存可见性
    volatile boolean cacheValid;
    //可重入读写锁
    final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
 
    void processCachedData() {
    	//首先获取读锁
      rwl.readLock().lock();
      //发现没有缓存数据则放弃读锁，获取写锁
      if (!cacheValid) {
        // Must release read lock before acquiring write lock
        rwl.readLock().unlock();
        rwl.writeLock().lock();
        try {
          // Recheck state because another thread might have
          // acquired write lock and changed state before we did.
          if (!cacheValid) {
            data = ...
            cacheValid = true;
          }
          // Downgrade by acquiring read lock before releasing write lock
          rwl.readLock().lock();
        } finally {
        	//进行锁降级
          rwl.writeLock().unlock();
           // Unlock write, still hold read
        }
      }
 
      try {
        use(data);
      } finally {
        rwl.readLock().unlock();
      }
    }
}
```

但是我看了上面的代码, 从代码逻辑上感觉, 即使不用到锁降级, 也不会影响整体的数据。
因为cacheValid被volatile修饰, 只要有写线程进入代码, 已经把这个标识改为true了, 如果直接释放写锁, 其他写线程依然不会进入第二个if判断, 因为volatile的可见性, 其他线程拿到的标识已经改为true。所以不会影响data的变化。

到目前为止, 我还没有发现有合适的案例在解释锁降级的目的。