## <center>MySQL 分库分表</center>

> 为什么要分库分表?

当一张表的数据达到几千万时, 查询一次所花的时间会很长。这时如果有联合查询的话, 可能会卡死, 甚至会导致系统崩溃。所以产生数据库性能问题一般有下面三点:

- 数据量

    MySQL单库数据量在5000万以内性能比较好, 超过**阈值**后性能会随着数据量的增大而变弱。MySQL单表的数据量在500w-1000w之间性能较好, 超过1000w性能也会下降

- 磁盘

    单个服务的磁盘是有限制的, 如果并发压力下, 所以的请求都访问同一个节点, 肯定会对磁盘IO造成非常大的影响

- 数据库连接

    数据库连接是非常少的资源, 如果一个库中既有用户, 商品, 订单相关的数据, 那么海量用户同时操作时, 数据库连接很有可能消耗很快, 成为瓶颈。

分库分表的目的就是**减少数据库的负载, 提高数据库的效率, 缩短查询时间。**

`MySQL`执行一条sql的过程:
1. 收到sql
2. 把sql放到排队队列中
3. 执行sql
4. 返回结果

在这过程执行最花时间的地方在于: 排队等待的时间, sql的执行时间。

如果有2个sql都要同时修改同一张表的同一条数据, mysql对这种情况的处理是: 一种表锁定(MyISAM存储引擎), 一种是行锁定(InnoDB存储引擎)。

表锁定表示其他操作都不能对这张表进行操作,必须等当前对表的操作完成后才行。行锁定是等对这条数据操作完了, 其他人才能对这条数据进行操作。如果数据太多, 一次执行的时间太长, 等待的时间就越长, 这也是我们为什么要分表的原因。

### 1. 分库分表

#### 1.1 垂直拆分

- 垂直分库

    垂直分库针对的是一个系统中不同业务进行拆分, 比如用户一个库, 商品一个库, 订单一个库。但是如果垂直分库后还将这些库放在同一个服务器上, 只是分到了不同的库, 这样虽然会减少单库的压力, 但是随着用户量的增加, 整个数据库的处理能力还是会下降, 但是单个服务的磁盘空间, 内存也会大打折扣。所以要将多个库拆分到多个服务器上。

- 垂直分表

    简单说就是"大表拆小表"。基于表字段进行拆分。一般是表中的字段较多, 将不常用的, 数据较大, 长度较长(例如text类型字段)拆分到"扩展表"。

![mysql_chuizhichaifen](/database/MySQL/MySQL_Cluster/images/mysql_chuizhichaifen.png)

垂直拆分的优点很明显: **拆分后业务清晰, 拆分规则也很明确, 可以直接根据服务划分进行拆分; 数据维护也很简单**。

缺点: **部分业务表无法join, 只能通过接口方式解决, 提高了系统复杂度; 受每种业务不同的限制存在单库性能瓶颈, 不易数据扩展跟性能提高, 事务的处理上也比较复杂。**

由于垂直拆分是按照业务的分类将表分散到不同的库, 所以有些业务表会过于庞大, 存在单库读写与存储瓶颈, 所以就需要水平拆分来解决。

#### 1.2 水平拆分

- 水平分表

    这是基于全表的拆分, 将一个数据量很大的表, 分成多个小表, 减少单表的数据量, 提高查询效率。

- 水平分库分表

    将单张表的数据库切分到多个服务器上, 每个服务器具有相应的库与表, 只是表中数据集合不同。水平分库分表能够有效的缓解单机和单库的性能瓶颈和压力。

![mysql_shuipingchaifen](/database/MySQL/MySQL_Cluster/images/mysql_shuipingchaifen.png)

对于水平拆分可以理解为是按照数据行的拆分, 就是讲表中的某些行拆分到一个数据库, 而另外的某些行有拆分到其他的数据库中。

拆分数据就需要定义分片规则, 拆分的第一原则就是拆分维度。比如: 从会员表的角度来看, 商户订单表查询会员某天某月的某个订单, 那么就需要按照会员结合日期来拆分, 不同的数据按照会员ID做分组, 这样所有的数据查询join都会在单库内解决; 但是如果从商户的角度来看, 要查某个商家某天的所有订单数, 就需要按照商户ID来拆分; 但是如果系统既想按会员拆分, y又想按商户拆分, 一定会很困难, 所以需要找到合适的分片规则。

数据的拆分有优点, 也有缺点;

**优点:** 
- join操作基本可以通过数据库来完成; 
- 不存在单库大数据, 高并发的性能瓶颈; 
- 客户端改动较少; 
- 提交了性能的稳定性功能负载能力。

**缺点:** 
- 拆分规则难以抽象; 
- 分片事务一致性难以解决;
- 数据多次扩展难度跟维护量极大;
- 跨库join性能较差

#### 1.3 分库分表的策略

- 范围分片

    从[1, 10000]一张表, [100001, 200000]一张表

- 时间分片

    按照月份, 季度, 年份分片

- 地理位置分片

- Hash取模分片(重点)

    大部分情况进行水平拆分时, 数据的拆分都是通过Hash取模的策略。因为这种方式易于水平扩展。一般采用 `mode 2^n`这种一致性Hash。

    ![mysql_hashmod](/database/MySQL/MySQL_Cluster/images/mysql_hashmod.png)

    ![mysql_hashmod](/database/MySQL/MySQL_Cluster/images/mysql_hashmod.png)

    例如: 有一个订单库, 分库分表的方案为32*32, 也就是通过UserId的后四位取模32分到32个库中, 同时在将UserId后四位除以32然后再取模32将每个库分为32个表, 这样总分为1024张表, 线上部署分为8个集群, 每个集群4个库。

    那么为什么这种方式是易于水平扩展呢?

    1. 当数据库性能达到瓶颈时, 有两种方法:

        - 按照现有规则不变可以直接扩展到32个数据库集群, 就是一个库一个服务, 总共有32个库, 就直接可拆分为32个数据库集群, 一个库32张表。
        - 如果32个集群也无法满足要求, 那么将分库分表规则调整为`(32*2^n)*(32/2^n)`, 可以达到最多1024个集群。这种情况下, 最多就可以一个库一张表, 一个库一个集群, 也就是1024个集群。

    2. 单表容量达到瓶颈时(或者上面的1024张表都无法满足需求):

        如果单表都已经突破了极限, 那么此时分库的规则不变, 单库中的表在进行取模拆分, 因为只取最后四位取模, 所以最多可以拆分8192个表。

#### 1.4 解决思路

上面说了垂直拆分跟水平拆分的优缺点, 它们都有共同的缺点:

- 引入分布式事务的问题
- 跨节点join问题
- 跨节点合并排序分页问题
- 多数据源管理问题

对于数据源管理, 目前主要有两种方式:

1. 客户端模式, 在每个应用程序模块中配置管理自己需要的一个或多个数据源, 直接访问各个数据库, 在模块内完成数据的整合;
2. 通过中间代理层来统一管理所有的数据源, 后端数据库集群对前端应用程序透明。

大部分人在面对切分问题, 都会选择第二种方案。使用中间件让应用程序更加简单, 不用考虑很多问题, 而且易于扩展。

### 2. 分库分表工具

1. sharding-sphere

    Sharding-Sphere是一套开源的分布式数据库中间件解决方案组成的生态圈，它由Sharding-JDBC、Sharding-Proxy和Sharding-Sidecar这3款相互独立的产品组成。他们均提供标准化的数据分片、读写分离、柔性事务和数据治理功能，可适用于如Java同构、异构语言、容器、云原生等各种多样化的应用场景。

2. TDDL

    淘宝根据自己的业务特点开发的TDDL(Taobao Distributed Data Layer)主要解决了分库分表对应用的透明化以及异构数据库之间的数据复制，它是一个基于集中式配置的 jdbc datasource实现，具有主备，读写分离，动态数据库配置等功能。

3. Mycat

    基于阿里开源的Cobar产品研发的数据库中间件


