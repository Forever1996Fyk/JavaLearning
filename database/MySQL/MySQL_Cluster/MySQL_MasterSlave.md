## <center>MySQL 主从复制</center>

> 为什么需要MySQL主从复制?

简单来说, MySQL主从复制主要有两点作用: 
1. 数据的备份: 如果主数据库宕机, 可以快速将业务系统切换到从数据库上, 可以避免数据丢失
2. 提高系统性能: 对数据库的读写都在同一个数据库服务器操作, 业务系统性能会降低。所以才用主从复制的架构, 主节点上只执行写操作, 从节点上只执行读操作。

> 对于上面的第二点, 我们知道对于事务的操作，是针对写操作的, 读操作是不需要事务的。但是如果读写都在一个数据库中, 那么读写操作都受到了事务的管理, 这样对系统的性能是大大降低的。所以采用主从复制来实现读写分离。

### 1. 主从复制(读写分离)的原理

`MySQL`的主从复制实现的原理就是 **`binlog日志`**, 那么我们主节点负责数据库写操作,而从节点负责读操作, 这样在从节点上不需要使用事务, 能够大大提高数据库的性能。那么这个时候面临的问题就是从节点如何来同步主节点数据的问题, 就用到了 **`binlog日志`**。

这个不光是 **`binlog日志`**, `MySQL`的的很多日志信息都是非常重要的, 比如: 慢查询日志等等。这个后面会慢慢学习。

### 2. 主从节点配置

#### 2.1 配置Master节点

在主数据库服务中, 需要进行数据库配置:

1. 创建用户并授权, 比如新建用户`repl`。 SQL语句为: `create user 'username' identified by 'password'`

    ```sql
        create user 'repl' identified by `repl`
    ```

2. 授权。用户必须具备`replication slave`权限, 除此之外不需要其他的权限。SQL语句为: `grant replication slave on on *.* to 'username'@'host' identified by 'password'`

    ```sql
        grant replication slave on on *.* to 'repl'@'192.168.100.%' identified by 'repl'
    ```

3. 开启 **`binlog`日志**

具体参考[binlog详解](/database/MySQL/MySQL_Cluster/MySQL_Binlog.md)

#### 2.2 配置Slave节点

在从数据库中进行配置:

1. 修改`my.inf`配置文件

```vim
[mysqld]
server-id=2
relay-log-index=relay-log-bin.index
relay-log=relay-bin
```