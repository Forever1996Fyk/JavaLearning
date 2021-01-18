## JVM调优--实战

### 1. 查看当前GC

我们通过`-XX:+PrintCommandLineFlags -version`查看当前Java的垃圾回收器。

常见的垃圾回收器组合为:

- `-XX:+USeSerialGC`: Serial New(DefNew) + Serial Old;

    > 小型程序。默认情况下不会是这种选项。

- `-XX:+UseParNewGC`: ParNew + Serial Old;

    > 这个组合很少使用(某些版本已废弃)

- `-XX:+UseConcMarkSweepGC`: ParNew + CMS + Serial Old

- `-XX:+USeParallelGC`: Parallel Scavenge + Parallel Old (1.8默认) [PS + PO]

- `-XX:+UseParallelOldGC`: Parallel Scavenge + Parallel Old

- `-XX:+UseG1GC`: G1

Linux中没有找到默认查看GC的命令, 而windows可以打印UseParallelGC

- 通过`java -XX:+PrintCommandLineFlags -version`
- 通过GC日志来分辨

### 2. JVM调优基本命令行参数

> 标准: - 开头, 所有的HotSPot都支持
>
> 非标准: -X 开头, 特定版本HotSpot支持特定命令
>
> 不稳定: -XX 开头, 下个版本可能取消

**基本命令:**

1. `java -XX:+PrintFlagsInitial` 默认参数值

2. `java -XX:+PrintFlagsFinal` 最终参数值

3. `java -XX+PrintFlagsFinal | grep xxx` 找到对应的参数

4. `java -XX:+PrintFlagsFinal -version | wc -l` 统计总共多少个参数(jdk1.8是728个)

### 3. JVM调优基础概念

> 所谓调优, 首先确定, 追求什么?
>
>吞吐量优先, 还是响应时间优先? 还是在满足一定的响应时间的情况下, 要求达到多大的吞吐量。

例如: 

后台的计算, 数据挖掘, 一般都是吞吐量优先: 选择PS+PO;

大型的电商网站, GUI界面, API的开发 一般都是响应时间优先: 选择G1(jdk1.8之后)

1. 吞吐量: 用户代码时间 / (用户代码执行时间 + 垃圾回收时间)

2. 响应时间: STW越短, 响应时间越好

### 3. 调优规划

- 根据适合的场景选择合适的垃圾回收器组合;

- 选定CPU(越高越好)

- 计算内存需求

- 设定分代年龄大小

- 观察GC日志情况

### 4. 优化服务器环境

1. 有一个50万PV的资料类网站（从磁盘提取文档到内存）原服务器32位，1.5G的堆，用户反馈网站比较缓慢，因此公司决定升级，新的服务器为64位，16G的堆内存，结果用户反馈卡顿十分严重，反而比以前效率更低了?

    - 为什么原网站变慢?

    > 很多用户浏览数去, 大量数据加载到内存, 内存不足, 频繁GC, STW时间长, 响应时间变慢

    - 为什么升级内存会更卡顿?

    >因为内存越大, Full GC 时间越长, STW时间长, 响应时间变慢

    - 如何解决?

    > 更换垃圾收集器组合。将 PS+PO收集器 更换为 ParNew + CMS 或者直接使用G1。
