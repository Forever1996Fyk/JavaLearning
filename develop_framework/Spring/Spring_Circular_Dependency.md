## Spring 循环依赖

我们都知道Spring为了解决循环依赖使用了三级缓存。

但是其中有很多的细节, Spring做了很多的操作, 其中还涉及到AOP的代理对象问题, 所以Spring的循环依赖并不能简单的用三级缓存回答就行了。

### 1. 三级缓存介绍

#### 一级缓存 singletonObejcts

用于保存beanName和创建Bean实例之间的关系

```java
private final Map<String, Object> singletonObjects = new ConcurrentHashMap(256);
```

#### 二级缓存 earlySingletonObjects

保存提前曝光的单例bean对象

```java
private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);
```

#### 三级缓存 singletonFactories

保存beanName和创建bean实例之间的关系.

注意: 这里创建的Bean是刚刚实例化的Bean, 还没有进行属性赋值和初始化阶段。

三级缓存的目的就是为了, 保证在循环引用的过程中能获取到, 不关心这个Bean能不能用。

```java
private final Map<String, Object> singletonFactories = new HashMap(16);
```

### 2. 原理分析

Spring决绝循环依赖的核心思想在于 **<font color="red">提前曝光!</font>**

1. 通过构建函数创建对象A; (这个A对象只是刚刚实例化, 还没有注入属性和调用init方法);

2. 当为对象A注入属性时, 需要注入对象B, 于是递归调用doGetBean方法区获取B对象, 但是发现缓存中没有B对象, 于是将**半成品A对象放入三级缓存中**;

3. 通过构造函数创建B对象; (此时B对象也是刚刚实例化, 还没有注入属性和调用init方法);

4. 当为对象B注入属性时, 需要注入对象A, 于是先从一级缓存中获取, 如果获取不到, 就从二级缓存中获取, 如果二级也获取不到, 就从三级缓存中获取 半成品对象A;

5. B对象将半成品对象A注入属性, 并开始初始化, 然后将对象B加入一级缓存, 移除二级, 三级缓存, 并且返回完成体的对象B;

6. 半成品对象A获取到完成体的对象B, 并注入属性, 开始初始化, 并将对象A加入一级缓存, 移除二级, 三级缓存。

上面就是大致整个解决循环依赖的流程, 要理解原理, 最好的方法就是阅读源码, 从创建Bean的方法

`AbstractAutowireCapableBeanFactor#doCreateBean`入手。

