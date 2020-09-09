## <center>MySQL binlog 详解</center>

### 1. MySQL操作binlog相关命令

MySQL5.7默认是不开启binlog日志的, binlog文件的位置可以再my.cnf配置文件中查看。也可以使用mysql的命令行查看。

```shell
show variables like '%log_bin%';
```

也可以查看当面mysql的binlog情况

```shell
show master status
```

每次我们重启MySQL服务就会自动生成一个binlog文件, 或者我们手动刷新binlog文件, 也会新创建一个binlog文件

```shell
flush logs
```

全部清空文件, 注意这是非常危险的操作, 一般是不会用的

```shell
reset master
```

### 2. 开启binlog日志

如果根据上面的命令发现binlog并未开启, 那么需要手动开启binlog日志。

找到MySQL服务的`my.cnf`主配置文件, 添加两行:

```shell
server-id=123456 # 这个服务id, 不能和其他集群中机器重复。
log-bin=/var/lib/mysql/mysql-bin # 这是指定binlog文件存放的路径
```

完成上面配置, 需要重启mysql服务 

```shell
service mysqld restart
```

启动成功之后, 我们可以再用上面的命令查看配置是否生效

```shell
show variables like '%log_bin%'
```

![binlog_start](/database/MySQL/MySQL_Cluster/images/binlog_start.png)

也可以定位到我们配置的文件目录下查看是否存在binlog文件