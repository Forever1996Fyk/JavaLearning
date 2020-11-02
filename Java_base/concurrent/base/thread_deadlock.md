## DeadLock ———— 死锁

> 对于死锁的原理并不难理解, 但是在实际开发中死锁的产生会有点莫名其妙的。可能你认为这段代码没有问题, 但是在实际运行中却有可能产生死锁, 所以在代码编写的过程中要时刻注意是否会产生死锁。

### 1. 死锁的产生

先看下面一段代码:

```java
public class DeadLock {

    private OtherService otherService;

    public DeadLock(OtherService otherService) {
        this.otherService = otherService;
    }

    private final Object lock = new Object();

    public void m1() {
        synchronized (lock) {
            System.out.println("m1");
            otherService.s1();
        }
    }

    public void m2() {
        synchronized (lock) {
            System.out.println("m2");
        }
    }
}
```

```java
public class OtherService {

    private final Object lock = new Object();

    private DeadLock deadLock;

    public void s1() {
        synchronized (lock) {
            System.out.println("s1 =========");
        }
    }

    public void s2() {
        synchronized (lock) {
            System.out.println("s2 =========");
            deadLock.m2();
        }
    }

    public void setDeadLock(DeadLock deadLock) {
        this.deadLock = deadLock;
    }
}
```

```java
public class DeadLockTest {
    public static void main(String[] args) {
        OtherService otherService = new OtherService();
        DeadLock deadLock = new DeadLock(otherService);
        otherService.setDeadLock(deadLock);

        new Thread() {
            @Override
            public void run() {
                while (true) {
                    deadLock.m1();
                }
            }
        }.start();

        new Thread() {
            @Override
            public void run() {
                while (true) {
                    otherService.s2();
                }
            }
        }.start();
    }
}
```
当运行上面的代码一段时间之后, 就会发现整个程序直接卡死了, 不会在输出了。这就是因为产生了死锁。通过上面的代码可以很明显的看出死锁产生的原因:

**当线程Thread0调用deadLock.m1()时, 它需要获取obj1的锁, 而m1()方法的内部又调用了otherService.s1()。在otherService.s1()的内部需要获取obj2的锁。当线程Thread1调用deadLock.s2()方法内部又需要调用deadLock.m2()的方法, m2()方法又需要获取obj1的锁, 这样产生了一个获取锁的循环, 如果其中有一个方法一直没有释放锁的话, 就会导致这个程序一直处于等待获取锁的状态, 也就是程序卡死了。**

死锁产生的四个必要条件:

1. **互斥条件**: 每个资源都只能有一个线程获取, 其他线程必须处于阻塞状态
2. **请求并持有**: 线程1持有资源A时, 并且还要去请求资源B, 而资源B被线程2占有, 导致线程1占有资源A不释放
3. **不可强占**: 线程1持有资源A, 那么就必须线程1释放资源A, 不能被其他线程释放
4. **循环等待条件**: 多个线程之间形成一种环形等待资源的关系。{T0,T1,T2...Tn,T0}, 每一个线程都在等待前一个线程释放资源

### 2. 死锁的检测

那么在生产环境中,如何发现死锁呢?

1. **Jstack命令**:  jstack是java虚拟机自带的堆栈跟踪工具。Jstack工具可以用于生成java虚拟机当前时刻的线程快照。线程快照是当前java虚拟机内每一条线程正在执行的方法堆栈的集合，生成线程快照的主要目的是定位线程出现长时间停顿的原因，如线程间死锁、死循环、请求外部资源导致的长时间等待等。 线程出现停顿的时候通过jstack来查看各个线程的调用堆栈，就可以知道没有响应的线程到底在后台做什么事情，或者等待什么资源。

    先利用jqs命令,找出java程序的进程id, 在用jstack [ID] 查看堆栈的信息(: 具体如何查看jstack打印的信息, 会在JVM学习中说明)。如下示例:

    ```shell
    E:\workspaceIDEAPro\ThreadLearning\target\classes\com\yk\concurrency\base\chapter7>jps
    6896 Jps
    10084
    11204 ZbExamTeacherApplication
    11572 DeadLockTest
    13236 Launcher
    18168 Bootstrap
    9592 RemoteMavenServer
    13340 Launcher
    16540 Launcher

    E:\workspaceIDEAPro\ThreadLearning\target\classes\com\yk\concurrency\base\chapter7>jstack 11572
    2020-10-26 15:27:02
    Full thread dump Java HotSpot(TM) 64-Bit Server VM (25.31-b07 mixed mode):

    "DestroyJavaVM" #13 prio=5 os_prio=0 tid=0x0000000004d1e800 nid=0x2d5c waiting on condition [0x0000000000000000]
    java.lang.Thread.State: RUNNABLE

    "Thread-1" #12 prio=5 os_prio=0 tid=0x000000001fb7b800 nid=0x493c waiting for monitor entry [0x000000002077f000]
    java.lang.Thread.State: BLOCKED (on object monitor)
            at com.yk.concurrency.base.chapter8.DeadLock.m2(DeadLock.java:28)
            - waiting to lock <0x000000076b605fe8> (a java.lang.Object)
            at com.yk.concurrency.base.chapter8.OtherService.s2(OtherService.java:24)
            - locked <0x000000076b602de8> (a java.lang.Object)
            at com.yk.concurrency.base.chapter8.DeadLockTest$2.run(DeadLockTest.java:39)

    "Thread-0" #11 prio=5 os_prio=0 tid=0x000000001fb77800 nid=0x33e0 waiting for monitor entry [0x000000002067f000]
    java.lang.Thread.State: BLOCKED (on object monitor)
            at com.yk.concurrency.base.chapter8.OtherService.s1(OtherService.java:17)
            - waiting to lock <0x000000076b602de8> (a java.lang.Object)
            at com.yk.concurrency.base.chapter8.DeadLock.m1(DeadLock.java:22)
            - locked <0x000000076b605fe8> (a java.lang.Object)
            at com.yk.concurrency.base.chapter8.DeadLockTest$1.run(DeadLockTest.java:30)

    "Service Thread" #10 daemon prio=9 os_prio=0 tid=0x000000001fafd800 nid=0x3d38 runnable [0x0000000000000000]
    java.lang.Thread.State: RUNNABLE

    "C1 CompilerThread2" #9 daemon prio=9 os_prio=2 tid=0x000000001faa8800 nid=0x2da0 waiting on condition [0x0000000000000000]
    java.lang.Thread.State: RUNNABLE

    "C2 CompilerThread1" #8 daemon prio=9 os_prio=2 tid=0x000000001fa43000 nid=0x42b4 waiting on condition [0x0000000000000000]
    java.lang.Thread.State: RUNNABLE

    "C2 CompilerThread0" #7 daemon prio=9 os_prio=2 tid=0x000000001fa40800 nid=0x3d44 waiting on condition [0x0000000000000000]
    java.lang.Thread.State: RUNNABLE

    "Monitor Ctrl-Break" #6 daemon prio=5 os_prio=0 tid=0x000000001e56e800 nid=0x1c18 runnable [0x000000002007e000]
    java.lang.Thread.State: RUNNABLE
            at java.net.SocketInputStream.socketRead0(Native Method)
            at java.net.SocketInputStream.read(SocketInputStream.java:150)
            at java.net.SocketInputStream.read(SocketInputStream.java:121)
            at sun.nio.cs.StreamDecoder.readBytes(StreamDecoder.java:284)
            at sun.nio.cs.StreamDecoder.implRead(StreamDecoder.java:326)
            at sun.nio.cs.StreamDecoder.read(StreamDecoder.java:178)
            - locked <0x000000076b717180> (a java.io.InputStreamReader)
            at java.io.InputStreamReader.read(InputStreamReader.java:184)
            at java.io.BufferedReader.fill(BufferedReader.java:161)
            at java.io.BufferedReader.readLine(BufferedReader.java:324)
            - locked <0x000000076b717180> (a java.io.InputStreamReader)
            at java.io.BufferedReader.readLine(BufferedReader.java:389)
            at com.intellij.rt.execution.application.AppMainV2$1.run(AppMainV2.java:64)

    "Attach Listener" #5 daemon prio=5 os_prio=2 tid=0x000000001f9c3800 nid=0xa20 waiting on condition [0x0000000000000000]
    java.lang.Thread.State: RUNNABLE

    "Signal Dispatcher" #4 daemon prio=9 os_prio=2 tid=0x000000001e52e800 nid=0x3a3c runnable [0x0000000000000000]
    java.lang.Thread.State: RUNNABLE

    "Finalizer" #3 daemon prio=8 os_prio=1 tid=0x0000000004f18800 nid=0x4b2c in Object.wait() [0x000000001f87f000]
    java.lang.Thread.State: WAITING (on object monitor)
            at java.lang.Object.wait(Native Method)
            - waiting on <0x000000076b3062f8> (a java.lang.ref.ReferenceQueue$Lock)
            at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:142)
            - locked <0x000000076b3062f8> (a java.lang.ref.ReferenceQueue$Lock)
            at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:158)
            at java.lang.ref.Finalizer$FinalizerThread.run(Finalizer.java:209)

    "Reference Handler" #2 daemon prio=10 os_prio=2 tid=0x0000000004f10000 nid=0x41d4 in Object.wait() [0x000000001f77f000]
    java.lang.Thread.State: WAITING (on object monitor)
            at java.lang.Object.wait(Native Method)
            - waiting on <0x000000076b305d68> (a java.lang.ref.Reference$Lock)
            at java.lang.Object.wait(Object.java:502)
            at java.lang.ref.Reference$ReferenceHandler.run(Reference.java:157)
            - locked <0x000000076b305d68> (a java.lang.ref.Reference$Lock)

    ```
2. **JConsole工具**: jconsole是JDK自带的监控工具，在JDK/bin目录下可以找到。它用于连接正在运行的本地或者远程的JVM，对运行在Java应用程序的资源消耗和性能进行监控，并画出大量的图表，提供强大的可视化界面。而且本身占用的服务器内存很小，甚至可以说几乎不消耗。

    使用方式很简单, 如下示例:
    ![jconsole_1](/image/jconsole_1.png)
    ![jconsole_2](/image/jconsole_2.png)

### 3. 避免死锁

如何避免死锁的产生,其实是在与程序员的编程能力。

- 在开发过程中, 对需要用到线程指定一个唯一的名称, 这样在出现问题时, 更容易定位问题。
- 在开发过程中, 应该尽量避免`synchronized`关键字的嵌套, 否则很有可能发生死锁。
- 银行家算法(暂未了解)
- 破坏死锁产生的条件

    > 上面我们说了, 死锁产生的条件, 其中只有**请求并持有条件**, **循环等待条件**可以被破坏, 其他两个无法被修改。我们可以保证获取资源的有序性来破坏条件。<br/>
    > 如何保证获取资源的有序性? 其实并不难, 只要保证多个线程在获取资源的顺序上保持一致即可。也就是说, 线程1, 线程2都同时去获取资源A, 再同时去获取资源B。如下代码:

    ```java
    public class NotDeadLock {
        private final Object A = new Object();
        private final Object B = new Object();

        public static void T1() {
            synchronized(A) {
                ....

                synchronized(B) {

                }
            }
        }

        public static void T2() {
            synchronized(A) {
                ....

                synchronized(B) {
                    
                }
            }
        }
    }
    ```




