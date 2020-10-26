## 经典面试问题————实现生产者消费者需求

对于生产者消费者问题, 不同的语言, 都有不同的实现方式, 不同的算法。在java中对于生产者消费者模式, 基本上就是用线程来实现的。这也不难理解, 一个线程用于生产数据, 另一个线程用户消费数据。

### 生产者消费者模式_Version1

先看下面一段代码, 最基本的生产者消费者模式:

```java
public class ProduceConsumerVersion1 {

    private int i = 1;

    private final Object lock = new Object();

    /**
     * 生产者, 写入数据
     */
    private void produce() {
        synchronized (lock) {
            System.out.println("P->" + (i++));
        }
    }

    /**
     * 消费者 读出数据
     */
    private void consume() {
        synchronized (lock) {
            System.out.println("C->" + i);
        }
    }

    public static void main(String[] args) {
        ProduceConsumerVersion1 pc1 = new ProduceConsumerVersion1();
        new Thread("Produce") {
            @Override
            public void run() {
                while (true) {
                    pc1.produce();
                }
            }
        }.start();

        new Thread("Consume"){
            @Override
            public void run() {
                while (true) {
                    pc1.consume();
                }
            }
        }.start();

    }
}
```
只是最基础的生产者消费者模式, 但是有很大的问题。在测试的过程中发现, `produce`会不断的产生新的数据, 但是`consume`并不会立马去读新的数据。因为只有当`consume`线程抢到object锁才会去读数据, 这就导致了原来的数据没有读到, 新的数据又被覆盖了。

产生这个情况的原因是, **这两个线程之间没有通讯, `produce`线程生成数据, 没有通知`consume`线程去读取。**

### 2. 利用wait, notify实现生产者消费者模式_Version2

```java
public class ProduceConsumerVersion2 {
    private int i = 0;

    final private Object lock = new Object();

    // 表示判断生产者是否生产了数据. false 表示没有生产数据
    private volatile boolean isProduced = false;

    public void produce() {
        synchronized (lock) {

            // isProduced = true 表示 已经生成了, 那么生产者就需要 等待 消费者消费
            if (isProduced) {
                try {
                    lock.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            } else { // isProduced = false 表示 消费者已经消费了数据, 需要 唤醒 生产者生产数据, 并且生产完数据后, 需要把isProduced = true, 表示已经生产完了, 需要消费者继续去消费
                i++;
                System.out.println("P->" + i);

                lock.notify();
                isProduced = true;
            }
        }
    }

    public void consume() {
        synchronized (lock) {
            // isProduced = true, 表示生产者已经生产数据了, 所以 唤醒 消费者去消费数据, 并且 isProduced = false, 表示已经消费完了, 需要生产者继续生产
            if (isProduced) {
                System.out.println("C->" + i);
                lock.notify();
                isProduced = false;
            } else {// isProduced = false, 表示生产者没有生成数据, 所以让消费者 等待 生产者生产数据
                try {
                    lock.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    public static void main(String[] args) {
        ProduceConsumerVersion2 pc2 = new ProduceConsumerVersion2();
        new Thread(n) {
            @Override
            public void run() {
                while (true) {
                    pc2.produce();
                }
            }
        }.start();

        new Thread(n){
            @Override
            public void run() {
                while (true) {
                    pc2.consume();
                }
            }
        }.start();
    }
}
```

上面代码中, 用到wait(), notify()方法来实现生产者消费者。首先最重要的一点, 一定要了解清楚**wait(), notify()方法是Object提供的final方法, 无法被被修改和继承, 它们并不是Thread类的方法。**



