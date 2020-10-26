## 自定义线程执行与强制关闭

之前我们知道线程通过start()方法启动之后, 如果在执行过程中出现阻塞, 导致线程无法停止, 这样就会造成资源的浪费。所以如何优雅的关闭线程, 也是线程中的一大问题。

> 所以我们根据不同的场景分析不同的关闭线程的方式!

### 1. 利用判断标识true&false, 是否执行代码

例如程序中你需要批量导入数据, 但是导入的时间比较长, 所以使用多线程去处理。创建多个线程分别去获取数据, 但是如果在线程执行过程中出现问题, 可能线程并没有停止, 而是阻塞。那这样就会造成资源的浪费。

```java
public class ThreadCloseGraceful {

    private static class Worker extends Thread {
        private volatile boolean start = true;
        @Override
        public void run() {
            while (start) {
                //to do something
            }
        }

        public void shutdown() {
            this.start = false;
        }
    }

    public static void main(String[] args) {
        Worker worker = new Worker();
        worker.start();

        try {
            Thread.sleep(10000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        worker.shutdown();
    }
}
```

### 2. 强制关闭线程

例如进行网络连接的操作, 你可能期望是3s之内连接成功, 但是可能经过了30s也没成功, 导致这个线程阻塞了。如果在操作中block了, 这样就无法用interrupt中断, 也没有机会去改变flag。 下面的例子就很好的说明了一点: interrupt()方法只是改变中断状态，不会中断一个正在运行的线程。在Thread类中提供了一个stop()方法, 可以强制关闭线程, 但是这个方法已经被废弃了, 因为存在安全问题。所以如果需要强制的关闭线程, 需要实现自己的方法。

```java
public class ThreadCloseForce {
    private static class Worker extends Thread {
        private boolean flag = true;
        @Override
        public void run() {
            while (flag) {

            }

            // to do something
        }
    }

    public static void main(String[] args) {
        Worker worker = new Worker();
        worker.start();

        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        worker.interrupt();
    }
}
```

### 3. 自定义线程执行与强制关闭

> 虽然我们在实际开发中可能并不会直接用到原生的线程开发, 大部分情况下都是线程池, 它提供了大量的api供开发者使用。 但是自己实现一些原生线程的相关操作, 能够更好的帮助我们理解线程的原理, 也能在面试阶段能够更好的与面试官交流!

实现思路:

**可以定义一个runner线程, 把这个runner线程设为执行线程的一个守护线程。如果执行线程中断, 生命周期结束, 那么守护线程也就结束了。**

```java
public class ThreadService {

    private Thread executeThread;

    private boolean finished = false;

    /**
     *
     * 整体的代码逻辑如下:
     *
     * 1. 我们定义一个全局的执行线程 executeThread 和一个结束标识 finished = false。在execute(Runnable task)方法中, 传入Runnable。
     *
     * 2. 实例化executeThread线程, 并实现run()方法, 在run方法中, 定义一个runner线程, 传入task任务接口。并且将这个线程设置为executeThread的守护线程。
     * (这样一旦这个executeThread关闭了, 那这个守护线程也就关闭了, 也就是这个task任务也就关闭了)
     *
     * 3. 然后启动runner(注意启动完runner，一定要join()), 并且将 finished设为true。
     *
     * 4. 在启动executeThread线程。
     *
     * 5. 创建shutdown(long mills)方法, 该方法是判断任务执行时间是否超过定义的时间, 如果超过就interrupt executeThread线程, 如此守护线程 task 也就关闭了。
     *
     * @param task
     */
    public void execute(Runnable task) {
        executeThread = new Thread() {
            @Override
            public void run() {
                Thread runner = new Thread(task);
                runner.setDaemon(true);

                runner.start();
                try {
                    // 必须要join, 否则这个守护线程, 可能在main线程执行完之后, 还没有启动。这样就没有意义了
                    runner.join();
                    finished = true;
                } catch (InterruptedException e) {
                    //e.printStackTrace();
                }
            }
        };

        executeThread.start();
    }

    /**
     *  这里要注意程序执行的逻辑。
     *
     *  当main线程调用上面的execute方法时, 直到执行到runner.join()方法时, task线程才会启动。当task线程启动后, main线程才开始执行后面的代码(因为用了join())。
     *
     *  也就是说当执行到runner.join()方法时, Task线程开始运行了, 但是此时还没有把finished设为true, 就直接执行main线程的shutdown()方法了。
     *
     *  此时, finished = false, main线程就会进入while循环, 判断这个时间是否超过指定的时间。
     *
     *  如果超过了时间就中断executeThread, 如果在指定的时间内, task线程执行完了, 那么此时就会执行finished = true, 这样就会跳出while循环。
     *
     * @param mills
     */
    public void shutdown(long mills) {
        long currentTime = System.currentTimeMillis();
        while (!finished) {
            if ((System.currentTimeMillis() - currentTime) >= mills) {
                System.out.println("任务超时, 需要结束!");
                executeThread.interrupt();
                break;
            }

            try {
                executeThread.sleep(1);
            } catch (InterruptedException e) {
                System.out.println("执行线程被打断!");
                break;
            }
        }

        finished = false;
    }
}
```

上面代码有一个细节要注意的是, 在shutdown()时判断是否超时。如果传入中断的时间可能10000ms, 但是执行任务的时间可能只有5000ms, 那么守护线程也不会等待10000ms才结束, 也会直接结束。

```java
public class ThreadCloseForce1 {

    public static void main(String[] args) {
        ThreadService service = new ThreadService();
        long start = System.currentTimeMillis();
        service.execute(() -> {
            // load a very heavy resource.
            while (true) {

            }

            // 如果执行线程任务, 只用了5000ms就结束了, 那么守护线程也不会等待10000ms才会结束, 也会直接结束。
            try {
                Thread.sleep(5000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        service.shutdown(10000);
        long end = System.currentTimeMillis();
        System.out.println(end - start);
    }
}
```

