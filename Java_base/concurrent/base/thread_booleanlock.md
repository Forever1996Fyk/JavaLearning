## 利用boolean值实现显示锁的功能

### 1. `synchronized`关键字存在的问题

如果多个线程调用一个`synchronized`方法, 如果这个执行这个的方法的时间太长, 那么其中一个线程就一直占用锁, 无法释放, 其他线程就只能处于等待状态, 只能通过其他外部中断的方法, 中断线程的执行, 如下面代码的:

```java
public class SynchronizedProblem {


    public static void main(String[] args) throws InterruptedException {
        new Thread() {
            @Override
            public void run() {
                System.out.println("t1");
                SynchronizedProblem.run();
            }
        }.start();

        Thread.sleep(1000);

        Thread t2 = new Thread() {
            @Override
            public void run() {
                System.out.println("t2");
                SynchronizedProblem.run();
            }
        };

        t2.start();
        Thread.sleep(2000);
        t2.interrupt();
        System.out.println(t2.isInterrupted());

    }

    private synchronized static void run() {
        System.out.println(Thread.currentThread().getName());
        while (true) {
        }
    }
}
```

从上面的代码看出, 如果持有锁的时间太长, 只能通过interrupt()方法, 中断相应的线程。才能保证程序不阻塞。那么有没有好的方法可以解决这个问题?

> 我们利用一个boolean标识, 才实现自定义的显示锁, 可以显示的获取锁, 释放锁, 定义超时时间。

### 2. 显示锁

直接上代码:

先定义一个`interface Lock`:

```java
public interface Lock {
    class TimeOutException extends Exception {
        public TimeOutException(String message) {
            super(message);
        }
    }

    void lock() throws InterruptedException;

    void lock(long mills) throws InterruptedException, TimeoutException;

    void unlock();

    Collection<Thread> getBlockedThread();

    int getBlockedSize();
}
```

实现`Lock`接口:

```java
public class BooleanLock implements Lock {

    //初始值 表示是否获得锁, false表示未获得, true表示已获得
    private boolean initValue;

    // 阻塞线程集合
    private Collection<Thread> blockedThreadCollection = new ArrayList<>();

    //这里指定的当前线程, 表示只能是 哪个线程获得锁，那个线程才能去释放锁
    private Thread currentThread;

    public BooleanLock() {
        this.initValue = false;
    }

    @Override
    public synchronized void lock() throws InterruptedException {
        // 如果已经达到了锁, 那么BooleanLock的实例就wait
        while (initValue) {
            blockedThreadCollection.add(Thread.currentThread());
            this.wait();
        }
        blockedThreadCollection.remove(Thread.currentThread());
        this.initValue = true;
        this.currentThread = Thread.currentThread();
    }

    @Override
    public synchronized void lock(long mills) throws InterruptedException, TimeoutException {
        if (mills <= 0) {
            lock();
        }
        long hasRemaining = mills;
        long endTime = System.currentTimeMillis() + mills;
        while (initValue) {
            if (hasRemaining <= 0) {
                throw new TimeoutException("Time out");
            }
            blockedThreadCollection.add(Thread.currentThread());
            this.wait(mills);
            hasRemaining = endTime - System.currentTimeMillis();
        }
        blockedThreadCollection.remove(Thread.currentThread());
        this.initValue = true;
        this.currentThread = Thread.currentThread();
    }

    @Override
    public synchronized void unlock() {
        if (Thread.currentThread() == currentThread) {
            this.initValue = false;
            Optional.of(Thread.currentThread() + " released the lock monitor.").ifPresent(System.out::println);
            // 注意，只有notifyAll之后, 之后的线程才会进入可执行状态, 后面的线程才会继续执行
            this.notifyAll();       
        }
    }

    @Override
    public Collection<Thread> getBlockedThread() {
        return Collections.unmodifiableCollection(blockedThreadCollection);
    }

    @Override
    public int getBlockedSize() {
        return blockedThreadCollection.size();
    }
}
```

编写测试类 `LockTest`:

```java
public class LockTest {
    public static void main(String[] args) {
        final BooleanLock booleanLock = new BooleanLock();
        Stream.of("T1", "T2", "T3", "T4").forEach(name -> new Thread(() -> {
            try {
                booleanLock.lock(100L);
                Optional.of(Thread.currentThread().getName() + " have the lock Monitor").ifPresent(System.out::println);
                work();
            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (TimeoutException e) {
                Optional.of(Thread.currentThread().getName() + " time out").ifPresent(System.out::println);
            } finally {
                booleanLock.unlock();
            }
        }, name).start());
    }

    private static void work() throws InterruptedException {
        Optional.of(Thread.currentThread().getName() + " is Working...").ifPresent(System.out::println);
        Thread.sleep(10_000);
    }
}
```

测试结果:

```txt
T1 have the lock Monitor
T1 is Working...
T2 time out
T3 time out
T4 time out
Thread[T1,5,main] released the lock monitor.
```