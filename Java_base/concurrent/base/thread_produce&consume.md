## 经典面试问题————实现生产者消费者需求

对于生产者消费者问题, 不同的语言, 都有不同的实现方式, 不同的算法。在java中对于生产者消费者模式, 基本上就是用线程来实现的。这也不难理解, 一个线程用于生产数据, 另一个线程用户消费数据。

### 1.生产者消费者模式_Version1

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

那么如何实现线程间的通讯? 那就是使用Object的方法, `wait()`, `notify()`, `npotifyAll()`。

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

但是这个版本中依然是存在问题的, 在一个生产者, 一个消费者的环境下, 上面的代码是没有问题的。
如果是多个生成者多个消费者的环境中就会产生问题。因为`wait()`,`notify()`并不会指定某一个对应的线程去操作, 这就导致所有的线程都处在wait()状态, 也不会产生死锁, 但是程序也不会运行。

### 3. 生产者消费者模式_Version3

```java
public class ProduceConsumerVersion3 {
    private int i = 0;

    final private Object lock = new Object();

    // 表示判断生产者是否生产了数据. false 表示没有生产数据
    private volatile boolean isProduced = false;

    public void produce() {
        synchronized (lock) {

            // isProduced = true 表示 已经生成了, 那么生产者就需要 等待 消费者消费。
            // 这里使用 while 表示 所有的生产者 只要有一个生产了数据, 所有的生产者都会进入这个循环 并且处于等待状态
            while (isProduced) {
                try {
                    /**
                     * 这里一定要注意的是, wait(), notify()是 Object提供的方法
                     */
                    lock.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            i++;
            System.out.println("P->" + i);

            // notifyAll() 唤醒所有等待锁的线程, 然后再次同时竞争抢锁
            lock.notifyAll();
            isProduced = true;
        }
    }

    public void consume() {
        synchronized (lock) {
            // isProduced = true, 表示生产者已经生产数据了, 所以 唤醒 消费者去消费数据, 并且 isProduced = false, 表示已经消费完了, 需要生产者继续生产
            while (!isProduced) {
                try {
                    lock.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            System.out.println("C->" + i);
            lock.notifyAll();
            isProduced = false;
        }
    }

    public static void main(String[] args) {
        ProduceConsumerVersion3 pc3 = new ProduceConsumerVersion3();
        Stream.of("P1", "P2").forEach(n -> {
            new Thread(n) {
                @Override
                public void run() {
                    while (true) {
                        pc3.produce();
                    }
                }
            }.start();
        });

        Stream.of("C1", "C2").forEach(n -> {
            new Thread(n) {
                @Override
                public void run() {
                    while (true) {
                        pc3.consume();
                    }
                }
            }.start();
        });
    }
}
```


上面代码中, 用到wait(), notifyAll()方法来实现生产者消费者。首先最重要的几点, 一定要了解清楚
##### 1. **`wait()`, `notify()`, `notifyAll()`方法是Object提供的final方法, 无法被被修改和继承, 它们并不是Thread类的方法。**

##### 2. **`wait()`,`notify()`,`notifyAll()`的方法必须要在`synchronized`修饰的代码块或者方法内使用, 也就是说当前线程一定要获取锁。**
- `wait()`会释放当前线程的对象锁, 既然要释放锁, 那就必须先获取锁, 而`synchronized`就是同步锁, 线程能执行其中的代码, 就必须要获取锁。

    > 对于`wait()`方法为什么要释放锁, 其实也不难理解。从代码中可以看出, 不管是生产者线程, 还是消费者线程, 拿到的都是同一个Object锁。所以无论是生产者还是消费者线程进入代码块, 一定会获取到Object锁, 那么另一个线程就无法进入代码块执行, 直到释放了Object锁, 才会继续竞争锁资源。

- `notify()`与`notifyAll()`使用唤醒被wait阻塞的线程, 并且要释放持有的锁给wait所在的线程。那么还是一样的道理, 既然要释放锁, 那就必须先获取锁, 所以notify()也必须在`synchronized`修饰的代码块或者方法内。

##### 3. **线程在调用`wait()`,`notify()`,`notifyAll()`,`synchronized`的变化**

> 这其实是一个对象内部所的调度问题, Java中对象锁的模型。
> JVM会为`synchronized(obj)`的obj对象维护两个集合, `Entry Set`, `Wait Set`。<br/>
> 对于`Entry Set`: 如果线程A已经持有对象锁, 此时如果有其他线程也想获得该对象锁的话, 它只能进入`Entry Set`, 并且处于线程的blocked状态<br/>
> 对于`Wait Set`: 如果线程A调用了wait方法, 那么线程A会释放该对象的锁, 进入到`Wait Set`, 并且处于线程的waiting状态。<br/>

所以

- 对于在`Entry Set`中的线程, 当对象锁被释放时, JVM会唤醒处于`Entry Set`中的某一个线程, 那么这个线程的状态就从blocked转变为runnable。

- 对于`Wait Set`中的线程, 当对象的notify()方法被调用时, JVM会唤醒处于`Wait Set`某一个线程, 这个线程的状态就从waiting转变为blocked; 如果调用`nofityAll()`方法, `Wait Set`中的全部线程都会转变为blocked状态。所有`Wait Set`中被唤醒的线程都会转移到`Entry Set`中。**但是要注意的是, 被唤醒的线程虽然需要重新抢锁, 但是不会再执行之前的代码, 而是从等待的地方开始执行**

##### 4. 生成者消费者最容易出现错误的问题

既然知道了`wait()`, `notify()`, `notifyAll()`, `synchronized`之间的关系, 对于version3的代码需要注意两个问题:

1. **<font color="red">在进行判断是否生产数据的时候, 用的是while循环, 而不是if判断</font>**

**因为`wait()`的线程永远不能确定其他线程会在什么状态下`notify()`, 所以必须在被唤醒, 抢占锁并且从`wait()`方法退出的时候再次进行指定条件判断。**

> 对于上面这段话, 只要把整个线程执行的过程写出来基本上就清楚了。如果不用while 而用if, 按照下面的执行顺序, 就会产生问题

| 线程 | 判断标识 | 状态 | 结果 |
| -- | -- | -- | -- |
| 消费者C1抢到锁 | isProduced=false|进入if判断, C1 waiting|等待消费|
| 消费者C2抢到锁 | isProduced=false|进入if判断, C2 waiting|等待消费|
| 生产者P1抢到锁 | isProduced=false|跳过if判断, 生产数据, notifyAll() C1,C2唤醒, isProduced=true | 生产成功|
| 消费者C1抢到锁 | isProduced=true |跳出if判断, 消费数据成功,  isProduced=false| 消费成功 |
| 消费者C2抢到锁 | isProduced=false |跳出if判断, 但此时数据已经被C1消费了, 无法继续消费| 消费失败 |

从上面的流程可以看到, 在多生产者, 多消费者的情况下, 极有可能出现唤醒生产者的是另一个生产者, 或者唤醒消费者的是另一个消费者, 这种情况下, 如果不是while, 就直接跳出了判断, 导致生产过度, 或者消费过度的情况。**所以, 生产者消费者模式永远要把wait()放到循环语句中**

2. **<font color="red">唤醒线程必须要使用`notifyAll()`, 而不是`notify()`。</font>**

> 这个问题大家可能会有一个疑问, notify()是唤醒一个线程, notifyAll()是唤醒全部线程, 但是唤醒之后, 不管是notify(), 还是notifyAll(), 最终拿到锁的只会有一个线程, 那它们之间有什么区别呢?

如果用`notify()`, 而不用`notiffyAll()`的话, 我们还是用一个执行顺序, 来说明会出现的问题:

| 线程 | 判断标识 | 状态 | 结果 |
| -- | -- | -- | -- |
| 消费者C1抢到锁 | isProduced=false|进入while循环, C1 waiting|等待消费|
| 消费者C2抢到锁 | isProduced=false|进入while循环, C2 waiting|等待消费|
| 生产者P1抢到锁 | isProduced=false|跳过while循环, 生产数据, notify() C1唤醒, isProduced=true | 生产成功|
| 生产者P2抢到锁 | isProduced=true|进入while循环, P2 waiting | 生产失败|
| 消费者C1抢到锁 | isProduced=true |跳出while循环, 消费数据成功,  notify() C2唤醒,  isProduced=false| 消费成功 |
| 消费者C2抢到锁 | isProduced=false |进入while循环, C2 waiting| 消费失败 |

根据上面的执行顺序会发现, 最终的结果是生产者P2, 消费者P2都处于`Wait Set`中互相等待, 万一生产者P1退出了, 那就导致了没有生产者生产数据, 只剩下P2, C2一直等待, 出现死锁情况。

所以总结一句话, **之所以尽量使用`nofityAll()`的原因是, `nofity()`非常容器导致死锁。**当然notifyAll并不一定都是优点，毕竟一次性将Wait Set中的线程都唤醒是一笔不菲的开销，如果你能handle你的线程调度，那么使用`notify()`也是有好处的。











