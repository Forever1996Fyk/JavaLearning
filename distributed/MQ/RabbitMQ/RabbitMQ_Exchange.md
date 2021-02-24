## RabbitMQ 交换器Exchange

### 1. 交换器分类

RabbitMQ的Exchange分为四类:

- direct(默认)
- headers
- fanout
- topic

其中header交换器允许匹配AMQP消息的header而非路由键, 除此之外header交换器和direct交换器完全一致, 但是性能很差, 几乎用不到, 所以这里不作讲解。

### 2. direct交换器

direct为默认的交换器类型, 根据消息携带的路由键(rounting key)将消息投递给对应的队列。

而绑定键(binding key)的作用就是, 把交换器和队列按照路由的规则绑定起来。 

代码示例如下:

- **发送端:**

    ```java
    Connection conn = connectionFactoryUtil.GetRabbitConnection();
    Channel channel = conn.createChannel();
    // 声明队列【参数说明：参数一：队列名称，参数二：是否持久化；参数三：是否独占模式；参数四：消费者断开连接时是否删除队列；参数五：消息其他参数】
    channel.queueDeclare(config.QueueName, false, false, false, null);
    String message = String.format("当前时间：%s", new Date().getTime());
    // 推送内容【参数说明：参数一：交换机名称；参数二：队列名称，参数三：消息的其他属性-路由的headers信息；参数四：消息主体】
    channel.basicPublish("", config.QueueName, null, message.getBytes("UTF-8"));
    ```

- **接收端, 持续接收消息:**

    ```java
    Connection conn = connectionFactoryUtil.GetRabbitConnection();
    Channel channel = conn.createChannel();
    // 声明队列【参数说明：参数一：队列名称，参数二：是否持久化；参数三：是否独占模式；参数四：消费者断开连接时是否删除队列；参数五：消息其他参数】
    channel.queueDeclare(config.QueueName, false, false, false, null);
    Consumer defaultConsumer = new DefaultConsumer(channel) {
        @Override
        public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties,
                byte[] body) throws IOException {
            String message = new String(body, "utf-8"); // 消息正文
            System.out.println("收到消息 => " + message);
            channel.basicAck(envelope.getDeliveryTag(), false); // 手动确认消息【参数说明：参数一：该消息的index；参数二：是否批量应答，true批量确认小于当前id的消息】
        }
    };
    channel.basicConsume(config.QueueName, false, "", defaultConsumer);
    ```

- **接收端, 获取单条消息:**

    ```java
    Connection conn = connectionFactoryUtil.GetRabbitConnection();
    Channel channel = conn.createChannel();
    channel.queueDeclare(config.QueueName, false, false, false, null);
    GetResponse resp = channel.basicGet(config.QueueName, false);
    String message = new String(resp.getBody(), "UTF-8");
    channel.basicAck(resp.getEnvelope().getDeliveryTag(), false); // 消息确认
    ```

> 这里要注意的是, 不能使用for循环单个消息来替代持续消息消费, 因为一个线程在不断的死循环获取数据, 对CPU消耗较大, 而且性能很低。利用socket的心跳机制, 更合适。


#### 2.1 公平调度

当接收端订阅者有多个时, direct会轮询公平的分发给每个订阅者(订阅者消息确认正常)。

#### 2.2 消息的发后既忘特性

**发后既忘是指接受者不知道消息的来源, 如果想要指定消息的发送者, 需要包含在发送内容里面。**

#### 2.3 消息确认

如果程序接收了消息, 但是忘记确认接收时, 消息在队列的状态会从"Ready"变为"Unacked"。如果消息收到为确认, Rabbit将不会给这个程序发送更多的消息, 因为Rabbit认为你没有准备好接收下一条消息, 此条消息会一直保持Unacked状态, 直到你确认了消息, 或者断开Rabbit的连接。

**但是利用这一点, 可以让程序进行延迟确认消息, 直到程序处理完相应的业务逻辑, 也可以有效的防止Rabbit给你过多的消息, 导致程序崩溃。**

```java
Connection conn = connectionFactoryUtil.GetRabbitConnection();
Channel channel = conn.createChannel();
channel.queueDeclare(config.QueueName, false, false, false, null);
GetResponse resp = channel.basicGet(config.QueueName, false);
String message = new String(resp.getBody(), "UTF-8");

// basicAck就是消息确认, [参数1：消息的id；参数2：是否批量应答，true批量确认小于次id的消息。]
channel.basicAck(resp.getEnvelope().getDeliveryTag(), false);
```

但是无论如何, **消费者消费的每条消息都必须确认!**

#### 2.4 消息拒绝

消息拒绝有两种:

- 断开与Rabbit的连接, 这样Rabbit会重新把消息分派给另一个消费者;

- 通过代码方式拒绝消息, 这样Rabbit会把消息发送到一个特殊的"死信"队列, 用来存放被拒绝而不重新放入队列的消息。

    消息拒绝代码:

    ```java
    Connection conn = connectionFactoryUtil.GetRabbitConnection();
    Channel channel = conn.createChannel();
    channel.queueDeclare(config.QueueName, false, false, false, null);
    GetResponse resp = channel.basicGet(config.QueueName, false);
    String message = new String(resp.getBody(), "UTF-8");
    channel.basicReject(resp.getEnvelope().getDeliveryTag(), true); //消息拒绝
    ```

### 3. fanout交换器 ———— 发布/订阅模式

fanout有别于direct交换器, fanout是一种`发布/订阅`模式的交换器。当发送一条消息时, 交换器会消息广播到所有跟这个交换器有关的队列上。

这是`fanout`交换器与`direct`交换器最大的区别:

- `fanout`交换器是一种发布订阅模式, 客户端通过订阅交换器的方式, 一旦服务端发送消息到交换器, 就广播到各个订阅者。

- **direct交换器本质上是通过socket的方式, 利用客户端与服务端的连接传递消息。**

- fanout和direct的区别最多是在接收端, fanout需要绑定队列到对应的交换器用于订阅消息。

代码示例:

- **发送端:**

    ```java
    final String ExchangeName = "fanoutec"; // 交换器名称
    Connection conn = connectionFactoryUtil.GetRabbitConnection();
    Channel channel = conn.createChannel();
    channel.exchangeDeclare(ExchangeName, "fanout"); // 声明fanout交换器
    String message = "时间：" + new Date().getTime();
    channel.basicPublish(ExchangeName, "", null, message.getBytes("UTF-8"));
    ```

- **接收端:**

    ```java
    Connection conn = connectionFactoryUtil.GetRabbitConnection();
    Channel channel = conn.createChannel();
    channel.exchangeDeclare(ExchangeName, "fanout"); // 声明fanout交换器
    String queueName = channel.queueDeclare().getQueue(); // 声明队列
    channel.queueBind(queueName, ExchangeName, "");
    Consumer consumer = new DefaultConsumer(channel) {
        @Override
        public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties,
                byte[] body) throws IOException {
            String message = new String(body, "UTF-8");
        }
    };
    channel.basicConsume(queueName, true, consumer);
    ```

**对于fanout交换器来说, routingKey(路由键)是无效的, 这个参数是被忽略的。**


#### 3. topic交换器————匹配订阅模式

topic交换器与fanout类似, 但是可以更加灵活的匹配自己想要订阅的信息, 需要使用`routingKey`路由键进行消息(规则)匹配。