## MySQL原理解析 -- 索引

**索引(index)** 是帮助MySQL高效获取数据的数据结构, 索引是有序的。

### 1. 为什么要用索引?

索引的目的在于提高查询效率, 利用索引我们可以很快的定位到所需要的数据位置。

> 

索引在MySQL是存储引擎用于快速找到记录的一种数据结构。

**这里要注意的是, MySQL的索引是一种数据结构(B Tree); 它跟数组中的索引有本质的区别, 不是通过索引号来查找数据。(再说了, 数组本身就是一种数据结构)**

索引优化应该是对查询性能优化最有效的手断了, 能够轻易的将查询性能提高好几个数量级。

![mysql_index1](/image/mysql_index1.png)

类似于使用二叉查找树查询数据。

### 2. 索引的原理

索引的本质是**通过不断的缩小想要获取数据的范围来筛选最终的结果。**

但是数据库要复杂的很多, 因为不仅面临等值查询, 还有范围查询(>, <, between, in)、模糊查询(like), 并集查询(or)等等。所以数据库也是采用分段查询。

> 最简单的如果1000条数据, 1到100分成第一段, 101到200分为第二段, 201到300分为第三段... 这样查询第250条数据, 只需要找第三段就可以了, 一下去除了90%的无效数据。
>
> 但是如果是1000万的数据, 应该怎么分?
> 
> 我们可能会想到搜索树, 平均复杂度是lgN, 查询性能也很不多错。但是数据库实现比较复杂。
> 
> **数据是保存在磁盘上的, 因为一般的搜索树(二叉查找树, 红黑树等)的深度过大, 而造成磁盘IO读写过于频繁, 导致效率低下。**

### 3. 索引的数据结构

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

### 4. 索引的分类

索引类型可以分为5种:

1. 普通索引index: 加速查找

2. 唯一索引

    - 主键索引: primary key: 加速查找+约束(不能为空且唯一), 这也是默认的索引;
    - 唯一索引: unique: 加速查找+约束(可以有多个null, 且唯一)

3. 联合索引

    - primary key(id, name): 联合主键索引

    - unique(id, name): 联合唯一索引

    - index(id, name): 联合普通索引

4. 全文索引full text: 一般用于搜索很长的文章时, 效果更好

5. 空间索引: 一般不用

### 5. 索引语法

#### 5.1 创建索引

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

#### 5.2 查看索引

语法:

```txt
show index from table_name;
```

例如: 查看city表的索引信息:

```sql
show index from city \G;
```

\G 是查询格式

#### 5.3 删除索引

语法:

```sql
drop index index_name on table_name;
```

- index_name 索引名称;

- table_name 表名;

#### 5.4 alter命令


1. alter table table_name add primary key(column_list);

    该语句添加一个主键, 也就是索引值必须是唯一的, 且不能为null

2. alter table table_name add unique index_name(column_list);

    该语句创建索引的值必须是唯一的, null可以重复

3. alter table table_name add index index_name(column_list);

    添加普通索引, 索引值可以出现多次

4. alter table table_name add fulltext index_name(column_list);

    添加全文索引
