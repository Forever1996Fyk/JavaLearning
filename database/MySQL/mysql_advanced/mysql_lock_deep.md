## MySQL原理解析 -- 锁机制

在数据库中, 数据基本上都是共享的方式提供出去, 既然是共享的, 很明显就会出现线程安全问题。**于是MySQL锁机制的出现, 解决数据并发访问的一致性, 有效性, 甚至对于锁本身也要解决锁冲突引发的并发访问性能问题。**

### 1. 锁分类

**从对数据操作的粒度分类:**

1. 表锁: 操作时, 会锁定整个表;

2. 行锁: 操作时, 会锁定当前操作行;

3. 间隙锁: 

**对数据操作的类型分类:**

1. 读锁(共享锁): 对同一条数据, 多个读操作可以同时进行而不会互相影响; 

2. 写锁(排它锁): 当前操作没有完成前, 会排斥其他获取写锁和读锁操作。

我们在学习多线程的锁`lock`时, 有一个`ReentrantReadWriteLock`就是读写锁的意思, 只有当所有的请求都是读操作时, 是不会阻塞其他线程; 但是只要存在写操作, 不管是读锁还是写锁, 都只能同时存在一个线程可以获取锁。

在MySQL中, 读锁与写锁的用法也是如此: 

读锁是共享锁, 只要是读操作就允许多个请求共同获取读锁; 但是如果是写操作, 则只会有一个请求获取锁, 其他请求必须等待锁的释放;

写锁是排它锁, 只要是获取到写锁, 其他请求不管是读操作还是写操作都只能等待写锁释放, 才能进行操作。


### 2. MySQL锁简介

MySQL的锁机制根据不同的存储引擎, 会支持不同的锁机制。

| 存储引擎 | 表级锁 | 行级锁 | 页面锁 |
| -- | -- | -- | --
| MyISAM | 支持 | 不支持 | 不支持 |
| InnoDB | 支持 | 支持 | 不支持 |
| Memory | 支持 | 不支持 | 不支持 |
| BDB | 支持 | 不支持 | 支持 |

MySQL这三种锁的特性大致如下:

| 锁类型 | 特点 |
| -- | -- |
| 表级锁 | 偏向MyISAM存储引擎, 开销小, 加锁快; 不会出现死锁, 锁定粒度大, 发生锁冲突的概率最高, 并发度最低 |
| 行级锁 | 偏向InnoDB存储引擎, 开销大, 加锁慢; 会出现死锁, 锁定粒度最小, 发生锁冲突的概率最低, 并发度最高 |
| 页面锁 | 开销和加锁时间界于表锁和行锁之间；会出现死锁；锁定粒度界于表锁和行锁之间，并发度一般。|

根据上面的简介, 其实很难确定那种锁更好。只能说根据不同的业务需求采用不同的锁机制。

**从锁的角度来说, 表级锁更合适以查询为主, 只有少量按索引条件更新数据的应用; 而行级锁更适合大量按索引条件并发更新不同数据, 同时有需要查询的场景。**

### 3. MyISAM 表锁

`MyISAM`存储引擎只支持表锁。在执行哈讯语句(select)前, 会自动给涉及的所有表加读锁, 在执行更新操作(update, install, delete)前, 会自动给涉及的表加写锁, 这个过程不需要用户干预, 因此用户一般不需要直接用`lock table`命令给`MyISAM`表显式加锁。

**在自动加锁的情况下, `MyISAM`会一次获得sql语句所需要的全部锁, 这也正是MyISA表不会出现死锁的原因。**

> 显式加锁语法:
> - 加读锁:
>```sql
lock table table_name read;
>```
>
> - 加写锁:
> ```sql
lock table table_name write;
>```

默认情况下, `MyISAM`的写锁比读锁优先级更高, 也就是说: 当一个锁释放时, 这个锁会优先给写锁队列中等待的线程获取锁请求, 然后在给读锁队列中获取锁请求。

所以`MyISAM`表不适合有大量更新操作和查询操作。因为获取写锁后, 其他线程不能做任何操作, 包括读取数据, 大量的更新会阻塞查询操作; 而长时间的查询操作, 也会使写线程"饿死"。

> 应用中应该尽量避免出现长时间的查询操作, 可以通过使用中间表等措施对SQL语句进行一定的分解, 每一段的查询都能在较短时间完成, 从而减少锁的冲突。如果复杂查询不可避免, 应该尽量安排在数据库空闲时段执行, 比如一些定期统计可以安排在夜间执行。


### 4. InnoDB 行锁与表锁


InnoDB与MyISAM的最大不同点:

1. InnoDB支持事务, 而MyISAM不支持

2. InnoDB支持表锁和行锁, MyISAM只支持表锁

InnoDB实现了两种类型的行锁:

- 共享锁(S): 又称为读锁, 简称S锁,共享锁就是多个事务对于同一数据可以共享一把锁，都能访问到数据，但是只能读不能修改; 如果当前事务需要对该记录进行更新操作, 则很有可能造成死锁。

- 排它锁(X): 又称为写锁, 简称X锁, 排他锁就是不能与其他锁并存，如一个事务获取了一个数据行的排他锁，其他事务就不能再获取该行的其他锁，包括共享锁和排他锁, 只能等待获取锁, 但是可以查询该记录;

#### 4.1 InnoDB加锁方式


对于`update`, `delete`, `insert`语句, InnoDB会自动给涉及数据集加上排它锁(X); 对于普通的select语句, InnoDB不会加任何锁。

加锁方式分为两种: 隐式锁定, 显示锁定。

- **隐式锁定:**

    InnoDB会根据事务隔离级别在需要的时候自动加锁, 而且锁只有执行commit或者rollback时才会释放。

- **可以显式的给记录集加共享锁或排它锁:**

    ```sql
    共享锁(S): select * from table_name where ... lock in share mode

    排它锁(X): select * from table_name where ... for update
    ```

    - **select for update:** 为了让自己查到的数据是最新数据, 并且查到后的数据只允许自己来修改时, 需要用到`for update`子句; 
    
        > 在业务繁忙时, 如果事务没有及时commit或rollback, 就会导致其他事务长时间等待, 从而影响数据库并发量;

    - **select lock in share mode:**  为了确保自己查到的是最新的数据, 并且不允许其他事务修改数据, 但是自己也不一定能够修改数据, 因为可能其他事务也获取了数据的S锁, 所以无法修改。

        > 在业务繁忙时, 如果给数据加上S锁, 但是不能够对数据进行修改, 如果不及时commit或者rollback, 也会造成大量事务等待。

#### 4.2 行锁升级为表锁

- 当我们使用行锁时, 如果不通过索引条件查询数据, 那么InnoDB会对表中所有行加锁, 实际效果就跟表锁一样, 我们称为**行锁升级为表锁**。

    > 这也说明了, **行锁一定是作用在索引上的。**

- 不论使用**主键索引, 唯一索引, 普通索引**, InnoDB都会使用行锁对数据加锁;

- 只有执行计划真正使用了索引, 才能使用行锁; 也就是说, 如果我们创建了索引, 也确实使用了索引, 但是由于使用不当导致索引失效, 如果MySQL进行全表扫描, InnoDB依然会使用表锁;

    > 这也说明了, 在分析SQL执行时(explain), 优化索引的重要性。

- 要注意的是, InnoDB的行锁是针对索引加的锁, 不是针对数据记录本身; 也就是说, 如果多个session连接访问不同行记录, 但是如果使用相同的索引列作为条件, 依然会导致锁冲突。(必须等待锁释放, 其他session才能获取锁)

#### 4.3. 间隙锁

当我们使用范围条件, 并使用共享锁或排它锁时, InnoDB会给符合条件的数据行加锁; 
而对于在这个范围内, 但是不存在的数据,叫做**间隙(GAP)**, InnoDB也会对这个间隙加锁, 也就是间隙锁(Next-Key锁)。

例如:

1. 对于session-1, session-2我们先关闭事务自动提交; 

2. 然后session-1根据id进行范围更新数据: id<4

    ![mysql_lock1](/image/mysql_lock1.png)

3. session-2插入id为2的记录, 此时id=2的记录, 并不存在于表中, 但是id=2依然在id<4的范围内, 此时处于阻塞状态;

    ![mysql_lock2](/image/mysql_lock2.png)

4. session-1提交事务;

5. session-2解除阻塞, 执行插入操作, 然后提交事务。

很显然, 在使用范围条件查询或者更新锁定记录时, InnoDB这种"间隙锁"机制, 会阻塞符合条件范围内的并发插入, 这会导致严重的锁等待, 所以在实际开发中, 优化业务逻辑, 尽量使用相等条件在访问更新数据, 避免使用范围条件。

例如下面的一段sql:

```sql
以id为主键为例，目前还没有id=22的行
 
Session1:
select * from t3 where id=22 for update;
Empty set (0.00 sec)
 
session2:
select * from t3 where id=23  for update;
Empty set (0.00 sec)
 
Session1:
insert into t3 values(22,'ac','a',now());
锁等待中……
 
Session2:
insert into t3 values(23,'bc','b',now());
ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction

```

上面的例子可以看到, 我们给id=22,23不存在的行加上排它锁, 依然会出现死锁, 因为存在间隙锁。

锁住的范围定义如下:

- 如果表中目前已存在id为(11, 12), 那么就锁住(12, 无穷大)的行数据

- 如果表中已存在(11, 30), 那么就锁住(11, 30)的行数据

对于这种死锁的解决办法是:

```sql
insert into t3(xx,xx) on duplicate key update `xx`='XX';
```

利用mysql特有的语法来解决问题。**因为insert语句对于主键来说, 插入的行不管是否存在, 都只有行锁。**

#### 4.4 InnoDB使用间隙锁的目的

InnoDB使用间隙锁的目的主要有两点:

1. 防止幻读, 满足相关隔离级别的要求;

2. 满足恢复和复制的需要;

**MySQL通过`binlog`日志录入执行成功的insert, update, delete等更新数据的语句, 并实现mysql数据库的恢复和主从复制。**

MySQL的恢复机制(主从复制就是在从数据库(slave mysql)不断的基于`binlog`的恢复)基本原理:

1. 重新执行binlog中的sql语句;

2. mysql的binlog是按照事务提交的先后顺序记录的, 恢复也是按照这个顺序执行。

所以, MySQl的恢复机制要求: 在一个事务未提交前, 其他事务不能插入任何记录, 也就是不允许出现幻读。

#### 4.5 InnoDB在不同隔离级别下的一致性

对于许多的SQL, 隔离级别越高, InnoDB给数据行加的锁越严格, 产生锁冲突的可能性就越高, 从而并发性事务处理性能就会降低。

因此, 我们在应用中, 应该尽量使用较低的隔离级别, 来减少锁争用的几率。一般情况下, 通过优化业务逻辑, 大部分情况使用 **`Read Commited`** 隔离级别就足够了。

#### 4.6 获取InnoDB行锁争用情况

可以通过检查InnoDB_row_lock 状态变量来分析系统上的行锁争夺情况:

```sql
show status like 'innodb_row_lock%';
```
### 5. 死锁

MySQL的死锁与多线程开发中出现死锁基本一致:

- **当多个事务在同一资源上相互占用, 并且都在等待对象释放资源, 就会导致死锁。**

#### 5.1 检查死锁

我们可以通过`show engine innodb status`查看死锁日志。

```txt
//事务1相关信息
*** (1) TRANSACTION:
TRANSACTION 50E, ACTIVE 66 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 3 lock struct(s), heap size 376, 2 row lock(s)
MySQL thread id 7, OS thread handle 0x27448, query id 82 localhost 127.0.0.1 root Updating
//当前事务正在执行的sql语句
update tb1 set c1= 10 where id =5
//以下信息记录了锁等待信息
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
//正在申请主键索引行记录的x锁
RECORD LOCKS space id 0 page no 3328 n bits 72 index `PRIMARY` of table `test`.`tb1` trx id 50E lock_mode X locks rec but not gap waiting
Record lock, heap no 5 PHYSICAL RECORD: n_fields 5; compact format; info bits 0

//事务2相关信息
*** (2) TRANSACTION:
TRANSACTION 50F, ACTIVE 47 sec starting index read
mysql tables in use 1, locked 1
3 lock struct(s), heap size 376, 2 row lock(s)
MySQL thread id 8, OS thread handle 0x277e4, query id 83 localhost 127.0.0.1 root Updating
update tb1 set c1= 10 where id =5
//正在持有的锁：主键索引为5的行记录级别的S锁
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 0 page no 3328 n bits 72 index `PRIMARY` of table `test`.`tb1` trx id 50F lock mode S locks rec but not gap
Record lock, heap no 5 PHYSICAL RECORD: n_fields 5; compact format; info bits 0
 0: len 4; hex 80000005; asc     ;;
 1: len 6; hex 00000000050d; asc       ;;
 2: len 7; hex 8b00000d080110; asc        ;;
 3: len 4; hex 80000005; asc     ;;
 4: len 4; hex 80000005; asc     ;;

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 0 page no 3328 n bits 72 index `PRIMARY` of table `test`.`tb1` trx id 50F lock_mode X locks rec but not gap waiting
Record lock, heap no 5 PHYSICAL RECORD: n_fields 5; compact format; info bits 0
 0: len 4; hex 80000005; asc     ;;
 1: len 6; hex 00000000050d; asc       ;;
 2: len 7; hex 8b00000d080110; asc        ;;
 3: len 4; hex 80000005; asc     ;;
 4: len 4; hex 80000005; asc     ;;

WE ROLL BACK TRANSACTION (2)
```

InnoDB存储引擎能检测到死锁的循环依赖, 并立即返回一个错误

#### 5.2 解决死锁

死锁发生后, 只有提交或者回滚部分事务, 才能打破死锁。更加直接一点的话, 可能

#### 5.3 InnoDB避免死锁

- 使用`select * for update`添加排它锁(X);

- 在事务中如果要更新数据, 应该直接申请排它锁, 而不应先申请共享锁、更新时再申请排他锁，因为这时候当用户再申请排他锁时，其他事务可能又已经获得了相同记录的共享锁，从而造成锁冲突，甚至死锁;

- 改变事务隔离级别(一般情况下不可能轻易的修改隔离级别)