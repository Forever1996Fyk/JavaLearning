## MySQL原理解析 -- 索引基础

**索引(index)** 是帮助MySQL高效获取数据的数据结构, 索引是有序的。

### 1. 为什么要用索引?

索引的目的在于提高查询效率, 利用索引我们可以很快的定位到所需要的数据位置。

> 

索引在MySQL是存储引擎用于快速找到记录的一种数据结构。

**这里要注意的是, MySQL的索引是一种数据结构(B Tree); 它跟数组中的索引有本质的区别, 不是通过索引号来查找数据。(再说了, 数组本身就是一种数据结构)**

索引优化应该是对查询性能优化最有效的手断了, 能够轻易的将查询性能提高好几个数量级。

![mysql_index1](/image/mysql_index1.png)

类似于使用二叉查找树查询数据。

### 2. 索引的数据结构

索引是在MySQL的存储引擎层中实现的, 所以每种存储引擎的索引不一定完全相同, 而且也不是所有的存储引擎都支持所有的索引类型。

MySQL目前提供了4种索引:

- **B-Tree索引:** 最常见的索引类型

- **Hash索引:** 只有Memory引擎支持, 使用场景简单

- **R-Tree空间索引:** MyISAM引擎的一个特殊索引, 主要用于地理空间数据类型, 使用较少;

- **Full-text全文索引:** 全文索引也是MyISAM的一个特殊索引类型，主要用于全文索引，InnoDB从Mysql5.6版本开始支持全文索引

|索引|InnoDB引擎|MyISAM引擎|Memory引擎|
|-- | -- |-- |-- |
| B-Tree索引 | 支持 | 支持 | 支持 |
| Hash索引 | 不支持 | 不支持 | 支持 |
| R-Tree索引 | 不支持 | 支持 | 不支持 |
| Full-text全文索引 | 5.6版本后支持 | 支持 | 不支持 |

一般情况下,默认都是B+树(多路搜索树)结构组织的索引。 其中**聚集索引, 复合索引, 前缀索引, 唯一索引**默认都是B+树索引。


> 每一种数据结构都不是凭空产生的, 在学习B+树之前, 我们一定要明白, MySQL需要什么样的数据结构?

其实很简单, 每次查找数据把磁盘IO次数控制在一个很小的数量级, 最好是常数数量级, 那么一个高度可控的多路搜索树正好可以解决这个问题, 也就是B+树。

![mysql_index2](/image/mysql_index2.png)

如上图, 是一颗B+树, 关于B+树的原理, 可以参考这篇文章[B Tree原理解析](database/MySQL/mysql_advanced/mysql_b_tree.md)。


### 3. 索引语法

#### 3.1 创建索引

语法:

```txt
create [unique|fulltext|spatial] index index_name [index_type] on table_name(col_name);
```

- index_name: 为索引命名;

- index_type: 索引数据类型, 默认就是btree。

- table_name: 表名

- col_name: 列名

- [unique|fulltext|spatial]: 索引类型, 默认就是unique



例如: 为city表的city_name字段创建索引;

```sql
create index idx_city_name on city(city_name);
```

#### 3.2 查看索引

语法:

```txt
show index from table_name;
```

例如: 查看city表的索引信息:

```sql
show index from city \G;
```

\G 是查询格式

#### 3.3 删除索引

语法:

```sql
drop index index_name on table_name;
```

- index_name 索引名称;

- table_name 表名;

#### 3.4 alter命令


1. alter table table_name add primary key(column_list);

    该语句添加一个主键, 也就是索引值必须是唯一的, 且不能为null

2. alter table table_name add unique index_name(column_list);

    该语句创建索引的值必须是唯一的, null可以重复

3. alter table table_name add index index_name(column_list);

    添加普通索引, 索引值可以出现多次

4. alter table table_name add fulltext index_name(column_list);

    添加全文索引
