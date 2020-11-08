## J.U.C重入锁: StampedLock

StampedLock是为了优化可重入读写锁性能的一个工具。相比于`ReentrantReadWriteLock`主要多了乐观读的功能, 但是不支持重入。

我们知道:

- `ReentrantLock`相比与`Synchronized`:

    > `ReentrantLock`的API更加灵活, 而`Synchronized`是隐藏式锁, 锁的获取释放都是由JVM来控制的, 无法进行扩展, `ReentrantLock`的原理是自旋锁, `Synchronized`是重锁。

- `ReentrantReadWriteLock` 相对于 `ReentrantLock`:

    >在写操作时是需要加锁的, 但是在读操作是不需要加锁的。所以使用ReentrantReadWriteLock, 在多个线程进行读操作时，效率是很高的。

但是会有这样的一个问题:

假如有100个线程在使用读写分离锁这样的机制, 但是其中有99个线程需要读操作, 1个线程需要写操作, 这样就会导致"写饥饿"的情况。

也就是说写操作一直抢不到锁, 更新很慢, 但是查询很快。基于这种问题, 产生了`StampedLock`。

### StampedLock分析

`StampedLock`对读操作是乐观的, 但是这是相对于写操作而言的。为什么这么说呢?

> 比如在进行读操作, 拿到了读的锁, 那么写线程是拿不到锁的, 那么这个读操作就是悲观的。<br/>
> 但是如果在读的过程中, 发现允许写线程时可以进行抢锁, 这样写线程就会改变数据, 那么在读的过程中就会判断stamped标签是否发生改变, 如果stamped发生改变, 那就说明数据已经发生了改变, 那么读就不成功, 读锁就会释放。如果stamped还没有发生变化, 那读操作依然可以去读取数据。

所以简单的理解, 所谓悲观和乐观都是相对的.
- 如果在读的过程中, 写操作是拿不到锁的, 那么读操作就是悲观的
- 如果在读的过程中, 写操作依然可以抢锁, 那么读操作就是乐观的

我们用下面的代码, 来演示`StampedLock`的用法:

```java
public class StampedLockExample {

    private final static StampedLock lock = new StampedLock();
    private final static List<Long> data = Lists.newArrayList();
    public static void main(String[] args) {
        final ExecutorService executorService  = Executors.newFixedThreadPool(100);
        Runnable readTask = () -> {
            while (true) {
                read();
            }
        };
        Runnable writeTask = () -> {
            while (true) {
                write();
            }
        };

        executorService.submit(readTask);
        executorService.submit(readTask);
        executorService.submit(readTask);
        executorService.submit(readTask);
        executorService.submit(readTask);
        executorService.submit(readTask);
        executorService.submit(readTask);
        executorService.submit(readTask);
        executorService.submit(writeTask);
        executorService.submit(writeTask);
    }

    private static void read() {
        // 这是一个乐观锁, 而且不会被阻塞住
        long stamped = lock.tryOptimisticRead();

        //判断stamped是否被修改, 如果被修改说明, 有写操作对数据进行修改, 那么读操作就拿不到锁
        if (lock.validate(stamped)) {
            stamped = lock.readLock();
            try {
                Optional.of(data.stream().map(String::valueOf).collect(Collectors.joining("#", "R-", ""))).ifPresent(System.out::println);
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                lock.unlockRead(stamped);
            }
        }
    }

    private static void write() {
        long stamp = -1;
        try {
            stamp = lock.writeLock();
            data.add(System.currentTimeMillis());
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlockWrite(stamp);
        }
    }
}
```