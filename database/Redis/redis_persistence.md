## Redis-持久化

我们知道Redis的数据是基于内存存储的, 所以一旦重启服务器, 就会导致数据丢失, 所以必须要通过持久化机制保证数据存储。

**Redis的持久化是将内存中的数据持久备份到硬盘中, 在服务重启时可以恢复之前的数据。**

Redis目前提供了两种持久化方式:

1. RDB, 即`Redis DataBase`: 把Redis服务中内存的数据保存到一个dump文件中, dump文件是数据的集合;

2. AOP, 即`Append-only file`: 把所有对Redis服务器进行修改的命令保存到一个AOF命令中, AOF文件是命令的集合。

### 1. RDB

RDB是Redis的内存快照, 它是在某一个时间点将Redis的内存数据全量写入一个临时dump文件, 当写入完成后, 用该临时文件替换上一次持久化生成的文件, 这样就完成了一次持久化过程。

#### 1.1 RDB相关配置

```shell
#dbfilename：持久化数据存储在本地的文件
dbfilename dump.rdb

#dir：持久化数据存储在本地的路径，如果是在/redis/src下启动的redis-cli，则数据会存储在当前src目录下
dir ./

##snapshot触发的时机，save <seconds> <changes>  
##一般来说我们需要根据系统变更操作密集程度来认真评估这个值
##可以通过 “save “””来关闭snapshot功能  
save 900 1
save 300 10
save 60 10000

##当snapshot时出现错误无法继续时，是否阻塞客户端“变更操作”，“错误”可能因为磁盘已满/磁盘故障/OS级别异常等 
stop-writes-on-bgsave-error yes

##是否启用rdb文件压缩，默认为“yes”，压缩往往意味着“额外的cpu消耗”，同时也意味这较小的文件尺寸以及较短的网络传输时间  
rdbcompression yes  
```

### 1.2 触发过程

**RDB持久化触发分为手动和自动两种方式。** 其中手动方式有两种, save命令和bgsave命令。

- **save命令:** 阻塞Redis服务, 直到整个RDB持久化完成。RDB是全量持久化, 如果内存的数据量大, 则造成长时间的阻塞, 这样会影响业务。所以一把不推荐采用这种方式。

- **bgsave命令:** 该模式下的RDB持久化由子进程完成Redis进程接收到该命令后, 会fork操作创建一个子进程, 持久化过程有子进程完成。具体流程如下:

    ![redis_persistence1](/image/redis_persistence1.png)

    1. 客户端发送bgsave命令, Redis进程首先判断当前是否存在其他子进程在执行操作, 如RDB或者AOF子进程。如果有, 则立刻返回, 否则执行步骤2;

    2. Redis进程执行fork操作创建子进程, 在fork操作时, Redis父进程会阻塞;

    3. Redis父进程fork操作完成后, bgsave命令返回`Backgroud saving started`信息并不再阻塞Redis父进程, 可以继续响应执行其他命令;

    4. fork的子进程则根据Redis父进程的内存数据生成RDB文件, 完成后替换原来的RDB文件。同时发送消息通知Redis父进程表示RDB操作已完成, 父进程则更新统计信息。

bgsave命令是手动触发RDB持久化的, 但是不可能每次都需要手动触发, 所以就有了自动触发方式:`save m n`。

**save m n**

该方式在redis.conf中进行了说明, m表示"间隔时间", n表示"变更次数"。只有同时符合这两个条件才会触发。

```shell

# 更改了1个key时间间隔900s进行持久化存储
save 900 1

# 更改了10个key时间间隔300s进行持久化存储
save 300 10

# 更改了10000个key时间间隔60s进行持久化存储
save 60 10000
```

#### 1.3 RDB数据恢复

RDB文件的载入工作是在服务器启动时自动加载的, 如果Redis没有蛇者AOF, 那么Redis在启动时就会检测RDB文件(redis-check-rdb命令), 并自动载入。

载入RDB文件过程中, Redis服务会一直处于阻塞状态, 知道完成为止。如果RDB文件损坏了, 则会载入失败, 那么Redis也会启动失败。

#### 1.4 RDB模式优缺点

- 优点:

    - 由于RDB文件是二进制文件, 所以加载速度快于AOF方式

    - fork子进程方式, 不会阻塞

    - RDB文件存储Redis某一时刻的全量数据, 所以它非常适合做数据备份和全量复制的场景

- 缺点:

    - 无法实时持久化, 就会存在数据丢失问题。定时执行持久化过程, 如果这个过程中服务器崩溃了, 会导致这段时间的数据全部丢失。

