## 线程基础

### 1. 为什么要用线程?

先看下面一段代码：

```java
public class TryConcurrency {

    /**
     * 在调用main函数时, 其实系统已经启动了一个主线程, 用来执行对应的程序
     *
     * JVM 在调用main函数时, 会创建一个main线程, 被JVM调用。 这是一个非守护线程(守护线程会在后面解释)
     *
     * 但是JVM不仅仅只有main线程, 还是很多守护线程, 例如：Finalizer垃圾回收线程, RMI线程等
     * @param args
     */
    public static void main(String[] args) {

        // 在只执行下面两个方法的情况下, 必须要先执行第一个方法, 在执行第二个方法, 效率很低
        Common.readFromDataBase();
        Common.writeDataToFile();
        try {
            Thread.sleep(1000 * 30L);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

上面代码可以发现, 当我们处理两块逻辑时, 一个是从数据库读取数据, 另一个是将读取的数据写入到文件中, 必须要按照顺序执行, 也就是串行化。如果某一阶段处理时间很长, 另一个方法只能等待, 这样效率很低。

**所以使用多线程的目的, 最主要的就是提升程序处理任务的效率。**

### 2. 什么是线程?

计算机的核心是CPU, 它承担了所有的计算任务。单个CPU一次只能运行一个任务。

**进程:** 可以理解为程序的执行过程, 应用程序一旦执行, 就是一个进程。每个进程都有自己的独立地址空间, 每启动一个进程, 系统就会为它分配地址空间, 建立数据表来维护代码段, 堆栈段和数据段。

**线程:** 程序执行的最小单位。

**进程与线程的区别?**

1. 同一个进程的所有线程共享本进程的地址空间, 但是不同的进程之间的地址空间是独立的。
2. 同一个进程的所有线程共享本进程的资源, 例如：内存, CPU, IO等。但是进程之间相互独立
3. 线程不能独立存在, 它必须依赖于进程。
4. 同一个进程的所有线程共享资源, 当一个线程发生崩溃时, 此进程也会发生崩溃。 但是各个进程之间的资源是独立的，因此当一个进程崩溃时，不会影响其他进程。因此进程比线程健壮。

**进程与线程的选择取决条件:**

1. 在程序中, 如果需要频繁的创建和销毁的使用线程, 因为进程创建和销毁开销很大(需要不停的分配资源), 但是线程的调用只是改变CPU的执行, 开销小;
2. 如果需要程序更加的稳定安全时，可以选择进程。如果追求速度，就选择线程。

**CPU就像一座工厂, 时刻在运行。进程就是工厂的车间, 一个车间里可以有很多工人, 他们协同完成一个任务。线程就相当于车间里的工人, 一个线程可以包括多个线程。**


### 3. 线程的创建

创建线程常见的有两种方式:
- 继承Thread类
- 实现runnable接口

其实Thread类也是实现了runnable接口, 实现了run方法, 只不过Thread类内部自己封装了一些方法。

我建议在并发编程中, 还是实现`runnable`接口创建线程, 因为Java可以实现多个接口, 但是只能继承一个父类。这样实现接口, 对自己的定义类扩展性更高。

我们对上面的Java代码通过多线程进行重构:

```java
public class CreateThread1 {

    /**
     * 在调用main函数时, 其实系统已经启动了一个主线程, 用来执行对应的程序
     *
     * JVM 在调用main函数时, 会创建一个main线程, 被JVM调用。 这是一个非守护线程(守护线程会在后面解释)
     *
     * 但是JVM不仅仅只有main线程, 还是很多守护线程, 例如：Finalizer垃圾回收线程, RMI线程等
     * @param args
     */
    public static void main(String[] args) {
        execute();
    }

    private static void execute() {
        /**
         * 通过这个创建线程的方式执行代码
         */
        new Thread("read-data-thread"){
            @Override
            public void run() {
                Common.readFromDataBase();
            }
        }.start();

        new Thread("write-file-thread"){
            @Override
            public void run() {
                Common.writeDataToFile();
            }
        }.start();
    }
}
```

上面的代码通过创建两个线程, 分别处理读取数据, 写入文件的任务, 这是异步执行的, 在读取数据的同时也可以将读出的数据写入文件。大大提高了任务的效率。

这里其实有很多的问题值得去考虑:

1. **线程启动的时机是什么?**

> 注意, `new Thread()`只是创建了一个java的实例而已, 并不代表一个线程, 必须要调用`start()`方法才会启动线程。<br/>
> 而且这个`start()`方法是不能执行两次的, 通过分析Thread的start()源码: <br/>
> `start()`会调用 一个原生`native`方法`start0()`, 这是一个c++的方法, 在这个`start0()`方法中会调用`run()`方法。<br/>
> 这里的`run()`方法其实是用了模板方法的方式. 可以重写`run()`方法, 让这个`start0()`方法去执行被覆盖的`run()`方法。

可能这里经常会看到面试题, 就是线程启动的方法是`run()`, 还是`start()`。这里给出了答案, `start()`方法是启动线程的方法, `run()`方法中是编写我们业务处理任务的代码。

2. **线程的生命周期?(这里是重点)**

线程的生命周期从简单来说分为: `new`, `runnable`, `running`, `block`, `terminate`。

> 1. `new Thread()`创建线程。 不管是后面的线程池, 还是`join()`方法, 这一定是基本的创建线程的第一步。<br/>
> 2. 调用`start()`方法。表示当前线程启动了, 处于`runnable`, 即**可执行状态**。 但是要注意,这里线程只有具备执行的能力, 并不意味着立即开始执行<br/>
> 3. 当线程被CPU调度(dispatch)时, 此时线程就处于running状态, 即执行状态。但是当CPU调度到其他线程时, `running`状态可能还会变成`runnable`。
> 4. 如果在`running`过程中, 该线程被`blocked`, 例如: `sleep`, 那么此线程处于阻塞状态。如果被`blocked`, 无法立即回到`running`状态, 必须要先到`runnable`状态
> 5. 在线程在`running`状态正常结束, 或者在`blocked`状态中被打断, 例如: 在`runnable`状态中由于某些原因导致线程死亡, 就会进入终结terminated状态.

### 4. 守护线程

在说守护线程之前, 先看一个需求: 

**在建立网络连接时, Client A <-----------> Server B, A与B进行长连接, 在长连接中, A需要不断的向B发送心跳包, 来确认服务是否还正常。也就是说在连接创建好之后, 还需要维护心跳, 而这个心跳跟业务是没有关系的， 跟连接发送报文信息也没有关系, 心跳只是一个维护作用。<font color="red">如果长连接的线程断开了, 那么这个发检测心跳的线程就不需要存在了。这时候就需要设置守护线程</font>**

> 创建连接之后, 在开启了`DaemonThread`守护线程, 用于health check。如果不设置守护线程,那么及时建立连接操作的线程(可能是主线程)已经死了, 但是这个守护线程依然可以存在进行心跳检测, 这样就会造成内存的损耗(因为连接都不存在了, 还要心跳干嘛)。

**<font color="red">所以守护线程, 可以理解为守护主线程的一个线程, 一旦主线程结束了, 不管这个守护线程是不是active, 都直接结束</font>**

```java
public class DaemonThread {

    public static void main(String[] args) {
        Thread t = new Thread(() -> {
            // 在Thread里面再创建一个线程
            Thread innerThread = new Thread(() -> {
                try {
                    while (true) {
                        System.out.println("Do some thing for health check");
                        Thread.sleep(1_000);
                    }
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            });


            innerThread.setDaemon(true); // 如果Daemon设置为true, 就表示, 这个线程时守护线程, 而且必须要在start之前调用。
            innerThread.start();

            try {
                Thread.sleep(1_000);
                System.out.println("T Thread finish done.");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
//        t.setDaemon(true);
        t.start();
    }
}
```

从上面的代码可以看到, 我们在一个线程Thread内部有创建了一个线程innerThread, 并设置为守护线程`innerThread.setDaemon(true)`。只要外部线程挂了, 那么这个内部线程也会结束。