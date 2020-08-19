## <center>Dubbo 集群</center>

> 为什么要部署集群?

为了避免单点故障, 应用通常至少会部署两台服务器上。那么就会出现一个问题, 服务消费者需要决定选择哪个服务提供者进行调用。而且服务调用失败时的处理措施也是必须考虑的。

**为了处理浙西问题, Dubbo定义了集群接口`Cluster`以及`Cluster Invoker`。集群`Cluster`用途是将多个服务提供者合并为一个`Cluster Invoker`, 并将这个`Invoker`暴露给服务消费者, 这样服务消费者只需通过这个`Invoker`进行远程调用即可, 至于具体调用哪个服务提供者, 以及调用失败之后如何处理等问题, 都无需关心。**

集群模块是服务提供者和服务消费者的中间层, 服务消费者只需专心处理远程调用相关事宜(例如: 发请求, 接受服务提供者返回的数据等)。

Dubbo提供多种集群实现, 包含但不限于`Failover Cluster`、`Failfast Cluster`和`Failsafe Cluster`等。

### 1. 集群容错

![集群](/distributed/RPC/img/cluster.jpg)

Dubbo主要提供5中容错方式:

- Failover Cluster - 失败自动切换
- Failfast Cluster - 快速失败
- Failsafe Cluster - 失败安全
- Failback Cluster - 失败自动恢复
- Forking Cluster - 并行调用多个服务提供者

### 2. 