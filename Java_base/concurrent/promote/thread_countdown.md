## CountDown设计模式

CountDown是一个同步工具, 用来协调多个线程之间的同步。 在Java1.5, JDK出来一个工具类`CountDownLatch`, 我们后面会介绍。

它能够是一个线程在等待另外的一些线程完成各自工作之后, 再继续执行。**使用一个计数器进行实现。计数器初始值为线程的数量, 当每一个线程完成自己的任务后, 计数器的值就会减一, 当计数器的值为0时, 表示所有的线程都已经完成了任务, 然后通过唤醒方式, 继续执行后面的任务。**

> 我们这里使用wait(), notifyAll()方法自定义实现CountDown的功能。

### 代码实现

其实代码的实现并不难, 定义一个计数变量初始值为0, 每次线程执行完一个任务, 就加1, 并且线程处于等待状态, 直到这个计数变量达到总数, 就唤醒所有等待的线程。

```java
public class CountDown {

    private final int total;

    private int counter = 0;

    public CountDown(int total) {
        this.total = total;
    }

    public void down() {
        synchronized (this) {
            this.counter++;
            this.notifyAll();
        }
    }

    public void await() throws InterruptedException {
        synchronized (this) {
            while (counter != total) {
                this.wait();
            }
        }
    }
}
```

测试:

```java
public class CustomCountDown {

    private static final Random random = new Random(System.currentTimeMillis());

    public static void main(String[] args) throws InterruptedException {
        final CountDown latch = new CountDown(5);
        System.out.println("准备多线程处理任务.");

        IntStream.rangeClosed(1, 5).forEach(i -> new Thread(() -> {
            System.out.println(Thread.currentThread().getName() + " is working.");
            try {
                Thread.sleep(random.nextInt(1000));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            latch.down();
        }, String.valueOf(i)).start());

        latch.await();
        System.out.println("多线程任务全部结束, 准备第二阶段任务");
        System.out.println(".............");
        System.out.println("finish");
    }
}
```

结果:

```txt
准备多线程处理任务.
1 is working.
3 is working.
2 is working.
5 is working.
4 is working.
多线程任务全部结束, 准备第二阶段任务
.............
finish
```