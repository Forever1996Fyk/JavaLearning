## 多线程的单例设计模式

单例模式是在实际开发中比较常用的一种设计模式, 如果要求这个类的实例来程序中只允许能有唯一一个, 那么可以使用单例模式来解决这个问题。但是在多线程开发中, 单例模式却处处存在陷阱。

### 1. 饿汉式-单例模式

我们利用私有的不可变静态实例, 来实现一个最简单的单例模式, 如下:

```java
public class SimpleSingletonObject {
    private static final SimpleSingletonObject instance = new SimpleSingletonObject();

    private SimpleSingletonObject() {

    }

    public static SimpleSingletonObject getInstance() {
        return instance;
    }
}
```

上面的最简单的单例模式, 同时也是线程安全的, 因为它是静态常量, 所以在程序启动的时候就会被加载, 而且被finial修饰无法被改变。

**这种方式在程序一启动就会new一个对象放在内存中, 如果很长时间都用不到这个实例, 就会占用内存资源, 所以这种方式被称为`饿汉式单例`**

### 2. 懒汉式-单例模式

之前饿汉式单例, 存在占用内存资源的问题。所以提出一种改进的方式, **当调用到该对象时, 才会去创建唯一的实例, 这种方式被称为`懒汉式单例`**

```java
public class LazySingletonObject {
    private static LazySingletonObject instance;

    private LazySingletonObject() {

    }

    public static LazySingletonObject getInstance() {
        if (instance == null) {
            instance = new LazySingletonObject();
        }
        return LazySingletonObject.instance;
    }
}
```

上面的代码名称存在线程安全问题, 因为存在共享变量, 而且对共享变量进行修改操作。所以当有两个线程A, B同时调用`getInstance()`方法时, 线程A在进行instance = new LazySingletonObject()前, 可能线程B也进入if判断, 此时instance依然为null。这就导致了, 在程序中产生不止一个LazySingletonObject实例。

### 3. 懒汉式-单例模式 加锁

通过上面的分析, 可以通过加锁的方式来解决线程安全问题。

```java
public class LazyLockSingletonObject {
    private static LazyLockSingletonObject instance;

    private LazyLockSingletonObject() {

    }

    public synchronized static LazyLockSingletonObject getInstance() {
        if (instance == null) {
            instance = new LazyLockSingletonObject();
        }
        return LazyLockSingletonObject.instance;
    }
}
```

这种方式确实不存在线程安全问题, 但是加了`synchronized`, 就会导致每次调用`getInstance()`方法都只能进入一个线程, 虽然除了第一次调用这个方法会进行实例化, 但是后面的操作实际上是一个读操作, 只是获取这个对象的实例, 所以存在性能上的缺陷。

### 3. 双重检查单例模式

```java
public class DoubleCheckLazySingletonObject {
    private static DoubleCheckLazySingletonObject instance;

    private DoubleCheckLazySingletonObject() {

    }

    public static DoubleCheckLazySingletonObject getInstance() {
        if (instance == null) {
            synchronized (DoubleCheckLazySingletonObject.class) {
                if (instance == null) {
                    instance = new DoubleCheckLazySingletonObject();
                }
            }
        }

        return DoubleCheckLazySingletonObject.instance;
    }
}
```

对于双重检查模式, 代码并不复杂。通过在加锁前和加锁后判断实例是否为空, 来保证始终只有一个实例。

> double check model 依然存在一个严重的问题 ———— 指令重排序<br/>
> 简单来说, 在初始化 instance = new DoubleCheckLazySingletonObject() 的过程中, instance已经指向DoubleCheckLazySingletonObject对象所在的堆内存地址了, 但是此时DoubleCheckLazySingletonObject并没有完成在堆内存的完全加载完成(这里面是关于JVM的一些调优问题)<br/>
> 所以当线程A进入实例化过程中, DoubleCheckLazySingletonObject并没有完全的在JVM中加载成功, 但是已经返回instance的内存地址了, 所以instance!=null, 而线程B进行判断, 就会直接返回不完整的实例。

### 4. volatile double check - 单例模式

对于双重检查模式的问题, 只需要在静态私有变量前加上`volatile`关键字修饰即可。(具体的`volatile`关键字会在后面解析)

```java
public class DoubleCheckLazySingletonObject {
    private volatile static DoubleCheckLazySingletonObject instance;

    private DoubleCheckLazySingletonObject() {

    }

    public static DoubleCheckLazySingletonObject getInstance() {
        if (instance == null) {
            synchronized (DoubleCheckLazySingletonObject.class) {
                if (instance == null) {
                    instance = new DoubleCheckLazySingletonObject();
                }
            }
        }

        return DoubleCheckLazySingletonObject.instance;
    }
}
```

**`volatile`关键字的目的就是明确的告诉编译器, 不要对这个对象进行优化, 也就是禁止重排序。**这样就可以解决之前的问题。

### 5. 优雅的单例模式

##### 方式一:

```java
public class ElegantSingletonObject {

    private ElegantSingletonObject() {

    }

    /**
     * InstanceHolder类是静态内部类, 只有在被主动调用的时候才会加载一次
     */
    private static class InstanceHolder {
        private final static ElegantSingletonObject instance = new ElegantSingletonObject();

    }

    public static ElegantSingletonObject getInstance() {
        return InstanceHolder.instance;
    }
}
```
在项目开发中可以使用这种方式来实现单例模式, 这种方式没有加锁, 也没有判断, 但是确实是线程安全的。

> 当第一次调用这个方法时, 会主动加载 InstanceHolder静态内部类, 在初始化InstanceHolder时, 就会加载内部的静态实例ElegantSingletonObject, 而且只会被加载一次。

##### 方式二:

```java
public class EnumSingletonObject {
    private EnumSingletonObject() {

    }

    private enum Singleton {
        INSTANCE;

        private final EnumSingletonObject instance;

        Singleton() {
            instance  = new EnumSingletonObject();
        }

        public EnumSingletonObject getInstance() {
            return instance;
        }
    }

    public static EnumSingletonObject getInstance() {
        return Singleton.INSTANCE.getInstance();
    }
}
```

这是java开发者推荐的方式, 枚举单例模式, 但是我感觉写起来并不优雅。

> 调用Singleton.INSTANCE时, 就会加载枚举类的构造方法, 然后创建EnumSingletonObject实例, 枚举类是线程安全的, 而且只会加载一次。




