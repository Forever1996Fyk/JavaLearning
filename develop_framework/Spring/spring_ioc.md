## Spring IOC原理解析

学习Spring就不得不提IOC特性, IOC的意思是控制反转, 将对象的创建完全交给Spring容器来完成。

既然是创建对象, 那么肯定运用到了工厂模式, 因为工厂模式的目的就是为了防止业务代码中出现大量的new操作;

> 但是Spring是如何知道程序需要哪些对象实例?

    Spring利用Java反射的性质, 通过提供类的全限定名, 再利用反射来创建对象。

**所以其实Spring IOC容器的工作原理, 就是 <font color='red'>工厂模式+反射</font> 来实现控制反转。** 至于依赖注入, 其实是包含在反射的操作中, 在利用反射时, 需要获取对象的构造器或者setter方法, 才能真正的赋予属性。

### 1. IOC 初始化认识

我们先看下面的一段代码:

```java
ClassPathResource resource = new ClassPathResource("bean.xml");
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
reader.loadBeanDefinitions(resource);
```

对于上面的代码, 我们可以判断IOC容器的使用过程:

- 获取资源 resource `new ClassPathResource("bean.xml");`

    我们都知道, 在使用Spring时, 都会先创建xxxBean.xml文件,这些文件中就定义Bean的相关信息;

- 获取BeanFactory `new DefaultListableBeanFactory();`

- 根据BeanFactory创建`BeanDefinitionReader`对象, 该Reader对象为资源的解析器;

- 加载BeanDefinition对象。

    利用`BeanDefinitionReader`解析resource资源, 从而获取BeanDefinition。

![spring_ioc1](/image/spring_ioc1.png)


### 2. IOC 初始原理

IOC初始化其实基本上就分为3部分:

- **Resource资源定位**

    我们一般用外部资源来描述Bean对象, 利用我们最先使用的xml文件, 以及SpringBoot的yaml文件, 还有注解, 还包括properties文件, 都是一种外部资源来描述Bean对象。

    所以初始化IOC容器的第一步就是需要找到这些外部资源, 并封装到Rescource对象中。

- **解析资源获取`BeanDefinition`对象**

    通过`BeanDefinitionReader`对象来读取解析Resource资源, 也就是将用户定义的外部资源的Bean信息, 封装成IOC内部的数据结构: `BeanDefinition`。

- **`BeanDefinition`注册**

    向IOC容器注册第二步解析好的`BeanDefinition`对象, 这个过程是通过`BeanDefinitionRegistery`接口来实现的。
    
    它的原理就是通过一个`HashMap`容器, key就是beanName, value就是`BeanDefinition`实例, 来维护这些`BeanDefinition`。

    > 这里要注意的是, 这个过程并没有完成依赖注入, 依赖注入是发生在应用第一次调用`getBean()`向容器索要Bean时。我们可以通过设置预处理, 对某个Bean设置`lazyinit`属性, 那么这个Bean就会在IOC容器初始化时完成依赖注入。

### 3. Resource资源定位

我们这里不做详细介绍, 想要了解的可以看这篇博客 [IOC 之 Spring 统一资源加载策略](http://cmsblogs.com/?p=2656)

我们只需要知道Spring IOC容器的第一步就是定位外部资源, 并加载到对应的资源对象中, 例如上面代码的 `ClassPathResource`对象。

外部资源的类型可以是:

- xml

- properties

- yaml

- 注解

### 4. BeanDefinition的载入和解析

通过上面的代码, 我可以看出这个步骤的源码肯定是在 `reader.loadBeanDefinitions(resource);`中

这里也不做过多的解析, 因为这一步就是将定位的外部资源, 根据不同资源, 解析成我们所需要的`BeanDefinition`对象。

### 5. BeanDefinition的注册

经过上面的解析, 将资源中Bean标签解析成一个个`BeanDefinition`对象, 然后就需要将这些`BeanDefinition`注册到IOC容器中。

源码如下:

```java
 protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
        BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
        if (bdHolder != null) {
            bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
            try {
                // Register the final decorated instance.
                // 这是注册BeanDefinition的核心代码
                BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
            }
            catch (BeanDefinitionStoreException ex) {
                getReaderContext().error("Failed to register bean definition with name '" +
                        bdHolder.getBeanName() + "'", ele, ex);
            }
            // Send registration event.
            getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
        }
    }
```

调用`BeanDefinitionReaderUtils.registerBeanDefinition()`注册, 最终的实现是在DefaultListableBeanFactory中:

```java
@Override
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
        throws BeanDefinitionStoreException {

    // 省略一堆校验

    BeanDefinition oldBeanDefinition;

    oldBeanDefinition = this.beanDefinitionMap.get(beanName);
        // 省略一堆 if
        this.beanDefinitionMap.put(beanName, beanDefinition);
    }
    else {
        if (hasBeanCreationStarted()) {
            // Cannot modify startup-time collection elements anymore (for stable iteration)
            synchronized (this.beanDefinitionMap) {
                this.beanDefinitionMap.put(beanName, beanDefinition);
                List<String> updatedDefinitions = new ArrayList<>(this.beanDefinitionNames.size() + 1);
                updatedDefinitions.addAll(this.beanDefinitionNames);
                updatedDefinitions.add(beanName);
                this.beanDefinitionNames = updatedDefinitions;
                if (this.manualSingletonNames.contains(beanName)) {
                    Set<String> updatedSingletons = new LinkedHashSet<>(this.manualSingletonNames);
                    updatedSingletons.remove(beanName);
                    this.manualSingletonNames = updatedSingletons;
                }
            }
        }
        else {
            // Still in startup registration phase
            this.beanDefinitionMap.put(beanName, beanDefinition);
            this.beanDefinitionNames.add(beanName);
            this.manualSingletonNames.remove(beanName);
        }
        this.frozenBeanDefinitionNames = null;
    }

    if (oldBeanDefinition != null || containsSingleton(beanName)) {
        resetBeanDefinition(beanName);
    }
}
```

这段代码最核心的部分就是`this.beanDefinitionMap.put(beanName, beanDefinition)`, 这就是利用一个Map集合对象来存放, key是BeanName, value是BeanDefinition。

> 这也是我们在定义资源中的Bean信息时, BeanName是不能重复的原因, 因为Map中key是不能重复的。

### 6. 总结

通过上面的分析, 我们可以很轻松的画出IOC容器初始化的流程图:

![spring_ioc2](/image/spring_ioc2.png)

如果研究源码也可以画出代码的详解执行图:

![spring_ioc3](/image/spring_ioc3.png)