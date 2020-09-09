## <center>MySQL 主主复制</center>

MySQL的主主复制就是两台节点互为主从。它是在MySQL主从的基础上搭建的, 只要搭建好MySQL主从, 在搭建主主复制就非常简单了。

在原来的主从的基础上操作:

1. 开启原从节点binlog日志
2. 原从节点创建权限用户
3. 原主节点的master指向从节点
4. 在原主节点执行start slave命令