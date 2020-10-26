## Synchronized关键字

在之前的学习中, 不管是使用场景还是代码示例, 其实都存在一个很大的问题。那就是线程安全问题。

> 什么是线程安全问题?<br/>
> **在多线程环境下, 对共享变量进行修改操作, 就会产生线程安全问题。**

`Synchronized`关键字是解决线程安全问题的一个重要办法之一。使用`synchronized`关键字修饰某个方法或者代码块, 可以保证每次执行这段代码都只能有一个线程通过。

> 学习`synchronized`是非常重要的, 其中的原理涉及的很深。包括Java内存模型, c++, 汇编级别, 操作系统。想要正在理解`synchronized`还是需要花很长的时间

### 1. 同步代码块

```java
public class SynchronizedTest {
    private final static Object LOCK = new Object();

    public static void main(String[] args) {
        synchronized (LOCK) {
                try {
                    Thread.sleep(200_000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
        }
    }
}
```

对于锁的概念需要注意的是:**`synchronized(obj)`的意思对这个obj上锁, 也就是monitor。所以在多线程环境下, 只有获取到这个对象锁的钥匙的线程才能够执行里面的程序,而如何获取到钥匙, 这就需要多线程去不断的竞争。只有唯一的一个线程抢到锁, 才会去执行其中的代码。所以如果有两个线程同时到达synchronized(obj), 那么这两个线程是竞争关系, 只有当某个线程抢到锁才能执行自己的代码, 有可能Thread0获取到, 也有可能Thread1获取到。**

如果我们编译上面的那段代码, 切换到SynchronizedTest.class的同级目录, 然后用```javap -v  SynchronizedTest.class```查看字节码文件, 如下:

```shell
 public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=4, args_size=1
         0: ldc           #2                  // class com/yk/concurrency/base/chapter7/SynchronizedTest
         2: dup
         3: astore_1
         4: monitorenter
         5: ldc2_w        #3                  // long 200000l
         8: invokestatic  #5                  // Method java/lang/Thread.sleep:(J)V
        11: goto          19
        14: astore_2
        15: aload_2
        16: invokevirtual #7                  // Method java/lang/InterruptedException.printStackTrace:()V
        19: aload_1
        20: monitorexit
        21: goto          29
        24: astore_3
        25: aload_1
        26: monitorexit
        27: aload_3
        28: athrow
        29: return
```

我们发现在添加`synchronized`关键字之后, 独有的指令。执行同步代码块首先要执行`monitorenter`指令, 退出的时候`monitorexit`指令。也就是说, 使用`synchronized`,关键在于必须要获取对象的监视器`monitor`进行获取, 当线程获取`monitor`后才能往下执行, 否则只能等待。而这个获取过程是互斥的, 即同一时刻只有一个线程能够获取到`monitor`。

**要注意的是, 上面的字节码文件有两个`monitorexit`指令, 因为`synchronized`关键字会保证线程无论以哪种方式退出都会释放锁, 所有有两个`monitorexit`一个是正常执行退出释放锁, 另一个是发生异常退出释放锁。**

### 2. 同步方法

```java
public class SynchronizedThis {

    public static void main(String[] args) {
        ThisLock thisLock = new ThisLock();
        new Thread("T1") {
            @Override
            public void run() {
                thisLock.m1();
            }
        }.start();

        new Thread("T2") {
            @Override
            public void run() {
                thisLock.m2();
            }
        }.start();

    }
}

class ThisLock {
    public synchronized void m1() {
        try {
            System.out.println(Thread.currentThread().getName());
            Thread.sleep(10_000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public synchronized void m2() {
        try {
            System.out.println(Thread.currentThread().getName());
            Thread.sleep(10_000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

>上面的代码, 方法m1(), m2()都加了`synchronized`关键字, 那么`synchronized`锁的是什么对象?

其实测试的方法很简单, 当m1(),m2()都加`synchronized`时, 会发现两个线程调用m1,m2会有明显的抢锁时间差距, 但是如果只有m1()方法被`synchronized`关键字修饰, 那么几乎会同时调用m1,m2。这就说明了, 上面的代码, `synchronized`锁的是同一个对象。

同步方法, 没有指定对象时, 此时拿到锁的对象是 **this --> 当前对象的实例**。 但是这就导致了一个问题: 当多个线程去竞争这个this锁, 此时线程Thread0抢到了方法锁, 并且执行方法中的逻辑, 如果这个Thread0 直接处理完里面所有的逻辑, 并且释放锁, 此时Thread1拿到锁时, 发现任务已经被Thread0完成了, 就没必要执行了。**也就是说, 方法锁, 锁的对象时this, 有可能导致只有一个线程在工作, 其他线程在等待锁并且拿到的时候, 什么事都没做, 就结束了, 这样是不合理的。**


