## JVM 运行时数据区 Runtime Data Area

class文件通过类加载子系统将class信息, 存储到JVM的运行时数据区, 下面就把运行时数据区进行划分

### 1. Java内存空间

JVM内存布局规定了Java在运行过程中内存申请, 分配, 管理的策略, 保证JVM高效稳定运行。**不同的JVM对内存的划分方式和管理机制存在部分差异。(这里我们主要还是对Hotspot进行分析)**

![jvm_runtimedataarea](/image/jvm_runtimedataarea.png)

JDK8的元数据区+JIT编译产物, 就是JDK8之前的方法区的概念

### 2. Java内存分区简介

Java虚拟机规定了程序运行期间会使用到的运行时数据区, 其中一些会随着虚拟机启动而创建, 随着虚拟机退出而销毁。另一些则是与线程一一对应, 这些与线程对应的数据区域会随着线程开始和结束而创建和销毁。

如下图, 灰色区域为线程私有的, 红色区域为多线程共享的:

![jvm_runtimedataarea1](/image/jvm_runtimedataarea1.png)

### 3. Java中的线程与进程

- 每个线程: 都包括程序计数器, 栈, 本地方法栈;

- 线程共享: 堆, 堆外内存(方法区, 永久代或元空间, 代码缓存)

![jvm_runtimedataarea2](/image/jvm_runtimedataarea2.png)

> - PC: 程序计数器
> - VMS: 虚拟机栈
> - NMS: 本地方法栈

一般来说, JVM优化的95%是优化堆区, 5%优化的时方法区, 至于栈区优化很少。

对于线程来说:

- 线程时一个程序里的最小运行单元, JVM允许一个程序有多个线程并行执行;

- 在HotSpot JVM中, 每个线程都与操作系统的本地线程直接映射。

    >当一个Java线程准备好执行以后, 此时操作系统的本地线程也同时创建。Java线程执行终止, 本地线程也会回收。
