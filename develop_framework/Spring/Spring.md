## Spring基础知识点

Spring的重要性应该不需要说明了。 这篇文章主要是加深对Spring的理解, 对之前Spring学习的一个总结。最主要的还是针对面试中常见的问题所延伸出来的思考。自己在学习的过程中也是结合以前写的博客以及查阅了很多资料, 自己总结出来的理解。

我并不建议去阅读Spring的源码, 因为我认为这对我们没有什么太大的帮助而且很枯燥。所以这篇文章对源码的解析很少, 最主要是根据一些概念举出实例。

### 1. 如何学习Spring?
二话不说, 学习Spring的最佳途径是官网。Spring 官网：[https://spring.io/](https://spring.io/)。但是这是全英文的网站, 所以即使翻译过来, 有的专业词汇也可能很模糊, 所以这里提供一个中文的网站[https://www.w3cschool.cn/wkspring/pesy1icl.html](https://www.w3cschool.cn/wkspring/pesy1icl.html)

不管是学习Spring Boot还是 SpringCloud, 官网的文档才是真正开始的地方!!! 因为里面几乎涵盖了框架中所有的模块以及解释.

### 2. Spring模块
Spring官网列出的Spring的6个特征:
* 核心技术: 依赖注入(DI), 面向切面编程(AOP), 事件(events), 资源, i18n, 验证, 数据绑定, 类型转换, SpEL。
* 测试: 模拟对象, TestContext框架, Spring MVC测试, WebTestClient。
* 数据访问: 事务, DAO支持, JDBC, ORM, 编组XML。
* Web支持: Spring MVC和Spring WebFlux Web 框架。
* 集成: 远程处理, JMS, JCA, JMX, 电子邮件, 任务调度, 缓存。
* 语言: Kotlin, Groovy, 动态语言。

> 面试官有可能会问: **你既然用过Spring, 那你简单的描述一下, Spring常用的模块有哪些?**

<font color="red">这个问题看似可能很小, 但是面试官可能会从你对这些模块的了解程度上进行判断, 你在学习使用某个框架的时候是否认真的去了解这个框架, 还是只是会用而已。</font>

* Spring Core: Spring核心, 可以说Spring其他所有功能都需要依赖该类库。主要提供IOC依赖注入功能。
* Spring Aspects: 该模块与AspectJ的集成提供支持。
* Spring AOP: 提供了面向切面编程的实现。
* Spring JDBC: Java数据库连接。
* Spring JMS: Java消息服务。
* Spring ORM: 用于支持Hibernate等ORM工具。
* Spring Web: 为创建Web应用程序提供支持。
* Spring Test: 提供第JUnit和TestNG测试的支持。

对于上面的问题, 面试官可能会用另一种方式提问, 但是我们自己在学习过程应该也会有所疑问, 那就是为什么要用Spring?
我们就可以根据Spring常用模块的方面来进行解释了。

### 3. 为什么要用Spring?

> <font color="red">这个问题如果是面试官提出, 你可以简化回答, 也可以具体回答, 因为不管你怎么回答, 面试官也一定会延伸的问你具体的描述!!!</font>

几乎Spring所做的任何事, 最核心的就是： **简化Java的开发**。 其实你可以通过上面的官方文档进行回答, 但是文档毕竟涵盖的内容较多, 最主要的是总结成几句话, 方便理解。具体回答的点以及说明: 

- **轻量级, 非侵入性编程**

    (1) Spring是轻量级的框架, 所谓轻量级就是从大小和开销两方面Spring都是轻量的, 完整的Spring框架可以在一个1MB多的JAR文件中发布, 而且Spring所需要的的处理开销也是很小的。

    (2) 非侵入性, Spring不会强迫程序实现它的接口或继承它的类, 最多可能使用Spring注解。也就是说, 即使你用了Spring, 你写的代码在非Spring应用中也发挥相同的作用。

- **通过依赖注入和面向接口实现松耦合**

    依赖注入(DI)和控制反转(IOC)其实表达的是同一个东西, 简单来说就是, 任何一个有意义的应用中, 肯定会有多个类组成。
    没有Spring的时候, 对象负责管理自己的相互关联的对象引用, 这样就会导致高耦合和难以测试的代码。代码示例方便理解:
    ```java
        // 未使用Spring的情况
        /**
         * Train类在自己的构造器中创建了People对象, 这就造成了对象之间的紧耦合。这个火车可以载人, 但是如果这个火车载物品可能就不行了
         */
        public class Train implements Transport{
            private People people;
            public Train(People people) {
                people = new People();
            }
            public void catchGoods(){
                people.doSomthing();
            }
        }

    ```
    **耦合是具有两面性, 一方面紧密的耦合代码, 难以测试, 难以理解, 修改一处可能会引起别的bug, 另一方面完全没有耦合的代码什么都干不了**

    使用Spring之后, 对象的依赖关系由负责关联的对象的第三方组件(例如 构造器)来完成, 对象不需要自行创建, 依赖注入(DI)会将所依赖的关系自动交给目标对象, 而不是让对象自己获取, 这也就是控制反转(IOC)的核心
    ```java
        // 使用Spring的情况
        /**
         * People对象不再有Train自行创建, 而是使用构造器参数传进来, 这就是依赖注入: 构造器注入,就实现了松耦合。
         */
        public class Train implements Transport{
            private People people;
            public Train(People people) {
                this.people = people;
            }
            public void catchGoods(){
                people.doSomthing();
            }
        }
    ```

    **创建应用组件之间关联的行为通常称为装配!!!** Spring有多种装配Bean的方式： XML, 基于Java的注解配置等等方式。

    **负责对象的创建和组装的功能是由Spring应用上下文完成, Spring有多种上下文的实现, 它们的区别主要在于如何加载配置。** Spring上下文, Bean等概念会在[Spring IOC & Bean](develop_framework/Spring/IOC_Bean.md)中进行解析

    IOC和DI具体的原理实现会在[Spring IOC & Bean](develop_framework/Spring/IOC_Bean.md)中进行分析。

- **面向切面编程**

    系统是由不同的组件组成, 除了实现业务功能, 通常还有些贯穿整个项目的各个组件。如果没有系统性的处理这部分, 那么代码中会含有大量的重复代码。如果你只是吧这些单独抽取出来作为一个模块, 让其他模块只是调用它的方法, 那这个方法还是会出现在各个模块。

    AOP是面向切面编程, 它将那些贯穿整个项目的组件服务模块化, 并以**声明的方式应用到需要的模块**, 这样其他模块只关注自身的业务, 不需要了解这些服务的相关逻辑和代码。

    用专业术语解释就是, 通过预编译方式和运行期间动态代理实现程序功能统一维护的技术。利用AOP可以对业务逻辑的各个部分进行隔离，从而使得业务逻辑各部分之间的耦合度降低，提高程序的可重用性，同时提高了开发的效率。**所以AOP的原理是动态代理**

    > AOP(面向切面编程)与OOP(面向对象编程)的区别?
        * OOP是针对业务处理过程的实体及其属性和行为进行抽象封装, 使得业务逻辑划分更加清晰;
        * AOP是针对业务处理过程中的某个步骤, 或者阶段, 降低业务逻辑各部分之间的耦合性。
        举一个例子, 银行取钱。 这一行为涉及到的取钱人, 办理人, 经理等等业务实体, 以及他们的行为, 所具有的属性, 肯定是用过OOP方式进行设计; 而进入银行需要权限验证, 这一动作或者步骤, 是AOP的领域。

    AOP具体的原理实现会在[Spring AOP](/develop_framework/Spring/Spring_AOP.md)中进行分析。


    





    





