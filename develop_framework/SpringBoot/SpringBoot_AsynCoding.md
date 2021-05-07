## Spring Boot 异步编程

这里简单的通过代码介绍一下SpringBoot的异步编程。

我个人认为异步编程与异步任务都是差不多的, 都是为了处理一些代码逻辑无关的任务。例如： 发送短信, 发送邮件, app消息推送, 记录日志等等。

### 1. Future模式

Future模式是多线程开发中常见的设计模式, 它的核心思想是**异步调用**。
当我们执行一个方法时, 如果这个方法有多个耗时的任务需要同时去做, 而且并不在意返回的结果时, 可以立即返回结果, 然后后台慢慢处理任务。也可以等任务全部完成, 返回给客户端。

### 2. SpringBoot开启异步编程

通过Spring提供的两个注解会更加简单的实现异步编程:

1. `@EnableAsync`: 通过在配置类或者启动类上加`@EnableAsync`注解开启对异步方法的支持。
2. `@Async`: 可以标注在类上或者方法上, 标注在类上代表这个类的所有方法都是异步方法。

### 3. 自定义 TaskExecutor

Web应用中使用线程是非常常用的操作, 但是如果希望线程能够被Spring管理, 这样既能够使用应用程序的其他组件, 也不会带来任何影响, 而且可以优雅的关闭线程。

所以Spring提供了`TaskExecutor`作为任务执行的抽象, 使用它执行线程来处理任务。它与`java.util.concurrent`包下的`Executor`接口很像, 但是不同的是`TaskExecutor`接口用到了Java8的语法。 `@FunctionalInterface`声明这个接口是一个函数式接口。

`org.springframework.core.task.TaskExecutor`:

```java
@FunctionalInterface
public interface TaskExecutor extends Executor {
    void execute(Runnable var1);
}
```

![TaskExecutor](/image/TaskExecutor.png)

如果没有自定义`Executor`, Spring将创建一个`SimpleAsyncTaskExector`并使用它。

```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

  private static final int CORE_POOL_SIZE = 6;
  private static final int MAX_POOL_SIZE = 10;
  private static final int QUEUE_CAPACITY = 100;

  @Bean
  public Executor taskExecutor() {
    // Spring 默认配置是核心线程数大小为1，最大线程容量大小不受限制，队列容量也不受限制。
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    // 核心线程数
    executor.setCorePoolSize(CORE_POOL_SIZE);
    // 最大线程数
    executor.setMaxPoolSize(MAX_POOL_SIZE);
    // 队列大小
    executor.setQueueCapacity(QUEUE_CAPACITY);
    // 当最大池已满时，此策略保证不会丢失任务请求，但是可能会影响应用程序整体性能。
    executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
    executor.setThreadNamePrefix("My ThreadPoolTaskExecutor-");
    executor.initialize();
    return executor;
  }
}
```

`ThreadPoolTaskExecutor`常见概念:

- **Core Pool Size**: 核心线程数, 定义了最小可以同时运行的线程数量。
- **Queue Capacity**: 当新任务来时, 先判断当前运行的线程数是否达到核心线程数, 如果达到, 新任务就会存在队列中。
- **Maximum Pool Size**: 当队列中存放的任务达到队列容量时, 当前可以同时运行的线程数量变为最大线程数。也就是核心线程数Core Pool Size --> 最大线程数Maximum Pool Size

一般情况下不会将队列大小设为: `Integer.MAX_VALUE`, 因为这样系统内存可能都无法分配这么大的队列, 也不会将核心线程数和最大线程数这位同样的大小, 这样的话最大线程数就没有意义了, 而且也无法确定当前CPU和内存利用率的具体情况。

> 如果队列已满, 而且当前同时运行的线程达到最大线程数时, 如果还有新任务过来如何处理?

Spring默认使用的`ThreadPoolExecutor.AbortPolicy`。也就是`TheadPoolExecutor`抛出`RejectedExecutionException`异常来拒绝新的任务, 这表示会丢失对这个任务的处理。

对于可伸缩的程序, 可以使用`ThreadPookExecutor.CallerRunsPolicy`。 当达到最大线城数时, 这个策略会提供可伸缩的队列。

`ThreadPoolTaskExecutor`饱和策略定义: 

如果当前同时运行的线程数量达到最大线程数, 且任务队列也填满时, 可以定义一下策略。

- **ThreadPoolExecutor.AbortPolicy**: 抛出`RejectedExecutorException`来拒绝新任务的处理。
- **ThreadExecutor.CallerRunsPolicy**: 调用执行自己的线程运行任务, 增加队列容量。但是这种策略会降低对新任务提交的速度, 影响程序的整体性能。
- **ThreadPoolExecutor.DiscardPolicy**: 不处理新任务, 直接丢弃掉。
- **ThreadPoolExecutor.DiscardOldestPolicy**: 丢弃最早的未处理的任务请求。


### 4. 编写异步代码

事实上在实际的项目开发中, 异步编程式比较复杂的, 而且调试很麻烦, 所以做好异步编程是需要一定的编程功底的。

举例的方法是循环查找对应的字符串, 通过`@Async`注解, 定义一个异步方法, 而且返回结果值是`CompletableFuture.completedFuture(results)`, 这意味着我们需要返回结果, 而且必须是任务完成之后再返回。

```java
@Service
public class AsyncService {

  private static final Logger logger = LoggerFactory.getLogger(AsyncService.class);

  private List<String> movies =
      new ArrayList<>(
          Arrays.asList(
              "Forrest Gump",
              "Titanic",
              "Spirited Away",
              "The Shawshank Redemption",
              "Zootopia",
              "Farewell ",
              "Joker",
              "Crawl"));

  /** 示范使用：找到特定字符/字符串开头的电影 */
  @Async
  public CompletableFuture<List<String>> completableFutureTask(String start) {
    // 打印日志
    logger.warn(Thread.currentThread().getName() + "start this task!");
    // 找到特定字符/字符串开头的电影
    List<String> results =
        movies.stream().filter(movie -> movie.startsWith(start)).collect(Collectors.toList());
    // 模拟这是一个耗时的任务
    try {
      Thread.sleep(1000L);
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
    //返回一个已经用给定值完成的新的CompletableFuture。
    return CompletableFuture.completedFuture(results);
  }
}
```

```java
@RestController
@RequestMapping("/async")
public class AsyncController {
  @Autowired 
  AsyncService asyncService;

  @GetMapping("/movies")
  public String completableFutureTask() throws ExecutionException, InterruptedException {
    //开始时间
    long start = System.currentTimeMillis();
    // 开始执行大量的异步任务
    List<String> words = Arrays.asList("F", "T", "S", "Z", "J", "C");
    List<CompletableFuture<List<String>>> completableFutureList =
        words.stream()
            .map(word -> asyncService.completableFutureTask(word))
            .collect(Collectors.toList());
    // CompletableFuture.join（）方法可以获取他们的结果并将结果连接起来
    List<List<String>> results = completableFutureList.stream().map(CompletableFuture::join).collect(Collectors.toList());
    // 打印结果以及运行程序运行花费时间
    System.out.println("Elapsed time: " + (System.currentTimeMillis() - start));
    return results.toString();
  }
```

**上面的代码是当所有任务执行完成之后才返回结果, 这种情况是对应于需要返回结果的客户端请求的情况下, 如果我们不需要返回结果呢?**

例如: 我们需要上传大文件到系统, 一般情况下, 我们是等待文件上传完毕后再返回用户信息, 但是这样会很慢。但是使用异步方法, 当用户上传之后就立马返回给用户信息, 然后后台在默默的处理上传任务。**当然这样如果文件上传失败的话, 系统也需要补偿机制, 比如发消息通知用户**。

所以改变上面的代码, 没有返回值 :

```java
@Async
public void completableFutureTask(String start) {
  ......
  //这里可能是系统对任务执行结果的处理，比如存入到数据库等等......
  //doSomeThingWithResults(results);
}
```

```java
@GetMapping("/movies")
  public String completableFutureTask() throws ExecutionException, InterruptedException {
    // Start the clock
    long start = System.currentTimeMillis();
    // Kick of multiple, asynchronous lookups
    List<String> words = Arrays.asList("F", "T", "S", "Z", "J", "C");
        words.stream()
            .forEach(word -> asyncService.completableFutureTask(word));
    // Wait until they are all done
    // Print results, including elapsed time
    System.out.println("Elapsed time: " + (System.currentTimeMillis() - start));
    return "Done";
  }
```