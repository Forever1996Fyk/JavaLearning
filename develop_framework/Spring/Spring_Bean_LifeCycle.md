## Spring Bean的生命周期

前面我们已经学习了SpringIOC的原理以及Bean加载的过程, 这篇文章主要就是更加深入的了解Bean加载过程的细节, 总而言之就是Bean的生命周期!!

- [Spring IOC 原理](develop_framework/Spring/spring_ioc.md)

- [Spring Bean加载过程](develop_framework/Spring/spring_beanloader.md)

Spring Bean的生命周期是Spring面试的热点问题, 这个问题是考察队Spring的细节了解, 也是考察对Spring的宏观认识, 所以回答好这个问题, 会给面试官不错的影响。

但是本文不仅仅只是针对面试, 其实还是为了更深入的了解Spring, 提升自己!!!


### 1. Bean生命周期的宏观认识

从整体来看其实, **Bean的生命周期只有4个阶段**。只不过网上有很多的文章或者视频都是直接把很对扩展点糅合在一起, 虽然大部分是正确的, 但是这样会极大的影响我们了解Spring, 难以记忆。

要彻底搞清楚Spring的生命周期, 就必须要牢牢记住, Bean的四个主体阶段:

- `实例化 Instantiation`

- `属性赋值 Populate`

- `初始化 Initialization`

- `销毁 Destruction`

实例化和属性赋值对应构造方法和setter方法的注入，初始化和销毁是用户能自定义扩展的两个阶段。

在这四步直接穿插着各种扩展点, 从而构成了整个Bean的生命周期。

我们知道构建Bean的逻辑都在`doCreateBean`方法中, 其实我们把这个源码简化一下, 其实就是顺序调用以下三个方法:

1. `createBeanInstance` ---> 实例化

2. `populateBean`       ---> 属性赋值

3. `initializeBean`     ---> 初始化

```java
// 忽略了无关代码
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args) throws BeanCreationException {

   // Instantiate the bean.
   BeanWrapper instanceWrapper = null;
   if (instanceWrapper == null) {
       // 实例化阶段！
      instanceWrapper = createBeanInstance(beanName, mbd, args);
   }

   // Initialize the bean instance.
   Object exposedObject = bean;
   try {
       // 属性赋值阶段！
      populateBean(beanName, mbd, instanceWrapper);
       // 初始化阶段！
      exposedObject = initializeBean(beanName, exposedObject, mbd);
   }
}
```
至于销毁, 是在容器关闭时调用的, 详见`CofigurableApplicationConetxt#close()`。

### 2. 常用扩展点

在Bean的生命周期中, 相关的扩展点非常多, 最大的问题不是不知道, 而是记不住。只有非常深入了解Spring, 才能更好的记忆。

对于扩展点, 我们可以分为两大类:

- 影响多个Bean的接口

- 只调用一次的接口

#### 2.1 影响多个Bean的接口

实现这些接口的Bean会切入到多个Bean的生命周期中, 所以这些接口的功能非常强大, Spring内部扩展经常使用这些接口, 例如: 自动注入, AOP的实现。

最重要的两个接口:

- `BeanPostProcessor`

- `InstantiationAwareBeanPostProcessor`

其实`InstantiationAwareBeanPostProcessor`继承了`BeanPostProcessor`, 是两父子。

```java
InstantiationAwareBeanPostProcessor extends BeanPostProcessor
```

所以AOP的实现, 就是利用`BeanPostProcessor`。(在后面针对AOP的源码分析会提到)

`InstantiationAwareBeanPostProcessor`作用于**实例化**的前后, `BeanPostProcessor`作用于**初始化**阶段的前后。如下图:

![bean_life_cycle1](/image/bean_life_cycle1.webp)

##### InstantiationAwareBeanPostProcessor源码分析

- `postProcessBeforeInstantiation`调用点:

```java
@Override
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
        throws BeanCreationException {

    try {
        // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
        Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
        if (bean != null) {
            return bean;
        }
    }
    
    try {   
        // 上文提到的doCreateBean方法，可以看到
        // postProcessBeforeInstantiation方法在创建Bean之前调用
        Object beanInstance = doCreateBean(beanName, mbdToUse, args);
        if (logger.isTraceEnabled()) {
            logger.trace("Finished creating instance of bean '" + beanName + "'");
        }
        return beanInstance;
    }
    
}

```

我们看到`resolveBeforeInstantiation`方法, 这个方法就是for循环遍历所有实现`InstantiationAwareBeanPostProcessor`接口的实现类, 并调用`postProcessBeforeInstantiation`方法.

它是在`doCreateBean`方法之前调用的。也就是在bean实例化之前调用`InstantiationAwareBeanPostProcessor`的`postProcessBeforeInstantiation`方法

官方的注释可以看出 

**Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.**

翻译过来就是:

**给BeanPostProcessors一个返回代理而不是目标bean实例的机会**

我们可以看到在实例化bean之前, 如果实现了`InstantiationAwareBeanPostProcessor`, 它会直接返回这个bean的一个代理对象, 直接返回。 **<font color='red'>但是要注意的是, 这并不是AOP的实现方式, 因为此时对象都还没有实例化也不能直接生成一个代理类。 `postProcessBeforeInstantiation`只是为了判断指定的bean是否存在实例化快捷方式, 判断是否只需要生成代理对象即可</font>**

- `postProcessBeforeInstantiation`调用点:

```java
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {

    // Give any InstantiationAwareBeanPostProcessors the opportunity to modify the
    // state of the bean before properties are set. This can be used, for example,
    // to support styles of field injection.
    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
        for (BeanPostProcessor bp : getBeanPostProcessors()) {
            if (bp instanceof InstantiationAwareBeanPostProcessor) {
                InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
                    return;
                }
            }
        }
    }

   // 忽略后续的属性赋值操作代码
}
```

我们可以看到`InstantiationAwareBeanPostProcessor`的`postProcessBeforeInstantiation`方法是在属性赋值方法内调用的。

而且`InstantiationAwareBeanPostProcessor`重写了`postProcessBeforeInstantiation`接口, 会返回一个boolean类型, 如果返回false可以阻断属性赋值阶段, 也就说, 如果执行`postProcessBeforeInstantiation`错误, 就不会有下面的操作了。

#### 2.2 只调用一次的接口

这一类的接口特点就是功能丰富, 常用语用于自定义扩展。比如:

1. Aware类型的接口

2. 生命周期接口

##### 2.2.1 Aware类型的接口

Aware类型的接口就是让我们能够拿到Spring容器中的一些资源, 基本上就是见名知意, 比如: 

- `BeanNameAware`可以拿到BeanName。

- `BeanClassLoaderAware`可以拿到Bean的类加载器

- `BeanFactoryAware`可以拿到加载Bean的工厂类型

**<font color='red'>要注意的是Aware接口的调用时机, 所有的Aware方法都是在初始化阶段之前调用的!</font>**

```java
// 见名知意，初始化阶段调用的方法
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {

    // 这里调用的是三个Bean开头的Aware
    invokeAwareMethods(beanName, bean);

    Object wrappedBean = bean;

    wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    // 下文即将介绍的InitializingBean调用点
    invokeInitMethods(beanName, wrappedBean, mbd);
    // BeanPostProcessor的另一个调用点
    wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);

    return wrappedBean;
}
```

上面的代码可以看到, 在执行`invokeInitMethods`方法之前, 调用了`invokeAwareMethods`。正是我们之前说的, 三个Aware接口的方法.

```java
private void invokeAwareMethods(String beanName, Object bean) {
    if (bean instanceof Aware) {
        if (bean instanceof BeanNameAware) {
            ((BeanNameAware) bean).setBeanName(beanName);
        }
        if (bean instanceof BeanClassLoaderAware) {
            ClassLoader bcl = getBeanClassLoader();
            if (bcl != null) {
                ((BeanClassLoaderAware) bean).setBeanClassLoader(bcl);
            }
        }
        if (bean instanceof BeanFactoryAware) {
            ((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
        }
    }
}
```

而且`BeanPostProcessor`的调用时机也能在此体现, 包围了`invokeInitMethods`方法, 也就是说

> 在Bean初始化前, 先调用了Aware接口的实现方法, 然后调用了`BeanPostProcessor`的前置方法`postProcessBeforeInitialization`, 初始化之后, 调用后置方法`postProcessAfternitialization`


##### 2.2.2 简单的两个生命周期接口

