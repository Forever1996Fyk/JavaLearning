> JUC编程

之前我们已经学习了Java并发编程的基础篇和提升篇, 了解Java并发编程中的非常重要基本方法的概念, 以及多线程的设计模式的使用。

而JUC则是真正在实际开发中, 面试中, 极其重要的一点。是否能够熟练的使用JUC包下的工具类, 能够直接的体现出开发者在Java并发编程中能力。


J.U.C并发包, 即`java.util.concurrent`包, 是JDK的并发工具包, 是JDK1.5之后, 由`Doug Lea`实现并引入。

整合`Java.util.concurrent`包, 按照功能可以大致划分如下:

- `juc-locks`轻量级锁框架
- `juc-atomic`原子类框架
- `juc-sync`同步器框架
- `juc-collections`集合框架
- `juc-executors`线程池框架

我们也会跟着上面的功能, 一步一步的学习其中的工具类以及API, 了解每个类的功能, 以及底层的原理。