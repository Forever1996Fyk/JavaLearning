## Spring Bean加载解析

Spring IOC容器的作用是, 以某种方式加载配置资源信息, 将其解析注册到容器内部, 然后根据其信息绑定整个系统的对象, 最终组装成一个可用的轻量级容器。

上面这段话的意思其实就是把Spring IOC的实现功能分成两个阶段, 容器初始化和加载bean阶段。

- **容器初始化:** 这个阶段之前已经学习了, 可以直接参考这篇文章[IOC初始化解析](develop_framework/Spring/spring_ioc.md)

- **加载Bean阶段:** 经过IOC容器初始化之后, 程序中定义的Bean信息已经全部加载到系统中, 当我们触发`getBean()`方法时, 则会触发加载Bean阶段。容器会先检查所请求的对象是否已经初始化完成, 如果没有, 则会根据注册的bean信息实例化请求的对象, 并为其注册依赖, 然后将其返回给请求方。

### 1. Bean加载过程

我们知道, Bean的加载其实是发生在, 第一次程序获取Bean的过程中, 才真正的加载Bean。所以我们需要了解其中的源码。

在`AbstractApplicationContext.getBean(String name)`方法中:

```java

// AbstractApplicationContent.java

@Override
public Object getBean(String name) throws BeansException {
    assertBeanFactoryActive();
    return getBeanFactory().getBean(name);
}
```

其中又调用了`AbstractBeanFactory.getBean(String name)`:

```java

// AbstractBeanFactory.java

@Override
public Object getBean(String name) throws BeansException {
    return doGetBean(name, null, null, false);
}
```

看到这里我们发现, 其中获取Bean的核心方法就是`AbstractBeanFactory.doGetBean()`方法, 该方法接收四个参数:

- name: 要获取bean的名字

- requiredType: 要获取Bean的类型

- args: 创建Bean时传递的参数;

- typeCheckOnly: 是否为类型检查。

`doGetBean()`的代码非常长, 逻辑也非常复杂, 但是其中的内容非常重点, 它能解决我们大部分的Spring的问题:

```java
protected <T> T doGetBean(
        final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
        throws BeansException {

    // 获取 beanName, 这里是一个转换动作, 把name转换为beanName, 这我们可以不管
    final String beanName = transformedBeanName(name);
    Object bean;

    // Eagerly check singleton cache for manually registered singletons.
    // 从缓存中或实例工厂中获取bean
    Object sharedInstance = getSingleton(beanName);
    if (sharedInstance != null && args == null) {
        if (logger.isDebugEnabled()) {
            if (isSingletonCurrentlyInCreation(beanName)) {
                logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
                        "' that is not fully initialized yet - a consequence of a circular reference");
            }
            else {
                logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
            }
        }
        bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
    }

    else {
        // Fail if we're already creating this bean instance:
        // We're assumably within a circular reference.
        // 这里的英文注释, 提出的其实就是一个循环依赖问题
        // 因为Spring只解决单例模式下的循环依赖, 在原型模式下如果出现循环依赖会抛出异常
        if (isPrototypeCurrentlyInCreation(beanName)) {
            throw new BeanCurrentlyInCreationException(beanName);
        }

        // Check if bean definition exists in this factory.
        // 如果容器中没有找到, 则从父类容器中加载
        BeanFactory parentBeanFactory = getParentBeanFactory();
        if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
            // Not found -> check parent.
            String nameToLookup = originalBeanName(name);
            if (args != null) {
                // Delegation to parent with explicit args.
                return (T) parentBeanFactory.getBean(nameToLookup, args);
            }
            else {
                // No args -> delegate to standard getBean method.
                return parentBeanFactory.getBean(nameToLookup, requiredType);
            }
        }

        // 如果不是仅仅做类型检查则是创建bean，这里需要记录
        if (!typeCheckOnly) {
            markBeanAsCreated(beanName);
        }

        try {
            // 从容器中获取 beanName 相应的 GenericBeanDefinition，并将其转换为 RootBeanDefinition
            final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
            checkMergedBeanDefinition(mbd, beanName, args);

            // Guarantee initialization of beans that the current bean depends on.
            // 处理所依赖的 bean
            String[] dependsOn = mbd.getDependsOn();
            if (dependsOn != null) {
                for (String dependsOnBean : dependsOn) {
                    // 若给定的依赖 bean 已经注册为依赖给定的bean
                     // 循环依赖的情况
                    if (isDependent(beanName, dependsOnBean)) {
                        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                "Circular depends-on relationship between '" + beanName + "' and '" + dependsOnBean + "'");
                    }
                    registerDependentBean(dependsOnBean, beanName);
                    getBean(dependsOnBean);
                }
            }

            // Create bean instance.
            // bean 实例化
             // 单例模式
            if (mbd.isSingleton()) {
                sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
                    @Override
                    public Object getObject() throws BeansException {
                        try {
                            // 真正的Bean开始实例化
                            return createBean(beanName, mbd, args);
                        }
                        catch (BeansException ex) {
                            // Explicitly remove instance from singleton cache: It might have been put there
                            // eagerly by the creation process, to allow for circular reference resolution.
                            // Also remove any beans that received a temporary reference to the bean.

                            // 从单例缓存中删除bean实例
                            // 因为单例模式下为了解决循环依赖问题, 可能这个bean实例已经存在了, 所以出现异常需要销毁它
                            destroySingleton(beanName);
                            throw ex;
                        }
                    }
                });
                bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
            }

            // 如果bean类型是原型模式
            else if (mbd.isPrototype()) {
                // It's a prototype -> create a new instance.
                Object prototypeInstance = null;
                try {
                    beforePrototypeCreation(beanName);
                    prototypeInstance = createBean(beanName, mbd, args);
                }
                finally {
                    afterPrototypeCreation(beanName);
                }
                bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
            }

            else {
                 // 从指定的 scope 下创建 bean
                String scopeName = mbd.getScope();
                final Scope scope = this.scopes.get(scopeName);
                if (scope == null) {
                    throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
                }
                try {
                    Object scopedInstance = scope.get(beanName, new ObjectFactory<Object>() {
                        @Override
                        public Object getObject() throws BeansException {
                            beforePrototypeCreation(beanName);
                            try {
                                return createBean(beanName, mbd, args);
                            }
                            finally {
                                afterPrototypeCreation(beanName);
                            }
                        }
                    });
                    bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
                }
                catch (IllegalStateException ex) {
                    throw new BeanCreationException(beanName,
                            "Scope '" + scopeName + "' is not active for the current thread; consider " +
                            "defining a scoped proxy for this bean if you intend to refer to it from a singleton",
                            ex);
                }
            }
        }
        catch (BeansException ex) {
            cleanupAfterBeanCreationFailure(beanName);
            throw ex;
        }
    }

    // Check if required type matches the type of the actual bean instance.
    // 检查需要的类型是否符合 bean 的实际类型
    if (requiredType != null && bean != null && !requiredType.isAssignableFrom(bean.getClass())) {
        try {
            return getTypeConverter().convertIfNecessary(bean, requiredType);
        }
        catch (TypeMismatchException ex) {
            if (logger.isDebugEnabled()) {
                logger.debug("Failed to convert bean '" + name + "' to required type [" +
                        ClassUtils.getQualifiedName(requiredType) + "]", ex);
            }
            throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
        }
    }
    return (T) bean;
}
```

可以看到`doGetBean`的代码是非常复杂, 我们不需要完全的知道其中的情况, 只需要了解大致的执行流程, 以及其中的核心方法; 例如: Bean实例化, 如何解决循环依赖。

我们对上面的代码进行简化, 大致步骤如下:

1. 获取beanName;

    这里的转换beanName的作用是因为, 传递的参数name不一定就是beanName, 也可能是aliasName, 也可能是FactoryBean, 所以需要调用transformedBeanName()方法对name进行转换

2. 根据beanName从单例缓存中获取bean; 如果获取成功, 则对bean进行实例化处理;

    从代码中可以看到:
    ```java
        Object sharedInstance = getSingleton(beanName);
        if (sharedInstance != null && args == null) {
            if (logger.isDebugEnabled()) {
                if (isSingletonCurrentlyInCreation(beanName)) {
                    logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
                            "' that is not fully initialized yet - a consequence of a circular reference");
                }
                else {
                    logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
                }
            }
            bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
        }
    ```

    如果从缓存中得到了bean, 则需要调用`getObjectForBeanInstance()`对bean进行实例化处理, 因为缓存中记录的是最原始的bean状态, 并不是最终的bean。

3. 如果单例缓存中不存在bean, 则先进行依赖的处理, 然后创建bean实例; 在分析bean作用域;

    我们知道在单例模式下bean在整个过程只会被创建一次, 所以在创建bean实例前会先判断bean的作用域, 如果是单例, 则会第一次创建后将该bean加载到单例缓存中, 后面在获取bean就会从缓存中获取。

4. 最后检查需要的类型是否符合 bean 的实际类型


### 2. 具体分析Bean加载

从上面的解析可以看到, 在获取bean时, 会对bean进行加载, 其中又分为几种情况:

1. 从[单例缓存中获取bean](develop_framework/Spring/spring_beansingleton.md)

2. 从parentBeanFactory中加载bean并进行依赖处理

3. 还要对各种scope类型bean进行创建