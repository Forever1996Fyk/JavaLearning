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
