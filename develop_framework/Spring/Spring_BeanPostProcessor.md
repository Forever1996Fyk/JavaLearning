## Spring BeanPostProcessor接口源码分析

BeanPostProcessor接口是Spring扩展点中可能是最重要的一个接口了, Spring AOP的功能就是基于这个扩展点实现的, 所以我们在学习BeanPostProcessor时, 也会分析Spring AOP的实现原理。

### 1. BeanPostProcessor的创建时机

我们通过上一篇分析的[Bean生命周期](develop_framework/Spring/Spring_Bean_LifeCycle.md)的文章中, 可以知道BeanPostProcessor的执行时机是在Bean的初始化前后。所以`BeanPostProcessor`又被称为 **后置处理器**。

> `InstantiationAwareBeanPostProcessor`继承了`BeanPostProcessor`, 所以也算是`BeanPostProcessor`类型的接口。 但是它的调用时机是在Bean的实例化前后, 所以严格来说,  `BeanPostProcessor`的执行时机是在Bean的实例化与初始化前后。

那这就意味着, BeanPostProcessor对象就必须要在Bean创建之前被创建, 这样才能调用。

我们回到`AbstractApplicationContext#refresh()`方法中, 发现这个方法中调用了`registerBeanPostProcessors`方法, 这里就会把所有的BeanPostProcessor的对象都创建好了, 而普通Bean的创建是在`finishBeanFactoryInitialization`方法中执行的;

```java
public void refresh() throws BeansException, IllegalStateException {
    synchronized(this.startupShutdownMonitor) {
        this.prepareRefresh();
        ConfigurableListableBeanFactory beanFactory = this.obtainFreshBeanFactory();
        this.prepareBeanFactory(beanFactory);

        try {
            // 注册BeanPostProcessor列表
            this.registerBeanPostProcessors(beanFactory);

            // 普通Bean的创建
            this.finishBeanFactoryInitialization(beanFactory);
        } catch (BeansException var9) {
            if (this.logger.isWarnEnabled()) {
                this.logger.warn("Exception encountered during context initialization - cancelling refresh attempt: " + var9);
            }

            this.destroyBeans();
            this.cancelRefresh(var9);
            throw var9;
        } finally {
            this.resetCommonCaches();
        }

    }
}
```

### 2. BeanPostProcessor 是 IOC与AOP的桥梁

我们都知道AOP的实现原理是动态代理, 如果我们在使用Spring注解时, 在调用Bean的地方debug时, 就会发现, 注入的Bean都不是原来的对象, 而是一个代理对象。

这也就意味着**最终Spring 初始化Bean时, 放进容器的必须是代理对象, 而不是原先的对象。**

> 注意: 我这里说的是 当我们使用AOP这个功能时, 生成的才是代理对象

那么Spring是怎么做到这点的呢? 正是利用了`BeanPostProcessor`。

我们只需要写一个实现`BeanPostProcessor`接口的类, 在`postProcessBeforeInitialization`或者`postProcessAfterInitialization`方法中, 对对象进行判断, 看这个Bean是否需要织入切面逻辑, 如果需要, 那就根据业务需要生成一个代理对象, 然后直接返回这个代理对象, 那么最终注入容器的, 就是目标对象的代理对象了。

这个服务于Spring AOP的`BeanPostProcessor`, 叫做 **`AnnotationAwareAspectJAutoProxyCreator`**

我们可以看到这个类的结构图:

![spring_aop_beanpostprocessor](/image/spring_aop_beanpostprocessor.png)

我们可以看到, 这个类的父类`AbstractAutoProxyCreator`实现了`SmartInstantiationAwareBeanPostProcessor`接口, 而这个接口又继承了`InstantiationAwareBeanPostProcessor`接口, 根据我们之前的分析, 这个接口的调用时机, 其实是在Bean实例化前后。

而当我们在深入到`AbstractAutoProxyCreator`的源码时, 会发现它在重写`postProcessAfterInitialization`方法中, 创建了代理对象:

```java
public abstract class AbstractAutoProxyCreator extends ProxyProcessorSupport
		implements SmartInstantiationAwareBeanPostProcessor, BeanFactoryAware {
	/**
	 * Create a proxy with the configured interceptors if the bean is
	 * identified as one to proxy by the subclass.
	 * @see #getAdvicesAndAdvisorsForBean
	 */
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

    	/**
	 * Wrap the given bean if necessary, i.e. if it is eligible for being proxied.
	 * @param bean the raw bean instance
	 * @param beanName the name of the bean
	 * @param cacheKey the cache key for metadata access
	 * @return a proxy wrapping the bean, or the raw bean instance as-is
	 */
	protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
        
        ...

		// Create proxy if we have advice.
		Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
		if (specificInterceptors != DO_NOT_PROXY) {
			this.advisedBeans.put(cacheKey, Boolean.TRUE);
			Object proxy = createProxy(
					bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
			this.proxyTypes.put(cacheKey, proxy.getClass());
			return proxy;
		}

		this.advisedBeans.put(cacheKey, Boolean.FALSE);
		return bean;
	}

    /**
	 * Create an AOP proxy for the given bean.
	 * @param beanClass the class of the bean
	 * @param beanName the name of the bean
	 * @param specificInterceptors the set of interceptors that is
	 * specific to this bean (may be empty, but not null)
	 * @param targetSource the TargetSource for the proxy,
	 * already pre-configured to access the bean
	 * @return the AOP proxy for the bean
	 * @see #buildAdvisors
	 */
	protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
			@Nullable Object[] specificInterceptors, TargetSource targetSource) {

        ....

		ProxyFactory proxyFactory = new ProxyFactory();
		proxyFactory.copyFrom(this);

        ....

		return proxyFactory.getProxy(getProxyClassLoader());
	}

}
```


所以 Spring AOP的实现原理是动态代理, 而Spring AOP的实现方式是`BeanPostProcessor`。