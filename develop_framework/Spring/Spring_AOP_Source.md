## Spring AOP源码解析

通过前面几篇文章的学习, 我们知道Spring AOP是通过动态代理, 结合BeanPostProcessor 生成代理对象并注入到IOC容器。

这篇文章就具体的分析AOP的源码。

### 1. Spring AOP入口

对于AOP这样的非标准或者自定义命名空间的元素, Spring会从spring.handlers文件中的对应关系找到相应的处理类, 然后用过`init()`方法注册一些处理器。

`spring.handlers`:

```properties
http\://www.springframework.org/schema/aop=org.springframework.aop.config.AopNamespaceHandler
```

```java
public class AopNamespaceHandler extends NamespaceHandlerSupport {

	/**
	 * Register the {@link BeanDefinitionParser BeanDefinitionParsers} for the
	 * '{@code config}', '{@code spring-configured}', '{@code aspectj-autoproxy}'
	 * and '{@code scoped-proxy}' tags.
	 */
	@Override
	public void init() {
		// In 2.0 XSD as well as in 2.5+ XSDs
		registerBeanDefinitionParser("config", new ConfigBeanDefinitionParser());
		registerBeanDefinitionParser("aspectj-autoproxy", new AspectJAutoProxyBeanDefinitionParser());
		registerBeanDefinitionDecorator("scoped-proxy", new ScopedProxyBeanDefinitionDecorator());

		// Only in 2.0 XSD: moved to context namespace in 2.5+
		registerBeanDefinitionParser("spring-configured", new SpringConfiguredBeanDefinitionParser());
	}

}
```

然后我们直接跟进`init()`方法的中的 `AspectJAutoProxyBeanDefinitionParser`类, 一直找到`AopConfigUtils`类, 就会发现其中有三个固定的, 代理构造器:



```java
public abstract class AopConfigUtils {

	/**
	 * The bean name of the internally managed auto-proxy creator.
	 */
	public static final String AUTO_PROXY_CREATOR_BEAN_NAME =
			"org.springframework.aop.config.internalAutoProxyCreator";

	/**
	 * Stores the auto proxy creator classes in escalation order.
	 */
	private static final List<Class<?>> APC_PRIORITY_LIST = new ArrayList<>(3);

	static {
		// Set up the escalation list...
		APC_PRIORITY_LIST.add(InfrastructureAdvisorAutoProxyCreator.class);
		APC_PRIORITY_LIST.add(AspectJAwareAdvisorAutoProxyCreator.class);
		APC_PRIORITY_LIST.add(AnnotationAwareAspectJAutoProxyCreator.class);
	}
}
```

这三个类分别对应使用AOP的解析方式:

- InfrastructureAdvisorAutoProxyCreator: 自动代理创建者, 仅考虑基础结构Advisor Bean,而忽略任何应用程序定义的Advisor;

- AspectJAwareAdvisorAutoProxyCreator:  AspectJ 的 AOP实现方式;

- AnnotationAwareAspectJAutoProxyCreator: 注解AOP的实现方式;

这三个类都继承了AbstractAutoProxyCreator, 而这个类实现了`InstantiationAwareBeanPostProcessor`接口, 所以这三个类都可以作为我们分析AOP源码的入口。

我们这里只解析`AnnotationAwareAspectJAutoProxyCreator`源码, 也是平时我们开发比较常用的方式。 利用`@AspectJ`作为切面的AOP方式。

我们[Spring BeanPostProcessor接口源码分析](develop_framework/Spring/Spring_BeanPostProcessor.md)中可以知道, AOP的实现原理是动态代理, 而`BeanPostProcessor`是IOC与AOP的桥梁。

所以我们的重点当然就是实现的`postProcessBeforeInstantiation`方法和`postProcessAfterInstantiation`方法。

### 2. AOP解析: postProcessBeforeInstantiation

AOP的`postProcessBeforeInstantiation`实现, 是比较复杂的, 不仅仅是代码的逻辑复杂, 其中还有各种复杂继承关系, 所以看起来是有点困难的, 所以我这里只是总结出结论。

```java
@Override
public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) {
    Object cacheKey = getCacheKey(beanClass, beanName);

    if (!StringUtils.hasLength(beanName) || !this.targetSourcedBeans.contains(beanName)) {
        if (this.advisedBeans.containsKey(cacheKey)) {
            return null;
        }
        if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
            this.advisedBeans.put(cacheKey, Boolean.FALSE);
            return null;
        }
    }

    // Create proxy here if we have a custom TargetSource.
    // Suppresses unnecessary default instantiation of the target bean:
    // The TargetSource will handle target instances in a custom fashion.
    TargetSource targetSource = getCustomTargetSource(beanClass, beanName);
    if (targetSource != null) {
        if (StringUtils.hasLength(beanName)) {
            this.targetSourcedBeans.add(beanName);
        }
        Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);
        Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);
        this.proxyTypes.put(cacheKey, proxy.getClass());
        return proxy;
    }

    return null;
}
```

这个方法, 看起来不长, 但是如果深入`shouldSkip`方法, 就会发现非常复杂, 而且要注意的是`shouldSkip`被`AspectJAwareAdvisorAutoProxyCreator`重写了。

`shouldSkip`方法的主要目的：

1. 为了验证, 要不要跳过这个Bean, 如果这个Bean本身就是切面类的话, 那么就不会被代理了;

2. 找出被切面注释的方法, 再根据切点类型进行排序, 排序方式就是我们熟悉的 `Around.class, Before.class, After.class, AfterReturning.class, AfterThrowing.class `, 这个顺序觉得了切点执行的调用顺序。

执行完`shouldSkip`方法之后, 接下来还会判断这个Bean是否实现了`TargetSource`接口, 如果没有实现, 依然不会生成代理对象。

所以总结出来一句话, **一般情况下,AOP执行BeanPostProcessor 的postProcessBeforeInstantiation 只是生成实现`TargetSource`接口的代理对象; 而注解开发AOP并不是调用postProcessBeforeInstantiation方法。**

### 3. AOP解析: postProcessAfterInstantiation

当我们更进AOP的postProcessAfterInstantiation源码时, 就能知道, Spring AOP生成代理对象的方法就是在此方法中。

`AbstractAutoProxyCreator.java`

```java
@Override
public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
    if (bean != null) {
        Object cacheKey = getCacheKey(bean.getClass(), beanName);
        if (this.earlyProxyReferences.remove(cacheKey) != bean) {
            return wrapIfNecessary(bean, beanName, cacheKey);
        }
    }
    return bean;
}

protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
    ...

    Object proxy = createProxy(
        bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
    this.proxyTypes.put(cacheKey, proxy.getClass());
    return proxy;

    ...
}

protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
        @Nullable Object[] specificInterceptors, TargetSource targetSource) {

    ....

    return proxyFactory.getProxy(getProxyClassLoader());
}

```

我们再继续跟进`proxyFactory.getProxy`方法, 这是一个代理工厂, 专门生成AOP代理对象:

`ProxyFactory.java`

```java
public Object getProxy(@Nullable ClassLoader classLoader) {
    return createAopProxy().getProxy(classLoader);
}
```

`ProxyFactory`父类`ProxyCreatorSupport`实现了`createAopProxy`方法

最终, 我们发现在`DefaultAopProxyFactory.java`中 有一个`createAopProxy`方法:

```java
public class DefaultAopProxyFactory implements AopProxyFactory, Serializable {

	@Override
	public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
			Class<?> targetClass = config.getTargetClass();
			if (targetClass == null) {
				throw new AopConfigException("TargetSource cannot determine target class: " +
						"Either an interface or a target is required for proxy creation.");
			}
			if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
				return new JdkDynamicAopProxy(config);
			}
			return new ObjenesisCglibAopProxy(config);
		}
		else {
			return new JdkDynamicAopProxy(config);
		}
	}
}
```

看到这我们应该有所熟悉, 这个方法会根据目标类的实现类型判断选择代理方式:

1. 如果目标对象(Bean)实现了接口, 会用JDK代理模式

2. 如果目标对象(Bean)没有实现接口, 会用cglib代理

3. 如果我们在使用AOP时, 配置了`proxy-target-class`属性为true, 那就会强制使用cglib, 但是前提目标类不是接口。

接下来, 我们看`JdkDynamicAopProxy`中的`getProxy`方法:

```java
@Override
public Object getProxy(@Nullable ClassLoader classLoader) {
    if (logger.isTraceEnabled()) {
        logger.trace("Creating JDK dynamic proxy: " + this.advised.getTargetSource());
    }
    Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised, true);
    findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
    return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
}
```

这就是之前学习动态代理的操作。

### 4. 总结

至此, 我们把Spring AOP的执行流程总结一下, 应该还是比较清晰的。

当Spring启动时, 开始构建Bean; 

1. 在Bean实例化之前, 执行所有实现`InstantiationAwareBeanPostProcessor`接口的类的`postProcessBeforeInitialization`方法, 

2. AOP就会执行`AbstractAutoProxyCreator`的`postProcessBeforeInitialization`方法, 判断Bean是否实现了TargetSource接口, 如果实现, 则生成代理类;

3. Bean实例化结束;

4. 在属性赋值之前, 执行所有实现`InstantiationAwareBeanPostProcessor`的`postProcessAfterInitialization`;

5. AOP就会执行`AbstractAutoProxyCreator`的`postProcessAfterInitialization`方法, 从而创建带有切面注解的Bean的代理对象, 然后返回;

6. 在后面Bean的生命周期中, 包括属性赋值, 初始化Bean, 都是代理对象。





