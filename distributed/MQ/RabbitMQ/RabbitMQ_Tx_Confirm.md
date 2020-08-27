## <center>RabbitMQ事务和Confirm消息确认</center>

我们知道, **如果要保证消息的可靠性, 需要对消息进行持久化处理, 然而消息持久化除了需要代码的设置之外, 还有非常重要的一步就是保证你的消息顺利进入Broker(代理服务器)**

![rabbit_process](/distributed/MQ/img/rabbit_process.png)

正常情况下, 如果消息经过交换器进入队列就可以完成消息的持久化, 但是如果消息在没有到达Broker之前出现意外, 那就造成消息丢失。

RabbitMQ有两种方式来解决这个问题:

1. 通过AMQP提供的事务机制实现;
2. 使用发送者确认模式实现。

### 1. 事务使用

事务的实现主要是对信道(Channel)的设置:

- `channel.txSelect()`声明启动事务模式;
- `channel.txComment()`提交事务;
- `channel.txRollback()`回滚事务;

在发送消息之前需要声明channel为事务模式, 提交或者回滚事务即可。所以客户端与服务端交互的流程为:

- 客户端发送给服务器Tx.Select(开启事务模式)
- 服务端返回Tx.Select-OK(开始事务模式OK)
- 推送消息
- 客户端发送给事务提交Tx.Commit
- 服务端返回Tx.Commit-OK

上面是就完成事务的交互流程, 如果其中任意一个环节出了问题, 就会抛出`IOException`异常, 这样用户就可以拦截异常进行事务回滚, 或者决定要不要重复消息。

但是既然已经有了事务模式, 为什么还需要发送方模式呢? **因为事务的性能非常的差。**


### 2. Confirm发送方确认模式

Confirm发送方确认模式使用和事务类似, 也是通过设置Channel进行发送发确认的。

> Confirm的三种实现方式:

- `channel.watForConfirms()`普通发送方确认模式;
- `channel.waitForConfirmsOrDie()`批量确认模式;
- `channel.addConfirmListener()`异步监听发送方确认模式。




