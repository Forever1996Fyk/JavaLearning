## J.U.C并发工具类: Semaphore

`Semaphore`是信号量的意思, 控制访问多个共享资源的计数器, 和`CountDownLatch`一样, 本质上是一个**共享锁**。

下面我们就一个停车场的简单例子来阐述Semaphore：

> 假设停车场仅有5个停车位，一开始停车场没有车辆所有车位全部空着，然后先后到来三辆车，停车场车位够，安排进去停车，然后又来三辆，这个时候由于只有两个停车位，所有只能停两辆，其余一辆必须在外面候着，直到停车场有空车位，当然以后每来一辆都需要在外面候着。当停车场有车开出去，里面有空位了，则安排一辆车进去（至于是哪辆 要看选择的机制是公平还是非公平）。

从程序角度看，停车场就相当于信号量`Semaphore`，其中许可数`permits`为5，车辆就相对线程。当来一辆车时，许可数就会减 1 ，当停车场没有车位了（许可书 == 0 ），其他来的车辆需要在外面等候着。如果有一辆车开出停车场，许可数 + 1，然后放进来一辆车。

当一个线程想要访问某个共享资源时, 它必须要先获取许可证`permits`, 当`permits > 0`时, 可以获取该资源并且`permits-1`, 如果`permits = 0`, 则表示全部的共享资源已经被其他线程全部占用, 线程必须要等待其他线程释放资源。当线程释放资源时`permits + 1`,  

### 实现分析

`Semaphore`提供了两个构造函数:
- `Semaphore(int permits)`:创建具有给定的许可数和非公平的Semaphore
- `Semaphore(int permits, boolean fair)`: 创建具有给定的许可数和给定的公平设置的 Semaphore。

```java
public Semaphore(int permits) {
    sync = new NonfairSync(permits);
}

public Semaphore(int permits, boolean fair) {
    sync = fair ? new FairSync(permits) : new NonfairSync(permits);
}
```


##### 1. 获取许可证

`Semaphore`提供`acquire()`方法来获取一个许可证:

```java
public void acquire() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
```

##### 2. 释放许可证

`Semaphore`提供`release()`方法来释放许可证:

```java
public void release() {
    sync.releaseShared(1);
}
```

### 应用实例

我们利用`Semaphore`的性质, 再来实现一个自定义显示锁:

```java
public class SemaphoreExample1 {

    public static void main(String[] args) {
        final SemaphoreLock lock = new SemaphoreLock();

        for (int i = 0; i < 2; i++) {
            new Thread(() -> {
                try {
                    System.out.println(Thread.currentThread().getName() + " is Running");
                    lock.lock();
                    System.out.println(Thread.currentThread().getName() + " get the $SemaphoreLock");
                    TimeUnit.SECONDS.sleep(10);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    lock.unlock();
                }

                System.out.println(Thread.currentThread().getName() + " release.");

            }).start();
        }
    }

    static class SemaphoreLock {
        private final Semaphore semaphore = new Semaphore(1);

        public void lock() throws InterruptedException {
            // 申请一个许可证
            semaphore.acquire();
        }

        public void unlock() {
            // 释放许可证
            semaphore.release();
        }
    }
}
```