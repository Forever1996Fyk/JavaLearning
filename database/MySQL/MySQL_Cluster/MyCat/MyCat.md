## <center>MyCat</center>

之前我们学习了, 当数据存储量过大, 超过了了服务器的负载时, 就会导致一系列的问题。然后提出了分库分表的概念, 并出现了很多相关的策略, 比如: 垂直切分, 水平切分。但是这些策略都需要对数据库进行切分。

> 什么是数据切分?

通过某种特定的条件, 将存放在同一个数据库中的数据分散存放到多个数据库(主机)上, 分散单台设备负载的效果。

> 什么是MyCat?

用官方的话来说, MyCat是一个开源的分布式数据库系统, 是一个实现了MySQL协议的服务器, 客户端用户可以看做是一个数据库代理。MyCat后面连接的是MySQL Server, 类似于存储引擎InnoDB, MyISAM等。所以, MyCat本身不存储数据, 数据还是存储在MySQL上, 数据可靠性以及事务等都是MySQL保证的。

### 1. MyCat原理

MyCat原理最重要的操作就是**拦截**。它拦截用户发送过来的SQL语句, 先对SQL语句进行特定的分析: 比如分片分析, 路由分析, 读写分离分析, 缓存分析等。然后将此SQL发往真实的数据库, 并将返回的结果做适当的处理, 最终再返回给用户。

![mycat_principle](/database/MySQL/MySQL_Cluster/images/mycat_principle.png)

上图中, Orders表被分为三个分片datanode(简称dn), 这三个分片分布在两台MySQL服务(DataHost)上, 即`datanode = database@dadahost`。每个分片的规则按照分片字段(sharding column) `prov`以及分片函数(rule function) `字符串枚举方式`。 

当MyCat拦截一个SQL时, 先解析这个SQL, 查找相关的表, 然后根据分片规则, 获取SQL中分片的字段值, 并且匹配分片函数, 然后将SQL发往这些分片执行, 最后收集和处理所有分片返回的数据结果, 并输出到客户端(即我们的应用程序)。 上图中的SQL, 其中`prov=wuhan`, 根据分片函数, wuhan返回dn1， 于是SQL就发送给`DB1@MySQL1`, 获取DB1上的查询结果, 并返回给客户端。

如果将SQL改为
```sql
select * from Orders where prov in ('wuhan', 'beijing')
```
那么SQL就会发给MySQL1, MySQL2执行, 然后结果集合并输出给客户端。但是问题来了, 一般业务中都会有`Order By`和`Limit`操作, 此时就需要在MyCat中进行二次处理。而最复杂的就是表的join问题, 因此MyCat有独特的**ER分片, 全局表**等解决方法。

### 2. MyCat中相关概念

#### 2.1 逻辑库(schema)

对于客户端开发人员来说, 并不需要知道中间件的存在, 他们只需要知道数据库即可, 所以MyCat数据库中间件可以认为是一个或者多个数据库集群构成的逻辑库。

#### 2.2 逻辑表(table)

既然有了逻辑库, 那么逻辑库中的表就是逻辑表。对于客户端来说, 读写数据的表就是逻辑表。逻辑表可以是数据切分后, 分布在一个或多个分片库中, 也可以不做数据切分, 只有一个表构成。

##### 2.2.1 分片表

分片表, 是那些原来恨到的数据表, 经过切分到多个数据库的表, 这样每个分片都有一部分数据, 所有的分片构成了完整的数据。

例如在MyCat中 `t_node`就属于分片表, 数据按照规则被分到dn1,dn2两个分片节点中。

```xml
<table name="t_node" primaryKey="ID" autoIncrement="true" dataNode="dn1,dn2" rule="rule1"/>
```

##### 2.2.2 非分片表

不是很大的数据表, 没有进行切分还是存储在一个数据表中。

```xml
<table name="t_node" primaryKey="ID" autoIncrement="true" dataNode="dn1" rule="rule1"/>
```

##### 2.2.3 ER表

**ER(Entity-Relationship)**表示实体关系, 基于ER关系的数据分片策略, 子表的记录与所关联的父表记录存放在同一个数据分片上, 也就是说子表依赖于父表, 通过表分组(Table Group)保证数据join不会跨库操作。

**表分组(Table Group)是解决跨分片数据join的一种很好的解决方案。**

##### 2.2.4 全局表

在实际的业务系统中, 一般存在大量的类似数据字典的表, 这些表都有几个特点:

- 变动不频繁
- 数据量总体变化不大
- 数据规模不大

对于这类表, 在分片的情况下, 这类表与业务边之间的关联, 就比较麻烦。所以MyCat通过数据冗余来解决这类表的join, 也就是所有分片都有一份数据的拷贝, 所以将这类表定义为全局表。

**数据冗余也是解决跨分片数据join的一种很好的解决方案。**

#### 2.3 分片节点(dataNode)

数据切分后, 一个大表被切分到不同的分片数据库上, 每个表分片所在的数据库就是分片节点(dataNode)。

#### 2.4 节点主机(dataHost)

一个或多个分片节点(dataNode)所在的机器就是节点主机(dataHost)。