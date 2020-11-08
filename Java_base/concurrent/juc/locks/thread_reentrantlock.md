## J.U.C重入锁: ReentrantLock

`ReentrantLock`, 可重入锁。它可以等同于`synchronized`的使用, 但是`ReentrantLock`比`synchronized`更强大, 更灵活。

> 什么是可重入锁?

可重入锁的字面意思是"可以重新进入的锁"**即允许同一个线程多次获取同一把锁**。比如一个递归函数中有加锁操作, 递归过程中这个锁会阻塞自己吗?如果不会, 那这个锁就是**可重入锁**(因为这个原因可重入锁也叫做递归锁)

JDK提供的所有现成的Lock实现类, 包括`synchronized`关键字都是都可重入锁。

`ReentrantLock`还提供了公平锁和非公平锁的选择。构造方法可以接受一个公平的参数(默认非公平锁), 当设置为true时,表示公平锁, 否则为非公平锁。

**公平锁与非公平锁的区别在于: 公平锁中线程获取锁是顺序的。**但是公平锁的效率比非公平锁效率低一些, 在多线程访问下, 公平锁吞吐量要低。

### ReentrantLock分析

在ReentrantLock中核心的代码就是获取锁和释放锁。

```java
ReentrantLock lock = new ReentrantLock();

//获取锁
lock.lock();
lock.lockInterruptibly();
lock.tryLock();

// 释放锁
lock.unlock();
```


##### 1.1 获取锁

对于获取锁的方法, 都是在`ReentrantLock`中定义的私有内部非`Sync`中实现的, 而`Sync`继承了AQS。

```java
public void lock() {
    sync.lock();
}

abstract static class Sync extends AbstractQueuedSynchronizer { ... }
```

而`Sync`又有两个子类: 公平锁`FairSync`, 非公平锁`NonfairSync`。Sync内部定义了lock()的抽象方法, 都是由这两个子类去实现的。

我们看到`NonfairSync`中的lock()方法:

```java
final void lock() {
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}

protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);
}
```

它调用了AQS中的`acquire()`方法, 并且重写了`tryAcquire()`方法。也正如在[AQS](../thread_aqs.md)中所说的那样, 由子类自定义同步器去实现对应的方法。

实现的主要逻辑是: 

1. **首先判断同步状态state是否为0, 如果是表示该资源还没有被线程获取, 直接通过CAS获取同步状态(也就是通过CAS更新state为1), 如果CAS成功则返回true。**
2. **如果state!=0, 则判断当前线程是否为获取锁的线程, 如果是则重新获取锁, 成功返回true。成功获取锁的线程再次获取锁, 只是增加state(这就是线程可重入)**

对于三种不同的获取锁的方法, 它们的区别是:
1. `lock()`不会阻止线程的打断, 如果一个线程在等待状态时调用interrupt()方法时, 是无法打断的
2. `lockInterruptibly()`表示线程是可以被中断的。如果被中断则抛出中断异常
3. `tryLock()`会去尝试的获取锁, 它会有一个boolean的返回值, 如果获取不到锁, 就直接返回false。

##### 1.2 释放锁

对于释放锁的逻辑其实不用多说, `unlock()`方法底层就是调用AQS的`release()`方法, 并且重写了`tryRelease()`。

最终的结果就是释放资源时, 如果state==0, 则将持有资源的线程设置为null, free=true, 表示释放成功。

##### 1.3 公平锁与非公平锁

看过AQS都知道, 等待的线程都是存放在一个FIFO队列中。那么公平锁和非公平锁的区别就在于**是否按照FIFO的顺序来唤醒线程!!!**

所以我们看公平锁的`tryAcquire(int arg)`:
```java
/**
 * Fair version of tryAcquire.  Don't grant access unless
 * recursive call or no waiters or is first.
 */
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
}
```

相比较非公平锁, 公平锁在获取同步状态是多了一个限制条件: `hasQueuedPredecessors()`, 该方法主要是判断当前线程是否位于CLH队列中的第一个。如果是则返回true, 否则返回false。


##### 1.4 ReentrantLock与Synchronized的区别

之前提到ReentrantLock提供了比synchronized更加灵活和强大的锁机制，那么它的灵活和强大之处在哪里呢？

1. 与`Synchronized`相比, `ReentrantLock`提供更多灵活更加全面的功能。例如, 时间获取锁`tryLock(long timeout, TimeUnit unit)`, 可中断锁`tryInterruptibly()`等
2. `ReentrantLock`提供的可轮询的锁请求, 它会尝试的获取锁。但是`synchronized`一旦进入锁请求要么成功, 要么阻塞。所以相比于`synchronized`而言, `ReentrantLock`不容易产生死锁。