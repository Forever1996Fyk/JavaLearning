## JVM简介

### 1. 什么是JVM?

JVM全称`Java Virtual Machine`, Java虚拟机。由一套**字节码指令集, 寄存器, 栈, 垃圾回收堆, 存储方法域**等组成。

JVM屏蔽了与操作系统平台相关的信息, 使得Java程序只需要生成在Java虚拟机上运行的字节码, 就可以多种平台无需修改的运行, 这就是Java**一次编译, 到处运行**的原因。

JVM在操作系统中的地位:

![jvm_os](/image/jvm_os.png)

### 2. JRE、 JDK、 JVM之间的关系

- **`JRE(Java Runtime Environment, Java运行环境)`** 是Java平台, 所有程序都要在JRE下才能运行。包括JVM和Java核心类库和支持文件。

- **`JDK(Java Development Kit, Java开发工具包)`** 是用来编译, 调试Java程序的开发工具包。包括Java工具(javac/java/jdb等)和Java基础类库(Java API)。

- **`JVM(Java Virtual Machine, Java虚拟机)`** 是JRE的一部分。JVM主要工作是解释自己的指令集, 也就是字节码, 并映射到本地的CPU指令集和OS的系统调用。Java语言是跨平台运行的, 不同的操作系统会有不同的JVM映射规则, 所以与操作系统无关, 完全跨平台性。

> 这三者, JDK是Java开发的核心, 是一层层嵌套的关系。**JDK > JRE > JVM**

下图表示了JDK, JRE, JVM三者间的关系:

![jdk_jre_jvm](/image/jdk_jre_jvm.png)

### 3. JVM基本结构

![jvm_structure](/image/jvm_structure.png)

- 方法区和堆区是所有线程共享的内存区域;

- Java栈(Java Stack), 本地方法栈(Native Method Stack), 程序计数器(Program Counter Register)是线程私有的内存区域;(**其实这个在并发编程的文章中, 提到了Java内存模型中提到了。私有变量不会产生线程安全问题, 就是因为私有变量存放在Java栈中, 而程序计数器就是在线程上下文切换时保存上次执行的指令地址。**)

- <font color="red">Java栈又称为JVM虚拟机栈</font>

- **方法区, 在JDK8之前被称为永久代, JDK8之后又叫做元空间Metaspace**

    - 方法区用于存储被虚拟机加载的类型, 常量, 静态变量, 即时编译器(JIT编译器, Just-In-Time Compiler)编译后的代码等数据;

    - 虽然Java虚拟机规范把方法区描述为堆的一个逻辑部分, 但是它有一个别名叫做`Non-Heap`(非堆), 目的应该是为了与Java堆内存区分开。

> Java代码大致执行流程是: <vr>
> Java源程序(.java文件) ---> 通过编译命令javac ---> 字节码文件(.class) ---> 类装载子系统生成反射类, 也就是存入方法区 ---> 运行时数据区(上图的五部分) ---> 执行引擎(解释和即时编译(JIT)) ---> 操作系统(Win, Linux, Mac JVM)

#### 3.1 栈的指令集架构和寄存器的指令集架构

由于是跨平台的设计, java的指令都是根据栈来设计的, 因为不同平台CPU架构不同, 所以不能设计为基于寄存器的。

二者的区别:

- `栈`: 跨平台性, 指令集小, 指令多, 执行效率比寄存器差;
- `寄存器`: 指令很少, 每个平台有不同的对应指令。

一些简单的查看命令:

```java
// 反编译
javap -V xxx.class

// 打印程序执行的进程
jps
```

#### 3.2 Hotspot中方法去的变动

关于方法区的结构, 再过去的版本jdk1.6/1.7/1.8都有变动:

- **jdk1.6以及之前的版本**: 存在永久代(permanent generation), 静态变量, 字符串常量池存放在永久代上;

- **jdk1.7**: 有永久代, 但是已经开始"<font color="red">去永久代</font>", 字符串常量池, 静态变量保存在堆内存中;

- **jdk1.8以及之后的版本**: 无永久代, 类型信息, 字段, 方法, 常量保存在本地内存的元空间, 但字符串常量池, 静态变量保存在堆内存。

![jdk6&7](/image/jdk6&7.png)

![jdk8](/image/jdk8.png)

要注意的是: **jdk8之后的版本, 元空间就是方法区, 不再使用虚拟机内存, 而是使用本地内存。**

> 看到上面的图可以明白, 方法区是一个概念, 而真正的实现方式是永久代或者是元空间。