## 线程池拒绝策略

之前学习了线程池, 我们知道当工作线程达到核心线程数corePoolSize时, 新提交的任务就会进入阻塞队列, 如果阻塞队列也满了, 那么线程池就会扩容到最大线程数maximumPoolSize; 如果此时还提交新的任务, 那么就会执行拒绝策略
`RejectedExecutionHandler`。

在`ThreadPoolExecutor`中封装了4中不同的拒绝策略, 我们一个一个去了解。

先看下面的代码:

```java
public class ExecutorServiceExample2 {

    public static void main(String[] args) throws InterruptedException {
//        AbortPolicyTest();
//        DiscardPolicyTest();
//        CallerRunsPolicyTest();
        DiscardOldestPolicyTest();
    }

    private static void AbortPolicyTest() throws InterruptedException {
        ExecutorService executorService = new ThreadPoolExecutor(1, 2, 30, TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(1), r -> {
            Thread thread = new Thread(r);
            return thread;
        }, new ThreadPoolExecutor.AbortPolicy());

        for (int i = 0; i < 3; i++) {
            executorService.execute(() -> {
                try {
                    TimeUnit.SECONDS.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            });
        }

        TimeUnit.SECONDS.sleep(1);
        executorService.execute(() -> System.out.println("x"));
    }

    private static void DiscardPolicyTest() throws InterruptedException {
        ExecutorService executorService = new ThreadPoolExecutor(1, 2, 30, TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(1), r -> {
            Thread thread = new Thread(r);
            return thread;
        }, new ThreadPoolExecutor.DiscardPolicy());

        for (int i = 0; i < 3; i++) {
            executorService.execute(() -> {
                try {
                    TimeUnit.SECONDS.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            });
        }

        TimeUnit.SECONDS.sleep(1);
        executorService.execute(() -> System.out.println("x"));
        System.out.println("==============");
    }

    private static void CallerRunsPolicyTest() throws InterruptedException {
        ExecutorService executorService = new ThreadPoolExecutor(1, 2, 30, TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(1), r -> {
            Thread thread = new Thread(r);
            return thread;
        }, new ThreadPoolExecutor.CallerRunsPolicy());

        for (int i = 0; i < 3; i++) {
            executorService.execute(() -> {
                try {
                    TimeUnit.SECONDS.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            });
        }

        TimeUnit.SECONDS.sleep(1);
        executorService.execute(() -> {
            System.out.println("x");
            System.out.println(Thread.currentThread().getName());
        });
        System.out.println("==============");
    }

    private static void DiscardOldestPolicyTest() throws InterruptedException {
        ExecutorService executorService = new ThreadPoolExecutor(1, 2, 30, TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(1), r -> {
            Thread thread = new Thread(r);
            return thread;
        }, new ThreadPoolExecutor.DiscardOldestPolicy());

        for (int i = 0; i < 3; i++) {
            executorService.execute(() -> {
                try {
                    TimeUnit.SECONDS.sleep(5);
                    System.out.println("I come from lambda.");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            });
        }

        TimeUnit.SECONDS.sleep(1);
        executorService.execute(() -> {
            System.out.println("x");
            System.out.println(Thread.currentThread().getName());
        });
        System.out.println("==============");
    }


}
```

四种拒绝策略:

- `AbortPolicy`: 提交任务时会抛出异常;
- `DiscardPolicy`: 提交的任务会直接忽略不会执行;
- `CallerRunsPolicy`: 提交的任务会交给提交任务的线程(代码中是main线程)来执行, 不会交给线程池中的线程;
- `DiscardOldestPolicy`: 当提交任务时, 会将阻塞队列queue中最先进入的任务移除掉, 将新提交的任务加入队列中。
