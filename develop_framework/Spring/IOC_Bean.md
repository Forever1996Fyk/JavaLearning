## <center>Spring IOC容器</center>

Spring容器是Spring Framework的核心。容器将创建对象, 把它们关联一起, 配置对象, 并管理他们的整个生命周期从创建到销毁。Spring容器视同依赖注入(DI)来管理组成一个应用程序的组件。这些对象称为Spring Beans。

IOC容器是具有依赖注入功能的容器, 可以创建对象, IOC容器负责实例化, 定位, 配置应用程序中的对象以及建立对象之间的依赖。通常我们new一个实例, 控制权有程序员控制, 而"控制反转"是指new实例的工作不由程序员实现,而是交给Spring容器。其中IOC容器代表为**BeanFactory,  ApplicationContent**。

> 这里面试可能会问, 谈谈你对Spring IOC和DI的理解？(基本上只要答出下面的三点, 面试官就知道你是理解Spring IOC和DI了)
 * 这两个东西的关系就在于, 控制反转(IOC)是依赖倒置原则的一种代码设计思路, 而实现这种思路的方法是依赖注入(DI)!!!
 * 对于一个有意义的应用, 会有很多的类组成或者依赖, 那么这些依赖就会引出一个问题, 耦合程度。一旦我们对某个类的代码进行修改, 却需要修改所有以它作为依赖的类, 那么这样的耦合程度太高, 维护成本太大。所以需要控制反转(IOC), 即上层类控制下层类, 我们用依赖注入来实现控制反转。而依赖注入就是把底层类作为参数传入上层类, 实现上层类对下层类的控制, 控制方式有**构造函数注入, Setter注入, 接口注入**。它们都是为了实现控制反转。
 * 因为使用依赖注入, 所以在初始化过程中就不可避免会出现大量的new实例(因为我们要不断的new底层类, 传入上层类中)。而IOC容器就是可以自动的对你的代码进行初始化, 也就是将new的过程交给容器, 你只需要去维护一个xml或者configuration的java代码。而且容器就像创建Bean的工厂一样, 我们只需要直接拿Bean就可以了, 不需要了解其中的细节。

 可以参考这篇文章, 写的很通俗易懂[https://www.zhihu.com/question/23277575/answer/169698662](https://www.zhihu.com/question/23277575/answer/169698662)

### 1. Spring BeanFactory容器

这个Spring中最简单, 最基础的IOC容器, 主要的功能是为依赖注入(DI)提供支持。

<font color="red">注意:</font> **BeanFactory是存在Spring中的, 但是现在在项目中一般我们是不用BeanFactory的, 而是用它的实现类。 它只是为IOC容器提供一个入口, 目的是向后兼容已经存在的和Spring整合在一起的第三方框架**。

> BeanFactory的每个子接口, 以及实现类都有使用的场合, 他主要是为了区分在 Spring 内部对象的传递和转化过程中, 对对象数据访问所做的限制!!!


### 2. Spring ApplicationContext容器
Application Context是BeanFactory的子接口, 也称为Spring上下文。它是Spring中较高级的容器,它能提供更多企业级的服务, 扩展了BeanFactory接口:
    * ApplicationEventPublisher: 事件发布功能。包括容器启动事件, 关闭事件。
    * MessageSource: 为应用提供i18国际化消息访问功能。
    * ResourcePatternResolver: 让容器可以通过带前缀的资源文件路径装载Spring的配置文件。
    * LifeCycle: 提供start(), stop()方法，用于控制异步处理。

前面提到的应用上下文的说法, 其他在很多地方都提到过, 但是很抽象(面试一般不会问这么抽象的问题, 这里只是我个人的一些理解)。
> 对于Spring上下文, 我的理解是 在运行环境中对Spring执行信息的描述。比如:
```java
    public interface ApplicationContext extends ListableBeanFactory, HierarchicalBeanFactory,
		MessageSource, ApplicationEventPublisher, ResourcePatternResolver { ... }
```
从ApplicationContext的定义中看出继承很多接口, 所以ApplicationContext就是Spring在运行环境中对BeanFactory, MessageSource..等执行信息的描述。
这样的话, 不仅仅是ApplicationContext, 还有Spring容器中其他的Context接口, 也是如此。

### 3. Spring Bean定义
Spring中的容器几乎都是以Bean为基础的, 换句话说Spring是将所有组件都当做Bean来进行管理。
Bean是一个被实例化, 组装并通过Spring IOC容器所管理的对象。

Bean在Spring容器中的组装流程： (一般面试官会这样问, 简述一下Bean与Spring容器的关系?)
> 1. Spring容器读取Bean的配置信息并放入Bean定义注册表中,
2. 根据Bean注册表实例化Bean,
3. 将Bean实例放入Spring容器中, 即Bean缓存池 ,
4. 应用程序使用Bean
![Bean与Spring容器的关系图](/develop_framework/Spring/img/Bean与Spring容器的关系图.jpg)
