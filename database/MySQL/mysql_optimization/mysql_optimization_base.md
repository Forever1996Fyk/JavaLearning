## MySQL优化 -- 优化SQL基础

SQL语句的优化, 是开发人员必须掌握的技能,也是面试高频点。根据特定的场景与需求, 有特定的优化方式, 下面介绍的就是优化SQL的一些基本知识点。

### 1. 查看SQL执行频率

当与MySQL服务器连接成功时, 可以通过`show [session | global] status`命令查看服务器状态信息。

> `session`表示当前连接的信息, `global`表示所有连接统计信息, 如果不写, 默认是`session`。


* 下面的命令显示了当前Session中所有统计参数的值:

    ```sql
    show status like 'Com_______';
    ```

    ![mysql_optimization1](/image/mysql_optimization1.png)


* 下面的命令是查看innodb引擎数据库操作的行信息:

    ```sql
    show status like 'Innodb_rows_%';
    ```

    ![mysql_optimization2](/image/mysql_optimization2.png)

重点要知道下面参数的意义:

| 参数 | 含义 |
| -- | -- |
|Com_select | 执行 select 操作的次数，一次查询只累加 1。|
|Com_insert | 执行 INSERT 操作的次数，对于批量插入的 INSERT 操作，只累加一次。|
|Com_update | 执行 UPDATE 操作的次数。
|Com_delete | 执行 DELETE 操作的次数。
|Innodb_rows_read | select 查询返回的行数。
|Innodb_rows_inserted | 执行 INSERT 操作插入的行数。
|Innodb_rows_updated | 执行 UPDATE 操作更新的行数。
|Innodb_rows_deleted | 执行 DELETE 操作删除的行数。
|Connections | 试图连接 MySQL 服务器的次数。
|Uptime | 服务器工作时间。
|Slow_queries | 慢查询的次数。

### 2. 定位低效率执行SQL

可以有两种方式定位执行效率低下的SQL语句:

1. 慢查询日志: 记录查询超过一定时间的SQl语句到日志文件

    > 用`--log-slow-queries[=file_name]`选项启动, mysqld会记录包含所有执行时间超过`long_query_time`秒的SQL语句的日志文件。

2. `show processlist`: 可以实时的查看SQL执行情况

    > 慢查询日志只有在查询结束之后才记录, 利用`show processlist`命令查看当前MySQL正在进行的线程, 包括线程的状态, 是否锁表等；

    ![mysql_optimization3](/image/mysql_optimization3.png)

    - id列: 用户登录mysql时, 系统分配的"connection_id", 可以使用函数`connection_id()`查看;

    - user列: 显示当前用户。如果不是root, 只显示用户权限范围的sql语句;

    - host列: 显示这个语句从哪个ip哪个端口上发出, 跟踪出现问题语句的用户;

    - db列: 线程这个进程连接的数据库;

    - command列: 显示当前执行的名称, 一般为休眠(sleep), 查询(query), 连接(connect)等;

    - time列: 当前状态持续时间, 单位是秒;

    - state列: 显示使用当前连接的sql语句的状态。这是很重要的查看sql执行效率指标。以查询为例, 可能要经过
    `copying to tmp table`、`sorting result`、`sending data`等状态;

    - info列: 显示这个sql语句, 也是判断问题语句的重要依据。

### 3. explain分析

我们通过上面的方法可以查到执行效率低的SQL语句后, 在利用 **`explain`**命令获取执行select语句的信息, 例如:

```sql
explain select * from tb_item where id = 1;
```

![mysql_optimization4](/image/mysql_optimization4.png)


每个字段的含义如下:

| 字段 | 含义 |
| -- | -- |
| id | select查询的序列号, 表示查询执行select语句的顺序, id越大, 优先级越高 |
| select_type | 表示select的类型, 一般有`SIMPLE`(单表查询, 不使用表连接或子查询)、`PRIMARY`(主查询, 即最外层的查询)、`UNION`(union语句连接的第二个sql或者后面的sql)、`SUBQUERY`(子查询的第一个SQL)
| table | 输出结果集的表 |
| type | 表连接类型, 性能由好到差类型为(**system->const->eq_ref->ref->ref_or_null->index_merge->index_subquery->range->index->all**) |
| possible_keys | 查询时可能使用的索引 |
| key | 实际查询使用的索引 |
| key_len | 索引字段长度 |
| rows | 扫描行的数量 |
| extra | 执行情况的说明和描述 |

#### 3.1 explain之id

id字段表示select查询的顺序, 或者操作表的顺序。一般有三种情况:

1. id相同: 表示操作表的顺序是从上而下的;

2. id不同: id值越大, 优先级越高, 越先被执行;

3. id有相同, 也有不同: id相同可以认为是一组, 从上往下执行; 但是依然是id越大, 优先级越高;

#### 3.2 explain之select_type

- `SIMPLE`: 单表select查询, 不包含子查询, union

- `PRIMARY`: 查询中如果包含任何子查询, 最外层的查询标记为该标识;

    ```sql
    explain select u.* from t_user u where u.id in (select ur.user_id from user_role ur where ur.role_id = 1)
    ```

    ![mysql_optimization5](/image/mysql_optimization5.png)

- `SUBQUERY`: 在Select或者where中包含的子查询

- `DERIVED`: 在from列表中包含的子查询, **因为from中的子查询是把查询结果放到临时表中, 所以这些查询标记为`DERIVED`**

    ```sql
    explain select u.* from t_user u, (select * from t_role) r
    ```

    ![mysql_optimization6](/image/mysql_optimization6.png)

- `UNION` union之后的select语句, 被标记为UNION; 如果UNION包含在from的子查询中, 那么外层的Select会被标记为`DERIVED`

- `UNION RESULT`: 从UNION中获取结果的select


#### 3.3 explain之type

type显示访问类型, 是分析sql效率的重要指标:

- `NULL`: MySQL不访问任何表, 索引, 直接返回结果;

- `system`: 表只有一行记录(等于系统表), 这是const类型的特例, 一般不会出现

- `const`: 简单来说, **通过主键或者唯一索引只查询到一条数据**

    > const 用于比较`primary key`或者`unique`索引, 因为只匹配一行数据(主键与唯一索引都是唯一性)所以很快、

- `eq_ref`: 表示根据唯一索引, 进行关联查询, 只查出一条数据;

- `ref`: 表示根据非唯一性索引, 进行关联查询, 查出多条数据;

- `range`: where之后出现between, <, >, in等查询范围的操作标识;

- `index`: 遍历整个索引树的标识; 也就是说查询的数据正好是对应的索引列;

    ```sql
    explain select id from t_user;
    ```

    ![mysql_optimization7](/image/mysql_optimization7.png)

- `all`: 查询的数据既有索引列, 也有非索引列;

    password不是索引列

    ```sql
    explain select id, password from t_user;
    ```

    ![mysql_optimization8](/image/mysql_optimization8.png)

上面的结果从最好到最坏依次为:

NULL -> system -> const -> eq_ref -> ref -> range -> index -> all

**一般来说, 我们需要保证查询最多达到range级别, 最好达到ref级别.**

#### 3.4 explain之extra

extra是sql执行的额外信息。一般如下:

| extra | 含义 |
| -- | -- |
| using filesort | 说明mysql会对数据使用一个外部的索引排序, 而不是按照表内的索引顺序进行读取, 也就是**文件排序**, 效率很低 |
| using temporary | 使用临时表保存中间结果, MySQL对查询结果进行排序时使用临时表。例如: order by 和 grouy by; 效率低 |
| using index | 表示select操作使用了覆盖索引, 避免访问表的数据行, 效率高 |


### 4. show profile 分析 SQL

#### 4.1 开启profile

`show profiles` 能够在SQL执行后, 分析每个SQL执行的时间, 以及每个步骤所花费的事件。

通过下面的命令, 判断MySQL版本是否支持profile:

```sql
select @@having_profiling;
```

输出YES, 就表明是支持的。

但是默认profile是关闭的, 可以通过set语句在session级别开启profiling:

```sql
select @@profiling;
```

输出0, 表示关闭。

```sql
set profiling=1; //开启profiling;
```

#### 4.2 使用profile

通过profile, 我们嫩egg更清楚的了解SQL执行的过程以及花费时间。

- 首先我们可以先执行一系列的操作:

    ```sql
    show databases;

    use db01; 

    show tables; 

    select * from tb_item where id < 5; 

    select count(*) from tb_item;
    ```

- 然后在执行`show profiles`命令:

    ![mysql_optimization9](/image/mysql_optimization9.png)

- 通过`show profile for query [query_id]`语句可以查看SQL执行每个线程的消耗时间:

    ![mysql_optimization10](/image/mysql_optimization10.png)

    获取到最消耗时间的线程后, MySQL支持进一步选择 **`all`, `block io`, `context switch`, `page faults`** 等明细类型查看MySQL在什么资源商耗费过高的时间。例如选择查看CPU耗费时间:

    ![mysql_optimization11](/image/mysql_optimization11.png)


### 5. trace分析优化器执行计划

MySQL5.6提供了对SQL的跟踪trace, 通过trace文件能够进一步了解为什么优化器选择A计划, 而不是选择B计划。

- 开启trace; 

    ```sql
    set optimizer_trace="enabled=on", end_markers_in_json = on;
    ```

- 设置trace的格式为json; 

- 设置trace醉倒能够使用的内存大小, 避免解析过程中因为默认内存过小而不能够完整展示。

    ```sql
    set optimizer_trace_max_mem_size=1000000;
    ```

执行SQL语句:

```sql
select * from tb_item where id < 4;
```

最后检查`information_schema.optimizer_trace`就可以知道MySQL是如何执行SQL的:

```sql
select * from information_schema.optimizer_trace\G;
```

```json
*************************** 1. row *************************** QUERY: 
select * from tb_item where id < 4 
TRACE: { 
    "steps": [ 
        { 
            "join_preparation": { 
                "select#": 1, 
                "steps": [ 
                    { "expanded_query": "/* select#1 */ select `tb_item`.`id` AS `id`,`tb_item`.`title` AS `title`,`tb_item`.`price` AS `price`,`tb_item`.`num` AS `num`,`tb_item`.`categoryid` AS `categoryid`,`tb_item`.`status` AS `status`,`tb_item`.`sellerid` AS `sellerid`,`tb_item`.`createtime` AS `createtime`,`tb_item`.`updatetime` AS `updatetime` from `tb_item` where (`tb_item`.`id` < 4)" 
                    } 
                    ] /* steps */ 
                } /* join_preparation */ 
            },
            { "join_optimization": { 
                "select#": 1, 
                "steps": [ 
                    { 
                        "condition_processing": { 
                            "condition": "WHERE", 
                            "original_condition": "(`tb_item`.`id` < 4)", 
                            "steps": [ 
                                {"transformation": "equality_propagation", "resulting_condition": "(`tb_item`.`id` < 4)" 
                                },
                                { "transformation": "constant_propagation", "resulting_condition": "(`tb_item`.`id` < 4)" 
                                }
                            ]
                        }
                    }
                ]
            }
        }
    ]
}
```








