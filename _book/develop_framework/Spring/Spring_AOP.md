## <center>Spring AOP</center>

Spring AOP(Aspect-Oriented Programming)面向切面编程, 是Spring极为重要的特性之一, 在实际的开发中也经常会用到, 也是面试提问高频知识点。

在一开始学习[Spring](/develop_framework/Spring/Spring.md)的时候, 就已经提到AOP的基本概念和作用。
这里无码要深入的学习AOP的核心技术, 因为太重要了!!!


> 谈谈你对Spring AOP的理解? <font color="red">(我觉得如果面试官问这样的问题, 最好是比较白话的方式去回答, 而不要用太多的专业术语, 因为如果太多专业术语, 会感觉你是从网上背下来的, 而不是你自己的理解。)</font>

AOP是OOP(面向对象编程)的一种延续, AOP的目的也是为了解决代码重复问题, 降低模块之间的耦合度。 只不过它并不像普通的代码抽离到一个通用方法, 然后调用。 因为即使抽取成类或者方法, 还是会出现重复的代码, 依附在我们业务类的方法逻辑中, 在我们的业务代码中反复调用。所以我们需要将这些与业务无关, 但是相同的代码抽取到一个独立的模块中, 利用动态代理的技术, 将这些模块织入到目标对象方法中。这就是AOP做的事, 或者说AOP的原理就是动态代理。

如果不了解动态代理, 可以去看我以前一篇动态代理学习的文章[https://www.cnblogs.com/FanJava/p/9700561.html](https://www.cnblogs.com/FanJava/p/9700561.html)

### 1. AOP的专业术语

> 我这里尽量的简化了解释, 如果要了解具体的可以去Spring中文文档[https://www.w3cschool.cn/wkspring/izae1h9w.html](https://www.w3cschool.cn/wkspring/izae1h9w.html)

其实专业术语是不太好理解的, 最好学习的方式就是用一个代码示例,来理解。
```java

//业务代码接口
public interface UserService {
    void save();
    void delete();
    void update();
    void find();
}

//业务代码接口实现类
public class UserServiceImpl implements UserService {
    //这些方法都是业务逻辑，一般来说在进行业务方法进行事务管理
    @Override
    public void save() {
        //如果不使用动态代理每个方法都需要进行打开事务，提交事务的操作
        //System.out.println("打开事务");
        System.out.println("保存用户");
        //System.out.println("提交事务");
    }

    @Override
    public void delete() {
        System.out.println("删除用户");
    }

    @Override
    public void update() {
        System.out.println("修改用户");
    }

    @Override
    public void find() {
        System.out.println("查找用户");
    }
}


// 创建代理工厂代理Service
public class UserServiceProxyFactory implements InvocationHandler{

    private UserService us;

    public UserServiceProxyFactory(UserService us) {
        super();
        this.us = us;
    }

    public UserService getUserServiceProxy() {
        //生成动态代理
        UserService usProxy  = (UserService) Proxy.newProxyInstance(UserServiceProxyFactory.class.getClassLoader(),
                UserServiceImpl.class.getInterfaces(), this);//第三个参数，必须要实现InvocationHandler接口

        return usProxy;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        //使用动态代理，只需要写一次即可
        System.out.println("打开事务");
        Object invoke = method.invoke(us, args);//方法的执行，需要方法所存在的对象，所以需要有实例。创建一个构造方法，来传递一个执行方法所对应的实例对象
        System.out.println("提交事务");
        return invoke;
    }
}

public class Demo {
    public static void main(String[] args) {
        UserService us = new UserServiceImpl();
        UserServiceProxyFactory factory = new UserServiceProxyFactory(us);
        UserService usProxy = factory.getUserServiceProxy();
        usProxy.save();
    }
}
```

> AOP的专业术语有哪些, 具体说它们的意思?(或者直接问相关术语的解释)
- **连接点(Join point)**:
    Spring AOP基于动态代理, 所以是方法拦截。目标对象, 每个成员方法都可以成为连接点。(UserServiceImpl中的save(), delete(), update(), find()都可以称为连接点)

- **切点(Pointcut)**:
    上面知道了每个方法都称为连接点, 如果具体定位到某一个方法就成为切点。(在Demo类中, 执行了目标对象的save()方法, 所以save()方法就是切点)

- **目标对象(Target Object)**:
    被代理的对象。(上面代码中的UserService就是目标对象)

- **增强/通知(Advice)**:
    表示添加到切点的逻辑代码(例如: 事务管理, 日志管理...), 并确定这些逻辑代码添加在切点的位置(Spring AOP提供了5中Advice类型: 前置, 后置, 返回, 异常, 环绕)。

    (在上面代码中"打开事务", "关闭事务"的代码就是 `增强/通知`)

- **织入(Weaving)**:
    将`增强/通知`添加到目标对象的过程。

- **引入/引介(Introduction)**:
    允许添加新方法或属性到现有的类中。

- ***切面(Aspect)**:
    `切点`和`增强/通知`共同组成一个切面。现在由于Spring Boot大热, 注解开发也是火热。Spring AOP也有注解开发。所以使用`@Aspect`注解的类就是切面


### 2. Spring AOP原理

上面已经说过Spring AOP的原理就是动态代理, 所以这里主要说一下动态代理的原理。

> 什么是动态代理?

简单来说, 就是本来让我自己做的事, 交给别人来做, 请的人就是代理对象。**在程序运行过程中产生代理对象就是动态代理, 而程序运行中能够产生对象的原理就是反射技术。**
也就是说, 代理类并不是在Java代码中定义的, 而是在运行时运用反射技术动态生成的, 可以更灵活的对代理类的方法进行统一处理, 而不用修改每个代理类的方法。

> 动态代理的实现原理是什么?

**定义一个实现`InvocationHandler`接口的中介类, 并实现`invoke()`方法, 当调用代理类对象的方法时, 这个调用会转送到`invoke()`方法中。 在`invoke()`方法中对代理对象的调用方法进行增强。** 

参考这篇文章写得很详细[https://juejin.im/post/5ad3e6b36fb9a028ba1fee6a](https://juejin.im/post/5ad3e6b36fb9a028ba1fee6a)


### 3. Spring AOP和AspectJ AOP有什么区别?(我觉得一般面试不会问这个问题, 但是还是要了解一下)

**Spring AOP属于运行时增强, 而AspectJ是编译时增强。**Spring AOP基于代理(Proxying), 而AspectJ基于字节码操作。

Spring AOP集成了AspectJ, 而AspectJ比Spring AOP功能更加强大, 但是Spring AOP更加简单。

如果切面比较少, 两者性能差异不大。但是, 当切面太多的话, 最好选择AspectJ, 因为AspectJ 比 Spring AOP效率更高。



