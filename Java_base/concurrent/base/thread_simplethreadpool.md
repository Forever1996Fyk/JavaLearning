## 自定义简单线程池

> 我们利用之前学习的知识, `wait()`, `notify()`, `synchronized`来实现一个简易线程池

对于线程池的实现, 最主要的是实现的思路, 其次才是代码。所以实现线程池前必须要了解线程池的基本参数:

1. 核心线程数————线程池初始化时, 需要加载的线程数量
2. 任务队列————提交任务时, 先将任务存入到任务队列
3. 任务队列数————存储任务的最大数量
4. 最大线程数————当可用的核心线程数为0, 任务队列也满的时候, 需要把核心线程数扩容到最大线程数
5. 拒绝策略————当任务达到最大线程数时, 在提交任务就会执行相应的拒绝策略

构建线程池的步骤:

##### 1. 线程池初始化

线程池在初始化时, 就需要加载数量为核心线程数的线程队列, 这样使用线程池, 就需要再创建和销毁线程.

```java

// 线程池中总线程数
private int size;

// 任务队列数
private int queueSize;

// 最少线程数
private int min;

// 最大线程数
private int max;

// 活动线程数
private int active;

    //任务队列默认大小
private final static int DEFAULT_TASK_QUEUE_SIZE = 2000;

private static volatile int seq = 0;

private final static String THREAD_PREFIX = "SIMPLE_THREAD_POOL-";

private final static ThreadGroup GROUP = new ThreadGroup("Pool_Group");

// 任务队列
private final static LinkedList<Runnable> TASK_QUEUE = new LinkedList<>();

// 线程队列
private final static List<WorkerTask> THREAD_QUEUE = new ArrayList<>();

// 判断线程池是否被销毁
private volatile boolean destroy = false;


/**
    * 在new SimpleThreadPool时会加载init()方法, 创建size数量的线程
    */
public SimpleThreadPool(int min, int active, int max, int queueSize, DiscardPolicy discardPolicy) {
    this.min = min;
    this.active = active;
    this.max = max;
    this.queueSize = queueSize;
    this.discardPolicy = discardPolicy;
    init();
}

private void init() {
    // 先判断线程数min是否足够
    for (int i = 0; i < min; i++) {
        createWorkTask();
    }
    this.size = min;
    this.start();
}

private void createWorkTask() {
    WorkerTask task = new WorkerTask(GROUP, THREAD_PREFIX + (seq++));
    task.start();
    THREAD_QUEUE.add(task);
}

/**
    * 线程状态
    */
private enum TaskState {
    FREE, RUNNING, BLOCKED, DEAD
}

private static class WorkerTask extends Thread {
    private volatile TaskState taskState = TaskState.FREE;

    public WorkerTask(ThreadGroup threadGroup, String name) {
        super(threadGroup, name);
    }

    public TaskState getTaskState() {
        return this.taskState;
    }

    @Override
    public void run() {
        OUTER:
        while (this.taskState != TaskState.DEAD) {
            Runnable runnable;
            synchronized (TASK_QUEUE) {
                while (TASK_QUEUE.isEmpty()) {
                    try {
                        taskState = TaskState.BLOCKED;
                        TASK_QUEUE.wait();
                    } catch (InterruptedException e) {
                        System.out.println("Close.");
                        break OUTER;
                    }
                }
                //队列先进先出, 这里remove first
                runnable = TASK_QUEUE.removeFirst();
            }

            if (runnable != null) {
                taskState = TaskState.RUNNING;
                runnable.run();
                taskState = TaskState.FREE;
            }
        }
    }

    public void close() {
        this.taskState = TaskState.DEAD;
    }
}
```

上面的代码需要注意的是, 当createWorkTask时, 如果任务队列TASK_QUEUE为空时, 创建的所有线程都会wait, 当有任务提交时, 任务队列不为空, 才会唤醒线程, 继续去执行相应的任务, 所以当提交任务时, 需要往任务队列中添加任务, 并且notifyAll()。

##### 2. submit提交任务方法

```java
public void submit(Runnable runnable) {
    if (destroy) {
        throw new IllegalStateException("这个线程池已经销毁了, 不允许提交任务");
    }
    synchronized (TASK_QUEUE) {
        TASK_QUEUE.addLast(runnable);
        TASK_QUEUE.notifyAll();
    }
}
```

##### 3. 线程池shutdown

关闭线程池有几点需要注意: 

(1) 如果任务队列不为空, 那就说明有任务还在等待执行, 此时是不能直接关闭线程池, 否则任务没做完就关闭了, 导致错误。

(2) 当线程队列中线程的状态不是blocked, 也不会关闭。

```java
public void shutdown() throws InterruptedException {
    while (!TASK_QUEUE.isEmpty()) {
        Thread.sleep(50);
    }
    synchronized (THREAD_QUEUE) {
        int initVal = THREAD_QUEUE.size();
        while (initVal > 0) {
            for (WorkerTask task : THREAD_QUEUE) {
                if (task.getTaskState() == TaskState.BLOCKED) {
                    task.interrupt();
                    task.close();
                    initVal--;
                } else {
                    Thread.sleep(10);
                }
            }
        }
    }
    this.destroy = true;
    System.out.println("线程池已经关闭了");
}
```

##### 4. 线程池中也需要创建线程, 来维护线程池中的扩充因子, 例如最大最小线程数

根据上面线程池参数的说明, 有三个因素决定线程池是否执行任务, 核心线程数, 任务队列大小, 最大线程数。

- 所以需要在线程池内部开启一个线程, 不断的监控线程池中的线程是否已达到扩充要求, 也就是根据size判断, 循环创建线程, createWorkTask; 

- 而当任务队列为空时, 则需要降低线程池中的线程数, 循环移除线程队列中的线程, 也就是遍历THREAD_QUEUE, 每关闭,中断,移除一个线程,直到线程数为active或者min

```java
/**
    * 线程池的线程: 用于维护控制线程池中的扩充因子, 例如最大最小线程数等。
    */
@Override
public void run() {
    while (!destroy) {
        System.out.printf("Pool#Min:%d, Active:%d, Max:%d, Current:%d,QueueSize:%d\n",
                this.min, this.active, this.max, this.size, TASK_QUEUE.size());
        try {
            Thread.sleep(5_000L);
            // 如果任务队列的任务数已经大于活跃的线程数, 并且线程池大小小于活跃的线程数, 那么就扩充线程池的大小。
            if (TASK_QUEUE.size() > active && size < active) {
                for (int i = size; i < active; i++) {
                    createWorkTask();
                }
                System.out.println("The Pool incremented to active");
                size = active;
            } else if (TASK_QUEUE.size() > max && size < max) {// 如果任务队列的任务数已经大于最大的线程数, 并且线程池大小小于最大的线程数, 那么就扩充线程池的大小到最大线程数。
                for (int i = size; i < max; i++) {
                    createWorkTask();
                }
                System.out.println("The Pool incremented to max");
                size = max;
            }
            synchronized (THREAD_QUEUE) {
                // 如果任务队列已经空了, 那就占用TASK_QUEUE的锁, 降低线程池的线程数量
                if (TASK_QUEUE.isEmpty() && size > active) {
                    System.out.println("=======Reduce=======");
                    int releaseSize = size - active;
                    for (Iterator<WorkerTask> it = THREAD_QUEUE.iterator(); it.hasNext(); ) {
                        if (releaseSize <= 0) {
                            break;
                        }

                        WorkerTask task = it.next();
                        task.close();
                        task.interrupt();
                        it.remove();
                        releaseSize--;
                    }
                    size = active;
                }
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

##### 5. 拒绝策略

当线程池中的线程数达到最大线程, 且任务队列已经满的时候,此时再添加任务, 就会执行拒绝策略。我们这里就抛出一个异常

```java
private final DiscardPolicy discardPolicy;

private final static DiscardPolicy DEFAULT_DISCARD_POLICY = () -> {
    throw new DiscardException("Discard This Task.");
};

public interface DiscardPolicy {
    void discard() throws DiscardException;
}

public static class DiscardException extends RuntimeException {
    public DiscardException(String message) {
        super(message);
    }
}
```

修改submit()代码:

```java
public void submit(Runnable runnable) {
    if (destroy) {
        throw new IllegalStateException("这个线程池已经销毁了, 不允许提交任务");
    }
    synchronized (TASK_QUEUE) {
        if (TASK_QUEUE.size() > queueSize) {
            discardPolicy.discard();
        }
        TASK_QUEUE.addLast(runnable);
        TASK_QUEUE.notifyAll();
    }
}
```

以上基本上一个简易的线程池就已经完成了, 下面的完整的代码, 以及测试用例:

```java
public class SimpleThreadPool extends Thread {
    // 线程池中总线程数
    private int size;

    // 任务队列数
    private int queueSize;

    // 最少线程数
    private int min;

    // 最大线程数
    private int max;

    // 活动线程数
    private int active;

     //任务队列默认大小
    private final static int DEFAULT_TASK_QUEUE_SIZE = 2000;

    private static volatile int seq = 0;

    private final static String THREAD_PREFIX = "SIMPLE_THREAD_POOL-";

    private final static ThreadGroup GROUP = new ThreadGroup("Pool_Group");

    // 任务队列
    private final static LinkedList<Runnable> TASK_QUEUE = new LinkedList<>();

    // 线程队列
    private final static List<WorkerTask> THREAD_QUEUE = new ArrayList<>();

    // 判断线程池是否被销毁
    private volatile boolean destroy = false;

    private final DiscardPolicy discardPolicy;

    private final static DiscardPolicy DEFAULT_DISCARD_POLICY = () -> {
        throw new DiscardException("Discard This Task.");
    };

    public SimpleThreadPool() {
        this(4, 8, 12, DEFAULT_TASK_QUEUE_SIZE, DEFAULT_DISCARD_POLICY);
    }

    /**
     * 在new SimpleThreadPool时会加载init()方法, 创建size数量的线程
     */
    public SimpleThreadPool(int min, int active, int max, int queueSize, DiscardPolicy discardPolicy) {
        this.min = min;
        this.active = active;
        this.max = max;
        this.queueSize = queueSize;
        this.discardPolicy = discardPolicy;
        init();
    }

    private void init() {
        // 先判断线程数min是否足够
        for (int i = 0; i < min; i++) {
            createWorkTask();
        }
        this.size = min;
        this.start();
    }

    public void submit(Runnable runnable) {
        if (destroy) {
            throw new IllegalStateException("这个线程池已经销毁了, 不允许提交任务");
        }
        synchronized (TASK_QUEUE) {
            if (TASK_QUEUE.size() > queueSize) {
                discardPolicy.discard();
            }
            TASK_QUEUE.addLast(runnable);
            TASK_QUEUE.notifyAll();
        }
    }

    /**
     * 线程池的线程: 用于维护控制线程池中的扩充因子, 例如最大最小线程数等。
     */
    @Override
    public void run() {
        while (!destroy) {
            System.out.printf("Pool#Min:%d, Active:%d, Max:%d, Current:%d,QueueSize:%d\n",
                    this.min, this.active, this.max, this.size, TASK_QUEUE.size());
            try {
                Thread.sleep(5_000L);
                // 如果任务队列的任务数已经大于活跃的线程数, 并且线程池大小小于活跃的线程数, 那么就扩充线程池的大小。
                if (TASK_QUEUE.size() > active && size < active) {
                    for (int i = size; i < active; i++) {
                        createWorkTask();
                    }
                    System.out.println("The Pool incremented to active");
                    size = active;
                } else if (TASK_QUEUE.size() > max && size < max) {// 如果任务队列的任务数已经大于最大的线程数, 并且线程池大小小于最大的线程数, 那么就扩充线程池的大小到最大线程数。
                    for (int i = size; i < max; i++) {
                        createWorkTask();
                    }
                    System.out.println("The Pool incremented to max");
                    size = max;
                }
                synchronized (THREAD_QUEUE) {
                    // 如果任务队列已经空了, 那就占用TASK_QUEUE的锁, 降低线程池的线程数量
                    if (TASK_QUEUE.isEmpty() && size > active) {
                        System.out.println("=======Reduce=======");
                        int releaseSize = size - active;
                        for (Iterator<WorkerTask> it = THREAD_QUEUE.iterator(); it.hasNext(); ) {
                            if (releaseSize <= 0) {
                                break;
                            }

                            WorkerTask task = it.next();
                            task.close();
                            task.interrupt();
                            it.remove();
                            releaseSize--;
                        }
                        size = active;
                    }
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    private void createWorkTask() {
        WorkerTask task = new WorkerTask(GROUP, THREAD_PREFIX + (seq++));
        task.start();
        THREAD_QUEUE.add(task);
    }

    public void shutdown() throws InterruptedException {
        while (!TASK_QUEUE.isEmpty()) {
            Thread.sleep(50);
        }
        synchronized (THREAD_QUEUE) {
            int initVal = THREAD_QUEUE.size();
            while (initVal > 0) {
                for (WorkerTask task : THREAD_QUEUE) {
                    if (task.getTaskState() == TaskState.BLOCKED) {
                        task.interrupt();
                        task.close();
                        initVal--;
                    } else {
                        Thread.sleep(10);
                    }
                }
            }
        }
        this.destroy = true;
        System.out.println("线程池已经关闭了");
    }

    public int getSize() {
        return size;
    }

    public int getQueueSize() {
        return queueSize;
    }

    public boolean isDestroy() {
        return this.destroy;
    }

    private enum TaskState {
        FREE, RUNNING, BLOCKED, DEAD
    }

    public static class DiscardException extends RuntimeException {
        public DiscardException(String message) {
            super(message);
        }
    }

    public interface DiscardPolicy {
        void discard() throws DiscardException;
    }

    /**
     *
     */
    private static class WorkerTask extends Thread {
        private volatile TaskState taskState = TaskState.FREE;

        public WorkerTask(ThreadGroup threadGroup, String name) {
            super(threadGroup, name);
        }

        public TaskState getTaskState() {
            return this.taskState;
        }

        @Override
        public void run() {
            OUTER:
            while (this.taskState != TaskState.DEAD) {
                Runnable runnable;
                synchronized (TASK_QUEUE) {
                    while (TASK_QUEUE.isEmpty()) {
                        try {
                            taskState = TaskState.BLOCKED;
                            TASK_QUEUE.wait();
                        } catch (InterruptedException e) {
                            System.out.println("Close.");
                            break OUTER;
                        }
                    }
                    //队列先进先出, 这里remove first
                    runnable = TASK_QUEUE.removeFirst();
                }

                if (runnable != null) {
                    taskState = TaskState.RUNNING;
                    runnable.run();
                    taskState = TaskState.FREE;
                }
            }
        }

        public void close() {
            this.taskState = TaskState.DEAD;
        }
    }

    public static void main(String[] args) throws InterruptedException {
//        SimpleThreadPool threadPool = new SimpleThreadPool(6, 10, SimpleThreadPool.DEFAULT_DISCARD_POLICY);
        SimpleThreadPool threadPool = new SimpleThreadPool();
        for (int i = 0; i < 40; i++) {
            int finalI = i;
            threadPool.submit(new Runnable() {
                @Override
                public void run() {
                    System.out.println("当前任务线程 " + finalI + "服务" + Thread.currentThread() + " 开始。");
                    try {
                        Thread.sleep(3000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    System.out.println("当前任务线程 " + finalI + "服务" + Thread.currentThread() + " 结束。");
                }
            });
        }

        Thread.sleep(10_000);
        threadPool.shutdown();
//        threadPool.submit(() -> System.out.println("再次提交任务"));
    }
}
```