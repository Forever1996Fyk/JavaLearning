## JVM堆--参数设置

下面是一些常用的堆空间的参数设置命令:

1. `-XX:PrintFlagsInitial`: 查看所有参数的默认初始值

2. `-XX:PrintFlagsFinal`: 查看所有参数的最终值(可能存在修改, 不是初始值)

    > 具体查看某个参数的指令:
    > - `jps`: 查看当前运行的进程;
    > - `jinfo -flag SurvivorRatio 进程id`: 查看年轻代中Eden和S0/S1空间的比例。

3. `-Xms`: 初始堆空间内存(默认为物理内存的1/64)

4. `-Xmx`: 最大堆空间内存(默认为物理内存的1/4)

5. `-Xmn`: 设置年轻代大小(初始值和最大值)

6. `-XX:NewRatio`: 设置年轻代与老年代在堆空间的占比

7. `-XX:SurvivorRatio`: 设置年轻代中Eden和S0/S1空间的比例

8. `-XX:MaxTenuringThreshold`: 设置年轻代幸存者对象的最大年龄(默认15)

9. `-XX:+PrintGCDetails`: 输出详细的GC处理日志

    - 打印GC简要信息: `-XX:PringGC`, `-verbose:gc`

10. `-XX:HanlePromotionFailure`: 是否设置空间分配担保

