## Future设计模式

> 试想这样的一个场景: <br/>
> 如果一个任务分为work1, work2, work3。 如果work3需要用到work1的结构, 但是work1执行的事件很长, 导致work2一直处于等待状态。那么这种情况就可以使用future, 这样work1, work2可以并行, 当执行到work3时, 可以从work1的future中获取到相应的结果。

所以Future设计模式, 又被称为并行设计模式, 它的核心在于: 去除主线程的等待时间, 原本需要等待的时间段可以用于处理其他业务逻辑。

![future时序图](/image/future_model_timing.png)

根据上图我们可以看到, 传统的调用是串行化, 必须要等待返回数据后才能继续其他操作; 而future模式客户端在等待结果返回的过程中, 仍然可以处理其他逻辑。

Future模式中主要参与者如下:

| 参与者 | 作用 |
| -- | -- |
| Client  | 客户端 |
| FutureService | 返回一个Future对象, 并开启Thread线程去处理真正的业务逻辑 |
| Future | 返回数据的接口 |
| FutureTask | 提交的任务 |
| AsynFuture | Future的实现, 用于装配真正的数据 |

Future对应的UML图:

![future_model](/image/future_model.png)

### Future模式的代码实现

创建Future接口:

```java
public interface Future<T> {

    T get() throws InterruptedException;
}
```

创建Future的实现类:

```java
public class AsynFuture<T> implements  Future<T> {

    private volatile boolean done = false;

    private T result;

    public void done(T result) {
        synchronized (this) {
            this.result = result;
            this.done = true;
            this.notifyAll();
        }
    }

    @Override
    public T get() throws InterruptedException {
        synchronized (this) {
            while (!done) {
                this.wait();
            }
        }
        return result;
    }
}
```

创建FutureService, 用于提交任务并且返回一个Future实例:

```java
public class FutureService {

    public <T> Future<T> submit(final FutureTask<T> task) {
        AsynFuture<T> asynFuture = new AsynFuture<>();
        new Thread(() -> {
            T result = task.call();
            asynFuture.done(result);
        }).start();
        return asynFuture;
    }

    /**
     * 加入consumer, 回调的方式。可以主动的返回数据, 而不需要去get()
     * @param task
     * @param consumer
     * @param <T>
     * @return
     */
    public <T> Future<T> submit(final FutureTask<T> task, final Consumer<T> consumer) {
        AsynFuture<T> asynFuture = new AsynFuture<>();
        new Thread(() -> {
            T result = task.call();
            asynFuture.done(result);
            consumer.accept(result);
        }).start();
        return asynFuture;
    }

}
```

创建FutureTask, 返回实际的返回对象和工作内容:

```java
public interface FutureTask<T> {
    T call();
}
```

创建Client:

```java
/**
     * Future         -> 代表是未来的一个凭据
     * FutureTask     -> 将你的调用逻辑进行隔离
     * FutureService  -> 桥接 Future和FutureTask
     * @param args
     * @throws InterruptedException
     */
    public static void main(String[] args) throws InterruptedException {
        /*// 如果是同步调用中, 必须要等待10s之后, 才能返回结果
        String result = get();
        System.out.println(result);*/
        FutureService futureService = new FutureService();
//        Future<String> future = futureService.submit(() -> {
//            try {
//                Thread.sleep(10000L);
//            } catch (InterruptedException e) {
//                e.printStackTrace();
//            }
//            return "finish";
//        });

        System.out.println("=============");
        System.out.println("do other thing.");
        Thread.sleep(1000);
        System.out.println("=============");

        // 但是使用这种方式, 还是需要主动的去获取数据,如果这个线程早就处理完了, 不应该要主动的获取, 而是通过一种方式, 当数据处理完成时, 直接返回。
        // System.out.println(future.get());

        futureService.submit(() -> {
            try {
                Thread.sleep(10000L);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return "finish";
        }, System.out::println);
    }

    /**
     * 假设这是一个非常耗时的操作
     * @return
     * @throws InterruptedException
     */
    public static String get() throws InterruptedException {
        Thread.sleep(10000L);
        return "finish";
    }
```

从上面的例子中, 我们可以看到了, 我对future其实做了一个优化, 对submit方法, 做了一个重写, 当futureTask任务做完时, 可以主动返回数据, 不需要去主动获取。

