## <center>RocketMQ 重要知识点</center>

### 1. RocketMQ的架构

`RocketMQ`技术架构中有四大角色: `NameServer`, `Broker`, `Producer`, `Consumer`。

- `Broker`: 主要负责消息的存储, 投递和查询以及服务高可用保证。也就是消息队列服务器, 生产者生产消息到`Broker`, 消费者从`Broker`拉去消息并消费。
    
    **一个`Topic`分布在多个`Broker`上, 一个`Broker`可以配置多个`Topic`, 它们是多对多的关系。**

    如果某个`Topic`消息量很大, 应该给他多配置集合队列, 并且**尽量多分布在不同的`Broker`上, 以减轻某个`Broker`的压力。**

    `Topic`消息量都比较均匀的情况下, 如果某个`Broker`上的队列越多, 则该`Broker`压力越大。

    ![rocket_topic_broker](/distributed/MQ/img/rocket_topic_broker.png)

- `NameServer`: 这是一个**注册中心**, 主要提供两个功能: **`Broker`管理和路由信息管理。** `Broker`将自己的信息注册到`NameServer`中, 即`Broker`的路由表。消费者和生产者就从`NameServer`中获取路由表然后根据对应的`Broker`进行通信。

- `Producer`: 消息发布的角色, 支持分布式集群方式部署
- `Consumer`: 消息消费的角色, 支持分布式集群方式部署。支持以push推, pull拉两种模式对消息进行消费。提供实时消息订阅机制。

![rocket_nameserver](/distributed/MQ/img/rocket_nameserver.png)

> 通过上图，可能会有疑问, 这个`NameServer`的用处是什么呢, 好像完全是多余的。直接`Producer`, `Consumer`和`Broker`之间进行生产消息, 消费消息就可以了?

这个问题还是之前所讲到的, `Broker`是需要保证高可用的, 如果整个系统仅仅靠着一个`Broker`来维持的话, 那么`Broker`的压力过大, 会导致消息中间件服务的崩溃。所以我们需要使用多个`Broker`来保证**负载均衡。**

但是如果消费者和生成者直接和多个`Broker`相连, 那么当`Broker`修改的时候必定会影响到每个生成者和消费者, 这样又会产生耦合问题, 而`NameServer`注册中心就是解决这个问题。

### 2. RocketMQ集群架构

![rocket_cluster](/distributed/MQ/img/rocket_cluster.png)

1. `Broker`做了集群并还进行了主从部署, 由于消息分布在各个`Broker`上, 一旦某个`Broker`宕机, 则该`Broker`上的消息读写都会受到影响。所以`RocketMQ`提供了`maseter/slave`的结构, **`slave`定时从`master`同步数据, 如果`master`宕机, 则`slave`提供消费服务, 但是不能写入消息。**

2. 为了保证高可用, `NameServer`也是集群部署, 但是它是**去中心化**。`NameServer`集群没有组节点, 在`RocketMQ`中是通过**单个`Broker`和所有`NameServer`保持长连接**, 并且在每隔30秒`Broker`会向所有`NameServer`发送心跳, 心跳中包含了自身的`Topic`配置信息。

3. 在生产者需要向`Broker`发送消息时, **需要先从`NameServer`获取关于`Broker`的路由信息**, 然后通过**轮询**向每个队列生产数据到达**负载均衡**的效果。

4. 消费者通过`NameServer`获取所有`Broker`的路由信息后, 向`Broker`发送`Pull`请求来获取信息数据。`Consumer`可以通过**广播(Broadcast)和集群(Cluster)**获取数据。
广播模式下, 一条消息会发送给**同一个消费组中的所有消费者**, 集群模式下消息只会发送一个消费者。






