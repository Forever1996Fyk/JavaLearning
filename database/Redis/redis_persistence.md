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

### 2. AOF

**RDB最大的问题就是它提供的持久化策略不是实时的, 会存在数据丢失的问题。**

所以Redis提供了AOF方案。

`AOF`全称`append only file`, 它是将每一行对Redis数据进行修改的命令以独立日志的方式存储起来。所以我们可以直接利用该日志文件, 还原所有操作, 达到恢复数据的目的。**它的目的是解决了数据持久化的实时性。**

> AOF默认关闭, 需要在配置文件redis.conf中开启, `appendonly yes`。

redis.conf 中的AOF相关配置如下:

```conf
## aof功能的开关，默认为“no”，修改为 “yes” 开启  
## 只有在“yes”下，aof重写/文件同步等特性才会生效  
appendonly no  

## 指定aof文件名称  
appendfilename appendonly.aof  

## 指定aof操作中文件同步策略，有三个合法值：always everysec no,默认为everysec  
# appendfsync always  
 appendfsync everysec 
# appendfsync no 

##在aof-rewrite期间，appendfsync是否暂缓文件同步，"no"表示“不暂缓”，“yes”表示“暂缓”，默认为“no”  
no-appendfsync-on-rewrite no  

## aof文件rewrite触发的最小文件尺寸(mb,gb),只有大于此aof文件大于此尺寸是才会触发rewrite，默认“64mb”，建议“512mb”  
auto-aof-rewrite-min-size 64mb  

## 相对于“上一次”rewrite，本次rewrite触发时aof文件应该增长的百分比。  
## 每一次rewrite之后，redis都会记录下此时“新aof”文件的大小(例如A)，那么当aof文件增长到A*(1 + p)之后  
## 触发下一次rewrite，每一次aof记录的添加，都会检测当前aof文件的尺寸。  
auto-aof-rewrite-percentage 100  
```

#### 2.1 AOF执行流程

AOF总共分为三个流程:

1. 命令写入

2. 文件同步

3. 文件重写

#### 2.2 命令写入

Redis在命令写入时, 会先将命令写入到`aof_buf`缓冲区中, 因为Redis是单线程的, 如果每次直接追加到aof日志文件中就相当于进行一次磁盘IO, 那么性能肯定会受到影响。

#### 2.3 文件同步

命令写入`aof_buf`缓冲区, 然后根据不同的策略刷新到硬盘中。Redis提供了三种不同的不同策略: `always`, `everysec`, `no`, 由appendfsunc控制。

- `always`: 每天命令都会同步到硬盘中, 是最安全的方式, 但是对硬盘压力较大, 无法满足Redis高性能的要求, 所以一般不采用这种策略;

- `everysec`: 每秒同步一次, 这是redis推荐的方式, 但是如果遇到服务器故障, 可能会丢失最近2秒的记录;

    这里之所以最多会丢失2秒的记录, 因为当Redis主线程将AOF缓冲区的数据写入文件之前, 会与上次同步时间作对比, 如果大于2秒, 那么主线程就会进入阻塞状态。而此时如果Redis故障, 就会导致2秒内的数据丢失。

    ![redis_persistence2](/image/redis_persistence2.png)
    

- `no`: Redis服务并不会直接参与同步, 而是将控制权交给操作系统, 根据实际情况来触发同步, 但是这种方式不可控, 所以一般也不采用。

#### 2.4 文件重写

随着命令的不断写入, AOF文件会越来越大, 最终导致"数据恢复"的时间很长, 而且一些历史操作是可以废弃的(比如超时, del等), 所以redis提供了**文件重写**功能:

1. 手动触发: `bgrewriteaof`命令

2. 自动触发: 由`auto-aof-rewrite-min-size`和`auto-aof-rewrite-percentage`参数来确定自动触发时机。

    - `auto-aof-rewrite-min-size`: 运行AOF重写时文件最小体积;

    - `auto-aof-rewrite-percentage`: 当前AOF文件空间和上一次重写AOF文件空间的比值;

    - 触发时机 = 当前AOF文件空间 > AOF重写时文件最小体积

Redis主要重写AOF文件以下方面:

1. 已过期的数据不再写入文件;

2. 保留最终命令。例如: `set k1 v1`, `set k1 v2` ...  `set k1 vn`, 类似于这样的命令, 只会保留最后一个即可;

3. 删除无用的命令。例如: `set k1 v1`, `del k1`。这样的命令是可以不写入文件;

4. 多条命令合并成一条。 例如: `lpush list a`, `lpush list b`, `lpush list c`, 可以转化为`lpush list a b c`。

### 3. AOF优缺点

优点:

- 相对于RDB而言, AOF更加安全, 默认同步策略为`everysec`即每秒同步一次;

- AOF提供了不同的同步策略;

- AOF写入性能更高, 因为AOF是`append-only`文件追加方式写入, 而RDB是全量写入。

缺点:

- 由于AOF日志文件是命令级别的, 所以相对于RDB紧致的二进制文件加载速度更慢;

- AOF开启后, 支持写操作的QPS比RDB要低。

    > 因为在AOF重写阶段, 可能会导致Redis阻塞。例如在主进程将重写的缓冲区写入到新的AOF文件。

### 4. RDB 和 AOF混合模式

我们知道RDB和AOF都有各自的优缺点:

- RDB能够快速的存储和恢复数据, 但是服务器故障时可能会丢失大量的数据, 无法保证数据的实时性和安全性;

- AOF能够实时的持久化数据, 但是在存储和恢复数据时效率更低。

在Redis4.0推出了**RDB-AOF混合持久化**方案。

**RDB-AOF混合模式**的原理如下:

在AOF重写阶段会创建一个同时包含RDB数据和AOF数据的AOF文件, 其中RDB数据位于AOF文件的开头, RDB存储所有的数据状态, 重写操作执行后的Redis命令会继续追加在AOF文件末尾, 也就是RDB数据之后。

在Redis重启时, 可以先加载RDB的内容, 然后在加载AOF日志内容, 这样重启的效率更高, 而且在Redis运行阶段Redis命令又会以AOF方式追加到日志文件中, 保证了数据的实时性和安全性。

