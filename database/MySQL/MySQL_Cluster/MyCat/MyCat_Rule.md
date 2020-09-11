## <center>MyCat分片规则</center>

在数据切分处理中, 尤其是水平拆分, 中间件最终要处理的过程就是数据的切分, 数据的聚合。选择合适的切分规则, 很重要。

之前也了解了两条重要原则, 一个是数据冗余, 一个是表分组(Table Group), 这都是业务上规避跨库join的很好的方式。

### 1. 分片枚举

通过在配置文件中配置可能的枚举id, 自己配置分片。比如有些业务需要按照省份或者区县来保存, 而全国省份区县固定的, 这类业务使用本条规则。

```xml
<tableRule name="sharding-by-intfile">
    <rule>
        <columns>user_id</columns>
        <algorithm>hash-int</algorithm>
    </rule>
</tableRule>
<function name="hash-int" class="org.opencloudb.route.function.PartitionByFileMap">
    <property name="mapFile">partition-hash-int.txt</property>
    <property name="type">0</property>
    <property name="defaultNode">0</property>
</function>

```

partition-hash-int.txt 配置：
```text
10000=0

10010=1

DEFAULT_NODE=1
```

其中分片函数配置中, mapFile标识配置文件名称, type默认值为0, 0表示Integer, 非零表示String, 所有的节点配置都是从0开始, 0代表节点1。

### 2. 固定分片Hash算法

此规则是对二进制进行取模运算, 取id的二进制低10位, 即id二进制&1111111111。此算法的优点在于如果按照10进制取模运算, 在连续插入1-10时, 会被分到1-10个分片, 增大了插入的事务控制难度, 而此算法根据二进制则可能分到连续的分片,减少插入事务控制难度。

```xml
<tableRule name="rule1">
    <rule>
        <columns>user_id</columns>
        <algorithm>func1</algorithm>
    </rule>
</tableRule>

<function name="func1" class="org.opencloudb.route.function.PartitionByLong">
    <property name="partitionCount">2,1</property>
    <property name="partitionLength">256,512</property>
</function>
```

配置说明:

`columns`标识将要分片的表字段, `algorithm`分片函数,

`partitionCount`分片个数列表, `partitionLength`分片范围列表

分区长度: 默认为最大`z^n=1024`, 即最大支持1024分区

约束:

count,length两个数组的长度必须是一致的。

### 3. 范围约定

此分片适用于, 提前规划好分片字段某个范围属于哪个分片

`start <= range <= end`

```xml
<tableRule name="auto-sharding-long"> 
    <rule> 
        <columns>user_id</columns> 
        <algorithm>rang-long</algorithm> 
    </rule> 
</tableRule> 
<function name="rang-long" class="org.opencloudb.route.function.AutoPartitionByLong"> 
    <property name="mapFile">autopartition-long.txt</property> 
    <property name="defaultNode">0</property> 
</function>
```

autopartition-long.txt

```text
0-500M=0
500M-1000M=1
1000M-2000M=2

或者

0-10000000=0
10000001-20000000=1
```

defaultNode超过范围后的默认节点。所有节点配置都是从0开始, 并且0代表节点1.

### 4. 取模

这个规则是对分片字段取模运算

```xml
<tableRule name="mod-long">
    <rule>
        <columns>user_id</columns>
        <algorithm>mod-long</algorithm>
    </rule>
</tableRule>
<function name="mod-long" class="org.opencloudb.route.function.PartitionByMod">
    <!-- how many data nodes -->
    <property name="count">3</property>
</function>
```

### 5. 按日期(天)分片

```xml
<tableRule name="sharding-by-date">
    <rule>
        <columns>create_time</columns>
        <algorithm>sharding-by-date</algorithm>
    </rule>
</tableRule>

<function name="sharding-by-date" class="org.opencloudb.route.function.PartitionByDate">
    <property name="dateFormat">yyyy-MM-dd</property>
    <property name="sBeginDate">2014-01-01</property>
    <property name="sEndDate">2014-01-02</property>
    <property name="sPartionDay">10</property>
</function>

```

- dataFormat: 日期格式
- sBeginDate: 开始日期
- sEndDate: 结束日期
- SpartionDay: 分区天数, 寂寞人从开始日期算起, 每10天一个分区

如果配置了sEndDate,测表示数据达到了这个日期的分片后循环从开始分片插入

### 6. 一致性Hash

一致性Hash有效解决了分布式数据的扩容问题

```xml
<tableRule name="sharding-by-murmur">
    <rule>
        <columns>user_id</columns>
        <algorithm>murmur</algorithm>
    </rule>
</tableRule>
<function name="murmur" class="org.opencloudb.route.function.PartitionByMurmurHash">
    <property name="seed">0</property><!-- 默认是 0-->
    <property name="count">2</property><!-- 要分片的数据库节点数量，必须指定，否则没法分片-->
    <property name="virtualBucketTimes">160</property><!-- 一个实际的数据库节点被映射为这么多虚拟节点，默认是 160 倍，也就是虚拟节点数是物理节点数的 160 倍-->
    <!--
    <property name="weightMapFile">weightMapFile</property>
    节点的权重，没有指定权重的节点默认是 1。以 properties 文件的格式填写，以从 0 开始到 count-1 的整数值也就
    是节点索引为 key，以节点权重值为值。所有权重值必须是正整数，否则以 1 代替 -->
    <!--
    <property name="bucketMapPath">/etc/mycat/bucketMapPath</property>
    用于测试时观察各物理节点与虚拟节点的分布情况，如果指定了这个属性，会把虚拟节点的 murmur hash 值与物理节
    点的映射按行输出到这个文件，没有默认值，如果不指定，就不会输出任何东西 -->
</function>

```

### 7. 其他规则

MyCat提供了很多分片规则, 这里就不一一举例了。可以去MyCat的官网文档进行查看。