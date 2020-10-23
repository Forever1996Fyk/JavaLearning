## Thread Join方法

其实Thread.join这个api很少使用, 在实际项目中基本上是不会用到。

### 1. Thread.join的作用

对于`Thread.join`的作用, 很多人都会有误解, 认为`Thread.join`是用来保证线程的顺序性。但实际上并不是这样的。看下面的一段代码:

```java
public class ThreadJoin {

    public static void main(String[] args) throws InterruptedException {
        /**
         * 先执行t1完之后, 在执行main线程
         */
        Thread t1 = new Thread(() -> {
            IntStream.range(1, 1000)
                    .forEach(i -> System.out.println(Thread.currentThread().getName() + "->" + i));
        });

        Thread t2 = new Thread(() -> {
            IntStream.range(1, 1000)
                    .forEach(i -> System.out.println(Thread.currentThread().getName() + "->" + i));
        });

        t1.start();
        t2.start();

        /**
         * 当前线程执行完, 线程死了之后 才会执行后面的线程, 必须要在start()后调用
         *
         * 注意: 这里当前线程指的是main线程, 也就是说上面的两个线程依然会交叉输入, t1, t2都执行结束之后, 在执行main线程的代码.
         *
         * 所以join可以理解为, 在所有子线程都执行完的情况下, 在执行main线程
         */
        t1.join();
        t2.join();

        Optional.of("All of tasks finish done.").ifPresent(System.out::println);
        IntStream.range(1, 1000)
                .forEach(i -> System.out.println(Thread.currentThread().getName() + "->" + i));
    }
}
```

这里代码中的注释说的很清楚, 首先join()必须要在start()之后调用。其次, 当调用join()方法时, 调用join()方法的线程会处于阻塞状态(上面的代码就是主线程main处于阻塞状态, 如果这里另外启动一个线程t3来调用t1.join(), t2.join(), 那么t3线程就会处于阻塞状态)。**但是要注意的是, 当前线程(这里指的main)会阻塞, 但是对于t1,t2两个线程启动, 依然不会保证顺序执行。所以认为join可以保证线程的顺序性是错误的。**

所以上面代码打印出来的结果, t1, t2线程分别交叉打印1-1000, 等到t1,t2都打印完成后, main才开始打印。

### 2. Thread.join的原理
先贴上源码:

```java
public class Thread implements Runnable {
    ...
    public final void join() throws InterruptedException {
        join(0);
    }
    ...
    public final synchronized void join(long millis) throws InterruptedException {
        long base = System.currentTimeMillis();
        long now = 0;
        if (millis < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }
        if (millis == 0) { //判断是否携带阻塞的超时时间，等于0表示没有设置超时时间
            while (isAlive()) {//isAlive获取线程状态，无线等待直到previousThread线程结束
                wait(0); //调用Object中的wait方法实现线程的阻塞
            }
        } else { //阻塞直到超时
            while (isAlive()) { 
                long delay = millis - now;
                if (delay <= 0) {
                    break;
                }
                wait(delay);
                now = System.currentTimeMillis() - base;
            }
        }
    }
    ...
```

从源码中可以看到, join方法的本质调用的是Object中的wait/notify/notifyAll方法来实现的通信。

当main线程执行join方法时, 它会拿到t1,t2线程的锁, 然后调用wait方法去阻塞。而在线程执行完毕以后会有一个唤醒操作, 只是我们不需要关心, 因为在jdk底层已经帮我们做了。

### 3. 使用场景

在实际开发中, 很少会用到join的方法, 因为有很多工具类会实现跟join相关的功能。

> 案例: 多台机器去采集数据, 并保存到数据库。我们想要获取采集过程所花费的时间。

```java
public class ThreadJoin3 {
    public static void main(String[] args) throws InterruptedException {
        long startTime = System.currentTimeMillis();
        Thread t1 = new Thread(new CaptureRunnable("M1", 10000L));
        Thread t2 = new Thread(new CaptureRunnable("M2", 30000L));
        Thread t3 = new Thread(new CaptureRunnable("M3", 15000L));

        t1.start();
        t2.start();
        t3.start();

        /**
         * 在不用join的情况下, main线程在t1,t2,t3还没执行完, 就保存了开始结束时间, 这样很明显是不符合我们的需求的, 所以这时候就可以使用join。
         *
         * 在t1,t2,t3都执行完的情况下, 在保存时间。
         */
        t1.join();
        t2.join();
        t3.join();

        long endTime = System.currentTimeMillis();
        System.out.printf("save data begin timestamp is: %s, end timestamp is : %s \n", startTime, endTime);
    }
}

class CaptureRunnable implements Runnable {
    private String machineName;

    private long spendTime;

    public CaptureRunnable(String machineName, long spendTime) {
        this.machineName = machineName;
        this.spendTime = spendTime;
    }

    @Override
    public void run() {
        // do the really capture data
        try {
            Thread.sleep(spendTime);
            System.out.printf(machineName + " completed data capture at timestamp [%s] and successful \n", System.currentTimeMillis());
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public String getResult() {
        return machineName + " finish.";
    }
}
```


