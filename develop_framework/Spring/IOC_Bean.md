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

> Bean在Spring容器中的组装流程： (一般面试官会这样问, 简述一下Bean与Spring容器的关系?)

1. Spring容器读取Bean的配置信息并放入Bean定义注册表中,
2. 根据Bean注册表实例化Bean,
3. 将Bean实例放入Spring容器中, 即Bean缓存池 ,
4. 应用程序使用Bean

![Bean与Spring容器的关系图](/develop_framework/Spring/img/Bean与Spring容器的关系图.jpg)

有三种方式配置Bean提供给Spring容器
* 基于XML的配置文件
* 基于注解配置
* 基于Java的配置

### 4. Spring Bean作用域(重点, 这里是面试高频提问点)
> 这是面试常问的问题, Spring中的Bean作用域有哪些?

* singleton: 唯一Bean实例, Spring在每次需要时都返回同一个bean实例, Spring中的Bean默认都是单例的(这是重点, 还会有很多扩展问题, 比如: 单例Bean的线程安全问题)
* prototype: 每次从容器中调用Bean时，都返回一个新的实例, 即每次调用getBean()时, 相当于执行newXxxBean()。
* request: 每次HTTP请求都会创建一个新的Bean, 这个Bean仅在当前HTTP request内有效, 也就是说该作用域仅适用于WebApplicationContext环境
* session: 同一个HTTP Session共享一个Bean，不同Session使用不同的Bean，仅适用于WebApplicationContext环境
* global-session: 全局session作用域, 仅仅在基于portlet的web应用中才有意义，Spring5已经没有了。Portlet是能够生成语义代码(例如：HTML)片段的小型Java Web插件。它们基于portlet容器，可以像servlet一样处理HTTP请求。但是，与 servlet 不同，每个 portlet 都有不同的会话

> 将一个类声明为Spring的Bean的注解有哪些?

我们一般使用`@Autowired`注解自动装配Bean, 想要把类标识成可用于`@Autowirded`注解自动装配的Bean的类, 采用以下注解实现:
* `@Component`: 通用注解, 可以标注任意类为`Spring`组件。如果一个Bean不知道属于那一层, 可以使用这个注解。 
* `@Repository`: 对应持久层即DAO层, 主要用于数据库相关操作。
* `@Service`: 对应服务层, 主要涉及一个复杂的业务逻辑, 需要用到Dao层
* `@Controller`: 对应Spring MVC控制层, 主要用于接收用户请求并调用Service层返回数据给前端。

Spring中Bean的作用域大多数都是默认单例的, `@Component`, `@Service`, `@repository`都是默认单例, (注意`@Controller`默认多例)

> 那么问题来了, 为什么Spring大多数都是默认单例呢?(这也是面试常见问题)

**为了提高性能!!!** 
* 单例Bean只有第一次创建新的Bean, 后面每次使用都会复用这个Bean, 所以不会频繁创建对象。(这里还可以扩展一下, Spring会通过反射或者cglib来生成bean实例, 其次给对象分配内存也会涉及复杂算法, 这都是耗性能的操作)
* 减少JVM垃圾回收, 由于对象创建少了, 自然回收的对象也少了。
* 缓存快速获取, Bean被声明为单例时, 这个对象会保存在一个Map里, 需要这个Bean时会先从这个map缓存中查看有没有, 有的话直接使用, 没有的话才实例化一个新的对象。


> Spring Bean单例的劣势(Spring 单例Bean是否存在线程安全问题)?

单例bean不是线程安全的!!! 或者换句话说 单例Bean存在线程安全问题。首先线程安全问题最根本的原因是, 在多线程环境下对这个对象非静态成员变量的写操作, 才会导致线程安全问题(对这个问题看下面代码)

```java
    //多线程环境下, count是非静态成员变量, 对这个变量进行写操作, 才会出现线程安全问题, 这也是多线程情况下, 出现线程安全的原因
    public class Test {
        private int count = 0;

        // 如果有两个线程同时调用run()方法, 那么每个线程输出的count基本上是不会连续的。
        // 但是每个线程输出的count2是连续的,因为count2是局部变量, 而局部变量不存在线程安全问题 
        public void run() {
            int count2 = 0;
            count++;
            count2++;
            System.out.println(count);
            System.out.println(count2);
        }
    }
```
而单例Bean对象中, 可能是存在成员变量的, 所以就会有线程安全问题。

比较常见的解决方案:
1. 在Bean对象中尽量避免定义可变的成员变量。
2. 在类定义中定义一个ThreadLocal成员变量, 将需要修改的成员变量保存在ThreadLocal中。(这也是常用的处理线程安全问题的方案所以很重要, 具体原理操作会在后面解析)

> 根据上面的理解, 又出现了一个新的问题。`@Service`, `@repository`都是默认单例的, 而单例又是线程不安全的, 那么我们在平时开发中service和dao层是怎么保证线程安全的?

这个问题的答案在上面其实已经提过了。Spring在实现Service和DAO对象时, 就是使用了ThreadLocal这个类(这个是核心)。

Spring通过ThreadLocal类将有状态的可变成员变量(例如数据库连接Connection)本地线程化, 从而做到线程安全。在一次请求响应的处理线程中, 这个线程贯通Controller, Service, Dao三层, 通过ThreadLocal使得所有关联的对象引用到的都是同一个变量。(事实上这里面还涉及到了Spring事务管理, 这个后面也会解析)


### 5. Spring Bean生命周期(重点)
Bean的生命周期可以比较简单的表达为: Bean的定义——Bean的初始化——Bean的使用——Bean的销毁

虽然表述简单, 但是这其中有很多步骤的实现。下面就是具体的Spring Bean生命周期:

* Bean容器找到配置文件中Spring Bean的定义。
* Bean容器利用Java Reflection API(Java反射)创建一个Bean的实例。
* 如果涉及到一些属性值, 利用Setter方法设置属性值。
* 如果Bean实现了BeanNameAware接口, 调用setBeanName()方法, 传入Bean的名字。
* 如果Bean实现了BeanClassLoaderAware接口, 调用setBeanClassLoader()方法, 传入ClassLoader对象的实例。
* 与上面类似, 如果实现了***Aware接口, 就调用相应的方法。
* 如果有和加载这个Bean的Spring容器相关的BeanPostProcessor对象, 就执行postProcessBeforeInitialization()方法。
* 如果Bean在配置文件中定义包含`init-method`属性, 执行指定的方法。
* 如果有和加载这个Bean的Spring容器相关的BeanPostProcessor对象, 就执行postProcessAfterInitialization()方法。
* 当要销毁Bean的时候, 如果Bean实现了DisposableBean接口, 就执行destory()方法。
* 当要销毁Bean的时候, 如果Bean在配置文件中定义包含`destory-method`属性, 执行指定的方法。

![Spring Bean的生命周期图](/develop_framework/Spring/img/SpringBean生命周期.jpg)

如果想要学习更加细节的代码操作, 可以参考这篇文章[https://yemengying.com/2016/07/14/spring-bean-life-cycle/ ](https://yemengying.com/2016/07/14/spring-bean-life-cycle/ )


### 6. 自动装配

装配在[Spring](/develop_framework/Spring/Spring.md)中说过, 创建应用组件之间关联的行为通常称为装配。**现在你可以理解为, Bean与Bean之间关联起来叫做装配。自动装配: 就是Spring容器自动找到Bean之间的关联, 并给Bean装配与其关联的属性。**

自动装配的方式有4中: 
* no: 表示关闭自动装配。
* byName: 根据Bean名称进行自动装配。 
* byType: 根据Bean的类型自动装配。
* constructor: 根据构造函数自动装配。

实现自动装配的方法常用的有两种: XML配置文件, 基于Java的注解。这里只介绍注解方式

> 谈谈你对**注解式自动装配**的理解?

首先我们要知道的是, 注解只起到标识作用, 没有实际的操作。所以对某一属性上使用`@Autowired`时, 表示要将这个Bean的引用作为当前类的属性, 所以当Spring容器启动时, 会找到带有`@Autowired`的注解Bean, 将其装配。

> 谈谈`@Autowired`的用法?

`@Autowired`在属性和setter方法上使用时, 是通过`byType`进行自动装配; 在Bean的构造函数上使用`@Autowired`时, 是通过`constructor`进行自动装配。当存在多个相同类型或名称的Bean时, 就要用`@Qualifier`指定Bean的名称。

> `@Autowired`与`@Resource`的区别?

`@Resource`的作用相当于`@Autowired`, 用法也大致相同。不同点:
* `@Autowired`是Spring的注解, `@Resource`是jdk1.6开始支持的注解。
* `@Autowired`默认按照Bean的类型自动装配, `@Resource`默认按照Bean名称自动装配(这也是最主要的区别)

### 7. Spring处理循环依赖

我们知道Spring IOC容器根据依赖注入的原则实现了控制反转。但是如果有这样一个场景如何解决?

> 当我们注入一个对象A时, 需要注入对象A中标记了某些注解的属性,这些属性也就是对象A的依赖。把对象A中的依赖都初始化完成, 对象A才算是创建完成。如果对象A中有一个属性是对象B, 而对象B中有个属性是对象A, 那么对象A和对象B在Spring中就会产生循环依赖, 如果不作处理, 就会出现**创建对象A --> 处理A的依赖B --> 创建对象B --> 处理B的依赖A --> 创建对象A --> 处理A的依赖B --> 创建对象B....**这样无限的循环下去。

Spring提供了一种方式来处理这种循环依赖。基本思路: **虽然初始化Bean, 必须要注入Bean里的依赖, 才算初始化成功, 但是并不要求Bean中依赖的依赖也要注入成功, 只要依赖对象的构造方法执行完了, 这个依赖对象就算已经存在了, 注入就算成功了, 至于依赖的依赖, 可以在后面继续初始化。所以, 初始化一个Bean, 先调用Bean的构造方法, 这个对象在内存中已经存在了(注意此时对象中的依赖还没有别注入), 然后保存这个对象, 当发生循环依赖时, 直接拿到之前保存的对象, 就不需要在创建对象了, 于是循环依赖就被终止了。**

例如:

> 对象A与对象B是循环依赖的。
> 1. 创建对象A, 调用A的构造, 并把A保存下来
> 2. 准备注入对象A的依赖, 发现对象A依赖对象A, 那么开始创建对象B
> 3. 调用B的构造, 并把B保存下来
> 4. 准备注入B的构造, 发现B依赖对象A, 对象A之前已经创建了, 直接获取A并把A注入B中(**注意此时的对象A还没有完全注入成功, 对象A中的对象B还没有注入**), 于是B创建成功
> 5. 把创建成功的B注入A, 于是A也创建成功。

在Spring源码中, 在注入对象的过程中, 调用了这个方法:

```java
// 这段代码在AbstractBeanFactory类的doGetBean()方法中

/**
* 返回值Objecy就是要创建的对象, beanName就是要创建的对象的类名
**/
Object sharedInstance = this.getSingleton(beanName);

/**
 * getSingleton具体实现
 */
@Nullable
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    // 首先从singletonObjecys中获取对象, 当Spring准备创建一个对象时, singletonObjects列表中是没有这个对象的
    Object singletonObject = this.singletonObjects.get(beanName);

    /**
     * 这里的判断除了判断 singletonObject==null 之外, 还有一个 isSingletonCurrentlyInCreation 的判断。
     * 因为当Spring初始化一个依赖注入的对象, 但还没注入对象属性的时候, Spring会把这个Bean加入 singletonCurrentlyInCreation这个Set集合中,
     * 也就是把这个对象标记为正在创建的状态, 这样如果Soring发现要创建的Bean在singletonObjects中没有, 但是 singletonCurrentlyInCreation中存在, 
     * 基本上就认定为循环依赖(在创建Bean的过程中发现又要创建这个Bean, 说明Bean的某个依赖又依赖了这个Bean)。
     * 
     */
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        synchronized (this.singletonObjects) {
            /**
             *  这里调用 earlySingletonObjects 列表, 这是为了循环依赖而存在的列表, 
             * 这是个预创建的对象列表, 刚刚创建的对象一般不会在这个列表里
             * 
             */
            singletonObject = this.earlySingletonObjects.get(beanName);

            /**
             * 如果 earlySingletonObjects列表中也没有则从 singletonFactories 中获取
             */
            if (singletonObject == null && allowEarlyReference) {
                ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                /**
                 * 走到这一步基本上就确定我们要创建的对象发生了循环依赖, 然后Spring获取singletonFactory总的对象, 将这个对象加入到
                 * earlySingletonObjects中, 然后把该对象从 singletonFactories中删除。
                 */
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
```

根据上面的源码可以看出, 解决循环依赖的关键方法使用三层列表来查询依赖的对象是否存在:

- **`singletonObjects`**
- **`earlySingletonObjects`**
- **`singletonFactories`**

**这里要注意的是处理出现依赖时, 对象会先进入`singletonFactories` 然后`earlySingletonObjects`, 然后`singletonObjects`**。

所以按照这个逻辑以及源码, 如果循环依赖, 此时`singletonFactories`中一般会存在目标对象, 例如：

对象A与对象B循环依赖, 初始化对象A(执行构造方法), 还没有注入对象A的依赖时, 就会把A放入`singletonFactories`, 
然后开始注入A的依赖, 发现对象B, 于是开始初始化B, 将对象B放入`singletonFactories`, 然后开始注入B的依赖, 又会发现A, 
根据上面的源码解析, 我们知道此时对象A已经在`singletonCurrentlyInCreation`列表中, 所以就会进入if逻辑判断, 不会再创建新的对象A了。

```java

/**
 * 执行完上面的方法代码之后, 还会执行下面方法的代码, 会把Bean完整的进行初始化和依赖注入, 
 */
getSingleton(String beanName, ObjectFactory<?> singletonFactory);


/**
 * getSingleton(String beanName, ObjectFactory<?> singletonFactory)中有一个子方法 addSingleton(String beanName, Object singletonObject)
 * 
 * 这个方法处理是已经注入完依赖的Bean, 把Bean放入singletonObjects中, 并把Bean从earlySingletonObjects和singletonFactories中删除, 这个方法和上面的方法组成了
 * Spring处理循环依赖的逻辑
 */
protected void addSingleton(String beanName, Object singletonObject) {
    synchronized (this.singletonObjects) {
        this.singletonObjects.put(beanName, singletonObject);
        this.singletonFactories.remove(beanName);
        this.earlySingletonObjects.remove(beanName);
        this.registeredSingletons.add(beanName);
    }
}
```

最后总结一下, Spring处理循环依赖在代码中的实现过程, 如下:

| 步骤 | 操作 | 列表数据变化 |
| -- | -- | -- | -- |
| 1 | 初始化对象A | singletonFactories: <br>earlySingletonObjects: <br>singletonObjects: |
| 2 | 调用A的构造, 把A放入singletonFactories(注意此时没有注入依赖) | singletonFactories: A <br>earlySingletonObjects: <br>singletonObjects: |
| 3 | 开始注入A的依赖, 发现A依赖对象B | singletonFactories:A <br>earlySingletonObjects: <br>singletonObjects: |
| 4 | 开始初始化对象B | singletonFactories:A,B <br>earlySingletonObjects: <br>singletonObjects: |
| 5 | 开始注入B的依赖, 发现B依赖对象A | singletonFactories:A,B <br>earlySingletonObjects: <br>singletonObjects: |
| 6 | 开始初始化对象A, 发现A在singletonFactories中已存在, 则直接获取A, 把A放入earlySingletonObjects, 并从singletonFactories删除A | singletonFactories:B <br>earlySingletonObjects:A <br>singletonObjects: |
| 7 | 对象B依赖注入完成 | singletonFactories:B <br>earlySingletonObjects:A <br>singletonObjects: |
| 8 | 对象B创建完成, 把B放入singletonObjects, 并从singletonFactories,earlySingletonObjects删除B | singletonFactories: <br>earlySingletonObjects:A <br> singletonObjects: B |
| 9 | 对象B注入给A, 继续注入A的其他依赖, 直到A注入完成 | singletonFactories:  <br>earlySingletonObjects:A <br>singletonObjects: B|
| 10 | 对象A创建完成, 把A放入singletonObjects, 并从singletonFactories,earlySingletonObjects删除A | singletonFactories:  <br>earlySingletonObjects: <br>singletonObjects: A, B |
| 11 | 循环依赖处理结束, A和B都初始化和注入完成 | singletonFactories:  <br>earlySingletonObjects: <br>singletonObjects: A, B|


