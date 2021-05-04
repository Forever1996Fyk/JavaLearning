## <center>Spring中用到的设计模式总结</center>

JDK中有哪些设计模式? Spring中有哪些设计模式?这两个问题都是面试中比较常见的。下面我就根据我自己的理解去总结一下Spring中的设计模式有哪些, 对于源码的解读并没有很深入, 主要还是总结。对于具体的设计模式的学习, 会在Java基础中作为重点推出。

### 1. 工厂模式

对于这个我们最熟悉的就是Spring的`BeanFactory`或者`ApplicationContext`创建Bean对象。

**两者对比:**
- `BeanFactory:` 延迟注入(使用某个bean时才会注入), 相比如`ApplicationContext`内存占用更少, 启动速度更快。

- `ApplicationContext:` 容器启动时, 一次性创建所有Bean。`ApplicationContext`扩展了`BeanFactory`。

### 2. 单例模式

**使用单例模式的好处**：
- 对于频繁使用的对象, 可以省略创建对象所花费的时间。
- 由于new操作的减少, 也减小了系统内存的压力, 减轻GC压力, 缩短GC停顿时间。

Spring中Bean的默认作用域就是singleton(单例)的, 在源码中Spring通过`ConcurrentHashMap`实现对Bean的注册, 保持Bean的唯一性, 从而实现单例模式。下面是Spring实现单例Bean的核心代码:
```java
// 通过 ConcurrentHashMap（线程安全） 实现单例注册表
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<String, Object>(64);

public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
        Assert.notNull(beanName, "'beanName' must not be null");
        synchronized (this.singletonObjects) {
            // 检查缓存中是否存在实例  
            Object singletonObject = this.singletonObjects.get(beanName);
            if (singletonObject == null) {
                //...省略了很多代码
                try {
                    singletonObject = singletonFactory.getObject();
                }
                //...省略了很多代码
                // 如果实例对象在不存在，我们注册到单例注册表中。
                addSingleton(beanName, singletonObject);
            }
            return (singletonObject != NULL_OBJECT ? singletonObject : null);
        }
    }
    //将对象添加到单例注册表
    protected void addSingleton(String beanName, Object singletonObject) {
            synchronized (this.singletonObjects) {
                this.singletonObjects.put(beanName, (singletonObject != null ? singletonObject : NULL_OBJECT));

            }
        }
}

```

### 3. 代理模式

#### 代理模式在AOP中的应用

这个在[Spring_AOP](/develop_framework/Spring/Spring_AOP.md)中已经提到过了, Spring AOP的原理就是动态代理, 也就使用了代理模式。


### 4. 模板方法模式

Spring中`JdbcTemplate`, `HibernateTemplate`等以Template结尾的对数据库操作的类, 它们就是用了模板模式。一般情况下, 我们都是使用继承的方式来实现模板模式, 但是在Spring中使用Callback模式与模板方法模式结合, 既达到了代码复用的效果, 同时增加了灵活性。

> 什么是模板方法模式?

模板方法模式是一种行为设计模式, 它定义一个操作的算法的框架, 但是把一些具体的实现步骤延迟到子类中, 使得子类可以不改变一个算法的结构就可以重定义该算法的实现方式。
```java
public abstract class Template {
    //这是我们的模板方法
    public final void TemplateMethod(){
        PrimitiveOperation1();  
        PrimitiveOperation2();
        PrimitiveOperation3();
    }

    protected void  PrimitiveOperation1(){
        //当前类实现
    }

    //被子类实现的方法
    protected abstract void PrimitiveOperation2();
    protected abstract void PrimitiveOperation3();

}
public class TemplateImpl extends Template {

    @Override
    public void PrimitiveOperation2() {
        //当前类实现
    }

    @Override
    public void PrimitiveOperation3() {
        //当前类实现
    }
}
```

### 5. 观察者模式

观察者模式是一种对象行为模式。对象与对象之间有依赖关系, 当一个对象发生改变时, 所依赖的对象也会做出反应。

**Spring事件驱动模型**就是观察者模式很经典的应用, 而且很多场景都可以解耦业务代码。比如我们每次添加商品时都需要重新更新商品索引, 这个时候就可以利用观察者模式解决问题。

#### Spring事件驱动模型的三种角色

##### 事件角色

`ApplicationEvent`充当事件角色, 继承了`EventObject`并实现了`Serializable`接口。

Spring中默认的事件都是继承并实现`ApplicationEvent`类:
- `ContextStartedEvent`: `ApplicationContext`启动后触发的事件。
- `ContextStoppedEvent`: `ApplicationContext`停止后触发的事件。
- `ContetRefreshedEvent`: `ApplicationContext`初始化或刷新完成后触发的事件。
- `ContextCloseEvent`: `ApplicationContext`关闭后触发的事件。

![ApplicationEvent抽象UML图](/image/ApplicationEvent抽象UML.png)

##### 事件监听者角色

`ApplicationListener`充当事件监听者角色, 它是一个接口只有一个方法`onApplicationEvent()`来处理`ApplicationEvent`。

在实际开发中我们需要实现`ApplicationListener`接口实现`onApplicationEvent()`方法即可完成监听事件。

```java
package org.springframework.context;
import java.util.EventListener;

@FunctionalInterface
public interface ApplicationListener<E extends ApplicationEvent> extends EventListener {
    void onApplicationEvent(E var1);
}
```

##### 事件发布者角色

`ApplicationEventPublish`充当事件发布者, 它也是一个接口。实现这个方法的源码比较复杂, 所以这里就不在分析。

```java
@FunctionalInterface
public interface ApplicationEventPublisher {
    default void publishEvent(ApplicationEvent event) {
        this.publishEvent((Object)event);
    }

    void publishEvent(Object var1);
}
```

##### Spring的事件流程总结

1. 定义一个事件: 实现一个继承自`ApplicationEvent`, 并且重写相应的构造函数。
2. 定义一个事件监听者: 实现`ApplicationListener`接口, 重写`onApplicationEvent()`方法。
3. 使用事件发布者发布消息: 可以通过`ApplicationEventPublisher`的`publishEvent()`方法发布消息。

代码示例:

```java
// 定义一个事件,继承自ApplicationEvent并且写相应的构造函数
public class DemoEvent extends ApplicationEvent{
    private static final long serialVersionUID = 1L;

    private String message;

    public DemoEvent(Object source,String message){
        super(source);
        this.message = message;
    }

    public String getMessage() {
         return message;
          }


// 定义一个事件监听者,实现ApplicationListener接口，重写 onApplicationEvent() 方法；
@Component
public class DemoListener implements ApplicationListener<DemoEvent>{

    //使用onApplicationEvent接收消息
    @Override
    public void onApplicationEvent(DemoEvent event) {
        String msg = event.getMessage();
        System.out.println("接收到的信息是："+msg);
    }

}
// 发布事件，可以通过ApplicationEventPublisher  的 publishEvent() 方法发布消息。
@Component
public class DemoPublisher {

    @Autowired
    ApplicationContext applicationContext;

    public void publish(String message){
        //发布事件
        applicationContext.publishEvent(new DemoEvent(this, message));
    }
}

当调用 DemoPublisher 的 publish() 方法的时候，比如 demoPublisher.publish("你好") ，控制台就会打印出:接收到的信息是：你好 。

```

### 6. 适配器模式

适配器模式(Adapter Pattern)将一个接口转换成用户希望的另一个接口, 适配器模式使接口不兼容的类一起工作。

#### Spring AOP中的适配器模式

上面提到了Spring AOP的代理模式, 但是Spring AOP的`增强/通知(Advice)`使用到了适配器模式, 核心的接口就是`AdvisorAdapter`。

Advice常用的类型有: `BeforeAdvice(前置通知)`, `AfterAdvice(后置通知)`, `AfterReturningAdvice(返回通知)`, `AroundAdvice(环绕通知)`, `ThrowAdvice(异常通知)`。每个类型Advice都有对应的拦截器: `MethodBeforeAdviceInterceptor` ... 等等。Spring每种类型的通知Advice要通过适配器, 适配成`MethodInterceptor`接口类型的对象。(`MethodBeforeAdviceInterceptor`负责适配`MethodBeforeAdvice`)

#### Spring MVC中的适配器模式

在Spring MVC中, `DispatcherServlet`根据请求信息调用`HandlerMapping`, 解析请求对应的`Handler`(也就是我们常用的`Controller`控制器), 开始由`HandlerAdapter`适配器处理。`HandlerAdpoter`作为期望接口, 具体的适配器实现类用于对目标类进行适配, `Controller`作为需要适配的类。

> 为什么要在Spring MVC中使用适配器模式?

Spring MVC中的`Controller`种类比较多, 不同类型的`Controller`通过不同的方法对请求进行处理。如果不利用适配器模式, `DispatcherServlet`直接获取对应类型的`Controller`, 需要自行判断, 如同下面的代码一样:

```java
if(mappedHandler.getHandler() instanceof MultiActionController){  
   ((MultiActionController)mappedHandler.getHandler()).xxx  
}else if(mappedHandler.getHandler() instanceof XXX){  
    ...  
}else if(...){  
   ...  
}  

```

看到上面的代码估计大家都明白了, 如果在增加一个`Controller`类型, 就要在上面代码中再加入一行判断语句, 这样程序就难以维护, 也违背了设计模式的开闭原则, 即对扩展开放, 对修改关闭。


### 7. 装饰者模式

装饰者模式可以动态的给对象添加一些额外的属性或者行为。比较通俗的说, 我们想要修改原有的功能, 但是无码又不愿直接取修改原有的代码时, 设计一个Decorator套在原来的代码外面。

在JDK中`InputStream`就是用的这种模式。

Spring中用到装饰器模式在类名上会包含`Wrapper`或者`Decorator`。这些类基本上都是动态地给一个对象添加一些额外的职责。