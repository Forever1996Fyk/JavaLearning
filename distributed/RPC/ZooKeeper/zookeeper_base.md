## <center>ZooKeeper 基础</center>

**ZooKeeper为我们提供了高可用, 高性能, 稳定的分布式数据一致性解决方案, 通常被用于实现数据发布/订阅, 负载均衡, 命名服务, 分布式协调/通知, 集群管理, Master选举, 分布式锁, 分布式队列等功能。**

而且, **ZooKeeper将数据保存在内存中, 性能很高。在"读操作"多于"写操作"的情况下, 尤其地高性能。因为"写操作"会导致所有的服务器都处于同步状态。**

### 1. ZooKeeper特点

- 顺序一致性: 从同一客户端发起的事务请求, 最终会严格的按照顺序被应用到ZooKeeper中。
- 原子性: 所有事务请求的处理结果在整个集群中所有机器上的应用情况是一致的, 也就是说, 要么整个集群所有机器都成功应用某一个事务, 要么都没应用。
- 单一系统映像: 无论客户端镰刀哪一个ZooKeeper服务器上, 看到的服务端数据模型都是一致的。
- 可靠性: 一旦一次更改请求被应用, 更改的结果就会被持久化, 直到被下一次更改覆盖。

### 2. ZooKeeper 典型应用场景

1. **分布式锁:** 通过创建唯一节点获得分布式锁, 当获得锁的一方执行相关代码或者是挂掉之后就释放锁
2. **命名服务:** 可以通过ZooKeeper的殊勋节点生成全局唯一ID
3. **数据发布/订阅:** 通过`Watcher机制`可以很方便的实现数据发布/订阅。当你将数据发布到ZooKeeper被监听的节点上, 其他机器可通过监听ZooKeeper上节点的变化来实现配置的动态更新。

这些功能的实现基本都得益于ZooKeeper可以保存数据的功能, 但是ZooKeeper不适合保存大量数据。

### 3. 哪些项目用到了ZooKeeper?

1. **Kafka:** ZooKeeper主要为Kafka提供Broker和Topic的注册以及多个Partition的负载均衡等功能
2. **Hbase:** ZooKeeper为Hbase提供确保整个集群只有一个Master以及保存和提供regionserver状态信息(是否在线)等功能。
2. **Hadoop:** ZooKeeper为Namenode提供高可用支持。

### <font color="red">4. ZooKeeper重要概念(*)</font>

#### 4.1 Data Model(数据模型)

ZooKeeper数据模型采用层次化的多叉树结构, 每个节点上可以存储数据, 这些数据可以是数字, 字符串或者是二进制序列。并且每个节点还可以拥有N个子节点, 最上层是根节点以"/"来代表。每个数据节点在ZooKeeper中被称为**znode**, 它是ZooKeeper中数据的最小单员, 每个**znode**都有一个唯一的路径标识。

**这里要注意的是: ZooKeeper主要是用来协调服务的, 而不是用来存储业务数据的, 所以znode是无法存储比较大的数据。ZooKeeper给出的上限是每个节点的数据大小最大是1M。**

![zookeeper_znode](/distributed/RPC/img/zookeeper_znode.png)


#### 4.2 znode(数据节点)

znode是ZooKeeper的最小单元。数据都存放在znode上。

##### 4.2.1 znode 4种类型

- **持久节点(persistent):** 一旦创建就一直存在即使ZooKeeper集群宕机, 除非执行删除操作。
- **临时节点(ephemeral):** 临时节点的生命周期与**客户端会话(session)**绑定, 会话消失那么节点消失。而且, **临时节点只能做叶子节点, 不能创建子节点。**
- **持久顺序节点(persistent_sequential):** 除了具有持久节点的特性之外, 子节点的名称还具有顺序性。比如: `/node1/app000001`, `/node1/app000002`。
- **临时顺序节点(ephemetal_sequential):** 除了具有临时节点的特性之外, 还具有顺序性。

##### 4.2.2 znode数据结构

每个znode由2部分组成:

- **stat:** 状态信息
- **data:** 节点存放的数据的具体内容

**stat状态信息的具体内容**

|znode 状态信息	| 解释 |
| --- | --- |
| cZxid |	create ZXID，即该数据节点被创建时的事务 id |
| ctime |	create time，即该节点的创建时间 |
| mZxid |	modified ZXID，即该节点最终一次更新时的事务 id |
| mtime |	modified time，即该节点最后一次的更新时间 |
| pZxid |	该节点的子节点列表最后一次修改时的事务 id，只有子节点列表变更才会更新 pZxid，子节点内容变更不会更新 |
| cversion |	子节点版本号，当前节点的子节点每次变化时值增加 1 |
| dataVersion |	数据节点内容版本号，节点创建时为 0，每更新一次节点内容(不管内容有无变化)该版本号的值增加 1 |
| aclVersion |	节点的 ACL 版本号，表示该节点 ACL 信息变更次数 |
| ephemeralOwner |	创建该临时节点的会话的 sessionId；如果当前节点为持久节点，则 ephemeralOwner=0 |
| dataLength |	数据节点内容长度 |
| numChildren |	当前节点的子节点个数 |

#### 4.3 ACL(权限控制)

ZooKeeper采用ACL(AccessControllLists)策略来进行权限控制, 类似于UNIX文件系统的权限控制。

对于znode操作的权限, ZooKeeper提供了5种:

- **create:** 能创建子节点
- **read:** 能获取节点数据和列出其子节点
- **write:** 能设置/更新节点数据
- **delete:** 能删除子节点
- **admin**: 能设置节点ACL的权限

其中**create, delete**这两种权限都是针对**子节点**的权限控制。

对于身份认证, 提供了4种方式:

- **world:** 默认方式, 所有用户都可无条件访问
- **auth:** 不使用任何id, 代表任何已认证的用户
- **digest:** 用户名, 密码认证方式
- **ip:** 对指定ip进行限制

#### <font color="red">4.4 Watcher(事件监听器*)</font>

`Wathcer(事件监听器)`是ZooKeeper中非常重要的特性。ZooKeeper允许用户在指定节点上注册Watcher, 并且在某些事件触发时, ZooKeeper服务端会将事件通知到对应的客户端上。

![zookeeper_watcher](/distributed/RPC/img/zookeeper_watcher.png)

#### 4.4 会话(Session)

Session可以看作是ZooKeeper服务器与客户端之间的一个TCP长连接, 通过这个连接, 客户端能够通过心跳检测与服务器保持有效的会话, 也能够向ZooKeeper服务器发送请求并接受响应, 同时还能够通过连接接收来自服务器的Watcher事件通知。

Session有一个属性是: `sessionTimeout`, 该属性表示会话超时时间。当服务器压力太大, 网络故障或是客户端主动断开连接等原因导致客户端连接断开时, 只要在`sessionTimeout`规定的时间内能够重新连接上集群中任意一台机器, 那么之前创建的会话依然有效。

在为客户端创建会话之前, 服务端会为每个客户端分配一个全局唯一的`sessionID`。

### 5. ZooKeeper集群

组成ZooKeeper服务的服务器都会在内存中维护当前的服务器状态, 并且每台服务器之间都相互保持通信。集群间通过ZAB协议(ZooKeeper Atomic Broadcast)来保持数据的一致性。

**最典型集群模式: Master/Slave模式(主备模式)。** 在这种模式中, 通常Master服务器作为主服务器提供写服务, 其他的Slave服务器从服务器通过异步复制的方式获取Master服务器最新的数据提供读服务。

#### 5.1 ZooKeeper集群角色

ZooKeeper不是传统的Master/Slave概念, 而是引入Leader, Follower和Observer三种角色。

![zookeeper_cluster](/distributed/RPC/img/zookeeper_cluster.png)

ZooKeeper集群中的所有机器通过一个**Leader选举过程**来选定一台称为"Leader"的机器, Leader既可以为客户端提供写服务又能提供读服务。Follower和Observer都只能提供读服务。Follower和Observer唯一的区别在于Observer机器不参与Leader的选举过程, 也不参与写操作的"过半写成功"策略。因此Observer机器可以在不影响写性能的情况下提升集群的读性能。

| 角色 | 说明 |
| --- | --- |
| Leader | 为客户端提供读写服务, 负责投票的发起和决议, 更新系统状态 |
| Follower | 为客户端提供读服务, 如果是写服务则转发给Leader。在选举过程中参与投票 |
| Observer | 为客户端提供读服务, 如果是写服务则转发给Leader。不参与选举过程的投票, 也不参与“过半写成功”策略。在不影响写性能的情况下提升集群的读性能。 |

**当Leader服务器出现网络中断, 崩溃退出与重启等异常情况时, 就会进入Leader选举过程, 这个过程会选举产生新的Leader服务器。**

#### 5.2 ZooKeeper集群中的服务器状态

- **looking:** 寻找Leader
- **leading:** Leader状态, 对应的节点为Leader
- **following:** Follower状态, 对应节点为Follower
- **observer:** Observer状态, 对应节点为Observer, 该节点不参与Leader选举

#### 5.3 ZooKeeper集群为什么最好是奇数台?

**ZooKeeper集群在几个服务器宕机情况下, 如果剩下的ZooKeeper服务器个数大于宕机的个数, 整个ZooKeeper才依然可用。**

假如集群中有n台ZooKeeper服务器, 也就是说剩下的服务数必须大于n/2。

例如: 我们有3台服务器, 那么最大允许宕机1台服务器, 如果我们有4台服务器也同样只允许宕机1台。如果我们有5台服务器, 那么最大允许宕机2台服务器, 如果我们有6台服务器也同样只能宕机2台。所以何必增加哪一个不必要的ZooKeeper服务器呢?

### 6 ZAB协议和Paxos算法

#### 6.1 ZAB协议介绍

ZAB(ZooKeeper Atomic Broadcast 原子广播)协议是分布式协调服务ZooKeeper专门设计的一种支持崩溃恢复的原子广播协议。**在ZooKeeper中, 主要依赖ZAB协议来实现分布式数据一致性。**

#### 6.2 ZAB协议两种基本模式

- **崩溃恢复:** 当整个服务框架在启动过程中, 或是当Leader服务器出现网络中断, 崩溃退出与重启等异常情况时, ZAB协议就会进入恢复模式并选举产生新的Leader服务器。
- **消息广播:** 当集群中已经有过半的Follower服务器完成了和Leader服务器的状态同步, 那么整个服务框架就可以进行消息广播模式。
