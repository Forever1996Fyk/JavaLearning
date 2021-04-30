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

### 3. 源码分析

我们先看下面的源码:

> 1. 在构造Bean对象之后, 将对象提前曝光到缓存中, 这个曝光的对象仅仅是构造完, 还没有注入属性和初始化。

```java
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory
        implements AutowireCapableBeanFactory {
    protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
            throws BeanCreationException {   

        // 创建Bean实例
        instanceWrapper = createBeanInstance(beanName, mbd, args);         
        ……
        // 是否提前曝光
        boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
                isSingletonCurrentlyInCreation(beanName));
        if (earlySingletonExposure) {
            if (logger.isTraceEnabled()) {
                logger.trace("Eagerly caching bean '" + beanName +
                        "' to allow for resolving potential circular references");
            }
            addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
        }
        ……

        // 注入属性
        populateBean(beanName, mbd, instanceWrapper);

        // 初始化
		exposedObject = initializeBean(beanName, exposedObject, mbd);
    }   
}     
```



> 2. 提前曝光的对象被放入`Map<String, ObjectFactory<?>> singletonFactories`缓存中, 也就是三级缓存, 并不是直接将Bean放入缓存, 而是包装成`ObjectFactory`对象在放入。

```java
public class DefaultSingletonBeanRegistry extends SimpleAliasRegistry implements SingletonBeanRegistry {
    protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
        Assert.notNull(singletonFactory, "Singleton factory must not be null");
        synchronized (this.singletonObjects) {
            // 一级缓存
            if (!this.singletonObjects.containsKey(beanName)) {
                // 三级缓存
                this.singletonFactories.put(beanName, singletonFactory);
                // 二级缓存
                this.earlySingletonObjects.remove(beanName);
                this.registeredSingletons.add(beanName);
            }
        }
    }
}
public interface ObjectFactory<T> {
    T getObject() throws BeansException;
}    
```

> 3. 为什么要包装一层`ObjectFactory`对象?

我们知道在用AOP时, 会创建Bean的代理对象, 如果其他对象需要注入的是Bean的代理对象, 那么Spring 就必须提前创建好这个Bean的代理才行, 但是Spring是无法提前知道这个对象是不是有循环依赖的情况, 所以正常情况下(无循环依赖), Spring都是在Bean初始化完成后, 再创建Bean的嗯对象。所以Spring有两种选择:

1. 不管有没有循环依赖, 都提前创建好代理对象, 并将代理对象放入缓存, 出现循环依赖时, 其他对象就可以获取到代理对象并注入;

2. 不提前创建好代理对象, 再出现循环依赖时被其他对象注入时, 才生成代理对象。


**Spring选择了第二种方式, 那怎么做到提前曝光对象而又不生成代理?**

Spring是在对象外层包一层`ObjectFactory`, 提前曝光的是`ObjectFactory`对象, 在被注入时在`ObjectFactory.getObject`方式内实时生成代理对象, 并将生成好的代理对象放入到第二级缓存`Map<String, Object> earlySingletonObjects`。

我们看下`addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));`中的 `getEarlyBeanReference`方法:

```java
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory
        implements AutowireCapableBeanFactory {

    protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
        Object exposedObject = bean;
        if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
            for (BeanPostProcessor bp : getBeanPostProcessors()) {
                if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
                    SmartInstantiationAwareBeanPostProcessor ibp = (SmartInstantiationAwareBeanPostProcessor) bp;
                    exposedObject = ibp.getEarlyBeanReference(exposedObject, beanName);
                }
            }
        }
        return exposedObject;
    }
}
```

这段代码, 是执行所有`SmartInstantiationAwareBeanPostProcessor`接口的`getEarlyBeanReference`方法, 

而在[Spring AOP源码解析](develop_framework/Spring/Spring_AOP_Source.md)这篇文章中, 生成代理对象的核心类`AbstractAutoProxyCreator`正好实现了这个接口, 并且实现了这个方法, 返回了代理对象:

```java
public abstract class AbstractAutoProxyCreator extends ProxyProcessorSupport
        implements SmartInstantiationAwareBeanPostProcessor, BeanFactoryAware {
    private final Map<Object, Object> earlyProxyReferences = new ConcurrentHashMap<>(16);
            
    @Override
    public Object getEarlyBeanReference(Object bean, String beanName) {
        Object cacheKey = getCacheKey(bean.getClass(), beanName);
        this.earlyProxyReferences.put(cacheKey, bean);
        return wrapIfNecessary(bean, beanName, cacheKey);
    }        
}        
```

看到这个`wrapIfNecessary`方法, 应该是比较熟悉的, 这正是生成代理对象的方法。 为了防止对象在后面的初始化(init)时重复代理，在创建代理时，earlyProxyReferences缓存会记录已代理的对象。

> 4. 注入属性和初始化

提前曝光后:

1. 通过`populateBean`方法注入属性, 再注入其他Bean对象时, 会先去一级, 二级, 三级缓存中取, 如果缓存没有, 就创建该对象并注入;

2. 通过`initializeBean`方法初始化对象, 包含创建代理。

```java
// 从缓存中获取要注入的单例对象
public class DefaultSingletonBeanRegistry extends SimpleAliasRegistry implements SingletonBeanRegistry {
    protected Object getSingleton(String beanName, boolean allowEarlyReference) {
        // 一级缓存
        Object singletonObject = this.singletonObjects.get(beanName);
        if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
            synchronized (this.singletonObjects) {
                // 二级缓存
                singletonObject = this.earlySingletonObjects.get(beanName);
                if (singletonObject == null && allowEarlyReference) {
                    // 三级缓存
                    ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                    if (singletonFactory != null) {
                        singletonObject = singletonFactory.getObject();
                        this.earlySingletonObjects.put(beanName, singletonObject);
                        this.singletonFactories.remove(beanName);
                    }
                }
            }
        }
        return singletonObject;
    }
}   
```

> 5. 放入已完成创建的单例缓存

在经过下面步骤之后, 最终通过`addSingleton`方法将最终生成的可用Bean放入一级缓存, 并把二级, 三级缓存删除。

1. AbstractBeanFactory.doGetBean ->
2. DefaultSingletonBeanRegistry.getSingleton ->
3. AbstractAutowireCapableBeanFactory.createBean ->
4. AbstractAutowireCapableBeanFactory.doCreateBean ->
5. DefaultSingletonBeanRegistry.addSingleton

```java
public class DefaultSingletonBeanRegistry extends SimpleAliasRegistry implements SingletonBeanRegistry {

    /** Cache of singleton objects: bean name to bean instance. */
    private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

    /** Cache of singleton factories: bean name to ObjectFactory. */
    private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);

    /** Cache of early singleton objects: bean name to bean instance. */
    private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);

    protected void addSingleton(String beanName, Object singletonObject) {
        synchronized (this.singletonObjects) {
            this.singletonObjects.put(beanName, singletonObject);
            this.singletonFactories.remove(beanName);
            this.earlySingletonObjects.remove(beanName);
            this.registeredSingletons.add(beanName);
        }
    }
}    
```

### 4. 二级缓存的疑问

通过上面的分析, 就会有一个疑问, 我们之所以要包装一层`ObjectFactory`对象是为了保证只有当发生循环依赖被其他对象注入时, 才实时生成代理对象。

但是如果我们不利用`ObjectFactory`, 直接将生成的代理对象放入二级缓存中, 是否可行?

1. 在提前曝光半成品时，直接执行getEarlyBeanReference创建到代理，并放入到缓存earlySingletonObjects中，这时候earlySingletonObjects里放的不再是实际对象，而是代理对象，代理对象的target(即实际对象)是半成品。

2. 有了上一步，那就不需要通过ObjectFactory来延迟执行getEarlyBeanReference，也就不需要singletonFactories这这个第三级缓存。

这种处理方式可行吗？
这里做个试验，对AbstractAutowireCapableBeanFactory做个小改造，在放入三级缓存之后立刻取出并放入二级缓存，这样三级缓存的作用就完全被忽略掉，就相当于只有二级缓存。

![Spring_Circular_Dependency](/image/Spring_Circular_Dependency.png)

```java
public abstract class AbstractAutowireCapableBeanFactory extends AbstractBeanFactory
        implements AutowireCapableBeanFactory {
    protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
            throws BeanCreationException {            
        ……
        // 是否提前曝光
        boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
                isSingletonCurrentlyInCreation(beanName));
        if (earlySingletonExposure) {
            if (logger.isTraceEnabled()) {
                logger.trace("Eagerly caching bean '" + beanName +
                        "' to allow for resolving potential circular references");
            }
            addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
            // 立刻从三级缓存取出放入二级缓存
            getSingleton(beanName, true);
        }
        ……
    }   
}     
```

测试结果是可以的，并且从源码上分析可以得出两种方式性能是一样的，并不会影响到Sping启动速度。


*那为什么Sping不选择二级缓存方式，而是要额外加一层缓存?*

**如果要使用`二级缓存`解决循环依赖, 意味着Bean在构造完后就创建代理对象, 这样违背了Spring设计原则。**

**我们知道, Spring AOP创建代理对象是通过`AnnotationAwareAspectJAutoProxyCreator`的后置处理器来完成的, 在`postProcessAfterInitialization`方法中对初始化后的Bean对象生成代理对象, 完成AOP代理。**

**如果出现了循环依赖, 那就只有给Bean先创建代理, 但是如果没有发生循环依赖那就应该按照Spring的设计原则来, 根据Bean的生命周期, 在Bean初始化之后再完成代理, 而不是在实例化后, 生成代理。**

> **也就是说, 个人认为 Spring之所以采用三级缓存的原因, 就是当生成的对象是代理对象时, 保证在正常情况下(无循环依赖)Spring生产代理对象是在Bean初始化之后。
>但是如果出现循环依赖, 那么就没有办法必须要在Bean初始化和注入属性之前, 生成代理对象。**

### 5. 代理对象的循环依赖

如果对象A和对象B循环依赖, 并且都是相互代理, 那创建的顺序就是:

1. A半成品加入第三级缓存;

2. A注入属性B -> 创建B对象(此时不是代理对象) -> B半成品加入第三级缓存;

3. B注入属性A -> 从第三级缓存中获取时, 生成A代理对象, 从第三级缓存移除A对象, 并且把A代理对象加入第二级缓存 (此时A是半成品的, B注入的是A代理对象);

4. B注入属性完成后, 开始初始化, 初始化完成后, 开始生产B代理对象(此时B是完成品) -> 从第三级缓存移除B对象, B代理对象加入第一级缓存;

5. A半成品注入B代理对象;

6. 从第二级缓存移除A代理对象, A代理对象加入第一级缓存。
