## 生产者消费者模式

这种模式, 我们在基础阶段已经学习过了。之前我们使用wait(), notifyAll()方法实现了生产者消费者模式, 并且详细的介绍了wait(), notify(), notifyAll()方法。[base_produce&consume](/Java_base/concurrent/base/thread_produce&consume.md)

> 之前所使用的生产者消费者模式, 虽然有多个生产者生产数据, 但是每次都只能生产一次, 消费者消费一次。但是这样的效率很低, 不符合我们实际项目中的需求。

对于我们实现开发中, 一般要求生产者能够不断的生产数据, 而消费者也可以不断的消费。**我们利用消息队列的形式, 生产者不断的往数据队列中不断生产数据, 直到达到队列的最大值; 而消费者不断的从队列中消费数据, 直到队列为空**

### 代码实现

##### 1. 创建数据实体
```java
public class Message {

    private String data;

    public Message(String data) {
        this.data = data;
    }

    public String getData() {
        return data;
    }
}
```

##### 2. 创建数据队列

```java
public class MessageQueue {
    private final LinkedList<Message> queue;

    private final static int DEFAULT_MAX_LIMIT = 100;

    private final int limit;

    public MessageQueue() {
        this(DEFAULT_MAX_LIMIT);
    }

    public MessageQueue(final int limit) {
        this.limit = limit;
        this.queue = new LinkedList<>();
    }

    public void put(final Message message) throws InterruptedException {
        synchronized (queue) {
            while (queue.size() > limit) {
                queue.wait();
            }
            queue.addLast(message);
            queue.notifyAll();
        }
    }

    public Message take() throws InterruptedException {
        synchronized (queue) {
            while (queue.isEmpty()) {
                queue.wait();
            }

            Message message = queue.removeFirst();
            queue.notifyAll();
            return message;
        }
    }

    public int getMaxLimit() {
        return this.limit;
    }

    public int getMessageSize() {
        synchronized (queue) {
            return queue.size();
        }
    }

}
```

##### 3. 创建生产者线程

```java
public class ProducerThread extends Thread {

    private final MessageQueue messageQueue;

    private final static Random random = new Random(System.currentTimeMillis());

    private final static AtomicInteger counter = new AtomicInteger(0);

    public ProducerThread(MessageQueue messageQueue, int seq) {
        super("producer" + seq);
        this.messageQueue = messageQueue;
    }

    @Override
    public void run() {
        while (true) {
            try {
                Message message = new Message("Message-" + counter.getAndIncrement());
                messageQueue.put(message);
                System.out.println(Thread.currentThread().getName() + " put message " + message.getData());
                Thread.sleep(random.nextInt(1000));
            } catch (InterruptedException e) {
                break;
            }
        }
    }
}
```

##### 4. 创建消费者线程

```java
public class ConsumerThread extends Thread {

    private final MessageQueue messageQueue;

    private final static Random random = new Random(System.currentTimeMillis());

    public ConsumerThread(MessageQueue messageQueue, int seq) {
        super("Consumer" + seq);
        this.messageQueue = messageQueue;
    }

    @Override
    public void run() {
        while (true) {
            try {
                Message message = messageQueue.take();
                System.out.println(Thread.currentThread().getName() + " take a message " + message.getData());
                Thread.sleep(random.nextInt(1000));
            } catch (InterruptedException e) {
                break;
            }
        }
    }
}
```

##### 5. 测试

```java
public class ProducerAndConsumerClient {

    public static void main(String[] args) {
        final MessageQueue messageQueue = new MessageQueue();

        new ProducerThread(messageQueue, 1).start();
        new ProducerThread(messageQueue, 2).start();
        new ProducerThread(messageQueue, 3).start();
        new ConsumerThread(messageQueue, 1).start();
        new ConsumerThread(messageQueue, 2).start();
    }
}
```

结果:

```txt
producer2 put message Message-2
producer1 put message Message-1
Consumer2 take a message Message-2
producer3 put message Message-0
Consumer1 take a message Message-0
Consumer1 take a message Message-1
producer3 put message Message-3
Consumer1 take a message Message-3
producer1 put message Message-4
Consumer2 take a message Message-4
producer2 put message Message-5
Consumer2 take a message Message-5
producer3 put message Message-6
Consumer1 take a message Message-6
producer1 put message Message-7
producer2 put message Message-8
Consumer2 take a message Message-7
Consumer1 take a message Message-8
producer3 put message Message-9
Consumer1 take a message Message-9
producer1 put message Message-10

...
```