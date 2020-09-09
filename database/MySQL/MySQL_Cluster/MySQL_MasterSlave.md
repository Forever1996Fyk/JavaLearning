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

主从节点的版本最好是一样的, 5.7 或者8.x的版本

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

    ```shell
    [mysqld]
    # 注意这个server-id需要唯一
    server-id=1
    # 开启二进制日志功能(必须)
    log-bin=mysql-bin
    ```

     具体参考[binlog详解](/database/MySQL/MySQL_Cluster/MySQL_Binlog.md)

4. 重启Master服务

    ```shell
    service mysql restart
    ```

#### 2.2 配置Slave节点

在从数据库中进行配置:

1. 修改`my.inf`配置文件

    ```shell
    [mysqld]
    server-id=2
    relay-log-index=relay-log-bin.index
    relay-log=relay-bin
    ```

    配置完成之后需要重启mysql服务

2. 连接Master节点

    进入从节点的mysql环境, 执行下面的命令

    ```shell
    change master to master_host='192.168.100.48',master_port=3306,master_user='repl',master_password='repl',master_log_file='master-bin.000001',
    master_log_pos=0;
    ```
    解释一下这个命令: 

    - `master_host`: 主节点的host地址
    - `master_port`: 主节点的端口号, 一般默认是3306
    - `master_user`: 主节点拥有`replication slave`权限的用户名
    - `master_password`: 上面对应用户的密码
    - `master_log_file`: 指定主节点开启的binlog日志
    - `master_log_pos`: 指定对应的主节点binlog日志的指令位置

    上面的命令如果执行错误的话, 主要有两点

    - 主节点连接不上, 需要检查主节点的配置, 尤其是用户名密码, 权限等问题
    - 执行binlog日志错误, 一般是指定的`master_log_file`错误, 或者是`master_log_pos`的位置错误, 此时需要查看主节点的binlog日志, 找到需要执行的position数值.

3. 开启slave

    ```shell
    start slave
    ```

4. 查看状态

    ```shell
    show slave status \G
    ```

    查看状态主要是关注两个地方: `Slave_IO_Running`, `Slave_SQL_Runing`。这两个都为yes才可以启动从节点。具体出错的原因, 会有日志提示

### 3. Docker环境下的主从配置

如果实在Docker环境下配置MySQL主从节点, 需要进入docker的容器的内部, 然后具体配置跟上面一样。

#### 3.1 配置Master

1. 通过命令进入Master容器内部 

    ```shell
    docker exec -it 容器id/name /bin/bash
    ```
2. 切换到`/etc/mysql`目录下, 使用`vi my.cnf`进行编辑。此时会报出`bash:vi: command not found`, 需要在docker容器内部安装vim。

    ```shell
    apt-get install vim
    ```
3. 如果第二步出错, `unable to locate package vim`, 那么执行下面的命令

    ```shell
    apt-get update

    apt-get install vim
    ```

4. 配置`my.cnf`, 重启MySQL服务, `service mysql restart`

5. 创建用户

6. 授权

#### 3.2 配置Slave

这个操作跟Master节点一样
