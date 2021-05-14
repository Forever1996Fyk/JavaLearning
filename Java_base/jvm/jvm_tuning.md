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

2. 系统CPU经常100%, 如何调优? (面试高频)

    > 对于这种问题, 基本上步骤都是相同的, 只是命令有点区别!

        (1) 查找消耗CPU最高的进程PID
        (2) 根据进程ID查找消耗CPU最高的线程PID
        (3) 根据线程ID查出对于的Java程序, 在进行处理

    排查过程如下:

    - 使用`top`命令找出CPU占用最高的进程, 如果确实java程序引起的, 直接可以使用`jps`命令

    - 使用`top -H -p [进程pid]`命令查询进程的所有线程情况

        > -H 表示以线程的维度展示, 默认以进程维度展示, 有多种方式可以查看线程情况:
        > 1. top -Hp [pid] -d 1 -n 1
        >
        > 2. ps -Lfp [pid]
        >
        > 3. ps -mp [pid] -o THREAD, tid, time
        >
        > -p: 仅监视进程的id
        > -c: 显示命令行
        > -h: 当系统有多个CPU时, 个别CPU的状态信息被隐藏,只显示平均状态值
        > -d N: 显示两次刷新时间的间隔, 比如: -d 5, 标识两次刷新时间间隔为5s

    - 将查到的线程ID转换为16进制;

        > `printf "%x\n" [线程pid]`

    - 使用`jstack [线程pid] > [线程pid].tdump`将线程栈导出文件并使用`cat`命令查看

        > pid.tdump文件后缀名随意, 通常以tdump结尾

    - 使用`jstack [线程pid] | grep [16进制线程id]` 查看堆栈(栈帧)信息, 也就是定位到出问题的方法

    - 查看工作线程占比高, 还是垃圾回收线程占比高

3. 系统内存飙高, 如何查找问题? (面试高频)

    - 依然利用`top`命令查看内存占用情况; 如果确实java程序引起的, 直接可以使用`jps`命令;

    - 利用`jmap`命令, 导出堆内存情况

         1. jmap [pid]
        
         2. jmap -histo:live [pid] > a.log, 查看当前Java程序创建的活跃对象数和占用内存大小

         3. jamp -histo [pid] | head -20 , 查找有多少对象产生
         
         4. jamp -dump:live,format=b,file=xxx.bin [pid], 然后利用MAT, jprofile等工具查看是否存在内存泄露

        > jmap执行期间会对进程产生很大影响, 甚至卡顿, 有的项目场景并不适合。怎么解决?
        >
        > 1. 设定参数HeapDump, `-XX:+HeapDumpOnOutOfMemoryError` 在出现OOM时会自动产生dump文件; (不是很专业, 因为大项目一般多有监控, 内存增长就会报警)
        >
        > 2. 很多服务器备份, 是高可用的, 停掉这台服务器对其他服务器不影响;
        >
        > 3. 在线定位
        >
        > 4. 在测试环境中压测


4. 如何监控JVM

    - jstat, jvisualvm, jprofiler, arthas, top...

5. 利用`jinfo` 查看JVM对应的信息

6. `jstat -gc` 动态观察GC情况

    > `jstat -gc [pid] 500` : 每500ms打印GC情况

7. 如何定位OOM?

    > 不能回答使用图形界面, 例如MAT, jvisualvm等

    - 已上线的系统不能用图形界面, 因为需要暴露端口, 不安全。所以可以使用`arthas`。

    - 图形界面用在测试的时候进行监控! (压测观察)

8. 怎么找到死锁程序?

    - 通过`jps`定位java进程id

    - jstack定位线程情况:

        ```java
        WAITING BLOCKED 
        eg.
        waiting on <0x0000000088ca3310> (a java.lang.Object)
        ```

        加入有一个进程中100个线程, 很多线程都在 **`waiting on <xxx>`**, 一定要找到是哪个线程持有这把锁!!! 

        怎么找?

        **根据`jstack dump`信息, 找<xx>锁信息, 看哪个线程持有这把锁并且是`RUNNABLE`状态。**

### 5. 在线排查工具arthas

* thread定位线程问题
* dashboard 观察系统情况
* jad反编译
    * 动态代理生成类的问题定位
    * 第三方的类
    * 版本问题(确定自己最新提交的版本是不是被使用)
* redefine 热替换
* sc - search class 搜索类
* watch 监听方法的返回值







