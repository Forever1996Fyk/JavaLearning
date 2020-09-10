## <center>MyCat 配置解析</center>

之前所说的MyCat相关概念几乎有在MyCat的配置行得以体现, 尤其是`schema.xml`, `server.xml`, `rule.xml`。

### 1. schema.xml

`schema.xml`是MyCat中重要的配置文件之一, 管理者MyCat的逻辑库, 表, 分片规则, DataNode以及DataSource。

#### 1.1 schema标签

```xml
<schema name="TESTDB" checkSQLschema="false" sqlMaxLimit="100">
    <table />
</schema>
```

schema标签用于定义MyCat实例中的逻辑库, MyCat可有多个逻辑库, 每个逻辑库都有自己相关配置。逻辑库的概念和MySQL数据库中的DataBase相同。

##### 1.1.1 dataNode

```xml
<schema name="TESTDB" checkSQLschema="false" sqlMaxLimit="100" dataNode="dn1">
    
</schema>
```

在1.3版本中如果`schema`配置了dataNode,则不能配置分片表, 而1.4版本可以。这个属性表示绑定所配置的具体database上, 可以直接访问这个database。

##### 1.1.2 checkSQLschema

当该值设置为true时, 如果我们执行语句

```sql
select * from TESTDB.mytable
```

则MyCat会把语句修改为

```sql
select * from mytable
```

也就是把表示schema的字符去掉, 避免发送到后端数据库执行时报错 **(ERROR testdb.mytable doesn't exist)**。

但是如果执行的语句所带的并不是schema指定的名字, 那么MyCat就会报错。

##### 1.1.3 sqlMaxLimit

当设置该参数时, 表示每条执行的SQL语句, 如果没有加上limit语句, MYCat也会自动加上所对应的值。如果不设置该参数, MyCat会默认查询所有的值。

**注意: 如果该schema逻辑库指定的并不是拆分库, 那么这个属性就不会生效, 需要手动添加limit语句**

#### 1.2 table标签

```xml
<table name="mytable" dataNode="dn1,dn2,dn3" rule="auto-sharding-long"></table>
```

Table标签定义了MyCat中的逻辑表。

##### 1.2.1 name属性

定义逻辑表的表名, 这个名字相当于在数据库执行`create table 'tableName'`命令一样, 同一个schema标签中定义的表名必须唯一。

##### 1.2.2 dataNode属性

逻辑表中的dataNode, 需要跟`dataNode`标签中name属性的值对应。如果需要定义的dn过多可以使用表达式

```xml
<table name="mytable" dataNode="multipleDn$0-99,mutipleDn2$100-199" rule="auto-sharding-long"></table>

<dataNode name="multipleDn" dataHost="localhost" database="db$0-99"></dataNode>
<dataNode name="multipleDn2" dataHost="localhost" database="db$0-99"></dataNode>
```

**注意: database属性所指的是真实的数据库名称, 上面的例子需要在真实的MySQL服务中创建名称为db0到db99的数据库**

##### 1.2.3 rule属性

该属性指定逻辑表要使用的规则名称, 规则名称定义在**rule.xml**中, 必须与tableRule标签 name属性对应。

##### 1.2.4 primaryKey属性

该逻辑表对应真实表的主键, 例如: 分片的规则是使用非主键进行分片的, 那么在使用主键查询时, 就会发送查询语句到所有配置的dataNode上, 如果使用该属性配置真实表的主键, 那么MyCat会缓存主键与具体的dataNode的信息, 那么再次使用非主键进行查询的时候就不会进行广播式的查询, 就会直接发送语句到具体的dataNode上。

##### 1.2.5 type属性

该属性定义了逻辑表的类型,目前逻辑表只有"全局表"和"普通表"。

- 全局表: global
- 普通表: 不指定global即可

#### 1.3 chlidTable标签

childTable标签用于定义E-R分片的子表, 通过标签上的属性与父表进行关联。

```xml
<table name="mytable" dataNode="dn1,dn2" rule="sharding-by-intfile">
    <childTable name="mychildtable" joinkey="mytable_id" parentKey="id" />
</table>
```
##### 1.3.1 joinKey属性

插入子表的时候回使用这个列的值查找父表存储的数据节点

##### 1.3.2 parentKey属性

该属性指定的值一般为与父表建立关联关系的列名。首先获取joinKey的值, 在通过parentKey属性执行的列名产生查询语句, 通过执行该语句得到父表存储在哪个分片上。从而确定子表存储的位置。

#### 1.4 dataNode标签

dataNode标签定义了MyCat中的数据节点, 也就是数据分片, 一个dataNode标签就是一个独立的数据分片。

```xml
<dataNode name="dn1" dataHost="localhost" database="db1"></dataNode>
```

##### 1.4.1 database属性

该属性定义该分片的对应的具体数据库实例上的具体数据库。因为每个库上建立的表和表结构是一样的, 所以这样可以轻松的对表进行水平拆分。

#### 1.5 dataHost标签

dataHost标签是schema.xml中最基本的标签, 直接定义了具体的数据库实例, 读写分离配置和心跳语句。

```xml
<dataHost name="localhost" maxCon="1000" minCon="10" balance="0" writeType="0" dbType="mysql" dbDriver="native">
    <heartbeat>select user()</heartbeat>

    <!-- 可以配置多个写节点 -->
    <writeHost host="hostM1" url="localhost:3306" user="root" password="root">
        <!-- 可以配置多个读节点 -->
        <readHost host="hostS1" url="localhost:3306" user="root" password="root" />
    </writeHost>
</dataHost>
```

##### 1.5.1 balance属性

负载均衡类型, 目前取值有4种:

1. `balance="0"`, 不开启读写分离机制, 所有读操作都发送到当前可用的writeHost上。
2. `balance="1"`, 全部的readHost与writeHost参与select语句的负载均衡, 也就是说, 双主双从模式(M1->S1, M2->S2, 并且M1与M2互为主备), 正常情况下, M2,S1,S2都参与select语句的负载均衡
3. `balance="2"`, 所有读操作都随机在readHost, writeHost上分发
4. `balance="3"`, 所有读请求随机的分发到 writerHost 对应的 readhost 执行，writerHost 不负担读压力, 注意 `balance=3` 只在 1.4 及其以后版本有，1.3 没有。

#### 1.6 heartbeat标签

这个标签指名用于和后端数据进行心跳检查的语句。

MySQl可以使用

```sql
select user()
```

Oracle可以使用

```sql
select 1 from dual
```

### 2. server.xml

`server.xml`保存了所有mycat需要的系统配置信息, 在源码中直接映射为`SystemConfig`类。主要用于定义登录MyCat的用户和权限。

### 3. rule.xml

`rule.xml`中定义了对表进行拆分所涉及到的规则定义。我们可以灵活对表使用不同的分片算法。

```xml
<tableRule name="rule1"> 
    <rule> 
        <columns>id</columns> 
        <algorithm>func1</algorithm> 
    </rule> 
</tableRule>
```

#### 3.1 tableRule标签

该标签定义表的规则, name属性指定唯一的规则名称, 内嵌的rule标签则指定对物理表中的哪一列进行拆分和使用什么路由算法。

columns内指定要拆分的列名字。

algorithm使用function标签中的name属性。

#### 3.2 function标签

```xml
<function name="hash-int" class="org.opencloudb.route.function.PartitionByFileMap"> 
    <property name="mapFile">partition-hash-int.txt</property>
</function>
```

name指定算法的名称, class指定路由算法具体类的全限名, property为具体算法需要的一些属性